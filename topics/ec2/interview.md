# EC2 — Interview Questions

---

## Easy

### 1. What is Amazon EC2, and what problem does it solve?

**Answer:**
Amazon Elastic Compute Cloud (EC2) is a web service that provides resizable virtual compute capacity in the cloud. It solves the problem of having to invest in physical hardware upfront by allowing users to launch virtual servers (called *instances*) on demand, pay only for what they use, and scale capacity up or down within minutes. This eliminates the need for data center management, hardware procurement, and capacity planning for fixed workloads.

---

### 2. What is an AMI (Amazon Machine Image), and what does it contain?

**Answer:**
An AMI is a pre-configured template used to launch EC2 instances. It contains:
- **Root volume snapshot** — the operating system (e.g., Amazon Linux, Ubuntu, Windows Server)
- **Launch permissions** — which AWS accounts can use the AMI
- **Block device mapping** — specifies the volumes to attach at launch (EBS or instance store)

AMIs can be AWS-provided, AWS Marketplace AMIs (third-party), community AMIs, or custom AMIs you create yourself. They are region-specific but can be copied across regions.

---

### 3. What are the main EC2 instance purchasing options?

**Answer:**

| Option | Description |
|---|---|
| **On-Demand** | Pay per second/hour with no commitment. Best for unpredictable workloads. |
| **Reserved Instances (RI)** | 1- or 3-year commitment. Up to 72% discount vs. On-Demand. |
| **Spot Instances** | Bid on unused EC2 capacity. Up to 90% discount. Can be interrupted. |
| **Savings Plans** | Flexible pricing model with commitment to $/hour spend. Covers EC2, Fargate, Lambda. |
| **Dedicated Hosts** | Physical server dedicated to your use. Useful for licensing compliance. |
| **Dedicated Instances** | Instances on hardware dedicated to a single customer, but hardware not exclusively yours. |
| **Capacity Reservations** | Reserve capacity in a specific AZ without a billing commitment. |

---

### 4. What is the difference between stopping and terminating an EC2 instance?

**Answer:**

| Action | Effect on Instance | Effect on EBS Root Volume | Effect on Instance Store | Billing |
|---|---|---|---|---|
| **Stop** | Instance shuts down; can be restarted | Data preserved | Data lost | No instance charge while stopped; EBS storage still billed |
| **Terminate** | Instance is permanently deleted | Deleted by default (if `DeleteOnTermination` = true) | Data lost | No further instance or EBS charges |

> **Note:** Stopped instances retain their private IP address (within a VPC) and their Elastic IP (if assigned). The public IP is released unless an Elastic IP is used.

---

### 5. What is an EC2 Security Group, and how does it differ from a Network ACL?

**Answer:**

| Feature | Security Group | Network ACL (NACL) |
|---|---|---|
| **Level** | Instance level | Subnet level |
| **State** | Stateful (return traffic automatically allowed) | Stateless (inbound and outbound rules evaluated separately) |
| **Rules** | Allow rules only | Allow and Deny rules |
| **Evaluation** | All rules evaluated before decision | Rules evaluated in order (lowest number first) |
| **Default** | Deny all inbound, allow all outbound | Allow all inbound and outbound |

Security Groups act as a virtual firewall at the instance/ENI level. NACLs are an additional layer of defense at the subnet boundary.

---

## Medium

### 1. Explain EC2 instance types and how you would choose the right one for a workload.

**Answer:**
EC2 instance types are grouped into families based on their compute, memory, storage, and networking characteristics:

| Family | Use Case | Examples |
|---|---|---|
| **General Purpose (T, M)** | Balanced CPU/memory; web servers, small DBs | `t3.micro`, `m6i.large` |
| **Compute Optimized (C)** | High CPU; batch processing, gaming, HPC | `c6i.xlarge`, `c7g.2xlarge` |
| **Memory Optimized (R, X, z)** | Large in-memory DBs, SAP HANA, real-time analytics | `r6i.4xlarge`, `x2idn.32xlarge` |
| **Storage Optimized (I, D, H)** | High sequential I/O; NoSQL, data warehousing | `i4i.xlarge`, `d3.2xlarge` |
| **Accelerated Computing (P, G, Inf, Trn)** | ML training/inference, GPU rendering | `p4d.24xlarge`, `g5.xlarge` |
| **HPC Optimized (Hpc)** | High-performance computing, tightly coupled workloads | `hpc6a.48xlarge` |

**Selection criteria:**
1. **CPU vs. Memory ratio** — Does the workload need more CPU or RAM?
2. **Burstable vs. fixed performance** — T-series instances use CPU credits; suitable for variable workloads.
3. **Network throughput** — Latency-sensitive apps may need enhanced networking (ENA).
4. **Storage requirements** — NVMe instance store vs. EBS-optimized.
5. **Processor architecture** — x86 (Intel/AMD) vs. ARM (Graviton) for cost optimization.
6. **Cost** — Graviton-based instances (e.g., `m7g`) offer up to 40% better price/performance.

