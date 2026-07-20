# SQS — Interview Questions

## Easy

---

**Q1. What is Amazon SQS and what problem does it solve?**

**Answer:**
Amazon Simple Queue Service (SQS) is a fully managed message queuing service that enables decoupling and scaling of microservices, distributed systems, and serverless applications. It allows producers to send messages to a queue without requiring the consumer to be available at the same time, solving the problem of tight coupling between application components. This asynchronous communication pattern improves fault tolerance, scalability, and resilience.

---

**Q2. What are the two types of queues available in SQS?**

**Answer:**
- **Standard Queue:** Offers maximum throughput, best-effort ordering, and at-least-once delivery. Messages may be delivered more than once and may arrive out of order. Supports nearly unlimited transactions per second (TPS).
- **FIFO Queue (First-In-First-Out):** Guarantees that messages are processed exactly once and in the exact order they are sent. Supports up to 3,000 messages per second with batching (300 without). Ideal for use cases where order and exactly-once processing are critical.

---

**Q3. What is the maximum size of a message in SQS?**

**Answer:**
The maximum message size in SQS is **256 KB**. If you need to send larger payloads, you can use the **SQS Extended Client Library** (available for Java), which stores the actual message body in **Amazon S3** and places a reference pointer in the SQS message. This supports payloads up to **2 GB**.

---

**Q4. What is the visibility timeout in SQS?**

**Answer:**
The visibility timeout is the period of time during which SQS prevents other consumers from receiving and processing a message after one consumer has already retrieved it. This prevents multiple consumers from processing the same message simultaneously. The default visibility timeout is **30 seconds**, with a minimum of **0 seconds** and a maximum of **12 hours**. If the consumer fails to delete the message within the timeout period, the message becomes visible again in the queue for reprocessing.

---

**Q5. What is the maximum retention period for messages in SQS?**

**Answer:**
Messages in SQS can be retained for a minimum of **60 seconds** and a maximum of **14 days**. The default retention period is **4 days**. Messages that are not consumed within the retention period are automatically deleted by SQS. This setting is configurable at the queue level using the `MessageRetentionPeriod` attribute.

---

## Medium

---

**Q1. Explain the difference between short polling and long polling in SQS. When would you use each?**

**Answer:**
- **Short Polling:** SQS queries only a subset of its servers to find available messages and returns immediately, even if the queue is empty. This can result in empty responses and increased API calls, leading to higher costs.
- **Long Polling:** SQS waits for the duration specified (1–20 seconds) for a message to become available before returning a response. This eliminates empty responses when the queue has no messages, reduces the number of API calls, and lowers costs.

**When to use:**
- Use **short polling** when your application requires an immediate response regardless of whether messages are available, or when messages are expected to arrive very frequently.
- Use **long polling** (recommended for most cases) to reduce cost and CPU usage on the consumer side. Enable it by setting `WaitTimeSeconds` to a value between 1 and 20 seconds, or by configuring the queue's `ReceiveMessageWaitTimeSeconds` attribute.

**Best Practice:** Long polling is almost always preferred. AWS recommends enabling it at the queue level to reduce empty receives and associated costs.

---

**Q2. What is a Dead Letter Queue (DLQ) in SQS and how does it work?**

**Answer:**
A Dead Letter Queue (DLQ) is a separate SQS queue where messages that cannot be successfully processed are redirected after a specified number of failed processing attempts. It acts as a safety net to isolate problematic messages for debugging without blocking the main queue.

**How it works:**
1. You create a separate SQS queue to serve as the DLQ.
2. You configure the source queue's **redrive policy**, specifying:
   - `deadLetterTargetArn`: The ARN of the DLQ.
   - `maxReceiveCount`: The number of times a message can be received before being moved to the DLQ (1–1000).
3. When a message's `ApproximateReceiveCount` exceeds `maxReceiveCount`, SQS automatically moves it to the DLQ.

**Important considerations:**
- The DLQ must be of the same type as the source queue (Standard → Standard DLQ, FIFO → FIFO DLQ).
- The message retention period of the DLQ should be set longer than the source queue to allow time for investigation.
- AWS provides **DLQ Redrive** functionality to move messages back to the source queue after fixing the underlying issue.
- CloudWatch alarms should be configured on the `ApproximateNumberOfMessagesVisible` metric of the DLQ to alert on failures.

---

**Q3. How does SQS integrate with AWS Lambda, and what are the key configuration considerations?**

**Answer:**
SQS can act as an event source for AWS Lambda through an **Event Source Mapping (ESM)**. Lambda polls the SQS queue on your behalf and invokes your function with a batch of messages.

**Key configuration considerations:**

| Parameter | Description |
|---|---|
| **Batch Size** | 1–10,000 messages per invocation (default: 10) |
| **Batch Window** | Maximum time Lambda waits to gather a full batch (0–300 seconds) |
| **Concurrency** | Lambda scales up to 1,000 concurrent executions, adding 60 instances/minute |
| **Visibility Timeout** | Should be set to at least 6x the Lambda function timeout |

