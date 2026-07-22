# Secrets Manager

## What is it?

**AWS Secrets Manager** is a fully managed secrets management service that enables you to securely store, retrieve, rotate, audit, and control access to secrets throughout their lifecycle. It falls under the **Security, Identity, and Compliance** category of AWS services.

A "secret" in this context can be:
- Database credentials (username/password)
- API keys and tokens
- OAuth tokens
- SSH keys
- TLS/SSL certificates
- Any arbitrary text or binary data up to 64 KB

Secrets Manager goes beyond simple storage — it provides **automated secret rotation** using AWS Lambda, deep integration with AWS services, and fine-grained access control via IAM, making it a cornerstone of secrets lifecycle management in enterprise AWS environments.

---

## Why do we need it?

### The Problem

Traditionally, developers hardcode credentials directly into application code, configuration files, or environment variables. This creates serious security risks:

- **Credential exposure**: Secrets checked into version control (e.g., GitHub) are a leading cause of data breaches.
- **Manual rotation pain**: Rotating passwords manually across distributed applications is error-prone and often delayed.
- **Audit gaps**: No centralized visibility into who accessed which secret and when.
- **Sprawl**: Secrets duplicated across EC2 instances, Lambda functions, and containers become impossible to manage.

### When to Use It

| Scenario | Use Secrets Manager |
|---|---|
| Database credentials for RDS/Aurora | ✅ Yes — native rotation support |
| Third-party API keys (Stripe, Twilio) | ✅ Yes — custom rotation Lambda |
| Application configuration (non-sensitive) | ❌ Use SSM Parameter Store |
| TLS certificates with renewal | ✅ Yes — with ACM integration |
| Secrets requiring cross-account access | ✅ Yes — resource-based policies |
| High-volume secret reads (millions/day) | ⚠️ Cache with SDK or use SSM Standard |

### Real Business Scenarios

1. **E-commerce Platform**: A retail company stores RDS Aurora MySQL credentials in Secrets Manager. Passwords rotate every 30 days automatically — no developer intervention, no downtime.
2. **FinTech Application**: A payments processor stores Stripe API keys and rotates them quarterly, with full CloudTrail audit logs for compliance (PCI-DSS).
3. **Multi-tenant SaaS**: Each customer gets isolated database credentials stored as separate secrets, accessed by Lambda functions using IAM roles.

---

## Internal Working

### Secret Storage

Secrets Manager stores secrets as **encrypted JSON key-value pairs** (or plain text) in a highly available, durable backend. Internally:

1. When you create a secret, Secrets Manager calls **AWS KMS** to generate a unique **Data Encryption Key (DEK)** using envelope encryption.
2. The secret value is encrypted with the DEK using **AES-256-GCM**.
3. The encrypted DEK is stored alongside the encrypted secret value.
4. The plaintext DEK is never persisted — it's generated on-demand during decryption.

### Secret Versioning

Every secret maintains a **version staging system**:

```
Secret: prod/myapp/db-password
├── Version A (AWSCURRENT)  → current active secret
├── Version B (AWSPREVIOUS) → previous version (kept for rollback)
└── Version C (AWSPENDING)  → new version being rotated in
```

- **AWSCURRENT**: The version your application should use.
- **AWSPREVIOUS**: Retained for rollback scenarios.
- **AWSPENDING**: Created during rotation before cutover.

Versions are identified by a **UUID** and can have multiple staging labels simultaneously.

### Rotation Mechanism

The rotation process uses an **AWS Lambda function** that executes a **4-step rotation process**:

```
Step 1: createSecret   → Lambda creates new credentials in AWSPENDING
Step 2: setSecret      → Lambda sets the new credentials on the target service
Step 3: testSecret     → Lambda verifies the new credentials work
Step 4: finishSecret   → Secrets Manager promotes AWSPENDING → AWSCURRENT
```

If any step fails, the rotation is aborted, and the original `AWSCURRENT` secret remains valid.

### Retrieval Flow

```
Application → GetSecretValue API → Secrets Manager → KMS Decrypt → Plaintext Secret → Application
```

The SDK caches the secret in memory (with TTL) to reduce API calls and latency.

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                     AWS Secrets Manager                      │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │   Secret     │   │   Version    │   │  Rotation      │  │
│  │   Metadata   │──▶│   Store      │   │  Configuration │  │
│  │  (Name, ARN, │   │ (Encrypted   │   │ (Schedule,     │  │
│  │   Tags, etc) │   │  Values)     │   │  Lambda ARN)   │  │
│  └──────────────┘   └──────┬───────┘   └────────────────┘  │
│                             │                               │
└─────────────────────────────┼───────────────────────────────┘
                              │ Envelope Encryption
                    ┌─────────▼─────────┐
                    │     AWS KMS       │
                    │  (Customer CMK    │
                    │   or AWS Managed) │
                    └───────────────────┘
