# Containers on AWS (ECS & EKS)

## Container Services Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AWS Container Services                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Orchestration:          │   Compute:          │   Registry:           │
│   ┌───────────────────┐   │   ┌──────────────┐  │   ┌────────────────┐  │
│   │ ECS (AWS native)  │   │   │ EC2          │  │   │ ECR            │  │
│   │ EKS (Kubernetes)  │   │   │ Fargate      │  │   │ (Private       │  │
│   └───────────────────┘   │   └──────────────┘  │   │  Docker Hub)   │  │
│                           │                     │   └────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘

ECS vs EKS Decision:
┌──────────────────────────┬───────────────────────────────────────────────┐
│         ECS              │                    EKS                        │
├──────────────────────────┼───────────────────────────────────────────────┤
│ Simpler, AWS-native      │ Kubernetes standard                          │
│ Tighter AWS integration  │ Portability across clouds                    │
│ No control plane cost    │ $0.10/hour for control plane                 │
│ Less learning curve      │ K8s expertise required                       │
│ Good for AWS-only        │ Good for multi-cloud/K8s investment          │
└──────────────────────────┴───────────────────────────────────────────────┘
```

---

## ECS (Elastic Container Service)

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          ECS Cluster                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      ECS Service                                 │   │
│   │  (manages desired count, load balancing, deployments)            │   │
│   │                                                                  │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │   │
│   │   │    Task      │  │    Task      │  │    Task      │         │   │
│   │   │ (1+ contain- │  │              │  │              │         │   │
│   │   │    ers)      │  │              │  │              │         │   │
│   │   └──────────────┘  └──────────────┘  └──────────────┘         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Running on: EC2 instances OR Fargate (serverless)                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Task Definition

```json
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "ENV", "value": "production"}
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:db-pass"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      }
    }
  ]
}
```

### Service Configuration

```json
{
  "serviceName": "my-service",
  "cluster": "my-cluster",
  "taskDefinition": "my-app:1",
  "desiredCount": 3,
  "launchType": "FARGATE",
  "networkConfiguration": {
    "awsvpcConfiguration": {
      "subnets": ["subnet-xxx", "subnet-yyy"],
      "securityGroups": ["sg-xxx"],
      "assignPublicIp": "DISABLED"
    }
  },
  "loadBalancers": [
    {
      "targetGroupArn": "arn:aws:elasticloadbalancing:...",
      "containerName": "app",
      "containerPort": 8080
    }
  ],
  "deploymentConfiguration": {
    "minimumHealthyPercent": 50,
    "maximumPercent": 200,
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    }
  },
  "enableExecuteCommand": true
}
```

### EC2 vs Fargate

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EC2 Launch Type                                       │
├─────────────────────────────────────────────────────────────────────────┤
│ ✓ Full control over instances                                           │
│ ✓ GPU support                                                           │
│ ✓ Spot instances for cost savings                                       │
│ ✓ Better for steady-state workloads                                     │
│ ✗ Manage EC2 fleet, capacity, patching                                  │
│ ✗ Cluster capacity planning                                             │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    Fargate Launch Type                                   │
├─────────────────────────────────────────────────────────────────────────┤
│ ✓ Serverless - no EC2 management                                        │
│ ✓ Pay per task (vCPU + memory)                                          │
│ ✓ Automatic scaling                                                      │
│ ✓ Better security isolation                                             │
│ ✗ Higher cost at scale                                                  │
│ ✗ Limited to supported CPU/memory combinations                          │
│ ✗ No GPU support                                                        │
└─────────────────────────────────────────────────────────────────────────┘

Cost Comparison (approximate):
- Fargate: ~$0.04/vCPU-hour + ~$0.004/GB-hour
- EC2 (m5.large): ~$0.096/hour = ~$0.048/vCPU-hour
- Spot EC2: 60-90% cheaper than On-Demand

Breakeven: Fargate often cheaper at low utilization (<60%)
           EC2 cheaper at high, steady utilization
```

---

## EKS (Elastic Kubernetes Service)

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         EKS Cluster                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │              Control Plane (AWS Managed)                         │   │
│   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │   │
│   │   │ API      │  │ etcd     │  │ Scheduler│  │Controller│       │   │
│   │   │ Server   │  │          │  │          │  │ Manager  │       │   │
│   │   └──────────┘  └──────────┘  └──────────┘  └──────────┘       │   │
│   │               (Runs across 3 AZs automatically)                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                      │
│                                   ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │              Data Plane (You Manage or Fargate)                  │   │
│   │   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐      │   │
│   │   │   Node        │  │   Node        │  │   Node        │      │   │
│   │   │ (EC2/Fargate) │  │               │  │               │      │   │
│   │   │  ┌─────────┐  │  │  ┌─────────┐  │  │  ┌─────────┐  │      │   │
│   │   │  │   Pod   │  │  │  │   Pod   │  │  │  │   Pod   │  │      │   │
│   │   │  └─────────┘  │  │  └─────────┘  │  │  └─────────┘  │      │   │
│   │   └───────────────┘  └───────────────┘  └───────────────┘      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Node Types

