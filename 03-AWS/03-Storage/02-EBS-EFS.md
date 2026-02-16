# EBS & EFS (Block and File Storage)

## EBS Overview

```
EBS = Elastic Block Store (Network-attached block storage)

Key Characteristics:
- Attached to single EC2 instance (mostly)
- Persist independently from instance
- AZ-scoped (cannot attach cross-AZ)
- Snapshots for backup/migration
- Encryption at rest

┌─────────────────────────────────────────────────────────────────────────┐
│                           Availability Zone                              │
│                                                                          │
│    ┌─────────────┐         ┌─────────────┐                              │
│    │     EC2     │◄────────│    EBS      │                              │
│    │  Instance   │ Network │   Volume    │                              │
│    └─────────────┘         └─────────────┘                              │
│                                                                          │
│    Note: Multi-Attach allows io1/io2 to attach to multiple instances    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## EBS Volume Types

### SSD-Backed Volumes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      General Purpose SSD                                 │
├─────────────────────┬───────────────────────────────────────────────────┤
│ gp3 (Recommended)   │ - 3,000 IOPS baseline (free)                      │
│                     │ - Up to 16,000 IOPS ($0.005/IOPS over 3000)       │
│                     │ - Up to 1,000 MB/s throughput                     │
│                     │ - IOPS/throughput independent of size             │
│                     │ - $0.08/GB-month                                  │
├─────────────────────┼───────────────────────────────────────────────────┤
│ gp2 (Legacy)        │ - 3 IOPS/GB (burst to 3,000)                      │
│                     │ - Max 16,000 IOPS (5,334 GB+)                     │
│                     │ - Throughput tied to IOPS                         │
│                     │ - $0.10/GB-month                                  │
└─────────────────────┴───────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                      Provisioned IOPS SSD                                │
├─────────────────────┬───────────────────────────────────────────────────┤
│ io2 Block Express   │ - Up to 256,000 IOPS                              │
│                     │ - Up to 4,000 MB/s throughput                     │
│                     │ - Sub-millisecond latency                         │
│                     │ - 99.999% durability                              │
│                     │ - Multi-Attach capable                            │
├─────────────────────┼───────────────────────────────────────────────────┤
│ io2                 │ - Up to 64,000 IOPS                               │
│                     │ - 500:1 IOPS-to-GB ratio                          │
│                     │ - 99.999% durability                              │
│                     │ - Multi-Attach capable                            │
├─────────────────────┼───────────────────────────────────────────────────┤
│ io1 (Legacy)        │ - Up to 64,000 IOPS                               │
│                     │ - 50:1 IOPS-to-GB ratio                           │
│                     │ - 99.9% durability                                │
└─────────────────────┴───────────────────────────────────────────────────┘
```

### HDD-Backed Volumes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Throughput Optimized HDD                            │
├─────────────────────┬───────────────────────────────────────────────────┤
│ st1                 │ - Max 500 IOPS                                    │
│                     │ - Max 500 MB/s throughput                         │
│                     │ - $0.045/GB-month                                 │
│                     │ - Cannot be boot volume                           │
│                     │ - Big data, data warehouses, log processing       │
└─────────────────────┴───────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         Cold HDD                                         │
├─────────────────────┬───────────────────────────────────────────────────┤
│ sc1                 │ - Max 250 IOPS                                    │
│                     │ - Max 250 MB/s throughput                         │
│                     │ - $0.015/GB-month                                 │
│                     │ - Cannot be boot volume                           │
│                     │ - Infrequent access, lowest cost                  │
└─────────────────────┴───────────────────────────────────────────────────┘
```

### Decision Tree

```
Volume Selection:
                         ┌─────────────────┐
                         │  Boot Volume?   │
                         └────────┬────────┘
                    ┌─────────────┴─────────────┐
                    ▼                           ▼
              ┌─────────┐                 ┌─────────┐
              │   Yes   │                 │   No    │
              └────┬────┘                 └────┬────┘
                   │                           │
                   ▼                           ▼
         ┌─────────────────┐         ┌─────────────────┐
         │  gp3 or io2     │         │ IOPS or         │
         │  (SSD required) │         │ Throughput?     │
         └─────────────────┘         └────────┬────────┘
                              ┌───────────────┴───────────────┐
                              ▼                               ▼
                        ┌─────────┐                     ┌─────────┐
                        │  IOPS   │                     │ Through-│
                        │  focus  │                     │   put   │
                        └────┬────┘                     └────┬────┘
                             │                               │
              ┌──────────────┴──────────────┐               │
              ▼                             ▼               ▼
      ┌──────────────┐            ┌──────────────┐   ┌──────────────┐
      │ < 16K IOPS?  │            │ > 16K IOPS?  │   │ st1 or sc1   │
      │    gp3       │            │ io2/io2 BE   │   │ (HDD cheap)  │
      └──────────────┘            └──────────────┘   └──────────────┘
