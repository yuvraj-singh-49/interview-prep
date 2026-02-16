# Event-Driven Architecture

## What is Event-Driven Architecture?

A design pattern where services communicate through events rather than direct calls.

```
Traditional (Request-Driven):
┌─────────┐  1. Create order  ┌─────────┐
│  Order  │──────────────────▶│ Payment │
│ Service │  2. Charge        │ Service │
└─────────┘                   └────┬────┘
     │                             │
     │  3. Update inventory        │
     ▼                             ▼
┌─────────┐  4. Send email    ┌─────────┐
│Inventory│                   │  Email  │
└─────────┘                   │ Service │
                              └─────────┘

Event-Driven:
┌─────────┐                   ┌─────────┐
│  Order  │   OrderCreated    │ Payment │
│ Service │──────┬────────────│ Service │
└─────────┘      │            └─────────┘
                 │
            ┌────▼────┐
            │  Event  │       ┌─────────┐
            │   Bus   │───────│Inventory│
            └────┬────┘       └─────────┘
                 │
                 │            ┌─────────┐
                 └────────────│  Email  │
                              │ Service │
                              └─────────┘
```

---

## Core Concepts

### Events

```javascript
// Event = Fact that happened in the past
const orderCreatedEvent = {
  type: 'OrderCreated',
  timestamp: '2024-01-15T10:30:00Z',
  correlationId: 'uuid-123',
  data: {
    orderId: '12345',
    userId: 'user-789',
    items: [
      { productId: 'prod-1', quantity: 2, price: 29.99 }
    ],
    total: 59.98
  },
  metadata: {
    source: 'order-service',
    version: '1.0'
  }
};

// Event types:
// 1. Domain events - Business occurrences (OrderPlaced, PaymentReceived)
// 2. Integration events - Cross-service communication
// 3. System events - Technical occurrences (ServiceStarted, ErrorOccurred)
```

### Event Producers & Consumers

```javascript
// Producer - Emits events
class OrderService {
  async createOrder(orderData) {
    // Business logic
    const order = await this.repository.save(orderData);

    // Emit event
    await this.eventBus.publish('OrderCreated', {
      orderId: order.id,
      userId: order.userId,
      items: order.items,
      total: order.total
    });

    return order;
  }
}

// Consumer - Reacts to events
class NotificationService {
  constructor(eventBus) {
    eventBus.subscribe('OrderCreated', this.handleOrderCreated.bind(this));
    eventBus.subscribe('OrderShipped', this.handleOrderShipped.bind(this));
  }

  async handleOrderCreated(event) {
    await this.sendEmail({
      to: event.data.userEmail,
      template: 'order-confirmation',
      data: event.data
    });
  }
}
```

---

## Event-Driven Patterns

### Pub/Sub (Publish-Subscribe)

```
Publisher doesn't know subscribers

┌──────────┐                   ┌──────────┐
│Publisher │  OrderCreated     │Subscriber│
│          │──────┬────────────│   A      │
└──────────┘      │            └──────────┘
                  │
            ┌─────▼─────┐      ┌──────────┐
            │  Topic    │──────│Subscriber│
            │           │      │    B     │
            └───────────┘      └──────────┘
                  │
                  └────────────┌──────────┐
                               │Subscriber│
                               │    C     │
                               └──────────┘
```

### Event Sourcing

```javascript
// Store events as source of truth, derive state
class EventStore {
  constructor() {
    this.events = [];
  }

  async append(streamId, event) {
    this.events.push({
      streamId,
      ...event,
      position: this.events.length,
      timestamp: Date.now()
    });
  }

  async getStream(streamId) {
    return this.events.filter(e => e.streamId === streamId);
  }
}

// Rebuild state from events
class Order {
  static fromEvents(events) {
    const order = new Order();
    for (const event of events) {
      order.apply(event);
    }
    return order;
  }

  apply(event) {
    switch (event.type) {
      case 'OrderCreated':
        this.id = event.data.orderId;
        this.status = 'created';
        this.items = event.data.items;
        break;
      case 'OrderPaid':
        this.status = 'paid';
        this.paidAt = event.timestamp;
        break;
      case 'OrderShipped':
        this.status = 'shipped';
        this.trackingNumber = event.data.trackingNumber;
        break;
      case 'OrderCancelled':
        this.status = 'cancelled';
        this.cancelReason = event.data.reason;
        break;
    }
  }
}

// Usage
const events = await eventStore.getStream('order-123');
const order = Order.fromEvents(events);
```

### CQRS (Command Query Responsibility Segregation)

```
Commands (Write Side):
┌──────────┐     ┌──────────────┐     ┌──────────┐
│ Command  │────▶│   Command    │────▶│  Event   │
│   API    │     │   Handler    │     │  Store   │
└──────────┘     └──────────────┘     └────┬─────┘
                                           │
                                      Event Published
                                           │
                                           ▼
Queries (Read Side):                ┌──────────────┐
┌──────────┐     ┌──────────────┐   │  Projector   │
│  Query   │────▶│  Read Model  │◀──│  (Updates    │
│   API    │     │  (Optimized) │   │   read model)│
└──────────┘     └──────────────┘   └──────────────┘
```

