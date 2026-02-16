# IAM (Identity and Access Management)

## Core Concepts

```
IAM Components:
┌─────────────────────────────────────────────────────────────────────────┐
│                              IAM                                         │
├─────────────────┬─────────────────┬─────────────────┬───────────────────┤
│     Users       │     Groups      │     Roles       │     Policies      │
│  (People/Apps)  │ (Collection of  │ (Assumed by     │ (JSON permissions)│
│                 │     Users)      │  services/users)│                   │
└─────────────────┴─────────────────┴─────────────────┴───────────────────┘

Key Principle: LEAST PRIVILEGE
- Grant only permissions needed
- Start with zero permissions
- Add as needed
```

---

## IAM Users, Groups, and Roles

### Users

```
IAM User = Identity with long-term credentials

Credentials:
- Password (console access)
- Access Keys (programmatic access)
  - Access Key ID: AKIAIOSFODNN7EXAMPLE
  - Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

Best Practices:
- One user per person (no sharing)
- Use groups for permissions
- Rotate credentials regularly
- Enable MFA for all users
- Use roles instead of access keys when possible
```

### Groups

```
IAM Group = Collection of users with shared permissions

Example Structure:
┌─────────────────────────────────────────────────────────────────────────┐
│                              Groups                                      │
├─────────────────┬─────────────────┬─────────────────┬───────────────────┤
│   Developers    │     DevOps      │     Admins      │    ReadOnly       │
│                 │                 │                 │                   │
│ - EC2 access    │ - Full EC2      │ - Admin access  │ - Read-only       │
│ - S3 read       │ - S3 full       │ - IAM access    │   all services    │
│ - CloudWatch    │ - CloudWatch    │ - Billing       │                   │
└─────────────────┴─────────────────┴─────────────────┴───────────────────┘

Rules:
- Users can belong to multiple groups
- Groups cannot contain other groups
- Permissions are additive (union of all group policies)
```

### Roles

```
IAM Role = Identity assumed temporarily

Use Cases:
1. EC2 Instance Role      → EC2 accesses S3/DynamoDB
2. Lambda Execution Role  → Lambda accesses other services
3. Cross-Account Role     → Account A accesses Account B resources
4. SAML/OIDC Federation   → External IdP users access AWS
5. Service-Linked Role    → AWS service accesses your resources

Role Trust Policy (who can assume):
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "ec2.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

---

## IAM Policies

### Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadWrite",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "192.168.1.0/24"
        }
      }
    }
  ]
}
```

### Policy Elements

| Element | Description | Example |
|---------|-------------|---------|
| **Version** | Policy language version | "2012-10-17" |
| **Statement** | Container for permissions | Array of statements |
| **Sid** | Statement identifier | "AllowS3Read" |
| **Effect** | Allow or Deny | "Allow" |
| **Principal** | Who (for resource policies) | "arn:aws:iam::123:user/bob" |
| **Action** | What actions | "s3:GetObject" |
| **Resource** | Which resources | "arn:aws:s3:::bucket/*" |
| **Condition** | When to apply | IP, time, MFA, tags |

### Policy Types

```
1. Identity-based Policies (attached to users/groups/roles)
   ├── AWS Managed Policies (pre-built by AWS)
   ├── Customer Managed Policies (you create)
   └── Inline Policies (embedded in identity)

2. Resource-based Policies (attached to resources)
   ├── S3 Bucket Policies
   ├── SQS Queue Policies
   ├── KMS Key Policies
   └── Lambda Resource Policies

3. Permissions Boundaries (max permissions for identity)

4. Service Control Policies (SCPs - for Organizations)

5. Session Policies (for temporary credentials)
```

### Policy Evaluation Logic

```
                    ┌─────────────────┐
                    │ Explicit Deny?  │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
        ┌─────────┐                   ┌─────────┐
        │   Yes   │                   │   No    │
        └────┬────┘                   └────┬────┘
             │                             │
             ▼                             ▼
        ┌─────────┐                ┌──────────────┐
        │  DENY   │                │ SCP Allow?   │
        └─────────┘                │ (if in Org)  │
                                   └──────┬───────┘
                                          │
                            ┌─────────────┴─────────────┐
                            │                           │
                            ▼                           ▼
                      ┌─────────┐                 ┌─────────┐
                      │   No    │                 │   Yes   │
                      └────┬────┘                 └────┬────┘
                           │                          │
                           ▼                          ▼
                      ┌─────────┐           ┌────────────────┐
                      │  DENY   │           │ Identity-based │
                      └─────────┘           │   Allow?       │
                                            └───────┬────────┘
                                                    │
                                      ┌─────────────┴─────────────┐
                                      │                           │
                                      ▼                           ▼
                                ┌─────────┐                 ┌─────────┐
                                │   Yes   │                 │   No    │
                                └────┬────┘                 └────┬────┘
                                     │                          │
                                     ▼                          ▼
                                ┌─────────┐           ┌────────────────┐
                                │  ALLOW  │           │ Resource-based │
                                └─────────┘           │   Allow?       │
                                                      └───────┬────────┘
                                                              │
                                                ┌─────────────┴─────────┐
                                                ▼                       ▼
                                          ┌─────────┐             ┌─────────┐
                                          │   Yes   │             │   No    │
                                          └────┬────┘             └────┬────┘
                                               │                      │
                                               ▼                      ▼
                                          ┌─────────┐            ┌─────────┐
                                          │  ALLOW  │            │  DENY   │
                                          └─────────┘            └─────────┘

Summary: Explicit Deny > SCP > Identity Allow > Resource Allow > Default Deny
```

