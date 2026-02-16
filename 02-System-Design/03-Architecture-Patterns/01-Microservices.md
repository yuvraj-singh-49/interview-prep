# Microservices Architecture

## Monolith vs Microservices

```
Monolith:
┌─────────────────────────────────────────┐
│              Application                 │
│  ┌─────────┬─────────┬─────────────────┐│
│  │  Users  │ Orders  │ Payments │ ... ││
│  └─────────┴─────────┴─────────────────┘│
│                    │                     │
│            Single Database               │
└─────────────────────────────────────────┘

Microservices:
┌─────────┐  ┌─────────┐  ┌──────────┐  ┌───────┐
│  Users  │  │ Orders  │  │ Payments │  │ Email │
│ Service │  │ Service │  │ Service  │  │Service│
└────┬────┘  └────┬────┘  └────┬─────┘  └───┬───┘
     │            │            │            │
   ┌─┴─┐       ┌──┴──┐      ┌──┴──┐      ┌──┴──┐
   │DB │       │ DB  │      │ DB  │      │Queue│
   └───┘       └─────┘      └─────┘      └─────┘
```

### Comparison

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| Complexity | Simple | Complex |
| Deployment | All at once | Independent |
| Scaling | Entire app | Per service |
| Technology | Single stack | Polyglot |
| Team size | Works for small | Better for large |
| Data consistency | Easy (ACID) | Eventual |
| Debugging | Easier | Distributed tracing |

---

## When to Use Microservices

### Good Fit

```
✓ Large, complex applications
✓ Multiple teams working independently
✓ Different parts have different scaling needs
✓ Different parts need different technologies
✓ Frequent, independent deployments needed
✓ Organization has DevOps maturity
```

### Bad Fit

```
✗ Small team (< 10 developers)
✗ New/uncertain domain
✗ Simple application
✗ Tight deadlines
✗ Limited DevOps experience
✗ Strong coupling between features
```

---

## Service Design Principles

### Single Responsibility

```
Bad: OrderService handles orders, payments, and shipping
Good: Separate services for each concern

┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│OrderService │────▶│PaymentService│────▶│ShippingService│
└─────────────┘     └──────────────┘     └──────────────┘
```

### Domain-Driven Design (DDD)

```
Bounded Contexts:

┌─────────────────────────────────────────────────────────┐
│                    E-Commerce Domain                     │
├─────────────────┬─────────────────┬─────────────────────┤
│  Catalog        │  Ordering       │  Shipping           │
│  Context        │  Context        │  Context            │
│                 │                 │                     │
│  - Product      │  - Order        │  - Shipment         │
│  - Category     │  - LineItem     │  - Carrier          │
│  - Inventory    │  - Payment      │  - Tracking         │
└─────────────────┴─────────────────┴─────────────────────┘

Each context has its own:
- Database schema
- Domain models (Product in Catalog ≠ Product in Ordering)
- Team ownership
```

### Service Boundaries

```javascript
// Questions to ask:
// 1. Can this be deployed independently?
// 2. Does it have a clear business capability?
// 3. Does it own its data?
// 4. Can a small team own it?

// Good boundary: Auth Service
// - Clear capability (authentication/authorization)
// - Owns user credentials data
// - Changes independently of other services
// - Could be replaced entirely

// Bad boundary: Helper Service
// - Too generic
// - Used by everything (high coupling)
// - No clear data ownership
```

---

## Inter-Service Communication

### Synchronous (Request/Response)

```
REST:
┌─────────┐  GET /users/123  ┌─────────┐
│ OrderSvc│─────────────────▶│ UserSvc │
└─────────┘◀─────────────────└─────────┘
              { user data }

gRPC:
┌─────────┐  GetUser(123)   ┌─────────┐
│ OrderSvc│─────────────────▶│ UserSvc │
└─────────┘◀─────────────────└─────────┘
              User proto
```

### Asynchronous (Events)

```
Event-Driven:
┌─────────┐  OrderCreated   ┌─────────┐   ┌─────────┐
│ OrderSvc│────────────────▶│ Message │──▶│ EmailSvc│
└─────────┘                 │  Bus    │──▶│PaymentSvc│
                            └─────────┘──▶│InvntySvc│
```

### Communication Patterns

