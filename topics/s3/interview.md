# S3 — Interview Questions

---

## Easy

### 1. What is Amazon S3, and what are its primary use cases?

**Answer:**
Amazon S3 (Simple Storage Service) is an object storage service that offers industry-leading scalability, data availability, security, and performance. It stores data as objects within buckets, where each object consists of data, metadata, and a unique key.

**Primary use cases:**
- Static website hosting
- Backup and disaster recovery
- Data lakes and big data analytics
- Content distribution (images, videos, documents)
- Application data storage
- Log archiving and storage

---

### 2. What is the maximum object size you can store in S3, and what is the maximum size for a single PUT operation?

**Answer:**
- **Maximum object size:** 5 TB (terabytes)
- **Maximum single PUT operation size:** 5 GB (gigabytes)
- For objects larger than 5 GB, you **must** use the **Multipart Upload API**
- AWS recommends using Multipart Upload for objects **larger than 100 MB** for improved throughput and reliability

---

### 3. What are S3 storage classes, and when would you use each?

**Answer:**

| Storage Class | Use Case |
|---|---|
| **S3 Standard** | Frequently accessed data, low latency required |
| **S3 Intelligent-Tiering** | Unknown or changing access patterns |
| **S3 Standard-IA** | Infrequently accessed but requires rapid access |
| **S3 One Zone-IA** | Infrequent access, non-critical, single AZ |
| **S3 Glacier Instant Retrieval** | Archive data needing millisecond retrieval |
| **S3 Glacier Flexible Retrieval** | Archive, retrieval in minutes to hours |
| **S3 Glacier Deep Archive** | Long-term archive, retrieval in 12 hours |

---

### 4. What is an S3 bucket policy, and how does it differ from an S3 ACL?

**Answer:**
- **Bucket Policy:** A resource-based JSON policy attached to a bucket that grants or denies permissions to AWS accounts, IAM users, roles, or anonymous users. It is the recommended and more flexible method for access control.
- **ACL (Access Control List):** A legacy XML-based access control mechanism that grants basic read/write permissions to specific AWS accounts or predefined groups (e.g., public, authenticated users).

**Key differences:**
- Bucket policies support fine-grained conditions (IP, MFA, TLS, etc.); ACLs do not
- Bucket policies are evaluated by IAM; ACLs are evaluated by S3 separately
- AWS recommends disabling ACLs (using **Object Ownership = Bucket owner enforced**) and relying solely on bucket policies

---

### 5. What is S3 versioning, and why would you enable it?

**Answer:**
S3 versioning is a feature that keeps multiple variants of an object in the same bucket. When enabled, S3 automatically assigns a unique version ID to every object stored.

**Reasons to enable versioning:**
- **Accidental deletion protection:** Deleted objects receive a delete marker rather than being permanently removed
- **Overwrite protection:** Previous versions are preserved when an object is overwritten
- **Recovery:** You can restore any previous version of an object
- **Compliance:** Maintains a full history of object changes

**States:** Versioning can be in one of three states — *Unversioned* (default), *Versioning-enabled*, or *Versioning-suspended*.

---

## Medium

### 1. Explain S3 data consistency model and how it has evolved.

**Answer:**
Prior to December 2020, S3 provided **eventual consistency** for overwrite PUTs and DELETEs, which meant that after updating or deleting an object, you might still read the old version for a short period.

**Since December 2020**, Amazon S3 provides **strong read-after-write consistency** for all S3 operations:
- `PUT` of new objects
- Overwrite `PUT` of existing objects
- `DELETE` operations
- `LIST` operations

**What this means practically:**
- After a successful `PUT`, any subsequent `GET` or `LIST` will return the new data immediately
- After a successful `DELETE`, the object will no longer be returned in `GET` or `LIST`
- This consistency is automatic — no configuration required
- It applies to all regions and all storage classes

**Important caveat:** S3 does **not** provide locking mechanisms. If two clients write to the same key simultaneously, the last write wins, but both writes are acknowledged as successful.

---

### 2. How does S3 Multipart Upload work, and when should you use it?

**Answer:**
Multipart Upload allows uploading a single object as a set of parts. Each part is a contiguous portion of the object's data. Parts can be uploaded independently, in any order, and in parallel.

**How it works:**
1. **Initiate:** Call `CreateMultipartUpload` → receive an `UploadId`
2. **Upload parts:** Call `UploadPart` for each chunk (minimum 5 MB per part, except the last part; maximum 10,000 parts)
3. **Complete:** Call `CompleteMultipartUpload` with the list of part numbers and ETags → S3 assembles the final object
4. **Abort (optional):** Call `AbortMultipartUpload` to cancel and clean up

