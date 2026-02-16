# Load Balancing

## What is Load Balancing?

Distributing incoming network traffic across multiple servers to ensure no single server is overwhelmed, improving reliability and performance.

```
                         ┌─────────────┐
                         │    Load     │
       Clients ─────────▶│  Balancer   │
                         └──────┬──────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
       ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
       │  Server 1   │   │  Server 2   │   │  Server 3   │
       └─────────────┘   └─────────────┘   └─────────────┘
```

---

## Load Balancing Algorithms

### 1. Round Robin

Distributes requests sequentially across servers.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (cycle repeats)
```

**Pros:** Simple, even distribution
**Cons:** Doesn't consider server load or capacity

```nginx
# Nginx configuration
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

### 2. Weighted Round Robin

Assigns weights based on server capacity.

```
Server A (weight: 5) → Gets 5 requests
Server B (weight: 3) → Gets 3 requests
Server C (weight: 2) → Gets 2 requests
```

```nginx
upstream backend {
    server backend1.example.com weight=5;
    server backend2.example.com weight=3;
    server backend3.example.com weight=2;
}
```

### 3. Least Connections

Routes to server with fewest active connections.

```
Server A: 10 connections
Server B: 5 connections  ← New request goes here
Server C: 8 connections
```

**Best for:** Long-lived connections, varying request durations

```nginx
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
}
```

### 4. Weighted Least Connections

Combines least connections with server weights.

```
Effective load = active_connections / weight

Server A: 10 conn, weight 5 → load = 2.0
Server B: 5 conn, weight 2  → load = 2.5
Server C: 3 conn, weight 2  → load = 1.5 ← Choose this
```

### 5. IP Hash

Routes based on client IP (sticky sessions).

```javascript
function getServer(clientIP, servers) {
  const hash = hashFunction(clientIP);
  const index = hash % servers.length;
  return servers[index];
}
// Same client always goes to same server
```

```nginx
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
}
```

**Best for:** Session affinity requirements

### 6. Least Response Time

Routes to server with fastest response time + fewest connections.

```
Server A: 50ms avg, 10 connections
Server B: 30ms avg, 12 connections ← Likely chosen
Server C: 100ms avg, 5 connections
```

### 7. Random

Randomly selects a server (surprisingly effective).

```javascript
function getServer(servers) {
  return servers[Math.floor(Math.random() * servers.length)];
}
```

### 8. Consistent Hashing

For distributed caching/sharding - minimizes redistribution when servers change.

```javascript
class ConsistentHash {
  constructor(nodes, virtualNodes = 100) {
    this.ring = new Map();
    this.sortedKeys = [];

    for (const node of nodes) {
      for (let i = 0; i < virtualNodes; i++) {
        const key = this.hash(`${node}-${i}`);
        this.ring.set(key, node);
        this.sortedKeys.push(key);
      }
    }
    this.sortedKeys.sort((a, b) => a - b);
  }

  hash(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash);
  }

  getNode(key) {
    const hash = this.hash(key);
    for (const ringKey of this.sortedKeys) {
      if (hash <= ringKey) {
        return this.ring.get(ringKey);
      }
    }
    return this.ring.get(this.sortedKeys[0]);
  }
}
```

---

## Types of Load Balancers

### Layer 4 (Transport Layer)

Operates on TCP/UDP level - routes based on IP and port.

```
┌──────────────────────────────────────────────┐
│ Source IP: 192.168.1.1                       │
│ Dest IP: 10.0.0.1 (LB)                       │
│ Source Port: 54321                           │
│ Dest Port: 443                               │
│ [Encrypted payload - LB can't read this]    │
└──────────────────────────────────────────────┘
```

**Pros:**
- Very fast (no payload inspection)
- Lower latency
- Protocol agnostic

**Cons:**
- Can't make routing decisions based on content
- Limited to connection-level info

**Examples:** AWS NLB, HAProxy (TCP mode)

### Layer 7 (Application Layer)

Operates on HTTP level - can inspect headers, URLs, cookies.

```
┌──────────────────────────────────────────────┐
│ GET /api/users HTTP/1.1                      │
│ Host: api.example.com                        │
│ Cookie: session=abc123                       │
│ X-User-Region: us-west                       │
└──────────────────────────────────────────────┘
       │
       ▼
Route based on path, headers, cookies, etc.
```

**Pros:**
- Content-based routing
- URL rewriting
- SSL termination
- Compression
- Caching

**Cons:**
- Higher latency
- More resource intensive
- Must understand protocol

**Examples:** AWS ALB, Nginx, HAProxy (HTTP mode)

### Comparison

| Feature | Layer 4 | Layer 7 |
|---------|---------|---------|
| Speed | Faster | Slower |
| SSL Termination | No | Yes |
| Content Routing | No | Yes |
| WebSocket | Pass-through | Full support |
| Health Checks | TCP/Port | HTTP/Custom |
| Sticky Sessions | IP-based | Cookie-based |

---

## Health Checks

### Passive Health Checks

Monitor actual traffic for failures.

```nginx
upstream backend {
    server backend1.example.com max_fails=3 fail_timeout=30s;
    server backend2.example.com max_fails=3 fail_timeout=30s;
}
# After 3 failures, mark as down for 30 seconds
```

### Active Health Checks

Periodically probe servers.

```nginx
# Nginx Plus / OpenResty
upstream backend {
    zone backend 64k;
    server backend1.example.com;
    server backend2.example.com;

    health_check interval=5s fails=3 passes=2;
}

location /health {
    proxy_pass http://backend;
    health_check uri=/healthz;
}
```

