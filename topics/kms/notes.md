# KMS

## What is it?

**AWS Key Management Service (KMS)** is a fully managed, multi-tenant cryptographic key management service that enables you to create, store, rotate, and control access to encryption keys used to protect your data across AWS services and custom applications.

| Attribute | Detail |
|-----------|--------|
| **Official Name** | AWS Key Management Service (AWS KMS) |
| **Category** | Security, Identity & Compliance |
| **Type** | Managed Service (HSM-backed) |
| **Compliance** | FIPS 140-2 Level 3 (CloudHSM-backed), FIPS 140-2 Level 2 (standard) |
| **Key Standard** | AES-256-GCM (symmetric), RSA 2048/3072/4096, ECC P-256/P-384 (asymmetric) |

KMS acts as a **centralized key store and cryptographic authority** — it never exposes raw key material to callers. Instead, it performs cryptographic operations (encrypt, decrypt, sign, verify) on your behalf inside FIPS-validated Hardware Security Modules (HSMs). This is the fundamental design principle: the key itself never leaves KMS in plaintext.

---

## Why do we need it?

### The Problem Without KMS

Without a managed key service, organizations face several critical challenges:

1. **Key sprawl** — Encryption keys scattered across application code, config files, S3 buckets, and developer laptops.
2. **No auditability** — No centralized record of who used which key, when, and for what.
3. **Manual rotation** — Rotating keys requires application downtime and complex re-encryption logic.
4. **HSM cost & complexity** — Purchasing and managing dedicated HSMs is expensive and operationally heavy.
5. **Compliance gaps** — Regulators (PCI-DSS, HIPAA, SOC 2, GDPR) require demonstrable key management controls.

### When to Use KMS

| Scenario | Why KMS |
|----------|---------|
| Encrypting S3 objects, RDS databases, EBS volumes | Native AWS service integration (SSE) |
| Signing JWTs or code artifacts | Asymmetric key signing |
| Multi-region data encryption | Multi-Region Keys |
| Third-party SaaS data isolation | Customer Managed Keys (CMK) per tenant |
| Regulatory compliance (HIPAA, PCI-DSS) | Full audit trail via CloudTrail |
| Secrets management with envelope encryption | Data Key generation |

### Real Business Scenarios

- **Healthcare SaaS**: A patient record system encrypts each patient's data with a unique KMS-derived data key, ensuring PHI is isolated and auditable.
- **FinTech**: A payment processor uses KMS asymmetric keys to sign transaction payloads, providing non-repudiation.
- **E-Commerce**: An online retailer uses KMS to encrypt customer PII in DynamoDB, with automatic annual key rotation enabled.

---

## Internal Working

### Core Cryptographic Model: Envelope Encryption

KMS uses **envelope encryption** as its foundational pattern. You never encrypt large datasets directly with a KMS key — instead:

```
┌─────────────────────────────────────────────────────────────┐
│                    Envelope Encryption Flow                  │
│                                                             │
│  1. Your app calls GenerateDataKey(KeyId)                   │
│                                                             │
│  2. KMS returns:                                            │
│     ├── Plaintext Data Key (DEK) — use to encrypt data      │
│     └── Encrypted Data Key (eDEK) — store alongside data    │
│                                                             │
│  3. App encrypts data with plaintext DEK (AES-256-GCM)      │
│                                                             │
│  4. App discards plaintext DEK from memory                  │
│                                                             │
│  5. App stores: { encrypted_data + eDEK }                   │
│                                                             │
│  To Decrypt:                                                │
│  6. App calls Decrypt(eDEK) → KMS returns plaintext DEK     │
│  7. App decrypts data locally using plaintext DEK           │
└─────────────────────────────────────────────────────────────┘
```

### Key Hierarchy

```
AWS KMS Root Keys (HSM-protected, never leave KMS)
        │
        ├── Customer Master Key (CMK) / KMS Key
        │       │
        │       └── Data Encryption Keys (DEKs)
        │               │
        │               └── Your actual data (encrypted)
        │
        └── AWS Managed Keys (aws/s3, aws/rds, etc.)
```

### HSM Architecture

- KMS keys are stored in **FIPS 140-2 validated HSMs** distributed across multiple Availability Zones.
- Each KMS key has **multiple copies** across AZs for durability and availability.
- Key material is **never exported in plaintext** from the HSM boundary.
- All cryptographic operations happen **inside the HSM**.

### Request Processing Pipeline

