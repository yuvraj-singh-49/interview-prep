# Design: Unique ID Generator

## Requirements

### Functional Requirements
- Generate globally unique IDs
- IDs should be sortable by time (optional but common)
- Support high throughput generation
- Work in distributed environment

### Non-Functional Requirements
- Uniqueness guarantee (no collisions)
- High availability
- Low latency (<1ms per ID)
- Scale: 10K+ IDs/second per node
- No coordination required (ideal)

---

## ID Generation Strategies

### Comparison Matrix

```
┌─────────────────┬──────────┬─────────┬──────────┬────────────┬───────────┐
│ Strategy        │ Sortable │ Size    │ Throughput│ Coordination│ Collisions│
├─────────────────┼──────────┼─────────┼──────────┼────────────┼───────────┤
│ UUID v4         │ No       │ 128-bit │ Very High│ None       │ ~Zero     │
│ UUID v7         │ Yes      │ 128-bit │ Very High│ None       │ ~Zero     │
│ Snowflake       │ Yes      │ 64-bit  │ High     │ Clock sync │ None      │
│ Database Auto   │ Yes      │ 64-bit  │ Low      │ Central DB │ None      │
│ Redis INCR      │ Yes      │ 64-bit  │ Medium   │ Central    │ None      │
│ ULID            │ Yes      │ 128-bit │ Very High│ None       │ ~Zero     │
│ NanoID          │ No       │ Variable│ Very High│ None       │ ~Zero     │
└─────────────────┴──────────┴─────────┴──────────┴────────────┴───────────┘
```

---

## UUID (Universally Unique Identifier)

### UUID v4 (Random)

```javascript
const crypto = require('crypto');

function generateUUIDv4() {
  // 128 bits of random data
  const bytes = crypto.randomBytes(16);

  // Set version (4) and variant bits
  bytes[6] = (bytes[6] & 0x0f) | 0x40;  // Version 4
  bytes[8] = (bytes[8] & 0x3f) | 0x80;  // Variant

  // Format as string
  const hex = bytes.toString('hex');
  return [
    hex.slice(0, 8),
    hex.slice(8, 12),
    hex.slice(12, 16),
    hex.slice(16, 20),
    hex.slice(20, 32)
  ].join('-');
}

// Example: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
```

**Pros:**
- Simple, no coordination needed
- Statistically unique (122 random bits)

**Cons:**
- Not sortable
- 36 characters as string
- Poor index locality in databases

### UUID v7 (Time-ordered)

```javascript
function generateUUIDv7() {
  const timestamp = Date.now();

  // 48 bits: Unix timestamp in milliseconds
  const timestampBytes = Buffer.alloc(6);
  timestampBytes.writeUIntBE(timestamp, 0, 6);

  // 74 bits: Random
  const randomBytes = crypto.randomBytes(10);

  // Combine
  const bytes = Buffer.concat([timestampBytes, randomBytes]);

  // Set version (7) and variant
  bytes[6] = (bytes[6] & 0x0f) | 0x70;  // Version 7
  bytes[8] = (bytes[8] & 0x3f) | 0x80;  // Variant

  const hex = bytes.toString('hex');
  return [
    hex.slice(0, 8),
    hex.slice(8, 12),
    hex.slice(12, 16),
    hex.slice(16, 20),
    hex.slice(20, 32)
  ].join('-');
}

// Example: "018e5b3c-9a2b-7f12-8d4e-5b3c9a2b7f12"
// Note: Lexicographically sortable by time!
```

---

## Snowflake ID (Twitter)

### Structure

```
 64-bit ID:
┌─────────────────────────────────────────────────────────────────────┐
│ 0 │      41 bits timestamp      │ 10 bits │    12 bits sequence    │
│   │     (milliseconds)          │ machine │      (per ms)          │
│   │                             │   ID    │                        │
└─────────────────────────────────────────────────────────────────────┘
   1            41                    10              12

- Sign bit: Always 0 (positive number)
- Timestamp: Milliseconds since custom epoch (41 bits = 69 years)
- Machine ID: Datacenter (5 bits) + Worker (5 bits) = 1024 machines
- Sequence: Counter within same millisecond (4096 IDs/ms/machine)
```

### Implementation

