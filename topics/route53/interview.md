# Route53 — Interview Questions

---

## Easy

### Q1. What is Amazon Route 53, and what are its primary functions?

**Answer:**
Amazon Route 53 is a highly available, scalable, and fully managed Domain Name System (DNS) web service provided by AWS. Its three primary functions are:

1. **Domain Registration** — You can register domain names (e.g., `example.com`) directly through Route 53.
2. **DNS Routing** — It translates human-readable domain names into IP addresses that computers use to connect to each other.
3. **Health Checking** — It monitors the health and availability of your application endpoints and can route traffic away from unhealthy resources automatically.

The name "Route 53" is a reference to TCP/UDP port 53, which is the standard port used by DNS.

---

### Q2. What is the difference between a Hosted Zone and a Domain in Route 53?

**Answer:**
- **Domain**: A human-readable name (e.g., `example.com`) that is registered with a domain registrar. Route 53 can act as the registrar.
- **Hosted Zone**: A container for DNS records that defines how traffic is routed for a specific domain and its subdomains. There are two types:
  - **Public Hosted Zone**: Manages DNS records for internet-facing traffic.
  - **Private Hosted Zone**: Manages DNS records for traffic within one or more Amazon VPCs.

A domain registration and a hosted zone are separate entities. You can register a domain with Route 53 (which automatically creates a hosted zone) or register elsewhere and point to Route 53 name servers manually.

---

### Q3. What are the most common DNS record types supported by Route 53?

**Answer:**

| Record Type | Purpose |
|-------------|---------|
| **A** | Maps a hostname to an IPv4 address |
| **AAAA** | Maps a hostname to an IPv6 address |
| **CNAME** | Maps a hostname to another hostname (cannot be used at zone apex) |
| **MX** | Mail exchange records for email routing |
| **NS** | Name server records — identifies DNS servers for the hosted zone |
| **SOA** | Start of Authority — stores administrative info about the zone |
| **TXT** | Text records used for domain verification, SPF, DKIM |
| **PTR** | Reverse DNS lookup (IP to hostname) |
| **SRV** | Service locator records |
| **CAA** | Certificate Authority Authorization |

Route 53 also supports **Alias records**, which are an AWS extension to DNS and map to AWS resources like ELB, CloudFront, and S3.

---

### Q4. What is an Alias record in Route 53, and how does it differ from a CNAME?

**Answer:**

| Feature | Alias Record | CNAME Record |
|---------|-------------|--------------|
| **Works at zone apex** | ✅ Yes (e.g., `example.com`) | ❌ No |
| **Targets** | AWS resources (ELB, CloudFront, S3, API GW, etc.) | Any hostname |
| **DNS query charges** | Free (no extra charge) | Standard charges apply |
| **TTL** | Set by Route 53 automatically | Configurable |
| **Health check integration** | Native AWS integration | Limited |
| **Returns** | A or AAAA records directly | Points to another CNAME |

**Key insight:** An Alias record is a Route 53-specific extension. When a DNS query is made, Route 53 resolves the Alias target and returns the actual IP address, making it behave like an A/AAAA record. This is why it can be used at the zone apex where CNAMEs are prohibited by RFC 1912.

---

### Q5. What are the routing policies available in Route 53?

**Answer:**
Route 53 supports the following routing policies:

1. **Simple** — Routes traffic to a single resource. Can return multiple values; client picks one randomly.
2. **Weighted** — Distributes traffic across multiple resources based on assigned weights (0–255).
3. **Latency-based** — Routes traffic to the region with the lowest network latency for the end user.
4. **Failover** — Routes traffic to a primary resource; switches to secondary if the primary is unhealthy.
5. **Geolocation** — Routes traffic based on the geographic location of the user (continent, country, or US state).
6. **Geoproximity** — Routes traffic based on geographic location of users and resources; uses a bias to expand/shrink routing coverage areas. Requires Route 53 Traffic Flow.
7. **Multi-value Answer** — Returns up to 8 healthy records at random; acts as a basic client-side load balancer.
8. **IP-based** — Routes traffic based on the originating IP address of the client (CIDR-based routing).

---

## Medium

### Q1. Explain Route 53 Health Checks. What types exist, and how do they integrate with DNS failover?

**Answer:**

Route 53 Health Checks continuously monitor the health of your endpoints and can trigger DNS failover when an endpoint becomes unhealthy.

**Types of Health Checks:**

1. **Endpoint Health Checks** — Monitors an endpoint by IP address or domain name using HTTP, HTTPS, or TCP. Route 53 sends requests from multiple locations globally (approximately 15 health checkers worldwide).

2. **Calculated Health Checks** — Combines the results of multiple child health checks using boolean logic (AND, OR, NOT). Useful for monitoring complex multi-component systems.

3. **CloudWatch Alarm Health Checks** — Monitors the state of a CloudWatch alarm. Useful for private resources inside a VPC that Route 53 health checkers cannot reach directly.