**Behavior:**
- Lambda polls the queue using long polling and scales automatically based on queue depth.
- For **Standard Queues**, Lambda scales aggressively, adding up to 60 concurrent instances per minute.
- For **FIFO Queues**, Lambda scales up to the number of active message groups.
- If the Lambda function fails to process a batch, the entire batch becomes visible again (unless partial batch response is configured).

**Partial Batch Response:**
By enabling `ReportBatchItemFailures`, Lambda only returns failed message IDs, allowing successful messages to be deleted while failed ones are retried individually.

**Error handling:**
- Configure a DLQ on the SQS queue (not the Lambda function) for failed message handling.
- Set appropriate `maxReceiveCount` to prevent infinite retry loops.

---

**Q4. What is message deduplication in FIFO queues and how does it work?**

**Answer:**
Message deduplication in FIFO queues ensures that duplicate messages are not delivered within a **5-minute deduplication interval**. If a message with the same deduplication ID is sent within this window, SQS accepts the message but does not deliver it again.

**Two methods of deduplication:**

1. **Content-Based Deduplication:**
   - Enable the `ContentBasedDeduplication` attribute on the queue.
   - SQS generates a SHA-256 hash of the message body and uses it as the deduplication ID automatically.
   - Suitable when message content is unique.

2. **Message Deduplication ID:**
   - The producer explicitly provides a `MessageDeduplicationId` with each `SendMessage` call.
   - Gives the producer full control over deduplication logic.
   - Required when message content alone is not sufficient for deduplication (e.g., same payload sent for different business reasons).

**Important:** The deduplication ID must be unique within the 5-minute window. After 5 minutes, the same ID can be reused. This mechanism works at the message group level and across the entire FIFO queue.

---

**Q5. How does SQS handle message ordering, and what are the trade-offs between Standard and FIFO queues?**

**Answer:**
**Standard Queue Ordering:**
- Provides **best-effort ordering** — messages are generally delivered in the order they are sent, but this is not guaranteed.
- Multiple availability zones and distributed processing can cause out-of-order delivery.
- Supports **at-least-once delivery**, meaning duplicates are possible.
- No concept of message groups.

**FIFO Queue Ordering:**
- Guarantees **strict ordering** within a **Message Group ID**.
- Messages with the same `MessageGroupId` are delivered in the exact order they were sent.
- Different message groups can be processed in parallel, enabling both ordering and parallelism.
- Supports **exactly-once processing** within the deduplication window.

**Trade-offs:**

| Aspect | Standard Queue | FIFO Queue |
|---|---|---|
| Throughput | Nearly unlimited | 3,000 msg/s (with batching), 300 msg/s (without) |
| Ordering | Best-effort | Strict (per message group) |
| Delivery | At-least-once | Exactly-once |
| Use case | High-throughput, order-insensitive | Order-critical, deduplication required |
| Cost | Lower | Higher (~10% more) |
| DLQ support | Standard DLQ | FIFO DLQ |

**Design tip:** If you need high throughput with ordering, use multiple FIFO queues or partition your workload across many message groups to maximize parallelism within the FIFO ordering guarantee.

---

## Hard

---

**Q1. Describe the internal architecture of SQS and how it achieves high availability and durability.**

**Answer:**
SQS is built on a massively distributed, redundant infrastructure spread across multiple **Availability Zones (AZs)** within an AWS Region.

**Internal Architecture:**

- **Message Storage:** When a message is sent to SQS, it is stored redundantly across **multiple servers and multiple AZs**. SQS uses a distributed data store (similar to a distributed key-value store) where each message is written to multiple nodes before the `SendMessage` call returns successfully. This ensures durability even if individual nodes or AZs fail.

- **Queue Metadata:** Queue configuration data (attributes, policies, etc.) is also replicated across multiple AZs.

- **Polling Mechanism:** When a consumer calls `ReceiveMessage`, SQS queries a subset of its servers. For short polling, only a subset is queried (which can cause empty responses even when messages exist). For long polling, SQS queries all servers and waits, ensuring more consistent results.

- **Visibility Timeout Implementation:** SQS tracks message visibility using distributed state. When a message is received, its visibility state is updated across the distributed system. This is why there can be brief inconsistencies during the visibility timeout window.

**High Availability:**
- SQS is a **regional service** with built-in multi-AZ redundancy. No single AZ failure can cause message loss.
- AWS guarantees **99.9% availability** for SQS.
- There is no concept of primary/replica — SQS is a peer-to-peer distributed system.

**Durability:**
- Messages are stored on multiple servers across multiple AZs before the send acknowledgment is returned.
- Even during server failures, messages are not lost because copies exist on other nodes.
- SQS does not offer cross-region replication natively; for cross-region DR, you must implement custom replication logic.

