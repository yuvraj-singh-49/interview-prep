# DynamoDB

## Overview

```
DynamoDB = Fully managed NoSQL database

Key Characteristics:
- Key-value and document store
- Single-digit millisecond latency
- Unlimited throughput and storage
- Built-in replication (3 AZs)
- Serverless (no instances to manage)

Data Model:
┌─────────────────────────────────────────────────────────────────────────┐
│                           Table                                          │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                         Item (Row)                                 │  │
│  │  ┌──────────────────┬──────────────────┬──────────────────────┐   │  │
│  │  │ Partition Key    │ Sort Key         │ Attributes           │   │  │
│  │  │ (Required)       │ (Optional)       │ (Flexible schema)    │   │  │
│  │  │ pk: "USER#123"   │ sk: "ORDER#456"  │ status: "shipped"    │   │  │
│  │  │                  │                  │ total: 99.99         │   │  │
│  │  └──────────────────┴──────────────────┴──────────────────────┘   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  Item max size: 400 KB                                                  │
│  Attribute types: String, Number, Binary, Boolean, Null, List, Map, Set │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Primary Keys

### Key Types

```
1. Simple Primary Key (Partition Key only)
┌─────────────────────────────────────────────────────────────────────────┐
│  Table: Users                                                            │
│  ┌──────────────────┬──────────────────────────────────────────────────┐│
│  │   user_id (PK)   │   Attributes                                     ││
│  ├──────────────────┼──────────────────────────────────────────────────┤│
│  │   user_001       │   name: "Alice", email: "alice@example.com"      ││
│  │   user_002       │   name: "Bob", email: "bob@example.com"          ││
│  └──────────────────┴──────────────────────────────────────────────────┘│
│                                                                          │
│  Use when: Each item is uniquely identified by one attribute            │
└─────────────────────────────────────────────────────────────────────────┘

2. Composite Primary Key (Partition Key + Sort Key)
┌─────────────────────────────────────────────────────────────────────────┐
│  Table: Orders                                                           │
│  ┌──────────────────┬──────────────────┬───────────────────────────────┐│
│  │  user_id (PK)    │  order_id (SK)   │   Attributes                  ││
│  ├──────────────────┼──────────────────┼───────────────────────────────┤│
│  │  user_001        │  order_001       │   total: 150.00               ││
│  │  user_001        │  order_002       │   total: 75.50                ││
│  │  user_002        │  order_001       │   total: 200.00               ││
│  └──────────────────┴──────────────────┴───────────────────────────────┘│
│                                                                          │
│  Use when: Need to group related items and query within groups          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Partition Distribution

```
How DynamoDB stores data:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  Partition Key: "user_001"  ──hash──▶  Partition A                      │
│  Partition Key: "user_002"  ──hash──▶  Partition B                      │
│  Partition Key: "user_003"  ──hash──▶  Partition A                      │
│  Partition Key: "user_004"  ──hash──▶  Partition C                      │
│                                                                          │
│  Each partition:                                                         │
│  - Up to 10 GB storage                                                  │
│  - 3000 RCU / 1000 WCU throughput                                       │
│                                                                          │
│  Hot partition problem:                                                  │
│  If all requests go to one partition key → throttling                   │
│                                                                          │
│  Solution: Design for uniform distribution                               │
│  - Use high-cardinality partition keys                                  │
│  - Add suffix for hot items (e.g., user_001#1, user_001#2)             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Read/Write Operations

### Basic Operations

```javascript
// PutItem - Create or replace entire item
{
  TableName: "Orders",
  Item: {
    user_id: "user_001",
    order_id: "order_123",
    status: "pending",
    total: 99.99
  }
}

// GetItem - Read single item (requires full primary key)
{
  TableName: "Orders",
  Key: {
    user_id: "user_001",
    order_id: "order_123"
  }
}

