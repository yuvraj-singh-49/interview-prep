# AWS Lambda

## Overview

```
Lambda = Serverless compute service

You provide:        AWS manages:
- Code              - Servers
- Configuration     - Scaling
- Triggers          - Patching
                    - HA
                    - Monitoring

Pricing:
- Pay per request ($0.20 per 1M requests)
- Pay per compute time (GB-seconds)
- Free tier: 1M requests, 400,000 GB-seconds/month
```

---

## Lambda Function Configuration

### Key Settings

| Setting | Options | Impact |
|---------|---------|--------|
| **Memory** | 128 MB - 10,240 MB | CPU scales proportionally |
| **Timeout** | 1 sec - 15 min | Max execution time |
| **Ephemeral Storage** | 512 MB - 10 GB | /tmp directory |
| **Concurrency** | Reserved/Provisioned | Scaling behavior |
| **VPC** | Optional | Access private resources |

### Memory-CPU Relationship

```
Memory (MB)    vCPU (approximate)
128            0.083
512            0.33
1024           0.58
1769           1.0 (full vCPU)
3538           2.0
10240          6.0

Rule of thumb:
- CPU-bound: Increase memory for more CPU
- Memory-bound: Increase memory for more RAM
- Network-bound: Memory doesn't help much
```

---

## Invocation Models

### Synchronous Invocation

```
┌──────────┐    invoke    ┌──────────┐
│  Client  │─────────────▶│  Lambda  │
│          │◀─────────────│          │
└──────────┘   response   └──────────┘

- API Gateway, ALB, SDK direct invocation
- Client waits for response
- Errors returned directly to client
- Retry logic is client's responsibility
```

### Asynchronous Invocation

```
┌──────────┐    invoke    ┌──────────┐    process    ┌──────────┐
│  S3/SNS  │─────────────▶│  Queue   │──────────────▶│  Lambda  │
│          │    (async)   │(internal)│               │          │
└──────────┘              └──────────┘               └──────────┘

- S3, SNS, EventBridge, CloudWatch Events
- Immediate return (202 Accepted)
- Built-in retry (2 retries with backoff)
- Dead Letter Queue for failures
- Destination for success/failure routing
```

### Event Source Mapping

```
┌──────────┐    poll     ┌──────────┐    batch    ┌──────────┐
│ SQS/     │◀────────────│  Lambda  │◀────────────│  Lambda  │
│ Kinesis/ │   (Lambda   │ Service  │   invoke    │ Function │
│ DynamoDB │   polls)    │          │             │          │
└──────────┘             └──────────┘             └──────────┘

- Lambda polls the source
- Batches records for efficiency
- Retry depends on source
- Checkpointing for streams
```

---

## Event Sources

### Common Triggers

```
API-Driven:
├── API Gateway (REST, HTTP, WebSocket)
├── Application Load Balancer
└── Lambda Function URLs

Event-Driven:
├── S3 (object created, deleted)
├── DynamoDB Streams
├── Kinesis Data Streams
├── SQS
├── SNS
├── EventBridge
└── CloudWatch Events

Scheduled:
├── EventBridge Scheduler
└── CloudWatch Events (cron/rate)

Other:
├── Cognito (user pool triggers)
├── CloudFront (Lambda@Edge)
├── IoT
└── Alexa Skills
```

### Event Source Mapping Configuration

```javascript
// SQS Event Source Mapping
{
  "EventSourceArn": "arn:aws:sqs:us-east-1:123:my-queue",
  "FunctionName": "my-function",
  "BatchSize": 10,
  "MaximumBatchingWindowInSeconds": 5,
  "FunctionResponseTypes": ["ReportBatchItemFailures"],
  "ScalingConfig": {
    "MaximumConcurrency": 100
  }
}

// Kinesis Event Source Mapping
{
  "EventSourceArn": "arn:aws:kinesis:us-east-1:123:stream/my-stream",
  "FunctionName": "my-function",
  "StartingPosition": "TRIM_HORIZON",
  "BatchSize": 100,
  "ParallelizationFactor": 10,
  "MaximumRetryAttempts": 3,
  "BisectBatchOnFunctionError": true,
  "DestinationConfig": {
    "OnFailure": {
      "Destination": "arn:aws:sqs:us-east-1:123:dlq"
    }
  }
}
```

