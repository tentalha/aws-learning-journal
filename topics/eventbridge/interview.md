# EventBridge — Interview Questions

---

## Easy

### 1. What is Amazon EventBridge and what problem does it solve?

**Answer:**
Amazon EventBridge is a fully managed, serverless event bus service that enables event-driven architectures by routing events between AWS services, SaaS applications, and custom applications. It solves the problem of tightly coupled integrations by acting as a central event router — producers publish events without needing to know who consumes them, and consumers subscribe to only the events they care about. This decouples microservices, reduces operational overhead, and enables scalable, loosely coupled architectures.

---

### 2. What are the three types of event buses in EventBridge?

**Answer:**
1. **Default Event Bus** — Automatically created in every AWS account and receives events from AWS services (e.g., EC2 state changes, S3 events via CloudTrail).
2. **Custom Event Bus** — Created by users to receive events from custom applications or internal microservices. You can create multiple custom buses per account.
3. **Partner Event Bus** — Receives events from third-party SaaS providers (e.g., Zendesk, Datadog, Shopify) that are integrated with the AWS Partner Event Source program.

---

### 3. What is an EventBridge Rule and what does it consist of?

**Answer:**
An EventBridge Rule is a configuration that defines:
- **Event Pattern** — A JSON filter that matches incoming events based on their attributes (source, detail-type, account, region, or any field within the event detail).
- **Schedule** (optional alternative to pattern) — A cron or rate expression that triggers the rule on a time basis.
- **Targets** — One or more AWS resources or services that receive the matched events (up to 5 targets per rule).

When an event matches the rule's pattern, EventBridge automatically routes a copy of the event to each configured target.

---

### 4. What is the maximum event size supported by EventBridge?

**Answer:**
EventBridge supports a maximum event size of **256 KB**. If your payload exceeds this limit, a common pattern is to store the large payload in Amazon S3 or DynamoDB and include only a reference (such as an S3 object key) in the EventBridge event. This is known as the "Claim Check" pattern.

---

### 5. What is the difference between EventBridge and Amazon SNS?

**Answer:**

| Feature | EventBridge | Amazon SNS |
|---|---|---|
| **Filtering** | Rich, content-based JSON filtering | Attribute-based filtering only |
| **Sources** | AWS services, SaaS, custom apps | Custom publishers only |
| **Schema Registry** | Yes | No |
| **Targets** | 20+ AWS service targets | Limited targets (Lambda, SQS, HTTP, etc.) |
| **Replay** | Yes (archive & replay) | No |
| **Use Case** | Event-driven architecture, complex routing | Simple pub/sub fan-out |

EventBridge is more powerful for complex routing logic; SNS is simpler and lower latency for basic fan-out scenarios.

---

## Medium

### 1. How does EventBridge content-based filtering work, and what operators are supported?

**Answer:**
EventBridge content-based filtering allows rules to match events based on the values of specific fields within the event JSON. The filter is expressed as a JSON pattern where you specify field paths and their expected values or conditions.

**Supported matching operators:**
- **Exact match** — `"source": ["myapp.orders"]`
- **Prefix match** — `"detail.url": [{"prefix": "https://"}]`
- **Suffix match** — `"detail.filename": [{"suffix": ".jpg"}]`
- **Anything-but** — `"detail.status": [{"anything-but": ["ERROR"]}]`
- **Numeric match** — `"detail.price": [{"numeric": [">", 100, "<=", 500]}]`
- **Exists** — `"detail.optionalField": [{"exists": true}]`
- **IP address (CIDR)** — `"detail.sourceIp": [{"cidr": "10.0.0.0/8"}]`
- **Null** — `"detail.value": [null]`