---

### 2. What are EC2 placement groups, and when would you use each type?

**Answer:**
Placement groups control how EC2 instances are physically placed within AWS infrastructure. There are three types:

**1. Cluster Placement Group**
- Places instances close together within a **single AZ** on the same underlying hardware rack.
- Provides **low latency** and **high network throughput** (up to 100 Gbps with enhanced networking).
- **Use case:** HPC, tightly coupled distributed applications, big data jobs requiring fast node-to-node communication.
- **Risk:** All instances share the same physical hardware; a rack failure can affect all.

**2. Spread Placement Group**
- Spreads instances across **distinct underlying hardware** (different racks).
- Maximum **7 instances per AZ** per group.
- **Use case:** Small number of critical instances that must be isolated from each other (e.g., primary/secondary database nodes, Zookeeper quorum).

**3. Partition Placement Group**
- Divides instances into logical **partitions** (up to 7 per AZ), each on separate racks.
- Instances in one partition do not share hardware with instances in another.
- Supports hundreds of instances per group.
- **Use case:** Large distributed and replicated workloads like HDFS, HBase, Cassandra, Kafka.

> **Key rule:** You cannot merge placement groups, and a running instance cannot be moved into a placement group without stopping it first.

---

### 3. How does EC2 Auto Scaling work, and what are the different scaling policies?

**Answer:**
EC2 Auto Scaling automatically adjusts the number of EC2 instances in an **Auto Scaling Group (ASG)** based on demand, ensuring application availability and cost efficiency.

**Core components:**
- **Launch Template/Configuration** — Defines instance configuration (AMI, type, SG, user data).
- **Auto Scaling Group** — Defines min, max, and desired capacity; spans multiple AZs.
- **Scaling Policies** — Rules that trigger scaling actions.

**Scaling Policy Types:**

| Policy Type | Description | Use Case |
|---|---|---|
| **Target Tracking** | Maintains a target metric (e.g., 60% CPU). ASG automatically adds/removes instances. | Most common; simple to configure. |
| **Step Scaling** | Scales by a defined number of instances based on alarm breach size. | When you need fine-grained control per alarm threshold. |
| **Simple Scaling** | Adds/removes a fixed number after a CloudWatch alarm triggers; has cooldown period. | Legacy; largely replaced by step/target tracking. |
| **Scheduled Scaling** | Scales at a specific time (e.g., scale up at 8 AM every weekday). | Predictable traffic patterns. |
| **Predictive Scaling** | Uses ML to forecast load and proactively scale. | Cyclical patterns; avoids reactive lag. |

**Cooldown period:** Prevents ASG from launching/terminating instances before the previous scaling activity takes effect. Default is 300 seconds.

**Instance warm-up:** Time allowed for a new instance to start contributing to metrics before being included in ASG health checks.

---

### 4. Explain EBS volume types and when to use each.

**Answer:**
Amazon EBS (Elastic Block Store) provides persistent block storage for EC2. Volume types are divided into **SSD-backed** and **HDD-backed**:

**SSD-backed (IOPS-optimized):**

| Type | Max IOPS | Max Throughput | Use Case |
|---|---|---|---|
| `gp3` | 16,000 | 1,000 MB/s | General purpose; boot volumes, dev/test, most workloads. Default choice. |
| `gp2` | 16,000 | 250 MB/s | Legacy general purpose. `gp3` is preferred. |
| `io2 Block Express` | 256,000 | 4,000 MB/s | Mission-critical DBs (Oracle, SAP HANA), sub-millisecond latency. |
| `io1` | 64,000 | 1,000 MB/s | I/O-intensive DBs. Older generation of `io2`. |

**HDD-backed (throughput-optimized):**

| Type | Max IOPS | Max Throughput | Use Case |
|---|---|---|---|
| `st1` | 500 | 500 MB/s | Big data, data warehouses, log processing. Frequent access. |
| `sc1` | 250 | 250 MB/s | Cold data, infrequent access, lowest cost. |

**Key considerations:**
- `gp3` decouples IOPS and throughput from volume size (unlike `gp2`), making it more cost-effective.
- Only `io1`/`io2` support **Multi-Attach** (attach one volume to multiple instances in the same AZ).
- HDD volumes **cannot** be used as boot volumes.
- EBS volumes are **AZ-specific**; use snapshots to move data across AZs or regions.

---

### 5. What is EC2 Instance Metadata Service (IMDS), and how does IMDSv2 improve security?

**Answer:**
The **Instance Metadata Service (IMDS)** is an endpoint available at `http://169.254.169.254` that allows EC2 instances to access metadata about themselves without using IAM credentials. This includes:
- Instance ID, type, AMI ID
- Public/private IP addresses
- IAM role credentials (temporary credentials from the attached role)
- User data scripts

