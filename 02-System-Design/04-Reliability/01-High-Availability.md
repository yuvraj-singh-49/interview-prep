# High Availability

## Availability Metrics

### Measuring Availability

```
Availability = Uptime / (Uptime + Downtime)

Example:
- 99% availability = 3.65 days downtime/year
- 99.9% = 8.76 hours/year
- 99.99% = 52.6 minutes/year
- 99.999% = 5.26 minutes/year

"Nines" Reference:
┌────────────┬──────────────────┬─────────────────┐
│ Nines      │ Downtime/Year    │ Downtime/Month  │
├────────────┼──────────────────┼─────────────────┤
│ 99%        │ 3.65 days        │ 7.3 hours       │
│ 99.9%      │ 8.76 hours       │ 43.8 minutes    │
│ 99.99%     │ 52.6 minutes     │ 4.38 minutes    │
│ 99.999%    │ 5.26 minutes     │ 26 seconds      │
└────────────┴──────────────────┴─────────────────┘
```

### SLA, SLO, SLI

```
SLI (Service Level Indicator):
- Metric you measure
- e.g., "Response time", "Error rate", "Availability"

SLO (Service Level Objective):
- Target for your SLI
- e.g., "99.9% of requests succeed"
- e.g., "p99 latency < 200ms"

SLA (Service Level Agreement):
- Contract with consequences
- e.g., "99.9% uptime or we refund 10%"
```

---

## Redundancy Patterns

### Active-Passive (Failover)

```
┌─────────────┐         ┌─────────────┐
│   Active    │         │   Passive   │
│  (Primary)  │────────▶│  (Standby)  │
│             │ heartbeat│             │
└──────┬──────┘         └──────┬──────┘
       │                       │
       ▼                       │
   Traffic                     │
                               ▼
                   (Takes over on failure)

Pros:
- Simple
- Cost effective (standby does less work)

Cons:
- Failover time (minutes)
- Standby resource waste
```

### Active-Active

```
┌─────────────┐         ┌─────────────┐
│   Active    │◀───────▶│   Active    │
│  (Node 1)   │  sync   │  (Node 2)   │
└──────┬──────┘         └──────┬──────┘
       │                       │
       └───────────┬───────────┘
                   │
              Load Balancer
                   │
               Traffic

Pros:
- No failover time
- Better resource utilization
- Higher capacity

Cons:
- More complex
- Data synchronization challenges
- Higher cost
```

### N+1 Redundancy

```
Traffic requires N servers
Deploy N+1 servers

Example:
- Need 4 servers at peak load
- Deploy 5 servers
- Any 1 can fail without impact
```

### Geographic Redundancy

```
                    ┌──────────────────┐
                    │    Global DNS    │
                    │  (Route 53, etc) │
                    └────────┬─────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   US-East     │   │   EU-West     │   │   AP-South    │
│   Region      │   │   Region      │   │   Region      │
│               │   │               │   │               │
│  App + DB     │   │  App + DB     │   │  App + DB     │
└───────────────┘   └───────────────┘   └───────────────┘
```

---

## Failure Modes

### Types of Failures

```
1. Crash Failures
   - Server dies
   - Process terminates
   - Solution: Redundancy, auto-restart

2. Omission Failures
   - Network packet dropped
   - Timeout
   - Solution: Retries, redundant paths

3. Timing Failures
   - Response too slow
   - Solution: Timeouts, caching, scaling

4. Byzantine Failures
   - Incorrect/malicious responses
   - Solution: Validation, consensus protocols
```

### Failure Domains

```
┌─────────────────────────────────────────────────────────────┐
│ Region (US-East-1)                                          │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Availability Zone (us-east-1a)                          │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Rack                                                │ │ │
│ │ │ ┌───────────┐ ┌───────────┐ ┌───────────┐          │ │ │
│ │ │ │  Server   │ │  Server   │ │  Server   │          │ │ │
│ │ │ └───────────┘ └───────────┘ └───────────┘          │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Availability Zone (us-east-1b)                          │ │
│ │                      ...                                │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

Design for:
- Single server failure: Basic redundancy
- Rack failure: Spread across racks
- AZ failure: Multi-AZ deployment
- Region failure: Multi-region deployment
```

---

## High Availability Architecture

### Stateless Services

```javascript
// Stateless = Easier HA
// Any instance can handle any request

// Bad: State in memory
let sessionData = {};
app.get('/data', (req, res) => {
  res.json(sessionData[req.userId]);
});

// Good: State externalized
app.get('/data', async (req, res) => {
  const data = await redis.get(`session:${req.userId}`);
  res.json(data);
});
```

### Database HA

```
PostgreSQL HA Setup:

┌─────────────────────────────────────────────────────────────┐
│                        PgBouncer                             │
│                     (Connection Pool)                        │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ Primary  │──▶│ Replica  │──▶│ Replica  │
        │ (Write)  │   │ (Read)   │   │ (Read)   │
        └──────────┘   └──────────┘   └──────────┘
              │
              ▼
        ┌──────────┐
        │ Patroni  │ (Automatic failover)
        └──────────┘
```

### Message Queue HA

```
RabbitMQ Cluster:

┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Node 1    │───▶│   Node 2    │───▶│   Node 3    │
│  (Mirror)   │◀───│  (Mirror)   │◀───│  (Mirror)   │
└─────────────┘    └─────────────┘    └─────────────┘

Messages mirrored across nodes
Any node failure: Others continue
```

