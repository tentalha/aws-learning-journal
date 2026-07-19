# SNS

## What is it?

**Amazon Simple Notification Service (SNS)** is a fully managed, serverless **pub/sub (publish-subscribe) messaging and mobile notification service** provided by AWS. It falls under the category of **Application Integration** services.

SNS enables decoupled, asynchronous communication between distributed systems, microservices, and serverless applications. It allows a single message published to a **topic** to be simultaneously delivered to multiple **subscribers** — a pattern known as **fan-out**.

SNS supports two primary messaging paradigms:
- **Topic-based pub/sub**: Publishers send messages to a named topic, and all subscribers to that topic receive the message.
- **Direct messaging (Mobile Push)**: Push notifications directly to mobile devices via platform-specific gateways (APNs, FCM, ADM, etc.).

**Key identifiers:**
- **Service type**: Messaging / Pub-Sub
- **Delivery model**: Push-based (SNS pushes to subscribers)
- **Protocol support**: HTTP/HTTPS, Email, Email-JSON, SMS, SQS, Lambda, Mobile Push, Kinesis Data Firehose
- **Topic types**: Standard and FIFO (First-In-First-Out)

---

## Why do we need it?

### The Problem It Solves

In traditional monolithic architectures, components communicate directly and synchronously. This creates **tight coupling** — if one component fails or is slow, it cascades failures across the system. As systems scale, direct point-to-point integrations become unmanageable (N×M connections for N producers and M consumers).

SNS solves this by introducing a **message broker** that decouples producers from consumers:

```
Without SNS:
Producer → Consumer A
Producer → Consumer B   (N×M tight coupling)
Producer → Consumer C

With SNS:
Producer → SNS Topic → Consumer A
                     → Consumer B  (1×N loose coupling)
                     → Consumer C
```

### Business Scenarios

| Scenario | How SNS Helps |
|---|---|
| **E-commerce order processing** | Order service publishes to SNS; inventory, billing, and shipping services subscribe independently |
| **Real-time alerting** | CloudWatch alarms publish to SNS → email/SMS/PagerDuty notifications |
| **Mobile push notifications** | Marketing platform sends promotions to millions of mobile devices |
| **System event broadcasting** | S3 events trigger SNS → fan-out to multiple SQS queues for parallel processing |
| **Microservices orchestration** | Service A completes work → publishes event → multiple downstream microservices react |
| **Fraud detection** | Transaction service publishes events → fraud detection, audit log, and analytics consume independently |

### When to Use SNS vs. Alternatives

| Criteria | SNS | SQS | EventBridge |
|---|---|---|---|
| Fan-out to multiple consumers | ✅ Best choice | ❌ Single consumer | ✅ Good choice |
| Message queuing/buffering | ❌ | ✅ Best choice | ❌ |
| Event filtering by rules | ⚠️ Basic | ❌ | ✅ Best choice |
| Mobile push notifications | ✅ Best choice | ❌ | ❌ |
| SMS/Email delivery | ✅ Best choice | ❌ | ❌ |

---

## Internal Working

### Message Lifecycle

```
1. Publisher calls SNS API (Publish)
        ↓
2. SNS validates message, applies topic policies
        ↓
3. Message stored redundantly across multiple AZs (briefly)
        ↓
4. SNS evaluates subscriptions + filter policies
        ↓
5. Message delivered in parallel to all matching subscribers
        ↓
6. Delivery retried on failure (with exponential backoff)
        ↓
7. Undeliverable messages sent to Dead Letter Queue (if configured)
```

### Core Internal Mechanisms

#### Message Storage
SNS is **not a persistent message store** (unlike SQS). Messages are stored transiently, replicated across multiple Availability Zones within a region for durability during the delivery window. Once delivered (or retry exhausted), messages are discarded.

#### Delivery Retry Logic
SNS uses a **tiered retry policy** for HTTP/HTTPS endpoints:

```
Phase 1 - Immediate: 3 retries, no delay
Phase 2 - Pre-backoff: 2 retries, 1 second delay
Phase 3 - Backoff: 10 retries, exponential backoff (min 1s, max 20s)
Phase 4 - Post-backoff: up to 100,000 retries, 20 second delay
```

Total retry duration: **up to 23 days** for HTTP endpoints.

For SQS and Lambda subscribers, SNS retries **3 times** before sending to DLQ.

#### Fan-out Mechanism
When a message is published, SNS:
1. Identifies all active subscriptions for the topic
2. Evaluates **subscription filter policies** for each subscriber
3. Dispatches messages **in parallel** to all matching subscribers
4. Each delivery is independent — one subscriber's failure does not affect others

#### Message Deduplication (FIFO Topics)
FIFO topics use a **5-minute deduplication window**. Messages with the same `MessageDeduplicationId` within this window are discarded. This is achieved via SHA-256 hashing of the message body (content-based) or an explicit deduplication ID.

