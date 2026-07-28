# Terraform — Interview Questions

---

## Easy

### 1. What is Terraform, and what problem does it solve?

**Answer:**
Terraform is an open-source Infrastructure as Code (IaC) tool created by HashiCorp. It allows engineers to define, provision, and manage cloud and on-premises infrastructure using a declarative configuration language called HCL (HashiCorp Configuration Language).

**Problems it solves:**
- Manual, error-prone infrastructure provisioning via consoles or CLIs
- Lack of version control and auditability for infrastructure changes
- Difficulty reproducing environments consistently across dev/staging/prod
- Managing infrastructure drift between desired and actual state
- Multi-cloud and multi-provider infrastructure management from a single tool

---

### 2. What is the difference between `terraform plan` and `terraform apply`?

**Answer:**

| Command | Purpose |
|---|---|
| `terraform plan` | Previews the changes Terraform will make. It compares the current state with the desired configuration and outputs an execution plan. No changes are made to real infrastructure. |
| `terraform apply` | Executes the changes described in the plan. It provisions, modifies, or destroys resources as needed to match the desired configuration. |

**Best Practice:** Always run `terraform plan` before `terraform apply` to review changes, especially in production. You can also save a plan file: `terraform plan -out=tfplan` and apply it deterministically with `terraform apply tfplan`.

---

### 3. What is a Terraform provider, and can you give an example?

**Answer:**
A **provider** is a plugin that enables Terraform to interact with APIs of cloud platforms, SaaS services, or other infrastructure targets. Providers are responsible for understanding API interactions and exposing resources and data sources.

**Example — AWS Provider:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

Common providers include:
- `hashicorp/aws` — Amazon Web Services
- `hashicorp/azurerm` — Microsoft Azure
- `hashicorp/google` — Google Cloud Platform
- `hashicorp/kubernetes` — Kubernetes clusters
- `hashicorp/vault` — HashiCorp Vault

---

### 4. What is Terraform state, and why is it important?

**Answer:**
Terraform **state** is a JSON file (`terraform.tfstate`) that maps your Terraform configuration to real-world infrastructure resources. It acts as the source of truth for what Terraform knows about the infrastructure it manages.

**Why it's important:**
- **Tracking:** Terraform uses state to determine what resources exist and what changes need to be made
- **Dependency mapping:** State stores metadata and resource dependencies
- **Performance:** For large infrastructures, state allows Terraform to avoid querying every resource on every run
- **Collaboration:** Remote state (e.g., in S3) enables teams to share a consistent view of infrastructure

**Risk:** If state is lost or corrupted, Terraform loses track of existing resources, which can lead to duplicate resource creation or inability to manage existing infrastructure.

---

### 5. What is the purpose of `terraform init`?

**Answer:**
`terraform init` is the first command run in any new or cloned Terraform project. It initializes the working directory and performs the following tasks:

1. **Downloads providers** defined in the configuration into `.terraform/providers/`
2. **Downloads modules** referenced in the configuration into `.terraform/modules/`
3. **Configures the backend** for remote state storage (e.g., S3, Terraform Cloud)
4. **Creates a lock file** (`.terraform.lock.hcl`) to pin provider versions for consistency

```bash
terraform init
# With backend reconfiguration
terraform init -reconfigure
# With backend migration
terraform init -migrate-state
```

It must be re-run whenever you add new providers, modules, or change backend configuration.

---

## Medium

### 1. Explain Terraform modules. How do you create and use them?

**Answer:**
A **module** is a container for multiple Terraform resources that are used together. Every Terraform configuration is technically a module (the "root module"). Child modules are reusable, self-contained packages of Terraform configuration.

**Why use modules:**
- Encapsulate and reuse common infrastructure patterns (e.g., a standard VPC setup)
- Enforce organizational standards and best practices
- Reduce code duplication across environments
- Simplify complex configurations by abstracting details

**Creating a module (directory structure):**
```
modules/
  vpc/
    main.tf
    variables.tf
    outputs.tf
    README.md
```

