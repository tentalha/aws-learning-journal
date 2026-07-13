# Load Balancer

## What is it?

**AWS Elastic Load Balancing (ELB)** is a fully managed load balancing service that automatically distributes incoming application traffic across multiple targets — such as Amazon EC2 instances, containers, IP addresses, and Lambda functions — in one or more Availability Zones (AZs).

ELB falls under the **Networking & Content Delivery** category in AWS and is a foundational building block for highly available, fault-tolerant, and scalable architectures.

AWS offers **four types** of load balancers:

| Type | Layer | Best For |
|------|-------|----------|
| **Application Load Balancer (ALB)** | Layer 7 (HTTP/HTTPS/gRPC) | Web apps, microservices, container-based apps |
| **Network Load Balancer (NLB)** | Layer 4 (TCP/UDP/TLS) | Ultra-high performance, static IP, gaming, IoT |
| **Gateway Load Balancer (GWLB)** | Layer 3 (IP packets) | Third-party virtual network appliances (firewalls, IDS/IPS) |
| **Classic Load Balancer (CLB)** | Layer 4 & 7 | Legacy workloads (deprecated, not recommended) |

---

## Why do we need it?

### The Problem

Without a load balancer, all traffic hits a **single server**. This creates:

- **Single Point of Failure (SPOF)**: If the server crashes, the application goes down.
- **Scalability Ceiling**: One server can only handle limited concurrent users.
- **Uneven Load Distribution**: Some servers may be overloaded while others sit idle.
- **No Health Awareness**: Traffic continues to route to unhealthy instances.
- **No SSL Termination**: Each server must manage its own TLS certificates.

### When to Use It

- **E-commerce platforms** experiencing traffic spikes during sales events (Black Friday, Prime Day)
- **Microservices architectures** where different URL paths route to different services
- **Multi-AZ deployments** requiring fault tolerance and zero-downtime deployments
- **Blue/Green or Canary deployments** where traffic must be gradually shifted
- **Regulatory environments** requiring TLS termination and centralized certificate management
- **Gaming backends** requiring ultra-low latency TCP connections (NLB)
- **Security appliance insertion** using GWLB to inspect all traffic through firewalls

### Real Business Scenarios

1. **Netflix-style streaming platform**: ALB routes `/api/*` to backend microservices and `/stream/*` to media servers using path-based routing.
2. **Financial trading system**: NLB handles millions of TCP connections per second with microsecond latency.
3. **Enterprise security**: GWLB routes all traffic through Palo Alto firewalls before reaching application servers.

---

## Internal Working

### How ELB Works Under the Hood

```
Client Request
     │
     ▼
┌─────────────────────────────────────────┐
│           DNS Resolution                │
│  (ELB DNS name → Load Balancer Node IPs)│
└─────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────┐
│         Load Balancer Node              │
│  (One per AZ, managed by AWS)           │
│  ┌─────────────────────────────────┐   │
│  │  Listener (Port 443, HTTPS)     │   │
│  │  ↓                              │   │
│  │  Rules Evaluation               │   │
│  │  ↓                              │   │
│  │  Target Group Selection         │   │
│  │  ↓                              │   │
│  │  Health Check Verification      │   │
│  │  ↓                              │   │
│  │  Algorithm (Round Robin, etc.)  │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────┐
│         Target (EC2, ECS, Lambda)       │
└─────────────────────────────────────────┘
```

### Step-by-Step Request Flow (ALB)

1. **DNS Resolution**: Client resolves the ALB's DNS name (e.g., `my-alb-123456.us-east-1.elb.amazonaws.com`) via Route 53. DNS returns multiple IP addresses (one per AZ node).

2. **Connection Establishment**: Client connects to the nearest/resolved ALB node IP. For HTTPS, TLS handshake occurs at the ALB (SSL termination).

3. **Listener Evaluation**: The ALB listener checks incoming port/protocol and evaluates **rules** in priority order (1–50,000).

4. **Rule Matching**: Rules can match on:
   - Host headers (`api.example.com`)
   - URL paths (`/api/v1/*`)
   - HTTP methods (`GET`, `POST`)
   - Query strings (`?version=2`)
   - HTTP headers (custom headers)
   - Source IP (CIDR-based)

5. **Target Group Selection**: The matched rule forwards the request to a **Target Group** (a logical grouping of targets).

