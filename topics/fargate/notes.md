# Fargate

## What is it?

**AWS Fargate** is a serverless compute engine for containers that works with both **Amazon Elastic Container Service (ECS)** and **Amazon Elastic Kubernetes Service (EKS)**. It belongs to the **Container Services** category within AWS.

Fargate eliminates the need to provision, configure, and manage EC2 instances (the underlying server infrastructure) to run containerized applications. Instead, you define your container's CPU and memory requirements, and AWS automatically provisions the right amount of compute resources, runs your containers, and scales them on demand.

> **Official Definition:** AWS Fargate is a technology that you can use with Amazon ECS to run containers without having to manage servers or clusters of Amazon EC2 instances.

**Key identifiers:**
- **Service Type:** Serverless container compute engine
- **Launch Type:** Fargate (used within ECS and EKS)
- **Compute Model:** Per-task/per-pod resource allocation
- **Underlying Infrastructure:** Fully managed by AWS (invisible to users)

---

## Why do we need it?

### The Problem It Solves

Before Fargate, running containers on AWS required managing EC2-based clusters:

| Pain Point | Traditional EC2-based ECS | Fargate |
|---|---|---|
| Server provisioning | Manual or via ASG | Automatic |
| OS patching | Customer responsibility | AWS managed |
| Cluster capacity planning | Required | Not needed |
| Idle resource waste | Common | Eliminated |
| Security hardening of hosts | Required | AWS managed |
| Scaling complexity | High | Simplified |

### When to Use It

1. **Microservices Architecture** – Teams want to focus on application logic, not infrastructure management.
2. **Batch Processing Jobs** – Run short-lived, resource-intensive tasks without pre-provisioning servers.
3. **CI/CD Pipelines** – Spin up containers on demand for build and test workloads.
4. **Variable/Unpredictable Workloads** – Applications with spiky traffic patterns where over-provisioning EC2 is wasteful.
5. **Small Engineering Teams** – Startups or teams without dedicated DevOps/infrastructure engineers.
6. **Regulated Industries** – Where isolating workloads at the compute level (no shared kernel) is a compliance requirement.

### Real Business Scenarios

- **E-commerce company** running seasonal sale promotions needs burst capacity for order processing microservices without maintaining a large standing EC2 cluster year-round.
- **Healthcare SaaS** needs HIPAA-compliant container isolation where each customer's workload runs in isolated compute environments.
- **Media streaming company** runs nightly video transcoding jobs as Fargate tasks that spin up, process, and terminate — paying only for actual processing time.
- **FinTech startup** with a 3-person engineering team deploys APIs and background workers without hiring a dedicated infrastructure team.

---

## Internal Working

### How Fargate Provisions Compute

When you launch a Fargate task, the following sequence occurs under the hood:

```
User defines Task Definition (CPU, Memory, Container Image)
         │
         ▼
ECS Control Plane receives RunTask API call
         │
         ▼
Fargate Scheduler selects an appropriate compute zone
         │
         ▼
AWS provisions a micro-VM (using Firecracker or similar isolation tech)
         │
         ▼
A dedicated kernel is allocated per task (no shared host OS)
         │
         ▼
Container runtime (containerd) pulls the image from ECR/Docker Hub
         │
         ▼
Container starts, network interface (ENI) is attached
         │
         ▼
Task runs; AWS monitors health, restarts if needed
         │
         ▼
Task stops → compute resources are fully released and billed stops
```

### Isolation Model

Fargate uses **Firecracker** — an open-source Virtual Machine Monitor (VMM) developed by AWS — to provide strong isolation between tasks. Each Fargate task gets:

- **Its own dedicated kernel** (not shared with other customers)
- **Its own network namespace** (via an Elastic Network Interface)
- **Isolated ephemeral storage**
- **Dedicated CPU and memory allocation**

This is fundamentally different from traditional container runtimes where containers on the same host share the host OS kernel.

### Fargate Runtime Versions

Fargate platform versions control the runtime environment:

| Platform Version | Key Features |
|---|---|
| **1.4.0** (Latest for ECS) | 20 GiB ephemeral storage, Fargate VPC endpoints, secrets injection improvements |
| **1.3.0** | Secrets Manager integration, SSM Parameter Store |
| **1.2.0** | Task metadata endpoint v3 |
| **1.1.0** | Exec into containers support |
| **LATEST** | Alias for the most recent stable version |

For **EKS Fargate**, the Fargate profile concept is used instead of platform versions.

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                          AWS Account                            │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Amazon ECS Cluster                    │  │
│  │                                                          │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │  │
│  │  │   Service   │    │   Service   │    │  Scheduled  │  │  │
│  │  │  (Web API)  │    │ (Worker)    │    │    Task     │  │  │
│  │  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘  │  │
│  │         │                  │                  │          │  │
│  │  ┌──────▼──────────────────▼──────────────────▼──────┐  │  │
│  │  │              Fargate Launch Type                   │  │  │
│  │  │                                                    │  │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐        │  │  │
│  │  │  │  Task 1  │  │  Task 2  │  │  Task 3  │        │  │  │
│  │  │  │ [ENI]    │  │ [ENI]    │  │ [ENI]    │        │  │  │
│  │  │  │ Container│  │ Container│  │ Container│        │  │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘        │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │   ECR    │  │   ALB    │  │CloudWatch│  │ Secrets Mgr  │  │
│  │(Registry)│  │(Load Bal)│  │ (Logs)   │  │  (Secrets)   │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

