# Distributed Locking & Coordination

## Overview

```
Why Distributed Locks?
- Prevent race conditions across services
- Ensure mutual exclusion for shared resources
- Coordinate leader election
- Implement distributed transactions

Challenges:
- Network partitions
- Process failures (holding lock)
- Clock drift
- Split-brain scenarios
```

---

## Lock Requirements

### Properties of a Good Distributed Lock

```
1. Mutual Exclusion (Safety)
   - At most one client holds the lock at any time

2. Deadlock Freedom (Liveness)
   - Lock is eventually released (TTL/fencing)

3. Fault Tolerance
   - Lock service survives node failures

4. Low Latency
   - Lock acquisition should be fast

5. Fairness (optional)
   - Requests served in order
```

---

## Redis-Based Locking

### Simple Lock (Single Redis)

```javascript
class SimpleRedisLock {
  constructor(redis) {
    this.redis = redis;
  }

  async acquire(key, ttlMs = 10000) {
    const token = crypto.randomUUID();
    const acquired = await this.redis.set(
      key,
      token,
      'NX',           // Only if not exists
      'PX', ttlMs     // Expire after TTL
    );

    if (acquired === 'OK') {
      return { key, token, ttl: ttlMs };
    }
    return null;
  }

  async release(lock) {
    // Only release if we own the lock (atomic with Lua)
    const script = `
      if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
      else
        return 0
      end
    `;
    return await this.redis.eval(script, 1, lock.key, lock.token);
  }

  async extend(lock, additionalMs) {
    const script = `
      if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("PEXPIRE", KEYS[1], ARGV[2])
      else
        return 0
      end
    `;
    return await this.redis.eval(script, 1, lock.key, lock.token, additionalMs);
  }
}
```

### Redlock Algorithm (Multi-Node)

```javascript
// Redlock: Distributed lock across N Redis instances
// Requires majority (N/2 + 1) to acquire lock
class Redlock {
  constructor(redisClients, options = {}) {
    this.clients = redisClients;
    this.quorum = Math.floor(this.clients.length / 2) + 1;
    this.retryCount = options.retryCount || 3;
    this.retryDelay = options.retryDelay || 200;
    this.driftFactor = options.driftFactor || 0.01;
  }

  async acquire(resource, ttlMs) {
    const token = crypto.randomUUID();

    for (let attempt = 0; attempt < this.retryCount; attempt++) {
      const startTime = Date.now();
      let successCount = 0;

      // Try to acquire on all instances
      const results = await Promise.allSettled(
        this.clients.map(client =>
          this.tryAcquire(client, resource, token, ttlMs)
        )
      );

      successCount = results.filter(r => r.status === 'fulfilled' && r.value).length;

      // Calculate time elapsed
      const elapsed = Date.now() - startTime;
      const drift = Math.round(ttlMs * this.driftFactor) + 2;
      const validityTime = ttlMs - elapsed - drift;

      // Check if we got quorum and have valid time remaining
      if (successCount >= this.quorum && validityTime > 0) {
        return {
          resource,
          token,
          validUntil: Date.now() + validityTime,
          attempts: attempt + 1
        };
      }

      // Failed - release any locks we acquired
      await this.releaseAll(resource, token);

      // Wait before retry
      await this.delay(this.retryDelay + Math.random() * this.retryDelay);
    }

    throw new Error('Failed to acquire lock after retries');
  }

  async tryAcquire(client, resource, token, ttlMs) {
    try {
      const result = await client.set(resource, token, 'NX', 'PX', ttlMs);
      return result === 'OK';
    } catch {
      return false;
    }
  }

  async release(lock) {
    await this.releaseAll(lock.resource, lock.token);
  }

  async releaseAll(resource, token) {
    const script = `
      if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
      else
        return 0
      end
    `;

    await Promise.allSettled(
      this.clients.map(client =>
        client.eval(script, 1, resource, token)
      )
    );
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
const redlock = new Redlock([redis1, redis2, redis3, redis4, redis5]);

async function criticalSection() {
  const lock = await redlock.acquire('my-resource', 10000);
  try {
    // Do critical work
    await processExclusively();
  } finally {
    await redlock.release(lock);
  }
}
```

