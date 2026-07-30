# AWS Config — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using AWS Config via the CLI, ensure the following are in place:

**AWS CLI Installation & Version**
```bash
aws --version
# Requires AWS CLI v2.x recommended
aws configure
```

**Required IAM Permissions**

Attach the following managed policies or equivalent inline permissions to your IAM user/role:

- `AWS_ConfigRole` — Managed policy for the AWS Config service role
- `AmazonS3FullAccess` (scoped) — For delivery channel S3 bucket access
- `AWSCloudTrailReadOnlyAccess` — For recording CloudTrail-related resources

**Minimum IAM Policy for Config Operations**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "config:*",
        "s3:PutObject",
        "s3:GetBucketAcl",
        "sns:Publish",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

**Service-Linked Role for AWS Config**
```bash
# Create the AWS Config service-linked role
aws iam create-service-linked-role \
  --aws-service-name config.amazonaws.com
```

**Set Default Region**
```bash
export AWS_DEFAULT_REGION=us-east-1
export AWS_PROFILE=my-devops-profile
```

---

## Core Commands

### 1. Put Configuration Recorder

Creates or updates the configuration recorder used to record resource configurations.

```bash
aws configservice put-configuration-recorder \
  --configuration-recorder '{
    "name": "default",
    "roleARN": "arn:aws:iam::123456789012:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig",
    "recordingGroup": {
      "allSupported": true,
      "includeGlobalResourceTypes": true
    }
  }'
```

**Example Output:** *(No output on success — returns HTTP 200)*

---

### 2. Put Delivery Channel

Configures the delivery channel that AWS Config uses to deliver configuration snapshots and notifications.

```bash
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "my-config-bucket-123456789012",
    "s3KeyPrefix": "config-logs",
    "snsTopicARN": "arn:aws:sns:us-east-1:123456789012:my-config-topic",
    "configSnapshotDeliveryProperties": {
      "deliveryFrequency": "TwentyFour_Hours"
    }
  }'
```

---

### 3. Start Configuration Recorder

Starts recording configurations of the AWS resources you have specified.

```bash
aws configservice start-configuration-recorder \
  --configuration-recorder-name default
```

---

### 4. Stop Configuration Recorder

Stops recording configurations of the AWS resources.

```bash
aws configservice stop-configuration-recorder \
  --configuration-recorder-name default
```

---

### 5. Describe Configuration Recorders

Returns details about the current configuration recorders.

```bash
aws configservice describe-configuration-recorders
```

**Example Output:**
```json
{
  "ConfigurationRecorders": [
    {
      "name": "default",
      "roleARN": "arn:aws:iam::123456789012:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig",
      "recordingGroup": {
        "allSupported": true,
        "includeGlobalResourceTypes": true,
        "resourceTypes": []
      }
    }
  ]
}
```

---

### 6. Describe Configuration Recorder Status

Shows the current status of the configuration recorder, including whether it is recording.

```bash
aws configservice describe-configuration-recorder-status \
  --configuration-recorder-names default
```

**Example Output:**
```json
{
  "ConfigurationRecordersStatus": [
    {
      "name": "default",
      "lastStartTime": "2024-01-15T10:30:00.000Z",
      "recording": true,
      "lastStatus": "SUCCESS",
      "lastStatusChangeTime": "2024-01-15T10:30:05.000Z"
    }
  ]
}
```

---

### 7. Put Config Rule

Creates or updates an AWS Config rule for evaluating compliance.

```bash
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "Description": "Checks that S3 buckets do not allow public read access",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::S3::Bucket"]
    },
    "ConfigRuleState": "ACTIVE"
  }'
```

---

### 8. Describe Config Rules

Lists details about your AWS Config rules.

```bash
aws configservice describe-config-rules \
  --config-rule-names s3-bucket-public-read-prohibited ec2-instance-no-public-ip
```

**Example Output:**
```json
{
  "ConfigRules": [
    {
      "ConfigRuleName": "s3-bucket-public-read-prohibited",
      "ConfigRuleArn": "arn:aws:config:us-east-1:123456789012:config-rule/config-rule-abc123",
      "ConfigRuleId": "config-rule-abc123",
      "Description": "Checks that S3 buckets do not allow public read access",
      "Source": {
        "Owner": "AWS",
        "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
      },
      "ConfigRuleState": "ACTIVE",
      "CreatedBy": ""
    }
  ]
}
```

