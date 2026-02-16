# Database Replication

## Why Replicate?

```
1. High Availability   - Survive node failures
2. Read Scalability    - Distribute read load
3. Geographic Locality - Data closer to users
4. Disaster Recovery   - Backup in different location
```

---

## Replication Topologies

### Single Leader (Master-Slave)

```
                    ┌─────────────┐
                    │   Leader    │◄──── All writes
                    │  (Primary)  │
                    └──────┬──────┘
                           │
                      Replication
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
   ┌───────────┐     ┌───────────┐     ┌───────────┐
   │ Follower  │     │ Follower  │     │ Follower  │
   │ (Replica) │     │ (Replica) │     │ (Replica) │
   └───────────┘     └───────────┘     └───────────┘
         ▲                 ▲                 ▲
         └─────────────────┴─────────────────┘
                        Reads
```

**Pros:**
- Simple to understand
- Clear consistency model
- No write conflicts

**Cons:**
- Leader is bottleneck for writes
- Leader failure requires failover
- Replication lag

### Multi-Leader (Master-Master)

```
   ┌───────────┐          ┌───────────┐
   │  Leader   │◄────────▶│  Leader   │
   │  (US)     │          │  (EU)     │
   └───────────┘          └───────────┘
        │                       │
   Reads/Writes            Reads/Writes
        │                       │
   US Users               EU Users
```

**Pros:**
- Writes in multiple regions
- Better write performance
- Tolerates leader failure

**Cons:**
- Write conflicts possible
- Complex conflict resolution
- Eventual consistency

### Leaderless (Peer-to-Peer)

```
    ┌───────────┐
    │   Node    │
    └───────────┘
         /\
        /  \
       /    \
      /      \
     ▼        ▼
┌───────┐  ┌───────┐
│ Node  │◀▶│ Node  │
└───────┘  └───────┘

Client writes to multiple nodes
Client reads from multiple nodes
Quorum determines success
```

**Examples:** Cassandra, DynamoDB, Riak

**Pros:**
- No single point of failure
- High availability
- Flexible consistency

**Cons:**
- Complex
- Conflict resolution needed
- Read/write coordination

---

## Synchronous vs Asynchronous Replication

### Synchronous

```
Write → Leader → Replicas (wait for all/some) → Confirm
```

```javascript
async function writeSync(data) {
  // Write to primary
  await primary.write(data);

  // Wait for all replicas
  await Promise.all(replicas.map(r => r.write(data)));

  return { success: true };
}
```

**Pros:** Strong consistency
**Cons:** Higher latency, availability issues if replica down

### Asynchronous

```
Write → Leader → Confirm → (Background) Replicas
```

```javascript
async function writeAsync(data) {
  // Write to primary
  await primary.write(data);

  // Confirm immediately
  const result = { success: true };

  // Replicate in background
  setImmediate(() => {
    replicas.forEach(r => r.write(data).catch(console.error));
  });

  return result;
}
```

**Pros:** Lower latency, better availability
**Cons:** Replication lag, potential data loss

### Semi-Synchronous

```
Write → Leader → At least 1 replica → Confirm → Other replicas async
```

```javascript
async function writeSemiSync(data) {
  await primary.write(data);

  // Wait for at least one replica
  await Promise.race(replicas.map(r => r.write(data)));

  // Others async
  setImmediate(() => {
    replicas.slice(1).forEach(r => r.write(data));
  });

  return { success: true };
}
```

---

## Replication Methods

### Statement-Based

Replicate SQL statements.

```
Leader executes: INSERT INTO users VALUES (1, 'John', NOW())
Sends to replicas: "INSERT INTO users VALUES (1, 'John', NOW())"

Problems:
- Non-deterministic functions (NOW(), RAND())
- Auto-increment IDs
- Triggers and stored procedures
```

### Write-Ahead Log (WAL) Shipping

Replicate low-level log entries.

```
WAL Entry: {
  lsn: 12345,
  operation: INSERT,
  table: users,
  data: { id: 1, name: 'John', created_at: '2024-01-15 10:30:00' }
}
```

**Pros:** Exact byte-level replication
**Cons:** Coupled to storage engine, harder upgrades

### Logical Replication

Replicate row-level changes.

```sql
-- PostgreSQL logical replication
CREATE PUBLICATION my_pub FOR TABLE users, orders;

-- On replica
CREATE SUBSCRIPTION my_sub
CONNECTION 'host=primary dbname=mydb'
PUBLICATION my_pub;
```

**Pros:** Cross-version, selective replication
**Cons:** More complex setup

### Row-Based (MySQL)

```
Event: UPDATE
Table: users
Where: id = 1
Before: { name: 'John' }
After:  { name: 'Jonathan' }
```

---

## Handling Replication Lag

### Read-Your-Writes Consistency

```javascript
// Track writes per user
const userWriteTimestamps = new Map();

async function write(userId, data) {
  const result = await primary.write(data);
  userWriteTimestamps.set(userId, Date.now());
  return result;
}

async function read(userId, query) {
  const lastWrite = userWriteTimestamps.get(userId);

  if (lastWrite && Date.now() - lastWrite < 10000) {
    // Recent write - read from primary
    return primary.query(query);
  }

  // Safe to read from replica
  return replica.query(query);
}
```

### Monotonic Reads

```javascript
// Track last read position per user
const userReadPositions = new Map();

async function read(userId, query) {
  const minPosition = userReadPositions.get(userId) || 0;

  // Find replica at or past this position
  const replica = await selectReplicaWithMinPosition(minPosition);
  const result = await replica.query(query);

  // Update position
  userReadPositions.set(userId, replica.currentPosition);

  return result;
}
```

