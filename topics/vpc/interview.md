# VPC — Interview Questions

---

## Easy

### 1. What is an Amazon VPC, and why is it used?

**Answer:**
Amazon Virtual Private Cloud (VPC) is a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define. It gives you complete control over your networking environment, including:
- Selection of your own IP address range (CIDR block)
- Creation of subnets (public and private)
- Configuration of route tables and network gateways
- Control of inbound and outbound traffic using Security Groups and Network ACLs

VPCs are used to isolate workloads, enforce security boundaries, and control how resources communicate with each other and the internet.

---

### 2. What is the difference between a Public Subnet and a Private Subnet?

**Answer:**

| Feature | Public Subnet | Private Subnet |
|---|---|---|
| Internet Gateway | Associated via route table | No route to Internet Gateway |
| Internet Access | Direct inbound/outbound | Via NAT Gateway/Instance (outbound only) |
| Use Cases | Load balancers, bastion hosts, web servers | Databases, application servers, internal services |
| Public IP Assignment | Typically auto-assigned | Not auto-assigned |

A subnet becomes **public** when its route table has a route directing `0.0.0.0/0` traffic to an **Internet Gateway (IGW)**. A subnet is **private** when it has no such route.

---

### 3. What is a Security Group, and how does it differ from a Network ACL (NACL)?

**Answer:**

| Feature | Security Group | Network ACL |
|---|---|---|
| Level | Instance/ENI level | Subnet level |
| State | **Stateful** (return traffic automatically allowed) | **Stateless** (must explicitly allow both directions) |
| Rules | Allow rules only | Allow and Deny rules |
| Rule Evaluation | All rules evaluated together | Rules evaluated in order (lowest number first) |
| Default Behavior | Deny all inbound, allow all outbound | Allow all inbound and outbound |

**Key takeaway:** Security Groups are the first line of defense at the instance level; NACLs provide an additional layer at the subnet boundary.

---

### 4. What is an Internet Gateway (IGW)?

**Answer:**
An Internet Gateway is a horizontally scaled, redundant, and highly available VPC component that enables communication between instances in your VPC and the internet. It serves two purposes:
1. **Provides a target** in route tables for internet-routable traffic
2. **Performs NAT** for instances with public IPv4 addresses

An IGW is attached to a VPC (one IGW per VPC at a time). For an instance to be reachable from the internet, it must:
- Be in a subnet with a route to the IGW
- Have a public IP or Elastic IP address
- Have a Security Group that allows the relevant inbound traffic

---

### 5. What is a NAT Gateway, and when would you use it?

**Answer:**
A NAT (Network Address Translation) Gateway allows instances in a **private subnet** to initiate outbound traffic to the internet (e.g., for software updates, API calls) while preventing the internet from initiating inbound connections to those instances.

**Key characteristics:**
- Managed by AWS (no patching or administration required)
- Deployed in a **public subnet** with an Elastic IP address
- Route table in the private subnet must point `0.0.0.0/0` to the NAT Gateway
- **Not free** — billed per hour and per GB of data processed
- For high availability, deploy one NAT Gateway per Availability Zone

**Use cases:** EC2 instances in private subnets that need to pull patches, call external APIs, or access AWS services without exposing themselves to inbound internet traffic.

---

## Medium

### 1. Explain VPC Peering. What are its limitations?

**Answer:**
VPC Peering is a networking connection between two VPCs that enables routing traffic between them using private IPv4 or IPv6 addresses. Instances in either VPC can communicate as if they are within the same network.

**How it works:**
- One VPC sends a peering request; the other accepts it
- Route tables in **both** VPCs must be updated to direct traffic for the peer's CIDR to the peering connection
- Security Groups must allow the relevant traffic
- Works across accounts and across AWS regions (inter-region VPC peering)

**Limitations:**
1. **No transitive peering** — If VPC A peers with VPC B, and VPC B peers with VPC C, VPC A **cannot** communicate with VPC C through VPC B. Each pair needs its own peering connection.
2. **Overlapping CIDR blocks** — VPCs with overlapping IP ranges cannot be peered.
3. **No edge-to-edge routing** — You cannot use a peering connection to route traffic through a VPN, Direct Connect, or Internet Gateway of the peer VPC.
4. **Scalability** — Managing many peering connections becomes complex at scale (consider AWS Transit Gateway instead).
5. **DNS resolution** — Must be explicitly enabled for private DNS hostnames to resolve across peered VPCs.

---

### 2. What is AWS Transit Gateway, and how does it differ from VPC Peering?

**Answer:**
AWS Transit Gateway (TGW) is a network transit hub that connects VPCs, AWS accounts, VPNs, and Direct Connect gateways through a central, managed service.

**Comparison:**

