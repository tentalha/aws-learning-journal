# Lambda

## What is it?

**AWS Lambda** is a serverless, event-driven compute service that lets you run code without provisioning or managing servers. It belongs to the **Compute** category of AWS services.

Lambda automatically manages the underlying compute infrastructure, including server and operating system maintenance, capacity provisioning, automatic scaling, and logging. You upload your code (called a **function**), define a trigger, and Lambda handles the rest.

- **Official Name:** AWS Lambda
- **Category:** Serverless Compute
- **Runtime Support:** Node.js, Python, Java, Go, Ruby, .NET (C#), and custom runtimes via Lambda Layers
- **Execution Model:** Function-as-a-Service (FaaS)
- **Billing Unit:** Per invocation + duration (GB-seconds)

---

## Why do we need it?

### Problems It Solves

Traditional server-based architectures require you to:
- Provision and manage EC2 instances or containers
- Pay for idle compute time
- Handle scaling logic manually
- Manage OS patches, security updates, and runtime environments

Lambda eliminates all of this overhead.

### When to Use Lambda

| Scenario | Why Lambda Fits |
|---|---|
| Sporadic, unpredictable traffic | Pay only when code runs; scales to zero |
| Event-driven processing | Native integration with 200+ AWS services |
| Short-lived tasks (< 15 min) | Designed for burst, ephemeral workloads |
| Microservices backends | Small, single-responsibility functions |
| Data transformation pipelines | Process S3 objects, Kinesis streams, DynamoDB streams |
| Scheduled jobs (cron) | EventBridge Scheduler triggers |
| Real-time file/stream processing | S3 events, Kinesis, DynamoDB Streams |

### Real Business Scenarios

1. **E-commerce:** A user uploads a product image → Lambda resizes it into multiple dimensions and stores them in S3.
2. **FinTech:** Every transaction written to DynamoDB triggers Lambda to perform fraud analysis in real time.
3. **SaaS:** API Gateway + Lambda serves as the backend for a REST API, scaling from 0 to millions of requests with no infrastructure changes.
4. **IoT:** Sensor data arriving via MQTT/Kinesis is processed by Lambda to detect anomalies.

---

## Internal Working

### Execution Environment Lifecycle

Lambda's internal execution model follows a well-defined lifecycle:

```
┌─────────────────────────────────────────────────────────┐
│                  Lambda Execution Lifecycle              │
│                                                          │
│  1. INIT Phase                                           │
│     ├── Download function code & layers                  │
│     ├── Start runtime (e.g., Node.js process)           │
│     └── Run initialization code (outside handler)       │
│                                                          │
│  2. INVOKE Phase                                         │
│     ├── Execute handler function                         │
│     ├── Return response / write to output               │
│     └── Function completes                               │
│                                                          │
│  3. SHUTDOWN Phase (after idle timeout)                  │
│     ├── Runtime receives SIGTERM                         │
│     └── Execution environment is frozen/destroyed       │
└─────────────────────────────────────────────────────────┘
```

### Cold Start vs. Warm Start

**Cold Start** occurs when Lambda must:
1. Allocate a new execution environment (microVM via Firecracker)
2. Download the function package
3. Initialize the runtime
4. Run the initialization code

**Warm Start** occurs when an existing execution environment is reused:
- The handler is invoked directly
- Connections, caches, and global variables persist

```
Cold Start Timeline:
[Container Alloc] → [Code Download] → [Runtime Init] → [Handler Init] → [Handler Exec]
     ~100ms              ~50ms             ~200ms            ~Xms            ~Xms

Warm Start Timeline:
[Handler Exec]
    ~Xms
```

### Firecracker MicroVM

Lambda uses **AWS Firecracker**, an open-source virtualization technology:
- Each execution environment runs in an isolated microVM
- Boots in ~125 milliseconds
- Provides hardware-level isolation between tenants
- Uses Linux KVM for virtualization

### Execution Environment Reuse

- AWS maintains a pool of warm execution environments
- After function completion, the environment is "frozen" (not destroyed immediately)
- Subsequent invocations may reuse the same environment (warm start)
- AWS does not guarantee environment reuse — your code must be stateless

### Concurrency Model

```
                    ┌─────────────────────────────┐
 Invocation 1 ───►  │  Execution Environment #1   │
 Invocation 2 ───►  │  Execution Environment #2   │
 Invocation 3 ───►  │  Execution Environment #3   │
                    └─────────────────────────────┘
```

Each concurrent invocation gets its own isolated execution environment. Lambda does **not** share environments between concurrent requests.

---

## Architecture

### Core Components

```
┌──────────────────────────────────────────────────────────────────┐
│                         AWS Lambda Architecture                   │
│                                                                    │
│  ┌──────────┐    ┌───────────────┐    ┌──────────────────────┐   │
│  │ Trigger  │───►│  Event Source │───►│   Lambda Function    │   │
│  │ (Event)  │    │   Mapping     │    │                      │   │
│  └──────────┘    └───────────────┘    │  ┌────────────────┐  │   │
│                                       │  │  Handler Code  │  │   │
│  ┌──────────┐                         │  └────────────────┘  │   │
│  │  IAM     │──────────────────────►  │  ┌────────────────┐  │   │
│  │  Role    │                         │  │   Layers       │  │   │
│  └──────────┘                         │  └────────────────┘  │   │
│                                       │  ┌────────────────┐  │   │
│  ┌──────────┐                         │  │  Environment   │  │   │
│  │  VPC     │──────────────────────►  │  │  Variables     │  │   │
│  │  Config  │                         │  └────────────────┘  │   │
│  └──────────┘                         └──────────────────────┘   │
│                                                  │                │
│                                                  ▼                │
│                                       ┌──────────────────────┐   │
│                                       │   CloudWatch Logs    │   │
│                                       │   X-Ray Traces       │   │
│                                       └──────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

| Component | Description |
|---|---|
| **Function** | Your code + configuration (runtime, memory, timeout) |
| **Handler** | Entry point method Lambda calls when invoked |
| **Execution Role** | IAM role granting Lambda permissions to AWS resources |
| **Event Source Mapping** | Configuration linking event sources (SQS, Kinesis) to Lambda |
| **Layers** | Shared libraries/dependencies packaged separately |
| **Aliases** | Named pointers to function versions (e.g., `prod`, `staging`) |
| **Versions** | Immutable snapshots of function code + configuration |
| **Destinations** | Route async invocation results to SQS, SNS, EventBridge, Lambda |
| **Extensions** | Augment Lambda with monitoring, security, governance tools |

### Invocation Types

| Type | Description | Use Case |
|---|---|---|
| **Synchronous** | Caller waits for response | API Gateway, SDK direct calls |
| **Asynchronous** | Lambda queues event, returns immediately | S3 events, SNS, EventBridge |
| **Event Source Mapping** | Lambda polls source | SQS, Kinesis, DynamoDB Streams, MSK |

### Architectural Patterns

1. **API Backend Pattern:** `Client → API Gateway → Lambda → DynamoDB`
2. **Fan-Out Pattern:** `SNS → Multiple Lambda functions (parallel processing)`
3. **Stream Processing:** `Kinesis → Lambda (batch processing)`
4. **Orchestration:** `Step Functions → Lambda (workflow)`
5. **Strangler Fig:** Incrementally migrate monolith to Lambda-based microservices

---

## Real World Example

### Scenario: Serverless Image Processing Pipeline

**Business Need:** An e-commerce platform needs to automatically resize product images uploaded by vendors into thumbnail (100x100), medium (400x400), and large (800x800) formats.

#### Step-by-Step Walkthrough

**Step 1: Vendor uploads original image**
```
Vendor → PUT /products/images/product-001.jpg → S3 Bucket (raw-images-bucket)
```

**Step 2: S3 Event Notification triggers Lambda**
```json
{
  "Records": [{
    "eventSource": "aws:s3",
    "eventName": "ObjectCreated:Put",
    "s3": {
      "bucket": { "name": "raw-images-bucket" },
      "object": { "key": "products/product-001.jpg", "size": 2048000 }
    }
  }]
}
```

**Step 3: Lambda function processes the image**
```python
import boto3
import json
from PIL import Image
import io

s3 = boto3.client('s3')
DEST_BUCKET = 'processed-images-bucket'
SIZES = {
    'thumbnail': (100, 100),
    'medium': (400, 400),
    'large': (800, 800)
}

def handler(event, context):
    record = event['Records'][0]
    src_bucket = record['s3']['bucket']['name']
    src_key = record['s3']['object']['key']
    
    # Download original image
    response = s3.get_object(Bucket=src_bucket, Key=src_key)
    image_data = response['Body'].read()
    image = Image.open(io.BytesIO(image_data))
    
    # Generate resized versions
    for size_name, dimensions in SIZES.items():
        resized = image.copy()
        resized.thumbnail(dimensions)
        
        buffer = io.BytesIO()
        resized.save(buffer, format='JPEG', quality=85)
        buffer.seek(0)
        
        dest_key = f"{size_name}/{src_key}"
        s3.put_object(
            Bucket=DEST_BUCKET,
            Key=dest_key,
            Body=buffer,
            ContentType='image/jpeg'
        )
        print(f"Uploaded {size_name} version: {dest_key}")
    
    return {'statusCode': 200, 'processed': src_key}
```

**Step 4: Processed images stored in destination bucket**
```
processed-images-bucket/
  ├── thumbnail/products/product-001.jpg
  ├── medium/products/product-001.jpg
  └── large/products/product-001.jpg
```

**Step 5: CloudFront serves images globally**
```
User Request → CloudFront → processed-images-bucket → Cached Response
```

**Step 6: Lambda sends success notification via SNS**
```
Lambda → SNS Topic → Email/SQS → Vendor Dashboard Update
```

#### Configuration Details
- **Runtime:** Python 3.12
- **Memory:** 1024 MB (image processing is memory-intensive)
- **Timeout:** 60 seconds
- **Layer:** Pillow library packaged as Lambda Layer
- **IAM Role:** Read from `raw-images-bucket`, Write to `processed-images-bucket`

---

## Advantages

1. **No Server Management:** Zero infrastructure provisioning, patching, or maintenance.

2. **Automatic Scaling:** Scales from 0 to thousands of concurrent executions automatically. No pre-warming required (Provisioned Concurrency aside).

3. **Pay-Per-Use:** Billed only for actual compute time in 1ms increments. No charges when idle.

4. **High Availability:** Lambda runs across multiple AZs by default. No HA configuration needed.

5. **Native AWS Integration:** First-class integration with 200+ AWS services as event sources or destinations.

6. **Multiple Runtime Support:** Node.js, Python, Java, Go, Ruby, .NET, and custom runtimes (bring your own container).

7. **Container Image Support:** Deploy Lambda functions as container images up to 10 GB.

8. **Built-in Fault Tolerance:** Lambda maintains compute capacity across multiple AZs.

9. **Versioning & Aliases:** Blue/green deployments and traffic shifting with weighted aliases.

10. **Lambda SnapStart (Java):** Reduces cold start latency for Java functions by up to 90% using snapshot caching.

11. **Graviton2 Support:** Run functions on ARM64 architecture for up to 34% better price-performance.

12. **Extensions API:** Integrate third-party monitoring, security, and observability tools without modifying function code.

---

## Limitations

### Hard Limits (Service Quotas)

| Limit | Value |
|---|---|
| Maximum execution timeout | **15 minutes** |
| Maximum memory | **10,240 MB (10 GB)** |
| Minimum memory | **128 MB** |
| Ephemeral storage (`/tmp`) | **512 MB – 10,240 MB** |
| Deployment package size (zip) | **50 MB (compressed), 250 MB (uncompressed)** |
| Container image size | **10 GB** |
| Environment variables size | **4 KB total** |
| Concurrent executions (default per region) | **1,000** (soft limit, can be increased) |
| Burst concurrency limit | **500–3,000** (varies by region) |
| Layers per function | **5** |
| Function payload (sync) | **6 MB (request + response)** |
| Function payload (async) | **256 KB** |
| `/tmp` storage | **512 MB – 10 GB** |
| VPC ENI limits | Shared across functions in VPC |

### Architectural Limitations

- **Stateless by design:** Cannot maintain in-memory state between invocations (use ElastiCache, DynamoDB)
- **Cold starts:** Latency penalty on first invocation (especially Java/.NET)
- **Not suitable for long-running tasks:** 15-minute max timeout
- **Limited local storage:** `/tmp` is ephemeral and not shared between environments
- **Vendor lock-in:** Lambda-specific event models and IAM integration
- **Debugging complexity:** Distributed tracing required for end-to-end visibility
- **Network egress costs:** Lambda in VPC still incurs NAT Gateway costs for internet access

---

## Best Practices

### Code & Design

1. **Keep functions small and single-purpose.** Follow the Single Responsibility Principle — one function, one task.

2. **Move initialization code outside the handler.** SDK clients, DB connections, and configuration loading should be initialized once at the module level, not inside the handler.

```javascript
// ✅ GOOD - initialized once per execution environment
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const client = new DynamoDBClient({ region: 'us-east-1' });

exports.handler = async (event) => {
  // client is reused on warm invocations
};

// ❌ BAD - re-initialized on every invocation
exports.handler = async (event) => {
  const client = new DynamoDBClient({ region: 'us-east-1' });
};
```

3. **Use environment variables for configuration.** Never hardcode endpoints, table names, or feature flags.

4. **Implement idempotency.** Lambda may invoke your function more than once (especially async). Use idempotency keys or check-before-write patterns.

5. **Use Lambda Powertools.** AWS Lambda Powertools (Python/TypeScript/Java) provides structured logging, tracing, metrics, and idempotency utilities.

### Performance

6. **Right-size memory allocation.** More memory = more CPU. Use AWS Lambda Power Tuning (Step Functions tool) to find the optimal memory setting.

7. **Use Provisioned Conc