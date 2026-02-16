# ElastiCache

## Overview

```
ElastiCache = Managed in-memory caching service

Engines:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Redis                              Memcached                          │
│   ├── Data structures (strings,      ├── Simple key-value              │
│   │   lists, sets, hashes, etc.)     ├── Multi-threaded                │
│   ├── Persistence options            ├── No persistence                │
│   ├── Replication & HA               ├── No replication                │
│   ├── Pub/Sub                        ├── Simple scaling                │
│   ├── Lua scripting                  └── Lower memory overhead         │
│   ├── Cluster mode                                                      │
│   └── Geospatial support                                                │
│                                                                          │
│   Choose Redis for:                  Choose Memcached for:             │
│   - Complex data types               - Simple caching                  │
│   - Persistence required             - Multi-threaded perf             │
│   - Pub/Sub needed                   - Large cache pools               │
│   - Replication/HA needed            - Simplest use case               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Redis Architecture

### Cluster Mode Disabled

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Redis Cluster Mode Disabled                           │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    Replication Group                             │   │
│   │                                                                  │   │
│   │   Primary                    Replicas                           │   │
│   │   (Read/Write)               (Read-only)                        │   │
│   │                                                                  │   │
│   │   ┌─────────┐    async       ┌─────────┐    ┌─────────┐        │   │
│   │   │ Primary │───────────────▶│ Replica │    │ Replica │        │   │
│   │   │  Node   │                │  Node   │    │  Node   │        │   │
│   │   │ (AZ-a)  │───────────────▶│ (AZ-b)  │    │ (AZ-c)  │        │   │
│   │   └─────────┘                └─────────┘    └─────────┘        │   │
│   │                                                                  │   │
│   │   Endpoints:                                                     │   │
│   │   - Primary: writes                                             │   │
│   │   - Reader: load-balanced reads                                 │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Max: 1 shard, up to 5 read replicas                                   │
│   Use when: Data fits in one node, need HA                              │
└─────────────────────────────────────────────────────────────────────────┘
```

### Cluster Mode Enabled

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Redis Cluster Mode Enabled                            │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    Sharded Cluster                               │   │
│   │                                                                  │   │
│   │   Shard 1            Shard 2            Shard 3                 │   │
│   │   (Slots 0-5460)     (Slots 5461-10922) (Slots 10923-16383)    │   │
│   │                                                                  │   │
│   │   ┌─────────┐        ┌─────────┐        ┌─────────┐            │   │
│   │   │ Primary │        │ Primary │        │ Primary │            │   │
│   │   └────┬────┘        └────┬────┘        └────┬────┘            │   │
│   │        │                  │                  │                  │   │
│   │   ┌────┴────┐        ┌────┴────┐        ┌────┴────┐            │   │
│   │   │ Replica │        │ Replica │        │ Replica │            │   │
│   │   └─────────┘        └─────────┘        └─────────┘            │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Data Distribution:                                                     │
│   - Keys hashed to slots (0-16383)                                      │
│   - Each shard owns range of slots                                      │
│   - Online resharding supported                                          │
│                                                                          │
│   Max: 500 shards, 500 replicas total                                   │
│   Use when: Need > 500 GB or > 1M ops/sec                               │
└─────────────────────────────────────────────────────────────────────────┘
```

### Multi-AZ & Automatic Failover

```
Failover Process:
1. Primary node fails
2. ElastiCache detects failure (~30 seconds)
3. Promotes replica to primary
4. Updates DNS endpoint
5. New primary accepts writes

Total failover time: ~60 seconds

Configuration:
- Multi-AZ: Distribute replicas across AZs
- Auto-failover: Automatic promotion
- Minimum 1 replica for failover
```

---

## Caching Strategies

### Cache-Aside (Lazy Loading)

```
Application manages cache explicitly

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. App checks cache                                                    │
│   ┌─────────┐    get(key)    ┌─────────┐                               │
│   │   App   │───────────────▶│  Cache  │                               │
│   └─────────┘◀───────────────└─────────┘                               │
│        │      cache miss (null)                                         │
│        │                                                                │
│   2. App reads from DB                                                  │
│        │       select        ┌─────────┐                               │
│        └───────────────────▶│   DB    │                               │
│        ◀───────────────────-└─────────┘                               │
│               data                                                      │
│                                                                          │
│   3. App updates cache                                                   │
│        │       set(key,data) ┌─────────┐                               │
│        └───────────────────▶│  Cache  │                                │
│                              └─────────┘                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Pros:
- Only requested data cached
- Cache failures don't break app
- Simple to implement

