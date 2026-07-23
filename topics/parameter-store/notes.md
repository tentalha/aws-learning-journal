# Parameter Store

## What is it?

**AWS Systems Manager Parameter Store** is a fully managed, secure, hierarchical storage service for configuration data and secrets management. It is a capability of **AWS Systems Manager (SSM)** and falls under the **Management & Governance** category of AWS services.

Parameter Store provides a centralized repository to store configuration data such as:
- Plaintext strings (database URLs, feature flags, application settings)
- Encrypted secrets (passwords, API keys, license codes)
- Structured data (JSON blobs representing complex configurations)

Parameters can be stored as three types:
| Type | Description |
|------|-------------|
| `String` | Any plaintext value |
| `StringList` | Comma-separated list of values |
| `SecureString` | Encrypted value using AWS KMS |

Parameter Store supports two **tiers**:
- **Standard Tier**: Free, up to 10,000 parameters, 4 KB max size
- **Advanced Tier**: Paid, up to 100,000 parameters, 8 KB max size, supports parameter policies (TTL, expiration notifications)

---

## Why do we need it?

### The Problem

In traditional application development, configuration values and secrets are often:
- Hardcoded directly into source code (catastrophic for security)
- Stored in environment variables that are not centrally managed
- Committed to version control repositories (GitHub leaks happen daily)
- Duplicated across multiple environments without consistency
- Rotated manually, leading to stale credentials and security vulnerabilities

### Business Scenarios

**Scenario 1 — Multi-Environment Configuration Management**
A fintech company runs their application across `dev`, `staging`, and `production` environments. Each environment has different database connection strings, API endpoints, and feature flags. Without Parameter Store, developers maintain separate `.env` files per environment, leading to drift and human error.

**Scenario 2 — Secret Management for Microservices**
An e-commerce platform has 30+ microservices, each needing access to third-party API keys (Stripe, SendGrid, Twilio). Storing these in Parameter Store with KMS encryption ensures secrets are never exposed in code or logs.

**Scenario 3 — CI/CD Pipeline Configuration**
A DevOps team needs to inject environment-specific values during deployment pipelines (CodePipeline, Jenkins, GitHub Actions) without exposing sensitive data in pipeline definitions.

**Scenario 4 — Compliance and Auditing**
A healthcare company (HIPAA-compliant) must demonstrate that all access to sensitive configuration data is logged, audited, and encrypted. Parameter Store + CloudTrail provides this out of the box.

**When to use Parameter Store vs. Secrets Manager:**
- Use **Parameter Store** for general configuration + lightweight secrets (cost-sensitive, no automatic rotation needed)
- Use **Secrets Manager** when you need built-in automatic rotation, cross-account sharing, or are managing database credentials with native rotation support

---

## Internal Working

### Parameter Storage and Versioning

```
Client Request → SSM API Endpoint → Authorization (IAM + KMS) → Parameter Store Backend → DynamoDB-like Storage
```

1. **API Layer**: All interactions go through the SSM API (`ssm.amazonaws.com`). AWS handles endpoint routing, TLS termination, and request validation.

2. **Authorization Layer**:
   - IAM evaluates whether the caller has permission (`ssm:GetParameter`, `ssm:PutParameter`, etc.)
   - For `SecureString` parameters, KMS authorization is evaluated separately — the caller must also have `kms:Decrypt` permission on the KMS key

3. **Storage Layer**: AWS internally manages a highly durable, replicated storage backend. Parameters are stored with full version history. Every `PutParameter` call creates a new version (versions are integers starting at 1). AWS retains all historical versions.

4. **Encryption Flow for SecureString**:
   ```
   PutParameter (SecureString)
   ├── SSM receives plaintext value
   ├── SSM calls KMS GenerateDataKey
   ├── KMS returns plaintext data key + encrypted data key
   ├── SSM encrypts value using plaintext data key (AES-256-GCM)
   ├── SSM stores: encrypted value + encrypted data key
   └── Plaintext data key is discarded from memory
   
   GetParameter (SecureString)
   ├── SSM retrieves encrypted value + encrypted data key
   ├── SSM calls KMS Decrypt with encrypted data key
   ├── KMS returns plaintext data key (after IAM/KMS policy check)
   ├── SSM decrypts value using plaintext data key
   └── Returns plaintext value to caller (over TLS)
   ```

5. **Hierarchy and Path Resolution**: Parameters are stored with a `/`-delimited path structure. The hierarchy is logical (not physical), but enables powerful IAM path-based permissions and bulk retrieval via `GetParametersByPath`.

