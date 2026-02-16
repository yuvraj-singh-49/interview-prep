# Database Fundamentals

## ACID Properties

### Atomicity

All operations in a transaction complete or none do.

```sql
-- Either both happen or neither
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- If any fails, ROLLBACK
```

### Consistency

Database moves from one valid state to another.

```sql
-- Constraints ensure consistency
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  balance DECIMAL(10,2) CHECK (balance >= 0),  -- No negative balance
  user_id INT REFERENCES users(id)             -- Valid user
);
```

### Isolation

Concurrent transactions don't interfere with each other.

```
Isolation Levels (weakest to strongest):

READ UNCOMMITTED - Can see uncommitted changes (dirty reads)
READ COMMITTED   - Only see committed changes
REPEATABLE READ  - Same query returns same results in transaction
SERIALIZABLE     - Full isolation (slowest)
```

### Durability

Committed transactions survive system failures.

```
Write-Ahead Logging (WAL):
1. Write to log first
2. Then write to data files
3. On crash, replay log to recover
```

---

## BASE Properties (NoSQL)

Opposite of ACID for distributed systems:

```
Basically Available  - System always responds (maybe stale data)
Soft state          - State may change without input (replication)
Eventually consistent - Data will become consistent over time
```

---

## SQL vs NoSQL

### Relational (SQL)

```sql
-- Structured data with relationships
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(100)
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),
  total DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Join for related data
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.created_at > '2024-01-01';
```

**When to use:**
- Complex queries with joins
- Transactions required
- Data integrity critical
- Schema is well-defined

**Databases:** PostgreSQL, MySQL, SQL Server

### Document Store (NoSQL)

```javascript
// MongoDB - Flexible schema
db.users.insertOne({
  _id: ObjectId(),
  email: "john@example.com",
  name: "John",
  orders: [
    { total: 99.99, items: ["item1", "item2"] },
    { total: 49.99, items: ["item3"] }
  ],
  preferences: {
    theme: "dark",
    notifications: true
  }
});

// Query embedded data
db.users.find({ "preferences.theme": "dark" });
```

**When to use:**
- Flexible/evolving schema
- Hierarchical data
- Rapid development
- Read-heavy workloads

**Databases:** MongoDB, CouchDB, DynamoDB

### Key-Value Store

```javascript
// Redis - Simple key-value
SET user:123 '{"name":"John","email":"john@example.com"}'
GET user:123
DEL user:123
EXPIRE user:123 3600
```

**When to use:**
- Caching
- Session storage
- Real-time data
- Simple lookups

**Databases:** Redis, Memcached, DynamoDB

### Wide-Column Store

```
// Cassandra - Column families
CREATE TABLE user_activity (
  user_id UUID,
  activity_date DATE,
  activity_time TIMESTAMP,
  activity_type TEXT,
  metadata MAP<TEXT, TEXT>,
  PRIMARY KEY ((user_id, activity_date), activity_time)
) WITH CLUSTERING ORDER BY (activity_time DESC);

// Optimized for time-series queries
SELECT * FROM user_activity
WHERE user_id = ? AND activity_date = '2024-01-15';
```

**When to use:**
- Time-series data
- Write-heavy workloads
- Massive scale
- Known query patterns

**Databases:** Cassandra, HBase, ScyllaDB

### Graph Database

```cypher
// Neo4j - Relationships first-class
CREATE (john:Person {name: 'John'})
CREATE (jane:Person {name: 'Jane'})
CREATE (john)-[:KNOWS {since: 2020}]->(jane)

// Find friends of friends
MATCH (p:Person {name: 'John'})-[:KNOWS*2]->(fof:Person)
RETURN DISTINCT fof.name
```

**When to use:**
- Complex relationships
- Social networks
- Recommendation engines
- Fraud detection

**Databases:** Neo4j, Amazon Neptune, JanusGraph

---

## Database Comparison Matrix

| Feature | PostgreSQL | MongoDB | Redis | Cassandra |
|---------|------------|---------|-------|-----------|
| Data Model | Relational | Document | Key-Value | Wide-Column |
| Schema | Fixed | Flexible | None | Fixed |
| ACID | Full | Per-document | Limited | Tunable |
| Scale | Vertical (+ read replicas) | Horizontal | Horizontal | Horizontal |
| Joins | Yes | Limited ($lookup) | No | No |
| Best For | Complex queries | Flexibility | Speed | Write-heavy |

