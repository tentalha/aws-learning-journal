# RDS — Interview Questions

---

## Easy

### 1. What is Amazon RDS and what problem does it solve?

**Answer:**
Amazon Relational Database Service (RDS) is a managed database service that makes it easier to set up, operate, and scale relational databases in the cloud. It handles routine database tasks such as provisioning, patching, backup, recovery, failure detection, and repair — freeing developers to focus on application development rather than database administration. RDS supports multiple database engines: MySQL, PostgreSQL, MariaDB, Oracle, Microsoft SQL Server, and Amazon Aurora.

---

### 2. What are the database engines supported by Amazon RDS?

**Answer:**
Amazon RDS supports the following database engines:
- **Amazon Aurora** (MySQL-compatible and PostgreSQL-compatible)
- **MySQL**
- **PostgreSQL**
- **MariaDB**
- **Oracle Database**
- **Microsoft SQL Server**

Each engine has multiple version options, and AWS manages patching and minor version upgrades (optionally).

---

### 3. What is a Multi-AZ deployment in RDS and why would you use it?

**Answer:**
A Multi-AZ deployment automatically provisions and maintains a **synchronous standby replica** of your RDS instance in a different Availability Zone. In the event of a planned or unplanned outage (hardware failure, AZ disruption, DB instance failure), RDS automatically fails over to the standby replica with minimal downtime — typically 60–120 seconds.

**Key reasons to use it:**
- High availability and fault tolerance
- Automatic failover without manual intervention
- The standby is not used for read traffic (it is purely for HA)
- Reduces impact of maintenance windows

---

### 4. What is the difference between a Read Replica and a Multi-AZ standby in RDS?

**Answer:**

| Feature | Read Replica | Multi-AZ Standby |
|---|---|---|
| **Purpose** | Read scaling | High availability |
| **Replication** | Asynchronous | Synchronous |
| **Readable** | Yes | No |
| **Failover target** | Manual promotion | Automatic failover |
| **Cross-region** | Yes | Yes (with Multi-AZ clusters) |
| **Separate endpoint** | Yes | No (same endpoint, auto-switched) |

Read Replicas offload read traffic from the primary; the standby in Multi-AZ is a silent failover target.

---

### 5. What is an RDS snapshot and how does it differ from automated backups?

**Answer:**

| Feature | Automated Backups | Manual Snapshots |
|---|---|---|
| **Trigger** | Automatic, daily | Manual or via API/CLI |
| **Retention** | 0–35 days | Retained until deleted |
| **Point-in-time recovery** | Yes (up to the second) | No (only to snapshot time) |
| **Deleted with instance** | Yes (if retention > 0, final snapshot optional) | No |
| **Storage cost** | Free up to DB size | Standard S3 storage rates |

Automated backups enable point-in-time recovery by combining daily snapshots with transaction logs. Manual snapshots are persistent and useful for long-term retention or before major changes.

---

## Medium

### 1. Explain RDS storage types and when you would choose each one.

**Answer:**
RDS offers three storage types backed by Amazon EBS:

**1. General Purpose SSD (gp2/gp3)**
- `gp2`: Baseline 3 IOPS/GB, bursts up to 3,000 IOPS. Good for dev/test and small-to-medium workloads.
- `gp3`: Decoupled IOPS and throughput from storage size. Provides 3,000 IOPS baseline with up to 16,000 IOPS and 1,000 MB/s throughput independently. More cost-effective than `gp2` for most workloads.
- **Use when:** General-purpose databases, cost-sensitive environments, workloads that don't need sustained high IOPS.

**2. Provisioned IOPS SSD (io1/io2)**
- Designed for I/O-intensive workloads requiring consistent, low-latency performance.
- Up to 256,000 IOPS (with io2 Block Express) and 4,000 MB/s throughput.
- IOPS-to-storage ratio: up to 500:1 for io2.
- **Use when:** Production OLTP databases, latency-sensitive applications, high-throughput workloads.

**3. Magnetic (Standard)**
- Legacy storage type, not recommended for new workloads.
- Lower performance, no burst capability.
- **Use when:** Rarely — only for backward compatibility or very low-cost, infrequently accessed data.

**Recommendation:** Use `gp3` as the default for most workloads and `io1/io2` for high-performance production databases.

---

### 2. How does RDS encryption work, and what are its limitations?

**Answer:**
**RDS Encryption at Rest:**
- Uses AWS Key Management Service (KMS) with AES-256 encryption.
- Encrypts the DB instance, automated backups, Read Replicas, and snapshots.
- Encryption must be enabled **at creation time** — you cannot encrypt an existing unencrypted instance directly.
- **Workaround to encrypt an unencrypted instance:** Take a snapshot → copy the snapshot with encryption enabled → restore from the encrypted snapshot.

**RDS Encryption in Transit:**
- Uses SSL/TLS certificates. You can enforce SSL connections via parameter groups (e.g., `rds.force_ssl=1` for PostgreSQL/SQL Server).

