# Fargate — AWS CLI Commands

## Setup & Configuration

AWS Fargate is a serverless compute engine for containers that works with both Amazon ECS and Amazon EKS. There is no dedicated Fargate CLI — you interact with Fargate through the `aws ecs` and `aws eks` CLI namespaces.

### Prerequisites

```bash
# Install or update the AWS CLI
pip install --upgrade awscli

# Configure CLI credentials
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json

# Verify identity
aws sts get-caller-identity
```

### Required IAM Permissions

The following IAM policies are commonly required for Fargate operations:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:*",
        "ecr:*",
        "logs:*",
        "iam:PassRole",
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "elasticloadbalancing:*",
        "servicediscovery:*",
        "application-autoscaling:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### Required IAM Roles

```bash
# Task Execution Role — allows ECS to pull images and write logs
# Attach: arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Task Role — permissions your application code needs at runtime
# Attach: custom policies based on your app's AWS API usage
```

---

## Core Commands

### 1. Create an ECS Cluster for Fargate

```bash
aws ecs create-cluster \
  --cluster-name my-fargate-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
      capacityProvider=FARGATE,weight=1,base=1 \
      capacityProvider=FARGATE_SPOT,weight=3 \
  --settings name=containerInsights,value=enabled \
  --tags key=Environment,value=production key=Team,value=platform \
  --region us-east-1
```

**What it does:** Creates a new ECS cluster configured to use both `FARGATE` (on-demand) and `FARGATE_SPOT` (interruptible, cheaper) capacity providers. Container Insights is enabled for CloudWatch monitoring.

**Example output:**
```json
{
  "cluster": {
    "clusterArn": "arn:aws:ecs:us-east-1:123456789012:cluster/my-fargate-cluster",
    "clusterName": "my-fargate-cluster",
    "status": "ACTIVE",
    "registeredContainerInstancesCount": 0,
    "runningTasksCount": 0,
    "pendingTasksCount": 0,
    "activeServicesCount": 0,
    "capacityProviders": ["FARGATE", "FARGATE_SPOT"]
  }
}
```

---

### 2. Register a Task Definition

```bash
aws ecs register-task-definition \
  --family my-fargate-task \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu "512" \
  --memory "1024" \
  --execution-role-arn arn:aws:iam::123456789012:role/ecsTaskExecutionRole \
  --task-role-arn arn:aws:iam::123456789012:role/my-task-role \
  --container-definitions '[
    {
      "name": "my-app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-fargate-task",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "environment": [
        { "name": "APP_ENV", "value": "production" },
        { "name": "PORT", "value": "8080" }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:my-db-password"
        }
      ],
      "cpu": 512,
      "memory": 1024
    }
  ]' \
  --region us-east-1
```

**What it does:** Registers a new Fargate-compatible task definition. Specifies `awsvpc` network mode (required for Fargate), vCPU/memory at the task level, container image, port mappings, CloudWatch logging, environment variables, and Secrets Manager integration.

---

### 3. Create a Fargate Service

```bash
aws ecs create-service \
  --cluster my-fargate-cluster \
  --service-name my-fargate-service \
  --task-definition my-fargate-task:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-0abc12345,subnet-0def67890],
    securityGroups=[sg-0123456789abcdef0],
    assignPublicIp=ENABLED
  }" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-tg/abc123,
    containerName=my-app,
    containerPort=8080" \
  --health-check-grace-period-seconds 60 \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100" \
  --region us-east-1
```

**What it does:** Creates a Fargate service that maintains 2 running task replicas, attaches them to an ALB target group, and configures rolling deployment settings.

---

### 4. Run a One-Off Fargate Task

```bash
aws ecs run-task \
  --cluster my-fargate-cluster \
  --task-definition my-fargate-task:1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-0abc12345],
    securityGroups=[sg-0123456789abcdef0],
    assignPublicIp=ENABLED
  }" \
  --overrides '{
    "containerOverrides": [
      {
        "name": "my-app",
        "command": ["python", "manage.py", "migrate"],
        "environment": [
          { "name": "RUN_MIGRATIONS", "value": "true" }
        ]
      }
    ]
  }' \
  --count 1 \
  --region us-east-1
```

**What it does:** Launches a single Fargate task with a command override — useful for database migrations, batch jobs, or one-off scripts without a long-running service.

---

### 5. List Running Tasks in a Cluster

```bash
aws ecs list-tasks \
  --cluster my-fargate-cluster \
  --service-name my-fargate-service \
  --desired-status RUNNING \
  --region us-east-1
```

**Example output:**
```json
{
  "taskArns": [
    "arn:aws:ecs:us-east-1:123456789012:task/my-fargate-cluster/a1b2c3d4e5f6",
    "arn:aws:ecs:us-east-1:123456789012:task/my-fargate-cluster/b2c3d4e5f6a7"
  ]
}
```

---

### 6. Describe Tasks (Detailed)

```bash
aws ecs describe-tasks \
  --cluster my-fargate-cluster \
  --tasks \
    arn:aws:ecs:us-east-1:123456789012:task/my-fargate-cluster/a1b2c3d4e5f6 \
    arn:aws:ecs:us-east-1:123456789012:task/my-fargate-cluster/b2c3d4e5f6a7 \
  --region us-east-1
```

