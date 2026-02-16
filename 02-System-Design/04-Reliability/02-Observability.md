# Observability

## Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────┐
│                      Observability                           │
├─────────────────┬─────────────────┬─────────────────────────┤
│     Logs        │     Metrics     │      Traces             │
│                 │                 │                         │
│ What happened   │ How is system   │ How requests flow       │
│ (events)        │ performing?     │ through system?         │
│                 │ (aggregates)    │                         │
│ DEBUG, ERROR    │ CPU: 75%        │ Request → A → B → C     │
│ messages        │ Latency p99     │ with timing             │
└─────────────────┴─────────────────┴─────────────────────────┘
```

---

## Logging

### Structured Logging

```javascript
// Bad: Unstructured logs
console.log('User 123 created order 456 for $99.99');

// Good: Structured logs (JSON)
const logger = require('pino')();

logger.info({
  event: 'order_created',
  userId: '123',
  orderId: '456',
  amount: 99.99,
  currency: 'USD'
});

// Output:
// {"level":"info","event":"order_created","userId":"123","orderId":"456","amount":99.99,"currency":"USD","time":1234567890}
```

### Log Levels

```javascript
const logger = require('winston').createLogger({
  level: process.env.LOG_LEVEL || 'info'
});

// Severity levels (most to least severe)
logger.error('Database connection failed', { error: err });
logger.warn('Cache miss rate high', { rate: 0.8 });
logger.info('Order created', { orderId: '123' });
logger.debug('Query executed', { sql: query, time: 45 });
logger.trace('Function entered', { fn: 'processOrder' });

// In production: info and above
// In debugging: debug and above
```

### Correlation IDs

```javascript
const { v4: uuid } = require('uuid');

// Middleware to add correlation ID
app.use((req, res, next) => {
  req.correlationId = req.headers['x-correlation-id'] || uuid();
  res.set('x-correlation-id', req.correlationId);
  next();
});

// Include in all logs
app.use((req, res, next) => {
  req.logger = logger.child({ correlationId: req.correlationId });
  next();
});

// Use in handlers
app.get('/orders/:id', async (req, res) => {
  req.logger.info('Fetching order', { orderId: req.params.id });
  // ...
});

// Pass to downstream services
async function callUserService(userId, correlationId) {
  return fetch(`http://user-service/users/${userId}`, {
    headers: { 'x-correlation-id': correlationId }
  });
}
```

### Centralized Logging (ELK Stack)

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  Service A  │──▶│             │   │             │
└─────────────┘   │  Logstash   │──▶│Elasticsearch│
┌─────────────┐   │ (collector) │   │  (storage)  │
│  Service B  │──▶│             │   │             │
└─────────────┘   └─────────────┘   └──────┬──────┘
┌─────────────┐                           │
│  Service C  │──▶                        ▼
└─────────────┘                    ┌─────────────┐
                                   │   Kibana    │
                                   │ (dashboard) │
                                   └─────────────┘
```

---

## Metrics

### Types of Metrics

```javascript
const prometheus = require('prom-client');

// 1. Counter - Only increases (requests, errors)
const requestCounter = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status']
});

requestCounter.inc({ method: 'GET', path: '/api/users', status: '200' });

// 2. Gauge - Can increase or decrease (concurrent connections, queue size)
const queueSize = new prometheus.Gauge({
  name: 'queue_size',
  help: 'Current queue size'
});

queueSize.set(42);
queueSize.inc();
queueSize.dec(5);

// 3. Histogram - Distribution of values (latency)
const requestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Request duration in seconds',
  labelNames: ['method', 'path'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5]
});

const end = requestDuration.startTimer({ method: 'GET', path: '/api' });
// ... process request
end(); // Records duration

// 4. Summary - Similar to histogram with percentiles
const requestSummary = new prometheus.Summary({
  name: 'http_request_duration_summary',
  help: 'Request duration summary',
  percentiles: [0.5, 0.9, 0.99]
});
```

### Key Metrics (USE & RED)

```
USE Method (for resources):
U - Utilization: % time resource is busy
S - Saturation: Amount of work queued
E - Errors: Count of error events

RED Method (for services):
R - Rate: Requests per second
E - Errors: Failed requests per second
D - Duration: Latency distribution

Example Prometheus queries:

# Request rate
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m]) /
rate(http_requests_total[5m])

# Latency p99
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# CPU utilization
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Prometheus + Grafana

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'api-service'
    static_configs:
      - targets: ['api:3000']

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
```

```javascript
// Expose metrics endpoint
const express = require('express');
const prometheus = require('prom-client');

prometheus.collectDefaultMetrics();

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(await prometheus.register.metrics());
});
```

---

## Distributed Tracing

### Trace Concepts

```
Trace: Full journey of a request through the system
Span: Single operation within a trace

Request Flow:
┌─────────────────────────────────────────────────────────────┐
│ Trace ID: abc-123                                           │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Span: API Gateway (50ms)                                │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Span: Order Service (40ms)                          │ │ │
│ │ │ ┌───────────────────────┐ ┌───────────────────────┐ │ │ │
│ │ │ │ Span: DB Query (10ms) │ │ Span: Payment (20ms)  │ │ │ │
│ │ │ └───────────────────────┘ └───────────────────────┘ │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### OpenTelemetry Implementation

```javascript
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { SimpleSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');

// Initialize tracing
const provider = new NodeTracerProvider();
provider.addSpanProcessor(new SimpleSpanProcessor(
  new JaegerExporter({ endpoint: 'http://jaeger:14268/api/traces' })
));
provider.register();

// Auto-instrument HTTP and Express
registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation()
  ]
});

