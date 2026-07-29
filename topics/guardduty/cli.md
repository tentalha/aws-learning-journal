# GuardDuty — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using GuardDuty CLI commands, ensure the following are in place:

**AWS CLI Version**
```bash
aws --version
# Requires AWS CLI v2.x recommended (minimum v1.16+)
```

**Configure AWS CLI Profile**
```bash
aws configure --profile guardduty-admin
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

### Required IAM Permissions

Attach the following managed policies or create a custom policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "guardduty:*",
        "iam:CreateServiceLinkedRole",
        "organizations:EnableAWSServiceAccess",
        "organizations:ListDelegatedAdministrators",
        "organizations:RegisterDelegatedAdministrator"
      ],
      "Resource": "*"
    }
  ]
}
```

**AWS Managed Policies:**
- `AmazonGuardDutyFullAccess` — Full access to GuardDuty
- `AmazonGuardDutyReadOnlyAccess` — Read-only access

**Set default region and output format:**
```bash
export AWS_DEFAULT_REGION=us-east-1
export AWS_DEFAULT_OUTPUT=json
```

---

## Core Commands

### 1. Enable GuardDuty (Create Detector)

```bash
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --data-sources '{"S3Logs":{"Enable":true},"Kubernetes":{"AuditLogs":{"Enable":true}},"MalwareProtection":{"ScanEc2InstanceWithFindings":{"EbsVolumes":true}}}' \
  --tags '{"Environment":"Production","Owner":"SecurityTeam"}'
```

**What it does:** Enables GuardDuty in the current AWS account and region by creating a detector. Returns a `DetectorId` used in all subsequent commands.

**Example Output:**
```json
{
    "DetectorId": "abc123def456abc123def456abc123de"
}
```

---

### 2. List All Detectors

```bash
aws guardduty list-detectors
```

**What it does:** Lists all GuardDuty detector IDs in the current region and account.

**Example Output:**
```json
{
    "DetectorIds": [
        "abc123def456abc123def456abc123de"
    ]
}
```

---

### 3. Get Detector Details

```bash
aws guardduty get-detector \
  --detector-id abc123def456abc123def456abc123de
```

**What it does:** Retrieves configuration details for a specific detector including status, data sources, and finding frequency.

**Example Output:**
```json
{
    "CreatedAt": "2024-01-15T10:30:00Z",
    "FindingPublishingFrequency": "FIFTEEN_MINUTES",
    "ServiceRole": "arn:aws:iam::123456789012:role/aws-service-role/guardduty.amazonaws.com/AWSServiceRoleForAmazonGuardDuty",
    "Status": "ENABLED",
    "UpdatedAt": "2024-03-01T08:00:00Z",
    "DataSources": {
        "CloudTrail": {"Status": "ENABLED"},
        "DNSLogs": {"Status": "ENABLED"},
        "FlowLogs": {"Status": "ENABLED"},
        "S3Logs": {"Status": "ENABLED"},
        "Kubernetes": {"AuditLogs": {"Status": "ENABLED"}},
        "MalwareProtection": {"ScanEc2InstanceWithFindings": {"EbsVolumes": {"Status": "ENABLED"}}}
    },
    "Tags": {
        "Environment": "Production"
    }
}
```

---

### 4. List Findings

```bash
aws guardduty list-findings \
  --detector-id abc123def456abc123def456abc123de \
  --finding-criteria '{"Criterion":{"severity":{"Gte":7},"service.archived":{"Eq":["false"]}}}' \
  --sort-criteria '{"AttributeName":"severity","OrderBy":"DESC"}' \
  --max-results 50
```

**What it does:** Lists finding IDs filtered by criteria such as severity (≥7 = High), archived status, and sorted by severity descending.

**Example Output:**
```json
{
    "FindingIds": [
        "f1abc123def456abc123def456abc123",
        "f2abc123def456abc123def456abc456"
    ]
}
```

---

### 5. Get Finding Details

```bash
aws guardduty get-findings \
  --detector-id abc123def456abc123def456abc123de \
  --finding-ids f1abc123def456abc123def456abc123 f2abc123def456abc123def456abc456
```

**What it does:** Retrieves full details for one or more findings including threat intelligence, affected resources, and recommended actions.

