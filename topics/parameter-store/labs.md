# Parameter Store — Hands-On Labs

## Lab 1: Getting Started with Parameter Store

### Objective

In this lab, you will learn the fundamentals of AWS Systems Manager Parameter Store. You will create, retrieve, update, and delete parameters of different types (String, StringList, and SecureString). By the end of this lab, you will understand how Parameter Store works, how to organize parameters using hierarchies, and how to retrieve parameter values using both the AWS Console and the AWS CLI.

### Prerequisites

**AWS Services Required:**
- AWS Systems Manager (Parameter Store)
- AWS KMS (for SecureString parameters)
- AWS IAM

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:PutParameter",
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath",
        "ssm:DeleteParameter",
        "ssm:DescribeParameters",
        "ssm:ListTagsForResource",
        "ssm:AddTagsToResource",
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- AWS Console access
- Bash-compatible terminal (Linux/macOS) or PowerShell (Windows)
- Default AWS region set (this lab uses `us-east-1`)

**Estimated Cost:** < $0.05 (Standard parameters are free; Advanced parameters cost $0.05/month each)

**Estimated Duration:** 45 minutes

---

### Steps

#### Step 1: Navigate to Parameter Store in the Console

**Console:**
1. Sign in to the AWS Management Console.
2. In the search bar at the top, type **Systems Manager** and click the result.
3. In the left navigation panel, scroll down to **Application Management** and click **Parameter Store**.
4. You will see the Parameter Store dashboard. If this is a new account, the list will be empty.

**CLI — Verify CLI access to Parameter Store:**
```bash
aws ssm describe-parameters \
  --region us-east-1 \
  --output table
```

**Expected Output:**
```
--------------------------------------------
|           DescribeParameters             |
+------------------------------------------+
||              Parameters                ||
|+------------------------------------------+|
```
> ✅ **Verify:** The command runs without error. An empty table is expected if no parameters exist yet.

---

#### Step 2: Create a Standard String Parameter

**Console:**
1. Click **Create parameter**.
2. Fill in the following fields:
   - **Name:** `/myapp/dev/app-name`
   - **Description:** `Application name for dev environment`
   - **Tier:** `Standard`
   - **Type:** `String`
   - **Data type:** `text`
   - **Value:** `my-awesome-app`
3. Under **Tags**, click **Add tag**:
   - Key: `Environment`, Value: `dev`
   - Key: `Project`, Value: `myapp`
4. Click **Create parameter**.

**CLI:**
```bash
aws ssm put-parameter \
  --name "/myapp/dev/app-name" \
  --description "Application name for dev environment" \
  --value "my-awesome-app" \
  --type "String" \
  --tier "Standard" \
  --tags "Key=Environment,Value=dev" "Key=Project,Value=myapp" \
  --region us-east-1
```

**Expected Output:**
```json
{
    "Version": 1,
    "Tier": "Standard"
}
```

> ✅ **Verify:** The parameter appears in the Parameter Store console list, or run:
```bash
aws ssm get-parameter \
  --name "/myapp/dev/app-name" \
  --region us-east-1
```
Expected output:
```json
{
    "Parameter": {
        "Name": "/myapp/dev/app-name",
        "Type": "String",
        "Value": "my-awesome-app",
        "Version": 1,
        "LastModifiedDate": "...",
        "ARN": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/dev/app-name",
        "DataType": "text"
    }
}
```

---

#### Step 3: Create a StringList Parameter

**Console:**
1. Click **Create parameter**.
2. Fill in:
   - **Name:** `/myapp/dev/allowed-regions`
   - **Description:** `Allowed AWS regions for dev`
   - **Tier:** `Standard`
   - **Type:** `StringList`
   - **Value:** `us-east-1,us-west-2,eu-west-1`
3. Click **Create parameter**.

**CLI:**
```bash
aws ssm put-parameter \
  --name "/myapp/dev/allowed-regions" \
  --description "Allowed AWS regions for dev" \
  --value "us-east-1,us-west-2,eu-west-1" \
  --type "StringList" \
  --tier "Standard" \
  --tags "Key=Environment,Value=dev" "Key=Project,Value=myapp" \
  --region us-east-1
```

> ✅ **Verify:**
```bash
aws ssm get-parameter \
  --name "/myapp/dev/allowed-regions" \
  --region us-east-1 \
  --query "Parameter.Value" \
  --output text
```
Expected output:
```
us-east-1,us-west-2,eu-west-1
```

