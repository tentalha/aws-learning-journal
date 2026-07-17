# Aurora — Interview Questions

---

## Easy

### 1. What is Amazon Aurora and how does it differ from standard RDS MySQL/PostgreSQL?

**Answer:**
Amazon Aurora is a fully managed, cloud-native relational database engine built by AWS that is compatible with MySQL and PostgreSQL. Unlike standard RDS MySQL/PostgreSQL, Aurora uses a distributed, fault-tolerant storage layer that automatically replicates data 6 ways across 3 Availability Zones. Aurora delivers up to 5x the throughput of standard MySQL and up to 3x the throughput of standard PostgreSQL, while offering faster failover, automatic storage scaling (up to 128 TiB), and a shared storage architecture that decouples compute from storage.

---

### 2. What are Aurora Read Replicas and how many can you have?

**Answer:**
Aurora Read Replicas are read-only database instances that share the same underlying storage volume as the primary (writer) instance. Because they share storage, replication lag is typically in the single-digit milliseconds. You can have up to **15 Aurora Read Replicas** per Aurora cluster (compared to 5 for standard RDS). Read Replicas can also serve as automatic failover targets, and Aurora will promote the replica with the highest priority tier (lowest numbered tier) in the event of a primary failure.

---

### 3. What is an Aurora Cluster Endpoint and a Reader Endpoint?

**Answer:**
- **Cluster Endpoint (Writer Endpoint):** Points to the current primary (writer) instance in the cluster. It automatically updates to point to the new primary after a failover. Applications should use this for write operations.
- **Reader Endpoint:** Load-balances read connections across all available Aurora Read Replicas in the cluster. If no replicas exist, it routes to the primary instance. This simplifies read scaling without requiring application-level logic to track individual replica endpoints.

---

### 4. How does Aurora handle storage scaling?

**Answer:**
Aurora storage is fully managed and scales automatically. The storage starts at 10 GiB and grows in **10 GiB increments** automatically as your data grows, up to a maximum of **128 TiB**. You do not need to pre-provision storage or worry about running out of disk space. Storage is billed based on actual usage, not provisioned capacity. This is different from standard RDS where you must pre-provision storage and manually scale it.

---

### 5. What is Aurora Serverless and when would you use it?

**Answer:**
Aurora Serverless is an on-demand, auto-scaling configuration for Amazon Aurora where the database automatically starts up, shuts down, and scales compute capacity based on application demand. It is measured in **Aurora Capacity Units (ACUs)**. You would use Aurora Serverless for:
- Infrequently used applications (e.g., internal tools, development/test environments)
- Applications with unpredictable or intermittent workloads
- New applications where sizing is unknown
- Multi-tenant SaaS applications with variable load

There are two versions: **Aurora Serverless v1** (older, coarser scaling) and **Aurora Serverless v2** (fine-grained scaling in 0.5 ACU increments, supports more features like Multi-AZ and Global Databases).

---

## Medium

### 1. Explain the Aurora storage architecture and how it achieves durability and availability.

**Answer:**
Aurora's storage architecture is fundamentally different from traditional databases:

- **Distributed Storage Volume:** Aurora uses a distributed, shared storage volume that spans 3 Availability Zones. The data is divided into **10 GiB segments** called Protection Groups.
- **6-Way Replication:** Each segment is replicated 6 times across 3 AZs (2 copies per AZ). Aurora only requires **4 out of 6 writes** to acknowledge a write as successful (quorum write), meaning it can tolerate the loss of an entire AZ plus one additional node without losing write availability.
- **Read Quorum:** Aurora requires only **3 out of 6** copies for a read quorum.
- **No Binary Log Replication:** Unlike standard MySQL replication, Aurora does not replicate data pages or binary logs to replicas. Instead, all instances share the same storage volume. Only redo log records are written to storage.
- **Continuous Backup to S3:** Aurora continuously backs up data to Amazon S3 with no performance impact. This enables point-in-time restore (PITR) to any second within the backup retention period (1–35 days).
- **Self-Healing:** Aurora storage is continuously scanned for errors and repaired automatically using data from other segments.

