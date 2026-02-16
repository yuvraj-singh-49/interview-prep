# VPC (Virtual Private Cloud)

## Overview

```
VPC = Isolated virtual network in AWS

Key Components:
┌─────────────────────────────────────────────────────────────────────────┐
│                              VPC (10.0.0.0/16)                          │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Availability Zone A          │  Availability Zone B             │   │
│   │                               │                                  │   │
│   │  ┌────────────────────────┐  │  ┌────────────────────────┐      │   │
│   │  │ Public Subnet          │  │  │ Public Subnet          │      │   │
│   │  │ 10.0.1.0/24           │  │  │ 10.0.2.0/24           │      │   │
│   │  │ ┌──────┐ ┌──────────┐ │  │  │ ┌──────┐ ┌──────────┐ │      │   │
│   │  │ │ NAT  │ │ Bastion  │ │  │  │ │ ALB  │ │ Public   │ │      │   │
│   │  │ │ GW   │ │ Host     │ │  │  │ │      │ │ Instance │ │      │   │
│   │  │ └──────┘ └──────────┘ │  │  │ └──────┘ └──────────┘ │      │   │
│   │  └────────────────────────┘  │  └────────────────────────┘      │   │
│   │                               │                                  │   │
│   │  ┌────────────────────────┐  │  ┌────────────────────────┐      │   │
│   │  │ Private Subnet         │  │  │ Private Subnet         │      │   │
│   │  │ 10.0.3.0/24           │  │  │ 10.0.4.0/24           │      │   │
│   │  │ ┌──────┐ ┌──────────┐ │  │  │ ┌──────┐ ┌──────────┐ │      │   │
│   │  │ │ App  │ │ App      │ │  │  │ │ App  │ │ App      │ │      │   │
│   │  │ │ Srvr │ │ Server   │ │  │  │ │ Srvr │ │ Server   │ │      │   │
│   │  │ └──────┘ └──────────┘ │  │  │ └──────┘ └──────────┘ │      │   │
│   │  └────────────────────────┘  │  └────────────────────────┘      │   │
│   │                               │                                  │   │
│   │  ┌────────────────────────┐  │  ┌────────────────────────┐      │   │
│   │  │ Data Subnet           │  │  │ Data Subnet           │      │   │
│   │  │ 10.0.5.0/24           │  │  │ 10.0.6.0/24           │      │   │
│   │  │ ┌──────┐ ┌──────────┐ │  │  │ ┌──────┐              │      │   │
│   │  │ │ RDS  │ │ ElastiC- │ │  │  │ │ RDS  │              │      │   │
│   │  │ │      │ │ ache     │ │  │  │ │Standby│              │      │   │
│   │  │ └──────┘ └──────────┘ │  │  │ └──────┘              │      │   │
│   │  └────────────────────────┘  │  └────────────────────────┘      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌──────────────┐                              ┌──────────────┐        │
│   │ Internet GW  │                              │ VPC Endpoint │        │
│   │ (Ingress/    │                              │ (S3, DynamoDB)│        │
│   │  Egress)     │                              └──────────────┘        │
│   └──────────────┘                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## CIDR & IP Addressing

### CIDR Basics

```
CIDR = Classless Inter-Domain Routing

Format: IP_address/prefix_length
Example: 10.0.0.0/16

┌─────────────────────────────────────────────────────────────────────────┐
│ CIDR          │ Netmask         │ Available IPs │ Use Case             │
├───────────────┼─────────────────┼───────────────┼──────────────────────┤
│ /16           │ 255.255.0.0     │ 65,536        │ Large VPC            │
│ /20           │ 255.255.240.0   │ 4,096         │ Medium VPC           │
│ /24           │ 255.255.255.0   │ 256           │ Typical subnet       │
│ /27           │ 255.255.255.224 │ 32            │ Small subnet         │
│ /28           │ 255.255.255.240 │ 16            │ Minimum subnet       │
└───────────────┴─────────────────┴───────────────┴──────────────────────┘