// UpdateItem - Modify specific attributes
{
  TableName: "Orders",
  Key: {
    user_id: "user_001",
    order_id: "order_123"
  },
  UpdateExpression: "SET #s = :status, updated_at = :now",
  ExpressionAttributeNames: { "#s": "status" },
  ExpressionAttributeValues: {
    ":status": "shipped",
    ":now": "2024-01-15T10:30:00Z"
  }
}

// DeleteItem - Remove item
{
  TableName: "Orders",
  Key: {
    user_id: "user_001",
    order_id: "order_123"
  }
}
```

### Query vs Scan

```
Query (Efficient):
- Requires partition key
- Optional sort key conditions
- Returns items in sort key order
- Uses index efficiently

// Query: Get all orders for a user
{
  TableName: "Orders",
  KeyConditionExpression: "user_id = :uid AND order_id BEGINS_WITH :prefix",
  ExpressionAttributeValues: {
    ":uid": "user_001",
    ":prefix": "2024-"
  }
}

Scan (Expensive):
- Reads entire table
- Filters applied after read (still charged)
- Use only for admin/analytics
- Consider parallel scan for large tables

// Scan: Find all pending orders (avoid if possible)
{
  TableName: "Orders",
  FilterExpression: "#s = :status",
  ExpressionAttributeNames: { "#s": "status" },
  ExpressionAttributeValues: { ":status": "pending" }
}

// Better: Create GSI on status and query instead
```

---

## Secondary Indexes

### Global Secondary Index (GSI)

```
GSI = Different partition key, eventually consistent

┌─────────────────────────────────────────────────────────────────────────┐
│  Base Table: Orders                                                      │
│  PK: user_id    SK: order_id                                            │
│                                                                          │
│  ┌──────────────────┬──────────────────┬───────────────────────────────┐│
│  │  user_id         │  order_id        │   status                      ││
│  ├──────────────────┼──────────────────┼───────────────────────────────┤│
│  │  user_001        │  order_001       │   shipped                     ││
│  │  user_001        │  order_002       │   pending                     ││
│  │  user_002        │  order_003       │   pending                     ││
│  └──────────────────┴──────────────────┴───────────────────────────────┘│
│                                                                          │
│  GSI: StatusIndex                                                        │
│  PK: status    SK: order_id                                             │
│                                                                          │
│  ┌──────────────────┬──────────────────┬───────────────────────────────┐│
│  │  status (PK)     │  order_id (SK)   │   user_id (projected)         ││
│  ├──────────────────┼──────────────────┼───────────────────────────────┤│
│  │  pending         │  order_002       │   user_001                    ││
│  │  pending         │  order_003       │   user_002                    ││
│  │  shipped         │  order_001       │   user_001                    ││
│  └──────────────────┴──────────────────┴───────────────────────────────┘│
│                                                                          │
│  Now can query: "Get all pending orders"                                │
└─────────────────────────────────────────────────────────────────────────┘

GSI Characteristics:
- Separate throughput (RCU/WCU)
- Eventually consistent only
- Can have different partition + sort key
- Up to 20 GSIs per table
- Projected attributes (ALL, KEYS_ONLY, INCLUDE)
```

### Local Secondary Index (LSI)

```
LSI = Same partition key, different sort key, strongly consistent option

┌─────────────────────────────────────────────────────────────────────────┐
│  Base Table: Orders                                                      │
│  PK: user_id    SK: order_id                                            │
│                                                                          │
│  LSI: OrdersByDate                                                       │
│  PK: user_id    SK: order_date                                          │
│                                                                          │
│  Query: "Get user_001's orders sorted by date"                          │
│  - Uses same partition key (user_id)                                    │
│  - Different sort key (order_date instead of order_id)                  │
└─────────────────────────────────────────────────────────────────────────┘

LSI Characteristics:
- Must be created at table creation
- Same partition key as base table
- Shares throughput with base table
- Supports strongly consistent reads
- Up to 5 LSIs per table
- 10 GB limit per partition key
```

---

## Capacity Modes

### Provisioned Capacity

```
Provisioned = Pre-allocate capacity units

