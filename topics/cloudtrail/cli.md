# CloudTrail — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using CloudTrail CLI commands, ensure the following are in place:

**AWS CLI Installation & Configuration**
```bash
# Verify AWS CLI version (v2 recommended)
aws --version

# Configure default profile
aws configure

# Configure a named profile
aws configure --profile cloudtrail-admin
```

**Required IAM Permissions**

Attach the following managed policies or equivalent custom policies to your IAM user/role:

| Policy | Purpose |
|---|---|
| `AWSCloudTrail_FullAccess` | Full CloudTrail management |
| `AWSCloudTrail_ReadOnlyAccess` | Read-only trail inspection |
| `AmazonS3FullAccess` | Required for trail log bucket management |
| `CloudWatchLogsFullAccess` | Required for CloudWatch Logs integration |

**Minimum IAM Policy for CloudTrail Operations**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudtrail:*",
        "s3:CreateBucket",
        "s3:PutBucketPolicy",
        "s3:GetBucketPolicy",
        "logs:CreateLogGroup",
        "logs:DescribeLogGroups",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

**Set Default Region**
```bash
export AWS_DEFAULT_REGION=us-east-1
export AWS_PROFILE=cloudtrail-admin
```

---

## Core Commands

### 1. Create a Trail

```bash
aws cloudtrail create-trail \
  --name my-management-trail \
  --s3-bucket-name my-cloudtrail-logs-bucket \
  --s3-key-prefix cloudtrail-logs \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation
```

**What it does:** Creates a new CloudTrail trail that captures API activity across all AWS regions. Log file validation ensures logs have not been tampered with after delivery.

**Example Output:**
```json
{
    "Name": "my-management-trail",
    "S3BucketName": "my-cloudtrail-logs-bucket",
    "S3KeyPrefix": "cloudtrail-logs",
    "IncludeGlobalServiceEvents": true,
    "IsMultiRegionTrail": true,
    "TrailARN": "arn:aws:cloudtrail:us-east-1:123456789012:trail/my-management-trail",
    "LogFileValidationEnabled": true,
    "HomeRegion": "us-east-1"
}
```

---

### 2. Start Logging

```bash
aws cloudtrail start-logging \
  --name my-management-trail
```

**What it does:** Activates logging for the specified trail. Trails are not automatically logging after creation — this command must be run explicitly.

**Example Output:**
```bash
# No output on success; exit code 0 indicates success
```

---

### 3. Stop Logging

```bash
aws cloudtrail stop-logging \
  --name my-management-trail
```

**What it does:** Pauses log delivery for a trail without deleting it. Useful for cost management during maintenance windows.

---

### 4. Get Trail Status

```bash
aws cloudtrail get-trail-status \
  --name my-management-trail
```

**What it does:** Returns the current status of a trail including whether logging is active, the last log delivery time, and any errors.

**Example Output:**
```json
{
    "IsLogging": true,
    "LatestDeliveryAttemptTime": "2024-03-15T10:22:31Z",
    "LatestDeliveryAttemptSucceeded": "2024-03-15T10:22:31Z",
    "LatestNotificationAttemptTime": "",
    "LatestNotificationAttemptSucceeded": "",
    "LatestDeliveryTime": "2024-03-15T10:22:31Z",
    "LatestDigestDeliveryTime": "2024-03-15T09:00:00Z",
    "LatestDeliveryError": "",
    "LatestNotificationError": "",
    "StartLoggingTime": "2024-01-01T00:00:00Z",
    "StopLoggingTime": ""
}
```

---

### 5. Describe Trails

```bash
aws cloudtrail describe-trails \
  --include-shadow-trails false \
  --trail-name-list my-management-trail my-data-trail
```

**What it does:** Returns configuration details for one or more trails. The `--include-shadow-trails false` flag limits output to trails in the current region only.

**Example Output:**
```json
{
    "trailList": [
        {
            "Name": "my-management-trail",
            "S3BucketName": "my-cloudtrail-logs-bucket",
            "S3KeyPrefix": "cloudtrail-logs",
            "IncludeGlobalServiceEvents": true,
            "IsMultiRegionTrail": true,
            "HomeRegion": "us-east-1",
            "TrailARN": "arn:aws:cloudtrail:us-east-1:123456789012:trail/my-management-trail",
            "LogFileValidationEnabled": true,
            "HasCustomEventSelectors": true,
            "HasInsightSelectors": false,
            "IsOrganizationTrail": false
        }
    ]
}
```

