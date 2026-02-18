# Disaster Recovery on AWS

## Overview

```
DR Fundamentals:

RTO (Recovery Time Objective): Maximum acceptable downtime
RPO (Recovery Point Objective): Maximum acceptable data loss

Example: RTO=4 hours, RPO=1 hour
├── System can be down for up to 4 hours
└── Can lose up to 1 hour of data

┌─────────────────────────────────────────────────────────────────────────────┐
│                         DR Strategies Spectrum                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Backup &     Pilot       Warm         Multi-Site       Multi-Region       │
│  Restore      Light       Standby      Active-Active    Active-Active      │
│                                                                             │
│  ◄──────────────────────────────────────────────────────────────────────►  │
│                                                                             │
│  Hours        Minutes     Minutes      Seconds          Near-zero          │
│  RTO          RTO         RTO          RTO              RTO                │
│                                                                             │
│  $            $$          $$$          $$$$             $$$$$              │
│  Cost         Cost        Cost         Cost             Cost               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## DR Strategies

### 1. Backup and Restore

```
Strategy: Store backups in another region, restore when needed

┌─────────────────────────────────────────────────────────────────────────────┐
│                        Normal Operation                                      │
│                                                                             │
│    Primary Region (us-east-1)           DR Region (us-west-2)              │
│    ┌────────────────────────┐           ┌────────────────────────┐         │
│    │                        │           │                        │         │
│    │   ┌──────────────┐     │           │   (No infrastructure)  │         │
│    │   │  Application │     │           │                        │         │
│    │   └──────────────┘     │  Backups  │   ┌────────────────┐   │         │
│    │   ┌──────────────┐     │───────────│──►│   S3 Bucket    │   │         │
│    │   │   Database   │     │           │   │   (Backups)    │   │         │
│    │   └──────────────┘     │           │   └────────────────┘   │         │
│    │   ┌──────────────┐     │           │   ┌────────────────┐   │         │
│    │   │   S3 Data    │─────│───────────│──►│   S3 Replica   │   │         │
│    │   └──────────────┘     │  CRR      │   └────────────────┘   │         │
│    │                        │           │                        │         │
│    └────────────────────────┘           └────────────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        After Disaster                                        │
│                                                                             │
│    Primary Region (FAILED)              DR Region (us-west-2)              │
│    ┌────────────────────────┐           ┌────────────────────────┐         │
│    │                        │           │                        │         │
│    │         ╳ ╳ ╳          │  Restore  │   ┌──────────────┐     │         │
│    │       (Offline)        │───────────│──►│  Application │     │         │
│    │                        │           │   └──────────────┘     │         │
│    │                        │           │   ┌──────────────┐     │         │
│    │                        │           │   │   Database   │     │         │
│    │                        │           │   │ (from backup)│     │         │
│    │                        │           │   └──────────────┘     │         │
│    │                        │           │                        │         │
│    └────────────────────────┘           └────────────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Characteristics:
├── RTO: Hours (depends on data size and complexity)
├── RPO: Hours (last backup point)
├── Cost: Lowest ($)
└── Best for: Non-critical systems, cost-sensitive workloads

Implementation:
- Automated daily backups (RDS snapshots, EBS snapshots)
- Cross-region S3 replication
- AMI copies to DR region
- Infrastructure as Code for quick provisioning
- Regular restore testing
```

### 2. Pilot Light

```
Strategy: Core components always running, scale up on failover

