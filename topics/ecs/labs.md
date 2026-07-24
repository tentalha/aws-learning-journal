# ECS — Hands-On Labs

## Lab 1: Getting Started with ECS

### Objective

In this lab, you will deploy a containerized web application using Amazon Elastic Container Service (ECS) with AWS Fargate. You will learn how to create an ECS cluster, define a task definition, and run a container without managing any underlying EC2 infrastructure. By the end of this lab, you will have a running NGINX web server accessible via a public IP address.

### Prerequisites

**AWS Services Required:**
- Amazon ECS (Fargate)
- Amazon ECR (optional — we'll use a public Docker Hub image)
- Amazon VPC (default VPC is acceptable)
- Amazon CloudWatch (for logs)

**IAM Permissions Required:**
- `AmazonECS_FullAccess`
- `AmazonVPC_FullAccess`
- `CloudWatchLogsFullAccess`
- `IAMFullAccess` (to create ECS task execution role)

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- Docker installed (optional for this lab)
- A web browser

**Estimated Cost:** < $0.05 (assuming cleanup within 1 hour)  
**Estimated Duration:** 30–45 minutes

---

### Steps

#### Step 1: Create an ECS Cluster

**Console:**
1. Navigate to the **ECS Console** → [https://console.aws.amazon.com/ecs](https://console.aws.amazon.com/ecs)
2. Click **Clusters** in the left navigation pane.
3. Click **Create cluster**.
4. Enter the cluster name: `lab1-fargate-cluster`
5. Under **Infrastructure**, select **AWS Fargate (serverless)**.
6. Leave all other settings as default.
7. Click **Create**.

**CLI:**
```bash
aws ecs create-cluster \
  --cluster-name lab1-fargate-cluster \
  --capacity-providers FARGATE \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1 \
  --region us-east-1
```

**Verify:**
```bash
aws ecs describe-clusters \
  --clusters lab1-fargate-cluster \
  --region us-east-1 \
  --query "clusters[0].{Name:clusterName,Status:status}"
```

**Expected Output:**
```json
{
    "Name": "lab1-fargate-cluster",
    "Status": "ACTIVE"
}
```

---

#### Step 2: Create the ECS Task Execution IAM Role

This role allows ECS to pull container images and write logs to CloudWatch.

**Console:**
1. Navigate to **IAM Console** → **Roles** → **Create role**.
2. Select **AWS service** → **Elastic Container Service** → **Elastic Container Service Task**.
3. Click **Next**.
4. Search for and attach: `AmazonECSTaskExecutionRolePolicy`.
5. Click **Next**.
6. Role name: `ecsTaskExecutionRole`
7. Click **Create role**.

**CLI:**
```bash
# Create the trust policy document
cat > /tmp/ecs-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the IAM role
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file:///tmp/ecs-trust-policy.json

# Attach the managed policy
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

**Verify:**
```bash
aws iam get-role \
  --role-name ecsTaskExecutionRole \
  --query "Role.{RoleName:RoleName,Arn:Arn}"
```

**Expected Output:**
```json
{
    "RoleName": "ecsTaskExecutionRole",
    "Arn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole"
}
```

> **Note:** If the role `ecsTaskExecutionRole` already exists in your account, skip this step.

---

#### Step 3: Create a CloudWatch Log Group

**Console:**
1. Navigate to **CloudWatch** → **Log groups** → **Create log group**.
2. Log group name: `/ecs/lab1-nginx`
3. Retention: **7 days**
4. Click **Create**.

**CLI:**
```bash
aws logs create-log-group \
  --log-group-name /ecs/lab1-nginx \
  --region us-east-1

aws logs put-retention-policy \
  --log-group-name /ecs/lab1-nginx \
  --retention-in-days 7 \
  --region us-east-1
```

**Verify:**
```bash
aws logs describe-log-groups \
  --log-group-name-prefix /ecs/lab1-nginx \
  --query "logGroups[0].{Name:logGroupName,Retention:retentionInDays}"
```

**Expected Output:**
```json
{
    "Name": "/ecs/lab1-nginx",
    "Retention": 7
}
```

---

#### Step 4: Register a Task Definition

**Console:**
1. In the ECS Console, click **Task definitions** → **Create new task definition**.
2. Task definition family: `lab1-nginx-task`
3. Launch type: **AWS Fargate**
4. Operating system: **Linux/X86_64**
5. CPU: **0.25 vCPU**, Memory: **0.5 GB**
6. Task execution role: `ecsTaskExecutionRole`
7. Under **Container**, click **Add container**:
   - Container name: `nginx`
   - Image URI: `nginx:latest`
   - Port mappings: `80` (TCP)
8. Expand **Logging** → Enable **Use log collection** → select **Amazon CloudWatch**:
   - Log group: `/ecs/lab1-nginx`
   - Region: `us-east-1`
   - Stream prefix: `nginx`
9. Click **Create**.

**CLI:**
```bash
# Get your account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create task definition JSON
cat > /tmp/lab1-task-def.json << EOF
{
  "family": "lab1-nginx-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "nginx:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/lab1-nginx",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "nginx"
        }
      }
    }
  ]
}
EOF

