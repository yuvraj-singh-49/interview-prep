# API Gateway

## Overview

```
API Gateway = Fully managed API management service

┌─────────────────────────────────────────────────────────────────────────┐
│                      API Gateway Types                                   │
├─────────────────────┬─────────────────────┬─────────────────────────────┤
│     REST API        │     HTTP API        │     WebSocket API           │
│                     │                     │                             │
│  - Full features    │  - Simpler, cheaper │  - Real-time two-way       │
│  - Request/response │  - 70% cheaper      │  - Persistent connections  │
│    transformation   │  - Lower latency    │  - Chat, gaming, alerts    │
│  - API keys, plans  │  - JWT authorizers  │  - Route selection         │
│  - Caching          │  - CORS, OIDC       │  - Connection management   │
│  - WAF integration  │  - No caching       │                             │
└─────────────────────┴─────────────────────┴─────────────────────────────┘
```

---

## REST API Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Client ──▶ API Gateway ──▶ Integration ──▶ Backend                    │
│                                                                          │
│   Request Flow:                                                          │
│   ┌────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐       │
│   │ Client │───▶│ Method   │───▶│ Integr-  │───▶│ Lambda/      │       │
│   │        │    │ Request  │    │ ation    │    │ HTTP/AWS     │       │
│   │        │    │          │    │ Request  │    │ Service      │       │
│   └────────┘    └──────────┘    └──────────┘    └──────────────┘       │
│        ▲             │               │                │                 │
│        │             ▼               ▼                ▼                 │
│   ┌────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐       │
│   │Response│◀───│ Method   │◀───│ Integr-  │◀───│ Backend      │       │
│   │        │    │ Response │    │ ation    │    │ Response     │       │
│   │        │    │          │    │ Response │    │              │       │
│   └────────┘    └──────────┘    └──────────┘    └──────────────┘       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Resources and Methods

```
API Structure:
/                           ← Root resource
├── /users                  ← Resource
│   ├── GET                 ← Method (list users)
│   ├── POST                ← Method (create user)
│   └── /{userId}           ← Path parameter
│       ├── GET             ← Get specific user
│       ├── PUT             ← Update user
│       └── DELETE          ← Delete user
├── /orders
│   └── ...
└── /products
    └── ...

// Path parameters
/users/{userId}/orders/{orderId}

// Query parameters
/users?status=active&limit=10

// Greedy path (proxy)
/{proxy+}  ← Catches all paths
```

---

## Integration Types

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Integration Type    │ Use Case                                          │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Lambda              │ Most common - invoke Lambda function              │
│                     │ Proxy or non-proxy mode                           │
├─────────────────────┼───────────────────────────────────────────────────┤
│ HTTP                │ Backend HTTP endpoints                            │
│                     │ On-premises, other services                       │
├─────────────────────┼───────────────────────────────────────────────────┤
│ AWS Service         │ Direct AWS service integration                    │
│                     │ DynamoDB, SQS, Step Functions                    │
├─────────────────────┼───────────────────────────────────────────────────┤
│ VPC Link            │ Private resources in VPC                          │
│                     │ NLB-backed services                               │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Mock                │ Return static response                            │
│                     │ Testing, stubs                                    │
└─────────────────────┴───────────────────────────────────────────────────┘
```

### Lambda Proxy Integration

```javascript
// Lambda receives full request context
exports.handler = async (event) => {
  // event structure
  const {
    httpMethod,           // GET, POST, etc.
    path,                 // /users/123
    pathParameters,       // { userId: '123' }
    queryStringParameters,// { status: 'active' }
    headers,              // Request headers
    body,                 // Request body (string)
    requestContext        // API Gateway context
  } = event;

  // Parse body
  const data = JSON.parse(body || '{}');

  // Return response
  return {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({
      message: 'Success',
      userId: pathParameters?.userId
    })
  };
};
```

### AWS Service Integration (Direct DynamoDB)

```
Direct DynamoDB Integration (no Lambda):
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   API Gateway ──▶ DynamoDB                                              │
│                                                                          │
│   Integration Request:                                                   │
│   {                                                                      │
│     "TableName": "Users",                                               │
│     "Key": {                                                            │
│       "userId": { "S": "$input.params('userId')" }                      │
│     }                                                                    │
│   }                                                                      │
│                                                                          │
│   Benefits:                                                              │
│   - Lower latency (no Lambda cold start)                                │
│   - Lower cost (no Lambda invocation)                                   │
│   - Less code to maintain                                               │
│                                                                          │
│   Limitations:                                                           │
│   - Limited transformation                                              │
│   - Simple operations only                                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Authorization

