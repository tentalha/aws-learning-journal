# Terraform

## What is it?

Terraform is an open-source **Infrastructure as Code (IaC)** tool developed by HashiCorp that enables users to define, provision, and manage cloud infrastructure using a declarative configuration language called **HCL (HashiCorp Configuration Language)**. While not an AWS-native service, Terraform is one of the most widely adopted IaC tools in the AWS ecosystem and is officially supported by AWS through the **AWS Provider for Terraform**.

- **Category:** Infrastructure as Code (IaC) / DevOps Tooling
- **Developed by:** HashiCorp (acquired by IBM in 2024)
- **License:** Business Source License 1.1 (BSL) as of Terraform v1.6+; OpenTofu is the open-source fork under MPL 2.0
- **Current Stable Version:** Terraform v1.9.x (as of 2024)
- **AWS Provider Version:** `hashicorp/aws` ~> 5.x
- **AWS Native Equivalent:** AWS CloudFormation / AWS CDK

Terraform allows engineers to write human-readable configuration files that describe the desired state of infrastructure. It then calculates the difference between the current state and the desired state and applies only the necessary changes — a concept known as **idempotent infrastructure management**.

```hcl
# Simple example: Defining an S3 bucket in Terraform
resource "aws_s3_bucket" "example" {
  bucket = "my-production-bucket-2024"

  tags = {
    Environment = "Production"
    Team        = "Platform"
  }
}
```

---

## Why do we need it?

### The Problem It Solves

Before IaC tools like Terraform, infrastructure was provisioned manually through AWS Console clicks, shell scripts, or ad-hoc CLI commands. This approach introduced significant operational risks:

| Problem | Impact |
|---|---|
| Manual provisioning | Human error, inconsistency across environments |
| No version control | No audit trail, no rollback capability |
| Snowflake servers | Environments drift over time, impossible to reproduce |
| Slow provisioning | Hours to days to stand up new environments |
| Multi-cloud complexity | Different tools for AWS, Azure, GCP |
| Knowledge silos | Only one person knows how infrastructure was built |

### Business Scenarios Where Terraform Excels

**1. Multi-Environment Management**
A fintech company needs identical staging, QA, and production environments on AWS. With Terraform modules and workspaces, they can provision all three environments from a single codebase, ensuring consistency and reducing configuration drift.

**2. Disaster Recovery Setup**
An e-commerce platform needs to replicate their entire AWS infrastructure in a secondary region (us-east-1 → us-west-2) within hours of a disaster event. Terraform can reproduce the entire stack by changing a single variable.

**3. Startup Scaling**
A SaaS startup begins on a simple EC2 + RDS setup. As they grow, they need to migrate to EKS, add CloudFront, introduce WAF, and configure complex IAM roles. Terraform enables incremental infrastructure evolution with a full history of every change.

