# Design a URL Shortener (TinyURL)

## Overview

A URL shortener creates short aliases for long URLs. This is a classic system design interview question that covers many fundamental concepts.

---

## Step 1: Requirements

### Functional Requirements
1. Given a long URL, generate a short URL
2. Given a short URL, redirect to original URL
3. Optional: Custom short URLs
4. Optional: Analytics (click count, location, etc.)
5. Optional: URL expiration

### Non-Functional Requirements
- **Scale**: 100M new URLs per month
- **Latency**: Redirect < 100ms
- **Availability**: 99.99% uptime
- **Read:Write ratio**: 10:1 (reads are redirects)

### Out of Scope
- User accounts
- Rate limiting (mention briefly)
- Spam detection

---

## Step 2: Estimations

### Traffic
```
New URLs: 100M / month
         = 100M / (30 * 24 * 3600)
         = ~40 writes/second

Redirects (10:1): 400 reads/second
Peak (2x): 800 reads/second
```

### Storage (5 years)
```
Total URLs: 100M * 12 * 5 = 6 billion URLs

Per URL entry:
- Short URL: 7 chars = 7 bytes
- Long URL: avg 200 bytes
- Created at: 8 bytes
- Expiry: 8 bytes
- User ID: 8 bytes
Total: ~250 bytes

Total storage: 6B * 250 bytes = 1.5 TB
```

### Bandwidth
```
Write: 40 * 250 bytes = 10 KB/s
Read: 400 * 250 bytes = 100 KB/s
```

### Memory (Cache - 20% hot URLs)
```
20% of daily reads cached
Daily reads: 400 * 86400 = 35M
20% unique: 7M URLs
Cache size: 7M * 250 bytes = 1.75 GB
```

---

## Step 3: API Design

### Create Short URL
```
POST /api/v1/shorten
Request:
{
  "longUrl": "https://example.com/very/long/path",
  "customAlias": "my-link",  // optional
  "expiresAt": "2025-12-31"  // optional
}

Response:
{
  "shortUrl": "https://tiny.url/abc1234",
  "longUrl": "https://example.com/very/long/path",
  "expiresAt": "2025-12-31"
}
```

### Redirect (GET)
```
GET /{shortCode}
Response: 301/302 Redirect to long URL
```

### Get Analytics (Optional)
```
GET /api/v1/stats/{shortCode}
Response:
{
  "clicks": 1500,
  "createdAt": "2024-01-15",
  "topCountries": [...]
}
```

---

## Step 4: Database Schema

```sql
CREATE TABLE urls (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  short_code VARCHAR(7) UNIQUE NOT NULL,
  long_url VARCHAR(2048) NOT NULL,
  user_id BIGINT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expires_at TIMESTAMP,
  click_count INT DEFAULT 0,

  INDEX idx_short_code (short_code),
  INDEX idx_user_id (user_id)
);

-- For analytics (optional)
CREATE TABLE clicks (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  url_id BIGINT NOT NULL,
  clicked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  ip_address VARCHAR(45),
  user_agent TEXT,
  country VARCHAR(2),

  INDEX idx_url_id (url_id),
  INDEX idx_clicked_at (clicked_at)
);
```

### Database Choice
**SQL (PostgreSQL)** is suitable because:
- Strong consistency needed (no duplicate short codes)
- Simple query patterns
- ACID transactions for uniqueness
- 1.5 TB is manageable with sharding

---

## Step 5: Short Code Generation

### Option 1: Base62 Encoding of Auto-Increment ID

```python
ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"

def encode_base62(num):
    if num == 0:
        return ALPHABET[0]

    result = []
    while num > 0:
        result.append(ALPHABET[num % 62])
        num //= 62

    return ''.join(reversed(result))

def decode_base62(s):
    num = 0
    for char in s:
        num = num * 62 + ALPHABET.index(char)
    return num
```

**Pros**: Predictable, unique, efficient
**Cons**: Sequential (predictable), requires centralized counter

