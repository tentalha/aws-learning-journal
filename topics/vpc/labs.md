# VPC — Hands-On Labs

## Lab 1: Getting Started with VPC

### Objective
In this lab, you will create a custom Amazon VPC from scratch. You will learn how to define a CIDR block, create public and private subnets across two Availability Zones, attach an Internet Gateway, configure route tables, and launch EC2 instances into each subnet. By the end, you will understand the fundamental building blocks of AWS networking.

### Prerequisites

**AWS Services Used:**
- Amazon VPC
- Amazon EC2
- AWS IAM

**Required IAM Permissions:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*Vpc*",
        "ec2:*Subnet*",
        "ec2:*InternetGateway*",
        "ec2:*RouteTable*",
        "ec2:*SecurityGroup*",
        "ec2:RunInstances",
        "ec2:DescribeInstances",
        "ec2:TerminateInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS Management Console access
- AWS CLI v2 installed and configured (`aws configure`)
- An existing EC2 Key Pair (or create one during the lab)
- Basic familiarity with IP addressing and CIDR notation

**Estimated Cost:** < $0.10 if cleaned up within 1 hour  
**Estimated Duration:** 45–60 minutes  
**AWS Region:** `us-east-1` (N. Virginia) — adjust as needed

---

### Steps

#### Step 1: Create a Custom VPC

**Console:**
1. Navigate to **VPC Dashboard** → **Your VPCs** → **Create VPC**
2. Select **VPC only**
3. Configure:
   - **Name tag:** `lab1-vpc`
   - **IPv4 CIDR block:** `10.0.0.0/16`
   - **IPv6 CIDR block:** No IPv6 CIDR block
   - **Tenancy:** Default
4. Click **Create VPC**

**CLI:**
```bash
# Create the VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab1-vpc}]' \
  --query 'Vpc.VpcId' \
  --output text)

echo "VPC ID: $VPC_ID"

# Enable DNS hostnames (best practice)
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-hostnames

# Enable DNS resolution
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-support
```

**✅ Verify:**
```bash
aws ec2 describe-vpcs \
  --vpc-ids $VPC_ID \
  --query 'Vpcs[0].{ID:VpcId,CIDR:CidrBlock,State:State,DNS:EnableDnsHostnames}'
```
**Expected Output:**
```json
{
    "ID": "vpc-0abc123def456789",
    "CIDR": "10.0.0.0/16",
    "State": "available",
    "DNS": true
}
```

---

#### Step 2: Create Subnets

You will create four subnets — two public and two private — spread across two Availability Zones.

**Console:**
1. Navigate to **VPC Dashboard** → **Subnets** → **Create subnet**
2. Select `lab1-vpc` from the VPC dropdown
3. Create each subnet with the following settings:

| Subnet Name              | AZ           | CIDR Block    | Type    |
|--------------------------|--------------|---------------|---------|
| `lab1-public-subnet-1a`  | us-east-1a   | 10.0.1.0/24   | Public  |
| `lab1-public-subnet-1b`  | us-east-1b   | 10.0.2.0/24   | Public  |
| `lab1-private-subnet-1a` | us-east-1a   | 10.0.11.0/24  | Private |
| `lab1-private-subnet-1b` | us-east-1b   | 10.0.12.0/24  | Private |

**CLI:**
```bash
# Public Subnet in AZ 1a
PUB_SUBNET_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab1-public-subnet-1a}]' \
  --query 'Subnet.SubnetId' \
  --output text)

# Public Subnet in AZ 1b
PUB_SUBNET_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab1-public-subnet-1b}]' \
  --query 'Subnet.SubnetId' \
  --output text)

# Private Subnet in AZ 1a
PRIV_SUBNET_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.11.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab1-private-subnet-1a}]' \
  --query 'Subnet.SubnetId' \
  --output text)

# Private Subnet in AZ 1b
PRIV_SUBNET_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.12.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab1-private-subnet-1b}]' \
  --query 'Subnet.SubnetId' \
  --output text)

# Enable auto-assign public IPs for public subnets
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_1A \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_1B \
  --map-public-ip-on-launch

echo "Public Subnet 1A: $PUB_SUBNET_1A"
echo "Public Subnet 1B: $PUB_SUBNET_1B"
echo "Private Subnet 1A: $PRIV_SUBNET_1A"
echo "Private Subnet 1B: $PRIV_SUBNET_1B"
```

**✅ Verify:**
```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].{Name:Tags[?Key==`Name`].Value|[0],CIDR:CidrBlock,AZ:AvailabilityZone,State:State}' \
  --output table
```
**Expected Output:** A table showing all 4 subnets with `State: available`.

---

#### Step 3: Create and Attach an Internet Gateway

