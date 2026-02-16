# Design: Distributed Cache

## Requirements

### Functional Requirements
- Get/Set/Delete operations
- TTL support
- Eviction policies (LRU, LFU)
- Atomic operations (INCR, DECR)
- Data structures (strings, lists, sets, hashes)

### Non-Functional Requirements
- Sub-millisecond latency
- High availability (99.99%)
- Scale: 1M+ operations/second
- Memory-efficient
- Horizontal scalability

---

## High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                         Clients                                    │
└────────────────────────────────┬──────────────────────────────────┘
                                 │
┌────────────────────────────────▼──────────────────────────────────┐
│                     Cache Client Library                           │
│              (Connection pooling, routing, failover)               │
└────────────────────────────────┬──────────────────────────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
            ▼                    ▼                    ▼
    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
    │  Cache Node   │    │  Cache Node   │    │  Cache Node   │
    │    (Shard 1)  │    │    (Shard 2)  │    │    (Shard 3)  │
    └───────┬───────┘    └───────┬───────┘    └───────┬───────┘
            │                    │                    │
            ▼                    ▼                    ▼
    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
    │   Replica 1   │    │   Replica 2   │    │   Replica 3   │
    └───────────────┘    └───────────────┘    └───────────────┘
```

---

## Data Distribution

### Consistent Hashing

```javascript
class ConsistentHash {
  constructor(virtualNodes = 150) {
    this.ring = new Map();      // position -> node
    this.sortedPositions = [];  // sorted positions for binary search
    this.virtualNodes = virtualNodes;
  }

  hash(key) {
    // Use MD5 or MurmurHash for better distribution
    let hash = 0;
    for (let i = 0; i < key.length; i++) {
      hash = ((hash << 5) - hash + key.charCodeAt(i)) | 0;
    }
    return Math.abs(hash) % (2 ** 32);
  }

  addNode(node) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const virtualKey = `${node.id}:${i}`;
      const position = this.hash(virtualKey);
      this.ring.set(position, node);
      this.sortedPositions.push(position);
    }
    this.sortedPositions.sort((a, b) => a - b);
  }

  removeNode(node) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const virtualKey = `${node.id}:${i}`;
      const position = this.hash(virtualKey);
      this.ring.delete(position);
      const idx = this.sortedPositions.indexOf(position);
      if (idx !== -1) this.sortedPositions.splice(idx, 1);
    }
  }

  getNode(key) {
    if (this.sortedPositions.length === 0) return null;

    const keyHash = this.hash(key);

    // Binary search for first position >= keyHash
    let left = 0, right = this.sortedPositions.length;
    while (left < right) {
      const mid = (left + right) >> 1;
      if (this.sortedPositions[mid] < keyHash) {
        left = mid + 1;
      } else {
        right = mid;
      }
    }

    // Wrap around
    const position = this.sortedPositions[left % this.sortedPositions.length];
    return this.ring.get(position);
  }

  // Get N replicas for a key
  getNodes(key, count) {
    const nodes = new Set();
    const keyHash = this.hash(key);

    let idx = this.findStartIndex(keyHash);

    while (nodes.size < count && nodes.size < this.ring.size) {
      const position = this.sortedPositions[idx % this.sortedPositions.length];
      const node = this.ring.get(position);
      nodes.add(node);
      idx++;
    }

    return Array.from(nodes);
  }
}
```

---

## Cache Node Implementation

### In-Memory Store

```javascript
class CacheStore {
  constructor(maxMemoryBytes) {
    this.data = new Map();
    this.expirations = new Map();  // key -> expireAt
    this.lruList = new DoublyLinkedList();
    this.keyToNode = new Map();
    this.maxMemory = maxMemoryBytes;
    this.usedMemory = 0;
  }

  get(key) {
    // Check expiration
    if (this.isExpired(key)) {
      this.delete(key);
      return null;
    }

    const value = this.data.get(key);
    if (value !== undefined) {
      // Move to front (most recently used)
      this.updateLRU(key);
    }
    return value;
  }

  set(key, value, ttlSeconds = null) {
    const size = this.estimateSize(key, value);

    // Evict if necessary
    while (this.usedMemory + size > this.maxMemory && this.data.size > 0) {
      this.evictLRU();
    }

    // Delete old value if exists
    if (this.data.has(key)) {
      this.delete(key);
    }

    // Store new value
    this.data.set(key, value);
    this.usedMemory += size;

    // Add to LRU list
    const node = this.lruList.addToFront(key);
    this.keyToNode.set(key, node);

    // Set expiration
    if (ttlSeconds) {
      this.expirations.set(key, Date.now() + ttlSeconds * 1000);
    }

    return true;
  }

