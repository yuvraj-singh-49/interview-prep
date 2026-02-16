# Design: Rate Limiter

## Requirements

### Functional Requirements
- Limit requests per time window
- Different limits for different users/APIs
- Return clear error when rate limited
- Low latency (shouldn't slow down requests)

### Non-Functional Requirements
- Highly available
- Minimal memory footprint
- Distributed (work across multiple servers)
- Low latency overhead

---

## Rate Limiting Algorithms

### 1. Token Bucket

```
┌─────────────────────────────────────┐
│          Token Bucket                │
│  ┌─────────────────────────────────┐│
│  │ 🪙 🪙 🪙 🪙 🪙                   ││  Capacity: 10 tokens
│  │                                 ││  Refill: 1 token/second
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
         │
         ▼
Request arrives → Take 1 token → Allow
No tokens? → Reject (429)
```

```javascript
class TokenBucket {
  constructor(capacity, refillRate) {
    this.capacity = capacity;      // Max tokens
    this.tokens = capacity;        // Current tokens
    this.refillRate = refillRate;  // Tokens per second
    this.lastRefill = Date.now();
  }

  allow() {
    this.refill();

    if (this.tokens >= 1) {
      this.tokens -= 1;
      return true;
    }
    return false;
  }

  refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    const tokensToAdd = elapsed * this.refillRate;

    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }
}

// Usage
const limiter = new TokenBucket(10, 1); // 10 capacity, 1 token/sec
if (limiter.allow()) {
  // Process request
} else {
  // Return 429
}
```

**Pros:** Allows bursts, smooth rate limiting
**Cons:** Memory for each bucket

### 2. Leaky Bucket

```
         Requests
            │
            ▼
┌─────────────────────────┐
│   Queue (Fixed Size)    │
│  [req1][req2][req3]     │
└───────────┬─────────────┘
            │
            ▼ Fixed rate output
        Processor
```

```javascript
class LeakyBucket {
  constructor(capacity, leakRate) {
    this.capacity = capacity;
    this.leakRate = leakRate;  // Requests processed per second
    this.queue = [];
    this.lastLeak = Date.now();
  }

  allow() {
    this.leak();

    if (this.queue.length < this.capacity) {
      this.queue.push(Date.now());
      return true;
    }
    return false;
  }

  leak() {
    const now = Date.now();
    const elapsed = (now - this.lastLeak) / 1000;
    const toRemove = Math.floor(elapsed * this.leakRate);

    this.queue.splice(0, toRemove);
    this.lastLeak = now;
  }
}
```

**Pros:** Smooth output rate
**Cons:** No burst handling, requests wait in queue

### 3. Fixed Window Counter

```
     Window 1          Window 2          Window 3
   [0:00-1:00]       [1:00-2:00]       [2:00-3:00]
   count: 98         count: 50         count: 0
   limit: 100        limit: 100        limit: 100

Window resets at boundary.
```

```javascript
class FixedWindowCounter {
  constructor(windowSize, limit) {
    this.windowSize = windowSize; // milliseconds
    this.limit = limit;
    this.counts = new Map();
  }

  allow(key) {
    const window = Math.floor(Date.now() / this.windowSize);
    const windowKey = `${key}:${window}`;

    const count = this.counts.get(windowKey) || 0;

    if (count >= this.limit) {
      return false;
    }

    this.counts.set(windowKey, count + 1);
    this.cleanup();
    return true;
  }

  cleanup() {
    const currentWindow = Math.floor(Date.now() / this.windowSize);
    for (const key of this.counts.keys()) {
      const window = parseInt(key.split(':')[1]);
      if (window < currentWindow - 1) {
        this.counts.delete(key);
      }
    }
  }
}
```

**Pros:** Simple, memory efficient
**Cons:** Burst at window boundaries (200 requests in 2 seconds at boundary)

### 4. Sliding Window Log

```
Store timestamp of each request, count within window.

Requests: [0:00:30, 0:00:45, 0:01:00, 0:01:15, 0:01:30]
Window: 1 minute

At 0:01:45, looking back 1 minute:
Valid: [0:00:45, 0:01:00, 0:01:15, 0:01:30] = 4 requests
```

```javascript
class SlidingWindowLog {
  constructor(windowSize, limit) {
    this.windowSize = windowSize;
    this.limit = limit;
    this.logs = new Map();
  }

  allow(key) {
    const now = Date.now();
    const windowStart = now - this.windowSize;

    if (!this.logs.has(key)) {
      this.logs.set(key, []);
    }

    const timestamps = this.logs.get(key);

    // Remove old entries
    while (timestamps.length > 0 && timestamps[0] < windowStart) {
      timestamps.shift();
    }

    if (timestamps.length >= this.limit) {
      return false;
    }

    timestamps.push(now);
    return true;
  }
}
```

**Pros:** Accurate, no boundary issues
**Cons:** High memory (stores all timestamps)

### 5. Sliding Window Counter (Recommended)

```
Combines fixed window efficiency with sliding accuracy.

Previous window count: 80 (weight: 0.25)
Current window count: 20 (weight: 0.75)
Weighted count: 80 × 0.25 + 20 × 0.75 = 35
Limit: 100 → Allowed
```

```javascript
class SlidingWindowCounter {
  constructor(windowSize, limit) {
    this.windowSize = windowSize;
    this.limit = limit;
    this.windows = new Map();
  }

  allow(key) {
    const now = Date.now();
    const currentWindow = Math.floor(now / this.windowSize);
    const previousWindow = currentWindow - 1;
    const windowProgress = (now % this.windowSize) / this.windowSize;

    const currentKey = `${key}:${currentWindow}`;
    const previousKey = `${key}:${previousWindow}`;

    const currentCount = this.windows.get(currentKey) || 0;
    const previousCount = this.windows.get(previousKey) || 0;

    // Weighted count
    const count = previousCount * (1 - windowProgress) + currentCount;

    if (count >= this.limit) {
      return false;
    }

    this.windows.set(currentKey, currentCount + 1);
    return true;
  }
}
```

**Pros:** Memory efficient, accurate, smooth limiting
**Cons:** Slight approximation

---

## Distributed Rate Limiting

### Using Redis

```javascript
const Redis = require('ioredis');
const redis = new Redis();

class DistributedRateLimiter {
  constructor(limit, windowSeconds) {
    this.limit = limit;
    this.window = windowSeconds;
  }

  async allow(key) {
    const now = Math.floor(Date.now() / 1000);
    const windowKey = `ratelimit:${key}:${now - (now % this.window)}`;

    const multi = redis.multi();
    multi.incr(windowKey);
    multi.expire(windowKey, this.window + 1);

    const results = await multi.exec();
    const count = results[0][1];

    return count <= this.limit;
  }
}

// Token bucket with Redis
async function tokenBucketAllow(key, capacity, refillRate) {
  const script = `
    local key = KEYS[1]
    local capacity = tonumber(ARGV[1])
    local refillRate = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])

    local bucket = redis.call('HMGET', key, 'tokens', 'lastRefill')
    local tokens = tonumber(bucket[1]) or capacity
    local lastRefill = tonumber(bucket[2]) or now

    local elapsed = now - lastRefill
    tokens = math.min(capacity, tokens + elapsed * refillRate)

    if tokens >= 1 then
      tokens = tokens - 1
      redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
      redis.call('EXPIRE', key, 3600)
      return 1
    else
      return 0
    end
  `;

  const result = await redis.eval(
    script,
    1,
    `ratelimit:${key}`,
    capacity,
    refillRate,
    Date.now() / 1000
  );

  return result === 1;
}
```

