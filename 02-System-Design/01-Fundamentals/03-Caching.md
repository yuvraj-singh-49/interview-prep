# Caching

## What is Caching?

Storing copies of data in a faster storage layer to reduce latency and load on the primary data source.

```
┌────────┐     ┌─────────┐     ┌──────────┐
│ Client │────▶│  Cache  │────▶│ Database │
└────────┘     └─────────┘     └──────────┘
                   │
            Hit? Return (fast)
            Miss? Query DB → Store → Return
```

---

## Cache Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                        Fastest                              │
├─────────────────────────────────────────────────────────────┤
│  L1 Cache     │ ~0.5ns  │ ~32KB    │ Per CPU core           │
├─────────────────────────────────────────────────────────────┤
│  L2 Cache     │ ~7ns    │ ~256KB   │ Per CPU core           │
├─────────────────────────────────────────────────────────────┤
│  L3 Cache     │ ~20ns   │ ~8MB     │ Shared across cores    │
├─────────────────────────────────────────────────────────────┤
│  RAM          │ ~100ns  │ ~64GB    │ Main memory            │
├─────────────────────────────────────────────────────────────┤
│  Redis/       │ ~1ms    │ ~100GB   │ Distributed cache      │
│  Memcached    │         │          │                        │
├─────────────────────────────────────────────────────────────┤
│  SSD          │ ~100μs  │ ~1TB     │ Local disk             │
├─────────────────────────────────────────────────────────────┤
│  HDD          │ ~10ms   │ ~10TB    │ Spinning disk          │
├─────────────────────────────────────────────────────────────┤
│  Network      │ ~150ms  │ ∞        │ Remote service         │
│                        Slowest                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Caching Strategies

### 1. Cache-Aside (Lazy Loading)

Application manages the cache.

```javascript
async function getUser(userId) {
  // Check cache first
  const cached = await cache.get(`user:${userId}`);
  if (cached) {
    return JSON.parse(cached); // Cache hit
  }

  // Cache miss - query database
  const user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);

  // Store in cache for future requests
  await cache.setex(`user:${userId}`, 3600, JSON.stringify(user));

  return user;
}

// On update, invalidate cache
async function updateUser(userId, data) {
  await db.query('UPDATE users SET ? WHERE id = ?', [data, userId]);
  await cache.del(`user:${userId}`); // Invalidate
}
```

**Pros:**
- Simple to implement
- Cache only what's needed
- Resilient to cache failures

**Cons:**
- Initial request always slow
- Cache can become stale
- Application complexity

### 2. Read-Through Cache

Cache sits between application and database.

```javascript
// Cache library handles loading automatically
const cache = new ReadThroughCache({
  async loader(key) {
    const [, userId] = key.split(':');
    return db.query('SELECT * FROM users WHERE id = ?', [userId]);
  },
  ttl: 3600
});

async function getUser(userId) {
  // Cache handles miss automatically
  return cache.get(`user:${userId}`);
}
```

**Pros:**
- Simpler application code
- Consistent caching logic

**Cons:**
- Cache library must understand data source
- Less flexibility

### 3. Write-Through Cache

Write to cache and database synchronously.

```javascript
async function updateUser(userId, data) {
  // Write to database
  await db.query('UPDATE users SET ? WHERE id = ?', [data, userId]);

  // Write to cache immediately
  const user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);
  await cache.setex(`user:${userId}`, 3600, JSON.stringify(user));

  return user;
}
```

**Pros:**
- Cache always consistent
- No stale data

**Cons:**
- Higher write latency
- Cache may store unused data

### 4. Write-Behind (Write-Back) Cache

Write to cache immediately, persist to database asynchronously.

```javascript
class WriteBackCache {
  constructor() {
    this.writeQueue = [];
    this.flushInterval = setInterval(() => this.flush(), 1000);
  }

  async set(key, value) {
    await this.cache.set(key, value);
    this.writeQueue.push({ key, value, timestamp: Date.now() });
  }

  async flush() {
    const batch = this.writeQueue.splice(0, 100);
    if (batch.length === 0) return;

    await db.transaction(async (trx) => {
      for (const { key, value } of batch) {
        await trx.query('REPLACE INTO cache_data VALUES (?, ?)', [key, value]);
      }
    });
  }
}
```

