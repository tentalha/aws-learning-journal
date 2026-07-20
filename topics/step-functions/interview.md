# Step Functions — Interview Questions

---

## Easy

### 1. What is AWS Step Functions, and what problem does it solve?

**Answer:**
AWS Step Functions is a serverless orchestration service that lets you coordinate multiple AWS services into serverless workflows using visual state machines. It solves the problem of managing complex, multi-step application logic that would otherwise require custom code to handle retries, error handling, branching, and sequencing. Instead of embedding orchestration logic inside Lambda functions or other compute resources, Step Functions externalizes it into a declarative workflow definition written in Amazon States Language (ASL).

---

### 2. What is Amazon States Language (ASL)?

**Answer:**
Amazon States Language (ASL) is a JSON-based structured language used to define state machines in AWS Step Functions. It describes the states in a workflow, the transitions between them, input/output processing, retry logic, and error handling. Each state machine has a `StartAt` field pointing to the first state, and each state has a `Type` (e.g., `Task`, `Choice`, `Wait`, `Parallel`, `Map`, `Pass`, `Succeed`, `Fail`).

---

### 3. What are the different state types available in Step Functions?

**Answer:**
Step Functions supports eight state types:

| State Type | Purpose |
|---|---|
| **Task** | Performs work by invoking an AWS service or activity |
| **Choice** | Adds branching logic based on conditions |
| **Wait** | Pauses execution for a fixed time or until a timestamp |
| **Parallel** | Executes multiple branches simultaneously |
| **Map** | Iterates over an array of items |
| **Pass** | Passes input to output, optionally transforming it |
| **Succeed** | Terminates execution successfully |
| **Fail** | Terminates execution with an error |

---

### 4. What is the difference between Standard and Express Workflows?

**Answer:**

| Feature | Standard Workflow | Express Workflow |
|---|---|---|
| **Duration** | Up to 1 year | Up to 5 minutes |
| **Execution model** | Exactly-once | At-least-once |
| **Execution rate** | 2,000/sec | 100,000/sec |
| **Pricing** | Per state transition | Per execution duration + requests |
| **Audit history** | Full execution history in console | CloudWatch Logs only |
| **Use case** | Long-running, auditable workflows | High-volume, short-duration workflows |

Standard Workflows are best for business-critical processes requiring exactly-once execution semantics. Express Workflows are suited for high-volume event processing, IoT data ingestion, and streaming workloads.

---

### 5. How does Step Functions handle errors and retries?

**Answer:**
Step Functions provides built-in error handling at the state level using two mechanisms:

- **Retry:** Automatically retries a failed state a configurable number of times with exponential backoff. You specify `ErrorEquals` (which errors to catch), `IntervalSeconds` (initial wait), `MaxAttempts`, and `BackoffRate` (multiplier for each retry).
- **Catch:** If retries are exhausted or a specific error occurs, the `Catch` block redirects the workflow to a fallback state, allowing graceful degradation or compensating actions.

```json
"Retry": [
  {
    "ErrorEquals": ["States.TaskFailed"],
    "IntervalSeconds": 2,
    "MaxAttempts": 3,
    "BackoffRate": 2.0
  }
],
"Catch": [
  {
    "ErrorEquals": ["States.ALL"],
    "Next": "HandleError"
  }
]
```

---

## Medium

### 1. Explain the difference between `.waitForTaskToken` and `.sync:2` integration patterns. When would you use each?

**Answer:**
Step Functions supports several service integration patterns:

**Request-Response (default):**
Step Functions calls the service and moves to the next state immediately after the API call is accepted, without waiting for completion.

**`.sync:2` (Synchronous):**
Step Functions calls the service and waits for the job to complete before transitioning. It polls the service internally. This is supported for services like ECS, Glue, SageMaker, and Batch. Example resource: `arn:aws:states:::glue:startJobRun.sync:2`

**`.waitForTaskToken`:**
Step Functions passes a unique task token to the downstream system and pauses execution indefinitely until `SendTaskSuccess` or `SendTaskFailure` is called with that token. This is ideal for:
- Human approval workflows
- Calling external systems (on-premises, third-party APIs)
- Waiting for asynchronous callbacks from SQS consumers or external services

**When to use each:**
- Use `.sync:2` when the downstream AWS service supports native polling and you want Step Functions to manage the wait.
- Use `.waitForTaskToken` when the workflow must pause for an external event, a human decision, or a system that cannot be polled by Step Functions natively.

---

### 2. How does input/output processing work in Step Functions using `InputPath`, `Parameters`, `ResultSelector`, `ResultPath`, and `OutputPath`?

**Answer:**
Step Functions provides a pipeline of data transformation filters at each state:

1. **`InputPath`** — Selects a subset of the state's input using a JSONPath expression to pass to the task. Default is `$` (entire input).

2. **`Parameters`** — Constructs a new JSON object to send to the task. Can combine static values and dynamic values from the input using `.$` suffix notation.

