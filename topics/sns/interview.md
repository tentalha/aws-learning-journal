# SNS — Interview Questions

---

## Easy

### 1. What is Amazon SNS, and what is its primary use case?

**Answer:**
Amazon Simple Notification Service (SNS) is a fully managed pub/sub (publish-subscribe) messaging service that enables you to decouple microservices, distributed systems, and serverless applications. Its primary use case is **fan-out messaging** — sending a single message from a publisher to multiple subscribers simultaneously. Common use cases include application alerts, push notifications to mobile devices, email notifications, and triggering downstream processing pipelines.

---

### 2. What are the two main types of topics in SNS?

**Answer:**
SNS supports two types of topics:

- **Standard Topics:** Offer maximum throughput, best-effort ordering, and at-least-once delivery. Messages may be delivered out of order and potentially more than once. Suitable for high-throughput, order-insensitive workloads.
- **FIFO Topics (First-In-First-Out):** Guarantee strict message ordering and exactly-once delivery. They can only be subscribed to by SQS FIFO queues. Throughput is limited compared to Standard topics. Suitable for scenarios requiring strict ordering, such as financial transactions.

---

### 3. What are the supported subscription protocols in SNS?

**Answer:**
SNS supports the following subscription protocols:

| Protocol | Description |
|---|---|
| **HTTP/HTTPS** | Delivers messages to a web endpoint |
| **Email / Email-JSON** | Sends messages as email |
| **SQS** | Delivers messages to an SQS queue |
| **Lambda** | Invokes an AWS Lambda function |
| **SMS** | Sends text messages to mobile phones |
| **Mobile Push** | Sends push notifications (APNs, FCM, ADM, etc.) |
| **Kinesis Data Firehose** | Delivers messages to a Firehose delivery stream |

---

### 4. What is the maximum message size supported by SNS?

**Answer:**
SNS supports a maximum message size of **256 KB** per publish request. If you need to send larger payloads, a common pattern is to store the payload in Amazon S3 and publish only the S3 object reference (URL or key) in the SNS message. AWS also provides the **SNS Extended Client Library** (similar to the SQS Extended Client Library) to automate this pattern.

---

### 5. What is an SNS Topic ARN, and why is it important?

**Answer:**
An **ARN (Amazon Resource Name)** for an SNS topic is a unique identifier that follows the format:

```
arn:aws:sns:<region>:<account-id>:<topic-name>
```

**Example:**
```
arn:aws:sns:us-east-1:123456789012:MyTopic
```

The Topic ARN is important because:
- It is required to **publish messages** to a topic.
- It is used in **IAM policies** to grant or restrict access.
- Subscribers use it to **subscribe** to the topic.
- It uniquely identifies the topic across all AWS services and accounts.

---

## Medium

### 1. How does SNS message filtering work, and why is it useful?

**Answer:**
**SNS Message Filtering** allows each subscriber to receive only a subset of messages published to a topic by applying a **filter policy** (a JSON document) to the subscription. Without filtering, every subscriber receives every message.

**How it works:**
- The publisher includes **message attributes** (key-value pairs) when publishing a message.
- Each subscription has an optional **filter policy** that specifies which attribute values it wants to receive.
- SNS evaluates the filter policy against the message attributes and only delivers the message if there is a match.

**Example filter policy:**
```json
{
  "store": ["example_corp"],
  "event": ["order_placed", "order_updated"],
  "price_usd": [{"numeric": [">=", 100]}]
}
```

**Why it's useful:**
- **Reduces unnecessary processing:** Subscribers (e.g., Lambda functions, SQS queues) only receive relevant messages, reducing invocations and cost.
- **Simplifies architecture:** Instead of creating multiple topics for different message types, you can use a single topic with filtering.
- **Decouples publishers from subscribers:** Publishers don't need to know which subscribers want which messages.

**Filter policy scope:** By default, filtering is applied to message attributes. As of 2022, SNS also supports **filter policy scope on the message body**, allowing filtering based on the JSON message body content.

---

### 2. Explain the SNS fan-out pattern with SQS. When would you use it?

**Answer:**
The **SNS fan-out pattern** involves publishing a single message to an SNS topic, which then delivers the message to multiple SQS queues simultaneously. Each SQS queue has its own set of consumers that process messages independently.

**Architecture:**
```
Publisher → SNS Topic → SQS Queue A → Consumer A (e.g., Lambda)
                      → SQS Queue B → Consumer B (e.g., EC2)
                      → SQS Queue C → Consumer C (e.g., ECS)
```

**When to use it:**
1. **Parallel processing:** Multiple independent systems need to react to the same event (e.g., an order placed event triggers inventory update, email notification, and analytics ingestion simultaneously).
2. **Decoupling with durability:** SQS provides buffering so consumers can process at their own pace without message loss.
3. **Cross-account or cross-region delivery:** SNS can fan out to SQS queues in different AWS accounts or regions.
4. **Different processing logic:** Each downstream system applies its own business logic without affecting others.

