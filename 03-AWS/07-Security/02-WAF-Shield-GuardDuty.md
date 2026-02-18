# AWS Security Services: WAF, Shield, GuardDuty, Security Hub

## Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AWS Security Services                                │
├─────────────────────┬─────────────────────┬─────────────────────────────────┤
│ Perimeter Defense   │ Threat Detection    │ Security Posture                │
├─────────────────────┼─────────────────────┼─────────────────────────────────┤
│ WAF                 │ GuardDuty           │ Security Hub                    │
│ - Web ACLs          │ - ML-based          │ - Aggregates findings           │
│ - Rule groups       │ - VPC Flow Logs     │ - Compliance checks             │
│ - Bot control       │ - DNS logs          │ - Automated response            │
│                     │ - CloudTrail        │                                 │
│ Shield              │                     │ Inspector                       │
│ - DDoS protection   │ Macie               │ - Vulnerability scanning        │
│ - Standard (free)   │ - S3 data discovery │ - EC2, Lambda, ECR              │
│ - Advanced ($$$)    │ - PII detection     │                                 │
└─────────────────────┴─────────────────────┴─────────────────────────────────┘
```

---

## AWS WAF (Web Application Firewall)

### Core Concepts

```
WAF Architecture:

Internet → CloudFront/ALB/API Gateway → WAF Web ACL → Application
                                            │
                                     ┌──────┴──────┐
                                     │   Rules     │
                                     ├─────────────┤
                                     │ - Allow     │
                                     │ - Block     │
                                     │ - Count     │
                                     │ - CAPTCHA   │
                                     │ - Challenge │
                                     └─────────────┘

Web ACL Components:
┌─────────────────────────────────────────────────────────────────┐
│ Web ACL                                                         │
├─────────────────────────────────────────────────────────────────┤
│ Rules (evaluated in priority order):                            │
│                                                                 │
│ Priority 0: Rate-based rule (Block if >2000 req/5min)          │
│ Priority 1: IP block list (known bad IPs)                      │
│ Priority 2: AWS Managed Rules - SQL injection                  │
│ Priority 3: AWS Managed Rules - XSS                            │
│ Priority 4: Custom rule - Block /admin from non-VPN            │
│ Priority 5: Bot Control managed rule group                     │
│                                                                 │
│ Default Action: ALLOW (if no rules match)                      │
└─────────────────────────────────────────────────────────────────┘
```

### Rule Types

```yaml
# 1. Regular Rules - Match conditions
RegularRule:
  - IP match (allow/block specific IPs)
  - Geo match (block/allow countries)
  - String match (URI, headers, body)
  - Regex match
  - Size constraints
  - SQLi detection
  - XSS detection

# 2. Rate-Based Rules - DDoS/brute force protection
RateBasedRule:
  Limit: 2000  # requests per 5 minutes
  AggregateKeyType: IP  # or FORWARDED_IP
  ScopeDownStatement:
    # Optional: only count requests matching condition
    ByteMatchStatement:
      FieldToMatch:
        UriPath: {}
      SearchString: "/api/login"

# 3. Rule Groups - Reusable collections
RuleGroup:
  Capacity: 100  # WCU (Web ACL Capacity Units)
  Rules:
    - SQLi detection
    - XSS detection
    - Custom business rules
```

### AWS Managed Rules

```
Managed Rule Groups (maintained by AWS):

┌────────────────────────────┬────────────────────────────────────────────┐
│ Rule Group                 │ Protection Against                          │
├────────────────────────────┼────────────────────────────────────────────┤
│ AWSManagedRulesCommonRuleSet │ OWASP Top 10, common exploits            │
│ AWSManagedRulesSQLiRuleSet   │ SQL injection patterns                   │
│ AWSManagedRulesKnownBadInputs│ Known malicious inputs                   │
│ AWSManagedRulesLinuxRuleSet  │ Linux-specific exploits (LFI, etc)       │
│ AWSManagedRulesUnixRuleSet   │ Unix command injection                   │
│ AWSManagedRulesWindowsRuleSet│ Windows-specific exploits (PowerShell)   │
│ AWSManagedRulesPHPRuleSet    │ PHP exploits                             │
│ AWSManagedRulesWordPressRuleSet│ WordPress vulnerabilities              │
│ AWSManagedRulesATPRuleSet    │ Account takeover prevention              │
│ AWSManagedRulesBotControl    │ Bot detection and management             │
│ AWSManagedRulesACFPRuleSet   │ Account creation fraud prevention        │
└────────────────────────────┴────────────────────────────────────────────┘