```javascript
class SnowflakeGenerator {
  constructor(machineId, datacenterId = 0) {
    // Custom epoch: 2020-01-01 00:00:00 UTC
    this.epoch = 1577836800000n;

    // Bit allocations
    this.machineIdBits = 5n;
    this.datacenterIdBits = 5n;
    this.sequenceBits = 12n;

    // Max values
    this.maxMachineId = (1n << this.machineIdBits) - 1n;
    this.maxDatacenterId = (1n << this.datacenterIdBits) - 1n;
    this.maxSequence = (1n << this.sequenceBits) - 1n;

    // Bit shifts
    this.machineIdShift = this.sequenceBits;
    this.datacenterIdShift = this.sequenceBits + this.machineIdBits;
    this.timestampShift = this.sequenceBits + this.machineIdBits + this.datacenterIdBits;

    // Validate inputs
    if (BigInt(machineId) > this.maxMachineId || machineId < 0) {
      throw new Error(`Machine ID must be between 0 and ${this.maxMachineId}`);
    }
    if (BigInt(datacenterId) > this.maxDatacenterId || datacenterId < 0) {
      throw new Error(`Datacenter ID must be between 0 and ${this.maxDatacenterId}`);
    }

    this.machineId = BigInt(machineId);
    this.datacenterId = BigInt(datacenterId);
    this.sequence = 0n;
    this.lastTimestamp = -1n;
  }

  generate() {
    let timestamp = BigInt(Date.now());

    // Clock moved backwards - wait or throw
    if (timestamp < this.lastTimestamp) {
      throw new Error('Clock moved backwards. Refusing to generate ID.');
    }

    // Same millisecond - increment sequence
    if (timestamp === this.lastTimestamp) {
      this.sequence = (this.sequence + 1n) & this.maxSequence;

      // Sequence overflow - wait for next millisecond
      if (this.sequence === 0n) {
        timestamp = this.waitNextMillis(timestamp);
      }
    } else {
      // New millisecond - reset sequence
      this.sequence = 0n;
    }

    this.lastTimestamp = timestamp;

    // Compose ID
    const id =
      ((timestamp - this.epoch) << this.timestampShift) |
      (this.datacenterId << this.datacenterIdShift) |
      (this.machineId << this.machineIdShift) |
      this.sequence;

    return id.toString();
  }

  waitNextMillis(currentTimestamp) {
    let timestamp = BigInt(Date.now());
    while (timestamp <= currentTimestamp) {
      timestamp = BigInt(Date.now());
    }
    return timestamp;
  }

  // Parse ID back to components
  parse(id) {
    const idBigInt = BigInt(id);

    const timestamp = (idBigInt >> this.timestampShift) + this.epoch;
    const datacenterId = (idBigInt >> this.datacenterIdShift) & this.maxDatacenterId;
    const machineId = (idBigInt >> this.machineIdShift) & this.maxMachineId;
    const sequence = idBigInt & this.maxSequence;

    return {
      timestamp: new Date(Number(timestamp)),
      datacenterId: Number(datacenterId),
      machineId: Number(machineId),
      sequence: Number(sequence)
    };
  }
}

// Usage
const generator = new SnowflakeGenerator(1, 1);  // Machine 1, Datacenter 1
const id = generator.generate();
console.log(id);  // "1234567890123456789"
console.log(generator.parse(id));
// { timestamp: 2024-01-15T10:30:00.000Z, datacenterId: 1, machineId: 1, sequence: 0 }
```

### Throughput

```
Per machine: 4096 IDs/ms = 4,096,000 IDs/second
Total with 1024 machines: 4 billion IDs/second
```

---

## ULID (Universally Unique Lexicographically Sortable Identifier)

### Structure

```
 128 bits total:
┌───────────────────────────────────────────────────────────────────┐
│       48 bits timestamp        │        80 bits randomness        │
│     (milliseconds UTC)         │                                  │
└───────────────────────────────────────────────────────────────────┘

Encoded as 26 character string using Crockford's base32:
01ARZ3NDEKTSV4RRFFQ69G5FAV
└──────┘└────────────────────┘
timestamp    randomness
```

### Implementation

