# System Design Interview Framework

## The 4-Step Framework

```
┌─────────────────────────────────────────────────────────────┐
│                 System Design Interview                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Step 1: Requirements (5 min)                               │
│  ──────────────────────────                                 │
│  • Clarify functional requirements                          │
│  • Understand non-functional requirements                   │
│  • Define scope and constraints                             │
│                                                             │
│  Step 2: Estimation (5 min)                                 │
│  ────────────────────────                                   │
│  • Traffic estimation                                       │
│  • Storage estimation                                       │
│  • Bandwidth estimation                                     │
│                                                             │
│  Step 3: High-Level Design (15 min)                         │
│  ───────────────────────────────                            │
│  • Draw system architecture                                 │
│  • Identify major components                                │
│  • Define data flow                                         │
│                                                             │
│  Step 4: Deep Dive (15 min)                                 │
│  ──────────────────────────                                 │
│  • Detail critical components                               │
│  • Database schema                                          │
│  • API design                                               │
│  • Handle edge cases                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1: Requirements Gathering

### Questions to Ask

```
Functional Requirements:
• What are the core features?
• Who are the users?
• What actions can users perform?
• What data needs to be stored/retrieved?

Non-Functional Requirements:
• What's the expected scale? (users, requests/sec)
• What's the acceptable latency?
• Is eventual consistency acceptable?
• What's the availability requirement?

Constraints:
• Any technology constraints?
• Budget constraints?
• Timeline constraints?
• Existing systems to integrate with?
```

### Example: Design Twitter

```
Me: "Let me clarify the requirements..."

Functional:
✓ Post tweets (text, images)
✓ Follow/unfollow users
✓ View home timeline (posts from followed)
✓ Search tweets
✗ Direct messages (out of scope)
✗ Trends (out of scope)

Non-Functional:
• 500M users, 200M DAU
• Read-heavy (100:1 read to write)
• Feed latency < 200ms
• Eventually consistent is fine
• 99.9% availability

Constraints:
• Global user base
• Mobile-first design
```

---

## Step 2: Back-of-Envelope Estimation

### Traffic Estimation

```
Given: 500M users, 200M DAU

Tweets per day:
• 10% users tweet daily = 20M users
• Avg 2 tweets/user = 40M tweets/day
• Tweets/second = 40M / 86400 ≈ 500 TPS

Timeline reads per day:
• Each user reads timeline 5x/day
• 200M × 5 = 1B reads/day
• Reads/second = 1B / 86400 ≈ 12,000 RPS
```

### Storage Estimation

```
Tweet storage:
• Tweet: 280 chars + metadata ≈ 500 bytes
• 40M tweets/day × 500 bytes = 20 GB/day
• Per year: 20 GB × 365 = 7.3 TB/year

Media storage:
• 20% tweets have images
• 8M images/day × 500 KB = 4 TB/day
• Per year: 4 TB × 365 = 1.4 PB/year
```

### Bandwidth Estimation

```
Incoming (writes):
• 500 TPS × 500 bytes = 250 KB/s

Outgoing (reads):
• Timeline: 12K RPS × 100 tweets × 500 bytes
• = 600 MB/s
• Plus images: ~2 GB/s
```

### Quick Reference Numbers

```
Powers of 2:
2^10 = 1 KB (thousand)
2^20 = 1 MB (million)
2^30 = 1 GB (billion)
2^40 = 1 TB (trillion)

Time:
1 day = 86,400 seconds ≈ 100K seconds
1 month ≈ 2.5M seconds
1 year ≈ 30M seconds

Requests:
1M requests/day = ~12 requests/second
10M requests/day = ~120 requests/second
100M requests/day = ~1200 requests/second
```

---

## Step 3: High-Level Design

### Drawing the Architecture

```
1. Start with users/clients
2. Add load balancers
3. Add application servers
4. Add data stores
5. Add caches
6. Add async components (queues)
7. Show data flow arrows
```

### Example Architecture Template

```
┌──────────┐
│ Clients  │
│(Web/Mobile)│
└────┬─────┘
     │
┌────▼──────────────────────────────────────────────────────┐
│                     CDN / Edge                             │
└────┬──────────────────────────────────────────────────────┘
     │
