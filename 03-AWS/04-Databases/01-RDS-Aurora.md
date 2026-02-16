# RDS & Aurora (Relational Databases)

## RDS Overview

```
RDS = Managed Relational Database Service

Supported Engines:
┌─────────────────────────────────────────────────────────────────────────┐
│  MySQL    │  PostgreSQL  │  MariaDB  │  Oracle  │  SQL Server  │ Aurora │
└─────────────────────────────────────────────────────────────────────────┘

AWS Manages:                    You Manage:
- Hardware provisioning         - Schema design
- Database setup               - Query optimization
- Patching                     - Index management
- Backups                      - Application code
- Monitoring                   - Data
- Scaling (vertical)
- HA (Multi-AZ)
```

---

## RDS Instance Classes

```
Instance Classes:
┌─────────────────────────────────────────────────────────────────────────┐
│ Standard (db.m)     │ General purpose, balanced                         │
│ Memory (db.r)       │ Memory-intensive workloads                        │
│ Burstable (db.t)    │ Variable workloads, dev/test                     │
└─────────────────────────────────────────────────────────────────────────┘

Sizing Factors:
- vCPU count
- Memory (RAM)
- Network bandwidth
- EBS bandwidth

Example: db.r6g.2xlarge
         │ │ │ │
         │ │ │ └── Size (8 vCPU, 64 GB RAM)
         │ │ └──── Generation
         │ └────── Graviton (ARM) or blank for Intel
         └──────── Memory optimized
```

---

## High Availability

### Multi-AZ Deployment

```
Multi-AZ = Synchronous standby replica in different AZ

┌─────────────────────────────────────────────────────────────────────────┐
│                              Region                                      │
│                                                                          │
│   ┌─────────────────────────┐    ┌─────────────────────────┐           │
│   │       AZ-a              │    │       AZ-b              │           │
│   │                         │    │                         │           │
│   │   ┌─────────────────┐   │    │   ┌─────────────────┐   │           │
│   │   │    Primary      │───┼────┼──▶│    Standby      │   │           │
│   │   │   (Active)      │   │sync│   │   (Passive)     │   │           │
│   │   └─────────────────┘   │    │   └─────────────────┘   │           │
│   │          │              │    │                         │           │
│   │   ┌──────┴──────┐       │    │                         │           │
│   │   │  EBS Volume │       │    │                         │           │
│   │   └─────────────┘       │    │                         │           │
│   └─────────────────────────┘    └─────────────────────────┘           │
│                                                                          │
│   Endpoint: mydb.xxx.region.rds.amazonaws.com                           │
│   (DNS automatically updated on failover)                               │
└─────────────────────────────────────────────────────────────────────────┘

Failover Triggers:
- Primary instance failure
- AZ failure
- Instance type change
- Manual failover (maintenance)

Failover Time: 60-120 seconds
- DNS propagation
- Recovery process
```

### Read Replicas

```
Read Replicas = Asynchronous replicas for read scaling

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   ┌───────────┐        ┌───────────┐        ┌───────────┐              │
│   │  Primary  │──async─▶│  Replica  │        │  Replica  │              │
│   │           │──async──┼───────────┼──async─▶│ (Region B)│              │
│   └───────────┘        └───────────┘        └───────────┘              │
│        ▲                     ▲                     ▲                    │
│        │                     │                     │                    │
│   Writes only          Reads only           Cross-region               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Characteristics:
- Up to 5 read replicas per primary
- Cross-region replicas supported
- Replica lag (seconds to minutes)
- Can be promoted to standalone
- Separate connection endpoints

Use Cases:
- Read scaling (reporting, analytics)
- Cross-region disaster recovery
- Low-latency reads for global users
```

---

## Backups & Recovery

### Automated Backups

