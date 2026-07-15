# S3

## What is it?

**Amazon Simple Storage Service (Amazon S3)** is a fully managed, object storage service provided by AWS under the **Storage** category. It is designed to store and retrieve any amount of data from anywhere on the internet with high durability, availability, and scalability.

S3 stores data as **objects** within **buckets**. An object consists of:
- **Data** (the actual file content)
- **Key** (a unique identifier/path within the bucket)
- **Metadata** (system and user-defined key-value pairs)
- **Version ID** (when versioning is enabled)

S3 is not a file system or a block storage — it is a flat, key-value object store accessible via HTTP/HTTPS APIs. It supports objects from **0 bytes to 5 TB** in size and provides **11 nines (99.999999999%)** of durability.

---

## Why do we need it?

### Problems It Solves

| Problem | How S3 Solves It |
|---|---|
| Scalable storage without managing infrastructure | Fully managed, scales automatically |
| Durable long-term data retention | 11 nines durability via redundant storage |
| Globally accessible content delivery | Public/private endpoints, CloudFront integration |
| Data lake foundation | Stores structured, semi-structured, and unstructured data |
| Backup and disaster recovery | Cross-region replication, versioning |
| Static website hosting | Built-in static website hosting feature |

### Real Business Scenarios

1. **Media & Entertainment**: Netflix stores video assets, thumbnails, and metadata in S3 for global distribution.
2. **E-Commerce**: Product images, invoices, and customer documents stored and served from S3.
3. **Data Analytics**: Raw logs from application servers stored in S3 as the source for Athena/Redshift Spectrum queries.
4. **Backup & Archive**: Nightly database dumps stored in S3 with lifecycle rules to transition to Glacier after 30 days.
5. **Software Distribution**: Application binaries, packages, and installers hosted on S3 with versioning enabled.
6. **Machine Learning**: Training datasets and model artifacts stored in S3 for SageMaker pipelines.

---

## Internal Working

### Storage Architecture

S3 operates on a **distributed, highly redundant storage infrastructure** managed entirely by AWS. Internally:

1. **Object Ingestion**: When you upload an object, S3 receives the data via HTTP PUT (or multipart upload for large files). The data is split and stored across multiple **Availability Zones** (at least 3 AZs for Standard storage class).

2. **Data Placement**: S3 uses a proprietary distributed storage engine that:
   - Replicates data across multiple physical facilities and devices
   - Uses **erasure coding** techniques to ensure durability
   - Stores multiple copies across geographically separated hardware

3. **Namespace**: S3 uses a **flat namespace** — there are no real directories. The "/" character in keys creates a visual hierarchy in the console, but internally it's just part of the key string.

4. **Consistency Model**: Since December 2020, S3 provides **strong read-after-write consistency** for all operations (PUT, GET, LIST, DELETE). Previously, it offered eventual consistency for overwrites and deletes.

5. **Index Layer**: S3 maintains a distributed metadata index that maps object keys to their physical storage locations. This is why LIST operations can be slower for buckets with billions of objects.

6. **Multipart Upload**: For objects > 100 MB (recommended), S3 accepts parallel part uploads (up to 10,000 parts, each 5 MB–5 GB), then assembles them server-side. This enables:
   - Parallel upload threads
   - Resumable uploads
   - Improved throughput

7. **Request Rate Scaling**: S3 automatically partitions the keyspace based on request patterns. Each prefix can handle **3,500 PUT/COPY/POST/DELETE** and **5,500 GET/HEAD** requests per second. AWS distributes load across partitions automatically.

```
Client → S3 API Endpoint (HTTPS)
            ↓
        Load Balancer / Request Router
            ↓
        Metadata Service (Key → Location Mapping)
            ↓
        Distributed Storage Nodes (AZ-1, AZ-2, AZ-3)
            ↓
        Redundant Physical Disks / Erasure Coded Chunks
```

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS Region                           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    S3 Bucket                        │   │
│  │                                                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │  Object  │  │ Metadata │  │   Access Control  │  │   │
│  │  │  Store   │  │  Index   │  │   (ACL / Policy)  │  │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘  │   │
│  │                                                     │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │           Storage Classes                    │   │   │
│  │  │  Standard | IA | One Zone IA | Glacier | ... │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  │                                                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │Versioning│  │Lifecycle │  │   Replication     │  │   │
│  │  │          │  │  Rules   │  │   (CRR / SRR)     │  │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Storage Classes