┌─────────────────────────────────────────────────────────────────────────┐
│  Capacity Units:                                                         │
│                                                                          │
│  RCU (Read Capacity Unit):                                              │
│  - 1 RCU = 1 strongly consistent read/sec (up to 4 KB)                 │
│  - 1 RCU = 2 eventually consistent reads/sec (up to 4 KB)              │
│  - Items > 4 KB: Round up (8 KB item = 2 RCU)                          │
│                                                                          │
│  WCU (Write Capacity Unit):                                             │
│  - 1 WCU = 1 write/sec (up to 1 KB)                                    │
│  - Items > 1 KB: Round up (2.5 KB item = 3 WCU)                        │
│                                                                          │
│  Example calculation:                                                    │
│  - 100 reads/sec, 2 KB items, strongly consistent                       │
│  - RCU = 100 * ceil(2/4) = 100 * 1 = 100 RCU                           │
│                                                                          │
│  - 50 writes/sec, 3 KB items                                            │
│  - WCU = 50 * ceil(3/1) = 50 * 3 = 150 WCU                             │
└─────────────────────────────────────────────────────────────────────────┘

Auto Scaling:
- Set min/max capacity
- Target utilization (e.g., 70%)
- Scales based on consumption
- 1-2 minute scaling delay
```

### On-Demand Capacity

```
On-Demand = Pay per request

Pricing (example, us-east-1):
- $1.25 per million write request units
- $0.25 per million read request units

Best for:
- Unpredictable workloads
- New applications
- Spiky traffic
- Development/test

Can switch modes once every 24 hours
```

---

## Single-Table Design

### Overview

```
Single-Table Design = Store multiple entity types in one table

Traditional (Multi-table):
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│    Users      │  │    Orders     │  │   Products    │
└───────────────┘  └───────────────┘  └───────────────┘

Single-Table:
┌─────────────────────────────────────────────────────────────────────────┐
│                         MainTable                                        │
│  ┌──────────────────┬──────────────────┬───────────────────────────────┐│
│  │  PK              │  SK              │   Attributes                  ││
│  ├──────────────────┼──────────────────┼───────────────────────────────┤│
│  │  USER#123        │  USER#123        │   name, email (User entity)   ││
│  │  USER#123        │  ORDER#456       │   total, status (Order)       ││
│  │  USER#123        │  ORDER#789       │   total, status (Order)       ││
│  │  ORDER#456       │  ORDER#456       │   order details               ││
│  │  ORDER#456       │  PRODUCT#abc     │   quantity, price (line item) ││
│  │  PRODUCT#abc     │  PRODUCT#abc     │   name, price (Product)       ││
│  └──────────────────┴──────────────────┴───────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────┘

Benefits:
- Single query retrieves related data
- Reduced latency (no joins)
- Atomic transactions across entities
- Simplified deployment
```

### Access Pattern Design

```
Step 1: List access patterns

┌────┬──────────────────────────────────────────────────────────────────┐
│ #  │ Access Pattern                                                    │
├────┼──────────────────────────────────────────────────────────────────┤
│ 1  │ Get user by ID                                                   │
│ 2  │ Get all orders for user                                          │
│ 3  │ Get order by ID                                                  │
│ 4  │ Get all items in order                                           │
│ 5  │ Get orders by status (across all users)                         │
│ 6  │ Get user's orders in date range                                  │
└────┴──────────────────────────────────────────────────────────────────┘

Step 2: Design keys to support patterns

