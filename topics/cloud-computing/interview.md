# Cloud Computing — Interview Questions

---

## Easy

### 1. What is cloud computing, and what are its three primary service models?

**Answer:**
Cloud computing is the on-demand delivery of IT resources — including compute power, storage, databases, networking, software, and analytics — over the internet with pay-as-you-go pricing. Instead of owning and maintaining physical data centers and servers, organizations can access technology services from a cloud provider.

The three primary service models are:

- **IaaS (Infrastructure as a Service):** Provides virtualized computing resources over the internet. The cloud provider manages the underlying hardware; the customer manages the OS, middleware, and applications. *Example: Amazon EC2, Azure VMs.*
- **PaaS (Platform as a Service):** Provides a platform allowing customers to develop, run, and manage applications without managing the underlying infrastructure. *Example: AWS Elastic Beanstalk, Google App Engine.*
- **SaaS (Software as a Service):** Delivers software applications over the internet on a subscription basis. The provider manages everything. *Example: Salesforce, Google Workspace, Microsoft 365.*

---

### 2. What are the four cloud deployment models?

**Answer:**
The four cloud deployment models are:

| Model | Description | Example Use Case |
|---|---|---|
| **Public Cloud** | Resources owned and operated by a third-party provider, shared across multiple customers | Startups, web applications |
| **Private Cloud** | Cloud infrastructure operated solely for a single organization, on-premises or hosted | Government agencies, healthcare |
| **Hybrid Cloud** | Combination of public and private clouds, connected by technology allowing data and applications to be shared | Enterprises with legacy systems |
| **Multi-Cloud** | Use of multiple cloud service providers simultaneously | Avoiding vendor lock-in, geographic redundancy |

---

### 3. What is the difference between horizontal and vertical scaling?

**Answer:**

- **Vertical Scaling (Scale Up/Down):** Increasing or decreasing the capacity of an existing resource — for example, upgrading an EC2 instance from `t3.medium` to `t3.xlarge`. It is limited by the maximum size of a single machine and typically requires downtime.

- **Horizontal Scaling (Scale Out/In):** Adding or removing instances to distribute the load — for example, adding more EC2 instances behind a load balancer. This is preferred in cloud architectures because it offers theoretically unlimited scale and higher availability.

Cloud-native architectures favor horizontal scaling because it supports fault tolerance, elasticity, and avoids single points of failure.

---

### 4. What is the difference between availability and durability in cloud storage?

**Answer:**

- **Availability** refers to the percentage of time a system or service is accessible and operational. For example, Amazon S3 Standard offers **99.99% availability**, meaning the service is expected to be accessible almost all the time.

- **Durability** refers to the probability that data will not be lost over a given period. Amazon S3 offers **99.999999999% (11 nines) durability**, meaning data is extremely unlikely to be lost due to hardware failures, as it is replicated across multiple facilities.

**Key distinction:** A system can be highly durable (data is safe) but temporarily unavailable (cannot access it right now). Both metrics matter independently when choosing a storage solution.

---

### 5. What is a Region and an Availability Zone (AZ) in AWS?

**Answer:**

- **Region:** A geographic area containing multiple, isolated locations known as Availability Zones. Each AWS Region is completely independent. Examples include `us-east-1` (N. Virginia), `eu-west-1` (Ireland), and `ap-southeast-1` (Singapore). Customers choose regions based on latency, compliance, and service availability.

- **Availability Zone (AZ):** One or more discrete data centers within a Region, each with redundant power, networking, and connectivity. AZs within a Region are physically separated by meaningful distances (typically tens of miles) but are interconnected with high-bandwidth, low-latency networking.

**Why it matters:** Deploying across multiple AZs provides fault tolerance. If one AZ fails, applications continue running in others. Deploying across multiple Regions provides disaster recovery and geographic redundancy.

---

## Medium

### 1. Explain the AWS Shared Responsibility Model. How does it differ across IaaS, PaaS, and SaaS?