### Authorization Methods

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Method              │ Use Case                                          │
├─────────────────────┼───────────────────────────────────────────────────┤
│ API Keys            │ Simple client identification                      │
│                     │ Usage plans, throttling                           │
├─────────────────────┼───────────────────────────────────────────────────┤
│ IAM                 │ AWS users/roles                                   │
│                     │ Cross-account access                              │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Cognito User Pools  │ User authentication                               │
│                     │ JWT tokens, social login                          │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Lambda Authorizer   │ Custom authentication                             │
│                     │ Token or request-based                            │
└─────────────────────┴───────────────────────────────────────────────────┘
```

### Lambda Authorizer

```javascript
// Token-based authorizer
exports.handler = async (event) => {
  const token = event.authorizationToken;  // Bearer xxx

  try {
    // Validate token (JWT, custom, etc.)
    const decoded = verifyToken(token);

    return {
      principalId: decoded.userId,
      policyDocument: {
        Version: '2012-10-17',
        Statement: [{
          Action: 'execute-api:Invoke',
          Effect: 'Allow',
          Resource: event.methodArn
          // Or wildcard: 'arn:aws:execute-api:*:*:*'
        }]
      },
      context: {
        userId: decoded.userId,
        email: decoded.email
        // Available in $context.authorizer.xxx
      }
    };
  } catch (error) {
    throw new Error('Unauthorized');
  }
};

// Request-based authorizer (HTTP API)
exports.handler = async (event) => {
  const { headers, queryStringParameters } = event;

  // Validate request
  if (headers['x-api-key'] === 'valid-key') {
    return {
      isAuthorized: true,
      context: { clientId: 'client-123' }
    };
  }

  return { isAuthorized: false };
};
```

### Cognito Integration

```
Cognito User Pool Integration:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. User authenticates with Cognito                                    │
│      └── Gets ID token and Access token                                 │
│                                                                          │
│   2. Client sends request with Authorization header                     │
│      Authorization: Bearer <id_token>                                   │
│                                                                          │
│   3. API Gateway validates token                                         │
│      - Checks signature                                                 │
│      - Verifies expiration                                              │
│      - Validates claims                                                 │
│                                                                          │
│   4. Request proceeds to backend                                        │
│      - Claims available in $context.authorizer.claims                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

// Access claims in Lambda
const userId = event.requestContext.authorizer.claims.sub;
const email = event.requestContext.authorizer.claims.email;
```

---

## Request/Response Transformation

### Mapping Templates (VTL)

```
Request Transformation:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  Client Request:                                                         │
│  POST /users                                                             │
│  { "name": "John", "email": "john@example.com" }                        │
│                                                                          │
│  Mapping Template (VTL):                                                 │
│  {                                                                       │
│    "TableName": "Users",                                                │
│    "Item": {                                                            │
│      "userId": { "S": "$context.requestId" },                          │
│      "name": { "S": "$input.path('$.name')" },                         │
│      "email": { "S": "$input.path('$.email')" },                       │
│      "createdAt": { "S": "$context.requestTimeEpoch" }                 │
│    }                                                                     │
│  }                                                                       │
│                                                                          │
│  Backend (DynamoDB) receives transformed request                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

// Common VTL variables
$input.body                    // Raw request body
$input.json('$.field')         // JSON field
$input.params('paramName')     // Path/query parameter
$context.requestId             // Unique request ID
$context.identity.sourceIp     // Client IP
$context.authorizer.claims.sub // Cognito user ID
```

---

## Caching

```
API Gateway Caching:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Client ──▶ API Gateway ──cache hit──▶ Return cached response          │
│                    │                                                     │
│                    └──cache miss──▶ Backend ──▶ Cache response          │
│                                                                          │
│   Configuration:                                                         │
│   - TTL: 0-3600 seconds (default 300)                                   │
│   - Cache size: 0.5 GB - 237 GB                                         │
│   - Per-key caching (path, query, headers)                              │
│   - Encryption at rest                                                  │
│                                                                          │
│   Cache Keys:                                                            │
│   - Path parameters                                                     │
│   - Query strings (selected or all)                                     │
│   - Headers (selected)                                                  │
│                                                                          │
│   Invalidation:                                                          │
│   - Header: Cache-Control: max-age=0                                    │
│   - Flush entire cache in console                                       │
│                                                                          │
│   Cost: $0.02 - $3.80 per hour (depends on size)                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Throttling & Quotas

```
Throttling Levels:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Account Level (default):                                               │
│   - 10,000 requests/second                                              │
│   - 5,000 burst                                                         │
│                                                                          │
│   Stage Level:                                                           │
│   - Override account defaults                                           │
│   - Per stage throttling                                                │
│                                                                          │
│   Method Level:                                                          │
│   - Fine-grained per endpoint                                           │
│   - Example: POST /orders: 100 req/s                                    │
│                                                                          │
│   Usage Plans:                                                           │
│   - API key-based quotas                                                │
│   - Daily/weekly/monthly limits                                         │
│   - Throttle by client                                                  │
│                                                                          │
│   Response when throttled: 429 Too Many Requests                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Usage Plans

```
Usage Plan Configuration:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Plan: Basic                                                            │
│   ├── Throttle: 100 req/sec, burst 200                                  │
│   ├── Quota: 10,000 requests/day                                        │
│   └── API Keys: key-123, key-456                                        │
│                                                                          │
│   Plan: Premium                                                          │
│   ├── Throttle: 1000 req/sec, burst 2000                                │
│   ├── Quota: 1,000,000 requests/month                                   │
│   └── API Keys: key-789                                                 │
│                                                                          │
│   Client Request:                                                        │
│   x-api-key: key-123                                                    │
│                                                                          │
│   API Gateway checks:                                                    │
│   1. Valid API key?                                                     │
│   2. Within throttle limit?                                             │
│   3. Within quota?                                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## CORS Configuration

