# CloudWatch & Observability

## CloudWatch Overview

```
CloudWatch = Unified monitoring and observability

┌─────────────────────────────────────────────────────────────────────────┐
│                         CloudWatch Pillars                               │
├─────────────────┬─────────────────┬─────────────────┬───────────────────┤
│    Metrics      │      Logs       │     Alarms      │     Dashboards    │
│                 │                 │                 │                   │
│  - AWS services │  - Application  │  - Thresholds   │  - Visualization  │
│  - Custom       │  - CloudTrail   │  - Anomaly      │  - Cross-account  │
│  - High-res     │  - VPC Flow     │  - Composite    │  - Auto-refresh   │
└─────────────────┴─────────────────┴─────────────────┴───────────────────┘
```

---

## CloudWatch Metrics

### Metric Concepts

```
Metric Structure:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Namespace:    AWS/EC2                                                 │
│   MetricName:   CPUUtilization                                          │
│   Dimensions:   InstanceId=i-12345, AutoScalingGroupName=my-asg         │
│   Timestamp:    2024-01-15T12:30:00Z                                    │
│   Value:        45.5                                                    │
│   Unit:         Percent                                                 │
│                                                                          │
│   Dimensions = Key-value pairs that identify metric uniquely            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Retention:
- 1 second: 3 hours
- 1 minute: 15 days
- 5 minutes: 63 days
- 1 hour: 15 months

Resolution:
- Standard: 1 minute (default)
- High-resolution: 1 second (custom metrics)
```

### Common AWS Metrics

```
EC2 Metrics (Basic - 5 min, Detailed - 1 min):
- CPUUtilization
- NetworkIn/Out
- DiskReadOps/WriteOps
- StatusCheckFailed

NOT included (need Agent):
- Memory utilization
- Disk space
- Custom application metrics

RDS Metrics:
- CPUUtilization
- DatabaseConnections
- FreeableMemory
- ReadIOPS/WriteIOPS
- ReadLatency/WriteLatency

Lambda Metrics:
- Invocations
- Duration
- Errors
- Throttles
- ConcurrentExecutions
- IteratorAge (for streams)

ALB Metrics:
- RequestCount
- TargetResponseTime
- HTTPCode_Target_2XX/4XX/5XX
- HealthyHostCount
- ActiveConnectionCount
```

### Custom Metrics

```javascript
const { CloudWatchClient, PutMetricDataCommand } = require('@aws-sdk/client-cloudwatch');
const cloudwatch = new CloudWatchClient();

// Put custom metric
await cloudwatch.send(new PutMetricDataCommand({
  Namespace: 'MyApplication',
  MetricData: [
    {
      MetricName: 'OrdersProcessed',
      Dimensions: [
        { Name: 'Environment', Value: 'production' },
        { Name: 'Service', Value: 'order-service' }
      ],
      Value: 150,
      Unit: 'Count',
      Timestamp: new Date()
    },
    {
      MetricName: 'ProcessingTime',
      Dimensions: [
        { Name: 'Environment', Value: 'production' }
      ],
      StatisticValues: {
        SampleCount: 100,
        Sum: 5000,
        Minimum: 10,
        Maximum: 200
      },
      Unit: 'Milliseconds'
    }
  ]
}));

// High-resolution metric (1 second)
{
  StorageResolution: 1,  // 1 = high-res, 60 = standard
  ...
}
```

---

## CloudWatch Logs

### Log Structure

```
Log Hierarchy:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Log Group: /aws/lambda/my-function                                    │
│   │                                                                      │
│   ├── Log Stream: 2024/01/15/[$LATEST]abc123                           │
│   │   ├── Log Event: {"timestamp": ..., "message": "Processing..."}    │
│   │   ├── Log Event: {"timestamp": ..., "message": "Completed"}        │
│   │   └── ...                                                           │
│   │                                                                      │
│   ├── Log Stream: 2024/01/15/[$LATEST]def456                           │
│   │   └── ...                                                           │
│   │                                                                      │
│   └── Retention: 1 day - 10 years (or never expire)                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Log Insights

```sql
-- Query syntax
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

