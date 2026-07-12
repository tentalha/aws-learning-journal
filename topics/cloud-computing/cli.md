# Cloud Computing — AWS CLI Commands

## Setup & Configuration

Before using AWS CLI commands for cloud computing services, ensure the following prerequisites are met.

### Install and Configure AWS CLI

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version

# Configure AWS CLI with credentials
aws configure
# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: us-east-1
# Default output format [None]: json

# Configure a named profile
aws configure --profile my-cloud-profile

# Use environment variables instead
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-1"
```

### Required IAM Permissions

Attach these managed policies or equivalent custom policies to your IAM user or role:

```bash
# Attach AdministratorAccess for full access (use least privilege in production)
aws iam attach-user-policy \
  --user-name my-devops-user \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Attach EC2 full access
aws iam attach-user-policy \
  --user-name my-devops-user \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

# Attach S3 full access
aws iam attach-user-policy \
  --user-name my-devops-user \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Attach VPC full access
aws iam attach-user-policy \
  --user-name my-devops-user \
  --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess

# Verify current caller identity
aws sts get-caller-identity
```

**Example Output:**
```json
{
    "UserId": "AIDAIOSFODNN7EXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/my-devops-user"
}
```

---

## Core Commands

### 1. Launch an EC2 Instance

```bash
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.micro \
  --key-name my-keypair \
  --security-group-ids sg-0a1b2c3d4e5f67890 \
  --subnet-id subnet-0a1b2c3d4e5f67890 \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=my-web-server},{Key=Environment,Value=production}]' \
  --iam-instance-profile Name=my-ec2-instance-profile \
  --user-data file://user-data.sh
```

**What it does:** Launches a new EC2 virtual machine instance with a specified AMI, instance type, networking configuration, and startup script.

**Example Output:**
```json
{
    "Instances": [
        {
            "InstanceId": "i-0abcd1234efgh5678",
            "InstanceType": "t3.micro",
            "State": { "Name": "pending" },
            "PublicDnsName": "",
            "PrivateIpAddress": "10.0.1.45",
            "LaunchTime": "2024-01-15T10:30:00.000Z"
        }
    ]
}
```

---

### 2. Create an S3 Bucket

```bash
aws s3api create-bucket \
  --bucket my-cloud-storage-bucket-20240115 \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1

# Note: For us-east-1, omit --create-bucket-configuration
aws s3api create-bucket \
  --bucket my-cloud-storage-bucket-20240115 \
  --region us-east-1
```

**What it does:** Creates a new S3 bucket for object storage in the specified region.

---

### 3. Upload Files to S3

```bash
# Upload a single file
aws s3 cp /local/path/myfile.txt s3://my-cloud-storage-bucket-20240115/uploads/myfile.txt

# Sync an entire directory
aws s3 sync /local/website-files/ s3://my-cloud-storage-bucket-20240115/website/ \
  --delete \
  --exclude "*.tmp" \
  --include "*.html" \
  --include "*.css" \
  --include "*.js"
```

**What it does:** Uploads files or synchronizes a local directory to an S3 bucket, with optional filters.

---

### 4. Create a VPC

```bash
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-production-vpc},{Key=Environment,Value=production}]'
```

**What it does:** Creates a new Virtual Private Cloud (VPC) with a defined IP address range for isolating cloud resources.

**Example Output:**
```json
{
    "Vpc": {
        "VpcId": "vpc-0abc1234def56789a",
        "CidrBlock": "10.0.0.0/16",
        "State": "available",
        "IsDefault": false
    }
}
```

---

### 5. Create a Subnet

```bash
aws ec2 create-subnet \
  --vpc-id vpc-0abc1234def56789a \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=my-public-subnet-1a}]'
```

**What it does:** Creates a subnet within a VPC in a specific Availability Zone, enabling resource placement across AZs.

---

### 6. Create an Auto Scaling Group

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-asg-production \
  --launch-template LaunchTemplateId=lt-0abc1234def56789a,Version='$Latest' \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 3 \
  --vpc-zone-identifier "subnet-0a1b2c3d4e5f67890,subnet-0b2c3d4e5f678901a" \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --tags Key=Name,Value=my-asg-instance,PropagateAtLaunch=true
```

**What it does:** Creates an Auto Scaling Group that automatically adjusts the number of EC2 instances based on demand.

---

### 7. Create an Application Load Balancer

```bash
aws elbv2 create-load-balancer \
  --name my-app-load-balancer \
  --subnets subnet-0a1b2c3d4e5f67890 subnet-0b2c3d4e5f678901a \
  --security-groups sg-0a1b2c3d4e5f67890 \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --tags Key=Environment,Value=production
```

**What it does:** Creates an internet-facing Application Load Balancer to distribute HTTP/HTTPS traffic across multiple targets.

---

### 8. Create an RDS Database Instance

