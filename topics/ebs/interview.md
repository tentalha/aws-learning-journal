# EBS — Interview Questions

---

## Easy

### Q1. What is Amazon EBS and what problem does it solve?

**Answer:**
Amazon Elastic Block Store (EBS) is a high-performance, durable block storage service designed for use with Amazon EC2 instances. It provides persistent storage that exists independently of the EC2 instance lifecycle — meaning data persists even after the instance is stopped or terminated (unless the volume is configured to delete on termination).

EBS solves the problem of ephemeral storage on EC2. Without EBS, data stored on an instance store (local disk) is lost when the instance stops or fails. EBS volumes act like virtual hard drives that can be attached to EC2 instances, providing reliable, network-attached block storage for databases, file systems, and application data.

---

### Q2. What are the main EBS volume types and their use cases?

**Answer:**
EBS offers the following volume types:

| Volume Type | Category | Use Case |
|---|---|---|
| `gp3` | General Purpose SSD | Default for most workloads; cost-effective, predictable performance |
| `gp2` | General Purpose SSD | Legacy general-purpose; performance tied to volume size |
| `io2 Block Express` | Provisioned IOPS SSD | Highest performance; mission-critical DBs (Oracle, SQL Server) |
| `io1` | Provisioned IOPS SSD | Latency-sensitive transactional workloads |
| `st1` | Throughput Optimized HDD | Big data, data warehouses, log processing |
| `sc1` | Cold HDD | Infrequently accessed data; lowest cost |
| `Magnetic (standard)` | Legacy HDD | Legacy workloads; not recommended for new deployments |

**Key distinction:** SSD-backed volumes (`gp2`, `gp3`, `io1`, `io2`) are optimized for IOPS-intensive workloads. HDD-backed volumes (`st1`, `sc1`) are optimized for throughput-intensive, sequential workloads.

---

### Q3. What is an EBS Snapshot and how does it work?

**Answer:**
An EBS Snapshot is a point-in-time backup of an EBS volume stored in Amazon S3 (managed by AWS; you do not access the S3 bucket directly). Snapshots are **incremental** — the first snapshot copies all data, and subsequent snapshots only capture blocks that have changed since the last snapshot. This makes them storage-efficient and cost-effective.

Key characteristics:
- Snapshots are stored in S3 and are **region-specific** by default but can be copied to other regions.
- A new EBS volume can be created from a snapshot at any time.
- Snapshots can be shared with other AWS accounts or made public.
- Despite being incremental, each snapshot is a full restore point — you do not need all previous snapshots to restore.
- Snapshots can be automated using **Amazon Data Lifecycle Manager (DLM)** or **AWS Backup**.

---

### Q4. What is the difference between EBS and Instance Store?

**Answer:**

| Feature | EBS | Instance Store |
|---|---|---|
| **Persistence** | Persistent; survives stop/start | Ephemeral; lost on stop/terminate |
| **Performance** | Network-attached; slight latency | Physically attached; very low latency |
| **Max IOPS** | Up to 256,000 IOPS (io2 BE) | Millions of IOPS (NVMe SSDs) |
| **Cost** | Billed separately per GB/IOPS | Included in instance cost |
| **Snapshots** | Supported | Not supported |
| **Resize** | Can be resized dynamically | Fixed; cannot resize |
| **Use case** | Databases, OS volumes, persistent data | Caches, temp files, buffers |

**Bottom line:** Use EBS when data persistence is required. Use Instance Store for temporary, high-performance scratch space where data loss is acceptable.

---

### Q5. Can you attach one EBS volume to multiple EC2 instances simultaneously?

**Answer:**
Yes, but with important caveats. **EBS Multi-Attach** allows a single Provisioned IOPS SSD (`io1` or `io2`) volume to be attached to **up to 16 EC2 instances** simultaneously within the **same Availability Zone**.

Requirements and limitations:
- Only supported on `io1` and `io2` volume types.
- All instances and the volume must be in the **same AZ**.
- Each attached instance has full read/write access to the volume.
- The application or file system must be **cluster-aware** to manage concurrent writes and prevent data corruption (e.g., clustered file systems like GFS2, OCFS2, or applications that manage their own locking).
- **Standard file systems (ext4, XFS) are NOT safe** with Multi-Attach without a cluster-aware layer.
- Not supported on Nitro-based instances only — it requires Nitro instances.

---

## Medium

### Q1. Explain the difference between `gp2` and `gp3` EBS volumes. When would you choose one over the other?

**Answer:**
Both `gp2` and `gp3` are General Purpose SSD volumes, but they differ significantly in how performance scales and how they are priced.

**`gp2` (Legacy):**
- Baseline performance of **3 IOPS per GB**, with a minimum of 100 IOPS.
- Maximum of **16,000 IOPS** at 5,334 GB.
- Uses a **burst credit bucket** model — smaller volumes (<1 TB) can burst to 3,000 IOPS for short periods.
- Throughput scales with IOPS, maxing at **250 MiB/s**.
- Performance is **tightly coupled to volume size** — to get more IOPS, you must increase the volume size.