**Key benefits over direct SQS:**
- A single publish operation reaches all subscribers.
- Adding a new consumer is as simple as adding a new SQS subscription — no publisher changes needed.
- Each queue can have independent retry, DLQ, and visibility timeout settings.

---

### 3. What is SNS Dead Letter Queue (DLQ) support, and how does it work?

**Answer:**
SNS supports **Dead Letter Queues (DLQs)** at the **subscription level** (not the topic level). When SNS fails to deliver a message to a subscriber after exhausting all retry attempts, it can send the failed message to an SQS queue designated as the DLQ.

**How it works:**
1. SNS attempts to deliver a message to a subscriber endpoint.
2. If delivery fails (e.g., HTTP endpoint returns a 5xx error, Lambda function throws an error), SNS retries based on a **retry policy** with exponential backoff.
3. After all retries are exhausted, if a DLQ is configured for that subscription, the undelivered message is sent to the DLQ along with metadata about the failure.
4. Engineers can then inspect, replay, or alert on messages in the DLQ.

**Retry policy defaults (for HTTP/HTTPS):**
- Immediate retries: 3 attempts
- Pre-backoff phase: 2 attempts (1 second apart)
- Backoff phase: 10 attempts (exponential backoff, up to 20 seconds between retries)
- Post-backoff phase: 100,000 attempts (20 seconds apart)

**DLQ message attributes include:**
- `ERROR_MESSAGE` — reason for failure
- `ERROR_CODE` — HTTP status code or error type
- `SUBSCRIPTION_ARN` — which subscription failed

**Important note:** DLQ support is available for SQS, Lambda, HTTP/HTTPS, and Kinesis Data Firehose subscriptions. Email and SMS subscriptions do not support DLQs.

---

### 4. How does SNS handle message delivery retries for different endpoint types?

**Answer:**
SNS has different retry behaviors depending on the subscription protocol:

**HTTP/HTTPS Endpoints:**
- SNS follows a configurable retry policy with exponential backoff.
- Default: up to **100,015 attempts** over approximately **23 days**.
- You can customize the number of retries, minimum/maximum delay, and backoff function (linear, geometric, exponential).
- If the endpoint returns a 2xx response, delivery is considered successful.

**SQS:**
- SNS retries delivery if SQS is temporarily unavailable.
- Typically retries for **up to 23 days**.
- Failures are rare since SQS is highly available within a region.

**Lambda:**
- SNS invokes Lambda synchronously.
- If Lambda returns an error or throttling occurs, SNS retries with backoff.
- Lambda's own retry behavior (for async invocations) is separate.

**Email/SMS:**
- Limited retry behavior; these are best-effort delivery protocols.
- No DLQ support.

**Mobile Push (APNs, FCM):**
- SNS retries push notifications if the push provider is temporarily unavailable.
- Invalid device tokens result in immediate failure (no retry).

**Key insight:** SNS delivery retry policies are configured per-subscription and are separate from any retry logic in the consuming application.

---

### 5. What is the difference between SNS and SQS, and when would you use each?

**Answer:**

| Feature | SNS | SQS |
|---|---|---|
| **Model** | Pub/Sub (push-based) | Queue (pull-based) |
| **Delivery** | Push to multiple subscribers simultaneously | Pull by a single consumer group |
**Persistence** | No message persistence (fire and forget) | Messages persisted up to 14 days |
| **Consumers** | Multiple simultaneous subscribers | One consumer group per queue |
| **Ordering** | FIFO topic for ordered delivery | FIFO queue for ordered delivery |
| **Fan-out** | Native fan-out to multiple endpoints | Not natively; requires multiple queues |
| **Filtering** | Subscription-level filter policies | No built-in filtering |
| **Use case** | Broadcast events to multiple systems | Decouple producer/consumer, buffer workloads |

**When to use SNS:**
- You need to notify multiple systems of an event simultaneously.
- You want to send mobile push notifications, emails, or SMS.
- You need real-time event broadcasting.

**When to use SQS:**
- You need to buffer and decouple a producer from a single consumer.
- You need guaranteed message persistence and at-least-once processing.
- You need to handle traffic spikes gracefully.

**When to use both (SNS + SQS fan-out):**
- You need both fan-out AND durability/buffering.
- Multiple independent consumers need to process the same event at their own pace.

---

## Hard

### 1. How does SNS achieve high availability and durability, and what are its consistency guarantees?

**Answer:**
SNS is designed for high availability and durability through several architectural mechanisms:

**Replication:**
- SNS replicates messages across **multiple Availability Zones** within a region before acknowledging the publish request to the client.
- This ensures that a single AZ failure does not result in message loss.