Cons:
- Cache miss penalty (extra round trip)
- Data can become stale
- Initial request always slow

// Code pattern
async function getUser(userId) {
  let user = await cache.get(`user:${userId}`);

  if (!user) {
    user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);
    await cache.set(`user:${userId}`, user, { EX: 3600 });
  }

  return user;
}
```

### Write-Through

```
Write to cache and database synchronously

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. App writes to cache                                                 │
│   ┌─────────┐    set(key,data)   ┌─────────┐                           │
│   │   App   │───────────────────▶│  Cache  │                           │
│   └─────────┘                    └────┬────┘                           │
│                                       │                                 │
│   2. Cache writes to DB               │                                 │
│                                       │ write                           │
│                                       ▼                                 │
│                                  ┌─────────┐                           │
│                                  │   DB    │                           │
│                                  └─────────┘                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Pros:
- Cache always consistent
- Simplified read path
- No stale data

Cons:
- Write latency increased
- Cache churn (unused data cached)
- Need cache + DB transaction handling

Use with: DAX (DynamoDB Accelerator) does this automatically
```

### Write-Behind (Write-Back)

```
Write to cache, async write to database

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. App writes to cache (immediate return)                             │
│   ┌─────────┐    set(key,data)   ┌─────────┐                           │
│   │   App   │───────────────────▶│  Cache  │                           │
│   └─────────┘                    └────┬────┘                           │
│        ◀── ack                        │                                 │
│                                       │ async (batched)                 │
│   2. Cache writes to DB               │                                 │
│                                       ▼                                 │
│                                  ┌─────────┐                           │
│                                  │   DB    │                           │
│                                  └─────────┘                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Pros:
- Fast writes
- Batch DB writes (efficiency)
- Handles DB failures gracefully

Cons:
- Risk of data loss (cache failure)
- Eventual consistency
- Complex failure handling

Use for: High write throughput, acceptable data loss risk
```

### Cache Invalidation

```
Invalidation Strategies:

1. Time-based (TTL)
   - Set expiration on cache entries
   - Simple, eventual consistency
   cache.set('user:123', data, { EX: 3600 })  // 1 hour

2. Event-based
   - Invalidate on database changes
   - More complex, better consistency

   // On user update
   await db.updateUser(userId, data);
   await cache.del(`user:${userId}`);

3. Version-based
   - Include version in cache key
   - No explicit invalidation needed

   cache.set(`user:${userId}:v${version}`, data)

Common Patterns:
┌─────────────────────────────────────────────────────────────────────────┐
│ Pattern          │ When to Use                                          │
├──────────────────┼──────────────────────────────────────────────────────┤
│ Short TTL        │ Frequently changing data, acceptable staleness       │
│ Long TTL + Event │ Stable data, strong consistency required            │
│ No TTL + Event   │ Critical data, manual control                       │
│ Versioning       │ Complex relationships, multiple cache locations     │
└──────────────────┴──────────────────────────────────────────────────────┘
```

---

## Common Use Cases

### Session Store

```javascript
// Store session in Redis
const session = {
  userId: '123',
  email: 'user@example.com',
  cart: ['item1', 'item2'],
  loginTime: Date.now()
};

await redis.hset('session:abc123', session);
await redis.expire('session:abc123', 3600);  // 1 hour TTL

// Retrieve session
const sessionData = await redis.hgetall('session:abc123');

// Advantages:
// - Fast access across all app servers
// - Auto-expiration
// - Survives server restarts
```

### Rate Limiting

```javascript
// Sliding window rate limiter
async function checkRateLimit(userId, limit, windowSecs) {
  const key = `ratelimit:${userId}`;
  const now = Date.now();
  const windowStart = now - (windowSecs * 1000);

  // Remove old entries
  await redis.zremrangebyscore(key, 0, windowStart);

  // Count requests in window
  const count = await redis.zcard(key);

  if (count >= limit) {
    return false;  // Rate limited
  }

  // Add current request
  await redis.zadd(key, now, `${now}-${Math.random()}`);
  await redis.expire(key, windowSecs);

  return true;
}

