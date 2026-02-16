# Design: URL Shortener

## Requirements

### Functional Requirements
- Shorten long URLs to short URLs
- Redirect short URLs to original URLs
- Optional: Custom short URLs
- Optional: Analytics (click count, location)
- Optional: Expiration

### Non-Functional Requirements
- High availability
- Low latency redirects
- Scale: 100M URLs created/day, 10:1 read:write ratio

### Capacity Estimation

```
Writes: 100M URLs/day = ~1200 URLs/second
Reads: 1B redirects/day = ~12000 requests/second

Storage (5 years):
- 100M URLs/day × 365 × 5 = 182.5B URLs
- Avg URL: 500 bytes = 91 TB

Short URL length:
- Base62 (a-z, A-Z, 0-9) = 62 characters
- 62^7 = 3.5 trillion combinations (7 chars enough)
```

---

## High-Level Design

```
┌──────────┐     ┌─────────────┐     ┌─────────────┐
│  Client  │────▶│ Load        │────▶│ API Server  │
└──────────┘     │ Balancer    │     │             │
                 └─────────────┘     └──────┬──────┘
                                            │
                       ┌────────────────────┼────────────────────┐
                       ▼                    ▼                    ▼
                ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
                │   Cache     │      │  Database   │      │ Key Gen     │
                │   (Redis)   │      │ (Cassandra) │      │  Service    │
                └─────────────┘      └─────────────┘      └─────────────┘
```

---

## URL Shortening Approaches

### Approach 1: Hash and Truncate

```javascript
function shortenUrl(longUrl) {
  // Hash the URL
  const hash = crypto.createHash('md5').update(longUrl).digest('base64');

  // Take first 7 characters, make URL-safe
  const shortCode = hash
    .replace(/\+/g, '')
    .replace(/\//g, '')
    .substring(0, 7);

  return shortCode;
}

// Problem: Collisions!
// Solution: Check DB, if exists, append character and retry
```

### Approach 2: Counter with Base62

```javascript
const CHARSET = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';

function toBase62(num) {
  let result = '';
  while (num > 0) {
    result = CHARSET[num % 62] + result;
    num = Math.floor(num / 62);
  }
  return result.padStart(7, 'a'); // Ensure minimum length
}

// Use distributed counter (Redis INCR or DB sequence)
async function shortenUrl(longUrl) {
  const counter = await redis.incr('url_counter');
  const shortCode = toBase62(counter);
  await db.insert({ shortCode, longUrl });
  return shortCode;
}
```

### Approach 3: Pre-generated Keys (Recommended)

```javascript
// Key Generation Service
class KeyGenerator {
  constructor() {
    this.usedKeys = new Set();
    this.availableKeys = [];
  }

  // Pre-generate and store keys in DB
  async initialize() {
    // Generate batch of keys
    const keys = await db.query('SELECT key FROM key_pool WHERE used = false LIMIT 1000');
    this.availableKeys = keys.map(k => k.key);
  }

  async getKey() {
    if (this.availableKeys.length < 100) {
      await this.refillKeys();
    }
    return this.availableKeys.pop();
  }

  async refillKeys() {
    // Mark keys as used in DB and load new ones
    const keys = await db.query(`
      UPDATE key_pool
      SET used = true
      WHERE key IN (SELECT key FROM key_pool WHERE used = false LIMIT 1000)
      RETURNING key
    `);
    this.availableKeys.push(...keys.map(k => k.key));
  }
}

// Key Generation Worker (separate process)
async function generateKeys() {
  while (true) {
    const count = await db.query('SELECT COUNT(*) FROM key_pool WHERE used = false');
    if (count < 1000000) {
      const newKeys = [];
      for (let i = 0; i < 100000; i++) {
        newKeys.push(generateRandomBase62(7));
      }
      await db.insertMany('key_pool', newKeys);
    }
    await sleep(60000); // Check every minute
  }
}
```

---

## Database Schema

### URL Table

```sql
CREATE TABLE urls (
  short_code VARCHAR(10) PRIMARY KEY,
  long_url TEXT NOT NULL,
  user_id VARCHAR(50),
  created_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,
  click_count BIGINT DEFAULT 0
);

CREATE INDEX idx_urls_user ON urls(user_id);
CREATE INDEX idx_urls_created ON urls(created_at);
```

### Key Pool Table

```sql
CREATE TABLE key_pool (
  key VARCHAR(10) PRIMARY KEY,
  used BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_keys_unused ON key_pool(used) WHERE used = FALSE;
```

---

## API Design

