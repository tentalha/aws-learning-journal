# SQS — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Ensure the AWS CLI is installed and configured before running any SQS commands.

```bash
# Install AWS CLI (if not already installed)
pip install awscli --upgrade --user

# Configure AWS CLI with credentials and default region
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json

# Verify configuration
aws sts get-caller-identity
```

### Required IAM Permissions

Attach the following IAM policy to your user or role for full SQS access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:CreateQueue",
        "sqs:DeleteQueue",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl",
        "sqs:ListQueues",
        "sqs:ListQueueTags",
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:ChangeMessageVisibility",
        "sqs:PurgeQueue",
        "sqs:SetQueueAttributes",
        "sqs:TagQueue",
        "sqs:UntagQueue"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:*"
    }
  ]
}
```

### Useful Environment Variables

```bash
# Set a default SQS queue URL to avoid repeating it
export QUEUE_URL="https://sqs.us-east-1.amazonaws.com/123456789012/my-queue"

# Set default region
export AWS_DEFAULT_REGION="us-east-1"

# Set default output format
export AWS_DEFAULT_OUTPUT="json"
```

---

## Core Commands

### 1. Create a Standard Queue

```bash
aws sqs create-queue \
  --queue-name my-standard-queue \
  --attributes '{
    "VisibilityTimeout": "30",
    "MessageRetentionPeriod": "86400",
    "ReceiveMessageWaitTimeSeconds": "20"
  }'
```

**What it does:** Creates a new standard SQS queue with a 30-second visibility timeout, 1-day message retention, and long polling enabled (20 seconds).

**Example Output:**
```json
{
    "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue"
}
```

---

### 2. Create a FIFO Queue

```bash
aws sqs create-queue \
  --queue-name my-fifo-queue.fifo \
  --attributes '{
    "FifoQueue": "true",
    "ContentBasedDeduplication": "true",
    "VisibilityTimeout": "60"
  }'
```

**What it does:** Creates a FIFO (First-In-First-Out) queue that guarantees message ordering and exactly-once processing. FIFO queue names **must** end with `.fifo`.

**Example Output:**
```json
{
    "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123456789012/my-fifo-queue.fifo"
}
```

---

### 3. List All Queues

```bash
aws sqs list-queues
```

**What it does:** Lists all SQS queues in the current AWS account and region.

**Example Output:**
```json
{
    "QueueUrls": [
        "https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue",
        "https://sqs.us-east-1.amazonaws.com/123456789012/my-fifo-queue.fifo",
        "https://sqs.us-east-1.amazonaws.com/123456789012/my-dlq"
    ]
}
```

```bash
# Filter queues by name prefix
aws sqs list-queues --queue-name-prefix my-
```

---

### 4. Get Queue URL

```bash
aws sqs get-queue-url \
  --queue-name my-standard-queue
```

**What it does:** Retrieves the URL of an existing queue by name. Useful when you only know the queue name.

**Example Output:**
```json
{
    "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue"
}
```

---

### 5. Get Queue Attributes

```bash
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue \
  --attribute-names All
```

**What it does:** Retrieves all metadata and configuration attributes for a queue, including message counts, ARN, and policy settings.

**Example Output:**
```json
{
    "Attributes": {
        "QueueArn": "arn:aws:sqs:us-east-1:123456789012:my-standard-queue",
        "ApproximateNumberOfMessages": "5",
        "ApproximateNumberOfMessagesNotVisible": "2",
        "ApproximateNumberOfMessagesDelayed": "0",
        "CreatedTimestamp": "1700000000",
        "LastModifiedTimestamp": "1700001000",
        "VisibilityTimeout": "30",
        "MaximumMessageSize": "262144",
        "MessageRetentionPeriod": "86400",
        "DelaySeconds": "0",
        "ReceiveMessageWaitTimeSeconds": "20",
        "SqsManagedSseEnabled": "true"
    }
}
```

---

### 6. Send a Message

```bash
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue \
  --message-body '{"orderId": "ORD-001", "status": "pending", "amount": 99.99}' \
  --message-attributes '{
    "Source": {
      "DataType": "String",
      "StringValue": "order-service"
    },
    "Priority": {
      "DataType": "Number",
      "StringValue": "1"
    }
  }'
