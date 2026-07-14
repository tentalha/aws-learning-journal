# Route53

## What is it?

**Amazon Route 53** is a highly available, scalable, and fully managed **Domain Name System (DNS) web service** provided by AWS. It falls under the **Networking & Content Delivery** category of AWS services.

The name "Route 53" is a reference to TCP/UDP port 53, which is the standard port used for DNS queries. Route 53 performs three primary functions:

1. **Domain Registration** — Register and manage domain names (e.g., `example.com`)
2. **DNS Resolution** — Translate human-readable domain names into IP addresses (e.g., `192.0.2.1`) that computers use to route traffic
3. **Health Checking** — Monitor the health and availability of resources and route traffic away from unhealthy endpoints

Route 53 is compliant with **IPv4** and **IPv6**, supports all common DNS record types, and is designed to provide **100% availability SLA** — the only AWS service with such a guarantee.

---

## Why do we need it?

### The Problem It Solves

Without a managed DNS service, organizations face several challenges:
- **Manual DNS management** — Maintaining DNS servers is operationally complex and expensive
- **Global traffic routing** — Directing users to the nearest or healthiest server is non-trivial
- **Failover automation** — Detecting failures and rerouting traffic requires custom infrastructure
- **Scalability** — DNS infrastructure must handle millions of queries per second without degradation
- **Domain management** — Registering and renewing domains requires coordination with registrars

### When to Use Route 53

| Scenario | Why Route 53 |
|---|---|
| Hosting a public website | Register domain + create DNS records pointing to servers |
| Multi-region active-active | Latency-based routing sends users to closest region |
| Disaster recovery | Failover routing with health checks |
| Blue/Green deployments | Weighted routing to shift traffic gradually |
| Internal microservices | Private hosted zones within a VPC |
| Hybrid cloud | Resolver endpoints for on-premises DNS resolution |

### Real Business Scenarios

- **E-commerce platform**: Route 53 routes shoppers to the nearest AWS region, reducing page load times and improving conversion rates
- **SaaS application**: Uses health checks + failover to automatically redirect traffic from a failing primary region to a standby region, achieving near-zero downtime
- **Enterprise hybrid cloud**: Resolver endpoints allow on-premises servers to resolve AWS private DNS names and vice versa
- **A/B testing**: Weighted routing sends 10% of traffic to a new application version for controlled rollout

---

## Internal Working

### DNS Resolution Flow

```
User Browser
    │
    ▼
Recursive Resolver (ISP or Google 8.8.8.8)
    │
    ▼
Root Name Servers (.)
    │
    ▼
TLD Name Servers (.com, .org, etc.)
    │
    ▼
Route 53 Authoritative Name Servers
    │
    ▼
Returns IP Address → Browser connects to origin
```

### How Route 53 Processes a Query

