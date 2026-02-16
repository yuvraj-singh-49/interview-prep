# AWS Core Concepts

## Global Infrastructure

```
AWS Global Infrastructure:
┌─────────────────────────────────────────────────────────────────────────┐
│                              Regions                                     │
│                    (32+ geographic locations)                            │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    Availability Zones                            │   │
│   │              (2-6 isolated data centers per region)              │   │
│   │                                                                  │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │   │
│   │   │    AZ-a     │  │    AZ-b     │  │    AZ-c     │            │   │
│   │   │ Data Center │  │ Data Center │  │ Data Center │            │   │
│   │   │   Cluster   │  │   Cluster   │  │   Cluster   │            │   │
│   │   └─────────────┘  └─────────────┘  └─────────────┘            │   │
│   │         ↕ Low latency links (< 2ms)  ↕                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      Edge Locations                              │   │
│   │              (400+ PoPs for CloudFront CDN)                      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Concepts

| Concept | Description | Use Case |
|---------|-------------|----------|
| **Region** | Geographic area with 2+ AZs | Data residency, latency reduction |
| **Availability Zone** | Isolated data center(s) | High availability, fault isolation |
| **Edge Location** | CDN PoP | Low latency content delivery |
| **Local Zone** | Extension of region closer to users | Ultra-low latency (<10ms) |
| **Wavelength Zone** | AWS in telecom 5G networks | Mobile edge computing |

### Region Selection Criteria

```
1. Compliance & Data Residency
   └── GDPR requires EU data stay in EU

2. Latency to Users
   └── Choose region closest to majority of users

3. Service Availability
   └── Not all services available in all regions

4. Cost
   └── Prices vary by region (US East typically cheapest)

5. Disaster Recovery
   └── Secondary region for DR (cross-region replication)
```

---

## AWS Well-Architected Framework

### Six Pillars

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Well-Architected Framework                            │
├─────────────────┬─────────────────┬─────────────────┬───────────────────┤
│  Operational    │   Security      │  Reliability    │   Performance     │
│  Excellence     │                 │                 │   Efficiency      │
├─────────────────┼─────────────────┼─────────────────┼───────────────────┤
│  Cost           │  Sustainability │                 │                   │
│  Optimization   │                 │                 │                   │
└─────────────────┴─────────────────┴─────────────────┴───────────────────┘
```

#### 1. Operational Excellence

```
Key Principles:
- Perform operations as code (IaC)
- Make frequent, small, reversible changes
- Refine operations procedures frequently
- Anticipate failure
- Learn from all operational failures

AWS Services:
- CloudFormation / CDK (Infrastructure as Code)
- Systems Manager (operations management)
- Config (compliance tracking)
- CloudTrail (API audit)
```

#### 2. Security

```
Key Principles:
- Implement strong identity foundation
- Enable traceability
- Apply security at all layers
- Automate security best practices
- Protect data in transit and at rest
- Keep people away from data
- Prepare for security events

AWS Services:
- IAM (identity)
- KMS (encryption)
- WAF (web firewall)
- Shield (DDoS protection)
- GuardDuty (threat detection)
- Security Hub (unified view)
```

#### 3. Reliability

```
Key Principles:
- Automatically recover from failure
- Test recovery procedures
- Scale horizontally
- Stop guessing capacity
- Manage change through automation

AWS Services:
- Auto Scaling
- Multi-AZ deployments
- Route 53 (DNS failover)
- S3 (11 9s durability)
- Backup (centralized backup)
```

#### 4. Performance Efficiency

```
Key Principles:
- Democratize advanced technologies
- Go global in minutes
- Use serverless architectures
- Experiment more often
- Consider mechanical sympathy

AWS Services:
- CloudFront (CDN)
- ElastiCache (caching)
- Auto Scaling
- Lambda (serverless)
- Global Accelerator
```

#### 5. Cost Optimization