**Answer:**
The AWS Shared Responsibility Model defines the security and compliance obligations split between AWS and the customer.

**AWS is responsible for "Security OF the Cloud":**
- Physical security of data centers
- Hardware, networking, and global infrastructure
- Hypervisor and virtualization layer
- Managed services software (e.g., RDS database engine patches)

**Customer is responsible for "Security IN the Cloud":**
- Operating system patching (for EC2)
- Application-level security
- Identity and Access Management (IAM) configuration
- Data encryption (at rest and in transit)
- Network configuration (Security Groups, NACLs)
- Customer data

**How it shifts across service models:**

| Service Model | Customer Responsibility | AWS Responsibility |
|---|---|---|
| **IaaS (EC2)** | OS, middleware, runtime, applications, data | Hardware, hypervisor, networking |
| **PaaS (Elastic Beanstalk, RDS)** | Applications, data, some configuration | OS, runtime, middleware, hardware |
| **SaaS (Amazon Chime, WorkMail)** | User data, access management | Everything else |

As you move from IaaS → PaaS → SaaS, the customer's operational burden decreases but so does their control. Understanding this model is critical for passing compliance audits and designing secure architectures.

---

### 2. What is auto scaling, and how does it work in AWS? What are the different scaling policies?

**Answer:**
**Auto Scaling** automatically adjusts the number of compute resources in response to demand, ensuring you have the right amount of capacity at any given time — avoiding over-provisioning (cost waste) and under-provisioning (performance degradation).

**Core Components of AWS Auto Scaling:**
- **Launch Template/Configuration:** Defines what type of instance to launch (AMI, instance type, security groups, user data).
- **Auto Scaling Group (ASG):** Defines the minimum, maximum, and desired number of instances.
- **Scaling Policies:** Rules that trigger scaling actions.

**Types of Scaling Policies:**

1. **Target Tracking Scaling:** Automatically adjusts capacity to maintain a specific metric at a target value. *Example: Keep average CPU utilization at 50%.* This is the simplest and most commonly recommended approach.

2. **Step Scaling:** Scales in steps based on CloudWatch alarm thresholds. *Example: Add 2 instances when CPU > 70%, add 4 instances when CPU > 90%.* Provides more granular control.

3. **Simple Scaling:** Triggered by a single CloudWatch alarm, adds or removes a fixed number of instances. Has a cooldown period before the next action. Less sophisticated than step scaling.

4. **Scheduled Scaling:** Scales based on predictable, time-based patterns. *Example: Scale out every Monday at 8 AM when business hours begin.*

5. **Predictive Scaling:** Uses machine learning to forecast future traffic and proactively scales ahead of demand. Useful for applications with cyclical traffic patterns.

**Cooldown Period:** After a scaling activity, a cooldown period (default 300 seconds) prevents additional scaling actions, allowing metrics to stabilize.

---

### 3. What is a VPC, and what are its key components? How would you design a secure VPC?

**Answer:**
A **Virtual Private Cloud (VPC)** is a logically isolated section of the AWS cloud where you can launch AWS resources in a virtual network that you define. You have complete control over your virtual networking environment, including IP address ranges, subnets, route tables, and network gateways.

**Key Components:**

| Component | Purpose |
|---|---|
| **Subnets** | Subdivisions of the VPC's IP range; can be public or private |
| **Internet Gateway (IGW)** | Enables communication between VPC resources and the internet |
| **NAT Gateway** | Allows private subnet resources to initiate outbound internet traffic |
| **Route Tables** | Control traffic routing within the VPC |
| **Security Groups** | Stateful, instance-level firewall (allow rules only) |
| **Network ACLs (NACLs)** | Stateless, subnet-level firewall (allow and deny rules) |
| **VPC Peering** | Connects two VPCs privately |
| **VPC Endpoints** | Private connectivity to AWS services without internet |
| **Transit Gateway** | Hub-and-spoke model for connecting multiple VPCs |

