# Common AWS Architecture Patterns

## Three-Tier Web Application

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Internet                                        │
│                              │                                           │
│                        ┌─────┴─────┐                                    │
│                        │ Route 53  │                                    │
│                        └─────┬─────┘                                    │
│                              │                                           │
│                        ┌─────┴─────┐                                    │
│                        │CloudFront │                                    │
│                        └─────┬─────┘                                    │
│                              │                                           │
│  ┌───────────────────────────┴───────────────────────────┐              │
│  │                         VPC                            │              │
│  │  ┌─────────────────────────────────────────────────┐  │              │
│  │  │ Public Subnets                                   │  │              │
│  │  │  ┌─────────────────────────────────────────┐    │  │              │
│  │  │  │          Application Load Balancer       │    │  │              │
│  │  │  └─────────────────┬───────────────────────┘    │  │              │
│  │  └────────────────────┼────────────────────────────┘  │              │
│  │  ┌────────────────────┼────────────────────────────┐  │              │
│  │  │ Private Subnets    │                            │  │              │
│  │  │  ┌─────────────────┼───────────────────────┐    │  │              │
│  │  │  │     Auto Scaling Group (Web Tier)       │    │  │              │
│  │  │  │  ┌───────┐  ┌───────┐  ┌───────┐        │    │  │              │
│  │  │  │  │ EC2   │  │ EC2   │  │ EC2   │        │    │  │              │
│  │  │  │  └───┬───┘  └───┬───┘  └───┬───┘        │    │  │              │
│  │  │  └─────┼───────────┼───────────┼───────────┘    │  │              │
│  │  │        └───────────┼───────────┘                │  │              │
│  │  │                    │                            │  │              │
│  │  │  ┌─────────────────┼───────────────────────┐    │  │              │
│  │  │  │        ElastiCache (Session/Cache)      │    │  │              │
│  │  │  └─────────────────┬───────────────────────┘    │  │              │
│  │  └────────────────────┼────────────────────────────┘  │              │
│  │  ┌────────────────────┼────────────────────────────┐  │              │
│  │  │ Data Subnets       │                            │  │              │
│  │  │  ┌─────────────────┼───────────────────────┐    │  │              │
│  │  │  │           RDS Multi-AZ                  │    │  │              │
│  │  │  │    ┌─────────┐        ┌─────────┐       │    │  │              │
│  │  │  │    │ Primary │◄──────▶│ Standby │       │    │  │              │
│  │  │  │    └─────────┘        └─────────┘       │    │  │              │
│  │  │  └─────────────────────────────────────────┘    │  │              │
│  │  └─────────────────────────────────────────────────┘  │              │
│  └────────────────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────────────┘

Key Design Decisions:
- CloudFront for static content caching
- ALB for HTTPS termination and routing
- Auto Scaling for elasticity
- ElastiCache for session state (stateless servers)
- RDS Multi-AZ for database HA
- Private subnets for app/data layers
```

---

## Serverless Web Application

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   CloudFront ──────▶ S3 (Static assets: HTML, CSS, JS)                  │
│       │                                                                  │
│       │ /api/*                                                          │
│       ▼                                                                  │
│   API Gateway ──────▶ Lambda ──────▶ DynamoDB                           │
│       │                  │                                               │
│       │                  ├──────▶ S3 (file storage)                     │
│       │                  │                                               │
│       │                  └──────▶ SQS (async processing)                │
│       │                              │                                   │
│       │                         Lambda (worker)                          │
│       │                                                                  │
│   Cognito (Authentication)                                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Benefits:
- No servers to manage
- Pay per request
- Automatic scaling
- High availability built-in

When to Use:
- Variable/unpredictable traffic
- Event-driven workloads
- Rapid development
- Cost optimization for low-traffic apps
```

---

