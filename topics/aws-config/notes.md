# AWS Config

## What is it?

**AWS Config** is a fully managed AWS service that provides continuous monitoring, assessment, and auditing of your AWS resource configurations. It belongs to the **Management & Governance** category of AWS services.

AWS Config enables you to:
- **Record** configuration changes to AWS resources over time
- **Evaluate** resource configurations against desired policies using Config Rules
- **Audit** compliance and security posture across your AWS environment
- **Notify** teams when configurations drift from desired states

> **Official Definition:** AWS Config is a service that enables you to assess, audit, and evaluate the configurations of your AWS resources. Config continuously monitors and records your AWS resource configurations and allows you to automate the evaluation of recorded configurations against desired configurations.

---

## Why do we need it?

### The Problem It Solves

In dynamic cloud environments, resources are created, modified, and deleted constantly — often by multiple teams, automation scripts, or third-party tools. Without visibility into these changes:

- **Security vulnerabilities** go undetected (e.g., an S3 bucket accidentally made public)
- **Compliance violations** accumulate silently (e.g., unencrypted EBS volumes)
- **Incident root cause analysis** becomes extremely difficult
- **Audit requirements** (SOC 2, PCI-DSS, HIPAA, ISO 27001) cannot be met

### When to Use It

| Scenario | Why AWS Config Helps |
|---|---|
| Security auditing | Detect when security groups open port 22 to 0.0.0.0/0 |
| Compliance enforcement | Ensure all S3 buckets have server-side encryption enabled |
| Change management | Track who changed what and when across all resources |
| Incident investigation | Replay historical configurations to identify root cause |
| Cost governance | Identify unused or oversized resources over time |
| Multi-account governance | Aggregate compliance data across an AWS Organization |

### Real Business Scenarios

1. **Financial Services Firm:** A bank needs to prove to auditors that no production database was ever publicly accessible. AWS Config provides a historical record of every RDS instance configuration.

2. **Healthcare Provider:** A hospital must comply with HIPAA. AWS Config rules automatically flag any EC2 instance running without encrypted storage.

3. **E-commerce Platform:** After a security incident, the DevOps team needs to determine exactly when a security group was modified. AWS Config's configuration timeline provides a precise audit trail.

4. **Enterprise IT Governance:** A large enterprise with 50+ AWS accounts needs a centralized compliance dashboard. AWS Config Aggregator collects data from all accounts into a single view.

---

## Internal Working