Usage (CloudFormation):
WebACL:
  Type: AWS::WAFv2::WebACL
  Properties:
    DefaultAction:
      Allow: {}
    Scope: REGIONAL  # or CLOUDFRONT
    Rules:
      - Name: AWSManagedRulesCommonRuleSet
        Priority: 0
        Statement:
          ManagedRuleGroupStatement:
            VendorName: AWS
            Name: AWSManagedRulesCommonRuleSet
            ExcludedRules:
              - Name: SizeRestrictions_BODY  # Override if needed
        OverrideAction:
          None: {}  # Use rule group action
        VisibilityConfig:
          CloudWatchMetricsEnabled: true
          MetricName: CommonRuleSetMetric
          SampledRequestsEnabled: true
```

### Custom Rules Examples

```yaml
# Block specific countries
GeoBlockRule:
  Name: BlockHighRiskCountries
  Priority: 1
  Statement:
    GeoMatchStatement:
      CountryCodes:
        - RU
        - CN
        - KP
  Action:
    Block: {}

# Rate limit login endpoint
LoginRateLimit:
  Name: LoginBruteForceProtection
  Priority: 2
  Statement:
    RateBasedStatement:
      Limit: 100
      AggregateKeyType: IP
      ScopeDownStatement:
        ByteMatchStatement:
          FieldToMatch:
            UriPath: {}
          PositionalConstraint: STARTS_WITH
          SearchString: "/api/auth/login"
          TextTransformations:
            - Type: LOWERCASE
              Priority: 0
  Action:
    Block: {}

# Allow only specific IPs to admin
AdminIPRestriction:
  Name: AdminAccessControl
  Priority: 3
  Statement:
    AndStatement:
      Statements:
        - ByteMatchStatement:
            FieldToMatch:
              UriPath: {}
            PositionalConstraint: STARTS_WITH
            SearchString: "/admin"
        - NotStatement:
            Statement:
              IPSetReferenceStatement:
                ARN: !Ref AllowedAdminIPSet
  Action:
    Block:
      CustomResponse:
        ResponseCode: 403
        CustomResponseBodyKey: access-denied-body

# Block requests with suspicious headers
SuspiciousHeaderBlock:
  Name: BlockSuspiciousUserAgent
  Priority: 4
  Statement:
    OrStatement:
      Statements:
        - ByteMatchStatement:
            FieldToMatch:
              SingleHeader:
                Name: user-agent
            PositionalConstraint: CONTAINS
            SearchString: "sqlmap"
        - ByteMatchStatement:
            FieldToMatch:
              SingleHeader:
                Name: user-agent
            PositionalConstraint: CONTAINS
            SearchString: "nikto"
  Action:
    Block: {}
```

### Bot Control

```
Bot Control Categories:

┌─────────────────────────────────────────────────────────────────┐
│ Bot Categories                                                   │
├─────────────────┬───────────────────────────────────────────────┤
│ Verified Bots   │ Googlebot, Bingbot (allow by default)        │
│ Common Bots     │ Known bot signatures                          │
│ Targeted Bots   │ Advanced protection (ATP)                     │
└─────────────────┴───────────────────────────────────────────────┘

Actions:
- Allow: Let through
- Block: Deny access
- Count: Log only (monitoring mode)
- CAPTCHA: Challenge with CAPTCHA
- Challenge: Silent browser challenge (JavaScript)

Configuration:
BotControlRule:
  Name: BotControl
  Statement:
    ManagedRuleGroupStatement:
      VendorName: AWS
      Name: AWSManagedRulesBotControlRuleSet
      ManagedRuleGroupConfigs:
        - AWSManagedRulesBotControlRuleSet:
            InspectionLevel: COMMON  # or TARGETED
      RuleActionOverrides:
        - Name: CategoryHttpLibrary
          ActionToUse:
            Count: {}  # Monitor instead of block