// Usage: 100 requests per minute
const allowed = await checkRateLimit('user:123', 100, 60);
```

### Leaderboard

```javascript
// Add/update score
await redis.zadd('leaderboard:game1', score, `user:${userId}`);

// Get top 10
const top10 = await redis.zrevrange('leaderboard:game1', 0, 9, 'WITHSCORES');

// Get user's rank
const rank = await redis.zrevrank('leaderboard:game1', `user:${userId}`);

// Get users around a specific rank
const nearby = await redis.zrevrange('leaderboard:game1', rank - 5, rank + 5, 'WITHSCORES');
```

### Pub/Sub

```javascript
// Publisher
await redis.publish('notifications', JSON.stringify({
  type: 'new_message',
  userId: '123',
  message: 'Hello!'
}));

// Subscriber
const subscriber = redis.duplicate();
await subscriber.subscribe('notifications');

subscriber.on('message', (channel, message) => {
  const notification = JSON.parse(message);
  // Handle notification
});

// Use cases:
// - Real-time notifications
// - Chat systems
// - Event broadcasting
// - Cache invalidation across instances
```

### Distributed Locking

```javascript
// Acquire lock
async function acquireLock(lockName, ttlMs) {
  const lockKey = `lock:${lockName}`;
  const lockValue = `${process.pid}-${Date.now()}`;

  const acquired = await redis.set(lockKey, lockValue, {
    NX: true,  // Only if not exists
    PX: ttlMs  // Expiration in ms
  });

  return acquired ? lockValue : null;
}

