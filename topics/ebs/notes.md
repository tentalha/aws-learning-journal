# EBS

## What is it?

**Amazon Elastic Block Store (EBS)** is a high-performance, durable, block-level storage service designed for use with Amazon EC2 instances. It falls under the **Storage** category of AWS services.

EBS provides persistent block storage volumes that behave like raw, unformatted block devices. You can mount these volumes as devices on EC2 instances, format them with a filesystem, and use them just like a physical hard drive — except the storage is network-attached, highly available, and exists independently of the EC2 instance lifecycle.

Key identifiers:
- **Service Type:** Block Storage
- **Scope:** Availability Zone (AZ) — a volume exists within a single AZ
- **Access Model:** Attached to a single EC2 instance at a time (except Multi-Attach for `io1`/`io2`)
- **Persistence:** Data persists independently of EC2 instance state (stop, terminate)

---

## Why do we need it?

### The Problem It Solves

EC2 instance store (ephemeral storage) is physically attached to the host machine. When an instance stops, hibernates, or terminates, **all data on instance store is lost**. This is unacceptable for most real-world workloads.

EBS solves this by providing:
- **Durability:** Data survives instance stops, reboots, and failures
- **Flexibility:** Volumes can be detached and reattached to different instances
- **Snapshots:** Point-in-time backups stored in S3 (managed by AWS)
- **Performance control:** Choose IOPS, throughput, and volume type based on workload needs

### When to Use EBS

| Scenario | Reason |
|---|---|
| Relational databases (MySQL, PostgreSQL, Oracle) | Requires low-latency, consistent I/O |
| NoSQL databases (MongoDB, Cassandra) | High IOPS with predictable performance |
| Application servers with stateful data | Data must survive instance restarts |
| Boot volumes for EC2 instances | Persistent OS and application storage |
| Big data analytics (Hadoop, Spark) | Large sequential read/write workloads |
| ERP/CRM systems | Transactional workloads needing durability |

### Real Business Scenarios

1. **E-commerce Platform:** A retail company runs PostgreSQL on EC2. They need the database to survive instance maintenance windows, scale storage independently of compute, and take nightly backups — EBS snapshots handle all of this.

2. **Media Processing Pipeline:** A video encoding company needs burst I/O for short periods when transcoding video. `gp3` volumes provide the right balance of cost and performance.

3. **Financial Trading System:** A trading firm needs sub-millisecond latency and guaranteed IOPS for their order management system. `io2 Block Express` provides up to 256,000 IOPS with 99.999% durability.

---

## Internal Working

### How EBS Works Under the Hood

EBS is a **network-attached storage (NAS) system** built on a highly distributed, redundant infrastructure. Here's what happens at each layer:

#### 1. Physical Infrastructure
- EBS volumes are stored on a fleet of dedicated storage servers separate from EC2 host machines
- Data is replicated **within the same Availability Zone** across multiple storage nodes automatically
- AWS uses a proprietary high-speed, low-latency network fabric (not the public internet) to connect EC2 instances to EBS storage servers

#### 2. Volume Provisioning
When you create an EBS volume:
- AWS allocates storage blocks on their internal storage cluster
- A logical volume identifier (Volume ID) is created
- The volume is registered in the AZ-specific control plane
- No data is pre-written — blocks are zero-initialized on first access (lazy initialization for restored snapshots)

#### 3. Attachment Mechanism
When you attach an EBS volume to an EC2 instance:
- The EC2 hypervisor (Nitro System for modern instances) establishes a **NVMe-over-Fabric** or **virtio-blk** connection
- The OS sees the volume as a block device (e.g., `/dev/xvda`, `/dev/nvme0n1`)
- I/O operations travel over the internal AWS network with single-digit millisecond latency

#### 4. I/O Path
```
Application → OS Filesystem → Block Device Driver → EC2 Hypervisor (Nitro) → AWS Internal Network → EBS Storage Server
```

#### 5. Snapshot Mechanism
- Snapshots are **incremental** and stored in **Amazon S3** (in AWS-managed buckets, not your S3)
- The first snapshot copies all used blocks; subsequent snapshots copy only changed blocks
- Snapshots use a **copy-on-write** mechanism — the original volume data is preserved until the snapshot is finalized
- Restoring a volume from a snapshot uses **lazy loading**: the volume is available immediately, but blocks are pulled from S3 as they're first accessed (background initialization completes over time)

#### 6. Replication
- EBS automatically replicates data **within the AZ** to protect against hardware failure
- This is transparent to the user — no configuration needed
- Replication does NOT span AZs (you must use snapshots or EBS Multi-Attach for cross-AZ strategies)

