# VPC — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before running VPC CLI commands, ensure the following are in place:

**AWS CLI Installation & Authentication**
```bash
# Verify AWS CLI version (v2 recommended)
aws --version

# Configure default credentials and region
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json

# Or use a named profile
aws configure --profile my-vpc-profile
export AWS_PROFILE=my-vpc-profile
```

**Required IAM Permissions**

Attach the following IAM policy to your user or role. At minimum, you need:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc",
        "ec2:DeleteVpc",
        "ec2:DescribeVpcs",
        "ec2:ModifyVpcAttribute",
        "ec2:CreateSubnet",
        "ec2:DeleteSubnet",
        "ec2:DescribeSubnets",
        "ec2:CreateInternetGateway",
        "ec2:AttachInternetGateway",
        "ec2:DetachInternetGateway",
        "ec2:DeleteInternetGateway",
        "ec2:CreateRouteTable",
        "ec2:CreateRoute",
        "ec2:AssociateRouteTable",
        "ec2:DescribeRouteTables",
        "ec2:CreateSecurityGroup",
        "ec2:DescribeSecurityGroups",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:CreateNatGateway",
        "ec2:DescribeNatGateways",
        "ec2:AllocateAddress",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
```

**Set Default Output Format and Region**
```bash
# Set environment variables for convenience
export AWS_DEFAULT_REGION=us-east-1
export AWS_DEFAULT_OUTPUT=json

# Verify identity
aws sts get-caller-identity
```

---

## Core Commands

### 1. Create a VPC
```bash
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-production-vpc},{Key=Environment,Value=production}]'
```
**What it does:** Creates a new VPC with the specified IPv4 CIDR block and applies tags. Returns the VPC ID and configuration details.

**Example Output:**
```json
{
  "Vpc": {
    "VpcId": "vpc-0abc1234def567890",
    "State": "pending",
    "CidrBlock": "10.0.0.0/16",
    "DhcpOptionsId": "dopt-0123456789abcdef0",
    "InstanceTenancy": "default",
    "Ipv6CidrBlockAssociationSet": [],
    "CidrBlockAssociationSet": [
      {
        "AssociationId": "vpc-cidr-assoc-0abc123456789",
        "CidrBlock": "10.0.0.0/16",
        "CidrBlockState": { "State": "associated" }
      }
    ],
    "IsDefault": false,
    "Tags": [
      { "Key": "Name", "Value": "my-production-vpc" },
      { "Key": "Environment", "Value": "production" }
    ]
  }
}
```

---

### 2. Describe VPCs
```bash
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=my-production-vpc" \
  --query 'Vpcs[*].{ID:VpcId,CIDR:CidrBlock,State:State,Default:IsDefault}' \
  --output table
```
**What it does:** Lists all VPCs matching the filter. The `--query` flag formats output to show only relevant fields in a readable table.

**Example Output:**
```
-------------------------------------------------------
|                    DescribeVpcs                     |
+----------+----------------+----------+--------------+
|  CIDR    |    Default     |    ID    |    State     |
+----------+----------------+----------+--------------+
|  10.0.0.0/16 |  False   |  vpc-0abc1234def567890  |  available  |
+----------+----------------+----------+--------------+
```

---

### 3. Create a Public Subnet
```bash
aws ec2 create-subnet \
  --vpc-id vpc-0abc1234def567890 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=my-public-subnet-1a},{Key=Type,Value=public}]'
```
**What it does:** Creates a subnet within the specified VPC in a given Availability Zone. This subnet will later be configured as public by associating it with a route table that has an internet gateway route.

---

### 4. Create a Private Subnet
```bash
aws ec2 create-subnet \
  --vpc-id vpc-0abc1234def567890 \
  --cidr-block 10.0.10.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=my-private-subnet-1a},{Key=Type,Value=private}]'
```
**What it does:** Creates a private subnet. Traffic from this subnet will route through a NAT Gateway for outbound internet access.

---

### 5. Create and Attach an Internet Gateway
```bash
# Step 1: Create the internet gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=my-igw}]'

# Step 2: Attach it to the VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0abc1234def567890 \
  --vpc-id vpc-0abc1234def567890
```
**What it does:** Creates an Internet Gateway (IGW) and attaches it to the VPC, enabling internet connectivity for resources in public subnets.

---

### 6. Create a Route Table and Add a Route
```bash
# Create a public route table
aws ec2 create-route-table \
  --vpc-id vpc-0abc1234def567890 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=my-public-rt}]'

# Add a default route to the internet gateway
aws ec2 create-route \
  --route-table-id rtb-0abc1234def567890 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0abc1234def567890
```
**What it does:** Creates a route table and adds a default route (`0.0.0.0/0`) pointing to the Internet Gateway, making it a "public" route table.

---

### 7. Associate a Route Table with a Subnet
```bash
aws ec2 associate-route-table \
  --route-table-id rtb-0abc1234def567890 \
  --subnet-id subnet-0abc1234def567890
