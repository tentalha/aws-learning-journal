# WAF

## What is it?

**AWS WAF (Web Application Firewall)** is a managed security service that helps protect web applications and APIs against common web exploits, bots, and malicious traffic that could compromise security, availability, or consume excessive resources.

- **Official Service Name:** AWS WAF
- **Category:** Security, Identity & Compliance → Network Security
- **Current Version:** AWS WAF (v2, also referred to as WAFv2) — the original WAF Classic is deprecated for new deployments
- **Scope:** Regional (with global support when attached to CloudFront)

AWS WAF operates at **Layer 7 (Application Layer)** of the OSI model, allowing inspection and filtering of HTTP/HTTPS requests based on configurable rules before they reach your application.

```
Internet → CloudFront / ALB / API Gateway / AppSync
                    ↓
              [AWS WAF Rules Engine]
                    ↓
         Allow / Block / Count / CAPTCHA
                    ↓
            Your Web Application
```

---

## Why do we need it?

### The Problem

Modern web applications face a constantly evolving threat landscape:

- **SQL Injection (SQLi):** Attackers embed malicious SQL into input fields to manipulate databases.
- **Cross-Site Scripting (XSS):** Malicious scripts injected into web pages to steal session tokens or credentials.
- **DDoS at Layer 7:** HTTP floods that overwhelm application servers even if network-layer attacks are blocked.
- **Bot Traffic:** Scrapers, credential stuffers, and automated scanners consuming bandwidth and compute.
- **OWASP Top 10 Vulnerabilities:** A wide class of well-known attack patterns.
- **API Abuse:** Excessive API calls, parameter tampering, or unauthorized access attempts.

### When to Use AWS WAF

| Scenario | Why WAF |
|---|---|
| E-commerce platform | Protect checkout flows from SQL injection and credential stuffing |
| SaaS application | Rate-limit API endpoints, block malicious bots |
| Healthcare portal | Compliance (HIPAA) requires application-layer protections |
| Media streaming site | Block scrapers and content leeches |
| Financial services API | Prevent fraud via bot detection and geo-blocking |
| Gaming backend | Protect leaderboard APIs from manipulation |

### Business Justification

- Reduces risk of **data breaches** (average cost: $4.45M per IBM 2023 report)
- Helps achieve **compliance** (PCI-DSS, HIPAA, SOC 2)
- Prevents **revenue loss** from downtime caused by application-layer attacks
- Reduces engineering burden vs. building custom filtering logic

---

## Internal Working

### Request Processing Pipeline

When a request arrives at a WAF-protected resource, it goes through the following pipeline:

```
Incoming HTTP/HTTPS Request
          │
          ▼
┌─────────────────────────┐
│   Web ACL Evaluation    │
│                         │
│  1. Default Action      │◄── Applied if no rules match
│                         │
│  2. Rule Groups         │
│     ├── Rule 1 (P:0)    │◄── Lowest priority evaluated first
│     ├── Rule 2 (P:1)    │
│     ├── Rule Group A    │
│     │   ├── Rule A1     │
│     │   └── Rule A2     │
│     └── Rule 3 (P:100)  │
│                         │
│  3. First Match Wins    │◄── Evaluation stops on first match
└─────────────────────────┘
          │
          ▼
   Action: Allow / Block / Count / CAPTCHA / Challenge
```

### Rule Evaluation Logic

1. **Rules are evaluated in priority order** (lower number = higher priority).
2. **First matching rule's action is applied** — evaluation does not continue after a match (unless using `Count` action with `ContinueEvaluation`).
3. **Statements** define the matching conditions using various inspection components.
4. **Actions** define what happens when a rule matches.

### Matching Statements (Conditions)

AWS WAF supports these statement types:

| Statement Type | Description |
|---|---|
| `ByteMatchStatement` | Match specific strings in request components |
| `RegexMatchStatement` | Match regex patterns |
| `RegexPatternSetReferenceStatement` | Reference a reusable regex set |
| `IPSetReferenceStatement` | Match against IP address sets |
| `GeoMatchStatement` | Match based on geographic origin |
| `SizeConstraintStatement` | Match based on size of request component |
| `SqliMatchStatement` | Detect SQL injection patterns |
| `XssMatchStatement` | Detect cross-site scripting patterns |
| `RateBasedStatement` | Rate limiting over a time window |
| `ManagedRuleGroupStatement` | AWS or Marketplace managed rules |
| `LabelMatchStatement` | Match labels added by previous rules |
| `AndStatement` / `OrStatement` / `NotStatement` | Logical operators for combining statements |