VPC Limits:
- Min: /28 (16 IPs)
- Max: /16 (65,536 IPs)

AWS Reserved IPs (per subnet):
- .0: Network address
- .1: VPC router
- .2: DNS server
- .3: Reserved for future use
- .255: Broadcast (not used but reserved)

Example: 10.0.1.0/24 has 256 - 5 = 251 usable IPs
```

### IP Address Types

```
Public IP:
- Assigned from AWS pool
- Released when instance stops (unless Elastic IP)
- Required for direct internet access

Elastic IP:
- Static public IP
- Persists across stop/start
- Associated with AWS account
- Charged when not associated

Private IP:
- From VPC CIDR range
- Persists for instance lifetime
- Used for internal communication

Secondary Private IPs:
- Multiple IPs per ENI
- Use for hosting multiple services
- Floating IPs for failover
```

---

## Subnets

### Public vs Private Subnets

```
Public Subnet:
- Route table has route to Internet Gateway
- Instances can have public IPs
- Direct internet access

Private Subnet:
- No route to Internet Gateway
- Uses NAT for outbound internet
- More secure for backend services

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Public Subnet Route Table:                                             │
│   ┌──────────────────────┬──────────────────────────────────────────┐   │
│   │ Destination          │ Target                                    │   │
│   ├──────────────────────┼──────────────────────────────────────────┤   │
│   │ 10.0.0.0/16         │ local                                    │   │
│   │ 0.0.0.0/0           │ igw-xxxx (Internet Gateway)              │   │
│   └──────────────────────┴──────────────────────────────────────────┘   │
│                                                                          │
│   Private Subnet Route Table:                                            │
│   ┌──────────────────────┬──────────────────────────────────────────┐   │
│   │ Destination          │ Target                                    │   │
│   ├──────────────────────┼──────────────────────────────────────────┤   │
│   │ 10.0.0.0/16         │ local                                    │   │
│   │ 0.0.0.0/0           │ nat-xxxx (NAT Gateway)                   │   │
│   └──────────────────────┴──────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Subnet Design Best Practices

```
Multi-AZ Design:
- Spread subnets across 2-3 AZs
- Same tier in different AZs
- Balance subnet sizes

Typical 3-Tier Architecture:
┌───────────────────────────────────────────────────────────────────┐
│ Tier      │ AZ-a        │ AZ-b        │ AZ-c        │ CIDR Size  │
├───────────┼─────────────┼─────────────┼─────────────┼────────────┤
│ Public    │ 10.0.0.0/24 │ 10.0.1.0/24 │ 10.0.2.0/24 │ /24 each   │
│ Private   │ 10.0.10.0/24│ 10.0.11.0/24│ 10.0.12.0/24│ /24 each   │
│ Data      │ 10.0.20.0/24│ 10.0.21.0/24│ 10.0.22.0/24│ /24 each   │
│ Reserved  │ 10.0.100.0  │ onwards...  │             │ Future use │
└───────────┴─────────────┴─────────────┴─────────────┴────────────┘

Key Points:
- Leave room for growth
- Consistent naming convention
- Document IP allocations
```

---

## Internet Connectivity

### Internet Gateway

```
Internet Gateway (IGW):
- Horizontally scaled, redundant
- No bandwidth constraints
- Free (pay for data transfer)

Attach to VPC:
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-xxxx \
  --vpc-id vpc-xxxx

For internet access:
1. Attach IGW to VPC
2. Route table: 0.0.0.0/0 → IGW
3. Public IP on instance
4. Security group allows traffic
```

### NAT Gateway

