# Design: Web Crawler

## Requirements

### Functional Requirements
- Crawl web pages starting from seed URLs
- Extract and follow links
- Store page content for indexing
- Respect robots.txt
- Handle different content types
- Support recrawling for freshness

### Non-Functional Requirements
- Scalability: Crawl 1B pages/month
- Politeness: Don't overwhelm servers
- Robustness: Handle failures gracefully
- Extensibility: Support new content types
- Distributed: Run across multiple machines

### Capacity Estimation

```
Pages to crawl: 1B pages/month
= ~400 pages/second

Average page size: 500KB
Storage per month: 1B × 500KB = 500TB
Bandwidth: 400 × 500KB = 200 MB/s

URLs in frontier: ~10B (10 URLs per page average)
URL storage: 10B × 100 bytes = 1TB
```

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Seed URLs                                      │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         URL Frontier                                     │
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │
│    │ Priority    │  │ Politeness  │  │ Back Queue  │                   │
│    │ Queue       │  │ (per host)  │  │ Selector    │                   │
│    └─────────────┘  └─────────────┘  └─────────────┘                   │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────────────────────┐
         ▼                       ▼                       ▼               ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Fetcher 1     │    │   Fetcher 2     │    │   Fetcher N     │
│  (Worker)       │    │  (Worker)       │    │  (Worker)       │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘
         │                      │                      │
         └──────────────────────┼──────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Content Processor                                │
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │
│    │ Parser      │  │ Dedup       │  │ Link        │                   │
│    │ (HTML/etc)  │  │ Detection   │  │ Extractor   │                   │
│    └─────────────┘  └─────────────┘  └─────────────┘                   │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
     ┌─────────────────┐  ┌─────────────┐  ┌─────────────────┐
     │ Content Store   │  │ URL Store   │  │ New URLs        │
     │ (S3/HDFS)       │  │ (visited)   │  │ → Frontier      │
     └─────────────────┘  └─────────────┘  └─────────────────┘
```

---

## URL Frontier

### Priority & Politeness Queue

```javascript
class URLFrontier {
  constructor(redis, numBackQueues = 1000) {
    this.redis = redis;
    this.numBackQueues = numBackQueues;
  }

  async addURL(url, priority = 1) {
    const domain = this.extractDomain(url);

    // Check if URL already seen
    const seen = await this.redis.sismember('seen_urls', url);
    if (seen) return;

    // Mark as seen
    await this.redis.sadd('seen_urls', url);

    // Add to priority front queue
    await this.redis.zadd('front_queue', priority, url);

    // Map domain to back queue
    const backQueueId = this.getBackQueueId(domain);
    await this.redis.sadd(`back_queue:${backQueueId}`, url);
    await this.redis.hset('domain_to_queue', domain, backQueueId);
  }

  async getNextURL() {
    // Select a back queue that's ready (not rate limited)
    const readyQueue = await this.selectReadyQueue();
    if (!readyQueue) {
      return null;  // All queues rate limited
    }

    // Get URL from that queue
    const url = await this.redis.spop(`back_queue:${readyQueue}`);
    if (!url) return null;

    // Record crawl time for politeness
    const domain = this.extractDomain(url);
    await this.recordCrawlTime(domain);

    return url;
  }

  async selectReadyQueue() {
    const now = Date.now();

    // Find queues with URLs that haven't been crawled recently
    for (let i = 0; i < this.numBackQueues; i++) {
      const domain = await this.redis.hget(`queue_${i}:domain`, 'current');
      if (!domain) continue;

      const lastCrawl = await this.redis.hget('domain_last_crawl', domain);
      const minDelay = await this.getMinDelay(domain);

      if (!lastCrawl || (now - parseInt(lastCrawl)) > minDelay) {
        return i;
      }
    }

    return null;
  }

  async getMinDelay(domain) {
    // Check robots.txt crawl-delay
    const robotsDelay = await this.redis.hget('robots_delay', domain);
    if (robotsDelay) {
      return Math.max(parseInt(robotsDelay) * 1000, 1000);
    }

    // Default politeness delay
    return 1000;  // 1 second between requests to same domain
  }

