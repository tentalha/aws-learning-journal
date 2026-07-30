# Inspector — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using AWS Inspector CLI commands, ensure the following are in place:

**AWS CLI Version**
```bash
aws --version
# Requires AWS CLI v2.x or v1.x (>= 1.16.x)
```

**Configure AWS CLI**
```bash
aws configure
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
        "inspector2:*",
        "inspector:*",
        "ec2:DescribeInstances",
        "ecr:DescribeRepositories",
        "ecr:DescribeImages",
        "lambda:ListFunctions",
        "iam:CreateServiceLinkedRole",
        "organizations:ListAccounts",
        "organizations:DescribeOrganization"
      ],
      "Resource": "*"
    }
  ]
}
```

> **Note:** AWS Inspector has two versions:
> - **Amazon Inspector Classic** (`aws inspector`) — Legacy, EC2 only
> - **Amazon Inspector v2** (`aws inspector2`) — Current, supports EC2, ECR, Lambda, and Code scanning

**Enable Inspector v2 Service-Linked Role**
```bash
aws iam create-service-linked-role \
  --aws-service-name inspector2.amazonaws.com
```

---

## Core Commands

### 1. Enable Inspector v2

Enables Amazon Inspector v2 for the current account and specified resource types.

```bash
aws inspector2 enable \
  --resource-types EC2 ECR LAMBDA LAMBDA_CODE \
  --region us-east-1
```

**Example Output:**
```json
{
  "accounts": [
    {
      "accountId": "123456789012",
      "resourceStatus": {
        "ec2": "ENABLING",
        "ecr": "ENABLING",
        "lambda": "ENABLING",
        "lambdaCode": "ENABLING"
      },
      "status": "ENABLING"
    }
  ],
  "failedAccounts": []
}
```

---

### 2. Describe Inspector v2 Organization Configuration

Retrieves the Amazon Inspector configuration for an AWS Organization.

```bash
aws inspector2 describe-organization-configuration \
  --region us-east-1
```

**Example Output:**
```json
{
  "autoEnable": {
    "ec2": true,
    "ecr": true,
    "lambda": false,
    "lambdaCode": false
  },
  "maxAccountLimitReached": false
}
```

---

### 3. List Findings

Lists all Inspector v2 findings with optional filters.

```bash
aws inspector2 list-findings \
  --filter-criteria '{
    "findingStatus": [{"comparison": "EQUALS", "value": "ACTIVE"}],
    "severity": [{"comparison": "EQUALS", "value": "CRITICAL"}]
  }' \
  --sort-criteria '{"field": "SEVERITY", "sortOrder": "DESC"}' \
  --max-results 50 \
  --region us-east-1
```

**Example Output:**
```json
{
  "findings": [
    {
      "awsAccountId": "123456789012",
      "description": "A critical vulnerability was detected in package openssl",
      "findingArn": "arn:aws:inspector2:us-east-1:123456789012:finding/abc123def456",
      "firstObservedAt": "2024-01-15T10:30:00Z",
      "inspectorScore": 9.8,
      "packageVulnerabilityDetails": {
        "cvss": [{"baseScore": 9.8, "scoringVector": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H"}],
        "vulnerabilityId": "CVE-2023-12345",
        "vulnerablePackages": [
          {"name": "openssl", "version": "1.1.1k", "fixedInVersion": "1.1.1l"}
        ]
      },
      "severity": "CRITICAL",
      "status": "ACTIVE",
      "title": "CVE-2023-12345 - openssl",
      "type": "PACKAGE_VULNERABILITY"
    }
  ],
  "nextToken": "eyJhbGci..."
}
```

---

### 4. Get Finding

Retrieves details for a specific finding by ARN.

```bash
aws inspector2 get-finding-v2 \
  --finding-arn "arn:aws:inspector2:us-east-1:123456789012:finding/abc123def456" \
  --region us-east-1
```

---

### 5. List Coverage

Lists the resources currently monitored (covered) by Inspector v2.

```bash
aws inspector2 list-coverage \
  --filter-criteria '{
    "resourceType": [{"comparison": "EQUALS", "value": "AWS_EC2_INSTANCE"}],
    "scanStatusCode": [{"comparison": "EQUALS", "value": "ACTIVE"}]
  }' \
  --max-results 100 \
  --region us-east-1
```