```
NAT Gateway:
- Managed NAT service
- AZ-specific (deploy per AZ)
- Scales automatically (up to 45 Gbps)

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Private Instance ──▶ NAT Gateway ──▶ Internet Gateway ──▶ Internet   │
│   (10.0.3.50)         (Public subnet)  (VPC level)                     │
│                       (Elastic IP)                                       │
│                                                                          │
│   Outbound: ✓ (via NAT)                                                 │
│   Inbound:  ✗ (not initiated from internet)                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

High Availability:
- Deploy NAT Gateway per AZ
- Route table per AZ points to local NAT GW
- Avoids cross-AZ traffic

Pricing:
- $0.045/hour per NAT Gateway
- $0.045/GB processed
- Consider NAT instances for cost (but manage yourself)
```

### Egress-Only Internet Gateway

```
For IPv6 outbound only:
- IPv6 doesn't have NAT (all IPs are public)
- Egress-only allows outbound, blocks inbound
- Similar to NAT Gateway for IPv4

Use case:
- IPv6 instances need outbound internet
- Don't want inbound connections from internet
```

---

## Security

### Security Groups

```
Security Groups = Stateful firewall at instance level

Characteristics:
- Attached to ENI (not subnet)
- Stateful (return traffic auto-allowed)
- Allow rules only (no deny)
- Default: Deny all inbound, allow all outbound

┌─────────────────────────────────────────────────────────────────────────┐
│ Security Group: web-sg                                                   │
│                                                                          │
│ Inbound Rules:                                                           │
│ ┌──────────┬──────────┬──────────────────────┬──────────────────────┐   │
│ │ Type     │ Port     │ Source               │ Description          │   │
│ ├──────────┼──────────┼──────────────────────┼──────────────────────┤   │
│ │ HTTP     │ 80       │ 0.0.0.0/0           │ Public web access    │   │
│ │ HTTPS    │ 443      │ 0.0.0.0/0           │ Public web access    │   │
│ │ SSH      │ 22       │ 10.0.0.0/8          │ Internal SSH         │   │
│ └──────────┴──────────┴──────────────────────┴──────────────────────┘   │
│                                                                          │
│ Outbound Rules:                                                          │
│ ┌──────────┬──────────┬──────────────────────┬──────────────────────┐   │
│ │ Type     │ Port     │ Destination          │ Description          │   │
│ ├──────────┼──────────┼──────────────────────┼──────────────────────┤   │
│ │ All      │ All      │ 0.0.0.0/0           │ Allow all outbound   │   │
│ └──────────┴──────────┴──────────────────────┴──────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘

Best Practice - Reference other Security Groups:
- App SG allows traffic from Web SG (not IP ranges)
- Changes propagate automatically
- More maintainable
```

### Network ACLs

