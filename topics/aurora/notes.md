# Aurora

## What is it?

**Amazon Aurora** is a fully managed, cloud-native relational database engine built by AWS, available as part of the **Amazon Relational Database Service (RDS)** family. It is compatible with both **MySQL** (versions 5.6, 5.7, 8.0) and **PostgreSQL** (versions 11–16), meaning existing applications can connect to Aurora using standard MySQL or PostgreSQL drivers without code changes.

Aurora is classified as a **cloud-native OLTP (Online Transaction Processing) relational database** that combines the performance and availability of high-end commercial databases with the simplicity and cost-effectiveness of open-source databases.

**Key identifiers:**
- **Service category:** Managed Relational Database (RDS family)
- **Engine types:** Aurora MySQL, Aurora PostgreSQL
- **Deployment variants:** Aurora Provisioned, Aurora Serverless v1, Aurora Serverless v2, Aurora Global Database, Aurora Multi-Master (deprecated)
- **Storage type:** Distributed, shared, fault-tolerant cluster volume (not instance-local storage)

---

## Why do we need it?

### The Problem

Traditional relational databases (including standard RDS MySQL/PostgreSQL) suffer from several limitations in cloud environments:

| Problem | Traditional RDS | Aurora Solution |
|---|---|---|
| Replication lag | Async replication to replicas (seconds of lag) | Log-based replication with < 10ms replica lag |
| Failover time | 60–120 seconds | < 30 seconds (typically < 15s) |
| Storage scaling | Manual provisioning, resizing causes downtime | Auto-grows in 10 GiB increments up to 128 TiB |
| Write throughput | Limited by single-node I/O | Distributed storage layer with parallel writes |
| Durability | Single AZ or synchronous standby | 6 copies across 3 AZs by default |
| Read scaling | Limited replicas | Up to 15 Aurora Replicas |

### When to Use Aurora

**Use Aurora when:**
- You need **high availability** (99.99% SLA) for mission-critical workloads
- Your application requires **low-latency reads** across multiple replicas
- You expect **unpredictable or variable workloads** (use Serverless v2)
- You need **global distribution** with < 1 second RPO across regions
- You are migrating from commercial databases (Oracle, SQL Server) to open-source compatible engines
- You need **5x MySQL throughput** or **3x PostgreSQL throughput** at similar cost

**Real Business Scenarios:**

1. **E-commerce platform:** A retail company processes millions of transactions during Black Friday. Aurora's auto-scaling storage and read replicas handle peak load without pre-provisioning.

2. **SaaS application:** A multi-tenant SaaS product needs 99.99% uptime with automatic failover under 30 seconds. Aurora's multi-AZ cluster with automated failover satisfies the SLA.

3. **Financial services:** A fintech company needs ACID transactions with global replication for compliance reporting. Aurora Global Database provides cross-region reads with < 1 second lag.

4. **Healthcare application:** A patient records system requires encryption at rest, audit logging, and automatic backups. Aurora provides all of these natively.

5. **Gaming backend:** A mobile game with unpredictable player spikes uses Aurora Serverless v2 to scale compute from 0.5 ACUs to 128 ACUs in seconds without connection drops.

---

## Internal Working

Aurora's architecture fundamentally separates **compute** from **storage**, which is the core innovation that enables its performance and durability characteristics.

### The Aurora Storage Subsystem

Unlike traditional databases where each instance has its own local storage, Aurora uses a **shared distributed storage cluster** called the **Aurora Cluster Volume**.

```
┌─────────────────────────────────────────────────────────────┐
│                    Aurora Cluster Volume                     │
│                                                             │
│  AZ-1              AZ-2              AZ-3                   │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐          │
│  │ Copy 1   │      │ Copy 3   │      │ Copy 5   │          │
│  │ Copy 2   │      │ Copy 4   │      │ Copy 6   │          │
│  └──────────┘      └──────────┘      └──────────┘          │
│                                                             │
│  6 copies of data across 3 AZs (2 copies per AZ)           │
│  Quorum: 4/6 writes needed, 3/6 reads needed               │
└─────────────────────────────────────────────────────────────┘
```

**Key storage behaviors:**

1. **Quorum writes:** Aurora writes data to 4 out of 6 storage nodes before acknowledging a write to the client. This tolerates the loss of an entire AZ (2 nodes) plus one additional node.

2. **Quorum reads:** Aurora reads from 3 out of 6 nodes. This ensures consistency even during partial failures.

3. **Log-structured storage:** Aurora only writes **redo log records** to storage (not full data pages). The storage nodes reconstruct data pages from redo logs on demand. This dramatically reduces write amplification.

4. **Peer-to-peer repair:** Storage nodes continuously gossip with each other to detect and repair corrupted or missing segments without involving the database engine.