```

### Rotation Architecture

```
┌──────────────┐     Schedule/Manual     ┌──────────────────┐
│  Secrets     │────────────────────────▶│  EventBridge     │
│  Manager     │                         │  (cron trigger)  │
└──────┬───────┘                         └────────┬─────────┘
       │                                          │
       │ Invoke                                   │
       ▼                                          ▼
┌──────────────┐  createSecret/setSecret  ┌──────────────────┐
│    Lambda    │─────────────────────────▶│  Target Service  │
│  (Rotation   │                          │  (RDS, Redshift, │
│   Function)  │◀─────────────────────────│   Custom API)    │
└──────────────┘  testSecret/finishSecret └──────────────────┘
```

### Cross-Account Access Architecture

```
Account A (Secret Owner)          Account B (Consumer)
┌─────────────────────┐           ┌─────────────────────┐
│  Secrets Manager    │           │  Lambda / EC2       │
│  ┌───────────────┐  │           │  ┌───────────────┐  │
│  │ Resource-based│  │           │  │  IAM Role     │  │
│  │ Policy allows │◀─┼───────────┼──│  (Account B)  │  │
│  │ Account B     │  │           │  └───────────────┘  │
│  └───────────────┘  │           └─────────────────────┘
│  ┌───────────────┐  │
│  │  KMS Key      │  │
│  │  Policy allows│  │
│  │  Account B    │  │
│  └───────────────┘  │
└─────────────────────┘
```

### Key Architectural Patterns

1. **Sidecar Pattern**: A sidecar container fetches and refreshes secrets, exposing them to the main application container via a local socket.
2. **SDK Caching Pattern**: Use the AWS Secrets Manager Caching Client to reduce API calls and costs.
3. **Secret Replication Pattern**: Replicate secrets to multiple regions for multi-region active-active architectures.

---

## Real World Example

### Scenario: Automated RDS Credential Rotation for a Multi-Tier Web Application

**Context**: A company runs a 3-tier web app (React → Node.js API → RDS PostgreSQL). Security team mandates database password rotation every 30 days with zero downtime.

#### Step 1: Create the Secret

```bash
aws secretsmanager create-secret \
  --name "prod/webapp/rds-postgres" \
  --description "Production PostgreSQL credentials" \
  --secret-string '{"username":"webapp_user","password":"InitialP@ssw0rd","engine":"postgres","host":"prod-db.cluster-xyz.us-east-1.rds.amazonaws.com","port":5432,"dbname":"webapp_prod"}'
```

#### Step 2: Attach Resource Policy (for cross-account access)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_B_ID:role/webapp-api-role"
      },
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "*"
    }
  ]
}
```

#### Step 3: Enable Automatic Rotation

```bash
aws secretsmanager rotate-secret \
  --secret-id "prod/webapp/rds-postgres" \
  --rotation-lambda-arn "arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRDSPostgreSQLRotationSingleUser" \
  --rotation-rules AutomaticallyAfterDays=30
```

AWS provides **pre-built rotation Lambda functions** for:
- Amazon RDS (MySQL, PostgreSQL, Oracle, MSSQL)
- Amazon Redshift
- Amazon DocumentDB
- Amazon ElastiCache

#### Step 4: Application Retrieves Secret

```javascript
// Node.js API server startup
const secret = await getSecret("prod/webapp/rds-postgres");
const { username, password, host, port, dbname } = JSON.parse(secret);
const pool = new Pool({ user: username, password, host, port, database: dbname });
```

#### Step 5: Rotation Happens Automatically

```
Day 0:  Password = "InitialP@ssw0rd" (AWSCURRENT)
Day 30: Lambda creates new password (AWSPENDING)
        Lambda sets new password on RDS
        Lambda tests new password works
        Secrets Manager promotes AWSPENDING → AWSCURRENT
        Old password moves to AWSPREVIOUS
Day 60: Next rotation cycle begins
```

**Result**: Zero downtime, zero developer intervention, full audit trail in CloudTrail.

---

## Advantages

