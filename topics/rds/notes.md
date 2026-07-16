# RDS

## What is it?

**Amazon Relational Database Service (RDS)** is a fully managed relational database service provided by AWS under the **Database** category. It automates time-consuming administration tasks such as hardware provisioning, database setup, patching, and backups, allowing developers and architects to focus on application development rather than database management.

RDS supports the following database engines:

| Engine | Versions Supported |
|---|---|
| Amazon Aurora (MySQL-compatible) | MySQL 5.7, 8.0 compatible |
| Amazon Aurora (PostgreSQL-compatible) | PostgreSQL 12–16 compatible |
| MySQL | 8.0, 8.4 |
| PostgreSQL | 13, 14, 15, 16 |
| MariaDB | 10.5, 10.6, 10.11 |
| Oracle | 19c, 21c |
| Microsoft SQL Server | 2016, 2017, 2019, 2022 |

RDS is a **Platform as a Service (PaaS)** offering — AWS manages the OS, database engine installation, patching, and hardware, while you manage the schema, data, and application-level configurations.

---

## Why do we need it?

### The Problem It Solves

Managing relational databases on-premises or on raw EC2 instances involves significant operational overhead:

- **Manual patching**: Keeping database engines and OS up-to-date is time-consuming.
- **Backup complexity**: Implementing reliable backup and point-in-time recovery is non-trivial.
- **High availability**: Setting up Multi-AZ replication manually requires deep expertise.
- **Scalability challenges**: Scaling storage or compute on self-managed databases causes downtime.
- **Security hardening**: Configuring encryption, network isolation, and access controls requires specialized knowledge.

### When to Use RDS

| Scenario | Why RDS Fits |
|---|---|
| Existing applications using SQL | Lift-and-shift with minimal code changes |
| OLTP workloads | Optimized for transactional, row-based queries |
| Applications requiring ACID compliance | Full transactional support |
| Teams without DBA expertise | Managed service reduces operational burden |
| Regulated industries (HIPAA, PCI-DSS) | Compliance certifications available |

### Real Business Scenarios

1. **E-commerce Platform**: An online retailer needs a reliable, transactional database for order management. RDS with Multi-AZ provides automatic failover, ensuring orders are never lost during an AZ failure.

2. **SaaS Application**: A B2B SaaS company running a CRM system uses RDS PostgreSQL with Read Replicas to offload reporting queries from the primary instance, maintaining performance for transactional users.

3. **Financial Services**: A fintech startup uses RDS Oracle to migrate their existing Oracle-based core banking system to AWS, maintaining full Oracle compatibility while reducing infrastructure management overhead.

4. **Healthcare Application**: A hospital management system uses RDS MySQL with encryption at rest and in transit to comply with HIPAA requirements for patient data.

---

## Internal Working

### Database Instance Lifecycle

When you create an RDS instance, AWS performs the following behind the scenes:

```
User Request → RDS Control Plane → EC2 Instance Provisioning
                                 → EBS Volume Attachment
                                 → DB Engine Installation & Configuration
                                 → Security Group & VPC Configuration
                                 → Parameter Group Application
                                 → Automated Backup Configuration
                                 → CloudWatch Metrics Agent Setup
```

### Storage Architecture

RDS uses **Amazon EBS (Elastic Block Store)** as the underlying storage layer. There are three storage types:

1. **General Purpose SSD (gp2/gp3)**:
   - `gp2`: Baseline 3 IOPS/GB, burst up to 3,000 IOPS; 20 GB–64 TB
   - `gp3`: Baseline 3,000 IOPS regardless of size; can provision up to 16,000 IOPS independently; more cost-effective

2. **Provisioned IOPS SSD (io1/io2)**:
   - Designed for I/O-intensive workloads
   - Up to 256,000 IOPS (io2 Block Express)
   - IOPS-to-storage ratio: up to 1,000:1

3. **Magnetic (Standard)**:
   - Legacy; not recommended for new deployments
   - Limited to 1,000 IOPS

### Multi-AZ Replication Internals

```
Primary DB Instance (AZ-1)
        │
        │ Synchronous Block-Level Replication
        │ (via Amazon's internal network)
        ▼
Standby DB Instance (AZ-2)
        │
        └── Same EBS volume structure mirrored
```

- Replication happens at the **storage layer** (block-level), not the database layer
- The standby instance is **not accessible** for reads or writes
- Failover is automatic and typically completes in **60–120 seconds**
- DNS CNAME is updated to point to the standby upon failover

### Read Replica Replication Internals