```
Network ACLs (NACLs) = Stateless firewall at subnet level

Characteristics:
- Applied to entire subnet
- Stateless (must allow both directions)
- Allow AND deny rules
- Processed in order (rule number)
- Default: Allow all

┌─────────────────────────────────────────────────────────────────────────┐
│ NACL: public-nacl                                                        │
│                                                                          │
│ Inbound Rules:                                                           │
│ ┌──────┬────────┬──────────┬──────────────────┬─────────┬────────────┐  │
│ │ Rule │ Type   │ Protocol │ Port Range       │ Source  │ Action     │  │
│ ├──────┼────────┼──────────┼──────────────────┼─────────┼────────────┤  │
│ │ 100  │ HTTP   │ TCP      │ 80               │ 0.0.0.0/0│ ALLOW     │  │
│ │ 110  │ HTTPS  │ TCP      │ 443              │ 0.0.0.0/0│ ALLOW     │  │
│ │ 120  │ Custom │ TCP      │ 1024-65535       │ 0.0.0.0/0│ ALLOW     │  │
│ │ *    │ All    │ All      │ All              │ 0.0.0.0/0│ DENY      │  │
│ └──────┴────────┴──────────┴──────────────────┴─────────┴────────────┘  │
│                                                                          │
│ Outbound Rules:                                                          │
│ ┌──────┬────────┬──────────┬──────────────────┬─────────┬────────────┐  │
│ │ Rule │ Type   │ Protocol │ Port Range       │ Dest    │ Action     │  │
│ ├──────┼────────┼──────────┼──────────────────┼─────────┼────────────┤  │
│ │ 100  │ HTTP   │ TCP      │ 80               │ 0.0.0.0/0│ ALLOW     │  │
│ │ 110  │ HTTPS  │ TCP      │ 443              │ 0.0.0.0/0│ ALLOW     │  │
│ │ 120  │ Custom │ TCP      │ 1024-65535       │ 0.0.0.0/0│ ALLOW     │  │
│ │ *    │ All    │ All      │ All              │ 0.0.0.0/0│ DENY      │  │
│ └──────┴────────┴──────────┴──────────────────┴─────────┴────────────┘  │
│                                                                          │
│ Note: Ephemeral ports (1024-65535) needed for response traffic          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Security Group vs NACL

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Aspect            │ Security Group       │ NACL                        │
├───────────────────┼──────────────────────┼─────────────────────────────┤
│ Level             │ Instance (ENI)       │ Subnet                      │
│ State             │ Stateful             │ Stateless                   │
│ Rules             │ Allow only           │ Allow and Deny              │
│ Processing        │ All rules evaluated  │ Rules in order              │
│ Default           │ Deny all in          │ Allow all                   │
│ Changes           │ Immediate            │ Immediate                   │
│ Association       │ Multiple per ENI     │ One per subnet              │
└───────────────────┴──────────────────────┴─────────────────────────────┘

Use Together:
- Security Groups: Primary defense
- NACLs: Subnet-level controls, block known bad IPs
```

---

## VPC Connectivity

### VPC Peering

```
VPC Peering = Direct connection between two VPCs

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   VPC-A (10.0.0.0/16)              VPC-B (192.168.0.0/16)              │
│   ┌───────────────────┐            ┌───────────────────┐               │
│   │                   │            │                   │               │
│   │   ┌─────────┐     │            │     ┌─────────┐   │               │
│   │   │ EC2     │     │◄──Peering──▶│    │ EC2     │   │               │
│   │   └─────────┘     │  Connection│     └─────────┘   │               │
│   │                   │            │                   │               │
│   └───────────────────┘            └───────────────────┘               │
│                                                                          │
│   Route Table VPC-A:                                                     │
│   192.168.0.0/16 → pcx-xxxx (peering connection)                        │
│                                                                          │
│   Route Table VPC-B:                                                     │
│   10.0.0.0/16 → pcx-xxxx (peering connection)                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Characteristics:
- Non-transitive (A-B, B-C doesn't mean A-C)
- Cross-region supported
- Cross-account supported
- No overlapping CIDR
- No single point of failure
```

### Transit Gateway

```
Transit Gateway = Hub for multiple VPCs and on-premises

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│                        ┌───────────────────┐                            │
│                        │  Transit Gateway  │                            │
│                        │  (Hub)            │                            │
│                        └─────────┬─────────┘                            │
│                                  │                                       │
│         ┌────────────────────────┼────────────────────────┐             │
│         │            │           │           │            │             │
│         ▼            ▼           ▼           ▼            ▼             │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│   │  VPC-A   │ │  VPC-B   │ │  VPC-C   │ │  VPC-D   │ │On-Premises│    │
│   └──────────┘ └──────────┘ └──────────┘ └──────────┘ │ (VPN/DX)  │    │
│                                                       └──────────┘    │
│                                                                          │
│   Benefits:                                                              │
│   - Centralized connectivity                                            │
│   - Transitive routing                                                  │
│   - Route tables for segmentation                                       │
│   - Cross-region peering                                                │
│   - Scales to thousands of connections                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Use Cases:
- Multi-VPC architectures
- Shared services VPC
- Hybrid cloud connectivity
- Network segmentation
```

### VPC Endpoints