**4. Compliance and Governance**
A healthcare company (HIPAA-compliant) needs to enforce encryption at rest, specific VPC configurations, and audit logging across all AWS resources. Terraform combined with Sentinel (HashiCorp's policy engine) or OPA enforces these standards programmatically.

**5. Cost Management**
A data engineering team needs to spin up expensive EMR clusters for nightly batch jobs and destroy them afterward. Terraform's `apply` and `destroy` commands automate this lifecycle.

**6. Platform Engineering**
A large enterprise platform team needs to provide self-service infrastructure to 50+ development teams. They publish Terraform modules to a private registry that encode organizational best practices, allowing dev teams to provision compliant infrastructure without deep AWS expertise.

---

## Internal Working

### The Terraform Execution Model

Terraform operates through a well-defined lifecycle that transforms HCL configuration into real infrastructure:

```
┌─────────────────────────────────────────────────────────────────┐
│                    TERRAFORM EXECUTION FLOW                      │
│                                                                   │
│  .tf Files  →  terraform init  →  terraform plan  →  terraform apply │
│     │               │                  │                  │       │
│  HCL Config    Download          Diff Engine         API Calls   │
│  Variables     Providers         State Compare       to AWS      │
│  Modules       Lock File         Execution Plan      State Update│
└─────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Internal Process

#### 1. Configuration Parsing (Loading Phase)
Terraform reads all `.tf` files in the working directory and builds an **Abstract Syntax Tree (AST)** from the HCL. It resolves:
- Variable references (`var.`)
- Local values (`local.`)
- Data source references (`data.`)
- Resource references (`resource.`)
- Module calls

#### 2. Provider Initialization (`terraform init`)
- Downloads provider plugins (e.g., `hashicorp/aws`) from the **Terraform Registry** or a private mirror
- Stores providers in `.terraform/providers/`
- Creates or validates `.terraform.lock.hcl` for dependency locking
- Initializes the backend (where state is stored)

```
.terraform/
├── providers/
│   └── registry.terraform.io/
│       └── hashicorp/
│           └── aws/
│               └── 5.x.x/
│                   └── terraform-provider-aws_v5.x.x_x5
└── terraform.tfstate (local backend only)
```

#### 3. Dependency Graph Construction
Terraform builds a **Directed Acyclic Graph (DAG)** of all resources based on explicit (`depends_on`) and implicit (reference-based) dependencies. This graph determines:
- **Parallelism:** Independent resources are created simultaneously (default parallelism: 10)
- **Order:** Dependent resources are created sequentially

```
Example Dependency Graph:
aws_vpc → aws_subnet → aws_instance
                    ↗
aws_security_group
```

#### 4. State Comparison (`terraform plan`)
- Terraform reads the **current state** from the state file (local or remote)
- It calls AWS APIs via the provider to **refresh** the actual current state of resources
- It computes the **diff** between desired state (HCL) and current state
- Outputs a human-readable **execution plan** showing what will be created, modified, or destroyed

#### 5. Apply Phase (`terraform apply`)
- Terraform executes the plan by making API calls through the AWS Provider
- The AWS Provider translates Terraform resource definitions into AWS API calls (using the AWS Go SDK v2 internally)
- As each resource is created/updated/destroyed, the **state file is updated atomically**
- State locking prevents concurrent modifications (using DynamoDB for remote state)

#### 6. State Management
The state file (`terraform.tfstate`) is a JSON document that maps Terraform resource addresses to real AWS resource IDs:

```json
{
  "version": 4,
  "terraform_version": "1.9.0",
  "resources": [
    {
      "mode": "managed",
      "type": "aws_s3_bucket",
      "name": "example",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "id": "my-production-bucket-2024",
            "arn": "arn:aws:s3:::my-production-bucket-2024",
            "bucket": "my-production-bucket-2024"
          }
        }
      ]
    }
  ]
}
```

### Provider Architecture

The AWS Provider is a plugin binary that:
1. Implements the **Terraform Plugin Protocol** (gRPC-based for providers using the Plugin Framework)
2. Translates Terraform resource schemas into AWS API calls using the **AWS Go SDK v2**
3. Handles AWS authentication (IAM roles, access keys, SSO, instance profiles)
4. Maps AWS resource attributes to Terraform state attributes

---

## Architecture

### Core Architectural Components

```
┌──────────────────────────────────────────────────────────────────────┐
│                     TERRAFORM ARCHITECTURE                            │
│                                                                        │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────────────────┐   │
│  │  Terraform   │    │   Providers   │    │    Remote Backend       │   │
│  │    Core      │◄──►│  (AWS, Azure, │    │  (S3 + DynamoDB)       │   │
│  │  (Engine)    │    │   GCP, etc.) │    │                        │   │
│  └──────┬───────┘    └──────┬───────┘    └────────────────────────┘   │
│         │                   │                         ▲                │
│         │                   ▼                         │                │
│         │           ┌──────────────┐         ┌────────┴───────┐       │
│         │           │  AWS APIs    │         │  State File    │       │
│         │           │  (EC2, S3,   │         │  (tfstate)     │       │
│         │           │   RDS, etc.) │         └────────────────┘       │
│         │           └──────────────┘                                   │
│         ▼                                                               │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────────────────┐   │
│  │   Modules   │    │  Variables   │    │    Workspaces           │   │
│  │  (Reusable  │    │  & Outputs   │    │  (dev/staging/prod)     │   │
│  │  Components)│    │              │    │                        │   │
│  └─────────────┘    └──────────────┘    └────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