---

#### Step 4: Create a SecureString Parameter

SecureString parameters are encrypted using AWS KMS. By default, the AWS-managed key `aws/ssm` is used.

**Console:**
1. Click **Create parameter**.
2. Fill in:
   - **Name:** `/myapp/dev/db-password`
   - **Description:** `Database password for dev environment`
   - **Tier:** `Standard`
   - **Type:** `SecureString`
   - **KMS key source:** `My current account`
   - **KMS Key ID:** `alias/aws/ssm` (default)
   - **Value:** `SuperSecret@123!`
3. Click **Create parameter**.

**CLI:**
```bash
aws ssm put-parameter \
  --name "/myapp/dev/db-password" \
  --description "Database password for dev environment" \
  --value "SuperSecret@123!" \
  --type "SecureString" \
  --key-id "alias/aws/ssm" \
  --tier "Standard" \
  --tags "Key=Environment,Value=dev" "Key=Project,Value=myapp" \
  --region us-east-1
```

**Retrieve the SecureString (decrypted):**
```bash
aws ssm get-parameter \
  --name "/myapp/dev/db-password" \
  --with-decryption \
  --region us-east-1
```

**Retrieve WITHOUT decryption (shows ciphertext):**
```bash
aws ssm get-parameter \
  --name "/myapp/dev/db-password" \
  --region us-east-1
```

> ✅ **Verify:** When using `--with-decryption`, the `Value` field shows `SuperSecret@123!`. Without it, you see an encrypted blob.

---

#### Step 5: Retrieve Multiple Parameters at Once

**CLI — Get multiple specific parameters:**
```bash
aws ssm get-parameters \
  --names "/myapp/dev/app-name" "/myapp/dev/allowed-regions" "/myapp/dev/db-password" \
  --with-decryption \
  --region us-east-1
```

**CLI — Get all parameters under a path hierarchy:**
```bash
aws ssm get-parameters-by-path \
  --path "/myapp/dev" \
  --recursive \
  --with-decryption \
  --region us-east-1 \
  --output table
```

Expected output (table format):
```
------------------------------------------------------------
|                   GetParametersByPath                    |
+----------------------------------------------------------+
||                      Parameters                        ||
|+--------------------+----------+-----------------------+|
||        Name        |   Type   |         Value         ||
|+--------------------+----------+-----------------------+|
|| /myapp/dev/app-name| String   | my-awesome-app        ||
|| /myapp/dev/allowed-| StringList| us-east-1,us-west-2  ||
|| /myapp/dev/db-pass | SecureString| SuperSecret@123!   ||
|+--------------------+----------+-----------------------+|
```

> ✅ **Verify:** All three parameters appear in the output.

---

#### Step 6: Update a Parameter

**Console:**
1. Click on `/myapp/dev/app-name` in the parameter list.
2. Click **Edit**.
3. Change the **Value** to `my-awesome-app-v2`.
4. Click **Save changes**.

**CLI:**
```bash
aws ssm put-parameter \
  --name "/myapp/dev/app-name" \
  --value "my-awesome-app-v2" \
  --type "String" \
  --overwrite \
  --region us-east-1
```

**View parameter version history:**
```bash
aws ssm get-parameter-history \
  --name "/myapp/dev/app-name" \
  --region us-east-1 \
  --output table
```

Expected output:
```
------------------------------------------------------
|              GetParameterHistory                   |
+----------------------------------------------------+
||               Parameters                         ||
|+-------+---------+-----------------------------+  |
|| Version|  Value  |      LastModifiedDate       |  |
|+-------+---------+-----------------------------+  |
||   1   | my-awesome-app | 2024-01-15T10:00:00  |  |
||   2   | my-awesome-app-v2 | 2024-01-15T10:05:00||
|+-------+---------+-----------------------------+  |
```

> ✅ **Verify:** Version 2 shows the updated value `my-awesome-app-v2`.

---

#### Step 7: Retrieve a Specific Version

```bash
# Retrieve version 1 (the original value)
aws ssm get-parameter \
  --name "/myapp/dev/app-name:1" \
  --region us-east-1 \
  --query "Parameter.Value" \
  --output text
```

Expected output:
```
my-awesome-app
```

```bash
# Retrieve the latest version
aws ssm get-parameter \
  --name "/myapp/dev/app-name" \
  --region us-east-1 \
  --query "Parameter.Value" \
  --output text
```

Expected output:
```
my-awesome-app-v2
```