-- Parse and aggregate
fields @timestamp, @message
| parse @message "user=* action=* duration=*" as user, action, duration
| filter action = "login"
| stats avg(duration) as avg_duration by user
| sort avg_duration desc

-- Error analysis
fields @timestamp, @message
| filter @message like /Exception/
| stats count(*) as error_count by bin(1h)
| sort @timestamp

-- Lambda cold start analysis
fields @timestamp, @message, @duration, @billedDuration
| filter @type = "REPORT"
| filter @message like /Init Duration/
| parse @message "Init Duration: * ms" as init_duration
| stats avg(init_duration), max(init_duration), count(*) by bin(1h)
```

### Metric Filters

```
Create Metrics from Logs:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Log Group ──filter pattern──▶ CloudWatch Metric                       │
│                                                                          │
│   Filter Patterns:                                                       │
│   - Exact match: "ERROR"                                                │
│   - Multiple terms: "ERROR" "database"                                  │
│   - JSON: { $.level = "ERROR" }                                         │
│   - Space-delimited: [ip, user, timestamp, request, status=5*, size]   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

// Create metric filter
aws logs put-metric-filter \
  --log-group-name /aws/lambda/my-function \
  --filter-name ErrorCount \
  --filter-pattern "ERROR" \
  --metric-transformations \
    metricName=Errors,metricNamespace=MyApp,metricValue=1
```

### Log Subscriptions

```
Export logs in real-time:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Log Group ────subscription────▶ Kinesis Data Firehose ──▶ S3         │
│                                                                          │
│   Log Group ────subscription────▶ Kinesis Data Streams ──▶ Lambda      │
│                                                                          │
│   Log Group ────subscription────▶ Lambda ──▶ ElasticSearch            │
│                                                                          │
│   Log Group ────subscription────▶ OpenSearch (direct)                  │
│                                                                          │
│   Cross-Account:                                                         │
│   Account A Log Group ──▶ Account B Kinesis/Lambda                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## CloudWatch Alarms

### Alarm States

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   ┌─────────┐        ┌─────────┐        ┌─────────────────┐            │
│   │   OK    │◀──────▶│ ALARM   │◀──────▶│ INSUFFICIENT    │            │
│   │         │        │         │        │ DATA            │            │
│   └─────────┘        └─────────┘        └─────────────────┘            │
│                                                                          │
│   Transition triggers actions (SNS, Auto Scaling, EC2, etc.)            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Alarm Configuration

```javascript
// Standard threshold alarm
{
  AlarmName: 'High-CPU-Alarm',
  MetricName: 'CPUUtilization',
  Namespace: 'AWS/EC2',
  Dimensions: [{ Name: 'InstanceId', Value: 'i-12345' }],
  Statistic: 'Average',
  Period: 300,  // 5 minutes
  EvaluationPeriods: 3,  // 3 consecutive periods
  Threshold: 80,
  ComparisonOperator: 'GreaterThanThreshold',
  AlarmActions: ['arn:aws:sns:...:alerts'],
  OKActions: ['arn:aws:sns:...:alerts']
}

// Anomaly detection alarm
{
  AlarmName: 'Anomaly-Detection',
  Metrics: [{
    Id: 'm1',
    MetricStat: {
      Metric: {
        MetricName: 'RequestCount',
        Namespace: 'AWS/ApplicationELB'
      },
      Period: 60,
      Stat: 'Sum'
    }
  }, {
    Id: 'ad1',
    Expression: 'ANOMALY_DETECTION_BAND(m1, 2)'
  }],
  ThresholdMetricId: 'ad1',
  ComparisonOperator: 'LessThanLowerOrGreaterThanUpperThreshold'
}
```

### Composite Alarms