```javascript
// 1. Request/Response (synchronous)
async function createOrder(orderData) {
  // Call user service directly
  const user = await userService.getUser(orderData.userId);

  // Call payment service directly
  const payment = await paymentService.charge(orderData.total);

  return { order, user, payment };
}

// 2. Event-Driven (asynchronous)
async function createOrder(orderData) {
  const order = await db.insert(orderData);

  // Publish event - other services react
  await messageBus.publish('order.created', {
    orderId: order.id,
    userId: order.userId,
    items: order.items,
    total: order.total
  });

  return order;
}

// 3. Saga (distributed transaction)
async function createOrder(orderData) {
  const saga = new OrderSaga();

  try {
    await saga.reserveInventory(orderData.items);
    await saga.chargePayment(orderData.total);
    await saga.createShipment(orderData);
    await saga.complete();
  } catch (error) {
    await saga.compensate(); // Rollback all steps
    throw error;
  }
}
```

---

## Service Discovery

### Client-Side Discovery

```
┌────────┐     1. Query      ┌──────────────┐
│ Client │─────────────────▶│   Registry   │
└────┬───┘◀─────────────────│ (Consul/etcd)│
     │     2. Service IPs    └──────────────┘
     │
     │     3. Direct call
     ▼
┌──────────┐
│ Service  │
│ Instance │
└──────────┘
```

### Server-Side Discovery

```
┌────────┐     1. Request    ┌──────────────┐
│ Client │──────────────────▶│ Load Balancer│
└────────┘                   └───────┬──────┘
                                     │
                      2. Query ┌─────▼──────┐
                      Registry │  Registry  │
                               └────────────┘
                                     │
                      3. Route ┌─────▼──────┐
                               │  Service   │
                               └────────────┘
```

### Implementation

```javascript
// Consul service registration
const consul = require('consul')();

// Register service
consul.agent.service.register({
  name: 'order-service',
  id: `order-service-${process.pid}`,
  address: 'localhost',
  port: 3000,
  check: {
    http: 'http://localhost:3000/health',
    interval: '10s'
  }
});

// Discover service
async function getServiceUrl(serviceName) {
  const services = await consul.health.service({
    service: serviceName,
    passing: true
  });

  if (services.length === 0) {
    throw new Error(`No healthy instances of ${serviceName}`);
  }

  // Simple round-robin
  const instance = services[Math.floor(Math.random() * services.length)];
  return `http://${instance.Service.Address}:${instance.Service.Port}`;
}
```

---

## API Gateway

```
                           ┌─────────────────┐
                           │   API Gateway   │
                           │                 │
Clients ──────────────────▶│ - Auth          │
                           │ - Rate limiting │
                           │ - Routing       │
                           │ - Aggregation   │
                           │ - SSL           │
                           └────────┬────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          ▼                         ▼                         ▼
    ┌───────────┐            ┌───────────┐            ┌───────────┐
    │  Users    │            │  Orders   │            │ Products  │
    │  Service  │            │  Service  │            │  Service  │
    └───────────┘            └───────────┘            └───────────┘
```

### API Gateway Responsibilities

```javascript
// Express API Gateway example
const express = require('express');
const httpProxy = require('http-proxy-middleware');

const app = express();

// 1. Authentication
app.use(async (req, res, next) => {
  const token = req.headers.authorization;
  req.user = await authService.verify(token);
  next();
});

// 2. Rate limiting
app.use(rateLimit({
  windowMs: 60 * 1000,
  max: 100
}));

// 3. Routing
app.use('/api/users', httpProxy({
  target: 'http://user-service:3001'
}));

app.use('/api/orders', httpProxy({
  target: 'http://order-service:3002'
}));

// 4. Aggregation (BFF pattern)
app.get('/api/dashboard', async (req, res) => {
  const [user, orders, recommendations] = await Promise.all([
    userService.getProfile(req.user.id),
    orderService.getRecent(req.user.id),
    recommendationService.get(req.user.id)
  ]);

  res.json({ user, orders, recommendations });
});
```

---

## Data Management

### Database per Service

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ User Service│  │Order Service│  │Product Svc  │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
   ┌───▼───┐        ┌───▼───┐        ┌───▼───┐
   │  SQL  │        │ NoSQL │        │ SQL   │
   │(Users)│        │(Orders)│       │(Prods)│
   └───────┘        └───────┘        └───────┘

Benefits:
- Independent schema evolution
- Right database for each service
- No direct coupling

Challenges:
- No ACID across services
- Data duplication
- Cross-service queries
```

