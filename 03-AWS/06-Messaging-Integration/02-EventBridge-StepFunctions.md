# EventBridge & Step Functions

## EventBridge Overview

```
EventBridge = Serverless Event Bus

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Event Sources              Event Bus              Targets             │
│   ┌───────────────┐         ┌───────────┐         ┌───────────────┐    │
│   │ AWS Services  │────────▶│           │────────▶│ Lambda        │    │
│   │ (S3, EC2...)  │         │           │         │ SQS, SNS      │    │
│   ├───────────────┤         │  Default  │         │ Step Functions│    │
│   │ Custom Apps   │────────▶│  or       │────────▶│ API Gateway   │    │
│   │ (PutEvents)   │         │  Custom   │         │ Kinesis       │    │
│   ├───────────────┤         │  Bus      │         │ CloudWatch    │    │
│   │ SaaS Partners │────────▶│           │────────▶│ And more...   │    │
│   │ (Zendesk...)  │         │           │         │               │    │
│   └───────────────┘         └───────────┘         └───────────────┘    │
│                                  │                                       │
│                            Rules (filtering)                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Key Concepts:
- Event Bus: Container for events
- Rules: Filter and route events
- Targets: Destinations for matched events
- Schema Registry: Event structure definitions
```

---

## Event Structure

```json
{
  "version": "0",
  "id": "12345678-1234-1234-1234-123456789012",
  "detail-type": "EC2 Instance State-change Notification",
  "source": "aws.ec2",
  "account": "123456789012",
  "time": "2024-01-15T12:30:00Z",
  "region": "us-east-1",
  "resources": [
    "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0"
  ],
  "detail": {
    "instance-id": "i-1234567890abcdef0",
    "state": "stopped"
  }
}

Custom Event:
{
  "Source": "com.myapp.orders",
  "DetailType": "Order Created",
  "Detail": "{\"orderId\": \"123\", \"amount\": 99.99}",
  "EventBusName": "my-custom-bus"
}
```

---

## Event Rules

### Rule Patterns

```json
// Match specific source
{
  "source": ["aws.ec2"]
}

// Match detail fields
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["stopped", "terminated"]
  }
}

// Pattern matching
{
  "source": ["com.myapp.orders"],
  "detail": {
    "amount": [{"numeric": [">", 1000]}],          // Greater than 1000
    "status": [{"prefix": "SHIP"}],                 // Starts with SHIP
    "customer": [{"exists": true}],                 // Field exists
    "region": [{"anything-but": ["test"]}]          // Not "test"
  }
}

// Content filtering (arrays)
{
  "detail": {
    "items": {
      "category": ["electronics"]                   // Array contains
    }
  }
}
```

### Rule Targets

```
Target Types:
┌─────────────────────────────────────────────────────────────────────────┐
│ Target            │ Use Case                                            │
├───────────────────┼─────────────────────────────────────────────────────┤
│ Lambda            │ Process events, custom logic                        │
│ Step Functions    │ Orchestrate workflows                               │
│ SQS               │ Buffer events, fan-out                              │
│ SNS               │ Fan-out to multiple subscribers                     │
│ Kinesis           │ Stream processing                                   │
│ API Gateway       │ Invoke HTTP endpoints                               │
│ CloudWatch Logs   │ Archive events                                      │
│ EventBridge Bus   │ Cross-account, cross-region                         │
└───────────────────┴─────────────────────────────────────────────────────┘

Input Transformation:
{
  "InputPathsMap": {
    "orderId": "$.detail.orderId",
    "amount": "$.detail.amount"
  },
  "InputTemplate": "{\"order\": \"<orderId>\", \"value\": <amount>}"
}
```

---

## Scheduled Events

