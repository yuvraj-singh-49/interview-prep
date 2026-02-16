# Scalability

## What is Scalability?

The ability of a system to handle growing amounts of work by adding resources. A scalable system maintains performance as load increases.

---

## Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up)

Adding more power to existing machines.

```
Before: 1 server (4 CPU, 16GB RAM)
After:  1 server (32 CPU, 128GB RAM)
```

**Pros:**
- Simple to implement
- No code changes required
- No distributed system complexity
- Single point of management

**Cons:**
- Hardware limits (can't scale infinitely)
- Single point of failure
- Expensive at high end
- Downtime during upgrades

**When to Use:**
- Early-stage applications
- Database servers (often easier to scale up)
- Legacy applications
- When simplicity is priority

### Horizontal Scaling (Scale Out)

Adding more machines to the pool.

```
Before: 1 server handling 1000 RPS
After:  10 servers handling 10000 RPS
```

**Pros:**
- Theoretically unlimited scaling
- Better fault tolerance
- Cost-effective (commodity hardware)
- No downtime for scaling

**Cons:**
- Distributed system complexity
- Data consistency challenges
- Network overhead
- Requires load balancing

**When to Use:**
- High-traffic applications
- Microservices architecture
- When fault tolerance is critical
- Stateless services

---

## Scalability Dimensions

### 1. Traffic/Load Scalability

Handle increasing requests per second.

```
Metrics to monitor:
- Requests per second (RPS)
- Concurrent connections
- Response time percentiles (p50, p95, p99)
- Error rates under load
```

### 2. Data Scalability

Handle growing data volumes.

```
Strategies:
- Database sharding
- Data archival
- Cold/hot data separation
- Compression
```

### 3. Geographic Scalability

Serve users across regions.

```
Strategies:
- Multi-region deployment
- CDN for static assets
- Edge computing
- Data replication
```

---

## Stateless vs Stateful Services

### Stateless Services (Easier to Scale)

```javascript
// Stateless - each request is independent
app.get('/api/calculate', (req, res) => {
  const { a, b } = req.query;
  res.json({ result: Number(a) + Number(b) });
});

// Any instance can handle any request
// Perfect for horizontal scaling
```

### Stateful Services (Harder to Scale)

```javascript
// Stateful - maintains session state
const sessions = new Map();

app.post('/api/login', (req, res) => {
  const sessionId = generateId();
  sessions.set(sessionId, { user: req.body.user });
  res.cookie('session', sessionId);
});

// Requests must go to same instance
// OR externalize state (Redis, database)
```

### Externalizing State

```javascript
// Move state to external store
const redis = require('redis').createClient();

app.post('/api/login', async (req, res) => {
  const sessionId = generateId();
  await redis.set(`session:${sessionId}`, JSON.stringify({
    user: req.body.user
  }));
  res.cookie('session', sessionId);
});

// Now any instance can handle any request
// State is shared via Redis
```

---

## Common Scalability Patterns

### 1. Database Read Replicas

```
                    ┌─────────────┐
                    │   Primary   │
                    │  (Writes)   │
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │  Replica 1  │ │  Replica 2  │ │  Replica 3  │
    │   (Reads)   │ │   (Reads)   │ │   (Reads)   │
    └─────────────┘ └─────────────┘ └─────────────┘
```

```javascript
// Route reads to replicas, writes to primary
const primary = new Pool({ host: 'primary.db.com' });
const replica = new Pool({ host: 'replica.db.com' });

async function getUser(id) {
  return replica.query('SELECT * FROM users WHERE id = $1', [id]);
}

async function updateUser(id, data) {
  return primary.query('UPDATE users SET data = $1 WHERE id = $2', [data, id]);
}
```

### 2. Caching Layer

```
┌────────┐     ┌─────────┐     ┌──────────┐
│ Client │────▶│  Cache  │────▶│ Database │
└────────┘     │ (Redis) │     └──────────┘
               └─────────┘
                   │
            Cache Hit? Return
            Cache Miss? Query DB, Cache, Return
```

### 3. Message Queues for Async Processing

```
┌────────┐     ┌─────────┐     ┌──────────┐
│  API   │────▶│  Queue  │────▶│ Workers  │
│ Server │     │ (Kafka) │     │ (N pods) │
└────────┘     └─────────┘     └──────────┘
     │
  Return 202 Accepted immediately
  Process async in workers
```

### 4. Microservices Decomposition

```
Monolith:
┌─────────────────────────────────────┐
│  Users │ Orders │ Payments │ Email │
└─────────────────────────────────────┘
Everything scales together (wasteful)

Microservices:
┌───────┐ ┌────────┐ ┌──────────┐ ┌───────┐
│ Users │ │ Orders │ │ Payments │ │ Email │
│ (2x)  │ │ (10x)  │ │  (5x)    │ │ (3x)  │
└───────┘ └────────┘ └──────────┘ └───────┘
Scale each service independently
```

---

## Scalability Anti-Patterns

### 1. Shared Mutable State

```javascript
// BAD: In-memory shared state
let requestCount = 0;

app.get('/api/data', (req, res) => {
  requestCount++; // Won't work across instances
  res.json({ count: requestCount });
});

// GOOD: External shared state
app.get('/api/data', async (req, res) => {
  const count = await redis.incr('request_count');
  res.json({ count });
});
```

### 2. Synchronous Blocking Operations

```javascript
// BAD: Blocking operation in request path
app.post('/api/order', async (req, res) => {
  await processPayment(req.body);      // 2s
  await sendEmail(req.body.email);     // 1s
  await generateInvoice(req.body);     // 3s
  await updateInventory(req.body);     // 1s
  res.json({ success: true });         // Total: 7s blocking
});

// GOOD: Async processing
app.post('/api/order', async (req, res) => {
  const orderId = await createOrder(req.body);
  await queue.publish('order.created', { orderId, ...req.body });
  res.json({ orderId, status: 'processing' }); // Return immediately
});
```

### 3. N+1 Query Problem

```javascript
// BAD: N+1 queries (doesn't scale)
const users = await db.query('SELECT * FROM users');
for (const user of users) {
  user.orders = await db.query(
    'SELECT * FROM orders WHERE user_id = ?', [user.id]
  );
}
// 1 query + N queries = N+1 queries

// GOOD: Batch loading
const users = await db.query('SELECT * FROM users');
const userIds = users.map(u => u.id);
const orders = await db.query(
  'SELECT * FROM orders WHERE user_id IN (?)', [userIds]
);
// Group orders by user_id
```

---

## Measuring Scalability

### Key Metrics

```yaml
Throughput:
  - Requests per second (RPS)
  - Transactions per second (TPS)
  - Messages processed per second

Latency:
  - p50: 50th percentile (median)
  - p95: 95th percentile
  - p99: 99th percentile (tail latency)

Resource Utilization:
  - CPU usage
  - Memory usage
  - Network I/O
  - Disk I/O

Scalability Efficiency:
  - Linear: 2x resources = 2x throughput (ideal)
  - Sublinear: 2x resources = 1.5x throughput (acceptable)
  - Inverse: 2x resources = 0.8x throughput (problem!)
```

### Load Testing

```javascript
// Example k6 load test
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up
    { duration: '5m', target: 100 },   // Stay at 100 users
    { duration: '2m', target: 200 },   // Ramp up more
    { duration: '5m', target: 200 },   // Stay at 200 users
    { duration: '2m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% under 500ms
    http_req_failed: ['rate<0.01'],   // Error rate < 1%
  },
};

export default function () {
  const res = http.get('https://api.example.com/data');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

---

## Interview Tips

### Common Questions

1. **"How would you scale this system to 10x traffic?"**
   - Identify bottlenecks (database? compute? network?)
   - Add caching where appropriate
   - Consider async processing
   - Horizontal scaling for stateless services
   - Database sharding/read replicas

2. **"What's the difference between scaling up vs out?"**
   - Cover pros/cons of each
   - Mention use cases
   - Discuss cost implications

3. **"How do you handle state in a distributed system?"**
   - Externalize to Redis/database
   - Sticky sessions (if unavoidable)
   - Event sourcing for state reconstruction

### Key Numbers to Remember

```
L1 cache reference:         0.5 ns
L2 cache reference:           7 ns
RAM reference:              100 ns
SSD random read:         16,000 ns (16 μs)
HDD seek:             10,000,000 ns (10 ms)
Network round trip:  150,000,000 ns (150 ms)

1 server can handle: ~10k concurrent connections
Redis operations: ~100k ops/second
PostgreSQL: ~10k transactions/second
```