**IMDSv1 (Legacy):**
- Simple GET request to `http://169.254.169.254/latest/meta-data/`
- **Vulnerable to SSRF (Server-Side Request Forgery)** attacks — a malicious actor could trick the instance into making requests to IMDS and exfiltrating IAM credentials.

**IMDSv2 (Session-Oriented):**
- Requires a two-step process:
  1. **PUT request** to obtain a session token (with a TTL of 1 second to 6 hours):
     ```bash
     TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
       -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
     ```
  2. **GET request** using the token:
     ```bash
     curl -H "X-aws-ec2-metadata-token: $TOKEN" \
       http://169.254.169.254/latest/meta-data/instance-id
     ```
- **Mitigates SSRF** because SSRF exploits typically cannot set custom HTTP headers.
- Can be **enforced** at the instance level (`--metadata-options HttpTokens=required`) or account-wide via AWS Config/SCP.

> **Best practice:** Always enforce IMDSv2. You can also limit the hop limit (`HttpPutResponseHopLimit=1`) to prevent container workloads from accessing host metadata.

---

## Hard

### 1. Deep dive: How does EC2 networking work under the hood, including ENIs, ENA, and SR-IOV?

**Answer:**

**Elastic Network Interface (ENI):**
An ENI is a virtual network card that can be attached to an EC2 instance. Each instance has a primary ENI (`eth0`) and can have additional secondary ENIs. ENIs carry:
- Primary private IPv4 address
- Secondary private IPv4 addresses
- One Elastic IP per private IP
- One public IPv4 (on primary ENI)
- IPv6 addresses
- MAC address
- Security groups

ENIs are AZ-scoped and can be detached and re-attached to different instances (useful for failover scenarios — move the ENI with its private IP/EIP to a standby instance).

**Enhanced Networking (ENA — Elastic Network Adapter):**
Enhanced networking uses single-root I/O virtualization (SR-IOV) to provide higher I/O performance and lower CPU utilization compared to traditional virtualized network interfaces.

**SR-IOV (Single Root I/O Virtualization):**
- Allows a single physical NIC to appear as multiple virtual NICs directly to VMs.
- Bypasses the hypervisor for network I/O, reducing latency and CPU overhead.
- The **ENA driver** implements SR-IOV and supports:
  - Up to **100 Gbps** network bandwidth (on supported instances)
  - Lower latency (microsecond-level)
  - Higher packets per second (PPS)

**Elastic Fabric Adapter (EFA):**
- A network device for HPC and ML training workloads.
- Supports **OS-bypass** — applications can communicate directly with the EFA hardware using the **libfabric** API, completely bypassing the OS kernel network stack.
- Enables MPI (Message Passing Interface) and NCCL (NVIDIA Collective Communications Library) workloads to achieve near on-premises HPC performance.
- Only supported on specific instance types (e.g., `p4d`, `hpc6a`, `c5n`).

**Packet flow in EC2 networking:**
```
Application → Kernel TCP/IP Stack → ENA Driver (SR-IOV VF) 
→ Physical NIC → AWS Nitro Card → VPC Network
```

The **AWS Nitro System** offloads VPC networking, EBS I/O, and security to dedicated Nitro cards, freeing all host CPU resources for customer workloads — a fundamental architectural advantage over hypervisor-based systems.

---

### 2. Explain EC2 Spot Instance interruption handling, Spot Fleet, and EC2 Fleet in detail.

**Answer:**

**Spot Instance Mechanics:**
- Spot Instances use spare EC2 capacity at up to 90% discount.
- AWS can reclaim them with a **2-minute warning** (via instance metadata at `/latest/meta-data/spot/interruption-notice` or CloudWatch Events/EventBridge).
- Interruption behaviors: **terminate**, **stop**, or **hibernate** (configurable).
- Spot capacity pools are defined by instance type + OS + AZ.

**Handling Interruptions:**
```bash
# Poll for interruption notice
curl http://169.254.169.254/latest/meta-data/spot/interruption-notice
# Returns HTTP 404 if no interruption; 200 with action if interruption is imminent
```

Best practices:
1. **Checkpoint state** frequently to S3 or EBS.
2. Use **Spot interruption notices** via EventBridge to trigger graceful shutdown scripts.
3. Design stateless or fault-tolerant applications.
4. Use **diversification** across multiple instance types and AZs.

**Spot Fleet:**
A Spot Fleet is a collection of Spot Instances (and optionally On-Demand) that attempts to meet a target capacity. Key configurations:
- **Allocation strategies:**
  - `lowestPrice` — Launch from the lowest-priced pool. Risk: less diversified.
  - `diversified` — Spread across all pools. Best for availability.
  - `capacityOptimized` — Launch from pools with most available capacity. Reduces interruptions.
  - `priceCapacity