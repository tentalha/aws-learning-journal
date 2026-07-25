# Fargate — Interview Questions

## Easy

---

### Q1. What is AWS Fargate, and what problem does it solve?

**Answer:**
AWS Fargate is a serverless compute engine for containers that works with both Amazon ECS (Elastic Container Service) and Amazon EKS (Elastic Kubernetes Service). It eliminates the need to provision, configure, and manage EC2 instances to run containers.

**Problem it solves:** Traditionally, running containers on AWS required you to manage the underlying EC2 instances — patching the OS, right-sizing instance types, managing cluster capacity, and handling scaling of the host fleet. Fargate abstracts away all of that infrastructure, allowing developers to focus purely on defining and running their containerized applications.

---

### Q2. What is the difference between Fargate and EC2 launch types in ECS?

**Answer:**

| Feature | Fargate Launch Type | EC2 Launch Type |
|---|---|---|
| Infrastructure management | AWS manages it | You manage EC2 instances |
| Scaling | Task-level scaling | Instance + task-level scaling |
| Cost model | Pay per vCPU/memory per second | Pay for EC2 instances (running or idle) |
| SSH/access to host | Not possible | Possible via EC2 |
| Isolation | VM-level isolation per task | Container-level on shared host |
| Startup time | Slightly slower (cold start) | Faster if instances already running |

Use Fargate when you want operational simplicity. Use EC2 when you need specific instance types, GPU workloads, or greater cost optimization via Reserved/Spot Instances.

---

### Q3. What are the key resource configuration options when defining a Fargate task?

**Answer:**
When defining a Fargate task, you must specify:

- **CPU:** Defined in CPU units. Valid values: 256 (.25 vCPU), 512, 1024, 2048, 4096, 8192, 16384
- **Memory:** Defined in MiB. Valid ranges depend on the CPU value selected (e.g., 256 CPU → 512 MB to 2 GB memory)
- **Storage:** Each Fargate task gets ephemeral storage. Default is 20 GB; you can configure up to 200 GB (for tasks using platform version 1.4.0+)
- **Networking:** Fargate tasks use `awsvpc` network mode, giving each task its own Elastic Network Interface (ENI)

These are set at the **task definition** level, not the container level (though container-level soft/hard limits can also be set).

---

### Q4. What networking mode does Fargate use, and why is it significant?

**Answer:**
Fargate exclusively uses **`awsvpc` network mode**. This means each Fargate task gets its own **Elastic Network Interface (ENI)** with a private IP address from your VPC subnet.

**Why it's significant:**
- Tasks behave like first-class VPC citizens — you can apply Security Groups directly to tasks
- Tasks can communicate with other AWS services using VPC routing, VPC endpoints, and private DNS
- Each task gets a dedicated IP, simplifying network security policies
- No port mapping conflicts (unlike `bridge` mode on EC2)
- Enables fine-grained network access control at the task level

This is a major security and operational advantage over EC2 launch type with bridge networking.

---

### Q5. How does Fargate pricing work?

**Answer:**
Fargate pricing is based on the **vCPU and memory resources requested** for your task, billed **per second** (with a minimum of 1 minute).

**Pricing components:**
- **vCPU price:** Charged per vCPU per hour
- **Memory price:** Charged per GB per hour
- **Ephemeral storage:** First 20 GB is free; additional storage (up to 200 GB) is charged per GB per hour

**Cost optimization options:**
- **Fargate Spot:** Up to 70% discount compared to on-demand Fargate prices, using spare AWS capacity (tasks can be interrupted)
- **Compute Savings Plans:** Provide up to 50% savings in exchange for a 1 or 3-year commitment

You are only charged while the task is **running**, not when it is stopped or pending.

---

## Medium

---

### Q1. Explain how Fargate integrates with IAM. What is the difference between the Task Execution Role and the Task Role?

**Answer:**
Fargate tasks use **two distinct IAM roles**, each serving a different purpose:

**1. Task Execution Role (`ecsTaskExecutionRole`)**
- Used by the **ECS agent/Fargate infrastructure** on your behalf
- Required for:
  - Pulling container images from Amazon ECR
  - Publishing logs to CloudWatch Logs
  - Fetching secrets from AWS Secrets Manager or SSM Parameter Store (when referenced in task definitions)
- This role is assumed by the ECS service, not your application code

**2. Task Role**
- Used by **your application code running inside the container**
- Grants permissions for your application to interact with AWS services (e.g., S3, DynamoDB, SQS)
- Credentials are delivered via the **ECS Task Metadata endpoint** (similar to EC2 instance metadata)
- Each task can have a different Task Role, enabling fine-grained least-privilege access

**Key distinction:**
```
ECS Infrastructure → uses → Task Execution Role
Your Application Code → uses → Task Role
```

**Best practice:** Never embed AWS credentials in your container images. Always use the Task Role for application-level AWS API calls.

