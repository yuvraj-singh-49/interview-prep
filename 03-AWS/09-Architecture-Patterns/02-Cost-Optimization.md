# Cost Optimization

## Cost Optimization Framework

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Cost Optimization Pillars                             │
├─────────────────┬─────────────────┬─────────────────┬───────────────────┤
│   Right-Size    │  Purchase       │  Increase       │  Optimize         │
│   Resources     │  Options        │  Elasticity     │  Storage          │
│                 │                 │                 │                   │
│ - Compute       │ - Reserved      │ - Auto Scale    │ - Tiering         │
│   Optimizer     │   Instances     │ - Serverless    │ - Lifecycle       │
│ - CloudWatch    │ - Savings Plans │ - Spot          │   policies        │
│   metrics       │ - Spot          │ - Schedule      │ - Compression     │
│                 │   Instances     │                 │                   │
└─────────────────┴─────────────────┴─────────────────┴───────────────────┘
```

---

## Compute Cost Optimization

### EC2 Right-Sizing

```
Right-Sizing Process:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. Analyze Usage (Compute Optimizer, CloudWatch)                      │
│                                                                          │
│   Instance: m5.xlarge                                                   │
│   CPU Utilization: Avg 15%, Max 40%                                     │
│   Memory: Avg 25%, Max 45%                                              │
│                                                                          │
│   ─────────────────────────────────────────────                         │
│                                                                          │
│   2. Recommendation                                                      │
│                                                                          │
│   Current: m5.xlarge ($0.192/hr) = $140/month                          │
│   Recommended: m5.large ($0.096/hr) = $70/month                        │
│   Savings: 50%                                                          │
│                                                                          │
│   ─────────────────────────────────────────────                         │
│                                                                          │
│   3. Validate and Apply                                                  │
│                                                                          │
│   - Test in staging                                                     │
│   - Monitor after resize                                                │
│   - Automate with ASG instance refresh                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Tools:
- AWS Compute Optimizer
- AWS Cost Explorer Right-Sizing
- CloudWatch metrics
- Third-party tools (CloudHealth, Spot.io)
```

### Purchasing Options

```
EC2 Pricing Comparison:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Workload Analysis:                                                     │
│                                                                          │
│   ┌──────────────────────────────────────────────────────────────┐      │
│   │ Capacity                                                      │      │
│   │    │                                                          │      │
│   │    │     ┌──────────────────────────────┐                    │      │
│   │    │     │  On-Demand / Spot           │ (spikes)           │      │
│   │    │ ┌───┴──────────────────────────────┴───┐                │      │
│   │    │ │  Savings Plans (flexible)            │                │      │
│   │    │ │                                      │                │      │
│   │ ┌──┴─┴──────────────────────────────────────┴──┐             │      │
│   │ │  Reserved Instances (baseline)               │             │      │
│   │ └──────────────────────────────────────────────┘             │      │
│   │                                                               │      │
│   └───────────────────────────────────────────────────▶ Time     │      │
│                                                                          │
│   Strategy:                                                              │
│   - RI/Savings Plans: Predictable baseline (60-70%)                     │
│   - On-Demand: Buffer (10-20%)                                          │
│   - Spot: Fault-tolerant (10-30%)                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Savings Plans vs Reserved Instances:
┌───────────────────┬────────────────────┬────────────────────────────────┐
│ Feature           │ Savings Plans      │ Reserved Instances             │
├───────────────────┼────────────────────┼────────────────────────────────┤
│ Commitment        │ $/hour             │ Instance type                  │
│ Flexibility       │ Any instance/region│ Specific instance/AZ          │
│ Best For          │ Varying workloads  │ Predictable, specific needs   │
│ Discount          │ Up to 72%          │ Up to 72%                      │
└───────────────────┴────────────────────┴────────────────────────────────┘
```

### Lambda Optimization

```
Lambda Cost = Requests + Duration (GB-seconds)

Optimization Strategies:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. Right-size Memory                                                   │
│      - More memory = more CPU = faster execution                        │
│      - Sometimes higher memory is cheaper (less duration)               │
│      - Use AWS Lambda Power Tuning tool                                 │
│                                                                          │
│   2. Reduce Duration                                                     │
│      - Initialize outside handler (reuse)                               │
│      - Use smaller dependencies                                         │
│      - Optimize code                                                    │
│                                                                          │
│   3. Reduce Cold Starts                                                  │
│      - Provisioned Concurrency for predictable traffic                  │
│      - Keep functions warm (scheduled ping)                             │
│                                                                          │
│   4. Architecture                                                        │
│      - Batch processing (more items per invocation)                     │
│      - Direct service integrations (skip Lambda for simple ops)         │
│                                                                          │
│   Example:                                                               │
│   Before: 128MB, 5s duration = $0.00000834/invoke                       │
│   After:  256MB, 2s duration = $0.00000667/invoke (20% savings)        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Storage Cost Optimization