┌─────────────────────────────────────────────────────────────────────────────┐
│                        Normal Operation                                      │
│                                                                             │
│    Primary Region (us-east-1)           DR Region (us-west-2)              │
│    ┌────────────────────────┐           ┌────────────────────────┐         │
│    │                        │           │                        │         │
│    │   ┌──────────────┐     │           │   (No app servers)     │         │
│    │   │  App Server  │     │           │                        │         │
│    │   │   (Active)   │     │           │                        │         │
│    │   └──────────────┘     │           │                        │         │
│    │   ┌──────────────┐     │           │   ┌──────────────┐     │         │
│    │   │   RDS        │─────│───────────│──►│ RDS Read     │     │         │
│    │   │   Primary    │ Async│          │   │ Replica      │     │         │
│    │   └──────────────┘ Repl│           │   │ (Running)    │     │         │
│    │                        │           │   └──────────────┘     │         │
│    │                        │           │                        │         │
│    └────────────────────────┘           └────────────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        After Disaster                                        │
│                                                                             │
│    Primary Region (FAILED)              DR Region (us-west-2)              │
│    ┌────────────────────────┐           ┌────────────────────────┐         │
│    │                        │           │                        │         │
│    │         ╳ ╳ ╳          │  Scale    │   ┌──────────────┐     │         │
│    │       (Offline)        │───────────│──►│  App Server  │     │         │
│    │                        │    Up     │   │  (Launched)  │     │         │
│    │                        │           │   └──────────────┘     │         │
│    │                        │           │   ┌──────────────┐     │         │
│    │                        │           │   │ RDS          │     │         │
│    │                        │           │   │ (Promoted)   │     │         │
│    │                        │           │   └──────────────┘     │         │
│    │                        │           │                        │         │
│    └────────────────────────┘           └────────────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Characteristics:
├── RTO: 10-30 minutes
├── RPO: Minutes (replication lag)
├── Cost: Low-Medium ($$)
└── Best for: Business-critical systems with moderate RTO

Pilot Light Components (always running):
- Database replicas
- Core networking (VPC, subnets)
- DNS configuration ready
- Latest AMIs available

Scale-up Components (launched on failover):
- Application servers
- Load balancers
- Caches
```

### 3. Warm Standby

```
Strategy: Scaled-down but fully functional replica

┌─────────────────────────────────────────────────────────────────────────────┐
│                        Normal Operation                                      │
│                                                                             │
│    Primary Region (us-east-1)           DR Region (us-west-2)              │
│    ┌────────────────────────┐           ┌────────────────────────┐         │
│    │                        │           │                        │         │
│    │   ┌──────────────┐     │           │   ┌──────────────┐     │         │
│    │   │  ALB         │     │           │   │  ALB         │     │         │
│    │   └──────────────┘     │           │   └──────────────┘     │         │
│    │   ┌───┐ ┌───┐ ┌───┐   │           │   ┌───┐               │         │
│    │   │App│ │App│ │App│   │           │   │App│ (minimal)     │         │
│    │   └───┘ └───┘ └───┘   │           │   └───┘               │         │
│    │   ┌──────────────┐     │           │   ┌──────────────┐     │         │
│    │   │  RDS Multi-AZ│─────│───────────│──►│ RDS Replica  │     │         │
│    │   │  (Primary)   │ Async│          │   │ (Running)    │     │         │
│    │   └──────────────┘ Repl│           │   └──────────────┘     │         │
│    │                        │           │                        │         │
│    └────────────────────────┘           └────────────────────────┘         │
│                                                                             │
│    Route 53 Weighted: 100%              Route 53 Weighted: 0%              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        After Disaster                                        │
│                                                                             │
│    Primary Region (FAILED)              DR Region (us-west-2)              │
│    ┌────────────────────────┐           ┌────────────────────────┐         │
│    │                        │           │                        │         │
│    │         ╳ ╳ ╳          │  Scale    │   ┌──────────────┐     │         │
│    │       (Offline)        │───────────│──►│  ALB         │     │         │
│    │                        │    +      │   └──────────────┘     │         │
│    │                        │  Promote  │   ┌───┐ ┌───┐ ┌───┐   │         │
│    │                        │           │   │App│ │App│ │App│   │         │
│    │                        │           │   └───┘ └───┘ └───┘   │         │
│    │                        │           │   ┌──────────────┐     │         │
│    │                        │           │   │ RDS (Primary)│     │         │
│    │                        │           │   └──────────────┘     │         │
│    │                        │           │                        │         │
│    └────────────────────────┘           └────────────────────────┘         │
│                                                                             │
│                                         Route 53: 100%                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Characteristics:
├── RTO: Minutes
├── RPO: Seconds to minutes
├── Cost: Medium ($$$)
└── Best for: Business-critical with low RTO requirements

