# Service Communication Patterns

## Communication Styles

```
┌─────────────────────────────────────────────────────────────┐
│                  Communication Matrix                        │
├─────────────────┬─────────────────┬─────────────────────────┤
│                 │  One-to-One     │  One-to-Many            │
├─────────────────┼─────────────────┼─────────────────────────┤
│  Synchronous    │  Request/Reply  │  -                      │
│                 │  (REST, gRPC)   │                         │
├─────────────────┼─────────────────┼─────────────────────────┤
│  Asynchronous   │  Async Request  │  Publish/Subscribe      │
│                 │  (Queue)        │  (Kafka, SNS)           │
└─────────────────┴─────────────────┴─────────────────────────┘
```

---

## REST API

### Best Practices

```javascript
// Resource-based URLs
GET    /users           // List users
GET    /users/123       // Get user 123
POST   /users           // Create user
PUT    /users/123       // Update user 123
PATCH  /users/123       // Partial update
DELETE /users/123       // Delete user 123

// Nested resources
GET    /users/123/orders      // User's orders
POST   /users/123/orders      // Create order for user

// Query parameters for filtering/pagination
GET /users?status=active&page=2&limit=20&sort=-createdAt
```

### Response Format

```javascript
// Success response
{
  "data": {
    "id": "123",
    "name": "John",
    "email": "john@example.com"
  },
  "meta": {
    "requestId": "req-abc-123"
  }
}

// Collection response
{
  "data": [...],
  "pagination": {
    "total": 100,
    "page": 1,
    "limit": 20,
    "hasMore": true
  }
}

// Error response
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "details": [
      { "field": "email", "message": "Required field" }
    ]
  },
  "meta": {
    "requestId": "req-abc-123"
  }
}
```

### Versioning

```
// URL versioning
GET /v1/users
GET /v2/users

// Header versioning
GET /users
Accept: application/vnd.api+json; version=2

// Query parameter
GET /users?version=2
```

---

## gRPC

### Protocol Buffers

```protobuf
// user.proto
syntax = "proto3";

package user;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
}

message GetUserRequest {
  string id = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}
```

### gRPC Client

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const packageDefinition = protoLoader.loadSync('user.proto');
const userProto = grpc.loadPackageDefinition(packageDefinition).user;

const client = new userProto.UserService(
  'localhost:50051',
  grpc.credentials.createInsecure()
);

// Unary call
client.GetUser({ id: '123' }, (err, user) => {
  console.log(user);
});

// Server streaming
const stream = client.ListUsers({ pageSize: 10 });
stream.on('data', (user) => console.log(user));
stream.on('end', () => console.log('Done'));
```

### REST vs gRPC

| Feature | REST | gRPC |
|---------|------|------|
| Protocol | HTTP/1.1 or HTTP/2 | HTTP/2 |
| Format | JSON (text) | Protobuf (binary) |
| Schema | OpenAPI (optional) | Required (.proto) |
| Streaming | Limited | Full support |
| Browser support | Native | Requires proxy |
| Performance | Good | Better |

---

## GraphQL

### Schema

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  orders: [Order!]!
}

type Order {
  id: ID!
  total: Float!
  status: OrderStatus!
  items: [OrderItem!]!
}

enum OrderStatus {
  PENDING
  PAID
  SHIPPED
  DELIVERED
}

type Query {
  user(id: ID!): User
  users(first: Int, after: String): UserConnection!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
}
```

### Queries

```graphql
# Client requests exactly what they need
query GetUserWithOrders {
  user(id: "123") {
    name
    email
    orders {
      id
      total
      status
    }
  }
}

# Response matches query shape
{
  "data": {
    "user": {
      "name": "John",
      "email": "john@example.com",
      "orders": [
        { "id": "1", "total": 99.99, "status": "SHIPPED" }
      ]
    }
  }
}
```

### N+1 Problem Solution