### Option 2: Hash-Based (MD5/SHA256)

```python
import hashlib

def generate_short_code(long_url):
    hash_object = hashlib.md5(long_url.encode())
    hex_dig = hash_object.hexdigest()
    # Take first 7 characters
    return base62_encode(int(hex_dig[:12], 16))[:7]
```

**Pros**: Distributed, deterministic
**Cons**: Collisions possible, need collision handling

### Option 3: Pre-Generated Keys (Key Generation Service)

```
┌─────────────────┐
│ Key Generation  │
│    Service      │
│                 │
│ ┌─────────────┐ │
│ │ Used Keys   │ │
│ └─────────────┘ │
│ ┌─────────────┐ │
│ │ Unused Keys │ │
│ └─────────────┘ │
└────────┬────────┘
         │
         ▼
   App Servers
```

**Pros**: Fast, no collision, distributed
**Cons**: Additional component, key exhaustion handling

### Recommendation: Base62 + Distributed ID Generation

Use a distributed ID generator like:
- **Snowflake ID**: 64-bit unique IDs
- **ULID**: Lexicographically sortable
- **UUID v7**: Time-ordered UUIDs

```python
# Snowflake-like ID
# 41 bits: timestamp (69 years)
# 10 bits: machine ID (1024 machines)
# 12 bits: sequence (4096 per ms)

def generate_id(machine_id):
    timestamp = current_ms() - EPOCH
    sequence = get_next_sequence()

    id = (timestamp << 22) | (machine_id << 12) | sequence
    return encode_base62(id)
```

---

## Step 6: High-Level Architecture

```
                            ┌─────────────────────┐
                            │        CDN          │
                            │  (Static + Caching) │
                            └──────────┬──────────┘
                                       │
┌─────────┐     ┌─────────────┐     ┌──▼──────────┐
│ Clients │────▶│   DNS/LB    │────▶│ API Gateway │
└─────────┘     └─────────────┘     └──────┬──────┘
                                           │
              ┌────────────────────────────┼────────────────────────────┐
              │                            │                            │
              ▼                            ▼                            ▼
       ┌────────────┐             ┌────────────┐              ┌────────────┐
       │   Write    │             │   Read     │              │ Analytics  │
       │  Service   │             │  Service   │              │  Service   │
       └─────┬──────┘             └─────┬──────┘              └─────┬──────┘
             │                          │                           │
             │    ┌───────────────┐     │                          │
             │    │     Cache     │     │                          │
             │    │    (Redis)    │◀────┤                          │
             │    └───────────────┘     │                          │
             │                          │                          │
             ▼                          ▼                          ▼
       ┌─────────────────────────────────────────────────────────────────┐
       │                         Database                                 │
       │                   (PostgreSQL Cluster)                          │
       │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
       │   │ Shard 1  │  │ Shard 2  │  │ Shard 3  │  │ Shard 4  │      │
       │   └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
       └─────────────────────────────────────────────────────────────────┘
```

---

## Step 7: Detailed Component Design

### Write Path (Create Short URL)

```
1. Client sends POST /shorten with long URL
2. API Gateway validates request
3. Write Service:
   a. Check if URL already shortened (cache/DB)
   b. Generate unique short code
   c. Store in database
   d. Invalidate/update cache
4. Return short URL to client
```

### Read Path (Redirect)

```
1. Client requests GET /{shortCode}
2. CDN checks cache
   - Hit: Return redirect (301)
   - Miss: Forward to origin
3. Read Service:
   a. Check Redis cache
      - Hit: Return long URL
      - Miss: Query database
   b. Store in cache
4. Return 301/302 redirect
5. Async: Update click count
```

### 301 vs 302 Redirect

| Code | Type | Caching | Use Case |
|------|------|---------|----------|
| 301 | Permanent | Browser caches | Faster, less analytics |
| 302 | Temporary | No caching | Better analytics |

**Recommendation**: Use 302 for analytics, or 301 with server-side tracking.