```
Automated Backups:
- Daily snapshots during backup window
- Transaction logs every 5 minutes
- Point-in-time recovery (PITR)
- Retention: 1-35 days

┌─────────────────────────────────────────────────────────────────────────┐
│                    Point-in-Time Recovery                                │
│                                                                          │
│   Day 1 ────────────────────────────────────────────────────── Day 7    │
│     │                                                             │      │
│   Daily      Transaction logs captured                         Daily    │
│  Snapshot     continuously (5 min)                            Snapshot  │
│     │              │                                             │      │
│     └──────────────┴──────────── Restore to any point ───────────┘      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Recovery Process:
1. Restore creates NEW instance
2. Apply transaction logs to target time
3. Update application to use new endpoint
```

### Manual Snapshots

```
Manual Snapshots:
- User-initiated
- Persist until deleted
- Can copy across regions
- Can share with other accounts

// Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier new-instance \
  --db-snapshot-identifier my-snapshot \
  --db-instance-class db.r6g.large

// Copy to another region
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:123:snapshot:my-snap \
  --target-db-snapshot-identifier dr-snapshot \
  --region eu-west-1
```

---

## RDS Security

### Network Security

```
Network Architecture:
┌─────────────────────────────────────────────────────────────────────────┐
│                              VPC                                         │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    Private Subnets (DB Subnet Group)             │   │
│   │                                                                  │   │
│   │   ┌───────────────────┐    ┌───────────────────┐                │   │
│   │   │    RDS Primary    │    │    RDS Standby    │                │   │
│   │   │                   │    │                   │                │   │
│   │   │  Security Group   │    │  Security Group   │                │   │
│   │   │  - Port 3306      │    │  - Port 3306      │                │   │
│   │   │  - Source: App SG │    │  - Source: App SG │                │   │
│   │   └───────────────────┘    └───────────────────┘                │   │
│   │              ▲                      ▲                            │   │
│   └──────────────┼──────────────────────┼────────────────────────────┘   │
│                  │                      │                                │
│   ┌──────────────┴──────────────────────┴────────────────────────────┐   │
│   │                    Private Subnets (App Tier)                     │   │
│   │   ┌───────────────────┐                                          │   │
│   │   │   EC2 / Lambda    │                                          │   │
│   │   │   (App Security   │                                          │   │
│   │   │    Group)         │                                          │   │
│   │   └───────────────────┘                                          │   │
│   └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘

Best Practices:
- No public accessibility
- DB in private subnets only
- Security groups allow only app tier
```

### Encryption & Authentication

```
Encryption at Rest:
- AES-256 encryption
- KMS managed keys
- Encrypt at creation (cannot enable later)
- Snapshots and replicas inherit encryption

Encryption in Transit:
- SSL/TLS connections
- Enforce with parameter group: require_secure_transport=1
- Download RDS CA certificate for validation

IAM Authentication:
- Token-based authentication
- No password in connection string
- Centralized access control

// Generate IAM auth token
aws rds generate-db-auth-token \
  --hostname mydb.xxx.us-east-1.rds.amazonaws.com \
  --port 3306 \
  --username my_user

// Use token as password (valid 15 minutes)
mysql -h mydb.xxx.us-east-1.rds.amazonaws.com \
  -u my_user \
  --password=$TOKEN \
  --ssl-ca=rds-ca-2019-root.pem
```

---

## Aurora Overview

