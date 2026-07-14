# CloudFront — Interview Questions

---

## Easy

### Q1. What is Amazon CloudFront and what problem does it solve?

**Answer:**
Amazon CloudFront is a globally distributed Content Delivery Network (CDN) service provided by AWS. It solves the problem of latency and performance degradation when users are geographically distant from the origin server.

CloudFront works by caching content at **edge locations** (Points of Presence) around the world. When a user requests content, CloudFront serves it from the nearest edge location rather than routing the request all the way to the origin (e.g., an S3 bucket or EC2 instance). This reduces latency, decreases load on the origin, and improves the overall user experience.

Key benefits:
- **Low latency** — content served from the closest edge location
- **High throughput** — AWS backbone network is used for origin fetches
- **Security** — integrates with AWS Shield, WAF, and ACM
- **Cost reduction** — fewer requests hit the origin server

---

### Q2. What is the difference between an Origin and an Edge Location in CloudFront?

**Answer:**

| Concept | Description |
|---|---|
| **Origin** | The source of truth for your content. Can be an S3 bucket, an Application Load Balancer, an EC2 instance, an API Gateway, or any custom HTTP server. |
| **Edge Location** | A data center in CloudFront's global network where content is cached. There are 400+ edge locations worldwide. |
| **Regional Edge Cache** | An intermediate caching layer between edge locations and the origin. Larger cache that holds less popular content longer. |

When a user makes a request:
1. CloudFront checks the **edge location** cache (cache hit → serve immediately).
2. On a cache miss, CloudFront checks the **Regional Edge Cache**.
3. On another miss, CloudFront fetches from the **Origin**.

---

### Q3. What is a CloudFront Distribution and what are the two types?

**Answer:**
A **CloudFront Distribution** is the configuration entity that tells CloudFront where your origin is, how to cache content, and how to handle requests. You create a distribution to start using CloudFront.

There are two types:

1. **Web Distribution** — Used for HTTP/HTTPS content delivery. Supports websites, APIs, media streaming over HTTP, and static assets. This is the standard and most commonly used type.

2. **RTMP Distribution** — Used for streaming media files using Adobe's Real-Time Messaging Protocol (RTMP). **Note:** AWS deprecated RTMP distributions in December 2020, so this type is no longer available for new distributions.

Today, all new distributions are **Web Distributions**, which support both static and dynamic content, HTTP/2, HTTP/3, WebSocket, and more.

---

### Q4. What is a Cache Behavior in CloudFront?

**Answer:**
A **Cache Behavior** is a set of rules within a CloudFront distribution that defines how CloudFront handles requests for a specific URL path pattern.

Each cache behavior specifies:
- **Path pattern** — e.g., `/images/*`, `/api/*`, `*.jpg`
- **Origin** — which origin to forward the request to
- **Viewer Protocol Policy** — HTTP only, HTTPS only, or redirect HTTP to HTTPS
- **Allowed HTTP Methods** — GET/HEAD, GET/HEAD/OPTIONS, or all methods
- **Cache settings** — TTL values, cache key policies
- **Lambda@Edge or CloudFront Functions** — attached functions
- **Signed URLs/Cookies** — access restriction settings

A distribution has one **default cache behavior** (path pattern `*`) and can have multiple **additional cache behaviors** that are evaluated in order of precedence before the default.

**Example:** You might configure `/api/*` to bypass caching and forward all headers to an ALB, while `/static/*` caches aggressively in CloudFront.

---

### Q5. What is a Cache Hit Ratio and why does it matter?

**Answer:**
The **Cache Hit Ratio** is the percentage of requests that CloudFront serves directly from its cache (edge locations) without forwarding to the origin.

```
Cache Hit Ratio = (Cache Hits) / (Total Requests) × 100
```

**Why it matters:**
- **Performance** — Cache hits are served with much lower latency than origin fetches
- **Cost** — Fewer origin requests mean lower data transfer costs and reduced origin load
- **Scalability** — A high cache hit ratio means your origin can handle traffic spikes more easily

**Ways to improve cache hit ratio:**
- Increase TTL values
- Minimize the number of query strings, headers, and cookies forwarded to the origin (each unique combination creates a separate cache entry)
- Use **Cache Policies** to precisely control what's included in the cache key
- Use **Origin Shield** as an additional caching layer

A healthy cache hit ratio is typically **80%+** for static content workloads.

---

## Medium

### Q1. Explain the difference between CloudFront Cache Policies and Origin Request Policies. When would you use each?

**Answer:**
These two policies serve distinct purposes in controlling how CloudFront handles requests:

**Cache Policy** — Controls what information is included in the **cache key**. The cache key determines whether a request results in a cache hit or miss. It includes:
- Query strings
- HTTP headers
- Cookies
- Compression settings (Gzip, Brotli)

**Origin Request Policy** — Controls what information CloudFront includes when it **forwards a request to the origin** on a cache miss. This can include additional query strings, headers, and cookies that are *not* part of the cache key. This allows you to send information to the origin (like `User-Agent` or custom headers) without fragmenting your cache.

**Key distinction:**

| | Cache Policy | Origin Request Policy |
|---|---|---|
| **Purpose** | Defines cache key | Defines what's forwarded to origin |
| **Affects caching** | Yes — directly | No — only applies on cache miss |
| **Use case** | Maximize cache efficiency | Pass context to origin without hurting cache hit ratio |

**Example scenario:**
- You want to forward the `Authorization` header to your API origin (for authentication) but you don't want every unique token to create a separate cache entry.
- **Solution:** Don't include `Authorization` in the Cache Policy (so it doesn't fragment the cache). Include it in the Origin Request Policy so it's forwarded to the origin on cache misses.

**AWS Managed Policies:**
AWS provides managed cache policies (e.g., `CachingOptimized`, `CachingDisabled`) and managed origin request policies (e.g., `AllViewer`, `CORS-S3Origin`) that cover most common use cases.

---

### Q2. What is CloudFront Origin Shield and how does it work? When should you enable it?

**Answer:**
**Origin Shield** is an additional caching layer that sits between CloudFront's regional edge caches and your origin. It acts as a centralized cache to maximize cache hit rates and minimize the number of requests that reach your origin.

**How it works:**
```
User → Edge Location → Regional Edge Cache → Origin Shield → Origin
```
Without Origin Shield, multiple regional edge caches across the world may independently request the same uncached content from the origin, resulting in multiple origin fetches. With Origin Shield, all regional edge caches route through a single Origin Shield location, so only one request reaches the origin for a given piece of content.

**Benefits:**
- **Reduced origin load** — Significantly fewer requests reach the origin
- **Improved cache hit ratio** — Content cached at Origin Shield serves multiple regional caches
- **Better origin availability** — Origin is protected from traffic spikes
- **Reduced latency for origin fetches** — Origin Shield is placed in the AWS region closest to your origin

**When to enable Origin Shield:**
- Your origin has limited capacity or is expensive to scale
- You have a globally distributed audience (traffic from many different regions)
- Your content is not highly personalized (cacheable content)
- You're using a custom origin that's not in an AWS region with a regional edge cache
- You want to minimize data transfer costs from your origin

**When NOT to use it:**
- Highly dynamic, personalized content that can't be cached
- When added latency of an extra network hop is unacceptable
- Very low-traffic workloads where the cost doesn't justify the benefit

**Cost:** Origin Shield has an incremental cost per HTTP request that hits the Origin Shield layer.

---

### Q3. How does CloudFront handle HTTPS and what are the SSL/TLS configuration options available?

**Answer:**
CloudFront provides comprehensive HTTPS support with several configuration options:

**Viewer Protocol Policy (Client → CloudFront):**
- `HTTP and HTTPS` — Accepts both
- `Redirect HTTP to HTTPS` — Redirects HTTP requests to HTTPS (recommended)
- `HTTPS Only` — Rejects HTTP requests with a 403

**Origin Protocol Policy (CloudFront → Origin):**
- `HTTP Only` — CloudFront communicates with origin over HTTP
- `HTTPS Only` — CloudFront communicates with origin over HTTPS
- `Match Viewer` — Matches the protocol used by the viewer

**SSL/TLS Certificate Options:**

1. **Default CloudFront Certificate** — `*.cloudfront.net` domain. Free, no configuration needed. Limited to the default CloudFront domain.

2. **Custom SSL Certificate (ACM)** — Use AWS Certificate Manager to provision a certificate for your custom domain. Must be in **us-east-1** (N. Virginia) regardless of your distribution's origin region.

3. **Custom SSL Certificate (IAM)** — Upload third-party certificates to IAM. Less common, used for certificates not supported by ACM.

**SSL/TLS Protocol Versions:**
CloudFront supports TLS 1.0, 1.1, 1.2, and 1.3. Best practice is to use the `TLSv1.2_2021` security policy minimum, which only allows TLS 1.2 and 1.3.

**SNI vs Dedicated IP:**
- **SNI (Server Name Indication)** — Free. Modern browsers support SNI. CloudFront uses SNI to serve multiple SSL certificates from the same IP.
- **Dedicated IP** — $600/month per distribution. Required for legacy clients that don't support SNI (very rare today).

**Origin SSL/TLS:**
- CloudFront validates the origin's SSL certificate by default
- You can configure CloudFront to use specific TLS versions and cipher suites when connecting to the origin
- For S3 origins, HTTPS is handled automatically

---

### Q4. Explain how CloudFront integrates with AWS WAF. What types of threats can this combination protect against?

**Answer:**
AWS WAF (Web Application Firewall) integrates directly with CloudFront distributions, allowing you to inspect and filter HTTP/HTTPS requests at the edge before they reach your origin.

**How integration works:**
1. You create a **WAF Web ACL** (Access Control List) in the `us-east-1` region (required for CloudFront)
2. You associate the Web ACL with your CloudFront distribution
3. CloudFront evaluates every incoming request against the Web ACL rules before processing it
4. Requests that match blocking rules receive a 403 response from the edge — they never reach the origin

**Types of threats protected against:**

| Threat | WAF Mechanism |
|---|---|
| **SQL Injection** | AWS Managed Rule Group: `AWSManagedRulesSQLiRuleSet` |
| **XSS (Cross-Site Scripting)** | AWS Managed Rule Group: `AWSManagedRulesCommonRuleSet` |
| **DDoS (Layer 7)** | Rate-based rules, AWS Shield Advanced integration |
| **Bad bots / scrapers** | Bot Control managed rule group |
| **IP-based attacks** | IP Set rules (block/allow specific IPs or CIDR ranges) |
| **Geo-based restrictions** | Geo match rules (block traffic from specific countries) |
| **Known malicious IPs** | Amazon IP Reputation List managed rule group |
| **Account takeover** | ATP (Account Takeover Prevention) managed rule group |
| **Credential stuffing** | ATP with login page monitoring |

**Rule types:**
- **Managed Rule Groups** — Pre-built rules maintained by AWS or AWS Marketplace sellers
- **Custom Rules** — Rules you define based on IP, geo, URI, headers, body, query strings
- **Rate-based Rules** — Automatically block IPs that exceed a request threshold

**Key advantages of WAF at CloudFront vs. at origin:**
- Malicious traffic is blocked at the edge (400+ locations globally) before consuming origin resources
- Reduces origin load and protects against DDoS amplification
- Lower latency for legitimate users since WAF inspection happens at the nearest edge

---

### Q5. What are Lambda@Edge and CloudFront Functions? How do they differ and when would you use each?

**Answer:**
Both Lambda@Edge and CloudFront Functions allow you to run custom code at CloudFront edge locations, but they have significant differences in capabilities, performance, and use cases.

**CloudFront Functions:**
- Lightweight JavaScript functions running at **400+ edge locations**
- Sub-millisecond execution time
- Extremely high scale (millions of requests per second)
- Limited runtime environment (no network access, no file system)
- Maximum execution time: **1ms**
- Package size limit: **10 KB**
- Supported triggers: **Viewer Request** and **Viewer Response** only
- Cost: ~1/6th the cost of Lambda@Edge

**Lambda@Edge:**
- Full Node.js or Python Lambda functions running at **Regional Edge Caches** (~13 locations)
- Execution time up to **5 seconds** (viewer events) or **30 seconds** (origin events)
- Can make network calls, access file system, use environment variables
- Package size limit: **50 MB** (zipped)
- Supported triggers: Viewer Request, Viewer Response, Origin Request, Origin Response
- Must be deployed in **us-east-1**

**Comparison Table:**

| Feature | CloudFront Functions | Lambda@Edge |
|---|---|---|
| Runtime | JavaScript (ES5.1+) | Node.js, Python |
| Execution time | < 1ms | Up to 30s |
| Network access | No | Yes |
| Triggers | Viewer only | All 4 trigger points |
| Scale | 400+ PoPs | ~13 Regional Edge Caches |
| Cost | Lower | Higher |
| Use case | Simple transformations | Complex logic |

**Use CloudFront Functions for:**
- URL rewrites and redirects
- HTTP header manipulation (add security headers)
- Simple A/B testing
- Cache key normalization (e.g., normalize query string case)
- JWT validation (simple)

**Use Lambda@Edge for:**
- Fetching data from DynamoDB or other AWS services
- Complex authentication/authorization logic
- Server-side rendering at the edge
- Image resizing/transformation (using Sharp library)
- Origin selection based on request properties
- Content localization based on `Accept-Language` header

---

## Hard

### Q1. Deep dive into CloudFront's caching model. How does the cache key work, and what are the performance implications of different cache key configurations?

**Answer:**
The **cache key** is the unique identifier CloudFront uses to determine whether a cached response can serve an incoming request. Understanding the cache key is fundamental to optimizing CloudFront performance.

**Default Cache Key:**
By default, the cache key consists only of the **distribution domain name + URL path**. For example:
```
d1234abcd.cloudfront.net/images/logo.png
```

**Expanded Cache Key:**
You can expand the cache key to include:
- **Query strings** — `?color=red&size=large` creates a different cache entry than `?color=blue`
- **HTTP headers** — `Accept-Language: en-US` vs `Accept-Language: fr-FR`
- **Cookies** — `session_id=abc123`
- **Compression** — Whether to include `Accept-Encoding` (Gzip, Brotli)

**Cache Key Fragmentation Problem:**
Every unique combination of cache key components creates a separate cache entry. This is the most common cause of poor cache hit ratios.

**Example:**
If you include the `User-Agent` header in the cache key, and you have 1,000 different user agents, you now have 1,000 separate cache entries for the same content. Each entry must be individually populated, dramatically reducing your cache hit ratio.

**Mathematical impact:**
```
Effective Cache Entries = URLs × Query String Combinations × Header Values × Cookie Values
```

**Strategies for optimal cache key design:**

1. **Normalize before caching:**
   - Use CloudFront Functions to normalize query strings (sort alphabetically, lowercase)
   - Before: `?B=2&A=1` and `?A=1&B=2` create two cache entries for the same content
   - After normalization: both map to `?A=1&B=2` → single cache entry

2. **Use Cache Policies precisely:**
   - Only include query strings that *actually* affect the response
   - Use `AllowList` mode (whitelist specific parameters)