3. **`ResultSelector`** — Transforms the raw result from the task before it is used further. Useful for extracting only relevant fields from verbose API responses.

4. **`ResultPath`** — Specifies where in the original input to insert the task result. Using `$.taskResult` appends the result under that key, preserving the original input. Setting it to `null` discards the result.

5. **`OutputPath`** — Selects a portion of the combined state data to pass to the next state.

**Processing order:**
```
Raw Input → InputPath → Parameters → [Task Execution] → ResultSelector → ResultPath → OutputPath → Next State Input
```

**Example use case:** A Lambda function returns a large response object, but you only need one field. Use `ResultSelector` to extract it, then `ResultPath` to merge it into the original context without losing upstream data.

---

### 3. What are Step Functions Activities, and how do they differ from Lambda Task states?

**Answer:**
**Lambda Task states** invoke a Lambda function directly and synchronously (or with `.sync` patterns). Step Functions manages the invocation and waits for the Lambda response. This is tightly coupled to AWS compute.

**Activities** are a pull-based model where:
1. You register an activity ARN in Step Functions.
2. An external worker (running anywhere — EC2, on-premises, containers) polls Step Functions using `GetActivityTask`.
3. Step Functions holds the task until the worker picks it up, executes the work, and calls `SendTaskSuccess` or `SendTaskFailure`.
4. A heartbeat mechanism (`SendTaskHeartbeat`) keeps the task alive for long-running jobs.

**Key differences:**

| Aspect | Lambda Task | Activity |
|---|---|---|
| Worker location | AWS Lambda only | Anywhere (EC2, on-prem, etc.) |
| Invocation model | Push (Step Functions calls Lambda) | Pull (worker polls Step Functions) |
| Use case | Short-duration, serverless compute | Long-running or external compute |
| Heartbeat | Not applicable | Required for long tasks |

Activities are useful for legacy systems, GPU-intensive workloads, or scenarios where the compute environment cannot be triggered by AWS directly.

---

### 4. Describe how the `Map` state works, including its concurrency controls and nested execution modes.

**Answer:**
The `Map` state iterates over an array in the input and executes a defined set of states (an iterator) for each element, similar to a `forEach` loop. It supports two modes:

**Inline Mode (default):**
- The iterator runs within the same state machine execution.
- State transitions are counted toward the parent execution's limits.
- All iterations must complete before the Map state transitions to the next state.

**Distributed Mode (newer feature):**
- Each iteration is run as a separate child execution of a specified state machine.
- Supports processing very large datasets (up to 40 concurrent child executions per Map state, with tolerated failure thresholds).
- Uses `ItemReader` to read items from S3 or other sources directly, avoiding input size limits.
- Supports `ToleratedFailurePercentage` and `ToleratedFailureCount` to allow partial failures.

**Concurrency control:**
The `MaxConcurrency` parameter limits how many iterations run simultaneously:
- `0` = unlimited concurrency
- `1` = sequential processing
- Any positive integer = bounded parallelism

**Use cases:**
- Processing each record in a DynamoDB scan result
- Fan-out processing of S3 object lists
- Running the same workflow for multiple customers/tenants simultaneously

---

### 5. How do you implement idempotency in Step Functions workflows?

**Answer:**
Idempotency in Step Functions is important because retries, at-least-once delivery (Express Workflows), and manual re-executions can cause duplicate processing. Strategies include:

**1. Execution Name Deduplication (Standard Workflows):**
Each execution can be given a unique name. If you attempt to start an execution with the same name within 90 days, Step Functions returns the existing execution's ARN rather than starting a new one. This provides execution-level idempotency.

```python
sfn_client.start_execution(
    stateMachineArn='...',
    name='order-12345-payment',  # deterministic, unique name
    input=json.dumps(payload)
)
```

**2. Task-Level Idempotency:**
For Lambda functions called by Step Functions, use the `$$.Execution.Id` or a derived token as an idempotency key passed to the Lambda. Lambda Powertools provides an idempotency decorator that stores results in DynamoDB.