### S3 Optimization

```
S3 Storage Class Decision Tree:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   How often accessed?                                                    │
│       │                                                                  │
│       ├── Frequently ──────────▶ S3 Standard                            │
│       │                                                                  │
│       ├── Unknown/Variable ────▶ S3 Intelligent-Tiering                 │
│       │                                                                  │
│       ├── Monthly ─────────────▶ S3 Standard-IA                         │
│       │   (needs quick access)  (or One Zone-IA if non-critical)        │
│       │                                                                  │
│       └── Rarely/Archive ──────▶ Need instant access?                   │
│               │                      │                                   │
│               │                      ├── Yes ──▶ Glacier Instant         │
│               │                      │                                   │
│               │                      └── No ───▶ How soon needed?       │
│               │                              │                           │
│               │                              ├── Minutes ▶ Glacier Flex │
│               │                              └── Hours ──▶ Deep Archive │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Cost Savings Example (1 TB):
┌───────────────────┬───────────────┬───────────────────────────────────┐
│ Class             │ Monthly Cost  │ Savings vs Standard               │
├───────────────────┼───────────────┼───────────────────────────────────┤
│ Standard          │ $23.55        │ -                                 │
│ Standard-IA       │ $12.80        │ 46%                               │
│ One Zone-IA       │ $10.24        │ 57%                               │
│ Glacier Instant   │ $4.10         │ 83%                               │
│ Glacier Flexible  │ $3.69         │ 84%                               │
│ Deep Archive      │ $1.01         │ 96%                               │
└───────────────────┴───────────────┴───────────────────────────────────┘
```

### EBS Optimization

```
EBS Cost Optimization:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. Volume Type Selection                                               │
│      - gp3 is 20% cheaper than gp2 for same performance                │
│      - st1/sc1 for throughput/cold workloads                           │
│                                                                          │
│   2. Delete Unused Volumes                                               │
│      - Orphaned volumes from terminated instances                       │
│      - Use AWS Config rule: ec2-volume-inuse-check                     │
│                                                                          │
│   3. Snapshot Management                                                 │
│      - Delete old snapshots                                             │
│      - Use Data Lifecycle Manager                                       │
│      - Archive tier for long-term (75% cheaper)                        │
│                                                                          │
│   4. Right-size Volumes                                                  │
│      - Reduce over-provisioned volumes                                  │
│      - Monitor utilization with CloudWatch                              │
│                                                                          │
│   Cost Comparison (1 TB):                                                │
│   gp2: $100/month                                                       │
│   gp3: $80/month (3000 IOPS, 125 MB/s included)                        │
│   st1: $45/month                                                        │
│   sc1: $15/month                                                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Database Cost Optimization

### RDS Optimization

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. Right-size Instances                                                │
│      - Monitor CPU, memory, connections                                 │
│      - Use Performance Insights                                         │
│      - Consider Aurora Serverless v2 for variable workloads            │
│                                                                          │
│   2. Reserved Instances                                                  │
│      - 1-year: ~40% savings                                             │
│      - 3-year: ~60% savings                                             │
│                                                                          │
│   3. Storage Optimization                                                │
│      - gp3 instead of gp2                                               │
│      - Clean up unused storage                                          │
│      - Consider Aurora (auto-scaling storage)                           │
│                                                                          │
│   4. Multi-AZ vs Read Replicas                                          │
│      - Multi-AZ: 2x cost but HA                                         │
│      - Read Replicas: Scale reads, no Multi-AZ cost                    │
│                                                                          │
│   5. Development/Test                                                    │
│      - Smaller instance types                                           │
│      - Single-AZ                                                        │
│      - Stop when not in use                                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### DynamoDB Optimization

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. Capacity Mode Selection                                             │
│                                                                          │
│   On-Demand vs Provisioned:                                              │
│   - On-Demand: Unpredictable, variable traffic                         │
│   - Provisioned: Steady traffic (+ Auto Scaling)                       │
│   - Break-even: ~14% utilization                                        │
│                                                                          │
│   2. Reserved Capacity (Provisioned)                                     │
│      - Pay upfront for capacity units                                   │
│      - Up to 53% savings                                                │
│                                                                          │
│   3. Design Optimization                                                 │
│      - Efficient key design (avoid hot partitions)                     │
│      - Smaller items (less RCU/WCU)                                    │
│      - Use GSI sparingly (additional cost)                             │
│      - Eventually consistent reads (half the cost)                     │
│                                                                          │
│   4. TTL for Data Expiration                                            │
│      - Automatic, free deletion                                         │
│      - Reduces storage costs                                            │
│                                                                          │
│   Cost Example:                                                          │
│   On-Demand: $1.25/million writes, $0.25/million reads                 │
│   Provisioned: $0.00065/WCU/hour, $0.00013/RCU/hour                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Network Cost Optimization

```
Data Transfer Costs:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Free:                                                                  │
│   - Inbound from internet                                               │
│   - Same AZ (using private IP)                                          │
│   - To CloudFront origins                                               │
│   - VPC endpoints to S3/DynamoDB (gateway)                             │
│                                                                          │
│   Costs:                                                                 │
│   - Cross-AZ: $0.01/GB each way                                        │
│   - Cross-Region: $0.02/GB                                              │
│   - Internet Outbound: $0.09/GB (first 10TB)                           │
│   - NAT Gateway processing: $0.045/GB                                  │
│                                                                          │
│   Optimization Strategies:                                               │
│   1. Keep traffic in same AZ when possible                              │
│   2. Use VPC endpoints (Gateway = free, Interface = cheaper than NAT)  │
│   3. Use CloudFront (often cheaper than direct S3)                     │
│   4. Regional deployments (avoid cross-region)                          │
│   5. Compress data before transfer                                      │
│                                                                          │
│   CloudFront vs S3 Direct:                                               │
│   S3 to Internet: $0.09/GB                                              │
│   CloudFront to Internet: $0.085/GB (cheaper + caching)                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Cost Visibility Tools

