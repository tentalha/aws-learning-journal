# IAM — Hands-On Labs

## Lab 1: Getting Started with IAM

### Objective
In this lab, you will learn the foundational concepts of AWS Identity and Access Management (IAM). You will create an IAM user, assign it to a group with managed policies, generate access keys, and verify permissions using both the AWS Console and CLI. By the end, you will understand how users, groups, and policies work together to control access to AWS resources.

### Prerequisites
- An AWS account with root or administrator-level access
- AWS CLI v2 installed and configured (`aws configure`)
- Basic familiarity with the AWS Management Console
- No additional services required — IAM is global and free

### Steps

---

#### Step 1: Create an IAM Group with a Managed Policy

**Console:**
1. Sign in to the AWS Management Console.
2. Navigate to **IAM** → **User groups** → **Create group**.
3. Enter the group name: `dev-readonly-group`.
4. In the **Attach permissions policies** section, search for `AmazonS3ReadOnlyAccess`.
5. Check the box next to `AmazonS3ReadOnlyAccess`.
6. Click **Create user group**.

**CLI:**
```bash
# Create the IAM group
aws iam create-group --group-name dev-readonly-group

# Attach the managed policy to the group
aws iam attach-group-policy \
  --group-name dev-readonly-group \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

**Verify:**
```bash
# List groups to confirm creation
aws iam list-groups --query "Groups[?GroupName=='dev-readonly-group']"

# Verify the policy is attached
aws iam list-attached-group-policies --group-name dev-readonly-group
```

**Expected Output:**
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

#### Step 2: Create an IAM User

**Console:**
1. Navigate to **IAM** → **Users** → **Create user**.
2. Enter the username: `dev-user-01`.
3. Check **Provide user access to the AWS Management Console**.
4. Select **I want to create an IAM user**.
5. Choose **Custom password** and set a strong password.
6. Uncheck **Users must create a new password at next sign-in** (for lab purposes).
7. Click **Next**.

**CLI:**
```bash
# Create the IAM user
aws iam create-user --user-name dev-user-01

# Create a login profile (console password)
aws iam create-login-profile \
  --user-name dev-user-01 \
  --password "Lab@Password123!" \
  --no-password-reset-required
```

**Verify:**
```bash
# Confirm the user was created
aws iam get-user --user-name dev-user-01
```

**Expected Output:**
```json
{
    "User": {
        "UserName": "dev-user-01",
        "UserId": "AIDAXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::123456789012:user/dev-user-01",
        "Path": "/",
        "CreateDate": "2024-01-15T10:00:00+00:00"
    }
}
```

---

#### Step 3: Add the User to the Group

**Console:**
1. Navigate to **IAM** → **User groups** → `dev-readonly-group`.
2. Click the **Users** tab → **Add users**.
3. Check `dev-user-01` → **Add users**.

**CLI:**
```bash
# Add user to the group
aws iam add-user-to-group \
  --user-name dev-user-01 \
  --group-name dev-readonly-group

# Verify group membership
aws iam list-groups-for-user --user-name dev-user-01
```

**Expected Output:**
```json
{
    "Groups": [
        {
            "GroupName": "dev-readonly-group",
            "GroupId": "AGPAXXXXXXXXXXXXXXXXX",
            "Arn": "arn:aws:iam::123456789012:group/dev-readonly-group"
        }
    ]
}
```

---

#### Step 4: Create Access Keys for Programmatic Access

**Console:**
1. Navigate to **IAM** → **Users** → `dev-user-01`.
2. Click the **Security credentials** tab.
3. Under **Access keys**, click **Create access key**.
4. Select **Command Line Interface (CLI)**.
5. Acknowledge the recommendation and click **Next** → **Create access key**.
6. **Download the `.csv` file** — this is your only opportunity to save the secret key.

**CLI:**
```bash
# Create access keys programmatically
aws iam create-access-key --user-name dev-user-01
```

**Expected Output:**
```json
{
    "AccessKey": {
        "UserName": "dev-user-01",
        "AccessKeyId": "AKIAIOSFODNN7EXAMPLE",
        "Status": "Active",
        "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
        "CreateDate": "2024-01-15T10:05:00+00:00"
    }
}
```

> ⚠️ **Security Note:** Never commit access keys to source control. Treat the `SecretAccessKey` like a password.

---

#### Step 5: Test Permissions Using a Separate CLI Profile

```bash
# Configure a new CLI profile for dev-user-01
aws configure --profile dev-user-01
# Enter the AccessKeyId, SecretAccessKey, region (e.g., us-east-1), and output format (json)

# Test S3 read access (should SUCCEED)
aws s3 ls --profile dev-user-01

# Test EC2 describe (should FAIL — no EC2 permissions)
aws ec2 describe-instances --profile dev-user-01
```

**Expected Output for EC2 (Access Denied):**
```
An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation:
You are not authorized to perform this operation.
```

---

### Verification

Run the following checklist to confirm successful lab completion:

```bash
# 1. Verify group exists with correct policy
aws iam list-attached-group-policies --group-name dev-readonly-group

# 2. Verify user exists and is in the group
aws iam list-groups-for-user --user-name dev-user-01

