# SQS

## What is it?

**Amazon Simple Queue Service (SQS)** is a fully managed, distributed message queuing service provided by AWS under the **Application Integration** category. It enables decoupled communication between software components by allowing producers to send messages and consumers to retrieve and process them asynchronously.

SQS offers two types of queues:

| Queue Type | Description |
|---|---|
| **Standard Queue** | At-least-once delivery, best-effort ordering, nearly unlimited throughput |
| **FIFO Queue** | Exactly-once processing, strict message ordering, up to 3,000 messages/second with batching |

SQS is a **pull-based** service — consumers poll the queue to receive messages rather than having messages pushed to them. It is one of the oldest AWS services (launched in 2004) and is foundational to building resilient, scalable, and decoupled cloud architectures.

---

## Why do we need it?

### The Problem It Solves

In tightly coupled architectures, components communicate synchronously — if one component fails or slows down, it cascades failure through the entire system. SQS introduces **asynchronous decoupling**, allowing producers and consumers to operate independently.

### Core Problems Addressed

- **Tight coupling**: Services depend directly on each other's availability
- **Load spikes**: Sudden bursts of traffic overwhelm downstream services
- **Rate mismatches**: Producers generate data faster than consumers can process it
- **Fault tolerance**: A failed consumer causes message loss in direct integrations
- **Retry complexity**: Implementing retry logic manually is error-prone

### Real Business Scenarios

1. **E-commerce Order Processing**: A customer places an order → order service pushes a message to SQS → inventory, payment, and shipping services consume independently without blocking each other.

2. **Image/Video Processing Pipeline**: A user uploads a video → S3 triggers an SQS message → multiple EC2 workers poll the queue and process transcoding jobs at their own pace.

3. **IoT Data Ingestion**: Thousands of IoT sensors send telemetry data → SQS buffers the data → analytics services process at controlled rates to avoid database overload.

4. **Email/Notification Delivery**: A marketing campaign triggers millions of email tasks → SQS queues them → SES workers consume and send at a rate within SES limits.

5. **Microservices Communication**: Order service sends a "payment-received" event to SQS → fulfillment, analytics, and loyalty services each consume independently.

---

## Internal Working

### Message Lifecycle

```
Producer → Send Message → SQS Queue (Distributed Storage) → Consumer Polls → 
Message Becomes Invisible (Visibility Timeout) → Consumer Processes → 
Delete Message OR Visibility Timeout Expires (Message Reappears)
```

### Distributed Storage Layer

SQS stores messages across **multiple Availability Zones** using a highly redundant distributed system. When a message is sent, it is replicated to multiple servers before the send API returns success. This ensures durability even if individual servers fail.

### Message Visibility Timeout

When a consumer retrieves a message, SQS makes it **invisible** to other consumers for a configurable **Visibility Timeout** (default: 30 seconds, max: 12 hours). If the consumer successfully processes and deletes the message within this window, it is removed. If not, it reappears for another consumer to process.

```
[Message Sent] → [Available in Queue] → [Consumer Receives] → [Invisible Period]
                                                                      ↓
                                              [Deleted] ←── [Processing Complete]
                                              [Reappears] ←── [Timeout Expires / Crash]
```

### Long Polling vs Short Polling

| Aspect | Short Polling | Long Polling |
|---|---|---|
| Behavior | Returns immediately (even if empty) | Waits up to 20 seconds for messages |
| Empty responses | Frequent | Rare |
| Cost | Higher (more API calls) | Lower |
| Latency | Slightly lower | Slightly higher |
| Recommended | No | Yes |

Long polling is configured via `WaitTimeSeconds` (1–20 seconds) on `ReceiveMessage`.

### FIFO Queue Internals

FIFO queues use a **Message Group ID** to partition messages into ordered groups. Within a group, messages are processed in strict FIFO order. Different message groups can be processed in parallel. A **Message Deduplication ID** (or content-based deduplication) prevents duplicate messages within a 5-minute deduplication window.

### Dead Letter Queue (DLQ)

After a configurable number of receive attempts (`maxReceiveCount`), unprocessable messages are moved to a **Dead Letter Queue** — a separate SQS queue for failed messages. This prevents poison pill messages from blocking the queue indefinitely.

---

## Architecture

### Core Components

```
┌─────────────┐     SendMessage     ┌─────────────────────────────────┐
│  Producer   │ ──────────────────► │         SQS Queue               │
│ (App/Lambda)│                     │  ┌─────┐ ┌─────┐ ┌─────┐       │
└─────────────┘                     │  │ Msg │ │ Msg │ │ Msg │  ...  │
                                    │  └─────┘ └─────┘ └─────┘       │
┌─────────────┐    ReceiveMessage   │                                  │
│  Consumer   │ ◄────────────────── │  Visibility Timeout Applied      │
│ (EC2/Lambda)│                     │                                  │
│             │ ──────────────────► │  DeleteMessage on Success        │
└─────────────┘                     └─────────────────────────────────┘
                                              │ maxReceiveCount exceeded
                                              ▼
                                    ┌─────────────────┐
                                    │  Dead Letter     │
                                    │  Queue (DLQ)     │
                                    └─────────────────┘
```