```

**What it does:** Sends a single message to the specified queue with optional custom message attributes for filtering or routing.

**Example Output:**
```json
{
    "MD5OfMessageBody": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
    "MD5OfMessageAttributes": "b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5",
    "MessageId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

---

### 7. Send a Message to a FIFO Queue

```bash
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-fifo-queue.fifo \
  --message-body '{"event": "user-signup", "userId": "USR-42"}' \
  --message-group-id "user-events" \
  --message-deduplication-id "USR-42-signup-20240101"
```

**What it does:** Sends a message to a FIFO queue. `--message-group-id` groups related messages for ordering; `--message-deduplication-id` prevents duplicate processing within a 5-minute window.

---

### 8. Receive Messages

```bash
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue \
  --max-number-of-messages 10 \
  --wait-time-seconds 20 \
  --visibility-timeout 60 \
  --message-attribute-names All \
  --attribute-names All
```

**What it does:** Polls the queue for up to 10 messages using long polling (20 seconds). Messages become invisible to other consumers for 60 seconds.

**Example Output:**
```json
{
    "Messages": [
        {
            "MessageId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
            "ReceiptHandle": "AQEBwJnKyrHigUMZj6reyuD...very-long-receipt-handle...",
            "MD5OfBody": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
            "Body": "{\"orderId\": \"ORD-001\", \"status\": \"pending\", \"amount\": 99.99}",
            "Attributes": {
                "SenderId": "AIDAIENQZJOLO23YVJ4VO",
                "SentTimestamp": "1700001234567",
                "ApproximateReceiveCount": "1",
                "ApproximateFirstReceiveTimestamp": "1700001300000"
            }
        }
    ]
}
```

---

### 9. Delete a Message

```bash
aws sqs delete-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue \
  --receipt-handle "AQEBwJnKyrHigUMZj6reyuD...very-long-receipt-handle..."
```

**What it does:** Permanently deletes a processed message from the queue using its receipt handle. This confirms successful processing.

---

### 10. Send Messages in Batch

```bash
aws sqs send-message-batch \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue \
  --entries '[
    {
      "Id": "msg-1",
      "MessageBody": "{\"event\": \"order-created\", \"orderId\": \"ORD-101\"}"
    },
    {
      "Id": "msg-2",
      "MessageBody": "{\"event\": \"order-created\", \"orderId\": \"ORD-102\"}",
      "DelaySeconds": 10
    },
    {
      "Id": "msg-3",
      "MessageBody": "{\"event\": \"order-created\", \"orderId\": \"ORD-103\"}"
    }
  ]'
```

**What it does:** Sends up to 10 messages in a single API call, reducing costs and improving throughput. Each entry requires a unique `Id`.

**Example Output:**
```json
{
    "Successful": [
        {"Id": "msg-1", "MessageId": "uuid-1", "MD5OfMessageBody": "abc123"},
        {"Id": "msg-2", "MessageId": "uuid-2", "MD5OfMessageBody": "def456"},
        {"Id": "msg-3", "MessageId": "uuid-3", "MD5OfMessageBody": "ghi789"}
    ],
    "Failed": []
}
```

---

### 11. Delete Messages in Batch

```bash
aws sqs delete-message-batch \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue \
  --entries '[
    {"Id": "msg-1", "ReceiptHandle": "AQEBwJnKyrH...receipt-handle-1..."},
    {"Id": "msg-2", "ReceiptHandle": "AQEBxKoLzsI...receipt-handle-2..."},
    {"Id": "msg-3", "ReceiptHandle": "AQEByLpMatJ...receipt-handle-3..."}
  ]'
```

**What it does:** Deletes up to 10 messages in a single API call. More efficient than individual deletes when processing messages in bulk.

---

### 12. Set Queue Attributes

```bash
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue \
  --attributes '{
    "VisibilityTimeout": "120",
    "MessageRetentionPeriod": "345600",
    "MaximumMessageSize": "65536"
  }'
```

**What it does:** Updates configuration attributes on an existing queue. Here it sets a 2-minute visibility timeout, 4-day retention, and 64KB max message size.

---

### 13. Purge a Queue

```bash
aws sqs purge-queue \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue
```

**What it does:** Deletes **all messages** in the queue immediately. Useful for clearing test data. Note: Can only be called once every 60 seconds per queue.

> ⚠️ **Warning:** This action is irreversible. All messages will be permanently deleted.

---

### 14. Delete a Queue

```bash
aws sqs delete-queue \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue
```

**What it does:** Permanently deletes the queue and all its messages. The queue name may be reused after 60 seconds.

---

### 15. Tag a Queue

```bash
aws sqs tag-queue \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-standard-queue \
  --tags '{
    "Environment": "production",
    "Team": "platform",
    "CostCenter": "CC-1234",
    "Application": "order-processing"
  }'
```

**What it does:** Adds or updates resource tags on a queue for cost allocation, access control, and organizational purposes.

---

## Common Operations

### CREATE

```bash
# Create a standard queue with dead-letter queue (DLQ) redrive policy
# Step 1: Create the DLQ first
aws sqs create-queue \
  --queue-name my-app-dlq \
  --attributes '{"MessageRetentionPeriod": "1209600"}'

# Step 2: Get the DLQ ARN
DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-app-dlq \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)

# Step 3: Create the main queue with DLQ redrive policy
aws sqs create-queue \
  --queue-name my-app-queue \
  --attributes "{
    \"VisibilityTimeout\": \"30\",
    \"MessageRetentionPeriod\": \"86400\",
    \"ReceiveMessageWaitTimeSeconds\": \"20\",
    \"RedrivePolicy\": \"{\\\"deadLetterTargetArn\\\":\\\"${DLQ_ARN}\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\"
  }"

# Create an encrypted queue using SSE-SQS
aws sqs create-queue \
  --queue-name my-encrypted-queue \
  --attributes '{"SqsManagedSseEnabled": "true"}'

# Create a queue encrypted with a custom KMS key
aws sqs create-queue \
  --queue-name my-kms-queue \
  --attributes '{
    "KmsMasterKeyId": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123def456",
    "KmsDataKeyReusePeriodSeconds": "300"
  }'
```

---

### READ

```bash
# Get specific attributes (not all)
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-standard