**`gp3` (Recommended):**
- Baseline of **3,000 IOPS and 125 MiB/s** regardless of volume size.
- IOPS and throughput are **independently configurable** — you can provision up to 16,000 IOPS and 1,000 MiB/s without changing the volume size.
- Approximately **20% cheaper** per GB than `gp2`.
- No burst credit model — performance is consistent.

**When to choose:**
- **`gp3`** is the default choice for virtually all new workloads. It is cheaper, more predictable, and more flexible.
- **`gp2`** might still be encountered in existing environments or legacy AMIs. There is rarely a reason to choose `gp2` for new deployments.
- **Migration:** AWS provides tools to modify existing `gp2` volumes to `gp3` with zero downtime using Elastic Volumes.

**Example scenario:** A 100 GB `gp2` volume gets only 300 IOPS. The same 100 GB `gp3` volume gets 3,000 IOPS by default at a lower cost — a clear win for `gp3`.

---

### Q2. What is EBS Encryption and how does it work under the hood?

**Answer:**
EBS Encryption provides data-at-rest and data-in-transit (between EC2 and EBS) encryption using **AES-256** encryption. It is transparent to the operating system and application.

**How it works:**
1. When you create an encrypted EBS volume, AWS generates a **Data Encryption Key (DEK)** using **AWS KMS (Key Management Service)**.
2. The DEK is encrypted using a **Customer Master Key (CMK)** — either the AWS-managed key (`aws/ebs`) or a customer-managed CMK.
3. The encrypted DEK is stored with the volume metadata.
4. When an EC2 instance accesses the volume, the **Nitro card** (on Nitro instances) or the hypervisor requests KMS to decrypt the DEK using the CMK.
5. The decrypted DEK lives only in memory on the host hardware — it is never stored unencrypted on disk.
6. All I/O between the EC2 instance and the EBS volume is encrypted in transit using the DEK.

**Key points:**
- Snapshots of encrypted volumes are automatically encrypted.
- Volumes created from encrypted snapshots are automatically encrypted.
- You can enable **account-level encryption by default** so all new EBS volumes are encrypted automatically.
- Encryption has **negligible performance impact** on Nitro-based instances because encryption is handled by dedicated hardware.
- You **cannot directly encrypt an existing unencrypted volume**. The workaround is: snapshot → copy snapshot with encryption enabled → create new encrypted volume from the encrypted snapshot.

---

### Q3. What is EBS Elastic Volumes and what operations does it support?

**Answer:**
**Elastic Volumes** is a feature that allows you to dynamically modify EBS volume configuration **without detaching the volume or stopping the EC2 instance**. This enables zero-downtime storage changes.

**Supported modifications:**
1. **Increase volume size** — You can only increase size, never decrease it. After the API call, you must extend the OS-level file system to use the new space (e.g., `resize2fs` for ext4, `xfs_growfs` for XFS).
2. **Change volume type** — e.g., migrate from `gp2` to `gp3`, or from `gp3` to `io2`.
3. **Adjust provisioned IOPS** — Increase or decrease IOPS on `io1`, `io2`, or `gp3` volumes.
4. **Adjust throughput** — Increase or decrease throughput on `gp3` volumes.

**Constraints:**
- After a modification, you must wait **6 hours** before making another modification to the same volume.
- During modification, the volume goes through states: `modifying` → `optimizing` → `completed`. The volume is usable throughout.
- Supported on most current-generation instance types. Older instances may require a reboot.
- File system extension is a **manual OS-level step** — AWS resizes the block device but not the file system.

**Workflow example:**
```bash
# 1. Modify volume via AWS CLI
aws ec2 modify-volume --volume-id vol-xxxxxxxx --size 200 --volume-type gp3 --iops 6000

# 2. Check modification status
aws ec2 describe-volumes-modifications --volume-id vol-xxxxxxxx

# 3. Extend the partition (if needed)
sudo growpart /dev/xvda 1

# 4. Extend the file system
sudo resize2fs /dev/xvda1  # ext4
# OR
sudo xfs_growfs /         # XFS
```

---

### Q4. Explain EBS volume performance metrics and how you would troubleshoot high latency on an EBS volume.

**Answer:**
**Key EBS Performance Metrics (via CloudWatch):**

| Metric | Description |
|---|---|
| `VolumeReadOps` / `VolumeWriteOps` | Total I/O operations |
| `VolumeReadBytes` / `VolumeWriteBytes` | Throughput |
| `VolumeTotalReadTime` / `VolumeTotalWriteTime` | Total time for I/O operations |
| `VolumeIdleTime` | Time volume has no pending I/O |
| `VolumeQueueLength` | Number of I/O requests waiting |
| `BurstBalance` | Remaining burst credits (gp2, st1, sc1) |

