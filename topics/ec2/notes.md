# EC2

## What is it?

**Amazon Elastic Compute Cloud (Amazon EC2)** is a core AWS compute service that provides resizable, on-demand virtual server instances in the cloud. It belongs to the **Compute** category of AWS services.

EC2 allows you to launch virtual machines (called *instances*) running Linux, Windows, or macOS, with full control over the operating system, networking, storage, and security. Each instance runs on physical hardware managed by AWS within a specific Availability Zone (AZ) in an AWS Region.

EC2 is the foundational building block of most AWS architectures — it underpins services like Elastic Beanstalk, ECS on EC2, and EMR.

---

## Why do we need it?

### The Problem It Solves
Traditional on-premises infrastructure requires upfront capital expenditure, long procurement cycles, and manual capacity planning. Businesses either over-provision (waste money) or under-provision (face outages during traffic spikes).

EC2 solves this by providing:
- **Elastic capacity**: Launch or terminate instances in minutes
- **Pay-as-you-go**: Only pay for what you use
- **Global reach**: Deploy instances in any AWS Region worldwide
- **Full control**: Root/Administrator access to the OS

### When to Use EC2

| Scenario | Reason |
|---|---|
| Custom application servers | Need full OS control and custom runtimes |
| Legacy application lift-and-shift | App cannot be containerized or refactored |
| High-performance computing (HPC) | Specialized hardware (GPU, high memory) |
| Database hosting (self-managed) | Need specific DB configs not available in RDS |
| Dev/Test environments | Spin up and tear down on demand |
| Batch processing workloads | Run large jobs on demand, terminate after |

### Real Business Scenarios
- An e-commerce platform uses EC2 Auto Scaling to handle Black Friday traffic spikes without manual intervention
- A media company runs GPU-powered EC2 instances (p4d) for video transcoding workloads
- A fintech startup migrates its on-prem banking application to EC2 as the first step of cloud migration

---

## Internal Working

### Virtualization Layer
EC2 uses the **AWS Nitro System** — a purpose-built hypervisor and hardware offload platform. Nitro offloads network, storage, and security functions to dedicated hardware, leaving nearly all host resources for the customer instance.

```
┌─────────────────────────────────────────────────────────┐
│                    Physical Host (Bare Metal)            │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  EC2 Inst 1 │  │  EC2 Inst 2 │  │  EC2 Inst 3 │    │
│  │  (Customer) │  │  (Customer) │  │  (Customer) │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         │                │                │             │
│  ┌──────▼────────────────▼────────────────▼──────┐     │
│  │              Nitro Hypervisor                  │     │
│  └──────────────────────────────────────────────┘     │
│                                                         │
│  ┌───────────────┐  ┌───────────────┐                  │
│  │  Nitro Card   │  │  Nitro Card   │                  │
│  │  (Networking) │  │  (Storage/EBS)│                  │
│  └───────────────┘  └───────────────┘                  │
└─────────────────────────────────────────────────────────┘
```

### Instance Launch Flow
1. **API Request**: User calls `RunInstances` API (via Console, CLI, or SDK)
2. **Scheduler**: AWS placement scheduler selects an appropriate physical host
3. **AMI Boot**: The Amazon Machine Image (AMI) is used to create a root volume snapshot on EBS or instance store
4. **Network Attachment**: An Elastic Network Interface (ENI) is attached; a private IP is assigned from the subnet CIDR
5. **Security Groups**: Virtual firewall rules are applied at the hypervisor level
6. **Instance Metadata Service (IMDS)**: Available at `169.254.169.254` for instance metadata and IAM credentials
7. **User Data**: Bootstrap scripts run at first launch via `cloud-init`

### Instance States
```
Pending → Running → Stopping → Stopped → Terminated
                 ↘ Rebooting ↗
```

