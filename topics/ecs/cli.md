# ECS — AWS CLI Commands

## Setup & Configuration

### Prerequisites

- AWS CLI v2 installed (`aws --version`)
- AWS credentials configured (`aws configure`)
- Docker installed locally for image builds
- `jq` recommended for JSON parsing in scripts

### Required IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:*",
        "ecr:*",
        "iam:PassRole",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "elasticloadbalancing:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### Environment Variables (Recommended)

```bash
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECS_CLUSTER=my-ecs-cluster
export ECS_SERVICE=my-ecs-service
export TASK_FAMILY=my-task-family
```

---

## Core Commands

### 1. Create an ECS Cluster

```bash
aws ecs create-cluster \
  --cluster-name my-ecs-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=1 \
    capacityProvider=FARGATE_SPOT,weight=4,base=0 \
  --settings name=containerInsights,value=enabled \
  --tags key=Environment,value=production key=Team,value=platform \
  --region us-east-1
```

**What it does:** Creates a new ECS cluster with Fargate and Fargate Spot capacity providers, enables Container Insights for monitoring, and applies resource tags.

**Example output:**
```json
{
  "cluster": {
    "clusterArn": "arn:aws:ecs:us-east-1:123456789012:cluster/my-ecs-cluster",
    "clusterName": "my-ecs-cluster",
    "status": "ACTIVE",
    "registeredContainerInstancesCount": 0,
    "runningTasksCount": 0,
    "pendingTasksCount": 0,
    "activeServicesCount": 0,
    "capacityProviders": ["FARGATE", "FARGATE_SPOT"],
    "settings": [
      { "name": "containerInsights", "value": "enabled" }
    ]
  }
}
```

---

### 2. Register a Task Definition

```bash
aws ecs register-task-definition \
  --family my-task-family \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 512 \
  --memory 1024 \
  --execution-role-arn arn:aws:iam::123456789012:role/ecsTaskExecutionRole \
  --task-role-arn arn:aws:iam::123456789012:role/ecsTaskRole \
  --container-definitions '[
    {
      "name": "my-app-container",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "portMappings": [
        { "containerPort": 8080, "protocol": "tcp" }
      ],
      "essential": true,
      "environment": [
        { "name": "APP_ENV", "value": "production" }
      ],
      "secrets": [
        { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:my-db-password" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-task-family",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]' \
  --region us-east-1
```

**What it does:** Registers a new task definition revision with a Fargate-compatible container definition, including secrets injection, CloudWatch logging, and a health check.

---

### 3. Create an ECS Service

```bash
aws ecs create-service \
  --cluster my-ecs-cluster \
  --service-name my-ecs-service \
  --task-definition my-task-family:3 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-0abc12345def67890,subnet-0def98765abc43210],
    securityGroups=[sg-0123456789abcdef0],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-tg/abc123,
    containerName=my-app-container,
    containerPort=8080" \
  --health-check-grace-period-seconds 60 \
  --deployment-configuration "minimumHealthyPercent=100,maximumPercent=200" \
  --enable-execute-command \
  --region us-east-1
```

**What it does:** Creates a Fargate service with 2 desired tasks, attaches it to an ALB target group, enables ECS Exec for debugging, and configures a rolling deployment strategy.

---

### 4. List ECS Clusters

```bash
aws ecs list-clusters \
  --region us-east-1 \
  --output table
```

**Example output:**
```
----------------------------------------------------------
|                      ListClusters                      |
+--------------------------------------------------------+
||                      clusterArns                     ||
|+------------------------------------------------------+|
|| arn:aws:ecs:us-east-1:123456789012:cluster/my-ecs-cluster ||
|| arn:aws:ecs:us-east-1:123456789012:cluster/staging-cluster ||
|+------------------------------------------------------+|
```

---

### 5. Describe a Cluster

```bash
aws ecs describe-clusters \
  --clusters my-ecs-cluster \
  --include ATTACHMENTS CONFIGURATIONS SETTINGS STATISTICS TAGS \
  --region us-east-1
```

**What it does:** Returns detailed information about a cluster including capacity provider settings, Container Insights status, task counts, and tags.

---

### 6. List Services in a Cluster

```bash
aws ecs list-services \
  --cluster my-ecs-cluster \
  --launch-type FARGATE \
  --scheduling-strategy REPLICA \
  --region us-east-1
```

**Example output:**
```json
{
  "serviceArns": [
    "arn:aws:ecs:us-east-1:123456789012:service/my-ecs-cluster/my-ecs-service",
    "arn:aws:ecs:us-east-1:123456789012:service/my-ecs-cluster/my-worker-service"
  ]
}
```

---

### 7. Describe a Service

```bash
aws ecs describe-services \
  --cluster my-ecs-cluster \
  --services my-ecs-service \
  --region us-east-1
```

**What it does:** Returns detailed service configuration including desired/running/pending task counts, deployment status, events, and load balancer configuration.

---

### 8. Update a Service (Rolling Deploy / Scale)

```bash
aws ecs update-service \
  --cluster my-ecs-cluster \
  --service my-ecs-service \
  --task-definition my-task-family:4 \
  --desired-count 4 \
  --deployment-configuration "minimumHealthyPercent=50,maximumPercent=200" \
  --force-new-deployment \
  --region us-east-1
```

**What it does:** Updates the service to use a new task definition revision, scales to 4 tasks, and forces a new deployment even if nothing has changed in the task definition.

---

### 9. List Tasks in a Cluster

```bash
aws ecs list-tasks \
  --cluster my-ecs-cluster \
  --service-name my-ecs-service \
  --desired-status RUNNING \
  --region us-east-1
```

