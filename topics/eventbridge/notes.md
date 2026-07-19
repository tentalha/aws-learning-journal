# EventBridge

## What is it?

**Amazon EventBridge** (formerly Amazon CloudWatch Events) is a fully managed, serverless event bus service that enables you to build event-driven architectures by connecting application components using events. It belongs to the **Application Integration** category of AWS services.

EventBridge makes it easy to build loosely coupled, distributed event-driven applications at scale. It ingests events from:
- **AWS services** (e.g., EC2 state changes, S3 uploads, CodePipeline stages)
- **Custom applications** (your own microservices or monoliths)
- **SaaS partners** (e.g., Zendesk, Datadog, PagerDuty, Shopify)

EventBridge routes these events to targets such as Lambda, SQS, SNS, Step Functions, API Gateway, Kinesis, and many more — based on **rules** that match event patterns or schedules.

> **Key Distinction:** EventBridge is not a messaging queue (like SQS) or a pub/sub system (like SNS). It is an event router with rich filtering, schema discovery, and cross-account/cross-region capabilities.

---

## Why do we need it?

### The Problem It Solves

Traditional monolithic architectures tightly couple services, making them brittle and hard to scale. When Service A needs to notify Service B and Service C, you end up with:
- Hard-coded service URLs or ARNs
- Cascading failures when one service is down
- Difficult-to-maintain point-to-point integrations
- No audit trail of what events occurred

EventBridge solves this by acting as a **central nervous system** for your event-driven architecture.

### When to Use It

| Scenario | Why EventBridge |
|----------|----------------|
| Reacting to AWS service events | Native integration, zero code for event generation |
| SaaS integration | Pre-built partner event sources |
| Microservice decoupling | Producers don't know about consumers |
| Scheduled tasks (cron jobs) | Built-in scheduler without managing infrastructure |
| Cross-account event routing | Native multi-account support |
| Audit and compliance | All events can be archived and replayed |

### Real Business Scenarios

1. **E-commerce order processing**: When an order is placed (custom event), trigger inventory checks, payment processing, and email notifications simultaneously — each handled by separate Lambda functions.
2. **Security automation**: When AWS GuardDuty detects a threat, automatically trigger a Lambda to isolate the EC2 instance.
3. **SaaS integration**: When a Zendesk ticket is created, automatically create a Jira issue via Lambda.
4. **Data pipeline orchestration**: Trigger a Glue job when a new file lands in S3.

---

## Internal Working

### Event Flow Architecture

```
Event Source → Event Bus → Rules Engine → Targets
```

### Step-by-Step Internal Flow

1. **Event Ingestion**: An event source (AWS service, custom app, SaaS partner) publishes a JSON event to an event bus using the `PutEvents` API.

2. **Event Bus**: The event bus receives the event. There are three types:
   - **Default bus**: Receives all AWS service events automatically
   - **Custom bus**: For your application events
   - **Partner bus**: For SaaS partner events

3. **Rules Evaluation**: EventBridge evaluates the event against all rules defined on that bus. Rules contain **event patterns** (JSON-based filters) or **schedules** (cron/rate expressions).

4. **Pattern Matching**: EventBridge uses a **content-based filtering** engine. It matches events based on any JSON field, supports prefix matching, suffix matching, numeric ranges, IP address CIDR matching, and more.

5. **Target Invocation**: When a rule matches, EventBridge invokes one or more targets (up to 5 per rule). It can optionally **transform** the event payload before sending using **Input Transformers**.

6. **Retry Logic**: If a target is unavailable, EventBridge retries with exponential backoff for up to **24 hours** (185 times). Failed events can be sent to a **Dead Letter Queue (DLQ)** in SQS.

7. **Delivery Guarantee**: EventBridge provides **at-least-once delivery** — in rare cases, the same event may be delivered more than once. Targets should be idempotent.

### Schema Registry

EventBridge includes a **Schema Registry** that:
- Automatically discovers event schemas from your bus
- Stores schemas in OpenAPI 3.0 format
- Generates code bindings (Java, Python, TypeScript) for type-safe event handling

### EventBridge Pipes

A newer feature that provides **point-to-point integrations** between a source and a target with optional filtering, enrichment (via Lambda/Step Functions), and transformation — without writing custom glue code.

```
Source (SQS/DynamoDB Streams/Kinesis) → [Filter] → [Enrichment] → Target
```

### EventBridge Scheduler

