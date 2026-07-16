# EFS — Interview Questions

---

## Easy

---

**Q1. What is Amazon EFS, and what problem does it solve?**

**Answer:**
Amazon Elastic File System (EFS) is a fully managed, serverless, elastic NFS (Network File System) file storage service for use with AWS Cloud services and on-premises resources. It automatically grows and shrinks as you add and remove files, so you don't need to provision or manage capacity. It solves the problem of shared file storage — multiple EC2 instances, containers, or Lambda functions can concurrently read and write to the same file system, which is something EBS (block storage) cannot do natively across multiple instances.

---

**Q2. What are the two storage classes available in EFS?**

**Answer:**
EFS offers two primary storage classes:

1. **EFS Standard** — For frequently accessed files. Data is stored redundantly across multiple Availability Zones.
2. **EFS Standard-Infrequent Access (EFS Standard-IA)** — For files not accessed every day. Lower per-GB storage price but a per-GB retrieval fee applies.

Additionally, there are **One Zone** variants of both:
- **EFS One Zone** — Stored in a single AZ; lower cost.
- **EFS One Zone-IA** — Infrequent access files in a single AZ; lowest cost option.

---

**Q3. What network protocol does EFS use?**

**Answer:**
EFS uses the **NFSv4 (Network File System version 4)** and **NFSv4.1** protocols. Clients mount EFS file systems using standard NFS mount commands. Because it uses NFS, EFS is natively compatible with Linux-based operating systems. Windows is not natively supported — EFS is designed for Linux/Unix workloads.

---

**Q4. How does EFS differ from EBS?**

**Answer:**

| Feature | EFS | EBS |
|---|---|---|
| Type | File storage (NFS) | Block storage |
| Multi-attach | Yes, thousands of instances | Limited (EBS Multi-Attach for io1/io2 up to 16 instances in same AZ) |
| Scope | Multi-AZ (Regional) | Single AZ (typically) |
| Elasticity | Automatic, no provisioning | Must provision size upfront |
| OS Support | Linux only | Linux and Windows |
| Use Case | Shared file system, CMS, home dirs | Databases, boot volumes, single-instance apps |
| Pricing | Per GB stored + retrieval | Per GB provisioned |

---

**Q5. What is an EFS Mount Target?**

**Answer:**
A **Mount Target** is a network endpoint within a specific Availability Zone that allows EC2 instances in that AZ to connect to an EFS file system. Each mount target has an IP address and a DNS name. Best practice is to create one mount target per AZ in your VPC so that instances in any AZ can access the file system with low latency. Mount targets are associated with a security group that controls inbound NFS traffic (port 2049).

---

## Medium

---

**Q1. Explain EFS Performance Modes and when you would choose each one.**

**Answer:**
EFS offers two performance modes, selected at file system creation time and **cannot be changed afterward**:

1. **General Purpose (default)**
   - Lowest per-operation latency.
   - Ideal for latency-sensitive use cases: web serving, content management, home directories, development environments.
   - Supports up to 35,000 read IOPS and 7,000 write IOPS per file system.
   - Recommended for the vast majority of workloads.

2. **Max I/O**
   - Higher aggregate throughput and IOPS but with slightly higher latencies per operation.
   - Designed for highly parallelized workloads that can tolerate higher latency: big data analytics, media processing, genomics.
   - Scales to hundreds of thousands of IOPS across thousands of EC2 instances.
   - **Note:** EFS Elastic Throughput mode (introduced later) largely reduces the need for Max I/O in many scenarios.

**Choosing guidance:**
- Start with General Purpose. Only move to Max I/O if you are seeing the `PercentIOLimit` CloudWatch metric consistently approaching 100%.

---

**Q2. What are the EFS Throughput Modes and how do they work?**

**Answer:**
EFS provides three throughput modes:

1. **Elastic Throughput (recommended default)**
   - Automatically scales throughput up or down based on workload needs.
   - Can burst to 3 GiB/s for reads and 1 GiB/s for writes.
   - Pay only for what you use; no need to predict throughput requirements.
   - Best for spiky or unpredictable workloads.

2. **Bursting Throughput**
   - Throughput scales with the amount of data stored.
   - Baseline: 50 MiB/s per TiB of data stored.
   - Burst: Up to 100 MiB/s (for file systems under 1 TiB) or higher for larger file systems.
   - Uses a **credit system** — you earn credits when below baseline and spend them when bursting.
   - Monitor with `BurstCreditBalance` CloudWatch metric.

3. **Provisioned Throughput**
   - You specify throughput independent of storage size.
   - Useful when you need more throughput than Bursting provides for your current storage size.
   - Billed for the provisioned throughput above what Bursting would provide.
   - Example: A 100 GB file system that needs 200 MiB/s throughput.