  async recordCrawlTime(domain) {
    await this.redis.hset('domain_last_crawl', domain, Date.now());
  }

  getBackQueueId(domain) {
    // Consistent hash domain to back queue
    let hash = 0;
    for (let i = 0; i < domain.length; i++) {
      hash = ((hash << 5) - hash + domain.charCodeAt(i)) | 0;
    }
    return Math.abs(hash) % this.numBackQueues;
  }
}
```

### URL Priority Calculation

```javascript
class URLPrioritizer {
  calculatePriority(url, metadata = {}) {
    let priority = 0;

    // Domain authority
    const domainRank = this.getDomainRank(this.extractDomain(url));
    priority += domainRank * 0.3;

    // URL depth (shorter paths higher priority)
    const depth = url.split('/').length - 3;
    priority += (10 - Math.min(depth, 10)) * 0.1;

    // Freshness (if recrawling)
    if (metadata.lastCrawled) {
      const age = Date.now() - metadata.lastCrawled;
      const ageScore = Math.min(age / (86400000 * 30), 1);  // Up to 30 days
      priority += ageScore * 0.2;
    }

    // Content type preference
    if (this.isHighValueURL(url)) {
      priority += 0.2;
    }

    // Backlink count
    if (metadata.backlinks) {
      priority += Math.min(metadata.backlinks / 1000, 0.2);
    }

    return Math.min(priority, 1);
  }

  isHighValueURL(url) {
    // Prefer main content over images, CSS, etc.
    const lowValuePatterns = [
      /\.(jpg|png|gif|css|js|pdf|zip)$/i,
      /\/tag\//,
      /\/page\/\d+/,
      /\?.*sort=/
    ];

    return !lowValuePatterns.some(p => p.test(url));
  }
}
```

---

## Fetcher

### HTTP Fetcher with Retries

```javascript
class Fetcher {
  constructor(options = {}) {
    this.timeout = options.timeout || 30000;
    this.maxRetries = options.maxRetries || 3;
    this.userAgent = options.userAgent || 'MyBot/1.0';
  }

  async fetch(url) {
    // Check robots.txt first
    const allowed = await this.checkRobotsTxt(url);
    if (!allowed) {
      return { status: 'disallowed', url };
    }

    let lastError;
    for (let attempt = 0; attempt < this.maxRetries; attempt++) {
      try {
        const response = await this.makeRequest(url);
        return this.processResponse(url, response);
      } catch (error) {
        lastError = error;

        if (!this.isRetryable(error)) {
          break;
        }

        await this.delay(Math.pow(2, attempt) * 1000);
      }
    }

    return {
      status: 'error',
      url,
      error: lastError.message
    };
  }

  async makeRequest(url) {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeout);

    try {
      const response = await fetch(url, {
        headers: {
          'User-Agent': this.userAgent,
          'Accept': 'text/html,application/xhtml+xml',
          'Accept-Language': 'en-US,en;q=0.9',
          'Accept-Encoding': 'gzip, deflate'
        },
        redirect: 'follow',
        signal: controller.signal
      });

      return response;
    } finally {
      clearTimeout(timeoutId);
    }
  }

  processResponse(url, response) {
    return {
      status: 'success',
      url,
      finalUrl: response.url,
      statusCode: response.status,
      headers: Object.fromEntries(response.headers),
      contentType: response.headers.get('content-type'),
      contentLength: response.headers.get('content-length'),
      body: response
    };
  }

  async checkRobotsTxt(url) {
    const domain = new URL(url).origin;
    const robotsUrl = `${domain}/robots.txt`;

    try {
      const rules = await this.getRobotsRules(robotsUrl);
      return this.isAllowed(url, rules);
    } catch {
      // If robots.txt not available, allow crawling
      return true;
    }
  }

  isRetryable(error) {
    if (error.name === 'AbortError') return true;
    if (error.code === 'ECONNRESET') return true;
    if (error.code === 'ETIMEDOUT') return true;
    return false;
  }
}
```

### Robots.txt Parser

```javascript
class RobotsParser {
  constructor() {
    this.cache = new Map();
  }

