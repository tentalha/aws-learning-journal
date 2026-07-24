# ECS — Interview Questions

---

## Easy

### 1. What is Amazon ECS, and what problem does it solve?

**Answer:**
Amazon Elastic Container Service (ECS) is a fully managed container orchestration service that allows you to run, stop, and manage Docker containers on a cluster of EC2 instances or using AWS Fargate (serverless). It solves the problem of manually managing the placement, scaling, networking, and lifecycle of containers across a fleet of servers. ECS handles scheduling, health monitoring, load balancer integration, and service discovery, so developers can focus on building applications rather than managing infrastructure.

---

### 2. What is the difference between an ECS Task and an ECS Service?

**Answer:**
- **ECS Task:** A task is a running instance of a Task Definition. It is a one-time or short-lived execution of one or more containers. Think of it as a single "run" of your containerized application (e.g., a batch job or a one-off script).
- **ECS Service:** A service is a long-running construct that ensures a specified number of tasks are always running. If a task fails or stops, the ECS Service scheduler automatically replaces it. Services also integrate with load balancers and support rolling deployments.

**Key distinction:** Tasks are ephemeral; Services are persistent and self-healing.

---

### 3. What is an ECS Task Definition?

**Answer:**
A Task Definition is a JSON blueprint that describes one or more containers that form your application. It is analogous to a Dockerfile but at the orchestration level. Key fields include:

- **Container image** (ECR URI or Docker Hub)
- **CPU and memory** allocations
- **Port mappings**
- **Environment variables** and secrets
- **IAM Task Role** for AWS API permissions
- **Logging configuration** (e.g., CloudWatch Logs)
- **Volume mounts**
- **Network mode** (bridge, awsvpc, host)

Task Definitions are versioned; each update creates a new revision.

---

### 4. What are the two launch types available in ECS?

**Answer:**
1. **EC2 Launch Type:** You provision and manage a cluster of EC2 instances (Container Instances). ECS places tasks on these instances based on available CPU/memory resources. You are responsible for patching, scaling, and managing the underlying EC2 fleet.

2. **Fargate Launch Type:** Serverless compute for containers. You do not manage any EC2 instances. You specify CPU and memory at the task level, and AWS provisions the underlying infrastructure automatically. Fargate is ideal for teams that want to avoid infrastructure management overhead.

---

### 5. What is Amazon ECR, and how does it relate to ECS?

**Answer:**
Amazon Elastic Container Registry (ECR) is a fully managed Docker container registry that stores, manages, and deploys container images. Its relationship to ECS:

- ECR stores the Docker images that ECS tasks pull and run.
- ECS Task Definitions reference ECR image URIs (e.g., `123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest`).
- ECR integrates natively with IAM for access control, with no need for separate registry credentials.
- ECR supports image scanning for vulnerabilities, lifecycle policies, and cross-region/cross-account replication.

Together, ECR + ECS form a complete container build-store-run pipeline on AWS.

---

## Medium

### 1. Explain the ECS networking modes and when you would choose each.

**Answer:**
ECS supports four network modes defined in the Task Definition:

| Mode | Description | Use Case |
|------|-------------|----------|
| `bridge` | Docker's default NAT bridge network. Containers share the host's ENI with port mapping. | EC2 launch type; legacy workloads; multiple containers per host sharing ports dynamically. |
| `awsvpc` | Each task gets its own Elastic Network Interface (ENI) with a private IP in your VPC. | Required for Fargate; recommended for EC2 when you need task-level security groups, VPC flow logs, and fine-grained network control. |
| `host` | Container uses the host EC2 instance's network directly. No NAT overhead. | High-performance workloads needing maximum network throughput; not supported on Fargate. |
| `none` | No external network connectivity. | Isolated tasks that only communicate via volumes or IPC. |

**`awsvpc` is the recommended mode** for most modern workloads because:
- Each task gets its own security group (task-level firewall rules).
- Tasks are first-class VPC citizens — they appear in VPC flow logs.
- Supports AWS PrivateLink and VPC endpoints natively.
- Required by Fargate.

**Consideration:** `awsvpc` consumes ENIs per task. On EC2, this is bounded by the instance's ENI limit. Use **ENI Trunking** (VPC Trunk Interfaces) or Fargate to overcome this limitation at scale.

---

### 2. How does ECS Service Auto Scaling work, and what are the available scaling policies?

**Answer:**
ECS Service Auto Scaling uses **Application Auto Scaling** to automatically adjust the desired task count of an ECS Service based on metrics or schedules.

**Setup:**
1. Register the ECS service as a scalable target with Application Auto Scaling.
2. Define scaling policies attached to CloudWatch metrics.

**Scaling Policy Types:**

**a) Target Tracking Scaling (recommended):**
- Maintains a specific metric at a target value.
- Example: Keep average CPU utilization at 50%. ECS automatically adds/removes tasks.
- Supports: `ECSServiceAverageCPUUtilization`, `ECSServiceAverageMemoryUtilization`, `ALBRequestCountPerTarget`.