**When to use:**
- Objects larger than 100 MB (recommended by AWS)
- Objects larger than 5 GB (mandatory)
- Unreliable network connections (failed parts can be retried without restarting)
- Maximizing throughput by parallelizing uploads

**Important operational note:** Incomplete multipart uploads accumulate and incur storage costs. Always configure an **S3 Lifecycle rule** to abort incomplete multipart uploads after a set number of days (e.g., `AbortIncompleteMultipartUpload` after 7 days).

---

### 3. Describe S3 Lifecycle policies and provide a practical example.

**Answer:**
S3 Lifecycle policies automate the transition of objects between storage classes or the expiration (deletion) of objects based on rules you define. They help optimize storage costs without manual intervention.

**Two types of actions:**
- **Transition actions:** Move objects to a cheaper storage class after a specified number of days
- **Expiration actions:** Permanently delete objects (or delete expired delete markers/incomplete multipart uploads)

**Constraints for transitions (minimum days):**
- Standard → Standard-IA or One Zone-IA: minimum 30 days
- Any class → Glacier Instant Retrieval: minimum 90 days
- Any class → Glacier Deep Archive: minimum 180 days

**Practical example — Log file management:**
```json
{
  "Rules": [
    {
      "ID": "LogFileLifecycle",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/" },
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" },
        { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
      ],
      "Expiration": { "Days": 2555 },
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    }
  ]
}
```
This policy moves log files through progressively cheaper storage tiers and deletes them after ~7 years.

---

### 4. What is S3 Cross-Region Replication (CRR) and Same-Region Replication (SRR)? What are the prerequisites and limitations?

**Answer:**

**S3 Replication** automatically copies objects from a source bucket to one or more destination buckets.

**CRR (Cross-Region Replication):** Source and destination buckets are in different AWS regions.
**SRR (Same-Region Replication):** Source and destination buckets are in the same AWS region.

**Common use cases:**
- CRR: Disaster recovery, reducing latency for global users, meeting data sovereignty requirements
- SRR: Log aggregation, live replication between production and test accounts, compliance

**Prerequisites:**
1. Versioning must be **enabled** on both source and destination buckets
2. An **IAM role** must be created granting S3 permission to replicate objects
3. The bucket owner must have permission to replicate objects (relevant for cross-account)

**Limitations:**
- Only **new objects** written after replication is enabled are replicated (not existing objects — use **S3 Batch Replication** for existing objects)
- Objects encrypted with **SSE-C** cannot be replicated
- Objects in the source bucket that are **replicas** themselves are not replicated again (no chaining, unless explicitly configured)
- **Delete markers** are not replicated by default (must opt in)
- Permanent deletes (with version ID) are **never** replicated
- Lifecycle actions are not replicated

**Replication Time Control (RTC):** An optional feature that provides a 99.99% SLA to replicate 99.99% of objects within **15 minutes**, with replication metrics and notifications.

---

### 5. Explain S3 encryption options (SSE-S3, SSE-KMS, SSE-C, and client-side encryption).

**Answer:**

**Server-Side Encryption (SSE)** — encryption performed by AWS on your behalf:

| Type | Key Management | Use Case |
|---|---|---|
| **SSE-S3** | AWS manages keys (AES-256) | Simplest; no extra cost |
| **SSE-KMS** | AWS KMS manages keys | Audit trail, key rotation, cross-account access |
| **SSE-C** | Customer provides keys per request | Customer retains full key control |
| **DSSE-KMS** | Dual-layer KMS encryption | Regulatory requirements for double encryption |

**SSE-S3 (AES-256):**
- Header: `x-amz-server-side-encryption: AES256`
- No additional cost
- No audit trail for key usage

**SSE-KMS:**
- Header: `x-amz-server-side-encryption: aws:kms`
- Generates a data key per object using a CMK
- Full CloudTrail audit trail
- Additional KMS API call costs (can be significant at scale)
- Supports key rotation and cross-account access
- **KMS throttling** can become a bottleneck at high request rates — use **S3 Bucket Keys** to reduce KMS API calls by up to 99%

**SSE-C:**
- Customer provides the encryption key in the request headers
- S3 uses the key to encrypt, then **discards it** — S3 never stores the key
- HTTPS is mandatory
- AWS CLI and SDKs support this; S3 console does not

**Client-Side Encryption:**
- Data is encrypted **before** being sent to S3
- Customer manages encryption/decryption entirely
- AWS Encryption SDK or S3 client-side encryption library can be used

---

## Hard