### How AWS Config Works Internally

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Config Engine                        │
│                                                                 │
│  1. Discovery   →   2. Recording   →   3. Evaluation            │
│  (Resource         (Config Items)      (Rules Engine)           │
│   Detection)                                                    │
└─────────────────────────────────────────────────────────────────┘
```

#### Step 1: Resource Discovery
- When AWS Config is enabled, it begins by **discovering all existing resources** in the account/region
- It uses AWS APIs to enumerate resources (EC2, S3, RDS, IAM, etc.)
- Creates a **baseline snapshot** of all resource configurations

#### Step 2: Configuration Recording
- AWS Config acts as an **event listener** using AWS CloudTrail events and direct API polling
- When a resource changes, Config captures a **Configuration Item (CI)**
- Each CI is a point-in-time snapshot containing:
  - Resource metadata (ID, type, ARN, region)
  - Configuration attributes (all settings)
  - Relationships to other resources
  - AWS CloudTrail event IDs
  - Compliance status

#### Step 3: Storage
- Configuration Items are stored in an **Amazon S3 bucket** (the delivery channel)
- Config also maintains an internal **configuration history** database
- An **SNS topic** can be configured to receive real-time notifications

#### Step 4: Rule Evaluation
- **Config Rules** are evaluated when:
  - A configuration change occurs (change-triggered)
  - On a periodic schedule (periodic)
- Rules are evaluated by either:
  - **AWS Managed Rules**: Pre-built Lambda functions maintained by AWS
  - **Custom Rules**: Lambda functions or Guard policies you write
  - **AWS Config Conformance Packs**: Collections of rules

#### Step 5: Compliance Reporting
- Each resource is marked as `COMPLIANT`, `NON_COMPLIANT`, or `NOT_APPLICABLE`
- Results are visible in the Config console, CLI, and API
- Can trigger **Remediation Actions** via AWS Systems Manager Automation

### Configuration Item (CI) Deep Dive

```json
{
  "configurationItemVersion": "1.3",
  "configurationItemCaptureTime": "2024-01-15T10:30:00Z",
  "configurationStateId": 1705312200000,
  "awsAccountId": "123456789012",
  "configurationItemStatus": "OK",
  "resourceType": "AWS::EC2::SecurityGroup",
  "resourceId": "sg-0abc123def456",
  "resourceName": "production-web-sg",
  "ARN": "arn:aws:ec2:us-east-1:123456789012:security-group/sg-0abc123def456",
  "awsRegion": "us-east-1",
  "tags": { "Environment": "production" },
  "configuration": {
    "groupName": "production-web-sg",
    "description": "Web tier security group",
    "ipPermissions": [...]
  },
  "relationships": [
    {
      "resourceType": "AWS::EC2::VPC",
      "resourceId": "vpc-0xyz789",
      "relationshipName": "Is contained in Vpc"
    }
  ]
}
```

---

## Architecture

### Core Components

```
┌──────────────────────────────────────────────────────────────────────┐
│                          AWS Account / Region                        │
│                                                                      │
│  ┌─────────────┐    Changes     ┌──────────────────────────────────┐ │
│  │ AWS Resources│──────────────▶│        AWS Config Recorder        │ │
│  │ (EC2, S3,   │                │  (Configuration Recorder)        │ │
│  │  RDS, IAM…) │                └──────────────┬───────────────────┘ │
│  └─────────────┘                               │                     │
│                                                │ Config Items         │
│                                                ▼                     │
│                              ┌─────────────────────────────────┐    │
│                              │     Config Rules Engine          │    │
│                              │  ┌──────────┐  ┌─────────────┐  │    │
│                              │  │ Managed  │  │   Custom    │  │    │
│                              │  │  Rules   │  │   Rules     │  │    │
│                              │  └──────────┘  └─────────────┘  │    │
│                              └────────────┬────────────────────┘    │
│                                           │                          │
│              ┌────────────────────────────┼───────────────────┐     │
│              │                            │                   │     │
│              ▼                            ▼                   ▼     │
│  ┌──────────────────┐      ┌──────────────────┐  ┌──────────────┐  │
│  │   Amazon S3       │      │   Amazon SNS     │  │  Systems     │  │
│  │  (Config History  │      │  (Notifications) │  │  Manager     │  │
│  │   & Snapshots)    │      │                  │  │ (Remediation)│  │
│  └──────────────────┘      └──────────────────┘  └──────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### Multi-Account / Multi-Region Architecture (Aggregator)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS Organizations                                │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  Account A   │  │  Account B   │  │  Account C   │             │
│  │  (Dev)       │  │  (Staging)   │  │  (Prod)      │             │
│  │  AWS Config  │  │  AWS Config  │  │  AWS Config  │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                  │                     │
│         └─────────────────┼──────────────────┘                     │
│                           │ Authorization                           │
│                           ▼                                         │
│              ┌────────────────────────┐                            │
│              │  Config Aggregator     │                            │
│              │  (Central Account)     │                            │
│              │  - Aggregated View     │                            │
│              │  - Cross-account       │                            │
│              │    Compliance Data     │                            │
│              └────────────────────────┘                            │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

| Component | Description |
|---|---|
| **Configuration Recorder** | The engine that detects and records resource changes |
| **Delivery Channel** | Specifies where to deliver config history (S3) and notifications (SNS) |
| **Config Rules** | Evaluation logic (managed or custom) |
| **Conformance Packs** | A collection of rules and remediation actions packaged together |
| **Aggregator** | Collects config data from multiple accounts/regions |
| **Remediation** | Automated or manual fix actions via SSM Automation |
| **Query** | SQL-like querying of current resource configurations |

---

## Real World Example

### Scenario: Ensuring PCI-DSS Compliance for a Payment Processing Application

**Company:** FinPay Inc., a payment processor handling credit card transactions.

**Requirement:** PCI-DSS mandates that all data-at-rest must be encrypted, security groups must not expose unnecessary ports, and all changes must be auditable.

#### Step-by-Step Walkthrough

**Step 1: Enable AWS Config**
```bash
# Enable Config recorder for all resource types
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::123456789012:role/ConfigRole \
  --recording-group allSupported=true,includeGlobalResourceTypes=true
```

**Step 2: Configure Delivery Channel**
```bash
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "finpay-config-bucket",
    "snsTopicARN": "arn:aws:sns:us-east-1:123456789012:ConfigAlerts",
    "configSnapshotDeliveryProperties": {
      "deliveryFrequency": "TwentyFour_Hours"
    }
  }'
```