┌────▼──────────────────────────────────────────────────────┐
│                    Load Balancer                           │
└────┬──────────────────────────────────────────────────────┘
     │
┌────▼─────┐    ┌───────────┐    ┌───────────┐
│   API    │───▶│   Cache   │───▶│  Database │
│ Servers  │    │  (Redis)  │    │           │
└────┬─────┘    └───────────┘    └───────────┘
     │
┌────▼──────────────────────────────────────────────────────┐
│              Message Queue (Kafka/SQS)                     │
└────┬──────────────────────────────────────────────────────┘
     │
┌────▼─────┐
│ Workers  │
└──────────┘
```

### Component Descriptions

```
For each component, briefly explain:
• What it does
• Why it's needed
• Technology choice (and why)

Example:
"We'll use Redis as a cache layer to store pre-computed
feeds. This reduces database load and provides sub-millisecond
reads. We chose Redis over Memcached because we need
sorted sets for timeline ordering."
```

---

## Step 4: Deep Dive

### Areas to Deep Dive

```
Pick 2-3 based on interviewer interest:

1. Database Design
   • Schema design
   • Indexing strategy
   • Sharding approach

2. API Design
   • Endpoints
   • Request/response format
   • Authentication

3. Key Algorithms
   • Feed ranking
   • Search indexing
   • Rate limiting

4. Scaling
   • Bottleneck identification
   • Horizontal scaling
   • Caching strategy

5. Edge Cases
   • Celebrity problem
   • Hot spots
   • Failure scenarios
```

### Database Deep Dive Example

```sql
-- Posts table
CREATE TABLE posts (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    content VARCHAR(280),
    created_at TIMESTAMP,
    INDEX idx_user_time (user_id, created_at DESC)
);

-- Sharding strategy
-- Shard by user_id for user-specific queries
-- Use consistent hashing with 1024 virtual nodes

-- For timeline:
-- Option 1: Pre-materialized in Redis
-- Option 2: Fan-out on write to Cassandra

-- Trade-off discussion:
-- Redis: Faster reads, memory limited
-- Cassandra: Cheaper storage, good for write-heavy
```

---

## Common Pitfalls

### 1. Jumping to Solution

```
❌ "Let's use Kafka for messaging..."
✓ "Let me first understand the requirements..."
```

### 2. No Numbers

```
❌ "We'll need a lot of storage"
✓ "With 40M tweets/day at 500 bytes each, we need 20 GB/day"
```

### 3. Over-Engineering

```
❌ Adding every possible feature
✓ Focus on core requirements, mention extensions
```

### 4. Single Points of Failure

```
❌ Single database server
✓ Primary with replicas, mention failover
```

### 5. Ignoring Trade-offs

```
❌ "This solution is perfect"
✓ "This approach trades consistency for availability because..."
```

---

## Communication Tips

### Verbalize Your Thinking

```
"I'm thinking we need to consider..."
"One trade-off here is..."
"Let me validate this assumption..."
"I'm choosing X over Y because..."
```

### Check in with Interviewer

```
"Does this level of detail make sense?"
"Should I go deeper into this component?"
"Is there a specific area you'd like me to focus on?"
```

### Handle Unknown Topics

```
"I haven't worked with X directly, but my understanding is..."
"I would research this further, but my hypothesis is..."
"In similar situations, I would..."
```

---

## Practice Checklist

```
Before the interview:

□ Practice drawing on whiteboard/virtual board
□ Know common architectures by heart
□ Memorize key numbers for estimation
□ Practice explaining out loud
□ Time yourself (45 min for full design)

During the interview:

□ Listen carefully to requirements
□ Ask clarifying questions
□ Make reasonable assumptions
□ Start with high-level, then detail
□ Discuss trade-offs
□ Acknowledge limitations
```

---

## Common System Design Questions

```
Easy/Medium:
• URL shortener
• Pastebin
• Rate limiter
• Key-value store

Medium:
• Twitter timeline
• Instagram
• Web crawler
• Notification system
• Autocomplete/Typeahead

Medium/Hard:
• YouTube/Netflix
• Uber/Lyft
• Messenger/WhatsApp
• Google Docs
• Distributed cache

Hard:
• Search engine
• Ad serving system
• Stock exchange
• Distributed file system
• Video conferencing
```
