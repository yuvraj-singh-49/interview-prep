# CAP Theorem and Consistency

## CAP Theorem

In a distributed system, you can only guarantee 2 of 3 properties:

```
        Consistency
           /\
          /  \
         /    \
        /      \
       /   CA   \
      /          \
     /____________\
    CP            AP
   /                \
  /                  \
Partition        Availability
Tolerance
```

### Consistency (C)

Every read receives the most recent write or an error.

```
┌──────────┐     ┌──────────┐
│ Node A   │     │ Node B   │
│ value=5  │────▶│ value=5  │  ← Both nodes same
└──────────┘     └──────────┘
     │
  Write value=10
     │
     ▼
┌──────────┐     ┌──────────┐
│ Node A   │────▶│ Node B   │
│ value=10 │     │ value=10 │  ← Update propagated
└──────────┘     └──────────┘
```

### Availability (A)

Every request receives a response (not error), without guarantee of most recent write.

```
Request → Any available node → Response (even if stale)
```

### Partition Tolerance (P)

System continues operating despite network partitions.

```
┌──────────┐         ┌──────────┐
│ Node A   │    ✕    │ Node B   │
│ value=10 │◄───────▶│ value=5  │
└──────────┘ Network └──────────┘
             Partition

System must choose:
- Refuse requests (consistent but not available)
- Serve requests (available but not consistent)
```

---

## CAP Trade-offs in Real Systems

### CP Systems (Consistent + Partition Tolerant)

**Sacrifice availability during partitions.**

```
Examples:
- MongoDB (with majority writes)
- HBase
- Redis Cluster (in some modes)
- etcd
- Zookeeper

Use when: Data correctness is critical
- Financial transactions
- Inventory management
- Configuration systems
```

### AP Systems (Available + Partition Tolerant)

**Sacrifice consistency during partitions.**

```
Examples:
- Cassandra
- CouchDB
- DynamoDB (default settings)
- DNS

Use when: System must always respond
- Social media feeds
- Shopping carts
- Analytics
```

### CA Systems (Consistent + Available)

**Only possible without network partitions (single node).**

```
Examples:
- Single-node PostgreSQL
- Single-node MySQL

Not really distributed - network partitions
are inevitable in distributed systems.
```

---

## Consistency Models

### Strong Consistency

All nodes see the same data at the same time.

```javascript
// Write to primary, sync replicate to all
async function write(key, value) {
  await primary.write(key, value);
  await Promise.all(replicas.map(r => r.write(key, value)));
  // Only return after ALL replicas confirm
}

// Any node returns current value
async function read(key) {
  return anyNode.read(key); // Always correct
}
```

### Eventual Consistency

All replicas eventually converge to same value.

```javascript
// Write to any node
async function write(key, value) {
  await node.write(key, value);
  // Return immediately
  // Background: gossip to other nodes
}

// Read might return stale data
async function read(key) {
  return nearestNode.read(key); // Might be stale
}

// Eventually all nodes have same value
```

### Causal Consistency

Operations that are causally related are seen in the same order.

```
Client 1: POST "I got a new job!"     → A
Client 1: POST "Starting Monday!"     → B (caused by A)

All clients see: A before B (causal order)
But unrelated posts can appear in any order
```

### Read Your Own Writes

Client always sees their own updates.

```javascript
async function updateProfile(userId, data) {
  await db.write(`user:${userId}`, data);
  // Store version/timestamp
  session.lastWriteVersion = Date.now();
}

async function getProfile(userId) {
  const data = await db.read(`user:${userId}`);
  // Check if this replica has our write
  if (data.version < session.lastWriteVersion) {
    // Read from primary or wait
    return await primary.read(`user:${userId}`);
  }
  return data;
}
```

### Monotonic Reads

Once you read a value, subsequent reads won't return older values.

```
Client reads from Replica A: value = 10
Client reads from Replica B: value >= 10 (never 5)

Implementation: Stick to same replica or track versions
```

---

## Consistency Levels

### Tunable Consistency (Cassandra/DynamoDB)

```
Write Consistency:
- ONE: Write to 1 node → return
- QUORUM: Write to (N/2)+1 nodes → return
- ALL: Write to all nodes → return

Read Consistency:
- ONE: Read from 1 node
- QUORUM: Read from (N/2)+1 nodes, return latest
- ALL: Read from all nodes

Strong consistency formula:
R + W > N
Where R=read replicas, W=write replicas, N=total replicas

Example (3 replicas):
- R=2, W=2: Quorum reads + quorum writes = strong
- R=1, W=3: Any read after all-write = strong
- R=1, W=1: No guarantee = eventual
```

### Read-After-Write Patterns

```javascript
// Pattern 1: Read from primary after write
async function updateAndRead(id, data) {
  await primary.write(id, data);
  return await primary.read(id); // Read from same node
}

// Pattern 2: Version-based
async function updateAndRead(id, data) {
  const version = await primary.write(id, data);
  return await readWithMinVersion(id, version);
}

// Pattern 3: Synchronous replication
async function updateAndRead(id, data) {
  await writeToAllReplicas(id, data);
  return await anyReplica.read(id);
}
```

---

## Conflict Resolution

### Last Write Wins (LWW)

```javascript
// Simple but can lose data
const entry = {
  value: 'data',
  timestamp: Date.now(),
  nodeId: 'node-1'
};

function resolve(entries) {
  return entries.reduce((latest, entry) =>
    entry.timestamp > latest.timestamp ? entry : latest
  );
}
```

### Vector Clocks