---

## ZooKeeper-Based Locking

### Ephemeral Sequential Nodes

```javascript
class ZooKeeperLock {
  constructor(zk, lockPath) {
    this.zk = zk;
    this.lockPath = lockPath;
    this.myNode = null;
  }

  async acquire() {
    // Create ephemeral sequential node
    this.myNode = await this.zk.create(
      `${this.lockPath}/lock-`,
      null,
      zk.CreateMode.EPHEMERAL_SEQUENTIAL
    );

    while (true) {
      // Get all children
      const children = await this.zk.getChildren(this.lockPath);
      children.sort();

      const mySequence = this.myNode.split('-').pop();
      const myIndex = children.findIndex(c => c.endsWith(mySequence));

      // Am I the lowest? I have the lock!
      if (myIndex === 0) {
        return { path: this.myNode };
      }

      // Watch the node just before me
      const watchNode = `${this.lockPath}/${children[myIndex - 1]}`;

      await new Promise((resolve) => {
        this.zk.exists(watchNode, (event) => {
          // Node deleted - try again
          resolve();
        });
      });
    }
  }

  async release() {
    if (this.myNode) {
      await this.zk.delete(this.myNode);
      this.myNode = null;
    }
  }
}

// ZooKeeper automatically releases lock if client disconnects
// (ephemeral node is deleted)
```

### Read-Write Lock

```javascript
class ZooKeeperRWLock {
  constructor(zk, lockPath) {
    this.zk = zk;
    this.lockPath = lockPath;
  }

  async acquireRead() {
    const node = await this.zk.create(
      `${this.lockPath}/read-`,
      null,
      zk.CreateMode.EPHEMERAL_SEQUENTIAL
    );

    while (true) {
      const children = await this.zk.getChildren(this.lockPath);
      const writes = children.filter(c => c.startsWith('write-')).sort();

      // No writes before me? I have read lock!
      const mySeq = node.split('-').pop();
      const blockedBy = writes.find(w => w.split('-').pop() < mySeq);

      if (!blockedBy) {
        return { path: node, type: 'read' };
      }

      // Wait for blocking write to release
      await this.watchNode(`${this.lockPath}/${blockedBy}`);
    }
  }

  async acquireWrite() {
    const node = await this.zk.create(
      `${this.lockPath}/write-`,
      null,
      zk.CreateMode.EPHEMERAL_SEQUENTIAL
    );

    while (true) {
      const children = await this.zk.getChildren(this.lockPath);
      children.sort((a, b) => a.split('-').pop() - b.split('-').pop());

      const mySeq = node.split('-').pop();

      // Am I first? I have write lock!
      if (children[0].split('-').pop() === mySeq) {
        return { path: node, type: 'write' };
      }

      // Wait for node before me
      const myIndex = children.findIndex(c => c.split('-').pop() === mySeq);
      await this.watchNode(`${this.lockPath}/${children[myIndex - 1]}`);
    }
  }
}
```

---

## etcd-Based Locking

```javascript
const { Etcd3 } = require('etcd3');

class EtcdLock {
  constructor(client) {
    this.client = client;
  }

  async acquire(key, ttlSeconds = 10) {
    // Create lease
    const lease = this.client.lease(ttlSeconds);

    // Try to put with lease (like SET NX)
    const lockKey = `/locks/${key}`;

    try {
      await this.client.if(lockKey, 'Create', '==', 0)
        .then(this.client.put(lockKey).value('locked').lease(lease.grant))
        .commit();

      // Start keep-alive
      lease.on('lost', () => {
        console.log('Lock lease lost!');
      });

      return {
        key: lockKey,
        lease,
        release: async () => {
          await lease.revoke();
        }
      };
    } catch (e) {
      await lease.revoke();
      throw new Error('Failed to acquire lock');
    }
  }

  // Watch for lock release
  async waitForLock(key, timeoutMs = 30000) {
    const lockKey = `/locks/${key}`;

    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        watcher.cancel();
        reject(new Error('Timeout waiting for lock'));
      }, timeoutMs);

      const watcher = this.client.watch()
        .key(lockKey)
        .create();

      watcher.on('delete', () => {
        clearTimeout(timeout);
        watcher.cancel();
        resolve();
      });
    });
  }
}
```

