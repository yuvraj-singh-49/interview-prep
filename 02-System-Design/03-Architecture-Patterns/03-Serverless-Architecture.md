# Serverless Architecture

## What is Serverless?

```
Traditional:           Serverless:
┌─────────────┐       ┌─────────────┐
│   Server    │       │  Function   │
│  (always on)│       │ (on-demand) │
└─────────────┘       └─────────────┘
      │                     │
  Pay 24/7              Pay per use
  Scale manually        Auto-scale
  Manage infra          No infra
```

---

## Serverless Components

### Functions as a Service (FaaS)

```javascript
// AWS Lambda function
exports.handler = async (event, context) => {
  const { name } = JSON.parse(event.body);

  const result = await processData(name);

  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message: `Hello, ${name}!`, result })
  };
};

// Triggered by:
// - HTTP requests (API Gateway)
// - Queue messages (SQS)
// - Database changes (DynamoDB Streams)
// - File uploads (S3)
// - Scheduled events (CloudWatch Events)
```

### Backend as a Service (BaaS)

```
Managed services replace custom backend:

┌─────────────────────────────────────────────────────────────┐
│                      BaaS Components                         │
├─────────────────┬─────────────────┬─────────────────────────┤
│ Auth            │ Database        │ Storage                  │
│ (Firebase Auth, │ (DynamoDB,      │ (S3, Cloudinary)        │
│  Auth0, Cognito)│  Firestore)     │                         │
├─────────────────┼─────────────────┼─────────────────────────┤
│ Messaging       │ Search          │ Analytics               │
│ (SNS, Twilio)   │ (Algolia)       │ (Mixpanel, Amplitude)  │
└─────────────────┴─────────────────┴─────────────────────────┘
```

---

## Serverless Patterns

### API Backend

```
┌──────────┐     ┌─────────────┐     ┌──────────┐
│  Client  │────▶│ API Gateway │────▶│  Lambda  │
└──────────┘     └─────────────┘     └────┬─────┘
                                          │
                                    ┌─────▼─────┐
                                    │ DynamoDB  │
                                    └───────────┘
```

```yaml
# serverless.yml
service: my-api

provider:
  name: aws
  runtime: nodejs18.x

functions:
  getUsers:
    handler: handlers/users.get
    events:
      - http:
          path: users/{id}
          method: get

  createUser:
    handler: handlers/users.create
    events:
      - http:
          path: users
          method: post
```

### Event Processing

```
┌──────────┐     ┌─────────┐     ┌──────────┐     ┌─────────┐
│    S3    │────▶│  Event  │────▶│  Lambda  │────▶│ Database│
│ (upload) │     │ Trigger │     │(process) │     │ (store) │
└──────────┘     └─────────┘     └──────────┘     └─────────┘
```

```javascript
// Image processing on upload
exports.handler = async (event) => {
  const bucket = event.Records[0].s3.bucket.name;
  const key = event.Records[0].s3.object.key;

  // Download image
  const image = await s3.getObject({ Bucket: bucket, Key: key }).promise();

  // Process (resize, optimize)
  const processed = await sharp(image.Body)
    .resize(800, 600)
    .jpeg({ quality: 80 })
    .toBuffer();

  // Upload processed version
  await s3.putObject({
    Bucket: bucket,
    Key: `processed/${key}`,
    Body: processed
  }).promise();

  return { status: 'success' };
};
```

### Fan-Out Pattern

```
                    ┌──────────────┐
                    │    SNS       │
                    │   Topic      │
                    └──────┬───────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │ Lambda   │      │ Lambda   │      │ Lambda   │
   │ (Email)  │      │ (SMS)    │      │ (Push)   │
   └──────────┘      └──────────┘      └──────────┘
```

```javascript
// Publisher
await sns.publish({
  TopicArn: 'arn:aws:sns:...:notifications',
  Message: JSON.stringify({
    userId: '123',
    type: 'order_placed',
    data: { orderId: '456' }
  })
}).promise();

// Each Lambda subscriber handles independently
```

### Scheduled Tasks

```yaml
functions:
  dailyReport:
    handler: handlers/reports.daily
    events:
      - schedule: cron(0 9 * * ? *)  # 9 AM daily

  cleanup:
    handler: handlers/cleanup.run
    events:
      - schedule: rate(1 hour)
```

---

## Cold Starts

### What is Cold Start?

```
First request (cold):
┌──────────┐     ┌─────────────────────────────────────────┐
│ Request  │────▶│ Download code → Init runtime → Execute │
└──────────┘     │   (100-500ms)    (50-200ms)   (Xms)    │
                 └─────────────────────────────────────────┘
                          Cold start overhead

Subsequent requests (warm):
┌──────────┐     ┌──────────┐
│ Request  │────▶│ Execute  │
└──────────┘     └──────────┘
                    (Xms)
```

### Mitigation Strategies

```javascript
// 1. Provisioned Concurrency (AWS)
// Pre-warm N instances

// serverless.yml
functions:
  api:
    handler: handler.api
    provisionedConcurrency: 5

// 2. Keep functions warm with scheduled pings
functions:
  api:
    handler: handler.api
    events:
      - http: ...
      - schedule:
          rate: rate(5 minutes)
          input:
            source: "serverless-plugin-warmup"

// 3. Optimize initialization
// Move outside handler
const db = require('./db'); // Runs once per container
const config = JSON.parse(process.env.CONFIG);

exports.handler = async (event) => {
  // Only handler code runs per request
  return await db.query(...);
};

// 4. Use smaller runtimes
// Node.js/Python < Java/C#

// 5. Minimize dependencies
// Use tree-shaking, avoid large SDKs
```

