# Step Functions

## What is it?

**AWS Step Functions** is a fully managed serverless orchestration service that enables you to coordinate multiple AWS services into serverless workflows using visual state machines. It falls under the **Application Integration** category in the AWS ecosystem.

Step Functions allows you to model complex business logic as a series of steps (states) defined in **Amazon States Language (ASL)**, a JSON-based structured language. Each state can perform work, make decisions, wait for external events, or handle errors — all without writing glue code or managing infrastructure.

There are two workflow types:
- **Standard Workflows**: Exactly-once execution, up to 1 year duration, full audit history via execution history.
- **Express Workflows**: At-least-once execution, up to 5 minutes duration, designed for high-volume, short-duration workloads.

---

## Why do we need it?

### The Problem

Modern cloud applications are rarely monolithic. They involve chaining multiple services — Lambda functions, DynamoDB operations, SNS notifications, ECS tasks, and third-party APIs — in a specific sequence with conditional logic, error handling, retries, and parallel processing. Without an orchestration layer, developers are forced to:

- Write complex, brittle "glue code" in Lambda functions that call other Lambda functions
- Manually handle retries and exponential backoff
- Lose visibility into where a workflow failed
- Hardcode timeouts and polling mechanisms
- Tightly couple services together

### The Solution Step Functions Provides

Step Functions externalizes workflow logic from application code, providing:

- **Durable state management** between steps
- **Built-in error handling** with retry and catch mechanisms
- **Visual workflow monitoring** via the AWS Console
- **Decoupled service orchestration** without tight coupling

### Real Business Scenarios

| Scenario | Without Step Functions | With Step Functions |
|---|---|---|
| **E-commerce Order Processing** | Lambda calling Lambda, losing state on failure | State machine tracking each order stage |
| **ETL Data Pipeline** | Cron jobs with polling loops | Event-driven pipeline with wait states |
| **ML Model Training** | Manual job monitoring scripts | Automated training, evaluation, deployment |
| **User Onboarding** | Complex conditional logic in monolith | Visual multi-step approval workflow |
| **Financial Transaction Processing** | Fragile chained API calls | Auditable, retry-safe transaction flow |

---

## Internal Working

### State Machine Execution Engine

When you start a Step Functions execution, the service:

1. **Instantiates a new execution context** with a unique execution ARN
2. **Loads the state machine definition** (ASL JSON) from the registered state machine
3. **Begins at the `StartAt` state** defined in the ASL
4. **Evaluates state type** and dispatches accordingly:
   - `Task` → Calls the configured resource (Lambda, ECS, etc.)
   - `Choice` → Evaluates conditions against the current input
   - `Wait` → Pauses execution for a duration or until a timestamp
   - `Parallel` → Forks into concurrent branches
   - `Map` → Iterates over an array of items
   - `Pass` → Passes input to output, optionally transforming
   - `Succeed` / `Fail` → Terminal states

### State Transitions

Each state receives a JSON input document, processes it, and produces a JSON output document that becomes the input for the next state. You can control this flow using:

- **`InputPath`**: Filters the input using a JsonPath expression
- **`Parameters`**: Constructs a new JSON payload to pass to the resource
- **`ResultSelector`**: Filters the raw result from the resource
- **`ResultPath`**: Specifies where to place the result in the state's output
- **`OutputPath`**: Filters the final output before passing to the next state

### Execution History

For **Standard Workflows**, every state transition is logged to the execution history (up to 25,000 events per execution). This provides a complete audit trail. **Express Workflows** use CloudWatch Logs instead for high-throughput scenarios.

### Heartbeat and Timeout Mechanisms

For long-running Task states using the **waitForTaskToken** pattern:
- The task sends a token to an external system
- Step Functions waits until `SendTaskSuccess` or `SendTaskFailure` is called
- A `HeartbeatSeconds` timeout can be set to detect stalled workers

### Synchronous vs. Asynchronous Integration