6. **Health Check Filtering**: Only healthy targets receive traffic. Health checks run independently (HTTP GET to `/health`, checking for 200 OK).

7. **Load Balancing Algorithm**:
   - ALB: **Round Robin** (default) or **Least Outstanding Requests (LOR)**
   - NLB: **Flow Hash** (5-tuple: src IP, src port, dst IP, dst port, protocol)
   - GWLB: **5-tuple or 3-tuple** flow hash

8. **Request Forwarding**: The ALB node forwards the request to the selected target. For ALB, it adds `X-Forwarded-For`, `X-Forwarded-Proto`, and `X-Forwarded-Port` headers.

9. **Response Return**: Target sends response back to the ALB node, which returns it to the client.

### Health Check Mechanism

```
ALB Node → HTTP GET /health → Target
                              ↓
                     Response 200 OK → Healthy ✅
                     Response 5xx    → Unhealthy ❌
                     Timeout         → Unhealthy ❌

Thresholds:
- Healthy threshold:   2 consecutive successes → mark healthy
- Unhealthy threshold: 3 consecutive failures  → mark unhealthy
- Interval:            30 seconds (default)
- Timeout:             5 seconds (default)
```

### Cross-Zone Load Balancing

- **Enabled**: Each LB node distributes traffic evenly across **all registered targets in all AZs**.
- **Disabled**: Each LB node distributes traffic only to targets **in its own AZ**.

```
Cross-Zone ENABLED:
AZ-A Node (50% traffic) → distributes to ALL 6 targets (each ~16.7%)
AZ-B Node (50% traffic) → distributes to ALL 6 targets (each ~16.7%)

Cross-Zone DISABLED:
AZ-A Node (50% traffic) → distributes to 2 AZ-A targets (each 25%)
AZ-B Node (50% traffic) → distributes to 4 AZ-B targets (each 12.5%)
```

> **Note**: Cross-zone load balancing is **enabled by default and free** for ALB. For NLB and GWLB, it's **disabled by default** and incurs inter-AZ data transfer charges when enabled.

---

## Architecture

### Core Components

```
┌──────────────────────────────────────────────────────────────────┐
│                        VPC (us-east-1)                           │
│                                                                  │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │  Public Subnet  │    │  Public Subnet  │                     │
│  │   (AZ-A)        │    │   (AZ-B)        │                     │
│  │  ┌───────────┐  │    │  ┌───────────┐  │                     │
│  │  │ ALB Node  │  │    │  │ ALB Node  │  │                     │
│  │  └─────┬─────┘  │    │  └─────┬─────┘  │                     │
│  └────────┼────────┘    └────────┼────────┘                     │
│           │                      │                               │
│  ┌────────▼────────┐    ┌────────▼────────┐                     │
│  │ Private Subnet  │    │ Private Subnet  │                     │
│  │   (AZ-A)        │    │   (AZ-B)        │                     │
│  │  ┌──────────┐   │    │  ┌──────────┐   │                     │
│  │  │  EC2 #1  │   │    │  │  EC2 #3  │   │                     │
│  │  ├──────────┤   │    │  ├──────────┤   │                     │
│  │  │  EC2 #2  │   │    │  │  EC2 #4  │   │                     │
│  │  └──────────┘   │    │  └──────────┘   │                     │
│  └─────────────────┘    └─────────────────┘                     │
│                                                                  │
│  Components:                                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Listener: HTTPS:443 → Rules → Target Groups             │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │ Rule 1: /api/* → Target Group: API (EC2 #1, #2)     │ │   │
│  │  │ Rule 2: /img/* → Target Group: Images (EC2 #3, #4)  │ │   │
│  │  │ Default: /*   → Target Group: Web (All EC2s)        │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

#### 1. Listeners
- Define the **port and protocol** the LB listens on (e.g., HTTPS:443)
- Each LB can have **multiple listeners** (e.g., HTTP:80 and HTTPS:443)
- HTTP:80 listener can redirect to HTTPS:443 (301 redirect)

#### 2. Rules
- Ordered by **priority** (1 = highest, default = lowest)
- Each rule has **conditions** (what to match) and **actions** (what to do)
- Actions: `forward`, `redirect`, `fixed-response`, `authenticate-oidc`, `authenticate-cognito`

#### 3. Target Groups
- Logical grouping of targets
- Targets can be: EC2 instances, IP addresses, Lambda functions, ALBs (for NLB)
- Each target group has its own **health check configuration**
- Supports **weighted target groups** for traffic splitting

#### 4. Security Groups (ALB/CLB only)
- Control inbound/outbound traffic to the load balancer
- NLB does **not** have security groups (uses NACLs and target security groups)

#### 5. Subnets
- Must specify **at least 2 subnets in different AZs**
- ALB/NLB nodes are deployed in these subnets
- For internet-facing LBs, subnets must have an Internet Gateway route

### ALB vs NLB vs GWLB Decision Tree

```
Need to inspect HTTP headers/paths?
├── YES → Use ALB
│         Need WebSocket/HTTP2/gRPC? → ALB supports all
└── NO
    ├── Need ultra-low latency or static IPs?
    │   └── YES → Use NLB
    ├── Need to insert security appliances?
    │   └── YES → Use GWLB
    └── Legacy EC2-Classic workload?
        └── YES → Use CLB (not recommended)
