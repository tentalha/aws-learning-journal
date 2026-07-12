# EC2 — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Ensure the AWS CLI is installed and configured before running any EC2 commands.

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure credentials and default region
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

### Required IAM Permissions

Attach the following managed policies or equivalent inline permissions to your IAM user/role:

| Policy | Purpose |
|---|---|
| `AmazonEC2FullAccess` | Full EC2 management |
| `AmazonEC2ReadOnlyAccess` | Read-only describe operations |
| `AmazonVPCFullAccess` | VPC and networking management |

```bash
# Attach EC2 full access policy to an IAM user
aws iam attach-user-policy \
  --user-name my-devops-user \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

# Verify current caller identity
aws sts get-caller-identity
```

### Verify EC2 CLI Access

```bash
# Quick sanity check — list available regions
aws ec2 describe-regions --output table
```

---

## Core Commands

### 1. Describe Instances

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --output json
```

**What it does:** Returns detailed information about all EC2 instances in the specified region, including instance type, state, public/private IPs, tags, security groups, and more.

**Example Output:**
```json
{
  "Reservations": [
    {
      "ReservationId": "r-0a1b2c3d4e5f67890",
      "Instances": [
        {
          "InstanceId": "i-0a1b2c3d4e5f67890",
          "InstanceType": "t3.medium",
          "State": { "Name": "running" },
          "PublicIpAddress": "54.123.45.67",
          "PrivateIpAddress": "10.0.1.25",
          "Tags": [{ "Key": "Name", "Value": "my-web-server" }]
        }
      ]
    }
  ]
}
```

---

### 2. Run (Launch) an Instance

```bash
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.medium \
  --key-name my-keypair \
  --security-group-ids sg-0a1b2c3d4e5f67890 \
  --subnet-id subnet-0a1b2c3d4e5f67890 \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=my-web-server},{Key=Environment,Value=production}]' \
  --region us-east-1
```

**What it does:** Launches one or more EC2 instances with the specified AMI, instance type, key pair, security group, and subnet. Tags are applied at launch time.

---

### 3. Stop an Instance

```bash
aws ec2 stop-instances \
  --instance-ids i-0a1b2c3d4e5f67890 \
  --region us-east-1
```

**What it does:** Gracefully stops a running EC2 instance. The instance retains its EBS volumes and configuration. You are not billed for compute time while stopped (EBS charges still apply).

**Example Output:**
```json
{
  "StoppingInstances": [
    {
      "InstanceId": "i-0a1b2c3d4e5f67890",
      "CurrentState": { "Code": 64, "Name": "stopping" },
      "PreviousState": { "Code": 16, "Name": "running" }
    }
  ]
}
```

---

### 4. Start an Instance

```bash
aws ec2 start-instances \
  --instance-ids i-0a1b2c3d4e5f67890 \
  --region us-east-1
```

**What it does:** Starts a previously stopped EC2 instance. The instance will be assigned a new public IP unless an Elastic IP is attached.

---

### 5. Terminate an Instance

```bash
aws ec2 terminate-instances \
  --instance-ids i-0a1b2c3d4e5f67890 \
  --region us-east-1
```

**What it does:** Permanently terminates an EC2 instance. By default, the root EBS volume is deleted. This action is **irreversible**.

---

### 6. Describe Instance Status

```bash
aws ec2 describe-instance-status \
  --instance-ids i-0a1b2c3d4e5f67890 \
  --include-all-instances \
  --region us-east-1
```

**What it does:** Returns system status checks and instance status checks for the specified instance. Useful for verifying health after launch or troubleshooting failures.

**Example Output:**
```json
{
  "InstanceStatuses": [
    {
      "InstanceId": "i-0a1b2c3d4e5f67890",
      "InstanceState": { "Name": "running" },
      "SystemStatus": { "Status": "ok" },
      "InstanceStatus": { "Status": "ok" }
    }
  ]
}
```

---

### 7. Create a Key Pair

```bash
aws ec2 create-key-pair \
  --key-name my-keypair \
  --key-type rsa \
  --key-format pem \
  --query 'KeyMaterial' \
  --output text \
  --region us-east-1 > my-keypair.pem

chmod 400 my-keypair.pem
```

**What it does:** Creates a new RSA key pair and saves the private key locally. The `chmod 400` ensures the key file has the correct permissions for SSH use.

---

### 8. Describe Security Groups

```bash
aws ec2 describe-security-groups \
  --group-ids sg-0a1b2c3d4e5f67890 \
  --region us-east-1
```

**What it does:** Returns detailed information about the specified security group, including all inbound and outbound rules.

---

### 9. Create a Security Group

```bash
aws ec2 create-security-group \
  --group-name my-web-sg \
  --description "Security group for web servers" \
  --vpc-id vpc-0a1b2c3d4e5f67890 \
  --region us-east-1
```

**What it does:** Creates a new security group in the specified VPC. Returns the new security group ID.

**Example Output:**
```json
{
  "GroupId": "sg-0b2c3d4e5f6789012"
}
```

---

### 10. Authorize Security Group Ingress

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0a1b2c3d4e5f67890 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0 \
  --region us-east-1
```

