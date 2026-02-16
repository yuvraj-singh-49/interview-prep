# Security at Scale

## Overview

```
Security Layers:
┌─────────────────────────────────────────────────────────────────┐
│                      Application Security                        │
│        (Auth, Authorization, Input Validation, CSRF)            │
├─────────────────────────────────────────────────────────────────┤
│                      Transport Security                          │
│              (TLS, Certificate Management, mTLS)                 │
├─────────────────────────────────────────────────────────────────┤
│                      Network Security                            │
│           (Firewalls, VPC, DDoS Protection, WAF)                │
├─────────────────────────────────────────────────────────────────┤
│                      Infrastructure Security                     │
│          (IAM, Secrets Management, Encryption at Rest)          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Authentication

### JWT Authentication

```javascript
const jwt = require('jsonwebtoken');

class AuthService {
  constructor() {
    this.accessTokenSecret = process.env.JWT_ACCESS_SECRET;
    this.refreshTokenSecret = process.env.JWT_REFRESH_SECRET;
    this.accessTokenExpiry = '15m';
    this.refreshTokenExpiry = '7d';
  }

  async login(email, password) {
    const user = await this.validateCredentials(email, password);

    const accessToken = this.generateAccessToken(user);
    const refreshToken = this.generateRefreshToken(user);

    // Store refresh token hash
    await this.storeRefreshToken(user.id, refreshToken);

    return { accessToken, refreshToken, user };
  }

  generateAccessToken(user) {
    return jwt.sign(
      {
        sub: user.id,
        email: user.email,
        roles: user.roles,
        type: 'access'
      },
      this.accessTokenSecret,
      { expiresIn: this.accessTokenExpiry }
    );
  }

  generateRefreshToken(user) {
    return jwt.sign(
      {
        sub: user.id,
        type: 'refresh',
        jti: generateId()  // Unique token ID for revocation
      },
      this.refreshTokenSecret,
      { expiresIn: this.refreshTokenExpiry }
    );
  }

  async verifyAccessToken(token) {
    try {
      const payload = jwt.verify(token, this.accessTokenSecret);

      if (payload.type !== 'access') {
        throw new Error('Invalid token type');
      }

      // Check if user is still active
      const user = await this.userService.getUser(payload.sub);
      if (!user || user.status !== 'active') {
        throw new Error('User not active');
      }

      return payload;
    } catch (error) {
      throw new AuthenticationError('Invalid token');
    }
  }

  async refresh(refreshToken) {
    const payload = jwt.verify(refreshToken, this.refreshTokenSecret);

    // Verify token exists in storage (not revoked)
    const stored = await this.getStoredRefreshToken(payload.sub, payload.jti);
    if (!stored) {
      throw new AuthenticationError('Token revoked');
    }

    const user = await this.userService.getUser(payload.sub);
    const newAccessToken = this.generateAccessToken(user);

    return { accessToken: newAccessToken };
  }

  async logout(userId, refreshToken) {
    const payload = jwt.decode(refreshToken);
    await this.revokeRefreshToken(userId, payload.jti);
  }

  async logoutAll(userId) {
    // Revoke all refresh tokens for user
    await this.revokeAllRefreshTokens(userId);
  }
}
```

### OAuth 2.0 / OIDC

```javascript
// OAuth 2.0 Authorization Code Flow
class OAuthService {
  async initiateOAuth(provider, redirectUri) {
    const state = generateSecureRandom(32);
    const codeVerifier = generateSecureRandom(64);  // PKCE
    const codeChallenge = base64url(sha256(codeVerifier));

    // Store state and verifier
    await redis.setex(`oauth:${state}`, 600, JSON.stringify({
      provider,
      redirectUri,
      codeVerifier
    }));

    const authUrl = new URL(this.providers[provider].authorizationUrl);
    authUrl.searchParams.set('client_id', this.providers[provider].clientId);
    authUrl.searchParams.set('redirect_uri', redirectUri);
    authUrl.searchParams.set('response_type', 'code');
    authUrl.searchParams.set('scope', 'openid profile email');
    authUrl.searchParams.set('state', state);
    authUrl.searchParams.set('code_challenge', codeChallenge);
    authUrl.searchParams.set('code_challenge_method', 'S256');

    return authUrl.toString();
  }