### Health Check Endpoints

```javascript
// Simple health check
app.get('/healthz', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

// Deep health check
app.get('/healthz/ready', async (req, res) => {
  try {
    // Check database
    await db.query('SELECT 1');
    // Check Redis
    await redis.ping();
    // Check external dependencies
    await fetch('https://api.dependency.com/health');

    res.status(200).json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({
      status: 'not ready',
      error: error.message
    });
  }
});
```

---

## SSL/TLS Termination

### At Load Balancer

```
Client ──HTTPS──▶ LB ──HTTP──▶ Backend
                  │
            SSL terminated here

Pros:
- Centralized certificate management
- Offloads CPU from backends
- Easier to rotate certificates

Cons:
- Traffic unencrypted internally
- LB sees plaintext data
```

### End-to-End Encryption

```
Client ──HTTPS──▶ LB ──HTTPS──▶ Backend
                  │
            SSL passthrough or re-encryption

Pros:
- Traffic encrypted everywhere
- Compliance requirements

Cons:
- Higher CPU usage
- More complex certificate management
```

### SSL Offloading Configuration

```nginx
# Nginx SSL termination
server {
    listen 443 ssl;
    ssl_certificate /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://backend;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## Session Persistence (Sticky Sessions)

### Cookie-Based Affinity

```nginx
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    sticky cookie srv_id expires=1h domain=.example.com path=/;
}
```

### Application-Level Sessions

```javascript
// Better approach: externalize sessions
const session = require('express-session');
const RedisStore = require('connect-redis').default;

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'your-secret',
  resave: false,
  saveUninitialized: false,
  cookie: { secure: true, maxAge: 3600000 }
}));

// Now any server can handle any request
// Sessions stored in Redis, not in-memory
```

---

## Load Balancer Patterns

### Active-Passive (Failover)

```
           ┌─────────────┐
           │  Active LB  │◀──── All traffic
           └──────┬──────┘
                  │ heartbeat
           ┌──────▼──────┐
           │ Passive LB  │◀──── Takes over on failure
           └─────────────┘
```

### Active-Active

```
           ┌─────────────┐
Client ───▶│    DNS      │
           └──────┬──────┘
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
  ┌─────────────┐     ┌─────────────┐
  │    LB 1     │     │    LB 2     │
  └─────────────┘     └─────────────┘
        │                   │
        └─────────┬─────────┘
                  ▼
            Backend Servers
```

### Global Server Load Balancing (GSLB)

```
                    ┌───────────────┐
                    │   DNS/GSLB    │
                    └───────┬───────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
    ┌───────────┐     ┌───────────┐     ┌───────────┐
    │  US-East  │     │  US-West  │     │  EU-West  │
    │    LB     │     │    LB     │     │    LB     │
    └───────────┘     └───────────┘     └───────────┘
          │                 │                 │
    Backend Pool      Backend Pool      Backend Pool
```

---

## Cloud Load Balancers

### AWS

```yaml
Application Load Balancer (ALB):
  - Layer 7
  - HTTP/HTTPS/gRPC
  - Path-based routing
  - Host-based routing
  - WebSocket support

Network Load Balancer (NLB):
  - Layer 4
  - TCP/UDP/TLS
  - Ultra-low latency
  - Static IP support
  - Millions of RPS

Classic Load Balancer:
  - Legacy (Layer 4 + basic Layer 7)
  - Being phased out
```

### GCP

```yaml
HTTP(S) Load Balancer:
  - Global, Layer 7
  - Auto-scaling
  - CDN integration

Network Load Balancer:
  - Regional, Layer 4
  - Pass-through
  - UDP support
```

---

## Configuration Examples

### HAProxy

```haproxy
global
    maxconn 50000

defaults
    mode http
    timeout connect 5s
    timeout client 50s
    timeout server 50s

frontend http_front
    bind *:80
    bind *:443 ssl crt /etc/ssl/cert.pem
    default_backend http_back

    # Route based on path
    acl is_api path_beg /api
    use_backend api_back if is_api

backend http_back
    balance roundrobin
    option httpchk GET /healthz
    server web1 192.168.1.1:8080 check
    server web2 192.168.1.2:8080 check
    server web3 192.168.1.3:8080 check backup

backend api_back
    balance leastconn
    server api1 192.168.1.10:3000 check weight 5
    server api2 192.168.1.11:3000 check weight 3
```

### Nginx

```nginx
upstream web_backend {
    least_conn;
    server 192.168.1.1:8080 weight=5;
    server 192.168.1.2:8080 weight=3;
    server 192.168.1.3:8080 backup;

    keepalive 32;
}

upstream api_backend {
    server 192.168.1.10:3000;
    server 192.168.1.11:3000;
}

server {
    listen 80;
    listen 443 ssl;

    location / {
        proxy_pass http://web_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    location /api {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## Interview Questions

### Q: When would you use Layer 4 vs Layer 7 load balancing?

**Layer 4:**
- Need lowest latency
- TCP/UDP protocols
- Don't need content inspection
- Simple port-based routing

**Layer 7:**
- Need path/header-based routing
- SSL termination
- WebSocket support
- Content-based decisions

### Q: How do you handle load balancer as a single point of failure?

- Deploy in pairs (active-passive or active-active)
- Use DNS-based failover
- Cloud managed LBs have built-in HA
- Health checking between LB pairs
- Virtual IP (VIP) failover with keepalived

### Q: How do sticky sessions impact scalability?

- Limit flexibility in routing
- Can cause uneven load distribution
- Server failure loses all sessions
- Better to externalize state (Redis)
- Use only when absolutely necessary
