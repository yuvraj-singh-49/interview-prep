# Fault Tolerance Patterns

## Overview

```
Fault tolerance = System continues operating despite failures

Key Patterns:
1. Circuit Breaker  - Fail fast when service is down
2. Retry           - Handle transient failures
3. Timeout         - Don't wait forever
4. Bulkhead        - Isolate failures
5. Fallback        - Graceful degradation
6. Rate Limiting   - Prevent overload
```

---

## Circuit Breaker

### States

```
       ┌─────────────────────────────────────────┐
       │                                         │
       ▼                     Success             │
┌──────────┐    Failures    ┌────────────────────┤
│  CLOSED  │───────────────▶│      OPEN         │
│(normal)  │    threshold   │  (fail fast)      │
└──────────┘                └─────────┬──────────┘
       ▲                              │
       │                         Timeout
       │                              │
       │                              ▼
       │                       ┌──────────┐
       │        Success        │HALF-OPEN │
       └───────────────────────│(testing) │
              or failure       └──────────┘
              → OPEN
```

### Implementation

```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.lastFailureTime = null;

    // Configuration
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 3;
    this.timeout = options.timeout || 30000;  // 30 seconds
    this.monitorWindow = options.monitorWindow || 60000;  // 1 minute
  }

  async execute(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime >= this.timeout) {
        this.state = 'HALF_OPEN';
        this.successCount = 0;
      } else {
        throw new CircuitBreakerOpenError('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.successThreshold) {
        this.state = 'CLOSED';
        this.failureCount = 0;
      }
    } else if (this.state === 'CLOSED') {
      this.failureCount = 0;
    }
  }

  onFailure() {
    this.lastFailureTime = Date.now();

    if (this.state === 'HALF_OPEN') {
      this.state = 'OPEN';
    } else if (this.state === 'CLOSED') {
      this.failureCount++;
      if (this.failureCount >= this.failureThreshold) {
        this.state = 'OPEN';
      }
    }
  }

  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      lastFailureTime: this.lastFailureTime
    };
  }
}

// Usage
const breaker = new CircuitBreaker({
  failureThreshold: 5,
  timeout: 30000
});

async function callExternalService() {
  return breaker.execute(async () => {
    const response = await fetch('http://external-service/api');
    if (!response.ok) throw new Error('Service error');
    return response.json();
  });
}
```

---

## Retry Pattern

### Exponential Backoff with Jitter

```javascript
class RetryPolicy {
  constructor(options = {}) {
    this.maxRetries = options.maxRetries || 3;
    this.baseDelay = options.baseDelay || 1000;
    this.maxDelay = options.maxDelay || 30000;
    this.shouldRetry = options.shouldRetry || this.defaultShouldRetry;
  }

  defaultShouldRetry(error) {
    // Retry on network errors and 5xx
    if (error.code === 'ECONNREFUSED') return true;
    if (error.status >= 500) return true;
    if (error.status === 429) return true;  // Rate limited
    return false;
  }

  calculateDelay(attempt) {
    // Exponential backoff
    const exponentialDelay = this.baseDelay * Math.pow(2, attempt);

    // Add jitter (random 0-100% of delay)
    const jitter = Math.random() * exponentialDelay;

    return Math.min(exponentialDelay + jitter, this.maxDelay);
  }

  async execute(fn) {
    let lastError;

    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error;

        if (attempt === this.maxRetries || !this.shouldRetry(error)) {
          throw error;
        }

        const delay = this.calculateDelay(attempt);
        console.log(`Retry attempt ${attempt + 1} after ${delay}ms`);
        await this.sleep(delay);
      }
    }

    throw lastError;
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
const retryPolicy = new RetryPolicy({
  maxRetries: 3,
  baseDelay: 1000,
  shouldRetry: (error) => error.status >= 500 || error.code === 'ECONNRESET'
});

const result = await retryPolicy.execute(async () => {
  return await fetch('http://api.example.com/data');
});
```

---

## Timeout Pattern