---

### 6. Get Trail

```bash
aws cloudtrail get-trail \
  --name my-management-trail
```

**What it does:** Returns settings information for one specific trail, including ARN and all configuration details.

---

### 7. Update a Trail

```bash
aws cloudtrail update-trail \
  --name my-management-trail \
  --s3-bucket-name my-new-cloudtrail-logs-bucket \
  --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:123456789012:log-group:CloudTrail/DefaultLogGroup:* \
  --cloud-watch-logs-role-arn arn:aws:iam::123456789012:role/CloudTrail_CloudWatchLogs_Role \
  --enable-log-file-validation
```

**What it does:** Modifies an existing trail's configuration. You can change the S3 bucket, add CloudWatch Logs integration, or update SNS notifications.

---

### 8. Delete a Trail

```bash
aws cloudtrail delete-trail \
  --name my-management-trail
```

**What it does:** Permanently deletes a trail. This stops log delivery but does **not** delete existing logs in the S3 bucket.

---

### 9. List Trails

```bash
aws cloudtrail list-trails
```

**What it does:** Lists all trails in all regions for the current account, including their home regions and ARNs.

**Example Output:**
```json
{
    "Trails": [
        {
            "TrailARN": "arn:aws:cloudtrail:us-east-1:123456789012:trail/my-management-trail",
            "Name": "my-management-trail",
            "HomeRegion": "us-east-1"
        },
        {
            "TrailARN": "arn:aws:cloudtrail:eu-west-1:123456789012:trail/my-eu-trail",
            "Name": "my-eu-trail",
            "HomeRegion": "eu-west-1"
        }
    ]
}
```

---

### 10. Lookup Events

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ConsoleLogin \
  --start-time 2024-03-01T00:00:00Z \
  --end-time 2024-03-15T23:59:59Z \
  --max-results 50
```

**What it does:** Searches the last 90 days of management events. Supports filtering by event name, username, resource name, resource type, event source, or event ID.

**Example Output:**
```json
{
    "Events": [
        {
            "EventId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
            "EventName": "ConsoleLogin",
            "ReadOnly": "No",
            "EventTime": "2024-03-15T08:30:00Z",
            "EventSource": "signin.amazonaws.com",
            "Username": "john.doe",
            "Resources": [],
            "CloudTrailEvent": "{...}"
        }
    ],
    "NextToken": "eyJhbGciOiJIUzI1NiJ9..."
}
```

---

### 11. Put Event Selectors

```bash
aws cloudtrail put-event-selectors \
  --trail-name my-management-trail \
  --event-selectors '[
    {
      "ReadWriteType": "All",
      "IncludeManagementEvents": true,
      "DataResources": [
        {
          "Type": "AWS::S3::Object",
          "Values": ["arn:aws:s3:::my-sensitive-bucket/"]
        },
        {
          "Type": "AWS::Lambda::Function",
          "Values": ["arn:aws:lambda:us-east-1:123456789012:function:my-critical-function"]
        }
      ]
    }
  ]'
```

**What it does:** Configures which API calls are logged by the trail. You can enable data events for S3, Lambda, DynamoDB, and other services in addition to management events.

---

### 12. Get Event Selectors

```bash
aws cloudtrail get-event-selectors \
  --trail-name my-management-trail
```

**What it does:** Returns the current event selector configuration for a trail, showing which data events and management events are being captured.

**Example Output:**
```json
{
    "TrailARN": "arn:aws:cloudtrail:us-east-1:123456789012:trail/my-management-trail",
    "EventSelectors": [
        {
            "ReadWriteType": "All",
            "IncludeManagementEvents": true,
            "DataResources": [
                {
                    "Type": "AWS::S3::Object",
                    "Values": ["arn:aws:s3:::my-sensitive-bucket/"]
                }
            ],
            "ExcludeManagementEventSources": []
        }
    ]
}
```

---

### 13. Put Insight Selectors

```bash
aws cloudtrail put-insight-selectors \
  --trail-name my-management-trail \
  --insight-selectors '[
    {"InsightType": "ApiCallRateInsight"},
    {"InsightType": "ApiErrorRateInsight"}
  ]'