---

### 9. Get Compliance Details by Config Rule

Returns compliance information for each resource evaluated against a specific rule.

```bash
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name s3-bucket-public-read-prohibited \
  --compliance-types NON_COMPLIANT \
  --limit 25
```

**Example Output:**
```json
{
  "EvaluationResults": [
    {
      "EvaluationResultIdentifier": {
        "EvaluationResultQualifier": {
          "ConfigRuleName": "s3-bucket-public-read-prohibited",
          "ResourceType": "AWS::S3::Bucket",
          "ResourceId": "my-public-bucket"
        },
        "OrderingTimestamp": "2024-01-15T08:00:00.000Z"
      },
      "ComplianceType": "NON_COMPLIANT",
      "ResultRecordedTime": "2024-01-15T08:05:00.000Z",
      "ConfigRuleInvokedTime": "2024-01-15T08:04:55.000Z"
    }
  ]
}
```

---

### 10. Get Compliance Summary by Config Rule

Returns a summary of compliance status for each of your Config rules.

```bash
aws configservice get-compliance-summary-by-config-rule
```

**Example Output:**
```json
{
  "ComplianceSummariesByConfigRule": [
    {
      "ConfigRuleName": "s3-bucket-public-read-prohibited",
      "Compliance": {
        "ComplianceType": "NON_COMPLIANT",
        "ComplianceContributorCount": {
          "CappedCount": 3,
          "CapExceeded": false
        }
      }
    }
  ]
}
```

---

### 11. Describe Delivery Channels

Returns details about the delivery channels configured for your account.

```bash
aws configservice describe-delivery-channels
```

**Example Output:**
```json
{
  "DeliveryChannels": [
    {
      "name": "default",
      "s3BucketName": "my-config-bucket-123456789012",
      "s3KeyPrefix": "config-logs",
      "snsTopicARN": "arn:aws:sns:us-east-1:123456789012:my-config-topic",
      "configSnapshotDeliveryProperties": {
        "deliveryFrequency": "TwentyFour_Hours"
      }
    }
  ]
}
```

---

### 12. Deliver Config Snapshot

Schedules delivery of a configuration snapshot to the Amazon S3 bucket in the specified delivery channel.

```bash
aws configservice deliver-config-snapshot \
  --delivery-channel-name default
```

**Example Output:**
```json
{
  "configSnapshotId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

---

### 13. List Discovered Resources

Lists the resource types that AWS Config has discovered in your account.

```bash
aws configservice list-discovered-resources \
  --resource-type AWS::EC2::Instance \
  --limit 50 \
  --include-deleted-resources false
```

**Example Output:**
```json
{
  "resourceIdentifiers": [
    {
      "resourceType": "AWS::EC2::Instance",
      "resourceId": "i-0abc1234def567890",
      "resourceName": "my-web-server",
      "resourceDeletionTime": null
    },
    {
      "resourceType": "AWS::EC2::Instance",
      "resourceId": "i-0def5678abc123456",
      "resourceName": "my-db-server",
      "resourceDeletionTime": null
    }
  ]
}
```

---

### 14. Get Resource Config History

Returns a list of configuration items for the specified resource.

```bash
aws configservice get-resource-config-history \
  --resource-type AWS::EC2::Instance \
  --resource-id i-0abc1234def567890 \
  --limit 10 \
  --chronological-order Reverse
```

**Example Output:**
```json
{
  "configurationItems": [
    {
      "version": "1.3",
      "accountId": "123456789012",
      "configurationItemCaptureTime": "2024-01-15T12:00:00.000Z",
      "configurationItemStatus": "OK",
      "resourceType": "AWS::EC2::Instance",
      "resourceId": "i-0abc1234def567890",
      "resourceName": "my-web-server",
      "awsRegion": "us-east-1",
      "availabilityZone": "us-east-1a",
      "tags": {
        "Environment": "production",
        "Team": "platform"
      }
    }
  ]
}
```

---

### 15. Start Config Rules Evaluation

Runs an on-demand evaluation for the specified Config rules against the last known configuration state.

```bash
aws configservice start-config-rules-evaluation \
  --config-rule-names \
    s3-bucket-public-read-prohibited \
    ec2-instance-no-public-ip \
    iam-root-access-key-check