  async getRules(domain) {
    if (this.cache.has(domain)) {
      return this.cache.get(domain);
    }

    try {
      const response = await fetch(`${domain}/robots.txt`);
      const text = await response.text();
      const rules = this.parse(text);

      this.cache.set(domain, rules);
      return rules;
    } catch {
      return { allowed: [], disallowed: [], crawlDelay: null };
    }
  }

  parse(robotsTxt) {
    const rules = {
      userAgents: {},
      sitemaps: []
    };

    let currentAgent = '*';
    const lines = robotsTxt.split('\n');

    for (let line of lines) {
      line = line.trim();
      if (!line || line.startsWith('#')) continue;

      const [directive, ...valueParts] = line.split(':');
      const value = valueParts.join(':').trim();

      switch (directive.toLowerCase()) {
        case 'user-agent':
          currentAgent = value;
          if (!rules.userAgents[currentAgent]) {
            rules.userAgents[currentAgent] = { allow: [], disallow: [], crawlDelay: null };
          }
          break;
        case 'disallow':
          rules.userAgents[currentAgent]?.disallow.push(value);
          break;
        case 'allow':
          rules.userAgents[currentAgent]?.allow.push(value);
          break;
        case 'crawl-delay':
          rules.userAgents[currentAgent].crawlDelay = parseFloat(value);
          break;
        case 'sitemap':
          rules.sitemaps.push(value);
          break;
      }
    }

    return rules;
  }

  isAllowed(url, rules, userAgent = '*') {
    const path = new URL(url).pathname;
    const agentRules = rules.userAgents[userAgent] || rules.userAgents['*'] || {};

    // Check allow rules first (they have priority)
    for (const pattern of agentRules.allow || []) {
      if (this.matches(path, pattern)) {
        return true;
      }
    }

    // Check disallow rules
    for (const pattern of agentRules.disallow || []) {
      if (this.matches(path, pattern)) {
        return false;
      }
    }

    return true;
  }

  matches(path, pattern) {
    // Convert robots.txt pattern to regex
    const regexPattern = pattern
      .replace(/\*/g, '.*')
      .replace(/\$/g, '$');

    return new RegExp(`^${regexPattern}`).test(path);
  }
}
```

---

## Content Processor

### HTML Parser & Link Extractor

```javascript
const cheerio = require('cheerio');

class ContentProcessor {
  process(url, html, contentType) {
    if (!contentType?.includes('text/html')) {
      return { links: [], text: null };
    }

    const $ = cheerio.load(html);

    return {
      title: this.extractTitle($),
      text: this.extractText($),
      links: this.extractLinks($, url),
      metadata: this.extractMetadata($)
    };
  }

  extractTitle($) {
    return $('title').text().trim() ||
           $('meta[property="og:title"]').attr('content') ||
           $('h1').first().text().trim();
  }

  extractText($) {
    // Remove non-content elements
    $('script, style, nav, header, footer, aside, .sidebar, .ads').remove();

    // Get main content
    const mainContent = $('main, article, .content, #content').first();
    const text = mainContent.length ? mainContent.text() : $('body').text();

    // Clean and normalize whitespace
    return text
      .replace(/\s+/g, ' ')
      .trim()
      .substring(0, 100000);  // Limit text length
  }

  extractLinks($, baseUrl) {
    const links = new Set();
    const base = new URL(baseUrl);

    $('a[href]').each((_, element) => {
      try {
        const href = $(element).attr('href');
        if (!href) return;

        // Resolve relative URLs
        const absoluteUrl = new URL(href, base).href;

        // Filter out unwanted links
        if (this.isValidLink(absoluteUrl)) {
          links.add(this.normalizeUrl(absoluteUrl));
        }
      } catch {
        // Invalid URL, skip
      }
    });

    return Array.from(links);
  }

  isValidLink(url) {
    try {
      const parsed = new URL(url);

      // Only HTTP/HTTPS
      if (!['http:', 'https:'].includes(parsed.protocol)) {
        return false;
      }

      // Skip common non-page extensions
      const skipExtensions = ['.jpg', '.png', '.gif', '.pdf', '.zip', '.exe'];
      if (skipExtensions.some(ext => parsed.pathname.endsWith(ext))) {
        return false;
      }

      // Skip fragments, mailto, javascript
      if (url.startsWith('#') || url.startsWith('mailto:') || url.startsWith('javascript:')) {
        return false;
      }

      return true;
    } catch {
      return false;
    }
  }