```
**What it does:** Associates the public route table with a subnet, making that subnet "public" — resources in it can route traffic to the internet.

---

### 8. Enable Auto-Assign Public IP on a Subnet
```bash
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-0abc1234def567890 \
  --map-public-ip-on-launch
```
**What it does:** Configures the subnet so that EC2 instances launched into it automatically receive a public IPv4 address.

---

### 9. Create a NAT Gateway
```bash
# First, allocate an Elastic IP address
aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=my-nat-eip}]'

# Then create the NAT Gateway in a public subnet
aws ec2 create-nat-gateway \
  --subnet-id subnet-0abc1234def567890 \
  --allocation-id eipalloc-0abc1234def567890 \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=my-nat-gw}]'
```
**What it does:** Allocates an Elastic IP and creates a NAT Gateway in a public subnet, allowing private subnet resources to initiate outbound internet connections without being directly reachable from the internet.

---

### 10. Create a Security Group
```bash
aws ec2 create-security-group \
  --group-name my-web-sg \
  --description "Security group for web servers" \
  --vpc-id vpc-0abc1234def567890 \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=my-web-sg}]'
```
**What it does:** Creates a new security group within the specified VPC. By default, it allows all outbound traffic and denies all inbound traffic.

---

### 11. Add Inbound Rules to a Security Group
```bash
# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc1234def567890 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow HTTPS from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc1234def567890 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Allow SSH from a specific IP range only
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc1234def567890 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.0/24
```
**What it does:** Adds inbound (ingress) rules to the security group. Each rule specifies the protocol, port, and source CIDR.

---

### 12. Create a Network ACL
```bash
aws ec2 create-network-acl \
  --vpc-id vpc-0abc1234def567890 \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=my-public-nacl}]'
```
**What it does:** Creates a stateless Network ACL (NACL) for the VPC. Unlike security groups, NACLs are stateless and require explicit rules for both inbound and outbound traffic.

---

### 13. Describe Route Tables
```bash
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-0abc1234def567890" \
  --query 'RouteTables[*].{ID:RouteTableId,Routes:Routes,Associations:Associations[*].SubnetId}' \
  --output json
```
**What it does:** Lists all route tables in the specified VPC, showing routes and associated subnets.

---

### 14. Delete a VPC
```bash
aws ec2 delete-vpc \
  --vpc-id vpc-0abc1234def567890
```
**What it does:** Deletes the specified VPC. **Note:** All dependencies (subnets, route tables, IGW, security groups, etc.) must be removed first or the command will fail.

---

### 15. Enable VPC Flow Logs
```bash
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-0abc1234def567890 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs/my-production-vpc \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/FlowLogsRole \
  --tag-specifications 'ResourceType=vpc-flow-log,Tags=[{Key=Name,Value=my-vpc-flow-logs}]'
```
**What it does:** Enables VPC Flow Logs to capture IP traffic information for the VPC and sends logs to CloudWatch Logs for monitoring and troubleshooting.

---

## Common Operations

### Create Operations

```bash
# Create VPC with IPv6 support
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --amazon-provided-ipv6-cidr-block \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-ipv6-vpc}]'

# Create a subnet with IPv6
aws ec2 create-subnet \
  --vpc-id vpc-0abc1234def567890 \
  --cidr-block 10.0.2.0/24 \
  --ipv6-cidr-block 2600:1f18:abcd:ef01::/64 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=my-ipv6-subnet}]'

# Create VPC peering connection
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-0abc1234def567890 \
  --peer-vpc-id vpc-0def4567abc890123 \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=my-vpc-peering}]'

# Create a VPC endpoint for S3 (Gateway type)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0abc1234def567890 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-0abc1234def567890 \
  --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=my-s3-endpoint}]'

# Create a VPC endpoint for SSM (Interface type)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0abc1234def567890 \
  --service-name com.amazonaws.us-east-1.ssm \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-0abc1234def567890 \
  --security-group-ids sg-0abc1234def567890 \
  --private-dns-enabled \
  --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=my-ssm-endpoint}]'
```

---

### Read / Describe Operations

```bash
# Describe all VPCs with full details
aws ec2 describe-vpcs \
  --output json

# Describe a specific VPC by ID
aws ec2 describe-vpcs \
  --vpc-ids vpc-0abc1234def567890

# Describe all subnets in a VPC
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0abc1234def567890" \
  --query 'Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch}' \
  --output table

# Describe internet gateways attached to a VPC
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=vpc-0abc1234def567890"

# Describe NAT gateways
aws ec2 describe-nat-gateways \
  --filter "Name=vpc-id,Values=vpc-0abc1234def567890" \
  