```
Key Principles:
- Implement cloud financial management
- Adopt a consumption model
- Measure overall efficiency
- Stop spending on undifferentiated heavy lifting
- Analyze and attribute expenditure

AWS Services:
- Cost Explorer
- Budgets
- Reserved Instances / Savings Plans
- Spot Instances
- S3 Intelligent-Tiering
```

#### 6. Sustainability

```
Key Principles:
- Understand your impact
- Establish sustainability goals
- Maximize utilization
- Use managed services
- Reduce downstream impact

AWS Services:
- Graviton processors (energy efficient)
- Managed services (shared infrastructure)
- Auto Scaling (right-sizing)
```

---

## Shared Responsibility Model

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     CUSTOMER RESPONSIBILITY                              │
│                    "Security IN the Cloud"                               │
├─────────────────────────────────────────────────────────────────────────┤
│  Customer Data                                                           │
├─────────────────────────────────────────────────────────────────────────┤
│  Platform, Applications, Identity & Access Management                    │
├─────────────────────────────────────────────────────────────────────────┤
│  Operating System, Network & Firewall Configuration                      │
├─────────────────────────────────────────────────────────────────────────┤
│  Client-side Data         │  Server-side Encryption  │  Network Traffic │
│  Encryption               │  (File System/Data)      │  Protection      │
├───────────────────────────┴──────────────────────────┴──────────────────┤
│                       AWS RESPONSIBILITY                                 │
│                    "Security OF the Cloud"                               │
├─────────────────────────────────────────────────────────────────────────┤
│  Software: Compute, Storage, Database, Networking                        │
├─────────────────────────────────────────────────────────────────────────┤
│  Hardware / AWS Global Infrastructure                                    │
│  Regions, Availability Zones, Edge Locations                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### Responsibility by Service Type

| Service Type | Customer Manages | AWS Manages |
|--------------|------------------|-------------|
| **IaaS (EC2)** | OS, apps, data, firewall | Hardware, hypervisor |
| **PaaS (RDS)** | Data, access control | OS, patching, HA |
| **SaaS (S3)** | Data, permissions | Everything else |
| **Serverless (Lambda)** | Code, IAM | Runtime, scaling, infra |

---

## Service Categories Overview

### Compute

| Service | Use Case | Key Feature |
|---------|----------|-------------|
| **EC2** | Full control VMs | Instance types, AMIs |
| **Lambda** | Event-driven functions | Pay per invocation |
| **ECS** | Docker containers | Task definitions |
| **EKS** | Kubernetes | Managed control plane |
| **Fargate** | Serverless containers | No EC2 management |
| **Elastic Beanstalk** | Quick app deployment | PaaS-like experience |

### Storage

| Service | Type | Durability | Use Case |
|---------|------|------------|----------|
| **S3** | Object | 11 9s | Files, backups, static hosting |
| **EBS** | Block | 99.999% | EC2 volumes, databases |
| **EFS** | File (NFS) | 11 9s | Shared file system |
| **FSx** | Managed FS | High | Windows/Lustre workloads |
| **Glacier** | Archive | 11 9s | Long-term backup |

### Database

| Service | Type | Use Case |
|---------|------|----------|
| **RDS** | Relational | MySQL, PostgreSQL, Oracle, SQL Server |
| **Aurora** | Relational | High performance MySQL/PostgreSQL |
| **DynamoDB** | NoSQL (KV/Doc) | High scale, low latency |
| **ElastiCache** | In-memory | Caching (Redis/Memcached) |
| **Redshift** | Data Warehouse | Analytics, OLAP |
| **DocumentDB** | Document | MongoDB compatible |
| **Neptune** | Graph | Graph databases |

### Networking

| Service | Purpose |
|---------|---------|
| **VPC** | Isolated network |
| **Route 53** | DNS |
| **CloudFront** | CDN |
| **API Gateway** | API management |
| **ELB** | Load balancing |
| **Direct Connect** | Dedicated connection |
| **Transit Gateway** | Multi-VPC connectivity |

