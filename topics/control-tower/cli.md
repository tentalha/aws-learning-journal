# Control Tower — AWS CLI Commands

## Setup & Configuration

### Prerequisites

AWS Control Tower uses two CLI namespaces:
- `aws controltower` — the primary Control Tower API (GA as of AWS CLI v2.9+)
- `aws organizations` — required for account/OU management

Ensure you have **AWS CLI v2.9.0 or later** installed:

```bash
aws --version
# aws-cli/2.13.0 Python/3.11.4 ...
```

### Required IAM Permissions

The following managed policies are recommended for full Control Tower CLI access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "controltower:*",
        "organizations:DescribeOrganization",
        "organizations:ListAccounts",
        "organizations:ListOrganizationalUnitsForParent",
        "organizations:DescribeOrganizationalUnit",
        "sso:ListInstances",
        "sso:DescribeRegisteredRegions",
        "cloudformation:DescribeStacks"
      ],
      "Resource": "*"
    }
  ]
}
```

### Profile & Region Configuration

Control Tower is a global service but must be called from the **home region** where it was set up (e.g., `us-east-1`):

```bash
# Configure a dedicated profile for Control Tower management
aws configure --profile controltower-admin
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json

# Set environment variables for convenience
export AWS_PROFILE=controltower-admin
export AWS_DEFAULT_REGION=us-east-1
```

---

## Core Commands

### 1. List All Landing Zones

```bash
aws controltower list-landing-zones \
  --profile controltower-admin \
  --region us-east-1
```

**What it does:** Returns all landing zones provisioned in your account. A landing zone is the multi-account environment set up by Control Tower.

**Example Output:**
```json
{
  "landingZones": [
    {
      "arn": "arn:aws:controltower:us-east-1:123456789012:landingzone/ABCDEF1234567890",
      "id": "ABCDEF1234567890"
    }
  ]
}
```

---

### 2. Get Landing Zone Details

```bash
aws controltower get-landing-zone \
  --landing-zone-identifier "arn:aws:controltower:us-east-1:123456789012:landingzone/ABCDEF1234567890" \
  --region us-east-1
```

**What it does:** Retrieves detailed configuration of a specific landing zone, including version, status, and manifest.

**Example Output:**
```json
{
  "landingZone": {
    "arn": "arn:aws:controltower:us-east-1:123456789012:landingzone/ABCDEF1234567890",
    "id": "ABCDEF1234567890",
    "version": "3.2",
    "status": "ACTIVE",
    "latestAvailableVersion": "3.3",
    "driftStatus": {
      "status": "NOT_DRIFTED"
    },
    "manifest": {
      "governedRegions": ["us-east-1", "us-west-2"],
      "organizationStructure": {
        "security": {
          "name": "Security"
        },
        "sandbox": {
          "name": "Sandbox"
        }
      },
      "centralizedLogging": {
        "accountId": "111122223333",
        "configurations": {
          "loggingBucket": {
            "retentionDays": 365
          },
          "accessLoggingBucket": {
            "retentionDays": 3650
          }
        },
        "enabled": true
      }
    }
  }
}
```

---

### 3. List All Enabled Controls

```bash
aws controltower list-enabled-controls \
  --target-identifier "arn:aws:organizations::123456789012:ou/o-exampleorgid11/ou-exmpl-15precabc" \
  --region us-east-1
```

**What it does:** Lists all controls (guardrails) currently enabled on a specific Organizational Unit (OU).

**Example Output:**
```json
{
  "enabledControls": [
    {
      "controlIdentifier": "arn:aws:controltower:us-east-1::control/AWS-GR_ENCRYPTED_VOLUMES",
      "targetIdentifier": "arn:aws:organizations::123456789012:ou/o-exampleorgid11/ou-exmpl-15precabc",
      "arn": "arn:aws:controltower:us-east-1:123456789012:enabledcontrol/GHIJKL9876543210",
      "statusSummary": {
        "lastOperationIdentifier": "op-abc123def456",
        "status": "SUCCEEDED"
      }
    }
  ]
}
```

---

### 4. Enable a Control on an OU

```bash
aws controltower enable-control \
  --control-identifier "arn:aws:controltower:us-east-1::control/AWS-GR_ENCRYPTED_VOLUMES" \
  --target-identifier "arn:aws:organizations::123456789012:ou/o-exampleorgid11/ou-exmpl-15precabc" \
  --region us-east-1
