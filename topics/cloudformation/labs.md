# CloudFormation — Hands-On Labs

## Lab 1: Getting Started with CloudFormation

### Objective
In this lab, you will learn the fundamentals of AWS CloudFormation by writing your first template and deploying a basic infrastructure stack. You will create a CloudFormation template that provisions an S3 bucket with versioning enabled, an SNS topic, and an output that references the created resources. By the end of this lab, you will understand the core concepts of templates, stacks, parameters, and outputs.

---

### Prerequisites

**AWS Services Required:**
- AWS CloudFormation
- Amazon S3
- Amazon SNS

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:*",
        "s3:*",
        "sns:*"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS Management Console access
- AWS CLI v2 installed and configured (`aws configure`)
- A text editor (VS Code recommended)
- Basic YAML knowledge

**Estimated Cost:** < $0.01 (resources will be deleted in cleanup)  
**Estimated Time:** 30–45 minutes

---

### Steps

#### Step 1: Create Your First CloudFormation Template

Create a new file named `lab1-basic-stack.yaml` with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  Lab 1 - Basic CloudFormation Stack
  Creates an S3 bucket with versioning and an SNS topic.

Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - prod
    Description: Select the environment type.

  BucketSuffix:
    Type: String
    Default: mylab001
    MinLength: 3
    MaxLength: 20
    AllowedPattern: '[a-z0-9\-]+'
    Description: Unique suffix for the S3 bucket name (lowercase, alphanumeric, hyphens only).

  NotificationEmail:
    Type: String
    Default: admin@example.com
    Description: Email address for SNS notifications.

Resources:

  # ── S3 Bucket ──────────────────────────────────────────────────────────────
  LabS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'cfn-lab1-${EnvironmentName}-${BucketSuffix}'
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName
        - Key: ManagedBy
          Value: CloudFormation
        - Key: Lab
          Value: Lab1

  # ── S3 Bucket Policy ───────────────────────────────────────────────────────
  LabS3BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref LabS3Bucket
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: DenyHTTP
            Effect: Deny
            Principal: '*'
            Action: 's3:*'
            Resource:
              - !GetAtt LabS3Bucket.Arn
              - !Sub '${LabS3Bucket.Arn}/*'
            Condition:
              Bool:
                'aws:SecureTransport': false

  # ── SNS Topic ──────────────────────────────────────────────────────────────
  LabSNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub 'cfn-lab1-${EnvironmentName}-notifications'
      DisplayName: CloudFormation Lab 1 Notifications
      Subscription:
        - Protocol: email
          Endpoint: !Ref NotificationEmail
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName
        - Key: ManagedBy
          Value: CloudFormation

Outputs:

  BucketName:
    Description: Name of the created S3 bucket
    Value: !Ref LabS3Bucket
    Export:
      Name: !Sub '${AWS::StackName}-BucketName'

  BucketArn:
    Description: ARN of the created S3 bucket
    Value: !GetAtt LabS3Bucket.Arn
    Export:
      Name: !Sub '${AWS::StackName}-BucketArn'

  SNSTopicArn:
    Description: ARN of the SNS topic
    Value: !Ref LabSNSTopic
    Export:
      Name: !Sub '${AWS::StackName}-SNSTopicArn'

  StackRegion:
    Description: Region where the stack is deployed
    Value: !Ref AWS::Region

  TemplateVersion:
    Description: CloudFormation template format version
    Value: !Ref AWS::StackId
```

**✅ Verify:** The file is saved and valid YAML (no tab characters — use spaces only).

---

#### Step 2: Validate the Template

Before deploying, always validate your template syntax.

**AWS Console:**
1. Open the [CloudFormation Console](https://console.aws.amazon.com/cloudformation).
2. Click **Create stack** → **With new resources (standard)**.
3. Select **Upload a template file** → Choose `lab1-basic-stack.yaml`.
4. CloudFormation will validate the template automatically on upload.

**AWS CLI:**
```bash
# Validate the template syntax
aws cloudformation validate-template \
  --template-body file://lab1-basic-stack.yaml

