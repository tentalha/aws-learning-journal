# Lambda — Interview Questions

---

## Easy

### 1. What is AWS Lambda, and what is its primary use case?

**Answer:**
AWS Lambda is a serverless, event-driven compute service that lets you run code without provisioning or managing servers. You upload your code, and Lambda automatically handles the compute infrastructure. Its primary use cases include:
- Running backend logic in response to events (e.g., S3 uploads, API Gateway requests, DynamoDB streams)
- Building microservices and APIs
- Data processing and transformation pipelines
- Scheduled tasks (via EventBridge/CloudWatch Events)

Lambda charges only for the compute time consumed (measured in milliseconds), making it cost-effective for workloads with variable or unpredictable traffic.

---

### 2. What runtimes does AWS Lambda support?

**Answer:**
Lambda supports the following managed runtimes:
- **Node.js** (e.g., 18.x, 20.x)
- **Python** (e.g., 3.11, 3.12)
- **Java** (e.g., 11, 17, 21)
- **Ruby** (e.g., 3.2)
- **.NET / C#** (e.g., .NET 8)
- **Go** (via `provided.al2023`)
- **Rust** (via `provided.al2023`)

Additionally, Lambda supports **custom runtimes** using the `provided.al2` or `provided.al2023` base images, allowing you to run virtually any language. You can also use **Lambda container images** (up to 10 GB) to package your function with a custom runtime environment.

---

### 3. What are the key limits/quotas for AWS Lambda?

**Answer:**
Key default and hard limits include:

| Resource | Limit |
|---|---|
| Maximum execution timeout | 15 minutes |
| Maximum memory allocation | 10,240 MB (10 GB) |
| Ephemeral storage (`/tmp`) | 512 MB – 10,240 MB |
| Deployment package size (zip, direct upload) | 50 MB (compressed) / 250 MB (uncompressed) |
| Container image size | 10 GB |
| Concurrency (default per region) | 1,000 (soft limit, can be increased) |
| Environment variable size | 4 KB total |
| Function layers | Up to 5 layers, 250 MB total unzipped |

Understanding these limits is critical for designing Lambda-based solutions appropriately.

---

### 4. What is the difference between synchronous and asynchronous Lambda invocation?

**Answer:**

| Aspect | Synchronous | Asynchronous |
|---|---|---|
| **Caller behavior** | Waits for function to complete and return response | Does not wait; Lambda queues the event |
| **Error handling** | Caller receives error immediately | Lambda retries up to 2 times (configurable) |
| **Sources** | API Gateway, ALB, Cognito, SDK direct calls | S3, SNS, EventBridge, SES |
| **Response** | Function result returned to caller | No direct response to caller |
| **Dead Letter Queue** | Not applicable | Supported (SQS or SNS) |
| **Event Age** | N/A | Max event age configurable (up to 6 hours) |

For asynchronous invocations, Lambda has a built-in internal event queue and manages retries automatically. You can configure **Destinations** (success/failure) to route results to SQS, SNS, EventBridge, or another Lambda function.

---

### 5. What is a Lambda execution role, and why is it important?

**Answer:**
A Lambda execution role is an **IAM role** that Lambda assumes when your function is invoked. It defines what AWS resources and actions the function is permitted to access. It is important because:

- **Principle of Least Privilege:** The role should grant only the minimum permissions needed (e.g., read from a specific S3 bucket, write to a specific DynamoDB table).
- **Security Boundary:** Without the correct permissions, the function cannot access other AWS services, preventing accidental or malicious data access.
- **CloudWatch Logs:** The role must include `logs:CreateLogGroup`, `logs:CreateLogStream`, and `logs:PutLogEvents` to allow Lambda to write logs.
- **VPC Access:** If the function runs in a VPC, the role needs `ec2:CreateNetworkInterface`, `ec2:DescribeNetworkInterfaces`, and `ec2:DeleteNetworkInterface`.

AWS provides the managed policy `AWSLambdaBasicExecutionRole` as a starting point, but production roles should be narrowly scoped.

---

## Medium

### 1. Explain Lambda cold starts: what causes them, how they impact performance, and how you can mitigate them.

**Answer:**
**What is a cold start?**
A cold start occurs when Lambda needs to initialize a new execution environment to handle an invocation. This happens when:
- There is no existing idle execution environment available (first invocation, after a period of inactivity, or when scaling out to handle concurrent requests)
- The function code or configuration has been updated

**Phases of a cold start:**
1. **Environment provisioning** — Lambda allocates a micro-VM (using Firecracker)
2. **Runtime initialization** — The language runtime is bootstrapped
3. **Function initialization** — Your initialization code outside the handler runs (imports, SDK clients, DB connections)

Cold starts typically add **100ms to several seconds** of latency, depending on the runtime, package size, and initialization logic. Java and .NET runtimes tend to have longer cold starts than Python or Node.js.

**Mitigation strategies:**

