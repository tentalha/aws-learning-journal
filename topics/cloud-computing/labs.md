# Cloud Computing — Hands-On Labs

## Lab 1: Getting Started with Cloud Computing

### Objective

In this lab, you will launch your first Amazon EC2 virtual machine in the AWS Cloud, connect to it remotely, and deploy a simple web application. You will experience firsthand the core cloud computing concepts of on-demand provisioning, elasticity, and pay-as-you-go pricing. By the end of this lab, you will have a running web server accessible from the public internet.

---

### Prerequisites

**AWS Services Required:**
- Amazon EC2
- Amazon VPC (default VPC)
- AWS Identity and Access Management (IAM)

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:DescribeInstances",
        "ec2:TerminateInstances",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:DeleteSecurityGroup",
        "ec2:CreateKeyPair",
        "ec2:DeleteKeyPair",
        "ec2:DescribeKeyPairs",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- SSH client (Terminal on macOS/Linux, PuTTY or Windows Terminal on Windows)
- Web browser
- Text editor

**Estimated Cost:** < $0.02 (assuming lab completion within 1–2 hours using `t2.micro` or Free Tier eligible `t3.micro`)

**Estimated Duration:** 45–60 minutes

---

### Steps

#### Step 1: Create a Key Pair for SSH Access

A key pair allows you to securely connect to your EC2 instance.