### Saga Pattern

```javascript
// Orchestration-based saga
class OrderSaga {
  constructor() {
    this.steps = [];
  }

  async execute(order) {
    try {
      // Step 1: Reserve inventory
      const inventoryReservation = await inventoryService.reserve(order.items);
      this.steps.push({ service: 'inventory', action: 'reserve', data: inventoryReservation });

      // Step 2: Process payment
      const payment = await paymentService.charge(order.userId, order.total);
      this.steps.push({ service: 'payment', action: 'charge', data: payment });

      // Step 3: Create shipment
      const shipment = await shippingService.createShipment(order);
      this.steps.push({ service: 'shipping', action: 'create', data: shipment });

      // Step 4: Confirm order
      return await orderService.confirm(order.id);
    } catch (error) {
      await this.compensate();
      throw error;
    }
  }

  async compensate() {
    // Reverse steps in order
    for (const step of this.steps.reverse()) {
      switch (step.service) {
        case 'inventory':
          await inventoryService.release(step.data.reservationId);
          break;
        case 'payment':
          await paymentService.refund(step.data.paymentId);
          break;
        case 'shipping':
          await shippingService.cancel(step.data.shipmentId);
          break;
      }
    }
  }
}
```

---

## Deployment Patterns

### Blue-Green Deployment

```
           ┌─────────────────┐
           │  Load Balancer  │
           └────────┬────────┘
                    │
        ┌───────────┼───────────┐
        │ (active)  │ (standby) │
        ▼           │           ▼
   ┌─────────┐      │      ┌─────────┐
   │  Blue   │      │      │  Green  │
   │  v1.0   │      │      │  v1.1   │
   └─────────┘      │      └─────────┘
                    │
Switch: Point LB to Green
Rollback: Point LB back to Blue
```

### Canary Deployment

```
           ┌─────────────────┐
           │  Load Balancer  │
           └────────┬────────┘
                    │
        ┌───────────┼───────────┐
        │   90%     │    10%    │
        ▼           │           ▼
   ┌─────────┐      │      ┌─────────┐
   │ Stable  │      │      │ Canary  │
   │  v1.0   │      │      │  v1.1   │
   └─────────┘      │      └─────────┘

Gradually increase canary traffic
Monitor metrics
Rollback if issues detected
```

---

## Testing Strategies

```javascript
// 1. Unit tests (isolated)
describe('OrderService', () => {
  it('should calculate total correctly', () => {
    const items = [{ price: 10, qty: 2 }, { price: 5, qty: 1 }];
    expect(OrderService.calculateTotal(items)).toBe(25);
  });
});

// 2. Integration tests (with dependencies)
describe('OrderService Integration', () => {
  it('should create order with payment', async () => {
    const mockPayment = nock('http://payment-service')
      .post('/charge')
      .reply(200, { paymentId: '123' });

    const order = await orderService.create({ total: 100 });
    expect(order.paymentId).toBe('123');
  });
});

// 3. Contract tests (API compatibility)
describe('Order API Contract', () => {
  it('should match provider contract', async () => {
    await pact.verify({
      provider: 'OrderService',
      consumer: 'CheckoutService',
      interactions: [...]
    });
  });
});

// 4. End-to-end tests
describe('Order Flow E2E', () => {
  it('should complete order lifecycle', async () => {
    const order = await api.createOrder(orderData);
    expect(order.status).toBe('created');

    await api.payOrder(order.id);
    const paid = await api.getOrder(order.id);
    expect(paid.status).toBe('paid');
  });
});
```

---

## Interview Questions

### Q: How do you handle distributed transactions?

1. **Saga pattern** - Sequence of local transactions with compensations
2. **Event sourcing** - Store events, derive state
3. **2PC** - Two-phase commit (avoid if possible)
4. **Eventual consistency** - Accept temporary inconsistency

### Q: How do you debug issues in microservices?

1. **Distributed tracing** - Jaeger, Zipkin
2. **Centralized logging** - ELK stack
3. **Correlation IDs** - Track requests across services
4. **Metrics/dashboards** - Prometheus, Grafana
5. **Service mesh** - Istio for observability

### Q: How do you handle service-to-service authentication?

1. **mTLS** - Mutual TLS between services
2. **JWT** - Pass tokens between services
3. **API keys** - Internal service keys
4. **Service mesh** - Handle auth at infrastructure level
