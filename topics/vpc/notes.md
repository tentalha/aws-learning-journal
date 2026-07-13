# VPC

## What is it?

**Amazon Virtual Private Cloud (Amazon VPC)** is a foundational AWS networking service that enables you to provision a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define. It falls under the **Networking & Content Delivery** category of AWS services.

A VPC closely resembles a traditional on-premises data center network but with the benefits of AWS's scalable infrastructure. You have complete control over your virtual networking environment, including:

- Selection of your own IP address range (IPv4 and IPv6)
- Creation of subnets
- Configuration of route tables and network gateways
- Security controls at multiple layers

VPC is a **regional service** — each VPC spans all Availability Zones within a single AWS Region, and you can create multiple VPCs per region (default limit: 5 per region, soft limit).

---

## Why do we need it?

### The Problem It Solves

Without VPC, all AWS resources would exist in a flat, shared network space — exposed to other customers and the public internet. VPC provides **network isolation, security segmentation, and controlled connectivity** for your cloud infrastructure.

### Core Problems Addressed

| Problem | VPC Solution |
|---|---|
| Public exposure of internal resources | Private subnets with no internet route |
| Lack of network segmentation | Multiple subnets across AZs |
| Uncontrolled traffic flow | Security Groups + NACLs |
| No private connectivity to on-premises | VPN Gateway / Direct Connect |
| Shared network with other AWS customers | Logical isolation via software-defined networking |

### Real Business Scenarios

1. **E-commerce Platform**: Web servers in public subnets serve customer traffic, while database servers in private subnets are completely isolated from the internet. Only the application tier can communicate with the database tier.

2. **Financial Services**: A bank needs strict network segmentation between its trading platform, customer portal, and internal analytics — each in separate VPCs with controlled peering.

3. **Healthcare SaaS**: HIPAA-compliant architecture requires that patient data never traverses the public internet. Private subnets + VPC endpoints + Direct Connect ensure all traffic stays on private networks.

4. **Hybrid Cloud**: A manufacturing company runs legacy ERP on-premises and migrates workloads incrementally to AWS, connecting both environments through a Site-to-Site VPN or AWS Direct Connect via a VPC.

5. **Multi-tenant SaaS**: Each enterprise customer gets their own isolated VPC, preventing cross-tenant data leakage.

---

## Internal Working

### Software-Defined Networking

Amazon VPC is built on **software-defined networking (SDN)** principles. Unlike physical network hardware, VPC uses virtualized networking components managed by AWS's Nitro hypervisor and custom network ASICs.

### How Traffic Flows Internally

```
Internet
    │
    ▼
Internet Gateway (IGW)
    │
    ▼
Router (Implicit VPC Router)
    │
    ├──► Route Table Evaluation
    │         │
    │    ┌────▼────┐    ┌─────────────┐
    │    │ Public  │    │  Private    │
    │    │ Subnet  │    │  Subnet     │
    │    └────┬────┘    └──────┬──────┘
    │         │                │
    │    Security Group   Security Group
    │    NACL Check       NACL Check
    │         │                │
    │    EC2 Instance     RDS Instance
    │
    └──► NAT Gateway (for private subnet outbound)
```

### Key Internal Mechanisms

1. **Implicit Router**: Every VPC has an invisible router that handles all routing decisions. You configure it via Route Tables but never manage the router directly.

2. **ENI (Elastic Network Interface)**: Every EC2 instance gets at least one ENI. Traffic to/from instances is filtered at the ENI level by Security Groups (stateful) and at the subnet boundary by NACLs (stateless).

3. **DHCP Option Sets**: VPC uses DHCP to assign IP addresses, DNS servers, and domain names to instances automatically.

4. **DNS Resolution**: VPC provides an internal DNS resolver at `169.254.169.253` (or the VPC base CIDR + 2, e.g., `10.0.0.2` for `10.0.0.0/16`). This resolves both AWS internal DNS names and public DNS.

5. **Packet Encapsulation**: VPC uses network virtualization that encapsulates packets at the hypervisor level. Physical servers don't see your VPC addresses — only the underlying AWS infrastructure IPs are visible on physical switches.

6. **ARP and MAC Spoofing Prevention**: VPC enforces that instances can only send/receive traffic from their assigned IPs and MACs, preventing ARP poisoning and IP spoofing attacks.

7. **Flow Logs**: VPC captures metadata about IP traffic using a sampling mechanism and stores it in CloudWatch Logs or S3.

---

## Architecture

### Core Architectural Components