  normalizeUrl(url) {
    const parsed = new URL(url);

    // Remove fragment
    parsed.hash = '';

    // Remove tracking parameters
    const trackingParams = ['utm_source', 'utm_medium', 'utm_campaign', 'fbclid', 'gclid'];
    trackingParams.forEach(param => parsed.searchParams.delete(param));

    // Normalize trailing slash
    if (parsed.pathname !== '/' && parsed.pathname.endsWith('/')) {
      parsed.pathname = parsed.pathname.slice(0, -1);
    }

    // Lowercase hostname
    parsed.hostname = parsed.hostname.toLowerCase();

    return parsed.href;
  }

  extractMetadata($) {
    return {
      description: $('meta[name="description"]').attr('content') ||
                   $('meta[property="og:description"]').attr('content'),
      keywords: $('meta[name="keywords"]').attr('content'),
      author: $('meta[name="author"]').attr('content'),
      publishDate: $('meta[property="article:published_time"]').attr('content'),
      canonical: $('link[rel="canonical"]').attr('href'),
      language: $('html').attr('lang')
    };
  }
}
```

---

## Duplicate Detection

### Simhash for Near-Duplicate Detection

```javascript
class SimHash {
  constructor(hashBits = 64) {
    this.hashBits = hashBits;
  }

  compute(text) {
    // Tokenize
    const tokens = this.tokenize(text);

    // Initialize vector
    const v = new Array(this.hashBits).fill(0);

    // For each token, add/subtract based on hash bits
    for (const token of tokens) {
      const hash = this.hashToken(token);

      for (let i = 0; i < this.hashBits; i++) {
        if ((hash >> BigInt(i)) & 1n) {
          v[i] += 1;
        } else {
          v[i] -= 1;
        }
      }
    }

    // Convert to fingerprint
    let fingerprint = 0n;
    for (let i = 0; i < this.hashBits; i++) {
      if (v[i] > 0) {
        fingerprint |= (1n << BigInt(i));
      }
    }

    return fingerprint;
  }

  tokenize(text) {
    // Create shingles (n-grams)
    const words = text.toLowerCase().split(/\s+/).filter(w => w.length > 2);
    const shingles = [];

    for (let i = 0; i <= words.length - 3; i++) {
      shingles.push(words.slice(i, i + 3).join(' '));
    }

    return shingles;
  }

  hashToken(token) {
    // FNV-1a hash
    let hash = 0xcbf29ce484222325n;
    for (const char of token) {
      hash ^= BigInt(char.charCodeAt(0));
      hash *= 0x100000001b3n;
      hash &= 0xffffffffffffffffn;  // Keep 64 bits
    }
    return hash;
  }

  hammingDistance(hash1, hash2) {
    const xor = hash1 ^ hash2;
    let distance = 0;

    for (let i = 0n; i < BigInt(this.hashBits); i++) {
      if ((xor >> i) & 1n) {
        distance++;
      }
    }

    return distance;
  }

  areSimilar(hash1, hash2, threshold = 3) {
    return this.hammingDistance(hash1, hash2) <= threshold;
  }
}

class DuplicateDetector {
  constructor(redis) {
    this.redis = redis;
    this.simhash = new SimHash();
  }

  async isDuplicate(url, content) {
    // Exact URL duplicate
    const exactMatch = await this.redis.sismember('crawled_urls', url);
    if (exactMatch) {
      return { isDuplicate: true, type: 'exact_url' };
    }

    // Content hash for exact content match
    const contentHash = this.hashContent(content);
    const contentMatch = await this.redis.sismember('content_hashes', contentHash);
    if (contentMatch) {
      return { isDuplicate: true, type: 'exact_content' };
    }

    // Simhash for near-duplicate
    const simhash = this.simhash.compute(content);
    const nearDuplicate = await this.findNearDuplicate(simhash);
    if (nearDuplicate) {
      return { isDuplicate: true, type: 'near_duplicate', similarTo: nearDuplicate };
    }

    // Store hashes
    await this.redis.sadd('crawled_urls', url);
    await this.redis.sadd('content_hashes', contentHash);
    await this.storeSimhash(url, simhash);

    return { isDuplicate: false };
  }