```

---

## AWS Shield

### Shield Standard vs Advanced

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Shield Standard vs Advanced                               │
├─────────────────────────┬───────────────────────────────────────────────────┤
│ Feature                 │ Standard (Free)      │ Advanced ($3000/mo)        │
├─────────────────────────┼──────────────────────┼────────────────────────────┤
│ Layer 3/4 Protection    │ ✓ Automatic          │ ✓ Enhanced                 │
│ Layer 7 Protection      │ ✗                    │ ✓ With WAF                 │
│ DRT Access              │ ✗                    │ ✓ 24/7 DDoS Response Team  │
│ Cost Protection         │ ✗                    │ ✓ Credits for scaling      │
│ Health-Based Detection  │ ✗                    │ ✓ Route 53 health checks   │
│ Real-time Metrics       │ Basic                │ ✓ Advanced visibility      │
│ Attack Forensics        │ ✗                    │ ✓ Post-attack analysis     │
│ WAF Included            │ ✗                    │ ✓ No additional charge     │
│ Firewall Manager        │ ✗                    │ ✓ No additional charge     │
└─────────────────────────┴──────────────────────┴────────────────────────────┘

Shield Advanced Protected Resources:
- CloudFront distributions
- Route 53 hosted zones
- Application Load Balancers
- Elastic IPs (for EC2, NLB)
- Global Accelerator accelerators
```

### Shield Advanced Architecture

```
DDoS Protection Architecture:

                    Attack Traffic
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Edge Locations                            │
│  ┌─────────────┐                                                │
│  │ Shield      │◄── Layer 3/4 filtering                        │
│  │ Standard    │    (SYN floods, UDP reflection)               │
│  └─────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CloudFront / ALB                              │
│  ┌─────────────┐  ┌─────────────┐                              │
│  │ Shield      │  │ WAF         │◄── Layer 7 filtering         │
│  │ Advanced    │  │ (Included)  │    (HTTP floods, slowloris)  │
│  └─────────────┘  └─────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
                    ┌───────────┐
                    │ Your App  │
                    └───────────┘

Shield Advanced Response:
1. Automatic detection and mitigation
2. DRT engagement for sophisticated attacks
3. WAF rule recommendations
4. Cost protection if scaling occurs
```

---

## Amazon GuardDuty

### How GuardDuty Works

```
GuardDuty Data Sources:

┌─────────────────────────────────────────────────────────────────────────────┐
│                         GuardDuty Analysis                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   VPC Flow Logs ─────┐                                                      │
│                      │                                                      │
│   DNS Logs ──────────┼────► ML Analysis ────► Findings ────► EventBridge   │
│                      │      Threat Intel       (Alerts)       Lambda        │
│   CloudTrail ────────┤      Anomaly Detection                Security Hub   │
│                      │                                                      │
│   S3 Data Events ────┤                                                      │
│                      │                                                      │
│   EKS Audit Logs ────┤                                                      │
│                      │                                                      │
│   RDS Login Events ──┘                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Finding Types (Examples):

┌─────────────────────────┬───────────────────────────────────────────────────┐
│ Category                │ Finding Examples                                   │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ EC2 Findings            │ - UnauthorizedAccess:EC2/SSHBruteForce            │
│                         │ - CryptoCurrency:EC2/BitcoinTool.B!DNS            │
│                         │ - Backdoor:EC2/DenialOfService.Dns                │
│                         │ - Trojan:EC2/BlackholeTraffic                     │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ IAM Findings            │ - UnauthorizedAccess:IAMUser/ConsoleLoginSuccess  │
│                         │ - Persistence:IAMUser/UserPermissions             │
│                         │ - CredentialAccess:IAMUser/AnomalousBehavior      │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ S3 Findings             │ - Policy:S3/BucketBlockPublicAccessDisabled       │
│                         │ - Exfiltration:S3/MaliciousIPCaller               │
│                         │ - Discovery:S3/MaliciousIPCaller                  │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ Kubernetes Findings     │ - PrivilegeEscalation:Kubernetes/Privileged       │
│                         │ - Execution:Kubernetes/ExecInKubeSystemPod        │
│                         │ - Impact:Kubernetes/MaliciousIPCaller             │
└─────────────────────────┴───────────────────────────────────────────────────┘
```

### GuardDuty Configuration

```bash
# Enable GuardDuty
aws guardduty create-detector --enable

# Enable additional protection plans
aws guardduty update-detector \
  --detector-id <detector-id> \
  --features '[
    {"Name": "S3_DATA_EVENTS", "Status": "ENABLED"},
    {"Name": "EKS_AUDIT_LOGS", "Status": "ENABLED"},
    {"Name": "RDS_LOGIN_EVENTS", "Status": "ENABLED"},
    {"Name": "RUNTIME_MONITORING", "Status": "ENABLED"}
  ]'

# Add trusted IP list (won't generate findings)
aws guardduty create-ip-set \
  --detector-id <detector-id> \
  --name "TrustedIPs" \
  --format TXT \
  --location s3://bucket/trusted-ips.txt \
  --activate

# Add threat intel list (generate findings for these)
aws guardduty create-threat-intel-set \
  --detector-id <detector-id> \
  --name "CustomThreatIntel" \
  --format TXT \
  --location s3://bucket/threat-ips.txt \
  --activate
```