---

## Health Checks

### Types of Health Checks

```javascript
// 1. Liveness - Is the process alive?
app.get('/health/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

// 2. Readiness - Can it handle traffic?
app.get('/health/ready', async (req, res) => {
  try {
    await db.query('SELECT 1');
    await redis.ping();
    res.status(200).json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});

// 3. Startup - Has it finished initializing?
let started = false;
app.get('/health/startup', (req, res) => {
  if (started) {
    res.status(200).json({ status: 'started' });
  } else {
    res.status(503).json({ status: 'starting' });
  }
});
```

### Kubernetes Health Checks

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
      failureThreshold: 3

    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5

    startupProbe:
      httpGet:
        path: /health/startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

---

## Graceful Degradation

### Feature Flags

```javascript
const featureFlags = {
  'recommendations': true,
  'advanced-search': true,
  'social-features': false  // Degraded
};

app.get('/product/:id', async (req, res) => {
  const product = await getProduct(req.params.id);

  let recommendations = [];
  if (featureFlags['recommendations']) {
    try {
      recommendations = await getRecommendations(product.id);
    } catch (error) {
      // Gracefully degrade
      console.error('Recommendations failed', error);
    }
  }

  res.json({ product, recommendations });
});
```

### Fallbacks

```javascript
async function getUserData(userId) {
  try {
    // Primary source
    return await userService.getUser(userId);
  } catch (error) {
    try {
      // Fallback to cache
      const cached = await cache.get(`user:${userId}`);
      if (cached) return JSON.parse(cached);
    } catch (cacheError) {
      console.error('Cache fallback failed');
    }

    // Ultimate fallback
    return {
      id: userId,
      name: 'Unknown User',
      status: 'degraded'
    };
  }
}
```

---

## Load Shedding

### Priority-Based

```javascript
class LoadShedder {
  constructor(maxLoad) {
    this.maxLoad = maxLoad;
    this.currentLoad = 0;
  }

  async process(request) {
    if (this.currentLoad >= this.maxLoad) {
      // Shed low priority requests
      if (request.priority === 'low') {
        throw new Error('Service overloaded');
      }
    }

    this.currentLoad++;
    try {
      return await this.handleRequest(request);
    } finally {
      this.currentLoad--;
    }
  }
}

// Priority assignment
app.use((req, res, next) => {
  if (req.path.includes('/critical/')) {
    req.priority = 'high';
  } else if (req.path.includes('/api/')) {
    req.priority = 'medium';
  } else {
    req.priority = 'low';
  }
  next();
});
```

### Admission Control

```javascript
// Reject requests early if overloaded
app.use((req, res, next) => {
  const cpuUsage = os.loadavg()[0] / os.cpus().length;

  if (cpuUsage > 0.9) {
    // 90% CPU - shed traffic
    if (Math.random() > 0.5) {
      return res.status(503).json({
        error: 'Service temporarily unavailable',
        retryAfter: 5
      });
    }
  }

  next();
});
```

---

## Zero Downtime Deployments

### Rolling Deployment

```
Time 0:  [v1] [v1] [v1] [v1]  ← 4 instances running v1
Time 1:  [v2] [v1] [v1] [v1]  ← Deploy v2 to 1st instance
Time 2:  [v2] [v2] [v1] [v1]  ← Deploy v2 to 2nd instance
Time 3:  [v2] [v2] [v2] [v1]  ← Deploy v2 to 3rd instance
Time 4:  [v2] [v2] [v2] [v2]  ← Complete
```

### Blue-Green

```
Blue (Current):  [v1] [v1] [v1] ← Serving traffic
Green (New):     [v2] [v2] [v2] ← Ready, not serving

DNS/LB Switch:
Blue:            [v1] [v1] [v1] ← Idle (rollback option)
Green:           [v2] [v2] [v2] ← Now serving traffic
```

### Canary

```
                    ┌─────────────┐
                    │ Load        │
                    │ Balancer    │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │ 95%                10%  │
              ▼                         ▼
        ┌──────────┐             ┌──────────┐
        │ Stable   │             │ Canary   │
        │   v1     │             │   v2     │
        └──────────┘             └──────────┘

Monitor canary metrics
If healthy: gradually increase canary traffic
If issues: route all traffic back to stable
```

---

## Interview Questions

### Q: How do you achieve 99.99% availability?

1. **Multi-AZ deployment** - Survive AZ failures
2. **No single points of failure** - Redundancy everywhere
3. **Automated failover** - < 1 minute recovery
4. **Health checks** - Quick failure detection
5. **Load balancing** - Distribute traffic
6. **Stateless services** - Any instance can serve
7. **Database replication** - Sync replicas
8. **Monitoring & alerting** - Catch issues fast
9. **Runbooks** - Quick incident response
10. **Regular testing** - Chaos engineering

### Q: What's the difference between HA and DR?

**High Availability (HA):**
- Minimize downtime during normal operations
- Automatic failover
- Same region/datacenter
- RTO: seconds to minutes

**Disaster Recovery (DR):**
- Recover from major disasters
- May require manual intervention
- Different regions/datacenters
- RTO: minutes to hours

### Q: How do you handle a cascading failure?

1. **Circuit breakers** - Stop calling failed services
2. **Timeouts** - Don't wait forever
3. **Bulkheads** - Isolate failures
4. **Rate limiting** - Prevent overload
5. **Fallbacks** - Graceful degradation
6. **Load shedding** - Reject excess traffic