A dedicated scheduling service (separate from EventBridge rules) that supports:
- **One-time schedules**: Run a task once at a specific time
- **Recurring schedules**: Rate-based or cron-based
- **Flexible time windows**: Allow tasks to run within a time window for load distribution
- **Timezone support**: Schedule in specific timezones

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        Amazon EventBridge                        │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐    │
│  │  Event Bus   │   │    Rules     │   │    Schema        │    │
│  │  - Default   │──▶│  - Patterns  │   │    Registry      │    │
│  │  - Custom    │   │  - Schedules │   │  - Discovery     │    │
│  │  - Partner   │   │  - Targets   │   │  - Bindings      │    │
│  └──────────────┘   └──────────────┘   └──────────────────┘    │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐    │
│  │  Archive &   │   │    Pipes     │   │   Scheduler      │    │
│  │  Replay      │   │  (P2P flows) │   │  (One-time/Recur)│    │
│  └──────────────┘   └──────────────┘   └──────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Event Structure

Every EventBridge event is a JSON object with a standard envelope:

```json
{
  "version": "0",
  "id": "12345678-1234-1234-1234-123456789012",
  "source": "com.mycompany.orders",
  "account": "123456789012",
  "time": "2024-01-15T10:30:00Z",
  "region": "us-east-1",
  "resources": ["arn:aws:..."],
  "detail-type": "Order Placed",
  "detail": {
    "orderId": "ORD-001",
    "customerId": "CUST-123",
    "amount": 99.99
  }
}
```

### Key Architectural Patterns

#### 1. Fan-Out Pattern
One event triggers multiple targets (up to 5 per rule, or more via SNS):
```
Order Event → Rule → Lambda (send email)
                   → SQS (inventory check)
                   → Step Functions (fulfillment workflow)
```

#### 2. Cross-Account Event Routing
```
Account A (Producer)          Account B (Consumer)
Custom Bus ──────────────────▶ Custom Bus ──▶ Lambda
           Resource Policy
           allows Account A
```

#### 3. Cross-Region Event Routing
Events can be forwarded to event buses in different AWS regions for global architectures.

#### 4. Archive and Replay
```
Events → Event Bus → Archive (S3-backed, configurable retention)
                   ↓
              Replay to same or different bus for debugging/reprocessing
```

#### 5. EventBridge Pipes (Point-to-Point)
```
DynamoDB Stream → Filter → Lambda (enrich) → EventBridge Bus → Lambda
```

---

## Real World Example

### Scenario: E-Commerce Order Processing System

**Business Requirement**: When a customer places an order, the system must:
1. Deduct inventory
2. Process payment
3. Send confirmation email
4. Notify the warehouse
5. Update analytics dashboard

**Architecture**:

```
Customer App
     │
     ▼
PutEvents API
     │
     ▼
Custom Event Bus (ecommerce-bus)
     │
     ├──▶ Rule: "Order Placed" → Lambda: InventoryService
     ├──▶ Rule: "Order Placed" → Lambda: PaymentService
     ├──▶ Rule: "Order Placed" → SQS: EmailQueue → Lambda: EmailService
     ├──▶ Rule: "Order Placed" → SNS: WarehouseTopic
     └──▶ Rule: "Order Placed" → Kinesis: AnalyticsStream
```

**Step-by-Step Walkthrough**:

**Step 1: Create a Custom Event Bus**
```bash
aws events create-event-bus --name ecommerce-bus
```

**Step 2: Define the Event Schema**
```json
{
  "source": ["com.mycompany.orders"],
  "detail-type": ["Order Placed"],
  "detail": {
    "status": ["CONFIRMED"],
    "amount": [{"numeric": [">", 0]}]
  }
}
```

**Step 3: Create Rules**
```bash
aws events put-rule \
  --name "ProcessNewOrder" \
  --event-bus-name "ecommerce-bus" \
  --event-pattern '{"source":["com.mycompany.orders"],"detail-type":["Order Placed"]}' \
  --state ENABLED
```

**Step 4: Add Targets**
```bash
aws events put-targets \
  --rule "ProcessNewOrder" \
  --event-bus-name "ecommerce-bus" \
  --targets '[
    {"Id":"1","Arn":"arn:aws:lambda:us-east-1:123:function:InventoryService"},
    {"Id":"2","Arn":"arn:aws:lambda:us-east-1:123:function:PaymentService"},
    {"Id":"3","Arn":"arn:aws:sqs:us-east-1:123:EmailQueue"}
  ]'
```

**Step 5: Publish an Event from Your Application**
```javascript
const { EventBridgeClient, PutEventsCommand } = require("@aws-sdk/client-eventbridge");

const client = new EventBridgeClient({ region: "us-east-1" });

await client.send(new PutEventsCommand({
  Entries: [{
    EventBusName: "ecommerce-bus",
    Source: "com.mycompany.orders",
    DetailType: "Order Placed",
    Detail: JSON.stringify({
      orderId: "ORD-001",
      customerId: "CUST-123",
      amount: 149.99,
      status: "CONFIRMED",
      items: [{ sku: "PROD-001", qty: 2 }]
    })
  }]
}));
```