**Key consideration:** If `BurstCreditBalance` is consistently depleting to zero, consider switching to Provisioned or Elastic throughput.

---

**Q3. How does EFS Lifecycle Management work, and what are the available lifecycle policies?**

**Answer:**
EFS Lifecycle Management automatically moves files between storage classes based on access patterns, reducing costs without requiring application changes.

**How it works:**
- A background process monitors file access timestamps.
- Files not accessed within the configured period are transitioned to the IA (Infrequent Access) storage class.
- When an IA file is accessed, it can optionally be moved back to Standard (using the **Transition into Standard** policy).

**Available Lifecycle Policies (Transition to IA):**
- After 1 day
- After 7 days
- After 14 days
- After 30 days
- After 60 days
- After 90 days
- After 180 days
- After 270 days
- After 365 days

**Transition back to Standard:** You can configure EFS to move files back to Standard on first access, which is useful for workloads with unpredictable access patterns.

**Important notes:**
- Files smaller than 128 KB are **not** moved to IA (overhead cost would exceed savings).
- Metadata (directory listings, file metadata) always remains in Standard storage.
- The transition does not affect file system namespace or application behavior.

---

**Q4. How do you secure an EFS file system? Describe all layers of security.**

**Answer:**
EFS security is multi-layered:

**1. Network Security (Transport Layer)**
- **VPC Security Groups:** Mount targets have security groups. Inbound rule must allow TCP port 2049 (NFS) from EC2 security groups.
- **VPC Isolation:** EFS is only accessible within the VPC (or via VPN/Direct Connect for on-premises).

**2. Encryption**
- **Encryption at Rest:** Uses AWS KMS. Enabled at file system creation (cannot be changed later). Uses either AWS-managed key (`aws/elasticfilesystem`) or a customer-managed KMS key.
- **Encryption in Transit:** Uses TLS via the `amazon-efs-utils` mount helper with the `tls` mount option. The ` stunnel` daemon handles the TLS tunnel transparently.

**3. IAM Authorization (Identity-based)**
- **IAM Policies:** Control who can call EFS API actions (CreateFileSystem, DeleteFileSystem, etc.).
- **EFS Resource-based Policies (File System Policies):** JSON policies attached directly to the file system. Can enforce:
  - `elasticfilesystem:ClientMount` — allow/deny mounting
  - `elasticfilesystem:ClientWrite` — allow/deny write access
  - `elasticfilesystem:ClientRootAccess` — allow/deny root access

**4. POSIX Permissions (File System Layer)**
- Standard Linux user/group/other permissions (chmod, chown).
- Access Points can enforce a specific POSIX user identity and root directory.

**5. EFS Access Points**
- Application-specific entry points with enforced POSIX user/group IDs.
- Enforce a root directory so applications cannot access files outside their designated path.
- Commonly used with Lambda and ECS to isolate application data.

---

**Q5. What is an EFS Access Point and why would you use one?**

**Answer:**
An **EFS Access Point** is an application-specific entry point into an EFS file system that simplifies managing application access to shared datasets.

**Key features:**
1. **Enforced Root Directory (`/path`):** The access point restricts the application's view to a specific directory. The application sees this path as `/` (root), preventing it from accessing files outside this directory.
2. **Enforced POSIX Identity:** You can specify a user ID (UID), group ID (GID), and secondary GIDs. All file operations through the access point use this identity, regardless of the NFS client's user.
3. **Automatic Directory Creation:** If the root directory doesn't exist, EFS can create it with specified ownership and permissions on first use.

**Use Cases:**
- **Lambda functions:** Lambda doesn't have a traditional user context. Access Points give it a consistent POSIX identity.
- **ECS/Fargate containers:** Multiple containers sharing one EFS file system, each with isolated directories via separate access points.
- **Multi-tenant applications:** Different teams or applications get isolated directory trees on the same file system.
- **Simplified IAM policy:** IAM policies can reference specific access point ARNs, making permissions more granular.

**Example mount command:**
```bash
mount -t efs -o tls,accesspoint=fsap-xxxxxxxxx fs-xxxxxxxx:/ /mnt/efs
```

---

## Hard

---

**Q1. Deep dive: How does EFS handle consistency, and what are the implications for distributed applications?**

**Answer:**
EFS provides **close-to-open consistency semantics**, which is the standard NFS consistency model. Understanding this is critical for distributed applications:

**Close-to-Open Consistency:**
- When a file is closed on one client, subsequent opens on any other client are guaranteed to see the latest data.
- Writes are not guaranteed to be immediately visible to other clients until the writing client closes the file.
- This is different from strong consistency (every read sees the latest write immediately) and eventual consistency (reads may lag significantly).

**Practical Implications:**

1. **Read-after-write within same client:** A client that writes and then reads the same file will see its own writes (assuming no intervening close/open is needed in some edge cases).