  async findNearDuplicate(simhash) {
    // Use simhash index for efficient lookup
    // Group by first k bits for candidate selection
    const prefix = (simhash >> 48n).toString(16);
    const candidates = await this.redis.smembers(`simhash_index:${prefix}`);

    for (const candidate of candidates) {
      const [url, hash] = candidate.split('|');
      if (this.simhash.areSimilar(simhash, BigInt(hash))) {
        return url;
      }
    }

    return null;
  }

  hashContent(content) {
    const crypto = require('crypto');
    return crypto.createHash('sha256').update(content).digest('hex');
  }
}
```

---

## Distributed Crawler

### Worker Coordination

```javascript
class CrawlerCoordinator {
  constructor(kafka, redis) {
    this.kafka = kafka;
    this.redis = redis;
    this.workerId = process.env.WORKER_ID;
  }

  async start() {
    // Register worker
    await this.registerWorker();

    // Start heartbeat
    this.startHeartbeat();

    // Start consuming URLs
    await this.consumeURLs();
  }

  async registerWorker() {
    await this.redis.hset('crawler_workers', this.workerId, JSON.stringify({
      startedAt: Date.now(),
      status: 'active'
    }));
  }

  startHeartbeat() {
    setInterval(async () => {
      await this.redis.hset('crawler_workers', this.workerId, JSON.stringify({
        lastHeartbeat: Date.now(),
        status: 'active',
        stats: this.getStats()
      }));
    }, 10000);
  }

  async consumeURLs() {
    const consumer = this.kafka.consumer({ groupId: 'crawler-workers' });
    await consumer.connect();
    await consumer.subscribe({ topic: 'urls-to-crawl' });

    await consumer.run({
      eachMessage: async ({ message }) => {
        const url = message.value.toString();
        await this.crawlURL(url);
      }
    });
  }

  async crawlURL(url) {
    const startTime = Date.now();

    try {
      // Fetch
      const response = await this.fetcher.fetch(url);

      if (response.status !== 'success') {
        await this.handleFailure(url, response);
        return;
      }

      // Process content
      const content = await response.body.text();
      const processed = this.contentProcessor.process(url, content, response.contentType);

      // Check for duplicates
      const dupCheck = await this.duplicateDetector.isDuplicate(url, processed.text);
      if (dupCheck.isDuplicate) {
        this.stats.duplicates++;
        return;
      }

      // Store content
      await this.storeContent(url, content, processed);

      // Queue new URLs
      for (const link of processed.links) {
        await this.kafka.producer.send({
          topic: 'discovered-urls',
          messages: [{ value: link }]
        });
      }

      this.stats.success++;
      this.stats.totalTime += Date.now() - startTime;

    } catch (error) {
      this.stats.errors++;
      await this.handleError(url, error);
    }
  }

  async storeContent(url, rawContent, processed) {
    // Store to S3/HDFS
    const key = this.generateStorageKey(url);

    await this.storage.put(key, {
      url,
      crawledAt: new Date().toISOString(),
      content: rawContent,
      title: processed.title,
      text: processed.text,
      metadata: processed.metadata,
      links: processed.links
    });

    // Update index
    await this.redis.hset('url_index', url, key);
  }
}
```

### URL Partitioning

```javascript
class URLPartitioner {
  constructor(numPartitions) {
    this.numPartitions = numPartitions;
  }

  // Partition by domain to ensure politeness
  getPartition(url) {
    const domain = new URL(url).hostname;
    return this.hashDomain(domain) % this.numPartitions;
  }

  hashDomain(domain) {
    let hash = 0;
    for (let i = 0; i < domain.length; i++) {
      hash = ((hash << 5) - hash + domain.charCodeAt(i)) | 0;
    }
    return Math.abs(hash);
  }
}

