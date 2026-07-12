# EC2 — Hands-On Labs

## Lab 1: Getting Started with EC2

### Objective
In this lab, you will launch your first Amazon EC2 instance, connect to it via SSH, install a simple web server, and verify it is accessible from the internet. By the end of this lab, you will understand the core concepts of EC2 including instance types, key pairs, security groups, and Elastic IPs.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active AWS account with billing enabled |
| IAM Permissions | `AmazonEC2FullAccess` or equivalent |
| Tools | AWS CLI v2 installed and configured (`aws configure`) |
| Region | `us-east-1` (N. Virginia) recommended |
| SSH Client | Terminal (macOS/Linux) or PuTTY / Windows Terminal (Windows) |

**Verify CLI is configured:**
```bash
aws sts get-caller-identity
```
Expected output:
```json
{
    "UserId": "AIDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

---

### Steps

#### Step 1: Create a Key Pair

A key pair is required to securely SSH into your EC2 instance.

**Console:**
1. Navigate to **EC2 → Network & Security → Key Pairs**
2. Click **Create key pair**
3. Set the following:
   - **Name:** `lab1-keypair`
   - **Key pair type:** RSA
   - **Private key file format:** `.pem` (macOS/Linux) or `.ppk` (Windows/PuTTY)
4. Click **Create key pair** — the file downloads automatically
5. Move the file to a safe location and restrict permissions

**CLI:**
```bash
# Create key pair and save private key
aws ec2 create-key-pair \
  --key-name lab1-keypair \
  --query 'KeyMaterial' \
  --output text > ~/lab1-keypair.pem

# Restrict permissions (macOS/Linux only)
chmod 400 ~/lab1-keypair.pem
```

**Verify:**
```bash
aws ec2 describe-key-pairs --key-names lab1-keypair
```
Expected output:
```json
{
    "KeyPairs": [
        {
            "KeyPairId": "key-0abc123...",
            "KeyName": "lab1-keypair",
            "KeyFingerprint": "..."
        }
    ]
}
```

---

#### Step 2: Create a Security Group

A security group acts as a virtual firewall controlling inbound and outbound traffic.

**Console:**
1. Navigate to **EC2 → Network & Security → Security Groups**
2. Click **Create security group**
3. Configure:
   - **Name:** `lab1-web-sg`
   - **Description:** `Security group for Lab 1 web server`
   - **VPC:** Select the default VPC
4. Under **Inbound rules**, add:
   - Type: `SSH`, Protocol: `TCP`, Port: `22`, Source: `My IP`
   - Type: `HTTP`, Protocol: `TCP`, Port: `80`, Source: `Anywhere (0.0.0.0/0)`
5. Click **Create security group**

**CLI:**
```bash
# Get default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)

echo "Default VPC: $VPC_ID"