| Strategy | Description |
|---|---|
| **Provisioned Concurrency** | Pre-warms a specified number of execution environments, eliminating cold starts for those instances |
| **Keep functions warm** | Use EventBridge to ping functions every few minutes (less reliable, not recommended for production) |
| **Minimize package size** | Smaller packages initialize faster; use tree-shaking, remove unused dependencies |
| **Move initialization outside the handler** | SDK clients, DB connections, and configuration should be initialized once and reused |
| **Choose a faster runtime** | Python and Node.js have lower cold start overhead than Java/.NET |
| **Use Lambda SnapStart (Java)** | Takes a snapshot of the initialized execution environment for Java 11+ functions |
| **Reduce VPC attachment** | VPC-attached functions had historically longer cold starts; AWS has improved this with pre-provisioned ENIs |

**Lambda SnapStart** (for Java Corretto 11+) is particularly powerful — Lambda takes a snapshot of the memory and disk state after initialization and restores it on subsequent invocations, dramatically reducing cold start time.

---

### 2. How does Lambda handle concurrency, and what is the difference between reserved and provisioned concurrency?

**Answer:**
**Concurrency basics:**
Lambda concurrency is the number of function instances handling requests simultaneously. By default, each invocation occupies one concurrent execution. The account-level concurrency limit is 1,000 per region (soft limit).

```
Concurrent executions = (Invocations per second) × (Average duration in seconds)
```

**Types of concurrency:**

**1. Unreserved Concurrency**
- The default pool shared across all functions in the account/region
- A single function can consume all available concurrency, potentially throttling others
- No guarantees on availability

**2. Reserved Concurrency**
- Sets a **maximum** concurrent execution limit for a specific function
- Guarantees the function never exceeds the limit (throttles beyond it)
- Also guarantees the function always has that capacity available from the account pool
- **Cost:** No additional charge, but the reserved capacity is subtracted from the account pool
- **Use case:** Prevent a runaway function from consuming all account concurrency; protect downstream systems from being overwhelmed

**3. Provisioned Concurrency**
- Pre-initializes a specified number of execution environments
- Eliminates cold starts for those instances
- **Cost:** Charged per provisioned concurrency-hour (even when idle)
- **Use case:** Latency-sensitive applications (e.g., real-time APIs, interactive user-facing features)
- Can be configured with **Application Auto Scaling** to scale provisioned concurrency based on schedules or utilization metrics

**Key distinction:**
- Reserved concurrency = **ceiling** (limits max scale)
- Provisioned concurrency = **floor** (guarantees warm instances)

---

### 3. What are Lambda Layers, and when should you use them?

**Answer:**
**What are Lambda Layers?**
A Lambda Layer is a `.zip` archive containing libraries, custom runtimes, data, or configuration files that can be shared across multiple Lambda functions. Layers are extracted to `/opt` in the function's execution environment.

**Benefits:**
- **Code reuse:** Share common libraries (e.g., database drivers, utility functions, AWS SDK extensions) across functions without packaging them in every deployment artifact
- **Smaller deployment packages:** Functions only contain business logic; shared dependencies live in layers
- **Separation of concerns:** Infrastructure teams can manage layers (e.g., security patches to shared libraries) independently of function code
- **Faster deployments:** Smaller function packages upload and deploy faster

**Limits:**
- Up to **5 layers** per function
- Total unzipped size of function + all layers ≤ **250 MB**
- Layers are versioned and immutable (each update creates a new version)

**When to use layers:**
- Shared internal libraries used by many functions
- Large ML model files or data files
- Custom runtimes (e.g., PHP, Perl)
- Third-party monitoring agents (e.g., Datadog, New Relic Lambda extension layers)
- AWS-provided layers (e.g., AWS SDK for Pandas / `AWSSDKPandas-Python39`)

**When NOT to use layers:**
- If only one function uses the dependency (just bundle it)
- For very large dependencies that exceed the 250 MB limit (use container images instead)
- When you need different versions of the same dependency per function (layers are shared and versioned)

---

### 4. How does Lambda integrate with Amazon SQS, and what are the key configuration parameters for this event source mapping?

**Answer:**
Lambda can poll an SQS queue and invoke your function with a batch of messages. This is an **event source mapping** — Lambda manages the polling infrastructure.

**How it works:**
1. Lambda's internal poller continuously long-polls the SQS queue
2. When messages are available, Lambda invokes your function with a batch
3. If the function succeeds, Lambda deletes the messages from the queue
4. If the function fails (throws an exception), Lambda does NOT delete the messages; they become visible again after the visibility timeout and are retried

**Key configuration parameters:**

| Parameter | Description | Recommended Setting |
|---|---|---|
| **Batch Size** | Number of messages per invocation (1–10,000 for Standard; 1–10 for FIFO) | Tune based on processing time |
| **Batch Window** | Max time Lambda waits to gather a full batch (0–300 seconds) | Use for cost optimization with low-traffic queues |
| **Maximum Concurrency** | Limits concurrent Lambda invocations from this event source | Protect downstream systems |
| **Bisect on Error** | Splits a failed batch in half to isolate bad records | Enable for debugging |
| **Maximum Retry Attempts** | How many times to retry a failed batch | Depends on idempotency |
| **Destination on Failure** | SQS DLQ or SNS topic for failed batches | Always configure |
| **Report Batch Item Failures** | Allows partial batch success (return `batchItemFailures`) | Highly recommended |