### Automated Response Pattern

```python
# Lambda function triggered by GuardDuty finding
import boto3
import json

ec2 = boto3.client('ec2')
sns = boto3.client('sns')

def lambda_handler(event, context):
    finding = event['detail']
    finding_type = finding['type']
    severity = finding['severity']

    # High severity EC2 compromise - isolate instance
    if severity >= 7 and 'EC2' in finding_type:
        instance_id = finding['resource']['instanceDetails']['instanceId']

        # Create isolated security group
        response = ec2.create_security_group(
            GroupName=f'isolated-{instance_id}',
            Description='Isolated security group for compromised instance',
            VpcId=finding['resource']['instanceDetails']['networkInterfaces'][0]['vpcId']
        )
        isolated_sg = response['GroupId']

        # Remove all ingress/egress rules
        ec2.revoke_security_group_egress(
            GroupId=isolated_sg,
            IpPermissions=[{'IpProtocol': '-1', 'IpRanges': [{'CidrIp': '0.0.0.0/0'}]}]
        )

        # Attach isolated SG to instance
        ec2.modify_instance_attribute(
            InstanceId=instance_id,
            Groups=[isolated_sg]
        )

        # Notify security team
        sns.publish(
            TopicArn='arn:aws:sns:region:account:security-alerts',
            Subject=f'[CRITICAL] EC2 Instance Isolated: {instance_id}',
            Message=json.dumps(finding, indent=2)
        )

        return {'statusCode': 200, 'body': f'Isolated {instance_id}'}

    # S3 findings - log and alert
    if 'S3' in finding_type:
        bucket_name = finding['resource']['s3BucketDetails'][0]['name']
        sns.publish(
            TopicArn='arn:aws:sns:region:account:security-alerts',
            Subject=f'[HIGH] S3 Security Finding: {bucket_name}',
            Message=json.dumps(finding, indent=2)
        )

    return {'statusCode': 200, 'body': 'Finding processed'}
```

---

## AWS Security Hub

### Security Hub Architecture

```
Security Hub - Central Security Dashboard:

┌─────────────────────────────────────────────────────────────────────────────┐
│                           AWS Security Hub                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Findings Aggregation                          │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │                                                                     │   │
│  │   GuardDuty ──────┐                                                │   │
│  │   Inspector ──────┤                                                │   │
│  │   Macie ──────────┼────► ASFF Format ────► Unified Dashboard       │   │
│  │   IAM Analyzer ───┤     (AWS Security     Prioritized findings     │   │
│  │   Firewall Mgr ───┤      Finding Format)                          │   │
│  │   3rd Party ──────┘                                                │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     Compliance Standards                             │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │   • AWS Foundational Security Best Practices                        │   │
│  │   • CIS AWS Foundations Benchmark                                   │   │
│  │   • PCI DSS                                                         │   │
│  │   • SOC 2                                                           │   │
│  │   • NIST 800-53                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Multi-Account Architecture:

                    ┌─────────────────────┐
                    │ Security Hub Admin  │
                    │ (Delegated Admin)   │
                    └──────────┬──────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │ Account A   │     │ Account B   │     │ Account C   │
    │ (Member)    │     │ (Member)    │     │ (Member)    │
    │             │     │             │     │             │
    │ GuardDuty   │     │ GuardDuty   │     │ GuardDuty   │
    │ Inspector   │     │ Inspector   │     │ Inspector   │
    │ Macie       │     │ Macie       │     │ Macie       │
    └─────────────┘     └─────────────┘     └─────────────┘
           │                   │                   │
           └───────────────────┴───────────────────┘
                               │
                    Findings aggregated to admin
```

### Security Standards and Controls