Step Functions supports two SDK integration patterns:
- **Request-Response** (default): Calls the service and moves to next state immediately
- **`.sync`**: Waits for the job/task to complete (e.g., `ecs:runTask.sync`)
- **`.waitForTaskToken`**: Pauses until an external callback with the task token

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────┐
│                    Step Functions                         │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │              State Machine Definition             │    │
│  │         (Amazon States Language / ASL)            │    │
│  └──────────────────────┬──────────────────────────┘    │
│                         │                                │
│  ┌──────────────────────▼──────────────────────────┐    │
│  │                  Execution Engine                 │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │    │
│  │  │  State 1 │→ │  State 2 │→ │   State N    │   │    │
│  │  │  (Task)  │  │ (Choice) │  │  (Parallel)  │   │    │
│  │  └──────────┘  └──────────┘  └──────────────┘   │    │
│  └──────────────────────┬──────────────────────────┘    │
│                         │                                │
│  ┌──────────────────────▼──────────────────────────┐    │
│  │              Execution History / Logs             │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
          │              │              │
    ┌─────▼──┐    ┌──────▼─┐    ┌──────▼──┐
    │ Lambda │    │  ECS   │    │  SQS    │
    └────────┘    └────────┘    └─────────┘
```

### Key Architectural Patterns

#### 1. Sequential Pipeline
```
[Validate Input] → [Process Data] → [Store Result] → [Send Notification]
```

#### 2. Parallel Fan-Out / Fan-In
```
                    ┌→ [Branch A: Email Notification]  ─┐
[Trigger Event] ────┼→ [Branch B: SMS Notification]   ─┼→ [Aggregate Results]
                    └→ [Branch C: Push Notification]  ─┘
```

#### 3. Dynamic Map (Iterator)
```
[Load Order Items] → [Map State: Process Each Item] → [Aggregate Results]
                           ↓ (for each item)
                     [Validate] → [Charge] → [Fulfill]
```

#### 4. Saga Pattern (Distributed Transactions)
```
[Reserve Inventory] → [Charge Payment] → [Create Shipment]
        ↓ (on failure)        ↓ (on failure)
[Release Inventory] ← [Refund Payment] ← [Cancel Shipment]
```

#### 5. Human Approval Workflow
```
[Submit Request] → [Send Approval Email with TaskToken]
                              ↓
                   [Wait for Callback (waitForTaskToken)]
                              ↓
              ┌── Approved ──→ [Process Request]
              └── Rejected ──→ [Notify Requester]
