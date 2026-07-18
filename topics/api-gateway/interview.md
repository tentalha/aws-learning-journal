# API Gateway — Interview Questions

---

## Easy

### 1. What is Amazon API Gateway, and what are its primary use cases?

**Answer:**
Amazon API Gateway is a fully managed AWS service that enables developers to create, publish, maintain, monitor, and secure APIs at any scale. It acts as a "front door" for applications to access backend services.

**Primary use cases include:**
- Exposing AWS Lambda functions as HTTP endpoints (serverless APIs)
- Proxying requests to backend HTTP services or AWS services
- Building RESTful or WebSocket APIs for web and mobile applications
- Managing API versioning, throttling, and authentication in one place

---

### 2. What are the three types of APIs you can create with API Gateway?

**Answer:**

| API Type | Description |
|---|---|
| **REST API** | Full-featured API with request/response transformation, usage plans, and API keys. Supports edge-optimized, regional, and private endpoints. |
| **HTTP API** | Lightweight, lower-latency, lower-cost alternative to REST API. Supports JWT authorizers and OIDC natively. Best for simple proxy use cases. |
| **WebSocket API** | Enables two-way stateful communication between client and server. Ideal for real-time apps like chat, gaming, and live dashboards. |

---

### 3. What is the difference between an edge-optimized and a regional API endpoint in API Gateway?

**Answer:**

- **Edge-Optimized:** Requests are routed through AWS CloudFront's globally distributed edge locations before reaching the API Gateway endpoint. Best for geographically distributed clients where reduced latency for global users is important.

- **Regional:** The API is deployed in a specific AWS region and clients connect directly to that region. Best for clients within the same region or when you want to manage your own CloudFront distribution for caching/WAF control.

- **Private:** Only accessible from within a VPC using an interface VPC endpoint (powered by AWS PrivateLink). Best for internal microservices.

---

### 4. What is a Stage in API Gateway?

**Answer:**
A **Stage** is a named reference to a specific deployment of your API. It represents a snapshot of the API configuration at a point in time. Common examples include `dev`, `staging`, and `prod`.

Key characteristics:
- Each stage has its own URL endpoint (e.g., `https://{api-id}.execute-api.{region}.amazonaws.com/{stage}`)
- Stage variables can be used to parameterize configurations (e.g., pointing to different Lambda aliases or backend URLs per stage)
- Throttling, logging, caching, and canary deployments can be configured per stage

---

### 5. How does API Gateway handle throttling?

**Answer:**
API Gateway uses a **token bucket algorithm** to throttle requests and protect backends from being overwhelmed.

Two levels of throttling:
- **Account-level default:** 10,000 requests per second (RPS) with a burst of 5,000 requests across all APIs in a region.
- **Stage/method-level:** You can set custom throttling limits per stage or per individual method.

When a request exceeds the limit, API Gateway returns an **HTTP 429 Too Many Requests** error. Usage plans can also enforce per-client throttling limits using API keys.

---

## Medium

### 1. Explain the request/response lifecycle in API Gateway. What transformations can occur?

**Answer:**
The lifecycle of a request through API Gateway involves several distinct phases:

```
Client → Method Request → Integration Request → Backend
                                                    ↓
Client ← Method Response ← Integration Response ← Backend
```

**Phase Descriptions:**

1. **Method Request:** Validates the incoming client request. Can enforce required headers, query string parameters, and request body schemas using JSON Schema validation. Can also handle authentication here.

2. **Integration Request:** Transforms the client request before forwarding it to the backend. Using **Mapping Templates** (written in Velocity Template Language / VTL), you can reshape the request body, add/remove headers, and map path/query parameters.

3. **Backend Execution:** The request is sent to the integration target (Lambda, HTTP endpoint, AWS service, Mock).

4. **Integration Response:** Transforms the backend response before returning it to the client. Maps HTTP status codes and applies VTL templates to reshape the response body.