**`modules/vpc/variables.tf`:**
```hcl
variable "cidr_block" {
  description = "CIDR block for the VPC"
  type        = string
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}
```

**`modules/vpc/main.tf`:**
```hcl
resource "aws_vpc" "this" {
  cidr_block = var.cidr_block

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}
```

**`modules/vpc/outputs.tf`:**
```hcl
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.this.id
}
```

**Calling the module from root:**
```hcl
module "production_vpc" {
  source      = "./modules/vpc"
  cidr_block  = "10.0.0.0/16"
  environment = "production"
}

# Consuming the module output
resource "aws_subnet" "public" {
  vpc_id     = module.production_vpc.vpc_id
  cidr_block = "10.0.1.0/24"
}
```

Modules can also be sourced from the **Terraform Registry**, **GitHub**, or **S3** buckets.

---

### 2. What is remote state, and how do you configure it in AWS?

**Answer:**
**Remote state** stores `terraform.tfstate` in a shared, remote location rather than locally. This is essential for team collaboration, security, and reliability.

**Benefits of remote state:**
- Enables team collaboration without state file conflicts
- Provides state locking to prevent concurrent modifications
- Stores state securely with encryption at rest
- Enables cross-workspace state data sharing via `terraform_remote_state`

**AWS Configuration (S3 + DynamoDB):**

First, create the S3 bucket and DynamoDB table (often bootstrapped separately):
```hcl
# bootstrap/main.tf
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-company-terraform-state"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**Backend configuration:**
```hcl
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "production/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

**State locking:** DynamoDB provides locking — when `terraform apply` runs, it writes a lock record. If another process tries to apply simultaneously, it fails with a lock error, preventing state corruption.

---

### 3. What are Terraform workspaces, and when should you use them?

**Answer:**
Terraform **workspaces** allow you to maintain multiple distinct state files for the same configuration within the same backend. Each workspace has its own isolated state.

**Workspace commands:**
```bash
terraform workspace list       # List all workspaces
terraform workspace new dev    # Create a new workspace
terraform workspace select prod # Switch to a workspace
terraform workspace show       # Show current workspace
terraform workspace delete dev  # Delete a workspace
```

**Using workspace in configuration:**
```hcl
locals {
  environment_config = {
    dev = {
      instance_type = "t3.micro"
      min_capacity  = 1
    }
    staging = {
      instance_type = "t3.small"
      min_capacity  = 2
    }
    prod = {
      instance_type = "t3.large"
      min_capacity  = 4
    }
  }

  config = local.environment_config[terraform.workspace]
}

resource "aws_instance" "app" {
  instance_type = local.config.instance_type
  # ...
}
```

**When to use workspaces:**
- Managing multiple environments (dev/staging/prod) with the same codebase
- Feature branch deployments (ephemeral environments)
- Testing infrastructure changes in isolation

**When NOT to use workspaces:**
- When environments have significantly different configurations
- When strict isolation between environments is required (separate AWS accounts preferred)
- Large teams where separate state backends per environment are safer

**Best practice:** For true environment isolation at scale, prefer **separate directories** or **separate AWS accounts** with separate backends over workspaces.

---

### 4. Explain the Terraform resource lifecycle and the `lifecycle` meta-argument.

**Answer:**
Every Terraform resource goes through a lifecycle: **Create → Read → Update → Delete**. The `lifecycle` meta-argument lets you customize this behavior.

