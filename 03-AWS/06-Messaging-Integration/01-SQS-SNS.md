# SQS & SNS (Messaging Services)

## SQS Overview

```
SQS = Simple Queue Service (Message Queue)

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Producer ──────▶ SQS Queue ──────▶ Consumer                           │
│                    (buffer)                                              │
│                                                                          │
│   Key Characteristics:                                                   │
│   - Fully managed, unlimited throughput                                 │
│   - Message retention: 1 minute to 14 days (default 4 days)            │
│   - Message size: Up to 256 KB                                          │
│   - At-least-once delivery (Standard) or exactly-once (FIFO)           │
│   - Consumer pulls messages                                             │
│                                                                          │
│   Use Cases:                                                             │
│   - Decoupling microservices                                            │
│   - Buffering writes to database                                        │
│   - Handling traffic spikes                                             │
│   - Async job processing                                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SQS Queue Types

### Standard Queue

```
Standard Queue Characteristics:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Throughput:      Unlimited (nearly)                                   │
│   Ordering:        Best effort (not guaranteed)                         │
│   Delivery:        At-least-once (possible duplicates)                  │
│                                                                          │
│   Message Flow:                                                          │
│   ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐                                        │
│   │ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │  →  May arrive: 2, 1, 3, 3, 5, 4     │
│   └───┘ └───┘ └───┘ └───┘ └───┘                                        │
│                                                                          │
│   Best For:                                                              │
│   - High throughput requirements                                        │
│   - When order doesn't matter                                           │
│   - When app handles duplicates                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### FIFO Queue

```
FIFO Queue Characteristics:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Throughput:      300 msg/s (3000 with batching)                       │
│   Ordering:        Guaranteed (within message group)                    │
│   Delivery:        Exactly-once (deduplication)                         │
│                                                                          │
│   Message Groups:                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │ Group A: │ 1 │ 2 │ 3 │  →  Arrives: 1, 2, 3 (in order)         │   │
│   │ Group B: │ 1 │ 2 │    │  →  Arrives: 1, 2 (in order)           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Message Group ID:                                                      │
│   - Orders for user_123: Group "user_123"                               │
│   - Guarantees order within group                                       │
│   - Groups processed in parallel                                        │
│                                                                          │
│   Deduplication:                                                         │
│   - Deduplication ID (content-based or explicit)                        │
│   - 5-minute deduplication window                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

// FIFO Queue naming: Must end with .fifo
my-queue.fifo
```

---

## SQS Message Lifecycle

```
Message Lifecycle:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. Producer sends message                                             │
│      └── Message stored in queue                                        │
│                                                                          │
│   2. Consumer receives message                                          │
│      └── Message becomes invisible (visibility timeout starts)         │
│                                                                          │
│   3. Consumer processes message                                         │
│      ├── Success: Delete message                                        │
│      └── Failure: Message reappears after visibility timeout           │
│                                                                          │
│   4. After max retries → Dead Letter Queue (DLQ)                        │
│                                                                          │
│                                                                          │
│   Timeline:                                                              │
│   ──────────────────────────────────────────────────────────────────    │
│   │ Send │ Receive │───Visibility Timeout───│ Reappear │                │
│   ──────────────────────────────────────────────────────────────────    │
│                     │     Processing...     │                           │
│                     └── Delete if success ──┘                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Visibility Timeout

```
Visibility Timeout (0 sec - 12 hours, default 30 sec):

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Too Short:                                                             │
│   - Message reappears before processing complete                        │
│   - Duplicate processing                                                │
│                                                                          │
│   Too Long:                                                              │
│   - Failed message takes long to retry                                  │
│   - Stuck messages block processing                                     │
│                                                                          │
│   Best Practice:                                                         │
│   - Set to 6x expected processing time                                  │
│   - Extend dynamically if needed (ChangeMessageVisibility)              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