5. **Method Response:** Defines the HTTP status codes and response headers the client will receive. Used to enforce response contracts.

**Transformation capabilities:**
- Body transformation via VTL mapping templates
- Header manipulation (add, rename, remove)
- Status code mapping (e.g., map a 500 from backend to a 502 for the client)
- Content-type conversion (e.g., XML to JSON)

> **Note:** HTTP APIs do not support mapping templates — they are proxy-only. Use REST APIs when transformation is required.

---

### 2. What are the different integration types available in API Gateway?

**Answer:**

| Integration Type | Description | Use Case |
|---|---|---|
| **AWS** | Calls an AWS service action directly. Requires mapping templates for request/response. | Directly invoking SQS, DynamoDB, SNS without Lambda |
| **AWS_PROXY** (Lambda Proxy) | Passes the entire request to Lambda as a structured event. Lambda controls the response. | Standard Lambda integrations — most common |
| **HTTP** | Forwards request to an external HTTP endpoint. Supports mapping templates. | Custom HTTP backends with transformation |
| **HTTP_PROXY** | Passes the request as-is to an external HTTP endpoint. No mapping templates. | Simple reverse proxy scenarios |
| **MOCK** | Returns a response without calling any backend. API Gateway generates the response. | Testing, stub APIs, CORS preflight responses |

**Key distinction:** `_PROXY` integrations pass the raw request through and expect the backend to handle all logic. Non-proxy integrations allow transformation via mapping templates but require more configuration.

---

### 3. How does authentication and authorization work in API Gateway? What are the options?

**Answer:**
API Gateway provides multiple mechanisms for securing APIs:

**1. IAM Authorization:**
- Uses AWS Signature Version 4 (SigV4) signing
- Ideal for service-to-service communication within AWS
- Caller must have IAM permissions to invoke the API (`execute-api:Invoke`)
- Supports resource-based policies to restrict access by IP, VPC, or AWS account

**2. Amazon Cognito User Pools:**
- Validates JWT tokens issued by Cognito User Pools
- API Gateway validates the token signature and expiry automatically
- No custom code required
- Available for REST APIs only (HTTP APIs use JWT authorizers for this)

**3. Lambda Authorizers (Custom Authorizers):**
- A Lambda function is invoked to validate a token (Bearer, API key, custom header)
- Returns an IAM policy document allowing or denying access
- Two types:
  - **Token-based:** Receives a token (e.g., JWT, OAuth)
  - **Request-based:** Receives full request context (headers, query params, etc.)
- Results can be cached by TTL to reduce Lambda invocations

**4. API Keys + Usage Plans:**
- Not a security mechanism per se — intended for identifying clients and enforcing quotas/throttling
- Should always be combined with another auth method

**5. JWT Authorizers (HTTP APIs only):**
- Native support for validating JWTs from any OIDC-compliant provider
- Simpler and lower-latency than Lambda authorizers for JWT validation

---

### 4. What is a Usage Plan in API Gateway, and how does it work with API Keys?

**Answer:**
A **Usage Plan** defines throttling and quota limits for a set of API stages and associates those limits with one or more API keys. It allows you to offer different tiers of API access to different customers.

**Components:**

- **API Key:** A string value that clients include in the `x-api-key` header. Identifies the calling client.
- **Throttle settings:** Per-key RPS and burst limits
- **Quota settings:** Maximum number of requests allowed over a specified period (day, week, month)

**How they work together:**

```
API Key (client identifier)
    ↓
Usage Plan (throttle: 100 RPS, quota: 10,000/day)
    ↓
Associated API Stage(s) (prod, v2)
```

**Common pattern for SaaS:**
- Free Tier: 10 RPS, 1,000 requests/day
- Pro Tier: 100 RPS, 100,000 requests/day
- Enterprise Tier: 1,000 RPS, unlimited