---

## Architecture

### Core Architectural Components

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS Availability Zone                     │
│                                                             │
│  ┌──────────────┐    NVMe/Network    ┌──────────────────┐  │
│  │  EC2 Instance│◄──────────────────►│   EBS Volume     │  │
│  │  (Nitro)     │                    │  (gp3/io2/st1)   │  │
│  └──────────────┘                    └────────┬─────────┘  │
│                                               │             │
│                                    ┌──────────▼──────────┐ │
│                                    │  EBS Storage Cluster │ │
│                                    │  (Multi-node replica)│ │
│                                    └─────────────────────-┘ │
└─────────────────────────────────────────────────────────────┘
           │
           │ Snapshot (Incremental)
           ▼
┌─────────────────────┐
│   Amazon S3         │
│ (AWS-managed bucket)│
│  Snapshot Storage   │
└─────────────────────┘
           │
           │ Copy Snapshot
           ▼
┌─────────────────────┐
│  Another Region     │
│  (DR Strategy)      │
└─────────────────────┘
```

### Volume Types Architecture

| Volume Type | Use Case | Max IOPS | Max Throughput | Max Size |
|---|---|---|---|---|
| `gp3` | General purpose SSD | 16,000 | 1,000 MB/s | 16 TiB |
| `gp2` | General purpose SSD (legacy) | 16,000 | 250 MB/s | 16 TiB |
| `io2 Block Express` | High-perf databases | 256,000 | 4,000 MB/s | 64 TiB |
| `io1` | Critical workloads | 64,000 | 1,000 MB/s | 16 TiB |
| `st1` | Throughput-optimized HDD | 500 | 500 MB/s | 16 TiB |
| `sc1` | Cold HDD | 250 | 250 MB/s | 16 TiB |
| `standard` | Magnetic (legacy) | 40-200 | 40-90 MB/s | 1 TiB |

### Key Architectural Patterns

#### Pattern 1: Root + Data Volume Separation
```
EC2 Instance
├── /dev/nvme0n1 → gp3 (Root, 30 GiB, OS + App)
└── /dev/nvme1n1 → io2 (Data, 500 GiB, Database)
```
Separating root and data volumes allows independent sizing, snapshot policies, and lifecycle management.

#### Pattern 2: RAID on EBS
- **RAID 0:** Stripe across multiple EBS volumes for higher IOPS/throughput (no redundancy)
- **RAID 1:** Mirror across volumes for redundancy (not typically needed since EBS already replicates)
- AWS recommends RAID 0 for performance, and relying on EBS snapshots for durability

#### Pattern 3: Multi-Attach (io1/io2 only)
- Attach a single `io1` or `io2` volume to up to **16 Nitro-based EC2 instances** in the same AZ
- Requires a cluster-aware filesystem (e.g., GFS2, OCFS2) — standard filesystems like ext4 will cause data corruption
- Used for high-availability applications like Oracle RAC

---

## Real World Example

### Scenario: High-Availability PostgreSQL Database on EC2

**Company:** FinTech startup running a transactional database for payment processing

**Requirements:**
- 10,000+ IOPS with consistent sub-2ms latency
- Daily automated backups with 30-day retention
- Ability to scale storage without downtime
- Encryption at rest for PCI-DSS compliance

#### Step-by-Step Walkthrough

**Step 1: Create the EBS Volume**
```bash
aws ec2 create-volume \
  --volume-type io2 \
  --size 500 \
  --iops 10000 \
  --availability-zone us-east-1a \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=postgres-data},{Key=Environment,Value=production}]'
```

**Step 2: Attach to EC2 Instance**
```bash
aws ec2 attach-volume \
  --volume-id vol-0abc123def456789 \
  --instance-id i-0123456789abcdef0 \
  --device /dev/sdf
```

**Step 3: Format and Mount (on EC2 instance)**
```bash
# Check the device
lsblk

# Format with ext4
sudo mkfs -t ext4 /dev/nvme1n1

# Create mount point
sudo mkdir /data/postgres

# Mount the volume
sudo mount /dev/nvme1n1 /data/postgres

# Persist mount across reboots
echo '/dev/nvme1n1 /data/postgres ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
```

**Step 4: Configure PostgreSQL to use the volume**
```bash
# Set data directory to EBS volume
sudo chown postgres:postgres /data/postgres
sudo -u postgres initdb -D /data/postgres
# Update postgresql.conf: data_directory = '/data/postgres'
```

**Step 5: Set Up Automated Snapshots with Data Lifecycle Manager**
```bash
aws dlm create-lifecycle-policy \
  --description "PostgreSQL daily snapshots" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::123456789012:role/AWSDataLifecycleManagerDefaultRole \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["VOLUME"],
    "TargetTags": [{"Key": "Name", "Value": "postgres-data"}],
    "Schedules": [{
      "Name": "DailySnapshots",
      "CreateRule": {"Interval": 24, "IntervalUnit": "HOURS", "Times": ["03:00"]},
      "RetainRule": {"Count": 30},
      "CopyTags": true
    }]
  }'
```

**Step 6: Modify Volume (scale up without downtime)**
```bash
# Increase size from 500 GiB to 1 TiB
aws ec2 modify-volume \
  --volume-id vol-0abc123def456789 \
  --size 1000 \
  --iops 20000

# After modification completes, resize the filesystem (no unmount needed)
sudo growpart /dev/nvme1n1 1
sudo resize2fs /dev/nvme1n1
```

**Result:** The PostgreSQL database runs with guaranteed 10,000 IOPS, encrypted storage, daily automated backups with 30-day retention, and the ability to scale storage online without any downtime.

---

## Advantages

1. **Persistent Storage:** Data survives EC2 instance stops, reboots, and terminations
2. **High Durability:** Automatically replicated within the AZ; `io2` volumes offer 99.999% durability (5 nines)
3. **Flexible Performance Tuning:** `gp3` allows independent configuration of IOPS and throughput without changing volume size
4. **Elastic Modification:** Increase size, change volume type, or adjust IOPS/throughput **without detaching or stopping** the instance (Elastic Volumes)
5. **Snapshot-Based Backups:** Incremental snapshots stored in S3 with cross-region copy support
6. **Encryption:** AES-256 encryption at rest and in transit with AWS KMS integration — zero performance overhead on Nitro instances
7. **Multi-Attach:** `io1`/`io2` volumes can be attached to up to 16 instances simultaneously for clustered applications
8. **Data Lifecycle Manager (DLM):** Automated snapshot scheduling and retention policies
9. **Fast Snapshot Restore (FSR):** Eliminates latency on first access when restoring from snapshots
10. **NVMe Interface:** Modern instances use NVMe over the Nitro hypervisor, providing very low latency
11. **AWS Backup Integration:** Centralized backup management with compliance and audit reporting
12. **Cost Efficiency:** `gp3` is ~20% cheaper than `gp2` with better baseline performance

---

## Limitations

### Hard Limits

| Constraint | Limit |
|---|---|
| Maximum volume size (`gp3`, `io1`, `io2`) | 16 TiB |
| Maximum volume size (`io2 Block Express`) | 64 TiB |
| Maximum IOPS per volume (`io2 Block Express`) | 256,000 |
| Maximum IOPS per volume (`gp3`, `io1`) | 16,000 / 64,000 |
| Maximum throughput per volume (`io2 Block Express`) | 4,000 MB/s |
| Maximum throughput per volume (`gp3`) | 1,000 MB/s |
| Volumes per EC2 instance (soft limit) | 40 |
| Multi-Attach instances per volume | 16 |
| Snapshots per volume (concurrent) | 5 |
| Default EBS volume limit per region | 5,000 (adjustable) |

### Architectural Limitations

1. **Single AZ Scope:** A volume is tied to one AZ — you cannot directly attach a volume to an instance in a different AZ. Cross-AZ access requires snapshot + restore.
2. **Single Instance Attachment:** Standard volumes can only be attached to one EC2 instance at a time (Multi-Attach is only for `io1`/`io2` with cluster-aware filesystems)
3. **Not a Shared Filesystem:** EBS is not NFS/SMB — for shared file access, use Amazon EFS or FSx
4. **No Direct Internet Access:** EBS volumes must be accessed through an EC2 instance — no direct API access to data
5. **Snapshot Consistency:** Application-consistent snapshots require quiescing the application or using VSS (Windows) — crash-consistent snapshots may not be sufficient for databases
6. **gp2 Burst Credits:** `gp2` volumes under 1 TiB rely on a burst credit model — sustained high IOPS can exhaust credits (use `gp3` instead)
7. **Throughput Limit on `gp2`:** Maximum 250 MB/s regardless of size (vs. 1,000 MB/s on `gp3`)
8. **Elastic Volumes Cooldown:** After a modification, you must wait 6 hours before modifying again
9. **Snapshot Costs Accumulate:** Incremental snapshots still cost money — without lifecycle policies, costs grow indefinitely
10. **Performance Dependency on Instance Type:** Maximum EBS bandwidth is also limited by the EC2 instance type's