# GuardDuty — Hands-On Labs

## Lab 1: Getting Started with GuardDuty

### Objective

In this lab, you will enable Amazon GuardDuty in a single AWS region, explore the GuardDuty console, generate sample findings to understand the finding types and severity levels, and learn how to interpret and act on threat intelligence findings. By the end of this lab, you will have a functioning GuardDuty detector and understand the core concepts of continuous threat detection.

### Prerequisites

**AWS Services Required:**
- Amazon GuardDuty
- AWS IAM
- AWS CloudTrail (recommended, auto-enabled by GuardDuty)

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "guardduty:*",
        "iam:CreateServiceLinkedRole",
        "iam:GetRole"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS Console access
- AWS CLI v2 installed and configured
- `jq` (optional, for JSON parsing)
- An AWS account with no existing GuardDuty detector in `us-east-1`

**Estimated Cost:** < $1.00 USD (30-day free trial applies to new accounts)  
**Estimated Duration:** 45–60 minutes

---

### Steps

#### Step 1: Enable GuardDuty via the AWS Console

**Console:**
1. Sign in to the AWS Management Console.
2. Navigate to **Services → Security, Identity & Compliance → GuardDuty**.
3. If this is your first time, you will see the GuardDuty welcome page. Click **Get Started**.
4. Review the data sources listed:
   - VPC Flow Logs
   - DNS Logs
   - AWS CloudTrail Management Events
5. Click **Enable GuardDuty**.

**CLI:**
```bash
# Enable GuardDuty and capture the detector ID
DETECTOR_ID=$(aws guardduty create-detector \
  --enable \
  --data-sources '{"S3Logs":{"Enable":true},"Kubernetes":{"AuditLogs":{"Enable":true}}}' \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --region us-east-1 \
  --query 'DetectorId' \
  --output text)

echo "GuardDuty Detector ID: $DETECTOR_ID"

# Save the detector ID for later steps
echo "export DETECTOR_ID=$DETECTOR_ID" >> ~/.bashrc
source ~/.bashrc
```

**Verify:**
```bash
# Confirm the detector is active
aws guardduty get-detector \
  --detector-id $DETECTOR_ID \
  --region us-east-1 \
  --query '{Status: Status, UpdatedAt: UpdatedAt, FindingPublishingFrequency: FindingPublishingFrequency}'
```

**Expected Output:**
```json
{
    "Status": "ENABLED",
    "UpdatedAt": "2024-01-15T10:00:00Z",
    "FindingPublishingFrequency": "FIFTEEN_MINUTES"
}
```

---

#### Step 2: Explore the GuardDuty Dashboard

**Console:**
1. In the GuardDuty console, click **Summary** in the left navigation pane.
2. Review the dashboard sections:
   - **Findings by severity** (High, Medium, Low)
   - **Top threat types**
   - **Top affected resources**
3. Note that the dashboard may be empty at this point — this is expected for a new detector.

**CLI:**
```bash
# List current findings (likely empty for a new detector)
aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --region us-east-1 \
  --query 'FindingIds'
```

**Expected Output:**
```json
[]
```

---

#### Step 3: Generate Sample Findings

GuardDuty provides a built-in mechanism to generate sample findings so you can explore finding types without waiting for real threats.

**Console:**
1. In the GuardDuty console, click **Settings** in the left navigation pane.
2. Scroll down to the **Sample findings** section.
3. Click **Generate sample findings**.
4. Wait approximately 60 seconds, then navigate to **Findings** in the left pane.
5. You should see a list of sample findings prefixed with `[SAMPLE]`.

**CLI:**
```bash
# Generate sample findings
aws guardduty create-sample-findings \
  --detector-id $DETECTOR_ID \
  --finding-types \
    "Backdoor:EC2/Spambot" \
    "Behavior:EC2/NetworkPortUnusual" \
    "CryptoCurrency:EC2/BitcoinTool.B!DNS" \
    "Recon:EC2/PortProbeUnprotectedPort" \
    "UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B" \
    "Trojan:EC2/BlackholeTraffic" \
  --region us-east-1

echo "Sample findings generated. Waiting 30 seconds..."
sleep 30

# List the generated finding IDs
FINDING_IDS=$(aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --region us-east-1 \
  --query 'FindingIds[]' \
  --output json)

echo "Finding IDs: $FINDING_IDS"
```