// Kafka producer with partitioning
async function queueURL(url, priority = 1) {
  const partition = partitioner.getPartition(url);

  await kafka.producer.send({
    topic: 'urls-to-crawl',
    messages: [{
      key: new URL(url).hostname,
      value: JSON.stringify({ url, priority }),
      partition
    }]
  });
}
```

---

## Recrawling Strategy

```javascript
class RecrawlScheduler {
  constructor() {
    this.minInterval = 3600000;      // 1 hour minimum
    this.maxInterval = 86400000 * 30; // 30 days maximum
  }

  calculateNextCrawl(url, history) {
    if (!history || history.length < 2) {
      return Date.now() + this.minInterval;
    }

    // Calculate change frequency
    let changes = 0;
    for (let i = 1; i < history.length; i++) {
      if (history[i].contentHash !== history[i - 1].contentHash) {
        changes++;
      }
    }

    const changeRate = changes / (history.length - 1);

    // Higher change rate = more frequent crawls
    let interval;
    if (changeRate > 0.8) {
      interval = this.minInterval;  // Very dynamic
    } else if (changeRate > 0.5) {
      interval = 86400000;  // Daily
    } else if (changeRate > 0.2) {
      interval = 86400000 * 7;  // Weekly
    } else {
      interval = this.maxInterval;  // Monthly
    }

    // Consider page importance
    const importance = this.getPageImportance(url);
    interval = interval / importance;

    return Date.now() + Math.max(interval, this.minInterval);
  }

  getPageImportance(url) {
    // Based on domain rank, backlinks, etc.
    return 1.0;
  }

  async scheduleRecrawls() {
    const urls = await db.query(`
      SELECT url, next_crawl_at
      FROM crawl_schedule
      WHERE next_crawl_at <= NOW()
      ORDER BY next_crawl_at
      LIMIT 10000
    `);

    for (const { url } of urls) {
      await urlFrontier.addURL(url, 0.5);
    }
  }
}
```

---

## Database Schema

```sql
-- Crawled pages
CREATE TABLE pages (
  id UUID PRIMARY KEY,
  url TEXT UNIQUE NOT NULL,
  domain TEXT NOT NULL,
  content_hash TEXT,
  simhash BIGINT,
  title TEXT,
  crawled_at TIMESTAMP,
  status_code INT,
  content_type TEXT,
  content_length INT,
  storage_key TEXT
);

CREATE INDEX idx_pages_domain ON pages(domain);
CREATE INDEX idx_pages_crawled ON pages(crawled_at);
CREATE INDEX idx_pages_simhash ON pages(simhash);

-- Links between pages
CREATE TABLE links (
  source_url TEXT NOT NULL,
  target_url TEXT NOT NULL,
  anchor_text TEXT,
  discovered_at TIMESTAMP,
  PRIMARY KEY (source_url, target_url)
);

CREATE INDEX idx_links_target ON links(target_url);

-- Crawl schedule
CREATE TABLE crawl_schedule (
  url TEXT PRIMARY KEY,
  next_crawl_at TIMESTAMP NOT NULL,
  last_crawled_at TIMESTAMP,
  crawl_count INT DEFAULT 0,
  avg_change_interval INTERVAL
);

CREATE INDEX idx_schedule_next ON crawl_schedule(next_crawl_at);
```

---

## Interview Discussion Points

### How to handle crawler traps?

1. **URL length limit** - Reject URLs over 2000 chars
2. **Depth limit** - Max 15 levels from seed
3. **Pattern detection** - Detect calendar, session IDs
4. **Domain limits** - Max pages per domain
5. **Duplicate content detection** - Simhash comparison

### How to prioritize URLs?

1. **PageRank** - Link-based importance
2. **Domain authority** - Historical quality
3. **Freshness** - How stale is the page
4. **Content type** - HTML > images
5. **Depth** - Shallower = higher priority

### How to ensure politeness?

1. **robots.txt** - Respect directives
2. **Crawl-delay** - Honor specified delays
3. **Per-domain rate limiting** - 1 request/second
4. **Time-based scheduling** - Crawl at low-traffic times
5. **Backoff on errors** - Slow down on 429/503

### How to scale to billions of pages?

1. **Horizontal scaling** - Add more workers
2. **Partitioning** - Shard by domain
3. **Distributed storage** - S3/HDFS
4. **Efficient dedup** - Bloom filters, simhash
5. **Incremental crawling** - Only recrawl changed pages
