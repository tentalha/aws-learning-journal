# API Gateway

## What is it?

**Amazon API Gateway** is a fully managed AWS service that enables developers to create, publish, maintain, monitor, and secure APIs at any scale. It acts as a "front door" for applications to access data, business logic, or functionality from backend services such as AWS Lambda functions, Amazon EC2, Amazon DynamoDB, or any publicly accessible HTTP endpoint.

API Gateway supports three types of APIs:

| API Type | Protocol | Best For |
|---|---|---|
| **REST API** | HTTP/HTTPS | Feature-rich REST APIs with full API management capabilities |
| **HTTP API** | HTTP/HTTPS | Low-latency, cost-effective REST APIs (simpler feature set) |
| **WebSocket API** | WebSocket | Real-time, two-way communication applications |

- **Service Category:** Networking & Content Delivery / Application Integration
- **Service Type:** Fully Managed PaaS
- **Availability:** Available in all AWS commercial regions and GovCloud

---

## Why do we need it?

### The Problem Without API Gateway

Without a managed API layer, teams face significant challenges:

- **No unified entry point:** Backend services are scattered, each with different endpoints, auth mechanisms, and rate limits.
- **Security exposure:** Backend services would need to be directly internet-facing, increasing the attack surface.
- **No throttling or rate limiting:** Uncontrolled traffic can overwhelm backends and lead to cascading failures.
- **No versioning:** Deploying new API versions without breaking existing clients is complex.
- **No monitoring:** Tracking request counts, latency, and errors across distributed services is difficult.
- **No transformation:** Clients and backends may speak different data formats or protocols.

### When to Use API Gateway

| Scenario | Why API Gateway |
|---|---|
| Serverless applications (Lambda) | Native integration, no server management |
| Microservices aggregation | Single entry point, routing to multiple services |
| Mobile/web backend | Auth, throttling, caching out of the box |
| Third-party API monetization | Usage plans, API keys, quota management |
| Legacy system modernization | Facade over existing HTTP endpoints |
| Real-time apps (chat, gaming) | WebSocket API support |

### Real Business Scenarios

1. **E-Commerce Platform:** A retailer exposes product catalog, cart, and order APIs through API Gateway, using Cognito for customer auth, Lambda for business logic, and DynamoDB for storage — all without managing servers.

2. **FinTech Application:** A payment processor uses API Gateway to enforce strict throttling (preventing fraud attempts), mutual TLS for client authentication, and WAF integration for OWASP protection.

3. **SaaS Multi-Tenant API:** A SaaS company uses API keys and usage plans to meter API consumption per customer tier (Free: 1000 req/day, Pro: 100,000 req/day, Enterprise: unlimited).

---

## Internal Working

### Request Lifecycle

When a client makes an API request to API Gateway, it goes through a well-defined pipeline:

```
Client Request
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│                    API Gateway Edge                      │
│  (CloudFront PoP for Edge-Optimized / Regional Endpoint) │
└─────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│                  Method Request                          │
│  - Validate API key                                      │
│  - Check authorization (IAM / Cognito / Lambda Auth)     │
│  - Validate request parameters & body (Model/Validator)  │
└─────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│                Integration Request                       │
│  - Map/transform incoming request (Mapping Templates)    │
│  - Set integration type (Lambda, HTTP, AWS Service, Mock)│
│  - Apply request parameters mapping                      │
└─────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│                  Backend Integration                     │
│  (Lambda / HTTP Endpoint / AWS Service / Mock)           │
└─────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│               Integration Response                       │
│  - Map backend response to API response                  │
│  - Apply response mapping templates                      │
│  - Handle error patterns                                 │
└─────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│                  Method Response                         │
│  - Define HTTP status codes                              │
│  - Set response headers                                  │
│  - Apply response models                                 │
└─────────────────────────────────────────────────────────┘
     │
     ▼
Client Response (with optional caching layer)
```

### Key Internal Components

#### 1. Stages
A **Stage** is a named reference to a deployment of your API (e.g., `dev`, `staging`, `prod`). Each stage:
- Has its own URL: `https://{api-id}.execute-api.{region}.amazonaws.com/{stage}`
- Can have independent throttle settings, caching, and logging configurations
- Supports stage variables (like environment variables for API Gateway)

#### 2. Deployments
A **Deployment** is a snapshot of your API configuration at a point in time. You must deploy your API to a stage for changes to take effect. This creates an immutable deployment artifact.

#### 3. Integration Types