```yaml
AWS Foundational Security Best Practices (Sample Controls):

IAM:
  IAM.1: IAM policies should not allow full "*" administrative privileges
  IAM.4: IAM root user access key should not exist
  IAM.6: Hardware MFA should be enabled for root user
  IAM.7: Password policies should have strong requirements

EC2:
  EC2.1: EBS snapshots should not be publicly restorable
  EC2.2: VPC default security group should restrict all traffic
  EC2.3: EBS volumes should be encrypted at rest
  EC2.18: Security groups should only allow unrestricted incoming
          traffic for authorized ports

S3:
  S3.1: S3 Block Public Access setting should be enabled
  S3.2: S3 buckets should prohibit public read access
  S3.4: S3 buckets should have server-side encryption enabled
  S3.5: S3 buckets should require SSL requests

RDS:
  RDS.1: RDS snapshots should be private
  RDS.2: RDS instances should prohibit public access
  RDS.3: RDS instances should have encryption at rest enabled
  RDS.6: Enhanced monitoring should be configured for RDS instances

# Enable standards
aws securityhub batch-enable-standards \
  --standards-subscription-requests '[
    {"StandardsArn": "arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.2.0"},
    {"StandardsArn": "arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0"}
  ]'
```

### Automated Remediation

```yaml
# CloudFormation - Auto-remediation for public S3 buckets
AutoRemediatePublicS3:
  Type: AWS::Events::Rule
  Properties:
    Name: auto-remediate-public-s3
    EventPattern:
      source:
        - aws.securityhub
      detail-type:
        - Security Hub Findings - Imported
      detail:
        findings:
          Compliance:
            Status:
              - FAILED
          ProductFields:
            StandardsControlArn:
              - prefix: "arn:aws:securityhub:::standards/aws-foundational-security-best-practices"
          Resources:
            Type:
              - AwsS3Bucket
    State: ENABLED
    Targets:
      - Id: RemediateLambda
        Arn: !GetAtt RemediationLambda.Arn

# Remediation Lambda
RemediationLambda:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: security-hub-auto-remediate
    Runtime: python3.11
    Handler: index.handler
    Code:
      ZipFile: |
        import boto3

        s3_control = boto3.client('s3control')
        securityhub = boto3.client('securityhub')

        def handler(event, context):
            for finding in event['detail']['findings']:
                if 'S3.1' in finding.get('GeneratorId', ''):
                    # Enable S3 Block Public Access
                    bucket_arn = finding['Resources'][0]['Id']
                    bucket_name = bucket_arn.split(':::')[-1]

                    s3 = boto3.client('s3')
                    s3.put_public_access_block(
                        Bucket=bucket_name,
                        PublicAccessBlockConfiguration={
                            'BlockPublicAcls': True,
                            'IgnorePublicAcls': True,
                            'BlockPublicPolicy': True,
                            'RestrictPublicBuckets': True
                        }
                    )

                    # Update finding status
                    securityhub.batch_update_findings(
                        FindingIdentifiers=[{
                            'Id': finding['Id'],
                            'ProductArn': finding['ProductArn']
                        }],
                        Workflow={'Status': 'RESOLVED'},
                        Note={
                            'Text': 'Auto-remediated: Enabled S3 Block Public Access',
                            'UpdatedBy': 'auto-remediation-lambda'
                        }
                    )

            return {'statusCode': 200}
```

---

## Amazon Inspector

### Inspector v2 Overview

```
Inspector Scanning:

┌─────────────────────────────────────────────────────────────────────────────┐
│                         Amazon Inspector v2                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Scan Targets:                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │    EC2      │  │   ECR       │  │   Lambda    │  │   Lambda    │       │
│  │  Instances  │  │   Images    │  │  Functions  │  │    Layers   │       │
│  │             │  │             │  │   (Code)    │  │             │       │
│  │ OS vuln     │  │ Container   │  │ Dependency  │  │ Dependency  │       │
│  │ Network     │  │ vulns       │  │ vulns       │  │ vulns       │       │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │
│                                                                             │
│  Vulnerability Sources:                                                     │
│  • CVE database                                                            │
│  • AWS security advisories                                                 │
│  • ALAS (Amazon Linux Security)                                            │
│  • NVD (National Vulnerability Database)                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Finding Severity:
Critical (9.0-10.0) → Immediate remediation required
High (7.0-8.9)      → Remediate within days
Medium (4.0-6.9)    → Remediate within weeks
Low (0.1-3.9)       → Remediate as time permits
Informational       → No immediate action needed
```

### Inspector Configuration

