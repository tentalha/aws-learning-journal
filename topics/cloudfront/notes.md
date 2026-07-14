# CloudFront

## What is it?

**Amazon CloudFront** is a fully managed, globally distributed **Content Delivery Network (CDN)** service provided by AWS. It belongs to the **Networking & Content Delivery** category of AWS services.

CloudFront accelerates the delivery of static and dynamic web content, APIs, video streams, and applications to end users by caching content at a global network of **edge locations** and **regional edge caches** physically closer to the user. It integrates natively with AWS services like S3, EC2, ALB, API Gateway, Lambda@Edge, and CloudFront Functions to provide a programmable, secure, and high-performance delivery layer.

**Key identifiers:**
- Service type: Content Delivery Network (CDN)
- Delivery protocol: HTTP/HTTPS, WebSocket, RTMP (legacy)
- Global infrastructure: 600+ Points of Presence (PoPs) across 90+ cities in 48 countries
- Supports HTTP/2 and HTTP/3 (QUIC)
- Integrated with AWS Shield Standard (DDoS protection) at no extra cost

---

## Why do we need it?

### The Problem Without CloudFront

Without a CDN, every user request travels from the client all the way to your **origin server** (e.g., an S3 bucket in `us-east-1` or an EC2 instance in `eu-west-1`). This introduces:

- **High latency** for geographically distant users (e.g., a user in Tokyo hitting a server in Virginia experiences 150–200ms round-trip time)
- **Origin overload** — every request hits your backend directly, increasing compute and bandwidth costs
- **Poor user experience** — slow page loads lead to higher bounce rates and lower conversion
- **No built-in DDoS protection** at the network edge
- **Increased egress costs** — data transfer from AWS regions is expensive at scale

### When to Use CloudFront

| Scenario | Why CloudFront Helps |
|---|---|
| Global e-commerce website | Serve product images and JS bundles from edge, reduce cart abandonment |
| Video streaming platform | Smooth HLS/DASH delivery via edge caching |
| SaaS application with global users | Reduce API latency with edge caching and Lambda@Edge |
| Static website on S3 | Serve content securely via HTTPS without exposing S3 directly |
| Software download distribution | Offload origin bandwidth, accelerate large file downloads |
| Real-time gaming | Reduce latency for leaderboard APIs and game assets |
| Financial services APIs | Enforce WAF rules at the edge before requests reach the origin |

### Business Impact
- Netflix, Amazon.com, and Airbnb use CDNs to reduce page load times by 50–80%
- A 100ms reduction in load time can increase conversion rates by ~1% (Akamai/Google research)
- CloudFront can reduce origin egress costs by caching frequently requested objects

---

## Internal Working

### Request Lifecycle

```
User Browser
     │
     ▼
DNS Resolution (Route 53 / User's DNS)
     │  ← Returns IP of nearest CloudFront Edge Location (Anycast routing)
     ▼
CloudFront Edge Location (PoP)
     │
     ├── Cache HIT? ──► Return cached response to user (fastest path)
     │
     └── Cache MISS?
           │
           ▼
     Regional Edge Cache (REC)
           │
           ├── Cache HIT? ──► Return cached response, populate edge PoP
           │
           └── Cache MISS?
                 │
                 ▼
           Origin (S3 / ALB / EC2 / API GW / Custom HTTP)
                 │
                 ▼
           Response travels back through REC → Edge → User
           (Response is cached at each layer per TTL rules)
```

### Layer-by-Layer Breakdown

#### 1. DNS-Based Routing (Anycast)
CloudFront uses **Anycast IP routing**. When a user resolves `d1234abcd.cloudfront.net`, AWS's global DNS infrastructure returns the IP address of the **closest edge location** based on network latency, not just geographic distance. This is handled transparently — users don't choose their edge location.

#### 2. Edge Locations (Points of Presence)
- Smallest and most numerous tier (600+)
- Store cached copies of content
- Execute **CloudFront Functions** and **Lambda@Edge** (Viewer request/response)
- Terminate TLS connections (SSL offloading)
- Apply WAF rules
- Cache TTL is typically shorter here

#### 3. Regional Edge Caches (RECs)
- ~13 globally distributed
- Larger cache storage than edge PoPs
- Sit between edge PoPs and origins
- Reduce origin load by serving as a second-tier cache
- Execute **Lambda@Edge** (Origin request/response)
- If an edge PoP has a cache miss, it checks the REC before going to origin

#### 4. Origin Fetch
When content is not cached at any tier:
- CloudFront opens a persistent, keep-alive TCP connection to the origin
- Uses **Origin Shield** (optional) as an additional caching layer in front of the origin
- Requests travel over AWS's private backbone network (not public internet) to the origin

#### 5. Cache Key and TTL
CloudFront determines what to cache based on the **cache key**:
- Default: URL path only
- Configurable: headers, query strings, cookies
- TTL controlled by: `Cache-Control: max-age`, `Expires` headers, or CloudFront cache policies
- Minimum TTL: 0 seconds, Maximum TTL: 31,536,000 seconds (1 year)

#### 6. Compression
CloudFront automatically compresses objects using **Gzip** or **Brotli** if:
- The viewer supports it (`Accept-Encoding` header)
- The object is compressible (text-based, JSON, HTML, CSS, JS)
- Object size is between 1,000 bytes and 10 MB

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    CloudFront Distribution                       │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │   Behaviors  │    │  Cache       │    │  Origin Groups   │  │
│  │  (Path-based │    │  Policies    │    │  (Failover)      │  │
│  │   routing)   │    │              │    │                  │  │
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  Origins     │    │  Functions   │    │  Security        │  │
│  │  (S3/ALB/   │    │  (Lambda@   │    │  (WAF/Shield/    │  │
│  │   Custom)    │    │   Edge/CF   │    │   OAC/Geo)       │  │
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

#### Distribution
The top-level CloudFront resource. Every distribution gets a unique domain (`xxxxx.cloudfront.net`). You can attach custom domains (CNAMEs) with SSL certificates from **ACM (us-east-1 only)**.

Two distribution types exist historically:
- **Web distributions** — HTTP/HTTPS content (current standard)
- **RTMP distributions** — Flash media streaming (deprecated, no longer supported for new distributions)

#### Origins
The source of truth for your content. CloudFront supports:

| Origin Type | Use Case |
|---|---|
| **S3 Bucket** | Static assets, SPAs, media files |
| **S3 Website Endpoint** | Static website hosting with redirect support |
| **Application Load Balancer** | Dynamic web applications |
| **EC2 Instance** | Custom web servers |
| **API Gateway** | REST/HTTP APIs |
| **Lambda Function URL** | Serverless compute |
| **Custom HTTP Origin** | Any publicly accessible HTTP server |
| **MediaPackage / MediaStore** | Video streaming |

#### Cache Behaviors
Path-pattern-based routing rules within a distribution. Each behavior maps a URL pattern to:
- A specific origin or origin group
- Cache policy
- Origin request policy
- Response headers policy
- Function associations (Lambda@Edge or CloudFront Functions)
- Viewer protocol policy (HTTP/HTTPS redirect, HTTPS-only)
- Allowed HTTP methods

**Example behavior patterns:**
```
/api/*          → ALB origin (no caching, forward all headers)
/images/*       → S3 origin (cache 7 days, compress)
/videos/*       → S3 origin (cache 30 days, range request support)
/*              → S3 origin (default behavior, cache 1 day)
```

#### Cache Policies
Managed or custom policies controlling what's included in the cache key:
- **Headers** — which request headers to forward and include in cache key
- **Query strings** — which query parameters to include
- **Cookies** — which cookies to forward
- AWS provides managed policies: `CachingOptimized`, `CachingDisabled`, `CachingOptimizedForUncompressedObjects`

#### Origin Request Policies
Controls what CloudFront includes in requests sent to the origin (without affecting the cache key):
- Forward headers, cookies, query strings to origin without caching on them
- AWS managed: `AllViewer`, `CORS-S3Origin`, `CORS-CustomOrigin`

#### Origin Shield
An optional, additional caching layer between RECs and your origin:
- Reduces origin load by centralizing cache misses through a single AWS region
- Improves cache hit ratio
- Costs extra (~$0.009–$0.0135/10K requests depending on region)
- Best for origins with high request rates or expensive compute

#### Origin Groups (Failover)
Provides **origin failover** for high availability:
- Configure a primary and secondary origin
- CloudFront automatically fails over on HTTP 500, 502, 503, 504, 403, or 404 responses
- Can be used with S3 Cross-Region Replication for disaster recovery

### Function Execution Points

```
Viewer Request  →  [CloudFront Functions / Lambda@Edge]
                         │
                    Edge Location
                         │
Origin Request  →  [Lambda@Edge]
                         │
                    Origin Server
                         │
Origin Response →  [Lambda@Edge]
                         │
                    Edge Location
                         │
Viewer Response →  [CloudFront Functions / Lambda@Edge]
                         │
                    User Browser
```

| Feature | CloudFront Functions | Lambda@Edge |
|---|---|---|
| Execution location | Edge PoPs only | Edge PoPs + RECs |
| Triggers | Viewer req/res only | All 4 triggers |
| Runtime | JavaScript (ES5.1) | Node.js, Python |
| Max execution time | 1ms | 5s (viewer) / 30s (origin) |
| Max memory | 2MB | 128MB–10GB |
| Max package size | 10KB | 1MB (viewer) / 50MB (origin) |
| Pricing | $0.10/1M invocations | $0.60/1M invocations |
| Use case | URL rewrites, header manipulation, auth | Complex logic, A/B testing, server-side rendering |

---

## Real World Example

### Scenario: Global E-Commerce Platform

**Company:** RetailCo — an online retailer with customers in North America, Europe, and Asia-Pacific.

**Infrastructure:**
- Product catalog (images, CSS, JS) stored in **S3 (us-east-1)**
- Dynamic APIs running on **ALB → ECS Fargate (us-east-1)**
- Authentication via **Cognito**

**Goal:** Serve global users with low latency, protect APIs, and reduce origin costs.

---

#### Step 1: Create the CloudFront Distribution

Configure a distribution with multiple origins:

```
Origin 1: S3 bucket (retailco-assets.s3.amazonaws.com)
Origin 2: ALB (api.retailco.com)
```

#### Step 2: Define Cache Behaviors

| Path Pattern | Origin | Cache Policy | Notes |
|---|---|---|---|
| `/api/*` | ALB | CachingDisabled | Dynamic API, no cache |
| `/images/*` | S3 | CachingOptimized (TTL: 7 days) | Product images |
| `/static/*` | S3 | CachingOptimized (TTL: 1 year) | Versioned CSS/JS |
| `/*` | S3 | CachingOptimized (TTL: 1 day) | HTML pages |

#### Step 3: Configure Origin Access Control (OAC)

Restrict S3 bucket access so only CloudFront can read it:

```json
// S3 Bucket Policy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontOAC",
    "Effect": "Allow",
    "Principal": {
      "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::retailco-assets/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789:distribution/EDFDVBD6EXAMPLE"
      }
    }
  }]
}
```

#### Step 4: Attach AWS WAF

Create a WAF WebACL with rules:
- AWS Managed Rules (Common Rule Set, Known Bad Inputs)
- Rate limiting: 1000 requests/5 minutes per IP
- Geo-blocking: Block requests from sanctioned countries
- SQL injection and XSS protection

#### Step 5: Add Lambda@Edge for Authentication

For `/api/*` requests, attach a Lambda@Edge function at **Viewer Request** to validate JWT tokens:

```javascript
// Lambda@Edge - Viewer Request
exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const headers = request.headers;
  
  const authHeader = headers['authorization']?.[0]?.value;
  if (!authHeader || !validateJWT(authHeader)) {
    return {
      status: '401',
      statusDescription: 'Unauthorized',
      body: JSON.stringify({ error: 'Invalid token' })
    };
  }
  return request; // Forward to origin
};
```

#### Step 6: Enable Logging and Monitoring

- Enable **CloudFront access logs** → S3 bucket
- Enable **Real-time logs** → Kinesis Data Streams → OpenSearch for dashboards
- Set CloudWatch alarms on `5xxErrorRate > 1%`

#### Step 7: Configure Custom Domain with SSL

```
Custom domain: www.retailco.com
SSL Certificate: ACM certificate in us-east-1
SSL Policy: TLSv1.2_2021 (modern security policy)
```

#### Result

| Metric | Before CloudFront | After CloudFront |
|---|---|---|
| Avg latency (Tokyo users) | 180ms | 12ms |
| Origin requests/day | 10M | 800K (92% cache hit) |
| Monthly egress cost | $2,400 | $380 |
| DDoS incidents | 3/month | 0 (WAF + Shield) |

---

## Advantages

1. **Global Low Latency** — 600+ edge locations ensure content is served from within milliseconds of any user worldwide

2. **High Cache Hit Ratios** — Two-tier caching (edge + regional edge cache) dramatically reduces origin load; typical deployments achieve 80–95% cache hit rates

3. **Deep AWS Integration** — Native integration with S3, ALB, API Gateway, Lambda, WAF, Shield, Route 53, ACM, and CloudWatch without additional configuration complexity

4. **Programmable Edge** — Lambda@Edge and CloudFront Functions allow custom logic at the edge without managing servers (URL rewrites, A/B testing, auth, personalization)

5. **Security Built-In** — AWS Shield Standard included at no cost, WAF integration, TLS termination, OAC for S3, geo-restriction, signed URLs/cookies

6. **HTTP/3 (