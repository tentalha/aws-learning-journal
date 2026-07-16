# EFS

## What is it?

**Amazon Elastic File System (Amazon EFS)** is a fully managed, serverless, elastic Network File System (NFS) service provided by AWS. It falls under the **Storage** category of AWS services.

EFS provides a simple, scalable, fully managed elastic NFS file system for use with AWS Cloud services and on-premises resources. It is built to scale on demand to petabytes without disrupting applications, growing and shrinking automatically as you add and remove files, eliminating the need to provision and manage capacity to accommodate growth.

- **Protocol**: NFSv4.0 and NFSv4.1
- **Storage Classes**: Standard, Standard-Infrequent Access (Standard-IA), One Zone, One Zone-IA
- **Throughput Modes**: Bursting, Provisioned, Elastic
- **Performance Modes**: General Purpose, Max I/O
- **File System Type**: POSIX-compliant shared file system

---

## Why do we need it?

### The Problem It Solves

Traditional block storage (EBS) is attached to a single EC2 instance and cannot be shared across multiple compute instances simultaneously. Object storage (S3) requires application-level code changes and does not support POSIX file system semantics. EFS fills the gap by providing **shared, concurrent file access** with standard file system interfaces.

### Key Problems Addressed

| Problem | How EFS Solves It |
|---|---|
| Shared file access across EC2 instances | Multiple instances mount the same file system simultaneously |
| Unpredictable storage capacity needs | Automatically grows/shrinks without pre-provisioning |
| Complex NFS infrastructure management | Fully managed — no servers to provision or patch |
| Cross-AZ shared storage | Data is replicated across multiple AZs automatically |
| On-premises to cloud file sharing | Accessible via AWS Direct Connect and VPN |

### Real Business Scenarios

1. **Content Management Systems (CMS)**: WordPress or Drupal installations running across multiple EC2 instances need shared access to media uploads, themes, and plugins.
2. **Big Data and Analytics**: Spark or Hadoop clusters where multiple worker nodes need to read/write to a shared dataset simultaneously.
3. **CI/CD Pipelines**: Build servers that need shared access to source code repositories and build artifacts.
4. **Home Directories**: Centralized user home directories accessible from any instance in a fleet.
5. **Lift-and-Shift Applications**: Legacy applications that depend on NFS-based shared storage being migrated to AWS.
6. **Machine Learning Training**: Distributed ML training jobs where multiple GPU instances access the same large dataset.

---

## Internal Working

### Storage Architecture

EFS is built on a **distributed storage architecture** that spans multiple Availability Zones. Internally, AWS manages a fleet of storage servers and metadata servers that work together to provide a consistent, highly available file system.