```bash
# Enable Inspector
aws inspector2 enable --resource-types EC2 ECR LAMBDA

# Check scanning status
aws inspector2 list-coverage \
  --filter-criteria '{"resourceType": [{"comparison": "EQUALS", "value": "AWS_EC2_INSTANCE"}]}'

# Get findings
aws inspector2 list-findings \
  --filter-criteria '{
    "severity": [{"comparison": "EQUALS", "value": "CRITICAL"}],
    "findingStatus": [{"comparison": "EQUALS", "value": "ACTIVE"}]
  }'

# Suppress findings (false positives)
aws inspector2 create-filter \
  --name "suppress-internal-apps" \
  --action SUPPRESS \
  --filter-criteria '{
    "resourceTags": [{"comparison": "EQUALS", "key": "Environment", "value": "development"}]
  }'
```

---

## Interview Discussion Points

### How would you design a security architecture for a web application?

```
Layered Security Approach:

1. Edge Layer (Perimeter)
   ├── CloudFront with WAF
   │   ├── OWASP managed rules
   │   ├── Rate limiting
   │   ├── Bot control
   │   └── Geo blocking
   └── Shield Advanced (if high-risk)

2. Network Layer
   ├── VPC with private subnets
   ├── Security groups (least privilege)
   ├── NACLs for subnet-level rules
   └── VPC endpoints for AWS services

3. Application Layer
   ├── ALB with WAF
   ├── HTTPS only (ACM certificates)
   ├── Security headers
   └── Input validation

4. Data Layer
   ├── RDS in private subnet
   ├── Encryption at rest (KMS)
   ├── Encryption in transit (SSL)
   └── Secrets Manager for credentials

5. Detection & Response
   ├── GuardDuty for threat detection
   ├── Security Hub for aggregation
   ├── Inspector for vulnerability scanning
   ├── CloudTrail for API logging
   └── EventBridge for automated response

6. Compliance & Governance
   ├── AWS Config rules
   ├── Security Hub standards
   ├── IAM policies (least privilege)
   └── Service Control Policies (SCPs)
```

### How do you handle a security incident in AWS?

```
Incident Response Framework:

1. Detection
   └── GuardDuty finding triggers EventBridge rule

2. Analysis
   ├── Review finding details
   ├── Check CloudTrail for API activity
   ├── Examine VPC Flow Logs
   └── Determine blast radius

3. Containment
   ├── Isolate affected resources
   │   ├── Quarantine EC2 with isolated SG
   │   ├── Disable compromised IAM keys
   │   └── Block IP addresses in WAF
   └── Preserve evidence (snapshots, logs)

4. Eradication
   ├── Remove malware/backdoors
   ├── Patch vulnerabilities
   └── Rotate all credentials

5. Recovery
   ├── Restore from clean backups
   ├── Re-enable services gradually
   └── Monitor closely for recurrence

6. Post-Incident
   ├── Root cause analysis
   ├── Update runbooks
   ├── Improve detection rules
   └── Security awareness training

Automation Example:
GuardDuty Finding → EventBridge → Step Functions
                                       │
    ┌──────────────────────────────────┴──────────────────────────────────┐
    │                                                                     │
    ▼                                  ▼                                  ▼
Isolate Instance              Notify Security Team              Create Forensic Copy
(Lambda)                      (SNS/Slack)                       (Lambda: snapshot)
```

### Multi-account security strategy?

```
Organization Security Architecture:

                    ┌─────────────────────────┐
                    │    Management Account   │
                    │    (Root, SCPs only)    │
                    └───────────┬─────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐     ┌───────────────────┐    ┌───────────────┐
│ Security OU   │     │ Workloads OU      │    │ Sandbox OU    │
├───────────────┤     ├───────────────────┤    ├───────────────┤
│ Log Archive   │     │ Production        │    │ Developer     │
│ - All logs    │     │ - Strict SCPs     │    │ accounts      │
│ - S3 buckets  │     │ - No public S3    │    │ - Limited     │
│ - Retention   │     │ - Approved AMIs   │    │   budget      │
│               │     │                   │    │               │
│ Security      │     │ Non-Production    │    │               │
│ Tooling       │     │ - Dev/Test        │    │               │
│ - GuardDuty   │     │ - More flexible   │    │               │
│   admin       │     │                   │    │               │
│ - Security    │     │                   │    │               │
│   Hub admin   │     │                   │    │               │
│ - Inspector   │     │                   │    │               │
└───────────────┘     └───────────────────┘    └───────────────┘

Key Patterns:
1. Delegated administration to Security account
2. SCPs prevent disabling security services
3. Centralized logging to Log Archive
4. Cross-account GuardDuty findings aggregation
5. Security Hub with organization-wide standards
6. AWS Config aggregator for compliance view
```