5. **Auto-growing segments:** The cluster volume is divided into **10 GiB segments** called **Protection Groups**. Each segment has 6 copies. As data grows, new segments are automatically added.

### Write Path

```
Application
    │
    ▼
Writer Instance (Primary)
    │
    ├─► Generates redo log records
    │
    ▼
Storage Nodes (4/6 quorum write)
    │
    ├─► Log records stored
    ├─► Pages reconstructed lazily
    └─► Acknowledgment sent to writer
         │
         ▼
    Writer acknowledges to application
```

### Read Path (Replicas)

```
Application
    │
    ▼
Reader Instance (Replica)
    │
    ├─► Receives log stream from writer (async, < 10ms lag)
    ├─► Maintains buffer cache
    │
    ▼
Storage Nodes (3/6 quorum read)
    │
    └─► Returns requested data pages
```

### Why Replicas Are Fast

Aurora replicas do **not** replay redo logs on their own storage. Instead:
- The writer sends log records to replicas via a shared memory segment
- Replicas apply log records to their **buffer pool (cache)** only
- If a replica needs a page not in cache, it reads directly from the shared storage cluster
- This means replica lag is typically **< 10 milliseconds** vs seconds for traditional async replication

---

## Architecture

### Cluster Architecture

```
                        ┌─────────────────────────────────┐
                        │         Aurora Cluster           │
                        │                                  │
                    ┌───┴───────────────────────────────┐  │
                    │         Cluster Endpoint           │  │
                    │  mydb.cluster-xxx.us-east-1.rds   │  │
                    └───┬───────────────────────────────┘  │
                        │                                  │
                        ▼                                  │
              ┌─────────────────┐                         │
              │  Writer Instance │ ◄── Single writer      │
              │  (Primary)       │     at a time          │
              └─────────────────┘                         │
                        │                                  │
          ┌─────────────┼─────────────┐                   │
          ▼             ▼             ▼                   │
   ┌──────────┐  ┌──────────┐  ┌──────────┐              │
   │ Replica 1│  │ Replica 2│  │ Replica N│              │
   │ (AZ-1)   │  │ (AZ-2)   │  │ (up to15)│              │
   └──────────┘  └──────────┘  └──────────┘              │
          │             │             │                   │
          └─────────────┴─────────────┘                   │
                        │                                  │
                        ▼                                  │
          ┌─────────────────────────────┐                 │
          │    Reader Endpoint           │                 │
          │  mydb.cluster-ro-xxx.rds    │                 │
          └─────────────────────────────┘                 │
                        │                                  │
                        ▼                                  │
          ┌─────────────────────────────────────────────┐ │
          │           Aurora Cluster Volume              │ │
          │    (Shared distributed storage, 128 TiB)    │ │
          └─────────────────────────────────────────────┘ │
                        └─────────────────────────────────┘
```

### Endpoint Types

| Endpoint Type | DNS Name Pattern | Routes To | Use Case |
|---|---|---|---|
| **Cluster Endpoint** | `cluster-xxx.region.rds.amazonaws.com` | Writer (Primary) | All writes, DDL |
| **Reader Endpoint** | `cluster-ro-xxx.region.rds.amazonaws.com` | Load balanced across replicas | Read-heavy queries |
| **Instance Endpoint** | `instance-xxx.region.rds.amazonaws.com` | Specific instance | Debugging, specific routing |
| **Custom Endpoint** | User-defined | Subset of instances | Specialized workloads |

### Aurora Global Database Architecture

```
Primary Region (us-east-1)
┌──────────────────────────────────┐
│  Writer + Up to 15 Replicas      │
│  Aurora Cluster Volume           │
│  Replication to secondary: ~1s  │
└──────────────┬───────────────────┘
               │ Storage-level replication
               │ (< 1 second lag, async)
               ▼
Secondary Region (eu-west-1)
┌──────────────────────────────────┐
│  Up to 16 Read-Only Instances    │
│  Can be promoted in < 1 minute  │
│  (up to 5 secondary regions)    │
└──────────────────────────────────┘
```

### Aurora Serverless v2 Architecture

```
Application
    │
    ▼
Aurora Serverless v2 Cluster
    │
    ├─► Writer (scales 0.5 to 128 ACUs)
    │       └─► Scales in ~1 second increments
    │
    ├─► Readers (independently scalable)
    │
    └─► Shared Aurora Storage Volume
            └─► Scales automatically to 128 TiB
```

**ACU (Aurora Capacity Unit):** 1 ACU ≈ 2 GiB RAM + proportional CPU + networking

---

## Real World Example

### Scenario: High-Traffic E-Commerce Platform

**Company:** RetailMax — an online retailer expecting 10x traffic spikes during sales events.