```
┌─────────────────────────────────────────────────────────────┐
│                        EFS File System                       │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Metadata   │  │   Metadata   │  │   Metadata   │      │
│  │   Server 1   │  │   Server 2   │  │   Server N   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│          │                │                  │              │
│  ┌───────▼────────────────▼──────────────────▼───────┐     │
│  │              Distributed Data Store                │     │
│  │         (Replicated across multiple AZs)           │     │
│  └────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### How Data is Stored and Retrieved

1. **Metadata Management**: EFS maintains a separate, highly available metadata layer that tracks file names, directories, permissions, and data block locations. Metadata operations (stat, open, close, rename) are handled by the metadata servers.

2. **Data Striping**: File data is striped across multiple storage nodes within the EFS infrastructure. This allows parallel I/O operations and contributes to high throughput.

3. **Replication**: For Standard storage class, data is automatically replicated across **at least three Availability Zones** within a region. This happens transparently without any configuration.

4. **NFS Protocol Handling**: When an EC2 instance mounts EFS, the NFS client on the instance communicates with the EFS **Mount Target** (an ENI in your VPC). The mount target acts as a network endpoint that routes NFS traffic to the underlying EFS infrastructure.

5. **Consistency Model**: EFS provides **close-to-open consistency semantics** — when a file is closed, subsequent opens on other clients will see the latest data. For concurrent writes, EFS uses file locking (NLM/NFSv4 locking).

### Throughput Mechanics

- **Bursting Throughput**: Based on a credit system. You earn credits when throughput is below the baseline (50 KB/s per GB of Standard storage) and consume credits during bursts (up to 100 MB/s for file systems under 1 TB, or 100 MB/s per TB for larger systems).
- **Provisioned Throughput**: Decouple throughput from storage size. You specify throughput in MiB/s regardless of storage amount.
- **Elastic Throughput**: Automatically scales throughput up and down based on workload needs. AWS manages the scaling transparently.

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                              VPC                                     │
│                                                                      │
│  ┌─────────────────┐    ┌─────────────────┐   ┌─────────────────┐  │
│  │   Subnet AZ-A   │    │   Subnet AZ-B   │   │   Subnet AZ-C   │  │
│  │                 │    │                 │   │                 │  │
│  │  ┌───────────┐  │    │  ┌───────────┐  │   │  ┌───────────┐  │  │
│  │  │EC2 Instance│  │    │  │EC2 Instance│  │   │  │EC2 Instance│  │  │
│  │  │ (NFS Client│  │    │  │ (NFS Client│  │   │  │ (NFS Client│  │  │
│  │  └─────┬─────┘  │    │  └─────┬─────┘  │   │  └─────┬─────┘  │  │
│  │        │        │    │        │        │   │        │        │  │
│  │  ┌─────▼─────┐  │    │  ┌─────▼─────┐  │   │  ┌─────▼─────┐  │  │
│  │  │   Mount   │  │    │  │   Mount   │  │   │  │   Mount   │  │  │
│  │  │  Target   │  │    │  │  Target   │  │   │  │  Target   │  │  │
│  │  │  (ENI)    │  │    │  │  (ENI)    │  │   │  │  (ENI)    │  │  │
│  │  └───────────┘  │    │  └───────────┘  │   │  └───────────┘  │  │
│  └─────────────────┘    └─────────────────┘   └─────────────────┘  │
│                                                                      │
│                    ┌──────────────────────┐                         │
│                    │    EFS File System    │                         │
│                    │  (Managed by AWS)    │                         │
│                    └──────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

#### 1. EFS File System
The logical entity you create. It has a unique **File System ID** (e.g., `fs-0123456789abcdef0`) and DNS name (e.g., `fs-0123456789abcdef0.efs.us-east-1.amazonaws.com`).

#### 2. Mount Targets
- An **Elastic Network Interface (ENI)** deployed in a specific subnet within your VPC
- Each mount target has an IP address and DNS name
- You should create one mount target **per Availability Zone** for high availability
- Mount targets are associated with **Security Groups** that control NFS traffic (port 2049)

#### 3. Access Points
- **Application-specific entry points** into the EFS file system
- Enforce a specific POSIX user identity (UID/GID) for all file operations
- Enforce a specific root directory for the application
- Simplify IAM-based access control
- Useful for containerized workloads (ECS, EKS)

#### 4. Storage Classes

| Class | Use Case | Durability |
|---|---|---|
| Standard | Frequently accessed files | Multi-AZ |
| Standard-IA | Infrequently accessed files (cost savings) | Multi-AZ |
| One Zone | Frequently accessed, single AZ | Single AZ |
| One Zone-IA | Infrequently accessed, single AZ | Single AZ |

#### 5. Lifecycle Management
Automatically moves files between storage classes based on access patterns. Configurable thresholds: 7, 14, 30, 60, or 90 days of inactivity.

### EFS with ECS/EKS Architecture

```
┌─────────────────────────────────────────────────┐
│                  ECS Cluster                     │
│                                                  │
│  ┌──────────────┐      ┌──────────────┐         │
│  │   Task 1     │      │   Task 2     │         │
│  │  Container   │      │  Container   │         │
│  │  /mnt/efs ──────────────/mnt/efs   │         │
│  └──────┬───────┘      └──────┬───────┘         │
│         │                     │                  │
│  ┌──────▼─────────────────────▼───────┐         │
│  │         EFS Volume Driver          │         │
│  └──────────────────┬─────────────────┘         │
└─────────────────────┼───────────────────────────┘
                       │ NFS Mount
                  ┌────▼─────┐
                  │   EFS    │
                  │  Mount   │
                  │  Target  │
                  └────┬─────┘
                       │
                  ┌────▼─────┐
                  │   EFS    │
                  │   File   │
                  │  System  │
                  └──────────┘
```

---

## Real World Example

### Scenario: Scalable WordPress Deployment with Shared Media Storage

A media company runs a high-traffic WordPress site. They need to scale horizontally (multiple EC2 instances behind a load balancer) while ensuring all instances share the same `wp-content/uploads` directory.

#### Step-by-Step Walkthrough

**Step 1: Create the EFS File System**
```bash
aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted \
  --tags Key=Name,Value=wordpress-shared-storage \
  --region us-east-1