## Event-Driven Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Order Service                                                          │
│       │                                                                  │
│       │ OrderCreated event                                              │
│       ▼                                                                  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      EventBridge                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│       │                    │                    │                        │
│       ▼                    ▼                    ▼                        │
│   ┌─────────┐         ┌─────────┐         ┌─────────┐                   │
│   │ SQS     │         │ SQS     │         │ SQS     │                   │
│   └────┬────┘         └────┬────┘         └────┬────┘                   │
│        │                   │                   │                         │
│        ▼                   ▼                   ▼                         │
│   ┌─────────┐         ┌─────────┐         ┌─────────┐                   │
│   │Inventory│         │ Payment │         │Notific- │                   │
│   │ Service │         │ Service │         │ation    │                   │
│   └─────────┘         └─────────┘         └─────────┘                   │
│                                                                          │
│   Pattern: Event sourcing with loose coupling                           │
│   Benefits: Independent scaling, failure isolation, easy to extend      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Microservices with ECS/EKS

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Route 53 ──▶ ALB ──▶ ECS/EKS Cluster                                  │
│                           │                                              │
│   ┌───────────────────────┼───────────────────────────┐                 │
│   │                       │                           │                 │
│   │   ┌───────────────────┼───────────────────────┐   │                 │
│   │   │      Service Discovery (Cloud Map)        │   │                 │
│   │   └───────────────────┬───────────────────────┘   │                 │
│   │                       │                           │                 │
│   │   ┌───────────┬───────┴───────┬───────────┐      │                 │
│   │   │           │               │           │      │                 │
│   │   ▼           ▼               ▼           ▼      │                 │
│   │ ┌─────┐   ┌─────┐         ┌─────┐     ┌─────┐   │                 │
│   │ │User │   │Order│         │Inv- │     │Pay- │   │                 │
│   │ │Svc  │   │Svc  │         │entory│    │ment │   │                 │
│   │ │     │   │     │         │Svc  │     │Svc  │   │                 │
│   │ └──┬──┘   └──┬──┘         └──┬──┘     └──┬──┘   │                 │
│   │    │         │               │           │      │                 │
│   │    ▼         ▼               ▼           ▼      │                 │
│   │ ┌─────┐   ┌─────┐         ┌─────┐     ┌─────┐   │                 │
│   │ │RDS  │   │Dynamo│        │RDS  │     │Dynamo│  │                 │
│   │ └─────┘   │DB   │         └─────┘     │DB   │   │                 │
│   │           └─────┘                     └─────┘   │                 │
│   │                                                 │                 │
│   │   X-Ray for tracing across services             │                 │
│   │   CloudWatch Container Insights                 │                 │
│   │   AppMesh for service mesh (optional)          │                 │
│   │                                                 │                 │
│   └─────────────────────────────────────────────────┘                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Data Lake Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Data Sources                                                           │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐                       │
│   │ Apps    │ │ IoT     │ │ Logs    │ │ External│                       │
│   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘                       │
│        │           │           │           │                             │
│        └───────────┴─────┬─────┴───────────┘                             │
│                          │                                               │
│   Ingestion              ▼                                               │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  Kinesis Data Firehose  │  AWS Glue  │  DMS  │  Direct Upload │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                          │                                               │
│   Storage                ▼                                               │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │                    S3 Data Lake                               │     │
│   │  ┌─────────────┬─────────────┬─────────────┐                 │     │
│   │  │ Raw Zone    │ Processed   │ Curated     │                 │     │
│   │  │ (Bronze)    │ (Silver)    │ (Gold)      │                 │     │
│   │  └─────────────┴─────────────┴─────────────┘                 │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                          │                                               │
│   Processing             ▼                                               │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  AWS Glue  │  EMR  │  Athena  │  Redshift Spectrum           │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                          │                                               │
│   Consumption            ▼                                               │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  QuickSight  │  SageMaker  │  API Gateway  │  Redshift       │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                                                                          │
│   Governance: Lake Formation, Glue Data Catalog                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Multi-Region Active-Active

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Route 53 (Latency-based routing)                                      │
│       │                                                                  │
│       ├─────────────────────────────────┬────────────────────────────┐  │
│       │                                 │                            │  │
│       ▼                                 ▼                            │  │
│   US-EAST-1                         EU-WEST-1                        │  │
│   ┌─────────────────────────┐       ┌─────────────────────────┐     │  │
│   │                         │       │                         │     │  │
│   │  CloudFront             │       │  CloudFront             │     │  │
│   │       │                 │       │       │                 │     │  │
│   │       ▼                 │       │       ▼                 │     │  │
│   │  ALB ──▶ ECS/EKS       │       │  ALB ──▶ ECS/EKS       │     │  │
│   │       │                 │       │       │                 │     │  │
│   │       ▼                 │       │       ▼                 │     │  │
│   │  Aurora Global DB       │◄─────▶│  Aurora Global DB       │     │  │
│   │  (Writer)              │ async │  (Reader/Promote)      │     │  │
│   │                         │       │                         │     │  │
│   │  DynamoDB Global        │◄─────▶│  DynamoDB Global        │     │  │
│   │  Tables                 │ async │  Tables                 │     │  │
│   │                         │       │                         │     │  │
│   │  ElastiCache            │       │  ElastiCache            │     │  │
│   │  (local cache)          │       │  (local cache)          │     │  │
│   │                         │       │                         │     │  │
│   └─────────────────────────┘       └─────────────────────────┘     │  │
│                                                                          │
│   Considerations:                                                        │
│   - Data replication lag (<1 second)                                    │
│   - Conflict resolution (last writer wins or app logic)                 │
│   - Cost (double infrastructure)                                        │
│   - Complexity (deployment, testing)                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Real-Time Data Processing

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Data Producers                                                         │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐                                  │
│   │ IoT     │ │ Clicks  │ │ Logs    │                                  │
│   └────┬────┘ └────┬────┘ └────┬────┘                                  │
│        │           │           │                                         │
│        └───────────┴─────┬─────┘                                         │
│                          │                                               │
│   Ingestion              ▼                                               │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │                   Kinesis Data Streams                        │     │
│   │              (Shards for parallel processing)                 │     │
│   └───────────────────────────────┬───────────────────────────────┘     │
│                                   │                                      │
│             ┌─────────────────────┼─────────────────────┐               │
│             │                     │                     │               │
│             ▼                     ▼                     ▼               │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐      │
│   │ Kinesis Data    │   │ Lambda          │   │ Kinesis Data    │      │
│   │ Analytics       │   │ (real-time      │   │ Firehose        │      │
│   │ (SQL queries)   │   │  processing)    │   │ (to S3/Redshift)│      │
│   └────────┬────────┘   └────────┬────────┘   └────────┬────────┘      │
│            │                     │                     │                │
│            ▼                     ▼                     ▼                │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐      │
│   │ OpenSearch      │   │ DynamoDB        │   │ S3 Data Lake    │      │
│   │ (dashboards)    │   │ (state store)   │   │ (archive)       │      │
│   └─────────────────┘   └─────────────────┘   └─────────────────┘      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Developer                                                              │
│       │                                                                  │
│       │ git push                                                        │
│       ▼                                                                  │
│   ┌─────────────────┐                                                   │
│   │   CodeCommit/   │                                                   │
│   │   GitHub        │                                                   │
│   └────────┬────────┘                                                   │
│            │ webhook                                                     │
│            ▼                                                             │
│   ┌─────────────────┐                                                   │
│   │  CodePipeline   │                                                   │
│   └────────┬────────┘                                                   │
│            │                                                             │
│   ┌────────┴────────────────────────────────────────────┐               │
│   │                                                      │               │
│   ▼                                                      │               │
│   ┌─────────────────┐                                   │               │
│   │   CodeBuild     │                                   │               │
│   │   (Build/Test)  │                                   │               │
│   └────────┬────────┘                                   │               │
│            │                                             │               │
│            ▼                                             │               │
│   ┌─────────────────┐                                   │               │
│   │   ECR (images)  │                                   │               │
│   │   S3 (artifacts)│                                   │               │
│   └────────┬────────┘                                   │               │
│            │                                             │               │
│   ┌────────┴───────────────────────────────┐            │               │
│   │                                        │            │               │
│   ▼                                        ▼            │               │
│   ┌─────────────────┐            ┌─────────────────┐   │               │
│   │  Deploy Dev     │            │  Manual Approval │   │               │
│   │  (CodeDeploy)   │            │                 │   │               │
│   └────────┬────────┘            └────────┬────────┘   │               │
│            │                              │             │               │
│            ▼                              ▼             │               │
│   ┌─────────────────┐            ┌─────────────────┐   │               │
│   │  Integration    │            │  Deploy Prod    │   │               │
│   │  Tests          │            │  (Blue/Green)   │   │               │
│   └─────────────────┘            └─────────────────┘   │               │
│                                                         │               │
│   Notifications: SNS to Slack/Email                     │               │
│                                                         │               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Interview Discussion Points