---

## State Management

### Stateless Functions

```javascript
// Bad: In-memory state
let counter = 0;
exports.handler = async (event) => {
  counter++; // Won't persist across invocations
  return { count: counter };
};

// Good: External state
exports.handler = async (event) => {
  const counter = await redis.incr('counter');
  return { count: counter };
};
```

### State Storage Options

```
┌─────────────────┬──────────────────┬─────────────────┐
│ Use Case        │ Service          │ Latency         │
├─────────────────┼──────────────────┼─────────────────┤
│ Key-Value       │ DynamoDB, Redis  │ ~10ms           │
│ Sessions        │ ElastiCache      │ ~1ms            │
│ Files           │ S3               │ ~50ms           │
│ Relational      │ RDS, Aurora      │ ~10ms           │
│ Workflow State  │ Step Functions   │ Managed         │
└─────────────────┴──────────────────┴─────────────────┘
```

### Step Functions (Orchestration)

```json
{
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:validateOrder",
      "Next": "ProcessPayment"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:processPayment",
      "Next": "CheckInventory",
      "Catch": [{
        "ErrorEquals": ["PaymentFailed"],
        "Next": "CancelOrder"
      }]
    },
    "CheckInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:checkInventory",
      "Next": "ShipOrder"
    },
    "ShipOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:shipOrder",
      "End": true
    },
    "CancelOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:cancelOrder",
      "End": true
    }
  }
}
```

---

## Database Connections

### Connection Pooling Challenge

```javascript
// Problem: Each Lambda creates new connection
exports.handler = async (event) => {
  const client = new Client();
  await client.connect();  // New connection every time!
  const result = await client.query('SELECT ...');
  await client.end();
  return result;
};

// Solution 1: Reuse connection across invocations
let client;
exports.handler = async (event) => {
  if (!client) {
    client = new Client();
    await client.connect();
  }
  return await client.query('SELECT ...');
};

// Solution 2: Use connection pooler (RDS Proxy, PgBouncer)
const client = new Client({
  host: 'my-proxy.proxy-xxx.region.rds.amazonaws.com',
  // Proxy manages connection pooling
});

// Solution 3: Use serverless-friendly databases
// - DynamoDB (HTTP-based, no connections)
// - Aurora Serverless Data API
// - PlanetScale, Neon (HTTP endpoints)
```

---

## Security

### IAM Roles

```yaml
# Principle of least privilege
provider:
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
          Resource: arn:aws:dynamodb:*:*:table/users

        - Effect: Allow
          Action:
            - s3:GetObject
          Resource: arn:aws:s3:::my-bucket/*
```

### Secrets Management

```javascript
// Use AWS Secrets Manager or Parameter Store
const AWS = require('aws-sdk');
const ssm = new AWS.SSM();

let cachedSecret;

async function getSecret() {
  if (cachedSecret) return cachedSecret;

  const result = await ssm.getParameter({
    Name: '/myapp/database-url',
    WithDecryption: true
  }).promise();

  cachedSecret = result.Parameter.Value;
  return cachedSecret;
}

exports.handler = async (event) => {
  const dbUrl = await getSecret();
  // Use secret
};
```

---

## Cost Optimization

### Pricing Model

```
Lambda pricing:
- Requests: $0.20 per 1M requests
- Duration: $0.0000166667 per GB-second

Example:
- 1M requests/month
- 1GB memory, 200ms average
- Cost = $0.20 + (1M × 0.2s × 1GB × $0.0000166667)
- Cost = $0.20 + $3.33 = $3.53/month

Compare to EC2 t3.small (~$15/month) always running
```

### Optimization Tips

```javascript
// 1. Right-size memory (affects CPU too)
// Test different memory settings

// 2. Minimize execution time
// Use async/parallel when possible
const [users, orders] = await Promise.all([
  getUsers(),
  getOrders()
]);

// 3. Use ARM64 (Graviton2) - 20% cheaper
// serverless.yml
provider:
  architecture: arm64

// 4. Batch operations
// Process multiple records per invocation
exports.handler = async (event) => {
  const records = event.Records;
  await Promise.all(records.map(processRecord));
};
```

---

## When to Use Serverless

### Good Fit

```
✓ Variable/unpredictable traffic
✓ Event-driven workloads
✓ Microservices
✓ Scheduled tasks
✓ Rapid prototyping
✓ Cost-sensitive (pay per use)
✓ APIs with <15 min execution
```

### Not Ideal

```
✗ Long-running processes (>15 min)
✗ Consistent high traffic (cheaper with servers)
✗ WebSocket connections (use dedicated service)
✗ Heavy computation (consider containers)
✗ Legacy applications
✗ Need for local filesystem
```

---

## Interview Questions

### Q: How do you handle long-running processes in serverless?

1. **Break into smaller functions** - Chain via SQS/Step Functions
2. **Step Functions** - Orchestrate multi-step workflows
3. **Fargate** - Use containers for long processes
4. **Increase timeout** - Lambda max is 15 minutes

### Q: How do you monitor serverless applications?

1. **CloudWatch Logs** - Automatic logging
2. **CloudWatch Metrics** - Invocations, errors, duration
3. **X-Ray** - Distributed tracing
4. **Third-party** - Datadog, Lumigo, Epsagon

### Q: How do you handle cold starts for latency-sensitive APIs?

1. **Provisioned Concurrency** - Pre-warm instances
2. **Smaller bundles** - Faster download/init
3. **Lighter runtime** - Node.js over Java
4. **Keep warm** - Scheduled pings
5. **Edge functions** - CloudFront/Lambda@Edge