```javascript
// Command handler (write)
class OrderCommandHandler {
  async handle(command) {
    switch (command.type) {
      case 'CreateOrder':
        const order = new Order(command.data);
        await this.eventStore.append('order-' + order.id, {
          type: 'OrderCreated',
          data: command.data
        });
        break;
    }
  }
}

// Projector (update read model)
class OrderProjector {
  constructor(eventBus, readDb) {
    eventBus.subscribe('OrderCreated', this.onOrderCreated.bind(this));
    eventBus.subscribe('OrderShipped', this.onOrderShipped.bind(this));
  }

  async onOrderCreated(event) {
    await this.readDb.insert('orders', {
      id: event.data.orderId,
      status: 'created',
      userId: event.data.userId,
      total: event.data.total,
      createdAt: event.timestamp
    });
  }

  async onOrderShipped(event) {
    await this.readDb.update('orders', event.data.orderId, {
      status: 'shipped',
      trackingNumber: event.data.trackingNumber,
      shippedAt: event.timestamp
    });
  }
}

// Query handler (read)
class OrderQueryHandler {
  async getOrdersByUser(userId) {
    // Read from optimized read model
    return this.readDb.query('orders', { userId });
  }
}
```

---

## Event Bus Implementation

### Simple In-Process

```javascript
class EventBus {
  constructor() {
    this.handlers = new Map();
  }

  subscribe(eventType, handler) {
    if (!this.handlers.has(eventType)) {
      this.handlers.set(eventType, []);
    }
    this.handlers.get(eventType).push(handler);
  }

  async publish(eventType, data) {
    const event = {
      type: eventType,
      data,
      timestamp: new Date().toISOString(),
      id: crypto.randomUUID()
    };

    const handlers = this.handlers.get(eventType) || [];
    await Promise.all(handlers.map(handler => handler(event)));
  }
}
```

### Kafka-Based

```javascript
const { Kafka } = require('kafkajs');

class KafkaEventBus {
  constructor() {
    this.kafka = new Kafka({ brokers: ['kafka:9092'] });
    this.producer = this.kafka.producer();
    this.consumers = new Map();
  }

  async publish(eventType, data) {
    await this.producer.connect();
    await this.producer.send({
      topic: eventType,
      messages: [{
        key: data.id?.toString(),
        value: JSON.stringify({
          type: eventType,
          data,
          timestamp: new Date().toISOString()
        })
      }]
    });
  }

  async subscribe(eventType, groupId, handler) {
    const consumer = this.kafka.consumer({ groupId });
    await consumer.connect();
    await consumer.subscribe({ topic: eventType });

    await consumer.run({
      eachMessage: async ({ message }) => {
        const event = JSON.parse(message.value.toString());
        await handler(event);
      }
    });
  }
}
```

---

## Event Schema Design

### Event Envelope

```javascript
{
  // Metadata
  "id": "evt-123-456-789",
  "type": "OrderCreated",
  "version": "1.0",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "correlationId": "req-abc-123",
  "causationId": "evt-previous-id",
  "source": "order-service",

  // Payload
  "data": {
    "orderId": "ord-123",
    "userId": "usr-456",
    "items": [...],
    "total": 99.99
  }
}
```

### Schema Evolution

```javascript
// Version 1
{
  "type": "UserCreated",
  "version": "1.0",
  "data": {
    "name": "John Doe"
  }
}

// Version 2 - Added field (backward compatible)
{
  "type": "UserCreated",
  "version": "2.0",
  "data": {
    "name": "John Doe",
    "email": "john@example.com"  // New field with default
  }
}

// Consumer handles both versions
function handleUserCreated(event) {
  const { name, email = 'unknown' } = event.data;
  // Handle missing email for v1 events
}
```

---

## Handling Failures

### Retry with Backoff

```javascript
class RetryableHandler {
  constructor(handler, options = {}) {
    this.handler = handler;
    this.maxRetries = options.maxRetries || 3;
    this.backoffMs = options.backoffMs || 1000;
  }

  async handle(event) {
    let lastError;

    for (let attempt = 0; attempt < this.maxRetries; attempt++) {
      try {
        return await this.handler(event);
      } catch (error) {
        lastError = error;
        const delay = this.backoffMs * Math.pow(2, attempt);
        await sleep(delay);
      }
    }

    // Max retries exceeded
    throw lastError;
  }
}
```

### Dead Letter Queue

