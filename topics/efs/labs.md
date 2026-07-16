# EFS — Hands-On Labs

## Lab 1: Getting Started with EFS

### Objective
In this lab, you will create your first Amazon Elastic File System (EFS), mount it on two EC2 instances in different Availability Zones, and verify that files written on one instance are immediately visible on the other. You will learn the fundamentals of EFS file system creation, security group configuration, mount target provisioning, and NFS mounting.

---

### Prerequisites

**AWS Services Required:**
- Amazon EFS
- Amazon EC2 (Amazon Linux 2023 AMI)
- Amazon VPC (default VPC is acceptable)

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticfilesystem:*",
        "ec2:*",
        "iam:CreateRole",
        "iam:AttachRolePolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- SSH client (Terminal on macOS/Linux, PuTTY or Windows Terminal on Windows)
- An existing EC2 key pair, or create one during the lab

**Environment Variables (set these before starting):**
```bash
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export LAB_NAME="efs-lab1"
```

---

### Steps

#### Step 1: Create a Security Group for EFS Mount Targets

EFS mount targets require a security group that allows inbound NFS traffic (port 2049) from your EC2 instances.

**Console:**
1. Navigate to **EC2 → Security Groups → Create security group**
2. Set **Security group name**: `efs-lab1-mount-target-sg`
3. Set **Description**: `Allow NFS inbound for EFS Lab 1`
4. Select your **default VPC**
5. Under **Inbound rules**, click **Add rule**:
   - Type: `NFS`
   - Protocol: `TCP`
   - Port: `2049`
   - Source: `0.0.0.0/0` *(we will tighten this in Lab 3)*
6. Click **Create security group**

**CLI:**
```bash
# Get the default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text \
  --region $AWS_REGION)

echo "VPC ID: $VPC_ID"

# Create the security group for EFS mount targets
EFS_SG_ID=$(aws ec2 create-security-group \
  --group-name "efs-lab1-mount-target-sg" \
  --description "Allow NFS inbound for EFS Lab 1" \
  --vpc-id $VPC_ID \
  --region $AWS_REGION \
  --query "GroupId" \
  --output text)

echo "EFS Security Group ID: $EFS_SG_ID"

# Add inbound NFS rule
aws ec2 authorize-security-group-ingress \
  --group-id $EFS_SG_ID \
  --protocol tcp \
  --port 2049 \
  --cidr 0.0.0.0/0 \
  --region $AWS_REGION
```

**✅ Verify:** The security group `efs-lab1-mount-target-sg` appears in the EC2 console with an inbound rule for TCP port 2049.

---

#### Step 2: Create the EFS File System

**Console:**
1. Navigate to **EFS → Create file system**
2. Click **Customize** (not "Create" — we want to see all options)
3. **General settings:**
   - Name: `efs-lab1-filesystem`
   - Storage class: `Standard`
   - Automatic backups: `Disabled` *(for lab purposes)*
   - Encryption: `Enable encryption at rest` ✓
   - KMS key: `aws/elasticfilesystem` (default)
4. Click **Next**
5. **Network settings:**
   - VPC: select your default VPC
   - For each Availability Zone, remove the default security group and add `efs-lab1-mount-target-sg`
6. Click **Next → Next → Create**

**CLI:**
```bash
# Create the EFS file system
EFS_ID=$(aws efs create-file-system \
  --creation-token "efs-lab1-$(date +%s)" \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=efs-lab1-filesystem Key=Lab,Value=efs-lab1 \
  --region $AWS_REGION \
  --query "FileSystemId" \
  --output text)

echo "EFS File System ID: $EFS_ID"

# Wait for the file system to become available
echo "Waiting for EFS to become available..."
aws efs describe-file-systems \
  --file-system-id $EFS_ID \
  --region $AWS_REGION \
  --query "FileSystems[0].LifeCycleState" \
  --output text

# Poll until available (run this a few times if needed)
watch -n 5 "aws efs describe-file-systems \
  --file-system-id $EFS_ID \
  --region $AWS_REGION \
  --query 'FileSystems[0].LifeCycleState' \
  --output text"
```

**Expected Output:**
```
EFS File System ID: fs-0abc1234def56789
available
```

---

#### Step 3: Create Mount Targets in Each Availability Zone

**Console:**
1. Select your new file system → **Network** tab → **Manage**
2. For each AZ, ensure `efs-lab1-mount-target-sg` is selected
3. Click **Save**

**CLI:**
```bash
# Get all subnet IDs in the default VPC
SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[*].SubnetId" \
  --output text \
  --region $AWS_REGION)

echo "Subnets: $SUBNETS"

# Create a mount target in each subnet
for SUBNET_ID in $SUBNETS; do
  echo "Creating mount target in subnet: $SUBNET_ID"
  aws efs create-mount-target \
    --file-system-id $EFS_ID \
    --subnet-id $SUBNET_ID \
    --security-groups $EFS_SG_ID \
    --region $AWS_REGION
done

# Verify mount targets
aws efs describe-mount-targets \
  --file-system-id $EFS_ID \
  --region $AWS_REGION \
  --query "MountTargets[*].{AZ:AvailabilityZoneName,State:LifeCycleState,IP:IpAddress}" \
  --output table
```

**Expected Output:**
```
----------------------------------------------
|         DescribeMountTargets               |
+---------------+-------------------+---------+
|      AZ       |        IP         |  State  |
+---------------+-------------------+---------+
|  us-east-1a   |  172.31.10.x      | available|
|  us-east-1b   |  172.31.20.x      | available|
|  us-east-1c   |  172.31.30.x      | available|
+---------------+-------------------+---------+
```

---

#### Step 4: Launch Two EC2 Instances in Different AZs