Warm Standby vs Pilot Light:
┌────────────────────────────────────────────────────────────────────────────┐
│                    Pilot Light          Warm Standby                       │
├────────────────────────────────────────────────────────────────────────────┤
│ App servers         Not running         Running (scaled down)              │
│ Load balancers      Not provisioned     Provisioned                        │
│ Database            Replica only        Replica (can serve reads)          │
│ Scale-up time       10-30 minutes       2-5 minutes                        │
│ Can take traffic?   No                  Yes (limited)                      │
└────────────────────────────────────────────────────────────────────────────┘
```

### 4. Multi-Site Active-Active

```
Strategy: Fully active in multiple regions simultaneously

┌─────────────────────────────────────────────────────────────────────────────┐
│                     Active-Active Operation                                  │
│                                                                             │
│    Region A (us-east-1)                 Region B (us-west-2)               │
│    ┌────────────────────────┐           ┌────────────────────────┐         │
│    │                        │           │                        │         │
│    │   ┌──────────────┐     │           │   ┌──────────────┐     │         │
│    │   │  ALB         │     │           │   │  ALB         │     │         │
│    │   └──────────────┘     │           │   └──────────────┘     │         │
│    │   ┌───┐ ┌───┐ ┌───┐   │           │   ┌───┐ ┌───┐ ┌───┐   │         │
│    │   │App│ │App│ │App│   │           │   │App│ │App│ │App│   │         │
│    │   └───┘ └───┘ └───┘   │           │   └───┘ └───┘ └───┘   │         │
│    │   ┌──────────────┐     │   Sync    │   ┌──────────────┐     │         │
│    │   │  Aurora      │◄────│───────────│──►│  Aurora      │     │         │
│    │   │  Global DB   │     │           │   │  Global DB   │     │         │
│    │   └──────────────┘     │           │   └──────────────┘     │         │
│    │                        │           │                        │         │
│    └────────────────────────┘           └────────────────────────┘         │
│              │                                     │                        │
│              └──────────────┬──────────────────────┘                        │
│                             │                                               │
│                    ┌────────────────┐                                       │
│                    │   Route 53     │                                       │
│                    │   Latency or   │                                       │
│                    │   Geolocation  │                                       │
│                    └────────────────┘                                       │
│                             │                                               │
│                    Users routed to                                          │
│                    nearest region                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                     After Region A Failure                                   │
│                                                                             │
│    Region A (FAILED)                    Region B (us-west-2)               │
│    ┌────────────────────────┐           ┌────────────────────────┐         │
│    │                        │           │                        │         │
│    │         ╳ ╳ ╳          │   All     │   ┌──────────────┐     │         │
│    │       (Offline)        │  Traffic  │   │  ALB         │     │         │
│    │                        │───────────│──►└──────────────┘     │         │
│    │                        │           │   ┌───┐ ┌───┐ ┌───┐   │         │
│    │                        │           │   │App│ │App│ │App│   │         │
│    │                        │           │   └───┘ └───┘ └───┘   │         │
│    │                        │           │   ┌──────────────┐     │         │
│    │                        │           │   │  Aurora      │     │         │
│    │                        │           │   │  (Promoted)  │     │         │
│    │                        │           │   └──────────────┘     │         │
│    │                        │           │                        │         │
│    └────────────────────────┘           └────────────────────────┘         │
│                                                                             │
│    Route 53 health check fails → Traffic shifts automatically              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Characteristics:
├── RTO: Near-zero (automatic failover)
├── RPO: Near-zero (synchronous or near-sync replication)
├── Cost: Highest ($$$$$)
└── Best for: Mission-critical, zero-downtime requirements

