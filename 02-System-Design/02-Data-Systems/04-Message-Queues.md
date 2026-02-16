# Message Queues

## Why Message Queues?

```
Without Queue (Synchronous):
┌────────┐     ┌────────┐     ┌────────┐
│ Client │────▶│ API    │────▶│ Email  │
│        │     │ Server │     │ Service│
└────────┘     └────────┘     └────────┘
     │              │              │
     │◄─────────────┴──────────────┘
     │        Wait for everything
     ▼        (slow, coupled, no retry)

With Queue (Asynchronous):
┌────────┐     ┌────────┐     ┌─────────┐     ┌────────┐
│ Client │────▶│ API    │────▶│  Queue  │────▶│ Worker │
│        │     │ Server │     │         │     │        │
└────────┘     └────────┘     └─────────┘     └────────┘
     │              │
     │◄─────────────┘
     │   Return immediately
     ▼   (fast, decoupled, reliable)
```

### Benefits

```
1. Decoupling      - Producer/consumer independent
2. Async processing - Don't wait for slow operations
3. Load leveling   - Handle traffic spikes
4. Reliability     - Messages persist if consumer down
5. Scalability     - Add consumers as needed
```

---

## Queue Patterns

### Point-to-Point (Work Queue)

```
                    ┌──────────┐
                    │ Producer │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │  Queue   │
                    │ [1,2,3,4]│
                    └────┬─────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │Consumer 1│   │Consumer 2│   │Consumer 3│
   │ gets: 1  │   │ gets: 2  │   │ gets: 3  │
   └──────────┘   └──────────┘   └──────────┘

Each message processed by ONE consumer
```

```javascript
// RabbitMQ work queue
const amqp = require('amqplib');

// Producer
async function sendTask(task) {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  await channel.assertQueue('tasks', { durable: true });
  channel.sendToQueue('tasks', Buffer.from(JSON.stringify(task)), {
    persistent: true
  });
}

// Consumer
async function processTask() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  await channel.assertQueue('tasks', { durable: true });
  channel.prefetch(1); // Process one at a time

  channel.consume('tasks', async (msg) => {
    const task = JSON.parse(msg.content.toString());
    await handleTask(task);
    channel.ack(msg); // Acknowledge completion
  });
}
```

### Publish-Subscribe (Fan-out)

```
                    ┌──────────┐
                    │ Publisher│
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ Exchange │
                    │ (fanout) │
                    └────┬─────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │ Queue A  │   │ Queue B  │   │ Queue C  │
   │(Email)   │   │(SMS)     │   │(Push)    │
   └──────────┘   └──────────┘   └──────────┘

Each message delivered to ALL subscribers
```

```javascript
// RabbitMQ pub-sub
// Publisher
async function publishEvent(event) {
  const channel = await getChannel();
  await channel.assertExchange('events', 'fanout', { durable: true });
  channel.publish('events', '', Buffer.from(JSON.stringify(event)));
}

// Subscriber
async function subscribeToEvents(handler) {
  const channel = await getChannel();
  await channel.assertExchange('events', 'fanout', { durable: true });

  // Each subscriber gets its own queue
  const { queue } = await channel.assertQueue('', { exclusive: true });
  await channel.bindQueue(queue, 'events', '');

  channel.consume(queue, (msg) => {
    handler(JSON.parse(msg.content.toString()));
    channel.ack(msg);
  });
}
```

### Topic-Based Routing

```
                    ┌──────────┐
                    │ Publisher│
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ Exchange │
                    │ (topic)  │
                    └────┬─────┘
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
    ▼                    ▼                    ▼
order.created      order.shipped      user.created
    │                    │                    │
    ▼                    ▼                    ▼
┌────────┐          ┌────────┐          ┌────────┐
│ Email  │          │ Track  │          │ Welcome│
│ Queue  │          │ Queue  │          │ Queue  │
└────────┘          └────────┘          └────────┘
Bound to:           Bound to:           Bound to:
order.*             order.shipped       user.*
```

```javascript
// Topic routing
channel.assertExchange('events', 'topic', { durable: true });

// Publish with routing key
channel.publish('events', 'order.created', Buffer.from(message));
channel.publish('events', 'user.registered', Buffer.from(message));

// Subscribe with pattern
channel.bindQueue(queue, 'events', 'order.*');     // All order events
channel.bindQueue(queue, 'events', '*.created');   // All created events
channel.bindQueue(queue, 'events', '#');           // All events
```

