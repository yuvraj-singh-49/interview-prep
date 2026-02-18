# CI/CD Pipeline on AWS

## AWS Developer Tools Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     AWS CI/CD Services                                   │
├─────────────────┬─────────────────┬─────────────────┬───────────────────┤
│  CodeCommit     │   CodeBuild     │   CodeDeploy    │   CodePipeline    │
│  (Source)       │   (Build)       │   (Deploy)      │   (Orchestrate)   │
│                 │                 │                 │                   │
│  Git repository │  Build & test   │  Deploy to      │  End-to-end       │
│  Managed by AWS │  Compile code   │  EC2, ECS,      │  workflow         │
│  IAM integrated │  Run tests      │  Lambda, On-prem│  Stages & actions │
└─────────────────┴─────────────────┴─────────────────┴───────────────────┘
```

---

## CodeBuild

### Build Project Configuration

```yaml
# buildspec.yml
version: 0.2

env:
  variables:
    NODE_ENV: production
  parameter-store:
    DB_PASSWORD: /app/prod/db-password
  secrets-manager:
    API_KEY: prod/api-key:api_key

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - npm ci

  pre_build:
    commands:
      - echo "Running pre-build steps"
      - npm run lint
      - npm run test:unit

  build:
    commands:
      - echo "Building application"
      - npm run build
      - echo "Building Docker image"
      - docker build -t $ECR_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION .

  post_build:
    commands:
      - echo "Pushing to ECR"
      - docker push $ECR_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo "Writing image definitions"
      - printf '[{"name":"app","imageUri":"%s"}]' $ECR_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yml
    - taskdef.json
  discard-paths: yes

cache:
  paths:
    - 'node_modules/**/*'

reports:
  jest-reports:
    files:
      - 'coverage/clover.xml'
    file-format: CLOVERXML
```

### Build Environment

```
Build Environments:
┌─────────────────────────────────────────────────────────────────────────┐
│ Compute Type        │ Memory │ vCPU │ Use Case                         │
├─────────────────────┼────────┼──────┼──────────────────────────────────┤
│ BUILD_GENERAL1_SMALL│ 3 GB   │ 2    │ Small projects, quick builds     │
│ BUILD_GENERAL1_MEDIUM│7 GB   │ 4    │ Standard builds                  │
│ BUILD_GENERAL1_LARGE│ 15 GB  │ 8    │ Large projects, parallel tests   │
│ BUILD_GENERAL1_2XLARGE│145 GB│ 72   │ Memory-intensive builds          │
└─────────────────────┴────────┴──────┴──────────────────────────────────┘

Docker Support:
- Privileged mode for Docker-in-Docker
- Custom Docker images
- ECR integration

// CodeBuild project in CDK
new codebuild.Project(this, 'BuildProject', {
  source: codebuild.Source.codeCommit({ repository: repo }),
  environment: {
    buildImage: codebuild.LinuxBuildImage.STANDARD_7_0,
    computeType: codebuild.ComputeType.MEDIUM,
    privileged: true  // For Docker builds
  },
  environmentVariables: {
    ECR_REPO: { value: ecrRepo.repositoryUri }
  },
  cache: codebuild.Cache.local(codebuild.LocalCacheMode.DOCKER_LAYER)
});
```

---

## CodeDeploy

### Deployment Strategies

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Deployment Strategies                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. In-Place (Rolling)                                                   │
│     ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                                    │
│     │ v1  │ │ v1  │ │ v2  │ │ v2  │  → One at a time                   │
│     └─────┘ └─────┘ └─────┘ └─────┘                                    │
│                                                                          │
│  2. Blue/Green (EC2/On-Premises)                                        │
│     ┌─────────────┐      ┌─────────────┐                                │
│     │  Blue (v1)  │      │ Green (v2)  │                                │
│     │  (current)  │  ──▶ │  (new)      │  → Traffic switch             │
│     └─────────────┘      └─────────────┘                                │
│                                                                          │
│  3. Blue/Green (ECS)                                                     │
│     ALB ──▶ Target Group 1 (Blue)                                       │
│         └─▶ Target Group 2 (Green)  → Weighted routing                 │
│                                                                          │
│  4. Canary                                                               │
│     10% ──▶ v2 (test)                                                   │
│     90% ──▶ v1 (stable)  → Gradual shift                               │
│                                                                          │
│  5. Linear                                                               │
│     10% every 10 min until 100%                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### AppSpec File (EC2/On-Premises)

```yaml
# appspec.yml
version: 0.0
os: linux

files:
  - source: /
    destination: /var/www/html

permissions:
  - object: /var/www/html
    owner: www-data
    group: www-data
    mode: 755
    type:
      - directory
      - file

hooks:
  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 300
      runas: root

  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 300
      runas: root

  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 300
      runas: root

  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 300
      runas: root
```

### AppSpec File (ECS)

```yaml
# appspec.yml for ECS
version: 0.0

Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:us-east-1:123:task-definition/app:1"
        LoadBalancerInfo:
          ContainerName: "app"
          ContainerPort: 8080