```
AWS Region
└── VPC (10.0.0.0/16)
    ├── Availability Zone A
    │   ├── Public Subnet (10.0.1.0/24)
    │   │   ├── EC2 (Web Tier)
    │   │   └── NAT Gateway
    │   └── Private Subnet (10.0.2.0/24)
    │       ├── EC2 (App Tier)
    │       └── RDS (DB Tier)
    ├── Availability Zone B
    │   ├── Public Subnet (10.0.3.0/24)
    │   │   └── EC2 (Web Tier)
    │   └── Private Subnet (10.0.4.0/24)
    │       ├── EC2 (App Tier)
    │       └── RDS (DB Replica)
    ├── Internet Gateway
    ├── Virtual Private Gateway (VPN)
    ├── Route Tables
    │   ├── Public Route Table
    │   └── Private Route Table
    ├── Security Groups
    ├── Network ACLs
    └── VPC Endpoints
        ├── Gateway Endpoint (S3, DynamoDB)
        └── Interface Endpoint (SSM, Secrets Manager, etc.)
```

### Component Details

#### 1. CIDR Blocks
- **Primary CIDR**: IPv4 block assigned at VPC creation (e.g., `10.0.0.0/16`)
- Allowed block sizes: `/16` (65,536 IPs) to `/28` (16 IPs)
- You can add up to **5 secondary CIDR blocks**
- IPv6 CIDR: `/56` block assigned by AWS (you cannot choose the range)
- AWS reserves **5 IPs per subnet**: network address, VPC router, DNS, future use, broadcast

#### 2. Subnets
- Subnets are **AZ-specific** (one AZ per subnet, but multiple subnets per AZ)
- **Public Subnet**: Has a route to an Internet Gateway
- **Private Subnet**: No direct route to IGW; uses NAT Gateway for outbound
- **Isolated Subnet**: No internet access whatsoever

#### 3. Internet Gateway (IGW)
- Horizontally scaled, redundant, highly available
- Performs **NAT for instances with public IPs** (translates private IP to public IP)
- One IGW per VPC maximum
- Free to attach; you pay for data transfer

#### 4. NAT Gateway
- Managed service in a public subnet
- Allows private subnet instances to initiate outbound internet traffic
- Does **not** allow inbound initiated connections from internet
- **AZ-specific** — deploy one per AZ for HA
- Supports up to 45 Gbps bandwidth

#### 5. Route Tables
- Each subnet must be associated with exactly one route table
- Main route table is created automatically; custom route tables can be created
- Routes evaluated by **longest prefix match** (most specific wins)

#### 6. Security Groups
- **Stateful** firewall at the instance/ENI level
- Only allow rules (no explicit deny)
- Default SG: allows all outbound, allows inbound from same SG
- Can reference other security groups as sources

#### 7. Network ACLs (NACLs)
- **Stateless** firewall at the subnet level
- Rules evaluated in **number order** (lowest first), first match wins
- Supports both allow and deny rules
- Default NACL: allows all inbound and outbound

#### 8. VPC Endpoints
- **Gateway Endpoints**: S3 and DynamoDB — route table entries, free
- **Interface Endpoints (AWS PrivateLink)**: ENI with private IP — hourly + data charge
- **Gateway Load Balancer Endpoints**: For third-party appliances

#### 9. VPC Peering
- One-to-one connection between two VPCs (same or different accounts/regions)
- Non-transitive: A↔B and B↔C does not mean A↔C
- No overlapping CIDR blocks allowed
- Intra-region peering is free; inter-region peering incurs data transfer charges

#### 10. Transit Gateway
- Hub-and-spoke model connecting multiple VPCs and on-premises networks
- Transitive routing supported
- Scales to thousands of VPCs

### Architecture Patterns

**Three-Tier Web Application Pattern**
```
Public Subnet:    ALB → (routes to App Tier)
Private Subnet 1: EC2 Auto Scaling Group (App Tier)
Private Subnet 2: RDS Multi-AZ (Data Tier)
```

**Hub-and-Spoke Pattern**
```
Transit Gateway (Hub)
├── Shared Services VPC
├── Production VPC
├── Development VPC
└── On-Premises (via VPN/Direct Connect)
```

---

## Real World Example

### Scenario: Multi-Tier E-Commerce Application

**Requirements**:
- High availability across 2 AZs
- Web tier publicly accessible
- Application and database tiers isolated
- Private access to S3 for product images
- Secure admin access without bastion host

#### Step-by-Step Walkthrough

**Step 1: Create the VPC**
```
VPC CIDR: 10.0.0.0/16
Region: us-east-1
DNS Hostnames: Enabled
DNS Resolution: Enabled
```

**Step 2: Create Subnets**
```
Public Subnet AZ-A:   10.0.1.0/24 (us-east-1a)
Public Subnet AZ-B:   10.0.2.0/24 (us-east-1b)
App Subnet AZ-A:      10.0.11.0/24 (us-east-1a)
App Subnet AZ-B:      10.0.12.0/24 (us-east-1b)
DB Subnet AZ-A:       10.0.21.0/24 (us-east-1a)
DB Subnet AZ-B:       10.0.22.0/24 (us-east-1b)
```

