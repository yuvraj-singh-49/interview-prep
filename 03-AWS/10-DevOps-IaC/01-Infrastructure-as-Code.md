# Infrastructure as Code (IaC)

## Overview

```
IaC = Managing infrastructure through code instead of manual processes

┌─────────────────────────────────────────────────────────────────────────┐
│                     IaC Options on AWS                                   │
├─────────────────────┬─────────────────┬─────────────────────────────────┤
│ CloudFormation      │ AWS CDK         │ Terraform                       │
│ (Native AWS)        │ (AWS, Code)     │ (Multi-cloud)                   │
│                     │                 │                                 │
│ - YAML/JSON         │ - TypeScript    │ - HCL                          │
│ - Direct AWS        │ - Python        │ - Provider model               │
│   integration       │ - Java, etc.    │ - State management             │
│ - StackSets for     │ - Synthesizes   │ - Rich ecosystem               │
│   multi-account     │   to CFN        │                                │
└─────────────────────┴─────────────────┴─────────────────────────────────┘

Benefits:
- Version control for infrastructure
- Repeatable deployments
- Consistency across environments
- Self-documenting infrastructure
- Review changes before applying
```

---

## CloudFormation

### Template Structure

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Three-tier web application

# Input parameters
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev
  InstanceType:
    Type: String
    Default: t3.micro

# Conditions for environment-specific config
Conditions:
  IsProd: !Equals [!Ref Environment, prod]

# Mappings for region-specific values
Mappings:
  RegionAMI:
    us-east-1:
      HVM64: ami-0123456789abcdef0
    eu-west-1:
      HVM64: ami-0987654321fedcba0

# Resource definitions
Resources:
  # VPC
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub ${Environment}-vpc

  # Public Subnet
  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true

  # EC2 Instance
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionAMI, !Ref 'AWS::Region', HVM64]
      InstanceType: !If [IsProd, m5.large, !Ref InstanceType]
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref WebSecurityGroup
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd

  # Security Group
  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server security group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

# Output values
Outputs:
  WebServerIP:
    Description: Public IP of web server
    Value: !GetAtt WebServer.PublicIp
    Export:
      Name: !Sub ${Environment}-WebServerIP
```

### Intrinsic Functions

```yaml
# Reference another resource
!Ref ResourceName
!Ref ParameterName

# Get attribute from resource
!GetAtt Resource.Attribute
!GetAtt EC2Instance.PublicIp

# Substitute variables in string
!Sub 'arn:aws:s3:::${BucketName}/*'
!Sub
  - 'https://${Domain}/api'
  - Domain: !Ref DomainName

# Join strings
!Join ['-', [!Ref Environment, 'bucket', !Ref AWS::AccountId]]

# Conditional
!If [ConditionName, TrueValue, FalseValue]

# Select from list
!Select [0, !GetAZs '']  # First AZ in region

# Split string
!Split [',', 'a,b,c']

# Import from another stack
!ImportValue ExportedValueName

# Base64 encode
!Base64 !Sub |
  #!/bin/bash
  echo "Hello ${Environment}"
```

### Stack Operations

```bash
# Create stack
aws cloudformation create-stack \
  --stack-name my-app-prod \
  --template-body file://template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=prod \
  --capabilities CAPABILITY_IAM

# Update stack (direct)
aws cloudformation update-stack \
  --stack-name my-app-prod \
  --template-body file://template.yaml

# Create change set (review before applying)
aws cloudformation create-change-set \
  --stack-name my-app-prod \
  --change-set-name update-instance-type \
  --template-body file://template.yaml

# Execute change set
aws cloudformation execute-change-set \
  --stack-name my-app-prod \
  --change-set-name update-instance-type

# Delete stack
aws cloudformation delete-stack --stack-name my-app-prod

# Detect drift
aws cloudformation detect-stack-drift --stack-name my-app-prod
```

### Nested Stacks

```yaml
# Parent template
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/bucket/network.yaml
      Parameters:
        Environment: !Ref Environment

  DatabaseStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/bucket/database.yaml
      Parameters:
        VPCId: !GetAtt NetworkStack.Outputs.VPCId
        SubnetIds: !GetAtt NetworkStack.Outputs.PrivateSubnetIds

  ApplicationStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: DatabaseStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/bucket/application.yaml
      Parameters:
        DatabaseEndpoint: !GetAtt DatabaseStack.Outputs.Endpoint

# Benefits:
# - Reusable components
# - Overcome 500 resource limit
# - Separate concerns
# - Independent updates
```

### StackSets (Multi-Account/Region)

```yaml
# Deploy to multiple accounts/regions
aws cloudformation create-stack-set \
  --stack-set-name security-baseline \
  --template-body file://security.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false