# 3. Verify access key is active
aws iam list-access-keys --user-name dev-user-01

# 4. Simulate permissions (IAM Policy Simulator via CLI)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):user/dev-user-01 \
  --action-names s3:ListBucket \
  --query "EvaluationResults[0].EvalDecision"
# Expected: "allowed"
```

---

### Cleanup

```bash
# 1. Remove user from group
aws iam remove-user-from-group \
  --user-name dev-user-01 \
  --group-name dev-readonly-group

# 2. Delete access keys (list first, then delete)
ACCESS_KEY_ID=$(aws iam list-access-keys --user-name dev-user-01 \
  --query "AccessKeyMetadata[0].AccessKeyId" --output text)

aws iam delete-access-key \
  --user-name dev-user-01 \
  --access-key-id $ACCESS_KEY_ID

# 3. Delete the console login profile
aws iam delete-login-profile --user-name dev-user-01

# 4. Delete the user
aws iam delete-user --user-name dev-user-01

# 5. Detach policy from group
aws iam detach-group-policy \
  --group-name dev-readonly-group \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 6. Delete the group
aws iam delete-group --group-name dev-readonly-group

# 7. Remove the local CLI profile
aws configure --profile dev-user-01 set aws_access_key_id ""
# Or manually remove from ~/.aws/credentials
```

> ✅ IAM itself has no cost, but access keys left active are a security risk. Always clean up unused credentials.

---

## Lab 2: Intermediate IAM Configuration

### Objective
In this lab, you will implement least-privilege access using a custom IAM policy with conditions, configure multi-factor authentication (MFA) enforcement, create an IAM role for cross-service access (EC2 → S3), and use IAM Access Analyzer to identify overly permissive policies. You will learn how conditions, resource-level permissions, and roles differ from basic user/group setups.

### Prerequisites
- Completion of Lab 1 (or equivalent IAM knowledge)
- AWS CLI v2 configured with administrator permissions
- At least one S3 bucket in your account (or create one during the lab)
- An EC2 key pair (optional, for role testing)
- `jq` installed for JSON parsing (optional but recommended)

### Steps

---

#### Step 1: Create a Custom IAM Policy with Conditions

We will create a policy that allows S3 access **only** to a specific bucket and **only** when MFA is present.

**Console:**
1. Navigate to **IAM** → **Policies** → **Create policy**.
2. Click the **JSON** tab and paste the policy below.
3. Click **Next** → Name it `S3BucketMFAPolicy` → **Create policy**.

**Policy JSON:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ListBuckets",
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    },
    {
      "Sid": "AllowSpecificBucketAccessWithMFA",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::lab2-secure-bucket-${AWS::AccountId}",
        "arn:aws:s3:::lab2-secure-bucket-${AWS::AccountId}/*"
      ],
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        },
        "NumericLessThan": {
          "aws:MultiFactorAuthAge": "3600"
        }
      }
    },
    {
      "Sid": "DenyAllWithoutMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ListVirtualMFADevices",
        "iam:ResyncMFADevice",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

**CLI:**
```bash
# Save the policy to a file
cat > s3-mfa-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ListBuckets",
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    },
    {
      "Sid": "AllowSpecificBucketAccessWithMFA",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::lab2-secure-bucket-ACCOUNT_ID",
        "arn:aws:s3:::lab2-secure-bucket-ACCOUNT_ID/*"
      ],
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        },
        "NumericLessThan": {
          "aws:MultiFactorAuthAge": "3600"
        }
      }
    }
  ]
}
EOF

# Get your account ID and substitute it
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/ACCOUNT_ID/$ACCOUNT_ID/g" s3-mfa-policy.json

# Create the policy
aws iam create-policy \
  --policy-name S3BucketMFAPolicy \
  --policy-document file://s3-mfa-policy.json \
  --description "S3 access restricted to specific bucket with MFA requirement"
```

**Verify:**
```bash
# Confirm policy was created
aws iam get-policy \
  --policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/S3BucketMFAPolicy \
  --query "Policy.{Name:PolicyName,CreateDate:CreateDate}"
```

---

#### Step 2: Create an S3 Bucket for Testing

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="lab2-secure-bucket-${ACCOUNT_ID}"

# Create the bucket
aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region us-east-1

# Enable versioning on the bucket
aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

# Block public access
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

echo "Bucket created: $BUCKET_NAME"
```

---

#### Step 3: Create an IAM Role for EC2 to Access S3

This role uses the **principle of least privilege** — EC2 instances can only read from the specific bucket.

**Create the trust policy (who can assume this role):**
```bash
cat > ec2-trust-policy.json << 'EOF'
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
EOF
```

**Create the permissions policy for the role:**
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > ec2-s3-permissions.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadFromSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:GetObjectVersion"
      ],
      "Resource": [
        "arn:aws:s3:::lab2-secure-bucket-${ACCOUNT_ID}",
        "arn:aws:s3:::lab2-secure-bucket-${ACCOUNT_ID}/*"
      ]
    },
    {
      "Sid": "DenyDeleteOperations",
      "Effect": "Deny",