This architecture means Aurora can survive the loss of 2 copies of data without affecting write availability, and the loss of 3 copies without affecting read availability.

---

### 2. What is Aurora Global Database and how does it work?

**Answer:**
Aurora Global Database is designed for globally distributed applications that require low-latency reads across regions and fast disaster recovery across regions.

**How it works:**
- There is **one primary AWS Region** that handles all write operations.
- Up to **5 secondary read-only regions** can be added.
- Replication from primary to secondary regions uses dedicated infrastructure at the storage layer (not application-level replication), achieving typical replication lag of **under 1 second**.
- Each secondary region can have up to **16 Read Replicas**.

**Key features:**
- **Disaster Recovery:** If the primary region becomes unavailable, you can promote a secondary region to become the new primary in typically under **1 minute** (RPO of ~1 second, RTO of ~1 minute).
- **Managed Planned Failover:** Allows you to switch the primary region with no data loss (RPO = 0).
- **Read Scaling:** Secondary regions serve low-latency reads to geographically distributed users.

**Use cases:** Global SaaS applications, financial applications requiring cross-region DR, gaming leaderboards with global reach.

---

### 3. How does Aurora Failover work and what are the priority tiers?

**Answer:**
Aurora failover is automatic and typically completes in **under 30 seconds** (often 10–15 seconds for Aurora vs. 60–120 seconds for standard RDS). Here's how it works:

**Failover Process:**
1. Aurora detects that the primary instance has failed.
2. The DNS record for the Cluster Endpoint is updated to point to the promoted replica.
3. The promoted replica restarts and becomes the new writer.
4. Applications using the Cluster Endpoint will reconnect automatically after a brief interruption.

**Priority Tiers (0–15):**
- Each Read Replica is assigned a priority tier from **0 (highest) to 15 (lowest)**.
- During failover, Aurora promotes the replica in the highest priority tier (lowest number).
- If multiple replicas share the same tier, Aurora promotes the **largest instance** (most compute).
- If sizes are also equal, Aurora picks one arbitrarily.

**Best Practices:**
- Assign your most capable replica to Tier 0 or Tier 1.
- Use at least 2 replicas for HA — one in the same AZ as the primary and one in a different AZ.
- Use the **Cluster Endpoint** in your application connection string, not the instance endpoint, to benefit from automatic DNS failover.

---

### 4. What is Aurora Backtrack and how does it differ from Point-in-Time Restore?

**Answer:**
**Aurora Backtrack** is a feature exclusive to **Aurora MySQL** that allows you to "rewind" a database cluster to a specific point in time **without restoring from a backup**.

**How Backtrack works:**
- Aurora continuously tracks changes using a change record stream stored in the Aurora storage layer.
- You can backtrack to any point within the **backtrack window** (up to 72 hours).
- The operation happens **in-place** on the existing cluster — no new cluster is created.
- Backtracking takes **seconds to minutes** depending on how far back you go.

**Comparison with Point-in-Time Restore (PITR):**

| Feature | Backtrack | PITR |
|---|---|---|
| Creates new cluster | No | Yes |
| Speed | Seconds to minutes | Minutes to hours |
| Window | Up to 72 hours | Up to 35 days |
| Compatibility | Aurora MySQL only | Aurora MySQL & PostgreSQL |
| Use case | Quick undo of mistakes | Longer-term recovery |
| Downtime | Brief (cluster restarts) | None on original cluster |

**When to use Backtrack:** Accidental `DROP TABLE`, bad data migration, unwanted batch update.
**When to use PITR:** Need to recover data from days/weeks ago, or need to keep the original cluster running while investigating.

---

### 5. Explain Aurora's connection management features: RDS Proxy and how it helps Aurora.