```

---

## EBS Snapshots

### Snapshot Basics

```
Snapshots = Point-in-time backups stored in S3

Characteristics:
- Incremental (only changed blocks)
- Can create volumes from snapshots
- Can copy across regions
- Can share with other accounts

Snapshot Lifecycle:
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Snapshot 1 │    │  Snapshot 2 │    │  Snapshot 3 │
│  (Full)     │    │  (Δ only)   │    │  (Δ only)   │
│   100 GB    │    │   10 GB     │    │   5 GB      │
└─────────────┘    └─────────────┘    └─────────────┘

Deleting Snapshots:
- Safe to delete any snapshot
- AWS automatically consolidates data
- Only unreferenced blocks removed
```

### Snapshot Operations

```bash
# Create snapshot
aws ec2 create-snapshot \
  --volume-id vol-1234567890 \
  --description "Daily backup"

# Copy to another region
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-1234567890 \
  --destination-region eu-west-1 \
  --description "DR copy"

# Create volume from snapshot (can change AZ, type, size)
aws ec2 create-volume \
  --snapshot-id snap-1234567890 \
  --availability-zone us-east-1a \
  --volume-type gp3 \
  --size 200
```

### Snapshot Best Practices

```
1. Automation
   - Use Amazon Data Lifecycle Manager (DLM)
   - Define schedules and retention
   - Cross-region copy for DR

2. Consistency
   - Stop I/O or freeze filesystem before snapshot
   - Use application-consistent methods for databases
   - Or accept crash-consistent snapshots

3. Cost Management
   - Delete old snapshots (DLM retention)
   - Use archive tier for long-term (75% cheaper)
   - Monitor with Cost Explorer

// Data Lifecycle Manager Policy
{
  "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
  "ResourceTypes": ["VOLUME"],
  "TargetTags": [{"Key": "Backup", "Value": "true"}],
  "Schedules": [{
    "Name": "DailySnapshots",
    "CreateRule": {"Interval": 24, "IntervalUnit": "HOURS"},
    "RetainRule": {"Count": 7},
    "CrossRegionCopyRules": [{
      "TargetRegion": "eu-west-1",
      "Encrypted": true,
      "RetainRule": {"Interval": 1, "IntervalUnit": "MONTHS"}
    }]
  }]
}
```

---

## EBS Multi-Attach

```
Multi-Attach = Single io1/io2 volume attached to multiple instances

┌─────────────────────────────────────────────────────────────────────────┐
│                         Same AZ Only                                     │
│                                                                          │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐                       │
│  │   EC2-1   │    │   EC2-2   │    │   EC2-3   │                       │
│  └─────┬─────┘    └─────┬─────┘    └─────┬─────┘                       │
│        │                │                │                              │
│        └────────────────┼────────────────┘                              │
│                         │                                                │
│                  ┌──────┴──────┐                                        │
│                  │  io2 Volume │                                        │
│                  │ Multi-Attach│                                        │
│                  └─────────────┘                                        │
└─────────────────────────────────────────────────────────────────────────┘

Requirements:
- io1 or io2 volume types only
- Nitro-based instances
- Same AZ for all instances
- Cluster-aware filesystem (GFS2, OCFS2) or application handling

Use Cases:
- Clustered databases (Oracle RAC)
- High-availability applications
- Reduce storage costs for shared data
```

---

## EBS Encryption

```
EBS Encryption:
- AES-256 encryption
- Handled transparently by EC2
- Minimal latency impact
- Uses KMS keys

What's Encrypted:
✓ Data at rest on volume
✓ Data in transit (instance ↔ volume)
✓ All snapshots
✓ Volumes created from snapshots

Encryption Options:
1. Default encryption (account setting)
   - All new volumes encrypted automatically
   - Uses aws/ebs key or custom CMK

2. Per-volume encryption
   - Specify at creation time
   - Cannot change after creation