**3. Conditional Checks in Tasks:**
Before performing a write operation, check if it has already been applied (e.g., check a DynamoDB item's status field). Use `Choice` states to skip already-completed steps.

**4. Distributed Mode Map with Unique Item IDs:**
Assign unique IDs to each item being processed and track processing state externally (e.g., in DynamoDB) so re-processing a previously completed item is a no-op.

**5. Task Token Tracking:**
Store issued task tokens in DynamoDB. If a callback arrives for an already-completed token, discard it gracefully.

---

## Hard

### 1. Deep dive: How does Step Functions achieve exactly-once execution semantics in Standard Workflows, and what are its limitations?

**Answer:**
Step Functions Standard Workflows guarantee **exactly-once execution semantics at the state transition level**, not necessarily at the task execution level. Understanding this distinction is critical.

**How it works:**
- Step Functions maintains durable execution state in its own managed storage (not exposed to users). Each state transition is recorded before the next state begins.
- If a state machine node fails mid-execution (e.g., during a Lambda invocation), Step Functions will retry the state from the last known checkpoint, not from the beginning.
- The execution history is append-only and immutable, providing an audit trail.

**Critical limitation — Task execution vs. state transition:**
"Exactly-once" applies to state transitions, not to the downstream task invocations. Consider this scenario:
1. Step Functions invokes a Lambda function.
2. Lambda executes successfully and returns a response.
3. The network call returning the response to Step Functions fails.
4. Step Functions retries the state, invoking Lambda **again**.

This means Lambda (or any Task target) can be invoked **more than once** even in Standard Workflows. Your tasks must be idempotent.

**Contrast with Express Workflows:**
Express Workflows are explicitly at-least-once, meaning the same state can execute multiple times without any guarantee of deduplication. They prioritize throughput and cost over consistency.

**Practical implications:**
- Always design Task states to be idempotent.
- Use the task token or execution ID as an idempotency key.
- For financial transactions, implement compensating transactions (Saga pattern) rather than relying on Step Functions' exactly-once guarantee alone.
- DynamoDB conditional writes are a powerful tool for ensuring idempotency in downstream tasks.

---

### 2. Explain the Saga pattern implementation using Step Functions. How do you handle distributed transactions and compensating transactions?

**Answer:**
The Saga pattern addresses the challenge of maintaining data consistency across multiple microservices without using distributed transactions (2PC). Each step in the saga has a corresponding compensating transaction that undoes its effect if a later step fails.

**Implementation in Step Functions:**

**Forward path:** Each Task state performs one step of the business transaction (e.g., reserve inventory → charge payment → create shipment).

**Compensation path:** If any step fails, the `Catch` block triggers a compensation chain that undoes completed steps in reverse order.

```
[ReserveInventory] → [ChargePayment] → [CreateShipment]
       ↓ (fail)           ↓ (fail)          ↓ (fail)
[ReleaseInventory] ← [RefundPayment] ← [CancelShipment]
```

**ASL structure (simplified):**
```json
{
  "StartAt": "ReserveInventory",
  "States": {
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:::function:reserve-inventory",
      "ResultPath": "$.inventoryResult",
      "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "HandleInventoryFailure"}],
      "Next": "ChargePayment"
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:::function:charge-payment",
      "ResultPath": "$.paymentResult",
      "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "ReleaseInventory"}],
      "Next": "CreateShipment"
    },
    "ReleaseInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:::function:release-inventory",
      "Next": "SagaFailed"
    }
  }
}
```

**Key design considerations:**

1. **Compensating transactions must also be idempotent** — A compensation step might itself fail and be retried.

2. **Backward recovery vs. forward recovery:**
   - Backward: Undo all completed steps (most common).
   - Forward: Retry the failed step with corrected input (use `Retry` blocks).

3. **Tracking saga state:** Use `ResultPath` to accumulate results and pass them through the workflow so compensation steps have access to the IDs/tokens needed to undo work (e.g., payment transaction ID for refunds).

4. **Partial compensation failures:** If a compensation step itself fails, you need a dead-letter mechanism — typically an SNS/SQS notification to a human operator or a separate cleanup workflow.

5. **Isolation:** Sagas do not provide isolation between concurrent executions. If two sagas run simultaneously, they may see each other's intermediate states. Use optimistic locking in your data stores.

---

### 3. How does Step Functions integrate with EventBridge, and how would you architect an event-driven workflow system that handles back-pressure and fan-out?

**Answer:**
**EventBridge → Step Functions integration:**
EventBridge can trigger Step Functions executions as a target of an EventBridge rule. This enables event-driven workflow initiation without polling. Step Functions can also emit events to EventBridge upon execution state changes (started, succeeded, failed, timed out, aborted) by enabling execution logging with EventBridge as a destination.

**Architecture for event-driven workflows with back-pressure and fan-out:**

```
[Event Sources] → [EventBridge Bus] → [SQS Queue] → [Lambda Trigger] → [Step Functions]
                                                         ↑
                                               (concurrency limiter)
```

**Handling back-pressure:**

1. **SQS as buffer:** Route EventBridge events to an SQS queue instead of directly to Step Functions. A Lambda function reads from SQS and starts Step Functions executions. SQS provides natural buffering.

2. **Lambda concurrency limits:** Set a reserved concurrency on the trigger Lambda to control the rate of Step Functions execution starts (max 2,000 starts/sec for Standard Workflows).

3. **SQS visibility timeout alignment:** Set the SQS visibility timeout slightly longer than the expected Lambda execution time to prevent duplicate processing if Lambda times out before deleting the message.

4. **Step Functions execution throttling:** If `StartExecution` calls are throttled (TooManyRequestsException), implement exponential backoff in the trigger Lambda. SQS + Lambda's built