  async handleCallback(code, state) {
    // Verify state
    const storedData = await redis.get(`oauth:${state}`);
    if (!storedData) {
      throw new Error('Invalid state');
    }

    const { provider, redirectUri, codeVerifier } = JSON.parse(storedData);
    await redis.del(`oauth:${state}`);

    // Exchange code for tokens
    const tokens = await this.exchangeCode(provider, code, redirectUri, codeVerifier);

    // Get user info
    const userInfo = await this.getUserInfo(provider, tokens.access_token);

    // Find or create user
    let user = await this.userService.findByOAuthId(provider, userInfo.sub);
    if (!user) {
      user = await this.userService.createFromOAuth(provider, userInfo);
    }

    // Generate our tokens
    return this.authService.login(user);
  }
}
```

### API Key Authentication

```javascript
class APIKeyService {
  async generateKey(userId, name, scopes, expiresAt = null) {
    const keyId = generateId();
    const secret = generateSecureRandom(32);

    // Store hashed secret
    const hashedSecret = await bcrypt.hash(secret, 12);

    await db.apiKeys.create({
      id: keyId,
      userId,
      name,
      hashedSecret,
      scopes,
      expiresAt,
      lastUsedAt: null,
      createdAt: new Date()
    });

    // Return full key only once (prefix_keyId_secret)
    return {
      key: `sk_${keyId}_${secret}`,
      id: keyId
    };
  }

  async validateKey(apiKey) {
    // Parse key
    const parts = apiKey.split('_');
    if (parts.length !== 3 || parts[0] !== 'sk') {
      throw new AuthenticationError('Invalid API key format');
    }

    const [, keyId, secret] = parts;

    // Get key from database
    const key = await db.apiKeys.findById(keyId);
    if (!key) {
      throw new AuthenticationError('Invalid API key');
    }

    // Check expiration
    if (key.expiresAt && key.expiresAt < new Date()) {
      throw new AuthenticationError('API key expired');
    }

    // Verify secret
    const valid = await bcrypt.compare(secret, key.hashedSecret);
    if (!valid) {
      throw new AuthenticationError('Invalid API key');
    }

    // Update last used
    await db.apiKeys.update(keyId, { lastUsedAt: new Date() });

    return key;
  }
}

// Middleware
async function apiKeyAuth(req, res, next) {
  const apiKey = req.headers['x-api-key'];
  if (!apiKey) {
    return next();  // Fall through to other auth methods
  }

  try {
    const key = await apiKeyService.validateKey(apiKey);
    req.apiKey = key;
    req.scopes = key.scopes;
    next();
  } catch (error) {
    res.status(401).json({ error: error.message });
  }
}
```

---

## Authorization

### Role-Based Access Control (RBAC)

```javascript
// Permission definitions
const PERMISSIONS = {
  // Resource-based permissions
  'users:read': 'Read user data',
  'users:write': 'Create/update users',
  'users:delete': 'Delete users',
  'orders:read': 'Read orders',
  'orders:write': 'Create/update orders',
  'orders:refund': 'Process refunds',
  'admin:access': 'Access admin panel'
};

// Role definitions
const ROLES = {
  viewer: ['users:read', 'orders:read'],
  editor: ['users:read', 'users:write', 'orders:read', 'orders:write'],
  admin: Object.keys(PERMISSIONS),
  support: ['users:read', 'orders:read', 'orders:refund']
};

class AuthorizationService {
  hasPermission(user, permission) {
    const userPermissions = this.getUserPermissions(user);
    return userPermissions.includes(permission);
  }

  getUserPermissions(user) {
    const permissions = new Set();

    for (const role of user.roles) {
      for (const permission of ROLES[role] || []) {
        permissions.add(permission);
      }
    }

    // Add any direct user permissions
    for (const permission of user.permissions || []) {
      permissions.add(permission);
    }

    return Array.from(permissions);
  }