```
Client Request (Encrypt/Decrypt)
        │
        ▼
KMS API Endpoint (TLS 1.2+)
        │
        ▼
IAM Policy Evaluation
        │
        ▼
KMS Key Policy Evaluation
        │
        ▼
Grant Evaluation (if applicable)
        │
        ▼
HSM Cluster (cryptographic operation)
        │
        ▼
CloudTrail Audit Log
        │
        ▼
Response to Client
```

---

## Architecture

### Key Types

```
KMS Keys
├── Symmetric Keys (AES-256-GCM)
│   ├── AWS Owned Keys (free, no visibility)
│   ├── AWS Managed Keys (aws/<service>)
│   └── Customer Managed Keys (CMK)
│
└── Asymmetric Keys
    ├── RSA (2048, 3072, 4096)
    │   ├── Encrypt/Decrypt
    │   └── Sign/Verify
    ├── ECC (P-256, P-384, secp256k1)
    │   └── Sign/Verify only
    └── SM2 (China regions only)
```

### Key Ownership Models

| Key Type | Control | Rotation | Cost | Visibility |
|----------|---------|----------|------|------------|
| AWS Owned | AWS | AWS | Free | None |
| AWS Managed | Shared | AWS (auto, 1yr) | Free | CloudTrail only |
| Customer Managed | Full | Manual or auto | $1/month | Full |
| Imported Key Material | Full | Manual only | $1/month | Full |

### Multi-Region Keys Architecture

```
Primary Key (us-east-1)
    mrk-1234567890abcdef0
        │
        ├── Replica Key (eu-west-1)
        │       mrk-1234567890abcdef0 (same key ID prefix)
        │
        └── Replica Key (ap-southeast-1)
                mrk-1234567890abcdef0
```
- Same key material replicated across regions.
- Ciphertext encrypted in one region can be decrypted in another.
- Use case: active-active multi-region applications, disaster recovery.

### Key Policy Architecture

```
KMS Key Policy (Resource-based)
    │
    ├── Root Account Statement (mandatory)
    │       "Principal": {"AWS": "arn:aws:iam::123456789012:root"}
    │
    ├── Key Administrators Statement
    │       (Create, Delete, Describe — NOT Encrypt/Decrypt)
    │
    └── Key Users Statement
            (Encrypt, Decrypt, GenerateDataKey, etc.)
```

---

## Real World Example

### Scenario: Multi-Tenant SaaS Application with Per-Tenant Encryption

**Context**: A B2B SaaS company stores sensitive client documents in S3. Each client's data must be encrypted with a separate key, and the company must prove to auditors that one client's key cannot decrypt another client's data.

#### Step-by-Step Walkthrough

**Step 1: Provision per-tenant KMS keys (Terraform/CLI)**
```bash
# Create a KMS key for Tenant A
aws kms create-key \
  --description "Tenant-A Document Encryption Key" \
  --key-usage ENCRYPT_DECRYPT \
  --tags TagKey=TenantId,TagValue=tenant-a
```

**Step 2: Create an alias for easy reference**
```bash
aws kms create-alias \
  --alias-name alias/tenant-a-docs \
  --target-key-id <key-id-from-step-1>
```

**Step 3: Attach a key policy restricting access to Tenant A's IAM role**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable root account access",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "TenantA Application Access",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:role/TenantAAppRole"},
      "Action": ["kms:GenerateDataKey", "kms:Decrypt"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com"
        }
      }
    }
  ]
}
```

**Step 4: Upload encrypted document to S3 using SSE-KMS**
```bash
aws s3 cp tenant-a-contract.pdf s3://saas-documents/tenant-a/ \
  --sse aws:kms \
  --sse-kms-key-id alias/tenant-a-docs
```

**Step 5: Application reads document (KMS auto-decrypts)**
```javascript
// S3 GetObject automatically triggers KMS Decrypt
const response = await s3Client.send(new GetObjectCommand({
  Bucket: 'saas-documents',
  Key: 'tenant-a/tenant-a-contract.pdf'
}));
```

**Step 6: Audit — Review CloudTrail for all key usage**
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=<key-id> \
  --start-time 2024-01-01 \
  --end-time 2024-01-31
```

**Result**: Each tenant's data is cryptographically isolated. If Tenant A's IAM role is compromised, Tenant B's data remains protected. Auditors can review the full CloudTrail log of every decrypt operation.

---

## Advantages

1. **Fully Managed HSM Backend** — No hardware to provision, patch, or manage. AWS handles HA and durability across AZs.

2. **Deep AWS Service Integration** — 100+ AWS services natively support KMS encryption (S3, RDS, EBS, DynamoDB, Lambda, SQS, SNS, Secrets Manager, etc.).

3. **Centralized Audit Trail** — Every cryptographic operation is logged to CloudTrail with caller identity, timestamp, key ID, and request parameters.