# Expected output — a JSON object listing Parameters and Capabilities:
# {
#   "Parameters": [
#     { "ParameterKey": "EnvironmentName", ... },
#     { "ParameterKey": "BucketSuffix", ... },
#     { "ParameterKey": "NotificationEmail", ... }
#   ],
#   "Description": "Lab 1 - Basic CloudFormation Stack ...",
#   "Capabilities": []
# }
```

**✅ Verify:** The CLI returns a JSON response without errors. If you see `An error occurred (ValidationError)`, check for YAML indentation issues.

---

#### Step 3: Deploy the Stack

**AWS Console:**
1. In the CloudFormation console, click **Create stack** → **With new resources (standard)**.
2. Select **Upload a template file** → Upload `lab1-basic-stack.yaml` → Click **Next**.
3. Set the following:
   - **Stack name:** `cfn-lab1-basic-stack`
   - **EnvironmentName:** `dev`
   - **BucketSuffix:** `mylab001` (must be globally unique — add your initials if needed)
   - **NotificationEmail:** your real email address
4. Click **Next** → **Next** → **Submit**.
5. Watch the **Events** tab as resources are created.

**AWS CLI:**
```bash
# Deploy the stack with parameters
aws cloudformation create-stack \
  --stack-name cfn-lab1-basic-stack \
  --template-body file://lab1-basic-stack.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=dev \
    ParameterKey=BucketSuffix,ParameterValue=mylab001 \
    ParameterKey=NotificationEmail,ParameterValue=admin@example.com \
  --tags \
    Key=Lab,Value=Lab1 \
    Key=Owner,Value=YourName

# Monitor stack creation progress
aws cloudformation wait stack-create-complete \
  --stack-name cfn-lab1-basic-stack

echo "Stack creation complete!"
```

**Expected Output:**
```
{
    "StackId": "arn:aws:cloudformation:us-east-1:123456789012:stack/cfn-lab1-basic-stack/abc12345-..."
}
```

**✅ Verify:** The stack status shows `CREATE_COMPLETE` in the console Events tab or via CLI.

---

#### Step 4: Inspect Stack Events and Resources

**AWS Console:**
1. Click on `cfn-lab1-basic-stack` in the CloudFormation console.
2. Explore the **Events**, **Resources**, **Outputs**, and **Parameters** tabs.

**AWS CLI:**
```bash
# View stack events
aws cloudformation describe-stack-events \
  --stack-name cfn-lab1-basic-stack \
  --query 'StackEvents[*].[Timestamp,ResourceStatus,ResourceType,LogicalResourceId]' \
  --output table

# View stack resources
aws cloudformation describe-stack-resources \
  --stack-name cfn-lab1-basic-stack \
  --query 'StackResources[*].[LogicalResourceId,ResourceType,ResourceStatus,PhysicalResourceId]' \
  --output table

# View stack outputs
aws cloudformation describe-stacks \
  --stack-name cfn-lab1-basic-stack \
  --query 'Stacks[0].Outputs' \
  --output table
```

**Expected Output (Resources table):**
```
---------------------------------------------------------------------------
|                       DescribeStackResources                            |
+-------------------+------------------+----------------+-----------------+
|  LabS3Bucket      | AWS::S3::Bucket  | CREATE_COMPLETE| cfn-lab1-dev-.. |
|  LabS3BucketPolicy| AWS::S3::BucketP.| CREATE_COMPLETE| cfn-lab1-dev-.. |
|  LabSNSTopic      | AWS::SNS::Topic  | CREATE_COMPLETE| arn:aws:sns:... |
+-------------------+------------------+----------------+-----------------+
```

**✅ Verify:** All three resources show `CREATE_COMPLETE` status.

---

#### Step 5: Update the Stack (Change Set)

Learn how to safely update a stack using Change Sets.

Create an updated template `lab1-basic-stack-v2.yaml` — add lifecycle rules to the S3 bucket:

```yaml
# Add this block inside LabS3Bucket Properties (after VersioningConfiguration):
      LifecycleConfiguration:
        Rules:
          - Id: MoveToIA
            Status: Enabled
            Transitions:
              - TransitionInDays: 30
                StorageClass: STANDARD_IA
            NoncurrentVersionTransitions:
              - TransitionInDays: 7
                StorageClass: STANDARD_IA
            NoncurrentVersionExpirationInDays: 90