```javascript
class ULIDGenerator {
  static ENCODING = '0123456789ABCDEFGHJKMNPQRSTVWXYZ';
  static ENCODING_LEN = 32;
  static TIME_LEN = 10;
  static RANDOM_LEN = 16;

  constructor() {
    this.lastTime = 0;
    this.lastRandom = new Uint8Array(10);
  }

  generate() {
    const now = Date.now();

    if (now === this.lastTime) {
      // Same millisecond - increment random portion
      this.incrementRandom();
    } else {
      // New millisecond - generate new random
      this.lastTime = now;
      crypto.getRandomValues(this.lastRandom);
    }

    return this.encodeTime(now) + this.encodeRandom();
  }

  encodeTime(time) {
    let encoded = '';
    for (let i = ULIDGenerator.TIME_LEN - 1; i >= 0; i--) {
      const mod = time % ULIDGenerator.ENCODING_LEN;
      encoded = ULIDGenerator.ENCODING[mod] + encoded;
      time = Math.floor(time / ULIDGenerator.ENCODING_LEN);
    }
    return encoded;
  }

  encodeRandom() {
    let encoded = '';
    for (let i = 0; i < 10; i++) {
      const byte = this.lastRandom[i];
      encoded += ULIDGenerator.ENCODING[byte >> 3];
      if (i < 9) {
        encoded += ULIDGenerator.ENCODING[((byte & 0x07) << 2) | (this.lastRandom[i + 1] >> 6)];
      }
    }
    return encoded.slice(0, ULIDGenerator.RANDOM_LEN);
  }

  incrementRandom() {
    for (let i = 9; i >= 0; i--) {
      if (this.lastRandom[i] === 255) {
        this.lastRandom[i] = 0;
      } else {
        this.lastRandom[i]++;
        break;
      }
    }
  }

  // Decode ULID
  static decode(ulid) {
    const timeStr = ulid.slice(0, 10);
    let time = 0;
    for (const char of timeStr) {
      time = time * 32 + this.ENCODING.indexOf(char);
    }
    return {
      timestamp: new Date(time),
      randomness: ulid.slice(10)
    };
  }
}

// Usage
const ulid = new ULIDGenerator();
console.log(ulid.generate());  // "01HQ3CKJ1PMEQ6Q5NYQFSMHX00"
```

**Pros:**
- 128-bit, like UUID
- Time-sortable (lexicographically)
- URL-safe (no special characters)
- Monotonic within same millisecond

---

## Database Sequence (Centralized)

### PostgreSQL Sequences

```sql
-- Create sequence
CREATE SEQUENCE order_id_seq
  START WITH 1
  INCREMENT BY 1
  NO MAXVALUE
  CACHE 100;  -- Pre-allocate 100 values for performance

-- Use in table
CREATE TABLE orders (
  id BIGINT PRIMARY KEY DEFAULT nextval('order_id_seq'),
  ...
);

-- Or get directly
SELECT nextval('order_id_seq');
```

### Batch Allocation Service

```javascript
class SequenceService {
  constructor(db, batchSize = 1000) {
    this.db = db;
    this.batchSize = batchSize;
    this.currentBatch = null;
    this.nextId = 0;
    this.maxId = 0;
  }

  async getNextId() {
    if (this.nextId >= this.maxId) {
      await this.allocateBatch();
    }

    return this.nextId++;
  }

  async allocateBatch() {
    // Atomically allocate a batch
    const result = await this.db.query(`
      UPDATE id_allocations
      SET next_value = next_value + $1
      WHERE name = $2
      RETURNING next_value - $1 as start_value
    `, [this.batchSize, 'order_ids']);

    this.nextId = result.rows[0].start_value;
    this.maxId = this.nextId + this.batchSize;
  }
}
```

**Pros:**
- Simple
- Strictly sequential
- Works with existing databases

**Cons:**
- Single point of failure
- Limited throughput
- Network latency

---

## Redis-Based ID Generation