```

**What it does:** Applies a specific Control Tower control (guardrail) to an OU. This is an asynchronous operation — use the returned `operationIdentifier` to track progress.

**Example Output:**
```json
{
  "operationIdentifier": "op-abc123def456ghi789"
}
```

---

### 5. Disable a Control on an OU

```bash
aws controltower disable-control \
  --control-identifier "arn:aws:controltower:us-east-1::control/AWS-GR_ENCRYPTED_VOLUMES" \
  --target-identifier "arn:aws:organizations::123456789012:ou/o-exampleorgid11/ou-exmpl-15precabc" \
  --region us-east-1
```

**What it does:** Removes a previously enabled control from the specified OU. Also asynchronous.

**Example Output:**
```json
{
  "operationIdentifier": "op-xyz987uvw654rst321"
}
```

---

### 6. Get Control Operation Status

```bash
aws controltower get-control-operation \
  --operation-identifier "op-abc123def456ghi789" \
  --region us-east-1
```

**What it does:** Polls the status of an asynchronous enable/disable control operation.

**Example Output:**
```json
{
  "controlOperation": {
    "operationType": "ENABLE_CONTROL",
    "startTime": "2024-01-15T10:30:00.000Z",
    "endTime": "2024-01-15T10:45:22.000Z",
    "status": "SUCCEEDED",
    "statusMessage": "AWS Control Tower successfully enabled control."
  }
}
```

---

### 7. List All Available Controls

```bash
aws controltower list-controls \
  --region us-east-1 \
  --max-results 50
```

**What it does:** Returns all available Control Tower controls (both proactive and detective) that can be applied to OUs.

**Example Output:**
```json
{
  "controls": [
    {
      "arn": "arn:aws:controltower:us-east-1::control/AWS-GR_ENCRYPTED_VOLUMES",
      "name": "AWS-GR_ENCRYPTED_VOLUMES",
      "description": "Detect whether Amazon EBS volumes attached to Amazon EC2 instances are encrypted."
    },
    {
      "arn": "arn:aws:controltower:us-east-1::control/AWS-GR_MFA_ENABLED_FOR_IAM_CONSOLE_ACCESS",
      "name": "AWS-GR_MFA_ENABLED_FOR_IAM_CONSOLE_ACCESS",
      "description": "Detect whether MFA is enabled for AWS IAM users."
    }
  ],
  "nextToken": "eyJuZXh0VG9rZW4..."
}
```

---

### 8. Get Details of a Specific Control

```bash
aws controltower get-control \
  --control-identifier "arn:aws:controltower:us-east-1::control/AWS-GR_ENCRYPTED_VOLUMES" \
  --region us-east-1
```

**What it does:** Returns metadata about a specific control including its behavior, implementation type, and severity.

**Example Output:**
```json
{
  "control": {
    "arn": "arn:aws:controltower:us-east-1::control/AWS-GR_ENCRYPTED_VOLUMES",
    "name": "AWS-GR_ENCRYPTED_VOLUMES",
    "description": "Detect whether Amazon EBS volumes attached to Amazon EC2 instances are encrypted.",
    "behavior": "DETECTIVE",
    "implementationType": "AWS_MANAGED",
    "severity": "HIGH",
    "controlMappings": [
      {
        "controlMappingGroups": [
          {
            "mappingType": "COMPLIANCE_FRAMEWORK",
            "item": {
              "commonControlArn": "arn:aws:controltower:::common-control/EncryptDataAtRest"
            }
          }
        ]
      }
    ]
  }
}
```

---

### 9. List Enabled Baselines

```bash
aws controltower list-enabled-baselines \
  --region us-east-1
```

**What it does:** Lists all baselines (sets of controls and configurations) currently enabled on targets such as OUs or accounts.

**Example Output:**
```json
{
  "enabledBaselines": [
    {
      "arn": "arn:aws:controltower:us-east-1:123456789012:enabledbaseline/MNOPQR1234567890",
      "baselineIdentifier": "arn:aws:controltower:us-east-1::baseline/AWSControlTowerBaseline",
      "baselineVersion": "4.0",
      "targetIdentifier": "arn:aws:organizations::123456789012:ou/o-exampleorgid11/ou-exmpl-15precabc",
      "statusSummary": {
        "status": "SUCCEEDED"
      }
    }
  ]
}
```

---

### 10. Enable a Baseline on a Target

```bash
aws controltower enable-baseline \
  --baseline-identifier "arn:aws:controltower:us-east-1::baseline/AWSControlTowerBaseline" \
  --baseline-version "4.0" \
  --target-identifier "arn:aws:organizations::123456789012:ou/o-exampleorgid11/ou-exmpl-newou123" \
  --region us-east-1
