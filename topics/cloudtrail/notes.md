# CloudTrail

## What is it?

**AWS CloudTrail** is a fully managed governance, compliance, and operational auditing service that records API calls and account activity across your AWS infrastructure. It belongs to the **Management & Governance** category of AWS services.

CloudTrail continuously monitors and records every action taken in your AWS environment — whether initiated by a user, role, AWS service, or application — and delivers these records as structured **event logs** to Amazon S3, CloudWatch Logs, or Amazon EventBridge. Each recorded entry is called a **CloudTrail event**, and it captures the *who*, *what*, *when*, and *where* of every AWS API interaction.

There are three types of events CloudTrail captures:

| Event Type | Description | Default Logging |
|---|---|---|
| **Management Events** | Control-plane operations (e.g., creating an S3 bucket, launching an EC2 instance) | ✅ Enabled by default |
| **Data Events** | Data-plane operations (e.g., S3 `GetObject`, Lambda `Invoke`) | ❌ Must be explicitly enabled |
| **Insights Events** | Anomalous API activity detection (unusual call volumes) | ❌ Must be explicitly enabled |

CloudTrail is **regional by default** but can be configured as an **organization trail** to cover all accounts in an AWS Organization.

---

## Why do we need it?

### The Problem It Solves

Without CloudTrail, AWS environments are essentially **black boxes** — you have no reliable way to answer questions like:
- Who deleted that production S3 bucket?
- Which IAM user changed the security group rules at 2 AM?
- Why did our Lambda function suddenly start failing after a deployment?
- Is someone exfiltrating data from our DynamoDB tables?

CloudTrail solves the **audit trail gap** in cloud environments where infrastructure changes happen at machine speed across distributed teams.

### Business Scenarios

**Scenario 1 — Compliance & Regulatory Requirements**
A financial services company must comply with PCI-DSS, SOC 2, and HIPAA. These standards require proof that all administrative actions on systems handling sensitive data are logged, retained, and tamper-proof. CloudTrail provides the immutable audit log required by auditors.

**Scenario 2 — Security Incident Response**
A DevOps team notices unusual charges on their AWS bill. Using CloudTrail, the security team traces the activity back to a compromised IAM access key that was used to spin up hundreds of EC2 instances for cryptocurrency mining.

**Scenario 3 — Operational Troubleshooting**
A production application breaks after a routine maintenance window. Engineers use CloudTrail to identify that a team member accidentally modified an IAM policy, removing permissions the application needed.

**Scenario 4 — Change Management Governance**
An enterprise with multiple development teams uses CloudTrail to enforce that all infrastructure changes go through their approved CI/CD pipeline, alerting on any out-of-band manual changes.

**Scenario 5 — Insider Threat Detection**
A healthcare organization monitors CloudTrail logs for unusual data access patterns — such as a single user downloading thousands of patient records — as part of their insider threat program.

---

## Internal Working

### Event Generation Pipeline

```
AWS API Call
     │
     ▼
AWS Service Endpoint (e.g., EC2, S3, IAM)
     │
     ▼
CloudTrail Service Layer (intercepts all API calls)
     │
     ▼
Event Validation & Enrichment
  ├── Source IP address
  ├── User identity (IAM ARN, role, federated user)
  ├── Request parameters
  ├── Response elements
  ├── AWS region
  └── Timestamp (UTC)
     │
     ▼
Event Buffering (approximately 15-minute delivery window)
     │
     ├──► S3 Bucket (compressed JSON log files)
     ├──► CloudWatch Logs (streaming, near-real-time)
     └──► EventBridge (near-real-time event routing)
```

### How CloudTrail Processes Events

1. **Interception**: Every AWS API call passes through the CloudTrail service layer, regardless of whether it's made via the AWS Console, CLI, SDK, or another AWS service.

2. **Identity Resolution**: CloudTrail resolves the caller's identity using the **STS (Security Token Service)** context, capturing the original identity even when assuming roles.

3. **Event Enrichment**: CloudTrail augments the raw API call with metadata including the source IP, user agent string, request ID, and event time.

4. **Batching & Compression**: Events are batched, compressed using **gzip**, and written as JSON log files to S3. Files are typically delivered within **15 minutes** of the API activity.

5. **Log File Validation**: CloudTrail generates a **digest file** every hour containing SHA-256 hashes of each log file delivered in the previous hour, plus the hash of the previous digest file — creating a cryptographic chain that proves log integrity.

