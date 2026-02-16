# Estimation Cheatsheet

## Powers of 2

```
2^10 = 1,024           ≈ 1 Thousand (KB)
2^20 = 1,048,576       ≈ 1 Million (MB)
2^30 = 1,073,741,824   ≈ 1 Billion (GB)
2^40 = 1,099,511,627,776 ≈ 1 Trillion (TB)
```

---

## Time Conversions

```
1 second = 1,000 milliseconds
1 minute = 60 seconds
1 hour = 3,600 seconds
1 day = 86,400 seconds ≈ 100,000 seconds
1 week = 604,800 seconds ≈ 600,000 seconds
1 month = 2,592,000 seconds ≈ 2.5 Million seconds
1 year = 31,536,000 seconds ≈ 30 Million seconds
```

---

## Request Rate Conversions

```
1M requests/day    = 12 requests/second
10M requests/day   = 116 requests/second ≈ 120 RPS
100M requests/day  = 1,157 requests/second ≈ 1,200 RPS
1B requests/day    = 11,574 requests/second ≈ 12,000 RPS
```

**Quick formula:** `Daily requests / 100,000 ≈ requests/second`

---

## Latency Numbers Every Programmer Should Know

```
Operation                              Time
─────────────────────────────────────────────────
L1 cache reference                     0.5 ns
L2 cache reference                     7 ns
Main memory reference (RAM)            100 ns
SSD random read                        16 μs (16,000 ns)
Read 1 MB sequentially from SSD        1 ms
HDD seek                               10 ms
Read 1 MB sequentially from HDD        30 ms
Send packet US to Europe round trip    150 ms
```

**Key takeaways:**
- Memory is 100x faster than SSD
- SSD is 1000x faster than HDD
- Network adds 100-200ms latency

---

## Storage Sizes

### Text

```
Character (ASCII)        = 1 byte
Character (UTF-8)        = 1-4 bytes
Tweet (280 chars)        ≈ 300 bytes
Email                    ≈ 50 KB
Web page (HTML)          ≈ 100 KB
```

### Images

```
Thumbnail (100x100)      ≈ 10 KB
Small image (400x400)    ≈ 100 KB
Medium image (1024x768)  ≈ 500 KB - 1 MB
High-res image           ≈ 2-5 MB
Raw/uncompressed         ≈ 10-50 MB
```

### Video

```
1 min SD video (480p)    ≈ 20 MB
1 min HD video (720p)    ≈ 60 MB
1 min Full HD (1080p)    ≈ 130 MB
1 min 4K video           ≈ 350 MB
```

### Audio

```
1 min MP3 (128kbps)      ≈ 1 MB
1 min high quality       ≈ 2-3 MB
```

---

## Common Scale Numbers

### Big Tech Scale

```
Google:
- 8.5B searches/day = 100K searches/second
- 720K hours of video uploaded to YouTube daily

Facebook:
- 2B DAU
- 100B messages/day on WhatsApp

Twitter:
- 500M tweets/day = 6K tweets/second
- 200M DAU

Netflix:
- 220M subscribers
- 15% of global downstream bandwidth

Amazon:
- 66K orders/hour during peak
- 12M products in catalog
```

### User Behavior

```
Social media user:
- Checks app 10-20 times/day
- Spends 30 min/day
- Posts 1-2 times/day
- Follows ~200 accounts

E-commerce user:
- Browses 10 pages per session
- 2-3% conversion rate
- Average order: $50-100

Messaging user:
- 50-100 messages/day
- Online 4 hours/day
- In 10-20 chat groups
```

---

## Database Performance

### Single Machine Limits

```
PostgreSQL/MySQL:
- ~10K transactions/second
- ~1M rows for full table scan (<1s)
- ~100M rows with good indexing

Redis:
- ~100K operations/second
- ~10GB practical limit per instance

MongoDB:
- ~50K writes/second
- ~100K reads/second

Elasticsearch:
- ~10K writes/second
- ~100K searches/second
```

### Network

```
Single server network:
- 1 Gbps = 125 MB/s theoretical
- 10 Gbps = 1.25 GB/s theoretical

Practical throughput ~60-70% of theoretical

Load balancer:
- HAProxy: 1M+ connections
- Nginx: 10K+ concurrent connections
```

