# REST API Design

## REST Principles

### Resource-Based URLs

```
Resources are nouns, not verbs

Good:
GET    /users          - List users
GET    /users/123      - Get user 123
POST   /users          - Create user
PUT    /users/123      - Replace user 123
PATCH  /users/123      - Update user 123
DELETE /users/123      - Delete user 123

Bad:
GET    /getUsers
POST   /createUser
POST   /users/123/delete
```

### HTTP Methods

```
Method   │ Idempotent │ Safe │ Use Case
─────────┼────────────┼──────┼─────────────────────
GET      │ Yes        │ Yes  │ Retrieve resource
POST     │ No         │ No   │ Create resource
PUT      │ Yes        │ No   │ Replace resource
PATCH    │ No         │ No   │ Partial update
DELETE   │ Yes        │ No   │ Remove resource
HEAD     │ Yes        │ Yes  │ Get headers only
OPTIONS  │ Yes        │ Yes  │ Get allowed methods
```

### Status Codes

```javascript
// 2xx Success
200 OK                  // General success
201 Created             // Resource created
202 Accepted            // Async operation started
204 No Content          // Success, no body (DELETE)

// 3xx Redirection
301 Moved Permanently   // Resource moved
304 Not Modified        // Use cache

// 4xx Client Errors
400 Bad Request         // Invalid request
401 Unauthorized        // Auth required
403 Forbidden           // No permission
404 Not Found           // Resource doesn't exist
409 Conflict            // State conflict
422 Unprocessable       // Validation failed
429 Too Many Requests   // Rate limited

// 5xx Server Errors
500 Internal Error      // Server error
502 Bad Gateway         // Upstream error
503 Service Unavailable // Temporarily down
504 Gateway Timeout     // Upstream timeout
```

---

## URL Design

### Nested Resources

```
# User's orders
GET /users/123/orders
POST /users/123/orders

# Order items
GET /users/123/orders/456/items

# Alternative: top-level with filter
GET /orders?userId=123

# When to nest:
# - Clear parent-child relationship
# - Child doesn't exist without parent
# - Max 2 levels deep
```

### Query Parameters

```
# Filtering
GET /products?category=electronics&price_min=100&price_max=500

# Sorting
GET /products?sort=price      # Ascending
GET /products?sort=-price     # Descending
GET /products?sort=category,price

# Pagination
GET /products?page=2&limit=20
GET /products?offset=40&limit=20
GET /products?cursor=eyJpZCI6MTIzfQ==

# Field selection (sparse fieldsets)
GET /users/123?fields=id,name,email

# Including related resources
GET /users/123?include=orders,profile
```

### Filtering Best Practices

```javascript
// Express implementation
app.get('/products', async (req, res) => {
  const {
    category,
    price_min,
    price_max,
    sort = 'created_at',
    page = 1,
    limit = 20
  } = req.query;

  const query = {};
  if (category) query.category = category;
  if (price_min || price_max) {
    query.price = {};
    if (price_min) query.price.$gte = Number(price_min);
    if (price_max) query.price.$lte = Number(price_max);
  }

  const sortOrder = sort.startsWith('-') ? -1 : 1;
  const sortField = sort.replace(/^-/, '');

  const products = await Product.find(query)
    .sort({ [sortField]: sortOrder })
    .skip((page - 1) * limit)
    .limit(Number(limit));

  res.json({
    data: products,
    pagination: {
      page: Number(page),
      limit: Number(limit),
      total: await Product.countDocuments(query)
    }
  });
});
```

---

## Request/Response Design

### Request Body

```javascript
// POST /users
{
  "name": "John Doe",
  "email": "john@example.com",
  "preferences": {
    "newsletter": true,
    "theme": "dark"
  }
}

// PATCH /users/123
{
  "email": "newemail@example.com"
}
```

### Response Body

```javascript
// Single resource
{
  "data": {
    "id": "123",
    "type": "user",
    "attributes": {
      "name": "John Doe",
      "email": "john@example.com"
    },
    "relationships": {
      "orders": {
        "links": { "related": "/users/123/orders" }
      }
    }
  },
  "meta": {
    "requestId": "req-abc-123"
  }
}

// Collection
{
  "data": [
    { "id": "1", "type": "user", ... },
    { "id": "2", "type": "user", ... }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "pages": 5,
    "hasNext": true,
    "hasPrev": false
  },
  "links": {
    "self": "/users?page=1",
    "next": "/users?page=2",
    "last": "/users?page=5"
  }
}
```

### Error Response

```javascript
// Validation error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "code": "invalid_format",
        "message": "Must be a valid email address"
      },
      {
        "field": "name",
        "code": "required",
        "message": "Name is required"
      }
    ]
  },
  "meta": {
    "requestId": "req-abc-123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}

// Not found
{
  "error": {
    "code": "NOT_FOUND",
    "message": "User not found",
    "resource": "user",
    "resourceId": "123"
  }
}
```

---

## Pagination

### Offset-Based

```javascript
// GET /products?page=2&limit=20
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 500,
    "pages": 25
  }
}

// Implementation
const page = parseInt(req.query.page) || 1;
const limit = parseInt(req.query.limit) || 20;
const offset = (page - 1) * limit;

const products = await db.query(`
  SELECT * FROM products
  ORDER BY created_at DESC
  LIMIT $1 OFFSET $2
