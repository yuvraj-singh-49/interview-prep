# API Design & Database Fundamentals

## Overview

Lead engineers design APIs and make database choices. This covers REST principles, API best practices, and database fundamentals frequently asked in interviews.

---

## REST API Principles

### HTTP Methods

| Method | Purpose | Idempotent | Safe | Request Body |
|--------|---------|------------|------|--------------|
| GET | Read resource | Yes | Yes | No |
| POST | Create resource | No | No | Yes |
| PUT | Replace resource | Yes | No | Yes |
| PATCH | Partial update | No* | No | Yes |
| DELETE | Remove resource | Yes | No | No |

*PATCH can be made idempotent with proper implementation

### RESTful URL Design

```
Good:
GET    /users              - List users
GET    /users/123          - Get user 123
POST   /users              - Create user
PUT    /users/123          - Replace user 123
PATCH  /users/123          - Update user 123
DELETE /users/123          - Delete user 123

GET    /users/123/orders   - Get orders for user 123
POST   /users/123/orders   - Create order for user 123

Bad:
GET    /getUsers
POST   /createUser
GET    /users/delete/123
POST   /users/123/delete
```

### Query Parameters vs Path Parameters

```
Path: Identify specific resources
GET /users/123
GET /orders/456

Query: Filter, sort, paginate
GET /users?status=active&sort=name&page=2
GET /products?category=electronics&price_min=100
```

---

## HTTP Status Codes

### Success (2xx)

| Code | Meaning | When to Use |
|------|---------|-------------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST creating resource |
| 204 | No Content | Successful DELETE, or PUT with no response body |

### Client Errors (4xx)

| Code | Meaning | When to Use |
|------|---------|-------------|
| 400 | Bad Request | Invalid request body, validation failed |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Resource conflict (duplicate, version mismatch) |
| 422 | Unprocessable Entity | Semantic errors in valid JSON |
| 429 | Too Many Requests | Rate limit exceeded |

### Server Errors (5xx)

| Code | Meaning | When to Use |
|------|---------|-------------|
| 500 | Internal Server Error | Unexpected server error |
| 502 | Bad Gateway | Upstream service error |
| 503 | Service Unavailable | Server overloaded or in maintenance |
| 504 | Gateway Timeout | Upstream service timeout |

---

## API Best Practices

### 1. Consistent Response Format

```json
// Success
{
  "data": {
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com"
  },
  "meta": {
    "requestId": "abc-123"
  }
}

// Error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address"
      }
    ]
  },
  "meta": {
    "requestId": "abc-124"
  }
}

// List with pagination
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 150,
    "totalPages": 8
  }
}
```

### 2. Versioning

```
URL Path (most common):
/api/v1/users
/api/v2/users

Header:
Accept: application/vnd.api+json;version=1

Query Parameter:
/api/users?version=1
```

### 3. Pagination

```
Offset-based (simple, but slow for large offsets):
GET /users?page=5&pageSize=20
GET /users?offset=80&limit=20

Cursor-based (better for real-time data):
GET /users?cursor=abc123&limit=20

Keyset (most efficient for large datasets):
GET /users?after_id=12345&limit=20
```

### 4. Filtering and Sorting

```
Simple filtering:
GET /products?category=electronics&status=active

Range filtering:
GET /products?price_min=100&price_max=500
GET /orders?created_after=2024-01-01

Sorting:
GET /users?sort=name          (ascending)
GET /users?sort=-created_at   (descending)
GET /users?sort=status,-name  (multiple fields)
```

### 5. Idempotency

```python
# Client sends idempotency key
POST /orders
Headers:
  Idempotency-Key: unique-request-id-123

# Server implementation
class OrderService:
    def create_order(self, data, idempotency_key):
        # Check if already processed
        existing = self.cache.get(idempotency_key)
        if existing:
            return existing  # Return cached response

        # Process new request
        order = self.process_order(data)

        # Cache result
        self.cache.set(idempotency_key, order, ttl=86400)
        return order
```

### 6. Rate Limiting

```
Response Headers:
X-RateLimit-Limit: 100       # Max requests per window
X-RateLimit-Remaining: 45    # Remaining requests
X-RateLimit-Reset: 1640000000 # Unix timestamp when limit resets
Retry-After: 30              # Seconds to wait (when 429)
```

### 7. HATEOAS (Hypermedia)

```json
{
  "data": {
    "id": 123,
    "name": "John Doe",
    "status": "active"
  },
  "links": {
    "self": "/users/123",
    "orders": "/users/123/orders",
    "deactivate": "/users/123/deactivate"
  }
}
```