| Feature | VPC Peering | Transit Gateway |
|---|---|---|
| Topology | Point-to-point (mesh) | Hub-and-spoke |
| Transitive Routing | Not supported | Supported |
| Scale | Difficult at scale (N*(N-1)/2 connections) | Highly scalable (thousands of VPCs) |
| Cross-account | Yes | Yes |
| Cross-region | Yes (inter-region peering) | Yes (inter-region peering) |
| Bandwidth | No limit per connection | Up to 50 Gbps per VPC attachment |
| Cost | No charge for peering itself (data transfer costs apply) | Per attachment + data processing fee |
| Route Management | Manual per VPC | Centralized route tables in TGW |

**When to use TGW:** When you have many VPCs (3+), need transitive routing, want centralized network management, or need to connect on-premises networks to multiple VPCs simultaneously.

**When to use VPC Peering:** Simple, low-cost connectivity between a small number of VPCs with no need for transitive routing.

---

### 3. How do VPC Endpoints work, and what are the different types?

**Answer:**
VPC Endpoints allow you to privately connect your VPC to supported AWS services and VPC endpoint services without requiring an Internet Gateway, NAT Gateway, VPN, or Direct Connect connection. Traffic remains within the AWS network.

**Types:**

**1. Gateway Endpoints**
- Supported services: **Amazon S3** and **Amazon DynamoDB** only
- Free of charge
- Works by adding an entry to the route table pointing the service's prefix list to the endpoint
- Does not use an ENI; no DNS changes required
- Not accessible from outside the VPC (no cross-region)

**2. Interface Endpoints (powered by AWS PrivateLink)**
- Supported services: Most AWS services (EC2, SNS, SQS, Secrets Manager, etc.) and third-party services
- Creates an **Elastic Network Interface (ENI)** with a private IP in your subnet
- Billed per hour per AZ and per GB of data processed
- Supports private DNS — the service's public DNS resolves to the private IP within the VPC
- Can be accessed from on-premises via VPN/Direct Connect

**3. Gateway Load Balancer Endpoints**
- Used with Gateway Load Balancers to route traffic to third-party virtual appliances (firewalls, IDS/IPS)
- Enables transparent inspection of traffic

**Best practice:** Use Gateway Endpoints for S3/DynamoDB (free), Interface Endpoints for other services when private connectivity is required.

---

### 4. Explain the concept of CIDR blocks in VPC. What are the rules and best practices?

**Answer:**
CIDR (Classless Inter-Domain Routing) notation defines the IP address range for a VPC or subnet using a base IP and a prefix length (e.g., `10.0.0.0/16`).

**VPC CIDR Rules:**
- Allowed block sizes: `/16` (65,536 IPs) to `/28` (16 IPs)
- Supported private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
- You can add up to **5 secondary CIDR blocks** to an existing VPC
- CIDR blocks cannot be modified after creation (only additional ones added)
- Peered VPCs cannot have overlapping CIDR blocks

**Subnet CIDR Rules:**
- Must be a subset of the VPC CIDR
- AWS reserves **5 IP addresses** per subnet:
  - `.0` — Network address
  - `.1` — VPC router
  - `.2` — DNS server
  - `.3` — Reserved for future use
  - `.255` — Broadcast address (not supported, but reserved)
- Minimum subnet size: `/28` (16 IPs, 11 usable)

**Best Practices:**
- Plan for growth — use larger CIDR blocks (`/16`) at the VPC level
- Avoid using `192.168.x.x` ranges if connecting to on-premises networks that commonly use this range
- Use non-overlapping CIDRs across all VPCs and on-premises networks from the start
- Allocate subnets by AZ and tier (e.g., `/24` per AZ per tier)
- Document your IP address management (IPAM) plan; use AWS VPC IP Address Manager (IPAM) for large environments

---

### 5. What is a Bastion Host, and how is it securely configured in a VPC?

**Answer:**
A Bastion Host (also called a Jump Server) is a specially hardened EC2 instance placed in a **public subnet** that acts as the single entry point for SSH/RDP access to instances in private subnets.

**Architecture:**
```
Internet → IGW → Public Subnet (Bastion Host) → Private Subnet (App/DB Servers)
```

**Secure Configuration Steps:**

1. **Placement:** Deploy in a public subnet with a dedicated Security Group
2. **Security Group for Bastion:**
   - Inbound: Allow SSH (port 22) or RDP (port 3389) **only from known IP ranges** (corporate IP, VPN egress IP), not `0.0.0.0/0`
   - Outbound: Allow SSH/RDP to private subnet CIDR
3. **Security Group for Private Instances:**
   - Allow inbound SSH/RDP **only from the Bastion's Security Group ID** (not its IP)
4. **Key Management:** Use SSH agent forwarding rather than storing private keys on the bastion
5. **Hardening:** Disable root login, enable MFA, keep OS patched, enable CloudTrail and VPC Flow Logs
6. **High Availability:** Deploy bastion hosts in multiple AZs behind an NLB or Auto Scaling Group

**Modern Alternative:** AWS Systems Manager Session Manager eliminates the need for a bastion host entirely by providing secure, audited shell access without opening inbound ports.