### 1. How does S3 achieve high durability (11 nines), and what architectural mechanisms underpin this?

**Answer:**
Amazon S3 Standard is designed for **99.999999999% (11 nines) durability** and **99.99% availability** over a given year.

**Architectural mechanisms:**

**1. Geographic redundancy across AZs:**
S3 Standard stores data across a **minimum of 3 Availability Zones** within a region. Each AZ is a distinct physical location with independent power, cooling, and networking.

**2. Erasure coding:**
S3 does not simply replicate objects three times. It uses **erasure coding** (similar to RAID-6) to break objects into fragments and store redundant fragments across multiple AZs. This allows S3 to reconstruct an object even if multiple fragments are lost simultaneously.

**3. Continuous integrity checking:**
- S3 calculates checksums (MD5, CRC32, CRC32C, SHA-1, SHA-256) on all stored data
- Background processes continuously verify data integrity
- Detected bit-rot or corruption is automatically repaired using redundant fragments
- **Additional checksum support:** You can specify a checksum algorithm at upload time, and S3 validates it on retrieval

**4. Self-healing:**
When S3 detects data loss or corruption in one AZ (e.g., disk failure, hardware failure), it automatically replicates data from surviving AZs to restore the required redundancy level.

**5. Versioning and MFA Delete:**
While not directly related to physical durability, versioning protects against logical data loss (accidental deletion/overwrite), which complements physical durability.

**Why 11 nines matters in practice:**
If you store 10 million objects, you can expect to lose a single object once every 10,000 years on average. This durability is a property of the storage system itself and is independent of the number of objects stored.

**Important distinction:** Durability ≠ Availability. S3 One Zone-IA offers the same 11 nines durability *within a single AZ*, but has lower availability (99.5%) and no AZ-level redundancy.

---

### 2. Explain S3 Object Lock, WORM compliance, and how you would design a compliant archival system.

**Answer:**

**S3 Object Lock** implements a **Write Once Read Many (WORM)** model that prevents objects from being deleted or overwritten for a fixed period or indefinitely. It is used for regulatory compliance (SEC Rule 17a-4, FINRA, HIPAA, CJIS).

**Two retention modes:**

| Mode | Description | Who Can Override |
|---|---|---|
| **Compliance** | No one can delete or modify, including root | Nobody — not even AWS |
| **Governance** | Protects from most users; privileged users can override | Users with `s3:BypassGovernanceRetention` permission |

**Two methods to set retention:**

1. **Retention Period:** Specify a fixed date or duration (days/years) during which the object is WORM-protected
2. **Legal Hold:** A toggle (on/off) with no expiry date; remains until explicitly removed by a user with `s3:PutObjectLegalHold`

**Prerequisites:**
- Versioning must be enabled (Object Lock is version-aware)
- Object Lock must be enabled at **bucket creation time** (cannot be enabled on existing buckets)
- A **default retention policy** can be set at the bucket level

**Designing a compliant archival system:**

```
Architecture:
┌─────────────────────────────────────────────────────────┐
│                    Ingestion Layer                       │
│  Application → SQS → Lambda (validation) → S3 PUT       │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│              Compliance S3 Bucket                        │
│  - Object Lock: COMPLIANCE mode                         │
│  - Retention: 7 years (regulatory requirement)          │
│  - SSE-KMS encryption with CMK                          │
│  - Versioning: Enabled                                  │
│  - MFA Delete: Enabled                                  │
│  - Access Logging: Enabled → separate audit bucket      │
│  - CloudTrail data events: Enabled                      │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   Monitoring Layer                       │
│  CloudTrail → CloudWatch → SNS (unauthorized attempts)  │
│  AWS Config Rules: s3-bucket-object-lock-enabled        │
│                    s3-bucket-versioning-enabled         │
└─────────────────────────────────────────────────────────┘
```

**Additional controls:**
- Use a **dedicated AWS account** for the compliance bucket (blast radius reduction)
- Apply **SCPs** via AWS Organizations to prevent disabling Object Lock or versioning
- Use **Macie** to detect sensitive data and ensure it lands in the correct bucket
- Apply **bucket policy** denying `s3:DeleteObject`, `s3:DeleteObjectVersion`, and `s3:PutBucketVersioning` even to administrators

---

### 3. Describe S3 performance optimization techniques for high-throughput workloads.

**Answer:**

**S3 baseline performance:**
- **3,500 PUT/COPY/POST/DELETE requests per second** per prefix per bucket
- **5,500 GET/HEAD requests per second** per prefix per bucket
- Prefixes are the path components before the object name (e.g., `bucket/folder1/subfolder1