**Console:**
1. Navigate to **EC2 → Launch Instances**
2. Launch **Instance A**:
   - Name: `efs-lab1-instance-a`
   - AMI: Amazon Linux 2023
   - Instance type: `t3.micro`
   - Key pair: select your existing key pair
   - Network: default VPC, subnet in `us-east-1a`
   - Security group: create new `efs-lab1-ec2-sg` allowing SSH (port 22) from your IP
   - Enable auto-assign public IP
3. Launch **Instance B** with identical settings but subnet in `us-east-1b`, name `efs-lab1-instance-b`

**CLI:**
```bash
# Get the latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-2023*-x86_64" \
    "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text \
  --region $AWS_REGION)

echo "AMI ID: $AMI_ID"

# Create EC2 security group
EC2_SG_ID=$(aws ec2 create-security-group \
  --group-name "efs-lab1-ec2-sg" \
  --description "EC2 instances for EFS Lab 1" \
  --vpc-id $VPC_ID \
  --region $AWS_REGION \
  --query "GroupId" \
  --output text)

# Allow SSH from your current IP
MY_IP=$(curl -s https://checkip.amazonaws.com)/32
aws ec2 authorize-security-group-ingress \
  --group-id $EC2_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr $MY_IP \
  --region $AWS_REGION

# Get subnets for us-east-1a and us-east-1b
SUBNET_1A=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=availabilityZone,Values=us-east-1a" \
  --query "Subnets[0].SubnetId" --output text --region $AWS_REGION)

SUBNET_1B=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=availabilityZone,Values=us-east-1b" \
  --query "Subnets[0].SubnetId" --output text --region $AWS_REGION)

# Launch Instance A
INSTANCE_A=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name YOUR_KEY_PAIR_NAME \
  --security-group-ids $EC2_SG_ID \
  --subnet-id $SUBNET_1A \
  --associate-public-ip-address \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=efs-lab1-instance-a},{Key=Lab,Value=efs-lab1}]' \
  --region $AWS_REGION \
  --query "Instances[0].InstanceId" \
  --output text)

# Launch Instance B
INSTANCE_B=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name YOUR_KEY_PAIR_NAME \
  --security-group-ids $EC2_SG_ID \
  --subnet-id $SUBNET_1B \
  --associate-public-ip-address \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=efs-lab1-instance-b},{Key=Lab,Value=efs-lab1}]' \
  --region $AWS_REGION \
  --query "Instances[0].InstanceId" \
  --output text)

echo "Instance A: $INSTANCE_A"
echo "Instance B: $INSTANCE_B"

# Wait for instances to be running
aws ec2 wait instance-running \
  --instance-ids $INSTANCE_A $INSTANCE_B \
  --region $AWS_REGION

echo "Both instances are running!"

# Get public IPs
IP_A=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_A \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text --region $AWS_REGION)

IP_B=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_B \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text --region $AWS_REGION)

echo "Instance A IP: $IP_A"
echo "Instance B IP: $IP_B"
```

> ⚠️ **Replace `YOUR_KEY_PAIR_NAME`** with your actual EC2 key pair name.

---

#### Step 5: Mount EFS on Both Instances

**SSH into Instance A:**
```bash
ssh -i ~/.ssh/your-key.pem ec2-user@$IP_A
```

**On Instance A — install EFS utilities and mount:**
```bash
# Install the amazon-efs-utils package
sudo dnf install -y amazon-efs-utils

# Create a mount point
sudo mkdir -p /mnt/efs

# Get your EFS file system ID (replace with your actual ID)
EFS_ID="fs-0abc1234def56789"

# Mount using EFS mount helper (recommended — handles TLS encryption in transit)
sudo mount -t efs -o tls $EFS_ID:/ /mnt/efs

# Verify the mount
df -h | grep efs
mount | grep efs
```

**Expected Output:**
```
127.0.0.1:/   8.0E   0  8.0E   0% /mnt/efs
```

> 💡 **Note:** EFS reports 8.0 Exabytes because it is a fully elastic, petabyte-scale file system. You are only charged for what you actually store.

**Write a test file on Instance A:**
```bash
echo "Hello from Instance A in us-east-1a — $(date)" | sudo tee /mnt/efs/hello.txt
ls -la /mnt/efs/
cat /mnt/efs/hello.txt
```

---

**SSH into Instance B (open a new terminal):**
```bash
ssh -i ~/.ssh/your-key.pem ec2-user@$IP_B
```

**On Instance B — mount and verify shared access:**
```bash
sudo dnf install -y amazon-efs-utils
sudo mkdir -p /mnt/efs

EFS_ID="fs-0abc1234def56789"
sudo mount -t efs -o tls $EFS_ID:/ /mnt/efs

# Read the file written by Instance A
cat /mnt/efs/hello.txt

# Write from Instance B
echo "Hello from Instance B in us-east-1b — $(date)" | sudo tee /mnt/efs/hello-b.txt
ls -la /mnt/efs/
```

**Back on Instance A — verify Instance B's file is visible:**
```bash
cat /mnt/efs/hello-b.txt
ls -la /mnt/efs/
```

**Expected Output (Instance A):**
```
Hello from Instance B in us-east-1b — Mon Jan 15 10:30:00 UTC 2024
total 8
drwxr-xr-x 2 root root 6144 Jan 15 10:25 .
drwxr-xr-x 3 root root   18 Jan 15 10:20 ..
-rw-r--r-- 1 root root   52 Jan 15 10:25 hello.txt
-rw-r--r-- 1 root root   52 Jan 15 10:30 hello-b.txt
```

---

#### Step 6: Configure Persistent Mount with /etc/fstab

**On both instances:**
```bash
# Add EFS to /etc/fstab for persistent mounting across reboots
EFS_ID="fs-0abc1234def56789"

echo "$EFS_ID:/ /mnt/efs efs defaults,_