**Pros:**
- Very fast writes
- Batching improves efficiency

**Cons:**
- Risk of data loss
- Complexity
- Eventual consistency

### 5. Refresh-Ahead

Proactively refresh cache before expiration.

```javascript
class RefreshAheadCache {
  async get(key) {
    const entry = await this.cache.get(key);
    if (!entry) return this.loadAndCache(key);

    const { value, expiresAt, createdAt } = JSON.parse(entry);
    const ttl = expiresAt - createdAt;
    const refreshThreshold = createdAt + (ttl * 0.75); // 75% of TTL

    // If past threshold, refresh async (don't wait)
    if (Date.now() > refreshThreshold) {
      this.loadAndCache(key).catch(console.error);
    }

    return value;
  }

  async loadAndCache(key) {
    const value = await this.loader(key);
    const expiresAt = Date.now() + this.ttl;
    await this.cache.setex(
      key,
      this.ttl / 1000,
      JSON.stringify({ value, expiresAt, createdAt: Date.now() })
    );
    return value;
  }
}
```

**Pros:**
- Reduces latency spikes
- Smoother performance

**Cons:**
- More complex
- Potential for unnecessary refreshes

---

## Cache Eviction Policies

### LRU (Least Recently Used)

Evicts items that haven't been accessed recently.

```javascript
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map();
  }

  get(key) {
    if (!this.cache.has(key)) return undefined;

    // Move to end (most recently used)
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  put(key, value) {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.capacity) {
      // Delete oldest (first item)
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }
}
```

### LFU (Least Frequently Used)

Evicts items with lowest access count.

```javascript
class LFUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map();
    this.frequencies = new Map();
    this.minFreq = 0;
  }

  get(key) {
    if (!this.cache.has(key)) return undefined;

    const item = this.cache.get(key);
    this.updateFrequency(key, item);
    return item.value;
  }

  updateFrequency(key, item) {
    const freq = item.freq;
    item.freq++;

    // Remove from current frequency list
    this.frequencies.get(freq).delete(key);
    if (this.frequencies.get(freq).size === 0) {
      this.frequencies.delete(freq);
      if (this.minFreq === freq) this.minFreq++;
    }

    // Add to new frequency list
    if (!this.frequencies.has(freq + 1)) {
      this.frequencies.set(freq + 1, new Set());
    }
    this.frequencies.get(freq + 1).add(key);
  }
}
```

### FIFO (First In, First Out)

Simple queue - oldest items evicted first.

### TTL (Time To Live)

Items expire after set duration.

```javascript
// Redis TTL
await redis.setex('key', 3600, 'value'); // Expires in 1 hour
```

### Random

Randomly select items to evict (surprisingly effective).

---

## Distributed Caching

### Redis

```javascript
const Redis = require('ioredis');

// Single instance
const redis = new Redis({
  host: 'localhost',
  port: 6379,
});

// Cluster mode
const cluster = new Redis.Cluster([
  { host: 'node1', port: 6379 },
  { host: 'node2', port: 6379 },
  { host: 'node3', port: 6379 },
]);

// Common operations
await redis.set('key', 'value');
await redis.setex('key', 3600, 'value'); // With TTL
await redis.get('key');
await redis.del('key');
await redis.mget('key1', 'key2', 'key3'); // Multi-get
await redis.incr('counter');
await redis.hset('user:1', 'name', 'John');
await redis.hgetall('user:1');
```

### Redis Data Structures