---

## Authentication & Authorization

### Common Patterns

```
API Key (simple, service-to-service):
Authorization: Api-Key YOUR_API_KEY

Bearer Token (JWT, OAuth):
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

Basic Auth (rarely used in APIs):
Authorization: Basic base64(username:password)
```

### JWT Structure

```
Header.Payload.Signature

Header: {"alg": "HS256", "typ": "JWT"}
Payload: {
  "sub": "user123",
  "name": "John Doe",
  "iat": 1640000000,
  "exp": 1640086400,
  "roles": ["user", "admin"]
}
Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

---

## Database Fundamentals

### SQL vs NoSQL

| Aspect | SQL (Relational) | NoSQL |
|--------|------------------|-------|
| **Schema** | Fixed, predefined | Flexible, dynamic |
| **Scaling** | Vertical (scale up) | Horizontal (scale out) |
| **ACID** | Strong guarantees | Varies (eventual consistency) |
| **Joins** | Native support | Limited or none |
| **Use Cases** | Complex queries, transactions | High throughput, flexible schema |
| **Examples** | PostgreSQL, MySQL | MongoDB, Cassandra, Redis |

### When to Use What

```
SQL:
- Complex relationships (e-commerce, banking)
- Need ACID transactions
- Complex queries with JOINs
- Data integrity is critical

NoSQL - Document (MongoDB):
- Flexible/evolving schema
- Hierarchical data
- Rapid development

NoSQL - Key-Value (Redis):
- Caching
- Session storage
- High-speed lookups

NoSQL - Wide Column (Cassandra):
- Time-series data
- Write-heavy workloads
- Distributed at scale

NoSQL - Graph (Neo4j):
- Social networks
- Recommendation engines
- Complex relationships
```

---

## SQL Essentials

### Indexing

```sql
-- Basic index
CREATE INDEX idx_users_email ON users(email);

-- Composite index (order matters!)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Unique index
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Partial index (PostgreSQL)
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
```

**Index Guidelines:**
- Index columns in WHERE, JOIN, ORDER BY
- Composite index: put equality columns first, then range
- Don't over-index (slows writes)
- B-tree for most cases, Hash for equality only

### Query Optimization

```sql
-- Bad: SELECT *
SELECT * FROM users;

-- Good: Select only needed columns
SELECT id, name, email FROM users;

-- Bad: N+1 query problem
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
-- ...

-- Good: Single query with JOIN or IN
SELECT * FROM orders WHERE user_id IN (1, 2, 3, ...);

-- Use EXPLAIN to analyze
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

### Transactions and Isolation Levels

```sql
-- Transaction
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- Or ROLLBACK on error
```

**Isolation Levels:**

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|------------|---------------------|--------------|
| Read Uncommitted | Yes | Yes | Yes |
| Read Committed | No | Yes | Yes |
| Repeatable Read | No | No | Yes |
| Serializable | No | No | No |

### Normalization

```
1NF: Atomic values, no repeating groups
2NF: 1NF + No partial dependencies
3NF: 2NF + No transitive dependencies

Example progression:
Unnormalized:
| order_id | customer_name | products |
| 1 | John | "Laptop, Mouse" |

1NF:
| order_id | customer_name | product |
| 1 | John | Laptop |
| 1 | John | Mouse |

2NF: Split into orders and order_items tables

3NF: Split customer info into customers table
```

### Common Patterns

```sql
-- Soft delete
UPDATE users SET deleted_at = NOW() WHERE id = 123;
SELECT * FROM users WHERE deleted_at IS NULL;

-- Pagination
SELECT * FROM users
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;

-- Keyset pagination (more efficient)
SELECT * FROM users
WHERE created_at < '2024-01-01'
ORDER BY created_at DESC
LIMIT 20;

-- Upsert (PostgreSQL)
INSERT INTO users (email, name)
VALUES ('test@example.com', 'Test')
ON CONFLICT (email)
DO UPDATE SET name = EXCLUDED.name;

-- Window functions
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) as rank,
       AVG(salary) OVER () as avg_salary
FROM employees;
```

---

## NoSQL Patterns

### Document Store (MongoDB-style)

```javascript
// Embedding (for 1:few relationships)
{
  "_id": "user123",
  "name": "John",
  "addresses": [
    { "type": "home", "city": "NYC" },
    { "type": "work", "city": "LA" }
  ]
}

// Referencing (for 1:many or many:many)
// Users collection
{ "_id": "user123", "name": "John" }

// Orders collection
{ "_id": "order456", "userId": "user123", "total": 100 }
```