> **Important:** API keys alone do NOT provide security. They identify the client but should be combined with IAM auth or a Lambda authorizer for actual authentication.

---

### 5. How does caching work in API Gateway, and when should you use it?

**Answer:**
API Gateway provides a built-in **response cache** at the stage level, backed by a dedicated cache cluster.

**Key characteristics:**
- Cache capacity: 0.5 GB to 237 GB
- Default TTL: 300 seconds (5 minutes); configurable from 0 to 3,600 seconds
- Cache can be enabled per stage and overridden per method
- Cache key is based on the request's method, path, query string parameters, and headers (configurable)
- Cache invalidation: Clients can bypass cache with the `Cache-Control: max-age=0` header (if allowed by the API configuration)
- Costs are incurred per cache instance hour

**When to use caching:**
- GET requests with stable, infrequently changing data
- Expensive backend operations (complex DB queries, slow Lambda cold starts)
- High-traffic APIs where the same request parameters are repeated frequently
- Reducing costs on downstream services (e.g., DynamoDB read costs)

**When NOT to use caching:**
- Highly personalized or user-specific responses (unless cache key includes user identifier)
- Real-time data requirements
- POST/PUT/DELETE requests (mutations should not be cached)
- Low-traffic APIs where cache cost exceeds benefit

---

## Hard

### 1. Explain how you would implement a canary deployment strategy using API Gateway. What are the mechanics and limitations?

**Answer:**
API Gateway supports **canary releases** at the stage level, allowing you to gradually shift traffic to a new deployment before fully promoting it.

**Mechanics:**

When you create a canary on a stage:
1. The stage is split into two "channels": the **current** (stable) channel and the **canary** channel
2. A configurable percentage of traffic (e.g., 5%) is routed to the canary deployment
3. Both channels share the same stage URL
4. Stage variables can be overridden specifically for the canary channel

```
Stage (prod)
├── 95% → Current Deployment (v42) → Lambda:prod alias
└──  5% → Canary Deployment (v43)  → Lambda:canary alias
```

**Configuration steps:**
```bash
aws apigateway create-deployment \
  --rest-api-id {api-id} \
  --stage-name prod \
  --canary-settings percentTraffic=5,deploymentId={new-deployment-id}
```

**Promoting or rolling back:**
- **Promote:** Updates the stage's main deployment to the canary deployment, then removes the canary. Traffic returns to 100% on the new version.
- **Rollback:** Deletes the canary configuration without promoting, returning 100% traffic to the original deployment.

**Integration with Lambda:**
For canary deployments to be meaningful end-to-end, combine with Lambda aliases:
- `prod` alias → Lambda version N (stable)
- `canary` alias → Lambda version N+1 (new)
- Use stage variables to map the canary channel to the canary Lambda alias

**Limitations:**
- Canary is only available at the stage level, not per-method
- Only one canary per stage at a time
- No automatic promotion based on metrics — must be triggered manually or via automation (e.g., CodeDeploy with Lambda)
- CloudWatch metrics for canary traffic are prefixed with `Canary` dimension but can be difficult to correlate with downstream errors

---

### 2. How does API Gateway integrate with VPC-hosted resources? What are the architectural patterns and trade-offs?

**Answer:**
API Gateway (public) cannot directly reach resources inside a VPC. Several patterns exist to bridge this gap:

**Pattern 1: VPC Link (REST API)**
- Creates a private integration using **AWS Network Load Balancer (NLB)** inside the VPC
- API Gateway connects to the NLB via a VPC Link (powered by AWS PrivateLink)
- NLB routes to EC2 instances, ECS tasks, or other VPC resources

```
Client → API Gateway → VPC Link → NLB → Private EC2/ECS/EKS
```

**Setup requirements:**
- NLB must be in the same region as the API
- The NLB's target group health checks must pass
- VPC Link creation takes several minutes (ENIs are provisioned in the VPC)

**Pattern 2: VPC Link for HTTP APIs**
- HTTP APIs support VPC Links targeting **Application Load Balancers (ALB)**, **NLBs**, or **AWS Cloud Map** service discovery
- More flexible than REST API VPC Links

**Pattern 3: Lambda as a Bridge**
- Lambda function runs inside the VPC (attached to VPC subnets)
- API Gateway (AWS_PROXY) invokes Lambda, which then calls VPC resources
- Simpler to set up but adds Lambda invocation latency and cost
- Lambda cold starts can affect tail latency

**Pattern 4: Private API Gateway**
- Deploy a **private REST API** accessible only from within the VPC via an Interface VPC Endpoint
- Useful for internal microservice communication
- Requires a resource policy to allow access from specific VPCs or VPC endpoints

**Trade-offs:**

| Pattern | Latency | Cost | Complexity | Use Case |
|---|---|---|---|---|
| VPC Link + NLB | Low | Medium | Medium | High-throughput, existing NLB |
| VPC Link + ALB (HTTP API) | Low | Medium | Low | HTTP API with ALB routing |
| Lambda Bridge | Medium | Higher | Low | Simple, low-traffic scenarios |
| Private API | Low | Low | Medium | Internal-only APIs |

---

### 3. Describe the cold start problem in API Gateway + Lambda integrations and all available mitigation strategies.

**Answer:**
While the cold start technically occurs in Lambda, it manifests as increased latency on API Gateway responses and is a critical concern for production APIs.

**Understanding the cold start chain:**
```
Client Request
    → API Gateway (processing: ~1-5ms)
    → Lambda Cold Start: Container init + Runtime init + Handler init
        ├── Container provisioning: 100-500ms
        ├── Runtime initialization (JVM, .NET): up to 1-2s
        └── Handler initialization (imports, DB connections): variable
    → Lambda Execution
    → Response
```

**Mitigation Strategies:**

**1. Provisioned Concurrency (most effective):**
- Pre-initializes a specified number of Lambda execution environments
- Eliminates cold starts for the provisioned count
- Can be set on a Lambda alias or version
- Cost: charged per GB-second of provisioned concurrency
- Auto Scaling: Use Application Auto Scaling to scale provisioned concurrency based on schedules or CloudWatch metrics

**2. Lambda SnapStart (Java 11+):**
- Takes a snapshot of the initialized execution environment after the `init` phase
- Restores from snapshot on subsequent invocations
- Reduces Java cold starts from 1-2s to ~200ms
- Available for Java 11 and Java 17 managed runtimes

**3. Keep-Warm (EventBridge Scheduled Pings):**
- Schedule EventBridge rules to invoke Lambda every 5 minutes
- Forces Lambda to maintain at least one warm instance
- Not reliable for high-concurrency scenarios
- Free but not guaranteed

**4. Runtime and Language Choice:**
- Prefer runtimes with fast init: Python, Node.js (~50-100ms) vs. Java, .NET (~500ms-2s)
- Use AWS Lambda Powertools to optimize initialization code

**5. Minimize Initialization Code:**
- Move expensive operations (DB connections, SDK clients) outside the handler but inside the module scope
- Use lazy initialization for rarely-used resources
- Reduce deployment package size (smaller packages = faster container init)

**6. Optimize Memory Allocation:**
- More memory = more CPU = faster initialization
- Use AWS Lambda Power Tuning tool to find optimal memory/cost balance

**7. HTTP API vs. REST API:**
- HTTP APIs have ~60% lower latency overhead compared to REST APIs
- For latency-sensitive APIs, HTTP API reduces the API Gateway processing time

**Monitoring cold starts:**
```
CloudWatch Logs Insights:
fields @timestamp, @duration, @initDuration
| filter @initDuration > 0
| stats avg(@initDuration), max(@initDuration), count() by bin(5m)
```
The presence of `@initDuration` in logs indicates a cold start occurred.

---

### 4. How would you