```

**Step 2: Create Mount Targets in Each AZ**
```bash
# For each subnet in each AZ
aws efs create-mount-target \
  --file-system-id fs-0123456789abcdef0 \
  --subnet-id subnet-abc123 \
  --security-groups sg-efs-mount-target

aws efs create-mount-target \
  --file-system-id fs-0123456789abcdef0 \
  --subnet-id subnet-def456 \
  --security-groups sg-efs-mount-target
```

**Step 3: Create an EFS Access Point for WordPress**
```bash
aws efs create-access-point \
  --file-system-id fs-0123456789abcdef0 \
  --posix-user Uid=33,Gid=33 \
  --root-directory "Path=/wordpress/uploads,CreationInfo={OwnerUid=33,OwnerGid=33,Permissions=755}" \
  --tags Key=Name,Value=wordpress-uploads-ap
```

**Step 4: Configure Security Groups**
- **EFS Security Group**: Allow inbound TCP port 2049 from the EC2 security group
- **EC2 Security Group**: Allow outbound TCP port 2049 to the EFS security group

**Step 5: Mount EFS on EC2 Instances (User Data Script)**
```bash
#!/bin/bash
# Install EFS mount helper
yum install -y amazon-efs-utils

# Create mount point
mkdir -p /var/www/html/wp-content/uploads

# Mount using EFS Access Point with TLS encryption
mount -t efs -o tls,accesspoint=fsap-0abc123def456 \
  fs-0123456789abcdef0:/ \
  /var/www/html/wp-content/uploads

# Add to fstab for persistence
echo "fs-0123456789abcdef0:/ /var/www/html/wp-content/uploads efs _netdev,tls,accesspoint=fsap-0abc123def456 0 0" >> /etc/fstab
```

**Step 6: Configure Lifecycle Policy**
```bash
aws efs put-lifecycle-configuration \
  --file-system-id fs-0123456789abcdef0 \
  --lifecycle-policies TransitionToIA=AFTER_30_DAYS,TransitionToPrimaryStorageClass=AFTER_1_ACCESS
```

**Step 7: Set Up Auto Scaling Group**
- Launch Template includes the User Data script above
- ASG scales EC2 instances based on CPU/request metrics
- All new instances automatically mount the same EFS file system

**Result**: Any media uploaded through any WordPress instance is immediately available to all other instances. The file system scales automatically from gigabytes to petabytes without any intervention.

---

## Advantages

1. **Fully Managed**: No infrastructure to provision, patch, or maintain. AWS handles all server management, patching, and hardware replacement.

2. **Elastic Scalability**: Automatically grows and shrinks as files are added or removed. No need to pre-allocate storage capacity.

3. **Shared Access**: Supports **thousands of concurrent NFS connections** from EC2 instances, ECS tasks, EKS pods, and Lambda functions simultaneously.

4. **High Availability and Durability**: Standard storage class replicates data across **3+ AZs** within a region, providing 99.999999999% (11 nines) durability.

5. **POSIX-Compliant**: Supports standard file system semantics including file locking, permissions, and ownership — making it compatible with virtually any Linux application.

6. **Multiple Throughput Modes**: Elastic throughput mode automatically scales to meet workload demands without manual intervention.

7. **Encryption**: Supports encryption at rest (KMS) and in transit (TLS) without application changes.

8. **Cross-Region Replication**: Built-in replication to another AWS region for disaster recovery.

9. **Lifecycle Management**: Automatically moves infrequently accessed files to cheaper storage classes (Standard-IA, One Zone-IA), reducing costs by up to 92%.

10. **IAM Integration**: Fine-grained access control using IAM policies and EFS Access Points.

11. **On-Premises Access**: Accessible from on-premises environments via AWS Direct Connect or VPN.

12. **Serverless Compatibility**: Can be mounted by AWS Lambda functions for stateful serverless workloads.

---

## Limitations

### Hard Limits

| Limit | Value |
|---|---|
| File system size | Unlimited (petabyte-scale) |
| Maximum file size | 52 TB |
| Maximum directory depth | 1000 levels |
| Maximum number of hard links | 177 |
| Maximum number of file systems per account per region | 1,000 (