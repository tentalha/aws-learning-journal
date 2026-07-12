# Cloud Computing

## What is it?

Cloud computing is the on-demand delivery of IT resources — including compute power, storage, databases, networking, analytics, machine learning, and more — over the internet with pay-as-you-go pricing. Rather than owning and maintaining physical data centers and servers, organizations can access technology services from a cloud provider such as Amazon Web Services (AWS) on an as-needed basis.

AWS defines cloud computing across three primary service models:

| Model | Full Name | Description |
|-------|-----------|-------------|
| **IaaS** | Infrastructure as a Service | Provides virtualized computing resources over the internet (EC2, S3, VPC) |
| **PaaS** | Platform as a Service | Provides a platform for developing and deploying applications (Elastic Beanstalk, RDS) |
| **SaaS** | Software as a Service | Delivers software applications over the internet (Amazon WorkMail, Chime) |

And across four deployment models:

- **Public Cloud** — Resources owned and operated by AWS, shared across multiple tenants
- **Private Cloud** — Resources used exclusively by a single organization (AWS Outposts)
- **Hybrid Cloud** — Combination of on-premises infrastructure with public cloud (AWS Direct Connect + VPC)
- **Multi-Cloud** — Use of multiple cloud providers simultaneously

AWS is the world's most comprehensive and broadly adopted cloud platform, offering over **200 fully featured services** from data centers globally across **33 geographic Regions** (as of 2024) and **105+ Availability Zones**.

---

## Why do we need it?

### Problems with Traditional On-Premises Infrastructure

Before cloud computing, organizations faced significant challenges:

- **Massive upfront capital expenditure (CapEx)** — Purchasing servers, networking equipment, and data center space required millions of dollars before a single line of code ran in production.
- **Capacity planning dilemmas** — Organizations either over-provisioned (wasting money) or under-provisioned (causing outages during peak traffic).
- **Long procurement cycles** — Ordering, shipping, racking, and configuring physical hardware took weeks to months.
- **Operational burden** — Teams spent enormous time on undifferentiated heavy lifting: patching OS, replacing failed drives, managing cooling systems.
- **Limited geographic reach** — Expanding globally required establishing physical presence in new regions — extremely costly and slow.
- **Disaster recovery complexity** — Maintaining a secondary data center for DR was prohibitively expensive for most businesses.

### Business Scenarios Where Cloud Computing is Essential

**Scenario 1: E-Commerce Startup**
A startup building an online marketplace cannot afford $500,000 in hardware. With AWS, they launch with EC2 instances for compute, RDS for database, and S3 for product images — paying only for what they use, scaling from 10 to 10 million users without re-architecting.

**Scenario 2: Seasonal Traffic Spikes**
A retail company experiences 10x traffic during Black Friday. On-premises, they'd need to permanently own 10x capacity. With AWS Auto Scaling, they scale up for 48 hours and scale back down — paying only for the burst period.

**Scenario 3: Global Expansion**
A SaaS company wants to serve European customers with low latency. Instead of building a European data center (12+ months, $2M+), they deploy to AWS `eu-west-1` (Ireland) in hours.

**Scenario 4: Machine Learning Workloads**
A data science team needs GPU clusters for model training — but only for 72-hour training runs. AWS EC2 P4d instances (NVIDIA A100 GPUs) are available on-demand with no long-term commitment.

**Scenario 5: Disaster Recovery**
A financial institution needs a DR strategy with RPO < 1 hour and RTO < 4 hours. AWS enables pilot light or warm standby DR architectures at a fraction of traditional DR costs.

---

## Internal Working

### Virtualization Layer

At its core, AWS cloud computing relies on **hardware virtualization** — specifically the **AWS Nitro System**, a purpose-built collection of hardware and software that underpins modern EC2 instances.