**Available lifecycle arguments:**

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"

  lifecycle {
    # Prevent accidental destruction of critical resources
    prevent_destroy = true

    # Create new resource before destroying old one (zero-downtime replacements)
    create_before_destroy = true

    # Ignore changes to specific attributes (e.g., managed outside Terraform)
    ignore_changes = [
      tags["LastModified"],
      user_data,
    ]

    # Custom condition checks (Terraform 1.2+)
    precondition {
      condition     = var.instance_type != "t2.micro"
      error_message = "Production must not use t2.micro instances."
    }

    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP."
    }
  }
}
```

**Key use cases:**

| Argument | Use Case |
|---|---|
| `prevent_destroy` | Protect production databases, stateful resources from accidental deletion |
| `create_before_destroy` | Zero-downtime replacements for Auto Scaling Groups, certificates |
| `ignore_changes` | Resources with external automation (e.g., ASG managing instance count) |
| `replace_triggered_by` | Force replacement when a dependent resource changes |

**`replace_triggered_by` example (Terraform 1.2+):**
```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    replace_triggered_by = [
      aws_security_group.web.id
    ]
  }
}
```

---

### 5. How do you manage sensitive data in Terraform?

**Answer:**
Managing secrets in Terraform requires careful consideration to avoid exposing sensitive values in state files, logs, or version control.

**1. Mark variables as sensitive:**
```hcl
variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true  # Redacts value in plan/apply output
}
```

**2. Use AWS Secrets Manager or SSM Parameter Store:**
```hcl
# Fetch secret from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/rds/master-password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  # ...
}
```

**3. Use SSM Parameter Store:**
```hcl
data "aws_ssm_parameter" "db_password" {
  name            = "/production/db/password"
  with_decryption = true
}
```

**4. Environment variables for provider credentials:**
```bash
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export TF_VAR_db_password="..."  # Terraform reads TF_VAR_ prefixed vars
```

**5. Encrypt state at rest:**
```hcl
terraform {
  backend "s3" {
    encrypt = true
    # ...
  }
}
```

**6. Use Vault provider for dynamic secrets:**
```hcl
provider "vault" {}

data "vault_generic_secret" "db_creds" {
  path = "secret/production/database"
}
```

**Important caveat:** Even with `sensitive = true`, values are still stored **in plain text in the state file**. Always encrypt your state backend and restrict access using IAM policies.

---

## Hard

### 1. Explain Terraform's dependency graph and how it handles parallelism. How can you influence execution order?

**Answer:**
Terraform builds a **directed acyclic graph (DAG)** of all resources before executing any changes. This graph determines the order of operations and enables parallel execution.

**How the graph is built:**
1. Terraform parses all `.tf` files and identifies resources, data sources, and their relationships
2. **Implicit dependencies** are detected via references (e.g., `aws_subnet.main.id` creates a dependency on `aws_subnet.main`)
3. **Explicit dependencies** are declared via `depends_on`
4. Terraform walks the graph, executing independent nodes in parallel (default: 10 parallel operations)

**Viewing the graph:**
```bash
terraform graph | dot -Tsvg > graph.svg
```

**Implicit dependency example:**
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id  # Implicit dependency — subnet waits for VPC
  cidr_block = "10.0.1.0/24"
}
```

**Explicit dependency with `depends_on`:**
Use `depends_on` when a dependency exists that Terraform cannot detect through references (e.g., IAM policy propagation):

```hcl
resource "aws_iam_role_policy_attachment" "lambda_policy" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_lambda_function" "processor" {
  function_name = "data-processor"
  role          = aws_iam_role.lambda_exec.arn

  # IAM changes can take seconds to propagate; explicit dependency ensures
  # the policy is attached before Lambda is created
  depends_on = [aws_iam_role_policy_attachment.lambda_policy]
}
```

**Controlling parallelism:**
```bash
terraform apply -parallelism=20  # Increase parallel operations
terraform apply -parallelism=1   # Sequential execution (debugging)
```

**Graph traversal rules:**
- **Create:** Traverse graph in dependency order (parents first)
- **Destroy:** Traverse graph in reverse dependency order (children first)
- Resources with no dependencies on each other execute in parallel

**Pitfall — `depends_on` on modules:**
```hcl
module "database" {
  source = "./modules/rds"
  # ...
}

module "application" {
  source     = "./modules/ecs"
  depends_on = [module.database]  # Entire module waits for entire database module
  # ...
}
```
Using `depends_on` on a module forces sequential execution of the entire module, which can significantly slow down applies. Prefer explicit