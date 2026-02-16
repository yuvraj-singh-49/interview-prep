# Stream Processing

## Batch vs Stream Processing

```
Batch Processing:
┌─────────────────────────────────────────┐
│  Data Lake  │ ──▶ │ Process │ ──▶ │ Output │
│  (static)   │     │ (hourly)│     │(delayed)│
└─────────────────────────────────────────┘
Latency: Hours
Example: Daily reports, ML training

Stream Processing:
┌─────────────────────────────────────────┐
│  Events  │ ──▶ │ Process │ ──▶ │ Output │
│(real-time)│    │(continuous)│  │(immediate)│
└─────────────────────────────────────────┘
Latency: Milliseconds to seconds
Example: Fraud detection, live dashboards
```

---

## Stream Processing Concepts

### Event Time vs Processing Time

```
Event Time:     When the event actually occurred
Processing Time: When the system processes the event

Event: User clicked at 10:00:00 (event time)
Network delay...
Arrived at processor at 10:00:05 (processing time)

Problem: Out-of-order events
Click A: event_time=10:00:01, arrives at 10:00:05
Click B: event_time=10:00:02, arrives at 10:00:03
Processing order: B, A (wrong!)
```

### Windowing

```
Tumbling Window:
|----Window 1----|----Window 2----|----Window 3----|
   events          events          events

Sliding Window:
|----Window 1----|
     |----Window 2----|
          |----Window 3----|
   (overlapping)

Session Window:
|--Session 1--|  gap  |--Session 2--|  gap  |--Session 3--|
   activity          activity            activity

Hopping Window:
|----Window 1----|
          |----Window 2----|
                    |----Window 3----|
   (fixed advance interval)
```

```javascript
// Tumbling window aggregation
class TumblingWindow {
  constructor(durationMs) {
    this.duration = durationMs;
    this.windows = new Map();
  }

  add(event) {
    const windowStart = Math.floor(event.timestamp / this.duration) * this.duration;

    if (!this.windows.has(windowStart)) {
      this.windows.set(windowStart, []);
    }
    this.windows.get(windowStart).push(event);
  }

  getWindow(timestamp) {
    const windowStart = Math.floor(timestamp / this.duration) * this.duration;
    return this.windows.get(windowStart) || [];
  }

  closeWindow(windowStart, callback) {
    const events = this.windows.get(windowStart);
    if (events) {
      callback(events);
      this.windows.delete(windowStart);
    }
  }
}
```

### Watermarks

Handle late-arriving events.

```
Watermark = "I've seen all events up to time T"

Timeline:
Events:    [t=10] [t=11] [t=12]      [t=9] (late!)
Watermark:   10     11     12   →   Can we still process t=9?

Allowed lateness: If watermark = 12 and allowed_lateness = 5
  → Events with t >= 7 still accepted
  → Events with t < 7 dropped
```

```javascript
class WatermarkTracker {
  constructor(allowedLateness) {
    this.watermark = 0;
    this.allowedLateness = allowedLateness;
    this.lateBuffer = [];
  }

  updateWatermark(eventTime) {
    if (eventTime > this.watermark) {
      this.watermark = eventTime;
    }
  }

  shouldProcess(eventTime) {
    if (eventTime >= this.watermark - this.allowedLateness) {
      return true;
    }
    // Too late, discard or send to side output
    return false;
  }
}
```

---

## Apache Kafka Streams

```java
// Kafka Streams topology
StreamsBuilder builder = new StreamsBuilder();

// Read from input topic
KStream<String, Order> orders = builder.stream("orders");

// Process: filter, map, aggregate
KTable<String, Long> orderCounts = orders
    .filter((key, order) -> order.getStatus().equals("COMPLETED"))
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count();

// Write to output topic
orderCounts.toStream().to("order-counts");

// Build and start
KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

```javascript
// Node.js with Kafka Streams (conceptual)
const { KafkaStreams } = require('kafka-streams');

const streams = new KafkaStreams(config);

const stream = streams.getKStream('orders');