Key Components:
- Aurora Global Database (< 1 second replication lag)
- DynamoDB Global Tables (multi-master)
- Route 53 health checks + failover routing
- Global Accelerator for faster failover
- S3 Cross-Region Replication
```

---

## AWS Services for DR

### Database Replication

```
Aurora Global Database:

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│    Primary Region                       Secondary Region                   │
│    ┌───────────────────────┐           ┌───────────────────────┐          │
│    │  Aurora Cluster       │           │  Aurora Cluster       │          │
│    │  ┌─────────────────┐  │           │  ┌─────────────────┐  │          │
│    │  │ Writer Instance │  │  Storage  │  │ Reader Instance │  │          │
│    │  └─────────────────┘  │   Layer   │  └─────────────────┘  │          │
│    │  ┌─────────────────┐  │   Repl    │  ┌─────────────────┐  │          │
│    │  │ Reader Instance │◄─│───────────│─►│ Reader Instance │  │          │
│    │  └─────────────────┘  │  < 1 sec  │  └─────────────────┘  │          │
│    │                       │           │                       │          │
│    └───────────────────────┘           └───────────────────────┘          │
│                                                                             │
│    Failover: Promote secondary cluster (RPO: ~1 second)                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

DynamoDB Global Tables:

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│    Region A                             Region B                           │
│    ┌───────────────────────┐           ┌───────────────────────┐          │
│    │  DynamoDB Table       │           │  DynamoDB Table       │          │
│    │                       │  Multi-   │                       │          │
│    │  ┌─────────────────┐  │  Master   │  ┌─────────────────┐  │          │
│    │  │ Read/Write      │◄─│───────────│─►│ Read/Write      │  │          │
│    │  │ (Active)        │  │  Repl     │  │ (Active)        │  │          │
│    │  └─────────────────┘  │  ~1 sec   │  └─────────────────┘  │          │
│    │                       │           │                       │          │
│    └───────────────────────┘           └───────────────────────┘          │
│                                                                             │
│    Both regions actively serve reads AND writes                            │
│    Conflict resolution: Last writer wins                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

RDS Cross-Region Read Replicas:

Primary (us-east-1)                  Replica (us-west-2)
┌─────────────────┐                  ┌─────────────────┐
│ RDS MySQL       │   Async Repl     │ RDS MySQL       │
│ (Read/Write)    │─────────────────►│ (Read Only)     │
└─────────────────┘   ~minutes lag   └─────────────────┘
                                              │
                                     Promote to Primary
                                     (manual, ~minutes)
```

### Route 53 for DR

```yaml
# Failover Routing Policy
PrimaryRecord:
  Type: AWS::Route53::RecordSet
  Properties:
    Name: api.example.com
    Type: A
    SetIdentifier: primary
    Failover: PRIMARY
    AliasTarget:
      DNSName: !GetAtt PrimaryALB.DNSName
      HostedZoneId: !GetAtt PrimaryALB.CanonicalHostedZoneID
      EvaluateTargetHealth: true
    HealthCheckId: !Ref PrimaryHealthCheck

SecondaryRecord:
  Type: AWS::Route53::RecordSet
  Properties:
    Name: api.example.com
    Type: A
    SetIdentifier: secondary
    Failover: SECONDARY
    AliasTarget:
      DNSName: !GetAtt SecondaryALB.DNSName
      HostedZoneId: !GetAtt SecondaryALB.CanonicalHostedZoneID

# Health Check
PrimaryHealthCheck:
  Type: AWS::Route53::HealthCheck
  Properties:
    HealthCheckConfig:
      Type: HTTPS
      ResourcePath: /health
      FullyQualifiedDomainName: primary.example.com
      RequestInterval: 10
      FailureThreshold: 2
    HealthCheckTags:
      - Key: Name
        Value: primary-health-check
```

### S3 Cross-Region Replication

```yaml
SourceBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: my-source-bucket
    VersioningConfiguration:
      Status: Enabled  # Required for CRR
    ReplicationConfiguration:
      Role: !GetAtt ReplicationRole.Arn
      Rules:
        - Id: ReplicateAll
          Status: Enabled
          Priority: 1
          DeleteMarkerReplication:
            Status: Disabled
          Filter:
            Prefix: ''
          Destination:
            Bucket: !Sub 'arn:aws:s3:::my-dest-bucket-${AWS::Region}'
            StorageClass: STANDARD
            ReplicationTime:
              Status: Enabled
              Time:
                Minutes: 15  # S3 RTC (Replication Time Control)
            Metrics:
              Status: Enabled
              EventThreshold:
                Minutes: 15