```
Primary DB Instance
        │
        │ Asynchronous Replication (engine-level binlog/WAL)
        ├──────────────────────────────────┐
        ▼                                  ▼
Read Replica 1 (same region)     Read Replica 2 (cross-region)
```

- Uses **native database replication** (MySQL binlog, PostgreSQL WAL streaming)
- Replicas can be **promoted** to standalone instances
- Up to **5 Read Replicas** per source instance (15 for Aurora)
- Cross-region replicas use **encrypted replication over TLS**

### Automated Backup Mechanism

```
Daily Snapshot (during backup window)
        +
Transaction Log Backup (every 5 minutes)
        =
Point-in-Time Recovery (PITR) within retention period (1–35 days)
```

Snapshots are stored in **Amazon S3** (managed by AWS, not visible in your S3 console).

---

## Architecture

### Single-AZ Architecture

```
┌─────────────────────────────────────────────┐
│                   VPC                        │
│  ┌──────────────────────────────────────┐   │
│  │         Private Subnet (AZ-1)         │   │
│  │  ┌────────────────────────────────┐  │   │
│  │  │       RDS Instance              │  │   │
│  │  │  ┌──────────┐ ┌─────────────┐  │  │   │
│  │  │  │ DB Engine│ │  EBS Volume │  │  │   │
│  │  │  └──────────┘ └─────────────┘  │  │   │
│  │  └────────────────────────────────┘  │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### Multi-AZ Architecture (High Availability)

```
┌──────────────────────────────────────────────────────────────────┐
│                              VPC                                  │
│                                                                   │
│  ┌─────────────────────────┐    ┌─────────────────────────────┐  │
│  │   Private Subnet (AZ-1) │    │   Private Subnet (AZ-2)     │  │
│  │                         │    │                             │  │
│  │  ┌───────────────────┐  │    │  ┌───────────────────────┐  │  │
│  │  │  Primary RDS DB   │  │    │  │   Standby RDS DB      │  │  │
│  │  │  (Read/Write)     │◄─┼────┼─►│   (Synchronous Repl.) │  │  │
│  │  └───────────────────┘  │    │  └───────────────────────┘  │  │
│  └─────────────────────────┘    └─────────────────────────────┘  │
│                                                                   │
│  ┌─────────────────────────┐                                     │
│  │   Public/App Subnet     │                                     │
│  │  ┌───────────────────┐  │                                     │
│  │  │   Application     │  │                                     │
│  │  │   Server (EC2)    │  │                                     │
│  │  └─────────┬─────────┘  │                                     │
│  └────────────┼────────────┘                                     │
│               │  DNS CNAME (mydb.cluster.rds.amazonaws.com)      │
│               └──────────────────────────────────────────────►   │
└──────────────────────────────────────────────────────────────────┘
```

### Read Replica Architecture (Read Scaling)

```
                    ┌──────────────────────┐
                    │   Application Layer   │
                    │                       │
                    │  Write Queries ──────►│
                    │  Read Queries  ──────►│
                    └──────┬───────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
   ┌──────────────────┐      ┌──────────────────────┐
   │  Primary RDS DB  │      │   Read Replica 1      │
   │  (Writes + Reads)│─────►│   (Read-Only)         │
   └──────────────────┘      └──────────────────────┘
              │
              │ Async Replication
              ▼
   ┌──────────────────────┐
   │   Read Replica 2     │
   │   (Cross-Region)     │
   └──────────────────────┘
```

### Key Architectural Components

| Component | Description |
|---|---|
| **DB Instance** | The compute layer running the database engine |
| **DB Subnet Group** | A collection of subnets (in multiple AZs) for the DB instance |
| **Parameter Group** | Engine configuration parameters (e.g., `max_connections`, `innodb_buffer_pool_size`) |
| **Option Group** | Additional features/plugins for specific engines (e.g., Oracle APEX, SQL Server TDE) |
| **Security Group** | Controls inbound/outbound traffic to the DB instance |
| **Endpoint** | DNS name used to connect to the database |
| **Maintenance Window** | Scheduled time for minor version upgrades and patches |
| **Backup Window** | Scheduled time for automated snapshot creation |

---

## Real World Example

### Scenario: E-Commerce Order Management System

**Business Context**: A retail company processes 50,000 orders per day with peak traffic during sales events. They need a highly available, scalable database that can handle transactional writes and reporting queries simultaneously.

#### Step 1: Create a DB Subnet Group

```
VPC: vpc-0abc123 (10.0.0.0/16)
Private Subnet 1: subnet-0111 (10.0.1.0/24) - us-east-1a
Private Subnet 2: subnet-0222 (10.0.2.0/24) - us-east-1b
Private Subnet 3: subnet-0333 (10.0.3.0/24) - us-east-1c