---

## Step 8: Caching Strategy

### What to Cache
- Short code → Long URL mapping
- Popular URLs (LRU eviction)
- Recently created URLs

### Cache Configuration (Redis)

```python
# Cache-Aside Pattern
def get_long_url(short_code):
    # Try cache first
    long_url = redis.get(f"url:{short_code}")

    if long_url:
        return long_url

    # Cache miss - query DB
    long_url = db.query(
        "SELECT long_url FROM urls WHERE short_code = ?",
        short_code
    )

    if long_url:
        # Cache for 24 hours
        redis.setex(f"url:{short_code}", 86400, long_url)

    return long_url
```

### Cache Invalidation
- **TTL-based**: Expire after 24 hours
- **Write-through**: Update cache on URL updates
- **Lazy invalidation**: Check expiry on read

---

## Step 9: Database Sharding

### Sharding Strategy: Hash-based on short_code

```python
def get_shard(short_code):
    return hash(short_code) % NUM_SHARDS
```

**Pros**: Even distribution
**Cons**: Range queries across shards

### Alternative: Range-based on ID

```
Shard 1: IDs 0 - 1B
Shard 2: IDs 1B - 2B
...
```

**Pros**: Easy range queries
**Cons**: Hotspots on newest shard

### Replication

Each shard has:
- 1 Primary (writes)
- 2 Replicas (reads)
- Async replication (< 100ms lag)

---

## Step 10: Analytics (Optional)

### Approach 1: Synchronous (Simple)

```python
def redirect(short_code):
    # Get URL
    long_url = get_long_url(short_code)

    # Increment counter
    db.execute(
        "UPDATE urls SET click_count = click_count + 1 WHERE short_code = ?",
        short_code
    )

    return redirect(long_url, 302)
```

**Problem**: Adds latency to redirects

### Approach 2: Async with Message Queue (Better)

```
Client → Read Service → Redirect (fast)
              │
              └──▶ Kafka → Analytics Service → DB
```

```python
def redirect(short_code):
    long_url = get_long_url(short_code)

    # Async: Send click event to Kafka
    kafka.send("clicks", {
        "short_code": short_code,
        "timestamp": now(),
        "ip": request.ip,
        "user_agent": request.user_agent
    })

    return redirect(long_url, 302)
```

---

## Step 11: Handling Edge Cases

### Custom Aliases
- Check if alias is available before accepting
- Reserve some keywords (api, admin, etc.)
- Validate alias format (alphanumeric, length)

### URL Expiration
- Add `expires_at` column
- Check on redirect
- Background job to clean expired URLs

### Collision Handling
```python
def create_short_url(long_url, max_retries=5):
    for _ in range(max_retries):
        short_code = generate_code()
        try:
            db.insert(short_code, long_url)
            return short_code
        except UniqueViolation:
            continue
    raise Exception("Could not generate unique code")
```

---

## Trade-offs Discussion

| Decision | Trade-off |
|----------|-----------|
| Base62 vs Hash | Predictability vs Simplicity |
| SQL vs NoSQL | Consistency vs Horizontal scaling |
| 301 vs 302 | Performance vs Analytics |
| Sync vs Async analytics | Simplicity vs Latency |
| Cache TTL | Freshness vs Cache hit rate |

---

## Scaling Considerations

### 10x Scale (1B URLs/month)
- Add more shards
- Increase cache capacity
- Add read replicas
- Consider NoSQL (DynamoDB)

### Global Scale
- Multi-region deployment
- CDN for redirect caching
- Regional databases
- DNS-based routing

---

## Key Takeaways

1. **Start simple**: Base62 encoding works for most cases
2. **Cache aggressively**: Most URLs are read-heavy
3. **Async analytics**: Don't slow down redirects
4. **Shard by short code**: Even distribution
5. **Consider 301 vs 302**: Based on analytics needs
6. **Plan for expiration**: URLs shouldn't live forever