```
EventBridge Scheduler:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Schedule Types:                                                        │
│                                                                          │
│   1. Rate-based:                                                         │
│      rate(5 minutes)                                                    │
│      rate(1 hour)                                                       │
│      rate(7 days)                                                       │
│                                                                          │
│   2. Cron-based:                                                         │
│      cron(0 12 * * ? *)      # Every day at 12:00 UTC                  │
│      cron(0 8 ? * MON-FRI *) # Weekdays at 8:00 UTC                    │
│      cron(0/15 * * * ? *)    # Every 15 minutes                        │
│                                                                          │
│   Format: cron(minute hour day-of-month month day-of-week year)         │
│                                                                          │
│   EventBridge Scheduler (newer):                                         │
│   - One-time schedules                                                  │
│   - Flexible time windows                                               │
│   - At-rest encryption                                                  │
│   - Dead letter queue                                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Cross-Account Events

```
Cross-Account Event Flow:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Account A (Source)           Account B (Target)                       │
│   ┌───────────────────────┐    ┌───────────────────────┐               │
│   │                       │    │                       │               │
│   │  Event Bus ───────────┼───▶│  Event Bus            │               │
│   │  (with rule targeting │    │  (with permissions)   │               │
│   │   Account B bus)      │    │                       │               │
│   │                       │    │  Rules → Targets      │               │
│   └───────────────────────┘    └───────────────────────┘               │
│                                                                          │
│   Setup:                                                                 │
│   1. Account B: Add resource policy to event bus                        │
│   2. Account A: Create rule with Account B bus as target                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

// Account B: Event Bus Policy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowAccountA",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::ACCOUNT_A:root"},
    "Action": "events:PutEvents",
    "Resource": "arn:aws:events:us-east-1:ACCOUNT_B:event-bus/my-bus"
  }]
}
```

---

## Step Functions Overview

```
Step Functions = Serverless Workflow Orchestration

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   State Machine = Workflow definition                                    │
│                                                                          │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐            │
│   │ Start   │───▶│ Task 1  │───▶│ Choice  │───▶│ Task 2  │──▶ End    │
│   └─────────┘    │(Lambda) │    │(Branch) │    │ (ECS)   │            │
│                  └─────────┘    └────┬────┘    └─────────┘            │
│                                      │                                  │
│                                      ▼                                  │
│                                 ┌─────────┐                            │
│                                 │ Task 3  │──▶ End                     │
│                                 │ (Batch) │                            │
│                                 └─────────┘                            │
│                                                                          │
│   Workflow Types:                                                        │
│   - Standard: Long-running, exactly-once, $0.025/1000 transitions      │
│   - Express: High-volume, at-least-once, $1/million executions         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## State Types

### Task State

```json
{
  "ProcessOrder": {
    "Type": "Task",
    "Resource": "arn:aws:lambda:us-east-1:123:function:ProcessOrder",
    "Parameters": {
      "orderId.$": "$.orderId",
      "staticValue": "constant"
    },
    "ResultPath": "$.processResult",
    "Retry": [
      {
        "ErrorEquals": ["States.Timeout", "Lambda.ServiceException"],
        "IntervalSeconds": 1,
        "MaxAttempts": 3,
        "BackoffRate": 2
      }
    ],
    "Catch": [
      {
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "HandleError"
      }
    ],
    "Next": "SendNotification"
  }
}
```

### Choice State

```json
{
  "CheckOrderAmount": {
    "Type": "Choice",
    "Choices": [
      {
        "Variable": "$.amount",
        "NumericGreaterThan": 1000,
        "Next": "HighValueOrder"
      },
      {
        "Variable": "$.status",
        "StringEquals": "RUSH",
        "Next": "ExpressProcessing"
      },
      {
        "And": [
          {"Variable": "$.customer.tier", "StringEquals": "premium"},
          {"Variable": "$.amount", "NumericGreaterThan": 100}
        ],
        "Next": "PremiumProcessing"
      }
    ],
    "Default": "StandardProcessing"
  }
}
```

### Parallel State

```json
{
  "ProcessInParallel": {
    "Type": "Parallel",
    "Branches": [
      {
        "StartAt": "UpdateInventory",
        "States": {
          "UpdateInventory": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:...:UpdateInventory",
            "End": true
          }
        }
      },
      {
        "StartAt": "SendNotification",
        "States": {
          "SendNotification": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:...:SendNotification",
            "End": true
          }
        }
      },
      {
        "StartAt": "UpdateAnalytics",
        "States": {
          "UpdateAnalytics": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:...:UpdateAnalytics",
            "End": true
          }
        }
      }
    ],
    "Next": "Completed"
  }
}
```