- **Pending**: Instance is being provisioned (billed not yet started)
- **Running**: Instance is active (billing begins)
- **Stopping**: Transitioning to stopped (EBS-backed only)
- **Stopped**: Instance halted, EBS persists (no compute billing, EBS still billed)
- **Terminated**: Instance deleted, root volume deleted by default

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS Region                               │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                        VPC                               │   │
│  │                                                          │   │
│  │  ┌──────────────────┐    ┌──────────────────┐           │   │
│  │  │  Public Subnet   │    │  Private Subnet  │           │   │
│  │  │  (AZ-1a)         │    │  (AZ-1b)         │           │   │
│  │  │                  │    │                  │           │   │
│  │  │  ┌────────────┐  │    │  ┌────────────┐  │           │   │
│  │  │  │  EC2 (Web) │  │    │  │  EC2 (App) │  │           │   │
│  │  │  │  + EIP     │  │    │  │            │  │           │   │
│  │  │  └────────────┘  │    │  └─────┬──────┘  │           │   │
│  │  └──────────────────┘    └────────┼─────────┘           │   │
│  │                                   │                      │   │
│  │  ┌────────────────────────────────▼─────────────────┐   │   │
│  │  │              EBS Volumes / EFS / S3               │   │   │
│  │  └───────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

| Component | Description |
|---|---|
| **AMI (Amazon Machine Image)** | Template containing OS, app, and config. Acts as a snapshot blueprint for instances |
| **Instance Type** | Defines vCPU, memory, network, and storage (e.g., `t3.medium`, `m6i.xlarge`) |
| **EBS (Elastic Block Store)** | Persistent block storage attached to instances; survives instance stop |
| **Security Groups** | Stateful virtual firewall at the instance level |
| **Key Pairs** | RSA/ED25519 key pairs for SSH (Linux) or RDP password decryption (Windows) |
| **Elastic IP (EIP)** | Static public IPv4 address that can be reassigned between instances |
| **ENI (Elastic Network Interface)** | Virtual network card; instances can have multiple ENIs |
| **Placement Groups** | Logical grouping strategies: Cluster, Spread, Partition |
| **User Data** | Bootstrap script executed on first launch |
| **Instance Metadata** | Runtime data accessible at `169.254.169.254` |

### Instance Type Families

| Family | Use Case | Example |
|---|---|---|
| **t** (Burstable) | Dev/test, low-traffic web | t3.micro, t4g.small |
| **m** (General Purpose) | Balanced workloads | m6i.xlarge, m7g.2xlarge |
| **c** (Compute Optimized) | CPU-intensive: gaming, ML inference | c7g.4xlarge |
| **r** (Memory Optimized) | In-memory DBs, big data analytics | r6i.8xlarge |
| **p/g** (GPU) | ML training, graphics rendering | p4d.24xlarge, g5.xlarge |
| **i** (Storage Optimized) | High IOPS, NoSQL databases | i4i.4xlarge |
| **hpc** (HPC) | Tightly coupled HPC | hpc7g.16xlarge |

### Placement Groups

```
Cluster Placement Group         Spread Placement Group
┌──────────────────┐            ┌──────────────────┐
│  Single AZ       │            │  Multiple AZs    │
│  Low latency     │            │  Max 7/AZ        │
│  High throughput │            │  Fault isolated  │
│  ┌──┐ ┌──┐ ┌──┐  │            │ ┌──┐  ┌──┐  ┌──┐ │
│  │  │ │  │ │  │  │            │ │  │  │  │  │  │ │
│  └──┘ └──┘ └──┘  │            │ └──┘  └──┘  └──┘ │
└──────────────────┘            └──────────────────┘
```

---

## Real World Example

### Scenario: Three-Tier Web Application Deployment

A retail company wants to deploy a scalable, highly available e-commerce application on AWS.

#### Step-by-Step Walkthrough

**Step 1: Network Setup**
```
- Create VPC: 10.0.0.0/16
- Public Subnets: 10.0.1.0/24 (AZ-1a), 10.0.2.0/24 (AZ-1b)  → Load Balancer
- Private App Subnets: 10.0.3.0/24 (AZ-1a), 10.0.4.0/24 (AZ-1b) → App Servers
- Private DB Subnets: 10.0.5.0/24 (AZ-1a), 10.0.6.0/24 (AZ-1b) → RDS
```

**Step 2: AMI Creation**
```
1. Launch a base Amazon Linux 2023 instance
2. Install Node.js, application code, nginx
3. Configure systemd services
4. Create a custom AMI: ami-ecommerce-v1.2
```

**Step 3: Launch Template**
```
- Instance type: m6i.large (2 vCPU, 8 GB RAM)
- AMI: ami-ecommerce-v1.2
- IAM Role: EC2AppRole (S3 read, Secrets Manager access)
- Security Group: allow 8080 from ALB SG only
- User Data: script to pull config from SSM Parameter Store
- EBS: 30 GB gp3 root volume, encrypted
```