---

### Q2. How does auto scaling work with Fargate on ECS? What scaling mechanisms are available?

**Answer:**
Fargate on ECS supports **Application Auto Scaling** at the service level. There are three primary scaling mechanisms:

**1. Target Tracking Scaling**
- Automatically adjusts the number of tasks to maintain a target metric value
- Common metrics: `ECSServiceAverageCPUUtilization`, `ECSServiceAverageMemoryUtilization`, `ALBRequestCountPerTarget`
- Simplest to configure; AWS manages scale-in/out logic

**2. Step Scaling**
- Scales based on CloudWatch alarm thresholds with configurable step adjustments
- Provides more granular control (e.g., add 2 tasks if CPU > 70%, add 5 tasks if CPU > 90%)
- Requires manual CloudWatch alarm configuration

**3. Scheduled Scaling**
- Scales tasks based on a cron schedule
- Useful for predictable load patterns (e.g., scale up before business hours)

**Key considerations:**
- Fargate tasks have a **cold start time** (typically 30–90 seconds), so scaling policies should account for warm-up periods
- Scale-in protection can be enabled per task to prevent in-flight work from being interrupted
- **Minimum/Maximum task counts** must be set to bound scaling behavior
- Fargate Spot tasks can be interrupted, so mix On-Demand and Spot with capacity provider strategies for resilience

---

### Q3. What is a Fargate capacity provider and how does it work with capacity provider strategies?

**Answer:**
**Capacity providers** are a mechanism that links ECS services to the underlying compute infrastructure. For Fargate, there are two built-in capacity providers:

- **`FARGATE`** — On-demand Fargate tasks
- **`FARGATE_SPOT`** — Spot Fargate tasks (interruptible, up to 70% cheaper)

**Capacity Provider Strategy** allows you to define how tasks are distributed across capacity providers:

```json
{
  "capacityProviderStrategy": [
    {
      "capacityProvider": "FARGATE",
      "base": 1,
      "weight": 1
    },
    {
      "capacityProvider": "FARGATE_SPOT",
      "weight": 4
    }
  ]
}
```

**Parameters:**
- **`base`:** Minimum number of tasks to run on this provider (only one provider can have a base value)
- **`weight`:** Relative proportion of tasks to run on each provider. In the example above, 1 in 5 tasks runs on FARGATE, 4 in 5 on FARGATE_SPOT

**Use case:** Run 1 guaranteed On-Demand task (base) for availability, and scale remaining tasks on Spot to reduce costs by ~70%.

**Spot interruption handling:** Fargate Spot tasks receive a **2-minute warning** before interruption via the ECS Task Metadata endpoint and EventBridge events, allowing graceful shutdown.

---

### Q4. How do you manage secrets and sensitive configuration in Fargate tasks?

**Answer:**
Fargate provides several secure mechanisms for injecting secrets into containers without hardcoding them:

**Option 1: AWS Secrets Manager**
```json
{
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/db/password"
    }
  ]
}
```
- Secrets are fetched at task launch and injected as environment variables
- Supports versioning and automatic rotation
- Can reference specific JSON keys: `arn:...:secret:name:json-key::`

**Option 2: AWS Systems Manager Parameter Store**
```json
{
  "secrets": [
    {
      "name": "API_KEY",
      "valueFrom": "arn:aws:ssm:us-east-1:123456789:parameter/prod/api-key"
    }
  ]
}
```
- SecureString parameters are decrypted using KMS at task launch
- Lower cost than Secrets Manager for simple key-value pairs

**Option 3: Environment Variables (not recommended for sensitive data)**
- Visible in task definition, CloudTrail, and potentially logs

**Security requirements:**
- The **Task Execution Role** must have permissions to access Secrets Manager/SSM and the KMS key used for encryption
- Secrets are fetched once at task start — if a secret rotates, tasks must be restarted to pick up new values
- For dynamic secret rotation, consider fetching secrets at runtime using the AWS SDK within the application

---

### Q5. How does logging work with Fargate, and what log drivers are supported?

**Answer:**
Fargate supports several **log drivers** configured in the task definition's `logConfiguration` section:

**Supported Log Drivers:**
| Driver | Destination |
|---|---|
| `awslogs` | Amazon CloudWatch Logs |
| `splunk` | Splunk |
| `awsfirelens` | Flexible routing via Fluent Bit/Fluentd |
| `fluentd` | Fluentd endpoint |
| `gelf` | Graylog Extended Log Format |
| `json-file` | Local file (limited use in Fargate) |
| `syslog` | Syslog endpoint |

**Most common: `awslogs`**
```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/my-app",
      "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "ecs"
    }
  }
}
```