```
Aurora = AWS-designed cloud-native database

Key Differentiators:
- Up to 5x throughput of MySQL, 3x of PostgreSQL
- Storage auto-scales (10 GB to 128 TB)
- 6-way replication across 3 AZs
- Up to 15 read replicas (vs 5 for RDS)
- Automatic failover (<30 seconds)

Aurora Architecture:
┌─────────────────────────────────────────────────────────────────────────┐
│                          Aurora Cluster                                  │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    Compute Layer                                 │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │   │
│   │   │   Primary   │  │  Replica 1  │  │  Replica N  │            │   │
│   │   │  (Writer)   │  │  (Reader)   │  │  (Reader)   │            │   │
│   │   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │   │
│   └──────────┼────────────────┼────────────────┼────────────────────┘   │
│              │                │                │                         │
│   ┌──────────┴────────────────┴────────────────┴────────────────────┐   │
│   │                 Shared Storage Layer                             │   │
│   │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │   │
│   │   │ AZ-a    │ │ AZ-a    │ │ AZ-b    │ │ AZ-b    │ │ AZ-c    │  │   │
│   │   │ Copy 1  │ │ Copy 2  │ │ Copy 3  │ │ Copy 4  │ │ Copy 5,6│  │   │
│   │   └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘  │   │
│   └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Writes: Quorum (4 of 6)    Reads: Quorum (3 of 6)                    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Aurora Features

### Aurora Endpoints

```
Endpoint Types:
┌─────────────────────────────────────────────────────────────────────────┐
│ Cluster Endpoint    │ Points to primary (writer)                       │
│ (Writer)            │ Use for: All writes, reads requiring consistency │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Reader Endpoint     │ Load balances across replicas                    │
│                     │ Use for: Read scaling, reporting                 │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Instance Endpoint   │ Connects to specific instance                    │
│                     │ Use for: Troubleshooting, specific routing       │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Custom Endpoint     │ User-defined group of instances                  │
│                     │ Use for: Analytical queries on larger instances  │
└─────────────────────┴───────────────────────────────────────────────────┘
```

### Aurora Serverless v2

```
Aurora Serverless v2:
- Auto-scaling compute (0.5 to 128 ACUs)
- Per-second billing
- No cold start (instant scaling)
- Full feature parity with provisioned

┌─────────────────────────────────────────────────────────────────────────┐
│                    Scaling Behavior                                      │
│                                                                          │
│  ACUs                                                                    │
│    │                    ┌──────┐                                        │
│ 32 │                    │      │                                        │
│    │              ┌─────┘      └─────┐                                  │
│ 16 │              │                  │                                  │
│    │        ┌─────┘                  └─────┐                            │
│  8 │        │                              │                            │
│    │  ┌─────┘                              └─────────┐                  │
│  2 │──┘                                              └────────          │
│    └──────────────────────────────────────────────────────────▶ Time   │
│                                                                          │
│  Min ACUs: 0.5 (can scale to zero with v2)                              │
│  Max ACUs: 128                                                          │
│  1 ACU ≈ 2 GB memory                                                    │
└─────────────────────────────────────────────────────────────────────────┘

Use Cases:
- Variable workloads
- Development/test
- New applications with unknown demand
- Multi-tenant SaaS
```

### Aurora Global Database

```
Aurora Global Database:
- Primary region: Read/write
- Secondary regions: Read-only (up to 5)
- RPO: ~1 second (typical)
- RTO: < 1 minute (promote secondary)

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   US-EAST-1 (Primary)                  EU-WEST-1 (Secondary)            │
│   ┌───────────────────┐               ┌───────────────────┐             │
│   │   Aurora Cluster  │──────────────▶│   Aurora Cluster  │             │
│   │                   │   Storage     │                   │             │
│   │   Writer + 2      │   Replication │   Up to 16        │             │
│   │   Readers         │   (<1 sec)    │   Readers         │             │
│   └───────────────────┘               └───────────────────┘             │
│                                                                          │
│   Use Cases:                                                             │
│   - Disaster recovery                                                   │
│   - Low-latency global reads                                            │
│   - Data locality compliance                                            │
└─────────────────────────────────────────────────────────────────────────┘

Failover:
1. Planned: Switchover (no data loss)
2. Unplanned: Promote secondary (may lose ~1 sec of data)
```

---

## Performance Optimization

### Query Performance

```
Performance Insights:
- DB load visualization
- Top SQL identification
- Wait event analysis
- Free for 7 days retention

// Common wait events
┌─────────────────────────────────────────────────────────────────────────┐
│ Wait Event          │ Indicates                                         │
├─────────────────────┼───────────────────────────────────────────────────┤
│ CPU                 │ Query execution, need query optimization         │
│ IO:DataFileRead     │ Buffer pool miss, need more memory or optimize   │
│ Lock:Wait           │ Lock contention, review transaction isolation    │
│ LWLock:BufferIO     │ Buffer pool contention                           │
│ Client:ClientRead   │ Application not reading results fast enough      │
└─────────────────────┴───────────────────────────────────────────────────┘

