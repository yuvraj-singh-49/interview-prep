# Networking Essentials

## DNS (Domain Name System)

### How DNS Works

```
User types: www.example.com

1. Browser Cache → Check local cache
2. OS Cache → Check /etc/hosts
3. Resolver (ISP) → Query recursive resolver
4. Root Server → "Ask .com TLD server"
5. TLD Server → "Ask example.com NS"
6. Authoritative NS → "IP is 93.184.216.34"
7. Response cached at each level
```

### DNS Record Types

```
A       - Maps domain to IPv4 address
          example.com → 93.184.216.34

AAAA    - Maps domain to IPv6 address
          example.com → 2606:2800:220:1:248:1893:25c8:1946

CNAME   - Alias to another domain
          www.example.com → example.com

MX      - Mail server
          example.com → mail.example.com (priority 10)

NS      - Nameserver
          example.com → ns1.example.com

TXT     - Text records (SPF, DKIM, verification)
          example.com → "v=spf1 include:_spf.google.com ~all"

SRV     - Service location
          _sip._tcp.example.com → sipserver.example.com:5060
```

### DNS-Based Load Balancing

```
Round Robin DNS:
example.com → 10.0.0.1
example.com → 10.0.0.2
example.com → 10.0.0.3
(Returns different IP each query)

Weighted DNS:
example.com → 10.0.0.1 (weight 70%)
example.com → 10.0.0.2 (weight 30%)

Geo DNS:
US users    → us-east.example.com
EU users    → eu-west.example.com
Asia users  → ap-south.example.com
```

### DNS TTL

```
High TTL (86400 = 24 hours):
- Less DNS traffic
- Slower failover
- Good for stable services

Low TTL (60 = 1 minute):
- Faster failover
- More DNS traffic
- Good for dynamic services
```

---

## CDN (Content Delivery Network)

### CDN Architecture

```
                    Origin Server
                    (San Francisco)
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ Edge PoP │   │ Edge PoP │   │ Edge PoP │
    │  Tokyo   │   │  London  │   │ São Paulo│
    └──────────┘   └──────────┘   └──────────┘
          ▲               ▲               ▲
     Asia Users      EU Users      SA Users
       (10ms)         (20ms)        (30ms)
```

### What CDNs Cache

```
Static Content:
- Images, CSS, JavaScript
- Fonts, videos
- PDF documents

Dynamic Content (with Edge Compute):
- Personalized pages
- API responses (with short TTL)
- A/B test variants
```

### CDN Configuration

```javascript
// Cache-Control headers for CDN
res.set('Cache-Control', 'public, max-age=31536000, immutable');
// static assets with hash in filename

res.set('Cache-Control', 'public, s-maxage=3600, max-age=60');
// CDN caches 1 hour, browser caches 1 minute

res.set('Cache-Control', 'private, no-store');
// Don't cache at CDN (user-specific data)
```

### CDN Benefits

```
Performance:
- Reduced latency (content closer to users)
- Offloads origin server
- Better Time to First Byte (TTFB)

Reliability:
- DDoS protection
- Origin shielding
- Automatic failover

Cost:
- Reduced bandwidth from origin
- Cheaper edge bandwidth
```

---

## HTTP/HTTPS

### HTTP Methods

```
GET     - Retrieve resource (safe, idempotent)
POST    - Create resource (not idempotent)
PUT     - Replace resource (idempotent)
PATCH   - Partial update (not idempotent)
DELETE  - Remove resource (idempotent)
HEAD    - GET without body (metadata only)
OPTIONS - Get allowed methods (CORS preflight)
```

### HTTP Status Codes

```
2xx Success:
200 OK                    - Success
201 Created               - Resource created
204 No Content            - Success, no body

3xx Redirection:
301 Moved Permanently     - Resource moved (cached)
302 Found                 - Temporary redirect
304 Not Modified          - Use cached version
307 Temporary Redirect    - Keep method
308 Permanent Redirect    - Keep method

4xx Client Error:
400 Bad Request           - Invalid request
401 Unauthorized          - Authentication required
403 Forbidden             - Not permitted
404 Not Found             - Resource doesn't exist
409 Conflict              - State conflict
429 Too Many Requests     - Rate limited

5xx Server Error:
500 Internal Server Error - Server error
502 Bad Gateway           - Upstream error
503 Service Unavailable   - Temporarily down
504 Gateway Timeout       - Upstream timeout
```

### HTTP/2 vs HTTP/1.1

```
HTTP/1.1:
- One request per connection
- Head-of-line blocking
- Headers sent as text
- No server push

HTTP/2:
- Multiplexed streams (parallel requests on single connection)
- Binary protocol (more efficient)
- Header compression (HPACK)
- Server push
- Stream prioritization
```

### HTTP/3 (QUIC)

```
Built on UDP instead of TCP:
- 0-RTT connection establishment
- No head-of-line blocking
- Better performance on lossy networks
- Connection migration (change IP/port)
```

---

## TCP vs UDP

### TCP (Transmission Control Protocol)

```
Features:
- Connection-oriented (3-way handshake)
- Reliable delivery (retransmission)
- Ordered packets
- Flow control
- Congestion control

3-Way Handshake:
Client → SYN → Server
Client ← SYN-ACK ← Server
Client → ACK → Server
(Connection established)

Use for:
- HTTP/HTTPS
- Email (SMTP, IMAP)
- File transfer (FTP)
- SSH
- Database connections
```

### UDP (User Datagram Protocol)

```
Features:
- Connectionless
- No delivery guarantee
- No ordering
- No congestion control
- Lower latency

Use for:
- Real-time gaming
- Video streaming
- VoIP
- DNS queries
- Live broadcasts
```