---

## Pricing Models

### Compute Pricing

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      EC2 Pricing Options                                 │
├─────────────────┬──────────────┬────────────────────────────────────────┤
│ On-Demand       │ Full price   │ Short-term, unpredictable workloads    │
├─────────────────┼──────────────┼────────────────────────────────────────┤
│ Reserved (1-3yr)│ Up to 72% off│ Steady-state, predictable usage        │
├─────────────────┼──────────────┼────────────────────────────────────────┤
│ Spot Instances  │ Up to 90% off│ Fault-tolerant, flexible workloads     │
├─────────────────┼──────────────┼────────────────────────────────────────┤
│ Savings Plans   │ Up to 72% off│ Commitment to $/hour, flexible usage   │
├─────────────────┼──────────────┼────────────────────────────────────────┤
│ Dedicated Hosts │ Premium      │ Compliance, licensing requirements     │
└─────────────────┴──────────────┴────────────────────────────────────────┘
```

### Storage Pricing

```
S3 Storage Classes (per GB/month, us-east-1):
┌─────────────────────┬─────────┬───────────────────────────────────────┐
│ Standard            │ $0.023  │ Frequently accessed                   │
│ Intelligent-Tiering │ $0.023+ │ Unknown/changing access patterns      │
│ Standard-IA         │ $0.0125 │ Infrequent access, rapid retrieval    │
│ One Zone-IA         │ $0.01   │ Infrequent, non-critical             │
│ Glacier Instant     │ $0.004  │ Archive, millisecond retrieval        │
│ Glacier Flexible    │ $0.0036 │ Archive, minutes to hours retrieval   │
│ Glacier Deep Archive│ $0.00099│ Archive, 12+ hour retrieval           │
└─────────────────────┴─────────┴───────────────────────────────────────┘
```

### Data Transfer Pricing

```
Key Rules:
1. Inbound to AWS: FREE
2. Same AZ: FREE
3. Cross-AZ (private IP): $0.01/GB each way
4. Cross-Region: $0.02/GB
5. Internet Outbound: $0.09/GB (first 10TB)

Cost Optimization:
- Use VPC endpoints for AWS services
- Keep traffic within same AZ when possible
- Use CloudFront for outbound (often cheaper)
```

---

## Interview Discussion Points

### How do you design for high availability?

```
1. Multi-AZ deployment
   - RDS Multi-AZ, ELB across AZs, ASG across AZs

2. Stateless applications
   - Store state in DynamoDB/ElastiCache, not local

3. Health checks and auto-recovery
   - ELB health checks, ASG replacement

4. Database HA
   - RDS Multi-AZ, Aurora replicas, DynamoDB global tables

5. DNS failover
   - Route 53 health checks, failover routing
```

### How do you choose between services?

```
EC2 vs Lambda:
- EC2: Long-running, predictable, specific runtime needs
- Lambda: Event-driven, short-duration, variable load

RDS vs DynamoDB:
- RDS: Complex queries, transactions, relational data
- DynamoDB: Simple queries, massive scale, key-value

ECS vs EKS:
- ECS: AWS-native, simpler, tighter integration
- EKS: Kubernetes expertise, portability, ecosystem

S3 vs EFS vs EBS:
- S3: Object storage, web content, backups
- EFS: Shared file system across instances
- EBS: Single instance block storage (database)
```

### What's your approach to cost optimization?

```
1. Right-sizing
   - Use Compute Optimizer recommendations
   - Start small, scale up based on metrics

2. Reserved capacity
   - Reserved Instances for steady workloads
   - Savings Plans for flexibility

3. Spot Instances
   - Batch processing, fault-tolerant workloads
   - Mix with On-Demand in ASG

4. Storage tiering
   - S3 lifecycle policies
   - Intelligent-Tiering for unknown patterns

5. Architectural changes
   - Serverless where appropriate
   - Caching to reduce compute/database load
```