```

---

## Common Operations

### Create Operations

**Create a Custom Lambda-Backed Config Rule**
```bash
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "custom-ec2-tag-check",
    "Description": "Ensures all EC2 instances have required tags",
    "Source": {
      "Owner": "CUSTOM_LAMBDA",
      "SourceIdentifier": "arn:aws:lambda:us-east-1:123456789012:function:my-config-rule-function",
      "SourceDetails": [
        {
          "EventSource": "aws.config",
          "MessageType": "ConfigurationItemChangeNotification"
        },
        {
          "EventSource": "aws.config",
          "MessageType": "OversizedConfigurationItemChangeNotification"
        }
      ]
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::EC2::Instance"]
    },
    "InputParameters": "{\"requiredTags\": \"Environment,Owner,CostCenter\"}"
  }'
```

**Create a Conformance Pack**
```bash
aws configservice put-conformance-pack \
  --conformance-pack-name operational-best-practices-for-s3 \
  --template-s3-uri s3://my-config-bucket-123456789012/conformance-packs/s3-best-practices.yaml \
  --delivery-s3-bucket my-config-bucket-123456789012 \
  --delivery-s3-key-prefix conformance-pack-results
```

**Create an Aggregator**
```bash
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name my-org-aggregator \
  --organization-aggregation-source '{
    "RoleArn": "arn:aws:iam::123456789012:role/ConfigAggregatorRole",
    "AllAwsRegions": true
  }'
```

---

### Read / Describe Operations

**Describe Delivery Channel Status**
```bash
aws configservice describe-delivery-channel-status \
  --delivery-channel-names default
```

**Get Compliance Details by Resource**
```bash
aws configservice get-compliance-details-by-resource \
  --resource-type AWS::S3::Bucket \
  --resource-id my-production-bucket \
  --compliance-types COMPLIANT NON_COMPLIANT
```

**Describe Conformance Packs**
```bash
aws configservice describe-conformance-packs \
  --conformance-pack-names operational-best-practices-for-s3
```

**Describe Aggregators**
```bash
aws configservice describe-configuration-aggregators \
  --configuration-aggregator-names my-org-aggregator
```

---

### Update Operations

**Update Recording Group (Specific Resource Types Only)**
```bash
aws configservice put-configuration-recorder \
  --configuration-recorder '{
    "name": "default",
    "roleARN": "arn:aws:iam::123456789012:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig",
    "recordingGroup": {
      "allSupported": false,
      "includeGlobalResourceTypes": false,
      "resourceTypes": [
        "AWS::EC2::Instance",
        "AWS::S3::Bucket",
        "AWS::IAM::Role",
        "AWS::RDS::DBInstance",
        "AWS::Lambda::Function"
      ]
    }
  }'
```

**Update Delivery Frequency**
```bash
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "my-config-bucket-123456789012",
    "configSnapshotDeliveryProperties": {
      "deliveryFrequency": "Six_Hours"
    }
  }'
```

---

### Delete Operations

**Delete a Config Rule**
```bash
aws configservice delete-config-rule \
  --config-rule-name custom-ec2-tag-check
```

**Delete a Delivery Channel**
```bash
# Must stop recorder first
aws configservice stop-configuration-recorder \
  --configuration-recorder-name default

aws configservice delete-delivery-channel \
  --delivery-channel-name default
```

**Delete a Configuration Recorder**
```bash
aws configservice delete-configuration-recorder \
  --configuration-recorder-name default
```

**Delete a Conformance Pack**
```bash
aws configservice delete-conformance-pack \
  --conformance-pack-name operational-best-practices-for-s3
```

**Delete Remediation Configuration**
```bash
aws configservice delete-remediation-configuration \
  --config-rule-name s3-bucket-public-read-prohibited
```

---

### List Operations

**List All Config Rules**
```bash
aws configservice describe-config-rules \
  --query 'ConfigRules[*].{Name:ConfigRuleName,