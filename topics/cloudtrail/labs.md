# CloudTrail — Hands-On Labs

## Lab 1: Getting Started with CloudTrail

### Objective
In this lab, you will enable AWS CloudTrail for your AWS account, create a trail that logs management events to an S3 bucket, and learn how to query and interpret CloudTrail event history. By the end, you will understand how CloudTrail captures API activity and how to locate specific events using the AWS Console and CLI.

---

### Prerequisites

**AWS Services Required:**
- AWS CloudTrail
- Amazon S3
- AWS IAM

**IAM Permissions Required:**
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
        "s3:ListBucket",
        "s3:GetObject",
        "iam:GetUser",
        "iam:ListUsers"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS Management Console access
- AWS CLI v2 installed and configured (`aws configure`)
- A text editor (for reviewing JSON output)
- `jq` (optional, for formatting CLI JSON output)

**Estimated Cost:** < $0.50 for the duration of this lab  
**Estimated Time:** 30–45 minutes

---

### Steps

#### Step 1: Create an S3 Bucket for CloudTrail Logs

**Console:**
1. Navigate to **S3** in the AWS Console.
2. Click **Create bucket**.
3. Set **Bucket name** to `cloudtrail-logs-<your-account-id>-lab1` (replace `<your-account-id>` with your 12-digit AWS account ID).
4. Set **AWS Region** to `us-east-1`.
5. Under **Block Public Access settings**, ensure **all four checkboxes are checked** (block all public access).
6. Leave all other settings as default and click **Create bucket**.

**CLI:**
```bash
# Set variables
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="cloudtrail-logs-${ACCOUNT_ID}-lab1"
REGION="us-east-1"

# Create the S3 bucket
aws s3api create-bucket \
  --bucket "$BUCKET_NAME" \
  --region "$REGION"

echo "Bucket created: $BUCKET_NAME"
```

**Verify:**
```bash
aws s3api head-bucket --bucket "$BUCKET_NAME" && echo "Bucket exists and is accessible"
```

**Expected Output:**
```
Bucket exists and is accessible
```

---

#### Step 2: Attach the Required Bucket Policy for CloudTrail

CloudTrail requires specific S3 bucket permissions to write log files.

**Console:**
1. In the S3 Console, click on your newly created bucket.
2. Go to the **Permissions** tab.
3. Scroll to **Bucket policy** and click **Edit**.
4. Paste the following policy (replace `<your-account-id>` and `<bucket-name>`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::<bucket-name>",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudtrail:us-east-1:<your-account-id>:trail/lab1-management-trail"
        }
      }
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::<bucket-name>/AWSLogs/<your-account-id>/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control",
          "AWS:SourceArn": "arn:aws:cloudtrail:us-east-1:<your-account-id>:trail/lab1-management-trail"
        }
      }
    }
  ]
}
```

5. Click **Save changes**.

**CLI:**
```bash
# Create the bucket policy file
cat > /tmp/cloudtrail-bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudtrail:${REGION}:${ACCOUNT_ID}:trail/lab1-management-trail"
        }
      }
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/AWSLogs/${ACCOUNT_ID}/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control",
          "AWS:SourceArn": "arn:aws:cloudtrail:${REGION}:${ACCOUNT_ID}:trail/lab1-management-trail"
        }
      }
    }
  ]
}
EOF

# Apply the bucket policy
aws s3api put-bucket-policy \
  --bucket "$BUCKET_NAME" \
  --policy file:///tmp/cloudtrail-bucket-policy.json