// Encrypt existing unencrypted volume
1. Create snapshot of unencrypted volume
2. Copy snapshot with encryption enabled
3. Create new volume from encrypted snapshot
4. Swap volumes (stop instance, detach, attach, start)
```

---

## EFS Overview

```
EFS = Elastic File System (Managed NFS)

Key Characteristics:
- Shared file system (NFS v4.1)
- Multi-AZ by default
- Automatic scaling (petabyte scale)
- Pay per GB stored
- POSIX-compliant

┌─────────────────────────────────────────────────────────────────────────┐
│                              Region                                      │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                         EFS File System                           │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│        │                      │                      │                   │
│        ▼                      ▼                      ▼                   │
│  ┌──────────┐          ┌──────────┐          ┌──────────┐              │
│  │ AZ-a     │          │ AZ-b     │          │ AZ-c     │              │
│  │          │          │          │          │          │              │
│  │ ┌──────┐ │          │ ┌──────┐ │          │ ┌──────┐ │              │
│  │ │Mount │ │          │ │Mount │ │          │ │Mount │ │              │
│  │ │Target│ │          │ │Target│ │          │ │Target│ │              │
│  │ └──┬───┘ │          │ └──┬───┘ │          │ └──┬───┘ │              │
│  │    │     │          │    │     │          │    │     │              │
│  │ ┌──┴───┐ │          │ ┌──┴───┐ │          │ ┌──┴───┐ │              │
│  │ │ EC2  │ │          │ │ EC2  │ │          │ │ EC2  │ │              │
│  │ │ ASG  │ │          │ │ ASG  │ │          │ │ ASG  │ │              │
│  │ └──────┘ │          │ └──────┘ │          │ └──────┘ │              │
│  └──────────┘          └──────────┘          └──────────┘              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## EFS Storage Classes & Modes

### Storage Classes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       EFS Storage Classes                                │
├─────────────────────┬───────────────────────────────────────────────────┤
│ Standard            │ - Frequently accessed data                        │
│                     │ - Multi-AZ redundancy                             │
│                     │ - $0.30/GB-month                                  │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Standard-IA         │ - Infrequently accessed                          │
│                     │ - Multi-AZ redundancy                             │
│                     │ - $0.025/GB-month + $0.01/GB access              │
├─────────────────────┼───────────────────────────────────────────────────┤
│ One Zone            │ - Single AZ                                       │
│                     │ - 47% cheaper than Standard                       │
│                     │ - $0.16/GB-month                                  │
├─────────────────────┼───────────────────────────────────────────────────┤
│ One Zone-IA         │ - Single AZ, infrequent access                   │
│                     │ - Cheapest option                                 │
│                     │ - $0.0133/GB-month + access fee                  │
└─────────────────────┴───────────────────────────────────────────────────┘

Lifecycle Management:
- Automatically move files to IA after N days (7-90)
- Move back to Standard on access
```

### Performance Modes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       EFS Performance Modes                              │
├─────────────────────┬───────────────────────────────────────────────────┤
│ General Purpose     │ - Latency-sensitive workloads                    │
│ (default)           │ - Web serving, CMS, home directories             │
│                     │ - Lower throughput ceiling                        │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Max I/O             │ - Higher throughput and IOPS                     │
│                     │ - Slightly higher latency                         │
│                     │ - Big data, media processing                      │
│                     │ - Highly parallel workloads                       │
└─────────────────────┴───────────────────────────────────────────────────┘

Throughput Modes:
┌─────────────────────┬───────────────────────────────────────────────────┐
│ Bursting            │ - Scales with storage size                        │
│                     │ - 100 MiB/s per TB stored                         │
│                     │ - Burst credits for spikes                        │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Elastic             │ - Automatically scales throughput                 │
│ (recommended)       │ - Up to 3+ GiB/s reads, 1+ GiB/s writes          │
│                     │ - Pay for what you use                            │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Provisioned         │ - Specify throughput independent of storage      │
│                     │ - 1-1024+ MiB/s                                   │
│                     │ - Predictable performance                         │
└─────────────────────┴───────────────────────────────────────────────────┘
```

---

## EFS Access & Security

### Mount Targets