**Example Output:**
```json
{
  "coveredResources": [
    {
      "accountId": "123456789012",
      "lastScannedAt": "2024-01-20T14:22:00Z",
      "resourceId": "i-0abcdef1234567890",
      "resourceMetadata": {
        "ec2": {
          "amiId": "ami-0abcdef1234567890",
          "platform": "LINUX",
          "tags": {"Name": "my-web-server", "Environment": "production"}
        }
      },
      "resourceType": "AWS_EC2_INSTANCE",
      "scanStatus": {"reason": "SUCCESSFUL", "statusCode": "ACTIVE"},
      "scanType": "PACKAGE"
    }
  ]
}
```

---

### 6. Get Inspector v2 Status

Checks whether Inspector v2 is enabled for the current account and which resource types are active.

```bash
aws inspector2 batch-get-account-status \
  --account-ids "123456789012" \
  --region us-east-1
```

**Example Output:**
```json
{
  "accounts": [
    {
      "accountId": "123456789012",
      "resourceStatus": {
        "ec2": "ENABLED",
        "ecr": "ENABLED",
        "lambda": "DISABLED",
        "lambdaCode": "DISABLED"
      },
      "state": {
        "errorCode": "NO_ERROR",
        "errorMessage": "",
        "status": "ENABLED"
      }
    }
  ],
  "failedAccounts": []
}
```

---

### 7. Create Filter

Creates a reusable filter for findings.

```bash
aws inspector2 create-filter \
  --name "critical-ec2-findings" \
  --description "Filter for critical EC2 vulnerabilities in production" \
  --action SUPPRESS \
  --filter-criteria '{
    "resourceType": [{"comparison": "EQUALS", "value": "AWS_EC2_INSTANCE"}],
    "severity": [{"comparison": "EQUALS", "value": "CRITICAL"}],
    "resourceTags": [{"comparison": "EQUALS", "key": "Environment", "value": "production"}]
  }' \
  --tags '{"Team": "security", "Purpose": "critical-filter"}' \
  --region us-east-1
```

**Example Output:**
```json
{
  "arn": "arn:aws:inspector2:us-east-1:123456789012:filter/abc123"
}
```

---

### 8. List Filters

Lists all saved Inspector v2 filters.

```bash
aws inspector2 list-filters \
  --action SUPPRESS \
  --region us-east-1
```

---

### 9. Get Findings Statistics

Retrieves aggregated statistics about findings.

```bash
aws inspector2 get-findings-report-status \
  --report-id "rpt-0abc123def456789" \
  --region us-east-1
```

```bash
# Get aggregated finding counts by severity
aws inspector2 list-finding-aggregations \
  --aggregation-type FINDING_TYPE \
  --aggregation-request '{
    "findingTypeAggregation": {
      "findingType": "PACKAGE_VULNERABILITY",
      "sortBy": "CRITICAL",
      "sortOrder": "DESC"
    }
  }' \
  --region us-east-1
```

---

### 10. Create Findings Report

Generates a findings report and exports it to an S3 bucket.

```bash
aws inspector2 create-findings-report \
  --report-format CSV \
  --s3-destination '{
    "bucketName": "my-inspector-reports-bucket",
    "keyPrefix": "inspector-reports/2024/01/",
    "kmsKeyArn": "arn:aws:kms:us-east-1:123456789012:key/my-kms-key-id"
  }' \
  --filter-criteria '{
    "findingStatus": [{"comparison": "EQUALS", "value": "ACTIVE"}]
  }' \
  --region us-east-1
```

**Example Output:**
```json
{
  "reportId": "rpt-0abc123def456789"
}
```

---

### 11. Associate Member Account (Delegated Admin)

Associates a member account with the Inspector v2 delegated administrator.

```bash
aws inspector2 associate-member \
  --account-id "987654321098" \
  --region us-east-1
```

---

### 12. List Members

Lists all member accounts associated with the delegated administrator.

```bash
aws inspector2 list-members \
  --only-associated true \
  --region us-east-1
```

**Example Output:**
```json
{
  "members": [
    {
      "accountId": "987654321098",
      "delegatedAdminAccountId": "123456789012",
      "relationshipStatus": "ENABLED",
      "updatedAt": "2024-01-10T08:00:00Z"
    }
  ]
}
```

---

### 13. Disable Inspector v2

Disables Inspector v2 for specified resource types.