```javascript
class EventProcessor {
  async process(event) {
    try {
      await this.handler(event);
    } catch (error) {
      if (event.retryCount >= 3) {
        // Move to dead letter queue
        await this.dlq.push({
          originalEvent: event,
          error: error.message,
          failedAt: new Date()
        });
      } else {
        // Retry
        await this.queue.push({
          ...event,
          retryCount: (event.retryCount || 0) + 1
        });
      }
    }
  }
}
```

### Idempotency

```javascript
class IdempotentHandler {
  constructor(handler, cache) {
    this.handler = handler;
    this.cache = cache;
  }

  async handle(event) {
    // Check if already processed
    const processed = await this.cache.get(`event:${event.id}`);
    if (processed) {
      console.log(`Event ${event.id} already processed, skipping`);
      return;
    }

    // Process event
    await this.handler(event);

    // Mark as processed
    await this.cache.set(`event:${event.id}`, true, { ttl: 86400 });
  }
}
```

---

## Event Ordering

### Per-Entity Ordering

```javascript
// Kafka: Use entity ID as partition key
await producer.send({
  topic: 'orders',
  messages: [{
    key: order.userId,  // All events for same user go to same partition
    value: JSON.stringify(event)
  }]
});

// Consumer processes in order per partition
```

### Causal Ordering

```javascript
// Include causation chain in events
{
  "id": "evt-3",
  "causationId": "evt-2",  // This event was caused by evt-2
  "correlationId": "saga-1",  // Part of same saga/transaction
  "type": "PaymentProcessed",
  "data": {...}
}

// Consumer can reconstruct order
function orderEvents(events) {
  // Build dependency graph
  const graph = new Map();
  for (const event of events) {
    graph.set(event.id, event.causationId);
  }
  // Topological sort
  return topologicalSort(events, graph);
}
```

---

## Sagas with Events

### Choreography (No Orchestrator)

```
Each service reacts to events and emits its own

OrderService              PaymentService            InventoryService
     │                          │                          │
     │─── OrderCreated ────────▶│                          │
     │                          │─── PaymentProcessed ────▶│
     │                          │                          │─── InventoryReserved ──┐
     │◀─────────────────────────────────────────────────────────────────────────────┘

Compensation:
     │                          │◀─── PaymentFailed ───────│
     │◀─── OrderCancelled ──────│                          │
```

### Orchestration (Central Coordinator)

```javascript
class OrderSaga {
  constructor(eventBus) {
    this.eventBus = eventBus;
    this.state = 'started';

    eventBus.subscribe('PaymentProcessed', this.onPaymentProcessed.bind(this));
    eventBus.subscribe('PaymentFailed', this.onPaymentFailed.bind(this));
    eventBus.subscribe('InventoryReserved', this.onInventoryReserved.bind(this));
    eventBus.subscribe('InventoryFailed', this.onInventoryFailed.bind(this));
  }

  async start(order) {
    this.orderId = order.id;
    await this.eventBus.publish('ProcessPayment', {
      orderId: order.id,
      amount: order.total
    });
    this.state = 'awaiting_payment';
  }

  async onPaymentProcessed(event) {
    if (event.data.orderId !== this.orderId) return;

    this.paymentId = event.data.paymentId;
    await this.eventBus.publish('ReserveInventory', {
      orderId: this.orderId,
      items: this.order.items
    });
    this.state = 'awaiting_inventory';
  }

  async onPaymentFailed(event) {
    if (event.data.orderId !== this.orderId) return;

    await this.eventBus.publish('OrderFailed', {
      orderId: this.orderId,
      reason: 'Payment failed'
    });
    this.state = 'failed';
  }

  async onInventoryReserved(event) {
    if (event.data.orderId !== this.orderId) return;

    await this.eventBus.publish('OrderCompleted', { orderId: this.orderId });
    this.state = 'completed';
  }

  async onInventoryFailed(event) {
    if (event.data.orderId !== this.orderId) return;

    // Compensate payment
    await this.eventBus.publish('RefundPayment', { paymentId: this.paymentId });
    await this.eventBus.publish('OrderFailed', {
      orderId: this.orderId,
      reason: 'Inventory unavailable'
    });
    this.state = 'failed';
  }
}
```

---

## Interview Questions

### Q: What are the pros and cons of event-driven architecture?

**Pros:**
- Loose coupling between services
- Easier to add new consumers
- Better scalability
- Natural audit log
- Resilience (async processing)

**Cons:**
- Eventual consistency complexity
- Debugging is harder
- Event ordering challenges
- Schema evolution complexity
- Learning curve

### Q: How do you ensure event ordering?

1. **Single partition** for related events
2. **Sequence numbers** in events
3. **Causation IDs** for causal ordering
4. **Saga orchestrator** for strict ordering
5. **Accept out-of-order** with idempotent handlers

### Q: How do you handle schema evolution?

1. **Backward compatible changes** only (add fields with defaults)
2. **Schema registry** (Avro, Protobuf)
3. **Event versioning** in type or envelope
4. **Consumer handles multiple versions**
5. **Transform/upgrade events** on read