  // Express middleware
  requirePermission(permission) {
    return (req, res, next) => {
      if (!req.user) {
        return res.status(401).json({ error: 'Authentication required' });
      }

      if (!this.hasPermission(req.user, permission)) {
        return res.status(403).json({ error: 'Insufficient permissions' });
      }

      next();
    };
  }
}

// Usage
app.get('/api/users',
  authenticate,
  authz.requirePermission('users:read'),
  usersController.list
);

app.post('/api/orders/:id/refund',
  authenticate,
  authz.requirePermission('orders:refund'),
  ordersController.refund
);
```

### Attribute-Based Access Control (ABAC)

```javascript
class ABACService {
  evaluate(subject, action, resource, context) {
    const policies = this.getPoliciesForAction(action);

    for (const policy of policies) {
      const result = this.evaluatePolicy(policy, { subject, action, resource, context });

      if (result === 'DENY') return false;
      if (result === 'ALLOW') return true;
    }

    return false;  // Default deny
  }

  evaluatePolicy(policy, request) {
    // Evaluate conditions
    for (const condition of policy.conditions) {
      if (!this.evaluateCondition(condition, request)) {
        return 'NOT_APPLICABLE';
      }
    }

    return policy.effect;
  }

  evaluateCondition(condition, request) {
    const { attribute, operator, value } = condition;

    // Get attribute value
    const attrValue = this.getAttribute(attribute, request);

    switch (operator) {
      case 'equals':
        return attrValue === value;
      case 'in':
        return value.includes(attrValue);
      case 'contains':
        return attrValue?.includes(value);
      case 'greaterThan':
        return attrValue > value;
      case 'lessThan':
        return attrValue < value;
      default:
        return false;
    }
  }
}

// Example policies
const policies = [
  {
    name: 'OwnDataAccess',
    effect: 'ALLOW',
    actions: ['read', 'update'],
    resources: ['user/*'],
    conditions: [
      { attribute: 'subject.id', operator: 'equals', value: 'resource.ownerId' }
    ]
  },
  {
    name: 'AdminFullAccess',
    effect: 'ALLOW',
    actions: ['*'],
    resources: ['*'],
    conditions: [
      { attribute: 'subject.role', operator: 'equals', value: 'admin' }
    ]
  },
  {
    name: 'BusinessHoursOnly',
    effect: 'DENY',
    actions: ['delete'],
    resources: ['*'],
    conditions: [
      { attribute: 'context.hour', operator: 'lessThan', value: 9 },
      { attribute: 'context.hour', operator: 'greaterThan', value: 17 }
    ]
  }
];
```

---

## Input Validation & Sanitization

```javascript
const Joi = require('joi');
const DOMPurify = require('isomorphic-dompurify');

// Schema validation
const userSchema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().min(8).max(128).required(),
  name: Joi.string().min(1).max(100).required(),
  age: Joi.number().integer().min(0).max(150).optional()
});

// Validation middleware
function validate(schema) {
  return (req, res, next) => {
    const { error, value } = schema.validate(req.body, {
      abortEarly: false,
      stripUnknown: true
    });

    if (error) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.details.map(d => ({
          field: d.path.join('.'),
          message: d.message
        }))
      });
    }

    req.body = value;
    next();
  };
}

// HTML sanitization
function sanitizeHTML(dirty) {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    ALLOWED_ATTR: ['href', 'target']
  });
}

// SQL injection prevention (parameterized queries)
async function getUser(userId) {
  // GOOD: Parameterized query
  const result = await db.query(
    'SELECT * FROM users WHERE id = $1',
    [userId]
  );

  // BAD: String concatenation (SQL injection vulnerable)
  // const result = await db.query(`SELECT * FROM users WHERE id = '${userId}'`);

  return result.rows[0];
}

// NoSQL injection prevention
async function findUser(query) {
  // Sanitize query operators
  const sanitizedQuery = {};
  for (const [key, value] of Object.entries(query)) {
    if (typeof value === 'object' && value !== null) {
      // Remove MongoDB operators
      const sanitized = {};
      for (const [k, v] of Object.entries(value)) {
        if (!k.startsWith('$')) {
          sanitized[k] = v;
        }
      }
      sanitizedQuery[key] = sanitized;
    } else {
      sanitizedQuery[key] = value;
    }
  }

  return db.users.find(sanitizedQuery);
}
```

---

## Rate Limiting

### Distributed Rate Limiting

```javascript
class DistributedRateLimiter {
  constructor(redis) {
    this.redis = redis;
  }