---

## Calculation Examples

### Example 1: Storage for Twitter

```
Given: 500M users, 10% post daily, 2 tweets each

Daily tweets:
500M × 10% × 2 = 100M tweets/day

Tweet size:
280 chars + metadata = 500 bytes

Daily storage:
100M × 500 bytes = 50 GB/day

Yearly storage:
50 GB × 365 = 18 TB/year

5-year storage:
18 TB × 5 = 90 TB
```

### Example 2: Bandwidth for Video Streaming

```
Given: 100M DAU, each watches 1 hour, HD video

Concurrent users (assume 10% at peak):
100M × 10% = 10M concurrent

Bandwidth per stream (720p):
~3 Mbps

Total bandwidth:
10M × 3 Mbps = 30 Pbps = 30,000 Tbps

(This is why Netflix uses CDN edge servers!)
```

### Example 3: Chat Message Throughput

```
Given: 50M DAU, 100 messages/user/day

Total messages:
50M × 100 = 5B messages/day

Messages per second:
5B / 86,400 ≈ 60K messages/second

Storage (avg message 100 bytes):
5B × 100 bytes = 500 GB/day
```

---

## Quick Estimation Formulas

### Request Rate

```
RPS = Daily requests / 86,400
Peak RPS = Average RPS × 2-3 (for spikes)
```

### Storage

```
Total storage = Items × Size per item × Retention period
Growth rate = Daily new items × Size per item
```

### Bandwidth

```
Bandwidth = Request rate × Response size
Peak bandwidth = Average bandwidth × 3-5
```

### Servers Needed

```
Servers = Total RPS / RPS per server
Add 30-50% for redundancy
Round up to handle peak
```

---

## Memory/Cache Sizing

### 80/20 Rule (Pareto)

```
20% of data serves 80% of requests
Cache the hot 20% in memory

Example:
- 1TB total data
- Cache 20% = 200GB
- Use Redis cluster with 200GB capacity
```

### Cache Hit Rate

```
With good caching:
- 80-90% hit rate is achievable
- 95%+ for static content

Miss penalty:
- Cache miss = DB query = 10-100ms
- Cache hit = 1-10ms
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│                    Quick Reference                          │
├─────────────────────────────────────────────────────────────┤
│ 1 million requests/day    ≈ 12 RPS                         │
│ 1 day                     ≈ 100K seconds                   │
│ 1 year                    ≈ 30M seconds                    │
│ 1 KB                      = 2^10 bytes                     │
│ 1 MB                      = 2^20 bytes                     │
│ 1 GB                      = 2^30 bytes                     │
│ 1 TB                      = 2^40 bytes                     │
│ RAM access                ≈ 100 ns                         │
│ SSD read                  ≈ 16 μs                          │
│ HDD seek                  ≈ 10 ms                          │
│ Network RTT               ≈ 150 ms                         │
│ Single server             ≈ 10K RPS                        │
│ Redis                     ≈ 100K ops/sec                   │
│ PostgreSQL                ≈ 10K TPS                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Practice Problems

### Problem 1: Design Instagram

```
Given:
- 1B users, 500M DAU
- 100M photos uploaded/day
- Users view 200 photos/day

Calculate:
1. Photo uploads/second = ?
2. Photo views/second = ?
3. Daily storage for new photos = ?
4. Yearly storage growth = ?

Answers:
1. 100M / 86,400 ≈ 1,200 uploads/sec
2. 500M × 200 / 86,400 ≈ 1.15M views/sec
3. 100M × 500 KB = 50 TB/day
4. 50 TB × 365 = 18 PB/year
```

### Problem 2: Design Uber

```
Given:
- 100M monthly riders, 5M drivers
- 20M rides/day
- Avg ride: 15 min

Calculate:
1. Concurrent rides at peak = ?
2. Location updates/second (update every 4s) = ?
3. Storage for ride history/day = ?

Answers:
1. 20M × 15/60 / 24 × 2 (peak factor) ≈ 400K concurrent
2. 5M drivers / 4s ≈ 1.25M updates/sec
3. 20M × 1 KB/ride = 20 GB/day
```