```javascript
// Use DataLoader for batching
const DataLoader = require('dataloader');

const orderLoader = new DataLoader(async (userIds) => {
  // Single query for all user IDs
  const orders = await db.query(
    'SELECT * FROM orders WHERE user_id IN (?)',
    [userIds]
  );

  // Group by user ID
  const ordersByUser = {};
  for (const order of orders) {
    if (!ordersByUser[order.userId]) {
      ordersByUser[order.userId] = [];
    }
    ordersByUser[order.userId].push(order);
  }

  return userIds.map(id => ordersByUser[id] || []);
});

// Resolver
const resolvers = {
  User: {
    orders: (user) => orderLoader.load(user.id)
  }
};
```

---

## Message Queues

### Request-Reply Pattern

```javascript
// Requester
const correlationId = uuid();

await channel.sendToQueue('request_queue', Buffer.from(JSON.stringify({
  action: 'getUser',
  userId: '123'
})), {
  correlationId,
  replyTo: 'reply_queue'
});

// Wait for reply
channel.consume('reply_queue', (msg) => {
  if (msg.properties.correlationId === correlationId) {
    const response = JSON.parse(msg.content.toString());
    console.log('Got response:', response);
  }
});

// Responder
channel.consume('request_queue', async (msg) => {
  const request = JSON.parse(msg.content.toString());
  const result = await processRequest(request);

  channel.sendToQueue(
    msg.properties.replyTo,
    Buffer.from(JSON.stringify(result)),
    { correlationId: msg.properties.correlationId }
  );
});
```

### Competing Consumers

```
Producer → Queue → Consumer 1
                 → Consumer 2  (only one gets each message)
                 → Consumer 3

// Scale by adding more consumers
```

---

## Service Mesh

### What is a Service Mesh?

```
Without Service Mesh:
┌──────────────┐    Direct calls    ┌──────────────┐
│   Service A  │───────────────────▶│   Service B  │
│              │                    │              │
│ Retry logic  │                    │              │
│ Circuit break│                    │              │
│ mTLS         │                    │              │
└──────────────┘                    └──────────────┘

With Service Mesh:
┌──────────────┐     ┌───────┐      ┌───────┐     ┌──────────────┐
│   Service A  │────▶│ Proxy │─────▶│ Proxy │────▶│   Service B  │
│              │     │(Envoy)│      │(Envoy)│     │              │
│ (just biz    │     └───────┘      └───────┘     │ (just biz    │
│  logic)      │         │              │         │  logic)      │
└──────────────┘         │              │         └──────────────┘
                         ▼              ▼
                    ┌──────────────────────────┐
                    │      Control Plane       │
                    │  (Istio, Linkerd, Consul)│
                    │  - Traffic management    │
                    │  - Security (mTLS)       │
                    │  - Observability         │
                    └──────────────────────────┘
```

### Service Mesh Features

```yaml
# Istio traffic management
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2  # Jason sees v2
  - route:
    - destination:
        host: reviews
        subset: v1  # Others see v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10    # 10% canary
```

---

## Circuit Breaker

### Implementation

```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 30000;
    this.state = 'CLOSED';
    this.failures = 0;
    this.lastFailure = null;
  }

  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailure > this.resetTimeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failures++;
    this.lastFailure = Date.now();

    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN';
    }
  }
}

// Usage
const breaker = new CircuitBreaker({ failureThreshold: 3 });

async function callService() {
  return breaker.call(async () => {
    return await fetch('http://service-b/api');
  });
}
```

### States

```
       ┌─────────────────────────────┐
       │                             │
       ▼                             │ Success
┌──────────┐    Failures    ┌────────┴──┐
│  CLOSED  │───────────────▶│   OPEN    │
└──────────┘    threshold   └─────┬─────┘
       ▲                          │
       │                     Timeout
       │                          │
       │                          ▼
       │                   ┌──────────┐
       │                   │HALF-OPEN │
       │                   └────┬─────┘
       │                        │
       │        Success         │
       └────────────────────────┘
              or failure → OPEN
```