**Key Limitations:**
1. You cannot disable encryption once enabled on an instance.
2. You cannot create an unencrypted Read Replica from an encrypted instance.
3. Cross-account snapshot sharing with encrypted snapshots requires sharing the KMS key.
4. The same KMS key must be in the same region; for cross-region copies, a different KMS key in the destination region is used.
5. Performance impact is minimal (hardware-accelerated AES).

---

### 3. Describe RDS parameter groups and option groups. How are they different?

**Answer:**

**Parameter Groups:**
- A collection of database engine configuration parameters (e.g., `max_connections`, `innodb_buffer_pool_size`, `query_cache_size`).
- **Static parameters** require a DB instance reboot to apply.
- **Dynamic parameters** apply immediately without a reboot.
- Each DB instance is associated with exactly one parameter group per engine.
- A default parameter group is created per engine/version; it cannot be modified. You must create a custom parameter group to make changes.

**Option Groups:**
- Enable optional, additional features for database engines.
- Examples: Oracle APEX, SQL Server Transparent Data Encryption (TDE), MySQL memcached support, Oracle Statspack.
- Not all engines support option groups in the same way — Aurora, for instance, doesn't use option groups.
- Options can be **persistent** (remain after removal from option group) or **permanent** (cannot be removed once added).

**Key Difference:**
- Parameter groups control **engine-level configuration settings**.
- Option groups control **optional feature plugins/modules** that extend engine functionality.

---

### 4. What is RDS Proxy and when should you use it?

**Answer:**
**RDS Proxy** is a fully managed, highly available database proxy that sits between your application and RDS/Aurora. It pools and shares database connections.

**How it works:**
- Maintains a warm pool of connections to the database.
- Applications connect to the proxy endpoint rather than directly to the DB.
- The proxy multiplexes many application connections into fewer database connections.

**When to use it:**

1. **Lambda functions:** Lambda can create thousands of concurrent connections during scaling spikes. RDS Proxy prevents connection exhaustion by pooling connections.
2. **Microservices with many short-lived connections:** Reduces connection overhead and database load.
3. **Failover improvement:** RDS Proxy reduces failover time by up to 66% because it maintains the connection pool and routes traffic to the new primary automatically.
4. **IAM authentication:** Proxy enforces IAM authentication, improving security posture.
5. **Secrets Manager integration:** Centralized credential management without application changes.

**Limitations:**
- Adds a small latency overhead (~1ms).
- Not suitable for long-running transactions that hold connections.
- Supported engines: MySQL, PostgreSQL, MariaDB, Aurora MySQL, Aurora PostgreSQL, SQL Server.

---

### 5. How does RDS handle maintenance windows and patching? What are the implications for production systems?

**Answer:**
**Maintenance Windows:**
- A weekly 30-minute window during which AWS may apply patches, updates, or modifications.
- You define the window (e.g., `sun:05:00-sun:05:30`), or AWS assigns one randomly.
- Types of maintenance: OS patching, DB engine minor version upgrades, hardware maintenance.

**Patching Behavior:**

| Deployment Type | Behavior During Patching |
|---|---|
| Single-AZ | Brief outage during patching |
| Multi-AZ | Standby is patched first, then failover occurs, then primary is patched. Minimal downtime (~60 seconds). |
| Aurora | Rolling updates with no downtime in most cases |

**Production Implications:**
1. **Schedule carefully:** Choose a low-traffic window (e.g., Sunday 3–4 AM local time).
2. **Pending modifications:** Some changes are applied during the maintenance window unless "Apply Immediately" is selected.
3. **Auto Minor Version Upgrade:** If enabled, AWS automatically applies minor engine upgrades. Disable this in production to control upgrade timing.
4. **Required maintenance:** Some patches (especially OS-level security patches) are mandatory and cannot be deferred indefinitely.
5. **Testing:** Always test patches in staging before they hit production.
6. **Monitoring:** Set CloudWatch alarms and subscribe to RDS Event Notifications for maintenance events.

---

## Hard

### 1. Deep dive: How does Aurora's storage architecture differ fundamentally from standard RDS, and what performance and durability advantages does this provide?

**Answer:**
**Standard RDS Storage Architecture:**
- Uses EBS volumes attached to the EC2 instance running the DB engine.
- Replication (in Multi-AZ) is done at the storage/block level via synchronous mirroring to a standby EBS volume in another AZ.
- Writes must complete on both primary and standby EBS volumes before acknowledging to the application.
- Read Replicas use asynchronous binlog/WAL replication, introducing replication lag.

**Aurora's Distributed Storage Architecture:**
Aurora fundamentally separates compute (DB engine) from storage:

1. **Distributed Storage Layer:**
   - Data is stored across a **shared, distributed storage cluster** spanning 3 AZs.
   - Each AZ has 2 storage nodes → **6 copies of data** across 3 AZs.
   - Uses a **quorum-based write model**: writes require acknowledgment from 4 of 6 nodes; reads require 3 of 6.
   - This means Aurora can tolerate losing an entire AZ plus one additional node without data loss.

2. **Log-Structured Storage:**
   - Aurora only writes **redo log records** (not full pages) to the storage layer.
   - The storage layer itself applies the redo logs to construct pages — reducing write amplification dramatically.
   - This reduces the write I/O by up to **7.7x** compared to MySQL.