```

**What it does:** Enables CloudTrail Insights on a trail to automatically detect unusual API activity patterns, such as spikes in API call rates or error rates.

---

### 14. Get Insight Selectors

```bash
aws cloudtrail get-insight-selectors \
  --trail-name my-management-trail
```

**What it does:** Returns the Insights configuration for the specified trail.

**Example Output:**
```json
{
    "TrailARN": "arn:aws:cloudtrail:us-east-1:123456789012:trail/my-management-trail",
    "InsightSelectors": [
        {
            "InsightType": "ApiCallRateInsight"
        },
        {
            "InsightType": "ApiErrorRateInsight"
        }
    ]
}
```

---

### 15. Validate Log File

```bash
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:us-east-1:123456789012:trail/my-management-trail \
  --start-time 2024-03-01T00:00:00Z \
  --end-time 2024-03-15T23:59:59Z \
  --verbose
```

**What it does:** Validates the integrity of CloudTrail log files using the digest files. Confirms whether logs have been modified, deleted, or forged after delivery to S3.

**Example Output:**
```
Validating log files for trail arn:aws:cloudtrail:us-east-1:123456789012:trail/my-management-trail

2024-03-15T10:00:00Z: s3://my-cloudtrail-logs-bucket/AWSLogs/123456789012/CloudTrail/us-east-1/2024/03/15/123456789012_CloudTrail_us-east-1_20240315T1000Z_abc123.json.gz	valid
2024-03-15T09:00:00Z: s3://my-cloudtrail-logs-bucket/AWSLogs/123456789012/CloudTrail/us-east-1/2024/03/15/123456789012_CloudTrail_us-east-1_20240315T0900Z_def456.json.gz	valid

Results requested for 2024-03-01T00:00:00Z to 2024-03-15T23:59:59Z
14 log files validated, 0 log files INVALID, 0 log files DELETED
```

---

## Common Operations

### Create Operations

```bash
# Create a basic single-region trail
aws cloudtrail create-trail \
  --name my-single-region-trail \
  --s3-bucket-name my-cloudtrail-logs-bucket

# Create a multi-region trail with SNS notifications
aws cloudtrail create-trail \
  --name my-multiregion-trail \
  --s3-bucket-name my-cloudtrail-logs-bucket \
  --is-multi-region-trail \
  --include-global-service-events \
  --sns-topic-name CloudTrailAlerts \
  --enable-log-file-validation

# Create an organization trail (requires AWS Organizations)
aws cloudtrail create-trail \
  --name my-org-trail \
  --s3-bucket-name my-org-cloudtrail-bucket \
  --is-organization-trail \
  --is-multi-region-trail \
  --enable-log-file-validation

# Create a trail with CloudWatch Logs integration
aws cloudtrail create-trail \
  --name my-cw-trail \
  --s3-bucket-name my-cloudtrail-logs-bucket \
  --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:123456789012:log-group:CloudTrailLogs:* \
  --cloud-watch-logs-role-arn arn:aws:iam::123456789012:role/CloudTrail_CWLogs_Role \
  --enable-log-file-validation
```

---

### Read / Describe Operations

```bash
# List all trails in the current account
aws cloudtrail list-trails

# Describe a specific trail
aws cloudtrail describe-trails \
  --trail-name-list my-management-trail

# Get full trail configuration
aws cloudtrail get-trail \
  --name my-management-trail

# Get current logging status
aws cloudtrail get-trail-status \
  --name my-management-trail

# Get event selectors
aws cloudtrail get-event-selectors \
  --trail-name my-management-trail

# Get insight selectors
aws cloudtrail get-insight-selectors \
  --trail-name my-management-trail

# List tags on a trail
aws cloudtrail list-tags \
  --resource-id-list arn:aws:cloudtrail:us-east-1:123456789012:trail/my-management-trail
```

---

### Update Operations

```bash
# Update the S3 bucket for a trail
aws cloudtrail update-trail \
  --name my-management-trail \
  --s3-bucket-name my-new-cloudtrail-logs-bucket

# Enable multi-region on an existing trail
aws cloudtrail update-trail \
  --name my-management-trail \
  --is-multi-region-trail

# Add