---

## Apache Kafka

### Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                         Kafka Cluster                          │
│                                                               │
│   Topic: orders                                               │
│   ┌─────────────────────────────────────────────────────────┐ │
│   │ Partition 0  │ Partition 1  │ Partition 2  │ Partition 3│ │
│   │ [0,1,2,3...] │ [0,1,2,3...] │ [0,1,2,3...] │ [0,1,2...] │ │
│   └─────────────────────────────────────────────────────────┘ │
│                                                               │
│   Broker 1        Broker 2        Broker 3                   │
│   (Leader P0)     (Leader P1)     (Leader P2,P3)             │
│   (Replica P1)    (Replica P2)    (Replica P0)               │
└───────────────────────────────────────────────────────────────┘
```

### Kafka vs Traditional Queues

| Feature | Kafka | RabbitMQ/SQS |
|---------|-------|--------------|
| Model | Log-based | Queue-based |
| Message retention | Time-based (days) | Until consumed |
| Consumer groups | Yes | Limited |
| Replay | Yes | No |
| Ordering | Per partition | Queue-wide |
| Throughput | Very high | Moderate |

### Kafka Producer

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'my-app',
  brokers: ['kafka1:9092', 'kafka2:9092']
});

const producer = kafka.producer();

async function sendMessage(topic, key, value) {
  await producer.connect();

  await producer.send({
    topic,
    messages: [
      { key, value: JSON.stringify(value) }
    ]
  });
}

// Batch sending
await producer.sendBatch({
  topicMessages: [
    {
      topic: 'orders',
      messages: orders.map(o => ({
        key: o.userId,
        value: JSON.stringify(o)
      }))
    }
  ]
});
```

### Kafka Consumer

```javascript
const consumer = kafka.consumer({ groupId: 'order-processor' });

async function consumeMessages() {
  await consumer.connect();
  await consumer.subscribe({ topic: 'orders', fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const order = JSON.parse(message.value.toString());
      await processOrder(order);
    }
  });
}

// Consumer with manual commits
await consumer.run({
  autoCommit: false,
  eachBatch: async ({ batch, commitOffsetsIfNecessary }) => {
    for (const message of batch.messages) {
      await processMessage(message);
    }
    await commitOffsetsIfNecessary();
  }
});
```

### Consumer Groups

```
Topic: orders (4 partitions)

Consumer Group: order-processors

┌─────────────┬─────────────┬─────────────┐
│ Consumer 1  │ Consumer 2  │ Consumer 3  │
│ P0, P1      │ P2          │ P3          │
└─────────────┴─────────────┴─────────────┘

- Each partition assigned to one consumer
- Adding consumers redistributes partitions
- More consumers than partitions = idle consumers
```

---

## AWS SQS

### Standard Queue

```javascript
const AWS = require('aws-sdk');
const sqs = new AWS.SQS();

// Send message
await sqs.sendMessage({
  QueueUrl: 'https://sqs.region.amazonaws.com/123/my-queue',
  MessageBody: JSON.stringify({ orderId: 123 }),
  MessageAttributes: {
    'orderType': {
      DataType: 'String',
      StringValue: 'express'
    }
  }
}).promise();

// Receive messages
const response = await sqs.receiveMessage({
  QueueUrl: queueUrl,
  MaxNumberOfMessages: 10,
  WaitTimeSeconds: 20,  // Long polling
  VisibilityTimeout: 30 // Seconds before retry
}).promise();

// Process and delete
for (const message of response.Messages) {
  await processMessage(JSON.parse(message.Body));
  await sqs.deleteMessage({
    QueueUrl: queueUrl,
    ReceiptHandle: message.ReceiptHandle
  }).promise();
}
```

### FIFO Queue

```javascript
// Guaranteed ordering, exactly-once processing
await sqs.sendMessage({
  QueueUrl: 'https://sqs.../my-queue.fifo',
  MessageBody: JSON.stringify(order),
  MessageGroupId: order.userId,  // Messages in same group are ordered
  MessageDeduplicationId: order.orderId  // Prevents duplicates
}).promise();
```