3. **Advantages:**
   - **Durability:** 6-way replication with quorum writes provides 99.999999999% (11 nines) durability.
   - **Performance:** Write latency is reduced because only log records (not full data pages) cross the network.
   - **Read Replicas:** Aurora replicas share the same storage volume, so replication lag is typically under 100ms with no I/O overhead on the primary.
   - **Crash Recovery:** Near-instantaneous — the storage layer handles recovery, not the DB engine.
   - **Backtrack:** Aurora can rewind a DB cluster to a specific point in time without restoring from a snapshot (up to 72 hours).
   - **Storage Auto-Scaling:** Storage grows automatically in 10GB increments up to 128TB.

4. **Aurora Global Database:**
   - Replicates storage-level changes to up to 5 secondary regions with typical lag < 1 second.
   - Uses dedicated replication infrastructure, not the DB engine's replication.

---

### 2. Explain RDS for Oracle licensing models, cross-region considerations, and how you would architect for compliance in a regulated industry.

**Answer:**
**Oracle Licensing Models on RDS:**

1. **License Included (LI):**
   - Oracle license is included in the hourly RDS price.
   - Available for: Oracle Standard Edition Two (SE2) only on db.t3, db.r5, db.m5 instance families.
   - SE2 is limited to 1 server with up to 2 sockets or 16 vCPUs.

2. **Bring Your Own License (BYOL):**
   - Use existing Oracle licenses subject to Oracle's licensing terms.
   - Available for: SE2, Enterprise Edition (EE), Standard Edition (SE), SE1.
   - Oracle EE requires BYOL — it's not available as License Included.
   - Must comply with Oracle's cloud licensing policies (Oracle counts vCPUs, not physical cores, in AWS).

**Cross-Region Considerations:**
- Oracle BYOL licenses must be valid for all regions where you deploy.
- Oracle's licensing terms prohibit certain cross-region replication scenarios — consult Oracle licensing specialists.
- Cross-region Read Replicas for Oracle are supported but require careful license compliance review.
- Data residency requirements may restrict which regions you can use.

**Compliance Architecture for Regulated Industries (e.g., HIPAA, PCI-DSS, FedRAMP):**

```
[Application Tier - Private Subnet]
         |
    [RDS Proxy - IAM Auth + TLS]
         |
[RDS Oracle - Multi-AZ, Encrypted (CMK)]
    Primary (AZ-1) <--sync--> Standby (AZ-2)
         |
    [Read Replica - AZ-3 or Cross-Region]
         |
[Automated Backups + Manual Snapshots]
    → Encrypted S3 (via KMS CMK)
    → Cross-region backup replication
```

**Compliance Controls:**
1. **Encryption:** Customer-managed KMS keys (CMK) with annual rotation. Separate keys per environment.
2. **Network Isolation:** RDS in private subnets, no public accessibility. Security groups with least-privilege rules. VPC Flow Logs enabled.
3. **Audit Logging:** Enable Oracle Audit Vault or native Oracle auditing. Export audit logs to CloudWatch Logs → S3 → Athena for analysis.
4. **Access Control:** IAM database authentication where possible. Secrets Manager for credential rotation (every 30–90 days per policy). Principle of least privilege for DB users.
5. **Change Management:** Parameter group and option group changes tracked via AWS Config and CloudTrail.
6. **Backup and Recovery:** Automated backups with 35-day retention. Cross-region snapshot copies for DR. Regular restore testing (quarterly).
7. **Monitoring:** Enhanced Monitoring (1-second granularity), Performance Insights, CloudWatch alarms for security events.
8. **Compliance Reporting:** AWS Config rules for RDS compliance checks (encryption enabled, Multi-AZ enabled, no public access, backup retention ≥ N days).

---

### 3. How would you troubleshoot and resolve a severe RDS performance degradation issue in production? Walk through your complete methodology.

**Answer:**
**Phase 1: Triage and Immediate Stabilization**

1. **Check RDS CloudWatch Metrics immediately:**
   - `CPUUtilization` — Is the CPU saturated?
   - `DatabaseConnections` — Connection exhaustion?
   - `ReadIOPS/WriteIOPS` and `ReadLatency/WriteLatency` — Storage bottleneck?
   - `FreeableMemory` — Memory pressure?
   - `SwapUsage` — Swapping to disk?
   - `ReplicaLag` — If Read Replicas are involved.
   - `DiskQueueDepth` — I/O queuing.

2. **Check RDS Events:**
   ```
   aws rds describe-events --source-identifier my-db --source-type db-instance
   ```

3. **Enable/Check Performance Insights:**
   - Identify top SQL statements by load (DB time).
   - Look for wait events: `io/file/sql/binlog`, `lock/table/sql/handler`, `CPU`.
   - Identify blocking sessions.

**Phase 2: Root Cause Analysis**

**Scenario A: CPU Saturation**
- Use Performance Insights to identify top queries by CPU.
- Check for missing indexes: `EXPLAIN` /