**Console:**
1. Navigate to **VPC Dashboard** → **Internet Gateways** → **Create internet gateway**
2. **Name tag:** `lab1-igw`
3. Click **Create internet gateway**
4. Select the new IGW → **Actions** → **Attach to VPC** → select `lab1-vpc`

**CLI:**
```bash
# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=lab1-igw}]' \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

echo "Internet Gateway ID: $IGW_ID"

# Attach to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID
```

**✅ Verify:**
```bash
aws ec2 describe-internet-gateways \
  --internet-gateway-ids $IGW_ID \
  --query 'InternetGateways[0].{ID:InternetGatewayId,State:Attachments[0].State,VPC:Attachments[0].VpcId}'
```
**Expected Output:**
```json
{
    "ID": "igw-0abc123def456789",
    "State": "available",
    "VPC": "vpc-0abc123def456789"
}
```

---

#### Step 4: Configure Route Tables

**Console:**
1. Navigate to **VPC Dashboard** → **Route Tables** → **Create route table**
2. Create a **Public Route Table:**
   - **Name:** `lab1-public-rt`
   - **VPC:** `lab1-vpc`
3. Select `lab1-public-rt` → **Routes** tab → **Edit routes** → **Add route:**
   - **Destination:** `0.0.0.0/0`
   - **Target:** `lab1-igw`
4. **Subnet Associations** tab → **Edit subnet associations** → select both public subnets

**CLI:**
```bash
# Create Public Route Table
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=lab1-public-rt}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

echo "Public Route Table ID: $PUB_RT_ID"

# Add default route to Internet Gateway
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# Associate public subnets with public route table
aws ec2 associate-route-table \
  --route-table-id $PUB_RT_ID \
  --subnet-id $PUB_SUBNET_1A

aws ec2 associate-route-table \
  --route-table-id $PUB_RT_ID \
  --subnet-id $PUB_SUBNET_1B

# Private subnets use the default (main) route table — no IGW route needed
# Tag the main route table for clarity
MAIN_RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.main,Values=true" \
  --query 'RouteTables[0].RouteTableId' \
  --output text)

aws ec2 create-tags \
  --resources $MAIN_RT_ID \
  --tags Key=Name,Value=lab1-private-rt
```

**✅ Verify:**
```bash
aws ec2 describe-route-tables \
  --route-table-ids $PUB_RT_ID \
  --query 'RouteTables[0].Routes[*].{Destination:DestinationCidrBlock,Target:GatewayId,State:State}'
```
**Expected Output:**
```json
[
    {"Destination": "10.0.0.0/16", "Target": "local",  "State": "active"},
    {"Destination": "0.0.0.0/0",  "Target": "igw-0abc123def456789", "State": "active"}
]
```

---

#### Step 5: Create Security Groups

**Console:**
1. Navigate to **VPC Dashboard** → **Security Groups** → **Create security group**
2. Create **Web Server SG:**
   - **Name:** `lab1-web-sg`
   - **Description:** `Allow HTTP and SSH inbound`
   - **VPC:** `lab1-vpc`
   - **Inbound rules:**
     - SSH (22) from `0.0.0.0/0` *(for lab only — restrict in production)*
     - HTTP (80) from `0.0.0.0/0`

**CLI:**
```bash
# Create Web Server Security Group
WEB_SG_ID=$(aws ec2 create-security-group \
  --group-name lab1-web-sg \
  --description "Allow HTTP and SSH inbound" \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=lab1-web-sg}]' \
  --query 'GroupId' \
  --output text)

echo "Web SG ID: $WEB_SG_ID"

# Allow SSH inbound
aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# Allow HTTP inbound
aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

---

#### Step 6: Launch a Test EC2 Instance in the Public Subnet

**Console:**
1. Navigate to **EC2** → **Instances** → **Launch instances**
2. Configure:
   - **Name:** `lab1-web-server`
   - **AMI:** Amazon Linux 2023 (free tier eligible)
   - **Instance type:** `t2.micro`
   - **Key pair:** Select your existing key pair
   - **Network settings:**
     - **VPC:** `lab1-vpc`
     - **Subnet:** `lab1-public-subnet-1a`
     - **Auto-assign public IP:** Enable
     - **Security group:** `lab1-web-sg`
3. **Advanced details** → **User data:**
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from Lab 1 VPC - $(hostname -f)</h1>" > /var/www/html/index.html
```
4. Click **Launch instance**

**CLI:**
```bash
# Get latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-*-x86_64" \
    "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

echo "Using AMI: $AMI_ID"

# Create user data script
cat > /tmp/userdata.sh << 'EOF'
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from Lab 1 VPC - $(hostname -f)</h1>" > /var/www/html/index.html
EOF

# Launch EC2 instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name YOUR_KEY_PAIR_NAME \
  --subnet-id $PUB_SUBNET_1A \
  --security-group-ids $WEB_SG_ID \
  --associate-public-ip-address \
  --user-data file:///tmp/userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=