```
Composite = Combine multiple alarms with AND/OR logic

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Composite Alarm: Service-Health                                        │
│   Rule: ALARM(CPU-High) AND (ALARM(Memory-High) OR ALARM(Errors-High))  │
│                                                                          │
│         ┌─────────────┐                                                 │
│         │ CPU-High    │─────┐                                           │
│         │ ALARM       │     │                                           │
│         └─────────────┘     │     ┌─────────────────┐                   │
│                             AND───│ Service-Health  │                   │
│   ┌─────────────┐           │     │ ALARM           │                   │
│   │ Memory-High │───┐       │     └─────────────────┘                   │
│   │ ALARM       │   OR──────┘                                           │
│   └─────────────┘   │                                                   │
│   ┌─────────────┐   │                                                   │
│   │ Errors-High │───┘                                                   │
│   │ OK          │                                                       │
│   └─────────────┘                                                       │
│                                                                          │
│   Benefits:                                                              │
│   - Reduce alarm noise                                                  │
│   - More accurate alerting                                              │
│   - Complex conditions                                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## X-Ray Tracing

```
X-Ray = Distributed tracing service

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Request Flow:                                                          │
│                                                                          │
│   Client ──▶ API Gateway ──▶ Lambda ──▶ DynamoDB                       │
│      │           │             │            │                           │
│      └───────────┴─────────────┴────────────┘                           │
│                        │                                                 │
│                  X-Ray Trace                                            │
│                        │                                                 │
│               ┌────────┴────────┐                                       │
│               │                 │                                       │
│            Segments         Subsegments                                 │
│            (services)       (calls within)                              │
│                                                                          │
│   Trace ID: 1-58406520-a006649127e371903a2de979                        │
│   Segment: API Gateway → Lambda → DynamoDB                              │
│   Annotations: userId=123, orderId=456                                  │
│   Metadata: request payload, response                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### X-Ray Integration

```javascript
const AWSXRay = require('aws-xray-sdk');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));  // Auto-instrument AWS calls

// Custom subsegment
exports.handler = async (event) => {
  const segment = AWSXRay.getSegment();

  // Add annotations (indexed, searchable)
  segment.addAnnotation('userId', event.userId);
  segment.addAnnotation('orderType', 'premium');

  // Add metadata (not indexed)
  segment.addMetadata('request', event);

  // Custom subsegment for business logic
  const subsegment = segment.addNewSubsegment('ProcessOrder');
  try {
    const result = await processOrder(event);
    subsegment.addMetadata('result', result);
    return result;
  } catch (error) {
    subsegment.addError(error);
    throw error;
  } finally {
    subsegment.close();
  }
};
```

### Service Map

```
X-Ray Service Map shows:
- Service dependencies
- Latency distribution
- Error rates
- Throughput

Analysis Features:
- Filter by annotation
- Compare time periods
- Trace analytics (query traces)
- Insights (anomaly detection)
```

---

## CloudTrail

```
CloudTrail = API activity logging

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Event Types:                                                           │
│                                                                          │
│   Management Events (default):                                           │
│   - API calls that modify resources                                     │
│   - Console sign-ins                                                    │
│   - Examples: RunInstances, CreateBucket, CreateUser                    │
│                                                                          │
│   Data Events (optional, additional cost):                              │
│   - S3 object-level operations (GetObject, PutObject)                  │
│   - Lambda function invocations                                         │
│   - DynamoDB item operations                                            │
│                                                                          │
│   Insights Events (optional):                                            │
│   - Unusual API activity detection                                      │
│   - Anomaly in call volume                                              │
│                                                                          │
│   Storage:                                                               │
│   - 90 days in CloudTrail console (free)                                │
│   - S3 bucket for longer retention                                      │
│   - CloudWatch Logs for real-time analysis                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### CloudTrail Event

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "AIDAEXAMPLE",
    "arn": "arn:aws:iam::123456789012:user/alice",
    "accountId": "123456789012",
    "userName": "alice"
  },
  "eventTime": "2024-01-15T12:30:00Z",
  "eventSource": "ec2.amazonaws.com",
  "eventName": "RunInstances",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "192.0.2.1",
  "userAgent": "aws-cli/2.0",
  "requestParameters": {
    "instanceType": "t3.micro",
    "imageId": "ami-12345678"
  },
  "responseElements": {
    "instancesSet": {
      "items": [{"instanceId": "i-abcdef123"}]
    }
  }
}
```