  // Sliding window counter
  async isAllowed(key, limit, windowSeconds) {
    const now = Date.now();
    const windowStart = now - (windowSeconds * 1000);

    const multi = this.redis.multi();

    // Remove old entries
    multi.zremrangebyscore(key, 0, windowStart);

    // Add current request
    multi.zadd(key, now, `${now}-${Math.random()}`);

    // Count requests in window
    multi.zcard(key);

    // Set expiry
    multi.expire(key, windowSeconds);

    const results = await multi.exec();
    const count = results[2][1];

    return {
      allowed: count <= limit,
      remaining: Math.max(0, limit - count),
      resetAt: new Date(now + windowSeconds * 1000)
    };
  }
}

// Tiered rate limiting
class TieredRateLimiter {
  constructor(redis) {
    this.redis = redis;
    this.tiers = {
      anonymous: { perSecond: 1, perMinute: 30, perHour: 500 },
      authenticated: { perSecond: 10, perMinute: 300, perHour: 5000 },
      premium: { perSecond: 50, perMinute: 1500, perHour: 25000 }
    };
  }

  async checkLimit(userId, tier) {
    const limits = this.tiers[tier] || this.tiers.anonymous;
    const baseKey = `ratelimit:${userId}`;

    const checks = await Promise.all([
      this.checkWindow(`${baseKey}:sec`, limits.perSecond, 1),
      this.checkWindow(`${baseKey}:min`, limits.perMinute, 60),
      this.checkWindow(`${baseKey}:hour`, limits.perHour, 3600)
    ]);

    const blocked = checks.find(c => !c.allowed);
    if (blocked) {
      return {
        allowed: false,
        retryAfter: blocked.resetAt,
        limit: blocked.limit,
        window: blocked.window
      };
    }

    return { allowed: true };
  }
}

// Rate limit middleware
function rateLimit(options) {
  const limiter = new TieredRateLimiter(redis);

  return async (req, res, next) => {
    const key = req.user?.id || req.ip;
    const tier = req.user?.tier || 'anonymous';

    const result = await limiter.checkLimit(key, tier);

    // Set rate limit headers
    res.set('X-RateLimit-Limit', result.limit);
    res.set('X-RateLimit-Remaining', result.remaining);
    res.set('X-RateLimit-Reset', result.resetAt?.toISOString());

    if (!result.allowed) {
      res.set('Retry-After', Math.ceil((result.retryAfter - Date.now()) / 1000));
      return res.status(429).json({
        error: 'Too many requests',
        retryAfter: result.retryAfter
      });
    }

    next();
  };
}
```

---

## Secrets Management

```javascript
// Using HashiCorp Vault
const vault = require('node-vault')({
  apiVersion: 'v1',
  endpoint: process.env.VAULT_ADDR,
  token: process.env.VAULT_TOKEN
});

class SecretsManager {
  constructor(vault) {
    this.vault = vault;
    this.cache = new Map();
    this.cacheTTL = 300000;  // 5 minutes
  }

  async getSecret(path) {
    // Check cache
    const cached = this.cache.get(path);
    if (cached && cached.expiresAt > Date.now()) {
      return cached.value;
    }

    // Fetch from Vault
    const result = await this.vault.read(`secret/data/${path}`);
    const secret = result.data.data;

    // Cache
    this.cache.set(path, {
      value: secret,
      expiresAt: Date.now() + this.cacheTTL
    });

    return secret;
  }

  async getDatabaseCredentials() {
    // Dynamic secrets - Vault generates temporary credentials
    const result = await this.vault.read('database/creds/myapp-role');

    return {
      username: result.data.username,
      password: result.data.password,
      leaseDuration: result.lease_duration
    };
  }
}

// Environment-based configuration
class ConfigService {
  async loadSecrets() {
    if (process.env.NODE_ENV === 'production') {
      // Load from Vault
      const secrets = await secretsManager.getSecret('myapp/config');
      Object.assign(process.env, secrets);
    }
    // In development, use .env file
  }
}
```

---

## Encryption

### Encryption at Rest

```javascript
const crypto = require('crypto');

class EncryptionService {
  constructor(masterKey) {
    this.masterKey = Buffer.from(masterKey, 'hex');
    this.algorithm = 'aes-256-gcm';
  }

  // Encrypt with authenticated encryption
  encrypt(plaintext) {
    const iv = crypto.randomBytes(12);
    const cipher = crypto.createCipheriv(this.algorithm, this.masterKey, iv);

    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');

    const authTag = cipher.getAuthTag().toString('hex');

    // Return IV + AuthTag + Ciphertext
    return `${iv.toString('hex')}:${authTag}:${encrypted}`;
  }

  decrypt(encryptedData) {
    const [ivHex, authTagHex, ciphertext] = encryptedData.split(':');

    const iv = Buffer.from(ivHex, 'hex');
    const authTag = Buffer.from(authTagHex, 'hex');

    const decipher = crypto.createDecipheriv(this.algorithm, this.masterKey, iv);
    decipher.setAuthTag(authTag);

    let decrypted = decipher.update(ciphertext, 'hex', 'utf8');
    decrypted += decipher.final('utf8');

    return decrypted;
  }

  // Field-level encryption
  encryptFields(data, fieldsToEncrypt) {
    const encrypted = { ...data };

    for (const field of fieldsToEncrypt) {
      if (encrypted[field]) {
        encrypted[field] = this.encrypt(JSON.stringify(encrypted[field]));
        encrypted[`${field}_encrypted`] = true;
      }
    }

    return encrypted;
  }
}

// Key rotation
class KeyRotationService {
  async rotateKey(oldKey, newKey) {
    // Get all encrypted records
    const records = await db.query('SELECT * FROM users WHERE ssn_encrypted = true');

    const oldEncryption = new EncryptionService(oldKey);
    const newEncryption = new EncryptionService(newKey);

    for (const record of records) {
      // Decrypt with old key
      const decrypted = oldEncryption.decrypt(record.ssn);

      // Re-encrypt with new key
      const reencrypted = newEncryption.encrypt(decrypted);

      // Update record
      await db.query('UPDATE users SET ssn = $1 WHERE id = $2', [reencrypted, record.id]);
    }
  }
}
```

---

## DDoS Protection

```javascript
// Multi-layer DDoS protection
class DDoSProtection {
  constructor() {
    this.ipRateLimit = new Map();
    this.blacklist = new Set();
    this.challengeTokens = new Map();
  }

  // Layer 1: IP reputation
  async checkIPReputation(ip) {
    if (this.blacklist.has(ip)) {
      return { blocked: true, reason: 'IP blacklisted' };
    }

    // Check against threat intelligence
    const reputation = await threatIntel.check(ip);
    if (reputation.score > 0.8) {
      this.blacklist.add(ip);
      return { blocked: true, reason: 'Bad reputation' };
    }

    return { blocked: false };
  }

  // Layer 2: Rate limiting by IP
  async checkRateLimit(ip) {
    const now = Date.now();
    const windowStart = now - 60000;  // 1 minute

    let ipData = this.ipRateLimit.get(ip);
    if (!ipData) {
      ipData = { requests: [] };
      this.ipRateLimit.set(ip, ipData);
    }

    // Clean old requests
    ipData.requests = ipData.requests.filter(t => t > windowStart);

    // Check limit
    if (ipData.requests.length > 1000) {
      return { blocked: true, reason: 'Rate limit exceeded' };
    }

    ipData.requests.push(now);
    return { blocked: false };
  }

  // Layer 3: Challenge-response for suspicious traffic
  async issueChallenge(ip) {
    const token = generateSecureRandom(32);
    const challenge = {
      token,
      answer: solvePuzzle(token),  // Proof of work
      expiresAt: Date.now() + 60000
    };

    this.challengeTokens.set(ip, challenge);
    return { challenge: token };
  }