**Advanced: AWS FireLens**
- Uses a **sidecar container** running Fluent Bit or Fluentd
- Routes logs to multiple destinations simultaneously (CloudWatch, S3, Kinesis, Elasticsearch, Datadog, etc.)
- Supports log filtering, enrichment, and transformation
- Requires a `firelens` container type in the task definition

**Important notes:**
- The Task Execution Role must have `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` permissions
- Fargate does **not** support the `journald` log driver
- Log streams are automatically created per task instance

---

## Hard

---

### Q1. Explain the Fargate platform versions and the significant changes introduced in platform version 1.4.0. When would you choose an older platform version?

**Answer:**
Fargate platform versions (PV) define the runtime environment for tasks, including the container runtime, kernel version, and available features. AWS periodically releases new platform versions with improvements.

**Key Platform Versions (ECS on Fargate):**

**Platform Version 1.3.0 (older)**
- 10 GB ephemeral storage
- No task-level ephemeral storage configuration
- Task ENI with limited networking features
- Secrets injection supported

**Platform Version 1.4.0 (significant release)**
Major changes:
- **Ephemeral storage:** Increased from 10 GB to 20 GB default, configurable up to 200 GB
- **Unified container runtime:** Moved from Docker daemon to **containerd** (via Firecracker microVMs)
- **Task-level ephemeral storage:** Shared across all containers in the task
- **EFS (Elastic File System) support:** Persistent storage via EFS volumes
- **Task metadata endpoint v4:** Richer metadata including network stats, container stats
- **AWS Distro for OpenTelemetry (ADOT) sidecar support**
- **Improved networking:** Better ENI management, faster task startup

**LATEST platform version:**
- AWS recommends using `LATEST` which maps to the most current stable version
- For Linux, `LATEST` currently maps to 1.4.0

**When to use an older PV:**
- Legacy applications relying on Docker-specific socket behaviors
- Applications using Docker daemon socket (`/var/run/docker.sock`) — not available in 1.4.0
- Compatibility issues with containerd runtime (rare but possible with certain base images)
- Debugging/rollback scenarios when `LATEST` introduces a regression

**EKS Fargate:**
- Uses Kubernetes version-aligned platform versions (e.g., `eks.1`, `eks.2`)
- Each EKS Fargate PV corresponds to a Kubernetes minor version
- AWS manages patching and updates of the underlying Fargate infrastructure

---

### Q2. Deep dive into Fargate networking: How does VPC networking, ENI trunking, and task density interact? What are the limitations?

**Answer:**
Fargate networking is more complex than it appears. Understanding ENI management is critical for production deployments.

**Standard ENI Allocation:**
- Each Fargate task gets a dedicated ENI (Elastic Network Interface)
- ENIs consume IP addresses from your VPC subnet
- **Problem:** EC2 instances have ENI limits. In ECS on EC2, ENI trunking helps, but in Fargate, the constraint is at the **subnet IP address level**

**ENI Trunking (ECS on EC2, not Fargate directly):**
- For EC2 launch type, ECS supports ENI trunking where a single "trunk" ENI can carry traffic for multiple tasks
- This doesn't apply to Fargate directly — each Fargate task always gets its own ENI

**Fargate-specific networking constraints:**

1. **Subnet IP exhaustion:**
   - Each task consumes one private IP from the subnet
   - A /24 subnet provides 251 usable IPs → maximum ~251 concurrent tasks
   - Plan subnet sizing carefully for large-scale deployments
   - **Solution:** Use larger subnets (/20 or /16) or multiple subnets across AZs

2. **Security Groups:**
   - Applied at the task ENI level (not container level)
   - All containers in a task share the same Security Group(s)
   - Up to 5 Security Groups per task

3. **Public IP assignment:**
   - Tasks in public subnets can be assigned a public IP
   - Tasks in private subnets need NAT Gateway for outbound internet access
   - For ECR image pulls in private subnets: use **VPC endpoints** for ECR, S3 (for image layers), and CloudWatch Logs to avoid NAT costs

4. **Inter-task communication:**
   - Tasks communicate via their private IPs
   - Use **Service Discovery (Cloud Map)** or a **load balancer** for service-to-service communication
   - Tasks in the same task definition share `localhost` networking

5. **IPv6 support:**
   - Fargate supports dual-stack (IPv4/IPv6) networking in VPCs configured for dual-stack

**VPC Endpoint strategy for cost optimization:**
```
Private Subnet → VPC Endpoint (ECR API) → ECR
Private Subnet → VPC Endpoint (ECR DKR) → ECR Docker Registry
Private Subnet → VPC Endpoint (S3 Gateway) → S3 (ECR layers)
Private Subnet → VPC Endpoint (CloudWatch Logs) → CloudWatch
```
This avoids NAT Gateway data processing charges (~$0.045/GB) for AWS API calls.

---

### Q3. How does Fargate achieve task isolation? Explain the underlying security architecture including Firecracker microVMs.

**Answer:**
Far