// Extend visibility timeout during processing
await sqs.changeMessageVisibility({
  QueueUrl: queueUrl,
  ReceiptHandle: message.ReceiptHandle,
  VisibilityTimeout: 120  // Extend by 2 minutes
});
```

### Dead Letter Queue

```
DLQ = Queue for failed messages

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Main Queue                                DLQ                         │
│   ┌──────────────────────┐                 ┌──────────────────────┐    │
│   │ maxReceiveCount: 3   │    After 3      │ Poison messages      │    │
│   │                      │    failures     │ for investigation    │    │
│   │   ┌───┐              │ ───────────────▶│   ┌───┐ ┌───┐       │    │
│   │   │msg│              │                 │   │msg│ │msg│       │    │
│   │   └───┘              │                 │   └───┘ └───┘       │    │
│   └──────────────────────┘                 └──────────────────────┘    │
│                                                                          │
│   Configuration:                                                         │
│   {                                                                      │
│     "RedrivePolicy": {                                                  │
│       "deadLetterTargetArn": "arn:aws:sqs:...:my-dlq",                 │
│       "maxReceiveCount": 3                                              │
│     }                                                                    │
│   }                                                                      │
│                                                                          │
│   Best Practices:                                                        │
│   - Always configure DLQ                                                │
│   - Set retention longer than main queue                                │
│   - Monitor DLQ depth (CloudWatch alarm)                                │
│   - Process DLQ messages (investigate, fix, redrive)                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SQS Polling

### Long Polling vs Short Polling

```
Short Polling (Default):
- Returns immediately (even if empty)
- May not return all messages
- More API calls, higher cost

Long Polling (Recommended):
- Waits up to 20 seconds for messages
- Reduces empty responses
- Reduces API calls and cost

// Enable long polling
{
  "ReceiveMessageWaitTimeSeconds": 20  // Queue level
}

// Or per request
await sqs.receiveMessage({
  QueueUrl: queueUrl,
  WaitTimeSeconds: 20  // Request level
});
```

### Consuming Messages

```javascript
// Basic consumer pattern
async function processMessages() {
  while (true) {
    const response = await sqs.receiveMessage({
      QueueUrl: QUEUE_URL,
      MaxNumberOfMessages: 10,  // Batch up to 10
      WaitTimeSeconds: 20,      // Long polling
      VisibilityTimeout: 60     // 1 minute to process
    });

    if (!response.Messages) continue;

    for (const message of response.Messages) {
      try {
        await processMessage(message);

        await sqs.deleteMessage({
          QueueUrl: QUEUE_URL,
          ReceiptHandle: message.ReceiptHandle
        });
      } catch (error) {
        console.error('Processing failed:', error);
        // Message will reappear after visibility timeout
      }
    }
  }
}

// Lambda trigger (event source mapping handles this)
exports.handler = async (event) => {
  const batchItemFailures = [];

  for (const record of event.Records) {
    try {
      await processMessage(record);
    } catch (error) {
      batchItemFailures.push({
        itemIdentifier: record.messageId
      });
    }
  }

  return { batchItemFailures };  // Partial batch failure
};
```

---

## SNS Overview