**Derived metric:**
- **Average Latency** = `VolumeTotalReadTime / VolumeReadOps` (or Write equivalent)

**Troubleshooting high EBS latency — systematic approach:**

1. **Check VolumeQueueLength:**
   - A consistently high queue length (>1 for HDD, >0 for SSD) indicates the volume cannot keep up with I/O demand.
   - This suggests IOPS or throughput exhaustion.

2. **Check if IOPS limit is being hit:**
   - Compare `VolumeReadOps + VolumeWriteOps` against the provisioned IOPS limit.
   - If at the limit, consider upgrading to `io1`/`io2` or increasing provisioned IOPS on `gp3`.

3. **Check BurstBalance (for gp2):**
   - If `BurstBalance` is depleted, the volume is throttled to its baseline IOPS (3 IOPS/GB).
   - Solution: Migrate to `gp3` which has no burst model, or increase volume size.

4. **Check EC2 instance limits:**
   - Each EC2 instance type has a maximum EBS bandwidth and IOPS limit (`EBSIOBalance%`, `EBSThroughputBalance%` metrics for burstable instances).
   - The instance itself may be the bottleneck, not the volume.

5. **Check for I/O type mismatch:**
   - HDD volumes (`st1`, `sc1`) are optimized for large, sequential I/O. Small, random I/O on these volumes will have very high latency.

6. **Check for noisy neighbor effects:**
   - While rare with EBS's isolated architecture, extremely heavy I/O on a shared host can occasionally cause transient latency spikes.

7. **Application-level checks:**
   - Verify the application is using appropriate I/O patterns (e.g., buffered writes, read-ahead).
   - Check if the OS has read-ahead configured appropriately.

---

### Q5. What is Amazon Data Lifecycle Manager (DLM) and how does it help manage EBS snapshots?

**Answer:**
**Amazon Data Lifecycle Manager (DLM)** is a fully managed service that automates the creation, retention, and deletion of EBS snapshots (and EBS-backed AMIs). It eliminates the need for custom Lambda functions or cron jobs for snapshot management.

**Core concepts:**

- **Policy:** A lifecycle policy defines what to back up, when, and how long to retain it.
- **Target resources:** Volumes are targeted using **resource tags** (e.g., `Backup: true`).
- **Schedule:** Defines frequency (hourly, daily, weekly, monthly, yearly) and retention count or age.
- **Retention rules:** Keep the last N snapshots or snapshots newer than N days.

**Policy types:**
1. **EBS Snapshot Policy** — Automates snapshots of individual EBS volumes.
2. **EBS-backed AMI Policy** — Automates AMI creation and deregistration.
3. **Cross-account copy event policy** — Copies snapshots shared from another account.

**Key features:**
- **Fast Snapshot Restore (FSR):** DLM can enable FSR on snapshots to eliminate the performance penalty of restoring volumes from snapshots.
- **Cross-region copy:** Automatically copy snapshots to other regions for disaster recovery.
- **Cross-account sharing:** Share snapshots with specific AWS accounts.
- **Snapshot archiving:** Automatically archive older snapshots to reduce costs.

**Example policy (conceptual):**
```
Policy:
  Target: Volumes tagged with "Environment: Production"
  Schedule: Daily at 02:00 UTC
  Retention: Keep last 30 snapshots
  Cross-region copy: Copy to us-west-2, retain for 7 days
```

**Cost consideration:** DLM itself has no additional charge — you pay only for the snapshot storage in S3.

**Comparison with AWS Backup:** AWS Backup is a more centralized service that manages backups across multiple AWS services (EBS, RDS, DynamoDB, EFS, etc.). DLM is EBS/AMI-specific but offers more granular EBS-specific features like FSR enablement.

---

## Hard

### Q1. Deep dive: How does EBS achieve durability and what is the replication mechanism?

**Answer:**
EBS is designed for **99.999% availability** and durability, achieved through several layers of redundancy and architecture decisions.

**Physical replication:**
- Every EBS volume is **synchronously replicated** across multiple physical storage servers within a single **Availability Zone**.
- This replication is transparent to the user and happens at the storage layer, not the application layer.
- The replication is **synchronous** — a write is not acknowledged to the EC2 instance until it has been written to multiple physical locations.
- This protects against individual disk failures and even server failures within an AZ.

**Important limitation:**
- EBS replication is **within a single AZ only**. An entire AZ failure would make the volume unavailable.
- For cross-AZ or cross-region durability, you must use **EBS Snapshots** (to S3, which is inherently multi-AZ) or **cross-region snapshot copies**.

**The EBS storage architecture:**
- EBS volumes are stored on a fleet of storage servers that are separate from EC2 compute hosts.
- The EC2 instance communicates with EBS storage over a **high-bandwidth, low-latency network