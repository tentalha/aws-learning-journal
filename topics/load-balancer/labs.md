# Load Balancer — Hands-On Labs

## Lab 1: Getting Started with Load Balancer

### Objective

In this lab, you will create an **Application Load Balancer (ALB)** and distribute HTTP traffic across two EC2 instances running a simple web application. By the end of this lab, you will understand:

- How to launch and configure EC2 instances as backend targets
- How to create a Target Group and register instances
- How to create an Application Load Balancer and attach a listener
- How traffic is distributed across healthy targets

**Architecture Overview:**

```
Internet → ALB (port 80) → Target Group → EC2 Instance A
                                        → EC2 Instance B
```

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account with billing enabled |
| IAM Permissions | `ec2:*`, `elasticloadbalancing:*` |
| Region | `us-east-1` (N. Virginia) recommended |
| Tools | AWS Management Console, AWS CLI v2 installed and configured |
| Key Pair | An existing EC2 Key Pair (or create one during the lab) |
| VPC | Default VPC with at least 2 public subnets in different AZs |

**Configure AWS CLI (if not already done):**

```bash
aws configure
# AWS Access Key ID: <your-access-key>
# AWS Secret Access Key: <your-secret-key>
# Default region name: us-east-1
# Default output format: json
```

---

### Steps

#### Step 1: Create a Security Group for EC2 Instances

**Console:**
1. Navigate to **EC2 → Security Groups → Create security group**
2. Set **Security group name**: `lab1-ec2-sg`
3. Set **Description**: `Security group for Lab 1 EC2 instances`
4. Select your **Default VPC**
5. Add **Inbound rule**: Type = `HTTP`, Port = `80`, Source = `0.0.0.0/0`
6. Add **Inbound rule**: Type = `SSH`, Port = `22`, Source = `My IP`
7. Click **Create security group**

**CLI:**

```bash
# Get the default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text)

echo "VPC ID: $VPC_ID"

# Create security group for EC2 instances
EC2_SG_ID=$(aws ec2 create-security-group \
  --group-name "lab1-ec2-sg" \
  --description "Security group for Lab 1 EC2 instances" \
  --vpc-id $VPC_ID \
  --query "GroupId" \
  --output text)

echo "EC2 Security Group ID: $EC2_SG_ID"

# Allow HTTP traffic
aws ec2 authorize-security-group-ingress \
  --group-id $EC2_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow SSH from your IP
MY_IP=$(curl -s https://checkip.amazonaws.com)/32
aws ec2 authorize-security-group-ingress \
  --group-id $EC2_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr $MY_IP

echo "SSH allowed from: $MY_IP"
```

**✅ Verify:** Security group `lab1-ec2-sg` appears in the EC2 console with two inbound rules (HTTP and SSH).

---

#### Step 2: Launch Two EC2 Instances

**Console:**
1. Navigate to **EC2 → Instances → Launch instances**
2. Set **Name**: `lab1-web-server-1`
3. Select **Amazon Linux 2023 AMI** (Free tier eligible)
4. Select **t2.micro** instance type
5. Select your **Key Pair**
6. Under **Network settings**, select `lab1-ec2-sg`
7. Expand **Advanced details → User data**, paste the script below
8. Click **Launch instance**
9. Repeat for a second instance named `lab1-web-server-2` (change the hostname in the user data)

**User data for Instance 1:**

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)
cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html>
<head><title>Lab 1 - Web Server 1</title></head>
<body style="background-color:#e8f4f8; font-family:Arial;">
  <h1>🌐 Web Server 1</h1>
  <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
  <p><strong>Availability Zone:</strong> $AZ</p>
  <p><strong>Server:</strong> lab1-web-server-1</p>
</body>
</html>
EOF
```

**User data for Instance 2:**

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)
cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html>
<head><title>Lab 1 - Web Server 2</title></head>
<body style="background-color:#fef9e7; font-family:Arial;">
  <h1>🌐 Web Server 2</h1>
  <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
  <p><strong>Availability Zone:</strong> $AZ</p>
  <p><strong>Server:</strong> lab1-web-server-2</p>
</body>
</html>
EOF
```

**CLI:**