```
┌─────────────────────────────────────────────────────────────┐
│                    Physical Host Server                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              AWS Nitro Hypervisor                    │   │
│  │  (Lightweight, security-focused, near bare-metal)   │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐   │
│  │  EC2 Instance│ │  EC2 Instance│ │   EC2 Instance   │   │
│  │  (Customer A)│ │  (Customer B)│ │   (Customer C)   │   │
│  │  vCPU: 4    │ │  vCPU: 8    │ │   vCPU: 16       │   │
│  │  RAM: 16GB  │ │  RAM: 32GB  │ │   RAM: 64GB      │   │
│  └──────────────┘ └──────────────┘ └──────────────────┘   │
│                                                              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐   │
│  │  Nitro Card  │ │  Nitro Card  │ │   Nitro Card     │   │
│  │  (Networking)│ │  (Storage)   │ │   (Security)     │   │
│  └──────────────┘ └──────────────┘ └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### AWS Nitro System Components

1. **Nitro Hypervisor** — A minimalist hypervisor that provides CPU and memory isolation with near bare-metal performance. Unlike traditional hypervisors (Xen), Nitro offloads most virtualization functions to dedicated hardware.

2. **Nitro Cards** — Custom ASICs (Application-Specific Integrated Circuits) that handle:
   - **Nitro Card for VPC** — All VPC networking (packet processing, security groups, encryption)
   - **Nitro Card for EBS** — Storage I/O processing, encryption
   - **Nitro Security Chip** — Hardware root of trust, firmware verification

3. **Nitro Enclaves** — Isolated compute environments for processing sensitive data with cryptographic attestation.

### Global Infrastructure Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Global Network                        │
│                    (Private fiber backbone)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              AWS Region (e.g., us-east-1)                │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌──────────┐ │  │
│  │  │  AZ: us-east-1a │  │  AZ: us-east-1b │  │  AZ: 1c  │ │  │
│  │  │  ┌───────────┐  │  │  ┌───────────┐  │  │ ┌──────┐ │ │  │
│  │  │  │ Data      │  │  │  │ Data      │  │  │ │ Data │ │ │  │
│  │  │  │ Center 1  │  │  │  │ Center 3  │  │  │ │Ctr 5 │ │ │  │
│  │  │  ├───────────┤  │  │  ├───────────┤  │  │ ├──────┤ │ │  │
│  │  │  │ Data      │  │  │  │ Data      │  │  │ │ Data │ │ │  │
│  │  │  │ Center 2  │  │  │  │ Center 4  │  │  │ │Ctr 6 │ │ │  │
│  │  │  └───────────┘  │  │  └───────────┘  │  │ └──────┘ │ │  │
│  │  └─────────────────┘  └─────────────────┘  └──────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ Edge Location│  │ Edge Location│  │  Regional Edge Cache    │ │
│  │ (CloudFront) │  │ (CloudFront) │  │  (CloudFront)          │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### How Resource Provisioning Works

1. **API Request** — User makes an API call (via Console, CLI, SDK, or IaC) to AWS control plane
2. **Authentication & Authorization** — AWS IAM validates identity and permissions
3. **Control Plane Processing** — AWS service control plane receives the request, validates parameters, checks quotas
4. **Resource Scheduling** — For compute, the placement engine selects an appropriate physical host based on capacity, placement groups, and constraints
5. **Hypervisor Configuration** — Nitro hypervisor creates an isolated VM with allocated vCPUs and memory
6. **Network Configuration** — VPC networking is configured via Nitro networking cards
7. **Storage Attachment** — EBS volumes are attached via Nitro storage cards over NVMe
8. **Instance Boot** — AMI (Amazon Machine Image) is loaded and the instance boots
9. **Data Plane Operation** — Once running, the instance operates on the data plane with customer workloads

---

## Architecture

### Core Architectural Pillars

#### 1. Regions and Availability Zones

```
                    ┌─────────────────────────────┐
                    │     AWS Region               │
                    │   (Geographic area)          │
                    │                              │
                    │  ┌──────┐  ┌──────┐  ┌────┐ │
                    │  │ AZ-1 │  │ AZ-2 │  │AZ-3│ │
                    │  │      │──│      │──│    │ │
                    │  │  DC  │  │  DC  │  │ DC │ │
                    │  └──────┘  └──────┘  └────┘ │
                    │   Low-latency private links   │
                    └─────────────────────────────┘