Optimization Strategies:
1. Query optimization (EXPLAIN, indexes)
2. Connection pooling (RDS Proxy)
3. Read scaling (replicas)
4. Caching (ElastiCache)
5. Right-sizing instances
```

### RDS Proxy

```
RDS Proxy = Managed connection pooler

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Lambda/App (many connections)                                         │
│        │                                                                │
│        ▼                                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                     RDS Proxy                                    │   │
│   │   - Connection pooling                                          │   │
│   │   - Multiplexing                                                │   │
│   │   - Failover handling                                           │   │
│   │   - IAM authentication                                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│        │                                                                │
│        ▼ (Pooled connections)                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │              RDS / Aurora                                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Benefits:
- 66% reduction in failover time
- 2x improvement in connection efficiency
- IAM authentication for Lambda
- Handles connection storms
```

---

## Interview Discussion Points

### When to use RDS vs Aurora?

```
Choose RDS When:
- Need specific engine version/feature
- Cost-sensitive (Aurora costs more)
- Simple HA sufficient (Multi-AZ)
- Storage predictable (no auto-scaling needed)
- Using Oracle or SQL Server

Choose Aurora When:
- Need maximum performance
- High availability critical (<30s failover)
- Read-heavy workloads (up to 15 replicas)
- Variable storage needs
- Global deployments (Aurora Global)
- Serverless use case (Aurora Serverless v2)

Cost Comparison:
- Aurora: ~20% more than RDS
- But: Better performance, storage efficiency
- Break-even often at moderate scale
```

### How do you design for zero-downtime migrations?

```
RDS to Aurora Migration:

Option 1: Snapshot Restore
- Create Aurora from RDS snapshot
- Downtime: Minutes to hours (depends on size)
- Simple but has downtime

Option 2: Aurora Read Replica
1. Create Aurora replica of RDS instance
2. Aurora syncs via native replication
3. Promote Aurora when caught up
4. Update application endpoint
- Downtime: Seconds (promotion time)

Option 3: DMS Continuous Replication
1. Set up DMS task (full load + CDC)
2. Replicate to Aurora
3. Cutover when caught up
- Downtime: Seconds (application cutover)

Key Considerations:
- Test in non-prod first
- Validate data integrity
- Plan rollback strategy
- Consider connection string management
```

### How do you handle database scaling?

```
Vertical Scaling:
- Change instance class (downtime: minutes)
- Multi-AZ reduces downtime (failover based)
- Aurora: Modify with minimal downtime

Horizontal Scaling (Reads):
- Add read replicas
- Aurora: Up to 15 replicas with reader endpoint
- RDS Proxy for connection management

Horizontal Scaling (Writes):
- Application-level sharding
- Or: Consider Aurora Global or DynamoDB
- RDS/Aurora = single writer

Storage Scaling:
- RDS: Manually increase (no downtime)
- Aurora: Auto-scales (no action needed)

Auto Scaling (Aurora):
- Auto Scaling for read replicas
- Based on CPU or connections
- Aurora Serverless v2 for compute
```

### How do you secure RDS in production?

```
Security Checklist:

1. Network
   □ Private subnets only (no public IP)
   □ Security groups restrict to app tier
   □ NACLs for additional control
   □ VPC endpoints for AWS services

2. Authentication
   □ IAM database authentication
   □ Secrets Manager for credentials
   □ Rotate credentials automatically
   □ Enforce password policies

3. Encryption
   □ Encryption at rest (KMS)
   □ Encryption in transit (SSL/TLS)
   □ Encrypt snapshots
   □ Customer-managed keys for compliance

4. Auditing
   □ Enable database activity streams
   □ CloudTrail for API calls
   □ Performance Insights for query analysis
   □ Enhanced monitoring

5. Access Control
   □ Least privilege IAM policies
   □ Database user permissions
   □ Parameter groups (disable dangerous features)
   □ Regular access reviews
```
