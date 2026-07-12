# IAM — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using IAM CLI commands, ensure the following are in place:

- **AWS CLI version 2.x** installed and configured (`aws --version`)
- **Credentials configured** via `aws configure` or environment variables
- **IAM permissions** — the calling identity needs appropriate IAM permissions

### Required IAM Permissions (Minimum)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:Get*",
        "iam:List*",
        "iam:CreateUser",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

### Initial Configuration

```bash
# Configure AWS CLI with credentials
aws configure

# Verify current identity
aws sts get-caller-identity

# Set default output format to JSON
aws configure set output json

# Set default region (IAM is global, but region is still used for endpoint routing)
aws configure set region us-east-1
```

### Example: Verify Caller Identity Output

```json
{
    "UserId": "AIDAIOSFODNN7EXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/admin-user"
}
```

---

## Core Commands

### 1. Create an IAM User

```bash
aws iam create-user \
  --user-name my-app-user \
  --tags Key=Environment,Value=Production Key=Team,Value=Backend
```

**What it does:** Creates a new IAM user with optional tags. Does not create credentials automatically.

**Example Output:**
```json
{
    "User": {
        "Path": "/",
        "UserName": "my-app-user",
        "UserId": "AIDAIOSFODNN7EXAMPLE",
        "Arn": "arn:aws:iam::123456789012:user/my-app-user",
        "CreateDate": "2024-01-15T10:30:00+00:00"
    }
}
```

---

### 2. Create an IAM Role

```bash
aws iam create-role \
  --role-name my-ec2-role \
  --assume-role-policy-document file://trust-policy.json \
  --description "Role for EC2 instances to access S3" \
  --tags Key=Environment,Value=Production
```

**`trust-policy.json` example:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**What it does:** Creates an IAM role with a trust policy defining who can assume it.

---

### 3. Attach a Managed Policy to a Role

```bash
aws iam attach-role-policy \
  --role-name my-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

**What it does:** Attaches an AWS managed or customer-managed policy to a role.

---

### 4. Create an Inline Policy for a Role

```bash
aws iam put-role-policy \
  --role-name my-ec2-role \
  --policy-name my-inline-s3-policy \
  --policy-document file://inline-policy.json
```

**`inline-policy.json` example:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    }
  ]
}
```

**What it does:** Embeds an inline policy directly into a role (not reusable across other entities).

---

### 5. Create an IAM Group

```bash
aws iam create-group \
  --group-name developers \
  --path /teams/
```

**What it does:** Creates an IAM group to organize users and apply shared permissions.

---

### 6. Add a User to a Group

```bash
aws iam add-user-to-group \
  --group-name developers \
  --user-name my-app-user
```

**What it does:** Adds an existing IAM user to an IAM group, granting them the group's permissions.

---

### 7. Create an Access Key for a User

```bash
aws iam create-access-key \
  --user-name my-app-user
```

**What it does:** Creates programmatic access credentials (Access Key ID + Secret Access Key) for an IAM user.

> ⚠️ **Important:** The `SecretAccessKey` is only shown once. Store it securely immediately.

**Example Output:**
```json
{
    "AccessKey": {
        "UserName": "my-app-user",
        "AccessKeyId": "AKIAIOSFODNN7EXAMPLE",
        "Status": "Active",
        "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
        "CreateDate": "2024-01-15T10:35:00+00:00"
    }
}
```

---

### 8. Create a Customer-Managed Policy

```bash
aws iam create-policy \
  --policy-name my-custom-s3-policy \
  --policy-document file://policy.json \
  --description "Custom policy for S3 access to app bucket" \
  --tags Key=Team,Value=Backend
```

**What it does:** Creates a standalone, reusable customer-managed IAM policy.

**Example Output:**
```json
{
    "Policy": {
        "PolicyName": "my-custom-s3-policy",
        "PolicyId": "ANPAIOSFODNN7EXAMPLE",
        "Arn": "arn:aws:iam::123456789012:policy/my-custom-s3-policy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "CreateDate": "2024-01-15T10:40:00+00:00"
    }
}
```

---

### 9. List All IAM Users

```bash
aws iam list-users \
  --output table
```

**What it does:** Lists all IAM users in the account with metadata.

---

### 10. Get Details of a Specific User

```bash
aws iam get-user \
  --user-name my-app-user
```

**What it does:** Retrieves detailed information about a specific IAM user.

---

### 11. List Policies Attached to a Role

```bash
aws iam list-attached-role-policies \
  --role-name my-ec2-role
```

**Example Output:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonS3ReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
        }
    ]
}
```

---

### 12. Create a Login Profile (Console Password)

```bash
aws iam create-login-profile \
  --user-name my-app-user \
  --password "TempP@ssw0rd!2024" \
  --password-reset-required
```

**What it does:** Enables AWS Management Console access for an IAM user with a temporary password.

---

### 13. Delete an IAM User

```bash
# Must remove dependencies first (access keys, group memberships, policies)
aws iam delete-login-profile --user-name my-app-user
aws iam remove-user-from-group --group-name developers --user-name my-app-user
aws iam delete-access-key --user-name my-app-user --access-key-id AKIAIOSFODNN7EXAMPLE
aws iam delete-user --user-name my-app-user
```

**What it does:** Deletes an IAM user after removing all associated resources (required before deletion).

---

### 14. Simulate a Policy (Test Permissions)

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/my-app-user \
  --action-names s3:GetObject s3:PutObject \
  --resource-arns arn:aws:s3:::my-app-bucket/*
```

**What it does:** Simulates IAM policy evaluation to test whether an identity can perform specified actions.

**Example Output:**
```json
{
    "EvaluationResults": [
        {
            "EvalActionName": "s3:GetObject",
            "EvalResourceName": "arn:aws:s3:::my-app-bucket/*",
            "EvalDecision": "allowed"
        },
        {
            "EvalActionName": "s3:PutObject",
            "EvalResourceName": "arn:aws:s3:::my-app-bucket/*",
            "EvalDecision": "implicitDeny"
        }
    ]
}
```

---

### 15. Generate a Credential Report

```bash
# Trigger report generation
aws iam generate-credential-report

# Download and view the report
aws iam get-credential-report \
  --output text \
  --query 'Content' | base64 --decode
```

**What it does:** Generates a CSV report of all IAM users and their credential statuses (MFA, access keys, last used, etc.).

---

## Common Operations

### Create Operations

```bash
# Create IAM user
aws iam create-user --user-name my-new-user

# Create IAM role
aws iam create-role \
  --role-name my-lambda-role \
  --assume-role-policy-document file://lambda-trust-policy.json

# Create IAM group
aws iam create-group --group-name data-scientists

# Create customer-managed policy
aws iam create-policy \
  --policy-name MyDynamoDBPolicy \
  --policy-document file://dynamodb-policy.json

# Create instance profile (for EC2)
aws iam create-instance-profile \
  --instance-profile-name my-ec2-instance-profile

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name my-ec2-instance-profile \
  --role-name my-ec2-role

# Create virtual MFA device
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name my-mfa-device \
  --outfile /tmp/qrcode.png \
  --bootstrap-method QRCodePNG
```

---

### Read / Describe Operations

```bash
# Get user details
aws iam get-user --user-name my-app-user

# Get role details
aws iam get-role --role-name my-ec2-role

# Get policy details
aws iam get-policy \
  --policy-arn arn:aws:iam::123456789012:policy/my-custom-s3-policy

# Get policy version (actual permissions document)
aws iam get-policy-version \
  --policy-arn arn:aws:iam::123456789012:policy/my-custom-s3-policy \
  --version-id v1

# Get inline role policy
aws iam get-role-policy \
  --role-name my-ec2-role \
  --policy-name my-inline-s3-policy

# Get role trust policy (assume role policy)
aws iam get-role \
  --role-name my-ec2-role \
  --query 'Role.AssumeRolePolicyDocument'

# Get account password policy
aws iam get-account-password-policy

# Get account summary
aws iam get-account-summary
```

---

### Update Operations

```bash
# Update user name
aws iam update-user \
  --user-name old-username \
  --new-user-name new-username

# Update role description
aws iam update-role \
  --role-name my-ec2-role \
  --description "Updated description for EC2 role"

# Update role trust policy
aws iam update-assume-role-policy \
  --role-name my-ec2-role \
  --policy-document file://updated-trust-policy.json

# Update access key status (deactivate)
aws iam update-access-key \
  --user-name my-app-user \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive

# Update login profile password
aws iam update-login-profile \
  --user-name my-app-user \
  --password "NewP@ssw0rd!2024" \
  --no-password-reset-required

# Update account password policy
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --allow-users-to-change-password \
  --max-password-age 90 \
  --password-reuse-prevention 5
```

---

### Delete Operations

```bash
# Detach managed policy from role
aws iam detach-role-policy \
  --role-name my-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Delete inline policy from role
aws iam delete-role-policy \
  --role-name my-ec2-role \
  --policy-name my-inline-s3-policy

# Delete role (must detach all policies first)
aws iam delete-role --role-name my-ec2-role

# Delete a managed policy
aws iam delete-policy \
  --policy-arn arn:aws:iam::123456789012:policy/my-custom-s3-policy

# Delete access key
aws iam delete-access-key \
  --user-name my-app-user \
  --access-key-id AKIAIOSFODNN7EXAMPLE

# Delete group (must remove all users and detach policies first)
aws iam delete-group --group-name developers

# Delete instance profile
aws iam delete-instance-profile \
  --instance-profile-name my-ec2-instance-profile
```

---

### List Operations

```bash
# List all users
aws iam list-users

# List all roles
aws iam list-roles

# List all groups
aws iam list-groups

# List all managed policies (customer-managed only)
aws iam list-policies --scope Local

# List all AWS managed policies
aws iam list-policies --scope AWS

# List groups for a user
aws iam list-groups-for-user --user-name my-app-user

# List users in a group
aws iam list-users-in-group --group-name developers

# List access keys for a user
aws iam list-access-keys --user-name my-app-user

# List all inline policies on a role
aws iam list-role-policies --role-name my-ec2-role

# List all attached policies on a role
aws iam list-attached-role-policies --role-name my-ec2-role

# List attached policies on a user
aws iam list-attached-user-policies --user-name my-app-user

# List instance profiles
aws iam list-instance-profiles

# List MFA devices for a user
aws iam list-mfa-devices --user-name my-app-user

# List all SAML providers
aws iam list-saml-providers

# List all OIDC providers
aws iam list-open-id-connect-providers
```

---

## Advanced Commands

### 1. Get IAM Authorization Details (Full Account Dump)

```bash
aws iam get-account-authorization-details \
  --filter User Role Group LocalManagedPolicy \
  --output json > iam