**Delivery Guarantees:**
- **Standard topics:** Provide **at-least-once delivery**, meaning a message may be delivered more than once. Consumers must be idempotent.
- **FIFO topics:** Provide **exactly-once delivery** (deduplication) using a `MessageDeduplicationId`. If the same ID is sent within a 5-minute deduplication interval, the duplicate is discarded.

**Consistency:**
- SNS is **eventually consistent** for metadata operations (e.g., creating subscriptions, updating filter policies). There may be a brief delay before all SNS nodes reflect the change.
- Message delivery is **not transactional** — if a publish call succeeds but a downstream subscriber fails, SNS retries delivery independently without rolling back the publish.

**Throughput:**
- Standard topics support virtually unlimited throughput (soft limit of 30,000 publishes/second, adjustable).
- FIFO topics support up to **300 publishes/second** or **10 MB/second** per topic.

**Cross-region considerations:**
- SNS is a **regional service**. For cross-region redundancy, you must replicate your topic architecture across regions and implement routing logic (e.g., via Route 53 or application-level failover).

**Operational insight:** For mission-critical systems, you should combine SNS with SQS DLQs, CloudWatch alarms on `NumberOfNotificationsFailed`, and cross-region replication strategies to achieve the desired resilience level.

---

### 2. Explain SNS FIFO topics in detail — ordering guarantees, deduplication, and limitations.

**Answer:**

**SNS FIFO Topics** were introduced to support use cases requiring strict message ordering and exactly-once delivery.

**Ordering Guarantees:**
- Messages are delivered in the **exact order they are published** within a **Message Group**.
- The `MessageGroupId` attribute partitions messages into independent ordered streams. Messages within the same group are strictly ordered; messages across different groups may be interleaved.
- This is analogous to Kafka partitions — each `MessageGroupId` is an independent ordered lane.

**Deduplication:**
SNS FIFO supports two deduplication methods:
1. **Content-based deduplication:** SNS computes a SHA-256 hash of the message body. If an identical hash is published within the 5-minute deduplication window, the duplicate is discarded.
2. **MessageDeduplicationId:** The publisher provides an explicit deduplication ID. SNS discards any message with the same ID within the 5-minute window.

**Subscriber Constraints:**
- FIFO topics can **only** be subscribed to by **SQS FIFO queues**.
- HTTP, Lambda, email, SMS, and mobile push subscriptions are **not supported**.
- This is by design — to maintain ordering guarantees end-to-end, the subscriber must also support FIFO semantics.

**Throughput Limitations:**
- Up to **300 publishes/second** (without batching) or **3,000 messages/second** with batching (10 messages per batch).
- This is significantly lower than Standard topics and must be factored into capacity planning.

**Message Filtering:**
- FIFO topics support subscription filter policies, but filtering is applied **after** ordering and deduplication — the sequence numbers are preserved even if a message is filtered out for a specific subscriber.

**When to use FIFO topics:**
- Financial transaction processing where order matters (e.g., debit before credit).
- Inventory management where stock updates must be applied in sequence.
- Event sourcing systems where event order is critical for state reconstruction.

---

### 3. How would you implement cross-account SNS message delivery, and what security considerations are involved?

**Answer:**

**Cross-account SNS delivery** allows a topic in Account A to deliver messages to subscribers in Account B (e.g., an SQS queue in Account B).

**Implementation Steps:**

**Step 1: SNS Topic Resource Policy (Account A)**
Grant Account B permission to subscribe to the topic:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_B_ID:root"
      },
      "Action": "sns:Subscribe",
      "Resource": "arn:aws:sns:us-east-1:ACCOUNT_A_ID:MyTopic",
      "Condition": {
        "StringEquals": {
          "sns:Protocol": "sqs"
        }
      }
    }
  ]
}
```

**Step 2: SQS Queue Policy (Account B)**
Allow the SNS topic from Account A to send messages to the queue:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:ACCOUNT_B_ID:MyQueue",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "arn:aws:sns:us-east-1:ACCOUNT_A_ID:MyTopic"
        }
      }
    }
  ]
}
```

**Step 3: Create the Subscription**
Account B (or Account A with appropriate permissions) creates the subscription using the SNS topic ARN and SQS queue ARN.

**Security Considerations:**

1. **Principle of Least Privilege:** Grant only the minimum required permissions. Use `Condition` blocks to restrict by source ARN, account ID, and protocol.
2. **Confused Deputy Problem:** Always use `aws:SourceArn` or `aws:SourceAccount` conditions in the SQS policy to prevent confused deputy attacks where a malicious SNS topic from another account sends messages to your queue.
3. **Encryption:** If the SNS topic uses SSE (Server-Side Encryption with KMS), the KMS key policy must grant the SNS service