---

## Observability Best Practices

### The Three Pillars

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│        Metrics              Logs              Traces                    │
│          │                   │                  │                       │
│          │                   │                  │                       │
│     "What's              "What                "Why is                  │
│      happening?"          happened?"          it slow?"                 │
│                                                                          │
│   - System health        - Event details      - Request flow           │
│   - Performance          - Debugging          - Dependencies           │
│   - Trends               - Audit              - Bottlenecks            │
│   - Alerting             - Forensics          - Latency breakdown      │
│                                                                          │
│   Tools:                 Tools:               Tools:                    │
│   - CloudWatch Metrics   - CloudWatch Logs    - X-Ray                  │
│   - Container Insights   - OpenSearch         - ServiceLens            │
│   - Lambda Insights      - S3 + Athena        │                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Monitoring Strategy

```
1. Define SLIs/SLOs
   - Latency: p99 < 200ms
   - Availability: 99.9%
   - Error rate: < 0.1%

2. Build Dashboards
   - Service overview
   - Customer-facing metrics
   - Infrastructure health

3. Set Meaningful Alarms
   - Alert on symptoms, not causes
   - Use composite alarms
   - Avoid alert fatigue

4. Implement Tracing
   - Instrument all services
   - Add business context (annotations)
   - Sample appropriately

5. Structured Logging
   - JSON format
   - Correlation IDs
   - Log levels (DEBUG, INFO, ERROR)

6. Runbooks
   - Document investigation steps
   - Automate where possible
   - Include remediation
```

---

## Interview Discussion Points

### How do you approach monitoring a new service?

```
Step-by-Step:

1. Identify Key Metrics
   - Request rate (throughput)
   - Error rate
   - Latency (p50, p95, p99)
   - Resource utilization

2. Define SLIs/SLOs
   - What does "healthy" look like?
   - User-facing metrics first
   - Internal metrics second

3. Instrument Application
   - Structured logging
   - Custom metrics
   - Distributed tracing

4. Create Dashboards
   - Golden signals (rate, errors, latency, saturation)
   - Drill-down capability
   - Business metrics

5. Set Up Alerts
   - Start with few critical alerts
   - Based on SLOs
   - Include runbooks

6. Iterate
   - Review alert fatigue
   - Add metrics as needed
   - Update runbooks
```

### How do you troubleshoot a production issue?

```
Systematic Approach:

1. Assess Impact
   - Which users/services affected?
   - What's the business impact?
   - How long has it been happening?

2. Check Recent Changes
   - Deployments?
   - Configuration changes?
   - Infrastructure changes?

3. Review Metrics
   - Dashboard for anomalies
   - Compare to baseline
   - Identify correlations

4. Examine Logs
   - Filter for errors
   - Look for patterns
   - Check timestamps

5. Trace Requests
   - X-Ray for slow requests
   - Identify bottlenecks
   - Check dependencies

6. Correlate Data
   - CloudTrail for API calls
   - VPC Flow Logs for network
   - Resource changes

7. Document & Fix
   - Root cause analysis
   - Implement fix
   - Update runbooks/alerts
```

### How do you reduce CloudWatch costs?

```
Cost Optimization:

1. Log Retention
   - Set appropriate retention periods
   - Archive to S3 Glacier
   - Delete unnecessary logs

2. Metric Resolution
   - Standard (1 min) vs High-res (1 sec)
   - High-res only when needed

3. Log Volume
   - Adjust log levels in production
   - Sample verbose logs
   - Filter before ingestion

4. Efficient Queries
   - Narrow time ranges
   - Use filters early
   - Avoid SELECT *

5. Dashboard Optimization
   - Reduce auto-refresh frequency
   - Use metric math instead of multiple queries
   - Cache dashboard data

6. Alternative Storage
   - Export to S3 for long-term
   - Use Athena for analysis
   - Consider open-source alternatives
```
