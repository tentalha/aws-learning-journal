# Load Balancer — Interview Questions

---

## Easy

### Q1. What is a Load Balancer and why is it used?

**Answer:**
A Load Balancer is a managed service that automatically distributes incoming application traffic across multiple targets (such as EC2 instances, containers, IP addresses, or Lambda functions) in one or more Availability Zones. It is used to:
- Improve application availability and fault tolerance
- Prevent any single server from becoming a bottleneck
- Enable horizontal scaling of applications
- Perform health checks and route traffic only to healthy targets
- Terminate SSL/TLS connections centrally

---

### Q2. What are the three types of Load Balancers available in AWS?

**Answer:**
AWS offers four types of Elastic Load Balancers (ELB):

| Type | Best For | Layer |
|------|----------|-------|
| **Application Load Balancer (ALB)** | HTTP/HTTPS traffic, microservices, content-based routing | Layer 7 |
| **Network Load Balancer (NLB)** | TCP/UDP/TLS traffic, ultra-low latency, static IPs | Layer 4 |
| **Gateway Load Balancer (GWLB)** | Third-party virtual appliances (firewalls, IDS/IPS) | Layer 3/4 |
| **Classic Load Balancer (CLB)** | Legacy EC2-Classic environments (deprecated) | Layer 4/7 |

---

### Q3. What is a Target Group in the context of AWS Load Balancers?

**Answer:**
A Target Group is a logical grouping of targets (EC2 instances, IP addresses, Lambda functions, or other ALBs) that a Load Balancer routes requests to. Key characteristics:
- Each target group has its own **health check configuration**
- Targets can be registered and deregistered without affecting the load balancer
- A single target can belong to multiple target groups
- ALB and NLB use target groups; CLB routes directly to instances
- You define the **protocol and port** at the target group level

---

### Q4. What is a health check in AWS Load Balancers?

**Answer:**
A health check is a periodic request sent by the Load Balancer to registered targets to determine if they are capable of receiving traffic. Key points:
- The load balancer sends requests to the configured **health check path** (e.g., `/health`)
- Targets that fail health checks are marked **unhealthy** and removed from rotation
- Targets that pass are marked **healthy** and receive traffic
- Configurable parameters include: **protocol, path, port, interval, timeout, healthy threshold, unhealthy threshold**
- Default HTTP health check expects a `2xx` or `3xx` response

---

### Q5. What is the difference between an Internet-facing and Internal Load Balancer?

**Answer:**
- **Internet-facing Load Balancer:** Has a **public DNS name** and public IP addresses. Routes traffic from the internet to targets in public or private subnets. Used for public-facing applications.
- **Internal Load Balancer:** Has only a **private DNS name** and private IP addresses. Routes traffic only from within the VPC or connected networks (VPN, Direct Connect). Used for internal microservices communication.

> **Key Note:** In both cases, the **targets (EC2 instances)** can reside in private subnets. Only the load balancer itself needs to be in a public subnet for internet-facing deployments.

---

## Medium

### Q1. How does an Application Load Balancer (ALB) support content-based routing? Provide examples.

**Answer:**
ALB operates at **Layer 7 (HTTP/HTTPS)** and can inspect the content of requests to make intelligent routing decisions. It supports the following routing rules:

**1. Host-based routing:**
Routes based on the `Host` header in the HTTP request.
```
api.example.com → Target Group: API Servers
www.example.com → Target Group: Web Servers
```

**2. Path-based routing:**
Routes based on the URL path.
```
/api/*     → Target Group: API microservice
/images/*  → Target Group: Image service
/checkout  → Target Group: Payment service
```

**3. HTTP header-based routing:**
```
Header: X-Custom-Header = "mobile" → Mobile Target Group
```

**4. HTTP method-based routing:**
```
GET  → Read replicas
POST → Primary write instances
```

**5. Query string routing:**
```
?version=v2 → New version target group
?version=v1 → Legacy target group
```

**6. Source IP CIDR-based routing**

Rules are evaluated in **priority order** (lowest number = highest priority), and a **default rule** handles unmatched requests. This enables ALB to serve as an entry point for **microservices architectures** without needing separate load balancers per service.

---

### Q2. Explain Sticky Sessions (Session Affinity) in Load Balancers. When should you use them and what are the drawbacks?

**Answer:**
**Sticky Sessions** (also called session affinity) ensure that requests from the same client are consistently routed to the same target. AWS implements this using **cookies**.

**Types of sticky session cookies:**
- **Duration-based stickiness (AWSALB cookie):** ALB generates a cookie that maps the client to a specific target. Configurable duration: 1 second to 7 days.
- **Application-based stickiness (AWSALBAPP cookie):** The application sets its own cookie; ALB uses it to maintain affinity.

**When to use:**
- Applications that store **session state locally** on the server (e.g., in-memory sessions)
- Legacy applications that cannot be refactored to use external session stores
- Shopping carts or multi-step workflows where state must persist

**Drawbacks:**
- **Uneven load distribution:** Some servers may become overloaded if they receive many long-lived sticky sessions
- **Single point of failure:** If a target becomes unhealthy, all its sticky sessions are lost
- **Scalability limitation:** Prevents true horizontal scaling benefits
- **Conflicts with modern architecture:** Stateless architectures with external session stores (ElastiCache, DynamoDB) are preferred

**Best Practice:** Use sticky sessions only as a short-term solution. The preferred approach is to externalize session state so any server can handle any request.

---

### Q3. What is Connection Draining (Deregistration Delay) and why is it important?

**Answer:**
**Connection Draining** (called **Deregistration Delay** in ALB/NLB) is the time the Load Balancer allows in-flight requests to complete before fully deregistering a target.

**How it works:**
1. A target is marked for deregistration (e.g., during scale-in, deployment, or manual removal)
2. The Load Balancer **stops sending new requests** to the deregistering target
3. Existing **in-flight connections** are allowed to complete within the configured delay period
4. After the delay expires (or all connections complete), the target is fully deregistered

**Configuration:**
- Range: **0 to 3600 seconds** (default: 300 seconds)
- Setting it to `0` disables connection draining (immediate deregistration)

**Why it's important:**
- Prevents **abrupt termination** of active user requests during deployments or scale-in events
- Ensures **graceful shutdowns** — users don't experience errors mid-transaction
- Critical for long-running operations (file uploads, streaming, payment processing)
- Essential for **zero-downtime deployments** with rolling updates

**Tuning guidance:**
- Set lower values (e.g., 30s) for short-lived API calls
- Set higher values for applications with long-running requests
- Monitor `DeregistrationDelay.ConnectionsNotYetDeregistered` CloudWatch metric

---

### Q4. How does SSL/TLS termination work with AWS Load Balancers? What are the options?

**Answer:**
SSL/TLS termination is the process of decrypting encrypted HTTPS traffic at the load balancer level. AWS provides several options:

**Option 1: SSL Termination at the Load Balancer (Most Common)**
```
Client ──[HTTPS]──► ALB (terminates SSL) ──[HTTP]──► Targets
```
- ALB/NLB decrypts traffic using a certificate stored in **AWS Certificate Manager (ACM)**
- Backend communication is unencrypted (acceptable within VPC)
- Reduces CPU overhead on application servers
- Centralized certificate management

**Option 2: SSL Pass-Through (NLB)**
```
Client ──[HTTPS]──► NLB (passes through) ──[HTTPS]──► Targets
```
- NLB operates at Layer 4 and can pass encrypted traffic directly to targets
- Targets handle SSL termination themselves
- Required for end-to-end encryption or mutual TLS (mTLS)

**Option 3: SSL Re-encryption (ALB → HTTPS Backend)**
```
Client ──[HTTPS]──► ALB (terminates + re-encrypts) ──[HTTPS]──► Targets
```
- ALB terminates the client SSL connection, then opens a new HTTPS connection to targets
- Provides end-to-end encryption while still allowing ALB to inspect HTTP headers

**Security Policies:**
- ALB/NLB support configurable **TLS security policies** (e.g., `ELBSecurityPolicy-TLS13-1-2-2021-06`)
- Policies define supported TLS versions and cipher suites
- Use the latest policies to enforce TLS 1.2+ and disable weak ciphers

**SNI (Server Name Indication):**
- ALB supports **multiple SSL certificates** via SNI
- Allows hosting multiple HTTPS domains on a single ALB listener

---

### Q5. What is the difference between an ALB and an NLB? When would you choose one over the other?

**Answer:**

| Feature | ALB | NLB |
|---------|-----|-----|
| OSI Layer | Layer 7 | Layer 4 |
| Protocols | HTTP, HTTPS, gRPC, WebSocket | TCP, UDP, TLS |
| Routing | Content-based (path, host, headers) | IP + Port |
| Latency | Milliseconds | Ultra-low (microseconds) |
| Static IP | No (DNS only) | Yes (Elastic IP per AZ) |
| Target Types | EC2, IP, Lambda, ALB | EC2, IP, ALB |
| WebSockets | Native support | Supported |
| Health Checks | HTTP/HTTPS | TCP/HTTP/HTTPS |
| Preserve Source IP | Via X-Forwarded-For header | Native (client IP preserved) |

**Choose ALB when:**
- Building HTTP/HTTPS microservices with path or host-based routing
- Need to route to Lambda functions
- Require WAF integration
- Building REST APIs or gRPC services
- Need user authentication via Cognito or OIDC

**Choose NLB when:**
- Require **static IP addresses** (e.g., whitelisting by clients or firewalls)
- Need **ultra-low latency** (gaming, financial trading, real-time applications)
- Handling **TCP/UDP traffic** (e.g., DNS, IoT, gaming protocols)
- Require **source IP preservation** natively
- Need to handle millions of requests per second
- Using **PrivateLink** to expose services

---

## Hard

### Q1. Explain the internal architecture of an ALB. How does it handle request processing, and what are the performance implications?

**Answer:**
Understanding ALB's internal architecture is critical for designing high-performance systems.

**Internal Components:**

**1. Load Balancer Nodes:**
- ALB deploys **one or more nodes per Availability Zone**
- Each node is a managed EC2 fleet that AWS scales automatically
- DNS resolution returns **all node IPs** for the enabled AZs
- Clients connect to the node closest to them (or via DNS round-robin)

**2. Request Processing Pipeline:**
```
Client Request
    │
    ▼
[TLS Termination] ← Certificate from ACM
    │
    ▼
[HTTP Parsing] ← Headers, path, method extracted
    │
    ▼
[Rule Evaluation] ← Rules evaluated in priority order
    │
    ▼
[Target Selection] ← Chosen from target group via algorithm
    │
    ▼
[Connection Pooling] ← ALB maintains connection pool to targets
    │
    ▼
[Request Forwarding] ← With X-Forwarded-For, X-Forwarded-Proto headers
```

**3. Cross-Zone Load Balancing:**
- **Enabled by default on ALB** (no additional charge)
- Each ALB node distributes traffic across all registered targets in **all enabled AZs**
- Without it, a node in AZ-A only routes to targets in AZ-A
- This prevents imbalance when AZs have different numbers of instances

**4. Routing Algorithm:**
- ALB uses **least outstanding requests (LOR)** algorithm by default
- LOR routes new requests to the target with the fewest active connections
- Alternatively, **round robin** can be configured per target group
- **Slow start mode** can gradually ramp up traffic to newly registered targets (30-900 seconds)

**5. Connection Management:**
- ALB maintains **persistent connection pools** to backend targets (HTTP keep-alive)
- This reduces TCP handshake overhead for backend connections
- The **idle timeout** (default 60 seconds) controls when idle connections are closed

**Performance Implications:**
- ALB **pre-warms** automatically but can be slow for sudden traffic spikes
- For predictable large traffic events, submit an **AWS support request for pre-warming**
- ALB adds ~1-2ms latency for rule processing vs. NLB's sub-millisecond passthrough
- Large numbers of complex routing rules can marginally increase processing time
- **WAF integration** adds additional latency (typically 1-5ms per request)

**Scaling Behavior:**
- ALB scales horizontally by adding more nodes
- The ALB DNS name resolves to multiple IPs — clients should **not cache DNS** (use short TTL)
- AWS scales ALB based on incoming traffic, but there is a brief lag during sudden spikes

---

### Q2. How would you design a multi-region active-active architecture using Load Balancers? Address failover, latency, and data consistency.

**Answer:**
This is a complex architecture requiring multiple AWS services working together.

**Architecture Overview:**
```
                    ┌─────────────────────┐
                    │    Route 53 (DNS)    │
                    │  Latency-based or    │
                    │  Geolocation routing │
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
    ┌─────────▼─────────┐             ┌─────────▼─────────┐
    │   Region: US-EAST │             │   Region: EU-WEST │
    │                   │             │                   │
    │  ┌─────────────┐  │             │  ┌─────────────┐  │
    │  │     ALB     │  │             │  │     ALB     │  │
    │  └──────┬──────┘  │             │  └──────┬──────┘  │
    │         │         │             │         │         │
    │  ┌──────▼──────┐  │             │  ┌──────▼──────┐  │
    │  │  ECS/EKS    │  │◄──────────►│  │  ECS/EKS    │  │
    │  │  Services   │  │  (data      │  │  Services   │  │
    │  └──────┬──────┘  │   sync)     │  └──────┬──────┘  │
    │         │         │             │         │         │
    │  ┌──────▼──────┐  │             │  ┌──────▼──────┐  │
    │  │  Aurora     │  │◄──────────►│  │  Aurora     │  │
    │  │  Global DB  │  │             │  │  Global DB  │  │
    │  └─────────────┘  │             │  └─────────────┘  │
    └───────────────────┘             └───────────────────┘
```

**Component Design:**

**1. Global Traffic Management (Route 53):**
- Use **latency-based routing** to direct users to the lowest-latency region
- Configure **health checks** on each ALB endpoint
- Set **failover routing** as a fallback — if one region's health check fails, all traffic routes to the healthy region