┌────┬────────────────────────┬────────────────────────┬─────────────────┐
│ #  │ PK                     │ SK                     │ Index           │
├────┼────────────────────────┼────────────────────────┼─────────────────┤
│ 1  │ USER#<id>              │ USER#<id>              │ Base table      │
│ 2  │ USER#<id>              │ ORDER#<id>             │ Base table      │
│ 3  │ ORDER#<id>             │ ORDER#<id>             │ Base table      │
│ 4  │ ORDER#<id>             │ ITEM#<id>              │ Base table      │
│ 5  │ status                 │ ORDER#<id>             │ GSI1            │
│ 6  │ USER#<id>              │ ORDER#<date>#<id>      │ Base table      │
└────┴────────────────────────┴────────────────────────┴─────────────────┘
```

### Example: E-commerce Schema

```javascript
// User entity
{
  PK: "USER#123",
  SK: "USER#123",
  type: "User",
  name: "Alice",
  email: "alice@example.com",
  GSI1PK: "USER#123",    // For looking up user's orders by date
  GSI1SK: "USER#123"
}

// Order entity
{
  PK: "USER#123",        // Get orders by user
  SK: "ORDER#2024-01-15#456",  // Sorted by date
  type: "Order",
  order_id: "456",
  status: "shipped",
  total: 150.00,
  GSI1PK: "shipped",     // Get orders by status
  GSI1SK: "ORDER#2024-01-15#456"
}

// Order line item
{
  PK: "ORDER#456",       // Get items in order
  SK: "ITEM#001",
  type: "OrderItem",
  product_id: "prod_abc",
  quantity: 2,
  price: 50.00
}

// Queries:
// 1. Get user: Query PK="USER#123", SK="USER#123"
// 2. Get user's orders: Query PK="USER#123", SK begins_with "ORDER#"
// 3. Get orders by status: Query GSI1 PK="shipped"
// 4. Get order items: Query PK="ORDER#456", SK begins_with "ITEM#"
```

---

## Transactions & Consistency

### Transactions

```javascript
// TransactWriteItems - Up to 100 items across tables
{
  TransactItems: [
    {
      Put: {
        TableName: "Orders",
        Item: { PK: "ORDER#789", SK: "ORDER#789", status: "created" }
      }
    },
    {
      Update: {
        TableName: "Inventory",
        Key: { PK: "PRODUCT#abc", SK: "PRODUCT#abc" },
        UpdateExpression: "SET quantity = quantity - :dec",
        ConditionExpression: "quantity >= :dec",
        ExpressionAttributeValues: { ":dec": 1 }
      }
    }
  ]
}

// Transaction cost: 2x normal operation
// All-or-nothing: If one fails, all roll back
```

### Conditional Writes

```javascript
// Optimistic locking with version
{
  TableName: "Products",
  Key: { product_id: "abc" },
  UpdateExpression: "SET price = :new_price, version = :new_v",
  ConditionExpression: "version = :current_v",
  ExpressionAttributeValues: {
    ":new_price": 29.99,
    ":new_v": 2,
    ":current_v": 1
  }
}

// Prevent overwrites
{
  TableName: "Users",
  Item: { user_id: "123", name: "Alice" },
  ConditionExpression: "attribute_not_exists(user_id)"
}
```

---

## DynamoDB Streams

```
Streams = Capture item-level changes

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  DynamoDB Table ──write──▶ Stream ──trigger──▶ Lambda                   │
│                            (ordered)           - Process changes        │
│                                                - Replicate to S3        │
│                                                - Update ElastiCache     │
│                                                - Send notifications     │
│                                                                          │
│  Stream Views:                                                           │
│  - KEYS_ONLY: Just the key attributes                                   │
│  - NEW_IMAGE: Entire item after modification                            │
│  - OLD_IMAGE: Entire item before modification                           │
│  - NEW_AND_OLD_IMAGES: Both before and after                            │
│                                                                          │
│  Retention: 24 hours                                                     │
│  Ordering: By partition key (shard)                                     │
└─────────────────────────────────────────────────────────────────────────┘