```
VPC Endpoints = Private access to AWS services

Gateway Endpoints (Free):
- S3 and DynamoDB only
- Route table entry
- No ENI needed

Interface Endpoints (Powered by PrivateLink):
- Most other AWS services
- Creates ENI in your subnet
- Private DNS for service
- Charged per hour + data

┌─────────────────────────────────────────────────────────────────────────┐
│                              VPC                                         │
│                                                                          │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │  Private Subnet                                                 │    │
│   │                                                                 │    │
│   │  ┌─────────┐                        ┌──────────────────┐       │    │
│   │  │ Lambda  │──────────────────────▶│ Interface        │       │    │
│   │  │         │  Secrets Manager      │ Endpoint         │       │    │
│   │  └─────────┘                        │ (ENI in subnet)  │       │    │
│   │                                      │ secretsmanager. │       │    │
│   │                                      │ region.amazonaws│       │    │
│   │                                      └────────┬─────────┘       │    │
│   │  ┌─────────┐                                 │                 │    │
│   │  │ EC2     │───────────────────┐             │                 │    │
│   │  │         │  S3 Access        │             │                 │    │
│   │  └─────────┘                    │             │                 │    │
│   │                                 │             │                 │    │
│   └─────────────────────────────────┼─────────────┼─────────────────┘    │
│                                     │             │                      │
│                                     ▼             ▼                      │
│                            ┌──────────────┐   (to AWS                   │
│                            │ Gateway      │   backbone)                 │
│                            │ Endpoint     │                             │
│                            │ (S3)         │                             │
│                            └──────────────┘                             │
│                                                                          │
│   Benefits:                                                              │
│   - No internet traversal                                               │
│   - Reduced data transfer costs                                         │
│   - Better security (private)                                           │
│   - Lower latency                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Hybrid Connectivity

### Site-to-Site VPN

```
VPN = Encrypted tunnel over internet

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   AWS                                        On-Premises                │
│   ┌───────────────────────┐                 ┌───────────────────────┐   │
│   │         VPC           │                 │   Corporate Data      │   │
│   │                       │                 │   Center              │   │
│   │  ┌─────────────────┐  │     Internet    │  ┌─────────────────┐  │   │
│   │  │ Virtual Private │  │◄───Encrypted───▶│  │ Customer        │  │   │
│   │  │ Gateway (VGW)   │  │     Tunnels     │  │ Gateway         │  │   │
│   │  └─────────────────┘  │    (2 tunnels)  │  └─────────────────┘  │   │
│   │                       │                 │                       │   │
│   └───────────────────────┘                 └───────────────────────┘   │
│                                                                          │
│   Bandwidth: Up to 1.25 Gbps per tunnel                                 │
│   Redundancy: 2 tunnels to different endpoints                          │
│   Encryption: IPSec                                                     │
│   Setup time: Minutes                                                   │
│   Cost: ~$0.05/hour + data transfer                                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Direct Connect

```
Direct Connect = Dedicated private connection

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   AWS                Direct Connect           On-Premises               │
│   ┌────────────┐     Location                ┌────────────────┐         │
│   │            │     ┌──────────┐            │                │         │
│   │ AWS Region │◄───▶│ DX       │◄──────────▶│ Corporate DC  │         │
│   │            │     │ Router   │  Private   │                │         │
│   │            │     │          │  Line      │                │         │
│   └────────────┘     └──────────┘            └────────────────┘         │
│                                                                          │
│   Connection Types:                                                      │
│   - Dedicated: 1, 10, 100 Gbps (physical port)                         │
│   - Hosted: 50 Mbps - 10 Gbps (via partner)                            │
│                                                                          │
│   Virtual Interfaces:                                                    │
│   - Private VIF: Access VPC                                             │
│   - Public VIF: Access AWS public services                              │
│   - Transit VIF: Access Transit Gateway                                 │
│                                                                          │
│   Benefits vs VPN:                                                       │
│   - Consistent latency                                                  │
│   - Higher bandwidth                                                    │
│   - Predictable performance                                             │
│   - No internet dependency                                              │
│                                                                          │
│   Setup time: Weeks to months                                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## VPC Flow Logs

```
Flow Logs = Network traffic logging

