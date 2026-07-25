# Fargate — Hands-On Labs

## Lab 1: Getting Started with Fargate

### Objective

In this lab, you will deploy your first containerized application using AWS Fargate on Amazon ECS. You will create an ECS cluster, define a task definition, and run a simple Nginx web server as a Fargate task. By the end, you will understand the core components of Fargate: clusters, task definitions, tasks, and services.

---

### Prerequisites

**AWS Services Required:**
- Amazon ECS (Elastic Container Service)
- Amazon ECR (optional — using public Docker Hub image)
- Amazon VPC (default VPC is sufficient)
- AWS IAM

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:*",
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole",
        "logs:CreateLogGroup",
        "logs:DescribeLogGroups"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- Docker (optional for this lab)
- A web browser

**Region:** `us-east-1` (all examples use this region)

---

### Steps

#### Step 1: Create an ECS Cluster

**Console:**
1. Navigate to **Amazon ECS** in the AWS Console.
2. Click **Clusters** → **Create cluster**.
3. Enter the cluster name: `fargate-lab-cluster`.
4. Under **Infrastructure**, select **AWS Fargate (serverless)**.
5. Leave all other settings as default.
6. Click **Create**.

**CLI:**
```bash
aws ecs create-cluster \
  --cluster-name fargate-lab-cluster \
  --capacity-providers FARGATE \
  --default-capacity-provider-strategy \
      capacityProvider=FARGATE,weight=1 \
  --region us-east-1
```

**✅ Verify:**
```bash
aws ecs describe-clusters \
  --clusters fargate-lab-cluster \
  --region us-east-1 \
  --query "clusters[0].{Name:clusterName,Status:status}"
```

**Expected Output:**
```json
{
    "Name": "fargate-lab-cluster",
    "Status": "ACTIVE"
}
```

---

#### Step 2: Create a CloudWatch Log Group

**Console:**
1. Navigate to **CloudWatch** → **Log groups**.
2. Click **Create log group**.
3. Enter name: `/ecs/fargate-lab-nginx`.
4. Set retention to **7 days**.
5. Click **Create**.

**CLI:**
```bash
aws logs create-log-group \
  --log-group-name /ecs/fargate-lab-nginx \
  --region us-east-1

aws logs put-retention-policy \
  --log-group-name /ecs/fargate-lab-nginx \
  --retention-in-days 7 \
  --region us-east-1
```

**✅ Verify:**
```bash
aws logs describe-log-groups \
  --log-group-name-prefix /ecs/fargate-lab-nginx \
  --region us-east-1 \
  --query "logGroups[0].{Name:logGroupName,Retention:retentionInDays}"
```

**Expected Output:**
```json
{
    "Name": "/ecs/fargate-lab-nginx",
    "Retention": 7
}
```

---

#### Step 3: Create the ECS Task Execution IAM Role

**Console:**
1. Navigate to **IAM** → **Roles** → **Create role**.
2. Select **AWS service** → **Elastic Container Service** → **Elastic Container Service Task**.
3. Attach the policy: `AmazonECSTaskExecutionRolePolicy`.
4. Name the role: `ecsTaskExecutionRole`.
5. Click **Create role**.

**CLI:**
```bash
# Create the trust policy file
cat > ecs-trust-policy.json << 'EOF'
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

# Create the role
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://ecs-trust-policy.json

# Attach the managed policy
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

**✅ Verify:**
```bash
aws iam get-role \
  --role-name ecsTaskExecutionRole \
  --query "Role.{Name:RoleName,Arn:Arn}"
```

**Expected Output:**
```json
{
    "Name": "ecsTaskExecutionRole",
    "Arn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole"
}
```

---

#### Step 4: Register a Task Definition

**Console:**
1. In ECS, click **Task definitions** → **Create new task definition**.
2. Enter family name: `fargate-lab-nginx`.
3. Under **Infrastructure requirements**:
   - Launch type: **AWS Fargate**
   - OS/Architecture: **Linux/X86_64**
   - CPU: **0.25 vCPU**
   - Memory: **0.5 GB**
4. Under **Task roles**:
   - Task execution role: `ecsTaskExecutionRole`
5. Under **Container details**:
   - Name: `nginx`
   - Image URI: `nginx:latest`
   - Container port: `80`
   - Protocol: `TCP`
6. Under **Logging**, enable **CloudWatch** and set log group to `/ecs/fargate-lab-nginx`.
7. Click **Create**.

**CLI:**
```bash
# Get your account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create task definition JSON
cat > fargate-nginx-task.json << EOF
{
  "family": "fargate-lab-nginx",
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
          "awslogs-group": "/ecs/fargate-lab-nginx",
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
  --cli-input-json file://fargate-nginx-task.json \
  --region us-east-1
```

**✅ Verify:**
```bash
aws ecs describe-task-definition \
  --task-definition fargate-lab-nginx \
  --region us-east-1 \
  --query "taskDefinition.{Family:family,Status:status,CPU:cpu,Memory:memory}"
```

**Expected Output:**
```json
{
    "Family": "fargate-lab-nginx",
    "Status": "ACTIVE",
    "CPU": "256",
    "Memory": "512"
}
```

---

#### Step 5: Create a Security Group for the Task

**CLI:**
```bash
# Get the default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text \
  --region us-east-1)

echo "VPC ID: $VPC_ID"

# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name fargate-lab-nginx-sg \
  --description "Security group for Fargate Nginx lab" \
  --vpc-id $VPC_ID \
  --region us-east-1 \
  --query "GroupId" \
  --output text)