`, [limit, offset]);
```

**Cons:** Slow for large offsets, inconsistent with concurrent inserts

### Cursor-Based

```javascript
// GET /products?cursor=eyJpZCI6MTIzfQ==&limit=20
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTQzfQ==",
    "hasMore": true
  }
}

// Implementation
function encodeCursor(data) {
  return Buffer.from(JSON.stringify(data)).toString('base64');
}

function decodeCursor(cursor) {
  return JSON.parse(Buffer.from(cursor, 'base64').toString());
}

app.get('/products', async (req, res) => {
  const { cursor, limit = 20 } = req.query;

  let query = 'SELECT * FROM products';
  const params = [parseInt(limit) + 1]; // Fetch one extra

  if (cursor) {
    const { id } = decodeCursor(cursor);
    query += ' WHERE id > $2';
    params.push(id);
  }

  query += ' ORDER BY id ASC LIMIT $1';

  const products = await db.query(query, params);
  const hasMore = products.length > limit;

  if (hasMore) products.pop(); // Remove extra

  res.json({
    data: products,
    pagination: {
      nextCursor: hasMore ? encodeCursor({ id: products[products.length - 1].id }) : null,
      hasMore
    }
  });
});
```

---

## Versioning

### URL Path Versioning

```
GET /v1/users
GET /v2/users

Pros:
- Explicit and clear
- Easy routing
- Cacheable

Cons:
- URL changes between versions
- Can't easily test multiple versions
```

### Header Versioning

```
GET /users
Accept: application/vnd.api+json; version=2

// Or custom header
GET /users
X-API-Version: 2

Pros:
- Clean URLs
- Same URL for all versions

Cons:
- Less visible
- Harder to test in browser
```

### Query Parameter

```
GET /users?version=2

Pros:
- Easy to test
- Explicit

Cons:
- Pollutes query string
- Easy to forget
```

### Versioning Strategy

```javascript
// Semantic versioning for breaking changes only
// v1 → v2 only when breaking changes

// Non-breaking changes (stay on same version):
// - Adding new fields
// - Adding new endpoints
// - Adding optional parameters

// Breaking changes (new version):
// - Removing fields
// - Changing field types
// - Changing endpoint behavior
```

---

## Authentication & Security

### API Key

```javascript
app.use((req, res, next) => {
  const apiKey = req.headers['x-api-key'];

  if (!apiKey || !isValidApiKey(apiKey)) {
    return res.status(401).json({
      error: { code: 'UNAUTHORIZED', message: 'Invalid API key' }
    });
  }

  req.client = await getClientByApiKey(apiKey);
  next();
});
```

### JWT Authentication

```javascript
const jwt = require('jsonwebtoken');

// Login - issue token
app.post('/auth/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await authenticateUser(email, password);

  const token = jwt.sign(
    { userId: user.id, email: user.email },
    process.env.JWT_SECRET,
    { expiresIn: '1h' }
  );

  res.json({ token, expiresIn: 3600 });
});

// Middleware - verify token
function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;

  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: { code: 'UNAUTHORIZED' } });
  }

  const token = authHeader.substring(7);

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch (error) {
    res.status(401).json({ error: { code: 'INVALID_TOKEN' } });
  }
}
```

### Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

// Global rate limit
app.use(rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max: 100,             // 100 requests per minute
  standardHeaders: true,
  message: {
    error: {
      code: 'RATE_LIMITED',
      message: 'Too many requests',
      retryAfter: 60
    }
  }
}));

// Per-endpoint rate limit
const strictLimit = rateLimit({
  windowMs: 60 * 1000,
  max: 10
});

app.post('/auth/login', strictLimit, loginHandler);
```

---

## HATEOAS

```javascript
// Hypermedia as the Engine of Application State
// Include links to related actions/resources

GET /orders/123

{
  "data": {
    "id": "123",
    "status": "pending",
    "total": 99.99
  },
  "links": {
    "self": "/orders/123",
    "pay": "/orders/123/pay",
    "cancel": "/orders/123/cancel",
    "items": "/orders/123/items",
    "user": "/users/456"
  },
  "actions": {
    "pay": {
      "method": "POST",
      "href": "/orders/123/pay",
      "fields": [
        { "name": "paymentMethod", "type": "string", "required": true }
      ]
    }
  }
}
```

---

## Interview Questions

### Q: How do you design a REST API for a new feature?

1. **Identify resources** - What entities are involved?
2. **Define endpoints** - CRUD operations needed?
3. **Design request/response** - What data is exchanged?
4. **Handle relationships** - Nested vs flat?
5. **Plan for pagination** - Large collections?
6. **Consider versioning** - Future compatibility?
7. **Add security** - Auth, rate limiting?
8. **Document** - OpenAPI spec

### Q: Offset vs cursor pagination - when to use which?

**Offset:**
- Simple implementation
- Random access needed (jump to page 5)
- Small datasets

**Cursor:**
- Large datasets
- Real-time data (new items added)
- Consistent pagination
- Better performance

### Q: How do you handle API versioning?

1. Only version on breaking changes
2. Use URL path versioning for simplicity
3. Support N-1 versions (current + previous)
4. Deprecation notices in headers
5. Migration guides for clients
6. Sunset dates communicated early
