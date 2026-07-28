# Terraform — Hands-On Labs

## Lab 1: Getting Started with Terraform

### Objective

In this lab, you will install Terraform, configure the AWS provider, and deploy your first infrastructure resources on AWS. You will create a VPC, a public subnet, and an EC2 instance using Terraform. By the end of this lab, you will understand Terraform's core workflow (`init`, `plan`, `apply`, `destroy`), how to write basic HCL configuration files, and how to manage state.

---

### Prerequisites

**AWS Services Used:**
- Amazon EC2
- Amazon VPC
- AWS IAM

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "vpc:*",
        "iam:GetUser"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- Terraform >= 1.6.0 ([Install Guide](https://developer.hashicorp.com/terraform/install))
- AWS CLI v2 configured with a named profile
- A code editor (VS Code with HashiCorp Terraform extension recommended)
- Git

**Verify Prerequisites:**
```bash
terraform -version
# Expected: Terraform v1.6.x or higher

aws --version
# Expected: aws-cli/2.x.x

aws sts get-caller-identity
# Expected: JSON output with your Account ID and ARN
```

---

### Steps

#### Step 1: Install and Configure Terraform

**Option A — Linux/macOS (using Homebrew):**
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Verify installation
terraform -version
```

**Option B — Linux (manual install):**
```bash
# Download the latest Terraform binary
TERRAFORM_VERSION="1.6.6"
wget https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip

# Unzip and move to PATH
unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Verify
terraform -version
```

**Option C — Windows (using Chocolatey):**
```powershell
choco install terraform
terraform -version
```

**Configure AWS CLI credentials:**
```bash
aws configure --profile terraform-lab
# AWS Access Key ID: <your-access-key>
# AWS Secret Access Key: <your-secret-key>
# Default region name: us-east-1
# Default output format: json

# Set the profile for this session
export AWS_PROFILE=terraform-lab
```

**✅ Verify Step 1:**
```bash
terraform -version
# Expected output:
# Terraform v1.6.6
# on linux_amd64

aws sts get-caller-identity --profile terraform-lab
# Expected: JSON with your AccountId
```

---

#### Step 2: Create Your Project Directory Structure

```bash
# Create and navigate to the project directory
mkdir -p ~/terraform-labs/lab1
cd ~/terraform-labs/lab1

# Create the initial file structure
touch main.tf variables.tf outputs.tf terraform.tfvars

# Verify structure
ls -la
# Expected:
# main.tf
# variables.tf
# outputs.tf
# terraform.tfvars
```

---

#### Step 3: Write the Terraform Configuration

**Create `variables.tf`:**
```hcl
# variables.tf

variable "aws_region" {
  description = "The AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Name prefix for all resources"
  type        = string
  default     = "tf-lab1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidr" {
  description = "CIDR block for the public subnet"
  type        = string
  default     = "10.0.1.0/24"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}
```

**Create `main.tf`:**
```hcl
# main.tf

terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region  = var.aws_region
  profile = "terraform-lab"

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}

# --- Data Sources ---

# Fetch the latest Amazon Linux 2023 AMI
data "aws_ami" "amazon_linux_2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# --- Networking ---

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidr
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-public-subnet"
    Type = "Public"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# --- Security Group ---

resource "aws_security_group" "web" {
  name        = "${var.project_name}-web-sg"
  description = "Security group for web server"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "All outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-web-sg"
  }
}

# --- EC2 Instance ---

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux_2023.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello from Terraform Lab 1!</h1>" > /var/www/html/index.html
    echo "<p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html
  EOF

  tags = {
    Name = "${var.project_name}-web-server"
  }
}
```

**Create `outputs.tf`:**
```hcl
# outputs.tf

output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_id" {
  description = "The ID of the public subnet"
  value       = aws_subnet.public.id
}

output "instance_id" {
  description = "The EC2 instance ID"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "The public IP address of the web server"
  value       = aws_instance.web.public_ip
}

output "web_url" {
  description = "URL to access the web server"
  value       = "http://${aws_instance.web.public_ip}"
}

output "ami_id" {
  description = "The AMI ID used for the instance"
  value       = data.aws_ami.amazon_linux_2023.id
}
```

**Create `terraform.tfvars`:**
```hcl
# terraform.tfvars
aws_region         = "us-east-1"
project_name       = "tf-lab1"
environment        = "dev"
vpc_cidr           = "10.0.0.0/16"
public_subnet_cidr = "10.0.1.0/24"
instance_type      = "t3.micro"
```

**✅ Verify Step 3:**
```bash
ls -la
# Expected files: main.tf, variables.tf, outputs.tf, terraform.tfvars
```

---

#### Step 4: Initialize Terraform

```bash
terraform init
```

**Expected Output:**
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...
- Installed hashicorp/aws v5.x.x (signed by HashiCorp)

Terraform has been successfully initialized!
```