### Comparison

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Yes | No |
| Reliability | Guaranteed | Best effort |
| Ordering | Yes | No |
| Speed | Slower | Faster |
| Overhead | Higher | Lower |

---

## WebSockets

### WebSocket Handshake

```
Client → HTTP Upgrade Request:
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server → HTTP 101 Response:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

(Now bidirectional communication)
```

### WebSocket vs HTTP Polling

```
HTTP Polling:
Client → Request → Server
Client ← Response ← Server
(Wait 1 second)
Client → Request → Server
... repeat forever

WebSocket:
Client ←→ Persistent Connection ←→ Server
(Messages flow both directions anytime)
```

### WebSocket Implementation

```javascript
// Server (Node.js with ws)
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  ws.on('message', (message) => {
    console.log('Received:', message);
    ws.send('Echo: ' + message);
  });

  ws.on('close', () => {
    console.log('Client disconnected');
  });
});

// Client
const ws = new WebSocket('ws://localhost:8080');

ws.onopen = () => {
  ws.send('Hello Server!');
};

ws.onmessage = (event) => {
  console.log('Received:', event.data);
};
```

---

## SSL/TLS

### TLS Handshake

```
1. Client Hello
   - Supported TLS versions
   - Supported cipher suites
   - Random number

2. Server Hello
   - Selected TLS version
   - Selected cipher suite
   - Random number
   - Server certificate

3. Client Key Exchange
   - Verify certificate
   - Generate pre-master secret
   - Send encrypted with server's public key

4. Change Cipher Spec
   - Both sides derive session keys
   - Switch to encrypted communication

5. Encrypted Application Data
```

### Certificate Chain

```
Root CA (pre-installed in browsers)
    │
    ▼
Intermediate CA (signed by root)
    │
    ▼
Server Certificate (signed by intermediate)
    │
    ▼
Your website: example.com
```

### TLS Configuration Best Practices

```nginx
# Nginx SSL configuration
server {
    listen 443 ssl http2;

    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000" always;
}
```

---

## Network Latency

### Latency Components

```
DNS Lookup:           ~20-120ms (cacheable)
TCP Handshake:        1 RTT (~50-150ms)
TLS Handshake:        2-3 RTTs (~100-450ms)
Request/Response:     1+ RTT (~50-150ms)

Total first request: 200-800ms
Subsequent requests: 50-150ms (keep-alive)
```

### Reducing Latency

```
1. CDN - Content closer to users
2. HTTP/2 - Multiplexing, reduced connections
3. Connection pooling - Reuse TCP connections
4. TLS session resumption - Skip full handshake
5. DNS prefetching - Resolve domains early
6. Preconnect - Establish connections early

<link rel="dns-prefetch" href="//api.example.com">
<link rel="preconnect" href="https://api.example.com">
```

---

## gRPC

### gRPC vs REST

```
REST:
- HTTP/1.1 or HTTP/2
- JSON (text, larger)
- Request/Response only
- Language agnostic (any HTTP client)

gRPC:
- HTTP/2 only
- Protocol Buffers (binary, smaller)
- Streaming (unary, server, client, bidirectional)
- Generated clients (type-safe)
```

### Protocol Buffers

```protobuf
// user.proto
syntax = "proto3";

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
  rpc CreateUser(User) returns (User);
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  repeated string roles = 4;
}

message GetUserRequest {
  int32 id = 1;
}
```

### gRPC Streaming

```javascript
// Server streaming
rpc ListUsers(Request) returns (stream User);
// Client calls once, server sends multiple responses

// Client streaming
rpc UploadData(stream DataChunk) returns (UploadResult);
// Client sends multiple messages, server responds once

// Bidirectional streaming
rpc Chat(stream Message) returns (stream Message);
// Both sides send messages independently
```

---

## Proxy Types

### Forward Proxy

```
Client → Forward Proxy → Internet
         (on client's behalf)

Uses:
- Privacy/anonymity
- Access control
- Caching
- Content filtering
```

### Reverse Proxy

```
Internet → Reverse Proxy → Backend Servers
           (on server's behalf)

Uses:
- Load balancing
- SSL termination
- Caching
- Security/WAF
- Compression
```

### API Gateway

```
                    ┌──────────────┐
                    │ API Gateway  │
Client ────────────▶│ - Auth       │
                    │ - Rate limit │
                    │ - Routing    │
                    └──────┬───────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
    ┌─────────┐       ┌─────────┐       ┌─────────┐
    │ User    │       │ Order   │       │ Product │
    │ Service │       │ Service │       │ Service │
    └─────────┘       └─────────┘       └─────────┘
```

---

## Interview Questions

### Q: How does HTTPS work?

1. **DNS resolution** - Get server IP
2. **TCP connection** - 3-way handshake
3. **TLS handshake** - Exchange keys, verify certificate
4. **Encrypted communication** - HTTP over encrypted channel
5. **Connection reuse** - Keep-alive for subsequent requests

### Q: What happens when you type a URL in browser?

1. Parse URL
2. Check browser cache (for DNS, resources)
3. DNS lookup
4. TCP connection
5. TLS handshake (if HTTPS)
6. Send HTTP request
7. Server processes request
8. Server sends response
9. Browser parses HTML
10. Browser requests subresources (CSS, JS, images)
11. Browser renders page

### Q: When would you use WebSockets vs HTTP?

**WebSockets:**
- Real-time bidirectional communication
- Chat applications
- Live updates (stock prices, sports scores)
- Gaming
- Collaborative editing

**HTTP (polling/SSE):**
- Simple request/response
- Infrequent updates
- SSE for server-only updates
- Compatibility requirements
- Simpler infrastructure