**b) Step Scaling:**
- Scales in/out by a specific number of tasks based on CloudWatch alarm thresholds.
- Allows different step adjustments for different alarm breach magnitudes.
- More control but more configuration.

**c) Scheduled Scaling:**
- Scale tasks up/down at specific times (cron expressions).
- Useful for predictable traffic patterns (e.g., scale up before business hours).

**Cluster Auto Scaling (for EC2 launch type):**
- Uses **Capacity Providers** with **Auto Scaling Groups (ASG)**.
- ECS Cluster Auto Scaling (CAS) manages the ASG to ensure enough EC2 capacity for pending tasks.
- Managed Scaling uses a `CapacityProviderReservation` metric to drive ASG scaling.

**Best Practice:** Combine Target Tracking on ECS tasks + Managed Scaling on the EC2 ASG for a fully automated two-tier scaling system.

---

### 3. What is an ECS Capacity Provider, and how does it differ from a launch type?

**Answer:**
A **Capacity Provider** is an abstraction that links an ECS cluster to a specific compute resource (an EC2 ASG or Fargate). It replaces the concept of specifying a launch type directly on a service or task.

**Key Differences:**

| Aspect | Launch Type | Capacity Provider |
|--------|-------------|-------------------|
| Configuration | Specified per task/service | Defined at cluster level, referenced by services |
| Flexibility | Binary choice (EC2 or Fargate) | Can mix multiple providers with weights |
| Auto Scaling | Manual ASG management | Managed Scaling built-in (for EC2 ASGs) |
| Fargate Spot | Not directly configurable | `FARGATE_SPOT` is a native capacity provider |

**Capacity Provider Strategy:**
You can define a strategy that splits traffic across multiple providers. Example:
```json
[
  { "capacityProvider": "FARGATE", "base": 1, "weight": 1 },
  { "capacityProvider": "FARGATE_SPOT", "base": 0, "weight": 3 }
]
```
This runs 1 guaranteed Fargate task and 3x more tasks on Fargate Spot (up to 70% cheaper, but interruptible).

**Managed Scaling (EC2 ASG):**
When enabled, ECS calculates the `CapacityProviderReservation` metric and signals the ASG to scale. The `targetCapacity` setting (e.g., 100%) determines how aggressively the ASG scales to meet task demand.

---

### 4. How do you pass secrets and sensitive configuration to ECS containers securely?

**Answer:**
Never hardcode secrets in Task Definitions or Docker images. ECS provides two secure mechanisms:

**1. AWS Secrets Manager Integration:**
```json
"secrets": [
  {
    "name": "DB_PASSWORD",
    "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/db/password"
  }
]
```
ECS pulls the secret at task launch time and injects it as an environment variable. Supports versioning and automatic rotation.

**2. AWS Systems Manager Parameter Store:**
```json
"secrets": [
  {
    "name": "API_KEY",
    "valueFrom": "arn:aws:ssm:us-east-1:123456789:parameter/prod/api-key"
  }
]
```
Use SecureString parameters (KMS-encrypted) for sensitive values.

**Required IAM Permissions:**
The **Task Execution Role** (not the Task Role) must have permissions to:
- `secretsmanager:GetSecretValue`
- `ssm:GetParameters`
- `kms:Decrypt` (if using customer-managed KMS keys)

**Additional Best Practices:**
- Use the **Task Role** for application-level AWS API calls; use the **Task Execution Role** only for ECS agent operations (pulling images, writing logs, fetching secrets).
- Rotate secrets using Secrets Manager's built-in rotation; ECS will pick up the new value on the next task launch.
- Use environment-specific secret paths (`/prod/db/password`, `/staging/db/password`) to isolate environments.

---

### 5. Explain ECS rolling deployments and how you can control deployment behavior.

**Answer:**
When you update an ECS Service (e.g., new Task Definition revision), ECS performs a **rolling update** by default.

**Key Deployment Parameters:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `minimumHealthyPercent` | Minimum % of desired tasks that must remain healthy during deployment | 50% (allows half the tasks to be replaced at once) |
| `maximumPercent` | Maximum % of desired tasks that can exist during deployment | 200% (allows double the tasks temporarily) |

**Example with 4 desired tasks:**
- `minimumHealthyPercent: 50`, `maximumPercent: 200`
- ECS can launch 4 new tasks (up to 8 total) before terminating old ones.
- Faster deployment but higher temporary cost.

**Deployment Circuit Breaker:**
ECS can automatically roll back a failed deployment:
```json
"deploymentConfiguration": {
  "deploymentCircuitBreaker": {
    "enable": true,
    "rollback": true
  }
}
```
If a specified number of tasks fail to reach a RUNNING state, ECS automatically rolls back to the last stable revision.