  delete(key) {
    if (!this.data.has(key)) return false;

    const value = this.data.get(key);
    this.usedMemory -= this.estimateSize(key, value);

    this.data.delete(key);
    this.expirations.delete(key);

    // Remove from LRU list
    const node = this.keyToNode.get(key);
    if (node) {
      this.lruList.remove(node);
      this.keyToNode.delete(key);
    }

    return true;
  }

  isExpired(key) {
    const expireAt = this.expirations.get(key);
    return expireAt && Date.now() > expireAt;
  }

  evictLRU() {
    // Remove least recently used (tail of list)
    const node = this.lruList.removeTail();
    if (node) {
      this.delete(node.key);
    }
  }

  updateLRU(key) {
    const node = this.keyToNode.get(key);
    if (node) {
      this.lruList.moveToFront(node);
    }
  }

  estimateSize(key, value) {
    // Rough estimation in bytes
    return key.length * 2 + JSON.stringify(value).length * 2;
  }
}
```

### LFU Eviction

```javascript
class LFUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map();        // key -> value
    this.frequencies = new Map();  // key -> frequency
    this.freqBuckets = new Map();  // frequency -> Set of keys
    this.minFreq = 0;
  }

  get(key) {
    if (!this.cache.has(key)) return null;

    // Increase frequency
    this.increaseFreq(key);

    return this.cache.get(key);
  }

  set(key, value) {
    if (this.capacity === 0) return;

    if (this.cache.has(key)) {
      this.cache.set(key, value);
      this.increaseFreq(key);
      return;
    }

    // Evict if at capacity
    if (this.cache.size >= this.capacity) {
      this.evictLFU();
    }

    // Add new key with frequency 1
    this.cache.set(key, value);
    this.frequencies.set(key, 1);

    if (!this.freqBuckets.has(1)) {
      this.freqBuckets.set(1, new Set());
    }
    this.freqBuckets.get(1).add(key);

    this.minFreq = 1;
  }

  increaseFreq(key) {
    const freq = this.frequencies.get(key);
    this.frequencies.set(key, freq + 1);

    // Remove from current frequency bucket
    this.freqBuckets.get(freq).delete(key);
    if (this.freqBuckets.get(freq).size === 0) {
      this.freqBuckets.delete(freq);
      if (this.minFreq === freq) this.minFreq++;
    }

    // Add to new frequency bucket
    if (!this.freqBuckets.has(freq + 1)) {
      this.freqBuckets.set(freq + 1, new Set());
    }
    this.freqBuckets.get(freq + 1).add(key);
  }

  evictLFU() {
    const bucket = this.freqBuckets.get(this.minFreq);
    const keyToEvict = bucket.values().next().value;

    bucket.delete(keyToEvict);
    if (bucket.size === 0) {
      this.freqBuckets.delete(this.minFreq);
    }

    this.cache.delete(keyToEvict);
    this.frequencies.delete(keyToEvict);
  }
}
```

---

## Client Library

```javascript
class CacheClient {
  constructor(nodes) {
    this.hashRing = new ConsistentHash();
    this.connections = new Map();

    for (const node of nodes) {
      this.hashRing.addNode(node);
      this.connections.set(node.id, this.createConnection(node));
    }
  }

  async get(key) {
    const node = this.hashRing.getNode(key);
    const conn = this.connections.get(node.id);

    try {
      return await conn.get(key);
    } catch (error) {
      // Failover to replica
      return this.getFromReplica(key, node);
    }
  }

  async set(key, value, ttl = null) {
    const nodes = this.hashRing.getNodes(key, 2);  // Primary + 1 replica

    // Write to primary
    const primary = this.connections.get(nodes[0].id);
    await primary.set(key, value, ttl);

    // Async write to replica
    if (nodes.length > 1) {
      const replica = this.connections.get(nodes[1].id);
      replica.set(key, value, ttl).catch(console.error);
    }
  }

  async delete(key) {
    const nodes = this.hashRing.getNodes(key, 2);

    await Promise.all(
      nodes.map(node =>
        this.connections.get(node.id).delete(key).catch(() => {})
      )
    );
  }