# S3 RTC guarantees 99.99% of objects replicated within 15 minutes
```

---

## DR Testing and Automation

### Chaos Engineering

```python
# AWS Fault Injection Simulator experiment
import boto3

fis = boto3.client('fis')

# Experiment: Terminate random EC2 instances
experiment_template = {
    'description': 'Terminate random instances in ASG',
    'targets': {
        'instances': {
            'resourceType': 'aws:ec2:instance',
            'resourceTags': {'Environment': 'prod'},
            'selectionMode': 'PERCENT(30)'  # 30% of instances
        }
    },
    'actions': {
        'terminate-instances': {
            'actionId': 'aws:ec2:terminate-instances',
            'targets': {'Instances': 'instances'}
        }
    },
    'stopConditions': [
        {
            'source': 'aws:cloudwatch:alarm',
            'value': 'arn:aws:cloudwatch:region:account:alarm:error-rate-high'
        }
    ],
    'roleArn': 'arn:aws:iam::account:role/FISRole'
}

# Create experiment
response = fis.create_experiment_template(**experiment_template)

# Run experiment
fis.start_experiment(
    experimentTemplateId=response['experimentTemplate']['id']
)
```

### DR Runbook Automation

```yaml
# Step Functions for DR Failover
DRFailoverStateMachine:
  Type: AWS::StepFunctions::StateMachine
  Properties:
    StateMachineName: dr-failover
    Definition:
      Comment: Automated DR failover procedure
      StartAt: DetectFailure
      States:
        DetectFailure:
          Type: Task
          Resource: arn:aws:lambda:region:account:function:check-primary-health
          Next: ConfirmFailover

        ConfirmFailover:
          Type: Choice
          Choices:
            - Variable: $.primaryHealthy
              BooleanEquals: true
              Next: NoActionNeeded
            - Variable: $.primaryHealthy
              BooleanEquals: false
              Next: NotifyTeam

        NotifyTeam:
          Type: Task
          Resource: arn:aws:states:::sns:publish.waitForTaskToken
          Parameters:
            TopicArn: arn:aws:sns:region:account:dr-notifications
            Message.$: States.Format('Primary region failure detected. Approve failover? TaskToken: {}', $$.Task.Token)
          TimeoutSeconds: 300
          Next: PromoteDatabase

        PromoteDatabase:
          Type: Task
          Resource: arn:aws:lambda:region:account:function:promote-aurora-secondary
          Catch:
            - ErrorEquals: ['States.ALL']
              ResultPath: $.error
              Next: FailoverFailed
          Next: UpdateDNS

        UpdateDNS:
          Type: Task
          Resource: arn:aws:lambda:region:account:function:update-route53
          Next: ScaleUp

        ScaleUp:
          Type: Task
          Resource: arn:aws:lambda:region:account:function:scale-asg
          Next: VerifyFailover

        VerifyFailover:
          Type: Task
          Resource: arn:aws:lambda:region:account:function:verify-dr-health
          Next: FailoverComplete

        FailoverComplete:
          Type: Succeed

        FailoverFailed:
          Type: Fail
          Error: FailoverError
          Cause: Failover procedure failed

        NoActionNeeded:
          Type: Succeed
```

### Regular DR Testing

```
DR Testing Schedule:

┌────────────────────────────────────────────────────────────────────────────┐
│ Test Type              │ Frequency     │ Scope                             │
├────────────────────────┼───────────────┼───────────────────────────────────┤
│ Backup Restoration     │ Monthly       │ Restore DB to test instance       │
│ Component Failover     │ Monthly       │ Fail individual components        │
│ Pilot Light Activation │ Quarterly     │ Full DR environment spin-up       │
│ Full DR Failover       │ Annually      │ Complete failover to DR region    │
│ Game Day               │ Annually      │ Unannounced failure injection     │
└────────────────────────┴───────────────┴───────────────────────────────────┘