---

## Common Policy Patterns

### Full Admin Access

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "*",
    "Resource": "*"
  }]
}
```

### Read-Only Access to Specific S3 Bucket

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-bucket"
    }
  ]
}
```

### Require MFA for Sensitive Actions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:TerminateInstances",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

### Restrict by IP Address

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
      "NotIpAddress": {
        "aws:SourceIp": ["192.168.1.0/24", "10.0.0.0/8"]
      }
    }
  }]
}
```

### Cross-Account Access

```json
// Account A: Trust Policy on Role
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::ACCOUNT-B-ID:root"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "unique-external-id"
      }
    }
  }]
}

// Account B: Policy to allow assuming role
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::ACCOUNT-A-ID:role/CrossAccountRole"
  }]
}
```

### Tag-Based Access Control (ABAC)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ec2:StartInstances",
      "ec2:StopInstances"
    ],
    "Resource": "*",
    "Condition": {
      "StringEquals": {
        "ec2:ResourceTag/Environment": "${aws:PrincipalTag/Environment}"
      }
    }
  }]
}
```

---

## IAM Best Practices

### Security Best Practices

```
1. Root Account
   - Enable MFA immediately
   - Don't use for daily tasks
   - No access keys for root
   - Use only for account-level tasks

2. Users
   - Individual users (no sharing)
   - Strong password policy
   - MFA for all users
   - Regular access key rotation (90 days)

3. Permissions
   - Least privilege
   - Use groups for permission assignment
   - Regular permission audits
   - Use conditions (IP, MFA, time)

4. Roles
   - Use roles instead of access keys
   - EC2/Lambda should use roles
   - Cross-account via roles, not users

5. Monitoring
   - Enable CloudTrail
   - Use IAM Access Analyzer
   - Review unused permissions
```

### Organizational Patterns

```
AWS Organizations + SCPs:
┌─────────────────────────────────────────────────────────────────────────┐
│                           Management Account                             │
│                         (billing, organizations)                         │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Production    │    │   Development   │    │    Security     │
│       OU        │    │       OU        │    │       OU        │
│                 │    │                 │    │                 │
│ SCP: Strict     │    │ SCP: Relaxed    │    │ SCP: Audit      │
│ - No public S3  │    │ - Allow most    │    │ - Read-only     │
│ - Specific      │    │ - Cost limits   │    │ - CloudTrail    │
│   regions only  │    │                 │    │   access        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## Common Interview Questions

### How would you implement least privilege?

```
1. Start with zero permissions
2. Use IAM Access Analyzer to identify needed permissions
3. Analyze CloudTrail for actual API calls
4. Create narrow policies based on actual usage
5. Use conditions to further restrict (IP, time, MFA)
6. Regular reviews with IAM Access Advisor
7. Use permission boundaries for delegation
```

### How do you secure cross-account access?

```
1. Use IAM roles (never share access keys)
2. Require external ID for third-party access
3. Use resource-based policies where possible
4. Implement conditions (source account, source IP)
5. Enable CloudTrail in both accounts
6. Regular audit of cross-account roles
```

### Explain the difference between identity-based and resource-based policies

```
Identity-based:
- Attached to users, groups, roles
- Specifies what identity can do
- No Principal element (implied)
- Example: "This user can read S3 bucket X"

Resource-based:
- Attached to resources (S3, SQS, KMS, etc.)
- Specifies who can access resource
- Requires Principal element
- Example: "Account Y can access this bucket"
- Enables cross-account without role assumption (for some services)

Key Difference:
- Identity + Resource-based: Either can allow (OR logic)
- Cross-account: Both must allow (AND logic)
```

### How do you handle temporary credentials?

```
1. IAM Roles + STS
   - AssumeRole: Cross-account, same-account
   - AssumeRoleWithSAML: SAML federation
   - AssumeRoleWithWebIdentity: OIDC federation
   - GetFederationToken: Federated user tokens

2. Session duration
   - Default: 1 hour
   - Max: 12 hours (role), 36 hours (federation)
   - Configure per-role: MaxSessionDuration

3. Session policies
   - Further restrict permissions for session
   - Intersection of role + session policy
```