```bash
aws rds create-db-instance \
  --db-instance-identifier my-production-db \
  --db-instance-class db.t3.medium \
  --engine mysql \
  --engine-version 8.0.35 \
  --master-username admin \
  --master-user-password MySecurePassword123! \
  --allocated-storage 100 \
  --storage-type gp3 \
  --vpc-security-group-ids sg-0a1b2c3d4e5f67890 \
  --db-subnet-group-name my-db-subnet-group \
  --multi-az \
  --backup-retention-period 7 \
  --tags Key=Environment,Value=production
```

**What it does:** Creates a managed relational database instance with Multi-AZ for high availability and automated backups.

---

### 9. Create a Lambda Function

```bash
aws lambda create-function \
  --function-name my-cloud-function \
  --runtime python3.12 \
  --role arn:aws:iam::123456789012:role/my-lambda-execution-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://my-function.zip \
  --timeout 30 \
  --memory-size 256 \
  --environment Variables='{DB_HOST=my-db.cluster.us-east-1.rds.amazonaws.com,ENVIRONMENT=production}' \
  --tags Environment=production,Service=my-cloud-function
```

**What it does:** Creates a serverless Lambda function that executes code in response to events without managing servers.

---

### 10. Create a CloudFormation Stack

```bash
aws cloudformation create-stack \
  --stack-name my-infrastructure-stack \
  --template-url https://s3.amazonaws.com/my-cloud-storage-bucket-20240115/templates/infrastructure.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=production \
    ParameterKey=InstanceType,ParameterValue=t3.medium \
    ParameterKey=KeyPairName,ParameterValue=my-keypair \
  --capabilities CAPABILITY_NAMED_IAM \
  --tags Key=Project,Value=my-cloud-project Key=Environment,Value=production \
  --on-failure ROLLBACK
```

**What it does:** Deploys a full infrastructure stack using Infrastructure as Code (IaC) templates, enabling repeatable deployments.

---

### 11. Create a CloudWatch Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name my-cpu-high-alarm \
  --alarm-description "Alert when CPU exceeds 80% for 5 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-0abcd1234efgh5678 \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --ok-actions arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --treat-missing-data notBreaching
```

**What it does:** Creates a CloudWatch alarm that triggers SNS notifications when CPU utilization exceeds a threshold.

---

### 12. Create an SNS Topic and Subscribe

```bash
# Create SNS topic
aws sns create-topic \
  --name my-alerts-topic \
  --tags Key=Environment,Value=production

# Subscribe an email endpoint
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --protocol email \
  --notification-endpoint devops-team@mycompany.com
```

**What it does:** Creates a Simple Notification Service topic for pub/sub messaging and subscribes an email address to receive alerts.

---

### 13. Create an ECS Cluster

```bash
aws ecs create-cluster \
  --cluster-name my-production-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=1 \
    capacityProvider=FARGATE_SPOT,weight=4 \
  --settings name=containerInsights,value=enabled \
  --tags key=Environment,value=production
```

**What it does:** Creates an ECS cluster with Fargate capacity providers for running containerized applications without managing servers.

---

### 14. Create a Route 53 Hosted Zone

```bash
aws route53 create-hosted-zone \
  --name mycompany.com \
  --caller-reference "$(date +%s)" \
  --hosted-zone-config Comment="Production hosted zone",PrivateZone=false
```

**What it does:** Creates a public DNS hosted zone for managing domain name routing to cloud resources.

---

### 15. Create a KMS Key

```bash
aws kms create-key \
  --description "Production data encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --policy file://key-policy.json \
  --tags TagKey=Environment,TagValue=production TagKey=Service,TagValue=data-encryption

# Create a human-readable alias
aws kms create-alias \
  --alias-name alias/my-production-key \
  --target-key-id arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012
```

**What it does:** Creates a Customer Managed Key (CMK) in AWS KMS for encrypting data at rest across AWS services.

---

## Common Operations

### Create Operations

```bash
# Create a security group
aws ec2 create-security-group \
  --group-name my-web-sg \
  --description "Security group for web servers" \
  --vpc-id vpc-0abc1234def56789a \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=my-web-sg}]'

# Add inbound rules to security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-0a1b2c3d4e5f67890 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id sg-0a1b2c3d4e5f67890 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Create an EC2 key pair
aws ec2 create-key-pair \
  --key-name my-keypair \
  --key-type rsa \
  --key-format pem \
  --query 'KeyMaterial' \
  --output text > my-keypair.pem
chmod 400 my-keypair.pem

# Create an IAM role
aws iam create-role \
  --role-name my-ec2-instance-role \
  --assume-role-policy-document file://ec2-trust-policy.json \
  --description "EC2 instance role for production servers"

# Create an EBS volume
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 100 \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125 \
  --encrypted \
  --kms-key-id alias/my-production-key \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=my-data-volume}]'

# Create an Elastic IP
aws ec2 allocate-address \
  --domain vpc