```javascript
class VectorClock {
  constructor() {
    this.clock = {};
  }

  increment(nodeId) {
    this.clock[nodeId] = (this.clock[nodeId] || 0) + 1;
  }

  merge(other) {
    for (const [nodeId, count] of Object.entries(other.clock)) {
      this.clock[nodeId] = Math.max(this.clock[nodeId] || 0, count);
    }
  }

  compare(other) {
    // Returns: 'before', 'after', 'concurrent'
    let dominated = false, dominates = false;

    const allNodes = new Set([
      ...Object.keys(this.clock),
      ...Object.keys(other.clock)
    ]);

    for (const nodeId of allNodes) {
      const thisVal = this.clock[nodeId] || 0;
      const otherVal = other.clock[nodeId] || 0;

      if (thisVal < otherVal) dominated = true;
      if (thisVal > otherVal) dominates = true;
    }

    if (dominated && !dominates) return 'before';
    if (dominates && !dominated) return 'after';
    if (dominated && dominates) return 'concurrent';
    return 'equal';
  }
}
```

### CRDTs (Conflict-free Replicated Data Types)

```javascript
// G-Counter (Grow-only counter)
class GCounter {
  constructor(nodeId) {
    this.nodeId = nodeId;
    this.counts = {};
  }

  increment() {
    this.counts[this.nodeId] = (this.counts[this.nodeId] || 0) + 1;
  }

  value() {
    return Object.values(this.counts).reduce((a, b) => a + b, 0);
  }

  merge(other) {
    for (const [nodeId, count] of Object.entries(other.counts)) {
      this.counts[nodeId] = Math.max(this.counts[nodeId] || 0, count);
    }
  }
}

// LWW-Register (Last Write Wins)
class LWWRegister {
  constructor() {
    this.value = null;
    this.timestamp = 0;
  }

  set(value) {
    this.value = value;
    this.timestamp = Date.now();
  }

  merge(other) {
    if (other.timestamp > this.timestamp) {
      this.value = other.value;
      this.timestamp = other.timestamp;
    }
  }
}

// G-Set (Grow-only set)
class GSet {
  constructor() {
    this.items = new Set();
  }

  add(item) {
    this.items.add(item);
  }

  merge(other) {
    for (const item of other.items) {
      this.items.add(item);
    }
  }
}
```

---

## PACELC Theorem

Extension of CAP: What happens when there's **no partition**?

```
If Partition (P):
  Choose Availability (A) or Consistency (C)
Else (E):
  Choose Latency (L) or Consistency (C)

Examples:
- DynamoDB: PA/EL (Available + Low latency)
- Cassandra: PA/EL (configurable)
- MongoDB: PC/EC (Consistent even if slower)
- PNUTS: PC/EL (Consistent during partitions, fast normally)
```

---

## Consensus Algorithms

### Paxos (Simplified)

```
Roles: Proposer, Acceptor, Learner

Phase 1 (Prepare):
Proposer → "Prepare(n)" → Acceptors
Acceptors → "Promise(n, previous_value)" → Proposer

Phase 2 (Accept):
Proposer → "Accept(n, value)" → Acceptors
Acceptors → "Accepted(n, value)" → Learners

Majority agreeing = Consensus reached
```

### Raft (More understandable)

```
Leader Election:
1. All nodes start as Followers
2. Timeout → Candidate → Request votes
3. Majority votes → Leader
4. Leader sends heartbeats

Log Replication:
1. Client → Leader: Write request
2. Leader appends to log
3. Leader → Followers: Replicate entry
4. Majority confirm → Entry committed
5. Leader → Client: Success
```

```javascript
// Simplified Raft state
class RaftNode {
  constructor() {
    this.state = 'follower'; // follower, candidate, leader
    this.currentTerm = 0;
    this.votedFor = null;
    this.log = [];
    this.commitIndex = 0;
  }

  startElection() {
    this.state = 'candidate';
    this.currentTerm++;
    this.votedFor = this.id;

    const votes = 1; // Vote for self
    // Request votes from all other nodes
    // If majority → become leader
  }

  appendEntries(entries, leaderId, term) {
    if (term >= this.currentTerm) {
      this.state = 'follower';
      this.currentTerm = term;
      // Append entries to log
      return { success: true };
    }
    return { success: false };
  }
}
```

---

## Real-World Examples

### Banking System (Strong Consistency)

```
Requirement: Account balance must never be wrong

Solution:
- Single primary database
- Synchronous replication
- Two-phase commit for transfers
- Sacrifice availability for correctness
```

### Social Media Feed (Eventual Consistency)

```
Requirement: High availability, global scale

Solution:
- Multi-region deployment
- Async replication
- User might see slightly stale feed
- New posts appear within seconds
```

### Shopping Cart (Session Consistency)

```
Requirement: User sees their own updates

Solution:
- Read-your-writes consistency
- Sticky sessions or versioning
- Cart data can be eventually consistent across regions
```

---

## Interview Questions

### Q: Explain CAP theorem with a real example.

**Scenario:** User posts a tweet in US, friend reads in Europe.

**CP choice:**
- Block friend's read until tweet replicates to Europe
- Or return error: "temporarily unavailable"
- Tweet always visible if returned

**AP choice:**
- Friend's read succeeds immediately
- Might not see new tweet yet (stale)
- Eventually tweet appears

### Q: How do you achieve strong consistency in a distributed system?

1. **Single leader** - All writes go through one node
2. **Synchronous replication** - Wait for all/majority replicas
3. **Consensus protocols** - Raft/Paxos for leader election and agreement
4. **Distributed transactions** - 2PC/3PC (but slow)

### Q: When would you choose eventual consistency?

- **High availability required** - System must always respond
- **Low latency required** - Can't wait for sync replication
- **Stale data acceptable** - Social feeds, likes, views
- **Geographic distribution** - Users worldwide
- **Write-heavy workloads** - Analytics, logging