**Step 3: Deploy PCI-DSS Conformance Pack**
```bash
aws configservice put-conformance-pack \
  --conformance-pack-name "PCI-DSS-Compliance" \
  --template-s3-uri "s3://finpay-config-bucket/pci-dss-conformance-pack.yaml" \
  --delivery-s3-bucket "finpay-config-results"
```

**Step 4: Create Custom Rule — Check RDS Encryption**
```yaml
# CloudFormation for custom Config rule
ConfigRuleRDSEncryption:
  Type: AWS::Config::ConfigRule
  Properties:
    ConfigRuleName: rds-storage-encrypted
    Source:
      Owner: AWS
      SourceIdentifier: RDS_STORAGE_ENCRYPTED
    Scope:
      ComplianceResourceTypes:
        - AWS::RDS::DBInstance
```

**Step 5: Set Up Auto-Remediation**
```yaml
# Automatically enable encryption on non-compliant S3 buckets
RemediationConfiguration:
  Type: AWS::Config::RemediationConfiguration
  Properties:
    ConfigRuleName: s3-bucket-server-side-encryption-enabled
    TargetType: SSM_DOCUMENT
    TargetId: AWS-EnableS3BucketEncryption
    Automatic: true
    MaximumAutomaticAttempts: 3
    RetryAttemptSeconds: 60
    Parameters:
      AutomationAssumeRole:
        StaticValue:
          Values:
            - arn:aws:iam::123456789012:role/RemediationRole
      BucketName:
        ResourceValue:
          Value: RESOURCE_ID
```

**Step 6: Query Current Compliance State**
```sql
-- AWS Config Advanced Query
SELECT
  resourceId,
  resourceName,
  resourceType,
  configuration.storageEncrypted,
  tags
WHERE
  resourceType = 'AWS::RDS::DBInstance'
  AND configuration.storageEncrypted = false
```

**Step 7: View Compliance Dashboard**
- Navigate to AWS Config Console → Compliance → View all non-compliant resources
- Export compliance report for PCI auditors
- Review configuration timeline for any resource that was recently modified

**Outcome:**
- FinPay achieves continuous compliance monitoring
- Auditors receive automated compliance reports
- Security team is alerted within minutes of any violation
- Auto-remediation fixes common issues without human intervention

---

## Advantages

1. **Continuous Compliance Monitoring**
   - 24/7 automated evaluation against compliance rules
   - Near-real-time detection of violations (within minutes of a change)

2. **Complete Audit Trail**
   - Full historical record of every configuration change
   - Tracks who made changes (via CloudTrail integration)
   - Point-in-time configuration retrieval

3. **Broad Resource Coverage**
   - Supports 350+ AWS resource types
   - Covers global resources (IAM) and regional resources
   - Custom resources via AWS Config Custom Resource Types

4. **Pre-Built Compliance Frameworks**
   - 130+ AWS Managed Rules available out-of-the-box
   - Conformance Packs for PCI-DSS, HIPAA, CIS, NIST, SOC 2

5. **Automated Remediation**
   - Integration with AWS Systems Manager Automation
   - Supports both automatic and manual remediation workflows

6. **Multi-Account / Multi-Region Visibility**
   - Config Aggregator provides a single pane of glass
   - Works seamlessly with AWS Organizations

7. **Advanced Query Capability**
   - SQL-like querying of current resource configurations
   - Enables custom compliance and inventory reports

8. **Integration-Rich Ecosystem**
   - Native integrations with CloudTrail, SNS, S3, Lambda, SSM, Security Hub

9. **No Infrastructure to Manage**
   - Fully managed, serverless service
   - No agents to install on resources

10. **Relationship Mapping**
    - Understands relationships between resources
    - Helps with impact analysis (e.g., which EC2 instances use this security group)

---

## Limitations

### Service Limits

| Limit | Default Value | Notes |
|---|---|---|
| Config rules per account per region | 500 | Can request increase |
| Conformance packs per account per region | 50 | Hard limit |
| Rules per conformance pack | 130 | Hard limit |
| Aggregators per account | 5 | Can request increase |
| Accounts per aggregator | 10,000 | Hard limit |
| Retention period for config history | 7 years (2,557 days) | Configurable |
| Advanced Query result size | 1 MB | Hard limit |
| Remediation executions per rule per hour | 500 | Hard limit |

### Functional Limitations

1. **Not Real-Time for All Resources**
   - Some resource types have polling intervals, not event-driven detection
   - Delay can be several minutes for certain resource types

2. **Regional Service**
   - Must be enabled