stream
  .filter(order => order.status === 'COMPLETED')
  .map(order => ({ key: order.userId, value: 1 }))
  .countByKey('5-minute-window')
  .to('order-counts');

streams.start();
```

---

## Apache Flink

```java
// Flink streaming job
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<Order> orders = env
    .addSource(new FlinkKafkaConsumer<>("orders", new OrderSchema(), properties));

// Process with event time
orders
    .assignTimestampsAndWatermarks(
        WatermarkStrategy
            .<Order>forBoundedOutOfOrderness(Duration.ofSeconds(5))
            .withTimestampAssigner((order, ts) -> order.getTimestamp())
    )
    .keyBy(Order::getUserId)
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .aggregate(new OrderAggregator())
    .addSink(new FlinkKafkaProducer<>("order-aggregates", new AggregateSchema(), properties));

env.execute("Order Processing");
```

### Flink State Management

```java
// Stateful processing
public class FraudDetector extends KeyedProcessFunction<String, Transaction, Alert> {
    // Managed state
    private ValueState<Double> runningTotal;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("runningTotal", Double.class);
        runningTotal = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(Transaction tx, Context ctx, Collector<Alert> out) {
        Double total = runningTotal.value();
        if (total == null) total = 0.0;

        total += tx.getAmount();

        if (total > 10000) {
            out.collect(new Alert(tx.getUserId(), "High spending detected"));
        }

        runningTotal.update(total);
    }
}
```

---

## Real-Time Analytics Patterns

### Lambda Architecture

```
                              ┌──────────────┐
                              │  Batch Layer │
                              │  (accurate,  │
              ┌──────────────▶│   slow)      │────┐
              │               └──────────────┘    │
┌──────────┐  │                                   │  ┌──────────┐
│  Events  │──┤                                   ├─▶│  Serve   │
└──────────┘  │                                   │  │  Layer   │
              │               ┌──────────────┐    │  └──────────┘
              └──────────────▶│ Speed Layer  │────┘
                              │ (approx,     │
                              │  fast)       │
                              └──────────────┘

Query = Batch result + Speed result (since last batch)
```

### Kappa Architecture

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│  Events  │────▶│ Stream Layer │────▶│  Serve   │
└──────────┘     │ (all data    │     │  Layer   │
                 │  as streams) │     └──────────┘
                 └──────────────┘

Reprocess by replaying stream from beginning
Simpler than Lambda (single path)
```

---

## Common Stream Processing Patterns

### Event Sourcing

```javascript
// Store events, derive state
class EventStore {
  constructor() {
    this.events = [];
    this.snapshots = new Map();
  }

  append(event) {
    this.events.push({
      ...event,
      timestamp: Date.now(),
      sequenceNumber: this.events.length
    });
  }

  getState(entityId, atTime = Date.now()) {
    let state = this.snapshots.get(entityId) || {};

    const events = this.events.filter(e =>
      e.entityId === entityId &&
      e.timestamp <= atTime &&
      e.sequenceNumber > (state.sequenceNumber || -1)
    );

    for (const event of events) {
      state = this.applyEvent(state, event);
    }

    return state;
  }

  applyEvent(state, event) {
    switch (event.type) {
      case 'OrderCreated':
        return { ...state, status: 'created', items: event.items };
      case 'OrderPaid':
        return { ...state, status: 'paid', paidAt: event.timestamp };
      case 'OrderShipped':
        return { ...state, status: 'shipped', trackingNumber: event.trackingNumber };
      default:
        return state;
    }
  }
}
```

### CQRS (Command Query Responsibility Segregation)

```
Commands (writes):
┌──────────┐     ┌──────────────┐     ┌──────────┐
│ Command  │────▶│ Command      │────▶│ Event    │
│  API     │     │ Handler      │     │ Store    │
└──────────┘     └──────────────┘     └──────────┘
                                            │
                                     Event Published
                                            │
                                            ▼
Queries (reads):                     ┌──────────┐
┌──────────┐     ┌──────────────┐    │ Read     │
│ Query    │────▶│ Read Model   │◀───│ Model    │
│  API     │     │ (optimized)  │    │ Updater  │
└──────────┘     └──────────────┘    └──────────┘
```