**Secure VPC Design (3-Tier Architecture):**

```
Internet
    |
Internet Gateway
    |
Public Subnet (AZ-1, AZ-2)
  - Application Load Balancer
  - NAT Gateway
    |
Private Subnet - App Tier (AZ-1, AZ-2)
  - EC2 / ECS / EKS workloads
    |
Private Subnet - Data Tier (AZ-1, AZ-2)
  - RDS (Multi-AZ)
  - ElastiCache
```

**Security Best Practices:**
- Never place databases in public subnets
- Use Security Groups as primary firewall; NACLs as secondary defense
- Enable VPC Flow Logs for network traffic auditing
- Use VPC Endpoints for S3, DynamoDB, and other AWS services to avoid internet traversal
- Apply the principle of least privilege in all Security Group rules
- Enable AWS GuardDuty for threat detection

---

### 4. Explain the differences between SQL and NoSQL databases in the cloud context. When would you choose each?

**Answer:**

**SQL (Relational) Databases:**
- Structured schema with tables, rows, and columns
- ACID (Atomicity, Consistency, Isolation, Durability) transactions
- Vertical scaling is traditional; horizontal scaling is complex
- Supports complex JOINs and aggregations
- *AWS Examples:* Amazon RDS (MySQL, PostgreSQL, Oracle, SQL Server), Amazon Aurora

**NoSQL Databases:**
- Schema-flexible (document, key-value, wide-column, graph)
- Eventually consistent by default (though strong consistency is often available)
- Designed for horizontal scaling from the ground up
- Optimized for specific access patterns
- *AWS Examples:* DynamoDB (key-value/document), ElastiCache (in-memory), Neptune (graph), Keyspaces (wide-column)

**Decision Framework:**

| Criteria | Choose SQL | Choose NoSQL |
|---|---|---|
| **Data structure** | Structured, relational data | Semi-structured, hierarchical, or variable data |
| **Transactions** | Complex multi-table ACID transactions | Simple, single-item operations |
| **Query patterns** | Ad-hoc, complex queries | Known, predictable access patterns |
| **Scale** | Moderate scale with complex relationships | Massive scale, millions of req/sec |
| **Consistency** | Strong consistency required | Eventual consistency acceptable |
| **Use cases** | Financial systems, ERP, e-commerce orders | User sessions, IoT data, catalogs, leaderboards |

**Important nuance:** Modern cloud databases blur these lines. Amazon Aurora Serverless scales dynamically. DynamoDB supports ACID transactions. Aurora offers a MySQL/PostgreSQL-compatible interface with NoSQL-like scalability.

---

### 5. What is serverless computing? What are its advantages, disadvantages, and key AWS serverless services?

**Answer:**
**Serverless computing** is a cloud execution model where the cloud provider dynamically manages the allocation and provisioning of servers. Developers write and deploy code without thinking about infrastructure. You pay only for the compute time consumed — there is no charge when code is not running.

**Key Characteristics:**
- No server provisioning or management
- Automatic scaling (including to zero)
- Pay-per-use billing (often millisecond granularity)
- Built-in high availability

**Advantages:**
- ✅ Reduced operational overhead — no OS patching, capacity planning
- ✅ Cost efficiency for variable or unpredictable workloads
- ✅ Faster time to market
- ✅ Automatic scaling without manual intervention
- ✅ Built-in fault tolerance

**Disadvantages:**
- ❌ **Cold starts:** Initial invocation latency when a function hasn't been used recently
- ❌ **Execution limits:** AWS Lambda has a 15-minute maximum execution time
- ❌ **Vendor lock-in:** Tight coupling with provider-specific services
- ❌ **Debugging complexity:** Distributed tracing and local testing are harder
- ❌ **Not suitable for:** Long-running processes, stateful applications, high-frequency constant workloads (can be more expensive than reserved instances)