**Answer:**
**RDS Proxy** is a fully managed database proxy that sits between your application and Aurora, pooling and sharing database connections.

**Why Aurora needs connection management:**
- Aurora instances have a maximum connection limit based on instance size (e.g., `db.r5.large` supports ~1,000 connections).
- Serverless applications (Lambda) can create thousands of concurrent connections, overwhelming the database.
- Opening/closing connections is expensive in terms of memory and CPU.

**How RDS Proxy helps:**

1. **Connection Pooling:** RDS Proxy maintains a pool of established connections to Aurora and multiplexes many application connections over fewer database connections. This can reduce database connections by **up to 99%** for Lambda workloads.

2. **Improved Failover:** During an Aurora failover, RDS Proxy automatically routes traffic to the new primary. Applications connected to the proxy experience reduced failover time (often **under 30 seconds**) without needing to handle reconnection logic.

3. **IAM Authentication:** RDS Proxy supports IAM-based authentication, eliminating the need to embed database credentials in application code. Credentials are stored in AWS Secrets Manager.

4. **Pinning:** When a session uses database features that cannot be safely multiplexed (e.g., temporary tables, `SET` statements), RDS Proxy "pins" that connection to a specific database connection. Excessive pinning reduces efficiency.

**Architecture:** Application → RDS Proxy Endpoint → Aurora Cluster Endpoint → Aurora Writer Instance

---

## Hard

### 1. Deep dive into Aurora's write path and how it achieves higher throughput than standard MySQL.

**Answer:**
Understanding Aurora's write path requires comparing it to traditional MySQL's write path:

**Traditional MySQL Write Path:**
1. Write to binlog
2. Write to InnoDB redo log
3. Write to InnoDB buffer pool (dirty page)
4. Replicate binlog to replicas
5. Eventually flush dirty pages to disk

This involves multiple I/O operations and network round trips for replication.

**Aurora MySQL Write Path:**
1. The writer instance generates **redo log records** only (no binlog required for replication).
2. Redo log records are sent in parallel to all **6 storage nodes** across 3 AZs.
3. Aurora waits for **4 out of 6 acknowledgments** (quorum write). The write is complete.
4. Storage nodes apply the redo log records asynchronously to materialize data pages.
5. Read Replicas receive the redo log records from storage and apply them to their buffer caches — **no data is sent over the network from the writer to replicas**.

**Key optimizations:**
- **Offloaded page materialization:** The writer never writes full data pages to storage; it only writes redo log records. Storage nodes materialize pages on demand. This reduces network I/O significantly.
- **Reduced write amplification:** Traditional MySQL writes data multiple times (binlog + redo log + data pages). Aurora writes only redo log records.
- **Asynchronous commits:** Aurora uses an asynchronous commit model where the writer can pipeline multiple transactions, reducing latency per transaction.
- **No double-write buffer:** Traditional InnoDB uses a double-write buffer to prevent partial page writes. Aurora's storage layer handles atomic writes, eliminating this overhead.
- **Parallel I/O to storage:** Writing to 6 nodes in parallel across a high-bandwidth network is faster than writing to a single local disk.

**Result:** Aurora achieves ~5x MySQL throughput primarily by reducing the number of I/O operations in the write path and leveraging a distributed storage system optimized for cloud infrastructure.

---

### 2. How would you design an Aurora cluster for a multi-tenant SaaS application that needs strict tenant isolation, variable load patterns, and cost optimization?

**Answer:**
This is a complex design problem with several valid approaches. Here's a comprehensive architecture:

**Option 1: Database-per-Tenant with Aurora Serverless v2**
- Each tenant gets their own Aurora Serverless v2 cluster.
- **Pros:** Complete isolation, independent scaling, easy tenant offboarding.
- **Cons:** Cost can be high for many tenants; operational overhead scales with tenant count.
- **Best for:** Enterprise SaaS with few high-value tenants.