---

## Hard

### 1. Deep dive into VPC Flow Logs — what do they capture, what do they miss, and how would you use them for security analysis?

**Answer:**
VPC Flow Logs capture information about IP traffic going to and from network interfaces in your VPC. They can be enabled at the VPC, subnet, or ENI level and published to CloudWatch Logs, S3, or Kinesis Data Firehose.

**Flow Log Record Fields (default):**
```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes windowstart windowend action log-status
```

**Custom fields available (v3+):**
`vpc-id`, `subnet-id`, `instance-id`, `tcp-flags`, `pkt-srcaddr`, `pkt-dstaddr`, `flow-direction`, `traffic-path`, `pkt-src-aws-service`, `pkt-dst-aws-service`

**What Flow Logs DO capture:**
- Accepted and rejected traffic
- Traffic to/from ENIs (including ELB, RDS, ElastiCache, etc.)
- Traffic to VPC endpoints
- Traffic to/from NAT Gateways
- DNS queries to Route 53 Resolver (with custom fields)

**What Flow Logs DO NOT capture:**
- Traffic to the Amazon DNS server (169.254.169.253) — unless using custom DNS
- DHCP traffic
- Traffic to/from the link-local address (169.254.169.254 — instance metadata)
- Traffic to/from the VPC router reserved address (x.x.x.1)
- Traffic mirrored by VPC Traffic Mirroring (captured separately)
- Packet payload content (only headers/metadata)

**Security Analysis Use Cases:**

1. **Port scanning detection:**
   ```sql
   -- Athena query to find rejected traffic patterns
   SELECT srcaddr, dstport, COUNT(*) as attempts
   FROM vpc_flow_logs
   WHERE action = 'REJECT' AND dstport < 1024
   GROUP BY srcaddr, dstport
   HAVING COUNT(*) > 100
   ORDER BY attempts DESC;
   ```

2. **Data exfiltration detection:** Query for unusually large outbound byte counts to unexpected destinations

3. **Lateral movement detection:** Look for unexpected internal traffic between subnets (e.g., private subnet to private subnet on unusual ports)

4. **Compliance:** Demonstrate that traffic to/from sensitive resources is controlled

**Performance Considerations:**
- Flow logs have an aggregation interval (1 or 10 minutes) — not real-time
- High-traffic environments generate enormous log volumes; use S3 + Athena for cost-effective analysis
- Enable `tcp-flags` field to distinguish SYN scans from established connections
- Use CloudWatch Metric Filters + Alarms for near-real-time alerting on specific patterns

---

### 2. Explain how AWS PrivateLink works internally and when you would architect a solution using it.

**Answer:**
AWS PrivateLink provides private connectivity between VPCs, AWS services, and on-premises applications without exposing traffic to the public internet. It uses **Interface VPC Endpoints** backed by **Network Load Balancers** on the service provider side.

**Internal Architecture:**

```
Consumer VPC                          Provider VPC
┌─────────────────────┐               ┌──────────────────────┐
│  EC2 Instance       │               │  NLB                 │
│       │             │               │   │                  │
│  Interface Endpoint │◄─PrivateLink─►│  Target Group        │
│  (ENI with priv IP) │               │   │                  │
│       │             │               │  EC2/ECS/Lambda      │
│  Private DNS        │               │                      │
└─────────────────────┘               └──────────────────────┘
```

**How it works:**
1. **Service Provider** creates a VPC Endpoint Service backed by an NLB
2. **Service Consumer** creates an Interface Endpoint in their VPC, specifying the endpoint service name
3. AWS creates an **ENI** in the consumer's specified subnets with a private IP from the subnet CIDR
4. The provider can require **acceptance** before connections are established (whitelisting)
5. **Private DNS:** The consumer can enable private DNS so the service's public hostname resolves to the endpoint's private IP within the VPC

**Key Properties:**
- Traffic never leaves the AWS network
- No VPC peering required — no CIDR overlap concerns
- One-directional: only the consumer initiates connections
- Works across accounts and regions (with inter-region support)
- Supports IPv4 and IPv6

**When to architect with PrivateLink:**

1. **SaaS on AWS:** A SaaS provider wants to offer their service to customers without giving customers access to the provider's VPC (e.g., security scanning tools, data platforms)

2. **Shared Services VPC:** Central services (logging, monitoring, authentication) in a shared VPC exposed to spoke VPCs without full VPC peering

3. **Cross-account service sharing:** Team A's microservice needs to be consumed by Team B's VPC without complex network topology

4. **On-premises to AWS services:** Combined with Direct Connect, allows on-premises systems to reach AWS services privately

5. **Compliance requirements:** Environments where data must never traverse the public internet

**vs. VPC Peering:**
- PrivateLink: One-directional, no CIDR overlap issues, service-level granularity
- VPC Peering: Bidirectional, full network access