**Example Output:**
```json
{
    "Findings": [
        {
            "AccountId": "123456789012",
            "Arn": "arn:aws:guardduty:us-east-1:123456789012:detector/abc123/finding/f1abc123",
            "CreatedAt": "2024-03-15T14:22:00Z",
            "Description": "EC2 instance i-0abc123def456 is communicating with a known malicious IP.",
            "Id": "f1abc123def456abc123def456abc123",
            "Region": "us-east-1",
            "Severity": 8.0,
            "Title": "EC2 instance is communicating with a known threat intelligence IP address.",
            "Type": "UnauthorizedAccess:EC2/MaliciousIPCaller.Custom",
            "UpdatedAt": "2024-03-15T14:22:00Z",
            "Service": {
                "Action": {
                    "ActionType": "NETWORK_CONNECTION",
                    "NetworkConnectionAction": {
                        "ConnectionDirection": "OUTBOUND",
                        "RemoteIpDetails": {
                            "IpAddressV4": "203.0.113.42"
                        }
                    }
                },
                "Archived": false,
                "Count": 3
            }
        }
    ]
}
```

---

### 6. Archive Findings

```bash
aws guardduty archive-findings \
  --detector-id abc123def456abc123def456abc123de \
  --finding-ids f1abc123def456abc123def456abc123 f2abc123def456abc123def456abc456
```

**What it does:** Archives one or more findings so they no longer appear in the active findings list. Useful for suppressing false positives.

---

### 7. Unarchive Findings

```bash
aws guardduty unarchive-findings \
  --detector-id abc123def456abc123def456abc123de \
  --finding-ids f1abc123def456abc123def456abc123
```

**What it does:** Restores previously archived findings back to the active findings list for review.

---

### 8. Create a Threat Intelligence Set (ThreatIntelSet)

```bash
aws guardduty create-threat-intel-set \
  --detector-id abc123def456abc123def456abc123de \
  --name "CustomMaliciousIPs" \
  --format TXT \
  --location "s3://my-guardduty-bucket/threat-intel/malicious-ips.txt" \
  --activate \
  --tags '{"Source":"InternalThreatFeed","Environment":"Production"}'
```

**What it does:** Creates a custom threat intelligence set from a file hosted in S3. GuardDuty uses this list to generate findings when resources communicate with listed IPs.

**Example Output:**
```json
{
    "ThreatIntelSetId": "ti123abc456def789abc123def456abc"
}
```

---

### 9. Create an IP Allow List (IPSet)

```bash
aws guardduty create-ip-set \
  --detector-id abc123def456abc123def456abc123de \
  --name "TrustedCorporateIPs" \
  --format TXT \
  --location "s3://my-guardduty-bucket/trusted-ips/corporate-ips.txt" \
  --activate \
  --tags '{"Purpose":"Whitelist","Owner":"NetworkTeam"}'
```

**What it does:** Creates a trusted IP list. GuardDuty will not generate findings for traffic from IPs in this list.

---

### 10. Create a Publishing Destination (Export Findings to S3)

```bash
aws guardduty create-publishing-destination \
  --detector-id abc123def456abc123def456abc123de \
  --destination-type S3 \
  --destination-properties '{"DestinationArn":"arn:aws:s3:::my-guardduty-findings-bucket","KmsKeyArn":"arn:aws:kms:us-east-1:123456789012:key/mrk-abc123def456abc123def456abc123de"}'
```

**What it does:** Configures GuardDuty to export findings to an S3 bucket, optionally encrypted with a KMS key. Useful for long-term storage and SIEM integration.

---

### 11. Invite a Member Account

```bash
aws guardduty invite-members \
  --detector-id abc123def456abc123def456abc123de \
  --account-details '[{"AccountId":"987654321098","Email":"security@member-company.com"}]' \
  --disable-email-notification \
  --message "Please accept this GuardDuty membership invitation from the central security account."
```

**What it does:** Sends an invitation to another AWS account to join as a member in a GuardDuty multi-account setup.

---

### 12. List Members

```bash
aws guardduty list-members \
  --detector-id abc123def456abc123def456abc123de \
  --only-associated true
```

**What it does:** Lists all member accounts associated with the GuardDuty administrator account.

**Example Output:**
```json
{
    "Members": [
        {
            "AccountId": "987654321098",
            "DetectorId": "def456abc123def456abc123def456ab",
            "Email": "security@member-company.com",
            "MasterId": "123456789012",
            "RelationshipStatus": "Enabled",
            "UpdatedAt": "2024-02-01T09:00:00Z"
        }
    ]
}
```

---

### 13. Create a Filter (Suppression Rule)