### Request Components Inspectable

- URI path
- Query string
- HTTP method
- HTTP headers (all or specific)
- HTTP body (first 8 KB by default, up to 64 KB with oversize handling)
- Cookies
- JSON body (with JSON parsing)
- Single query parameter or all query parameters

### Token-Based Bot Control

AWS WAF Bot Control uses **challenge tokens** stored in browser cookies to distinguish humans from bots:

```
Browser Request (no token)
        │
        ▼
WAF issues JavaScript Challenge
        │
        ▼
Browser executes JS, obtains token
        │
        ▼
Browser retries with token in cookie
        │
        ▼
WAF validates token → Allows request
```

---

## Architecture

### Core Components

```
┌──────────────────────────────────────────────────────────┐
│                        AWS WAF                           │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │                   Web ACL                         │   │
│  │                                                   │   │
│  │  ┌─────────────┐  ┌─────────────────────────┐   │   │
│  │  │  IP Sets    │  │   Regex Pattern Sets     │   │   │
│  │  └─────────────┘  └─────────────────────────┘   │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐   │   │
│  │  │           Rule Groups                     │   │   │
│  │  │  ┌──────────────┐ ┌───────────────────┐  │   │   │
│  │  │  │  AWS Managed │ │  Custom Rule      │  │   │   │
│  │  │  │  Rule Groups │ │  Groups           │  │   │   │
│  │  │  └──────────────┘ └───────────────────┘  │   │   │
│  │  └──────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  Default Action: [Allow | Block]                  │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  CloudWatch  │  │  Kinesis     │  │  S3 Logs     │  │
│  │  Metrics     │  │  Firehose    │  │  (via KDF)   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Deployment Architecture Patterns

#### Pattern 1: CloudFront + WAF (Global Edge Protection)

```
Users → Route 53 → CloudFront (Global WAF) → ALB → EC2/ECS
                        ↑
                   WAF (us-east-1)
                   [Global scope]
```

> **Note:** WAF Web ACLs for CloudFront must be created in `us-east-1` regardless of where CloudFront distributions are deployed.

#### Pattern 2: ALB + WAF (Regional Protection)

```
Users → Route 53 → ALB (Regional WAF) → Target Group → ECS/Lambda
                        ↑
                   WAF (same region)
                   [Regional scope]
```

#### Pattern 3: API Gateway + WAF

```
Mobile/Web Clients → API Gateway (REST/HTTP) → Lambda
                          ↑
                     WAF (Regional)
                     [Rate limiting, Auth bypass protection]
```

#### Pattern 4: AppSync + WAF

```
GraphQL Clients → AppSync → DynamoDB/Lambda
                     ↑
                WAF (Regional)
                [GraphQL-aware filtering]
```

### Web ACL Scope

| Scope | Use Case | Region Constraint |
|---|---|---|
| `CLOUDFRONT` | Protecting CloudFront distributions | Must be in `us-east-1` |
| `REGIONAL` | ALB, API Gateway, AppSync, Cognito | Same region as resource |

---

## Real World Example

### Scenario: Protecting an E-Commerce Platform

**Company:** RetailCo — an online retailer with 2M monthly users
**Infrastructure:** CloudFront → ALB → ECS (Node.js) → RDS Aurora
**Threats:** SQL injection, bot scraping of product prices, credential stuffing on login, DDoS

#### Step-by-Step Implementation

**Step 1: Create IP Sets for Known Malicious IPs**

```bash
aws wafv2 create-ip-set \
  --name "BlockedIPs" \
  --scope REGIONAL \
  --ip-address-version IPV4 \
  --addresses "192.0.2.0/24" "198.51.100.0/24" \
  --region us-east-1
```

**Step 2: Create Web ACL with layered rules**

```
Web ACL: RetailCo-WAF
Priority 0:  Block known bad IPs (IP Set)
Priority 1:  AWS Managed - AWSManagedRulesCommonRuleSet
Priority 2:  AWS Managed - AWSManagedRulesSQLiRuleSet
Priority 3:  AWS Managed - AWSManagedRulesBotControlRuleSet (Common)
Priority 4:  Rate limit /api/login → 100 req/5min per IP
Priority 5:  Rate limit /api/search → 500 req/1min per IP
Priority 6:  Block requests from sanctioned countries (GeoMatch)
Priority 7:  Block requests with body > 10KB to /api/checkout
Default:     Allow
```

**Step 3: Configure Login Endpoint Rate Limiting**

```json
{
  "Name": "RateLimitLogin",
  "Priority": 4,
  "Statement": {
    "RateBasedStatement": {
      "Limit": 100,
      "AggregateKeyType": "IP",
      "ScopeDownStatement": {
        "ByteMatchStatement": {
          "SearchString": "/api/login",
          "FieldToMatch": { "UriPath": {} },
          "TextTransformations": [{ "Priority": 0, "Type": "LOWERCASE" }],
          "PositionalConstraint": "STARTS_WITH"
        }
      }
    }
  },
  "Action": { "Block": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "RateLimitLogin"
  }
}
```

**Step 4: Enable WAF Logging to S3**

```bash
aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "arn:aws:wafv2:us-east-1:123456789:regional/webacl/RetailCo-WAF/abc123",
    "LogDestinationConfigs": [
      "arn:aws:firehose:us-east-1:123456789:deliverystream/aws-waf-logs-retailco"
    ],
    "RedactedFields": [
      {"SingleHeader": {"Name": "authorization"}}
    ]
  }'
```

**Step 5: Associate Web ACL with ALB**

```bash
aws wafv2 associate-web-acl \
  --web-acl-arn "arn:aws:wafv2:us-east-1:123456789:regional/webacl/RetailCo-WAF/abc123" \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/retailco-alb/xyz"
```

**Result:**
- SQL injection attempts blocked at WAF, never reaching RDS
- Bot scrapers identified and CAPTCHA-challenged
- Brute force on login throttled to 100 attempts/5 min per IP
- Traffic from sanctioned countries blocked
- Zero changes required to application code

---

## Advantages

### 1. **Managed Threat Intelligence**
AWS Managed Rule Groups are continuously updated by the AWS Threat Research Team based on real-world attack data — no manual signature updates required.

### 2. **Deep Request Inspection**
Inspects HTTP headers, URI, query strings, body, and cookies — full Layer 7 visibility that network firewalls cannot provide.

### 3. **Flexible Rule Logic**
Logical operators (`AND`, `OR`, `NOT`) enable complex, compound matching conditions for sophisticated threat detection.

### 4. **Label-Based Rule Chaining**
Rules can add labels to requests; subsequent rules can match on those labels — enabling multi-stage inspection pipelines.

### 5. **Near Real-Time Protection**
Rule changes propagate globally within seconds, enabling rapid response to emerging threats.

### 6. **Serverless & Fully Managed**
No infrastructure to provision, patch, or scale — AWS handles all capacity management.

### 7. **AWS Marketplace Rule Groups**
Third-party managed rules from vendors like F5, Imperva, and Fortinet extend protection with specialized threat intelligence.

### 8. **Cost-Effective at Scale**
Compared to deploying and managing a self-hosted WAF, AWS WAF has predictable pricing with no upfront costs.

### 9. **Bot Control & Fraud Prevention**
Dedicated managed rule groups for bot detection, account takeover prevention (ATP), and account creation fraud prevention (ACFP).

### 10. **CAPTCHA and Challenge Actions**
Native CAPTCHA integration without requiring third-party services.

---

## Limitations

### Hard Limits (WAFv2)

| Resource | Default Limit | Adjustable |
|---|---|---|
| Web ACLs per account per region | 100 | Yes |
| Rules per Web ACL | 1,500 WCU | Yes |
| Rule Groups per account per region | 100 | Yes |
| IP Sets per account per region | 100 | Yes |
| Regex Pattern Sets per account per region | 100 | Yes |
| Patterns per Regex Pattern Set | 10 | Yes |
| IP addresses per IP Set | 10,000 | Yes |
| Web ACL Capacity Units (WCU) per Web ACL | 1,500 | Yes (up to 5,000) |
| Labels per rule | 20 | No |
| Rate-based rule evaluation window | 1 min, 2 min, 5 min, 10 min | N/A |

### Functional Limitations

- **Body Inspection Limit:** By default, only the **first 8 KB** of the request body is inspected. Can be increased to 16 KB, 32 KB, or 64 KB using oversize handling, but larger payloads are not fully inspectable.
- **No Response Inspection (Standard):** WAF inspects requests, not responses (except with Fraud Control ATP/ACFP which can inspect responses).
- **WebSocket Limitation:** WAF does not inspect WebSocket traffic after the initial HTTP upgrade handshake.
- **gRPC:** Limited support — gRPC over HTTP/2 is not natively inspected as gRPC.
- **CloudFront Scope Only in us-east-1:** CloudF