### Key-Value (Redis patterns)

```python
# Caching
redis.set("user:123", json.dumps(user_data))
redis.expire("user:123", 3600)  # 1 hour TTL

# Rate limiting
key = f"rate_limit:{user_id}"
count = redis.incr(key)
if count == 1:
    redis.expire(key, 60)  # 1 minute window
if count > 100:
    raise RateLimitExceeded()

# Session storage
redis.hset(f"session:{session_id}", mapping={
    "user_id": "123",
    "created_at": "2024-01-01"
})

# Leaderboard
redis.zadd("leaderboard", {"player1": 100, "player2": 85})
redis.zrevrange("leaderboard", 0, 9)  # Top 10

# Pub/Sub
redis.publish("notifications", json.dumps({"type": "new_message"}))
```

---

## Caching Strategies

### Cache-Aside (Lazy Loading)

```python
def get_user(user_id):
    # Check cache first
    cached = cache.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    # Cache miss - fetch from DB
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)

    # Store in cache
    cache.set(f"user:{user_id}", json.dumps(user), ttl=3600)
    return user
```

### Write-Through

```python
def update_user(user_id, data):
    # Update DB
    db.query("UPDATE users SET ... WHERE id = ?", data, user_id)

    # Update cache
    cache.set(f"user:{user_id}", json.dumps(data), ttl=3600)
```

### Write-Behind (Write-Back)

```python
def update_user(user_id, data):
    # Update cache immediately
    cache.set(f"user:{user_id}", json.dumps(data))

    # Queue DB write for later (async)
    queue.push({"type": "update_user", "id": user_id, "data": data})
```

### Cache Invalidation

```python
# Time-based (TTL)
cache.set("key", value, ttl=3600)

# Event-based
def on_user_updated(user_id):
    cache.delete(f"user:{user_id}")
    cache.delete(f"user_profile:{user_id}")

# Version-based
cache.set(f"user:{user_id}:v{version}", value)
```

---

## Database Design Interview Questions

### 1. Design a URL Shortener

```sql
CREATE TABLE urls (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    click_count BIGINT DEFAULT 0
);

CREATE INDEX idx_urls_short_code ON urls(short_code);
CREATE INDEX idx_urls_expires ON urls(expires_at) WHERE expires_at IS NOT NULL;
```

### 2. Design a Social Feed

```sql
-- Users
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE
);

-- Posts
CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    content TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Follows (fan-out on read vs write)
CREATE TABLE follows (
    follower_id BIGINT REFERENCES users(id),
    following_id BIGINT REFERENCES users(id),
    PRIMARY KEY (follower_id, following_id)
);

-- Feed query (fan-out on read)
SELECT p.* FROM posts p
JOIN follows f ON p.user_id = f.following_id
WHERE f.follower_id = ?
ORDER BY p.created_at DESC
LIMIT 20;

-- Pre-computed feed (fan-out on write) - better for read-heavy
CREATE TABLE feed (
    user_id BIGINT,
    post_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, created_at, post_id)
);
```

### 3. Design an E-commerce Order System

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10, 2),
    stock_quantity INT
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT,
    status VARCHAR(20) DEFAULT 'pending',
    total_amount DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT REFERENCES orders(id),
    product_id BIGINT REFERENCES products(id),
    quantity INT,
    unit_price DECIMAL(10, 2)
);

-- Inventory management with optimistic locking
UPDATE products
SET stock_quantity = stock_quantity - 1,
    version = version + 1
WHERE id = ? AND version = ? AND stock_quantity > 0;
```

---

## Quick Reference

### API Checklist

- [ ] Consistent naming (nouns, plural)
- [ ] Proper HTTP methods and status codes
- [ ] Versioning strategy
- [ ] Pagination for lists
- [ ] Error response format
- [ ] Rate limiting
- [ ] Authentication
- [ ] Input validation
- [ ] CORS configuration

### Database Checklist

- [ ] Index frequently queried columns
- [ ] Use appropriate data types
- [ ] Consider normalization vs denormalization
- [ ] Plan for scaling (read replicas, sharding)
- [ ] Implement proper backups
- [ ] Monitor slow queries
- [ ] Use connection pooling

---

## Interview Tips

1. **Clarify requirements** - Read-heavy vs write-heavy, scale expectations
2. **Start simple** - Don't over-engineer initially
3. **Discuss trade-offs** - There's no perfect solution
4. **Consider edge cases** - What happens with high load, failures?
5. **Know your numbers** - Typical latencies, throughput expectations
6. **Security matters** - SQL injection, authentication, authorization