**Health Check Configuration Parameters:**
- **Protocol**: HTTP, HTTPS, TCP
- **Port**: The port to connect to
- **Path**: The URL path for HTTP/HTTPS checks
- **Request Interval**: 10 seconds (fast) or 30 seconds (standard)
- **Failure Threshold**: Number of consecutive failures before marking unhealthy (1–10)
- **String Matching**: Check response body for a specific string (first 5,120 bytes)

**Integration with DNS Failover:**
- When a health check fails, Route 53 automatically stops returning that record in DNS responses.
- In **Active-Passive Failover**, traffic shifts from the primary to the secondary record.
- In **Active-Active Failover**, Route 53 removes the unhealthy record from the pool.
- Health check status is visible in the Route 53 console and can trigger CloudWatch alarms and SNS notifications.

**Important:** Health checkers are located outside your VPC. For private resources, you must use CloudWatch metrics + alarms + Route 53 alarm-based health checks.

---

### Q2. What is the difference between Geolocation and Geoproximity routing policies? When would you use each?

**Answer:**

**Geolocation Routing:**
- Routes traffic based on the **geographic location of the DNS requester** (user).
- You define records for specific locations: continents, countries, or US states.
- A **default record** is recommended to handle traffic from locations not explicitly mapped.
- **Use case**: Legal/compliance requirements (e.g., EU users must be served from EU infrastructure), content localization, regional service restrictions.
- **Limitation**: No flexibility — if a user is in Germany, they always go to the EU endpoint regardless of load or capacity.

**Geoproximity Routing:**
- Routes traffic based on the **geographic location of both users AND resources**.
- Supports a **bias** value (−99 to +99) that artificially expands or shrinks the geographic area a resource serves.
  - Positive bias: Expands coverage (attracts more traffic)
  - Negative bias: Shrinks coverage (pushes traffic away)
- **Requires Route 53 Traffic Flow** (additional cost).
- **Use case**: Fine-grained traffic distribution across regions, gradually shifting traffic between regions, handling uneven geographic distributions.

**Key Difference Summary:**

| Feature | Geolocation | Geoproximity |
|---------|------------|-------------|
| Based on | User location only | User + resource location |
| Bias adjustment | ❌ No | ✅ Yes |
| Traffic Flow required | ❌ No | ✅ Yes |
| Default record | Recommended | Recommended |
| Flexibility | Low | High |

**When to use Geolocation**: Strict regulatory requirements, content localization.
**When to use Geoproximity**: Dynamic traffic shifting, gradual regional migrations, capacity management.

---

### Q3. How does Route 53 Traffic Flow work, and what problems does it solve?

**Answer:**

**Route 53 Traffic Flow** is a visual editor that allows you to create complex routing configurations using a policy tree with multiple routing rules combined together. It solves the problem of managing complex, multi-layered DNS routing logic that would be impossible or very difficult to manage with individual DNS records.

**Key Concepts:**
- **Traffic Policy**: A versioned routing policy document (JSON) created via the visual editor.
- **Policy Record**: Associates a traffic policy with a DNS name and hosted zone.
- **Traffic Policy Versions**: Immutable — you create new versions for changes, enabling rollback.

**Capabilities:**
- Combine multiple routing types in a single policy (e.g., Geolocation → Weighted → Failover).
- Reuse traffic policies across multiple DNS names (e.g., apply the same policy to `api.example.com` and `app.example.com`).
- Visual tree makes complex routing logic auditable and understandable.

**Example Architecture:**
```
User Request
    │
    ▼
[Geolocation Rule]
    ├── EU Users → [Weighted Rule: 70% EU-WEST-1, 30% EU-CENTRAL-1]
    │                   └── Each has Failover → [Health Check]
    └── US Users → [Latency Rule: US-EAST-1 vs US-WEST-2]
                        └── Each has Failover → [Health Check]
```

**Problems Solved:**
1. Eliminates the need to manually create dozens of interrelated DNS records.
2. Enables atomic policy updates (version-based).
3. Supports Geoproximity routing (only available through Traffic Flow).
4. Reduces human error in complex multi-region routing setups.

**Cost:** Traffic Flow has an additional charge per policy record per month (currently $50/month per policy record) plus standard DNS query charges.

---

### Q4. Explain Private Hosted Zones in Route 53. What are their limitations and common use cases?

**Answer:**

A **Private Hosted Zone** is a container for DNS records that Route 53 responds to only within one or more specified Amazon VPCs. Traffic is never exposed to the public internet.

**Configuration Requirements:**
- The VPC must have **DNS resolution** and **DNS hostnames** enabled (`enableDnsResolution` and `enableDnsHostnames` = true).
- The VPC must be associated with the private hosted zone.
- Multiple VPCs (even across different AWS accounts) can be associated with a single private hosted zone.

**Cross-Account Private Hosted Zone Association:**
- Not directly possible via the console — requires CLI/SDK.
- Steps: Authorize association from the hosted zone account, then create the association from the VPC account.

**Common Use Cases:**
1. **Internal service discovery** — `database.internal`, `cache.internal`
2. **Split-horizon DNS** — Same domain name resolves differently inside vs. outside the VPC (private zone overrides public zone)
3. **Microservices routing** — Internal API endpoints not exposed publicly
4. **Hybrid cloud** — On-premises resources resolving internal AWS DNS via Route 53 Resolver

**Limitations:**
1. Route 53 health checkers **cannot reach** private endpoints directly — must use CloudWatch alarm-based health checks.
2. Private hosted zones **do not support** DNSSEC.
3. DNS queries from on-premises networks require **Route 53 Resolver** (inbound endpoints) — they cannot query private hosted zones directly.
4. You cannot associate a private hosted zone with a VPC in a different region without explicit authorization.
5. The VPC's DHCP option set must use the default AWS DNS server (VPC+2 address) for Route 53 to work.

---

### Q5. What is Route 53 Resolver, and how does it enable DNS resolution in hybrid cloud environments?

**Answer:**

**Route 53 Resolver** (formerly Amazon DNS Resolver) is the DNS resolver built into every VPC that answers DNS queries for AWS resources and forwards queries to other resolvers. In hybrid environments, it bridges DNS resolution between on-premises networks and AWS VPCs.

**Components:**

**1. Inbound Endpoints:**
- Creates ENIs (Elastic Network Interfaces) in your VPC with IP addresses.
- On-premises DNS servers forward queries for AWS domains (e.g., `*.aws.internal`) to these IPs.
- Allows on-premises resources to resolve Route 53 private hosted zone records.

**2. Outbound Endpoints:**
- Creates ENIs in your VPC.
- Route 53 Resolver forwards DNS queries matching specified rules to on-premises DNS servers.
- Allows EC2 instances to resolve on-premises domain names (e.g., `*.corp.example.com`).

**3. Resolver Rules:**
- **Forward Rules**: Forward queries for specific domains to target IP addresses (on-premises DNS).
- **System Rules**: Built-in rules for AWS internal domains (cannot be modified).
- **Recursive Rules**: Default — queries not matching any rule are resolved by Route 53 Resolver itself.
- Rules can be shared across accounts using **AWS Resource Access Manager (RAM)**.

**Hybrid DNS Architecture:**
```
On-Premises Network                    AWS VPC
┌──────────────────┐                ┌─────────────────────────┐
│  Corporate DNS   │◄──Inbound──────│  Route 53 Resolver      │
│  Server          │                │  (Inbound Endpoint ENI) │
│                  │──Outbound─────►│  (Outbound Endpoint ENI)│
└──────────────────┘                │                         │
   Resolves:                        │  Private Hosted Zones   │
   *.aws.internal                   │  *.aws.internal          │
                                    └─────────────────────────┘
```

**Use Case Example:**
- EC2 instances query `db.corp.example.com` → Outbound endpoint forwards to on-premises DNS → Returns IP.
- On-premises servers query `rds.internal` → Forwarded to Inbound endpoint → Route 53 resolves from private hosted zone.

---

## Hard

### Q1. Deep dive into DNSSEC in Route 53. How does it work, and what are the operational considerations?

**Answer:**

**DNSSEC (Domain Name System Security Extensions)** adds a layer of security to DNS by enabling DNS responses to be cryptographically signed, protecting against DNS spoofing and cache poisoning attacks.

**How DNSSEC Works:**

1. **Key Signing Key (KSK)**: Signs the DNSKEY record set. Route 53 uses an asymmetric key stored in **AWS KMS** (customer-managed CMK in us-east-1).
2. **Zone Signing Key (ZSK)**: Signs all other record sets in the hosted zone. Managed entirely by Route 53 (you don't control this).
3. **DNSKEY Record**: Published in DNS, contains public keys used for verification.
4. **RRSIG Record**: Cryptographic signature for each record set, created with the ZSK.
5. **DS Record (Delegation Signer)**: Hash of the KSK's public key, stored in the parent zone (TLD). This creates the **chain of trust**.

**Chain of Trust:**
```
Root Zone (.) → signs → TLD (.com) → DS record → 
Your Zone (example.com) → DNSKEY → RRSIG on all records
```

**Route 53 DNSSEC Configuration Steps:**
1. Create a KSK in Route 53 (backed by a KMS CMK in us-east-1).
2. Enable DNSSEC signing on the hosted zone.
3. Route 53 begins signing all records with RRSIG records.
4. Add the DS record to the parent zone (TLD registrar) — this establishes the chain of trust.
5. Monitor DNSSEC status via CloudWatch metrics.

**Operational Considerations:**

**Key Rotation:**
- ZSK is automatically rotated by Route 53.
- KSK rotation is a manual process:
  1. Create a new KSK.
  2. Set new KSK status to ACTIVE.
  3. Update DS record at parent zone.
  4. Wait for TTL to expire (propagation).
  5. Delete old KSK.
  - **Risk**: If DS record isn't updated before old KSK is deleted, DNSSEC validation fails globally.