```javascript
// Strings
await redis.set('session:abc', JSON.stringify(sessionData));

// Hashes (for objects)
await redis.hset('user:123', {
  name: 'John',
  email: 'john@example.com',
  age: '30'
});
await redis.hget('user:123', 'name');

// Lists (for queues)
await redis.lpush('queue', 'task1');
await redis.rpop('queue');

// Sets (for unique items)
await redis.sadd('tags:article:1', 'javascript', 'node', 'cache');
await redis.smembers('tags:article:1');

// Sorted Sets (for leaderboards)
await redis.zadd('leaderboard', 100, 'player1', 200, 'player2');
await redis.zrevrange('leaderboard', 0, 9, 'WITHSCORES');

// Pub/Sub
await redis.publish('notifications', JSON.stringify(event));
redis.subscribe('notifications', (err, count) => {});
redis.on('message', (channel, message) => {});
```

### Memcached

```javascript
const Memcached = require('memcached');
const memcached = new Memcached('localhost:11211');

// Simple operations
memcached.set('key', 'value', 3600, (err) => {});
memcached.get('key', (err, data) => {});
memcached.del('key', (err) => {});
```

### Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data Types | Rich (strings, hashes, lists, sets, sorted sets) | Strings only |
| Persistence | Yes (RDB, AOF) | No |
| Replication | Yes | No |
| Pub/Sub | Yes | No |
| Lua Scripting | Yes | No |
| Memory Efficiency | Good | Better |
| Multi-threaded | No (single-threaded) | Yes |

---

## Cache Patterns

### Cache Key Design

```javascript
// Good key patterns
const patterns = {
  user: `user:${userId}`,
  userPosts: `user:${userId}:posts`,
  userPostsPaginated: `user:${userId}:posts:page:${page}`,
  sessionByToken: `session:${token}`,
  rateLimit: `rate:${ip}:${endpoint}`,
  searchResults: `search:${hashQuery(query)}`,
};

// Hash complex objects for keys
function hashQuery(query) {
  const normalized = JSON.stringify(query, Object.keys(query).sort());
  return crypto.createHash('md5').update(normalized).digest('hex');
}
```

### Cache Stampede Prevention

When cache expires, many requests hit database simultaneously.

```javascript
// Solution 1: Locking
async function getWithLock(key, loader) {
  const cached = await cache.get(key);
  if (cached) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  const acquired = await cache.set(lockKey, '1', 'NX', 'EX', 10);

  if (acquired) {
    try {
      const value = await loader();
      await cache.setex(key, 3600, JSON.stringify(value));
      return value;
    } finally {
      await cache.del(lockKey);
    }
  } else {
    // Wait and retry
    await sleep(100);
    return getWithLock(key, loader);
  }
}

// Solution 2: Probabilistic early expiration
async function getWithEarlyExpiry(key, loader, ttl) {
  const entry = await cache.get(key);
  if (entry) {
    const { value, expiresAt } = JSON.parse(entry);
    const remaining = expiresAt - Date.now();
    const shouldRefresh = Math.random() < Math.exp(-remaining / (ttl * 0.1));

    if (!shouldRefresh) return value;
  }

  const value = await loader();
  await cache.setex(key, ttl, JSON.stringify({
    value,
    expiresAt: Date.now() + (ttl * 1000)
  }));
  return value;
}
```

### Negative Caching

Cache "not found" results to prevent repeated DB lookups.

```javascript
async function getUser(userId) {
  const cached = await cache.get(`user:${userId}`);

  if (cached === 'NOT_FOUND') {
    return null; // Cached negative result
  }

  if (cached) {
    return JSON.parse(cached);
  }

  const user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);

  if (user) {
    await cache.setex(`user:${userId}`, 3600, JSON.stringify(user));
  } else {
    // Cache the "not found" for shorter duration
    await cache.setex(`user:${userId}`, 300, 'NOT_FOUND');
  }

  return user;
}
```

---

## HTTP Caching

### Cache-Control Headers

```javascript
// Express middleware
app.get('/api/static-data', (req, res) => {
  res.set('Cache-Control', 'public, max-age=86400'); // 24 hours
  res.json(data);
});

app.get('/api/user/:id', (req, res) => {
  res.set('Cache-Control', 'private, max-age=60'); // User-specific, 1 min
  res.json(user);
});

app.get('/api/sensitive', (req, res) => {
  res.set('Cache-Control', 'no-store'); // Never cache
  res.json(sensitiveData);
});
```

### Cache-Control Directives

