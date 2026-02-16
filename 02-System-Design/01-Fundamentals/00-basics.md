# System Design Fundamentals

## Overview

System design interviews test your ability to design large-scale distributed systems. As a Lead engineer, you're expected to drive these discussions, make trade-off decisions, and demonstrate deep understanding of scalability concepts.

---

## The System Design Interview Framework

### RESHADED Framework

| Step | Focus | Time |
|------|-------|------|
| **R**equirements | Clarify functional & non-functional | 5 min |
| **E**stimation | Back-of-envelope calculations | 5 min |
| **S**torage | Data model, schema design | 5 min |
| **H**igh-level Design | Components, architecture diagram | 10 min |
| **A**PI Design | Endpoints, contracts | 5 min |
| **D**etailed Design | Deep dive into 1-2 components | 10 min |
| **E**valuation | Trade-offs, alternatives | 5 min |
| **D**istinguish | Scalability, monitoring, security | 5 min |

---

## Step 1: Requirements Gathering

### Functional Requirements
- Core features the system must support
- User stories and use cases
- Data inputs and outputs

### Non-Functional Requirements

| Requirement | Questions to Ask |
|-------------|-----------------|
| **Scale** | How many users? DAU/MAU? |
| **Performance** | Latency requirements? P99? |
| **Availability** | Uptime SLA? 99.9%? 99.99%? |
| **Consistency** | Strong or eventual? |
| **Durability** | Data loss tolerance? |
| **Security** | Authentication? Encryption? |

### Example: URL Shortener

**Functional:**
- Create short URL from long URL
- Redirect short URL to original
- Custom aliases (optional)
- Analytics (optional)

**Non-Functional:**
- 100M new URLs/month
- 10:1 read:write ratio
- Low latency redirects (<100ms)
- High availability (99.99%)

---

## Step 2: Back-of-Envelope Estimation

### Common Numbers to Know

| Metric | Value |
|--------|-------|
| 1 day | 86,400 seconds (~100K) |
| 1 month | 2.5M seconds |
| 1 year | 31.5M seconds |
| QPS for 1M daily users | ~12 QPS average |
| 1 KB | 1,000 bytes |
| 1 MB | 1,000,000 bytes |
| 1 GB | 1,000,000,000 bytes |
| 1 TB | 1,000 GB |
| SSD random read | ~100 μs |
| Network round trip (same datacenter) | ~0.5 ms |
| Network round trip (cross-country) | ~50-100 ms |

### Estimation Template

```
Given: 100M new URLs per month

Writes:
- 100M / (30 * 24 * 3600) = ~40 writes/second

Reads (10:1 ratio):
- 40 * 10 = 400 reads/second

Storage (5 years):
- Each URL entry: 500 bytes (URL + metadata)
- 100M * 12 * 5 * 500 bytes = 3 TB

Bandwidth:
- Write: 40 * 500 bytes = 20 KB/s
- Read: 400 * 500 bytes = 200 KB/s
```

---

## Step 3: High-Level Design

### Common Architecture Patterns

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Clients   │────▶│ Load Balancer│────▶│ API Servers │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
       ┌───────────────────┬───────────────────┼───────────────────┐
       ▼                   ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Cache    │     │  Database   │     │ Object Store│     │Message Queue│
│   (Redis)   │     │ (PostgreSQL)│     │    (S3)     │     │  (Kafka)    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

### Key Components

| Component | Purpose | Examples |
|-----------|---------|----------|
| Load Balancer | Distribute traffic | AWS ELB, Nginx |
| API Gateway | Rate limiting, auth | Kong, AWS API Gateway |
| Application Servers | Business logic | Node.js, Java |
| Cache | Reduce latency | Redis, Memcached |
| Database | Persistent storage | PostgreSQL, MongoDB |
| CDN | Static content | CloudFront, Cloudflare |
| Message Queue | Async processing | Kafka, RabbitMQ |
| Object Storage | Files, media | S3, GCS |

---

## Core Concepts

### Scalability

**Vertical Scaling (Scale Up)**
- Add more resources to single machine
- Simpler but has limits
- Example: Bigger database server

**Horizontal Scaling (Scale Out)**
- Add more machines
- More complex but unlimited
- Requires stateless design

```
Vertical:           Horizontal:
┌─────────┐         ┌─────┐ ┌─────┐ ┌─────┐
│  BIG    │         │ App │ │ App │ │ App │
│ SERVER  │         └─────┘ └─────┘ └─────┘
└─────────┘              ▲     ▲     ▲
                         └─────┼─────┘
                               │
                         Load Balancer
```

### Load Balancing

**Algorithms:**
- **Round Robin**: Cycle through servers
- **Least Connections**: Route to least busy
- **IP Hash**: Consistent routing by IP
- **Weighted**: Route based on server capacity

**Layer 4 vs Layer 7:**
- Layer 4 (Transport): TCP/UDP, faster, no content inspection
- Layer 7 (Application): HTTP, can route by content/headers

### Caching

**Cache Strategies:**

```javascript
// Cache-Aside (Lazy Loading)
function getData(key) {
  let data = cache.get(key);
  if (data === null) {
    data = database.get(key);
    cache.set(key, data);
  }
  return data;
}

// Write-Through
function saveData(key, value) {
  cache.set(key, value);
  database.set(key, value);
}

// Write-Behind (Write-Back)
function saveData(key, value) {
  cache.set(key, value);
  // Database updated asynchronously
  queue.push({ key, value });
}
```

**Cache Eviction Policies:**
- **LRU** (Least Recently Used): Remove least recently accessed
- **LFU** (Least Frequently Used): Remove least accessed overall
- **FIFO**: Remove oldest entries
- **TTL**: Expire after time limit

### Database Choices