```

---

## Real World Example

### Scenario: Multi-Tier E-Commerce Platform

**Company**: RetailCo — an online retailer expecting 100,000 concurrent users during a flash sale.

**Architecture Goal**: Zero downtime, auto-scaling, microservices routing, and security compliance.

#### Step-by-Step Walkthrough

**Step 1: DNS & Entry Point**
```
Customer Browser
      │
      ▼
Route 53 (retail.com) → A Record (Alias) → ALB DNS
```

**Step 2: ALB Listener Configuration**
```yaml
Listener: HTTP:80
  Action: Redirect to HTTPS:443 (301)

Listener: HTTPS:443
  SSL Certificate: ACM (*.retail.com)
  Security Policy: ELBSecurityPolicy-TLS13-1-2-2021-06
  Rules:
    Priority 1:  Host=api.retail.com        → Forward → API Target Group
    Priority 2:  Host=retail.com, Path=/checkout/* → Forward → Checkout TG (with Cognito Auth)
    Priority 3:  Host=retail.com, Path=/products/* → Forward → Product TG
    Priority 4:  Host=retail.com, Path=/images/*   → Forward → CDN redirect (302)
    Default:     Forward → Frontend Target Group
```

**Step 3: Target Group Setup**

```
Target Group: "api-tg"
  Protocol: HTTP:8080
  Target Type: IP (ECS Fargate tasks)
  Health Check: GET /api/health → 200 OK
  Deregistration Delay: 30s (fast drain for rolling deploys)
  Algorithm: Least Outstanding Requests

Target Group: "checkout-tg"
  Protocol: HTTPS:443
  Target Type: instance (EC2 with PCI-DSS compliance)
  Health Check: GET /checkout/health → 200 OK
  Stickiness: Duration-based (1 hour) — maintains cart session

Target Group: "product-tg"
  Protocol: HTTP:3000
  Target Type: IP (ECS Fargate)
  Health Check: GET /products/health → 200 OK
```

**Step 4: Auto Scaling Integration**
```
Flash Sale Starts at 12:00 PM
      │
      ▼
ALB receives 10,000 req/sec → CloudWatch alarm triggers
      │
      ▼
Auto Scaling Group scales EC2 from 4 → 20 instances
      │
      ▼
New instances register with Target Group automatically
      │
      ▼
ALB health checks pass → instances receive traffic within 60s
```

**Step 5: Blue/Green Deployment**
```
Current (Blue): Product TG → v1 ECS Service (10 tasks)
New     (Green): Product TG v2 → v2 ECS Service (10 tasks)

Using Weighted Target Groups:
  Step 1: 90% → Blue TG, 10% → Green TG (canary)
  Step 2: 50% → Blue TG, 50% → Green TG
  Step 3: 0%  → Blue TG, 100% → Green TG (complete)
```

**Step 6: Security Layer**
```
Internet → WAF (AWS WAF on ALB) → ALB → Private Subnet Targets
                │
                ├── Block SQL injection
                ├── Rate limit: 2000 req/5min per IP
                ├── Block known bad IPs (IP reputation list)
                └── Allow only US/EU traffic (Geo-restriction)
```

---

## Advantages

### Operational Excellence
- ✅ **Fully managed**: AWS handles provisioning, scaling, patching, and HA of the load balancer infrastructure
- ✅ **Multi-AZ by default**: Automatically distributes traffic across multiple AZs for fault tolerance
- ✅ **Automated health checks**: Continuously monitors target health and routes only to healthy instances

###