6. **Encryption**: Log files can be encrypted using AWS KMS Customer Managed Keys (CMKs) before delivery to S3.

### Event Record Structure

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "AIDAXXXXXXXXXXXXXXXXX",
    "arn": "arn:aws:iam::123456789012:user/alice",
    "accountId": "123456789012",
    "userName": "alice"
  },
  "eventTime": "2024-01-15T14:23:45Z",
  "eventSource": "s3.amazonaws.com",
  "eventName": "DeleteBucket",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "203.0.113.42",
  "userAgent": "aws-cli/2.13.0",
  "requestParameters": {
    "bucketName": "my-production-bucket"
  },
  "responseElements": null,
  "requestID": "EXAMPLE123456789",
  "eventID": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "readOnly": false,
  "resources": [{
    "ARN": "arn:aws:s3:::my-production-bucket",
    "accountId": "123456789012",
    "type": "AWS::S3::Bucket"
  }],
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "123456789012"
}
```

### S3 Log File Path Structure

```
s3://my-trail-bucket/
  AWSLogs/
    {AccountId}/
      CloudTrail/
        {Region}/
          {Year}/
            {Month}/
              {Day}/
                {AccountId}_CloudTrail_{Region}_{Timestamp}_{UniqueString}.json.gz
```

---

## Architecture

### Single-Account Trail Architecture

```
┌─────────────────────────────────────────────────────┐
│                   AWS Account                        │
│                                                     │
│  ┌──────────┐   ┌──────────┐   ┌──────────────┐   │
│  │  AWS     │   │  AWS     │   │  AWS         │   │
│  │  Console │   │  CLI     │   │  SDK/Service │   │
│  └────┬─────┘   └────┬─────┘   └──────┬───────┘   │
│       │              │                │            │
│       └──────────────┴────────────────┘            │
│                      │                             │
│                      ▼                             │
│              ┌───────────────┐                     │
│              │  CloudTrail   │                     │
│              │  Trail        │                     │
│              └───────┬───────┘                     │
│                      │                             │
│         ┌────────────┼────────────┐                │
│         ▼            ▼            ▼                │
│    ┌─────────┐ ┌──────────┐ ┌──────────┐          │
│    │   S3    │ │CloudWatch│ │EventBridge│          │
│    │ Bucket  │ │  Logs    │ │           │          │
│    └────┬────┘ └─────┬────┘ └─────┬────┘          │
│         │            │            │                │
│    ┌────▼────┐  ┌────▼────┐  ┌───▼─────┐         │
│    │ Athena  │  │CloudWatch│  │ Lambda  │         │
│    │ Queries │  │ Alarms  │  │ Alerts  │         │
│    └─────────┘  └─────────┘  └─────────┘         │
└─────────────────────────────────────────────────────┘
```

### Multi-Account Organization Trail Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS Organizations                         │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Management Account                       │  │
│  │                                                      │  │
│  │  ┌─────────────────────────────────────────────┐    │  │
│  │  │         Organization Trail                   │    │  │
│  │  │  (Covers ALL accounts in the Organization)  │    │  │
│  │  └──────────────────┬──────────────────────────┘    │  │
│  │                     │                               │  │
│  │              ┌──────▼──────┐                        │  │
│  │              │  Central S3 │                        │  │
│  │              │   Bucket    │                        │  │
│  │              │  (Logging   │                        │  │
│  │              │   Account)  │                        │  │
│  │              └─────────────┘                        │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Dev Account  │  │ Prod Account │  │ Sec Account  │     │
│  │  (Events     │  │  (Events     │  │  (Events     │     │
│  │   forwarded) │  │   forwarded) │  │   forwarded) │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

| Component | Role |
|---|---|
| **Trail** | The configuration that defines what events to capture and where to deliver them |
| **Event Selectors** | Rules that filter which events (management, data, insights) are captured |
| **S3 Bucket** | Primary persistent storage for log files |
| **CloudWatch Logs Group** | Real-time log streaming for alerting and analysis |
| **KMS Key** | Optional encryption for log files at rest |
| **SNS Topic** | Optional notification when new log files are delivered |
| **Digest Files** | Hourly files for log integrity validation |
| **CloudTrail Lake** | Managed data lake for querying events using SQL |

### CloudTrail Lake Architecture

CloudTrail Lake is a newer feature that stores events in an **immutable, managed data store** optimized for SQL-based querying — eliminating the need to set up Athena + S3 + Glue separately.

```
Events → CloudTrail Lake Event Data Store → SQL Query Interface
                                          → Integration with AWS Glue
                                          → Amazon Athena Federation