```bash
# Inspect what was created
ls -la .terraform/
# Expected: providers directory with the AWS provider binary

cat .terraform.lock.hcl
# Expected: Provider version lock file with hashes
```

**✅ Verify Step 4:**
```bash
terraform version
# Should show Terraform version and provider versions
```

---

#### Step 5: Validate and Plan

```bash
# Validate the configuration syntax
terraform validate
# Expected: Success! The configuration is valid.

# Format the code
terraform fmt
# Expected: Lists any files that were reformatted (or nothing if already formatted)

# Generate and review the execution plan
terraform plan
```

**Expected Plan Output (abbreviated):**
```
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami           = "ami-xxxxxxxxxxxxxxxxx"
      + instance_type = "t3.micro"
      ...
    }

  # aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + cidr_block = "10.0.0.0/16"
      ...
    }

Plan: 7 to add, 0 to change, 0 to destroy.
```

```bash
# Save the plan to a file (best practice)
terraform plan -out=tfplan
```

**✅ Verify Step 5:**
```bash
# Inspect the saved plan
terraform show tfplan
# Expected: Detailed plan output showing all resources to be created
```

---

#### Step 6: Apply the Configuration

```bash
# Apply using the saved plan
terraform apply tfplan
```

**Expected Output:**
```
aws_vpc.main: Creating...
aws_vpc.main: Creation complete after 2s [id=vpc-xxxxxxxxxxxxxxxxx]
aws_internet_gateway.main: Creating...
aws_subnet.public: Creating...
...
aws_instance.web: Still creating... [30s elapsed]
aws_instance.web: Creation complete after 45s [id=i-xxxxxxxxxxxxxxxxx]

Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:

instance_id        = "i-xxxxxxxxxxxxxxxxx"
instance_public_ip = "54.x.x.x"
vpc_id             = "vpc-xxxxxxxxxxxxxxxxx"
web_url            = "http://54.x.x.x"
```

```bash
# View the Terraform state
terraform show

# List all resources in state
terraform state list
# Expected:
# data.aws_ami.amazon_linux_2023
# aws_instance.web
# aws_internet_gateway.main
# aws_route_table.public
# aws_route_table_association.public
# aws_security_group.web
# aws_subnet.public
# aws_vpc.main

# Inspect a specific resource
terraform state show aws_instance.web
```

**✅ Verify Step 6:**
```bash
# Get the web URL from outputs
terraform output web_url

# Test the web server (wait ~60 seconds for user_data to run)
curl $(terraform output -raw web_url)
# Expected: <h1>Hello from Terraform Lab 1!</h1>
```

---

#### Step 7: Modify Infrastructure (Terraform Workflow)

```bash
# Add a tag to the EC2 instance by editing main.tf
# In the aws_instance.web resource, add to tags:
```

Edit `main.tf` to add a tag to the instance:
```hcl
resource "aws_instance" "web" {
  # ... existing config ...
  tags = {
    Name    = "${var.project_name}-web-server"
    Updated = "true"          # <-- Add this line
  }
}
```

```bash
# See what will change
terraform plan
# Expected: 1 to change (in-place tag update)

# Apply the change
terraform apply -auto-approve
# Expected: aws_instance.web: Modifying... [id=i-xxxxxxxxxxxxxxxxx]
# Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

**✅ Verify Step 7:**
```bash
# Verify the tag was applied
aws ec2 describe-tags \
  --filters "Name=resource-id,Values=$(terraform output -raw instance_id)" \
  --query 'Tags[*].{Key:Key,Value:Value}' \
  --output table
```

---

### Verification

Run the following commands to confirm successful lab completion:

```bash
# 1. Verify all resources exist in state
terraform state list | wc -l
# Expected: 8 resources

# 2. Verify outputs are populated
terraform output
# Expected: All 6 outputs with real values

# 3. Verify the web server is accessible
curl -s $(terraform output -raw web_url) | grep "Hello from Terraform"
# Expected: <h1>Hello from Terraform Lab 1!</h1>

# 4. Verify VPC in AWS CLI
aws ec2 describe-vpcs \
  --filters "Name=tag:Project,Values=tf-lab1" \
  --query 'Vpcs[*].{VpcId:VpcId,CIDR:CidrBlock,State:State}' \
  --output table

# 5. Verify EC2 instance is running
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=tf-lab1" "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,State:State.Name,IP:PublicIpAddress}' \
  --output table
```

---

### Cleanup

>