1. **Query Initiation**: A user types `www.example.com` in their browser. The OS checks its local DNS cache first.
2. **Recursive Resolver**: If not cached, the query goes to a recursive resolver (configured in the OS — often the ISP's resolver or a public one like `8.8.8.8`).
3. **Root Name Servers**: The recursive resolver queries one of 13 root name server clusters to find which TLD servers handle `.com`.
4. **TLD Name Servers**: The `.com` TLD servers return the **authoritative name servers** for `example.com` (these are Route 53's name servers, e.g., `ns-1234.awsdns-00.com`).
5. **Route 53 Authoritative Resolution**: The recursive resolver queries Route 53's name servers. Route 53 evaluates the **routing policy**, checks **health check status** (if configured), and returns the appropriate record.
6. **Response Caching**: The recursive resolver caches the response for the **TTL** duration and returns the IP to the browser.
7. **Connection**: The browser connects directly to the IP address.

### Anycast Routing

Route 53 uses **Anycast routing** — the same IP address is advertised from multiple geographic locations simultaneously. When a DNS query is made, it is automatically routed to the **nearest Route 53 edge location** (one of 100+ worldwide), minimizing latency for DNS resolution itself.

### Health Check Mechanism

Route 53 health checkers are distributed globally across **15+ geographic regions**. They continuously send requests to monitored endpoints:
- **HTTP/HTTPS checks**: Monitor specific paths (e.g., `/health`)
- **TCP checks**: Verify port connectivity
- **CloudWatch alarm checks**: Evaluate metric-based health
- A resource is marked **unhealthy** when a configurable threshold of checkers report failure (default: 3 of 5 checkers)
- Health check status propagates within **~30 seconds** (standard) or **~10 seconds** (fast checks)

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        Route 53                                  │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │  Domain           │  │  Hosted Zones    │  │  Health       │  │
│  │  Registration    │  │                  │  │  Checks       │  │
│  │                  │  │  ┌────────────┐  │  │               │  │
│  │  - Register TLD  │  │  │  Public HZ │  │  │  - HTTP/HTTPS │  │
│  │  - Transfer in   │  │  └────────────┘  │  │  - TCP        │  │
│  │  - Auto-renew    │  │  ┌────────────┐  │  │  - CW Alarm   │  │
│  └──────────────────┘  │  │  Private HZ│  │  │  - Calculated │  │
│                         │  └────────────┘  │  └───────────────┘  │
│                         └──────────────────┘                      │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   Routing Policies                        │   │
│  │  Simple │ Weighted │ Latency │ Failover │ Geo │ IP-based │   │
│  │  Multivalue Answer │ Geoproximity                         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Resolver (DNS Firewall)                      │   │
│  │  Inbound Endpoints │ Outbound Endpoints │ Rules           │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Hosted Zones

A **Hosted Zone** is a container for DNS records for a domain. There are two types:

| Type | Description | Use Case |
|---|---|---|
| **Public Hosted Zone** | Resolves DNS queries from the internet | Public websites, APIs |
| **Private Hosted Zone** | Resolves DNS queries within one or more VPCs | Internal services, microservices |

Each hosted zone is automatically assigned **4 name servers** (NS records) from different TLDs to provide redundancy:
```
ns-1234.awsdns-00.com
ns-567.awsdns-00.net
ns-890.awsdns-00.org
ns-123.awsdns-00.co.uk
```

### DNS Record Types Supported

| Record Type | Purpose | Example |
|---|---|---|
| **A** | Maps hostname to IPv4 | `example.com → 1.2.3.4` |
| **AAAA** | Maps hostname to IPv6 | `example.com → 2001:db8::1` |
| **CNAME** | Maps hostname to another hostname | `www → example.com` |
| **MX** | Mail exchange records | `mail.example.com` |
| **NS** | Name server records | Delegates subdomain |
| **PTR** | Reverse DNS lookup | `1.2.3.4 → example.com` |
| **SOA** | Start of Authority | Zone metadata |
| **SRV** | Service locator | `_sip._tcp.example.com` |
| **TXT** | Text records | SPF, DKIM, domain verification |
| **CAA** | Certificate Authority Authorization | Restrict SSL issuers |
| **ALIAS** | AWS-specific; maps to AWS resources | `example.com → ALB DNS name` |

### ALIAS Records (AWS-Specific)

ALIAS records are a Route 53 extension that behave like CNAME records but with key differences:

| Feature | CNAME | ALIAS |
|---|---|---|
| Root domain support | ❌ No (zone apex) | ✅ Yes |
| AWS resource targets | ❌ No | ✅ Yes |
| DNS query charges | Charged | Free |
| TTL control | Yes | Managed by Route 53 |

ALIAS targets include: **ELB, CloudFront, S3 static websites, API Gateway, VPC endpoints, Global Accelerator, Elastic Beanstalk**

### Routing Policies

#### 1. Simple Routing
- Single record, one or multiple IP values
- No health checks on the record itself
- Random selection when multiple values returned
- Use case: Single resource serving traffic

#### 2. Weighted Routing
- Multiple records with weight values (0–255)
- Traffic distributed proportionally: `weight / sum(all weights)`
- Supports health checks
- Use case: A/B testing, canary deployments, gradual migration

```
Record A: weight=70 → Server 1 (70% traffic)
Record B: weight=20 → Server 2 (20% traffic)
Record C: weight=10 → Server 3 (10% traffic)
```

#### 3. Latency-Based Routing
- Routes to the AWS region with lowest latency for the user
- Based on actual measured latency between users and AWS regions
- Use case: Multi-region active-active for performance

#### 4. Failover Routing
- **Primary** and **Secondary** records
- Routes to primary when healthy; failover to secondary when primary fails
- Requires health checks on primary
- Use case: Active-passive disaster recovery

#### 5. Geolocation Routing
- Routes based on **user's geographic location** (continent, country, or US state)
- Default record for unmatched locations
- Use case: Regulatory compliance, content localization, language-specific content

#### 6. Geoproximity Routing (Traffic Flow only)
- Routes based on geographic location of users AND resources
- **Bias** value expands (+1 to +99) or shrinks (-1 to -99) the routing area
- Use case: Fine-grained geographic traffic control

#### 7. IP-Based Routing
- Routes based on client's IP address (CIDR blocks)
- Use case: Route ISP customers to specific endpoints, optimize costs

#### 8. Multivalue Answer Routing
- Returns up to 8 healthy records randomly
- Supports health checks (unlike Simple with multiple values)
- Not a replacement for load balancers — client-side load balancing
- Use case: Improve availability with multiple healthy endpoints

### Route 53 Resolver

```
On-Premises Network ◄──── Inbound Endpoint ◄──── Route 53 Resolver
                                                         │
                                                    Outbound Endpoint
                                                         │
                                                    ▼ Forwarding Rules
                                               On-Premises DNS Server
```

- **Inbound Endpoints**: Allow on-premises DNS resolvers to forward queries to Route 53 (resolve AWS private hosted zones from on-prem)
- **Outbound Endpoints**: Allow Route 53 to forward queries to on-premises DNS servers
- **Resolver Rules**: Define which domains are forwarded where

---

## Real World Example

### Scenario: Multi-Region E-Commerce Platform with Failover

**Company**: RetailCo — operates in US-East-1 (primary) and EU-West-1 (DR)

**Requirements**:
- Users in North America → US-East-1
- Users in Europe → EU-West-1
- Automatic failover if a region becomes unhealthy
- Zero-downtime deployments via weighted routing

**Step-by-Step Implementation**:

**Step 1: Register Domain**
```
Domain: retailco.com registered via Route 53 Domain Registration
Auto-renew: Enabled
Privacy protection: Enabled
```

**Step 2: Create Public Hosted Zone**
```
Hosted Zone: retailco.com
Type: Public
```

**Step 3: Create Health Checks**
```
Health Check 1: US-East-1
  - Endpoint: alb-us-east.retailco.com
  - Protocol: HTTPS
  - Port: 443
  - Path: /health
  - Interval: 30 seconds
  - Failure threshold: 3

Health Check 2: EU-West-1
  - Endpoint: alb-eu-west.retailco.com
  - Protocol: HTTPS
  - Port: 443
  - Path: /health
  - Interval: 30 seconds
  - Failure threshold: 3
```

**Step 4: Create Latency + Failover Records**

*Primary records with latency routing:*
```
Record: www.retailco.com
Type: A (ALIAS → ALB in us-east-1)
Routing Policy: Latency
Region: us-east-1
Health Check: HC-US-East-1
Set ID: primary-us-east

Record: www.retailco.com
Type: A (ALIAS → ALB in eu-west-1)
Routing Policy: Latency
Region: eu-west-1
Health Check: HC-EU-West-1
Set ID: primary-eu-west
```

**Step 5: Configure CloudWatch Alarms for Health Checks**
```
Alarm: Route53-HealthCheck-US-East-1
Metric: HealthCheckStatus
Threshold: < 1 (unhealthy)
Action: SNS notification → PagerDuty
```

**Step 6: Traffic Flow During Normal Operation**
```
US User → Route 53 → Evaluates latency → us-east-1 (lowest latency) → ALB → ECS Services
EU User → Route 53 → Evaluates latency → eu-west-1 (lowest latency) → ALB → ECS Services
```

**Step 7: Failover Scenario**
```
us-east-1 ALB becomes unhealthy
→ Health checkers detect failure within ~30s
→ Route 53 marks us-east-1 record unhealthy
→ US users automatically routed to eu-west-1
→ CloudWatch alarm fires → SNS → PagerDuty alert
→ Engineering team investigates and restores us-east-1
→ Health checks pass → Route 53 resumes routing to us-east-1
```

**Step 8: Blue/Green Deployment**
```
Current: www.retailco.com → v1 ALB (weight=100)

Deployment starts:
  Record 1: www.retailco.com → v1 ALB (weight=90)
  Record 2: www.retailco.com → v2 ALB (weight=10)

Monitor v2 metrics → gradually shift:
  Record 1: weight=0
  Record 2: weight=100

Deprecate v1 after validation
```

---

## Advantages

1. **100% Availability SLA** — The only AWS service with a 100% uptime SLA; backed by Anycast routing across 100+ edge locations globally.

2. **Tight AWS Integration** — Native ALIAS records for ELB, CloudFront, S3, API Gateway, and other AWS services with no additional DNS query charges.

3. **Rich Routing Policies** — Eight distinct routing policies (Simple, Weighted, Latency, Failover, Geolocation, Geoproximity, IP-based, Multivalue) enable sophisticated traffic management.

4. **Built-in Health Checks** —