---

## Fencing Tokens

### Problem: Lock Released But Client Still Operating

```
Timeline:
1. Client A acquires lock (token=1)
2. Client A pauses (GC, network delay)
3. Lock expires (TTL)
4. Client B acquires lock (token=2)
5. Client A resumes, thinks it has lock
6. Both clients write to shared resource! ❌
```

### Solution: Fencing Tokens

```javascript
class FencedLock {
  constructor(redis) {
    this.redis = redis;
    this.tokenKey = 'lock:fencing:token';
  }

  async acquire(resource, ttlMs) {
    // Get monotonically increasing fencing token
    const fencingToken = await this.redis.incr(this.tokenKey);
    const lockValue = JSON.stringify({ fencingToken, clientId: this.clientId });

    const acquired = await this.redis.set(
      resource,
      lockValue,
      'NX',
      'PX', ttlMs
    );

    if (acquired) {
      return { resource, fencingToken, ttl: ttlMs };
    }
    return null;
  }
}

// Storage service validates fencing token
class FencedStorage {
  constructor() {
    this.lastToken = new Map();  // resource -> last seen token
  }

  write(resource, data, fencingToken) {
    const lastToken = this.lastToken.get(resource) || 0;

    // Reject if token is stale
    if (fencingToken <= lastToken) {
      throw new Error('Stale fencing token - operation rejected');
    }

    // Accept write and update token
    this.lastToken.set(resource, fencingToken);
    this.doWrite(resource, data);
  }
}
```

---

## Leader Election

### Using ZooKeeper

```javascript
class LeaderElection {
  constructor(zk, electionPath) {
    this.zk = zk;
    this.electionPath = electionPath;
    this.myNode = null;
    this.isLeader = false;
    this.onBecomeLeader = null;
    this.onLoseLeadership = null;
  }

  async start() {
    // Create ephemeral sequential node
    this.myNode = await this.zk.create(
      `${this.electionPath}/candidate-`,
      null,
      zk.CreateMode.EPHEMERAL_SEQUENTIAL
    );

    await this.checkLeadership();
  }

  async checkLeadership() {
    const children = await this.zk.getChildren(this.electionPath);
    children.sort();

    const mySequence = this.myNode.split('/').pop();
    const isSmallest = children[0] === mySequence;

    if (isSmallest && !this.isLeader) {
      this.isLeader = true;
      if (this.onBecomeLeader) {
        this.onBecomeLeader();
      }
    } else if (!isSmallest) {
      this.isLeader = false;

      // Watch the node before me
      const myIndex = children.indexOf(mySequence);
      const watchNode = `${this.electionPath}/${children[myIndex - 1]}`;

      this.zk.exists(watchNode, async () => {
        await this.checkLeadership();
      });
    }
  }

  async resign() {
    if (this.myNode) {
      await this.zk.delete(this.myNode);
      this.myNode = null;
      this.isLeader = false;
      if (this.onLoseLeadership) {
        this.onLoseLeadership();
      }
    }
  }
}

// Usage
const election = new LeaderElection(zk, '/services/myservice/leader');

election.onBecomeLeader = () => {
  console.log('I am now the leader!');
  startLeaderTasks();
};

election.onLoseLeadership = () => {
  console.log('No longer the leader');
  stopLeaderTasks();
};

await election.start();
```