```javascript
class RedisIdGenerator {
  constructor(redis, keyPrefix = 'id') {
    this.redis = redis;
    this.keyPrefix = keyPrefix;
  }

  // Simple increment
  async getNextId(namespace = 'default') {
    const key = `${this.keyPrefix}:${namespace}`;
    return await this.redis.incr(key);
  }

  // Time-prefixed for sortability
  async getTimeBasedId(namespace = 'default') {
    const timestamp = Date.now();
    const key = `${this.keyPrefix}:${namespace}:${timestamp}`;

    const sequence = await this.redis.incr(key);
    await this.redis.expire(key, 2);  // Expire old keys

    // Combine: timestamp (13 digits) + sequence (padded)
    return `${timestamp}${sequence.toString().padStart(6, '0')}`;
  }

  // Batch allocation for high throughput
  async allocateBatch(namespace, size = 1000) {
    const key = `${this.keyPrefix}:${namespace}`;
    const end = await this.redis.incrby(key, size);
    const start = end - size + 1;

    return { start, end };
  }
}
```

---

## Distributed ID Service

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Load Balancer                             │
└─────────────────────────────┬───────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│  ID Generator   │   │  ID Generator   │   │  ID Generator   │
│  (Worker 1)     │   │  (Worker 2)     │   │  (Worker 3)     │
│  Machine ID: 1  │   │  Machine ID: 2  │   │  Machine ID: 3  │
└────────┬────────┘   └────────┬────────┘   └────────┬────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                      ┌────────▼────────┐
                      │   ZooKeeper     │
                      │ (Machine ID     │
                      │  coordination)  │
                      └─────────────────┘
```

### Machine ID Assignment with ZooKeeper

```javascript
class DistributedIdService {
  constructor(zk) {
    this.zk = zk;
    this.machineId = null;
    this.generator = null;
  }

  async initialize() {
    // Create ephemeral sequential node
    const path = await this.zk.create(
      '/snowflake/machines/worker-',
      null,
      zk.CreateMode.EPHEMERAL_SEQUENTIAL
    );

    // Extract sequence number as machine ID
    const sequence = path.split('-').pop();
    this.machineId = parseInt(sequence) % 1024;  // Max 1024 machines

    // Initialize generator with assigned machine ID
    this.generator = new SnowflakeGenerator(this.machineId % 32, Math.floor(this.machineId / 32));

    console.log(`Initialized with machine ID: ${this.machineId}`);
  }

  generate() {
    if (!this.generator) {
      throw new Error('Service not initialized');
    }
    return this.generator.generate();
  }
}
```

---

## Choosing the Right Strategy

### Decision Tree

```
Do you need IDs to be sortable by time?
├── No
│   └── UUID v4 (simplest, no coordination)
│
└── Yes
    ├── Need 64-bit (compact)?
    │   ├── Yes → Snowflake
    │   └── No
    │       ├── Need URL-safe string? → ULID
    │       └── UUID v7
    │
    └── Need strict ordering?
        ├── Yes → Database sequence
        └── No → Any time-based option
```

### Use Cases

| Use Case | Recommended |
|----------|-------------|
| Generic primary keys | UUID v7 |
| High-throughput events | Snowflake |
| URL slugs | NanoID / ULID |
| Financial transactions | Database sequence |
| Distributed logs | ULID |
| Legacy system compatibility | UUID v4 |
| Sharding key | Snowflake (extract timestamp) |

---

## Interview Discussion Points

### How to handle clock drift in Snowflake?

1. **NTP sync** - Keep clocks synchronized
2. **Detect backwards** - Refuse to generate, throw error
3. **Wait** - Sleep until clock catches up
4. **Logical clock** - Track last used timestamp
5. **Sequence overflow** - Use remaining sequence space

### How to ensure uniqueness across datacenters?

1. **Datacenter ID** - Embedded in ID (Snowflake)
2. **Range partitioning** - Different ID ranges per DC
3. **Coordination service** - ZooKeeper/etcd for machine IDs
4. **Random** - UUID has negligible collision probability
5. **Hybrid** - Timestamp + datacenter + random

### What happens if machine ID runs out?

1. **Recycle** - Reuse IDs from terminated instances
2. **Virtual workers** - Multiple generators per machine
3. **Expand bits** - Use 128-bit format
4. **Sharding** - Different ID spaces per service

### How to make IDs opaque (hide information)?

1. **Encryption** - Encrypt the ID
2. **Hashing** - Hash with secret key
3. **Shuffling** - Permute bits deterministically
4. **Encoding** - Base62/Base64 encoding