```

**What it does:** Applies a baseline configuration to an OU or account, enabling the standard set of Control Tower guardrails and configurations.

---

### 11. List Landing Zone Operations

```bash
aws controltower list-landing-zone-operations \
  --filter '{"statuses": ["SUCCEEDED", "FAILED", "IN_PROGRESS"]}' \
  --region us-east-1
```

**What it does:** Returns the history of all landing zone operations (create, update, reset) with their statuses.

**Example Output:**
```json
{
  "landingZoneOperations": [
    {
      "operationType": "UPDATE",
      "operationIdentifier": "op-lz-abc123xyz789",
      "status": "SUCCEEDED",
      "startTime": "2024-01-10T08:00:00.000Z",
      "endTime": "2024-01-10T08:45:00.000Z"
    }
  ]
}
```

---

### 12. Get a Landing Zone Operation

```bash
aws controltower get-landing-zone-operation \
  --operation-identifier "op-lz-abc123xyz789" \
  --region us-east-1
```

**What it does:** Retrieves the current status and details of a specific landing zone operation.

---

### 13. Reset a Landing Zone

```bash
aws controltower reset-landing-zone \
  --landing-zone-identifier "arn:aws:controltower:us-east-1:123456789012:landingzone/ABCDEF1234567890" \
  --region us-east-1
```

**What it does:** Resets the landing zone to repair drift or apply the latest configuration. This is a long-running operation.

**Example Output:**
```json
{
  "operationIdentifier": "op-lz-reset123abc456"
}
```

---

### 14. List Baselines

```bash
aws controltower list-baselines \
  --region us-east-1
```

**What it does:** Lists all available baseline definitions that can be applied to OUs and accounts.

---

### 15. Get Enabled Baseline Details

```bash
aws controltower get-enabled-baseline \
  --enabled-baseline-identifier "arn:aws:controltower:us-east-1:123456789012:enabledbaseline/MNOPQR1234567890" \
  --region us-east-1
```

**What it does:** Returns detailed status and configuration of a specific enabled baseline including parameter values.

---

## Common Operations

### Create / Enable

#### Enable a New Control on an OU
```bash
# Enable the MFA enforcement control
aws controltower enable-control \
  --control-identifier "arn:aws:controltower:us-east-1::control/AWS-GR_MFA_ENABLED_FOR_IAM_CONSOLE_ACCESS" \
  --target-identifier "arn:aws:organizations::123456789012:ou/o-exampleorgid11/ou-exmpl-15precabc" \
  --parameters '[{"key": "AllowedRegions", "value": "[\"us-east-1\",\"us-west-2\"]"}]' \
  --region us-east-1
```

#### Enable a Baseline with Parameters
```bash
aws controltower enable-baseline \
  --baseline-identifier "arn:aws:controltower:us-east-1::baseline/AWSControlTowerBaseline" \
  --baseline-version "4.0" \
  --target-identifier "arn:aws:organizations::123456789012:ou/o-exampleorgid11/ou-exmpl-newou123" \
  --parameters '[{"key": "IdentityCenterEnabledBaselineArn", "value": "arn:aws:controltower:us-east-1:123456789012:enabledbaseline/IDENTITYCENTER123"}]' \
  --region us-east-1
```

---

### Read / Describe

#### Get Details of an Enabled Control
```bash
aws controltower get-enabled-control \
  --enabled-control-identifier "arn:aws:controltower:us-east-1:123456789012:enabledcontrol/GHIJKL9876543210" \
  --region us-east-1
```

#### Describe a Landing Zone
```bash
aws controltower get-landing-zone \
  --landing-zone-identifier "arn:aws:controltower:us-east-1:123456789012:landingzone/ABCDEF1234567890" \
  --region us-east-1
```

---

### Update

#### Update a Landing Zone to a New Version
```bash
aws controltower update-landing-zone \
  --landing-zone-identifier "arn:aws:controltower:us-east-1:123456789012:landingzone/ABCDEF1234567890" \
  --version "3.3" \
  --manifest file://landing-zone-manifest.json \
  --region