```
1. Managed Node Groups
   - AWS manages node provisioning and lifecycle
   - Automatic updates and patching
   - Integration with ASG

2. Self-Managed Nodes
   - Full control over EC2 instances
   - Custom AMIs
   - Complex networking requirements

3. Fargate
   - Serverless nodes
   - Per-pod pricing
   - No node management
   - Some limitations (daemonsets, privileged containers)
```

### Key EKS Integrations

```yaml
# AWS Load Balancer Controller
# Provisions ALB/NLB automatically from Kubernetes

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
---
# IAM Roles for Service Accounts (IRSA)
# Pods get IAM permissions without instance role

apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/my-app-role

---
# External Secrets Operator
# Sync secrets from Secrets Manager to K8s

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: prod/db/password
```

---

## ECR (Elastic Container Registry)

### Repository Operations

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name my-app

# Build and push
docker build -t my-app .
docker tag my-app:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

### Lifecycle Policies

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 production images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Delete untagged images older than 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

### Image Scanning

```
ECR Image Scanning:
- Basic scanning: CVE database (free)
- Enhanced scanning: Inspector integration (cost)

Scan triggers:
- On push (automatic)
- Manual scan
- Continuous (enhanced only)

Best practice:
- Block deployments with critical vulnerabilities
- Automate in CI/CD pipeline
```

---

## Deployment Strategies

### ECS Deployment Types

```
Rolling Update (Default):
┌────┐ ┌────┐ ┌────┐    ┌────┐ ┌────┐ ┌────┐    ┌────┐ ┌────┐ ┌────┐
│ v1 │ │ v1 │ │ v1 │ → │ v2 │ │ v1 │ │ v1 │ → │ v2 │ │ v2 │ │ v2 │
└────┘ └────┘ └────┘    └────┘ └────┘ └────┘    └────┘ └────┘ └────┘
- minimumHealthyPercent: 50
- maximumPercent: 200

Blue/Green (with CodeDeploy):
┌─────────────────┐         ┌─────────────────┐
│  Blue (v1)      │   →     │  Green (v2)     │
│  ████████████   │   ALB   │  ████████████   │
│  (100% traffic) │  shift  │  (100% traffic) │
└─────────────────┘         └─────────────────┘

Options:
- AllAtOnce
- Linear (10% every 10 min)
- Canary (10% then 90%)
```

### Kubernetes Deployment Strategies

```yaml
# Rolling Update
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

---
# Blue/Green with Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    blueGreen:
      activeService: my-app-active
      previewService: my-app-preview
      autoPromotionEnabled: false

---
# Canary with Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 1h}
        - setWeight: 50
        - pause: {duration: 1h}
        - setWeight: 100
```

---

## Service Discovery

### ECS Service Discovery

```
Cloud Map Integration:
┌─────────────────────────────────────────────────────────────────────────┐
│                        AWS Cloud Map                                     │
│                                                                          │
│   Namespace: my-app.local                                               │
│   ├── Service: api                                                       │
│   │   └── api.my-app.local → 10.0.1.5, 10.0.2.6                        │
│   └── Service: worker                                                    │
│       └── worker.my-app.local → 10.0.1.7                                │
└─────────────────────────────────────────────────────────────────────────┘

ECS Service configuration:
{
  "serviceRegistries": [{
    "registryArn": "arn:aws:servicediscovery:...",
    "containerName": "app",
    "containerPort": 8080
  }]
}
```

### EKS Service Discovery

```yaml
# Kubernetes native service discovery
apiVersion: v1
kind: Service
metadata:
  name: my-api
spec:
  selector:
    app: my-api
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP

# Access via: my-api.default.svc.cluster.local
```

---

## Interview Discussion Points

### ECS vs EKS - How do you choose?

```
Choose ECS when:
- AWS-only environment
- Simpler orchestration needs
- Team not familiar with Kubernetes
- Cost-sensitive (no control plane cost)
- Tighter AWS service integration needed

Choose EKS when:
- Multi-cloud or hybrid strategy
- Team has Kubernetes expertise
- Need Kubernetes ecosystem (Helm, operators)
- Portability is important
- Complex networking/service mesh requirements
```

### How do you handle secrets in containers?

```
1. AWS Secrets Manager / Parameter Store
   - Task/pod pulls at runtime
   - Automatic rotation support
   - Audit trail

2. Container Environment Variables
   - ECS: secrets in task definition
   - EKS: External Secrets Operator

3. Mounted Volumes
   - Secrets mounted as files
   - More secure than env vars

Best practice:
- Never bake secrets into images
- Use IAM roles for AWS access
- Rotate secrets regularly
- Audit secret access
```

### How do you optimize container costs?

```
1. Right-size tasks/pods
   - Monitor actual usage
   - Adjust CPU/memory limits

2. Use Spot for fault-tolerant workloads
   - ECS: Capacity providers with Spot
   - EKS: Spot node groups

3. Fargate vs EC2 analysis
   - Fargate: variable workloads
   - EC2: steady, high utilization

4. Cluster optimization
   - Cluster Autoscaler / Karpenter
   - Bin packing efficiency
   - Turn off non-prod at night
```