#### Message Ordering (FIFO Topics)
FIFO topics maintain strict ordering within a **Message Group ID**. Messages with the same group ID are delivered in the exact order published. Different group IDs can be processed in parallel.

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                         SNS Architecture                         │
│                                                                   │
│  ┌──────────┐    Publish    ┌─────────────────────────────────┐  │
│  │Publisher │─────────────▶│           SNS Topic              │  │
│  │(App/AWS) │              │  ┌─────────────────────────────┐ │  │
│  └──────────┘              │  │  Topic Policy (Access Control│ │  │
│                            │  └─────────────────────────────┘ │  │
│                            │  ┌─────────────────────────────┐ │  │
│                            │  │  Subscriptions               │ │  │
│                            │  │  ┌──────────────────────┐   │ │  │
│                            │  │  │ Sub 1: SQS + Filter  │   │ │  │
│                            │  │  │ Sub 2: Lambda        │   │ │  │
│                            │  │  │ Sub 3: HTTP/HTTPS    │   │ │  │
│                            │  │  │ Sub 4: Email         │   │ │  │
│                            │  │  │ Sub 5: SMS           │   │ │  │
│                            │  │  │ Sub 6: Mobile Push   │   │ │  │
│                            │  │  │ Sub 7: Firehose      │   │ │  │
│                            │  │  └──────────────────────┘   │ │  │
│                            │  └─────────────────────────────┘ │  │
│                            └─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Standard Topic vs. FIFO Topic

| Feature | Standard Topic | FIFO Topic |
|---|---|---|
| **Throughput** | Nearly unlimited | 300 msg/s (3,000 with batching) |
| **Ordering** | Best-effort | Strict (per Message Group) |
| **Deduplication** | Not guaranteed | Yes (5-min window) |
| **Subscribers** | All protocols | SQS FIFO queues only |
| **Use case** | High throughput, tolerant of duplicates | Financial transactions, ordered events |

### Key Architectural Patterns

#### 1. Fan-Out Pattern
```
S3 Event ──▶ SNS Topic ──▶ SQS Queue A (Image Processing)
                       ──▶ SQS Queue B (Audit Logging)
                       ──▶ SQS Queue C (Analytics)
                       ──▶ Lambda (Real-time notification)
```

#### 2. SNS + SQS for Resilient Fan-Out
```
Publisher ──▶ SNS Topic ──▶ SQS Queue 1 ──▶ Consumer 1 (with retry)
                        ──▶ SQS Queue 2 ──▶ Consumer 2 (with retry)
```
This pattern provides both fan-out (SNS) and durability/buffering (SQS).

#### 3. Message Filtering Pattern
```
Order Service ──▶ SNS Topic
                      │
                      ├──▶ SQS (filter: orderType = "PREMIUM")
                      ├──▶ Lambda (filter: orderValue > 1000)
                      └──▶ Email (filter: orderStatus = "FAILED")
```

#### 4. Topic ARN Structure
```
arn:aws:sns:{region}:{account-id}:{topic-name}
arn:aws:sns:us-east-1:123456789012:MyOrderTopic
arn:aws:sns:us-east-1:123456789012:MyOrderTopic.fifo  (FIFO)
```

---

## Real World Example

### Scenario: E-Commerce Order Processing System

**Business Requirement**: When a customer places an order, the system must simultaneously:
1. Reserve inventory
2. Charge the payment
3. Send confirmation email
4. Update analytics dashboard
5. Notify the warehouse

**Architecture Walkthrough:**

```
Step 1: Customer places order via API Gateway
         ↓
Step 2: Order Service (Lambda) validates order
         ↓
Step 3: Order Service publishes to SNS topic "order-placed"
        Message: {
          "orderId": "ORD-12345",
          "customerId": "CUST-789",
          "items": [...],
          "total": 249.99,
          "orderType": "STANDARD"
        }
         ↓
Step 4: SNS fans out to subscribers:
  ├── SQS Queue → Inventory Service (reserves stock)
  │   Filter: {"orderType": ["STANDARD", "PREMIUM"]}
  │
  ├── SQS Queue → Payment Service (processes payment)
  │   Filter: (no filter - receives all)
  │
  ├── Lambda → Email Service (sends confirmation)
  │   Filter: (no filter - receives all)
  │
  ├── Kinesis Firehose → S3 → Analytics (stores for BI)
  │   Filter: (no filter - receives all)
  │
  └── SQS FIFO Queue → Warehouse System (ordered fulfillment)
      Filter: {"orderType": ["STANDARD", "PREMIUM"]}
```

**Implementation Details:**

```
Topic ARN: arn:aws:sns:us-east-1:123456789012:order-placed

Subscription 1 (Inventory):
  Protocol: SQS
  Endpoint: arn:aws:sqs:us-east-1:123456789012:inventory-queue
  Filter Policy: {"orderType": ["STANDARD", "PREMIUM", "EXPRESS"]}
  DLQ: arn:aws:sqs:us-east-1:123456789012:inventory-dlq

Subscription 2 (Payment):
  Protocol: SQS
  Endpoint: arn:aws:sqs:us-east-1:123456789012:payment-queue
  Redrive Policy: DLQ configured

Subscription 3 (Email):
  Protocol: Lambda
  Endpoint: arn:aws:lambda:us-east-1:123456789012:function:send-confirmation

Subscription 4 (Analytics):
  Protocol: Firehose
  Endpoint: arn:aws:firehose:us-east-1:123456789012:deliverystream/order-analytics
```

**Result**: All five systems receive the order event within milliseconds, process independently, and failures in one system don't block others.

---

## Advantages

1. **Fully Managed & Serverless**: No infrastructure to provision, patch, or scale. AWS handles all operational overhead.

2. **True Fan-Out**: Single publish operation delivers to unlimited subscribers simultaneously — impossible to replicate efficiently with direct API calls.

3. **Protocol Flexibility**: Supports 7+ delivery protocols in a single service (HTTP, SQS, Lambda, Email, SMS, Mobile Push, Firehose).

4. **High Throughput**: Standard topics support virtually unlimited message throughput (millions of messages per second).

5. **Message Filtering**: Subscription filter policies allow subscribers to receive only relevant messages, reducing unnecessary processing and cost.

6. **Mobile Push at Scale**: Native integration with APNs (Apple), FCM (Google), ADM (Amazon), Baidu, and WNS (Windows) — handles platform-specific formatting automatically.

7. **Cross-Account & Cross-Region**: Topics can be subscribed to by resources in different AWS accounts and regions.

8. **High Availability**: Automatically replicates across multiple AZs. No single point of failure.

9. **Flexible Message Size**: Supports messages up to **256 KB**. For larger payloads, use the SNS Extended Client Library (stores payload in S3).

10. **FIFO Guarantee**: FIFO topics provide exactly-once delivery and strict ordering when required.

11. **Pay-per-use**: No minimum fees or upfront commitments.

12. **Native AWS Integration**: Deep integration with CloudWatch, IAM, KMS, S3, Lambda, SQS, and more.

---

## Limitations

### Hard Limits (Default Quotas)

| Limit | Standard Topic | FIFO Topic |
|---|---|---|
| **Max message size** | 256 KB | 256 KB |
| **Topics per account per region** | 100,000 | 100,000 |
| **Subscriptions per topic** | 12,500,000 | 100 |
| **Publish throughput** | Unlimited (soft limit) | 300 msg/s (3,000 with batching) |
| **Message retention** | No persistence | No persistence |
| **Filter policy attributes** | 5 per subscription | 5 per subscription |
| **Filter policy size** | 256 KB | 256 KB |
| **SMS daily spend limit** | $1.00 (default, increasable) | N/A |

### Functional Limitations

1. **No Message Persistence**: SNS does not store messages. If a subscriber is unavailable and has no DLQ, messages are lost.

2. **No Message Replay**: Unlike Kinesis or EventBridge Archive, SNS cannot replay past messages to new subscribers.

3. **FIFO Subscriber Restriction**: FIFO topics only support **SQS FIFO queues** as subscribers — no Lambda, HTTP, or email.

4. **No Scheduled Delivery**: SNS delivers immediately; there is no built-in delay mechanism (use SQS delay queues instead).

5. **Email Subscription Confirmation**: Email subscriptions require manual confirmation — not suitable for automated pipelines.

6. **No Message Transformation**: SNS delivers messages as-is (with envelope). No built-in transformation (use Lambda or EventBridge for this).

7. **SMS Limitations**: SMS is not available in all regions; international SMS rates vary significantly; long messages are split.

8. **No Consumer Groups**: Unlike Kafka, there is no concept of consumer groups or offset management.

9. **Cross-Region Delivery**: SNS can deliver to cross-region SQS queues, but this is not supported for all protocols.

---

## Best Practices

### 1. Always Use SNS + SQS Fan-Out for Critical Workloads
Never subscribe Lambda or HTTP endpoints directly to SNS for critical workloads without SQS as a buffer. SQS provides durability, retry, and backpressure.

```
✅ Recommended:
Publisher → SNS → SQS → Lambda (durable, retryable)

⚠️ Use carefully:
Publisher → SNS → Lambda (no buffering, Lambda throttling loses messages)
```

### 2. Configure Dead Letter Queues (DLQ)
Always configure DLQs for SQS and Lambda subscriptions to capture undeliverable messages.

```json