2. **Cross-client visibility:** If Process A on Instance 1 writes to a file and keeps it open, Process B on Instance 2 may not see those writes until A closes the file (or flushes with `fsync()`).

3. **`fsync()` and `O_SYNC`:** Applications can use `fsync()` to force data to the EFS storage servers before closing, ensuring durability. Using `O_SYNC` flag makes every write synchronous, which is more durable but significantly impacts performance.

4. **Distributed Locking (NFSv4 Lock):** EFS supports NFSv4 mandatory and advisory locking via the `lockd` daemon. However, relying on file locks for application coordination across many clients can create bottlenecks. Applications should be designed to minimize lock contention.

5. **Metadata Consistency:** Directory operations (create, delete, rename) are strongly consistent — all clients immediately see the updated directory state.

**Performance vs. Consistency Trade-offs:**
- Using `sync` mount option forces synchronous writes for every operation — very durable but extremely slow.
- Default `async` mount option provides better performance but relies on client-side caching; data may be lost if the client crashes before flushing.
- The `amazon-efs-utils` with TLS uses `async` by default for performance.

**Application Design Recommendations:**
- Use EFS for workloads where close-to-open consistency is sufficient (most web/CMS/batch workloads).
- Avoid EFS for workloads requiring strict POSIX consistency across many concurrent writers (e.g., databases writing to the same files — use EBS instead).
- For shared configuration files that are written infrequently and read frequently, EFS is excellent.

---

**Q2. Explain EFS Replication, its architecture, and its limitations.**

**Answer:**
**EFS Replication** provides automatic, continuous, asynchronous replication of an EFS file system to another AWS Region or another AZ within the same Region.

**Architecture:**

```
Source Region (us-east-1)          Destination Region (eu-west-1)
┌─────────────────────────┐        ┌─────────────────────────────┐
│  EFS File System        │        │  EFS Replica File System    │
│  (Read/Write)           │───────▶│  (Read-Only)                │
│                         │  Async │                             │
│  RPO: Minutes           │        │  Can be promoted to R/W     │
└─────────────────────────┘        └─────────────────────────────┘
```

**Key Characteristics:**

1. **Asynchronous Replication:** Changes are replicated with a typical RPO (Recovery Point Objective) of minutes. The exact RPO depends on the rate of change and network conditions.

2. **Read-Only Replica:** The destination file system is read-only while replication is active. Applications cannot write to it directly.

3. **Automatic Setup:** AWS handles all replication infrastructure — no need to manage data transfer, encryption, or network configuration.

4. **Encryption:** Replication traffic is encrypted in transit. The destination file system can use a different KMS key than the source.

5. **Failover Process (Promotion):**
   - Stop replication on the destination.
   - The destination becomes a standalone read/write file system.
   - Update application mount points to point to the destination.
   - This is a manual process — EFS does not provide automatic failover.

6. **Cost:** You pay for storage in both regions plus data transfer costs for replication traffic.

**Limitations:**
- Only one replication destination per source file system.
- Cannot replicate to an existing file system (destination is always a new file system created by the replication process).
- Failback requires re-establishing replication in the reverse direction after the original source is recovered.
- The replica cannot be used as a read replica for performance scaling — it's purely for DR.
- Replication does not preserve file system policies — these must be re-applied after promotion.
- One Zone file systems can replicate to Standard or One Zone destinations, but cross-AZ replication to Standard is recommended for HA.

**RTO Considerations:**
- RTO depends on how quickly you can update DNS/mount points and restart applications.
- For faster RTO, pre-mount the replica file system on standby instances in the destination region (though it will be read-only until promoted).

---

**Q3. How would you troubleshoot high EFS latency in a production environment? Walk through your methodology.**

**Answer:**
High EFS latency is a common production issue. Here is a systematic troubleshooting methodology:

**Step 1: Establish Baseline and Identify the Type of Latency**

First, determine if the issue is:
- **Metadata latency** (directory listings, file opens, stat calls) — typically more impactful for workloads with many small files
- **Data latency** (actual read/write operations) — typically impacts throughput-heavy workloads

Use the EFS CloudWatch metrics to baseline:
- `TotalIOBytes` — overall throughput
- `DataReadIOBytes` / `DataWriteIOBytes` — read/write breakdown
- `MetadataIOBytes` — metadata operation volume
- `PercentIOLimit` — percentage of General Purpose mode I/O limit used

**Step 2: Check `PercentIOLimit`**
- If consistently above 80-90%, the file system is hitting the General Purpose I/O limit.
- **Resolution:** Switch to Max I/O performance mode (requires creating a new file system and migrating data) or switch to Elastic Throughput mode which may alleviate the issue.

**Step 3: Check `BurstCreditBalance`**
- If using Bursting Throughput mode and `BurstCreditBalance` is near zero, the file system is throttled.
- **Resolution:** Switch to