Testing Checklist:
□ Verify backups are restorable
□ Test database promotion
□ Validate DNS failover time
□ Confirm application functionality
□ Test data replication lag
□ Measure actual RTO/RPO
□ Document issues and update runbooks
□ Train team on procedures
```

---

## Interview Discussion Points

### How do you design a DR strategy for a critical application?

```
Decision Framework:

1. Understand Requirements
   ├── What is acceptable downtime? (RTO)
   ├── How much data loss is acceptable? (RPO)
   ├── What is the budget?
   └── What are compliance requirements?

2. Assess Application Architecture
   ├── Stateless vs stateful components
   ├── Database dependencies
   ├── External service dependencies
   └── Data synchronization complexity

3. Select Strategy Based on RTO/RPO

   RTO: Hours, RPO: Hours → Backup & Restore
   ├── Daily backups
   ├── Cross-region S3 replication
   ├── IaC for quick provisioning
   └── Cost: ~10% of primary

   RTO: 30 min, RPO: Minutes → Pilot Light
   ├── Database replicas always running
   ├── Core networking pre-configured
   ├── AMIs ready in DR region
   └── Cost: ~15-20% of primary

   RTO: 5 min, RPO: Seconds → Warm Standby
   ├── Scaled-down replica running
   ├── Database replica (read traffic)
   ├── Route 53 health checks
   └── Cost: ~30-50% of primary

   RTO: Seconds, RPO: ~0 → Active-Active
   ├── Aurora Global Database
   ├── Full capacity both regions
   ├── Global Accelerator
   └── Cost: ~200% of single region

4. Plan for Failback
   ├── How to return to primary?
   ├── Data sync back to primary
   └── Avoid split-brain scenarios
```

### What are the key considerations for multi-region DR?

```
Critical Considerations:

1. Data Consistency
   ├── Replication lag affects RPO
   ├── Conflict resolution (last writer wins?)
   ├── Transaction consistency across regions
   └── Application awareness of eventual consistency

2. Network Architecture
   ├── VPC peering or Transit Gateway
   ├── DNS propagation time (TTL settings)
   ├── Static IP requirements (Global Accelerator)
   └── Cross-region latency impact

3. State Management
   ├── Session data (Redis Global Datastore)
   ├── Uploaded files (S3 CRR)
   ├── Cache warming in DR region
   └── Configuration consistency

4. Dependencies
   ├── Third-party services (region-specific?)
   ├── API keys and secrets (replicate to DR)
   ├── Certificates (ACM doesn't replicate)
   └── IAM roles and policies

5. Testing
   ├── Regular failover drills
   ├── Chaos engineering
   ├── Runbook validation
   └── Team training

6. Cost Optimization
   ├── Spot instances in DR (warm standby)
   ├── Reserved capacity planning
   ├── Right-sizing DR resources
   └── S3 storage classes for backups
```

### How do you calculate RTO and RPO?

```
RTO Calculation:

Total RTO = Detection + Decision + Execution + Verification

Example (Warm Standby):
├── Detection (health check fails): 1 minute
├── Decision (manual approval): 5 minutes (or 0 if automated)
├── Database promotion: 1 minute (Aurora)
├── DNS propagation: 1 minute (low TTL)
├── ASG scale-up: 2 minutes
└── Verification: 2 minutes
Total RTO: ~12 minutes

RPO Calculation:

RPO = Last successful replication point

Example:
├── Aurora Global DB: < 1 second (typical)
├── RDS Cross-Region Replica: 1-5 minutes (async)
├── S3 CRR with RTC: < 15 minutes (99.99% SLA)
├── DynamoDB Global Tables: < 1 second (typical)
└── Daily backups: Up to 24 hours

Interview Tip:
"For a payment processing system, I'd recommend Active-Active
with Aurora Global Database. The near-zero RPO ensures we
don't lose financial transactions, while the automatic
failover provides the near-zero RTO required for customer
SLAs. The higher cost is justified by business criticality."
```