### Race Condition Handling

```javascript
// Problem: Multiple servers read count=99, all increment to 100
// Solution: Atomic operations with Lua scripts

const luaScript = `
  local current = redis.call('GET', KEYS[1])
  if current and tonumber(current) >= tonumber(ARGV[1]) then
    return 0
  end
  redis.call('INCR', KEYS[1])
  if not current then
    redis.call('EXPIRE', KEYS[1], ARGV[2])
  end
  return 1
`;

async function atomicRateLimit(key, limit, windowSeconds) {
  const result = await redis.eval(luaScript, 1, key, limit, windowSeconds);
  return result === 1;
}
```

---

## High-Level Architecture

```
┌──────────┐     ┌─────────────┐     ┌─────────────┐
│  Client  │────▶│ Rate Limiter│────▶│ API Server  │
└──────────┘     │  Middleware │     │             │
                 └──────┬──────┘     └─────────────┘
                        │
                 ┌──────▼──────┐
                 │    Redis    │
                 │   Cluster   │
                 └─────────────┘
```

### Middleware Implementation

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

// Global rate limit
app.use(rateLimit({
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:'
  }),
  windowMs: 60 * 1000,  // 1 minute
  max: 100,             // 100 requests per minute
  message: {
    error: 'Too many requests',
    retryAfter: 60
  },
  standardHeaders: true,  // Return rate limit info in headers
  legacyHeaders: false
}));