echo "Bucket policy applied successfully"
```

**Verify:**
```bash
aws s3api get-bucket-policy --bucket "$BUCKET_NAME" --query Policy --output text | python3 -m json.tool
```

**Expected Output:** The policy JSON printed to the terminal with both statements visible.

---

#### Step 3: Create a CloudTrail Trail

**Console:**
1. Navigate to **CloudTrail** in the AWS Console.
2. Click **Create trail**.
3. Set **Trail name** to `lab1-management-trail`.
4. Under **Storage location**, select **Use existing S3 bucket** and choose `cloudtrail-logs-<your-account-id>-lab1`.
5. Leave **Log file SSE-KMS encryption** unchecked for now.
6. Under **CloudWatch Logs**, leave disabled for Lab 1.
7. Click **Next**.
8. Under **Choose log events**, ensure **Management events** is selected.
9. Set **API activity** to **Read** and **Write**.
10. Click **Next**, review, and click **Create trail**.

**CLI:**
```bash
aws cloudtrail create-trail \
  --name "lab1-management-trail" \
  --s3-bucket-name "$BUCKET_NAME" \
  --include-global-service-events \
  --is-multi-region-trail false \
  --region "$REGION"

echo "Trail created successfully"
```

**Verify:**
```bash
aws cloudtrail describe-trails \
  --trail-name-list "lab1-management-trail" \
  --query "trailList[0].{Name:Name,S3Bucket:S3BucketName,Status:HasCustomEventSelectors}" \
  --output table
```

**Expected Output:**
```
----------------------------------------------------------------------
|                          DescribeTrails                            |
+-----------+----------------------------------+--------------------+
|   Name    |           S3Bucket               |       Status       |
+-----------+----------------------------------+--------------------+
|  lab1-management-trail | cloudtrail-logs-xxxx-lab1 | False         |
+-----------+----------------------------------+--------------------+
```

---

#### Step 4: Start Logging

**Console:**
1. In the CloudTrail Console, click on **lab1-management-trail**.
2. If the trail shows **Logging: OFF**, click the toggle to turn logging **ON**.

**CLI:**
```bash
aws cloudtrail start-logging \
  --name "lab1-management-trail"

# Confirm logging is active
aws cloudtrail get-trail-status \
  --name "lab1-management-trail" \
  --query "{IsLogging:IsLogging,LatestDeliveryTime:LatestDeliveryTime}" \
  --output table
```

**Expected Output:**
```
-----------------------------------------
|           GetTrailStatus              |
+--------------------+------------------+
|     IsLogging      |       True       |
+--------------------+------------------+
```

---

#### Step 5: Generate API Activity to Log

Run some harmless AWS API calls to generate events that CloudTrail will capture.

**CLI:**
```bash
# Generate several API events
aws iam list-users
aws s3 ls
aws sts get-caller-identity
aws ec2 describe-regions --output table

echo "API calls completed — events are being logged"
```

Wait **2–5 minutes** for events to appear in Event History.

---

#### Step 6: View Events in Event History

**Console:**
1. Navigate to **CloudTrail → Event history**.
2. Use the filter dropdown to filter by **Event name** = `ListUsers`.
3. Click on the event to see the full JSON record including:
   - `userIdentity` — who made the call
   - `eventTime` — when it occurred
   - `sourceIPAddress` — where it came from
   - `requestParameters` — what was requested

**CLI:**
```bash
# Look up the ListUsers event
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ListUsers \
  --max-results 3 \
  --query "Events[*].{EventName:EventName,EventTime:EventTime,Username:Username}" \
  --output table
```

**Expected Output:**
```
------------------------------------------------------
|                    LookupEvents                    |
+-------------+-------------------+-----------------+
|  EventName  |    EventTime      |    Username     |
+-------------+-------------------+-----------------+
|  ListUsers  | 2024-01-15T10:23  |  your-iam-user  |
+-------------+-------------------+-----------------+
```

```bash
# Get the full event JSON
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ListUsers \
  --max-results 1 \
  --query "Events[0].CloudTrailEvent" \
  --output text | python3 -m json.tool
```

---

#### Step 7: Verify Log Files in S3

After a few minutes, CloudTrail will deliver compressed log files to your S3 bucket.

**CLI:**
```bash
# List log files in the bucket (may take 5-15 minutes after trail creation)
aws s3 ls "s3://${BUCKET_NAME}/AWSLogs/${ACCOUNT_ID}/CloudTrail/${REGION}/" --recursive

# Download and inspect a log file (replace <date-path> with actual path from above)
# Example:
LOG_FILE=$(aws s3 ls "s3://${BUCKET_NAME}/AWSLogs/${ACCOUNT_ID}/CloudTrail/${REGION}/" \
  --recursive | sort | tail -1 | awk '{print $4}')

aws s3 cp "s3://${BUCKET_NAME}/${LOG_FILE}" /tmp/cloudtrail-log.json.gz
gunzip /tmp/cloudtrail-log.json.gz
cat /tmp/cloudtrail-log.json | python3 -m json.tool | head -80
```

**Expected Output:** A JSON file with a `Records` array containing multiple CloudTrail event objects.

---

### Verification

Run the following checklist to confirm lab completion:

```bash
echo "=== Lab 1 Verification Checklist ==="

# 1. Verify trail exists
echo -n "1. Trail exists: "
aws cloudtrail describe-trails --trail-name-list "lab1-management-trail" \
  --query "length(trailList)" --output text | grep -q "1" && echo "PASS" || echo "FAIL"

# 2. Verify logging is active
echo -n "2. Logging is active: "
aws cloudtrail get-trail-status --name "lab1-management-trail" \
  --query "IsLogging" --output text | grep -q "True" && echo "PASS" || echo "FAIL"

# 3. Verify S3 bucket exists
echo -n "3. S3 bucket exists: "
aws s3api head-bucket --bucket "$BUCKET_NAME" 2>/dev/null && echo "PASS" || echo "FAIL"

# 4. Verify events are being captured
echo -n "4. Events captured in history: "
EVENT_COUNT=$(aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ListUsers \
  --max-results 5 \
  --query "length(Events)" --output text)
[ "$EVENT_COUNT" -gt "0" ] && echo "PASS ($EVENT_COUNT events found)" || echo "FAIL"

echo "=== Verification Complete ==="
```

---

### Cleanup

> ⚠️ **Important:** Complete all cleanup steps to avoid ongoing S3 storage charges.

```bash
echo "=== Starting Lab 1 Cleanup ==="

# Step 1: Stop logging
aws cloudtrail stop-logging --name "lab1-management-trail"
echo "1. Logging stopped"

# Step 2: Delete the trail
aws cloudtrail delete-trail --name "lab1-management-trail"
echo "2. Trail deleted"

# Step 3: Empty the S3 bucket (required before deletion)
aws s3 rm "s3://${BUCKET_NAME}" --recursive
echo "3. Bucket contents deleted"

# Step 4: Delete the S3 bucket
aws s3api delete-bucket --bucket "$BUCKET_NAME" --region "$REGION"
echo "4. S3 bucket deleted"

# Step 5: Clean up temp files
rm -f /tmp/cloudtrail-bucket-policy.json /tmp/cloudtrail-log.json.gz /tmp/cloudtrail-log.json
echo "5. Temp files cleaned up"

echo "=== Cleanup Complete ==="
```

---

## Lab 2: Intermediate CloudTrail Configuration

### Objective
In this lab, you will configure a CloudTrail trail with advanced features including **CloudWatch Logs integration**, **SNS alerting for specific API events**, and **metric filters** to detect security-sensitive actions such as unauthorized API calls and IAM changes. You will also configure **data events** for S3 object-level logging and explore CloudTrail Insights.

---

### Prerequisites

**AWS Services Required:**
- AWS CloudTrail
- Amazon S3
- Amazon CloudWatch (Logs, Metrics, Alarms)
- Amazon SNS
- AWS IAM
- AWS KMS (optional encryption)

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudtrail:*",
        "s3:*",
        "cloudwatch:*",
        "logs:*",
        "sns:*",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole",
        "iam:GetRole"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 configured
- AWS Console access
- `jq` recommended for JSON parsing

**Estimated Cost:** ~$1–2 for the duration of this lab  
**Estimated Time:** 60–90 minutes

---