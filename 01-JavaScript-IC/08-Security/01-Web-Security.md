# Web Security for Frontend

## Overview

Understanding web security is essential for Lead-level engineers. This covers common vulnerabilities, prevention techniques, and security best practices.

---

## Cross-Site Scripting (XSS)

### Types of XSS

```javascript
// 1. Stored XSS - Malicious script stored on server
// Attacker submits comment: <script>stealCookies()</script>
// Stored in database, served to all users

// 2. Reflected XSS - Script in URL/request reflected back
// URL: https://site.com/search?q=<script>alert('XSS')</script>
// Server echoes query in response

// 3. DOM-based XSS - Script manipulates DOM directly
// URL: https://site.com/page#<script>alert('XSS')</script>
// Client-side JS reads hash and inserts into DOM
```

### XSS Prevention

```javascript
// 1. Never use innerHTML with user input
// BAD - vulnerable to XSS
element.innerHTML = userInput;

// GOOD - use textContent for text
element.textContent = userInput;

// 2. Escape HTML when rendering
function escapeHtml(str) {
  const div = document.createElement('div');
  div.textContent = str;
  return div.innerHTML;
}

// Or use a library
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userInput);

// 3. Use template literals safely
// BAD
const html = `<div>${userInput}</div>`;

// GOOD
const html = `<div>${escapeHtml(userInput)}</div>`;

// 4. React escapes by default
// Safe - React escapes automatically
<div>{userInput}</div>

// DANGEROUS - bypasses React's escaping
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// If needed, sanitize first
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />

// 5. Validate and sanitize URLs
function sanitizeUrl(url) {
  const parsed = new URL(url, window.location.origin);

  // Only allow http/https
  if (!['http:', 'https:'].includes(parsed.protocol)) {
    return '#';
  }

  return parsed.href;
}

// BAD - javascript: URLs can execute code
<a href={userUrl}>Link</a>

// GOOD
<a href={sanitizeUrl(userUrl)}>Link</a>
```

### Content Security Policy (CSP)

```javascript
// HTTP Header
// Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com

// Meta tag
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; script-src 'self' https://trusted.com">

// Common directives
const cspPolicy = {
  'default-src': "'self'",           // Default for all resources
  'script-src': "'self'",            // JavaScript sources
  'style-src': "'self' 'unsafe-inline'", // CSS sources
  'img-src': "'self' data: https:",  // Image sources
  'font-src': "'self'",              // Font sources
  'connect-src': "'self' https://api.example.com", // XHR/fetch
  'frame-src': "'none'",             // iframes
  'object-src': "'none'",            // Flash, etc.
  'base-uri': "'self'",              // <base> tag
  'form-action': "'self'",           // Form submissions
  'frame-ancestors': "'none'",       // Who can embed (like X-Frame-Options)
  'upgrade-insecure-requests': ''    // Upgrade HTTP to HTTPS
};

// Report violations
// Content-Security-Policy-Report-Only: ... ; report-uri /csp-report

// Nonce-based script allowlist
// Server generates nonce per request
// Content-Security-Policy: script-src 'nonce-abc123'
<script nonce="abc123">
  // This script will execute
</script>

// Hash-based allowlist
// Content-Security-Policy: script-src 'sha256-xyz...'
```

---

## Cross-Site Request Forgery (CSRF)

### How CSRF Works

```javascript
// Attacker's site contains:
<form action="https://bank.com/transfer" method="POST" id="csrf">
  <input type="hidden" name="to" value="attacker">
  <input type="hidden" name="amount" value="1000000">
</form>
<script>document.getElementById('csrf').submit();</script>

// If user is logged into bank.com, browser sends cookies
// Bank processes the request as legitimate
```

### CSRF Prevention