### How do you approach designing a new system on AWS?

```
Framework:

1. Understand Requirements
   - Functional requirements
   - Non-functional (scalability, latency, availability)
   - Constraints (budget, compliance, existing systems)

2. Start with Well-Architected
   - Operational Excellence
   - Security
   - Reliability
   - Performance Efficiency
   - Cost Optimization
   - Sustainability

3. Choose Appropriate Services
   - Managed services first
   - Serverless when possible
   - Right tool for the job

4. Design for Failure
   - Multi-AZ
   - Health checks
   - Auto-recovery
   - Graceful degradation

5. Plan for Scale
   - Auto Scaling from day one
   - Stateless design
   - Caching layers

6. Security by Design
   - Least privilege
   - Encryption everywhere
   - Network segmentation

7. Observability
   - Metrics, logs, traces
   - Alerting strategy
   - Dashboards
```

### How do you migrate an existing application to AWS?

```
Migration Strategies (6 Rs):

1. Rehost (Lift and Shift)
   - Move as-is to EC2
   - Fastest, minimal changes
   - Use AWS Application Migration Service

2. Replatform (Lift and Reshape)
   - Minor optimizations
   - RDS instead of self-managed DB
   - Managed services where easy

3. Refactor (Re-architect)
   - Significant changes for cloud-native
   - Microservices, serverless
   - Most benefit, most effort

4. Repurchase
   - Move to SaaS
   - Replace legacy with managed service

5. Retire
   - Decommission unused systems

6. Retain
   - Keep on-premises (for now)

Migration Process:
1. Discovery: Map applications and dependencies
2. Planning: Choose strategy per application
3. Design: Target architecture
4. Migration: Execute with validation
5. Operate: Optimize and iterate
```