# Get your current public IP
MY_IP=$(curl -s https://checkip.amazonaws.com)/32
echo "Your IP: $MY_IP"

# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name lab1-web-sg \
  --description "Security group for Lab 1 web server" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "Security Group ID: $SG_ID"

# Add SSH rule (your IP only)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr $MY_IP

# Add HTTP rule (public)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

echo "Security group configured successfully."
```

**Verify:**
```bash
aws ec2 describe-security-groups --group-ids $SG_ID \
  --query 'SecurityGroups[0].IpPermissions'
```

---

#### Step 3: Launch an EC2 Instance

**Console:**
1. Navigate to **EC2 → Instances → Launch Instances**
2. Configure:
   - **Name:** `lab1-web-server`
   - **AMI:** Amazon Linux 2023 (free tier eligible)
   - **Instance type:** `t2.micro` or `t3.micro` (free tier eligible)
   - **Key pair:** `lab1-keypair`
3. Under **Network settings**, select:
   - **VPC:** Default VPC
   - **Auto-assign Public IP:** Enable
   - **Security group:** Select existing → `lab1-web-sg`
4. Under **Advanced details → User data**, paste:
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from EC2 Lab 1! Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</h1>" > /var/www/html/index.html
```
5. Click **Launch instance**

**CLI:**
```bash
# Get the latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-*-x86_64" \
    "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

echo "AMI ID: $AMI_ID"

# User data script (base64 encoded)
USER_DATA=$(cat <<'EOF'
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from EC2 Lab 1! Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</h1>" > /var/www/html/index.html
EOF
)

# Launch the instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name lab1-keypair \
  --security-group-ids $SG_ID \
  --associate-public-ip-address \
  --user-data "$USER_DATA" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab1-web-server},{Key=Lab,Value=EC2-Lab1}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance ID: $INSTANCE_ID"
```

**Wait for instance to be running:**
```bash
echo "Waiting for instance to enter 'running' state..."
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
echo "Instance is now running!"
```

**Verify:**
```bash
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].{State:State.Name,PublicIP:PublicIpAddress,InstanceType:InstanceType}' \
  --output table
```
Expected output:
```
----------------------------------------------
|           DescribeInstances                |
+-------------+---------------+-------------+
| InstanceType|   PublicIP    |    State    |
+-------------+---------------+-------------+
|  t3.micro   | 54.x.x.x      |   running   |
+-------------+---------------+-------------+
```

---

#### Step 4: Connect via SSH

**Get the public IP:**
```bash
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "Public IP: $PUBLIC_IP"
```

**Connect via SSH:**
```bash
ssh -i ~/lab1-keypair.pem ec2-user@$PUBLIC_IP
```

> **Windows users:** Use PuTTY or Windows Terminal with the `.ppk` file, or use the EC2 Instance Connect feature in the Console.

**Console (EC2 Instance Connect):**
1. Select your instance in the EC2 console
2. Click **Connect → EC2 Instance Connect → Connect**

**Verify once connected:**
```bash
# Check web server status
sudo systemctl status httpd

# Check the web page content
curl http://localhost
```

Expected output:
```html
<h1>Hello from EC2 Lab 1! Instance ID: i-0abc123...</h1>
```

---

#### Step 5: Test Web Server Access

**From your local machine:**
```bash
curl http://$PUBLIC_IP
```

Or open a browser and navigate to `http://<your-public-ip>`

**Expected result:**
```
Hello from EC2 Lab 1! Instance ID: i-0abc1234567890def
```

---

### Verification

Run the following checklist to confirm successful lab completion:

```bash
# 1. Verify key pair exists
aws ec2 describe-key-pairs --key-names lab1-keypair --query 'KeyPairs[0].KeyName'

# 2. Verify security group rules
aws ec2 describe-security-groups --group-ids $SG_ID \
  --query 'SecurityGroups[0].IpPermissions[*].{Port:FromPort,Protocol:IpProtocol}'

# 3. Verify instance is running
aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].State.Name'

# 4. Verify HTTP response
curl -s -o /dev/null -w "%{http_code}" http://$PUBLIC_IP
# Expected: 200
```

✅ **Lab 1 is complete when:**
- [ ] Key pair created and `.pem` file secured
- [ ] Security group allows SSH from your IP and HTTP from anywhere
- [ ] EC2 instance is in `running` state
- [ ] SSH connection successful
- [ ] Web server returns HTTP 200 with instance ID in body

---

### Cleanup

> ⚠️ **Always clean up resources to avoid unexpected AWS charges.**

**Console:**
1. Navigate to **EC2 → Instances**
2. Select `lab1-web-server` → **Instance state → Terminate instance**
3. Navigate to **Security Groups** → Select `lab1-web-sg` → **Actions → Delete security group**
4. Navigate to **Key Pairs** → Select `lab1-keypair` → **Actions → Delete**

**CLI:**
```bash
# 1. Terminate the EC2 instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
echo "Terminating instance..."

# 2. Wait for termination
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID
echo "Instance terminated."

# 3. Delete security group (wait a moment after instance terminates)
aws ec2 delete-security-group --group-id $SG_ID
echo "Security group deleted."

# 4. Delete key pair from AWS (also delete local .pem file)
aws ec2 delete-key-pair --key-name lab1-keypair
rm -f ~/lab1-keypair.pem
echo "Key pair deleted."

echo "Cleanup complete!"
```

**Verify cleanup:**
```bash
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].State.Name'
# Expected: "terminated"
```

---

## Lab 2: Intermediate EC2 Configuration

### Objective
In this lab, you will configure a more production-like EC2 environment featuring an IAM Instance Profile, Elastic IP, EBS volume management, CloudWatch monitoring with custom metrics, and Systems Manager Session Manager for keyless access. You will also configure instance metadata and explore EC2 User Data for bootstrapping configuration management.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account with admin or power user access |
| IAM Permissions | `EC2FullAccess`, `IAMFullAccess`, `CloudWatchFullAccess`, `SSMFullAccess` |
| Tools | AWS CLI v2, `jq` installed |
| Prior Knowledge | Completion of Lab 1 or equivalent experience |
| Region | `us-east-1` |

**Install jq (if not already installed):**
```bash
# macOS
brew install jq

# Amazon Linux / RHEL
sudo yum install -y jq

# Ubuntu/Debian
sudo apt-get install -y jq
```

---

### Steps

#### Step 1: Create an IAM Role with Instance Profile

An IAM role allows EC2 instances to call AWS services without embedding credentials.

**Console:**
1. Navigate to **IAM → Roles → Create role**
2. Select **AWS service → EC2**
3. Attach policies:
   - `CloudWatchAgentServerPolicy`
   - `AmazonSSMManagedInstanceCore`
4. Name the role: `lab2-ec2-instance-role`
5. Create the role

**CLI:**
```bash
# Create trust policy document
cat > /tmp/ec2-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the IAM role
aws iam create-role \
  --role-name lab2-ec2-instance-role \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json \
  --description "EC2 role for Lab 2 - CloudWatch and SSM access"

# Attach managed policies
aws iam attach-role-policy \
  --role-name lab2-ec2-instance-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam attach-role-policy \
  --role-name lab2-ec2-instance-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Create instance profile and attach the role
aws iam create-instance-profile \
  --instance-profile-name lab2-ec2-instance-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name lab2-ec2-instance-profile \
  --role-name lab2-ec2-instance-role

echo "IAM role and instance profile created."
```

**Verify:**
```bash
aws iam get-instance-profile \
  --instance-profile-name lab2-ec2-instance-profile \
  --query 'InstanceProfile.Roles[0].RoleName'
# Expected: "lab2-ec2-instance-role"
```

---

#### Step 2: Create Security Group for SSM Access

With SSM Session Manager, you do **not** need port 22 open. The instance needs outbound HTTPS (443) to reach SSM endpoints.

**CLI:**
```bash
# Get default VPC
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)

# Create security group (no inbound SSH needed!)
SG_ID=$(aws ec2 