#### 1. Task Definition
The blueprint for your application. Defines:
- Container image(s) to use
- CPU and memory requirements
- Environment variables and secrets
- Logging configuration
- IAM roles
- Network mode (always `awsvpc` for Fargate)
- Port mappings

#### 2. ECS Service
Long-running, managed group of tasks:
- Maintains desired task count
- Integrates with load balancers
- Handles rolling deployments
- Supports auto-scaling policies

#### 3. Fargate Task
The actual running unit of work:
- Gets a dedicated ENI (Elastic Network Interface)
- Has its own private IP in your VPC
- Runs one or more containers defined in the Task Definition

#### 4. VPC Networking (awsvpc mode)
Every Fargate task gets its own ENI:
- Assigned a private IP from your subnet
- Security groups applied at the task level (not host level)
- Supports VPC Flow Logs for network monitoring

#### 5. EKS Fargate Profile
For Kubernetes workloads:
- Defines which pods run on Fargate using namespace/label selectors
- Each pod gets a dedicated Fargate task
- Fargate nodes appear as standard Kubernetes nodes

### Architectural Patterns

**Pattern 1: Web Application**
```
Internet → Route 53 → CloudFront → ALB → ECS Service (Fargate) → RDS
```

**Pattern 2: Async Worker**
```
SQS Queue → EventBridge Pipe → ECS Task (Fargate) → DynamoDB
```

**Pattern 3: Scheduled Batch**
```
EventBridge Scheduler → ECS RunTask (Fargate) → S3 / Redshift
```

---

## Real World Example

### Scenario: Multi-Tier E-Commerce Application

**Context:** A retail company wants to migrate their monolithic application to microservices using Fargate. They have:
- A public-facing REST API
- An order processing worker
- A nightly inventory sync batch job

**Step-by-Step Walkthrough:**

#### Step 1: Define the Network Foundation
```
VPC (10.0.0.0/16)
├── Public Subnets (10.0.1.0/24, 10.0.2.0/24)  ← ALB lives here
└── Private Subnets (10.0.3.0/24, 10.0.4.0/24) ← Fargate tasks live here
```

#### Step 2: Create Task Definitions

**API Task Definition:**
```json
{
  "family": "ecommerce-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/ecommerceApiTaskRole",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/ecommerce-api:latest",
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "environment": [
        { "name": "NODE_ENV", "value": "production" }
      ],
      "secrets": [
        { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:db-password" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/ecommerce-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

#### Step 3: Create the ECS Cluster
```bash
aws ecs create-cluster \
  --cluster-name ecommerce-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=1 \
    capacityProvider=FARGATE_SPOT,weight=4
```

#### Step 4: Deploy the API Service with ALB
```bash
aws ecs create-service \
  --cluster ecommerce-cluster \
  --service-name ecommerce-api-service \
  --task-definition ecommerce-api:1 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-abc123,subnet-def456],
    securityGroups=[sg-api123],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,
    containerName=api,containerPort=8080" \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100"
```

#### Step 5: Configure Auto Scaling
```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/ecommerce-cluster/ecommerce-api-service \
  --min-capacity 2 \
  --max-capacity 20

# Create scaling policy
aws application-autoscaling put-scaling-policy \
  --policy-name cpu-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/ecommerce-cluster/ecommerce-api-service \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    }
  }'
```

#### Step 6: Schedule the Nightly Batch Job
```bash
aws events put-rule \
  --name nightly-inventory-sync \
  --schedule-expression "cron(0 2 * * ? *)" \
  --state ENABLED

aws events put-targets \
  --rule nightly-inventory-sync \
  --targets '[{
    "Id": "inventory-sync-task",
    "Arn": "arn:aws:ecs:us-east-1:123456789:cluster/ecommerce-cluster",
    "RoleArn": "arn:aws:iam::123456789:role/EventBridgeECSRole",
    "EcsParameters": {
      "TaskDefinitionArn": "arn:aws:ecs:us-east-1:123456789:task-definition/inventory-sync:1",
      "LaunchType": "FARGATE",
      "NetworkConfiguration": {
        "awsvpcConfiguration": {
          "Subnets": ["subnet-abc123"],
          "SecurityGroups": ["sg-batch123"],
          "AssignPublicIp": "DISABLED"
        }
      }
    }
  }]'
```

#### Result Architecture
```
Users → ALB (Public Subnets)
         ├── /api/* → ECS API Service (3-20 Fargate tasks, Private Subnets)
         └── (scales based on CPU/memory)

SQS → EventBridge Pipe → Order Worker Service (1-10 Fargate tasks)

EventBridge Scheduler (2 AM UTC) → Inventory Sync Task (runs, completes, terminates)
```

---

## Advantages

### 1. No Infrastructure Management
- Zero EC2 instance provisioning, patching, or AMI management
- No cluster capacity planning required
- AWS handles OS-level security patches automatically

### 2. Strong Security Isolation
- Each task runs in its own dedicated micro-VM
- No shared kernel between tasks (unlike traditional containers)
- Security groups applied at the task level for granular control
- Meets compliance requirements for PCI-DSS, HIPAA, SOC 2

### 3. Granular Resource Allocation
- Pay for exactly the CPU/memory you define per task
- No wasted capacity from over-provisioned EC2 instances
- Right-sizing at the task level rather than the host level

### 4. Faster Developer Velocity
- Developers focus on application code, not infrastructure
- Simplified deployment pipeline (no AMI baking, no cluster management)
- Works with existing Docker containers without modification

### 5. Flexible Scaling
- Scale from 0 to thousands of