---

## Indexing

### How Indexes Work

```
Without Index (Full Table Scan):
┌────┬───────┬─────────────────────┐
│ id │ name  │ email               │
├────┼───────┼─────────────────────┤
│ 1  │ Alice │ alice@example.com   │ ← Check
│ 2  │ Bob   │ bob@example.com     │ ← Check
│ 3  │ Carol │ carol@example.com   │ ← Check
│... │ ...   │ ...                 │ ← Check all rows
└────┴───────┴─────────────────────┘

With Index (B-Tree Lookup):
                    ┌─────┐
                    │ M   │
                    └──┬──┘
              ┌───────┴───────┐
           ┌──┴──┐         ┌──┴──┐
           │ F   │         │ S   │
           └──┬──┘         └──┬──┘
         ┌───┴───┐       ┌───┴───┐
        A-F    G-M      N-S    T-Z
         │
    Found "Carol" in O(log n)
```

### Index Types

```sql
-- B-Tree (default, most common)
CREATE INDEX idx_users_email ON users(email);

-- Unique index
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Composite index
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);
-- Supports: WHERE user_id = ? AND created_at > ?
-- Supports: WHERE user_id = ?
-- Does NOT support: WHERE created_at > ? (without user_id)

-- Partial index
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Expression index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- Full-text search
CREATE INDEX idx_posts_search ON posts USING GIN(to_tsvector('english', content));
```

### Index Trade-offs

```
Pros:
- Faster reads (O(log n) vs O(n))
- Faster sorts
- Faster joins

Cons:
- Slower writes (index must be updated)
- Storage overhead
- Memory usage
```

### Query Optimization

```sql
-- Use EXPLAIN to analyze queries
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 123
ORDER BY created_at DESC
LIMIT 10;

-- Result shows:
-- Index Scan vs Sequential Scan
-- Estimated vs Actual rows
-- Execution time
```

---

## Transactions

### Transaction Patterns

```javascript
// Basic transaction
async function transferMoney(fromId, toId, amount) {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // Debit from source
    const debit = await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1 RETURNING balance',
      [amount, fromId]
    );

    if (debit.rowCount === 0) {
      throw new Error('Insufficient funds');
    }

    // Credit to destination
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toId]
    );

    await client.query('COMMIT');
    return { success: true };
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

### Isolation Levels

```javascript
// PostgreSQL isolation levels
await client.query('BEGIN ISOLATION LEVEL READ COMMITTED');
await client.query('BEGIN ISOLATION LEVEL REPEATABLE READ');
await client.query('BEGIN ISOLATION LEVEL SERIALIZABLE');
```

### Distributed Transactions (2PC)

```
Two-Phase Commit:

Phase 1 (Prepare):
Coordinator → "Can you commit?" → All participants
All participants → "Yes, prepared" → Coordinator

Phase 2 (Commit):
Coordinator → "Commit!" → All participants
All participants → "Done" → Coordinator

Problems:
- Coordinator failure blocks everything
- Slow (multiple round trips)
- Locks held during entire process
```

### Saga Pattern (Alternative to 2PC)

```javascript
// Saga: Series of local transactions with compensations
class OrderSaga {
  async execute(order) {
    const compensations = [];

    try {
      // Step 1: Reserve inventory
      await inventoryService.reserve(order.items);
      compensations.push(() => inventoryService.release(order.items));

      // Step 2: Process payment
      const paymentId = await paymentService.charge(order.total);
      compensations.push(() => paymentService.refund(paymentId));

      // Step 3: Create order
      const orderId = await orderService.create(order);
      compensations.push(() => orderService.cancel(orderId));

      // Step 4: Ship
      await shippingService.ship(orderId);

      return { success: true, orderId };
    } catch (error) {
      // Compensate in reverse order
      for (const compensate of compensations.reverse()) {
        await compensate();
      }
      throw error;
    }
  }
}
```

---

## Connection Pooling

```javascript
// PostgreSQL connection pool
const { Pool } = require('pg');