  // Batch operations for efficiency
  async mget(keys) {
    // Group keys by node
    const nodeKeys = new Map();
    for (const key of keys) {
      const node = this.hashRing.getNode(key);
      if (!nodeKeys.has(node.id)) {
        nodeKeys.set(node.id, []);
      }
      nodeKeys.get(node.id).push(key);
    }

    // Parallel fetch from each node
    const results = await Promise.all(
      Array.from(nodeKeys.entries()).map(async ([nodeId, keys]) => {
        const conn = this.connections.get(nodeId);
        return conn.mget(keys);
      })
    );

    // Merge results
    const merged = new Map();
    for (const result of results.flat()) {
      merged.set(result.key, result.value);
    }

    return keys.map(key => merged.get(key));
  }
}
```

---

## Replication

### Primary-Replica

```javascript
class CacheNode {
  constructor(isPrimary = true) {
    this.store = new CacheStore(1024 * 1024 * 1024);  // 1GB
    this.isPrimary = isPrimary;
    this.replicas = [];
    this.replicationLog = [];
  }

  async set(key, value, ttl) {
    // Write to local store
    this.store.set(key, value, ttl);

    if (this.isPrimary) {
      // Log for replication
      const entry = { op: 'SET', key, value, ttl, timestamp: Date.now() };
      this.replicationLog.push(entry);

      // Async replicate to secondaries
      this.replicateAsync(entry);
    }
  }

  async replicateAsync(entry) {
    for (const replica of this.replicas) {
      replica.applyReplication(entry).catch(error => {
        // Queue for retry
        this.replicationQueue.push({ replica, entry });
      });
    }
  }

  async applyReplication(entry) {
    switch (entry.op) {
      case 'SET':
        this.store.set(entry.key, entry.value, entry.ttl);
        break;
      case 'DELETE':
        this.store.delete(entry.key);
        break;
    }
  }
}
```

---

## Cache Patterns

### Cache-Aside

```javascript
async function getData(key) {
  // Check cache
  let data = await cache.get(key);
  if (data) return data;

  // Cache miss - load from database
  data = await database.get(key);

  // Store in cache
  if (data) {
    await cache.set(key, data, 3600);  // 1 hour TTL
  }

  return data;
}
```

### Write-Through

```javascript
async function setData(key, value) {
  // Write to database first
  await database.set(key, value);

  // Then update cache
  await cache.set(key, value);
}
```

### Write-Behind

```javascript
class WriteBackCache {
  constructor() {
    this.dirtyKeys = new Set();
    this.flushInterval = setInterval(() => this.flush(), 1000);
  }

  async set(key, value) {
    // Write to cache immediately
    await cache.set(key, value);
    this.dirtyKeys.add(key);
  }

  async flush() {
    const keysToFlush = Array.from(this.dirtyKeys);
    this.dirtyKeys.clear();

    for (const key of keysToFlush) {
      const value = await cache.get(key);
      await database.set(key, value);
    }
  }
}
```

---

## Monitoring

```javascript
class CacheMetrics {
  constructor() {
    this.hits = 0;
    this.misses = 0;
    this.evictions = 0;
    this.memoryUsed = 0;
  }

  recordHit() { this.hits++; }
  recordMiss() { this.misses++; }
  recordEviction() { this.evictions++; }

  getHitRate() {
    const total = this.hits + this.misses;
    return total > 0 ? this.hits / total : 0;
  }

  getStats() {
    return {
      hitRate: this.getHitRate(),
      hits: this.hits,
      misses: this.misses,
      evictions: this.evictions,
      memoryUsed: this.memoryUsed,
      keyCount: this.keyCount
    };
  }
}
```

---

## Interview Discussion Points

### How to handle cache stampede?

1. **Locking** - Single request populates cache
2. **Probabilistic expiration** - Random early refresh
3. **Background refresh** - Refresh before expiration
4. **Request coalescing** - Dedupe concurrent requests

### How to handle hot keys?

1. **Local caching** - Cache in application memory
2. **Key replication** - Replicate hot keys to multiple nodes
3. **Rate limiting** - Protect against abuse

### How to maintain consistency?

1. **TTL-based** - Eventually consistent with expiration
2. **Event-based invalidation** - Invalidate on database changes
3. **Write-through** - Always update cache on writes