6. **Versioning**: Each parameter maintains a version counter. The latest version is always returned by default. You can reference a specific version using `name:version` syntax (e.g., `/myapp/db/password:3`).

7. **Labels**: You can attach human-readable labels (e.g., `live`, `beta`, `v2`) to specific versions, enabling blue/green configuration management.

---

## Architecture

### Hierarchical Naming Structure

```
/
├── /myapp/
│   ├── /myapp/dev/
│   │   ├── /myapp/dev/db/host        → "dev-db.cluster.us-east-1.rds.amazonaws.com"
│   │   ├── /myapp/dev/db/port        → "5432"
│   │   ├── /myapp/dev/db/password    → [SecureString, KMS encrypted]
│   │   └── /myapp/dev/feature-flags  → '{"darkMode": true, "betaCheckout": false}'
│   ├── /myapp/staging/
│   │   └── ...
│   └── /myapp/prod/
│       └── ...
└── /shared/
    ├── /shared/ssl-certificate        → [SecureString]
    └── /shared/smtp-credentials       → [SecureString]
```

### Key Architectural Components

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS Account                               │
│                                                             │
│  ┌──────────┐    IAM     ┌─────────────────────────────┐   │
│  │  Lambda  │ ─────────► │   SSM Parameter Store API   │   │
│  │  EC2     │            │   (ssm.amazonaws.com)        │   │
│  │  ECS     │            └──────────────┬──────────────┘   │
│  │  EKS     │                           │                   │
│  └──────────┘                    ┌──────▼──────┐           │
│                                  │   Storage   │           │
│  ┌──────────┐    KMS Decrypt     │   Backend   │           │
│  │  AWS KMS │ ◄──────────────────│  (Versioned)│           │
│  │  (CMK)   │                    └─────────────┘           │
│  └──────────┘                                               │
│                                                             │
│  ┌──────────────┐   ┌─────────────┐   ┌────────────────┐   │
│  │ CloudTrail   │   │  CloudWatch │   │  EventBridge   │   │
│  │ (Audit Logs) │   │  (Metrics)  │   │  (Expiry Alerts│   │
│  └──────────────┘   └─────────────┘   └────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Cross-Account Architecture

```
Account A (Shared Services)           Account B (Application)
┌─────────────────────────┐          ┌──────────────────────┐
│  Parameter Store        │          │  Lambda Function     │
│  /shared/api-key        │          │                      │
│  [SecureString, CMK-A]  │          │  Assume Role →       │
│                         │◄─────────│  Cross-Account Role  │
│  KMS CMK (Key Policy    │          │  (ssm:GetParameter   │
│  allows Account B)      │          │   + kms:Decrypt)     │
└─────────────────────────┘          └──────────────────────┘
```

### Parameter Policy Architecture (Advanced Tier)

```
Parameter: /myapp/prod/api-key (Advanced Tier)
├── Expiration Policy    → Delete parameter after 90 days
├── ExpirationNotification → SNS alert 7 days before expiry
└── NoChangeNotification   → Alert if not rotated in 30 days
```

---

## Real World Example

### Scenario: Node.js Microservice with Database Credentials

**Context**: A SaaS company runs a Node.js order-processing microservice on ECS Fargate. The service needs database credentials, a third-party payment API key, and feature flags.

#### Step 1: Store Parameters

```bash
# Store database host (plaintext)
aws ssm put-parameter \
  --name "/orderservice/prod/db/host" \
  --value "orders-db.cluster-abc123.us-east-1.rds.amazonaws.com" \
  --type "String" \
  --tier "Standard"

# Store database password (encrypted with AWS-managed key)
aws ssm put-parameter \
  --name "/orderservice/prod/db/password" \
  --value "MyS3cur3P@ssword!" \
  --type "SecureString" \
  --key-id "alias/orderservice-key" \
  --tier "Standard"

# Store feature flags as JSON
aws ssm put-parameter \
  --name "/orderservice/prod/feature-flags" \
  --value '{"expressCheckout": true, "loyaltyPoints": false}' \
  --type "String"

# Store payment API key (encrypted, Advanced tier with expiry)
aws ssm put-parameter \
  --name "/orderservice/prod/payment/api-key" \
  --value "sk_live_abc123xyz789" \
  --type "SecureString" \
  --tier "Advanced" \
  --policies '[
    {
      "Type": "Expiration",
      "Version": "1.0",
      "Attributes": {
        "Timestamp": "2025-12-31T00:00:00.000Z"
      }
    },
    {
      "Type": "ExpirationNotification",
      "Version": "1.0",
      "Attributes": {
        "Before": "14",
        "Unit": "Days"
      }
    }
  ]'
```

#### Step 2: Create IAM Role for ECS Task

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/orderservice/prod/*"
    },
    {
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/your-key-id",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "ssm.us-east-1.amazonaws.com"
        }
      }
    }
  ]
}
```

#### Step 3: Application Startup — Fetch All Config at Boot

```javascript
// config.js - Fetch all parameters at application startup
const { SSMClient, GetParametersByPathCommand } = require("@aws-sdk/client-ssm");

async function loadConfig() {
  const client = new SSMClient({ region: "us-east-1" });
  const params = {};
  let nextToken;

  do {
    const response = await client.send(new GetParametersByPathCommand({
      Path: "/orderservice/prod/",
      Recursive: true,
      WithDecryption: true,
      NextToken: nextToken
    }));

    for (const param of response.Parameters) {
      // Convert /orderservice/prod/db/host → db.host
      const key = param.Name.replace("/orderservice/prod/", "").replace(/\//g, ".");
      params[key] = param.Value;
    }

    nextToken = response.NextToken;
  } while (nextToken);

  return params;
}

module.exports = { loadConfig };
```

#### Step 4: Use in Application

```javascript
// app.js
const { loadConfig } = require("./config");

let config;

async function start() {
  config = await loadConfig();
  
  // Connect to database using fetched credentials
  const dbPool = new Pool({
    host: config["db.host"],
    password: config["db.password"],
    port: 5432
  });

  const featureFlags = JSON.parse(config["feature-flags"]);
  
  if (featureFlags.expressCheckout) {
    // Enable express checkout routes
  }
}
```

#### Step 5: ECS Task Definition — Native SSM Integration

```json
{
  "containerDefinitions": [{
    "name": "order-service",
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/order-service:latest",
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:ssm:us-east-1:123456789012:parameter/orderservice/prod/db/password"
      },
      {
        "name": "PAYMENT_API_KEY",
        "valueFrom": "arn:aws:ssm:us-east-1:123456789012:parameter/orderservice/prod/payment/api-key"
      }
    ],
    "environment": [
      {
        "name": "ENVIRONMENT",
        "value": "production"
      }
    ]
  }]
}
```

---

## Advantages

1. **No Additional Infrastructure**: Fully managed service — no servers, databases, or replication to manage. AWS handles durability and availability.

2. **Free Tier for Standard Parameters**: Up to 10,000 standard parameters at no cost, making it accessible for small teams and startups.

3. **Native AWS Integration**: Deep integration with EC2, ECS, EKS, Lambda, CloudFormation, CodeBuild, and CodeDeploy without additional configuration.

4. **Hierarchical Organization**: Path-based naming enables clean organization by environment, application, and component. Bulk retrieval with `GetParametersByPath` reduces API calls.

5. **Version History**: Every parameter change is versioned. You can roll back to any previous version, audit what changed, and when.

6. **Labels for Blue/Green Config**: Attach labels like `live`, `canary`, or `v2.3` to specific versions, enabling zero-downtime configuration deployments.

7. **KMS Integration**: SecureString parameters use envelope encryption with AWS KMS, supporting both AWS-managed keys and customer-managed keys (CMK) for compliance requirements.

8. **Parameter Policies (Advanced Tier)**: Automate expiration, send notifications before expiry, and alert when parameters haven't been rotated — reducing security debt.

9. **CloudTrail Auditing**: Every API call is automatically logged to CloudTrail, providing a complete audit trail for compliance (SOC 2, HIPAA, PCI-DSS).

10. **Cross-Account Access**: Parameters can be shared across AWS accounts using IAM roles and KMS key policies, enabling centralized configuration management.

11. **High Throughput**: Supports up to 1,000 transactions per second (TPS) for standard parameters, scalable to 40,000 TPS with higher-throughput settings.

---

## Limitations

| Limitation | Standard Tier | Advanced Tier |
|-----------|---------------|---------------|
| Max parameters per account/region | 10,000 | 100,000 |
| Max parameter value size | 4 KB | 8 KB |
| Parameter policies | ❌ Not supported | ✅ Supported |
| Cost | Free | $0.05/parameter/month |
| API throughput (default