**Option 2: Schema-per-Tenant on Shared Aurora Cluster**
- All tenants share one Aurora cluster with separate schemas.
- Tenant isolation enforced at application layer using Row-Level Security (PostgreSQL) or views.
- **Aurora Serverless v2** scales compute based on aggregate load.
- **Pros:** Lower cost, simpler operations.
- **Cons:** "Noisy neighbor" problem; a rogue tenant can impact others.
- **Best for:** High-volume SaaS with many small tenants.

**Recommended Hybrid Architecture:**
```
Tier 1 (Enterprise Tenants):
  - Dedicated Aurora Serverless v2 cluster per tenant
  - Aurora Global Database for tenants with global presence
  - RDS Proxy for connection pooling

Tier 2 (SMB Tenants):
  - Shared Aurora Serverless v2 cluster (pool model)
  - Schema-per-tenant with PostgreSQL RLS
  - RDS Proxy mandatory

Tier 3 (Free/Trial Tenants):
  - Shared Aurora Serverless v2 with auto-pause enabled
  - Strict resource limits enforced at application layer
```

**Cost Optimization Strategies:**
1. **Aurora Serverless v2 with auto-pause** for idle tenants (v2 supports pause for serverless instances).
2. **Reserved Instances** for predictable baseline capacity on provisioned instances.
3. **Aurora I/O Optimized** pricing for I/O-intensive workloads (predictable cost, no per-I/O charges).
4. **Aurora cloning** for test environments — clones share storage with the source cluster (copy-on-write), dramatically reducing storage costs.

**Isolation and Security:**
- Separate IAM roles and Secrets Manager secrets per tenant tier.
- VPC security groups to restrict access.
- Encryption at rest with separate KMS keys per enterprise tenant.
- CloudTrail + Database Activity Streams for audit logging.

**Monitoring:**
- CloudWatch custom metrics per tenant (query tagging with tenant ID).
- Performance Insights with per-query attribution.
- Aurora Database Activity Streams → Kinesis → tenant-level analytics.

---

### 3. Explain Aurora's approach to crash recovery and how it differs from traditional database crash recovery.

**Answer:**
**Traditional Database Crash Recovery (InnoDB):**
1. On restart, MySQL scans the redo log from the last checkpoint.
2. Applies all redo log records to bring data pages to a consistent state (redo phase).
3. Identifies uncommitted transactions and rolls them back (undo phase).
4. This process can take **minutes to hours** for large databases with many dirty pages.

**Aurora's Crash Recovery:**

Aurora fundamentally changes crash recovery because of its storage architecture:

**Key insight:** Aurora storage nodes continuously apply redo log records to materialize pages. The storage layer is always in a consistent state from its perspective. The compute layer (DB instance) only holds a buffer cache.

**Aurora Crash Recovery Process:**
1. When the Aurora writer crashes, the storage layer is not affected — it continues operating independently.
2. A new writer instance starts (either via failover to a replica or instance restart).
3. The new writer reads the **volume durable LSN (VDL)** from the storage layer — the point up to which all 6 storage nodes have durably persisted redo log records.
4. The writer truncates any redo log records beyond the VDL (these were in-flight and not durably committed).
5. The writer reads the **volume complete LSN (VCL)** — the highest LSN at which a complete redo log record exists.
6. Recovery involves only reading recent redo log records from storage (not scanning the entire redo log from the last checkpoint).
7. Undo phase rolls back uncommitted transactions.

**Why Aurora recovery is faster:**
- **No checkpoint delay:** Traditional MySQL must wait for dirty pages to be flushed to disk at checkpoint. Aurora storage nodes continuously apply redo, so there's no large "dirty page" backlog.
- **Parallel recovery:** Multiple storage nodes can participate in recovery in parallel.
- **Smaller recovery scope:** Only redo log records since the last VDL need to be processed, not since the last full checkpoint.
- **No redo log scan on replica promotion:** Read Replicas already have an up-to-date