### Map State

```json
{
  "ProcessItems": {
    "Type": "Map",
    "ItemsPath": "$.orderItems",
    "ItemSelector": {
      "itemId.$": "$$.Map.Item.Value.itemId",
      "orderId.$": "$.orderId"
    },
    "MaxConcurrency": 10,
    "Iterator": {
      "StartAt": "ProcessItem",
      "States": {
        "ProcessItem": {
          "Type": "Task",
          "Resource": "arn:aws:lambda:...:ProcessItem",
          "End": true
        }
      }
    },
    "ResultPath": "$.processedItems",
    "Next": "AggregateResults"
  }
}

// Distributed Map (for large datasets)
{
  "ProcessLargeDataset": {
    "Type": "Map",
    "ItemReader": {
      "Resource": "arn:aws:states:::s3:getObject",
      "Parameters": {
        "Bucket": "my-bucket",
        "Key": "data.json"
      }
    },
    "ItemBatcher": {
      "MaxItemsPerBatch": 100
    },
    "MaxConcurrency": 1000
  }
}
```

### Wait State

```json
{
  "WaitForApproval": {
    "Type": "Wait",
    "Seconds": 3600,
    "Next": "CheckApproval"
  }
}

// Or wait until specific time
{
  "WaitUntilDeliveryTime": {
    "Type": "Wait",
    "TimestampPath": "$.deliveryTime",
    "Next": "StartDelivery"
  }
}
```

---

## Error Handling

```
Error Handling Strategies:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. Retry (automatic retry with backoff)                               │
│   {                                                                      │
│     "Retry": [{                                                         │
│       "ErrorEquals": ["States.Timeout"],                                │
│       "IntervalSeconds": 1,                                             │
│       "MaxAttempts": 3,                                                 │
│       "BackoffRate": 2                                                  │
│     }]                                                                   │
│   }                                                                      │
│                                                                          │
│   2. Catch (handle and redirect)                                        │
│   {                                                                      │
│     "Catch": [{                                                         │
│       "ErrorEquals": ["CustomError"],                                   │
│       "ResultPath": "$.error",                                          │
│       "Next": "HandleCustomError"                                       │
│     }, {                                                                 │
│       "ErrorEquals": ["States.ALL"],                                    │
│       "Next": "HandleAllErrors"                                         │
│     }]                                                                   │
│   }                                                                      │
│                                                                          │
│   Built-in Errors:                                                       │
│   - States.ALL (catch all)                                              │
│   - States.Timeout                                                      │
│   - States.TaskFailed                                                   │
│   - States.Permissions                                                  │
│   - Lambda.ServiceException                                             │
│   - Lambda.AWSLambdaException                                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Service Integrations

```
AWS SDK Integrations (200+ services):
┌─────────────────────────────────────────────────────────────────────────┐
│ Pattern              │ ARN Format                                       │
├──────────────────────┼──────────────────────────────────────────────────┤
│ Request-Response     │ arn:aws:states:::aws-sdk:service:apiAction      │
│ (wait for result)    │                                                 │
├──────────────────────┼──────────────────────────────────────────────────┤
│ Fire-and-Forget      │ arn:aws:states:::aws-sdk:service:apiAction      │
│ (return immediately) │ (without .sync suffix)                          │
└──────────────────────┴──────────────────────────────────────────────────┘

Examples:
// Invoke Lambda and wait
{
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": {
    "FunctionName": "MyFunction",
    "Payload.$": "$"
  }
}

// Start ECS task and wait for completion
{
  "Resource": "arn:aws:states:::ecs:runTask.sync",
  "Parameters": {
    "LaunchType": "FARGATE",
    "Cluster": "arn:aws:ecs:...",
    "TaskDefinition": "arn:aws:ecs:..."
  }
}

// Send message to SQS
{
  "Resource": "arn:aws:states:::sqs:sendMessage",
  "Parameters": {
    "QueueUrl": "https://sqs...",
    "MessageBody.$": "$.message"
  }
}