**What it does:** Adds an inbound rule to the specified security group allowing HTTPS (port 443) traffic from any IP address.

---

### 11. Describe AMIs

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
            "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].[ImageId,Name,CreationDate]' \
  --output table \
  --region us-east-1
```

**What it does:** Finds the latest Amazon Linux 2 AMI available in the region. The `sort_by` and `[-1]` selects the most recently created image.

---

### 12. Create an EBS Snapshot

```bash
aws ec2 create-snapshot \
  --volume-id vol-0a1b2c3d4e5f67890 \
  --description "Snapshot of my-web-server root volume - $(date +%Y-%m-%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=my-web-server-snapshot},{Key=Automated,Value=true}]' \
  --region us-east-1
```

**What it does:** Creates a point-in-time snapshot of an EBS volume. Snapshots are stored in S3 and can be used to create new volumes or AMIs.

---

### 13. Allocate and Associate an Elastic IP

```bash
# Step 1: Allocate a new Elastic IP
aws ec2 allocate-address \
  --domain vpc \
  --region us-east-1

# Step 2: Associate it with an instance
aws ec2 associate-address \
  --instance-id i-0a1b2c3d4e5f67890 \
  --allocation-id eipalloc-0a1b2c3d4e5f67890 \
  --region us-east-1
```

**What it does:** Allocates a static public IP address and permanently associates it with an EC2 instance. The IP persists even when the instance is stopped/started.

---

### 14. Describe VPCs

```bash
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=false" \
  --query 'Vpcs[*].[VpcId,CidrBlock,Tags[?Key==`Name`].Value|[0]]' \
  --output table \
  --region us-east-1
```

**What it does:** Lists all non-default VPCs in the region, displaying their IDs, CIDR blocks, and Name tags in a readable table format.

---

### 15. Wait for Instance to Be Running

```bash
aws ec2 wait instance-running \
  --instance-ids i-0a1b2c3d4e5f67890 \
  --region us-east-1

echo "Instance is now running!"
```

**What it does:** Polls the instance state every 15 seconds (up to 40 attempts) and returns only when the instance reaches the `running` state. Essential for scripting sequential operations after launch.

---

## Common Operations

### 🟢 Create Operations

```bash
# Create a VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-production-vpc}]' \
  --region us-east-1

# Create a subnet
aws ec2 create-subnet \
  --vpc-id vpc-0a1b2c3d4e5f67890 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=my-public-subnet-1a}]' \
  --region us-east-1

# Create an internet gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=my-igw}]' \
  --region us-east-1

# Attach internet gateway to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0a1b2c3d4e5f67890 \
  --vpc-id vpc-0a1b2c3d4e5f67890 \
  --region us-east-1

# Create an EBS volume
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 100 \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125 \
  --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=my-data-volume}]' \
  --region us-east-1

# Create a launch template
aws ec2 create-launch-template \
  --launch-template-name my-web-server-template \
  --version-description "v1 - Initial template" \
  --launch-template-data '{
    "ImageId": "ami-0c02fb55956c7d316",
    "InstanceType": "t3.medium",
    "KeyName": "my-keypair",
    "SecurityGroupIds": ["sg-0a1b2c3d4e5f67890"],
    "UserData": "IyEvYmluL2Jhc2gKeXVtIHVwZGF0ZSAteQo="
  }' \
  --region us-east-1
```

---

### 🔵 Read / Describe Operations

```bash
# Describe instances with filter by tag
aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=production" \
            "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
  --output table \
  --region us-east-1

# Describe subnets in a VPC
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0a1b2c3d4e5f67890" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table \
  --region us-east-1

# Describe route tables
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-0a1b2c3d4e5f67890" \
  --region us-east-1

# Describe EBS volumes
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" \
  --query 'Volumes[*].[VolumeId,Size,VolumeType,AvailabilityZone]' \
  --output table \
  --region us-east-1

# Describe snapshots owned by your account
aws ec2 describe-snapshots \
  --owner-ids self \
  --query 'Snapshots[*].[SnapshotId,VolumeId,StartTime,State,Description]' \
  --output table \
  --region us-east-1

# Get console output from an instance (useful for debugging boot issues)
aws ec2 get-console-output \
  --instance-id i-0a1b2c3d4e5f67890 \
  --latest \
  --region us-east-1
```

---

### 🟡 Update / Modify Operations

```bash
# Modify instance type (instance must be stopped)
aws ec2 modify-instance-attribute \
  --instance-id i-0a1b2c3d4e5f67890 \
  --instance-type '{"Value": "t3.large"}' \
  --region us-east-1

# Modify EBS volume (online resize — no downtime required)
aws ec2 modify-volume \
  --volume-id vol-0a1b2c3d4e5f67890 \
  --size 200 \
  --volume-type gp3 \
  --iops 6000 \
  --region us-east-1

# Add tags to an existing resource
aws ec2 create-tags \