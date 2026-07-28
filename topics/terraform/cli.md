# Terraform — AWS CLI Commands

> **Note:** Terraform is a HashiCorp IaC tool that interacts with AWS via the AWS provider, not directly through the AWS CLI. However, the AWS CLI is essential for configuring credentials, inspecting state-related AWS resources, bootstrapping backends (S3, DynamoDB), and debugging Terraform-managed infrastructure. This reference covers the AWS CLI commands most commonly used alongside Terraform workflows.

---

## Setup & Configuration

### Prerequisites

Before running Terraform against AWS, the following AWS CLI configurations and IAM permissions are required.

#### Required IAM Permissions (minimum for common Terraform workflows)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "dynamodb:*",
        "iam:*",
        "ec2:*",
        "sts:GetCallerIdentity",
        "sts:AssumeRole"
      ],
      "Resource": "*"
    }
  ]
}
```

#### Configure AWS CLI credentials

```bash
# Configure a named profile for Terraform use
aws configure --profile terraform-dev

# Set environment variables (alternative to profiles)
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-1"
export AWS_PROFILE="terraform-dev"
```

#### Verify identity before running Terraform

```bash
aws sts get-caller-identity --profile terraform-dev
```

**Example Output:**
```json
{
    "UserId": "AIDAIOSFODNN7EXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/terraform-svc-account"
}
```

---

## Core Commands

### 1. Verify AWS Identity (Pre-Terraform Check)

```bash
aws sts get-caller-identity \
  --profile terraform-dev \
  --output json
```

**What it does:** Confirms which AWS account and IAM principal Terraform will use. Always run this before `terraform plan` or `terraform apply` to avoid deploying to the wrong account.

---

### 2. Create S3 Bucket for Terraform Remote State

```bash
aws s3api create-bucket \
  --bucket my-terraform-state-123456789012 \
  --region us-east-1 \
  --profile terraform-dev
```

**What it does:** Creates the S3 bucket used as the Terraform remote backend. The bucket name must be globally unique; appending the account ID is a common pattern.

**Example Output:**
```json
{
    "Location": "/my-terraform-state-123456789012"
}
```

---

### 3. Enable S3 Bucket Versioning for State Files

```bash
aws s3api put-bucket-versioning \
  --bucket my-terraform-state-123456789012 \
  --versioning-configuration Status=Enabled \
  --profile terraform-dev
```

**What it does:** Enables versioning on the state bucket so every `terraform apply` creates a new state version, allowing rollback to previous states.

---

### 4. Enable S3 Server-Side Encryption for State Bucket

```bash
aws s3api put-bucket-encryption \
  --bucket my-terraform-state-123456789012 \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123def456"
      },
      "BucketKeyEnabled": true
    }]
  }' \
  --profile terraform-dev
```

**What it does:** Encrypts all Terraform state files at rest using AWS KMS. This is a security best practice for state files that may contain sensitive resource attributes.

---

### 5. Block Public Access on State Bucket

```bash
aws s3api put-public-access-block \
  --bucket my-terraform-state-123456789012 \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true" \
  --profile terraform-dev
```

**What it does:** Prevents the Terraform state bucket from ever being made public, protecting sensitive infrastructure state data.

---

### 6. Create DynamoDB Table for Terraform State Locking

```bash
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1 \
  --profile terraform-dev
```

**What it does:** Creates the DynamoDB table used by Terraform for state locking and consistency checking. Prevents concurrent `terraform apply` runs from corrupting the state file.

**Example Output:**
```json
{
    "TableDescription": {
        "TableName": "terraform-state-lock",
        "TableStatus": "CREATING",
        "TableArn": "arn:aws:dynamodb:us-east-1:123456789012:table/terraform-state-lock"
    }
}
```

---

### 7. List All Objects in the Terraform State Bucket

```bash
aws s3 ls s3://my-terraform-state-123456789012/ \
  --recursive \
  --human-readable \
  --profile terraform-dev
```

**What it does:** Lists all state files and their versions stored in the remote backend bucket, useful for auditing which workspaces and modules have state.

**Example Output:**
```
2024-01-15 10:23:45    2.3 KiB env:/dev/vpc/terraform.tfstate
2024-01-15 11:05:12    8.7 KiB env:/prod/app/terraform.tfstate
2024-01-16 09:15:33    1.1 KiB global/iam/terraform.tfstate
```

---

### 8. Download a Specific Terraform State File

```bash
aws s3 cp \
  s3://my-terraform-state-123456789012/env:/prod/app/terraform.tfstate \
  ./terraform.tfstate.backup \
  --profile terraform-dev
```

**What it does:** Downloads a copy of a remote state file for inspection or manual recovery. Useful when `terraform state` commands are unavailable.

---

### 9. Assume an IAM Role for Cross-Account Terraform Deployments

```bash
aws sts assume-role \
  --role-arn "arn:aws:iam::987654321098:role/TerraformDeployRole" \
  --role-session-name "terraform-prod-deploy-$(date +%s)" \
  --duration-seconds 3600 \
  --profile terraform-dev \
  --output json
```

**What it does:** Assumes a role in a target account (e.g., production) to allow Terraform to deploy cross-account resources. The returned credentials are exported as environment variables.

**Example Output:**
```json
{
    "Credentials": {
        "AccessKeyId": "ASIAIOSFODNN7EXAMPLE",
        "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
        "SessionToken": "AQoDYXdzEJr...",
        "Expiration": "2024-01-16T14:30:00Z"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAIOSFODNN7EXAMPLE:terraform-prod-deploy-1705412200",
        "Arn": "arn:aws:iam::987654321098:role/TerraformDeployRole"
    }
}
```

---

### 10. Check DynamoDB Lock Table for Stuck Locks

```bash
aws dynamodb scan \
  --table-name terraform-state-lock \
  --profile terraform-dev \
  --output table
```

**What it does:** Scans the DynamoDB lock table to find any active or stuck Terraform state locks. If a `terraform apply` was interrupted, a lock entry may remain and block future runs.

**Example Output:**
```
-----------------------------------------------------------------------
|                              Scan                                   |
+---------------------------------------------------------------------+
|  LockID                          |  Info                           |
|  my-terraform-state/prod.tfstate |  {"ID":"abc-123","Operation":.. |
+---------------------------------------------------------------------+
```

---

### 11. Delete a Stuck Terraform State Lock

```bash
aws dynamodb delete-item \
  --table-name terraform-state-lock \
  --key '{"LockID": {"S": "my-terraform-state-123456789012/env:/prod/app/terraform.tfstate"}}' \
  --profile terraform-dev
```

**What it does:** Manually removes a stuck state lock from DynamoDB. Use this only after confirming no other Terraform process is running. Equivalent to `terraform force-unlock` but done at the AWS layer.

---

### 12. List Versions of a State File (Audit Trail)

```bash
aws s3api list-object-versions \
  --bucket my-terraform-state-123456789012 \
  --prefix "env:/prod/app/terraform.tfstate" \
  --profile terraform-dev \
  --output json
```

**What it does:** Lists all historical versions of a specific state file. Useful for identifying a previous good state to restore after a failed apply.

**Example Output:**
```json
{
    "Versions": [
        {
            "VersionId": "Kb.yMLgIIpeJwkn6By5UZobVRSjpkIuj",
            "LastModified": "2024-01-16T09:15:33.000Z",
            "Size": 8734,
            "IsLatest": true
        },
        {
            "VersionId": "3sL4kqtJlcpXroDTDmJ+rmSpXd3dIbrHY",
            "LastModified": "2024-01-15T11:05:12.000Z",
            "Size": 8201,
            "IsLatest": false
        }
    ]
}
```

---

### 13. Restore a Previous State File Version

```bash
aws s3api copy-object \
  --bucket my-terraform-state-123456789012 \
  --copy-source "my-terraform-state-123456789012/env:/prod/app/terraform.tfstate?versionId=3sL4kqtJlcpXroDTDmJ+rmSpXd3dIbrHY" \
  --key "env:/prod/app/terraform.tfstate" \
  --profile terraform-dev
```

**What it does:** Restores a previous version of the state file by copying it as the new current version. This is the AWS-level equivalent of rolling back Terraform state.

---

### 14. Check if an ECR Repository Exists (Pre-Terraform Check)

```bash
aws ecr describe-repositories \
  --repository-names my-app-repo \
  --region us-east-1 \
  --profile terraform-dev \
  --output json 2>&1 || echo "Repository does not exist"
```

**What it does:** Checks for pre-existing AWS resources before running Terraform import commands. Useful in scripts that conditionally run `terraform import`.

---

### 15. Get Current AWS Region and Account (Terraform Variable Injection)

```bash
ACCOUNT_ID=$(aws sts get-caller-identity \
  --query "Account" \
  --output text \
  --profile terraform-dev)

REGION=$(aws configure get region --profile terraform-dev)

echo "Account: $ACCOUNT_ID | Region: $REGION"

# Pass to Terraform
terraform plan \
  -var="aws_account_id=$ACCOUNT_ID" \
  -var="aws_region=$REGION"
```

**What it does:** Dynamically retrieves account ID and region, then injects them as Terraform variables. Avoids hardcoding account-specific values in `.tfvars` files.

---

## Common Operations

### Create Operations

#### Create KMS Key for Terraform Secrets Encryption

```bash
aws kms create-key \
  --description "Terraform state encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS \
  --tags TagKey=ManagedBy,TagValue=Terraform \
  --profile terraform-dev \
  --output json
```

#### Create IAM Role for Terraform CI/CD Pipeline

```bash
aws iam create-role \
  --role-name TerraformCICDRole \
  --assume-role-policy-document file://trust-policy.json \
  --description "Role assumed by CI/CD pipeline for Terraform deployments" \
  --tags Key=ManagedBy,Value=Terraform \
  --profile terraform-dev
```

#### Create SSM Parameter for Terraform Remote State Config

```bash
aws ssm put-parameter \
  --name "/terraform/backend/state-bucket" \
  --value "my-terraform-state-123456789012" \
  --type SecureString \
  --description "Terraform remote state S3 bucket name" \
  --tags Key=ManagedBy,Value=Terraform \
  --profile terraform-dev
```

---

### Read / Describe Operations

#### Describe Terraform-Managed EC2 Instances

```bash
aws ec2 describe-instances \
  --filters "Name=tag:ManagedBy,Values=Terraform" \
             "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].{ID:InstanceId,Type:InstanceType,State:State.Name,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table \
  --profile terraform-dev
```

#### Read SSM Parameter (Terraform Backend Config)

```bash
aws ssm get-parameter \
  --name "/terraform/backend/state-bucket" \
  --with-decryption \
  --query "Parameter.Value" \
  --output text \
  --profile terraform-dev
```

#### Describe Terraform State Bucket Policy

```bash
aws s3api get-bucket-policy \
  --bucket my-terraform-state-123456789012 \
  --profile terraform-dev \
  --output json | python3 -m json.tool
```

---

### Update Operations

#### Update DynamoDB Table Billing Mode

```bash
aws dynamodb update-table \
  --table-name terraform-state-lock \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 \
  --profile terraform-dev
```

#### Update S3 Bucket Policy to Restrict Terraform State Access

```bash
aws s3api put-bucket-policy \
  --bucket my-terraform-state-123456789012 \
  --policy file://state-bucket-policy.json \
  --profile terraform-dev
```

#### Rotate KMS Key for State Encryption

```bash
aws kms enable-key-rotation \
  --key-id "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123def456" \
  --profile terraform-dev
```

---

### Delete Operations

#### Delete a Specific State File Version

```bash
aws s3api delete-object \
  --bucket my-terraform-state-123456789012 \
  --key "env:/dev/app/terraform.tfstate" \
  --version-id "Kb.yMLgIIpeJwkn6By5UZobVRSjpkIuj" \
  --profile terraform-dev
```

#### Delete the DynamoDB Lock Table (Teardown)

```bash
aws dynamodb delete-table \
  --table-name terraform-state-lock \
  --profile terraform-dev
```

#### Remove Terraform State Bucket (Force Empty First)

```bash
# Empty