// Manual span creation
const tracer = require('@opentelemetry/api').trace.getTracer('order-service');

async function processOrder(order) {
  const span = tracer.startSpan('processOrder');

  try {
    span.setAttribute('orderId', order.id);
    span.setAttribute('userId', order.userId);

    // Child span for database
    const dbSpan = tracer.startSpan('db.query', { parent: span });
    const result = await db.query('INSERT INTO orders...');
    dbSpan.end();

    span.setStatus({ code: SpanStatusCode.OK });
    return result;
  } catch (error) {
    span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
    span.recordException(error);
    throw error;
  } finally {
    span.end();
  }
}
```

### Context Propagation

```javascript
// HTTP headers carry trace context
// W3C Trace Context format:
// traceparent: 00-{trace-id}-{span-id}-{flags}

// Automatic with instrumentation libraries
fetch('http://payment-service/charge', {
  headers: {
    // Trace context automatically injected
    'content-type': 'application/json'
  }
});
```

---

## Alerting

### Alert Design Principles

```yaml
# Good alerts:
# 1. Actionable - Someone can do something about it
# 2. Urgent - Requires immediate attention
# 3. Relevant - Points to actual user impact

# Alerting rules (Prometheus)
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) /
          sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "p95 latency is {{ $value }}s"

      - alert: ServiceDown
        expr: up{job="api-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
```

### Alert Fatigue Prevention

```
Symptoms of bad alerting:
- Too many alerts → Ignored
- Non-actionable alerts → Frustration
- Flapping alerts → Noise

Solutions:
1. Set appropriate thresholds
2. Use "for" duration to avoid transient spikes
3. Group related alerts
4. Define clear severity levels
5. Regular review and tuning
```

---

## Dashboards

### Key Dashboard Principles

```
1. SLO Dashboard (Executive view)
   - Are we meeting our promises?
   - Error budget remaining
   - Availability this month

2. Service Dashboard (Team view)
   - Request rate
   - Error rate
   - Latency percentiles
   - Resource utilization
   - Dependencies status

3. Debugging Dashboard (Incident view)
   - Recent errors
   - Slow queries
   - Resource bottlenecks
   - Correlation with deploys
```

### Example Grafana Dashboard

```javascript
// Dashboard JSON (simplified)
{
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "targets": [{
        "expr": "sum(rate(http_requests_total[5m]))"
      }]
    },
    {
      "title": "Error Rate",
      "type": "singlestat",
      "targets": [{
        "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))"
      }],
      "thresholds": "0.01,0.05",
      "colors": ["green", "yellow", "red"]
    },
    {
      "title": "Latency Distribution",
      "type": "heatmap",
      "targets": [{
        "expr": "sum(rate(http_request_duration_seconds_bucket[5m])) by (le)"
      }]
    }
  ]
}
```

---

## Debugging Production Issues

### Debugging Workflow

```
1. DETECT
   Alert fires or user reports issue

2. TRIAGE
   - What's the impact? (users affected, revenue impact)
   - What changed? (deploys, config, traffic)

3. INVESTIGATE
   Metrics → Logs → Traces

   a. Check dashboards for anomalies
   b. Find error logs with correlation ID
   c. Trace request through services

4. MITIGATE
   - Rollback if deployment-related
   - Scale if capacity-related
   - Feature flag if code-related

5. RESOLVE
   - Fix root cause
   - Add monitoring for future
   - Document in postmortem
```

### Useful Queries

```
# Find errors in last hour
grep -i error /var/log/app/*.log | tail -100

# Elasticsearch query
{
  "query": {
    "bool": {
      "must": [
        { "match": { "level": "error" }},
        { "range": { "@timestamp": { "gte": "now-1h" }}}
      ]
    }
  }
}

# Find slow traces in Jaeger
Service: order-service
Operation: createOrder
Duration: >1s
```

---

## Interview Questions

### Q: How do you debug a slow API endpoint in production?

1. **Check metrics** - Is latency high for all requests or specific paths?
2. **Check traces** - Where is time being spent? (DB, external calls)
3. **Check logs** - Any errors or warnings?
4. **Check resources** - CPU, memory, connections
5. **Check dependencies** - Are downstream services slow?
6. **Check recent changes** - Any deploys or config changes?

### Q: What metrics would you monitor for a new service?

**Infrastructure:**
- CPU, memory, disk usage
- Network I/O

**Application:**
- Request rate (RPS)
- Error rate (4xx, 5xx)
- Latency (p50, p95, p99)
- Queue depths
- Connection pools

**Business:**
- Orders processed
- Revenue
- User signups

### Q: How do you set up alerting without alert fatigue?

1. **Start with SLOs** - Alert when breaching objectives
2. **Symptoms over causes** - Alert on user impact, not CPU
3. **Use thresholds wisely** - Not too sensitive
4. **Add duration** - Avoid transient spikes
5. **Prioritize** - Critical vs warning vs info
6. **Review regularly** - Tune or delete noisy alerts