**Step 3: Create and Attach Internet Gateway**
```
IGW: ecommerce-igw → Attach to VPC
```

**Step 4: Create NAT Gateways (one per AZ for HA)**
```
NAT-GW-AZ-A: Elastic IP in Public Subnet AZ-A
NAT-GW-AZ-B: Elastic IP in Public Subnet AZ-B
```

**Step 5: Configure Route Tables**
```
Public Route Table:
  10.0.0.0/16 → local
  0.0.0.0/0   → IGW
  Associate: Public Subnet AZ-A, AZ-B

Private Route Table AZ-A:
  10.0.0.0/16 → local
  0.0.0.0/0   → NAT-GW-AZ-A
  pl-xxxxx     → Gateway Endpoint (S3)
  Associate: App Subnet AZ-A, DB Subnet AZ-A

Private Route Table AZ-B:
  10.0.0.0/16 → local
  0.0.0.0/0   → NAT-GW-AZ-B
  pl-xxxxx     → Gateway Endpoint (S3)
  Associate: App Subnet AZ-B, DB Subnet AZ-B
```

**Step 6: Create Security Groups**
```
sg-alb (ALB Security Group):
  Inbound: 443 from 0.0.0.0/0
  Outbound: 8080 to sg-app

sg-app (Application Security Group):
  Inbound: 8080 from sg-alb
  Outbound: 5432 to sg-db, 443 to 0.0.0.0/0 (for NAT)

sg-db (Database Security Group):
  Inbound: 5432 from sg-app
  Outbound: None (or stateful allows responses)
```

**Step 7: Create VPC Endpoints**
```
Gateway Endpoint: S3 → Add to both private route tables
Interface Endpoint: SSM (Systems Manager) → for admin access
Interface Endpoint: SSM Messages → for Session Manager
Interface Endpoint: EC2 Messages → for Session Manager
```

**Step 8: Deploy Resources**
```
ALB → Public Subnets (AZ-A, AZ-B) with sg-alb
EC2 Auto Scaling Group → App Subnets with sg-app
RDS Multi-AZ → DB Subnets with sg-db
```

**Result**: 
- Customers access the application via ALB HTTPS
- App servers communicate with RDS on private network
- App servers access S3 via Gateway Endpoint (no internet)
- Admins access EC2 via SSM Session Manager (no bastion, no public IPs)
- All database traffic stays entirely within the VPC

---

## Advantages

1. **Complete Network Control**: Define IP ranges, subnets, routing, and gateways exactly as needed — mimicking on-premises network design in the cloud.

2. **Logical Isolation**: Resources in your VPC are logically isolated from other AWS customers, even on shared physical hardware, via virtualization.

3. **Multi-Layer Security**: Combine Security Groups (instance-level, stateful) with NACLs (subnet-level, stateless) for defense-in-depth.

4. **Hybrid Connectivity Options**: Connect to on-premises via Site-to-Site VPN, AWS Direct Connect, or AWS Transit Gateway for seamless hybrid architectures.

5. **High Availability by Design**: Subnets span AZs, enabling multi-AZ deployments for fault tolerance without additional networking complexity.

6. **Private AWS Service Access**: VPC Endpoints allow private connectivity to S3, DynamoDB, and 100+ AWS services without internet exposure.

7. **No Bandwidth Bottleneck**: The VPC router is infinitely scalable — no single network device becomes a bottleneck.

8. **IPv4 and IPv6 Support**: Dual-stack networking with both IPv4 and IPv6 addresses on the same VPC.

9. **Flexible IP Management**: Bring Your Own IP (BYOIP) addresses to AWS and use them within your VPC.

10. **Cost-Effective**: VPC itself is free; you only pay for optional components like NAT Gateway, VPN connections, and data transfer.

11. **Compliance-Friendly**: Supports regulatory requirements (HIPAA, PCI-DSS, SOC) through network isolation and private connectivity.

---

## Limitations

### Hard Limits (Per Region, Per Account)

| Resource | Default Limit | Maximum |
|---|---|---|
| VPCs per region | 5 | 100 (via support) |
| Subnets per VPC | 200 | 200 |
| IPv4 CIDR blocks per VPC | 5 | 5 |
| Internet Gateways per region | 5 | 100 (via support) |
| Elastic IPs per region | 5 | via support |
| Security Groups per VPC | 2,500 | 10,000 |
| Rules per Security Group | 60 inbound + 60 outbound | 1,000 |
| Security Groups per ENI | 5 | 16 |
| Route Tables per VPC | 200 | 200 |
| Routes per Route Table | 50 | 1,000 |
| NACLs per VPC | 200 | 200 |
| Rules per NACL | 20 | 40 |
| VPC Peering Connections | 50 | 125 |
| NAT Gateways per A