```bash
# Mount EFS on EC2 (Amazon Linux)
sudo yum install -y amazon-efs-utils

# Mount with encryption in transit
sudo mount -t efs -o tls fs-12345678:/ /mnt/efs

# fstab entry for persistent mount
fs-12345678:/ /mnt/efs efs _netdev,tls,iam 0 0

# Access Points (specific directory with POSIX permissions)
sudo mount -t efs -o tls,accesspoint=fsap-12345678 fs-12345678:/ /mnt/efs
```

### Security

```
EFS Security Layers:
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. Security Groups                                                       │
│    - Control network access to mount targets                            │
│    - Allow NFS (port 2049) from EC2 security groups                     │
├─────────────────────────────────────────────────────────────────────────┤
│ 2. IAM Policies                                                          │
│    - Control who can mount and what actions                             │
│    - elasticfilesystem:ClientMount, ClientWrite, ClientRootAccess       │
├─────────────────────────────────────────────────────────────────────────┤
│ 3. EFS File System Policies                                              │
│    - Resource-based policy on file system                               │
│    - Enforce encryption in transit                                      │
│    - Restrict root access                                               │
├─────────────────────────────────────────────────────────────────────────┤
│ 4. Access Points                                                         │
│    - Application-specific entry points                                  │
│    - Enforce user/group ID                                              │
│    - Enforce root directory                                             │
├─────────────────────────────────────────────────────────────────────────┤
│ 5. POSIX Permissions                                                     │
│    - Standard Linux file permissions                                    │
│    - User/group ownership                                               │
└─────────────────────────────────────────────────────────────────────────┘

// EFS File System Policy (enforce encryption)
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "*",
    "Condition": {
      "Bool": {
        "aws:SecureTransport": "false"
      }
    }
  }]
}
```

### Access Points

```
Access Points = Application-specific entry points

Use Cases:
- Multi-tenant applications
- Container workloads
- Enforcing POSIX permissions

Configuration:
┌─────────────────────────────────────────────────────────────────────────┐
│  Access Point: app-data                                                  │
│  ├── Root Directory: /app1/data                                         │
│  ├── POSIX User: UID 1000, GID 1000                                     │
│  └── Permissions: 755                                                   │
└─────────────────────────────────────────────────────────────────────────┘

// When app mounts via access point, it:
// - Sees /app1/data as root (/)
// - All operations run as UID/GID 1000
// - Cannot access other directories
```

---

## EBS vs EFS vs Instance Store

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Storage Comparison                                    │
├─────────────────┬────────────────┬────────────────┬─────────────────────┤
│ Feature         │ EBS            │ EFS            │ Instance Store      │
├─────────────────┼────────────────┼────────────────┼─────────────────────┤
│ Type            │ Block          │ File (NFS)     │ Block (local)       │
├─────────────────┼────────────────┼────────────────┼─────────────────────┤
│ Attach to       │ Single EC2*    │ Multiple EC2   │ Single EC2          │
├─────────────────┼────────────────┼────────────────┼─────────────────────┤
│ Persistence     │ Independent    │ Independent    │ Ephemeral           │
├─────────────────┼────────────────┼────────────────┼─────────────────────┤
│ AZ Scope        │ Single AZ      │ Multi-AZ       │ Single AZ           │
├─────────────────┼────────────────┼────────────────┼─────────────────────┤
│ Max IOPS        │ 256K (io2 BE)  │ 500K+          │ Millions            │
├─────────────────┼────────────────┼────────────────┼─────────────────────┤
│ Latency         │ <1ms           │ Low ms         │ ~100μs              │
├─────────────────┼────────────────┼────────────────┼─────────────────────┤
│ Max Size        │ 64 TB          │ Petabytes      │ Instance dependent  │
├─────────────────┼────────────────┼────────────────┼─────────────────────┤
│ Encryption      │ KMS            │ KMS            │ Instance dependent  │
├─────────────────┼────────────────┼────────────────┼─────────────────────┤
│ Pricing         │ Per GB + IOPS  │ Per GB used    │ Included            │
├─────────────────┼────────────────┼────────────────┼─────────────────────┤
│ Use Case        │ Databases,     │ Shared files,  │ Temp data, cache,   │
│                 │ boot volumes   │ web content    │ high-perf compute   │
└─────────────────┴────────────────┴────────────────┴─────────────────────┘
* io1/io2 supports Multi-Attach (up to 16 instances)
```

---

## FSx Family

```
FSx = Managed file systems for specific workloads