```
public          - Can be cached by any cache
private         - Only browser cache, not CDN
no-cache        - Must revalidate before using
no-store        - Never cache
max-age=N       - Fresh for N seconds
s-maxage=N      - CDN-specific max-age
must-revalidate - Must check if stale
immutable       - Never changes (for versioned assets)
```

### ETag Validation

```javascript
const crypto = require('crypto');

app.get('/api/data', async (req, res) => {
  const data = await getData();
  const etag = crypto.createHash('md5').update(JSON.stringify(data)).digest('hex');

  // Check if client has current version
  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end(); // Not Modified
  }

  res.set('ETag', etag);
  res.set('Cache-Control', 'private, max-age=0, must-revalidate');
  res.json(data);
});
```

---

## CDN Caching

### CDN Architecture

```
                     ┌─────────────┐
                     │   Origin    │
                     │   Server    │
                     └──────┬──────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
    ┌───────────┐     ┌───────────┐     ┌───────────┐
    │  Edge PoP │     │  Edge PoP │     │  Edge PoP │
    │  (NYC)    │     │  (London) │     │  (Tokyo)  │
    └───────────┘     └───────────┘     └───────────┘
          ▲                 ▲                 ▲
          │                 │                 │
       Users             Users             Users
```

### CDN Configuration

```javascript
// Cloudflare Workers
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const cache = caches.default;
  const url = new URL(request.url);

  // Check cache first
  let response = await cache.match(request);
  if (response) return response;

  // Fetch from origin
  response = await fetch(request);

  // Clone and cache
  if (response.ok) {
    const cacheResponse = response.clone();
    event.waitUntil(cache.put(request, cacheResponse));
  }

  return response;
}
```

---

## Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." - Phil Karlton

### Strategies

#### 1. TTL-Based

```javascript
await cache.setex('key', 3600, 'value'); // Auto-expires in 1 hour
```

#### 2. Event-Based

```javascript
// Pub/Sub invalidation
userService.on('user:updated', async (userId) => {
  await cache.del(`user:${userId}`);
  await pubsub.publish('cache:invalidate', { key: `user:${userId}` });
});

// Other services listen and invalidate
pubsub.subscribe('cache:invalidate', async (message) => {
  await localCache.del(message.key);
});
```

#### 3. Version-Based

```javascript
// Include version in cache key
const version = await cache.get('user:schema:version') || '1';
const cacheKey = `user:${userId}:v${version}`;

// On schema change, increment version
await cache.incr('user:schema:version');
// All old cache entries become orphaned (expire naturally)
```

#### 4. Tag-Based

```javascript
class TaggedCache {
  async set(key, value, tags = []) {
    await this.cache.set(key, JSON.stringify(value));
    for (const tag of tags) {
      await this.cache.sadd(`tag:${tag}`, key);
    }
  }

  async invalidateByTag(tag) {
    const keys = await this.cache.smembers(`tag:${tag}`);
    if (keys.length > 0) {
      await this.cache.del(...keys);
      await this.cache.del(`tag:${tag}`);
    }
  }
}

// Usage
await taggedCache.set('post:123', postData, ['user:456', 'blog']);
await taggedCache.invalidateByTag('user:456'); // Invalidates all user's posts
```

---

## Interview Questions

### Q: How do you handle cache consistency with database?

**Write-through** for strong consistency:
- Write to DB and cache atomically
- Slower writes, always consistent

**Cache-aside with invalidation** for eventual consistency:
- Invalidate cache on DB write
- Next read repopulates cache
- Small window of inconsistency

### Q: How do you prevent cache stampede?

- **Locking**: Single request refreshes cache
- **Probabilistic early refresh**: Refresh before actual expiry
- **Background refresh**: Separate process keeps cache warm
- **Request coalescing**: Dedupe concurrent requests

### Q: When would you NOT use caching?

- Data changes frequently and consistency is critical
- Data is unique per request (no reuse)
- Storage cost exceeds computation cost
- Security-sensitive data that shouldn't be replicated
- Low-traffic scenarios where complexity isn't justified