**AWS Serverless Ecosystem:**

| Service | Category | Use Case |
|---|---|---|
| AWS Lambda | Compute | Event-driven functions |
| Amazon API Gateway | API Management | RESTful/WebSocket APIs |
| AWS Fargate | Container Compute | Serverless containers |
| Amazon DynamoDB | Database | Serverless NoSQL |
| Amazon Aurora Serverless | Database | Serverless relational DB |
| Amazon S3 | Storage | Object storage |
| AWS Step Functions | Orchestration | Workflow coordination |
| Amazon EventBridge | Event Bus | Event-driven architecture |
| Amazon SQS/SNS | Messaging | Decoupled communication |

---

## Hard

### 1. Deep dive: How does Amazon DynamoDB achieve its performance guarantees at scale? Explain partition keys, hot partitions, and how to design for scale.

**Answer:**
**DynamoDB's Architecture:**
DynamoDB is a fully managed, multi-Region, multi-active key-value and document database. It achieves single-digit millisecond performance at any scale through a combination of consistent hashing, distributed storage, and adaptive capacity.

**How DynamoDB Partitions Data:**
- Data is distributed across multiple storage nodes using consistent hashing on the partition key.
- Each partition can handle up to **3,000 RCUs** (Read Capacity Units) and **1,000 WCUs** (Write Capacity Units) and store up to **10 GB** of data.
- DynamoDB automatically splits partitions when these limits are approached.

**The Hot Partition Problem:**
A hot partition occurs when a disproportionate amount of traffic hits a single partition. This happens when:
- The partition key has low cardinality (e.g., using `status` with values `active/inactive`)
- Traffic is concentrated on a small number of keys (e.g., a viral item in an e-commerce system)
- Temporal patterns cause bursts (e.g., all writes use a date-based key)

**Consequences:** Throttling errors (`ProvisionedThroughputExceededException`) even when overall table capacity is sufficient, because limits are per-partition.

**Design Strategies to Avoid Hot Partitions:**

1. **High-Cardinality Partition Keys:** Use UUIDs, user IDs, or device IDs rather than categorical values.

2. **Write Sharding:** Append a random suffix (1–N) to the partition key to distribute writes across N logical partitions. When reading, query all N shards and aggregate results.
   ```
   # Instead of: pk = "PRODUCT#123"
   # Use: pk = "PRODUCT#123#shard-" + random(1, 10)
   ```

3. **Composite Keys:** Combine multiple attributes to increase cardinality.

4. **Caching:** Place ElastiCache (DAX for DynamoDB specifically) in front of DynamoDB to absorb read traffic for popular items. DynamoDB Accelerator (DAX) provides microsecond latency for cached reads.

5. **Adaptive Capacity:** DynamoDB's built-in feature that automatically redistributes capacity to frequently accessed partitions. However, this is reactive and not a substitute for good key design.

6. **Time-Series Patterns:** For time-series data, use table-per-period strategy (e.g., separate tables for each month) to avoid one massive partition and enable efficient deletion of old data via table deletion rather than expensive scan-and-delete.

**Access Pattern-Driven Design:**
DynamoDB requires you to know your access patterns upfront. Use a single-table design where possible:
- Store multiple entity types in one table
- Use overloaded partition keys (e.g., `PK = USER#123`, `SK = PROFILE`, `SK = ORDER#456`)
- Use Global Secondary Indexes (GSIs) to support alternative query patterns
- GSIs have their own partition/sort key and can have different throughput settings

**Capacity Modes:**
- **Provisioned:** Specify RCUs and WCUs; use Auto Scaling; cheaper for predictable workloads
- **On-Demand:** Pay-per-request; DynamoDB scales automatically; better for unpredictable workloads (but ~7x more expensive per operation at steady state)

---

### 2. Explain the CAP theorem and its implications for distributed cloud systems. How do AWS services make trade-offs?

**Answer:**
**CAP Theorem**