**Requirements:**
- 99.99% uptime SLA
- Sub-100ms read latency
- Automatic failover < 30 seconds
- Global customers (US primary, EU secondary reads)
- Compliance: data encrypted, audit logs retained 90 days

#### Step-by-Step Architecture Walkthrough

**Step 1: Create Aurora Cluster**
```
Region: us-east-1
Engine: Aurora MySQL 8.0
Writer instance: db.r6g.2xlarge (8 vCPU, 64 GiB RAM)
Replicas: 2x db.r6g.xlarge across AZ-2 and AZ-3
Multi-AZ: Enabled (automatic)
Storage: Auto-scaling up to 128 TiB
```

**Step 2: Configure Application Connectivity**
```
Write operations → Cluster Endpoint (mydb.cluster-xxx.us-east-1.rds.amazonaws.com)
Read operations  → Reader Endpoint (mydb.cluster-ro-xxx.us-east-1.rds.amazonaws.com)
Connection pool  → RDS Proxy (handles connection management, reduces DB connections)
```

**Step 3: Implement Read/Write Splitting in Application**
```javascript
// Application code routes reads to reader endpoint
const writePool = createPool({ host: CLUSTER_ENDPOINT });
const readPool  = createPool({ host: READER_ENDPOINT });

// Product catalog queries → readPool
// Order creation → writePool
```

**Step 4: Add Global Database for EU Customers**
```
Primary: us-east-1 (writer + 2 readers)
Secondary: eu-west-1 (2 read-only instances)
EU customer reads served locally with < 1s lag
Failover: EU region can be promoted to writer in < 1 minute
```

**Step 5: Configure Auto Scaling for Replicas**
```
Metric: AuroraReplicaLag or CPUUtilization
Min replicas: 2
Max replicas: 15
Scale-out threshold: CPU > 70% for 5 minutes
Scale-in threshold: CPU < 30% for 15 minutes
```

**Step 6: Black Friday Traffic Spike**
```
Normal:     2 replicas handling 5,000 reads/sec
Black Friday: Aurora Auto Scaling adds 8 more replicas
              Total: 10 replicas handling 50,000 reads/sec
              Scale-out completes in ~5 minutes
              Reader endpoint automatically includes new replicas
```

**Step 7: Simulate Failover Test**
```bash
# Simulate AZ failure — Aurora automatically promotes replica
aws rds failover-db-cluster --db-cluster-identifier retailmax-cluster

# Result: New writer promoted in < 30 seconds
# Application reconnects via cluster endpoint automatically
# Zero data loss (synchronous log writes to 4/6 nodes)
```

**Step 8: Monitoring Setup**
```
CloudWatch Alarms:
  - AuroraReplicaLag > 100ms → SNS alert
  - DatabaseConnections > 80% of max → Scale up
  - FreeLocalStorage < 10% → Alert (for temp tables)
  - CPUUtilization > 80% → Scale up writer
```

**Outcome:** RetailMax achieved 99.995% uptime over 12 months, handled 50x normal traffic during sales events without pre-provisioning, and reduced database costs by 40% compared to their previous Oracle setup.

---

## Advantages

### Performance
- **5x throughput over MySQL RDS** and **3x over PostgreSQL RDS** at the same price point
- **< 10ms replica lag** vs seconds for traditional async replication
- **Parallel query** (Aurora MySQL): Pushes query processing down to the storage layer across thousands of storage nodes
- **Backtrack** (Aurora MySQL): Rewind database to a previous point in time without restoring from backup (up to 72 hours)

### Availability & Durability
- **99.99% availability SLA** (higher than standard RDS 99.95%)
- **6 copies of data across 3 AZs** — tolerates loss of 2 copies for writes, 3 copies for reads
- **Automatic failover < 30 seconds** to a replica (< 15 seconds typical)
- **Continuous backup to S3** — no performance impact, no backup window needed
- **Self-healing storage** — corrupted segments automatically repaired from peer nodes

### Scalability
- **Storage auto-scales** from 10 GiB to 128 TiB in 10 GiB increments — no downtime
- **Up to 15 Aurora Replicas** with load-balanced reader endpoint
- **Aurora Serverless v2** scales compute in fine-grained increments (0.5 ACU steps) in < 1 second
- **Aurora Auto Scaling** adds/removes replicas based on CloudWatch metrics

### Operational Simplicity
- **Fully managed** — AWS handles patching, backups, monitoring, replication
- **Zero-downtime patching** with rolling updates across replicas
- **Point-in-time restore** to any second within the backup retention period (1–35 days)
- **Clone databases** in seconds using copy-on-write — no data duplication until changes are made

### Cost Efficiency
- **Pay per I/O** (provisioned) or **per ACU-