**Step 4: Auto Scaling Group**
```
- Min: 2, Desired: 4, Max: 20
- Target Tracking: CPU 60%
- Lifecycle hooks for graceful shutdown
- Spread across 2 AZs
```

**Step 5: Load Balancer**
```
- ALB in public subnets
- Target Group: EC2 instances on port 8080
- Health check: GET /health → 200 OK
- HTTPS listener with ACM certificate
```

**Step 6: Deployment Flow**
```
User → Route 53 → ALB → EC2 App Instances → RDS Aurora
                              ↓
                         ElastiCache (session)
                              ↓
                         S3 (static assets)
```

**Result**: The application automatically scales from 2 to 20 instances based on CPU load, with no single point of failure across two Availability Zones.

---

## Advantages

- **Elasticity**: Scale up or down in minutes; no hardware procurement
- **Wide Instance Selection**: 500+ instance types covering virtually every workload profile
- **Full OS Control**: Root/Administrator access; install any software, custom kernels
- **Flexible Pricing Models**: On-Demand, Reserved, Spot, Savings Plans — optimize for any budget
- **Global Infrastructure**: Available in 30+ Regions, 90+ AZs
- **Nitro System Performance**: Near bare-metal performance with hardware-level security isolation
- **Rich Ecosystem**: Deep integration with 200+ AWS services
- **Mature Tooling**: Extensive CLI, SDK, CloudFormation, Terraform, CDK support
- **Graviton (ARM) Instances**: Up to 40% better price/performance vs x86 for compatible workloads
- **Dedicated Hardware Options**: Dedicated Instances and Dedicated Hosts for compliance requirements
- **Hibernate Support**: Preserve in-memory state across stops (for supported instance types)

---

## Limitations

| Limitation | Detail |
|---|---|
| **Default vCPU limit** | 32–96 vCPUs per Region per instance family (soft limit, can request increase) |
| **EIP limit** | 5 Elastic IPs per Region (soft limit) |
| **Security Groups per ENI** | 5 security groups per ENI (soft limit, max 16) |
| **Rules per Security Group** | 60 inbound + 60 outbound (soft limit) |
| **ENIs per instance** | Varies by instance type (e.g., t3.large = 3 ENIs) |
| **Instance Store** | Ephemeral — data lost on stop/terminate/hardware failure |
| **EBS Volume Limit** | 40 EBS volumes per instance (soft limit) |
| **Placement Group (Spread)** | Max 7 running instances per AZ |
| **User Data Size** | Max 16 KB (base64 encoded) |
| **AMI Limit** | 200 AMIs per Region (soft limit) |
| **Snapshot Limit** | 100,000 EBS snapshots per account |
| **No live migration** | AWS may need to stop/start instances for hardware maintenance |
| **IPv4 charges** | Public IPv4 addresses now cost $0.005/hr (as of Feb 2024) |

---

## Best Practices

### Security
- **Never use root credentials** — always use IAM roles attached to instances
- **Use IMDSv2** (Instance Metadata Service v2) — requires session-oriented requests, prevents SSRF attacks
- **Disable public IPs** on private instances; use NAT Gateway or VPC endpoints instead
- **Encrypt EBS volumes** by default (enable account-level default encryption)
- **Patch regularly** — use AWS Systems Manager Patch Manager for automated patching

### Reliability (AWS Well-Architected)
- **Deploy across multiple AZs** — minimum 2 AZs for any production workload
- **Use Auto Scaling Groups** even for single instances to enable automatic replacement
- **Implement health checks** at both ALB and ASG levels
- **Use Lifecycle Hooks** for graceful instance termination (drain connections, flush logs)

### Performance
- **Right-size instances** — use AWS Compute Optimizer recommendations
- **Use Graviton (ARM) instances** (m7g, c7g) for up to 40% better price-performance
- **Enable Enhanced Networking** (ENA) — available on most current generation instances
- **Use gp3 EBS volumes** instead of gp2 — better baseline IOPS at lower cost
- **Use placement groups** for latency-sensitive or high-throughput workloads

### Cost Optimization
- **Use Savings Plans or Reserved Instances** for predictable baseline workloads (1–3 year terms)
- **Use Spot Instances** for fault-tolerant, flexible workloads (up to 90% discount)
- **Set up Instance Scheduler** to stop dev/test instances outside business hours
- **Use AWS Compute Optimizer** to identify over-provisioned instances
- **