# Register the task definition
aws ecs register-task-definition \
  --cli-input-json file:///tmp/lab1-task-def.json \
  --region us-east-1
```

**Verify:**
```bash
aws ecs describe-task-definition \
  --task-definition lab1-nginx-task \
  --region us-east-1 \
  --query "taskDefinition.{Family:family,CPU:cpu,Memory:memory,Status:status}"
```

**Expected Output:**
```json
{
    "Family": "lab1-nginx-task",
    "CPU": "256",
    "Memory": "512",
    "Status": "ACTIVE"
}
```

---

#### Step 5: Create a Security Group for the Task

**Console:**
1. Navigate to **VPC Console** → **Security Groups** → **Create security group**.
2. Name: `lab1-ecs-sg`
3. Description: `Allow HTTP inbound for ECS lab1`
4. VPC: Select your **default VPC**
5. Inbound rules: Add rule → Type: **HTTP**, Source: **Anywhere-IPv4 (0.0.0.0/0)**
6. Click **Create security group**.

**CLI:**
```bash
# Get default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text \
  --region us-east-1)

echo "Default VPC: $VPC_ID"

# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name lab1-ecs-sg \
  --description "Allow HTTP inbound for ECS lab1" \
  --vpc-id $VPC_ID \
  --region us-east-1 \
  --query "GroupId" \
  --output text)

echo "Security Group ID: $SG_ID"

# Add inbound HTTP rule
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region us-east-1
```

**Verify:**
```bash
aws ec2 describe-security-groups \
  --group-ids $SG_ID \
  --query "SecurityGroups[0].{Name:GroupName,VPC:VpcId}" \
  --region us-east-1
```

**Expected Output:**
```json
{
    "Name": "lab1-ecs-sg",
    "VPC": "vpc-xxxxxxxxxxxxxxxxx"
}
```

---

#### Step 6: Run a Standalone ECS Task

**Console:**
1. Go to **ECS** → **Clusters** → `lab1-fargate-cluster`.
2. Click the **Tasks** tab → **Run new task**.
3. Launch type: **Fargate**
4. Task definition: `lab1-nginx-task` (latest revision)
5. Cluster: `lab1-fargate-cluster`
6. Under **Networking**:
   - VPC: Default VPC
   - Subnets: Select **at least one public subnet**
   - Security group: `lab1-ecs-sg`
   - Auto-assign public IP: **ENABLED**
7. Click **Create**.

**CLI:**
```bash
# Get a public subnet from the default VPC
SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
            "Name=map-public-ip-on-launch,Values=true" \
  --query "Subnets[0].SubnetId" \
  --output text \
  --region us-east-1)