```javascript
// 1. CSRF Tokens
// Server generates unique token per session/request

// Include in forms
<form action="/transfer" method="POST">
  <input type="hidden" name="_csrf" value="token123">
  <!-- other fields -->
</form>

// Include in AJAX requests
fetch('/api/transfer', {
  method: 'POST',
  headers: {
    'X-CSRF-Token': getCsrfToken()
  },
  body: JSON.stringify(data)
});

// Server validates token matches session

// 2. SameSite Cookies
// Set-Cookie: sessionId=abc123; SameSite=Strict
// Strict: Cookie only sent for same-site requests
// Lax: Sent for top-level navigation (GET only)
// None: Sent for all requests (requires Secure)

document.cookie = 'session=abc; SameSite=Strict; Secure';

// 3. Check Origin/Referer headers
function validateOrigin(req) {
  const origin = req.headers.origin || req.headers.referer;
  const allowedOrigins = ['https://mysite.com'];

  if (!origin || !allowedOrigins.some(o => origin.startsWith(o))) {
    throw new Error('Invalid origin');
  }
}

// 4. Custom headers for AJAX
// Browsers don't allow custom headers in cross-origin requests
// without CORS preflight
fetch('/api/data', {
  headers: {
    'X-Requested-With': 'XMLHttpRequest'
  }
});

// Server checks for header
if (!req.headers['x-requested-with']) {
  return res.status(403).send('Forbidden');
}
```

---

## Cross-Origin Resource Sharing (CORS)

### How CORS Works

```javascript
// Browser enforces same-origin policy
// CORS allows servers to relax restrictions

// Simple requests (GET, POST with simple headers)
// Browser sends request with Origin header
// Server responds with Access-Control-Allow-Origin

// Preflight requests (PUT, DELETE, custom headers)
// Browser sends OPTIONS request first
// Server responds with allowed methods/headers
// Then browser sends actual request
```

### CORS Headers

```javascript
// Server-side (Express example)
app.use((req, res, next) => {
  // Allow specific origin
  res.header('Access-Control-Allow-Origin', 'https://trusted-site.com');

  // Allow credentials (cookies)
  res.header('Access-Control-Allow-Credentials', 'true');

  // Allowed methods
  res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');

  // Allowed headers
  res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization');

  // Cache preflight response
  res.header('Access-Control-Max-Age', '86400');

  // Expose headers to client
  res.header('Access-Control-Expose-Headers', 'X-Request-Id');

  // Handle preflight
  if (req.method === 'OPTIONS') {
    return res.sendStatus(204);
  }

  next();
});

// Client-side fetch
fetch('https://api.example.com/data', {
  credentials: 'include',  // Send cookies
  headers: {
    'Content-Type': 'application/json'
  }
});

// Common CORS errors:
// - Missing Access-Control-Allow-Origin
// - Origin not allowed
// - Credentials without specific origin (can't use *)
// - Missing preflight response
```

---

## Secure Authentication

### Token Storage

```javascript
// Storage options and security implications

// 1. localStorage - Accessible to JS, vulnerable to XSS
localStorage.setItem('token', jwt);
// Don't store sensitive tokens if XSS is possible

// 2. sessionStorage - Like localStorage, cleared on tab close
sessionStorage.setItem('token', jwt);

// 3. httpOnly Cookie - Not accessible to JS
// Set-Cookie: token=jwt; HttpOnly; Secure; SameSite=Strict
// Best for auth tokens, protected from XSS

// 4. In-memory (React state) - Lost on refresh
// Good for short-lived tokens, combine with refresh token in httpOnly cookie

// Recommended approach:
// - Access token: Short-lived, stored in memory
// - Refresh token: httpOnly cookie
```

### JWT Best Practices

```javascript
// JWT structure: header.payload.signature

// Client-side JWT handling
function parseJwt(token) {
  const base64Url = token.split('.')[1];
  const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
  return JSON.parse(atob(base64));
}

// Check expiration
function isTokenExpired(token) {
  const payload = parseJwt(token);
  return Date.now() >= payload.exp * 1000;
}

// Refresh token flow
async function fetchWithAuth(url, options = {}) {
  let token = getAccessToken();

  if (isTokenExpired(token)) {
    token = await refreshAccessToken();
  }

  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`
    }
  });
}

// Token refresh
async function refreshAccessToken() {
  const response = await fetch('/auth/refresh', {
    method: 'POST',
    credentials: 'include'  // Send httpOnly refresh token
  });

  if (!response.ok) {
    // Refresh failed, redirect to login
    window.location.href = '/login';
    throw new Error('Session expired');
  }

  const { accessToken } = await response.json();
  setAccessToken(accessToken);
  return accessToken;
}
```

### Password Security

```javascript
// Client-side (never store/transmit plain passwords longer than needed)