```javascript
// Shorten URL
POST /api/shorten
{
  "longUrl": "https://example.com/very/long/url/path",
  "customAlias": "my-link",  // Optional
  "expiresAt": "2024-12-31"  // Optional
}

Response:
{
  "shortUrl": "https://short.ly/abc123",
  "shortCode": "abc123",
  "longUrl": "https://example.com/very/long/url/path",
  "expiresAt": "2024-12-31"
}

// Redirect (handled by server)
GET /:shortCode
→ 301/302 Redirect to longUrl

// Get URL info
GET /api/urls/:shortCode
{
  "shortCode": "abc123",
  "longUrl": "https://example.com/...",
  "clickCount": 1234,
  "createdAt": "2024-01-15"
}
```

---

## Implementation

```javascript
const express = require('express');
const redis = require('redis');

const app = express();
const cache = redis.createClient();

// Create short URL
app.post('/api/shorten', async (req, res) => {
  const { longUrl, customAlias, expiresAt } = req.body;

  // Validate URL
  if (!isValidUrl(longUrl)) {
    return res.status(400).json({ error: 'Invalid URL' });
  }

  // Check for existing URL (deduplication)
  const existing = await db.query(
    'SELECT short_code FROM urls WHERE long_url = ?',
    [longUrl]
  );
  if (existing) {
    return res.json({ shortCode: existing.short_code });
  }

  // Get short code
  let shortCode;
  if (customAlias) {
    // Check if custom alias is available
    const taken = await db.query(
      'SELECT 1 FROM urls WHERE short_code = ?',
      [customAlias]
    );
    if (taken) {
      return res.status(409).json({ error: 'Alias already taken' });
    }
    shortCode = customAlias;
  } else {
    shortCode = await keyGenerator.getKey();
  }

  // Store in database
  await db.query(
    'INSERT INTO urls (short_code, long_url, expires_at) VALUES (?, ?, ?)',
    [shortCode, longUrl, expiresAt]
  );

  res.json({
    shortUrl: `https://short.ly/${shortCode}`,
    shortCode,
    longUrl
  });
});

// Redirect
app.get('/:shortCode', async (req, res) => {
  const { shortCode } = req.params;

  // Check cache first
  let longUrl = await cache.get(`url:${shortCode}`);

  if (!longUrl) {
    // Cache miss - query database
    const result = await db.query(
      'SELECT long_url, expires_at FROM urls WHERE short_code = ?',
      [shortCode]
    );

    if (!result) {
      return res.status(404).send('URL not found');
    }

    if (result.expires_at && new Date(result.expires_at) < new Date()) {
      return res.status(410).send('URL expired');
    }

    longUrl = result.long_url;

    // Cache for 24 hours
    await cache.setex(`url:${shortCode}`, 86400, longUrl);
  }

  // Increment click count asynchronously
  incrementClickCount(shortCode);

  // Redirect
  res.redirect(301, longUrl);
});

// Async click counting
async function incrementClickCount(shortCode) {
  await redis.incr(`clicks:${shortCode}`);
  // Batch write to DB periodically
}
```

---

## Scaling Considerations

### Read Scaling

```
Cache Strategy:
- Cache popular URLs in Redis
- 80/20 rule: 20% of URLs get 80% of traffic
- TTL: 24 hours for most, longer for popular

CDN for redirects:
- Edge locations for faster redirects
- Cache redirect responses
```

### Write Scaling

```
Key Generation:
- Pre-generate keys in batches
- Multiple key servers with key ranges
- Server 1: keys 0-999,999
- Server 2: keys 1,000,000-1,999,999

Database:
- Sharding by short_code (consistent hashing)
- Write replicas for durability
```

### Analytics at Scale

```javascript
// Async analytics pipeline
app.get('/:shortCode', async (req, res) => {
  // ... redirect logic ...

  // Fire and forget analytics
  analyticsQueue.push({
    shortCode,
    timestamp: Date.now(),
    ip: req.ip,
    userAgent: req.headers['user-agent'],
    referer: req.headers.referer
  });
});

// Kafka consumer for analytics
consumer.on('message', async (event) => {
  // Batch insert to analytics DB
  await analyticsDb.insert('clicks', {
    short_code: event.shortCode,
    clicked_at: event.timestamp,
    country: geoIp.lookup(event.ip)?.country,
    device: parseUserAgent(event.userAgent).device
  });
});
```

---

## Interview Discussion Points

### Why Base62?

- URL-safe characters
- No special characters to encode
- Case-sensitive doubles combinations

### 301 vs 302 Redirect?

- **301 Permanent**: Better for SEO, cached by browser
- **302 Temporary**: Better for analytics (always hits server)

### How to prevent abuse?

- Rate limiting per IP/user
- URL validation (no malicious content)
- CAPTCHAs for anonymous users
- Block known malicious domains

### How to handle hot URLs?

- Multiple cache layers (CDN → Redis → DB)
- Read replicas
- Pre-warm cache for known viral content