### Using etcd

```javascript
class EtcdLeaderElection {
  constructor(client, electionName) {
    this.client = client;
    this.election = this.client.election(electionName);
    this.isLeader = false;
  }

  async campaign(value) {
    // Campaign for leadership
    const lease = this.client.lease(10);  // 10 second TTL

    await this.election.campaign(value);
    this.isLeader = true;

    // Keep renewing lease
    lease.on('lost', () => {
      this.isLeader = false;
      this.onLoseLeadership?.();
    });

    this.onBecomeLeader?.();
    return lease;
  }

  async resign() {
    await this.election.resign();
    this.isLeader = false;
  }

  async getLeader() {
    return await this.election.leader();
  }

  async observe(callback) {
    const watcher = await this.election.observe();
    watcher.on('change', (leader) => {
      callback(leader);
    });
  }
}
```

---

## Distributed Semaphore

```javascript
class DistributedSemaphore {
  constructor(redis, name, maxConcurrent) {
    this.redis = redis;
    this.name = name;
    this.maxConcurrent = maxConcurrent;
  }

  async acquire(timeout = 10000) {
    const token = crypto.randomUUID();
    const now = Date.now();
    const deadline = now + timeout;

    while (Date.now() < deadline) {
      // Clean up expired entries
      await this.redis.zremrangebyscore(this.name, 0, now - 30000);

      // Try to add ourselves
      const added = await this.redis.zadd(
        this.name,
        'NX',
        now,
        token
      );

      if (added) {
        // Check if we're within limit
        const rank = await this.redis.zrank(this.name, token);

        if (rank < this.maxConcurrent) {
          return { token, acquired: true };
        }

        // Over limit - remove and wait
        await this.redis.zrem(this.name, token);
      }

      // Wait and retry
      await this.delay(100);
    }

    return { acquired: false };
  }

  async release(token) {
    await this.redis.zrem(this.name, token);
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage: Limit to 10 concurrent API calls
const semaphore = new DistributedSemaphore(redis, 'api-limit', 10);

async function makeAPICall() {
  const permit = await semaphore.acquire();
  if (!permit.acquired) {
    throw new Error('Could not acquire permit');
  }

  try {
    return await callExternalAPI();
  } finally {
    await semaphore.release(permit.token);
  }
}
```

---

## Comparison

| Feature | Redis | ZooKeeper | etcd |
|---------|-------|-----------|------|
| Consistency | AP (single) / CP (cluster) | CP | CP |
| Automatic release | TTL only | Session (ephemeral) | Lease |
| Watch support | Pub/Sub | Native watches | Native watches |
| Performance | Very fast | Moderate | Fast |
| Complexity | Low | High | Medium |
| Use case | Caching + locks | Coordination | K8s, config |

---

## Interview Discussion Points

### How to handle lock holder crash?

1. **TTL/Lease** - Lock auto-expires
2. **Heartbeat** - Renew periodically, expire on miss
3. **Ephemeral nodes** - ZK deletes on session end
4. **Fencing tokens** - Reject stale operations

### How to prevent split-brain?

1. **Quorum** - Require majority agreement
2. **Fencing tokens** - Monotonic, validated by storage
3. **Consensus protocol** - Raft/Paxos (ZK, etcd)
4. **Network partition detection** - Self-fence on isolation

### When to use distributed locks vs other patterns?

| Pattern | Use When |
|---------|----------|
| Distributed lock | Mutual exclusion for shared resource |
| Optimistic locking | Low contention, can retry |
| Queue | Serialize processing |
| Idempotency | Accept duplicates safely |
| Saga | Distributed transactions |

### How to debug lock issues?

1. **Logging** - Log acquire/release with timestamps
2. **Metrics** - Lock wait time, hold time, failures
3. **Tracing** - Distributed traces across services
4. **Lock visualization** - Dashboard showing held locks
5. **Deadlock detection** - Monitor for circular waits