**Example output:**
```json
{
  "taskArns": [
    "arn:aws:ecs:us-east-1:123456789012:task/my-ecs-cluster/a1b2c3d4e5f6",
    "arn:aws:ecs:us-east-1:123456789012:task/my-ecs-cluster/b2c3d4e5f6a1"
  ]
}
```

---

### 10. Describe Running Tasks

```bash
aws ecs describe-tasks \
  --cluster my-ecs-cluster \
  --tasks \
    arn:aws:ecs:us-east-1:123456789012:task/my-ecs-cluster/a1b2c3d4e5f6 \
    arn:aws:ecs:us-east-1:123456789012:task/my-ecs-cluster/b2c3d4e5f6a1 \
  --region us-east-1
```

**What it does:** Returns full task details including container statuses, health check results, network interfaces, started/stopped timestamps, and stop reason.

---

### 11. Run a One-Off Task

```bash
aws ecs run-task \
  --cluster my-ecs-cluster \
  --task-definition my-task-family:4 \
  --launch-type FARGATE \
  --count 1 \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-0abc12345def67890],
    securityGroups=[sg-0123456789abcdef0],
    assignPublicIp=DISABLED
  }" \
  --overrides '{
    "containerOverrides": [
      {
        "name": "my-app-container",
        "command": ["python", "manage.py", "migrate"],
        "environment": [
          { "name": "RUN_MODE", "value": "migration" }
        ]
      }
    ]
  }' \
  --started-by "manual-migration-run" \
  --region us-east-1
```

**What it does:** Launches a one-off Fargate task with a command override — useful for database migrations, batch jobs, or admin scripts.

---

### 12. Stop a Task

```bash
aws ecs stop-task \
  --cluster my-ecs-cluster \
  --task arn:aws:ecs:us-east-1:123456789012:task/my-ecs-cluster/a1b2c3d4e5f6 \
  --reason "Manually stopped for debugging" \
  --region us-east-1
```

**What it does:** Stops a running task immediately. ECS will replace it if the task belongs to a service with a desired count greater than 0.

---

### 13. Delete a Service

```bash
# Scale down to 0 first
aws ecs update-service \
  --cluster my-ecs-cluster \
  --service my-ecs-service \
  --desired-count 0 \
  --region us-east-1

# Then delete
aws ecs delete-service \
  --cluster my-ecs-cluster \
  --service my-ecs-service \
  --force \
  --region us-east-1
```

**What it does:** Safely removes a service by first scaling it to zero, then deleting it. The `--force` flag allows deletion without scaling to zero first.

---

### 14. Delete a Cluster

```bash
aws ecs delete-cluster \
  --cluster my-ecs-cluster \
  --region us-east-1
```

**What it does:** Deletes an ECS cluster. The cluster must have no running tasks, services, or registered container instances before deletion.

---

### 15. List Task Definitions

```bash
aws ecs list-task-definitions \
  --family-prefix my-task-family \
  --status ACTIVE \
  --sort DESC \
  --region us-east-1
```

**Example output:**
```json
{
  "taskDefinitionArns": [
    "arn:aws:ecs:us-east-1:123456789012:task-definition/my-task-family:4",
    "arn:aws:ecs:us-east-1:123456789012:task-definition/my-task-family:3",
    "arn:aws:ecs:us-east-1:123456789012:task-definition/my-task-family:2"
  ]
}
```

---

## Common Operations

### Create

**Create a cluster:**
```bash
aws ecs create-cluster --cluster-name my-ecs-cluster --region us-east-1
```

**Register a task definition from a JSON file:**
```bash
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json \
  --region us-east-1
```

**Create a service:**
```bash
aws ecs create-service \
  --cli-input-json file://service-definition.json \
  --region us-east-1
```

**Create a capacity provider:**
```bash
aws ecs create-capacity-provider \
  --name my-capacity-provider \
  --auto-scaling-group-provider "autoScalingGroupArn=arn:aws:autoscaling:us-east-1:123456789012:autoScalingGroup:abc123:autoScalingGroupName/my-asg,
    managedScaling={status=ENABLED,targetCapacity=80,minimumScalingStepSize=1,maximumScalingStepSize=100},
    managedTerminationProtection=ENABLED" \
  --region us-east-1
```

---

### Read / Describe

**Describe clusters:**
```bash
aws ecs describe-clusters \
  --clusters my-ecs-cluster staging-cluster \
  --include STATISTICS TAGS \
  --region us-east-1
```

**Describe a task definition:**
```bash
aws ecs describe-task-definition \
  --task-definition my-task-family:4 \
  --include TAGS \
  --region us-east-1
```

**Describe container instances (EC2 launch type):**
```bash
aws ecs describe-container-instances \
  --cluster my-ecs-cluster \
  --container-instances \
    arn:aws:ecs:us-east-1:123456789012:container-instance/my-ecs-cluster/abc123 \
  --region us-east-1
```

**Get service events (last 10):**
```bash
aws ecs describe-services \
  --cluster my-ecs-cluster \
  --services my-ecs-service \
  --region us-east-1 \
  --query 'services[0].events[:10]' \
  --output table
```

---

### Update

**Force a new deployment:**
```bash
aws ecs update-service \
  --cluster my-ecs-cluster \
  --service my-ecs-service \
  --force-new-deployment \
  --region us-east-1
```

**Update desired count (scale):**
```bash
aws ecs update-service \
  --cluster my-ecs-cluster \
  --service my-ecs-service \
  --desired-count 6 \
  --region us-east-1
```

**Update cluster settings:**
```bash
aws ecs update-cluster-settings \
  --cluster my-ecs-cluster \
  --settings name=containerInsights