```bash
aws guardduty create-filter \
  --detector-id abc123def456abc123def456abc123de \
  --name "SuppressDevEnvironmentFindings" \
  --action ARCHIVE \
  --rank 1 \
  --finding-criteria '{
    "Criterion": {
      "resource.instanceDetails.tags.key": {"Eq": ["Environment"]},
      "resource.instanceDetails.tags.value": {"Eq": ["Development"]},
      "type": {"Eq": ["Recon:EC2/PortProbeUnprotectedPort"]}
    }
  }' \
  --description "Suppress port probe findings for development EC2 instances"
```

**What it does:** Creates an auto-archive (suppression) rule that automatically archives findings matching the specified criteria.

---

### 14. Update Finding Feedback

```bash
aws guardduty update-findings-feedback \
  --detector-id abc123def456abc123def456abc123de \
  --finding-ids f1abc123def456abc123def456abc123 \
  --feedback USEFUL \
  --comments "Confirmed malicious activity - incident ticket INC-2024-001 created"
```

**What it does:** Provides feedback on findings to help improve GuardDuty's machine learning models. Values: `USEFUL` or `NOT_USEFUL`.

---

### 15. Disable GuardDuty (Delete Detector)

```bash
aws guardduty delete-detector \
  --detector-id abc123def456abc123def456abc123de
```

**What it does:** Permanently disables GuardDuty for the account in the current region by deleting the detector. **Use with caution.**

---

## Common Operations

### Create Operations

**Create Detector with Full Data Sources**
```bash
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency ONE_HOUR \
  --data-sources '{
    "S3Logs": {"Enable": true},
    "Kubernetes": {"AuditLogs": {"Enable": true}},
    "MalwareProtection": {
      "ScanEc2InstanceWithFindings": {"EbsVolumes": true}
    }
  }'
```

**Create a Sample Finding (for testing)**
```bash
aws guardduty create-sample-findings \
  --detector-id abc123def456abc123def456abc123de \
  --finding-types \
    "UnauthorizedAccess:EC2/SSHBruteForce" \
    "CryptoCurrency:EC2/BitcoinTool.B" \
    "Recon:EC2/PortProbeUnprotectedPort"
```

**Create ThreatIntelSet**
```bash
aws guardduty create-threat-intel-set \
  --detector-id abc123def456abc123def456abc123de \
  --name "APTGroupIPs" \
  --format TXT \
  --location "s3://my-guardduty-bucket/threat-intel/apt-ips.txt" \
  --activate
```

---

### Read / Describe Operations

**Get Detector Configuration**
```bash
aws guardduty get-detector \
  --detector-id abc123def456abc123def456abc123de
```

**Get Findings Statistics**
```bash
aws guardduty get-findings-statistics \
  --detector-id abc123def456abc123def456abc123de \
  --finding-statistic-types COUNT_BY_SEVERITY \
  --finding-criteria '{"Criterion":{"service.archived":{"Eq":["false"]}}}'
```

**Example Output:**
```json
{
    "FindingStatistics": {
        "CountBySeverity": {
            "2.0": 45,
            "5.0": 12,
            "8.0": 3
        }
    }
}
```

**Get IPSet Details**
```bash
aws guardduty get-ip-set \
  --detector-id abc123def456abc123def456abc123de \
  --ip-set-id ipset123abc456def789abc123def456
```

**Get ThreatIntelSet Details**
```bash
aws guardduty get-threat-intel-set \
  --detector-id abc123def456abc123def456abc123de \
  --threat-intel-set-id ti123abc456def789abc123def456abc
```

**Get Master Account Info**
```bash
aws guardduty get-administrator-account \
  --detector-id abc123def456abc123def456abc123de
```

---

### Update Operations

**Update Detector Settings**
```bash
aws guardduty update-detector \
  --detector-id abc123def456abc123def456abc123de \
  --enable \
  --finding-publishing-frequency SIX_HOURS \
  --data-sources '{"S3Logs":{"Enable":true}}'
```

**Update IPSet**
```bash
aws guardduty update-ip-set \
  --detector-id abc123def456abc123def456abc123de \
  --ip-set-id ipset123abc456def789abc123def456 \
  --name "UpdatedTrustedIPs" \
  --location "s3://my-guardduty-bucket/trusted-ips/updated-corporate-ips.txt" \
  --activate
```

**Update ThreatIntelSet**
```bash
aws guardduty