┌─────────────────────────────────────────────────────────────────────────┐
│                         FSx Options                                      │
├─────────────────────┬───────────────────────────────────────────────────┤
│ FSx for Windows     │ - Native Windows file system (SMB)               │
│                     │ - Active Directory integration                    │
│                     │ - DFS namespaces, shadow copies                   │
│                     │ - Use: Windows workloads, SharePoint, SQL Server  │
├─────────────────────┼───────────────────────────────────────────────────┤
│ FSx for Lustre      │ - High-performance parallel file system          │
│                     │ - Sub-millisecond latency                         │
│                     │ - Hundreds of GB/s throughput                     │
│                     │ - S3 integration (hot data tier)                  │
│                     │ - Use: HPC, ML training, video processing         │
├─────────────────────┼───────────────────────────────────────────────────┤
│ FSx for NetApp      │ - Full NetApp ONTAP features                     │
│ ONTAP               │ - NFS, SMB, iSCSI                                │
│                     │ - Snapshots, cloning, replication                 │
│                     │ - Use: Enterprise workloads, SAP, Oracle          │
├─────────────────────┼───────────────────────────────────────────────────┤
│ FSx for OpenZFS     │ - ZFS file system                                │
│                     │ - Up to 1M IOPS, <0.5ms latency                  │
│                     │ - Snapshots, clones, compression                  │
│                     │ - Use: Linux workloads, databases, analytics      │
└─────────────────────┴───────────────────────────────────────────────────┘
```

---

## Interview Discussion Points

### How do you choose between EBS volume types?

```
Decision Framework:

1. Determine primary need: IOPS vs Throughput

2. For IOPS-sensitive (databases):
   - < 16K IOPS: gp3 (adjust IOPS)
   - 16K-64K IOPS: io2
   - > 64K IOPS: io2 Block Express
   - Multi-instance: io2 with Multi-Attach

3. For throughput-sensitive (big data, logs):
   - Large sequential reads: st1 (cheap)
   - Archival: sc1 (cheapest)
   - Need SSD: gp3 (up to 1000 MB/s)

4. Cost optimization:
   - gp3 vs gp2: gp3 is 20% cheaper
   - Right-size provisioned IOPS
   - Use st1/sc1 for cold data
```

### How do you design shared storage for EC2 fleet?

```
Options by use case:

1. Web content/media (many readers):
   → EFS (simple, scales automatically)
   → Or S3 + CloudFront (if serving to internet)

2. Configuration/shared state:
   → EFS (POSIX compliant)
   → Or SSM Parameter Store (small config)

3. Database cluster (Oracle RAC, etc.):
   → EBS Multi-Attach (io2)
   → Requires cluster-aware filesystem

4. HPC/ML training:
   → FSx for Lustre (extreme throughput)
   → S3 as data lake, Lustre as hot tier

5. Windows workloads:
   → FSx for Windows (native SMB)
   → Active Directory integration
```

### How do you handle EBS backups for DR?

```
Multi-Region DR Strategy:

1. Automated Snapshots
   - Data Lifecycle Manager (DLM)
   - Cross-region copy rules
   - Encryption with multi-region KMS key

2. Recovery Process
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Primary Region         │        DR Region                          │
   │                        │                                           │
   │ EBS Volume ──snapshot──│──copy──→ Snapshot ──create──→ EBS Volume │
   │                        │         (encrypted)           (attach)    │
   └─────────────────────────────────────────────────────────────────────┘

3. RPO/RTO Considerations
   - Snapshot frequency = RPO (minutes to hours)
   - Cross-region copy = additional delay
   - Volume creation from snapshot = minutes
   - Fast Snapshot Restore (FSR) for instant volumes

4. Testing
   - Regular DR drills
   - Automate with CloudFormation/Terraform
   - Validate data integrity
```

### When to use EFS vs S3?

```
Use EFS when:
- Need POSIX filesystem semantics
- Multiple EC2 instances need concurrent access
- Application expects file paths (/mnt/data/file.txt)
- Need file locking
- Latency < 10ms required

Use S3 when:
- Object storage is acceptable (PUT/GET)
- Web content delivery (CloudFront integration)
- Data lake / analytics
- Unlimited scale needed
- Cost is primary concern
- Durability is critical (11 9s)

Hybrid pattern:
- S3 for permanent storage
- EFS for working data
- Sync between them as needed
- Or FSx Lustre with S3 backend
```