| Storage Class | Use Case | Availability | Retrieval | Min Duration |
|---|---|---|---|---|
| **S3 Standard** | Frequently accessed data | 99.99% | Instant | None |
| **S3 Intelligent-Tiering** | Unknown/changing access patterns | 99.9% | Instant | None |
| **S3 Standard-IA** | Infrequently accessed | 99.9% | Instant | 30 days |
| **S3 One Zone-IA** | Infrequent, non-critical | 99.5% | Instant | 30 days |
| **S3 Glacier Instant** | Archive, rare access | 99.9% | Instant | 90 days |
| **S3 Glacier Flexible** | Archive, minutes-hours retrieval | 99.99% | 1–12 hrs | 90 days |
| **S3 Glacier Deep Archive** | Long-term archive | 99.99% | 12–48 hrs | 180 days |

### Key Architectural Patterns

1. **Data Lake Pattern**: S3 as the central repository with organized prefix structure (`s3://bucket/raw/`, `s3://bucket/processed/`, `s3://bucket/curated/`)
2. **Static Website Hosting**: S3 + CloudFront + Route 53 for globally distributed static sites
3. **Event-Driven Architecture**: S3 Event Notifications → Lambda/SQS/SNS → Processing pipeline
4. **Backup Hub Pattern**: Cross-region replication for DR, lifecycle policies for tiered archival
5. **Access Point Pattern**: Multiple access points per bucket for isolated access control

---

## Real World Example

### Scenario: E-Commerce Product Image Pipeline

**Context**: An e-commerce platform needs to handle product image uploads from sellers, auto-generate thumbnails, and serve images globally with low latency.

#### Step-by-Step Walkthrough

**Step 1: Bucket Setup**
```
s3://ecommerce-raw-images-prod/       ← Seller uploads raw images
s3://ecommerce-processed-images-prod/ ← Processed/resized images
```

**Step 2: Seller Uploads Image**
- Seller authenticates via the web app
- Backend generates a **pre-signed URL** (15-minute expiry) for direct upload to `ecommerce-raw-images-prod`
- Seller's browser uploads directly to S3 (bypasses application server)

**Step 3: S3 Event Notification Triggers**
- S3 emits an `s3:ObjectCreated:Put` event
- Event routed to an **SQS queue** for durability
- **Lambda function** polls the queue and processes each image:
  - Downloads raw image from S3
  - Generates multiple sizes (thumbnail 100x100, medium 400x400, large 800x800)
  - Uploads processed images to `ecommerce-processed-images-prod`
  - Stores metadata in DynamoDB

**Step 4: Global Distribution**
- CloudFront distribution points to `ecommerce-processed-images-prod`
- Cache-Control headers set to 1 year (images are immutable by key)
- CloudFront serves images from edge locations globally

**Step 5: Lifecycle Management**
- Raw images transition to **S3 Standard-IA** after 30 days
- Raw images expire after 365 days
- Processed images remain in **S3 Standard** for active serving

```
Seller → Pre-signed URL → S3 Raw Bucket
                              ↓ Event Notification
                           SQS Queue
                              ↓
                         Lambda Function
                              ↓
                    S3 Processed Bucket ← CloudFront ← End Users
                              ↓
                          DynamoDB (metadata)
```

---

## Advantages

1. **Extreme Durability**: 99.999999999% (11 nines) durability — designed to sustain concurrent loss of data in two facilities
2. **Massive Scalability**: No practical storage limit; automatically scales to exabytes
3. **Strong Consistency**: Strong read-after-write consistency for all S3 GET, PUT, LIST, DELETE operations
4. **Flexible Access Control**: Bucket policies, IAM policies, ACLs, Access Points, VPC endpoints
5. **Rich Ecosystem Integration**: Native integration with 100+ AWS services
6. **Lifecycle Management**: Automated tiering and expiration reduces costs without manual intervention
7. **Versioning**: Protects against accidental deletion and overwrites
8. **Event-Driven**: Native event notifications to Lambda, SQS, SNS, and EventBridge
9. **Query In-Place**: S3 Select and Athena allow SQL queries directly on S3 data
10. **Global Replication**: Cross-Region Replication (CRR) and Same-Region Replication (SRR)
11. **Static Website Hosting**: Built-in support for hosting static websites
12. **Pre-signed URLs**: Secure, time-limited access to private objects without exposing credentials
13. **Transfer Acceleration**: Uses CloudFront edge locations to speed up uploads from distant clients
14. **Cost Effective**: Pay only for what you store and transfer; no minimum fees for Standard class

---