const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'mydb',
  user: 'user',
  password: 'password',
  max: 20,                // Max connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Use pool for queries
async function getUser(id) {
  const result = await pool.query(
    'SELECT * FROM users WHERE id = $1',
    [id]
  );
  return result.rows[0];
}
```

### Pool Sizing

```
Optimal pool size formula (PostgreSQL):
connections = (core_count * 2) + effective_spindle_count

For SSDs: connections ≈ core_count * 2 to 4

Example:
- 8 core server with SSD
- Pool size: 16-32 connections

Too few: Queries wait for connections
Too many: Context switching overhead, resource exhaustion
```

---

## Query Patterns

### Pagination

```javascript
// Offset-based (simple but slow for large offsets)
const page = 5;
const limit = 20;
const offset = (page - 1) * limit;

await db.query(`
  SELECT * FROM posts
  ORDER BY created_at DESC
  LIMIT $1 OFFSET $2
`, [limit, offset]);
// Problem: OFFSET 10000 still scans 10000 rows

// Cursor-based (efficient for large datasets)
const lastId = 'abc123';

await db.query(`
  SELECT * FROM posts
  WHERE id < $1
  ORDER BY id DESC
  LIMIT $2
`, [lastId, limit]);
// Uses index, no scanning
```

### Batch Operations

```javascript
// Bad: N separate queries
for (const user of users) {
  await db.query('INSERT INTO users (name) VALUES ($1)', [user.name]);
}

// Good: Single batch insert
const values = users.map((u, i) => `($${i + 1})`).join(',');
await db.query(
  `INSERT INTO users (name) VALUES ${values}`,
  users.map(u => u.name)
);

// Better: COPY for bulk inserts (PostgreSQL)
const copyStream = client.query(copyFrom('COPY users (name) FROM STDIN'));
for (const user of users) {
  copyStream.write(`${user.name}\n`);
}
copyStream.end();
```

---

## Database Design Patterns

### Soft Deletes

```sql
-- Instead of DELETE
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP;

-- "Delete" a user
UPDATE users SET deleted_at = NOW() WHERE id = 123;

-- Query excludes deleted
SELECT * FROM users WHERE deleted_at IS NULL;
```

### Audit Logging

```sql
CREATE TABLE audit_log (
  id SERIAL PRIMARY KEY,
  table_name VARCHAR(50),
  record_id INT,
  action VARCHAR(10),
  old_data JSONB,
  new_data JSONB,
  changed_by INT,
  changed_at TIMESTAMP DEFAULT NOW()
);

-- Trigger to auto-log
CREATE TRIGGER audit_users
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION log_changes();
```

### Multi-Tenancy

```sql
-- Schema per tenant
CREATE SCHEMA tenant_acme;
CREATE TABLE tenant_acme.users (...);

-- Shared tables with tenant_id
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  tenant_id INT NOT NULL,
  name VARCHAR(100),
  INDEX idx_tenant (tenant_id)
);

-- Row-level security (PostgreSQL)
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON users
  USING (tenant_id = current_setting('app.tenant_id')::INT);
```

---

## Interview Questions

### Q: How do you choose between SQL and NoSQL?

**Choose SQL when:**
- Complex queries/relationships
- ACID transactions required
- Schema is stable and well-defined
- Reporting needs

**Choose NoSQL when:**
- Flexible/evolving schema
- Massive write throughput
- Horizontal scaling required
- Simple access patterns

### Q: What is database normalization?

Organizing data to reduce redundancy:

```
1NF: Atomic values (no arrays in cells)
2NF: No partial dependencies (all non-key columns depend on full key)
3NF: No transitive dependencies (non-key columns don't depend on each other)

Trade-off: More normalization = More joins but less redundancy
Denormalization: Intentionally duplicate data for read performance
```

### Q: How do you handle slow queries?

1. **EXPLAIN** the query to understand execution plan
2. **Add indexes** for WHERE/JOIN/ORDER BY columns
3. **Optimize query** (rewrite, remove unnecessary columns)
4. **Denormalize** if joins are the bottleneck
5. **Add caching** layer (Redis)
6. **Partition/shard** if data volume is the issue