```

---

## Real World Example

### Scenario: Security Incident Investigation at a FinTech Company

**Context**: A FinTech startup running on AWS receives a PagerDuty alert at 3 AM that an unusual number of IAM policy changes have occurred. The on-call security engineer must investigate.

#### Step 1: Initial Detection via CloudWatch Alarm

The team had pre-configured a CloudWatch Metric Filter on their CloudTrail → CloudWatch Logs integration:

```
Filter Pattern: { ($.eventName = "DeletePolicy") || 
                  ($.eventName = "AttachRolePolicy") || 
                  ($.eventName = "PutUserPolicy") }
```

This triggered an alarm when more than 5 such events occurred within 5 minutes.

#### Step 2: Query CloudTrail Lake for Context

The security engineer runs a SQL query in CloudTrail Lake:

```sql
SELECT
    eventTime,
    userIdentity.arn,
    eventName,
    sourceIPAddress,
    requestParameters,
    errorCode
FROM
    my-event-data-store
WHERE
    eventTime > '2024-01-15 02:00:00'
    AND eventTime < '2024-01-15 03:30:00'
    AND eventSource = 'iam.amazonaws.com'
ORDER BY
    eventTime DESC
LIMIT 100;
```

#### Step 3: Findings

Results show 47 IAM policy changes from a single IAM user `ci-deploy-bot` originating from IP `198.51.100.77` — an IP address not associated with the company's CI/CD infrastructure.

#### Step 4: Trace the Access Key

```sql
SELECT DISTINCT
    userIdentity.accessKeyId,
    userIdentity.arn,
    sourceIPAddress,
    COUNT(*) as event_count
FROM
    my-event-data-store
WHERE
    userIdentity.arn LIKE '%ci-deploy-bot%'
    AND eventTime > '2024-01-14 00:00:00'
GROUP BY
    userIdentity.accessKeyId,
    userIdentity.arn,
    sourceIPAddress;
```

This reveals the access key `AKIAXXXXXXXXXXXXXXXX` was used from two different IPs — the legitimate CI/CD server and the suspicious external IP.

#### Step 5: Remediation

1. **Immediately disable** the compromised access key via IAM
2. **Revoke all sessions** associated with the key
3. **Audit all changes** made during the compromise window
4. **Roll back** unauthorized IAM policy changes
5. **Enable CloudTrail Insights** to detect future anomalies automatically

#### Step 6: Post-Incident — Enable CloudTrail Insights

```bash
aws cloudtrail put-insight-selectors \
    --trail-name production-trail \
    --insight-selectors '[{"InsightType": "ApiCallRateInsight"}, 
                          {"InsightType": "ApiErrorRateInsight"}]'
```

#### Step 7: Prevent Recurrence

Add an EventBridge rule to automatically trigger a Lambda function that disables access keys when anomalous IAM activity is detected:

```
CloudTrail → EventBridge Rule (IAM events from unknown IPs) 
           → Lambda (disable key + notify Security team via SNS)
```

---

## Advantages

1. **Comprehensive Coverage**: Automatically records virtually every AWS API call across 200+ services without requiring any agent installation or application code changes.

2. **Immutable Audit Trail**: Log file integrity validation using SHA-256 cryptographic hashing ensures logs cannot be tampered with without detection.

3. **Organization-Wide Visibility**: A single organization trail can capture activity across all accounts in an AWS Organization, providing a unified audit view.

4. **Near-Real-Time Alerting**: Integration with CloudWatch Logs and EventBridge enables security teams to respond to threats within seconds of detection.

5. **Long-Term Retention**: Logs stored in S3 can be retained indefinitely with configurable lifecycle policies, meeting even the most stringent regulatory retention requirements (e.g., 7-year retention for financial records).

6. **Serverless & Fully Managed**: No infrastructure to provision, patch, or maintain. AWS handles all scaling, availability, and durability concerns.

7. **CloudTrail Lake**: Purpose-built querying capability eliminates the operational overhead of building an Athena + Glue + S3 query infrastructure.

8. **Multi-Region Support**: A single trail can be configured to capture events from all AWS regions, preventing blind spots.

9. **Integration Breadth**: Deep integration with CloudWatch, EventBridge, Athena, Security Hub, GuardDuty, and many more services.