**SQL (Relational):**
- Strong consistency, ACID transactions
- Complex queries with JOINs
- Structured data with relationships
- Examples: PostgreSQL, MySQL

**NoSQL:**
- **Document**: Flexible schema (MongoDB)
- **Key-Value**: Simple, fast (Redis, DynamoDB)
- **Wide-Column**: Large scale analytics (Cassandra)
- **Graph**: Relationships (Neo4j)

**When to use SQL vs NoSQL:**

| Use SQL | Use NoSQL |
|---------|-----------|
| Complex relationships | Simple key-value access |
| ACID required | High write throughput |
| Complex queries | Flexible schema |
| Structured data | Horizontal scaling |

---

## CAP Theorem

You can only guarantee 2 of 3:

```
        Consistency
           /\
          /  \
         /    \
        /  CA  \
       /________\
      /\        /\
     /  \  CP  /  \
    / AP \    /    \
   /______\  /______\
 Availability  Partition
               Tolerance
```

- **CA**: Single node systems (traditional RDBMS)
- **CP**: Consistent but may be unavailable during partitions (HBase, MongoDB)
- **AP**: Available but may return stale data (Cassandra, DynamoDB)

**In distributed systems, Partition Tolerance is mandatory**, so you choose between:
- **CP**: Sacrifice availability for consistency
- **AP**: Sacrifice consistency for availability

### PACELC Theorem

Extended CAP: If **P**artition, choose **A** or **C**; **E**lse, choose **L**atency or **C**onsistency.

---

## Consistency Models

### Strong Consistency
- All reads see the most recent write
- Simpler to reason about
- Higher latency

### Eventual Consistency
- Reads may see stale data temporarily
- Higher availability and performance
- Complexity in handling conflicts

### Read-Your-Writes Consistency
- User sees their own updates immediately
- Others may see stale data temporarily

---

## Database Scaling

### Replication

**Single Leader (Master-Slave):**
```
        ┌─────────┐
        │ Primary │ ◀── Writes
        └────┬────┘
             │ Replicate
    ┌────────┼────────┐
    ▼        ▼        ▼
┌───────┐┌───────┐┌───────┐
│Replica││Replica││Replica│ ◀── Reads
└───────┘└───────┘└───────┘
```

**Multi-Leader:**
- Multiple nodes accept writes
- Conflict resolution needed
- Good for multi-datacenter

**Leaderless:**
- Any node can accept reads/writes
- Quorum-based consistency
- Example: Cassandra, DynamoDB

### Sharding (Partitioning)

**Horizontal Sharding:**
```
User ID 1-1000     → Shard 1
User ID 1001-2000  → Shard 2
User ID 2001-3000  → Shard 3
```

**Sharding Strategies:**

| Strategy | Pros | Cons |
|----------|------|------|
| Range-based | Easy to understand | Hotspots possible |
| Hash-based | Even distribution | Range queries hard |
| Directory-based | Flexible | Directory is bottleneck |
| Geo-based | Data locality | Uneven distribution |

**Consistent Hashing:**
- Minimizes data movement when adding/removing nodes
- Used by: Cassandra, DynamoDB, Memcached

---

## Message Queues

### Use Cases
- Async processing
- Decoupling services
- Load leveling
- Event sourcing

### Patterns

**Point-to-Point (Queue):**
```
Producer → Queue → Consumer
```

**Pub/Sub:**
```
            ┌─→ Subscriber 1
Publisher → Topic ─→ Subscriber 2
            └─→ Subscriber 3
```

### Guarantees

| Guarantee | Description |
|-----------|-------------|
| At-most-once | May lose messages |
| At-least-once | May duplicate messages |
| Exactly-once | No loss, no duplicates (hardest) |

---

## Rate Limiting

### Algorithms

**Token Bucket:**
```javascript
class TokenBucket {
  constructor(capacity, refillRate) {
    this.tokens = capacity;
    this.capacity = capacity;
    this.refillRate = refillRate;
    this.lastRefill = Date.now();
  }

  allow() {
    this.refill();
    if (this.tokens > 0) {
      this.tokens--;
      return true;
    }
    return false;
  }

  refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(
      this.capacity,
      this.tokens + elapsed * this.refillRate
    );
    this.lastRefill = now;
  }
}
```

**Sliding Window Log:**
- Track timestamps of requests
- Count requests in window
- More accurate but more memory

**Fixed Window Counter:**
- Count per time window
- Simple but edge case issues

---

## Common System Design Questions

### Easy/Medium
1. **URL Shortener** (TinyURL)
2. **Paste Service** (Pastebin)
3. **Rate Limiter**
4. **Key-Value Store**
5. **Unique ID Generator**

### Medium
6. **Twitter/Social Feed**
7. **Instagram/Photo Sharing**
8. **Web Crawler**
9. **Notification System**
10. **Chat System** (WhatsApp)

### Hard
11. **YouTube/Video Streaming**
12. **Google Search**
13. **Uber/Ride Sharing**
14. **Distributed Cache**
15. **Stock Exchange**

---

## Interview Tips

1. **Don't dive into details immediately** - Start broad, then deep dive
2. **Ask clarifying questions** - Show you think about requirements
3. **State assumptions explicitly** - "I'm assuming 100M users"
4. **Discuss trade-offs** - There's no perfect solution
5. **Draw diagrams** - Visual communication is key
6. **Think out loud** - Share your reasoning process
7. **Be prepared to defend choices** - Know why you chose each component

---

## Key Takeaways

1. Start with **requirements** and **estimations**
2. Design for **scale from the start**
3. Understand **CAP theorem** trade-offs
4. Know when to use **SQL vs NoSQL**
5. **Caching** is crucial for performance
6. **Sharding** enables horizontal database scaling
7. **Message queues** decouple services
8. Always discuss **trade-offs**