### Consistent Prefix Reads

Ensure causally related writes are read in order.

```
Write 1: "Question: What's 2+2?"
Write 2: "Answer: 4"

Without consistent prefix, user might see:
"Answer: 4" (before seeing the question)

Solution: Ensure writes to same partition/shard
```

---

## Failover

### Automatic Failover

```
1. Detect failure (heartbeat timeout)
2. Elect new leader (consensus)
3. Reconfigure replicas
4. Update clients

Timeline:
0s     - Leader stops responding
30s    - Heartbeat timeout, failure detected
35s    - Election starts
40s    - New leader elected
45s    - Replicas reconfigured
50s    - Clients redirected
```

### Manual Failover

```sql
-- PostgreSQL pg_promote
SELECT pg_promote();

-- MySQL
STOP SLAVE;
RESET SLAVE ALL;
-- Configure as new primary
```

### Failover Challenges

```
Split Brain:
- Two nodes think they're leader
- Both accept writes
- Data divergence

Solution: Fencing
- STONITH (Shoot The Other Node In The Head)
- Quorum-based decisions
- Lease-based leadership
```

```javascript
// Lease-based leadership
class LeaderElection {
  async acquireLease() {
    const result = await etcd.put(
      'leader',
      this.nodeId,
      { lease: 30 } // 30 second lease
    );

    if (result.succeeded) {
      this.isLeader = true;
      this.startLeaseRenewal();
    }
  }

  startLeaseRenewal() {
    setInterval(async () => {
      await etcd.lease.keepAlive();
    }, 10000); // Renew every 10 seconds
  }
}
```

---

## Conflict Resolution

### Last Write Wins (LWW)

```javascript
// Simple but can lose data
function resolve(entries) {
  return entries.reduce((latest, entry) =>
    entry.timestamp > latest.timestamp ? entry : latest
  );
}

// Problem:
// Node A: SET user.name = "John" at T1
// Node B: SET user.email = "john@new.com" at T2
// T2 > T1, so email change wins, name change lost!
```

### Merge Function

```javascript
// Application-specific merge
function mergeShoppingCart(cartA, cartB) {
  const merged = { items: {} };

  // Union of all items
  for (const [itemId, quantity] of Object.entries(cartA.items)) {
    merged.items[itemId] = Math.max(
      quantity,
      cartB.items[itemId] || 0
    );
  }

  for (const [itemId, quantity] of Object.entries(cartB.items)) {
    if (!merged.items[itemId]) {
      merged.items[itemId] = quantity;
    }
  }

  return merged;
}
```

### Operational Transformation

```javascript
// For collaborative editing
// Transform operations to maintain consistency

// User A: Insert "X" at position 2
// User B: Insert "Y" at position 1

// Without transformation:
// Document: "abcd"
// A applies: "abXcd"
// B applies: "aYbXcd" (wrong position!)

// With transformation:
// B's operation transformed: Insert "Y" at position 1 + 1 = 2
// (because A's insert shifted positions)
```

### CRDTs

```javascript
// Grow-only Set - always mergeable
class GSet {
  constructor() {
    this.items = new Set();
  }

  add(item) {
    this.items.add(item);
  }

  merge(other) {
    // Union is always valid
    for (const item of other.items) {
      this.items.add(item);
    }
  }

  value() {
    return [...this.items];
  }
}
```

---

## Multi-Region Replication

```
┌─────────────────────────────────────────────────────────────┐
│                         Global                               │
│                                                             │
│   ┌───────────────┐              ┌───────────────┐         │
│   │   US-EAST     │              │   EU-WEST     │         │
│   │  ┌─────────┐  │   Async      │  ┌─────────┐  │         │
│   │  │ Primary │◄─┼──────────────┼─▶│ Primary │  │         │
│   │  └────┬────┘  │  Replication │  └────┬────┘  │         │
│   │       │       │              │       │       │         │
│   │  ┌────▼────┐  │              │  ┌────▼────┐  │         │
│   │  │Replicas │  │              │  │Replicas │  │         │
│   │  └─────────┘  │              │  └─────────┘  │         │
│   │               │              │               │         │
│   │   US Users    │              │   EU Users    │         │
│   └───────────────┘              └───────────────┘         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Conflict-Free Approaches

```javascript
// 1. Single primary per entity
const regionForUser = {
  'user_us_123': 'us-east',
  'user_eu_456': 'eu-west'
};

function getPrimaryForUser(userId) {
  return regionForUser[userId] || 'us-east';
}

// 2. Read local, write to home region
async function updateUserProfile(userId, data) {
  const homeRegion = getUserHomeRegion(userId);

  if (homeRegion !== currentRegion) {
    // Forward write to home region
    return await remoteRegion(homeRegion).updateUser(userId, data);
  }

  // Local write
  return await db.updateUser(userId, data);
}
```

---

## Interview Questions

### Q: How do you handle replication lag in a read-heavy application?

1. **Read-your-writes** - Route to primary after write
2. **Sticky sessions** - Same replica for session
3. **Synchronous replication** - For critical reads (slower)
4. **Version vectors** - Track data versions
5. **Causal consistency** - Order related operations

### Q: What happens if a replica falls too far behind?

1. **Catch up** - Replay WAL/binlog
2. **Snapshot + WAL** - If too far, snapshot then replay
3. **Rebuild** - Re-sync from scratch
4. **Remove from pool** - Don't serve reads until caught up

### Q: How do you choose between single-leader and multi-leader?

**Single-leader when:**
- Strong consistency required
- Single region deployment
- Simpler operations preferred

**Multi-leader when:**
- Multi-region deployment
- Write latency critical per region
- Can handle conflict resolution
- Offline operation needed (mobile)