| Type | Description | Use Case |
|---|---|---|
| `AWS_PROXY` (Lambda Proxy) | Passes entire request to Lambda, Lambda returns full response | Most Lambda integrations |
| `AWS` | Full control over request/response mapping to AWS services | Direct DynamoDB, SQS, SNS integration |
| `HTTP_PROXY` | Passes request as-is to HTTP backend | Simple HTTP forwarding |
| `HTTP` | Full control mapping to HTTP backend | Custom request/response transformation |
| `MOCK` | Returns response without calling backend | Testing, CORS preflight |

#### 4. Mapping Templates (VTL)
For non-proxy integrations, API Gateway uses **Velocity Template Language (VTL)** to transform request/response payloads. This allows reshaping JSON, extracting headers, and mapping parameters.

#### 5. Caching
API Gateway has a built-in **response cache** (0.5 GB to 237 GB). Cached responses are stored per stage and can be keyed on query strings and headers. TTL ranges from 0 to 3600 seconds (default: 300 seconds).

---

## Architecture

### Core Architectural Patterns

#### Pattern 1: Serverless REST API (Most Common)

```
                    ┌──────────────┐
                    │   Route 53   │
                    │ (Custom DNS) │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  CloudFront  │ (Optional CDN layer)
                    │    + WAF     │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  API Gateway │
                    │  (REST API)  │
                    └──────┬───────┘
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──┐  ┌──────▼──┐  ┌─────▼───┐
       │ Lambda  │  │ Lambda  │  │ Lambda  │
       │ /users  │  │/products│  │ /orders │
       └──────┬──┘  └──────┬──┘  └─────┬───┘
              │            │            │
       ┌──────▼──┐  ┌──────▼──┐  ┌─────▼───┐
       │DynamoDB │  │DynamoDB │  │   RDS   │
       └─────────┘  └─────────┘  └─────────┘
```

#### Pattern 2: API Gateway with VPC Integration (Private APIs)

```
Internet
    │
    ▼
API Gateway (Regional)
    │
    │ (VPC Link)
    ▼
Network Load Balancer (in VPC)
    │
    ▼
Private ECS/EKS Services or EC2 instances
    │
    ▼
Private RDS / ElastiCache
```

#### Pattern 3: WebSocket API for Real-Time Applications

```
Client 1 ──────┐
Client 2 ──────┤──► API Gateway ──► Lambda (connect/disconnect/message)
Client N ──────┘   (WebSocket)          │
                                        ▼
                                   DynamoDB
                                 (connection store)
                                        │
                              Lambda (broadcast)
                              sends to all connections
                              via Management API
```

#### Pattern 4: Direct AWS Service Integration

```
Client ──► API Gateway ──► DynamoDB (PutItem / GetItem)
       ──► API Gateway ──► SQS (SendMessage)
       ──► API Gateway ──► SNS (Publish)
       ──► API Gateway ──► Step Functions (StartExecution)
       ──► API Gateway ──► Kinesis (PutRecord)
```
This pattern eliminates Lambda as a middleman for simple CRUD operations, reducing latency and cost.

### Endpoint Types

| Type | Description | Use Case |
|---|---|---|
| **Edge-Optimized** | Deployed to CloudFront edge PoPs | Global clients, reduced latency worldwide |
| **Regional** | Deployed in a specific AWS region | Clients in same region, custom CloudFront setup |
| **Private** | Accessible only within VPC via VPC Endpoint | Internal microservices, no internet exposure |

---

## Real World Example

### Scenario: Building a Serverless E-Commerce Order Management API

**Business Requirement:** A retail company needs an API to create orders, retrieve order status, and cancel orders. It must support 10,000 concurrent users, require authentication, and have per-customer rate limiting.

#### Step-by-Step Walkthrough

**Step 1: Design the API Resources**
```
POST   /orders              → Create new order
GET    /orders/{orderId}    → Get order by ID
DELETE /orders/{orderId}    → Cancel order
GET    /orders?status=PENDING → List orders by status
```

**Step 2: Create the Lambda Functions**

Each endpoint maps to a Lambda function:
- `createOrderFunction` → validates cart, charges payment, writes to DynamoDB
- `getOrderFunction` → reads from DynamoDB, returns order details
- `cancelOrderFunction` → validates state machine, updates DynamoDB, triggers refund via SNS

**Step 3: Configure API Gateway**

```
1. Create REST API (Regional endpoint)
2. Create /orders resource
3. Add POST method → Lambda Proxy Integration → createOrderFunction
4. Create /orders/{orderId} resource
5. Add GET method → Lambda Proxy Integration → getOrderFunction
6. Add DELETE method → Lambda Proxy Integration → cancelOrderFunction
7. Enable request validation (body for POST, path params for GET/DELETE)
8. Attach Cognito User Pool Authorizer to all methods
```