# Add stack instances
aws cloudformation create-stack-instances \
  --stack-set-name security-baseline \
  --deployment-targets OrganizationalUnitIds=ou-xxxx \
  --regions us-east-1 eu-west-1 ap-southeast-1

# Use cases:
# - Security baselines across all accounts
# - Compliance controls
# - Logging infrastructure
# - IAM roles for cross-account access
```

---

## AWS CDK

### Project Structure

```
my-cdk-app/
├── bin/
│   └── app.ts           # Entry point, defines stacks
├── lib/
│   ├── network-stack.ts # VPC, subnets
│   ├── database-stack.ts
│   └── app-stack.ts
├── test/
│   └── app.test.ts      # Unit tests
├── cdk.json             # CDK configuration
├── package.json
└── tsconfig.json
```

### CDK Constructs

```typescript
// lib/app-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import { Construct } from 'constructs';

export class AppStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // L2 Construct - VPC with sensible defaults
    const vpc = new ec2.Vpc(this, 'AppVpc', {
      maxAzs: 3,
      natGateways: 1,
      subnetConfiguration: [
        { name: 'Public', subnetType: ec2.SubnetType.PUBLIC },
        { name: 'Private', subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
        { name: 'Isolated', subnetType: ec2.SubnetType.PRIVATE_ISOLATED }
      ]
    });

    // L2 Construct - RDS with best practices built-in
    const database = new rds.DatabaseInstance(this, 'Database', {
      engine: rds.DatabaseInstanceEngine.postgres({
        version: rds.PostgresEngineVersion.VER_15
      }),
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
      instanceType: ec2.InstanceType.of(
        ec2.InstanceClass.T3,
        ec2.InstanceSize.MICRO
      ),
      multiAz: true,
      allocatedStorage: 20,
      storageEncrypted: true,
      deletionProtection: true,
      backupRetention: cdk.Duration.days(7)
    });

    // L2 Construct - ECS Cluster
    const cluster = new ecs.Cluster(this, 'Cluster', {
      vpc,
      containerInsights: true
    });

    // L3 Construct - Fargate Service with ALB (Pattern)
    const service = new ecs_patterns.ApplicationLoadBalancedFargateService(
      this, 'Service', {
        cluster,
        taskImageOptions: {
          image: ecs.ContainerImage.fromRegistry('nginx'),
          environment: {
            DATABASE_HOST: database.dbInstanceEndpointAddress
          },
          secrets: {
            DATABASE_PASSWORD: ecs.Secret.fromSecretsManager(
              database.secret!, 'password'
            )
          }
        },
        desiredCount: 2,
        publicLoadBalancer: true
      }
    );

    // Allow service to connect to database
    database.connections.allowFrom(service.service, ec2.Port.tcp(5432));

    // Output
    new cdk.CfnOutput(this, 'LoadBalancerDNS', {
      value: service.loadBalancer.loadBalancerDnsName
    });
  }
}
```

### CDK Construct Levels

```
L1 (CFN Resources):
- Direct CloudFormation mapping
- CfnBucket, CfnInstance, CfnVPC
- Full control, verbose

L2 (AWS Constructs):
- Higher-level abstractions
- Bucket, Instance, Vpc
- Sensible defaults, less code
- Security best practices built-in

L3 (Patterns):
- Complete architectures
- ApplicationLoadBalancedFargateService
- LambdaRestApi
- Multiple resources combined

// Example of all three levels
// L1 - Low level
new s3.CfnBucket(this, 'L1Bucket', {
  bucketName: 'my-l1-bucket',
  versioningConfiguration: { status: 'Enabled' }
});

// L2 - Higher abstraction
new s3.Bucket(this, 'L2Bucket', {
  versioned: true,
  encryption: s3.BucketEncryption.S3_MANAGED,
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL
});

// L3 - Pattern
new s3deploy.BucketDeployment(this, 'DeployWebsite', {
  sources: [s3deploy.Source.asset('./website')],
  destinationBucket: websiteBucket,
  distribution: cloudFrontDistribution
});
```

### CDK Commands

```bash
# Initialize new project
cdk init app --language typescript

# Synthesize CloudFormation template
cdk synth

# Show diff between deployed and local
cdk diff

# Deploy stack
cdk deploy

# Deploy all stacks
cdk deploy --all

# Deploy with approval prompts disabled
cdk deploy --require-approval never

# Destroy stack
cdk destroy

# Bootstrap environment (first time)
cdk bootstrap aws://ACCOUNT-ID/REGION
```

### CDK Testing

```typescript
// test/app.test.ts
import * as cdk from 'aws-cdk-lib';
import { Template, Match } from 'aws-cdk-lib/assertions';
import { AppStack } from '../lib/app-stack';