**Example pattern:**
```json
{
  "source": ["com.myapp.orders"],
  "detail-type": ["OrderPlaced"],
  "detail": {
    "status": ["PENDING"],
    "amount": [{"numeric": [">", 1000]}]
  }
}
```
Only events matching ALL specified conditions are routed to the target. Fields not mentioned in the pattern are ignored, making patterns additive (AND logic between fields, OR logic within a field's array).

---

### 2. What is the EventBridge Schema Registry, and why is it important?

**Answer:**
The EventBridge Schema Registry is a centralized repository that automatically discovers and stores schemas for events flowing through your event buses. It solves the problem of event contract management in event-driven architectures.

**Key capabilities:**
- **Auto-discovery** — When enabled on an event bus, EventBridge automatically infers and registers schemas from observed events using JSON Schema draft-04 format.
- **Code bindings** — Generates strongly-typed code bindings (Java, Python, TypeScript) that you can download and use in your Lambda functions or applications, eliminating manual deserialization.
- **Versioning** — Tracks schema versions over time, allowing you to detect breaking changes.
- **OpenAPI support** — Schemas are stored in OpenAPI 3.0 format for AWS service events.

**Why it matters:**
- Reduces integration errors by providing a shared contract between producers and consumers.
- Speeds up development with auto-generated code.
- Enables governance — teams can browse available events before building new integrations.
- Integrates with the AWS Toolkit for IDEs (VS Code, IntelliJ) for in-editor event discovery.

**Cost note:** Schema discovery has a cost per event analyzed; schemas themselves are stored for free up to a limit.

---

### 3. Explain EventBridge Pipes and how they differ from standard EventBridge Rules.

**Answer:**
**EventBridge Pipes** is a feature that creates point-to-point integrations between a single source and a single target with optional filtering, enrichment, and transformation — all in a single, managed resource.

**Architecture of a Pipe:**
```
Source → [Filter] → [Enrichment] → [Target]
```

**Sources supported:** SQS, Kinesis, DynamoDB Streams, Kafka (MSK/self-managed), RabbitMQ.
**Enrichment:** Lambda, Step Functions, API Gateway, API Destinations.
**Targets:** 14+ AWS services including EventBridge buses, SQS, SNS, Lambda, Step Functions, etc.

**Key differences from Rules:**

| Aspect | EventBridge Rules | EventBridge Pipes |
|---|---|---|
| **Topology** | Fan-out (1 event → many targets) | Point-to-point (1 source → 1 target) |
| **Source** | Event bus only | Streaming/queue sources |
| **Enrichment** | Not built-in | Built-in enrichment step |
| **Batching** | No | Yes (from streaming sources) |
| **Polling** | No | Yes (polls SQS, Kinesis, etc.) |
| **Use Case** | Event routing & distribution | ETL-style pipeline with transformation |

**When to use Pipes:** When you need to consume from a streaming source, enrich the data, and deliver to a target — without writing custom Lambda glue code.

---

### 4. How does EventBridge handle event delivery failures and retries?

**Answer:**
EventBridge provides a robust retry mechanism with configurable dead-letter queues (DLQs) for handling delivery failures.

**Retry behavior:**
- EventBridge retries failed deliveries for up to **24 hours** using exponential backoff with jitter.
- The retry policy is configurable per target:
  - **Maximum retry attempts:** 0–185
  - **Maximum event age:** 60 seconds to 24 hours
- Retries only occur for certain failure types (e.g., target throttling, service unavailable). Permanent failures (e.g., invalid Lambda function name, permission denied) are **not retried**.

**Dead Letter Queues (DLQ):**
- You can configure an SQS queue as a DLQ per target.
- Events that exhaust retries or exceed the maximum age are sent to the DLQ.
- The DLQ message includes the original event plus metadata about why delivery failed.

**Important distinctions:**
- DLQs are configured at the **rule target level**, not the event bus level.
- Lambda's own retry behavior (2 async retries) is separate from EventBridge's retry mechanism.
- For guaranteed delivery, combining EventBridge DLQ with Lambda DLQ provides two layers of protection.

**Monitoring:** Use CloudWatch metrics (`FailedInvocations`, `DeadLetterErrors`) to track delivery issues.

---

### 5. What is EventBridge Archive and Replay, and when would you use it?

**Answer:**
**Archive** is a feature that allows you to store all or a filtered subset of events passing through an event bus in an EventBridge-managed store. **Replay** allows you to re-deliver archived events to one or more targets at a later time.

**Archive configuration:**
- Can be applied to any event bus (default, custom, or partner).
- Supports event filtering using the same pattern syntax as rules.
- Configurable retention period (indefinite or 1–N days).
- Storage is charged per GB.

**Replay configuration:**
- Specify a time range (start and end time) from the archive.
- Choose a destination event bus.
- Events are replayed at the original event time (preserving `time` field) but also include a replay name attribute.
- Replay speed is not controllable — it replays as fast as possible.

**Use cases:**
1. **Disaster recovery** — Replay events after a downstream service outage to reprocess missed events.
2. **New consumer onboarding** — Replay historical events to bootstrap a new microservice with past state.
3. **Bug fixes** — After fixing a Lambda bug, replay the events that failed to process correctly.
4. **Testing** — Replay production events against a new version of your consumer.
5. **Audit/compliance** — Maintain a complete event history for regulatory requirements.

**Limitation:** Replay delivers events to an event bus, not directly to a target. Rules on the destination bus then route replayed events to consumers.

---

## Hard

### 1. Deep dive into EventBridge cross-account and cross-region event routing. What are the architectural considerations?

**Answer:**
EventBridge supports both cross-account and cross-region event delivery, enabling centralized event architectures across an AWS Organization.

**Cross-Account Event Routing:**

**Method 1: Resource-based policy on target event bus**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowAccountBToPublish",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::ACCOUNT-B:root"},
    "Action": "events:PutEvents",
    "Resource": "arn:aws:events:us-east-1:ACCOUNT-A:event-bus/central-bus"
  }]
}
```

**Method 2: Org-level policy**
```json
{
  "Principal": "*",
  "Condition": {
    "StringEquals": {
      "aws:PrincipalOrgID": "o-xxxxxxxxxx"
    }
  }
}
```

**Cross-Region Routing:**
- Add an EventBridge event bus in another region as a target of a rule.
- EventBridge handles the cross-region delivery internally.
- No additional resource policies needed for same-account cross-region.

**Architectural patterns:**

1. **Hub-and-Spoke (Centralized Bus)**
   - Each account/region publishes to a central event bus in a security/networking account.
   - Central bus has rules that fan out to consumers.
   - Enables centralized monitoring, auditing, and governance.

2. **Mesh Pattern**
   - Each account has its own bus and forwards relevant events to other account buses.
   - More complex but avoids single point of failure.

**Key considerations:**
- **Latency:** Cross-region adds ~10-50ms latency. Design consumers to be idempotent.
- **Cost:** Cross-region/cross-account PutEvents are charged at standard rates in both source and destination regions.
- **IAM:** The rule's IAM role in the source account must have `events:PutEvents` permission on the target bus.
- **Event source preservation:** When an event crosses accounts, the `account` field in the event metadata reflects the originating account, but the `source` field is set by the producer. Use `account` for routing decisions, not `source`.
- **Encryption:** Events are encrypted in transit (TLS). For at-rest encryption of archives, use customer-managed KMS keys.
- **Organizational SCPs:** Ensure Service Control Policies don't block `events:PutEvents` across accounts.
- **Circular routing:** Carefully design patterns to avoid routing loops in mesh architectures.

---

### 2. How does EventBridge Scheduler differ from EventBridge scheduled rules, and when should you use each?

**Answer:**
**EventBridge Scheduler** (launched 2022) is a purpose-built scheduling service that is fundamentally different from EventBridge scheduled rules despite the naming similarity.

**EventBridge Scheduled Rules:**
- Rules on an event bus with a schedule expression (rate or cron).
- Limited to **one target per rule** (though you can have up to 5 targets per rule via fan-out).
- **300 scheduled rules per event bus** (soft limit).
- Minimum resolution: **1 minute**.
- Targets: Must be EventBridge-compatible targets.
- No time zone support for cron expressions (UTC only).
- No one-time schedules.

**EventBridge Scheduler:**
- Standalone service, not tied to an event bus.
- **Millions of schedules** supported per account.
- Minimum resolution: **1 minute** (same), but supports **flexible time windows**.
- **Time zone aware** — specify schedules in any IANA time zone.
- **One-time schedules** — trigger once at a specific date/time.
- **270+ AWS service targets** directly (not just EventBridge targets).
- **Flexible time windows** — deliver within a window (e.g., within 15 minutes of the scheduled time) to smooth out throttling.
- **Schedule groups** — organize and manage schedules at scale.
- **Automatic deletion** — one-time schedules can auto-delete after invocation.
- **Templated targets** — pass custom input to targets.

**When to use each:**

| Use Case | Use |
|---|---|
| Trigger Lambda every 5 minutes | Either (Scheduler preferred for scale) |
| 10,000 unique user reminder emails | Scheduler (scale, one-time) |
| React to AWS service events on a schedule | Scheduled Rule |
| Time-zone-aware daily report | Scheduler |
| Millions of IoT device polling schedules | Scheduler |
| Simple cron job for a single Lambda | Scheduled Rule (simpler) |

**Pricing difference:** Scheduled Rules are free (part of EventBridge). Scheduler charges per schedule invocation after a free tier.

**Migration consideration:** AWS recommends Scheduler for new high-scale scheduling use cases. Scheduled rules remain valid for simple, low-volume use cases.

---

### 3. Explain the EventBridge event structure in detail and how the `detail` field interacts with AWS service events vs. custom events.

**Answer:**
Every EventBridge event follows a standardized JSON envelope structure defined by AWS. Understanding this structure is critical for writing accurate event patterns and processing events correctly.

**Full event structure:**
```json
{
  "version": "0",
  "id": "12345678-1234-1234-1234-123456789012",
  "source": "aws.ec2",
  "account": "123456789012",
  "time": "2024-01-15T14:30:00Z",
  "region": "us-east-1",
  "resources": [
    "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0"
  ],
  "detail-type": "EC2 Instance State-change Notification",
  "detail": {
    "instance-id": "i-1234567890abcdef0",
    "state": "running"
  }
}
```

**Field breakdown:**

| Field | Description | Mutable by User |
|---|---|---|
| `version` | Always "0" | No |
| `id` | UUID, unique per event | No (auto-generated) |
| `source` | Event