#### 1. Terraform Core
The central engine responsible for:
- Parsing HCL configuration
- Building the dependency graph
- Executing plans and applies
- Managing state

#### 2. Providers
Plugins that bridge Terraform Core and cloud APIs:
- **AWS Provider** (`hashicorp/aws`): 1000+ AWS resource types
- **AWS Cloud Control Provider** (`hashicorp/awscc`): Uses AWS Cloud Control API
- Providers are versioned and locked via `.terraform.lock.hcl`

#### 3. State Backend
Where Terraform stores the state file:

| Backend | Use Case | State Locking |
|---|---|---|
| Local | Development only | No |
| S3 + DynamoDB | Production (most common) | Yes (DynamoDB) |
| Terraform Cloud | Managed solution | Yes (built-in) |
| HashiCorp Vault | Sensitive state | Yes |

#### 4. Modules
Reusable, composable infrastructure components:

```
modules/
├── vpc/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── eks-cluster/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── rds-postgres/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

#### 5. Workspaces
Isolated state environments within the same configuration:
- `terraform workspace new staging`
- `terraform workspace select production`
- Each workspace has its own state file

#### 6. Terraform Registry
Public and private module/provider repositories:
- **Public Registry:** `registry.terraform.io`
- **Private Registry:** Terraform Cloud / Enterprise
- **Self-hosted:** Artifactory, Nexus

### Recommended Project Structure

```
terraform-aws-project/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── production/
├── modules/
│   ├── networking/
│   ├── compute/
│   ├── database/
│   └── security/
├── .terraform.lock.hcl
├── versions.tf
└── README.md
```

---

## Real World Example

### Scenario: Deploying a Production-Grade Three-Tier Web Application on AWS

A retail company needs to deploy a scalable, highly available web application on AWS with:
- VPC with public/private subnets across 3 AZs
- Application Load Balancer
- Auto Scaling Group with EC2 instances
- RDS PostgreSQL (Multi-AZ)
- ElastiCache Redis cluster
- S3 for static assets + CloudFront CDN
- WAF for security
- CloudWatch monitoring

#### Step 1: Initialize Project Structure

```bash
mkdir terraform-retail-app && cd terraform-retail-app
mkdir -p modules/{vpc,compute,database,cdn,security}
mkdir -p environments/{dev,staging,production}
```

#### Step 2: Configure Remote Backend (`backend.tf`)

```hcl
# environments/production/backend.tf
terraform {
  required_version = ">= 1.9.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "retail-app-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/mrk-xxx"
    dynamodb_table = "retail-app-terraform-locks"
  }
}
```

#### Step 3: Create the VPC Module

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.common_tags, {
    Name = "${var.environment}-vpc"
  })
}

resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = merge(var.common_tags, {
    Name = "${var.environment}-public-subnet-${count.index + 1}"
    "kubernetes.io/role/elb" = "1"  # For EKS compatibility
  })
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + length(var.availability_zones))
  availability_zone = var.availability_zones[count.index]

  tags = merge(var.common_tags, {
    Name = "${var.environment}-private-subnet-${count.index + 1}"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(var.common_tags, {
    Name = "${var.environment}-igw"
  })
}

resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"

  tags = merge(var.common_tags, {
    Name = "${var.environment}-nat-eip-${count.index + 1}"
  })
}

resource "aws_nat_gateway" "main" {
  count         = length(var.availability_zones)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  depends_on = [aws_internet_gateway.main]

  tags = merge(var.common_tags, {
    Name = "${var.environment}-nat-gw-${count.index + 1