### Change Data Capture (CDC)

```
Database changes → Event stream → Downstream systems

┌──────────┐     ┌──────────┐     ┌──────────┐
│ Database │────▶│ Debezium │────▶│  Kafka   │
│ (MySQL)  │     │   (CDC)  │     │          │
└──────────┘     └──────────┘     └────┬─────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              ▼                        ▼                        ▼
        ┌──────────┐            ┌──────────┐            ┌──────────┐
        │ Search   │            │ Cache    │            │Analytics │
        │ Index    │            │ (Redis)  │            │ (DWH)    │
        └──────────┘            └──────────┘            └──────────┘
```

```javascript
// Debezium event format
{
  "before": { "id": 1, "name": "John" },      // Previous state
  "after": { "id": 1, "name": "Jonathan" },   // New state
  "source": {
    "table": "users",
    "ts_ms": 1610000000000
  },
  "op": "u"  // c=create, u=update, d=delete
}
```

---

## Exactly-Once Semantics

### Transactional Outbox Pattern

```
┌─────────────────────────────────┐
│         Single Transaction       │
│  ┌─────────────────────────────┐│
│  │ 1. Update business table    ││
│  │ 2. Insert into outbox table ││
│  └─────────────────────────────┘│
└─────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│    Outbox Reader (separate)     │
│  1. Read outbox table           │
│  2. Publish to Kafka            │
│  3. Mark as published           │
└─────────────────────────────────┘
```

```javascript
// Transactional outbox
async function createOrder(order) {
  await db.transaction(async (trx) => {
    // Business logic
    const orderId = await trx.insert('orders', order);

    // Write to outbox (same transaction)
    await trx.insert('outbox', {
      aggregate_type: 'Order',
      aggregate_id: orderId,
      event_type: 'OrderCreated',
      payload: JSON.stringify(order),
      created_at: new Date()
    });
  });
}

// Separate process reads outbox
async function publishOutboxEvents() {
  const events = await db.query(
    'SELECT * FROM outbox WHERE published = false ORDER BY created_at LIMIT 100'
  );

  for (const event of events) {
    await kafka.publish('orders', event.payload);
    await db.query(
      'UPDATE outbox SET published = true WHERE id = ?',
      [event.id]
    );
  }
}
```

---

## Handling Backpressure

```javascript
// Reactive streams with backpressure
class BackpressureHandler {
  constructor(maxBufferSize) {
    this.buffer = [];
    this.maxBufferSize = maxBufferSize;
    this.paused = false;
  }

  async push(event) {
    if (this.buffer.length >= this.maxBufferSize) {
      // Signal upstream to slow down
      this.paused = true;
      await this.waitForSpace();
    }
    this.buffer.push(event);
  }

  async process() {
    while (true) {
      if (this.buffer.length > 0) {
        const event = this.buffer.shift();
        await this.handleEvent(event);

        if (this.paused && this.buffer.length < this.maxBufferSize / 2) {
          this.paused = false;
          // Signal upstream can resume
        }
      } else {
        await sleep(10);
      }
    }
  }
}
```

---

## Interview Questions

### Q: How do you handle out-of-order events?

1. **Watermarks** - Track expected event time progress
2. **Allowed lateness** - Accept events within threshold
3. **Buffering** - Hold events until watermark passes
4. **Side outputs** - Route late events separately
5. **Reprocessing** - Update aggregates when late events arrive

### Q: When would you use stream vs batch processing?

**Stream:**
- Real-time requirements (<1min latency)
- Continuous data sources
- Time-sensitive actions (fraud, alerts)
- Live dashboards

**Batch:**
- Historical analysis
- ML model training
- Complex transformations
- Cost optimization (cheaper than 24/7 streaming)

### Q: How do you achieve exactly-once processing?

1. **Idempotent operations** - Same input = same output
2. **Transactional outbox** - Atomically write event + state
3. **Kafka transactions** - Atomic read-process-write
4. **Deduplication** - Track processed event IDs
5. **Checkpointing** - Save progress for recovery