```
AWS Cost Management Tools:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. Cost Explorer                                                       │
│      - Visualize and analyze costs                                      │
│      - Filter by service, account, tag                                  │
│      - Forecasting                                                      │
│                                                                          │
│   2. AWS Budgets                                                         │
│      - Set cost/usage budgets                                           │
│      - Alerts via SNS/Email                                             │
│      - Budget actions (stop instances)                                  │
│                                                                          │
│   3. Cost Allocation Tags                                                │
│      - Tag resources (project, environment, owner)                      │
│      - Track costs by business dimension                                │
│                                                                          │
│   4. AWS Cost and Usage Report (CUR)                                    │
│      - Most detailed cost data                                          │
│      - To S3 for Athena/QuickSight analysis                            │
│                                                                          │
│   5. Compute Optimizer                                                   │
│      - Right-sizing recommendations                                     │
│      - EC2, Lambda, EBS, ECS                                           │
│                                                                          │
│   6. Savings Plans/RI Recommendations                                   │
│      - Based on usage patterns                                          │
│      - In Cost Explorer                                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Interview Discussion Points

### How do you approach cost optimization?

```
Systematic Approach:

1. Visibility First
   - Enable Cost Allocation Tags
   - Set up Cost Explorer
   - Create dashboards by team/project

2. Quick Wins
   - Delete unused resources
   - Right-size over-provisioned
   - Stop dev/test during off-hours

3. Long-term Savings
   - Savings Plans for steady workloads
   - Reserved Instances for specific needs
   - Architecture optimization

4. Continuous Process
   - Regular cost reviews
   - Automated alerts on anomalies
   - FinOps culture (engineering owns costs)

5. Trade-offs
   - Cost vs performance
   - Cost vs availability
   - Cost vs operational complexity
```

### How do you balance cost with reliability?

```
Cost-Reliability Balance:

1. Tiered Approach
   - Production: Full HA, Multi-AZ
   - Staging: Single-AZ, smaller instances
   - Dev: Minimal, spot instances

2. Right Level of Redundancy
   - Not everything needs 99.99%
   - Match SLA to business need
   - Calculate cost of downtime

3. Smart Scaling
   - Scale down during off-peak
   - Use Spot for fault-tolerant components
   - Cache aggressively to reduce load

4. Disaster Recovery Options
   - Backup and restore: Cheapest, longest RTO
   - Pilot light: Moderate cost and RTO
   - Warm standby: Higher cost, lower RTO
   - Active-active: Highest cost, lowest RTO

Decision Matrix:
┌────────────────┬─────────────┬───────────┬───────────┐
│ Criticality    │ HA Level    │ RTO       │ Approach  │
├────────────────┼─────────────┼───────────┼───────────┤
│ Critical       │ Multi-Region│ Minutes   │ Active    │
│ High           │ Multi-AZ    │ <1 hour   │ Warm      │
│ Medium         │ Multi-AZ    │ Hours     │ Pilot     │
│ Low            │ Single-AZ   │ Days      │ Backup    │
└────────────────┴─────────────┴───────────┴───────────┘
```