---

## Message Delivery Guarantees

### At-Most-Once

```
Producer → Broker → Consumer
           │
        No ack/retry

Message may be lost, never redelivered
Use: Metrics, logs (acceptable loss)
```

### At-Least-Once

```
Producer → Broker → Consumer
              ↑         │
              └─────────┘
              Retry until acked

Message may be delivered multiple times
Use: Most applications (with idempotency)
```

### Exactly-Once

```
Producer → Broker → Consumer
    │         │         │
    └────────────────────
    Transactions + Idempotency

Hard to achieve, performance cost
Use: Financial transactions
```

### Idempotency

```javascript
// Make operations idempotent
async function processOrder(order) {
  // Check if already processed
  const existing = await db.query(
    'SELECT * FROM processed_orders WHERE id = ?',
    [order.id]
  );

  if (existing) {
    console.log('Order already processed, skipping');
    return;
  }

  // Process order
  await db.transaction(async (trx) => {
    await trx.insert('orders', order);
    await trx.insert('processed_orders', { id: order.id });
  });
}

// Or use idempotency key
async function processPayment(payment, idempotencyKey) {
  const result = await redis.set(
    `payment:${idempotencyKey}`,
    'processing',
    'NX', 'EX', 3600
  );

  if (!result) {
    // Already processing/processed
    return await getExistingResult(idempotencyKey);
  }

  const paymentResult = await executePayment(payment);
  await redis.set(`payment:${idempotencyKey}`, JSON.stringify(paymentResult));
  return paymentResult;
}
```

---

## Dead Letter Queues

```
Main Queue → Consumer → Success
                │
                └──────▶ Failure (retry)
                              │
                              └──────▶ Max retries
                                            │
                                            ▼
                                    Dead Letter Queue
                                    (for investigation)
```

```javascript
// SQS with DLQ
const queueParams = {
  QueueName: 'my-queue',
  Attributes: {
    RedrivePolicy: JSON.stringify({
      deadLetterTargetArn: 'arn:aws:sqs:region:123:my-dlq',
      maxReceiveCount: 3  // After 3 failures, move to DLQ
    })
  }
};

// Process DLQ separately
async function processDLQ() {
  const messages = await sqs.receiveMessage({
    QueueUrl: dlqUrl
  }).promise();

  for (const msg of messages.Messages) {
    // Log for investigation
    await logFailedMessage(msg);
    // Optionally reprocess or archive
  }
}
```

---

## Backpressure Handling

```javascript
// Rate limiting consumer
class RateLimitedConsumer {
  constructor(maxPerSecond) {
    this.tokens = maxPerSecond;
    this.maxTokens = maxPerSecond;

    setInterval(() => {
      this.tokens = Math.min(this.tokens + this.maxTokens, this.maxTokens);
    }, 1000);
  }

  async consume(queue) {
    while (true) {
      if (this.tokens > 0) {
        const message = await queue.receive();
        if (message) {
          this.tokens--;
          await this.process(message);
        }
      } else {
        await sleep(100); // Wait for tokens
      }
    }
  }
}

// Producer backpressure
async function produceWithBackpressure(messages) {
  const queueDepth = await getQueueDepth();

  if (queueDepth > 10000) {
    // Queue too deep, slow down
    throw new Error('Queue backpressure - try again later');
  }

  await sendMessages(messages);
}
```

---

## Interview Questions

### Q: When would you use Kafka vs RabbitMQ?

**Kafka:**
- Event streaming
- Log aggregation
- Replay/reprocessing needed
- Very high throughput
- Event sourcing

**RabbitMQ:**
- Task queues
- Complex routing
- Request-reply patterns
- Lower latency requirements
- Message priorities

### Q: How do you handle poison messages?

1. **Retry with backoff** - Exponential delays
2. **Max retry count** - After N failures, move to DLQ
3. **Dead letter queue** - Store for investigation
4. **Alerts** - Notify on DLQ growth
5. **Manual intervention** - Fix and replay

### Q: How do you ensure exactly-once processing?

1. **Idempotent consumers** - Same input = same output
2. **Deduplication** - Track processed message IDs
3. **Transactions** - Atomic consume + process
4. **Idempotency keys** - Client-provided unique keys