describe('AppStack', () => {
  const app = new cdk.App();
  const stack = new AppStack(app, 'TestStack');
  const template = Template.fromStack(stack);

  test('VPC Created with correct CIDR', () => {
    template.hasResourceProperties('AWS::EC2::VPC', {
      CidrBlock: '10.0.0.0/16'
    });
  });

  test('RDS Instance is Multi-AZ', () => {
    template.hasResourceProperties('AWS::RDS::DBInstance', {
      MultiAZ: true,
      StorageEncrypted: true
    });
  });

  test('Has expected number of subnets', () => {
    template.resourceCountIs('AWS::EC2::Subnet', 6);  // 2 per AZ
  });

  test('Security Group allows only port 443', () => {
    template.hasResourceProperties('AWS::EC2::SecurityGroup', {
      SecurityGroupIngress: Match.arrayWith([
        Match.objectLike({
          IpProtocol: 'tcp',
          FromPort: 443,
          ToPort: 443
        })
      ])
    });
  });
});
```

---

## IaC Best Practices

### Environment Management

```
Strategy: Separate stacks per environment

my-app-dev     ←── Dev parameters
my-app-staging ←── Staging parameters
my-app-prod    ←── Prod parameters

// CDK approach
const app = new cdk.App();

new AppStack(app, 'AppStack-Dev', {
  env: { account: '111111111111', region: 'us-east-1' },
  environment: 'dev',
  instanceType: 't3.micro'
});

new AppStack(app, 'AppStack-Prod', {
  env: { account: '222222222222', region: 'us-east-1' },
  environment: 'prod',
  instanceType: 'm5.large'
});
```

### Secrets Management

```typescript
// NEVER hardcode secrets in IaC

// Bad
const password = 'mysecretpassword';

// Good - Reference Secrets Manager
const secret = secretsmanager.Secret.fromSecretNameV2(
  this, 'DBSecret', 'prod/database/credentials'
);

// Good - Generate in template
const generatedSecret = new secretsmanager.Secret(this, 'Secret', {
  generateSecretString: {
    secretStringTemplate: JSON.stringify({ username: 'admin' }),
    generateStringKey: 'password',
    excludePunctuation: true
  }
});

// Reference in resources
new rds.DatabaseInstance(this, 'DB', {
  credentials: rds.Credentials.fromSecret(generatedSecret)
});
```

### Drift Detection

```bash
# CloudFormation drift detection
aws cloudformation detect-stack-drift --stack-name my-stack
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id xxx

# Common causes:
# - Manual console changes
# - CLI/SDK modifications
# - Auto Scaling changes
# - Other automation

# Prevention:
# - Restrict console access in prod
# - Use IaC for ALL changes
# - Regular drift checks in CI/CD
# - Alert on drift detected
```

---

## Interview Discussion Points

### How do you structure IaC for a multi-environment deployment?

```
Architecture:

1. Repository Structure
   infrastructure/
   ├── modules/           # Reusable components
   │   ├── vpc/
   │   ├── database/
   │   └── compute/
   ├── environments/
   │   ├── dev.tfvars
   │   ├── staging.tfvars
   │   └── prod.tfvars
   └── main.tf

2. Key Principles
   - Same code, different parameters
   - Promote artifacts between environments
   - Lock versions (modules, providers)
   - State isolation per environment

3. CI/CD Integration
   - PR triggers plan/diff
   - Merge triggers apply to dev
   - Manual approval for prod
   - Drift detection scheduled
```

### CloudFormation vs CDK vs Terraform?

```
Choose CloudFormation when:
- AWS-only environment
- Team familiar with YAML
- Need StackSets for multi-account
- Tight AWS service integration

Choose CDK when:
- Complex logic needed
- Prefer programming languages
- Want testing capabilities
- Team has development background
- Building reusable patterns

Choose Terraform when:
- Multi-cloud environment
- Strong state management needed
- Rich provider ecosystem
- Team knows HCL
- Need import existing resources

Hybrid approach common:
- Terraform for multi-cloud base
- CDK for complex AWS applications
- CloudFormation for AWS-specific features
```

### How do you handle IaC in a team?

```
Team Practices:

1. Code Review
   - All IaC changes through PR
   - Review for security, cost, best practices
   - Require approval from platform team

2. State Management
   - Remote state (S3 + DynamoDB locking)
   - State per environment
   - Never commit state files

3. CI/CD Pipeline
   - Lint and validate on PR
   - Plan output in PR comments
   - Auto-apply to lower environments
   - Manual approval for production

4. Documentation
   - README per module
   - Architecture diagrams
   - Runbooks for common operations

5. Versioning
   - Semantic versioning for modules
   - Lock provider versions
   - Track breaking changes
```