Log Levels:
- VPC level: All traffic in VPC
- Subnet level: Traffic in specific subnet
- ENI level: Traffic for specific interface

Destinations:
- CloudWatch Logs
- S3
- Kinesis Data Firehose

Log Format (default):
┌─────────────────────────────────────────────────────────────────────────┐
│ version account-id interface-id srcaddr dstaddr srcport dstport        │
│ protocol packets bytes start end action log-status                      │
│                                                                          │
│ Example:                                                                 │
│ 2 123456789012 eni-abc123 10.0.1.5 52.26.198.183 49152 443 6 25         │
│ 20000 1418530010 1418530070 ACCEPT OK                                   │
│                                                                          │
│ Analysis:                                                                │
│ - Source: 10.0.1.5:49152 (internal)                                    │
│ - Dest: 52.26.198.183:443 (external HTTPS)                             │
│ - Protocol: 6 (TCP)                                                     │
│ - Action: ACCEPT                                                        │
└─────────────────────────────────────────────────────────────────────────┘

Use Cases:
- Security analysis
- Troubleshooting connectivity
- Compliance auditing
- Cost optimization (identify traffic patterns)
```

---

## Interview Discussion Points

### How do you design a VPC for a 3-tier application?

```
Design Principles:

1. Network Segmentation
   - Public subnets: Load balancers only
   - Private subnets: Application tier
   - Data subnets: Databases, caches

2. Multi-AZ
   - Each tier across 2-3 AZs
   - NAT Gateway per AZ
   - Avoid cross-AZ dependencies

3. Security Layers
   - NACLs: Subnet-level rules
   - Security Groups: Instance-level
   - Reference SGs (not IPs)

4. Connectivity
   - VPC Endpoints for AWS services
   - NAT Gateway for outbound only
   - No direct internet to app/data tiers

Example CIDR Plan (/16 VPC):
- 10.0.0.0/24 - 10.0.2.0/24: Public (3 AZs)
- 10.0.10.0/24 - 10.0.12.0/24: Private App (3 AZs)
- 10.0.20.0/24 - 10.0.22.0/24: Private Data (3 AZs)
- 10.0.100.0/20: Reserved for growth
```

### How do you troubleshoot connectivity issues?

```
Systematic Approach:

1. Verify Security Groups
   - Source SG allows outbound
   - Destination SG allows inbound
   - Check port and protocol

2. Verify NACLs
   - Both inbound and outbound rules
   - Check ephemeral ports
   - Rules processed in order

3. Verify Route Tables
   - Route exists for destination
   - Route points to correct target
   - Subnet associated with route table

4. Check Internet Path
   - IGW attached to VPC?
   - Public IP on instance?
   - NAT Gateway for private subnets?

5. Check VPC Endpoints
   - Endpoint policy allows access?
   - DNS resolves to endpoint?

6. Use Flow Logs
   - Check ACCEPT vs REJECT
   - Identify which layer blocked

7. Reachability Analyzer
   - AWS tool for path analysis
   - Shows where path fails
```

### When to use Transit Gateway vs VPC Peering?

```
Use VPC Peering When:
- Small number of VPCs (<10)
- Simple hub-spoke not needed
- Lower cost requirement
- Direct connections only

Use Transit Gateway When:
- Many VPCs (10+)
- Need transitive routing
- Centralized network management
- Hybrid cloud (VPN/DX)
- Network segmentation via route tables
- Multi-region connectivity

Cost Consideration:
- Peering: Free (only data transfer)
- Transit Gateway: $0.05/GB + attachment fee

Scale Consideration:
- Peering: N*(N-1)/2 connections for full mesh
- Transit Gateway: N connections (star topology)
```