```bash
# Get the latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-*-x86_64" \
    "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

echo "AMI ID: $AMI_ID"

# Get two public subnets in different AZs
SUBNET_IDS=($(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=defaultForAz,Values=true" \
  --query "Subnets[0:2].SubnetId" \
  --output text))

SUBNET_1=${SUBNET_IDS[0]}
SUBNET_2=${SUBNET_IDS[1]}
echo "Subnet 1: $SUBNET_1, Subnet 2: $SUBNET_2"

# User data for instance 1 (base64 encoded)
USER_DATA_1=$(cat <<'EOF' | base64
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "<h1>Web Server 1 | Instance: $INSTANCE_ID | AZ: $AZ</h1>" > /var/www/html/index.html
EOF
)

# Launch Instance 1
INSTANCE_1_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --security-group-ids $EC2_SG_ID \
  --subnet-id $SUBNET_1 \
  --user-data "$USER_DATA_1" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab1-web-server-1}]' \
  --query "Instances[0].InstanceId" \
  --output text)

echo "Instance 1 ID: $INSTANCE_1_ID"

# User data for instance 2
USER_DATA_2=$(cat <<'EOF' | base64
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "<h1>Web Server 2 | Instance: $INSTANCE_ID | AZ: $AZ</h1>" > /var/www/html/index.html
EOF
)

# Launch Instance 2
INSTANCE_2_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --security-group-ids $EC2_SG_ID \
  --subnet-id $SUBNET_2 \
  --user-data "$USER_DATA_2" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab1-web-server-2}]' \
  --query "Instances[0].InstanceId" \
  --output text)

echo "Instance 2 ID: $INSTANCE_2_ID"

# Wait for instances to be running
echo "Waiting for instances to be in running state..."
aws ec2 wait instance-running \
  --instance-ids $INSTANCE_1_ID $INSTANCE_2_ID

echo "✅ Both instances are running!"
```

**✅ Verify:** Both instances show **Instance state: Running** and **Status check: 2/2 checks passed** in the EC2 console (may take 2–3 minutes).

---

#### Step 3: Create a Security Group for the ALB

**Console:**
1. Navigate to **EC2 → Security Groups → Create security group**
2. Set **Security group name**: `lab1-alb-sg`
3. Set **Description**: `Security group for Lab 1 ALB`
4. Select your **Default VPC**
5. Add **Inbound rule**: Type = `HTTP`, Port = `80`, Source = `0.0.0.0/0`
6. Click **Create security group**

**CLI:**

```bash
# Create ALB security group
ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name "lab1-alb-sg" \
  --description "Security group for Lab 1 ALB" \
  --vpc-id $VPC_ID \
  --query "GroupId" \
  --output text)

echo "ALB Security Group ID: $ALB_SG_ID"

# Allow HTTP from internet
aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

echo "✅ ALB security group created with HTTP inbound rule"
```

**✅ Verify:** Security group `lab1-alb-sg` exists with HTTP (port 80) inbound from `0.0.0.0/0`.

---

#### Step 4: Create a Target Group

**Console:**
1. Navigate to **EC2 → Target Groups → Create target group**
2. Choose **Instances** as target type
3. Set **Target group name**: `lab1-target-group`
4. Set **Protocol**: `HTTP`, **Port**: `80`
5. Select your **VPC**
6. Under **Health checks**: Protocol = `HTTP`, Path = `/`
7. Click **Next**
8. Select both EC2 instances (`lab1-web-server-1` and `lab1-web-server-2`)
9. Click **Include as pending below**, then **Create target group**

**CLI:**

```bash
# Create target group
TG_ARN=$(aws elbv2 create-target-group \
  --name "lab1-target-group" \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --health-check-protocol HTTP \
  --health-check-path "/" \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text)

echo "Target Group ARN: $TG_ARN"

# Register both instances
aws elbv2 register-targets \
  --target-group-arn $TG_ARN \
  --targets Id=$INSTANCE_1_ID Id=$INSTANCE_2_ID

echo "✅ Instances registered to target group"
```

**✅ Verify:** Target group `lab1-target-group` exists. Both instances show **Registered** status (health checks will show **Healthy** after 1–2 minutes).

---

#### Step 5: Create the Application Load Balancer

**Console:**
1. Navigate to **EC2 → Load Balancers → Create load balancer**
2. Choose **Application Load Balancer** → Click **Create**
3. Set **Load balancer name**: `lab1-alb`
4. Set **Scheme**: `Internet-facing`
5. Set **IP address type**: `IPv4`
6. Under **Network mapping**, select your VPC and select **at least 2 subnets** in different AZs
7. Under **Security groups**, select `lab1-alb-sg`
8. Under **Listeners and routing**:
   - Protocol: `HTTP`, Port: `80`
   - Default action: Forward to `lab1-target-group`
9. Click **Create load balancer**

**CLI:**

```bash
# Get all default subnets for multi-AZ
ALL_SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=defaultForAz,Values=true" \
  --query "Subnets[*].SubnetId" \
  --output text | tr '\t' ' ')

echo "Subnets: $ALL_SUBNET_IDS"

# Create the ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name