4. **Fine-Grained Access Control** — Combines IAM policies, key policies, and grants for three independent layers of authorization.

5. **Automatic Key Rotation** — Enable automatic annual rotation for symmetric CMKs with zero application changes required.

6. **Envelope Encryption at Scale** — GenerateDataKey enables high-throughput encryption without KMS API rate limits becoming a bottleneck.

7. **Multi-Region Keys** — Encrypt in one region, decrypt in another without re-encryption — critical for DR and global applications.

8. **BYOK (Bring Your Own Key)** — Import your own key material for compliance requirements that mandate key origin control.

9. **Asymmetric Key Support** — Sign/verify operations for code signing, JWT signing, and certificate workflows.

10. **Cost Efficiency** — $1/month per CMK is dramatically cheaper than managing dedicated HSMs ($1,500+/month for CloudHSM).

---

## Limitations

### Hard Limits

| Limit | Value |
|-------|-------|
| KMS keys per account per region | 100,000 (soft limit, adjustable) |
| Key aliases per account per region | 100,000 |
| Grants per KMS key | 50,000 |
| Max plaintext size for Encrypt API | 4 KB |
| Max ciphertext size for Decrypt API | 6,144 bytes |
| GenerateDataKey response (plaintext) | 4,096 bytes |
| Key policy document size | 32 KB |
| Request quota (default, varies by API) | 5,500 – 30,000 requests/second |

### Functional Limitations

- **No direct large-data encryption** — Must use envelope encryption for data > 4 KB.
- **Asymmetric keys cannot be auto-rotated** — Manual rotation only; requires re-encryption of all data.
- **Imported key material cannot be auto-rotated** — Customer is responsible for rotation.
- **KMS keys are region-scoped** (except Multi-Region Keys) — A key in `us-east-1` cannot be used directly in `eu-west-1`.
- **Deleted keys cannot be recovered** — 7–30 day waiting period before deletion; once deleted, all data encrypted with that key is permanently inaccessible.
- **Key material cannot be exported** — By design; use CloudHSM if you need exportable key material.
- **API throttling** — High-throughput applications must implement exponential backoff and data key caching.
- **CloudHSM-backed custom key stores** — Higher cost and operational complexity; single-AZ HSM cluster is a risk if not configured for HA.

---

## Best Practices

### Key Management

1. **Use Customer Managed Keys (CMKs) for sensitive workloads** — Never rely solely on AWS Managed Keys when you need key policy control or auditability at the key level.

2. **Enable automatic key rotation** — For symmetric CMKs, enable `EnableKeyRotation`. KMS retains old key versions to decrypt existing ciphertext.

3. **Use key aliases** — Reference keys by alias (`alias/my-app-key`) rather than key ID in application code to simplify rotation and key replacement.

4. **Apply the principle of least privilege to key policies** — Separate key administrators (who manage the key) from key users (who encrypt/decrypt).

5. **Never delete a KMS key without verifying no data depends on it** — Use CloudTrail to audit recent usage before scheduling deletion.

### Application Architecture

6. **Implement data key caching** — Use the AWS Encryption SDK's data key caching to reuse data keys for a configurable time window, reducing KMS API calls and cost.

```javascript
// AWS Encryption SDK caching example concept
const cache = new LocalCryptographicMaterialsCache(capacity: 1000);
const cachingCMM = new CachingCryptographicMaterialsManager({
  backingMaterials: keyring,
  cache,
  maxAge: 60 * 1000, // 60 seconds
  maxMessagesEncrypted: 100
});
```

7. **Use VPC Endpoints for KMS** — Deploy `com.amazonaws.<region>.kms` interface VPC endpoints to keep KMS traffic off the public internet.

8. **Set `kms:ViaService` conditions** — Restrict key usage to specific AWS services to prevent direct API abuse.

9. **Tag KMS keys** — Apply consistent tags (Environment, Application, Owner, CostCenter) for cost allocation and governance.

10. **Use Multi-Region Keys for active-active architectures** — Avoid cross-region KMS API calls which add latency and cross-region data transfer costs.

### Well-Architected Alignment

11. **Security Pillar** — Enable CloudTrail for all KMS API calls. Set up CloudWatch alarms for unauthorized access attempts (`kms:AccessDeniedException`).

12. **Reliability Pillar** — Implement exponential backoff for KMS API throttling. Cache data keys to reduce dependency on KMS availability.

13. **Cost Optimization Pillar** — Use data key caching to minimize `GenerateDataKey` calls. Audit unused keys and schedule deletion.

---

## Common Mistakes

### 1. Encrypting Large Data Directly with KMS Encrypt API
**Anti-pattern