**Throughput Scaling:**
- Standard queues scale horizontally by adding more partitions automatically.
- FIFO queues are limited by the number of message groups, which determines parallelism.

---

**Q2. How would you design a system to handle exactly-once processing with a Standard SQS queue (which only guarantees at-least-once delivery)?**

**Answer:**
Since Standard SQS queues guarantee at-least-once delivery, achieving exactly-once processing requires implementing **idempotency** at the consumer level.

**Strategy 1: Idempotency Key with a Database**
- Each SQS message should carry a unique **idempotency key** (e.g., `messageId` or a business-level UUID).
- Before processing, the consumer checks a **distributed data store** (DynamoDB, Redis, RDS) to see if this key has already been processed.
- If not processed, the consumer processes the message and atomically records the key as processed.
- If already processed, the consumer skips processing and deletes the message.

```
DynamoDB Table: ProcessedMessages
  - PK: idempotencyKey (String)
  - TTL: timestamp + 7 days (auto-cleanup)
  - status: "PROCESSING" | "COMPLETED"
```

**Strategy 2: Conditional Writes (DynamoDB)**
- Use DynamoDB's **conditional expressions** to write only if the item does not exist:
  ```
  PutItem with ConditionExpression: "attribute_not_exists(idempotencyKey)"
  ```
- If the condition fails (item exists), skip processing.
- This provides atomic check-and-set semantics.

**Strategy 3: Database Transactions**
- Use database-level transactions to combine the business operation and the idempotency record update atomically.
- Ensures that either both succeed or both fail together.

**Strategy 4: State Machine Pattern**
- Use AWS Step Functions with SQS to manage state transitions.
- Step Functions natively supports idempotency through its execution deduplication.

**Handling the "Processing" State:**
- A message might be received twice while still being processed (visibility timeout expired).
- Use a **"PROCESSING" status with a TTL** to detect and handle in-flight duplicates.
- If a second consumer finds "PROCESSING" status with a recent timestamp, it should back off.

**Additional Considerations:**
- Set an appropriate **visibility timeout** to prevent premature re-delivery.
- Use **DLQ** to handle messages that repeatedly fail idempotency checks.
- Monitor the `ApproximateNumberOfMessagesNotVisible` metric to detect visibility timeout issues.

---

**Q3. Explain SQS message filtering with SNS and how you would architect a fan-out pattern with selective consumption.**

**Answer:**
The **SNS-SQS Fan-out pattern** allows a single message published to an SNS topic to be delivered to multiple SQS queues simultaneously. **Subscription filter policies** enable selective message delivery based on message attributes.

**Architecture:**
```
Producer → SNS Topic → [Filter Policy A] → SQS Queue A → Consumer A
                     → [Filter Policy B] → SQS Queue B → Consumer B
                     → [Filter Policy C] → SQS Queue C → Consumer C
```

**Filter Policy Types (as of 2023):**

1. **Attribute-based filtering (original):** Filters based on message attributes (key-value pairs in the message envelope).
2. **Body-based filtering:** Filters based on the JSON message body content.

**Filter Policy Example:**
```json
{
  "eventType": ["ORDER_PLACED", "ORDER_CANCELLED"],
  "region": ["us-east-1", "eu-west-1"],
  "amount": [{"numeric": [">=", 100]}]
}
```

**Advanced Architecture — Multi-tier Fan-out:**
```
Event Producer
    ↓
SNS Topic (all events)
    ↓ ↓ ↓
SQS-A   SQS-B   SQS-C
(orders) (payments) (notifications)
    ↓       ↓         ↓
Lambda  EC2 ASG   Lambda
```

**Key Design Considerations:**

- **Message Attribute Limits:** Up to 10 message attributes per message; filter policies support up to 5 attribute conditions.
- **SQS Queue Policy:** Each SQS queue must have a resource policy allowing SNS to send messages:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"Service": "sns.amazonaws.com"},
    "Action": "sqs:SendMessage",
    "Condition": {"ArnEquals": {"aws:SourceArn": "arn:aws:sns:..."}}
  }
  ```
- **FIFO Considerations:** SNS FIFO topics can fan out to SQS FIFO queues, preserving ordering and deduplication end-to-end.
- **Message Transformation:** SNS can deliver raw messages (disable `RawMessageDelivery = false`) or wrap them in SNS envelope format. Enable `RawMessageDelivery` on SQS subscriptions to avoid parsing the SNS envelope.
- **Throughput:** SNS fan-out is nearly instantaneous; bottleneck is typically at the SQS/consumer layer.

**Operational Considerations:**
- Use **CloudWatch metrics** on each SQS queue to monitor per-service consumption rates.
- Configure **DLQs** on each SQS queue independently for isolated error handling.
- Use **AWS X-Ray** tracing across SNS and SQS for end-to-end observability.

---

**Q4. How does SQS handle backpressure, and