```
SNS = Simple Notification Service (Pub/Sub)

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Publisher ──────▶ SNS Topic ──────▶ Subscribers (fan-out)             │
│                                                                          │
│                        ┌──────▶ SQS Queue 1                             │
│   Event ──▶ Topic ─────┼──────▶ SQS Queue 2                             │
│                        ├──────▶ Lambda                                  │
│                        ├──────▶ HTTP Endpoint                           │
│                        ├──────▶ Email                                   │
│                        └──────▶ SMS                                     │
│                                                                          │
│   Key Characteristics:                                                   │
│   - Push-based delivery                                                 │
│   - Up to 12.5 million subscriptions per topic                         │
│   - Up to 100,000 topics                                                │
│   - Message filtering                                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### SNS vs SQS

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Feature          │ SNS                    │ SQS                        │
├──────────────────┼────────────────────────┼────────────────────────────┤
│ Model            │ Pub/Sub (push)         │ Queue (pull)               │
│ Persistence      │ No (fire and forget)   │ Yes (until consumed)       │
│ Consumers        │ Multiple (fan-out)     │ Single (or competing)      │
│ Message handling │ Immediate delivery     │ Polling required           │
│ Retry            │ Delivery policies      │ Visibility timeout         │
│ Ordering         │ FIFO topics available  │ Standard or FIFO           │
└──────────────────┴────────────────────────┴────────────────────────────┘

Common Pattern - SNS + SQS (Fanout):
┌───────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐
│ Publisher │────▶│ SNS Topic │────▶│ SQS Queue │────▶│ Consumer  │
└───────────┘     └─────┬─────┘     └───────────┘     └───────────┘
                        │
                        │           ┌───────────┐     ┌───────────┐
                        └──────────▶│ SQS Queue │────▶│ Consumer  │
                                    └───────────┘     └───────────┘

Benefits:
- Decoupling (producers don't know consumers)
- Each consumer processes independently
- Different processing speeds
- Easy to add new consumers
```

### Message Filtering

```
Message Filtering = Route messages to specific subscribers

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Message Attributes:                                                    │
│   {                                                                      │
│     "eventType": "order_created",                                       │
│     "customer_tier": "premium"                                          │
│   }                                                                      │
│                                                                          │
│   Subscription Filter Policies:                                          │
│                                                                          │
│   Order Service Filter:                                                  │
│   {                                                                      │
│     "eventType": ["order_created", "order_updated"]                     │
│   }                                                                      │
│                                                                          │
│   Premium Service Filter:                                                │
│   {                                                                      │
│     "customer_tier": ["premium", "enterprise"]                          │
│   }                                                                      │
│                                                                          │
│   Analytics Filter:                                                      │
│   {} // Empty = receives all messages                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Filter Policy Operators:
- Exact match: ["value1", "value2"]
- Prefix: [{"prefix": "order_"}]
- Numeric: [{"numeric": [">=", 100]}]
- Exists: [{"exists": true}]
- Anything-but: [{"anything-but": ["ignore"]}]
```

---

## SNS FIFO Topics

```
SNS FIFO Topics:
- Strict message ordering
- Exactly-once message delivery
- Deduplication

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   SNS FIFO Topic ──────▶ SQS FIFO Queue                                 │
│   (my-topic.fifo)       (my-queue.fifo)                                 │
│                                                                          │
│   Requirements:                                                          │
│   - Topic name ends with .fifo                                          │
│   - Can only subscribe FIFO queues                                      │
│   - Throughput: 300 TPS (10 MB/s)                                       │
│                                                                          │
│   Message Group ID:                                                      │
│   - Groups processed in parallel                                        │
│   - Order guaranteed within group                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Common Patterns

### Request-Response with Correlation

```javascript
// Producer: Send request with correlation ID
const correlationId = uuid();

await sqs.sendMessage({
  QueueUrl: REQUEST_QUEUE,
  MessageBody: JSON.stringify({ action: 'process', data: payload }),
  MessageAttributes: {
    CorrelationId: { DataType: 'String', StringValue: correlationId },
    ReplyTo: { DataType: 'String', StringValue: RESPONSE_QUEUE }
  }
});

// Wait for response with matching correlation ID
// (or use temporary queue per request)

// Consumer: Process and send response
exports.handler = async (event) => {
  for (const record of event.Records) {
    const correlationId = record.messageAttributes.CorrelationId.stringValue;
    const replyTo = record.messageAttributes.ReplyTo.stringValue;

    const result = await processRequest(JSON.parse(record.body));

    await sqs.sendMessage({
      QueueUrl: replyTo,
      MessageBody: JSON.stringify(result),
      MessageAttributes: {
        CorrelationId: { DataType: 'String', StringValue: correlationId }
      }
    });
  }
};
```

### Delayed Processing

```
Delay Options:

1. Message Delay (per message, up to 15 min):
await sqs.sendMessage({
  QueueUrl: queueUrl,
  MessageBody: body,
  DelaySeconds: 900  // 15 minutes
});

2. Queue Delay (default for all messages):
{
  "DelaySeconds": 60  // Queue-level setting
}

Use Cases:
- Scheduled tasks
- Rate limiting
- Retry with backoff
- Workflow timing
```

### Priority Queue Pattern

```
Multiple Queues for Priority:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   High Priority Queue    ──────▶ Consumer (weighted polling)            │
│   (check more frequently)       │                                       │
│                                 │                                       │
│   Normal Priority Queue  ──────▶│                                       │
│   (check less frequently)       │                                       │
│                                 │                                       │
│   Low Priority Queue     ──────▶│                                       │
│   (process when idle)           ▼                                       │
│                            Processing                                   │
│                                                                          │
│   Implementation:                                                        │
│   - 3 queues with different polling ratios                              │
│   - High: 60%, Normal: 30%, Low: 10%                                   │
│   - Or: Always drain high before normal                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Interview Discussion Points

### How do you ensure message ordering?

```
Strategies:

1. SQS FIFO Queue
   - Use message group ID for related messages
   - Guarantees order within group
   - Trade-off: 300 msg/s limit

2. Single Consumer
   - One consumer instance processes all messages
   - No parallelism, but guaranteed order
   - Not scalable

3. Partitioned Processing
   - Route related messages to same consumer
   - Like Kafka partitions
   - Consumer affinity (sticky sessions)

4. Sequence Numbers
   - Include sequence in message
   - Consumer reorders or rejects out-of-order
   - Application-level ordering
```

### How do you handle poison messages?

```
Poison Message = Message that always fails processing

Solutions:

1. Dead Letter Queue (Primary)
   - Configure maxReceiveCount
   - Failed messages move to DLQ
   - Investigate and fix

2. Circuit Breaker
   - Track failure rate
   - Stop processing if too many failures
   - Alert and investigate

3. Message Validation
   - Validate before queuing
   - Fail fast at producer

4. Graceful Degradation
   - Try alternate processing
   - Log and continue
   - Skip non-critical messages

5. Manual Intervention
   - DLQ monitoring
   - Human review for persistent failures
   - Replay after fix
```

### How do you scale SQS consumers?

```
Scaling Strategies:

1. Lambda (Automatic)
   - Event source mapping
   - Concurrent executions = up to 1000
   - Batch size tuning

2. EC2/ECS (Manual/Auto)
   - Monitor queue depth (ApproximateNumberOfMessages)
   - Auto Scaling based on metric
   - Target: Messages per consumer

// CloudWatch Alarm for scaling
{
  "MetricName": "ApproximateNumberOfMessages",
  "Namespace": "AWS/SQS",
  "Threshold": 1000,
  "ComparisonOperator": "GreaterThanThreshold"
}

3. Optimization
   - Long polling (reduce API calls)
   - Batch processing (10 messages)
   - Delete in batch
   - Increase visibility timeout

4. High Throughput
   - Multiple receive calls in parallel
   - Horizontal scaling (more consumers)
   - Consider Kinesis for extreme scale
```

### When to use SNS vs SQS vs EventBridge?

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Use Case                        │ Service                               │
├─────────────────────────────────┼───────────────────────────────────────┤
│ Simple fanout                   │ SNS + SQS                            │
│ Work queue (one consumer)       │ SQS                                  │
│ Buffering/decoupling           │ SQS                                  │
│ AWS service events             │ EventBridge                          │
│ Complex routing/filtering      │ EventBridge                          │
│ Scheduled events               │ EventBridge                          │
│ Cross-account events           │ EventBridge                          │
│ Human notifications            │ SNS (email, SMS)                     │
│ High-throughput streaming      │ Kinesis (not SQS/SNS)               │
└─────────────────────────────────┴───────────────────────────────────────┘
```