// Use HTTPS always
if (location.protocol !== 'https:') {
  location.replace(`https:${location.href.substring(location.protocol.length)}`);
}

// Password strength validation
function validatePassword(password) {
  const requirements = {
    minLength: password.length >= 12,
    hasUpper: /[A-Z]/.test(password),
    hasLower: /[a-z]/.test(password),
    hasNumber: /[0-9]/.test(password),
    hasSpecial: /[!@#$%^&*(),.?":{}|<>]/.test(password)
  };

  const score = Object.values(requirements).filter(Boolean).length;

  return {
    valid: score >= 4,
    requirements,
    score
  };
}

// Don't send password in URL
// BAD
fetch(`/login?password=${password}`);

// GOOD
fetch('/login', {
  method: 'POST',
  body: JSON.stringify({ password })
});
```

---

## Input Validation and Sanitization

### Client-Side Validation

```javascript
// Validate data format
function validateEmail(email) {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(email);
}

function validateUrl(url) {
  try {
    const parsed = new URL(url);
    return ['http:', 'https:'].includes(parsed.protocol);
  } catch {
    return false;
  }
}

// Sanitize input
function sanitizeInput(input) {
  return input
    .trim()
    .replace(/[<>]/g, '')  // Remove angle brackets
    .slice(0, 1000);        // Limit length
}

// Validate before use
function handleSubmit(data) {
  if (!validateEmail(data.email)) {
    throw new Error('Invalid email');
  }

  if (!data.name || data.name.length > 100) {
    throw new Error('Invalid name');
  }

  // Always validate on server too!
  return sendToServer(data);
}
```

### Preventing Injection

```javascript
// SQL Injection (server-side, but good to understand)
// BAD
const query = `SELECT * FROM users WHERE name = '${userInput}'`;

// GOOD - parameterized query
const query = 'SELECT * FROM users WHERE name = ?';
db.query(query, [userInput]);

// NoSQL Injection
// BAD - MongoDB
db.users.find({ username: userInput });
// If userInput = { $gt: '' }, matches all users

// GOOD - validate type
if (typeof userInput !== 'string') {
  throw new Error('Invalid input');
}
db.users.find({ username: userInput });

// Command Injection (server-side)
// BAD
exec(`convert ${userFilename} output.jpg`);

// GOOD - validate and escape
const safeFilename = userFilename.replace(/[^a-zA-Z0-9.-]/g, '');
exec(`convert ${safeFilename} output.jpg`);
```

---

## Secure Communication

### HTTPS Enforcement

```javascript
// Redirect HTTP to HTTPS
if (location.protocol !== 'https:' && location.hostname !== 'localhost') {
  location.replace(`https:${location.href.substring(location.protocol.length)}`);
}

// HSTS Header (server-side)
// Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

// Check for secure context
if (window.isSecureContext) {
  // Safe to use secure features
}

// Certificate pinning (mobile apps)
// Compare server certificate with known good certificate
```

### Subresource Integrity (SRI)

```html
<!-- Verify external scripts haven't been tampered with -->
<script
  src="https://cdn.example.com/library.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxGvIq3O4dYU="
  crossorigin="anonymous">
</script>

<link
  rel="stylesheet"
  href="https://cdn.example.com/styles.css"
  integrity="sha384-abc123..."
  crossorigin="anonymous">
```

```javascript
// Generate SRI hash
// openssl dgst -sha384 -binary file.js | openssl base64 -A

// Or use online tools/webpack plugins
```

---

## Clickjacking Prevention

```javascript
// X-Frame-Options Header (server-side)
// X-Frame-Options: DENY           // Never allow framing
// X-Frame-Options: SAMEORIGIN     // Only same origin can frame

// CSP frame-ancestors (preferred)
// Content-Security-Policy: frame-ancestors 'self'

// Client-side frame-busting (fallback)
if (window.top !== window.self) {
  window.top.location = window.self.location;
}

// Better frame-busting
<style id="antiClickjack">body { display: none !important; }</style>
<script>
  if (self === top) {
    document.getElementById('antiClickjack').remove();
  } else {
    top.location = self.location;
  }
</script>
```

---

## Sensitive Data Exposure

### Data Handling

```javascript
// Don't log sensitive data
// BAD
console.log('User login:', { email, password });

// GOOD
console.log('User login:', { email });

// Clear sensitive data from memory
function handleLogin(password) {
  try {
    await authenticate(password);
  } finally {
    password = null;  // Clear reference
  }
}

// Don't expose in error messages
// BAD
throw new Error(`User ${email} not found in database`);

// GOOD
throw new Error('Invalid credentials');

// Mask sensitive data in UI
function maskCreditCard(number) {
  return number.replace(/\d(?=\d{4})/g, '*');
}

function maskEmail(email) {
  const [local, domain] = email.split('@');
  return `${local[0]}***@${domain}`;
}
```

### Secure Storage

```javascript
// Don't store sensitive data unnecessarily
// Clear on logout
function logout() {
  localStorage.clear();
  sessionStorage.clear();
  // Clear cookies
  document.cookie.split(';').forEach(cookie => {
    const name = cookie.split('=')[0].trim();
    document.cookie = `${name}=; expires=Thu, 01 Jan 1970 00:00:00 GMT; path=/`;
  });
}

// Encrypt if you must store sensitive data
// Use Web Crypto API
async function encryptData(data, key) {
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const encoded = new TextEncoder().encode(JSON.stringify(data));

  const encrypted = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv },
    key,
    encoded
  );

  return {
    iv: Array.from(iv),
    data: Array.from(new Uint8Array(encrypted))
  };
}
```

---

## Security Headers Checklist

```
✓ Content-Security-Policy
✓ X-Content-Type-Options: nosniff
✓ X-Frame-Options: DENY
✓ X-XSS-Protection: 1; mode=block (legacy)
✓ Strict-Transport-Security
✓ Referrer-Policy: strict-origin-when-cross-origin
✓ Permissions-Policy (camera, microphone, geolocation)
```

---

## Interview Questions

### Q1: How do you prevent XSS attacks?

```javascript
// 1. Escape/encode user input before rendering
// 2. Use textContent instead of innerHTML
// 3. Implement Content Security Policy
// 4. Use frameworks that escape by default (React)
// 5. Sanitize HTML with DOMPurify if needed
// 6. Validate and sanitize URLs
// 7. Use HttpOnly cookies for sensitive tokens
```

### Q2: What's the difference between XSS and CSRF?

```javascript
// XSS: Attacker injects malicious script into your site
// - Executes in context of your site
// - Can steal data, perform actions as user
// - Prevented by: escaping, CSP, sanitization