### Common Architectural Patterns

#### 1. Fan-Out Pattern (SQS + SNS)
```
                    ┌──► SQS Queue A ──► Consumer A (Email Service)
Producer ──► SNS ───┤
                    └──► SQS Queue B ──► Consumer B (Analytics Service)
```

#### 2. Worker Fleet Pattern
```
                    ┌──► EC2 Worker 1
SQS Queue ──────────┼──► EC2 Worker 2  (Auto Scaling Group)
                    └──► EC2 Worker 3
```

#### 3. Lambda Event Source Mapping
```
SQS Queue ──► Lambda Trigger ──► Lambda Function (Batch Processing)
```
Lambda automatically polls SQS, scales concurrency, and deletes messages on success.

#### 4. Priority Queue Pattern
```
High Priority Queue ──► Dedicated Consumer (Always Running)
Low Priority Queue  ──► Shared Consumer (Scales Down When Idle)
```

#### 5. Saga Pattern (Microservices)
```
Order Service → SQS → Payment Service → SQS → Inventory Service → SQS → Shipping
     ↑                      │                        │
     └──── Compensating Transactions (Rollback) ─────┘
```

---

## Real World Example

### Scenario: E-Commerce Order Processing System

**Business Context**: An online retailer receives thousands of orders during flash sales. The order service must not block while payment, inventory, and shipping are processed.

#### Step-by-Step Walkthrough

**Step 1: Infrastructure Setup**
```
- Standard SQS Queue: orders-queue
- FIFO SQS Queue: payments-queue.fifo (for idempotency)
- DLQ: orders-dlq (maxReceiveCount: 3)
- SNS Topic: order-events
```

**Step 2: Customer Places Order**
```
Customer → API Gateway → Order Service (Lambda) → Validates order → 
Stores in DynamoDB → Sends message to SNS topic "order-events"
```

**Step 3: Fan-Out via SNS**
```
SNS "order-events" fans out to:
  → orders-queue (for inventory service)
  → payments-queue.fifo (for payment service)
  → analytics-queue (for reporting)
```

**Step 4: Payment Processing**
```
Payment Lambda polls payments-queue.fifo
Message Body: {
  "orderId": "ORD-12345",
  "amount": 299.99,
  "customerId": "CUST-789",
  "messageGroupId": "CUST-789"  // Per-customer ordering
}
→ Calls Stripe API
→ On success: Deletes message, publishes "payment-confirmed" event
→ On failure: Lets visibility timeout expire → retried up to 3 times → DLQ
```

**Step 5: Inventory Deduction**
```
Inventory EC2 workers (Auto Scaling Group) poll orders-queue
→ Deduct stock in RDS
→ If stock insufficient: Send to DLQ, trigger alert
→ If successful: Delete message, publish "inventory-reserved" event
```

**Step 6: DLQ Monitoring**
```
CloudWatch Alarm: ApproximateNumberOfMessagesVisible in orders-dlq > 0
→ SNS Alert → PagerDuty → On-call engineer investigates
→ Engineer replays DLQ messages after fixing the bug
```

**Step 7: Scaling During Flash Sale**
```
orders-queue ApproximateNumberOfMessagesVisible spikes to 50,000
→ CloudWatch Alarm triggers
→ Auto Scaling Group scales EC2 workers from 2 → 20 instances
→ Queue drains within minutes
→ ASG scales back down after queue depth normalizes
```

---

## Advantages

1. **Fully Managed**: No infrastructure provisioning, patching, or scaling of queue servers.

2. **Infinite Scalability**: Standard queues support virtually unlimited throughput with no pre-provisioning.

3. **High Durability**: Messages stored redundantly across 3 AZs — 99.999999999% (11 nines) durability.

4. **High Availability**: 99.9% SLA for Standard queues; messages survive AZ failures.

5. **Decoupling**: Producers and consumers are completely independent — different languages, runtimes, and scaling strategies.

6. **Cost-Effective**: Pay only for API calls made; no idle costs for the queue itself.

7. **Dead Letter Queue Support**: Built-in mechanism for handling poison pill messages without custom retry logic.

8. **Server-Side Encryption**: Integrated with AWS KMS for encryption at rest.

9. **Message Retention**: Messages can be retained for up to 14 days, providing a buffer for consumer outages.

10. **Lambda Integration**: Native event source mapping with automatic scaling, batching, and error handling.

11. **Exactly-Once Processing (FIFO)**: Eliminates duplicate processing in financial or order-critical workflows.

12. **Visibility Timeout Extension**: Consumers can extend timeout dynamically via `ChangeMessageVisibility` for long-running jobs.

---

## Limitations

### Hard Limits