**What it does:** Returns full details of specified tasks including health status, network interfaces (private IP, ENI ID), container statuses, and CPU/memory usage.

---

### 7. Update a Service (Rolling Deploy)

```bash
aws ecs update-service \
  --cluster my-fargate-cluster \
  --service my-fargate-service \
  --task-definition my-fargate-task:2 \
  --desired-count 4 \
  --force-new-deployment \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=50" \
  --region us-east-1
```

**What it does:** Triggers a rolling deployment to a new task definition revision, scales to 4 tasks, and forces replacement of all existing tasks even if the task definition hasn't changed.

---

### 8. Describe a Service

```bash
aws ecs describe-services \
  --cluster my-fargate-cluster \
  --services my-fargate-service \
  --region us-east-1
```

**Example output (truncated):**
```json
{
  "services": [
    {
      "serviceName": "my-fargate-service",
      "clusterArn": "arn:aws:ecs:us-east-1:123456789012:cluster/my-fargate-cluster",
      "status": "ACTIVE",
      "desiredCount": 2,
      "runningCount": 2,
      "pendingCount": 0,
      "launchType": "FARGATE",
      "taskDefinition": "arn:aws:ecs:us-east-1:123456789012:task-definition/my-fargate-task:2",
      "deployments": [
        {
          "status": "PRIMARY",
          "taskDefinition": "arn:aws:ecs:us-east-1:123456789012:task-definition/my-fargate-task:2",
          "desiredCount": 2,
          "runningCount": 2
        }
      ]
    }
  ]
}
```

---

### 9. Stop a Running Task

```bash
aws ecs stop-task \
  --cluster my-fargate-cluster \
  --task arn:aws:ecs:us-east-1:123456789012:task/my-fargate-cluster/a1b2c3d4e5f6 \
  --reason "Manual stop for maintenance" \
  --region us-east-1
```

**What it does:** Gracefully stops a specific running task. ECS will automatically replace it if it belongs to a service with a desired count > 0.

---

### 10. Delete a Service

```bash
# First scale down to 0
aws ecs update-service \
  --cluster my-fargate-cluster \
  --service my-fargate-service \
  --desired-count 0 \
  --region us-east-1

# Then delete the service
aws ecs delete-service \
  --cluster my-fargate-cluster \
  --service my-fargate-service \
  --force \
  --region us-east-1
```

**What it does:** Scales the service to zero tasks then deletes it. The `--force` flag allows deletion without scaling to zero first.

---

### 11. List Task Definitions

```bash
aws ecs list-task-definitions \
  --family-prefix my-fargate-task \
  --status ACTIVE \
  --sort DESC \
  --region us-east-1
```

**Example output:**
```json
{
  "taskDefinitionArns": [
    "arn:aws:ecs:us-east-1:123456789012:task-definition/my-fargate-task:3",
    "arn:aws:ecs:us-east-1:123456789012:task-definition/my-fargate-task:2",
    "arn:aws:ecs:us-east-1:123456789012:task-definition/my-fargate-task:1"
  ]
}
```

---

### 12. Deregister a Task Definition

```bash
aws ecs deregister-task-definition \
  --task-definition my-fargate-task:1 \
  --region us-east-1
```

**What it does:** Marks a specific task definition revision as `INACTIVE` so it cannot be used for new deployments. Does not affect currently running tasks.

---

### 13. Execute a Command in a Running Container (ECS Exec)

```bash
# First enable ECS Exec on the service
aws ecs update-service \
  --cluster my-fargate-cluster \
  --service my-fargate-service \
  --enable-execute-command \
  --region us-east-1

# Then exec into a running task
aws ecs execute-command \
  --cluster my-fargate-cluster \
  --task arn:aws:ecs:us-east-1:123456789012:task/my-fargate-cluster/a1b2c3d4e5f6 \
  --container my-app \
  --interactive \
  --command "/bin/sh" \
  --region us-east-1
```

**What it does:** Opens an interactive shell session directly into a running Fargate container — the Fargate equivalent of `docker exec`. Requires the SSM Session Manager plugin installed locally.

---

### 14. Configure Auto Scaling for a Fargate Service

```bash
# Register the scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/my-fargate-cluster/my-fargate-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 1 \
  --max-capacity 10 \
  --region us-east-1

# Create a target tracking scaling policy (CPU-based)
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/my-fargate-cluster/my-fargate-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name my-cpu-scaling-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }' \
  --region us-east-1
```

**What it does:** Registers the ECS service as an auto-scaling target and creates a CPU-based target tracking policy to scale between 1–10 tasks, targeting 70% average CPU utilization.

---

### 15. View CloudWatch Logs for a Fargate Task

```bash
# List log streams for the log group
aws logs describe-log-streams \
  --log-group-name /ecs/my-fargate-task \
  --order-by LastEventTime \
  --descending \
  --max-items 5 \
  --region us-east-1

# Tail recent log events
aws logs get-log-events \
  --log-group-name /ecs/my-fargate-task \
  --log-stream-name ecs/my-app/a1b2c3d4e5f6 \
  --start-from-head \
  --limit 100