// CSRF: Attacker tricks user into making unwanted request
// - Uses user's existing session/cookies
// - Limited to actions, can't read responses
// - Prevented by: CSRF tokens, SameSite cookies, origin checking
```

### Q3: Where should you store JWT tokens?

```javascript
// httpOnly cookies (recommended for auth tokens)
// - Protected from XSS
// - Vulnerable to CSRF (mitigate with SameSite)

// localStorage
// - Persists across sessions
// - Vulnerable to XSS

// Memory (React state)
// - Protected from XSS
// - Lost on refresh
// - Combine with refresh token in httpOnly cookie
```

### Q4: How does CORS protect users?

```javascript
// CORS is enforced by browsers to prevent:
// - Malicious sites reading data from other sites
// - Cross-origin requests without explicit permission

// Server must explicitly allow cross-origin requests
// Without CORS headers, browser blocks response

// CORS does NOT protect server from receiving requests
// (that's what CSRF protection is for)
```

---

## Key Takeaways

1. **Never trust user input** - Validate and sanitize everything
2. **Escape output** - Use textContent, framework escaping
3. **Implement CSP** - Defense in depth against XSS
4. **Use httpOnly cookies** - Protect tokens from XSS
5. **CSRF tokens** - Validate state-changing requests
6. **HTTPS everywhere** - Encrypt all communication
7. **Security headers** - Configure all relevant headers
8. **Principle of least privilege** - Minimize permissions