// Release lock
async function releaseLock(lockName, lockValue) {
  const lockKey = `lock:${lockName}`;

  // Lua script for atomic check-and-delete
  const script = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `;

  return await redis.eval(script, 1, lockKey, lockValue);
}
```

---

## Performance & Scaling

### Node Types

```
Node Type Selection:
┌─────────────────────────────────────────────────────────────────────────┐
│ Type          │ Memory    │ Network     │ Use Case                      │
├───────────────┼───────────┼─────────────┼───────────────────────────────┤
│ cache.t3.micro│ 0.5 GB    │ Low         │ Dev/test                      │
│ cache.r6g.large│ 13.07 GB │ Up to 10 Gbps│ Production (ARM, cost-effective)│
│ cache.r6g.xlarge│26.32 GB │ Up to 10 Gbps│ Medium workloads            │
│ cache.r6g.4xlarge│105 GB  │ Up to 10 Gbps│ Large datasets              │
│ cache.r6g.16xlarge│419 GB │ 25 Gbps     │ Very large datasets          │
└───────────────┴───────────┴─────────────┴───────────────────────────────┘

Graviton (r6g, m6g):
- Up to 20% better price/performance
- ARM-based processors
- Recommended for most workloads
```

### Scaling Strategies

```
Vertical Scaling:
- Change node type
- Requires replication group for zero downtime
- Cluster mode disabled: Failover during scaling
- Cluster mode enabled: Rolling update

Horizontal Scaling:
┌─────────────────────────────────────────────────────────────────────────┐
│ Scenario                     │ Solution                                 │
├──────────────────────────────┼──────────────────────────────────────────┤
│ Read scaling                 │ Add read replicas (up to 5 per shard)   │
│ Write scaling                │ Enable cluster mode, add shards         │
│ Data size > node memory      │ Enable cluster mode, distribute data    │
│ High availability            │ Multi-AZ with auto-failover            │
└──────────────────────────────┴──────────────────────────────────────────┘

Online Resharding (Cluster Mode):
- Add/remove shards without downtime
- Automatic slot rebalancing
- May impact performance during resharding
```

---

## Security

### Network Security

```
VPC Configuration:
┌─────────────────────────────────────────────────────────────────────────┐
│                              VPC                                         │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    Private Subnets                               │   │
│   │                                                                  │   │
│   │   ┌───────────────────┐    ┌───────────────────┐                │   │
│   │   │   ElastiCache     │    │   ElastiCache     │                │   │
│   │   │   Primary         │    │   Replica         │                │   │
│   │   │   Security Group  │    │   Security Group  │                │   │
│   │   │   - Port 6379     │    │   - Port 6379     │                │   │
│   │   │   - Source: App SG│    │   - Source: App SG│                │   │
│   │   └───────────────────┘    └───────────────────┘                │   │
│   │              ▲                      ▲                            │   │
│   └──────────────┼──────────────────────┼────────────────────────────┘   │
│                  │                      │                                │
│   ┌──────────────┴──────────────────────┴────────────────────────────┐   │
│   │                    Application Tier                               │   │
│   │   ┌───────────────────┐                                          │   │
│   │   │   EC2 / Lambda    │                                          │   │
│   │   │   (App SG)        │                                          │   │
│   │   └───────────────────┘                                          │   │
│   └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Encryption & Authentication

```
Encryption at Rest:
- AES-256 encryption
- Enabled at cluster creation
- Minimal performance impact

Encryption in Transit:
- TLS connections
- Enabled per cluster

Redis AUTH:
- Password authentication
- Set AUTH token at creation
- Required for Redis 6.0+ RBAC

// Redis AUTH
const redis = new Redis({
  host: 'mycluster.xxxxx.cache.amazonaws.com',
  port: 6379,
  password: 'my-auth-token',
  tls: {}  // Enable TLS
});

Redis RBAC (6.0+):
- User-based authentication
- ACL rules per user
- Granular permissions (read, write, commands)
```

---

## Interview Discussion Points

### How do you ensure cache consistency?

```
Strategies by consistency requirement:

Strong Consistency:
1. Write-through cache
   - Update cache and DB together
   - Accept write latency

2. Cache-aside with TTL=0
   - Always fetch from DB
   - Cache for repeated reads within request

Eventual Consistency:
1. Short TTL
   - Acceptable staleness window
   - Simple implementation

2. Event-driven invalidation
   - DB change triggers cache invalidation
   - Near real-time consistency

3. Version-based keys
   - Include version in cache key
   - Old versions naturally expire

Common Pattern:
- Write: Update DB, then invalidate cache (not update)
- Read: Cache-aside with appropriate TTL
- Why invalidate vs update: Simpler, avoids race conditions
```

### How do you handle cache failures?

```
Resilience Strategies:

1. Circuit Breaker
   - Detect cache failures
   - Fail-fast, bypass cache
   - Gradually retry

2. Fallback to DB
   async function getData(key) {
     try {
       const cached = await cache.get(key);
       if (cached) return cached;
     } catch (e) {
       // Log error, continue to DB
     }
     return await db.query(key);
   }

3. Local Cache Backup
   - In-memory LRU cache
   - Fallback when Redis unavailable
   - Reduces DB load during outages

4. Multi-Region
   - ElastiCache Global Datastore
   - Replicate across regions
   - Failover to secondary

5. Connection Pooling
   - Limit connections
   - Handle connection failures gracefully
```

### How do you size an ElastiCache cluster?

```
Sizing Factors:

1. Memory Requirements
   - Calculate data size
   - Account for Redis overhead (~25%)
   - Plan for peak data size

   Formula: Node memory > Data size * 1.25 / Number of shards

2. Throughput Requirements
   - Measure ops/second needed
   - Single node: ~100K ops/sec (varies)
   - Scale shards for write throughput
   - Scale replicas for read throughput

3. Connection Count
   - Max connections = 65,000 per node
   - Account for connection pooling
   - Larger nodes for more connections

Sizing Process:
1. Estimate data size (current + growth)
2. Estimate ops/sec (read/write ratio)
3. Choose node type (memory fits data)
4. Calculate shard count (write throughput)
5. Calculate replica count (read throughput)
6. Add 20-30% buffer

Example:
- 50 GB data, 100K reads/sec, 10K writes/sec
- Cluster mode enabled
- 3 shards (for write distribution)
- r6g.xlarge (26 GB * 3 = 78 GB capacity)
- 2 replicas per shard (read scaling)
```

### Redis vs Memcached decision?

```
Choose Redis When:
- Need complex data structures (lists, sets, sorted sets)
- Need persistence/durability
- Need replication and HA
- Need pub/sub
- Need Lua scripting
- Need transactions (MULTI/EXEC)
- Need geospatial features

Choose Memcached When:
- Simple key-value cache only
- Need multi-threaded performance
- Memory efficiency is critical
- Simpler operational model
- Don't need persistence
- Large object caching

Recommendation: Default to Redis
- More features, minimal downsides
- Memcached advantages are marginal
- Redis cluster mode addresses scale
```
