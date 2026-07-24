# ECS

## What is it?

**Amazon Elastic Container Service (ECS)** is a fully managed container orchestration service provided by AWS under the **Compute** category. It allows you to run, stop, and manage Docker containers on a cluster of virtual machines (EC2 instances) or using AWS Fargate (serverless compute).

ECS eliminates the need to install, operate, and scale your own cluster management infrastructure. It is deeply integrated with the AWS ecosystem, including IAM, VPC, ALB, CloudWatch, ECR, Secrets Manager, and more.

ECS supports two primary launch types:
- **EC2 Launch Type**: Containers run on EC2 instances that you manage within your cluster.
- **Fargate Launch Type**: Containers run on serverless infrastructure managed entirely by AWS — no EC2 instances to provision or manage.

ECS is also compatible with **ECS Anywhere**, which allows you to run ECS tasks on on-premises infrastructure or other cloud environments.

---

## Why do we need it?

### The Problem it Solves

Before container orchestration services like ECS, teams faced significant operational challenges:
- **Manual scaling**: Spinning up containers manually on EC2 was error-prone and slow.
- **No health management**: If a container crashed, no automated recovery existed.
- **No service discovery**: Containers had dynamic IPs, making inter-service communication difficult.
- **Resource bin-packing**: Efficiently placing containers on hosts to maximize resource utilization was complex.
- **Networking complexity**: Connecting containers across hosts securely required custom tooling.

### When to Use ECS

| Scenario | Recommendation |
|---|---|
| Microservices architecture | ✅ Ideal — each service as a separate ECS service |
| Batch processing jobs | ✅ Use ECS tasks with scheduled triggers |
| Lift-and-shift containerized apps | ✅ EC2 launch type with existing infrastructure |
| Serverless containers | ✅ Fargate launch type |
| Need Kubernetes | ❌ Consider EKS instead |
| Simple single-container app | ⚠️ App Runner may be simpler |

### Real Business Scenarios

1. **E-commerce Platform**: A retail company runs separate ECS services for product catalog, cart, payments, and notifications — each independently scaled and deployed.
2. **Media Processing**: A streaming company uses ECS Fargate tasks to transcode video files triggered by S3 uploads via EventBridge.
3. **SaaS Application**: A B2B SaaS provider runs multi-tenant microservices on ECS with ALB path-based routing to isolate tenant traffic.
4. **Nightly ETL Jobs**: A data engineering team runs ECS scheduled tasks every night to extract, transform, and load data into Redshift.

---

## Internal Working

### How ECS Works Internally

```
┌─────────────────────────────────────────────────────────┐
│                    ECS Control Plane                     │
│  (AWS Managed - Cluster Scheduler, API, State Manager)  │
└────────────────────────┬────────────────────────────────┘
                         │ API Calls / Agent Communication
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
    ┌─────────────┐ ┌─────────────┐ ┌──────────────┐
    │  EC2 Node 1 │ │  EC2 Node 2 │ │ Fargate Node │
    │ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌──────────┐ │
    │ │ECS Agent│ │ │ │ECS Agent│ │ │ │  Task    │ │
    │ └─────────┘ │ │ └─────────┘ │ │ └──────────┘ │
    │ ┌─────────┐ │ │ ┌─────────┐ │ └──────────────┘
    │ │Task 1   │ │ │ │Task 3   │ │
    │ │Task 2   │ │ │ │Task 4   │ │
    └─────────────┘ └─────────────┘
```

### Step-by-Step Internal Flow

1. **Task Definition Registration**: You define a Task Definition — a JSON blueprint describing container images, CPU/memory, ports, environment variables, IAM roles, logging, and volumes.

2. **Cluster Creation**: A logical grouping of compute resources (EC2 instances or Fargate capacity) is created. ECS registers compute resources into the cluster.

3. **ECS Agent**: On EC2 launch type, each EC2 instance runs the **ECS Agent** (an open-source Go application). The agent:
   - Polls the ECS control plane via long-polling HTTPS
   - Reports instance resource availability (CPU, memory)
   - Starts/stops containers via the Docker daemon
   - Sends container health status back to ECS

4. **Task Scheduling**: When you run a task or create a service, the ECS **Scheduler** selects the optimal placement based on:
   - Available CPU and memory
   - Placement constraints (e.g., `distinctInstance`, `memberOf`)
   - Placement strategies (e.g., `binpack`, `spread`, `random`)

5. **Task Execution**: The scheduler sends instructions to the ECS Agent on the selected host. The agent pulls the Docker image from ECR (or Docker Hub), creates the container, and starts it.

6. **Service Management**: For long-running workloads, ECS Services maintain a desired count of tasks. If a task fails health checks or crashes, ECS automatically replaces it.

7. **Fargate Internals**: With Fargate, AWS provisions micro-VMs (using Firecracker or similar technology) per task. Each task gets its own isolated kernel, eliminating noisy-neighbor issues. You don't see or manage the underlying host.

---

## Architecture

### Core Components