```javascript
class TimeoutPolicy {
  constructor(timeoutMs) {
    this.timeout = timeoutMs;
  }

  async execute(fn) {
    return Promise.race([
      fn(),
      this.createTimeout()
    ]);
  }

  createTimeout() {
    return new Promise((_, reject) => {
      setTimeout(() => {
        reject(new TimeoutError(`Operation timed out after ${this.timeout}ms`));
      }, this.timeout);
    });
  }
}

// Cascading timeouts for service calls
class ServiceClient {
  constructor() {
    // Shorter timeouts for faster failure detection
    this.userServiceTimeout = new TimeoutPolicy(2000);    // 2s
    this.paymentServiceTimeout = new TimeoutPolicy(5000); // 5s
    this.overallTimeout = new TimeoutPolicy(10000);       // 10s
  }

  async processOrder(order) {
    return this.overallTimeout.execute(async () => {
      const user = await this.userServiceTimeout.execute(() =>
        userService.getUser(order.userId)
      );

      const payment = await this.paymentServiceTimeout.execute(() =>
        paymentService.charge(order.total)
      );

      return { user, payment };
    });
  }
}
```

---

## Bulkhead Pattern

Isolate failures to prevent cascade.

```javascript
class Bulkhead {
  constructor(options = {}) {
    this.maxConcurrent = options.maxConcurrent || 10;
    this.maxQueue = options.maxQueue || 100;
    this.running = 0;
    this.queue = [];
  }

  async execute(fn) {
    // Check if we can run immediately
    if (this.running < this.maxConcurrent) {
      return this.run(fn);
    }

    // Check if queue is full
    if (this.queue.length >= this.maxQueue) {
      throw new BulkheadFullError('Bulkhead queue is full');
    }

    // Add to queue
    return new Promise((resolve, reject) => {
      this.queue.push({ fn, resolve, reject });
    });
  }

  async run(fn) {
    this.running++;
    try {
      return await fn();
    } finally {
      this.running--;
      this.processQueue();
    }
  }

  processQueue() {
    if (this.queue.length > 0 && this.running < this.maxConcurrent) {
      const { fn, resolve, reject } = this.queue.shift();
      this.run(fn).then(resolve).catch(reject);
    }
  }
}

// Separate bulkheads for different services
class ServiceClient {
  constructor() {
    // Critical services get more capacity
    this.paymentBulkhead = new Bulkhead({ maxConcurrent: 20, maxQueue: 50 });
    this.notificationBulkhead = new Bulkhead({ maxConcurrent: 5, maxQueue: 100 });
    this.analyticsBulkhead = new Bulkhead({ maxConcurrent: 3, maxQueue: 500 });
  }

  async processPayment(data) {
    return this.paymentBulkhead.execute(() =>
      paymentService.charge(data)
    );
  }

  async sendNotification(data) {
    return this.notificationBulkhead.execute(() =>
      notificationService.send(data)
    );
  }

  // Analytics bulkhead is separate - won't affect payments
  async trackEvent(event) {
    return this.analyticsBulkhead.execute(() =>
      analyticsService.track(event)
    );
  }
}
```

---

## Fallback Pattern

```javascript
class FallbackPolicy {
  constructor(fallbackFn) {
    this.fallback = fallbackFn;
  }

  async execute(fn) {
    try {
      return await fn();
    } catch (error) {
      console.warn('Primary failed, using fallback:', error.message);
      return this.fallback(error);
    }
  }
}

// Multi-level fallback
async function getProductPrice(productId) {
  // Try primary service
  try {
    return await priceService.getPrice(productId);
  } catch (e) {
    console.warn('Price service failed');
  }

  // Fallback to cache
  try {
    const cached = await cache.get(`price:${productId}`);
    if (cached) {
      return { price: cached, stale: true };
    }
  } catch (e) {
    console.warn('Cache lookup failed');
  }

  // Fallback to database
  try {
    const product = await db.products.findById(productId);
    return { price: product.basePrice, source: 'database' };
  } catch (e) {
    console.warn('Database lookup failed');
  }

  // Ultimate fallback - default price
  return { price: 0, error: true };
}
```