**Expected Output:**
```
Sample findings generated. Waiting 30 seconds...
Finding IDs: [
    "abc123def456...",
    "xyz789uvw012...",
    ...
]
```

---

#### Step 4: Inspect a Finding in Detail

**Console:**
1. In the **Findings** view, click on any finding (e.g., `CryptoCurrency:EC2/BitcoinTool.B!DNS`).
2. Review the finding details panel on the right:
   - **Finding type** and **Severity**
   - **Resource affected**
   - **Action** section (API call, network connection, etc.)
   - **Actor/Target** information
   - **Evidence** section with threat intelligence details
3. Note the **Finding ID** and **Count** fields.

**CLI:**
```bash
# Get the first finding ID
FIRST_FINDING=$(aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --region us-east-1 \
  --query 'FindingIds[0]' \
  --output text)

# Get detailed information about the finding
aws guardduty get-findings \
  --detector-id $DETECTOR_ID \
  --finding-ids $FIRST_FINDING \
  --region us-east-1 \
  --query 'Findings[0].{Title:Title,Severity:Severity,Type:Type,Description:Description,Region:Region}' \
  --output table
```

**Expected Output:**
```
------------------------------------------------------------------------------------------------------------
|                                              GetFindings                                                  |
+-------------+-----------------------------+----------+---------------------------------------------------+
|  Description| ...cryptocurrency mining... |  Region  |  us-east-1                                        |
+-------------+-----------------------------+----------+---------------------------------------------------+
|  Severity   | 5.0                         |  Title   |  [SAMPLE] EC2 instance querying a domain...       |
+-------------+-----------------------------+----------+---------------------------------------------------+
|  Type       | CryptoCurrency:EC2/...      |          |                                                   |
+-------------+-----------------------------+----------+---------------------------------------------------+
```

---

#### Step 5: Filter and Sort Findings

**Console:**
1. In the **Findings** view, click **Add filter criteria**.
2. Select **Severity** → **High** to filter for only high-severity findings.
3. Click the column header **Severity** to sort findings by severity.
4. Try filtering by **Finding type** to see all findings of a specific category.

**CLI:**
```bash
# Filter findings by severity (High = 7-8.9, Critical = 9+)
aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --finding-criteria '{
    "Criterion": {
      "severity": {
        "Gte": 7
      }
    }
  }' \
  --region us-east-1 \
  --query 'FindingIds'

# Filter findings by type
aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --finding-criteria '{
    "Criterion": {
      "type": {
        "Equals": ["CryptoCurrency:EC2/BitcoinTool.B!DNS"]
      }
    }
  }' \
  --region us-east-1
```

---

#### Step 6: Archive a Finding

**Console:**
1. Select a low-severity sample finding by checking its checkbox.
2. Click **Actions → Archive**.
3. Confirm the action.
4. Toggle the **Current** / **Archived** view to see the archived finding.

**CLI:**
```bash
# Get a low-severity finding to archive
LOW_FINDING=$(aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --finding-criteria '{
    "Criterion": {
      "severity": {
        "Lte": 4
      }
    }
  }' \
  --region us-east-1 \
  --query 'FindingIds[0]' \
  --output text)

# Archive the finding
aws guardduty archive-findings \
  --detector-id $DETECTOR_ID \
  --finding-ids $LOW_FINDING \
  --region us-east-1

echo "Finding $LOW_FINDING has been archived."

# Verify it is archived
aws guardduty get-findings \
  --detector-id $DETECTOR_ID \
  --finding-ids $LOW_FINDING \
  --region us-east-1 \
  --query 'Findings[0].Service.Archived'
```

**Expected Output:**
```
Finding abc123def456... has been archived.
true
```

---

### Verification

Run the following verification checklist:

```bash
#!/bin/bash
echo "=== GuardDuty Lab 1 Verification ==="

# Check 1: Detector exists and is enabled
STATUS=$(aws guardduty get-detector \
  --detector-id $DETECTOR_ID \
  --region us-east-1 \
  --query 'Status' \
  --output text 2>/dev/null)

if [ "$STATUS" = "ENABLED" ]; then
  echo "✅ Check 1 PASSED: GuardDuty detector is ENABLED"
else
  echo "❌ Check 1 FAILED: Detector not found or not enabled"
fi

# Check 2: Sample findings exist
FINDING_COUNT=$(aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --region us-east-1 \
  --query 'length(FindingIds)' \
  --output text)

if [ "$FINDING_COUNT" -gt "0" ]; then
  echo "✅ Check 2 PASSED: $FINDING_COUNT findings exist"
else
  echo "❌ Check 2 FAILED: No findings found"
fi

# Check 3: Archived findings exist
ARCHIVED_COUNT=$(aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --finding-criteria '{"Criterion":{"service.archived":{"Equals":["true"]}}}' \
  --region us-east-1 \
  --query 'length(FindingIds)' \
  --output text)

if [ "$ARCHIVED_COUNT" -gt "0" ]; then
  echo "✅ Check 3 PASSED: $ARCHIVED_COUNT archived findings exist"
else
  echo "❌ Check 3 FAILED: No archived findings found"
fi

echo "=== Verification Complete ==="
```

---

### Cleanup

> ⚠️ **Important:** Disabling GuardDuty will delete all findings and detector configuration. Only do this if you are done with all labs.

**If proceeding to Lab 2, skip cleanup and keep your detector.**

**Console:**
1. Navigate to **GuardDuty → Settings**.
2. Scroll to the bottom and click **Disable GuardDuty**.
3. In the confirmation dialog, type `Disable` and click **Disable**.

**CLI:**
```bash
# Archive all sample findings first (optional cleanup)
ALL_FINDINGS=$(aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --region us-east-1 \
  --query 'FindingIds' \
  --output json | jq -r '.[]' | tr '\n' ' ')

if [ -n "$ALL_FINDINGS" ]; then
  aws guardduty archive-findings \
    --detector-id $DETECTOR_ID \
    --finding-ids $ALL_FINDINGS \
    --region us-east-1
fi

# Delete the GuardDuty detector
aws guardduty delete-detector \
  --detector-id $DETECTOR_ID \
  --region us-east-1

echo "GuardDuty detector $DETECTOR_ID has been deleted."

# Unset environment variables
unset DETECTOR_ID
```

**Verify Cleanup:**
```bash
# This should return an error or empty list
aws guardduty list-detectors --region us-east-1
# Expected: {"DetectorIds": []}
```

---

## Lab 2: Intermediate GuardDuty Configuration

### Objective

In this lab, you will configure GuardDuty with custom threat intelligence lists (IP and domain lists), set up suppression rules to reduce alert fatigue, integrate GuardDuty findings with Amazon EventBridge for automated alerting, and create an SNS notification pipeline that sends email alerts for high-severity findings. You will also enable GuardDuty Malware Protection and S3 Protection features.

### Prerequisites

**AWS Services Required:**
- Amazon GuardDuty (from Lab 1, or re-enabled)
- Amazon EventBridge
- Amazon SNS
- Amazon S3
- AWS IAM
- AWS Lambda (optional, for automated response)

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "guardduty:*",
        "events:*",
        "sns:*",
        "s3:*",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole",
        "lambda:CreateFunction",
        "lambda:AddPermission"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2
- `jq`
- Python 3.x (for Lambda function)
- A valid email address for SNS notifications

**Estimated Cost:** < $2.00 USD  
**Estimated Duration:** 60–90 minutes

---

### Steps

#### Step 1: Re-enable GuardDuty (if cleaned up after Lab 1)

```bash
# Enable GuardDuty with enhanced data sources
DETECTOR_ID=$(aws guardduty create-detector \
  --enable \
  --data-sources '{
    "S3Logs": {"Enable": true},
    "Kubernetes": {"AuditLogs": {"Enable": true}},
    "MalwareProtection": {"ScanEc2InstanceWithFindings": {"EbsVolumes": true}}
  }' \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --region us-east-1 \
  --query 'DetectorId' \
  --output text)

echo "export DETECTOR_ID=$DETECTOR_ID" >> ~/.bashrc
source ~/.bashrc
echo "Detector ID: $DETECTOR_ID"
```

---

#### Step 2: Create a Custom Threat Intelligence IP List

You will create a custom list of known malicious IP addresses and upload it to S3 for GuardDuty to use.

**Create the S3 bucket and upload the threat list:**

```bash
# Set variables
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
THREAT_BUCKET="guardduty-threatintel-$ACCOUNT_ID"
REGION="us-east-1"

# Create S3 bucket
aws s3api create-bucket \
  --bucket