| Limit | Value |
|---|---|
| Maximum message size | 256 KB |
| Message retention period | 1 minute – 14 days (default: 4 days) |
| Maximum visibility timeout | 12 hours |
| Maximum long poll wait time | 20 seconds |
| Maximum batch size (send/receive/delete) | 10 messages |
| FIFO queue throughput (without batching) | 300 TPS |
| FIFO queue throughput (with batching) | 3,000 TPS |
| Standard queue throughput | Nearly unlimited |
| Maximum inflight messages (Standard) | 120,000 |
| Maximum inflight messages (FIFO) | 20,000 |
| Maximum delay seconds | 15 minutes |
| Message deduplication window (FIFO) | 5 minutes |
| Queue name length | 80 characters |
| FIFO queue name suffix | Must end in `.fifo` |

### Functional Limitations

- **No message priority**: Standard queues don't support native priority — requires multiple queues.
- **No message filtering on SQS directly**: Filtering is done at SNS level or by consumers.
- **No push delivery**: Pull-based only — consumers must actively poll.
- **No message ordering in Standard queues**: Best-effort ordering only.
- **Large messages require S3**: Messages > 256 KB must use the Extended Client Library with S3.
- **No built-in message routing**: Unlike RabbitMQ, SQS has no exchange/routing key concept.
- **FIFO throughput cap**: 3,000 TPS is a hard limit even with high-throughput mode (300 msg/s without batching, up to 70,000 msg/s with high-throughput FIFO enabled in supported regions).
- **DLQ must be same type**: Standard DLQ for Standard queues; FIFO DLQ for FIFO queues.

---

## Best Practices

### 1. Always Use Long Polling
```bash
# Set WaitTimeSeconds to 20 for maximum efficiency
aws sqs receive-message --queue-url <url> --wait-time-seconds 20
```
Reduces empty responses, lowers cost, and decreases latency.

### 2. Set Appropriate Visibility Timeout
- Set visibility timeout to **at least 6x the average processing time**
- Use `ChangeMessageVisibility` to extend for long-running tasks
- Too short → duplicate processing; Too long → slow failure recovery

### 3. Implement Dead Letter Queues
- Always configure a DLQ with `maxReceiveCount` of 3–5
- Monitor DLQ depth with CloudWatch alarms
- Set DLQ retention period longer than source queue (e.g., 14 days)

### 4. Use Batch Operations
```javascript
// Send up to 10 messages in one API call
const command = new SendMessageBatchCommand({ Entries: messages });
```
Reduces API calls by 10x, lowering costs significantly.

### 5. Implement Idempotent Consumers
- Consumers should handle duplicate messages gracefully (especially Standard queues)
- Use DynamoDB conditional writes or database unique constraints to prevent double-processing

### 6. Use Message Attributes for Metadata
- Store routing/filtering metadata in message attributes rather than parsing message body
- Enables efficient filtering without deserializing the payload

### 7. Separate Queues by Priority
- High-priority tasks → dedicated queue with more consumers
- Low-priority tasks → separate queue with fewer consumers
- Prevents low-priority work from starving high-priority tasks

### 8. Enable Server-Side Encryption
- Use SSE-KMS for sensitive data
- Use SSE-SQS (AWS-managed key) for lower-cost encryption

### 9. Use FIFO Queues Only When Necessary
- FIFO has throughput limits — use Standard if ordering isn't critical
- Use `MessageGroupId` strategically to maximize parallelism

### 10. Align with Well-Architected Framework
- **Reliability**: Use DLQ and retry mechanisms
- **Performance**: Use long polling and batching
- **Cost Optimization**: Batch operations, right-size consumer count
- **Security**: Least-privilege IAM, encryption at rest and in transit
- **Operational Excellence**: CloudWatch alarms on queue depth and DLQ

### 11. Use SQS Extended Client for Large Payloads
- For messages > 256 KB, use the [Amazon SQS Extended Client Library](https://github.com/awslabs/amazon-sqs-java-extended-client-lib)
- Stores payload in S3, sends S3 reference in SQS message

---

## Common Mistakes

### 1. ❌ Not Deleting Messages After Processing
**Problem**: If consumers don't explicitly call `DeleteMessage`, messages reappear after visibility timeout, causing duplicate processing.
**Fix**: Always delete messages in a `finally` block or after confirmed processing.

### 2. ❌ Setting Visibility Timeout Too Short
**Problem**: If processing takes longer than visibility timeout, the message becomes visible again and is processed by another consumer simultaneously.
**Fix**: Set timeout to at least 6x average processing time; use `ChangeMessageVisibility` for variable-length jobs.

### 3. ❌ Using Short Polling
**Problem**: Short polling returns immediately even if the queue is empty, wasting API calls and increasing costs.
**Fix**: Always use long polling (`WaitTimeSeconds: 20`).

### 4. ❌ Not Using a Dead Letter Queue
**Problem**: Poison pill messages (malformed, unprocessable) loop indefinitely, consuming resources and blocking other messages.
**Fix**: