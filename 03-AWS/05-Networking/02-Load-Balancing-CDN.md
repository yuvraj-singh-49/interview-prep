# Load Balancing & CDN

## Elastic Load Balancing Overview

```
ELB Types:
┌─────────────────────────────────────────────────────────────────────────┐
│ Type                │ Layer │ Use Case                                  │
├─────────────────────┼───────┼───────────────────────────────────────────┤
│ Application (ALB)   │ 7     │ HTTP/HTTPS, path routing, microservices  │
│ Network (NLB)       │ 4     │ TCP/UDP, high performance, static IP     │
│ Gateway (GWLB)      │ 3     │ Virtual appliances, firewalls            │
│ Classic (CLB)       │ 4/7   │ Legacy (avoid for new deployments)       │
└─────────────────────┴───────┴───────────────────────────────────────────┘
```

---

## Application Load Balancer (ALB)

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Internet                                                               │
│       │                                                                  │
│       ▼                                                                  │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │                Application Load Balancer                       │     │
│   │                                                                │     │
│   │   ┌─────────────────────────────────────────────────────────┐ │     │
│   │   │                    Listeners                             │ │     │
│   │   │   - HTTP:80    - HTTPS:443                              │ │     │
│   │   └────────────────────────┬────────────────────────────────┘ │     │
│   │                            │                                   │     │
│   │   ┌────────────────────────┴────────────────────────────────┐ │     │
│   │   │                    Rules (priority order)                │ │     │
│   │   │   1. Path: /api/*     → API Target Group                │ │     │
│   │   │   2. Host: m.app.com  → Mobile Target Group             │ │     │
│   │   │   3. Header: X-Ver=2  → V2 Target Group                 │ │     │
│   │   │   4. Default          → Web Target Group                │ │     │
│   │   └─────────────────────────────────────────────────────────┘ │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                            │                                             │
│         ┌──────────────────┼──────────────────┐                         │
│         ▼                  ▼                  ▼                          │
│   ┌───────────┐      ┌───────────┐      ┌───────────┐                  │
│   │ API TG    │      │ Mobile TG │      │ Web TG    │                  │
│   │ EC2/Cont. │      │ Lambda    │      │ EC2/Cont. │                  │
│   └───────────┘      └───────────┘      └───────────┘                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Routing Rules

```
Rule Conditions (can combine):
┌─────────────────────────────────────────────────────────────────────────┐
│ Condition Type    │ Example                                             │
├───────────────────┼─────────────────────────────────────────────────────┤
│ Path              │ /api/*, /images/*.jpg                               │
│ Host              │ api.example.com, *.example.com                      │
│ HTTP Header       │ X-Custom-Header: value                              │
│ HTTP Method       │ GET, POST, PUT                                      │
│ Query String      │ ?version=v2                                         │
│ Source IP         │ 192.168.1.0/24                                      │
└───────────────────┴─────────────────────────────────────────────────────┘

Rule Actions:
- forward: Send to target group
- redirect: HTTP redirect (301/302)
- fixed-response: Return static response
- authenticate-cognito: Cognito auth
- authenticate-oidc: OIDC auth

// Example: A/B Testing with weighted routing
{
  "Type": "forward",
  "ForwardConfig": {
    "TargetGroups": [
      {"TargetGroupArn": "arn:...:tg-v1", "Weight": 90},
      {"TargetGroupArn": "arn:...:tg-v2", "Weight": 10}
    ]
  }
}
```

### Target Groups

```
Target Types:
┌─────────────────────────────────────────────────────────────────────────┐
│ Type       │ Description                    │ Use Case                  │
├────────────┼────────────────────────────────┼───────────────────────────┤
│ instance   │ EC2 instances by ID           │ Traditional apps          │
│ ip         │ IP addresses                   │ Containers, on-prem      │
│ lambda     │ Lambda functions               │ Serverless               │
└────────────┴────────────────────────────────┴───────────────────────────┘

Health Checks:
{
  "Protocol": "HTTP",
  "Port": "traffic-port",
  "Path": "/health",
  "Interval": 30,
  "Timeout": 5,
  "HealthyThreshold": 2,
  "UnhealthyThreshold": 3,
  "Matcher": {"HttpCode": "200-299"}
}

Stickiness (Session Affinity):
- Duration-based: Cookie with TTL
- Application-based: Custom cookie
- Use for: Stateful apps (prefer stateless design)
```

### ALB Features

```
SSL/TLS:
- Terminate SSL at ALB
- Multiple certificates (SNI)
- ACM integration (free certs)
- Security policies (cipher suites)

Connection Features:
- HTTP/2 support
- WebSocket support
- Connection draining (deregistration delay)
- Idle timeout (default 60s)

Security:
- Security groups
- WAF integration
- AWS Shield protection
- Access logs to S3

Cross-Zone Load Balancing:
- Enabled by default (ALB)
- Distributes evenly across all AZ targets
- No extra charge
```

---

## Network Load Balancer (NLB)

### Characteristics

```
NLB Features:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Extreme Performance:                                                   │
│   - Millions of requests/second                                         │
│   - Ultra-low latency (~100 microseconds)                               │
│   - Handles volatile workloads                                          │
│                                                                          │
│   Static IP:                                                             │
│   - One static IP per AZ                                                │
│   - Or Elastic IP per AZ                                                │
│   - Whitelist-friendly                                                  │
│                                                                          │
│   Protocol Support:                                                      │
│   - TCP, UDP, TLS                                                       │
│   - Preserves source IP (default)                                       │
│                                                                          │
│   Use Cases:                                                             │
│   - Gaming                                                              │
│   - IoT                                                                 │
│   - Financial services                                                  │
│   - Anything needing static IP                                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### NLB vs ALB

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Feature                 │ ALB              │ NLB                        │
├─────────────────────────┼──────────────────┼────────────────────────────┤
│ Layer                   │ 7 (HTTP)         │ 4 (TCP/UDP)               │
│ Latency                 │ ~400ms           │ ~100μs                    │
│ Static IP               │ No (DNS only)    │ Yes                       │
│ Source IP preservation  │ X-Forwarded-For  │ Native                    │
│ Path-based routing      │ Yes              │ No                        │
│ SSL termination         │ Yes              │ Yes (TLS listener)        │
│ WebSocket               │ Yes              │ Yes (TCP)                 │
│ Health checks           │ HTTP/HTTPS       │ TCP/HTTP/HTTPS            │
│ Cross-zone (default)    │ Enabled          │ Disabled                  │
│ Pricing                 │ Higher           │ Lower                     │
└─────────────────────────┴──────────────────┴────────────────────────────┘

Choose ALB: HTTP/HTTPS apps, content routing, authentication
Choose NLB: Non-HTTP, static IP needed, extreme performance
```

---

## Route 53

### Overview

```
Route 53 = Managed DNS + Health Checking + Domain Registration

Components:
1. Hosted Zone: Container for DNS records
2. Records: DNS entries (A, AAAA, CNAME, etc.)
3. Health Checks: Monitor endpoint health
4. Routing Policies: Control traffic distribution
```

### Record Types

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Record Type │ Purpose                        │ Example                  │
├─────────────┼────────────────────────────────┼──────────────────────────┤
│ A           │ IPv4 address                   │ 192.0.2.1               │
│ AAAA        │ IPv6 address                   │ 2001:db8::1             │
│ CNAME       │ Alias to another domain        │ www → app.example.com   │
│ Alias       │ AWS resource (free, zone apex) │ example.com → ALB       │
│ MX          │ Mail server                    │ mail.example.com        │
│ TXT         │ Text (verification, SPF)       │ "v=spf1 ..."            │
│ NS          │ Name servers                   │ ns-xxx.awsdns-xxx.com   │
│ SOA         │ Start of authority             │ (automatic)              │
└─────────────┴────────────────────────────────┴──────────────────────────┘

Alias vs CNAME:
- Alias: Works at zone apex (example.com), free, AWS resources only
- CNAME: Cannot be zone apex, standard DNS, any target
```

### Routing Policies

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Policy        │ How it Works                  │ Use Case                │
├───────────────┼───────────────────────────────┼─────────────────────────┤
│ Simple        │ Single resource               │ Single server           │
│ Weighted      │ % traffic to each resource    │ A/B testing, migration │
│ Latency       │ Lowest latency region         │ Global apps            │
│ Failover      │ Primary/secondary             │ DR setup               │
│ Geolocation   │ User's location               │ Content localization   │
│ Geoproximity  │ Resource proximity + bias     │ Traffic shifting       │
│ Multivalue    │ Multiple healthy IPs          │ Simple load balancing  │
│ IP-based      │ Client IP ranges              │ ISP-specific routing   │
└───────────────┴───────────────────────────────┴─────────────────────────┘
```

### Health Checks

```
Health Check Types:
1. Endpoint: Monitor IP/domain
2. Calculated: Combine multiple checks (AND/OR)
3. CloudWatch Alarm: Based on metric

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Route 53 Health Check                                                  │
│   ┌───────────────────┐                                                 │
│   │   Checkers        │──────▶ Endpoint (HTTP/HTTPS/TCP)               │
│   │   (multiple       │                                                 │
│   │   locations)      │        Threshold: 3 failures = unhealthy       │
│   └───────────────────┘        Interval: 10s or 30s                    │
│           │                                                              │
│           ▼                                                              │
│   DNS resolves only to healthy endpoints                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Failover Configuration:
1. Create health check for primary
2. Create Failover records (primary + secondary)
3. Associate health check with primary
4. When primary fails → traffic to secondary
```

### DNS Failover Patterns

```
Active-Passive:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Route 53                                                               │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  Failover Record Set                                          │     │
│   │                                                                │     │
│   │  Primary (Health Check) ────▶ US-EAST-1 (Active)             │     │
│   │         │                                                      │     │
│   │         │ (unhealthy)                                         │     │
│   │         ▼                                                      │     │
│   │  Secondary ─────────────────▶ EU-WEST-1 (Passive)            │     │
│   │                                                                │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Active-Active with Latency:
- Latency-based routing
- Health checks on all endpoints
- Automatic failover to next-best latency
```

---

## CloudFront

### Overview

```
CloudFront = Content Delivery Network (CDN)

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Users Worldwide                                                        │
│       │                                                                  │
│       ▼                                                                  │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │                    Edge Locations (400+)                       │     │
│   │                                                                │     │
│   │   Cache Hit → Return cached content (fast)                    │     │
│   │   Cache Miss → Fetch from origin                              │     │
│   │                                                                │     │
│   └────────────────────────────┬──────────────────────────────────┘     │
│                                │                                         │
│                    ┌───────────┴───────────┐                            │
│                    ▼                       ▼                            │
│            ┌─────────────┐         ┌─────────────┐                     │
│            │ S3 Bucket   │         │ ALB/EC2     │                     │
│            │ (Static)    │         │ (Dynamic)   │                     │
│            └─────────────┘         └─────────────┘                     │
│                                                                          │
│   Benefits:                                                              │
│   - Lower latency (edge caching)                                        │
│   - Reduced origin load                                                 │
│   - DDoS protection (Shield)                                            │
│   - SSL/TLS termination                                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Distribution Configuration

```
Origins:
- S3 bucket (with OAC/OAI for private)
- Custom origin (ALB, EC2, any HTTP server)
- Origin groups (failover)

Behaviors:
┌─────────────────────────────────────────────────────────────────────────┐
│ Path Pattern │ Origin        │ Cache Policy  │ Settings                │
├──────────────┼───────────────┼───────────────┼─────────────────────────┤
│ /api/*       │ ALB           │ CachingDisabled│ Forward all headers    │
│ /images/*    │ S3-images     │ CachingOptimized│ Long TTL             │
│ Default (*)  │ S3-website    │ CachingOptimized│ Compress             │
└──────────────┴───────────────┴───────────────┴─────────────────────────┘

Cache Key:
- URL path
- Query strings (selected or all)
- Headers (selected)
- Cookies (selected)
```

### Caching

```
Cache Control:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Cache-Control Header (from origin):                                    │
│   - max-age=3600: Cache for 1 hour                                      │
│   - no-cache: Revalidate with origin                                    │
│   - no-store: Don't cache                                               │
│   - private: Browser only, not CDN                                      │
│                                                                          │
│   CloudFront Settings:                                                   │
│   - Minimum TTL: Floor for caching                                      │
│   - Maximum TTL: Ceiling for caching                                    │
│   - Default TTL: When origin doesn't specify                            │
│                                                                          │
│   Cache Invalidation:                                                    │
│   - Invalidate specific paths: /images/logo.png                         │
│   - Invalidate wildcards: /images/*                                     │
│   - Cost: 1000 free/month, then $0.005 each                            │
│   - Alternative: Version in filename (logo-v2.png)                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Origin Access Control (OAC)

```
Secure S3 Origin:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   User → CloudFront → S3 (private)                                      │
│                                                                          │
│   Without OAC:                                                           │
│   - S3 bucket must be public                                            │
│   - Security risk                                                       │
│                                                                          │
│   With OAC:                                                              │
│   - S3 bucket stays private                                             │
│   - CloudFront signs requests                                           │
│   - S3 bucket policy allows CloudFront only                             │
│                                                                          │
│   S3 Bucket Policy:                                                      │
│   {                                                                      │
│     "Statement": [{                                                      │
│       "Effect": "Allow",                                                │
│       "Principal": {                                                     │
│         "Service": "cloudfront.amazonaws.com"                           │
│       },                                                                 │
│       "Action": "s3:GetObject",                                         │
│       "Resource": "arn:aws:s3:::bucket/*",                              │
│       "Condition": {                                                     │
│         "StringEquals": {                                               │
│           "AWS:SourceArn": "arn:aws:cloudfront::123:distribution/XXX"  │
│         }                                                                │
│       }                                                                  │
│     }]                                                                   │
│   }                                                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Edge Functions

```
CloudFront Functions vs Lambda@Edge:
┌─────────────────────────────────────────────────────────────────────────┐
│ Feature              │ CloudFront Functions │ Lambda@Edge              │
├──────────────────────┼──────────────────────┼───────────────────────────┤
│ Runtime              │ JavaScript only      │ Node.js, Python          │
│ Execution time       │ <1ms                 │ Up to 30s (origin)       │
│ Memory               │ 2 MB                 │ Up to 10 GB              │
│ Package size         │ 10 KB                │ 50 MB                    │
│ Network access       │ No                   │ Yes                      │
│ Request body access  │ No                   │ Yes                      │
│ Triggers             │ Viewer only          │ Viewer + Origin          │
│ Price                │ $0.10/million        │ $0.60/million            │
└──────────────────────┴──────────────────────┴───────────────────────────┘

Use Cases:
CloudFront Functions:
- URL rewrites/redirects
- Header manipulation
- Cache key normalization
- Simple A/B testing

Lambda@Edge:
- Authentication/authorization
- Dynamic content generation
- Image optimization
- Bot detection
```

### Security Features

```
SSL/TLS:
- Free ACM certificates
- Custom SSL (SNI or dedicated IP)
- TLS 1.2/1.3 support
- Security policies (cipher suites)

DDoS Protection:
- AWS Shield Standard (free)
- Shield Advanced (paid)
- Automatic L3/L4 protection

WAF Integration:
- Web Application Firewall
- Custom rules
- Managed rule groups
- Rate limiting

Signed URLs/Cookies:
- Restrict access to authorized users
- Time-limited access
- IP restrictions
```

---

## Interview Discussion Points

### How do you design for global availability?

```
Multi-Region Architecture:

1. DNS Layer (Route 53)
   - Latency-based routing
   - Health checks + failover
   - Geographic routing for compliance

2. CDN Layer (CloudFront)
   - Edge caching globally
   - Origin failover groups
   - Edge functions for customization

3. Load Balancer Layer
   - ALB per region
   - Cross-zone load balancing
   - Health checks

4. Application Layer
   - Auto Scaling per region
   - Stateless design
   - Session storage in ElastiCache

5. Data Layer
   - Aurora Global Database
   - DynamoDB Global Tables
   - S3 Cross-Region Replication

Traffic Flow:
User → Route 53 → CloudFront → ALB → App → DB
        ↓
    Nearest region based on latency
```

### How do you secure a public-facing application?

```
Defense in Depth:

1. Edge Security
   - CloudFront + WAF
   - DDoS protection (Shield)
   - Bot management
   - Rate limiting

2. Network Security
   - ALB in public subnet
   - App in private subnet
   - Security groups (whitelist ALB only)
   - NACLs for additional control

3. Transport Security
   - HTTPS everywhere (ACM certs)
   - TLS 1.2+ only
   - HSTS headers
   - Certificate pinning (mobile)

4. Application Security
   - WAF rules (SQL injection, XSS)
   - Input validation
   - Authentication (Cognito/OIDC)
   - API throttling

5. Monitoring
   - VPC Flow Logs
   - ALB access logs
   - CloudFront logs
   - WAF logs
   - GuardDuty
```

### How do you handle a traffic spike?

```
Scaling Strategy:

1. Instant (already scaled)
   - CloudFront caching (absorb read traffic)
   - Over-provisioned minimum capacity
   - Provisioned concurrency (Lambda)

2. Fast (seconds to minutes)
   - ALB scales automatically
   - NLB scales automatically
   - Lambda scales automatically

3. Moderate (minutes)
   - EC2 Auto Scaling (target tracking)
   - ECS Service Auto Scaling
   - Predictive scaling

4. Slow (prepare in advance)
   - Warm EC2 capacity for known events
   - Increase RDS/ElastiCache size
   - Pre-warm CloudFront (request popular content)

Protect Backend:
- Caching at CloudFront
- Caching at application (ElastiCache)
- Queue overflow to SQS
- Rate limiting at WAF
- Circuit breakers in code
```