echo "Security Group ID: $SG_ID"

# Allow inbound HTTP traffic
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region us-east-1
```

**Console:**
1. Navigate to **VPC** → **Security Groups** → **Create security group**.
2. Name: `fargate-lab-nginx-sg`
3. VPC: Select the default VPC.
4. Add inbound rule: **HTTP (port 80)** from **0.0.0.0/0**.
5. Click **Create**.

**✅ Verify:**
```bash
aws ec2 describe-security-groups \
  --group-ids $SG_ID \
  --region us-east-1 \
  --query "SecurityGroups[0].{Name:GroupName,Id:GroupId}"
```

---

#### Step 6: Run a Fargate Task

**CLI:**
```bash
# Get public subnets from the default VPC
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0:2].SubnetId" \
  --output text \
  --region us-east-1 | tr '\t' ',')

echo "Subnet IDs: $SUBNET_IDS"

# Run the task
TASK_ARN=$(aws ecs run-task \
  --cluster fargate-lab-cluster \
  --task-definition fargate-lab-nginx \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_IDS],securityGroups=[$SG_ID],assignPublicIp=ENABLED}" \
  --region us-east-1 \
  --query "tasks[0].taskArn" \
  --output text)

echo "Task ARN: $TASK_ARN"
```

**Console:**
1. In ECS, click **Clusters** → `fargate-lab-cluster`.
2. Click **Tasks** → **Run new task**.
3. Select **Launch type: FARGATE**.
4. Task definition: `fargate-lab-nginx`.
5. Under **Networking**: select default VPC, select 2 subnets, choose `fargate-lab-nginx-sg`.
6. Enable **Auto-assign public IP**.
7. Click **Create**.

**✅ Verify — Wait for task to reach RUNNING state:**
```bash
# Wait for task to be running (may take 60-90 seconds)
aws ecs wait tasks-running \
  --cluster fargate-lab-cluster \
  --tasks $TASK_ARN \
  --region us-east-1

# Get the task's public IP
TASK_ID=$(echo $TASK_ARN | cut -d'/' -f3)

ENI_ID=$(aws ecs describe-tasks \
  --cluster fargate-lab-cluster \
  --tasks $TASK_ARN \
  --region us-east-1 \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" \
  --output text)

PUBLIC_IP=$(aws ec2 describe-network-interfaces \
  --network-interface-ids $ENI_ID \
  --region us-east-1 \
  --query "NetworkInterfaces[0].Association.PublicIp" \
  --output text)

echo "Task Public IP: $PUBLIC_IP"
echo "Open in browser: http://$PUBLIC_IP"
```

**Expected Result:** Opening `http://<PUBLIC_IP>` in a browser displays the default Nginx welcome page.

---

### Verification

Run the following commands to confirm the lab was completed successfully:

```bash
# 1. Cluster is ACTIVE
aws ecs describe-clusters \
  --clusters fargate-lab-cluster \
  --region us-east-1 \
  --query "clusters[0].status"
# Expected: "ACTIVE"

# 2. Task definition is registered
aws ecs list-task-definitions \
  --family-prefix fargate-lab-nginx \
  --region us-east-1
# Expected: Lists at least one task definition ARN

# 3. Task is RUNNING
aws ecs describe-tasks \
  --cluster fargate-lab-cluster \
  --tasks $TASK_ARN \
  --region us-east-1 \
  --query "tasks[0].lastStatus"
# Expected: "RUNNING"

# 4. Nginx responds
curl -s -o /dev/null -w "%{http_code}" http://$PUBLIC_IP
# Expected: 200
```

---

### Cleanup

> ⚠️ **Important:** Complete all cleanup steps to avoid ongoing charges.

```bash
# Step 1: Stop the running task
aws ecs stop-task \
  --cluster fargate-lab-cluster \
  --task $TASK_ARN \
  --region us-east-1

# Step 2: Deregister the task definition
TASK_DEF_ARN=$(aws ecs list-task-definitions \
  --family-prefix fargate-lab-nginx \
  --region us-east-1 \
  --query "taskDefinitionArns[0]" \
  --output text)

aws ecs deregister-task-definition \
  --task-definition $TASK_DEF_ARN \
  --region us-east-1

# Step 3: Delete the ECS cluster
aws ecs delete-cluster \
  --cluster fargate-lab-cluster \
  --region us-east-1

# Step 4: Delete the security group
aws ec2 delete-security-group \
  --group-id $SG_ID \
  --region us-east-1

# Step 5: Delete the CloudWatch log group
aws logs delete-log-group \
  --log-group-name /ecs/fargate-lab-nginx \
  --region us-east-1

# Step 6: Detach policy and delete IAM role
aws iam detach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

aws iam delete-role \
  --role-name ecsTaskExecutionRole

# Step 7: Clean up local files
rm -f ecs-trust-policy.json fargate-nginx-task.json
```

---

## Lab 2: Intermediate Fargate Configuration

### Objective

In this lab, you will deploy a scalable, load-balanced ECS Service using Fargate behind an Application Load Balancer (ALB). You will configure ECS Service Auto Scaling using target tracking, inject environment variables and secrets from AWS Secrets Manager into containers, and observe task placement and scaling behavior. By the end, you will understand how to run production-like workloads with Fargate services.

---

### Prerequisites

**AWS Services Required:**
- Amazon ECS with Fargate
- Application Load Balancer (ALB)
- AWS Secrets Manager
- AWS Application Auto Scaling
- Amazon CloudWatch
- Amazon VPC

**IAM Permissions Required:**
- All permissions from