> ✅ **Verify:** Version pinning works correctly and returns the expected historical value.

---

### Verification

Run the following commands to confirm all lab steps were completed successfully:

```bash
#!/bin/bash
echo "=== Lab 1 Verification ==="

echo ""
echo "1. Listing all /myapp/dev parameters:"
aws ssm get-parameters-by-path \
  --path "/myapp/dev" \
  --with-decryption \
  --region us-east-1 \
  --query "Parameters[*].{Name:Name,Type:Type,Version:Version}" \
  --output table

echo ""
echo "2. Verifying parameter versions:"
aws ssm get-parameter-history \
  --name "/myapp/dev/app-name" \
  --region us-east-1 \
  --query "Parameters[*].{Version:Version,Value:Value}" \
  --output table

echo ""
echo "3. Verifying SecureString decryption:"
DBPASS=$(aws ssm get-parameter \
  --name "/myapp/dev/db-password" \
  --with-decryption \
  --region us-east-1 \
  --query "Parameter.Value" \
  --output text)

if [ "$DBPASS" == "SuperSecret@123!" ]; then
  echo "✅ SecureString decryption successful"
else
  echo "❌ SecureString decryption failed"
fi
```

---

### Cleanup

Run the following to remove all resources created in this lab:

**CLI:**
```bash
# Delete all parameters created in this lab
aws ssm delete-parameters \
  --names \
    "/myapp/dev/app-name" \
    "/myapp/dev/allowed-regions" \
    "/myapp/dev/db-password" \
  --region us-east-1
```

Expected output:
```json
{
    "DeletedParameters": [
        "/myapp/dev/app-name",
        "/myapp/dev/allowed-regions",
        "/myapp/dev/db-password"
    ],
    "InvalidParameters": []
}
```

**Console:**
1. Go to **Systems Manager → Parameter Store**.
2. Select all parameters with the checkbox.
3. Click **Delete**.
4. Type `delete` in the confirmation box and click **Delete parameters**.

> ✅ **Verify cleanup:**
```bash
aws ssm get-parameters-by-path \
  --path "/myapp/dev" \
  --region us-east-1 \
  --query "Parameters" \
  --output text
# Expected: (empty output)
```

---

## Lab 2: Intermediate Parameter Store Configuration

### Objective

In this lab, you will explore advanced Parameter Store features including parameter hierarchies across multiple environments, AWS Lambda integration to retrieve parameters at runtime, CloudFormation dynamic references, and parameter policies for automatic expiration notifications. You will build a realistic multi-environment configuration system and integrate it with a Lambda function.

### Prerequisites

**AWS Services Required:**
- AWS Systems Manager (Parameter Store)
- AWS Lambda
- AWS IAM
- AWS CloudFormation
- AWS KMS
- AWS SNS (for notifications)
- AWS CloudWatch

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:*",
        "lambda:CreateFunction",
        "lambda:InvokeFunction",
        "lambda:GetFunction",
        "lambda:DeleteFunction",
        "lambda:UpdateFunctionCode",
        "lambda:UpdateFunctionConfiguration",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole",
        "iam:DeleteRole",
        "iam:DetachRolePolicy",
        "iam:GetRole",
        "kms:*",
        "cloudformation:*",
        "sns:CreateTopic",
        "sns:Subscribe",
        "sns:DeleteTopic",
        "cloudwatch:PutMetricAlarm",
        "cloudwatch:DeleteAlarms",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 configured
- Python 3.11+ (for Lambda function)
- `zip` utility
- jq (optional, for JSON parsing)

**Estimated Cost:** < $0.10

**Estimated Duration:** 75 minutes

---

### Steps

#### Step 1: Build a Multi-Environment Parameter Hierarchy

Create a realistic hierarchy for a web application across `dev`, `staging`, and `prod` environments.

```bash
#!/bin/bash
REGION="us-east-1"

echo "Creating multi-environment parameter hierarchy..."

# === DEV Environment ===
aws ssm put-parameter --region $REGION --name "/webapp/dev/config/app-version" \
  --value "1.0.0-dev" --type "String" --tier "Standard" \
  --tags "Key=Environment,Value=dev" "Key=App,Value=webapp" --overwrite

aws ssm put-parameter --region $REGION --name "/webapp/dev/config/log-level" \
  --value "DEBUG" --type "String" --tier "Standard" \
  --tags "Key=Environment,Value=dev" "Key=App,Value=webapp" --overwrite

aws ssm put-parameter --region $