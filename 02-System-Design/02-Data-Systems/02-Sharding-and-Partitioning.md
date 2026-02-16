# Database Sharding and Partitioning

## Partitioning vs Sharding

```
Partitioning: Dividing data within a single database instance
Sharding: Distributing data across multiple database instances

┌─────────────────────────────┐     ┌─────────────────────────────┐
│     Single Database          │     │    Multiple Databases       │
│  ┌─────┬─────┬─────┬─────┐  │     │  ┌─────┐ ┌─────┐ ┌─────┐   │
│  │ P1  │ P2  │ P3  │ P4  │  │     │  │ S1  │ │ S2  │ │ S3  │   │
│  └─────┴─────┴─────┴─────┘  │     │  └─────┘ └─────┘ └─────┘   │
│       PARTITIONING           │     │        SHARDING            │
└─────────────────────────────┘     └─────────────────────────────┘
```

---

## Partitioning Strategies

### Horizontal Partitioning (Row-based)

Split rows across partitions.

```sql
-- PostgreSQL range partitioning
CREATE TABLE orders (
  id SERIAL,
  user_id INT,
  created_at TIMESTAMP,
  total DECIMAL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2023 PARTITION OF orders
  FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Queries automatically route to correct partition
SELECT * FROM orders WHERE created_at = '2024-06-15';
-- Only scans orders_2024
```

### Vertical Partitioning (Column-based)

Split columns across tables.

```sql
-- Before: Wide table
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(255),
  profile_json TEXT,       -- Large
  avatar_blob BYTEA        -- Very large
);

-- After: Split by access pattern
CREATE TABLE users_core (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(255)
);

CREATE TABLE users_profile (
  user_id INT PRIMARY KEY REFERENCES users_core(id),
  profile_json TEXT
);

CREATE TABLE users_media (
  user_id INT PRIMARY KEY REFERENCES users_core(id),
  avatar_blob BYTEA
);
```

---

## Sharding Strategies

### 1. Range-Based Sharding

```
Shard by ranges of a key:

Shard 1: user_id 1 - 1,000,000
Shard 2: user_id 1,000,001 - 2,000,000
Shard 3: user_id 2,000,001 - 3,000,000
```

```javascript
function getShard(userId) {
  const shardSize = 1000000;
  const shardIndex = Math.floor(userId / shardSize);
  return shards[shardIndex];
}
```

**Pros:**
- Simple implementation
- Range queries efficient on single shard
- Easy to understand

**Cons:**
- Hotspots (new users all on latest shard)
- Uneven distribution
- Rebalancing is complex

### 2. Hash-Based Sharding

```
Shard = hash(key) % number_of_shards

hash("user_123") % 4 = 2 → Shard 2
hash("user_456") % 4 = 0 → Shard 0
```

```javascript
function getShard(key, numShards) {
  const hash = crypto.createHash('md5')
    .update(key.toString())
    .digest('hex');
  const hashInt = parseInt(hash.substring(0, 8), 16);
  return shards[hashInt % numShards];
}
```

**Pros:**
- Even distribution
- No hotspots

**Cons:**
- Range queries span all shards
- Adding shards requires resharding
- Loss of data locality

### 3. Consistent Hashing

Minimizes data movement when adding/removing shards.

```javascript
class ConsistentHash {
  constructor(nodes, replicas = 100) {
    this.ring = new Map();
    this.sortedKeys = [];

    for (const node of nodes) {
      for (let i = 0; i < replicas; i++) {
        const hash = this.hash(`${node}-${i}`);
        this.ring.set(hash, node);
        this.sortedKeys.push(hash);
      }
    }
    this.sortedKeys.sort((a, b) => a - b);
  }

  hash(key) {
    // Simple hash for illustration
    let h = 0;
    for (let i = 0; i < key.length; i++) {
      h = ((h << 5) - h + key.charCodeAt(i)) | 0;
    }
    return Math.abs(h);
  }

  getNode(key) {
    if (this.sortedKeys.length === 0) return null;

    const hash = this.hash(key);

    // Find first node clockwise
    for (const ringKey of this.sortedKeys) {
      if (hash <= ringKey) {
        return this.ring.get(ringKey);
      }
    }
    return this.ring.get(this.sortedKeys[0]);
  }

  addNode(node, replicas = 100) {
    for (let i = 0; i < replicas; i++) {
      const hash = this.hash(`${node}-${i}`);
      this.ring.set(hash, node);
      this.sortedKeys.push(hash);
    }
    this.sortedKeys.sort((a, b) => a - b);
  }

  removeNode(node, replicas = 100) {
    for (let i = 0; i < replicas; i++) {
      const hash = this.hash(`${node}-${i}`);
      this.ring.delete(hash);
      const idx = this.sortedKeys.indexOf(hash);
      if (idx !== -1) this.sortedKeys.splice(idx, 1);
    }
  }
}
```

```
Ring visualization:

      0
    /   \
   A     B       Key K hashes between B and C
  /       \      → Goes to shard C (first clockwise)
 |         |
  \       /
   D     C
    \   /
     180

Adding new node E between B and C:
- Only keys between B and E move to E
- Other shards unaffected
```

### 4. Directory-Based Sharding

Lookup service maps keys to shards.

```javascript
class ShardDirectory {
  constructor() {
    this.directory = new Map();
    this.shards = ['shard1', 'shard2', 'shard3'];
  }

  async getShardForKey(key) {
    // Check directory first
    if (this.directory.has(key)) {
      return this.directory.get(key);
    }

    // Assign to shard (could be any logic)
    const shard = this.shards[
      Math.floor(Math.random() * this.shards.length)
    ];
    this.directory.set(key, shard);
    return shard;
  }

  async migrateKey(key, newShard) {
    this.directory.set(key, newShard);
  }
}
```

**Pros:**
- Flexible key placement
- Easy migration

**Cons:**
- Directory is single point of failure
- Directory lookup adds latency
- Directory must be highly available

### 5. Geographic Sharding

```
US Users  → us-east-shard
EU Users  → eu-west-shard
Asia Users → ap-south-shard
```

**Pros:**
- Data locality
- Compliance (data residency)

**Cons:**
- Cross-region queries complex
- User migration between regions

---

## Shard Key Selection

### Good Shard Keys

```
High Cardinality:
- user_id ✓ (millions of unique values)
- order_id ✓
- email ✓

Even Distribution:
- hash(user_id) ✓
- Avoid sequential IDs without hashing

Query Pattern Alignment:
- If queries are by user, shard by user_id
- If queries are by region, shard by region
```

### Bad Shard Keys

```
Low Cardinality:
- status (only a few values) ✗
- country (uneven distribution) ✗
- boolean fields ✗

Monotonically Increasing:
- timestamp ✗ (hotspot on latest shard)
- auto-increment ID ✗

Frequently Changing:
- user_status ✗ (requires re-sharding)
```

---

## Handling Cross-Shard Operations

### Cross-Shard Queries

```javascript
// Query spanning multiple shards
async function searchUsers(query) {
  // Fan out to all shards
  const promises = shards.map(shard =>
    shard.query('SELECT * FROM users WHERE name LIKE ?', [`%${query}%`])
  );

  // Gather results
  const results = await Promise.all(promises);

  // Merge and sort
  return results
    .flat()
    .sort((a, b) => b.relevance - a.relevance)
    .slice(0, 100);
}
```

### Cross-Shard Joins

```
Avoid if possible! Strategies:

1. Denormalization
   - Duplicate data to avoid joins

2. Application-level joins
   - Query each shard separately
   - Join in application code

3. Broadcast small tables
   - Replicate lookup tables to all shards
```

### Cross-Shard Transactions

```javascript
// Saga pattern for distributed transactions
async function transferMoney(fromUserId, toUserId, amount) {
  const fromShard = getShardForUser(fromUserId);
  const toShard = getShardForUser(toUserId);

  try {
    // Step 1: Debit
    await fromShard.debit(fromUserId, amount);

    // Step 2: Credit
    try {
      await toShard.credit(toUserId, amount);
    } catch (error) {
      // Compensate step 1
      await fromShard.credit(fromUserId, amount);
      throw error;
    }

    return { success: true };
  } catch (error) {
    throw new Error('Transfer failed');
  }
}
```

---

## Rebalancing Shards

### When to Rebalance

```
- Shard is too hot (CPU/memory)
- Data distribution becomes uneven
- Adding new shards for capacity
- Removing shards for cost
```

### Rebalancing Strategies

#### Online Migration

```javascript
// Double-write during migration
async function write(key, value) {
  const oldShard = getOldShard(key);
  const newShard = getNewShard(key);

  if (migrationInProgress && oldShard !== newShard) {
    // Write to both
    await Promise.all([
      oldShard.write(key, value),
      newShard.write(key, value)
    ]);
  } else {
    await currentShard(key).write(key, value);
  }
}

// Background migration job
async function migrateData() {
  for (const key of keysToMigrate) {
    const value = await oldShard.read(key);
    await newShard.write(key, value);
    markKeyMigrated(key);
  }
}
```

#### Virtual Shards

```
Physical Shards: 3
Virtual Shards: 1024

Each physical shard owns ~341 virtual shards

To add a physical shard:
- Move virtual shards between physical shards
- No data rehashing needed
```

---

## Sharding Implementation Examples

### MongoDB Sharding

```javascript
// Enable sharding on database
sh.enableSharding("mydb")

// Shard a collection by user_id (hashed)
sh.shardCollection("mydb.orders", { user_id: "hashed" })

// Range-based sharding
sh.shardCollection("mydb.logs", { timestamp: 1 })
```

### PostgreSQL with Citus

```sql
-- Create distributed table
SELECT create_distributed_table('orders', 'user_id');

-- Queries automatically routed
SELECT * FROM orders WHERE user_id = 123;

-- Cross-shard queries work but slower
SELECT user_id, SUM(total) FROM orders GROUP BY user_id;
```

### Vitess (MySQL)

```yaml
# VSchema defines sharding
{
  "sharded": true,
  "vindexes": {
    "user_hash": {
      "type": "hash"
    }
  },
  "tables": {
    "users": {
      "column_vindexes": [
        {
          "column": "id",
          "name": "user_hash"
        }
      ]
    }
  }
}
```

---

## Interview Questions

### Q: How do you decide when to shard?

**Shard when:**
1. Single instance can't handle write load
2. Data size exceeds single machine capacity
3. Read replicas aren't enough
4. Geographic distribution needed

**Before sharding, try:**
- Vertical scaling
- Read replicas
- Caching
- Query optimization
- Connection pooling

### Q: How do you handle a hotspot shard?

1. **Add more replicas** for read hotspots
2. **Split the hot shard** into multiple shards
3. **Re-shard with different key** if pattern allows
4. **Add caching** in front of hot data
5. **Rate limit** problematic access patterns

### Q: What happens when you need to add a shard?

With simple hash sharding:
- Massive data movement (all keys rehash)
- Downtime or complex migration

With consistent hashing:
- Only ~1/N data moves (N = num shards)
- Minimal disruption

With virtual shards:
- Move virtual shards between physical
- Very flexible rebalancing