Hooks:
  - BeforeInstall: "LambdaFunctionToValidateBeforeInstall"
  - AfterInstall: "LambdaFunctionToValidateAfterInstall"
  - AfterAllowTestTraffic: "LambdaFunctionToRunTests"
  - BeforeAllowTraffic: "LambdaFunctionToValidateBeforeTraffic"
  - AfterAllowTraffic: "LambdaFunctionToValidateAfterTraffic"
```

### Rollback Configuration

```
Automatic Rollback Triggers:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  1. Deployment fails                                                     │
│     - Script errors                                                     │
│     - Health check failures                                             │
│     - Timeout                                                           │
│                                                                          │
│  2. CloudWatch Alarms                                                    │
│     - Error rate spike                                                  │
│     - Latency increase                                                  │
│     - Custom metrics                                                    │
│                                                                          │
│  Configuration:                                                          │
│  {                                                                       │
│    "autoRollbackConfiguration": {                                       │
│      "enabled": true,                                                   │
│      "events": [                                                        │
│        "DEPLOYMENT_FAILURE",                                            │
│        "DEPLOYMENT_STOP_ON_ALARM"                                       │
│      ]                                                                   │
│    },                                                                    │
│    "alarmConfiguration": {                                              │
│      "enabled": true,                                                   │
│      "alarms": [                                                        │
│        {"name": "HighErrorRate"},                                       │
│        {"name": "HighLatency"}                                          │
│      ]                                                                   │
│    }                                                                     │
│  }                                                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## CodePipeline

### Pipeline Structure

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CodePipeline                                      │
│                                                                          │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐  │
│  │ Source  │──▶│  Build  │──▶│  Test   │──▶│ Staging │──▶│  Prod   │  │
│  │         │   │         │   │         │   │ Deploy  │   │ Deploy  │  │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘  │
│                                                 │              │        │
│                                           [Auto]         [Manual        │
│                                                           Approval]     │
│                                                                          │
│  Stage Actions:                                                          │
│  - Source: CodeCommit, GitHub, S3, ECR                                  │
│  - Build: CodeBuild, Jenkins                                            │
│  - Test: CodeBuild, Device Farm, Third-party                           │
│  - Deploy: CodeDeploy, ECS, S3, CloudFormation, Lambda                 │
│  - Approval: Manual, SNS notification                                   │
│  - Invoke: Lambda, Step Functions                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pipeline Definition (CDK)

```typescript
import * as cdk from 'aws-cdk-lib';
import * as codepipeline from 'aws-cdk-lib/aws-codepipeline';
import * as codepipeline_actions from 'aws-cdk-lib/aws-codepipeline-actions';
import * as codebuild from 'aws-cdk-lib/aws-codebuild';

// Source
const sourceOutput = new codepipeline.Artifact();
const sourceAction = new codepipeline_actions.CodeCommitSourceAction({
  actionName: 'Source',
  repository: repo,
  branch: 'main',
  output: sourceOutput
});

// Build
const buildOutput = new codepipeline.Artifact();
const buildAction = new codepipeline_actions.CodeBuildAction({
  actionName: 'Build',
  project: buildProject,
  input: sourceOutput,
  outputs: [buildOutput]
});

// Deploy to Staging
const deployToStaging = new codepipeline_actions.EcsDeployAction({
  actionName: 'DeployToStaging',
  service: stagingService,
  input: buildOutput
});

// Manual Approval
const manualApproval = new codepipeline_actions.ManualApprovalAction({
  actionName: 'PromoteToProd',
  notificationTopic: approvalTopic,
  additionalInformation: 'Please review staging deployment before promoting to production'
});

// Deploy to Production (Blue/Green)
const deployToProd = new codepipeline_actions.CodeDeployEcsDeployAction({
  actionName: 'DeployToProd',
  deploymentGroup: prodDeploymentGroup,
  appSpecTemplateInput: buildOutput,
  taskDefinitionTemplateInput: buildOutput
});

// Pipeline
new codepipeline.Pipeline(this, 'Pipeline', {
  pipelineName: 'MyAppPipeline',
  stages: [
    { stageName: 'Source', actions: [sourceAction] },
    { stageName: 'Build', actions: [buildAction] },
    { stageName: 'DeployStaging', actions: [deployToStaging] },
    { stageName: 'Approval', actions: [manualApproval] },
    { stageName: 'DeployProd', actions: [deployToProd] }
  ]
});
```

### Cross-Account Pipeline

```
Cross-Account Deployment:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  DevOps Account                 Staging Account       Prod Account      │
│  (Pipeline)                     (Deploy Target)       (Deploy Target)   │
│                                                                          │
│  ┌───────────────┐             ┌───────────────┐     ┌───────────────┐ │
│  │ CodePipeline  │─────────────│ Assume Role   │     │ Assume Role   │ │
│  │               │             │               │     │               │ │
│  │ KMS Key       │◀────────────│ KMS Access    │     │ KMS Access    │ │
│  │ S3 Artifacts  │◀────────────│ S3 Access     │     │ S3 Access     │ │
│  └───────────────┘             └───────────────┘     └───────────────┘ │
│                                                                          │
│  Requirements:                                                           │
│  1. Cross-account IAM roles                                             │
│  2. KMS key policy allows target accounts                               │
│  3. S3 bucket policy allows target accounts                             │
│  4. Pipeline assumes role in target account                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## GitHub Actions with AWS

```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: my-app
  ECS_SERVICE: my-app-service
  ECS_CLUSTER: my-cluster

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push image
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Update ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: app
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