**Step 4: Configure Authorizer**
```
- Type: Cognito User Pool Authorizer
- User Pool: ecommerce-users-pool
- Token Source: Authorization header (Bearer token)
- Token Validation: Validates JWT signature, expiry, audience
```

**Step 5: Set Up Usage Plans & API Keys (B2B Partners)**
```
- Bronze Plan: 1,000 req/day, 10 req/sec burst
- Silver Plan: 50,000 req/day, 100 req/sec burst  
- Gold Plan: 500,000 req/day, 1,000 req/sec burst
- Create API key per partner, associate with appropriate plan
```

**Step 6: Enable Caching for GET Endpoints**
```
- Stage: prod
- Cache size: 6.1 GB
- Default TTL: 300 seconds
- Cache key: {orderId} path parameter
- Invalidation: Lambda writes to cache invalidation queue
```

**Step 7: Configure Custom Domain**
```
- Domain: api.mystore.com
- ACM Certificate: *.mystore.com (us-east-1 for edge-optimized)
- Base Path Mapping: /v1 → prod stage
- Route 53: Alias record → API Gateway domain
```

**Step 8: Enable Logging & Monitoring**
```
- Access logging: Custom JSON format to CloudWatch Logs
- Execution logging: INFO level for prod
- X-Ray tracing: Enabled
- CloudWatch Alarms: 5xx errors > 1%, latency P99 > 3 seconds
```

**Result:** A production-ready, serverless API handling 10K concurrent users with authentication, rate limiting, caching, and full observability — zero servers to manage.

---

## Advantages

1. **Fully Managed:** No infrastructure to provision, patch, or scale. AWS handles all the undifferentiated heavy lifting.

2. **Seamless Lambda Integration:** Native Lambda proxy integration with automatic permission management makes serverless development fast and simple.

3. **Multiple Authorization Mechanisms:** Supports IAM Sig V4, Cognito User Pools, Lambda Authorizers (custom auth), and API keys — often combined.

4. **Built-in Throttling & DDoS Protection:** Per-account, per-stage, and per-method throttling protects backends. Integrates with AWS Shield Standard by default.

5. **Request/Response Transformation:** VTL mapping templates allow protocol translation and payload reshaping without additional compute.

6. **Canary Deployments:** Built-in support for canary releases — route a percentage of traffic to a new API version while keeping most traffic on the stable version.

7. **SDK Generation:** API Gateway can auto-generate client SDKs for JavaScript, iOS, Android, and Java from your API definition.

8. **OpenAPI/Swagger Support:** Import and export API definitions in OpenAPI 3.0 format, enabling design-first API development.

9. **WebSocket Support:** Native WebSocket APIs with connection management, enabling real-time bidirectional communication without managing WebSocket servers.

10. **Cost Efficiency:** Pay per API call — no minimum fees or upfront costs. HTTP APIs are ~71% cheaper than REST APIs for applicable use cases.

11. **Multi-Region Availability:** Deploy APIs in multiple regions behind Route 53 for active-active or active-passive failover.

12. **Private API Support:** VPC-private APIs accessible only within your network, ideal for internal microservices.

---

## Limitations

### Hard Limits (Default Quotas — Adjustable)

| Limit | Default Value | Notes |
|---|---|---|
| Throttling burst limit | 5,000 req/sec (account-wide) | Adjustable via support ticket |
| Throttling steady-state rate | 10,000 req/sec (account-wide) | Adjustable via support ticket |
| Max integration timeout | **29 seconds** | **Hard limit — cannot be increased** |
| Max payload size | 10 MB | Hard limit for REST/HTTP APIs |
| Max WebSocket message size | 128 KB | Hard limit |
| Max WebSocket connection duration | 2 hours | Hard limit |
| Max stages per API | 10 | Adjustable |
| Max resources per REST API | 300 | Adjustable |
| Max APIs per account per region | 600 (REST) | Adjustable |
| Cache size options | 0.5 GB to 237 GB | Fixed tiers |
| Max Lambda authorizer result TTL | 3600 seconds | Hard limit |
| Max mapping template size | 300 KB | Hard limit |

### Feature Limitations

- **29-second timeout is critical:** Long-running operations (video processing, ML inference) cannot be handled synchronously. Must use async patterns (SQS + polling, Step Functions, WebSockets).
- **VTL Complexity:** Velocity Template Language is powerful but notoriously difficult to debug and maintain for complex transformations.
- **Cold Start Amplification:** Lambda cold starts behind API Gateway can cause latency spikes, especially for infrequently called endpoints.
- **No Native gRPC Support:** API Gateway does not support gRPC protocol natively (use ALB or