```javascript
// Enable CORS for REST API
// 1. Add OPTIONS method for preflight
// 2. Add CORS headers to responses

// Lambda response with CORS
return {
  statusCode: 200,
  headers: {
    'Access-Control-Allow-Origin': 'https://example.com',
    'Access-Control-Allow-Headers': 'Content-Type,Authorization',
    'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS',
    'Access-Control-Allow-Credentials': 'true'
  },
  body: JSON.stringify(data)
};

// HTTP API - Simple CORS
{
  "cors": {
    "allowOrigins": ["https://example.com"],
    "allowMethods": ["GET", "POST", "PUT", "DELETE"],
    "allowHeaders": ["Content-Type", "Authorization"],
    "exposeHeaders": ["X-Custom-Header"],
    "maxAge": 86400,
    "allowCredentials": true
  }
}
```

---

## Stages & Deployment

```
Deployment Model:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   API Definition ──deploy──▶ Stage                                      │
│                                                                          │
│   Stages:                                                                │
│   ├── dev   → https://xxx.execute-api.region.amazonaws.com/dev          │
│   ├── staging → https://xxx.execute-api.region.amazonaws.com/staging    │
│   └── prod  → https://xxx.execute-api.region.amazonaws.com/prod         │
│                                                                          │
│   Stage Variables:                                                       │
│   ├── Lambda alias: ${stageVariables.lambdaAlias}                       │
│   ├── Backend URL: ${stageVariables.backendUrl}                         │
│   └── Feature flags: ${stageVariables.featureEnabled}                   │
│                                                                          │
│   Canary Deployments:                                                    │
│   - Route % of traffic to canary                                        │
│   - Monitor metrics                                                     │
│   - Promote or rollback                                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Interview Discussion Points

### REST API vs HTTP API - When to use which?

```
Use REST API when:
- Need API key usage plans
- Need request/response transformation
- Need caching
- Need WAF integration
- Need detailed CloudWatch metrics
- Complex authorization needs

Use HTTP API when:
- Cost is primary concern (70% cheaper)
- Latency is critical
- Simple JWT/OIDC authorization
- Don't need transformation
- Native OpenID Connect

Feature Comparison:
┌────────────────────┬──────────┬──────────┐
│ Feature            │ REST API │ HTTP API │
├────────────────────┼──────────┼──────────┤
│ Caching            │ ✓        │ ✗        │
│ WAF                │ ✓        │ ✗        │
│ API Keys           │ ✓        │ ✗        │
│ Request validation │ ✓        │ ✗        │
│ Private APIs       │ ✓        │ ✓        │
│ JWT Authorizers    │ ✓        │ ✓        │
│ Lambda Authorizers │ ✓        │ ✓        │
│ Cost               │ Higher   │ Lower    │
│ Latency            │ Higher   │ Lower    │
└────────────────────┴──────────┴──────────┘
```

### How do you secure an API?

```
Defense in Depth:

1. Authentication
   - Cognito User Pools
   - Lambda Authorizers
   - IAM authentication

2. Authorization
   - Fine-grained permissions
   - Scope validation
   - Resource-based policies

3. Input Validation
   - Request validators
   - JSON schema validation
   - Parameter constraints

4. Rate Limiting
   - Throttling
   - Usage plans
   - Per-client limits

5. Encryption
   - TLS 1.2+
   - Certificate validation

6. WAF Integration
   - SQL injection protection
   - XSS protection
   - IP blocking
   - Rate-based rules

7. Monitoring
   - CloudWatch metrics
   - Access logs
   - CloudTrail
```

### How do you handle API versioning?

```
Versioning Strategies:

1. URL Path
   /v1/users
   /v2/users

   Pros: Clear, cacheable
   Cons: Multiple deployments

2. Stage Variables
   /dev → Lambda:dev
   /prod → Lambda:prod

   Same API, different backends

3. Custom Header
   X-API-Version: 2

   Pros: Clean URLs
   Cons: Less visible

4. Query Parameter
   /users?version=2

   Pros: Easy to test
   Cons: Cache complications

Recommended:
- URL path for major versions
- Backward compatibility for minor
- Deprecation policy (6-12 months)
- Clear documentation
```