**Partial batch failure handling:**
By returning `batchItemFailures` in the response, Lambda only retries the failed messages, not the entire batch. This prevents successfully processed messages from being reprocessed.

```python
def handler(event, context):
    batch_item_failures = []
    for record in event['Records']:
        try:
            process(record)
        except Exception:
            batch_item_failures.append({"itemIdentifier": record['messageId']})
    return {"batchItemFailures": batch_item_failures}
```

**FIFO vs Standard SQS:**
- **Standard:** Lambda scales up to 1,000 concurrent function instances per queue
- **FIFO:** Lambda processes one message group at a time (preserves order); concurrency = number of active message group IDs

---

### 5. What is the Lambda execution environment lifecycle, and how can you optimize it for performance?

**Answer:**
**Execution environment lifecycle:**

```
INIT Phase → INVOKE Phase → SHUTDOWN Phase
```

**1. INIT Phase (cold start only)**
- **Extension init:** Lambda extensions are initialized
- **Runtime init:** The language runtime is bootstrapped
- **Function init:** Code outside the handler runs (global scope)
- Duration: Billed as part of the first invocation (unless using Provisioned Concurrency, where it's billed separately)

**2. INVOKE Phase**
- The handler function is called
- The execution environment is frozen after the handler returns
- Subsequent invocations reuse the same environment (warm start)
- Duration: Billed per invocation

**3. SHUTDOWN Phase**
- Lambda sends a `SHUTDOWN` event to registered extensions
- Environment is terminated after a period of inactivity (typically ~15 minutes for warm environments)

**Optimization strategies leveraging the lifecycle:**

**Reuse initialization across invocations:**
```python
import boto3

# Initialized ONCE during INIT phase, reused across invocations
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('MyTable')

def handler(event, context):
    # Table client is already initialized; no overhead here
    response = table.get_item(Key={'id': event['id']})
    return response['Item']
```

**Connection pooling:**
- Initialize database connection pools outside the handler
- Be aware that connections may become stale if the environment is frozen for a long time — implement reconnection logic

**Caching:**
- Cache frequently accessed data (e.g., SSM parameters, Secrets Manager values) in global variables
- Add a TTL to refresh cached values periodically

**Lazy initialization:**
- For rarely used code paths, initialize dependencies only when needed to avoid slowing the INIT phase

**`/tmp` storage:**
- `/tmp` (512 MB–10 GB) persists across invocations within the same execution environment
- Use it to cache downloaded files, model artifacts, or compiled assets

---

## Hard

### 1. Deep dive: How does Lambda scale internally, and what are the nuances of scaling behavior that can cause throttling in production?

**Answer:**
**Internal scaling mechanism:**
Lambda uses a concept called **burst scaling** followed by **linear scaling**:

**Burst limits (initial scaling):**
Lambda can scale up to a burst limit per region immediately:
- **US East (N. Virginia), US West (Oregon), EU (Ireland):** 3,000 concurrent executions
- **Other regions:** 500–1,000 concurrent executions

After the burst limit is reached, Lambda scales at **500 additional concurrent executions per minute** until the account concurrency limit is reached.

**Throttling scenarios and root causes:**

**1. Account-level concurrency exhaustion**
All functions in the region share the 1,000 default limit. A single high-traffic function can exhaust the pool, causing `TooManyRequestsException` (HTTP 429) for all other functions.

*Solution:* Set reserved concurrency on critical functions; request a limit increase via Service Quotas.

**2. Reserved concurrency throttling**
If a function's reserved concurrency is set to 10 and 11 concurrent requests arrive, the 11th is throttled.

*Behavior by invocation type:*
- **Synchronous:** Returns 429 to the caller
- **Asynchronous:** Lambda retries for up to 6 hours (configurable)
- **SQS Event Source Mapping:** Messages remain in the queue and are retried; Lambda reduces polling rate

**3. Burst limit throttling**
During a sudden traffic spike that exceeds the burst limit, Lambda returns 429 until it can scale out.

*Solution:* Use SQS as a buffer to absorb traffic spikes; implement exponential backoff in callers.

**4. VPC ENI limits**
Lambda functions in a VPC share ENIs via Hyperplane (since 2020). However, each subnet has a maximum number of ENIs based on the subnet's available IP addresses. If all IPs are consumed, Lambda cannot scale further.

*Solution:* Use large subnets (at least /24 or larger) for Lambda; monitor `ENILimitExceeded` errors.

**5. Function-level throttling with SQS**
When Lambda is throttled while processing SQS messages, it implements an exponential backoff on polling. Messages remain in the queue but are not deleted. If the visibility timeout expires before Lambda can process them, messages become visible again and could be processed multiple times.

*Solution:* Set the SQS visibility timeout to at least 6