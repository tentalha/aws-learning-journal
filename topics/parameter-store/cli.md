# Parameter Store — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using AWS CLI with Parameter Store, ensure the following are in place:

**AWS CLI Version Check**
```bash
aws --version
# Recommended: aws-cli/2.x.x or higher
```

**Configure AWS CLI**
```bash
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

### Required IAM Permissions

Attach the following IAM policy to your user or role for full Parameter Store access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath",
        "ssm:PutParameter",
        "ssm:DeleteParameter",
        "ssm:DeleteParameters",
        "ssm:DescribeParameters",
        "ssm:AddTagsToResource",
        "ssm:ListTagsForResource",
        "ssm:RemoveTagsFromResource",
        "ssm:GetParameterHistory",
        "ssm:LabelParameterVersion"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:GenerateDataKey",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123def456"
    }
  ]
}
```

> **Note:** Parameter Store is part of AWS Systems Manager (SSM). All CLI commands use the `aws ssm` prefix.

---

## Core Commands

### 1. Put (Create/Update) a Parameter

```bash
aws ssm put-parameter \
  --name "/myapp/prod/database/password" \
  --value "SuperSecretPassword123!" \
  --type "SecureString" \
  --key-id "alias/my-param-store-key" \
  --description "Production database password for myapp" \
  --tags "Key=Environment,Value=prod" "Key=Application,Value=myapp" \
  --region us-east-1
```

**What it does:** Creates a new SecureString parameter encrypted with a specified KMS key. If the parameter already exists, the command fails unless `--overwrite` is included.

**Example Output:**
```json
{
    "Version": 1,
    "Tier": "Standard"
}
```

---

### 2. Get a Single Parameter

```bash
aws ssm get-parameter \
  --name "/myapp/prod/database/password" \
  --with-decryption \
  --region us-east-1
```

**What it does:** Retrieves a single parameter. `--with-decryption` is required to decrypt `SecureString` parameters.

**Example Output:**
```json
{
    "Parameter": {
        "Name": "/myapp/prod/database/password",
        "Type": "SecureString",
        "Value": "SuperSecretPassword123!",
        "Version": 1,
        "LastModifiedDate": "2024-03-15T10:23:45.123000+00:00",
        "ARN": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/prod/database/password",
        "DataType": "text"
    }
}
```

---

### 3. Get Multiple Parameters at Once

```bash
aws ssm get-parameters \
  --names \
    "/myapp/prod/database/host" \
    "/myapp/prod/database/port" \
    "/myapp/prod/database/username" \
  --with-decryption \
  --region us-east-1
```

**What it does:** Retrieves up to 10 parameters in a single API call. Returns both valid parameters and a list of invalid (not found) parameter names.

**Example Output:**
```json
{
    "Parameters": [
        {
            "Name": "/myapp/prod/database/host",
            "Type": "String",
            "Value": "myapp-prod.cluster-abc123.us-east-1.rds.amazonaws.com",
            "Version": 1,
            "LastModifiedDate": "2024-03-15T10:00:00.000000+00:00",
            "ARN": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/prod/database/host",
            "DataType": "text"
        }
    ],
    "InvalidParameters": [
        "/myapp/prod/database/port"
    ]
}
```

---

### 4. Get Parameters by Path (Hierarchical Retrieval)

```bash
aws ssm get-parameters-by-path \
  --path "/myapp/prod/" \
  --recursive \
  --with-decryption \
  --region us-east-1
```

**What it does:** Retrieves all parameters under a given path hierarchy. `--recursive` includes parameters in sub-paths.

**Example Output:**
```json
{
    "Parameters": [
        {
            "Name": "/myapp/prod/database/host",
            "Type": "String",
            "Value": "myapp-prod.cluster-abc123.us-east-1.rds.amazonaws.com",
            "Version": 1,
            "LastModifiedDate": "2024-03-15T10:00:00.000000+00:00",
            "ARN": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/prod/database/host",
            "DataType": "text"
        },
        {
            "Name": "/myapp/prod/database/password",
            "Type": "SecureString",
            "Value": "SuperSecretPassword123!",
            "Version": 1,
            "LastModifiedDate": "2024-03-15T10:23:45.123000+00:00",
            "ARN": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/prod/database/password",
            "DataType": "text"
        }
    ]
}
```

---

### 5. Describe Parameters (List with Metadata)

```bash
aws ssm describe-parameters \
  --parameter-filters \
    "Key=Path,Option=Recursive,Values=/myapp/prod" \
  --region us-east-1
```

**What it does:** Returns metadata about parameters (not values). Useful for auditing and discovering parameters without retrieving sensitive values.

**Example Output:**
```json
{
    "Parameters": [
        {
            "Name": "/myapp/prod/database/host",
            "Type": "String",
            "KeyId": "",
            "LastModifiedDate": "2024-03-15T10:00:00.000000+00:00",
            "LastModifiedUser": "arn:aws:iam::123456789012:user/devops-engineer",
            "Description": "Production database hostname",
            "Version": 1,
            "Tier": "Standard",
            "Policies": [],
            "DataType": "text"
        }
    ]
}
```

---

### 6. Delete a Parameter

```bash
aws ssm delete-parameter \
  --name "/myapp/prod/database/old-password" \
  --region us-east-1
```

**What it does:** Permanently deletes a single parameter and all its version history. This action is irreversible.

> **Note:** This command returns no output on success (HTTP 200 with empty body).

---

### 7. Delete Multiple Parameters

```bash
aws ssm delete-parameters \
  --names \
    "/myapp/staging/feature-flag-a" \
    "/myapp/staging/feature-flag-b" \
    "/myapp/staging/feature-flag-c" \
  --region us-east-1
```

**What it does:** Deletes up to 10 parameters in a single API call. Returns lists of deleted and invalid (not found) parameter names.

**Example Output:**
```json
{
    "DeletedParameters": [
        "/myapp/staging/feature-flag-a",
        "/myapp/staging/feature-flag-b"
    ],
    "InvalidParameters": [
        "/myapp/staging/feature-flag-c"
    ]
}
```

---

### 8. Overwrite an Existing Parameter

```bash
aws ssm put-parameter \
  --name "/myapp/prod/database/password" \
  --value "NewRotatedPassword456!" \
  --type "SecureString" \
  --key-id "alias/my-param-store-key" \
  --overwrite \
  --region us-east-1
```

**What it does:** Updates the value of an existing parameter. Without `--overwrite`, the command fails if the parameter already exists. Each overwrite creates a new version.

**Example Output:**
```json
{
    "Version": 2,
    "Tier": "Standard"
}
```

---

### 9. Get Parameter Version History

```bash
aws ssm get-parameter-history \
  --name "/myapp/prod/database/password" \
  --with-decryption \
  --region us-east-1
```

**What it does:** Retrieves the full version history of a parameter, including all previous values and metadata.

**Example Output:**
```json
{
    "Parameters": [
        {
            "Name": "/myapp/prod/database/password",
            "Type": "SecureString",
            "KeyId": "alias/my-param-store-key",
            "LastModifiedDate": "2024-03-15T10:23:45.123000+00:00",
            "LastModifiedUser": "arn:aws:iam::123456789012:user/devops-engineer",
            "Description": "Production database password for myapp",
            "Value": "SuperSecretPassword123!",
            "Version": 1,
            "Labels": [],
            "Tier": "Standard",
            "Policies": []
        },
        {
            "Name": "/myapp/prod/database/password",
            "Type": "SecureString",
            "KeyId": "alias/my-param-store-key",
            "LastModifiedDate": "2024-04-01T08:00:00.000000+00:00",
            "LastModifiedUser": "arn:aws:iam::123456789012:user/devops-engineer",
            "Description": "Production database password for myapp",
            "Value": "NewRotatedPassword456!",
            "Version": 2,
            "Labels": ["current"],
            "Tier": "Standard",
            "Policies": []
        }
    ]
}
```

---

### 10. Add Tags to a Parameter

```bash
aws ssm add-tags-to-resource \
  --resource-type "Parameter" \
  --resource-id "/myapp/prod/database/password" \
  --tags \
    "Key=Environment,Value=prod" \
    "Key=Application,Value=myapp" \
    "Key=ManagedBy,Value=terraform" \
    "Key=CostCenter,Value=engineering-platform" \
  --region us-east-1
```

**What it does:** Adds or updates tags on an existing parameter for cost allocation, access control, and resource organization.

---

### 11. List Tags for a Parameter

```bash
aws ssm list-tags-for-resource \
  --resource-type "Parameter" \
  --resource-id "/myapp/prod/database/password" \
  --region us-east-1
```

**What it does:** Returns all tags associated with a specific parameter.

**Example Output:**
```json
{
    "TagList": [
        {
            "Key": "Environment",
            "Value": "prod"
        },
        {
            "Key": "Application",
            "Value": "myapp"
        },
        {
            "Key": "ManagedBy",
            "Value": "terraform"
        }
    ]
}
```

---

### 12. Label a Parameter Version

```bash
aws ssm label-parameter-version \
  --name "/myapp/prod/database/password" \
  --parameter-version 2 \
  --labels "stable" "current" "approved" \
  --region us-east-1
```

**What it does:** Attaches human-readable labels to a specific parameter version. Labels can be used instead of version numbers when retrieving parameters.

**Example Output:**
```json
{
    "InvalidLabels": [],
    "ParameterVersion": 2
}
```

---

### 13. Create a StringList Parameter

```bash
aws ssm put-parameter \
  --name "/myapp/prod/allowed-cidr-blocks" \
  --value "10.0.0.0/8,172.16.0.0/12,192.168.0.0/16" \
  --type "StringList" \
  --description "Allowed CIDR blocks for production VPC" \
  --region us-east-1
```

**What it does:** Creates a `StringList` parameter — a comma-separated list of strings stored as a single parameter. Useful for storing multiple related values.

---

### 14. Create an Advanced Tier Parameter with a Policy

```bash
aws ssm put-parameter \
  --name "/myapp/prod/temp-api-key" \
  --value "TempApiKey789!" \
  --type "SecureString" \
  --tier "Advanced" \
  --policies '[
    {
      "Type": "Expiration",
      "Version": "1.0",
      "Attributes": {
        "Timestamp": "2024-12-31T23:59:59.000Z"
      }
    },
    {
      "Type": "ExpirationNotification",
      "Version": "1.0",
      "Attributes": {
        "Before": "15",
        "Unit": "Days"
      }
    }
  ]' \
  --region us-east-1
```

**What it does:** Creates an Advanced tier parameter (supports values up to 8KB) with an expiration policy. Sends EventBridge notifications 15 days before expiry.

---

### 15. Get a Specific Parameter Version by Label

```bash
aws ssm get-parameter \
  --name "/myapp/prod/database/password:stable" \
  --with-decryption \
  --region us-east-1
```

**What it does:** Retrieves a specific labeled version of a parameter using the `name:label` syntax, enabling blue/green or canary deployment patterns.

---

## Common Operations

### Create Operations

```bash
# Create a plain String parameter
aws ssm put-parameter \
  --name "/myapp/prod/api/base-url" \
  --value "https://api.myapp.com/v2" \
  --type "String" \
  --description "Base URL for the production API" \
  --region us-east-1

# Create a SecureString parameter using the default SSM KMS key
aws ssm put-parameter \
  --name "/myapp/prod/third-party/stripe-api-key" \
  --value "sk_live_abc123xyz789" \
  --type "SecureString" \
  --description "Stripe live API key" \
  --region us-east-1

# Create parameter with an AMI ID (special DataType for EC2 AMI validation)
aws ssm put-parameter \
  --name "/myapp/prod/ami/golden-image" \
  --value "ami-0abcdef1234567890" \
  --type "String" \
  --data-type "aws:ec2:image" \
  --description "Validated golden AMI for production EC2 instances" \
  --region us-east-1
```

---

### Read Operations

```bash
# Get a parameter value only (using query)
aws ssm get-parameter \
  --name "/myapp/prod/api/base-url" \
  --query "Parameter.Value" \
  --output text \
  --region us-