### OIDC Authentication (No Secrets)

```yaml
# IAM Role trust policy for GitHub Actions
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:*"
        }
      }
    }
  ]
}

# Benefits:
# - No long-lived credentials
# - Fine-grained repo/branch control
# - Audit trail
```

---

## CI/CD Best Practices

### Pipeline Security

```
Security Checklist:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  1. Secrets Management                                                   │
│     ✓ Use Secrets Manager/Parameter Store                               │
│     ✓ Never store secrets in code                                       │
│     ✓ Rotate credentials regularly                                      │
│     ✓ Use OIDC for GitHub Actions                                       │
│                                                                          │
│  2. IAM Least Privilege                                                  │
│     ✓ Minimal permissions for build role                                │
│     ✓ Separate roles per environment                                    │
│     ✓ No admin access in pipelines                                      │
│                                                                          │
│  3. Artifact Security                                                    │
│     ✓ Encrypt artifacts at rest (KMS)                                   │
│     ✓ Sign container images                                             │
│     ✓ Scan images for vulnerabilities                                   │
│                                                                          │
│  4. Pipeline Protection                                                  │
│     ✓ Branch protection rules                                           │
│     ✓ Required reviews for main                                         │
│     ✓ Manual approval for production                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Testing Strategy

```
Test Pyramid in Pipeline:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│                    /\                                                    │
│                   /  \         E2E Tests (few)                          │
│                  /────\        - After staging deploy                   │
│                 /      \       - Critical paths only                    │
│                /────────\                                                │
│               /          \     Integration Tests                        │
│              /────────────\    - CodeBuild post-build                   │
│             /              \   - API tests, DB tests                    │
│            /────────────────\                                            │
│           /                  \  Unit Tests (many)                       │
│          /────────────────────\ - CodeBuild build phase                 │
│         /                      \- Fast, isolated                        │
│                                                                          │
│   Pipeline Stages:                                                       │
│   Build → Unit Tests → Integration Tests → Deploy Staging → E2E → Prod │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Interview Discussion Points

### How do you design a CI/CD pipeline for zero-downtime deployments?

```
Strategy:

1. Blue/Green or Canary deployment
   - Two identical environments
   - Route traffic gradually
   - Instant rollback capability

2. Health Checks
   - Readiness probes before traffic
   - Liveness probes during deployment
   - Custom validation Lambda hooks

3. Database Migrations
   - Backward-compatible changes
   - Separate migration from deployment
   - Feature flags for schema changes

4. Rollback Automation
   - CloudWatch alarms trigger rollback
   - Keep previous version available
   - Automated smoke tests

5. Traffic Management
   - ALB weighted routing
   - Route 53 failover
   - Feature flags for gradual rollout

Implementation:
- ECS Blue/Green via CodeDeploy
- Linear10PercentEvery1Minute for canary
- Automatic rollback on alarm
```

### How do you handle secrets in CI/CD?

```
Secrets Management:

1. Storage
   - AWS Secrets Manager (recommended)
   - Systems Manager Parameter Store
   - Never in source code or env vars

2. Access
   - IAM roles for build/deploy processes
   - Short-lived credentials only
   - Audit all access (CloudTrail)

3. Rotation
   - Automated rotation (Secrets Manager)
   - No manual credential handling
   - Zero-downtime rotation

4. Build-time vs Runtime
   - Build: Parameter Store references
   - Runtime: Secrets Manager SDK calls
   - ECS: Secret injection from SM

Example buildspec.yml:
env:
  secrets-manager:
    DB_PASSWORD: prod/database:password
    API_KEY: prod/api:key

ECS Task Definition:
"secrets": [{
  "name": "DB_PASSWORD",
  "valueFrom": "arn:aws:secretsmanager:...:prod/database"
}]
```

### How do you handle rollbacks?

```
Rollback Strategies:

1. Automatic (Recommended)
   - Deploy failure → immediate rollback
   - Alarm triggered → automatic rollback
   - Health check failure → rollback

2. Manual
   - Re-deploy previous version
   - Keep artifacts for N versions
   - One-click rollback in console

3. Implementation
   CodeDeploy:
   - Keeps previous deployment
   - Blue/Green allows instant switch
   - AppSpec hooks for validation

   ECS:
   - Previous task definition available
   - Force new deployment with old version
   - Traffic shift back to blue

4. Database Considerations
   - Forward-only migrations
   - Feature flags for new features
   - Separate rollback scripts if needed

Best Practice:
- Test rollback in staging
- Document rollback procedures
- Practice chaos engineering
```
