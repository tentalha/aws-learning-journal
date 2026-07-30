# AWS Config — Hands-On Labs

## Lab 1: Getting Started with AWS Config

### Objective
In this lab, you will enable AWS Config in a single AWS Region, configure an S3 delivery channel, create a configuration recorder, and apply your first managed Config rule (`s3-bucket-public-read-prohibited`). By the end, you will understand how AWS Config continuously tracks resource configurations and evaluates compliance.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | An active AWS account with billing enabled |
| IAM Permissions | `AWSConfigRole`, `AmazonS3FullAccess`, `IAMFullAccess`, `AWSConfigUserAccess` |
| AWS CLI | Version 2.x installed and configured (`aws configure`) |
| Region | `us-east-1` (all lab commands use this region) |
| Tools | AWS Management Console, AWS CLI, a text editor |

> **Cost Estimate:** AWS Config charges ~$0.003 per configuration item recorded. Running this lab for a few hours should cost less than $1.00.

---

### Steps

#### Step 1: Create an S3 Bucket for Config Delivery

AWS Config requires an S3 bucket to deliver configuration snapshots and history files.

**Console:**
1. Open the [S3 Console](https://s3.console.aws.amazon.com/s3/).
2. Click **Create bucket**.
3. Set **Bucket name** to `aws-config-lab-delivery-<your-account-id>` (must be globally unique).
4. Set **AWS Region** to `us-east-1`.
5. Under **Block Public Access settings**, ensure **Block all public access** is checked.
6. Enable **Bucket Versioning**.
7. Click **Create bucket**.

**CLI:**
```bash
# Set your Account ID as a variable
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="aws-config-lab-delivery-${ACCOUNT_ID}"
REGION="us-east-1"

# Create the S3 bucket
aws s3api create-bucket \
  --bucket "${BUCKET_NAME}" \
  --region "${REGION}"

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket "${BUCKET_NAME}" \
  --versioning-configuration Status=Enabled

# Block all public access
aws s3api put-public-access-block \
  --bucket "${BUCKET_NAME}" \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

echo "Bucket created: ${BUCKET_NAME}"
```

**Attach the required S3 bucket policy for AWS Config:**
```bash
cat > /tmp/config-bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSConfigBucketPermissionsCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "config.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}",
      "Condition": {
        "StringEquals": {
          "AWS:SourceAccount": "${ACCOUNT_ID}"
        }
      }
    },
    {
      "Sid": "AWSConfigBucketDelivery",
      "Effect": "Allow",
      "Principal": {
        "Service": "config.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/AWSLogs/${ACCOUNT_ID}/Config/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control",
          "AWS:SourceAccount": "${ACCOUNT_ID}"
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket "${BUCKET_NAME}" \
  --policy file:///tmp/config-bucket-policy.json

echo "Bucket policy applied."
```

**✅ Verify:** Run `aws s3api get-bucket-policy --bucket "${BUCKET_NAME}"` and confirm the policy is returned without error.

---

#### Step 2: Create an IAM Role for AWS Config

**Console:**
1. Open the [IAM Console](https://console.aws.amazon.com/iam/).
2. Click **Roles → Create role**.
3. Select **AWS service**, then choose **Config** from the use case list.
4. Click **Next**, attach the managed policy `AWS_ConfigRole`.
5. Name the role `AWSConfigLabRole` and click **Create role**.

**CLI:**
```bash
# Create the trust policy document
cat > /tmp/config-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "config.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the IAM role
aws iam create-role \
  --role-name AWSConfigLabRole \
  --assume-role-policy-document file:///tmp/config-trust-policy.json

# Attach the AWS managed policy
aws iam attach-role-policy \
  --role-name AWSConfigLabRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWS_ConfigRole

# Retrieve the Role ARN for later use
ROLE_ARN=$(aws iam get-role \
  --role-name AWSConfigLabRole \
  --query "Role.Arn" \
  --output text)

echo "IAM Role ARN: ${ROLE_ARN}"
```

**✅ Verify:** Run `aws iam get-role --role-name AWSConfigLabRole` and confirm `"RoleName": "AWSConfigLabRole"` appears in the output.

---

#### Step 3: Enable the AWS Config Configuration Recorder

**Console:**
1. Open the [AWS Config Console](https://console.aws.amazon.com/config/).
2. If this is your first time, click **Get started**.
3. Under **Recording method**, select **Record all current and future resource types supported in this region**.
4. Under **AWS Config role**, choose **Use an existing AWS Config service-linked role** or select `AWSConfigLabRole`.
5. Under **Delivery method**, choose the S3 bucket you created: `aws-config-lab-delivery-<account-id>`.
6. Click **Next → Next → Confirm**.

**CLI:**
```bash
# Create the configuration recorder
aws configservice put-configuration-recorder \
  --configuration-recorder \
    name=default,roleARN="${ROLE_ARN}" \
  --recording-group \
    allSupported=true,includeGlobalResourceTypes=true \
  --region "${REGION}"

# Create the delivery channel
aws configservice put-delivery-channel \
  --delivery-channel \
    name=default,s3BucketName="${BUCKET_NAME}" \
  --region "${REGION}"

# Start the configuration recorder
aws configservice start-configuration-recorder \
  --configuration-recorder-name default \
  --region "${REGION}"

echo "Configuration recorder started."
```

**✅ Verify:**
```bash
aws configservice describe-configuration-recorder-status \
  --region "${REGION}" \
  --query "ConfigurationRecordersStatus[0].recording"
```
Expected output: `true`

---

#### Step 4: Create Your First Config Rule

Add the managed rule `s3-bucket-public-read-prohibited` to detect publicly readable S3 buckets.

**Console:**
1. In the AWS Config Console, click **Rules → Add rule**.
2. Search for `s3-bucket-public-read-prohibited`.
3. Select the rule and click **Next**.
4. Leave the default trigger (configuration changes) and click **Next → Save**.

**CLI:**
```bash
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "Description": "Checks that S3 buckets do not allow public read access.",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::S3::Bucket"]
    }
  }' \
  --region "${REGION}"

echo "Config rule created."
```

**✅ Verify:**
```bash
aws configservice describe-config-rules \
  --config-rule-names s3-bucket-public-read-prohibited \
  --region "${REGION}" \
  --query "ConfigRules[0].ConfigRuleState"
```
Expected output: `"ACTIVE"`

---

#### Step 5: Trigger a Compliance Evaluation

Create a non-compliant S3 bucket (public read enabled) to see Config flag it.

```bash
# Create a test bucket
TEST_BUCKET="config-lab-test-public-${ACCOUNT_ID}"

aws s3api create-bucket \
  --bucket "${TEST_BUCKET}" \
  --region "${REGION}"

# Disable block public access (making it potentially public)
aws s3api put-public-access-block \
  --bucket "${TEST_BUCKET}" \
  --public-access-block-configuration \
    "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Apply a public read ACL
aws s3api put-bucket-acl \
  --bucket "${TEST_BUCKET}" \
  --acl public-read

echo "Non-compliant bucket created: ${TEST_BUCKET}"
```

Wait 2–3 minutes, then check compliance:
```bash
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name s3-bucket-public-read-prohibited \
  --compliance-types NON_COMPLIANT \
  --region "${REGION}" \
  --query "EvaluationResults[*].EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId"
```

**✅ Expected output:** The `TEST_BUCKET` name should appear in the NON_COMPLIANT list.

---

### Verification

Run the following checklist to confirm successful lab completion:

```bash
echo "=== Lab 1 Verification ==="

echo "1. Config Recorder Status:"
aws configservice describe-configuration-recorder-status \
  --region "${REGION}" \
  --query "ConfigurationRecordersStatus[0].{Name:name,Recording:recording,LastStatus:lastStatus}"

echo "2. Delivery Channel:"
aws configservice describe-delivery-channels \
  --region "${REGION}" \
  --query "DeliveryChannels[0].{Name:name,S3Bucket:s3BucketName}"

echo "3. Config Rule Status:"
aws configservice describe-config-rule-evaluation-status \
  --config-rule-names s3-bucket-public-read-prohibited \
  --region "${REGION}" \
  --query "ConfigRulesEvaluationStatus[0].{Rule:ConfigRuleName,LastStatus:LastSuccessfulEvaluationTime}"

echo "4. Compliance Summary:"
aws configservice get-compliance-summary-by-config-rule \
  --region "${REGION}" \
  --query "ComplianceSummariesByConfigRule[?ConfigRuleName=='s3-bucket-public-read-prohibited']"
```

---

### Cleanup

> ⚠️ **Important:** Complete cleanup to avoid ongoing Config recording charges (~$0.003/config item).

```bash
# 1. Stop the configuration recorder
aws configservice stop-configuration-recorder \
  --configuration-recorder-name default \
  --region "${REGION}"

# 2. Delete the Config rule
aws configservice delete-config-rule \
  --config-rule-name s3-bucket-public-read-prohibited \
  --region "${REGION}"

# 3. Delete the delivery channel
aws configservice delete-delivery-channel \
  --delivery-channel-name default \
  --region "${REGION}"

# 4. Delete the configuration recorder
aws configservice delete-configuration-recorder \
  --configuration-recorder-name default \
  --region "${REGION}"

# 5. Delete the test S3 bucket (remove ACL first)
aws s3api put-bucket-acl \
  --bucket "${TEST_BUCKET}" \
  --acl private

aws s3 rb "s3://${TEST_BUCKET}" --force

# 6. Delete the delivery S3 bucket (empty it first)
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3 rb "s3://${BUCKET_NAME}" --force

# 7. Detach policy and delete IAM role
aws iam detach-role-policy \
  --role-name AWSConfigLabRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWS_ConfigRole

aws iam delete-role --role-name AWSConfigLabRole

echo "=== Lab 1 Cleanup Complete ==="
```

---

## Lab 2: Intermediate AWS Config Configuration

### Objective
In this lab, you will go beyond basic setup to implement custom Config rules using AWS Lambda, configure Config aggregation across multiple accounts/regions using an aggregator, set up SNS notifications for compliance changes, and use AWS Config conformance packs to enforce a group of related rules. You will simulate a multi-team governance scenario.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account; Lab 1 completed (or Config already enabled) |
| IAM Permissions | `AWSConfigRole`, `AWSLambdaFullAccess`, `AmazonSNSFullAccess`, `IAMFullAccess` |
| AWS CLI | Version 2.x configured for `us-east-1` |
| Python | 3.11+ (for Lambda function code) |
| Knowledge | Completion of Lab 1 or equivalent Config experience |

> **Cost Estimate:** Lambda invocations are within free tier for this lab. SNS costs are minimal (<$0.01). Config rules add ~$0.001/evaluation.

---

### Steps

#### Step 1: Re-enable AWS Config (if cleaned up from Lab 1)

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="aws-config-lab-delivery-${ACCOUNT_ID}"
REGION="us-east-1"

# Recreate S3 bucket
aws s3api create-bucket \
  --bucket "${BUCKET_NAME}" \
  --region "${REGION}"

aws s3api put-public-access-block \
  --bucket "${BUCKET_NAME}" \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Apply bucket policy (reuse policy from Lab 1)
cat > /tmp/config-bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSConfigBucketPermissionsCheck",
      "Effect": "Allow",
      "Principal": { "Service": "config.amazonaws.com" },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}"
    },
    {
      "Sid": "AWSConfigBucketDelivery",
      "Effect": "Allow",
      "Principal": { "Service": "config.amazonaws.com" },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/AWSLogs/${ACCOUNT_ID}/Config/*",
      "Condition": {
        "StringEquals": { "s3:x-amz-acl": "bucket-owner-full-control" }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket "${BUCKET_NAME}" \
  --policy file:///tmp/config-bucket-