---

## Cold Starts

### Understanding Cold Starts

```
Cold Start Timeline:
┌─────────────────────────────────────────────────────────────────────────┐
│  Download code  │  Start runtime  │  Init code  │  Handler execution   │
│    (~50ms)      │   (~100-200ms)  │  (varies)   │      (your code)     │
├─────────────────┴─────────────────┴─────────────┼─────────────────────── │
│              Cold Start Overhead                 │    Billed Duration    │
└─────────────────────────────────────────────────┴───────────────────────┘

Cold start factors:
- Runtime (Java/C# > Python/Node.js)
- Package size (larger = slower)
- VPC configuration (+200-400ms historically, improved now)
- Memory allocation (more memory = faster init)
```

### Reducing Cold Starts

```
1. Provisioned Concurrency
   - Pre-warm execution environments
   - Pay for always-on capacity
   - Best for latency-sensitive workloads

2. Code Optimization
   - Minimize package size
   - Lazy load dependencies
   - Initialize outside handler (reused in warm starts)

3. Runtime Choice
   - Python, Node.js have faster cold starts
   - Use custom runtimes carefully

4. Architecture
   - Keep functions warm with scheduled pings
   - Use provisioned for critical paths
   - Accept cold starts for non-critical paths
```

### Provisioned Concurrency

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Concurrency Types                                     │
├─────────────────┬───────────────────────────────────────────────────────┤
│ Unreserved      │ Shared pool (1000 default per region)                 │
├─────────────────┼───────────────────────────────────────────────────────┤
│ Reserved        │ Guaranteed capacity, but still cold starts           │
├─────────────────┼───────────────────────────────────────────────────────┤
│ Provisioned     │ Pre-initialized, no cold starts                      │
└─────────────────┴───────────────────────────────────────────────────────┘

// Configure provisioned concurrency
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier prod \
  --provisioned-concurrent-executions 100
```

---

## Lambda Best Practices

### Code Organization

```javascript
// handler.js

// Initialize OUTSIDE handler (reused in warm starts)
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();
const secretsManager = new AWS.SecretsManager();

// Cache secrets
let cachedSecret = null;

async function getSecret() {
  if (cachedSecret) return cachedSecret;
  const data = await secretsManager.getSecretValue({SecretId: 'my-secret'}).promise();
  cachedSecret = JSON.parse(data.SecretString);
  return cachedSecret;
}

// Handler function
exports.handler = async (event, context) => {
  // Reduce timeout for faster failure
  context.callbackWaitsForEmptyEventLoop = false;

  try {
    const secret = await getSecret();

    // Process event
    const result = await processEvent(event, secret);

    return {
      statusCode: 200,
      body: JSON.stringify(result)
    };
  } catch (error) {
    console.error('Error:', error);
    throw error;  // Let Lambda handle retry
  }
};
```

### Error Handling

```javascript
// Partial batch failure (SQS/Kinesis)
exports.handler = async (event) => {
  const failedRecords = [];

  for (const record of event.Records) {
    try {
      await processRecord(record);
    } catch (error) {
      failedRecords.push({
        itemIdentifier: record.messageId  // SQS
        // or record.kinesis.sequenceNumber for Kinesis
      });
    }
  }

  return {
    batchItemFailures: failedRecords
  };
};

// Dead Letter Queue configuration
{
  "DeadLetterConfig": {
    "TargetArn": "arn:aws:sqs:us-east-1:123:my-dlq"
  }
}

// Destinations (more flexible than DLQ)
{
  "DestinationConfig": {
    "OnSuccess": {
      "Destination": "arn:aws:sqs:us-east-1:123:success-queue"
    },
    "OnFailure": {
      "Destination": "arn:aws:sqs:us-east-1:123:failure-queue"
    }
  }
}
```

### VPC Configuration

```
When to use VPC:
✓ Access RDS, ElastiCache, internal APIs
✓ Compliance requirements
✗ Don't use if only accessing public AWS APIs

VPC Best Practices:
1. Use VPC endpoints for AWS services (S3, DynamoDB, Secrets Manager)
2. Place Lambda in private subnets
3. Use NAT Gateway only if needed for internet
4. Ensure sufficient ENI capacity (IPs in subnets)
```

---

## Lambda Layers

```
Layer = Reusable code/dependencies