  verifyChallenge(ip, answer) {
    const challenge = this.challengeTokens.get(ip);
    if (!challenge || challenge.expiresAt < Date.now()) {
      return false;
    }

    const valid = challenge.answer === answer;
    if (valid) {
      this.challengeTokens.delete(ip);
    }
    return valid;
  }
}

// WAF rules
const wafRules = [
  {
    name: 'SQL Injection',
    pattern: /(\b(select|insert|update|delete|drop|union|exec)\b)/i,
    action: 'block'
  },
  {
    name: 'XSS',
    pattern: /<script[^>]*>|javascript:/i,
    action: 'block'
  },
  {
    name: 'Path Traversal',
    pattern: /\.\.\//,
    action: 'block'
  }
];

function applyWAFRules(req) {
  const payload = JSON.stringify(req.body) + req.url + JSON.stringify(req.query);

  for (const rule of wafRules) {
    if (rule.pattern.test(payload)) {
      return { blocked: true, rule: rule.name };
    }
  }

  return { blocked: false };
}
```

---

## Security Headers

```javascript
const helmet = require('helmet');

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'strict-dynamic'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.example.com"],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      frameAncestors: ["'none'"],
      upgradeInsecureRequests: []
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  crossOriginEmbedderPolicy: true,
  crossOriginOpenerPolicy: { policy: 'same-origin' },
  crossOriginResourcePolicy: { policy: 'same-origin' }
}));

// Additional security headers
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
  next();
});
```

---

## Audit Logging

```javascript
class AuditLogger {
  async log(event) {
    const auditEntry = {
      id: generateId(),
      timestamp: new Date().toISOString(),
      actor: {
        id: event.userId,
        ip: event.ip,
        userAgent: event.userAgent
      },
      action: event.action,
      resource: {
        type: event.resourceType,
        id: event.resourceId
      },
      outcome: event.outcome,  // 'success' | 'failure'
      metadata: event.metadata
    };

    // Write to immutable audit log
    await this.writeToAuditLog(auditEntry);

    // Real-time alerting for suspicious activity
    if (this.isSuspicious(auditEntry)) {
      await this.alertSecurityTeam(auditEntry);
    }
  }

  isSuspicious(entry) {
    // Multiple failed logins
    // Access from unusual location
    // Privilege escalation attempts
    // Access to sensitive resources
    return false;
  }

  async writeToAuditLog(entry) {
    // Append-only storage (e.g., S3, Kafka, dedicated audit DB)
    await db.auditLogs.create(entry);

    // Also send to SIEM
    await siem.ingest(entry);
  }
}

// Middleware
function auditMiddleware(action) {
  return async (req, res, next) => {
    const originalSend = res.send;

    res.send = function(body) {
      auditLogger.log({
        userId: req.user?.id,
        ip: req.ip,
        userAgent: req.headers['user-agent'],
        action,
        resourceType: req.params.resourceType,
        resourceId: req.params.id,
        outcome: res.statusCode < 400 ? 'success' : 'failure',
        metadata: {
          method: req.method,
          path: req.path,
          statusCode: res.statusCode
        }
      });

      return originalSend.call(this, body);
    };

    next();
  };
}
```

---

## Interview Discussion Points

### How to prevent common attacks?

| Attack | Prevention |
|--------|------------|
| SQL Injection | Parameterized queries, ORM, input validation |
| XSS | Output encoding, CSP, sanitization |
| CSRF | CSRF tokens, SameSite cookies |
| SSRF | Allowlist URLs, disable redirects |
| XXE | Disable external entities |
| Injection | Input validation, least privilege |

### How to implement zero-trust?

1. **Never trust, always verify** - Authenticate every request
2. **Least privilege access** - Minimal permissions
3. **Micro-segmentation** - Network isolation
4. **Continuous validation** - Re-verify periodically
5. **Assume breach** - Encrypt everything, audit everything

### How to handle security at scale?

1. **Automated scanning** - SAST/DAST in CI/CD
2. **Centralized auth** - Single IAM service
3. **Security as code** - Infrastructure policies
4. **Monitoring** - SIEM, anomaly detection
5. **Incident response** - Playbooks, automation