---

## Combined Resilience

```javascript
class ResilientClient {
  constructor(serviceName, options = {}) {
    this.serviceName = serviceName;

    // Initialize all patterns
    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: options.failureThreshold || 5,
      timeout: options.circuitTimeout || 30000
    });

    this.retryPolicy = new RetryPolicy({
      maxRetries: options.maxRetries || 3,
      baseDelay: options.baseDelay || 1000
    });

    this.timeout = new TimeoutPolicy(options.timeout || 5000);

    this.bulkhead = new Bulkhead({
      maxConcurrent: options.maxConcurrent || 10,
      maxQueue: options.maxQueue || 100
    });
  }

  async call(fn, fallback) {
    // Bulkhead → Circuit Breaker → Timeout → Retry → Fallback
    try {
      return await this.bulkhead.execute(async () => {
        return await this.circuitBreaker.execute(async () => {
          return await this.retryPolicy.execute(async () => {
            return await this.timeout.execute(fn);
          });
        });
      });
    } catch (error) {
      if (fallback) {
        return fallback(error);
      }
      throw error;
    }
  }
}

// Usage
const userClient = new ResilientClient('user-service', {
  failureThreshold: 3,
  maxRetries: 2,
  timeout: 3000,
  maxConcurrent: 20
});

const user = await userClient.call(
  () => fetch('http://user-service/users/123'),
  (error) => ({ id: '123', name: 'Unknown', stale: true })
);
```

---

## Health Checks

```javascript
class HealthChecker {
  constructor() {
    this.checks = new Map();
  }

  register(name, checkFn, options = {}) {
    this.checks.set(name, {
      check: checkFn,
      timeout: options.timeout || 5000,
      critical: options.critical !== false
    });
  }

  async checkHealth() {
    const results = {};
    let healthy = true;

    for (const [name, config] of this.checks) {
      try {
        const start = Date.now();
        await Promise.race([
          config.check(),
          new Promise((_, reject) =>
            setTimeout(() => reject(new Error('Timeout')), config.timeout)
          )
        ]);

        results[name] = {
          status: 'healthy',
          latency: Date.now() - start
        };
      } catch (error) {
        results[name] = {
          status: 'unhealthy',
          error: error.message
        };

        if (config.critical) {
          healthy = false;
        }
      }
    }

    return {
      status: healthy ? 'healthy' : 'unhealthy',
      checks: results,
      timestamp: new Date().toISOString()
    };
  }
}

// Register checks
const health = new HealthChecker();

health.register('database', async () => {
  await db.query('SELECT 1');
}, { critical: true });

health.register('cache', async () => {
  await redis.ping();
}, { critical: true });

health.register('external-api', async () => {
  await fetch('http://api.example.com/health');
}, { critical: false });

// Expose endpoint
app.get('/health', async (req, res) => {
  const status = await health.checkHealth();
  res.status(status.status === 'healthy' ? 200 : 503).json(status);
});
```

---

## Interview Questions

### Q: How do you prevent cascading failures?

1. **Circuit breakers** - Stop calling failed services
2. **Timeouts** - Fail fast, don't block
3. **Bulkheads** - Isolate resource pools
4. **Rate limiting** - Prevent overload
5. **Fallbacks** - Graceful degradation
6. **Health checks** - Remove unhealthy instances

### Q: When to use retry vs circuit breaker?

**Retry:**
- Transient failures (network blips)
- Idempotent operations
- Low failure rate

**Circuit Breaker:**
- Persistent failures (service down)
- Protect downstream services
- High failure rate

**Best practice:** Use both together - retry first, circuit breaker wraps retry.

### Q: How to handle partial failures?

1. **Accept partial success** - Return what succeeded
2. **Compensate** - Rollback completed parts (Saga)
3. **Queue for retry** - Process failed parts later
4. **Notify user** - Let them retry manually