**Console:**
1. Navigate to the [EC2 Console](https://console.aws.amazon.com/ec2/).
2. In the left navigation pane, under **Network & Security**, click **Key Pairs**.
3. Click **Create key pair**.
4. Configure:
   - **Name:** `lab1-keypair`
   - **Key pair type:** RSA
   - **Private key file format:** `.pem` (macOS/Linux) or `.ppk` (Windows/PuTTY)
5. Click **Create key pair**. The private key file will download automatically.
6. Move the key to a secure location and set permissions:

```bash
mv ~/Downloads/lab1-keypair.pem ~/.ssh/
chmod 400 ~/.ssh/lab1-keypair.pem
```

**CLI:**
```bash
# Create the key pair and save the private key
aws ec2 create-key-pair \
  --key-name lab1-keypair \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/lab1-keypair.pem

# Set correct permissions
chmod 400 ~/.ssh/lab1-keypair.pem
```

**Verify:**
```bash
aws ec2 describe-key-pairs --key-names lab1-keypair
```

**Expected Output:**
```json
{
    "KeyPairs": [
        {
            "KeyPairId": "key-0abc123def456789",
            "KeyName": "lab1-keypair",
            "KeyFingerprint": "xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx",
            "Tags": []
        }
    ]
}
```

---

#### Step 2: Create a Security Group

A security group acts as a virtual firewall controlling inbound and outbound traffic.

**Console:**
1. In the EC2 Console left pane, under **Network & Security**, click **Security Groups**.
2. Click **Create security group**.
3. Configure:
   - **Security group name:** `lab1-web-sg`
   - **Description:** `Security group for Lab 1 web server`
   - **VPC:** Select your default VPC
4. Under **Inbound rules**, click **Add rule** twice to add:
   - **Rule 1:** Type = `SSH`, Protocol = `TCP`, Port = `22`, Source = `My IP`
   - **Rule 2:** Type = `HTTP`, Protocol = `TCP`, Port = `80`, Source = `Anywhere-IPv4` (`0.0.0.0/0`)
5. Click **Create security group**.

**CLI:**
```bash
# Get the default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)

echo "Default VPC ID: $VPC_ID"

# Create the security group
SG_ID=$(aws ec2 create-security-group \
  --group-name lab1-web-sg \
  --description "Security group for Lab 1 web server" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "Security Group ID: $SG_ID"

# Get your current public IP
MY_IP=$(curl -s https://checkip.amazonaws.com)/32
echo "Your IP: $MY_IP"

# Allow SSH from your IP only
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr $MY_IP

# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

**Verify:**
```bash
aws ec2 describe-security-groups --group-ids $SG_ID \
  --query 'SecurityGroups[0].IpPermissions'
```

**Expected Output:** Two inbound rules — one for port 22 (SSH) and one for port 80 (HTTP).

---

#### Step 3: Launch an EC2 Instance

**Console:**
1. In the EC2 Console, click **Launch instances**.
2. Configure:
   - **Name:** `lab1-web-server`
   - **AMI:** Amazon Linux 2023 AMI (Free Tier eligible)
   - **Instance type:** `t2.micro` (Free Tier) or `t3.micro`
   - **Key pair:** `lab1-keypair`
   - **Network settings:** Select existing security group → `lab1-web-sg`
3. Expand **Advanced details** and paste the following into **User data**:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
EC2_AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
EC2_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
cat > /var/www/html/index.html << EOF
<!DOCTYPE html>
<html>
<head><title>My First Cloud Server</title></head>
<body style="font-family: Arial; text-align: center; margin-top: 50px;">
  <h1>🚀 Welcome to Cloud Computing!</h1>
  <p><strong>Instance ID:</strong> $EC2_ID</p>
  <p><strong>Availability Zone:</strong> $EC2_AZ</p>
  <p><strong>Launched:</strong> $(date)</p>
</body>
</html>
EOF
```

4. Click **Launch instance**.

**CLI:**
```bash
# Get the latest Amazon Linux 2023 AMI ID for your region
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-2023*-x86_64" \
    "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

echo "AMI ID: $AMI_ID"

# Create user data script
cat > /tmp/userdata.sh << 'USERDATA'
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
EC2_AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
EC2_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
cat > /var/www/html/index.html << EOF
<!DOCTYPE html>
<html>
<head><title>My First Cloud Server</title></head>
<body style="font-family: Arial; text-align: center; margin-top: 50px;">
  <h1>🚀 Welcome to Cloud Computing!</h1>
  <p><strong>Instance ID:</strong> $EC2_ID</p>
  <p><strong>Availability Zone:</strong> $EC2_AZ</p>
  <p><strong>Launched:</strong> $(date)</p>
</body>
</html>
EOF
USERDATA

# Launch the instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name lab1-keypair \
  --security-group-ids $SG_ID \
  --user-data file:///tmp/userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab1-web-server},{Key=Lab,Value=Lab1}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance ID: $INSTANCE_ID"
```

**Verify — Wait for the instance to reach running state:**
```bash
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
echo "Instance is now running!"

# Get the public IP address
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "Public IP: $PUBLIC_IP"
```

**Expected Output:**
```
Instance is now running!
Public IP: 54.123.45.67
```

---

#### Step 4: Connect to the Instance via SSH

**Console:**
1. Select your instance in the EC2 Console.
2. Click **Connect** → **SSH client** tab.
3. Follow the displayed SSH command.

**CLI / Terminal:**
```bash
# SSH into the instance
ssh -i ~/.ssh/lab1-keypair.pem ec2-user@$PUBLIC_IP

# Once connected, verify the web server is running
sudo systemctl status httpd

# Check the web page content
curl http://localhost
```

**Expected Output:**
```
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled)
   Active: active (running) since ...
```

---

#### Step 5: Access the Web Application

**From your local machine:**
```bash
# Test HTTP access from your local terminal
curl http://$PUBLIC_IP

# Or open in browser
echo "Open this URL in your browser: http://$PUBLIC_IP"
```

**Expected Output:**
```html
<!DOCTYPE html>
<html>
<head><title>My First Cloud Server</title></head>
<body style="font-family: Arial; text-align: center; margin-top: 50px;">
  <h1>🚀 Welcome to Cloud Computing!</h1>
  <p><strong>Instance ID:</strong> i-0abc123def456789</p>
  <p><strong>Availability Zone:</strong> us-east-1a</p>
  ...
</body>
</html>
```

Open your browser and navigate to `http://<PUBLIC_IP>` — you should see the welcome page with your instance details.

---

### Verification

Run the following checklist to confirm successful lab completion:

```bash
# 1. Verify key pair exists
aws ec2 describe-key-pairs --key-names lab1-keypair \
  --query 'KeyPairs[0].KeyName' --output text
# Expected: lab1-keypair

# 2. Verify security group exists with correct rules
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=lab1-web-sg" \
  --query 'SecurityGroups[0].GroupId' --output text
# Expected: sg-xxxxxxxxxxxxxxxxx

# 3. Verify instance is running
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].State.Name' --output text
# Expected: running

# 4. Verify web server responds
curl -s -o /dev/null -w "%{http_code}" http://$PUBLIC_IP
# Expected: 200
```

✅ **Lab Complete** if all four checks return expected values and the web page is visible in your browser.

---

### Cleanup

> ⚠️ **Important:** Complete cleanup within the same session to avoid unexpected charges.

```bash
# Step 1: Terminate the EC2 instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
echo "Terminating instance: $INSTANCE_ID"

# Step 2: Wait for termination to complete
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID
echo "Instance terminated."

# Step 3: Delete the security group (must wait for instance termination first)
aws ec2 delete-security-group --group-id $SG_ID
echo "Security group deleted."

# Step 4: Delete the key pair
aws ec2 delete-key-pair --key-name lab1-keypair
echo "Key pair deleted from AWS."

# Step 5: Remove local private key file
rm -f ~/.ssh/lab1-keypair.pem
echo "Local key file removed."

# Step 6: Verify cleanup
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text
# Expected: terminated
```

---

## Lab 2: Intermediate Cloud Computing Configuration

### Objective

In this lab, you will move beyond a single server to implement a scalable, load-balanced web application using Amazon EC2 Auto Scaling and an Application Load Balancer (ALB). You will configure CloudWatch alarms to monitor CPU utilization and trigger automatic scaling events. By the end of this lab, you will understand how cloud elasticity works in practice — automatically adding or removing capacity based on demand.

---

### Prerequisites

**AWS Services Required:**
- Amazon EC2
- Amazon EC2 Auto Scaling
- Elastic Load Balancing (Application Load Balancer)
- Amazon CloudWatch
- Amazon VPC (default VPC with at least 2 subnets in different AZs)

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "elasticloadbalancing:*",
        "autoscaling:*",
        "cloudwatch:*",
        "iam:CreateServiceLinkedRole",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 configured
- `stress` tool (installed on EC2 instances via user data)
- Web browser
- `jq` (optional, for JSON parsing)

**Prior Knowledge:** Completion of Lab 1 or equivalent EC2 experience.

**Estimated Cost:** ~$0.10–$0.30 (2–3 hours with 2–3 `t2.micro` instances + ALB)

**Estimated Duration:** 90–120 minutes

---

### Steps

#### Step 1: Set Up Networking — Gather VPC and Subnet Information

**Console:**
1. Navigate to **VPC Console** → **Your VPCs**.
2. Note the