**Blue/Green Deployments with CodeDeploy:**
- Uses `CODE_DEPLOY` deployment controller instead of `ECS`.
- Creates a new "green" task set alongside the existing "blue" task set.
- Supports traffic shifting: **Canary** (e.g., 10% then 90%), **Linear** (e.g., 10% every 1 minute), or **All-at-once**.
- CodeDeploy manages ALB listener rule weights between target groups.
- Provides a configurable rollback window before terminating the blue environment.

---

## Hard

### 1. Deep dive: How does the ECS scheduler place tasks on EC2 container instances, and how can you influence placement?

**Answer:**
The ECS scheduler is responsible for determining which EC2 Container Instance a task runs on. It operates in two phases:

**Phase 1 — Filtering (Constraints):**
The scheduler filters out instances that don't meet hard requirements:
- Sufficient CPU/memory
- Required port availability (bridge mode)
- Attribute matching (`distinctInstance`, custom attributes)
- `placementConstraints` in the Task Definition or Service

**Constraint Types:**
```json
"placementConstraints": [
  { "type": "distinctInstance" },
  { "type": "memberOf", "expression": "attribute:ecs.instance-type =~ t3.*" }
]
```
- `distinctInstance`: Each task on a different instance (high availability).
- `memberOf`: Filter by instance attributes using the ECS cluster query language.

**Phase 2 — Scoring (Strategies):**
After filtering, remaining instances are ranked by placement strategies:

| Strategy | Behavior | Best For |
|----------|----------|----------|
| `spread` | Distribute tasks evenly across AZs or instances | High availability |
| `binpack` | Pack tasks tightly on fewest instances (by CPU or memory) | Cost optimization |
| `random` | Random placement | Testing; no specific preference |

**Combining Strategies (ordered, first match wins):**
```json
"placementStrategy": [
  { "type": "spread", "field": "attribute:ecs.availability-zone" },
  { "type": "binpack", "field": "memory" }
]
```
This first spreads across AZs, then within each AZ, packs tasks tightly — balancing HA with cost efficiency.

**Custom Instance Attributes:**
You can add custom attributes to EC2 instances (e.g., `gpu=true`, `tier=premium`) and use them in placement constraints:
```bash
aws ecs put-attributes --cluster my-cluster \
  --attributes name=gpu,value=true,targetType=container-instance,targetId=<instance-arn>
```

**Fargate Note:** Placement strategies and constraints are managed by AWS on Fargate; you only control AZ spreading via `awsvpc` subnet configuration.

---

### 2. Explain ECS Task IAM Roles vs. Task Execution Roles — how do they work technically, and what are common misconfigurations?

**Answer:**
These are two distinct IAM roles with fundamentally different purposes:

**Task Execution Role (`executionRoleArn`):**
- Used by the **ECS agent and Fargate agent** (not your application code).
- Required for:
  - Pulling images from ECR (`ecr:GetAuthorizationToken`, `ecr:BatchGetImage`)
  - Writing logs to CloudWatch Logs (`logs:CreateLogStream`, `logs:PutLogEvents`)
  - Fetching secrets from Secrets Manager or SSM Parameter Store at task launch
  - Pulling encrypted images (KMS decrypt)
- Attached to the **ECS agent process**, not the container.

**Task Role (`taskRoleArn`):**
- Used by **application code running inside the container**.
- Provides AWS API permissions to your application (e.g., S3, DynamoDB, SQS).
- Works via the **ECS Task Metadata Endpoint** — a local HTTP endpoint that provides temporary credentials.
- The container's AWS SDK automatically calls `http://169.254.170.2/v2/credentials/<UUID>` to retrieve credentials.
- Credentials are rotated automatically by ECS.

**Technical Flow (awsvpc mode):**
```
Container → AWS SDK → ECS Task Metadata Endpoint (169.254.170.2)
         → ECS Agent → STS AssumeRole → Temporary Credentials
         → SDK uses credentials for AWS API calls
```

**Common Misconfigurations:**

| Mistake | Impact | Fix |
|---------|--------|-----|
| Confusing Task Role with Execution Role | Application can't call AWS APIs OR ECS can't pull images | Understand which role serves which purpose |
| Giving `AdministratorAccess` to Task Role | Security blast radius | Apply least privilege; scope to specific resources |
| Missing `kms:Decrypt` on Execution Role | Task fails to start when using customer-managed KMS | Add KMS permissions to Execution Role |
| Not blocking EC2 instance metadata from containers (EC2 mode) | Containers can assume the EC2 instance role (privilege escalation) | Block `169.254.169.254` at container level or use `awsvpc` mode |
| Forgetting `sts:AssumeRole` trust policy | Task Role not assumed | Ensure `ecs-tasks.amazonaws.com` is in the trust policy |

**Security Best Practice — Block Instance Metadata:**
On EC2 launch type with bridge/host networking, containers can access the EC2 instance metadata endpoint and assume the instance profile role. Mitigate with:
```bash
# iptables rule to block container access to instance metadata
iptables --insert FORWARD 1 --in-interface docker+ \
  --destination 169.254.169.254/32 --jump DROP
```
Or preferably, use `awsv