// Per-user rate limit
const userRateLimit = rateLimit({
  store: new RedisStore({ client: redisClient }),
  windowMs: 60 * 1000,
  max: (req) => {
    // Different limits based on user tier
    if (req.user?.tier === 'premium') return 1000;
    if (req.user?.tier === 'basic') return 100;
    return 10; // Anonymous
  },
  keyGenerator: (req) => req.user?.id || req.ip
});

app.use('/api', userRateLimit);
```

### Response Headers

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1640000000

HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000000
Retry-After: 30
```

---

## Multi-Level Rate Limiting

```javascript
// Different limits for different scopes
const rateLimits = {
  global: { limit: 10000, window: 60 },      // 10K/min cluster-wide
  perIp: { limit: 100, window: 60 },         // 100/min per IP
  perUser: { limit: 1000, window: 60 },      // 1000/min per user
  perEndpoint: { limit: 50, window: 60 }     // 50/min per endpoint
};

async function checkAllLimits(req) {
  const checks = [
    { key: 'global', limit: rateLimits.global },
    { key: `ip:${req.ip}`, limit: rateLimits.perIp },
    { key: `user:${req.user?.id}`, limit: rateLimits.perUser },
    { key: `endpoint:${req.path}`, limit: rateLimits.perEndpoint }
  ];

  for (const check of checks) {
    if (!check.key) continue;

    const allowed = await rateLimiter.allow(
      check.key,
      check.limit.limit,
      check.limit.window
    );

    if (!allowed) {
      return { allowed: false, limitType: check.key.split(':')[0] };
    }
  }

  return { allowed: true };
}
```

---

## Interview Discussion Points

### Why rate limit at all?

1. Prevent abuse/DoS
2. Fair resource allocation
3. Cost control
4. Meet SLAs
5. Comply with external API limits

### Where to implement rate limiting?

1. **Client-side**: Prevent excessive requests
2. **Load balancer**: Hardware/Nginx
3. **API Gateway**: Centralized policy
4. **Application**: Fine-grained control
5. **Database**: Query limits

### How to handle distributed rate limiting?

1. **Centralized store** (Redis) - Single source of truth
2. **Eventual consistency** - Accept slight overage
3. **Local + sync** - Local limits, periodic sync
4. **Sticky sessions** - Route to same server

### What about race conditions?

1. **Lua scripts** in Redis (atomic)
2. **Redis MULTI/EXEC** transactions
3. **Optimistic locking** with CAS operations

### How to be fair during overload?

1. **Priority queues** - Premium users first
2. **Adaptive limits** - Reduce limits under load
3. **Circuit breakers** - Fail fast
4. **Graceful degradation** - Return cached data