```bash
aws inspector2 disable \
  --resource-types LAMBDA LAMBDA_CODE \
  --region us-east-1
```

---

### 14. Update Organization Configuration

Configures auto-enable settings for new accounts in an AWS Organization.

```bash
aws inspector2 update-organization-configuration \
  --auto-enable '{
    "ec2": true,
    "ecr": true,
    "lambda": true,
    "lambdaCode": false
  }' \
  --region us-east-1
```

---

### 15. Tag a Resource

Adds tags to an Inspector v2 resource (e.g., a filter).

```bash
aws inspector2 tag-resource \
  --resource-arn "arn:aws:inspector2:us-east-1:123456789012:filter/abc123" \
  --tags '{"Owner": "security-team", "CostCenter": "12345"}' \
  --region us-east-1
```

---

## Common Operations

### Create Operations

```bash
# Create a findings suppression filter
aws inspector2 create-filter \
  --name "suppress-dev-low-findings" \
  --action SUPPRESS \
  --filter-criteria '{
    "severity": [{"comparison": "EQUALS", "value": "LOW"}],
    "resourceTags": [{"comparison": "EQUALS", "key": "Environment", "value": "development"}]
  }' \
  --region us-east-1

# Create a CSV findings report
aws inspector2 create-findings-report \
  --report-format CSV \
  --s3-destination '{
    "bucketName": "my-security-reports",
    "keyPrefix": "inspector/monthly/",
    "kmsKeyArn": "arn:aws:kms:us-east-1:123456789012:key/my-kms-key-id"
  }' \
  --region us-east-1

# Create a CIS scan configuration
aws inspector2 create-cis-scan-configuration \
  --scan-name "my-cis-benchmark-scan" \
  --schedule '{"daily": {"startTime": {"timeOfDay": "02:00", "timezone": "UTC"}}}' \
  --security-level LEVEL_1 \
  --targets '{
    "accountIds": ["123456789012"],
    "targetResourceTags": {"Environment": ["production"]}
  }' \
  --region us-east-1
```

---

### Read / Describe Operations

```bash
# Describe a specific finding
aws inspector2 get-finding-v2 \
  --finding-arn "arn:aws:inspector2:us-east-1:123456789012:finding/abc123def456" \
  --region us-east-1

# Get account status
aws inspector2 batch-get-account-status \
  --account-ids "123456789012" "987654321098" \
  --region us-east-1

# Get free trial info
aws inspector2 get-free-trial-info \
  --account-ids "123456789012" \
  --region us-east-1

# Describe a CIS scan
aws inspector2 get-cis-scan-report \
  --scan-arn "arn:aws:inspector2:us-east-1:123456789012:cis-scan/scan-abc123" \
  --report-format PDF \
  --target-accounts "123456789012" \
  --s3-destination '{
    "bucketName": "my-cis-reports",
    "kmsKeyArn": "arn:aws:kms:us-east-1:123456789012:key/my-kms-key-id"
  }' \
  --region us-east-1
```

---

### Update Operations

```bash
# Update an existing filter
aws inspector2 update-filter \
  --filter-arn "arn:aws:inspector2:us-east-1:123456789012:filter/abc123" \
  --name "updated-filter-name" \
  --description "Updated description for production critical findings" \
  --action NONE \
  --region us-east-1

# Update organization auto-enable settings
aws inspector2 update-organization-configuration \
  --auto-enable '{"ec2": true, "ecr": true, "lambda": true, "lambdaCode": true}' \
  --region us-east-1

# Update encryption key for Inspector
aws inspector2 update-encryption-key \
  --resource-type FINDING \
  --scan-type NETWORK \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/my-kms-key-id" \
  --region us-east-1
```

---

### Delete Operations

```bash
# Delete a filter
aws inspector2 delete-filter \
  --arn "arn:aws:inspector2:us-east-1:123456789012:filter/abc123" \
  --region us-east-1

# Disassociate a member account
aws inspector2 disassociate-member \
  --account-id "987654321098" \
  --region us-east-1

# Delete a CIS scan configuration
aws inspector2 delete-cis-scan-configuration \
  --scan-configuration-arn "arn:aws:inspector2:us-east-1:123456789012:cis-scan-configuration/cfg-abc123" \
  --region us-east-1
```

---

### List Operations

```bash
# List all active