---

## Retry Patterns

### Exponential Backoff

```javascript
async function retryWithBackoff(fn, options = {}) {
  const { maxRetries = 3, baseDelay = 1000, maxDelay = 30000 } = options;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;

      // Don't retry non-retryable errors
      if (error.status === 400 || error.status === 404) {
        throw error;
      }

      const delay = Math.min(
        baseDelay * Math.pow(2, attempt) + Math.random() * 1000,
        maxDelay
      );

      console.log(`Attempt ${attempt + 1} failed, retrying in ${delay}ms`);
      await sleep(delay);
    }
  }
}

// Usage
const result = await retryWithBackoff(
  () => fetch('http://service/api'),
  { maxRetries: 5, baseDelay: 1000 }
);
```

### Retry with Jitter

```javascript
function calculateDelay(attempt, baseDelay) {
  // Full jitter - random between 0 and exponential delay
  const exponentialDelay = baseDelay * Math.pow(2, attempt);
  return Math.random() * exponentialDelay;

  // Equal jitter - half exponential + random
  // return (exponentialDelay / 2) + Math.random() * (exponentialDelay / 2);
}
```

---

## Bulkhead Pattern

Isolate failures to prevent cascading.

```javascript
class Bulkhead {
  constructor(maxConcurrent) {
    this.maxConcurrent = maxConcurrent;
    this.running = 0;
    this.queue = [];
  }

  async execute(fn) {
    if (this.running >= this.maxConcurrent) {
      // Queue the request
      return new Promise((resolve, reject) => {
        this.queue.push({ fn, resolve, reject });
      });
    }

    return this.run(fn);
  }

  async run(fn) {
    this.running++;
    try {
      return await fn();
    } finally {
      this.running--;
      this.processQueue();
    }
  }

  processQueue() {
    if (this.queue.length > 0 && this.running < this.maxConcurrent) {
      const { fn, resolve, reject } = this.queue.shift();
      this.run(fn).then(resolve).catch(reject);
    }
  }
}

// Separate bulkheads for different services
const paymentBulkhead = new Bulkhead(10);
const inventoryBulkhead = new Bulkhead(20);

// Payment service issues won't affect inventory calls
await paymentBulkhead.execute(() => paymentService.charge());
await inventoryBulkhead.execute(() => inventoryService.reserve());
```

---

## Timeouts

```javascript
async function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) => {
    setTimeout(() => reject(new Error('Timeout')), ms);
  });

  return Promise.race([promise, timeout]);
}

// Set aggressive timeouts
async function callService() {
  try {
    return await withTimeout(
      fetch('http://service/api'),
      5000  // 5 second timeout
    );
  } catch (error) {
    if (error.message === 'Timeout') {
      // Handle timeout specifically
      return fallbackValue;
    }
    throw error;
  }
}
```

---

## Interview Questions

### Q: When would you choose gRPC over REST?

**gRPC:**
- Internal service-to-service communication
- Need streaming (bidirectional)
- Performance critical
- Polyglot environment (code generation)
- Strict schema enforcement

**REST:**
- Public APIs
- Browser clients
- Simple CRUD operations
- Ecosystem tooling (caching, debugging)

### Q: How do you handle partial failures in microservices?

1. **Timeouts** - Don't wait forever
2. **Retries** - With exponential backoff and jitter
3. **Circuit breakers** - Fail fast when service is down
4. **Bulkheads** - Isolate failures
5. **Fallbacks** - Graceful degradation
6. **Caching** - Serve stale data if necessary

### Q: How do you ensure idempotency in service communication?

1. **Idempotency keys** - Client-provided unique IDs
2. **Request deduplication** - Track processed requests
3. **Idempotent operations** - GET, PUT, DELETE naturally idempotent
4. **Database constraints** - Unique constraints prevent duplicates