// DynamoDB put item
{
  "Resource": "arn:aws:states:::dynamodb:putItem",
  "Parameters": {
    "TableName": "Orders",
    "Item": {
      "orderId": {"S.$": "$.orderId"},
      "status": {"S": "PROCESSING"}
    }
  }
}
```

---

## Common Patterns

### Human Approval Workflow

```json
{
  "StartAt": "SubmitRequest",
  "States": {
    "SubmitRequest": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "CreateRequest"
      },
      "Next": "WaitForApproval"
    },
    "WaitForApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
      "Parameters": {
        "FunctionName": "SendApprovalEmail",
        "Payload": {
          "taskToken.$": "$$.Task.Token",
          "request.$": "$.request"
        }
      },
      "Next": "CheckApproval"
    },
    "CheckApproval": {
      "Type": "Choice",
      "Choices": [{
        "Variable": "$.approved",
        "BooleanEquals": true,
        "Next": "ProcessRequest"
      }],
      "Default": "RequestDenied"
    }
  }
}

// Callback to resume workflow
await stepfunctions.sendTaskSuccess({
  taskToken: token,
  output: JSON.stringify({ approved: true })
});
```

### Saga Pattern

```json
{
  "StartAt": "ReserveInventory",
  "States": {
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:ReserveInventory",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "FailWorkflow"
      }],
      "Next": "ProcessPayment"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:ProcessPayment",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "ReleaseInventory"
      }],
      "Next": "ShipOrder"
    },
    "ShipOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:ShipOrder",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "RefundPayment"
      }],
      "End": true
    },
    "RefundPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:RefundPayment",
      "Next": "ReleaseInventory"
    },
    "ReleaseInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:ReleaseInventory",
      "Next": "FailWorkflow"
    },
    "FailWorkflow": {
      "Type": "Fail",
      "Error": "WorkflowFailed",
      "Cause": "Compensating transactions completed"
    }
  }
}
```

---

## Interview Discussion Points

### When to use Step Functions vs Lambda orchestration?

```
Use Step Functions When:
- Complex workflows with branching
- Long-running processes (>15 min)
- Need visual workflow monitoring
- Human approval steps
- Retries and error handling at workflow level
- State machine logic (wait, parallel, map)

Use Lambda Orchestration When:
- Simple sequential calls
- Short duration (<15 min total)
- Lower cost requirement
- Simple error handling

Cost Comparison:
- Step Functions: $0.025/1000 state transitions
- Lambda: Only pay for compute time
- Break-even: ~4000 transitions = 1 hour Lambda

Decision Flow:
if (workflow.hasBranching || workflow.duration > 15min) {
  use StepFunctions;
} else if (workflow.steps < 3 && workflow.isSimple) {
  use LambdaOrchestration;
} else {
  use StepFunctions;  // Default for clarity
}
```

### How do you handle long-running workflows?

```
Patterns:

1. Step Functions Standard
   - Up to 1 year execution
   - Exactly-once processing
   - Audit history

2. Wait State
   - Pause for time-based delays
   - No compute during wait
   - Continue from where left off

3. Callback Pattern (.waitForTaskToken)
   - Pause until external signal
   - Human approval
   - External system completion

4. Activity Workers
   - Custom task implementation
   - Heartbeat for long tasks
   - Worker polls for work

5. Checkpoint Pattern
   - Save state to DynamoDB
   - Resume from checkpoint
   - Handle workflow restarts

Implementation Example:
- Order fulfillment: Standard workflow
- Wait for shipping: Wait state (24 hours)
- Customer approval: Callback pattern
- Large file processing: Activity worker
```

### EventBridge vs SNS for event-driven architecture?

```
Choose EventBridge When:
- Event routing/filtering needed
- AWS service events (native integration)
- Cross-account event sharing
- Schema registry needed
- Archive and replay events
- Scheduled events

Choose SNS When:
- Simple fanout to multiple subscribers
- Need SMS/Email notifications
- Lower latency requirement
- Higher throughput needed
- Simple pub/sub pattern

Hybrid Pattern:
- EventBridge for routing and filtering
- SNS+SQS for fanout to consumers
- Best of both worlds

Cost:
- EventBridge: $1/million events
- SNS: $0.50/million notifications
```