```

- **Region**: A physical location in the world with multiple, isolated, and physically separate AZs
- **Availability Zone (AZ)**: One or more discrete data centers with redundant power, networking, and connectivity
- **AZs are separated by meaningful distances** (miles apart) but connected with high-bandwidth, ultra-low-latency networking
- **Local Zones**: Extensions of AWS Regions that place compute, storage, and other services closer to large population centers
- **Wavelength Zones**: AWS infrastructure deployed within telecommunications providers' data centers for 5G edge computing

#### 2. Foundational Service Categories

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS Cloud Platform                            │
├─────────────┬──────────────┬──────────────┬──────────────────────── ┤
│   Compute   │   Storage    │  Networking  │      Database           │
│  ─────────  │  ─────────   │  ──────────  │  ──────────────────     │
│  EC2        │  S3          │  VPC         │  RDS                    │
│  Lambda     │  EBS         │  CloudFront  │  DynamoDB               │
│  ECS        │  EFS         │  Route 53    │  Aurora                 │
│  EKS        │  Glacier     │  Direct Conn │  ElastiCache            │
│  Fargate    │  FSx         │  API Gateway │  Redshift               │
├─────────────┴──────────────┴──────────────┴─────────────────────────┤
│   Security  │  Analytics   │  ML/AI       │  Developer Tools        │
│  ─────────  │  ─────────   │  ──────────  │  ──────────────────     │
│  IAM        │  Athena      │  SageMaker   │  CodePipeline           │
│  KMS        │  EMR         │  Rekognition │  CodeBuild              │
│  WAF        │  Kinesis     │  Bedrock     │  CodeDeploy             │
│  Shield     │  Glue        │  Comprehend  │  CloudFormation         │
│  GuardDuty  │  QuickSight  │  Polly       │  CDK                    │
└─────────────┴──────────────┴──────────────┴─────────────────────────┘
```

#### 3. Well-Architected Framework Pillars

The AWS Well-Architected Framework provides a consistent approach for evaluating cloud architectures:

| Pillar | Focus |
|--------|-------|
| **Operational Excellence** | Run and monitor systems to deliver business value |
| **Security** | Protect information, systems, and assets |
| **Reliability** | Recover from failures and meet demand |
| **Performance Efficiency** | Use computing resources efficiently |
| **Cost Optimization** | Avoid unnecessary costs |
| **Sustainability** | Minimize environmental impacts |

#### 4. Shared Responsibility Model

```
┌─────────────────────────────────────────────────────────┐
│               CUSTOMER RESPONSIBILITY                    │
│         "Security IN the Cloud"                         │
│                                                         │
│  • Customer Data                                        │
│  • Platform, Applications, IAM                          │
│  • OS, Network, Firewall Configuration                  │
│  • Client-side & Server-side Encryption                 │
│  • Network Traffic Protection                           │
├─────────────────────────────────────────────────────────┤
│                  AWS RESPONSIBILITY                      │
│            "Security OF the Cloud"                      │
│                                                         │
│  • Compute, Storage, Database, Networking               │
│  • Hardware / AWS Global Infrastructure                 │
│  • Regions, AZs, Edge Locations                         │
│  • Physical security of data centers                    │
└─────────────────────────────────────────────────────────┘
```

---

## Real World Example

### Scenario: Migrating a Traditional E-Commerce Platform to AWS

**Company**: RetailCo — A mid-size retailer with an on-premises monolithic application serving 50,000 daily users, struggling with Black Friday traffic spikes.

#### Current State (On-Premises)
- 10 physical web servers, 2 database servers
- Single data center in Chicago
- 4-hour planned maintenance windows monthly
- Black Friday: site crashes every year
- DR: none (too expensive)

#### Target State (AWS Architecture)

```
                          ┌─────────────────┐
                          │   Route 53      │
                          │  (DNS + Health  │
                          │   Checking)     │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │   CloudFront    │
                          │  (CDN + WAF)    │
                          └────────┬────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │     Application Load         │
                    │     Balancer (ALB)           │
                    └──────┬──────────────┬────────┘
                           │              │
              ┌────