echo "Subnet ID: $SUBNET_ID"

# Run the task
TASK_ARN=$(aws ecs run-task \
  --cluster lab1-fargate-cluster \
  --task-definition lab1-nginx-task \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SG_ID],assignPublicIp=ENABLED}" \
  --region us-east-1 \
  --query "tasks[0].taskArn" \
  --output text)

echo "Task ARN: $TASK_ARN"
```

**Verify — Wait for task to reach RUNNING state:**
```bash
# Poll until the task is running (may take 30-60 seconds)
aws ecs wait tasks-running \
  --cluster lab1-fargate-cluster \
  --tasks $TASK_ARN \
  --region us-east-1

echo "Task is now RUNNING!"

# Get the public IP
aws ecs describe-tasks \
  --cluster lab1-fargate-cluster \
  --tasks $TASK_ARN \
  --region us-east-1 \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" \
  --output text
```

```bash
# Get public IP from the ENI
ENI_ID=$(aws ecs describe-tasks \
  --cluster lab1-fargate-cluster \
  --tasks $TASK_ARN \
  --region us-east-1 \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" \
  --output text)

PUBLIC_IP=$(aws ec2 describe-network-interfaces \
  --network-interface-ids $ENI_ID \
  --query "NetworkInterfaces[0].Association.PublicIp" \
  --output text \
  --region us-east-1)

echo "Access your NGINX server at: http://$PUBLIC_IP"
```

**Expected Output:**
```
Task is now RUNNING!
Access your NGINX server at: http://3.91.xxx.xxx
```

Open the URL in your browser — you should see the **NGINX Welcome Page**.

---

### Verification

Run the following checks to confirm successful lab completion:

```bash
# 1. Cluster is ACTIVE
aws ecs describe-clusters \
  --clusters lab1-fargate-cluster \
  --query "clusters[0].status" --output text

# 2. Task definition is ACTIVE
aws ecs describe-task-definition \
  --task-definition lab1-nginx-task \
  --query "taskDefinition.status" --output text

# 3. Task is RUNNING
aws ecs describe-tasks \
  --cluster lab1-fargate-cluster \
  --tasks $TASK_ARN \
  --query "tasks[0].lastStatus" --output text

# 4. NGINX responds
curl -s -o /dev/null -w "%{http_code}" http://$PUBLIC_IP
# Expected: 200

# 5. Logs are appearing in CloudWatch
aws logs describe-log-streams \
  --log-group-name /ecs/lab1-nginx \
  --query "logStreams[*].logStreamName" \
  --output text
```

**✅ Lab 1 is complete when:**
- Cluster status = `ACTIVE`
- Task status = `RUNNING`
- Browser shows NGINX welcome page
- CloudWatch log streams exist under `/ecs/lab1-nginx`

---

### Cleanup

Run these commands in order to avoid ongoing charges:

```bash
# 1. Stop the running task
aws ecs stop-task \
  --cluster lab1-fargate-cluster \
  --task $TASK_ARN \
  --region us-east-1

# 2. Deregister the task definition
TASK_DEF_ARN=$(aws ecs describe-task-definition \
  --task-definition lab1-nginx-task \
  --query "taskDefinition.taskDefinitionArn" \
  --output text)

aws ecs deregister-task-definition \
  --task-definition $TASK_DEF_ARN \
  --region us-east-1

# 3. Delete the ECS cluster
aws ecs delete-cluster \
  --cluster lab1-fargate-cluster \
  --region us-east-1

# 4. Delete the security group
aws ec2 delete-security-group \
  --group-id $SG_ID \
  --region us-east-1

# 5. Delete the CloudWatch log group
aws logs delete-log-group \
  --log-group-name /ecs/lab1-nginx \
  --region us-east-1

# 6. Detach policy and delete IAM role (only if you created it in this lab)
aws iam detach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws