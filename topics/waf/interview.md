# WAF — Interview Questions

## Easy

---

### Q1. What is AWS WAF, and what is its primary purpose?

**Answer:**
AWS WAF (Web Application Firewall) is a managed security service that helps protect web applications and APIs from common web exploits and malicious traffic. It allows you to define rules that control which web requests are allowed or blocked based on conditions such as IP addresses, HTTP headers, HTTP body content, URI strings, SQL injection patterns, and cross-site scripting (XSS) patterns. AWS WAF can be deployed in front of Amazon CloudFront, Application Load Balancer (ALB), Amazon API Gateway, and AWS AppSync.

---

### Q2. What are the main components of AWS WAF?

**Answer:**
The main components of AWS WAF are:

- **Web ACL (Access Control List):** The top-level container that holds rules and rule groups. It defines the default action (Allow or Block) for requests that don't match any rule.
- **Rules:** Individual conditions that inspect web requests and define an action (Allow, Block, Count, CAPTCHA, or Challenge).
- **Rule Groups:** Reusable collections of rules that can be added to multiple Web ACLs. Can be AWS Managed, AWS Marketplace, or custom.
- **IP Sets:** Collections of IP addresses or CIDR ranges used in rules.
- **Regex Pattern Sets:** Collections of regular expressions used to match against request components.
- **Statements:** The logical conditions within a rule (e.g., AND, OR, NOT combinators).

---

### Q3. What AWS services can AWS WAF be associated with?

**Answer:**
AWS WAF can be associated with the following services:

- **Amazon CloudFront** (global, edge-level protection)
- **Application Load Balancer (ALB)** (regional)
- **Amazon API Gateway REST APIs** (regional)
- **AWS AppSync GraphQL APIs** (regional)
- **Amazon Cognito User Pools** (regional)
- **AWS App Runner** (regional)
- **AWS Verified Access** (regional)

Each association point allows WAF to inspect and filter traffic at different layers of the application stack.

---

### Q4. What is a Web ACL default action in AWS WAF?

**Answer:**
The **default action** in a Web ACL is the action taken on a web request when it does not match any of the rules defined in the Web ACL. There are two possible default actions:

- **Allow:** Requests that don't match any rule are forwarded to the protected resource (most common for production traffic).
- **Block:** Requests that don't match any rule are blocked with an HTTP 403 response (useful for allowlist-based security models).

The default action acts as a final catch-all and is critical to the overall security posture of the WAF configuration.

---

### Q5. What is the difference between AWS Managed Rules and custom rules in AWS WAF?

**Answer:**
- **AWS Managed Rules:** Pre-configured rule groups created and maintained by AWS security experts. They address common threats like OWASP Top 10, known bad IPs, SQL injection, XSS, and more. Examples include `AWSManagedRulesCommonRuleSet`, `AWSManagedRulesSQLiRuleSet`, and `AWSManagedRulesKnownBadInputsRuleSet`. These require no configuration effort and are updated automatically by AWS.

- **Custom Rules:** Rules that you define yourself to match specific patterns relevant to your application. You can inspect any part of the HTTP request (headers, body, URI, query string, etc.) using exact string matches, regex patterns, IP sets, size constraints, and more. Custom rules provide granular control tailored to your specific application's needs.

Both can be combined within the same Web ACL, with priorities determining evaluation order.

---

## Medium

---

### Q1. Explain how rule priority works in AWS WAF and why it matters.

**Answer:**
In AWS WAF, every rule and rule group within a Web ACL is assigned a **numeric priority**. Rules are evaluated in **ascending order of priority** (lowest number = highest priority, evaluated first). When a request matches a rule, the action defined in that rule is applied immediately and evaluation stops — subsequent rules are not evaluated (this is called **first-match termination**).

**Why it matters:**
- If you have a rule that blocks all requests from a specific country but also have an allowlist rule for trusted IPs in that country, the order determines the outcome. If the block rule has a lower priority number (evaluated first), even trusted IPs will be blocked.
- Misconfigurations in priority can lead to unintended blocks or bypassed security controls.
- Count rules (used for monitoring) should generally be placed before action rules to capture accurate metrics without interfering with traffic flow.

**Best practice:**
1. Place allowlist/trusted IP rules at the highest priority (lowest number).
2. Place rate-based rules and bot control rules next.
3. Place managed rule groups after custom rules.
4. Place broad block rules last.

---

### Q2. What is a Rate-Based Rule in AWS WAF, and how does it work?

**Answer:**
A **Rate-Based Rule** is a type of WAF rule that automatically blocks requests from IP addresses (or other aggregation keys) that exceed a defined request threshold within a **5-minute rolling window**.

**How it works:**
1. You define a **rate limit** (minimum 100 requests per 5 minutes).
2. You define an **aggregation key** — by default, this is the originating IP address, but you can also aggregate by:
   - HTTP header value
   - Query string value
   - Cookie value
   - URI path
   - HTTP method
   - A combination of the above (custom aggregation keys)
3. AWS WAF tracks request counts per aggregation key. When the threshold is exceeded, the rule action (Block, CAPTCHA, Challenge, or Count) is applied.
4. Once the request rate drops below the threshold, the block is automatically lifted.

**Use cases:**
- DDoS mitigation
- Brute-force login protection
- API abuse prevention
- Scraping prevention

**Important note:** Rate-based rules are evaluated in real-time but have a propagation delay of approximately 30 seconds before a newly exceeded threshold takes effect.

---

### Q3. What is the difference between the Count, Block, Allow, CAPTCHA, and Challenge actions in AWS WAF?

**Answer:**

| Action | Description |
|--------|-------------|
| **Allow** | Forwards the request to the protected resource. Can optionally insert custom headers. |
| **Block** | Drops the request and returns an HTTP 403 (or custom response code/body). |
| **Count** | Allows the request to continue being evaluated by subsequent rules, but increments a counter in CloudWatch metrics and sampled requests. Does NOT terminate evaluation. |
| **CAPTCHA** | Challenges the client with a CAPTCHA puzzle. If the client fails or doesn't respond, the request is blocked. Requires the AWS WAF CAPTCHA JavaScript integration on the client. |
| **Challenge** | Sends a silent browser challenge (JavaScript-based) to verify the client is a legitimate browser. Transparent to users. Used for bot detection. |

**Key distinctions:**
- **Count** is the only non-terminating action — it lets subsequent rules continue evaluation.
- **CAPTCHA** and **Challenge** are interactive verification mechanisms and are billed separately per challenge token.
- **Allow** at the rule level overrides the default Block action but does not override rules with higher priority that block the request.

---

### Q4. How does AWS WAF integrate with AWS Shield Advanced, and what additional protections does this provide?

**Answer:**
AWS WAF and AWS Shield Advanced are complementary services that work together to provide comprehensive DDoS protection:

**Integration points:**
- When you subscribe to Shield Advanced, WAF is included at no additional cost for resources protected by Shield Advanced (CloudFront distributions, ALBs, etc.).
- Shield Advanced can **automatically create WAF rules** in response to detected DDoS attacks through the **Shield Response Team (SRT)** and **automatic application layer DDoS mitigation**.

**Additional protections with Shield Advanced + WAF:**
1. **Automatic Application Layer DDoS Mitigation:** Shield Advanced can automatically deploy WAF rules to mitigate Layer 7 DDoS attacks without manual intervention. You must enable this feature and associate a WAF Web ACL.
2. **SRT Access:** The AWS Shield Response Team can access your WAF rules and logs to create custom mitigations during active attacks.
3. **DDoS Cost Protection:** AWS provides service credits for scaling costs incurred due to DDoS attacks on protected resources.
4. **Enhanced Visibility:** Shield Advanced provides detailed attack diagnostics and real-time metrics through the AWS Shield console.
5. **Proactive Engagement:** SRT can proactively contact you when attacks are detected.

**Best practice:** Always associate a WAF Web ACL with Shield Advanced-protected resources to enable automatic Layer 7 mitigation.

---

### Q5. Explain WAF logging and how you would use it for security analysis.

**Answer:**
AWS WAF provides detailed logging of all web requests (sampled or full) that are evaluated by a Web ACL.

**Enabling WAF Logging:**
WAF logs can be sent to three destinations:
1. **Amazon CloudWatch Logs** — For real-time monitoring and alerting.
2. **Amazon S3** — For long-term storage, compliance, and batch analysis.
3. **Amazon Kinesis Data Firehose** — For streaming to S3, Redshift, OpenSearch, or third-party SIEMs.

**Log contents include:**
- Timestamp
- Forwarded IP and source IP
- HTTP method, URI, headers, body (if enabled)
- Matched rule name and rule group
- Action taken (ALLOW, BLOCK, COUNT, CAPTCHA, CHALLENGE)
- Web ACL name and ARN
- Country of origin

**Security analysis use cases:**
1. **Threat hunting:** Query logs in Athena to identify patterns like repeated failed login attempts, SQL injection attempts, or unusual URI patterns.
2. **Rule tuning:** Identify false positives by reviewing COUNT rule matches before converting them to BLOCK.
3. **Incident response:** Reconstruct attack timelines and identify attack vectors.
4. **Compliance reporting:** Demonstrate that security controls are in place and functioning.
5. **Anomaly detection:** Use CloudWatch Metrics Insights or Amazon OpenSearch to build dashboards and alerts.

**Best practice:** Enable full logging (not just sampled requests) for security-sensitive applications, and use log field redaction to exclude sensitive data like Authorization headers from logs.

---

## Hard

---

### Q1. Describe the WAF Web ACL capacity unit (WCU) system and how you would architect a complex rule set within WCU constraints.

**Answer:**
**WCU (Web ACL Capacity Units)** is AWS WAF's resource accounting system that measures the processing cost of rules. Each Web ACL has a maximum capacity of **5,000 WCUs** (soft limit, can be increased). Different rule types consume different WCU amounts:

**WCU costs by rule type:**

| Rule Type | WCU Cost |
|-----------|----------|
| IP Set match (IPv4) | 1 WCU |
| IP Set match (IPv6) | 1 WCU |
| Geo match | 1 WCU |
| Size constraint | 2 WCUs |
| String match (exact) | 2 WCUs |
| String match (contains) | 10 WCUs |
| Regex match | 3–25 WCUs (per pattern) |
| SQL injection match | 20 WCUs |
| XSS match | 40 WCUs |
| Rate-based rule | 2 WCUs + statement cost |
| AWS Managed Rule Groups | Varies (e.g., CommonRuleSet = 700 WCUs) |

**Architectural strategies for WCU optimization:**

1. **Scope-down statements:** Apply expensive rules (SQL injection, XSS) only to relevant request components. Use a scope-down statement to first check if the URI matches `/api/` before applying the SQLi check, reducing unnecessary processing.

2. **Rule ordering:** Place cheap IP set and geo-match rules first to short-circuit evaluation for large volumes of blocked traffic before reaching expensive regex/SQLi rules.

3. **Consolidate regex patterns:** Use a single Regex Pattern Set with multiple patterns instead of multiple individual regex rules. Each Regex Pattern Set costs a base of 25 WCUs regardless of the number of patterns (up to the set limit).

4. **Use managed rule groups selectively:** Instead of deploying all managed rule groups, enable only the ones relevant to your application stack. Enable individual rules within groups in Count mode to identify which rules fire before enabling full Block mode.

5. **Custom rule groups:** Create reusable custom rule groups to share WCU-efficient rule sets across multiple Web ACLs.

6. **Body inspection limits:** WAF inspects only the first 8 KB of the request body by default (configurable up to 64 KB with additional cost). Design rules to match patterns within this limit.

**Monitoring WCU usage:** Use the `aws wafv2 describe-web-acl` API or the console's capacity indicator to monitor current WCU consumption and plan for growth.

---

### Q2. How would you implement a sophisticated bot mitigation strategy using AWS WAF Bot Control, and what are the limitations you need to account for?

**Answer:**
**AWS WAF Bot Control** is a managed rule group (`AWSManagedRulesBotControlRuleSet`) that provides bot detection and mitigation capabilities. It operates in two modes:

**Bot Control Modes:**
1. **Common mode:** Detects and manages common bots using traditional detection methods (signature-based, IP reputation). Lower cost.
2. **Targeted mode:** Uses advanced detection including browser fingerprinting, behavioral analysis, machine learning, and challenge-response mechanisms. Higher cost and accuracy.

**Sophisticated Bot Mitigation Architecture:**

**Layer 1 — IP Reputation (cheapest, first filter):**
```
Rule: AWSManagedRulesAmazonIpReputationList
Priority: 10
Action: Block
```

**Layer 2 — Bot Control Common Mode:**
```
Rule: AWSManagedRulesBotControlRuleSet (Common)
Priority: 20
Override specific rules to Count for tuning
```

**Layer 3 — Bot Control Targeted Mode (for sensitive endpoints):**
```
Rule: AWSManagedRulesBotControlRuleSet (Targeted)
Priority: 30
Scope-down: URI path matches /login, /checkout, /api/auth
```

**Layer 4 — Custom Rate-Based Rules:**
```
Rule: Rate limit by IP + URI path combination
Priority: 40
Threshold: 100 requests per 5 minutes per IP per URI
Action: CAPTCHA
```

**Layer 5 — Custom Bot Signatures:**
```
Rule: Block known bad User-Agent strings via regex
Priority: 50
```

**Handling legitimate bots:**
- Create an IP Set for verified bot IPs (Googlebot, Bingbot) verified via reverse DNS.
- Place an Allow rule for verified bots at priority 5 (before Bot Control rules).
- Use the `aws:UserAgent` header combined with IP verification.

**Limitations to account for:**

1. **CAPTCHA/Challenge requires JavaScript:** Headless API clients cannot complete CAPTCHA challenges. Apply targeted mode only to browser-facing endpoints, not pure API endpoints.

2. **Mobile app traffic:** Native mobile apps don't run in browsers, so fingerprinting doesn't work. Use mobile SDK token integration or separate the mobile API path from web paths.

3. **CDN/Proxy IP addresses:** If your traffic passes through a CDN before WAF, the source IP may be a CDN IP. Configure WAF to use the `X-Forwarded-For` header for IP-based rules.

4. **False positives with automation tools:** Legitimate monitoring tools, CI/CD pipelines, and health checks may be flagged. Add their IPs to an allowlist IP set.

5. **Cost:** Bot Control Targeted mode has per-request pricing in addition to standard WAF pricing. Scope it to high-value endpoints only.

6. **Token validity:** CAPTCHA/Challenge tokens have a configurable immunity time (default 300 seconds). Tune this based on session length requirements.

7. **Latency:** Challenge/CAPTCHA responses add latency. Monitor application performance metrics after enabling.

---

### Q3. Explain the WAF request inspection body size limits, the implications for your security posture, and how you would mitigate the risks.

**Answer:**
**Default Body Inspection Limits:**

AWS WAF inspects the request body up to a configurable size limit:
- **Default:** 8 KB (8,192 bytes)
- **Configurable options:** 8 KB, 16 KB, 32 KB, 64 KB
- **Content beyond the limit:** The oversize body content is **not inspected** by WAF rules.

**Oversize handling options (per Web ACL or per rule):**
1. **CONTINUE:** WAF evaluates the rule against the inspected portion only. Content beyond the limit is ignored.
2. **MATCH:** WAF treats the request as matching the rule if