## Limitations

### Hard Limits

| Constraint | Limit |
|---|---|
| Maximum object size | **5 TB** |
| Maximum single PUT upload | **5 GB** |
| Recommended multipart threshold | **100 MB** |
| Maximum parts in multipart upload | **10,000** |
| Minimum part size (multipart) | **5 MB** (except last part) |
| Bucket name length | **3–63 characters** |
| Maximum buckets per account | **100** (soft limit, can request up to 1,000) |
| Maximum object metadata size | **2 KB** |
| Maximum tags per object | **10** |
| Maximum bucket policy size | **20 KB** |
| Maximum lifecycle rules per bucket | **1,000** |
| Maximum replication rules per bucket | **1,000** |

### Behavioral Limitations

- **No native locking**: S3 is not designed for concurrent writes to the same object (use Object Lock for WORM, or DynamoDB for coordination)
- **LIST performance**: Listing millions of objects in a bucket is slow; consider S3 Inventory for bulk enumeration
- **No atomic rename**: Moving/renaming objects requires a copy + delete operation
- **Eventual consistency for bucket operations**: Bucket creation/deletion can take time to propagate globally
- **No partial object updates**: You must overwrite the entire object to update it
- **Cross-account replication complexity**: Requires careful IAM and bucket policy configuration
- **Minimum storage duration charges**: Standard-IA (30 days), Glacier (90 days), Deep Archive (180 days)
- **Retrieval fees**: Glacier and Deep Archive have per-GB retrieval costs

---

## Best Practices

### Security
- ✅ **Block Public Access** at the account level unless explicitly required
- ✅ Enable **S3 Object Lock** for compliance and WORM requirements
- ✅ Use **bucket policies** over ACLs (ACLs are a legacy access control mechanism)
- ✅ Enable **SSE-KMS** for sensitive data with customer-managed keys
- ✅ Use **VPC Endpoints (Gateway type)** to keep S3 traffic off the public internet
- ✅ Enable **MFA Delete** for versioned buckets containing critical data
- ✅ Use **Access Points** to simplify access control for shared datasets

### Performance
- ✅ Use **random prefixes or date-based prefixes** to distribute load across S3 partitions (though less critical after 2018 performance improvements)
- ✅ Use **multipart upload** for objects > 100 MB
- ✅ Use **S3 Transfer Acceleration** for uploads from geographically distant clients
- ✅ Use **byte-range fetches** to parallelize large object downloads
- ✅ Use **CloudFront** as a CDN in front of S3 for read-heavy workloads

### Cost Optimization (AWS Well-Architected: Cost Pillar)
- ✅ Implement **Lifecycle Policies** to transition objects to cheaper storage classes
- ✅ Enable **S3 Intelligent-Tiering** for data with unknown or changing access patterns
- ✅ Use **S3 Inventory** to audit storage and find optimization opportunities
- ✅ Enable **S3 Storage Lens** for organization-wide visibility
- ✅ Delete incomplete multipart uploads using lifecycle rules
- ✅ Use **S3 Select** to retrieve only needed data instead of downloading full objects

### Reliability (AWS Well-Architected: Reliability Pillar)
- ✅ Enable **Versioning** for critical data buckets
- ✅ Configure **Cross-Region Replication** for disaster recovery
- ✅ Use **Event Notifications** with SQS (not Lambda directly) for reliable event processing
- ✅ Set up **S3 Replication Time Control (RTC)** for predictable replication SLAs (99.99% of objects replicated within 15 minutes)

### Operational Excellence
- ✅ Use **S3 Storage Lens** and **CloudWatch** for monitoring
- ✅ Enable **Server Access Logging** or **AWS CloudTrail** for audit trails
- ✅ Tag all buckets with environment, team, cost center, and data classification tags
- ✅ Use **AWS Config** rules to detect non-compliant bucket configurations

---

## Common Mistakes

### 1. ❌ Making Buckets Publicly Accessible Unintentionally
**Problem**: Misconfigured bucket policies or ACLs expose sensitive data.
**Fix**: Enable **S3 Block Public Access** at the account level. Use pre-signed URLs for temporary access.

### 2. ❌ Using S3 as a File System
**Problem**: Treating S3 like a POSIX file system — expecting atomic renames, directory operations, or file locking.
**Fix**: Design around S3's object model. Use EFS or FSx for file system semantics.

### 3. ❌ Not Using Multipart Upload for Large Objects
**Problem**: Single PUT for large files (>100 MB) fails on network interruption and is slow.