```

### Workflow Types Comparison

| Feature | Standard | Express |
|---|---|---|
| Max Duration | 1 year | 5 minutes |
| Execution Rate | 2,000/sec (default) | 100,000/sec |
| Execution History | Stored in SF | CloudWatch Logs |
| Pricing | Per state transition | Per execution duration |
| Execution Semantics | Exactly-once | At-least-once |
| Idempotency | Built-in | Must implement manually |
| Use Case | Long-running, auditable | High-volume, short-duration |

---

## Real World Example

### E-Commerce Order Fulfillment Pipeline

**Scenario**: An online retailer needs to process orders involving inventory checks, payment processing, warehouse notification, and customer communication.

#### Step 1: Define the State Machine

```json
{
  "Comment": "E-Commerce Order Fulfillment",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ValidateOrder",
      "Next": "CheckInventory",
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException", "Lambda.TooManyRequestsException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["InvalidOrderError"],
          "Next": "OrderValidationFailed",
          "ResultPath": "$.error"
        }
      ]
    },
    "CheckInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:CheckInventory",
      "Next": "InventoryAvailable?",
      "ResultPath": "$.inventoryResult"
    },
    "InventoryAvailable?": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.inventoryResult.available",
          "BooleanEquals": true,
          "Next": "ProcessPaymentAndNotify"
        }
      ],
      "Default": "BackorderItem"
    },
    "ProcessPaymentAndNotify": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "ProcessPayment",
          "States": {
            "ProcessPayment": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ProcessPayment",
              "End": true
            }
          }
        },
        {
          "StartAt": "NotifyWarehouse",
          "States": {
            "NotifyWarehouse": {
              "Type": "Task",
              "Resource": "arn:aws:states:::sqs:sendMessage",
              "Parameters": {
                "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123456789012/WarehouseQueue",
                "MessageBody.$": "States.JsonToString($.order)"
              },
              "End": true
            }
          }
        }
      ],
      "Next": "SendConfirmationEmail",
      "Catch": [
        {
          "ErrorEquals": ["PaymentDeclinedError"],
          "Next": "HandlePaymentFailure"
        }
      ]
    },
    "SendConfirmationEmail": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:OrderConfirmations",
        "Message.$": "States.Format('Order {} confirmed!', $.order.id)"
      },
      "End": true
    },
    "BackorderItem": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:HandleBackorder",
      "End": true
    },
    "HandlePaymentFailure": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:HandlePaymentFailure",
      "End": true
    },
    "OrderValidationFailed": {
      "Type": "Fail",
      "Error": "OrderValidationError",
      "Cause": "The order failed validation checks"
    }
  }
}
```

#### Step 2: Execution Flow

1. **Customer places order** → API Gateway triggers Step Functions execution
2. **ValidateOrder state** → Lambda validates order data (items, address, etc.)
3. **CheckInventory state** → Lambda queries DynamoDB for stock levels
4. **InventoryAvailable? choice** → Routes based on stock availability
5. **ProcessPaymentAndNotify parallel** → Simultaneously charges card AND notifies warehouse
6. **SendConfirmationEmail** → SNS publishes confirmation to customer
7. **Execution completes** with `SUCCEEDED` status

#### Step 3: Error Handling in Action

If `ProcessPayment` Lambda throws `PaymentDeclinedError`:
- The `Catch` block intercepts the error
- Execution routes to `HandlePaymentFailure`
- Customer receives decline notification
- Inventory reservation is released
- Execution completes with `SUCCEEDED` (business failure handled gracefully)

---

## Advantages

1. **Visual Workflow Representation**: The AWS Console provides a graphical view of state machine execution, making debugging and monitoring intuitive for developers and non-technical stakeholders alike.

2. **Built-in Error Handling**: Native retry logic with configurable `IntervalSeconds`, `MaxAttempts`, `BackoffRate`, and `Catch` blocks eliminate boilerplate error handling code.

3. **Durable Execution State**: For Standard Workflows, state is persisted between steps, so a failure at step 7 of 10 doesn't require restarting from the beginning (with proper Catch/Retry configuration).

4. **No Infrastructure Management**: Fully serverless — no servers, containers, or workers to provision, patch, or scale.

5. **Broad AWS Service Integration**: Native SDK integrations with 200+ AWS services including Lambda, ECS, Fargate, Glue, SageMaker, DynamoDB, SNS, SQS, EventBridge, and more.

6. **Optimistic Concurrency**: Multiple executions of the same state machine run independently without interference.

7. **Long-Running Workflow Support**: Standard Workflows can run for up to 1 year, supporting human approval steps, batch jobs, and complex multi-day processes.

8. **Intrinsic Functions**: Built-in functions like `States.Format`, `States.JsonMerge`, `States.ArrayGetItem`, `States.MathAdd` reduce the need for Lambda functions just for data transformation.

9. **Audit Trail**: Complete execution history for compliance, debugging, and operational visibility.

10. **Express Workflow Performance**: Express Workflows can handle over 100,000 executions per second, suitable for high-throughput IoT, streaming, and microservice scenarios.

---

## Limitations

### Hard Limits

| Limit | Value |
|---|---|
| Maximum execution duration (Standard) | 1 year |
| Maximum execution duration (Express) | 5 minutes |
| Maximum state machine definition size | 1 MB |
| Maximum execution history events | 25,000 events |
| Maximum execution input/output size | 256 KB |
| Maximum activity task polling workers | Unlimited (but 1,000 concurrent per activity) |
| Maximum state transitions/sec (Standard, default) | 2,000 per account per region |
| Maximum execution starts/sec (Express, default) | 100,000 per account per region |
| Maximum concurrent Map iterations | 40 (default, adjustable) |
| Maximum nested Map depth | 5 levels |

### Soft Limits (Adjustable via Service Quotas)

- State machine count per account per region: 10,000
- Concurrent executions (Standard): 1,000,000
- API request rate: varies by API

### Functional Limitations

- **No built-in looping**: You must use the `Map` state or re-trigger executions for loops
- **No direct HTTP calls**: External HTTP endpoints require a Lambda intermediary (unless using EventBridge API Destinations)
- **256 KB payload limit**: Large data must be stored in S3/DynamoDB and referenced by pointer
- **ASL complexity**: Complex JsonPath expressions and intrinsic functions can become difficult to maintain
- **Express