DB Subnet Group: ecommerce-db-subnet-group
  → Spans all 3 AZs for maximum availability
```

#### Step 2: Create the Primary RDS Instance (Multi-AZ)

```
Engine: PostgreSQL 16.2
Instance Class: db.r6g.2xlarge (8 vCPU, 64 GB RAM)
Storage: gp3, 500 GB, 6,000 IOPS
Multi-AZ: Enabled
Backup Retention: 7 days
Backup Window: 03:00-04:00 UTC
Maintenance Window: Sunday 04:00-05:00 UTC
Encryption: Enabled (AWS-managed KMS key)
```

#### Step 3: Create Read Replicas for Reporting

```
Read Replica 1: db.r6g.xlarge (us-east-1) → Handles BI/Analytics queries
Read Replica 2: db.r6g.large (us-west-2)  → Disaster recovery + West Coast reads
```

#### Step 4: Application Connection Strategy

```python
# Write connection → Primary endpoint
WRITE_DB_HOST = "ecommerce-db.cluster.us-east-1.rds.amazonaws.com"

# Read connection → Read Replica endpoint
READ_DB_HOST  = "ecommerce-db-replica.us-east-1.rds.amazonaws.com"
```

#### Step 5: Configure Parameter Group

```
max_connections = 500
shared_buffers = 16GB (25% of RAM)
effective_cache_size = 48GB (75% of RAM)
work_mem = 32MB
log_slow_queries = ON
slow_query_log = 1
long_query_time = 2
```

#### Step 6: Set Up Monitoring and Alerts

```
CloudWatch Alarm: CPUUtilization > 80% → SNS notification
CloudWatch Alarm: FreeStorageSpace < 20GB → SNS notification
CloudWatch Alarm: DatabaseConnections > 450 → SNS notification
Performance Insights: Enabled (7-day retention)
Enhanced Monitoring: Enabled (60-second granularity)
```

#### Step 7: Simulate Failover During Maintenance

```bash
# Trigger a manual failover for testing
aws rds reboot-db-instance \
  --db-instance-identifier ecommerce-db \
  --force-failover

# Application experiences ~60-120 second outage
# DNS CNAME automatically updates to standby
# Application reconnects using same endpoint
```

#### Result

- **Availability**: 99.95% SLA with Multi-AZ
- **Read Scalability**: Reporting queries offloaded to Read Replica
- **RTO**: ~2 minutes (automatic failover)
- **RPO**: Near-zero (synchronous replication for Multi-AZ)

---

## Advantages

1. **Fully Managed**: AWS handles OS patching, database engine upgrades, hardware maintenance, and failure detection — reducing DBA overhead by up to 70%.

2. **Automated Backups & PITR**: Automated daily snapshots plus transaction log backups enable point-in-time recovery to any second within the retention period (up to 35 days).

3. **Multi-AZ High Availability**: Synchronous standby replication provides automatic failover with no data loss and minimal downtime (~60–120 seconds).

4. **Read Replicas for Horizontal Read Scaling**: Up to 5 Read Replicas per source instance allow read-heavy workloads to scale horizontally without modifying the primary.

5. **Storage Auto Scaling**: RDS can automatically increase storage capacity when free space is low, up to a configured maximum, with zero downtime.

6. **Security Integration**: Native integration with IAM, VPC, KMS encryption, SSL/TLS in-transit encryption, and AWS Secrets Manager for credential rotation.

7. **Multi-Engine Support**: Supports 6+ database engines, enabling lift-and-shift migrations with minimal application changes.

8. **Performance Insights**: Built-in database performance monitoring tool that identifies slow queries and resource bottlenecks with minimal performance overhead.

9. **Compliance Certifications**: PCI DSS, HIPAA, SOC 1/2/3, ISO 27001, FedRAMP — critical for regulated industries.

10. **Snapshot Sharing**: Manual snapshots can be shared across AWS accounts or copied to other regions for cross-account and cross-region DR strategies.

---

## Limitations

### Hard Limits

| Limitation | Value |
|---|---|
| Max DB instances per region | 40 (default; can request increase) |
| Max storage per instance | 64 TB (gp2/gp3/io1) |
| Max Read Replicas per source | 5 (15 for Aurora) |
| Max backup retention period | 35 days |
| Min backup retention period | 1 day (0 disables automated backups) |
| Max parameter groups per region | 50 |
| Max option groups per region | 20 |
| Max tags per resource | 50 |