| Advantage | Detail |
|---|---|
| **Automated Rotation** | Built-in rotation for RDS, Redshift, DocumentDB; custom Lambda for anything else |
| **Envelope Encryption** | Every secret encrypted with KMS; supports CMK for BYOK |
| **Fine-Grained Access** | IAM identity policies + resource-based policies + VPC endpoint policies |
| **Versioning** | Automatic version history; rollback capability via staging labels |
| **Audit Trail** | Every API call logged in CloudTrail with caller identity, timestamp, IP |
| **Multi-Region Replication** | Replicate secrets to up to 100 AWS regions |
| **Native AWS Integration** | Deep integration with RDS, ECS, EKS, Lambda, CodeBuild, CloudFormation |
| **High Availability** | Regionally redundant; SLA-backed availability |
| **Compliance** | Supports PCI-DSS, HIPAA, SOC, ISO, FedRAMP |
| **SDK Caching** | Official caching clients reduce latency and API costs |

---

## Limitations

### Hard Limits (Default Quotas)

| Limit | Value |
|---|---|
| Maximum secret size | 64 KB |
| Secrets per region per account | 500,000 |
| Resource-based policy size | 20 KB |
| Rotation Lambda timeout | 30 seconds per step |
| API rate limit: `GetSecretValue` | 10,000 RPS (can be increased) |
| API rate limit: `PutSecretValue` | 50 RPS |
| API rate limit: `RotateSecret` | 50 RPS |
| Maximum replication regions | 100 |
| Secret name length | 512 characters |
| Tag limit per secret | 50 tags |

### Functional Limitations

- **No native secret sharing UI**: Cross-account access requires manual policy configuration.
- **Rotation Lambda cold starts**: Can add latency to the rotation process.
- **Cost at scale**: At high read volumes (millions/day), costs accumulate — consider caching.
- **No built-in secret expiry enforcement**: You must implement expiry logic yourself.
- **Replication is read-only**: Replica secrets cannot be modified; changes must go to the primary.
- **No secret diffing**: Cannot compare versions natively.
- **Lambda VPC configuration**: Rotation Lambda must have network access to both Secrets Manager and the target service.

---

## Best Practices

### 1. Use a Consistent Naming Convention

```
{environment}/{application}/{secret-type}
prod/payment-service/stripe-api-key
staging/auth-service/jwt-secret
dev/analytics/redshift-credentials
```

### 2. Enable Automatic Rotation

Always enable rotation for long-lived credentials. Use the minimum rotation window appropriate for your compliance requirements (e.g., 30 days for PCI-DSS, 90 days for SOC 2).

### 3. Use SDK Caching

```javascript
// Use the caching client to avoid per-request API calls
import { SecretsManagerClient } from "@aws-sdk/client-secrets-manager";
// Cache TTL default: 1 hour; refresh before expiry
```

### 4. Least Privilege IAM

```json
{
  "Effect": "Allow",
  "Action": ["secretsmanager:GetSecretValue"],
  "Resource": "arn:aws:iam::123456789012:secret:prod/myapp/*",
  "Condition": {
    "StringEquals": {
      "secretsmanager:VersionStage": "AWSCURRENT"
    }
  }
}
```

### 5. Use Customer Managed Keys (CMK)

Always use a customer-managed KMS key rather than the AWS-managed default key for:
- Key rotation control
- Cross-account access
- Audit of key usage separately

### 6. Enable VPC Endpoints

Use **Interface VPC Endpoints** (AWS PrivateLink) for Secrets Manager to prevent traffic from traversing the public internet.

### 7. Tag Secrets for Cost Allocation

```json
{
  "Environment": "prod",
  "Application": "payment-service",
  "Owner": "platform-team",
  "CostCenter": "CC-1234"
}
```

### 8. Implement Secret Rotation Testing

Test rotation in non-production environments before enabling in production. Verify the rotation Lambda can reach both Secrets Manager and the target service.

### 9. Well-Architected Alignment

- **Security Pillar**: Protect secrets with KMS CMK, VPC endpoints, and least-privilege IAM.
- **Reliability Pillar**: Use multi-region replication for secrets used in multi-region architectures.
- **Operational Excellence**: Use CloudTrail + CloudWatch for monitoring and alerting on secret access patterns.

---

## Common Mistakes

### ❌ Mistake 1: Hardcoding Secret ARNs

```javascript
// BAD: Hardcoded ARN
const secretArn = "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db";

// GOOD: Use environment variables or parameter store for the ARN/name
const secretName = process.env.DB_SECRET_NAME;
```

### ❌ Mistake 2: Not Caching Secret Values

```javascript
// BAD: Fetching secret on every request (expensive + slow)
app.get("/data", async (req, res) => {
  const secret = await getSecret("prod/db"); // Called thousands of times/minute
});

// GOOD: Cache at application startup or use SDK caching client
```

### ❌ Mistake 3: Overly Broad IAM Permissions

```json
// BAD: Access to all secrets
{ "Action": "secretsmanager:*", "Resource": "*" }

// GOOD: Scoped to specific