```

**AWS Console:**
1. Select your stack → **Update** → **Replace current template**.
2. Upload the updated template → **Next** → **Next**.
3. On the **Review** page, click **View change set**.
4. Review the proposed changes before applying.

**AWS CLI:**
```bash
# Create a change set (does NOT apply changes yet)
aws cloudformation create-change-set \
  --stack-name cfn-lab1-basic-stack \
  --change-set-name lab1-add-lifecycle \
  --template-body file://lab1-basic-stack-v2.yaml \
  --parameters \
    ParameterKey=EnvironmentName,UsePreviousValue=true \
    ParameterKey=BucketSuffix,UsePreviousValue=true \
    ParameterKey=NotificationEmail,UsePreviousValue=true

# Wait for change set to be created
aws cloudformation wait change-set-create-complete \
  --stack-name cfn-lab1-basic-stack \
  --change-set-name lab1-add-lifecycle

# Review the change set BEFORE applying
aws cloudformation describe-change-set \
  --stack-name cfn-lab1-basic-stack \
  --change-set-name lab1-add-lifecycle \
  --query 'Changes[*].ResourceChange.[Action,ResourceType,LogicalResourceId,Replacement]' \
  --output table

# Apply the change set
aws cloudformation execute-change-set \
  --stack-name cfn-lab1-basic-stack \
  --change-set-name lab1-add-lifecycle

# Wait for update to complete
aws cloudformation wait stack-update-complete \
  --stack-name cfn-lab1-basic-stack

echo "Stack update complete!"
```

**✅ Verify:** Stack status returns to `UPDATE_COMPLETE`. The S3 bucket now has lifecycle rules configured.

---

### Verification

Run the following commands to confirm successful lab completion:

```bash
# 1. Confirm stack is in CREATE/UPDATE_COMPLETE state
aws cloudformation describe-stacks \
  --stack-name cfn-lab1-basic-stack \
  --query 'Stacks[0].StackStatus' \
  --output text
# Expected: UPDATE_COMPLETE

# 2. Confirm S3 bucket versioning is enabled
BUCKET_NAME=$(aws cloudformation describe-stacks \
  --stack-name cfn-lab1-basic-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`BucketName`].OutputValue' \
  --output text)

aws s3api get-bucket-versioning --bucket "$BUCKET_NAME"
# Expected: { "Status": "Enabled" }

# 3. Confirm SNS topic exists
aws cloudformation describe-stacks \
  --stack-name cfn-lab1-basic-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`SNSTopicArn`].OutputValue' \
  --output text
# Expected: arn:aws:sns:us-east-1:123456789012:cfn-lab1-dev-notifications

# 4. Confirm exported values are available
aws cloudformation list-exports \
  --query 'Exports[?starts_with(Name, `cfn-lab1-basic-stack`)]' \
  --output table
# Expected: Three exports listed (BucketName, BucketArn, SNSTopicArn)
```

**✅ Lab 1 is complete when:**
- [ ] Stack status is `UPDATE_COMPLETE`
- [ ] S3 bucket versioning is `Enabled`
- [ ] SNS topic ARN is visible in Outputs
- [ ] Three stack exports are listed
- [ ] You received an SNS subscription confirmation email

---

### Cleanup

> ⚠️ **Important:** S3 buckets must be empty before CloudFormation can delete them. Run the following steps in order.

```bash
# Step 1: Get the bucket name from stack outputs
BUCKET_NAME=$(aws cloudformation describe-stacks \
  --stack-name cfn-lab1-basic-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`BucketName`].OutputValue' \
  --output text)

echo "Bucket to empty: $BUCKET_NAME"

# Step 2: Delete all object versions (required for versioned buckets)
aws s3api delete-objects \
  --bucket "$BUCKET_NAME" \
  --delete "$(aws s3api list-object-versions \
    --bucket "$BUCKET_NAME" \
    --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' \
    --output json)" 2>/dev/null || echo "No versions to delete"

# Step 3: Delete all delete markers
aws s3api delete-objects \
  --bucket "$BUCKET_NAME" \
  --delete "$(aws s3api list-object-versions \
    --bucket "$BUCKET_NAME" \
    --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' \
    --output json)" 2>/dev/null || echo "No delete markers"

# Step 4: Delete the CloudFormation stack
aws cloudformation delete-stack \
  --stack-name cfn-lab1-basic-stack

# Step 5: Wait for deletion to complete
aws cloudformation wait stack-delete-complete \
  --stack-