```
┌──────────────────────────────────────────────────────────────────┐
│                          ECS Cluster                             │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                     ECS Service                            │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │  │
│  │  │    Task 1    │  │    Task 2    │  │    Task 3    │    │  │
│  │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │    │  │
│  │  │ │Container │ │  │ │Container │ │  │ │Container │ │    │  │
│  │  │ │  (App)   │ │  │ │  (App)   │ │  │ │  (App)   │ │    │  │
│  │  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │    │  │
│  │  │ ┌──────────┐ │  │              │  │              │    │  │
│  │  │ │Sidecar   │ │  │              │  │              │    │  │
│  │  │ │(Envoy)   │ │  │              │  │              │    │  │
│  │  │ └──────────┘ │  │              │  │              │    │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Task Definition          Service Discovery       Load Balancer  │
│  (Blueprint)              (Cloud Map)             (ALB/NLB)      │
└──────────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

#### 1. **Cluster**
- Logical boundary for ECS resources
- Can contain both EC2 and Fargate tasks
- Spans multiple Availability Zones
- Associated with a VPC

#### 2. **Task Definition**
- Immutable versioned JSON document
- Defines 1 to 10 containers per task
- Key parameters:
  - `image`: Docker image URI
  - `cpu` / `memory`: Resource allocation
  - `portMappings`: Container-to-host port mapping
  - `environment`: Environment variables
  - `secrets`: References to Secrets Manager / SSM Parameter Store
  - `logConfiguration`: CloudWatch Logs, Splunk, Firehose
  - `healthCheck`: Container health check command
  - `executionRoleArn`: Role for ECS agent to pull images and push logs
  - `taskRoleArn`: Role for the application code inside the container

#### 3. **Task**
- A running instance of a Task Definition
- Ephemeral by default (for batch/one-off jobs)
- Can be standalone or managed by a Service

#### 4. **Service**
- Maintains a desired count of tasks
- Integrates with ALB/NLB for load balancing
- Supports rolling updates and blue/green deployments (via CodeDeploy)
- Supports Service Auto Scaling

#### 5. **Container Agent**
- Runs on each EC2 instance in the cluster
- Open source: `aws/amazon-ecs-agent`
- Communicates with ECS control plane

#### 6. **Capacity Providers**
- Define the compute capacity for your cluster
- Types: `FARGATE`, `FARGATE_SPOT`, or Auto Scaling Groups (ASG)
- Capacity Provider Strategy: defines how tasks are distributed across providers

### Networking Modes

| Mode | Description | Use Case |
|---|---|---|
| `awsvpc` | Each task gets its own ENI and private IP | Recommended — Fargate requires this |
| `bridge` | Docker's default bridge networking | EC2 only, legacy |
| `host` | Container uses host's network | EC2 only, high performance |
| `none` | No external networking | Isolated tasks |

---

## Real World Example

### Scenario: Multi-Tier E-Commerce Microservices Platform

**Company**: RetailCo — an online retailer expecting 10x traffic during holiday season.

**Architecture Goal**: Deploy a containerized microservices application with independent scaling per service.

#### Services to Deploy:
- **API Gateway Service** (handles routing)
- **Product Catalog Service** (reads product data from RDS)
- **Order Service** (writes orders to DynamoDB)
- **Notification Service** (sends emails via SES)

#### Step-by-Step Walkthrough

**Step 1: Push Docker Images to ECR**
```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Build and push product catalog service
docker build -t product-catalog ./product-catalog
docker tag product-catalog:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/product-catalog:v1.2.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/product-catalog:v1.2.0
```

**Step 2: Create ECS Cluster**
```bash
aws ecs create-cluster \
  --cluster-name retailco-prod \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=1 \
    capacityProvider=FARGATE_SPOT,weight=4
```

**Step 3: Register Task Definition**
```json
{
  "family": "product-catalog-td",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/productCatalogTaskRole",
  "containerDefinitions": [
    {
      "name": "product-catalog",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/product-catalog:v1.2.0",
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "environment": [
        { "name": "NODE_ENV", "value": "production" }
      ],
      "secrets": [
        { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/db/password" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/product-catalog",
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

**Step 4: Create ECS Service with ALB**
```bash
aws ecs create-service \
  --cluster retailco-prod \
  --service-name product-catalog-svc \
  --task-definition product-catalog-td:1 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-abc123,subnet-def456],
    securityGroups=[sg-xyz789],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789:targetgroup/product-catalog-tg/abc,
    containerName=product-catalog,
    containerPort=8080" \
  --deployment-configuration "minimumHealthyPercent=100,maximumPercent=200"
```

**Step 5: Configure Auto Scaling**
- Set minimum tasks: 3, maximum: 30
- Scale out when CPU > 70% for 2 minutes
- Scale in when CPU < 30% for 10 minutes

**Step 6: Deploy Update (Rolling)**
- Update Task Definition with new image tag `v1.3.0`
- Update service → ECS performs rolling deployment
- New tasks start, pass health checks, old tasks drain from ALB and stop

**Result**: RetailCo successfully handles 10x holiday traffic with zero downtime deployments and automatic scaling.

---

## Advantages

1. **Fully Managed**: No control plane to manage, patch, or upgrade — AWS handles it entirely.

2. **Deep AWS Integration**: Native integration with ALB, NLB, IAM, CloudWatch, ECR, Secrets Manager, Service Connect, App Mesh, and more.

3. **Fargate Serverless**: With Fargate, zero infrastructure management. Pay only for vCPU and memory used by your tasks.

4. **Flexible Launch Types**: Mix EC2 (for cost optimization with reserved/spot instances) and Fargate (for simplicity) in the same cluster.

5. **ECS Anywhere**: Run ECS tasks on on-premises servers, edge locations, or other clouds — unified control plane.

6. **Mature and Battle-Tested**: Used by Amazon itself and thousands of enterprises at massive scale.

7. **Simpler than Kubernetes**: Lower operational overhead compared to EKS — ideal for teams without Kubernetes expertise.

8. **Cost-Effective Spot Integration**: FARGATE_SPOT offers up to 70% cost savings for fault-tolerant workloads.

9. **Blue/Green Deployments**: Native integration with AWS CodeDeploy for zero-downtime blue/green deployments.

10. **Service Connect**: Built-in service discovery and inter-service communication with metrics — no need for a separate service mesh.

11. **Fine-Grained IAM**: Separate `executionRole` (for ECS