**Step 6: Each Lambda receives the event and processes independently**
- `InventoryService`: Deducts stock from DynamoDB
- `PaymentService`: Charges the customer's card
- `EmailService`: Sends confirmation via SES
- All run in parallel, failures in one don't affect others

**Step 7: Monitor with CloudWatch**
- Set up metric alarms on `FailedInvocations`
- Enable EventBridge Pipes for dead-letter handling

---

## Advantages

| Advantage | Description |
|-----------|-------------|
| **Serverless** | No infrastructure to manage; scales automatically |
| **Native AWS Integration** | 200+ AWS services emit events to the default bus |
| **SaaS Integration** | Pre-built integrations with 35+ SaaS partners |
| **Content-Based Filtering** | Rich pattern matching reduces Lambda invocations |
| **Schema Discovery** | Auto-discovers and registers event schemas |
| **Cross-Account/Region** | Native support for multi-account architectures |
| **Archive & Replay** | Replay past events for debugging or reprocessing |
| **Input Transformation** | Transform event payloads before delivery |
| **Low Latency** | Typically sub-second event delivery |
| **Pay-per-Use** | No idle costs; pay only for events processed |
| **Decoupling** | Producers don't need to know about consumers |
| **EventBridge Pipes** | Zero-code point-to-point integrations |
| **EventBridge Scheduler** | Millions of one-time or recurring schedules |
| **Global Endpoints** | Active-active multi-region event routing for HA |

---

## Limitations

### Hard Limits

| Limit | Value |
|-------|-------|
| Event size | 256 KB per event |
| Targets per rule | 5 |
| Rules per event bus | 300 (soft limit, can increase) |
| Custom event buses per account/region | 100 (soft limit) |
| `PutEvents` batch size | 10 entries per API call |
| Event retention in archive | Configurable (indefinite or up to N days) |
| Retry duration | Up to 24 hours |
| Maximum retry attempts | 185 |
| Schema discovery event size | 8 KB |
| Pipes per account/region | 1,000 (soft limit) |
| Scheduler schedules | 1,000,000 per account/region |

### Functional Limitations

- **At-least-once delivery**: Not exactly-once; targets must be idempotent
- **No ordering guarantee**: Events may be delivered out of order
- **No message batching to Lambda**: Unlike SQS, each event is delivered individually (except via Pipes)
- **Pattern matching is JSON-only**: Cannot filter on binary payloads
- **No built-in message deduplication**: Unlike SQS FIFO
- **Cross-region latency**: Additional latency for cross-region routing
- **Partner events**: Limited to supported SaaS partners
- **Input transformer**: Limited transformation logic; complex transformations need Lambda
- **Scheduler**: Minimum schedule granularity is 1 minute for rate expressions

---

## Best Practices

### Event Design

1. **Use a consistent event schema**: Follow the CloudEvents specification or a custom standard. Include `version`, `correlationId`, and `timestamp` in every event detail.

2. **Use meaningful `source` and `detail-type`**: Use reverse-DNS notation for `source` (e.g., `com.mycompany.orders`) and descriptive past-tense verbs for `detail-type` (e.g., `Order Placed`, `Payment Failed`).

3. **Keep events small**: Stay well under the 256 KB limit. Store large payloads in S3 and include the S3 reference in the event (Claim Check pattern).

4. **Make events self-describing**: Include all necessary context in the event so consumers don't need to make additional API calls.

### Architecture

5. **Use custom event buses**: Separate your application events from AWS service events (default bus). Create separate buses per domain (e.g., `orders-bus`, `payments-bus`).

6. **Implement idempotent consumers**: Since EventBridge provides at-least-once delivery, use `eventId` or a business key for deduplication in your targets.

7. **Use Dead Letter Queues (DLQs)**: Configure an SQS DLQ for every rule target to capture failed event deliveries.

8. **Use Archive and Replay**: Enable archiving for critical event buses to support disaster recovery and debugging.

9. **Use EventBridge Pipes for enrichment**: Instead of writing Lambda glue code, use Pipes for filtering, enrichment, and transformation.

### Security

10. **Least-privilege IAM**: Grant EventBridge only the permissions it needs to invoke specific targets.

11. **Use resource-based policies for cross-account**: Define explicit resource-based policies on event buses rather than overly permissive account-wide policies.

12. **Validate event payloads**: Don't trust event data blindly; validate schemas in your Lambda functions.

###