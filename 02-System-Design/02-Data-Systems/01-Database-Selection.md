# Database Selection Guide

## Decision Framework

```
┌─────────────────────────────────────────────────────────────┐
│                    Database Selection                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. What's your data model?                                 │
│     → Relational (tables) → SQL                            │
│     → Document (JSON) → MongoDB/DynamoDB                   │
│     → Key-Value → Redis                                    │
│     → Graph → Neo4j                                        │
│     → Time-series → InfluxDB/TimescaleDB                   │
│                                                             │
│  2. What's your read/write ratio?                           │
│     → Read-heavy → Add read replicas, caching              │
│     → Write-heavy → Consider NoSQL, sharding               │
│                                                             │
│  3. What consistency do you need?                           │
│     → Strong → Traditional SQL                             │
│     → Eventual → NoSQL (Cassandra, DynamoDB)               │
│                                                             │
│  4. What's your scale?                                      │
│     → <1TB → Any database                                  │
│     → 1-10TB → Consider sharding                           │
│     → >10TB → Distributed databases                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Database Comparison

### Relational Databases

#### PostgreSQL

```yaml
Strengths:
  - ACID compliance
  - Rich data types (JSON, arrays, hstore)
  - Advanced indexing (GiST, GIN, BRIN)
  - Full-text search
  - Window functions
  - Excellent JSON support
  - Extensions (PostGIS, TimescaleDB)

Weaknesses:
  - Horizontal scaling is complex
  - Memory hungry
  - Complex configuration

Best For:
  - General purpose OLTP
  - Complex queries
  - Geographic data (PostGIS)
  - When you need flexibility

Scale: Vertical + read replicas (Citus for sharding)
```

#### MySQL

```yaml
Strengths:
  - Simple and fast
  - Large community
  - Good replication
  - InnoDB is battle-tested

Weaknesses:
  - Fewer advanced features than PostgreSQL
  - Window functions (added later)
  - JSON support less robust

Best For:
  - Simple CRUD applications
  - Read-heavy workloads
  - When simplicity matters

Scale: Primary-replica, MySQL Cluster
```

### Document Databases

#### MongoDB

```yaml
Strengths:
  - Flexible schema
  - Rich queries on nested documents
  - Horizontal scaling (sharding)
  - Aggregation framework
  - Change streams

Weaknesses:
  - No joins (use $lookup, limited)
  - Transactions added later (4.0+)
  - Memory usage
  - Data duplication

Best For:
  - Rapid prototyping
  - Content management
  - Product catalogs
  - Real-time analytics

Scale: Replica sets + sharding
```

```javascript
// MongoDB schema-less advantage
db.products.insertOne({
  name: "Laptop",
  specs: {
    cpu: "M2",
    ram: "16GB",
    storage: { type: "SSD", size: "512GB" }
  },
  reviews: [
    { user: "john", rating: 5, text: "Great!" }
  ],
  tags: ["electronics", "computers"]
});

// Different structure, same collection
db.products.insertOne({
  name: "T-Shirt",
  sizes: ["S", "M", "L"],
  colors: ["red", "blue"],
  material: "cotton"
});
```

#### DynamoDB

```yaml
Strengths:
  - Fully managed (AWS)
  - Predictable performance at any scale
  - Single-digit ms latency
  - Auto-scaling
  - Global tables (multi-region)

Weaknesses:
  - Limited query patterns (design around access patterns)
  - Expensive at scale
  - Vendor lock-in
  - No ad-hoc queries

Best For:
  - Serverless applications
  - Gaming leaderboards
  - Session storage
  - Known access patterns

Scale: Unlimited (AWS managed)
```

```javascript
// DynamoDB single-table design
const params = {
  TableName: 'Application',
  Item: {
    PK: 'USER#123',              // Partition key
    SK: 'PROFILE',               // Sort key
    name: 'John',
    email: 'john@example.com'
  }
};

// Same table, different entity
const orderParams = {
  TableName: 'Application',
  Item: {
    PK: 'USER#123',
    SK: 'ORDER#2024-01-15#001',
    total: 99.99,
    status: 'shipped'
  }
};

// Query user's orders
const query = {
  TableName: 'Application',
  KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
  ExpressionAttributeValues: {
    ':pk': 'USER#123',
    ':sk': 'ORDER#'
  }
};
```

### Key-Value Stores

#### Redis

```yaml
Strengths:
  - Extremely fast (in-memory)
  - Rich data structures
  - Pub/Sub
  - Lua scripting
  - Persistence options

Weaknesses:
  - Memory limited
  - Single-threaded (for commands)
  - Clustering complexity

Best For:
  - Caching
  - Session storage
  - Rate limiting
  - Real-time leaderboards
  - Pub/Sub messaging

Scale: Redis Cluster (16384 hash slots)
```

### Wide-Column Stores

#### Cassandra

```yaml
Strengths:
  - Linear scalability
  - High write throughput
  - No single point of failure
  - Tunable consistency
  - Multi-datacenter replication

Weaknesses:
  - No joins
  - Limited query flexibility
  - Read-before-write patterns are slow
  - Requires data modeling expertise

Best For:
  - Time-series data
  - IoT sensor data
  - Write-heavy workloads
  - Multi-region deployments

Scale: Add nodes, data redistributes automatically
```

```cql
-- Cassandra table design (query-driven)
CREATE TABLE user_activity (
  user_id UUID,
  activity_date DATE,
  activity_time TIMESTAMP,
  activity_type TEXT,
  PRIMARY KEY ((user_id, activity_date), activity_time)
) WITH CLUSTERING ORDER BY (activity_time DESC);

-- Efficient: Query by user and date
SELECT * FROM user_activity
WHERE user_id = ? AND activity_date = '2024-01-15';

-- Inefficient: Query by activity_type (requires ALLOW FILTERING)
```

### Graph Databases

#### Neo4j

```yaml
Strengths:
  - Native graph storage
  - Cypher query language (intuitive)
  - Index-free adjacency (fast traversals)
  - Visual query results

Weaknesses:
  - Doesn't scale horizontally easily
  - Memory intensive
  - Different paradigm (learning curve)

Best For:
  - Social networks
  - Recommendation engines
  - Fraud detection
  - Knowledge graphs
  - Network/dependency analysis

Scale: Causal clustering (limited horizontal scale)
```

```cypher
// Neo4j - Find friends of friends who like similar movies
MATCH (me:Person {name: 'John'})-[:FRIENDS*2]->(fof:Person),
      (me)-[:LIKES]->(m:Movie)<-[:LIKES]-(fof)
WHERE NOT (me)-[:FRIENDS]-(fof)
RETURN fof.name, COUNT(m) as commonMovies
ORDER BY commonMovies DESC
LIMIT 10
```

### Time-Series Databases

#### InfluxDB / TimescaleDB

```yaml
Strengths:
  - Optimized for time-series writes
  - Automatic data retention
  - Built-in aggregation functions
  - Compression
  - TimescaleDB: PostgreSQL compatible

Best For:
  - Metrics/monitoring
  - IoT sensor data
  - Financial data
  - Log analytics
```

### Search Engines

#### Elasticsearch

```yaml
Strengths:
  - Full-text search
  - Near real-time indexing
  - Distributed and scalable
  - Rich query DSL
  - Analytics and aggregations

Weaknesses:
  - Not a primary database
  - Eventually consistent
  - Resource intensive
  - Complex operations

Best For:
  - Search functionality
  - Log analysis (ELK stack)
  - Analytics
  - Autocomplete
```

---

## Decision Matrix

| Use Case | Recommended DB | Alternative |
|----------|---------------|-------------|
| General CRUD app | PostgreSQL | MySQL |
| E-commerce | PostgreSQL + Redis | MongoDB |
| Social network | PostgreSQL + Neo4j | MongoDB |
| Real-time analytics | ClickHouse | Elasticsearch |
| IoT/Time-series | TimescaleDB | InfluxDB, Cassandra |
| Session storage | Redis | DynamoDB |
| Content management | MongoDB | PostgreSQL (JSONB) |
| Gaming leaderboards | Redis | DynamoDB |
| Search | Elasticsearch | Algolia |
| Messaging queue | Redis | Kafka (for scale) |
| Logs | Elasticsearch | Loki |
| Cache | Redis | Memcached |

---

## Polyglot Persistence

Modern systems often use multiple databases:

```
┌─────────────────────────────────────────────────────────────┐
│                      E-Commerce System                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User Data ─────────────────────────────▶ PostgreSQL        │
│  (profiles, auth)                         (ACID, relations) │
│                                                             │
│  Product Catalog ───────────────────────▶ MongoDB           │
│  (flexible attributes)                    (schema-less)     │
│                                                             │
│  Shopping Cart ─────────────────────────▶ Redis             │
│  (fast, temporary)                        (in-memory)       │
│                                                             │
│  Order History ─────────────────────────▶ Cassandra         │
│  (append-only, time-series)              (write-heavy)      │
│                                                             │
│  Product Search ────────────────────────▶ Elasticsearch     │
│  (full-text, facets)                     (search engine)    │
│                                                             │
│  Recommendations ───────────────────────▶ Neo4j             │
│  (user-product graphs)                   (graph queries)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Q: How do you choose between PostgreSQL and MongoDB?

**Choose PostgreSQL when:**
- Complex relationships between entities
- Need ACID transactions
- Complex queries with joins
- Data integrity is critical
- Well-defined schema

**Choose MongoDB when:**
- Schema evolves frequently
- Hierarchical/nested data
- Horizontal scaling required
- Rapid prototyping
- Geographically distributed

### Q: When would you use Redis as a primary database?

Generally not recommended as primary, but acceptable for:
- Session data (ephemeral by nature)
- Rate limiting data
- Feature flags
- Real-time leaderboards (with persistence enabled)

Always have persistence (RDB/AOF) enabled and backups.

### Q: How do you handle data that doesn't fit one database type?

Use **polyglot persistence**:
1. Identify different data types and access patterns
2. Select best database for each use case
3. Implement data synchronization (CDC, events)
4. Accept eventual consistency between systems
5. Define source of truth for each data type