┌─────────────────────────────────────────────────────────────────────────┐
│                        Lambda Function                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                       Your Function Code                                 │
├────────────────────┬────────────────────┬───────────────────────────────┤
│     Layer 1        │      Layer 2       │         Layer 3               │
│  (Common libs)     │  (Monitoring)      │     (Custom runtime)          │
└────────────────────┴────────────────────┴───────────────────────────────┘

Benefits:
- Share code across functions
- Reduce deployment package size
- Separate dependencies from code
- Up to 5 layers per function
- Total size limit: 250 MB unzipped

// Layer structure
python/
  lib/
    python3.9/
      site-packages/
        my_package/

nodejs/
  node_modules/
    my-package/
```

---

## Lambda@Edge & CloudFront Functions

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Edge Computing Options                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   CloudFront Functions          │        Lambda@Edge                    │
│   - Viewer Request/Response     │  - Viewer + Origin Request/Response  │
│   - <1ms execution              │  - Up to 30s execution               │
│   - JavaScript only             │  - Node.js, Python                   │
│   - No network access           │  - Network access                    │
│   - 2MB code limit              │  - 50MB code limit                   │
│   - Very cheap (~$0.10/1M)      │  - More expensive (~$0.60/1M)        │
│                                                                          │
│   Use for:                      │  Use for:                            │
│   - URL rewrites                │  - Complex transformations           │
│   - Header manipulation         │  - Authentication                    │
│   - A/B testing                 │  - Dynamic origin selection          │
│   - Simple auth                 │  - Image optimization                │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Monitoring & Debugging

### CloudWatch Integration

```
Automatic Metrics:
- Invocations
- Duration
- Errors
- Throttles
- ConcurrentExecutions
- ProvisionedConcurrencyInvocations
- ProvisionedConcurrencySpilloverInvocations

Logs:
- Automatic logging to CloudWatch Logs
- Log group: /aws/lambda/<function-name>
- Structured logging recommended

// Example structured logging
console.log(JSON.stringify({
  level: 'INFO',
  message: 'Processing order',
  orderId: event.orderId,
  timestamp: new Date().toISOString()
}));
```

### X-Ray Tracing

```javascript
// Enable active tracing
{
  "TracingConfig": {
    "Mode": "Active"
  }
}

// Custom segments in code
const AWSXRay = require('aws-xray-sdk-core');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));

exports.handler = async (event) => {
  const segment = AWSXRay.getSegment();
  const subsegment = segment.addNewSubsegment('CustomOperation');

  try {
    // Your code
    subsegment.addAnnotation('orderId', event.orderId);
    const result = await processOrder(event);
    return result;
  } finally {
    subsegment.close();
  }
};
```

---

## Interview Discussion Points

### When to use Lambda vs EC2/ECS?

```
Use Lambda when:
- Event-driven workloads
- Variable/unpredictable traffic
- Short-duration tasks (<15 min)
- Don't want to manage infrastructure
- Cost optimization for sporadic workloads

Use EC2/ECS when:
- Long-running processes
- Predictable, steady workloads
- Need specific runtime/OS
- Complex networking requirements
- Cost optimization at high scale (breakeven ~1M invocations/day)
```

### How do you handle Lambda concurrency limits?

```
1. Monitor throttling
   - CloudWatch Throttles metric
   - Set alarms

2. Request limit increase
   - Default: 1000 concurrent (per region)
   - Request increase via Support

3. Reserved concurrency
   - Guarantee capacity for critical functions
   - Limit max for protection

4. Async with SQS
   - Buffer requests in SQS
   - Lambda processes at its own pace
   - No throttling errors to clients

5. Architecture
   - Fan-out pattern
   - Step Functions for orchestration
```

### How do you secure Lambda functions?

```
1. IAM
   - Least privilege execution role
   - Resource-based policies for invocation

2. VPC
   - Private subnets for sensitive data
   - Security groups for network control

3. Secrets
   - Secrets Manager or Parameter Store
   - Never hardcode credentials

4. Encryption
   - Environment variables encrypted at rest
   - Use KMS for sensitive data

5. Code security
   - Dependency scanning
   - No sensitive data in logs
   - Input validation
```