Use Cases:
- Materialized views
- Cross-region replication (Global Tables use this)
- Analytics pipelines
- Audit logging
- Cache invalidation
```

---

## Global Tables

```
Global Tables = Multi-region, multi-active replication

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   US-EAST-1                           EU-WEST-1                         │
│   ┌───────────────────┐               ┌───────────────────┐             │
│   │  DynamoDB Table   │◄─────────────▶│  DynamoDB Table   │             │
│   │  (Read/Write)     │  Async        │  (Read/Write)     │             │
│   └───────────────────┘  Replication  └───────────────────┘             │
│                          (<1 second)                                     │
│                                                                          │
│   Features:                                                              │
│   - Multi-active (write anywhere)                                       │
│   - Automatic conflict resolution (last writer wins)                    │
│   - Replication typically <1 second                                     │
│   - Same table name in all regions                                      │
│                                                                          │
│   Requirements:                                                          │
│   - Empty tables or existing global table                               │
│   - DynamoDB Streams enabled                                            │
│   - Same capacity mode and settings                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## DAX (DynamoDB Accelerator)

```
DAX = Fully managed in-memory cache for DynamoDB

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Application ──▶ DAX Cluster ──cache miss──▶ DynamoDB                  │
│                       │                                                 │
│                       └──cache hit (microseconds)                       │
│                                                                          │
│   Cache Types:                                                           │
│   1. Item Cache: GetItem, BatchGetItem results                          │
│   2. Query Cache: Query and Scan results                                │
│                                                                          │
│   Performance:                                                           │
│   - Microsecond latency (vs milliseconds)                               │
│   - Up to 10x read performance                                          │
│   - Handles millions of requests/sec                                    │
│                                                                          │
│   Considerations:                                                        │
│   - Eventually consistent reads only                                    │
│   - Same API (drop-in replacement)                                      │
│   - VPC only                                                            │
│   - Cost: Additional for cluster                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Interview Discussion Points

### When to use DynamoDB vs RDS?

```
Choose DynamoDB When:
- Known access patterns (key-value lookups)
- Need unlimited scale
- Single-digit ms latency required
- Serverless architecture
- Simple queries (no complex joins)
- High availability critical

Choose RDS/Aurora When:
- Complex queries with joins
- Ad-hoc query requirements
- ACID transactions across entities
- Existing relational schema
- Reporting/analytics workloads
- Unknown query patterns
```

### How do you handle hot partitions?

```
Problem: One partition key receives disproportionate traffic

Solutions:

1. Write Sharding
   - Add random suffix to partition key
   - user_123#1, user_123#2, ..., user_123#N
   - Query all shards and aggregate

2. Caching (DAX)
   - Cache hot items
   - Reduce DynamoDB reads

3. On-Demand Capacity
   - Handles spiky workloads better
   - No pre-provisioning needed

4. Better Key Design
   - Use composite keys
   - Distribute data across partitions
   - Avoid timestamp-based partition keys
```

### How do you design for a leaderboard?

```
Requirement: Get top N scores, get user's rank

Option 1: Sparse GSI
- GSI with score as sort key
- Query in descending order
- Limitation: Can't get specific user's rank efficiently

Option 2: Inverted Index Pattern
{
  PK: "LEADERBOARD#game1",
  SK: "SCORE#000099500#USER#123",  // Pad score for sorting
  user_id: "123",
  score: 99500
}

- Query with ScanIndexForward=false for top N
- To get rank: Query and count items with higher scores

Option 3: Materialized Rank
- Use Lambda + Streams to maintain rank
- Store rank as attribute
- Update ranks when scores change
- Trade-off: Write amplification
```

### How do you migrate from RDS to DynamoDB?

```
Migration Strategy:

1. Model Data for DynamoDB
   - Identify access patterns
   - Design single-table schema
   - Map relationships to item collections

2. Dual-Write Phase
   - Application writes to both
   - Validate consistency
   - Build confidence

3. Backfill Historical Data
   - AWS DMS for bulk migration
   - Custom scripts for transformation

4. Gradual Read Migration
   - Route reads to DynamoDB
   - Monitor performance
   - Compare results

5. Decommission RDS
   - Stop writes to RDS
   - Keep for rollback period
   - Final cleanup

Key Considerations:
- Denormalization (no joins)
- Transaction boundaries may change
- Different consistency models
- Reporting may need separate solution (export to S3 + Athena)
```
