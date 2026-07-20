# SQS — Hands-On Labs

## Lab 1: Getting Started with SQS

### Objective
In this lab, you will create your first Amazon SQS queue, send messages to it, receive and process those messages, and delete them. By the end, you will understand the core SQS lifecycle: **produce → queue → consume → delete**. You will also explore the difference between Standard and FIFO queues.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account with console access |
| IAM Permissions | `sqs:CreateQueue`, `sqs:SendMessage`, `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes`, `sqs:DeleteQueue` |
| Tools | AWS CLI v2 installed and configured (`aws configure`) |
| Region | `us-east-1` (or your preferred region) |

---

### Steps

#### Step 1: Create a Standard SQS Queue

**Console:**
1. Navigate to **Services → SQS** in the AWS Console.
2. Click **Create queue**.
3. Select **Standard** queue type.
4. Set the queue name to `lab1-orders-queue`.
5. Leave all other settings as default.
6. Click **Create queue**.

**CLI:**
```bash
aws sqs create-queue \
  --queue-name lab1-orders-queue \
  --region us-east-1
```

**Expected Output:**
```json
{
    "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123456789012/lab1-orders-queue"
}
```

> ✅ **Verify:** In the Console, the queue appears in the SQS queue list with status **Active**. Save the Queue URL — you'll need it in subsequent steps.

---

#### Step 2: Inspect Queue Attributes

**Console:**
1. Click on `lab1-orders-queue` in the queue list.
2. Review the **Details** tab — note the **ARN**, **URL**, and default settings.
3. Note the default values:
   - Visibility timeout: **30 seconds**
   - Message retention: **4 days**
   - Maximum message size: **256 KB**

**CLI:**
```bash
# Store the Queue URL in a variable
QUEUE_URL=$(aws sqs get-queue-url \
  --queue-name lab1-orders-queue \
  --query 'QueueUrl' \
  --output text \
  --region us-east-1)

echo "Queue URL: $QUEUE_URL"

# Get all attributes
aws sqs get-queue-attributes \
  --queue-url $QUEUE_URL \
  --attribute-names All \
  --region us-east-1
```

**Expected Output:**
```json
{
    "Attributes": {
        "QueueArn": "arn:aws:sqs:us-east-1:123456789012:lab1-orders-queue",
        "ApproximateNumberOfMessages": "0",
        "ApproximateNumberOfMessagesNotVisible": "0",
        "ApproximateNumberOfMessagesDelayed": "0",
        "CreatedTimestamp": "1700000000",
        "LastModifiedTimestamp": "1700000000",
        "VisibilityTimeout": "30",
        "MaximumMessageSize": "262144",
        "MessageRetentionPeriod": "345600",
        "DelaySeconds": "0",
        "ReceiveMessageWaitTimeSeconds": "0"
    }
}
```

> ✅ **Verify:** All attributes are returned. Note `ApproximateNumberOfMessages` is `0` — the queue is empty.

---

#### Step 3: Send Messages to the Queue

**Console:**
1. Click on `lab1-orders-queue`.
2. Click **Send and receive messages**.
3. In the **Message body** field, enter:
   ```json
   {"orderId": "ORD-001", "product": "Widget A", "quantity": 5}
   ```
4. Click **Send message**.
5. Repeat for two more messages:
   - `{"orderId": "ORD-002", "product": "Widget B", "quantity": 2}`
   - `{"orderId": "ORD-003", "product": "Widget C", "quantity": 10}`

**CLI:**
```bash
# Send first message
aws sqs send-message \
  --queue-url $QUEUE_URL \
  --message-body '{"orderId": "ORD-001", "product": "Widget A", "quantity": 5}' \
  --region us-east-1

# Send second message
aws sqs send-message \
  --queue-url $QUEUE_URL \
  --message-body '{"orderId": "ORD-002", "product": "Widget B", "quantity": 2}' \
  --region us-east-1

# Send third message
aws sqs send-message \
  --queue-url $QUEUE_URL \
  --message-body '{"orderId": "ORD-003", "product": "Widget C", "quantity": 10}' \
  --region us-east-1
```

**Expected Output (per message):**
```json
{
    "MD5OfMessageBody": "a1b2c3d4e5f6...",
    "MessageId": "12345678-1234-1234-1234-123456789012"
}
```

> ✅ **Verify:** Run the following and confirm `ApproximateNumberOfMessages` shows `3`:
```bash
aws sqs get-queue-attributes \
  --queue-url $QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages \
  --region us-east-1
```

---

#### Step 4: Receive Messages from the Queue

**Console:**
1. On the **Send and receive messages** page, scroll to **Receive messages**.
2. Click **Poll for messages**.
3. You will see up to 3 messages appear in the list.
4. Click on a message to view its body and attributes.

**CLI:**
```bash
# Receive up to 3 messages (MaxNumberOfMessages max is 10)
aws sqs receive-message \
  --queue-url $QUEUE_URL \
  --max-number-of-messages 3 \
  --visibility-timeout 30 \
  --wait-time-seconds 5 \
  --region us-east-1
```

**Expected Output:**
```json
{
    "Messages": [
        {
            "MessageId": "12345678-1234-1234-1234-123456789012",
            "ReceiptHandle": "AQEBwJ...very-long-string...==",
            "MD5OfBody": "a1b2c3d4e5f6...",
            "Body": "{\"orderId\": \"ORD-001\", \"product\": \"Widget A\", \"quantity\": 5}"
        }
    ]
}
```

> ⚠️ **Important:** Save the `ReceiptHandle` values — you need them to delete messages.

> ✅ **Verify:** Messages are returned. Note that `ApproximateNumberOfMessagesNotVisible` will now show the received messages (they are "in flight" during the visibility timeout).

---

#### Step 5: Delete Messages from the Queue

**Console:**
1. In the messages list (from polling), check the checkbox next to each message.
2. Click **Delete**.
3. Confirm deletion.

**CLI:**
```bash
# Store the receipt handle from the previous receive command
RECEIPT_HANDLE="AQEBwJ...your-receipt-handle...=="

# Delete the message
aws sqs delete-message \
  --queue-url $QUEUE_URL \
  --receipt-handle $RECEIPT_HANDLE \
  --region us-east-1

# Receive and delete remaining messages in a loop
aws sqs receive-message \
  --queue-url $QUEUE_URL \
  --max-number-of-messages 10 \
  --region us-east-1 | \
  jq -r '.Messages[].ReceiptHandle' | \
  while read handle; do
    aws sqs delete-message \
      --queue-url $QUEUE_URL \
      --receipt-handle "$handle" \
      --region us-east-1
    echo "Deleted message with handle: ${handle:0:20}..."
  done
```

> ✅ **Verify:** After deletion, confirm the queue is empty:
```bash
aws sqs get-queue-attributes \
  --queue-url $QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible \
  --region us-east-1
```
Both values should return `"0"`.

---

#### Step 6: Create a FIFO Queue and Compare

**Console:**
1. Click **Create queue**.
2. Select **FIFO** queue type.
3. Name it `lab1-orders-queue.fifo` (FIFO queues **must** end with `.fifo`).
4. Enable **Content-based deduplication**.
5. Click **Create queue**.

**CLI:**
```bash
aws sqs create-queue \
  --queue-name lab1-orders-queue.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=true \
  --region us-east-1

# Send a message to FIFO queue (requires MessageGroupId)
FIFO_URL=$(aws sqs get-queue-url \
  --queue-name lab1-orders-queue.fifo \
  --query 'QueueUrl' \
  --output text \
  --region us-east-1)

aws sqs send-message \
  --queue-url $FIFO_URL \
  --message-body '{"orderId": "ORD-001", "product": "Widget A"}' \
  --message-group-id "order-group-1" \
  --region us-east-1
```

> ✅ **Verify:** FIFO queue is created and message is accepted with a `SequenceNumber` in the response — confirming ordered delivery.

---

### Verification

Run the following checklist to confirm lab completion:

```bash
# 1. Verify Standard queue exists
aws sqs get-queue-url --queue-name lab1-orders-queue --region us-east-1

# 2. Verify FIFO queue exists
aws sqs get-queue-url --queue-name lab1-orders-queue.fifo --region us-east-1

# 3. Verify Standard queue is empty
aws sqs get-queue-attributes \
  --queue-url $(aws sqs get-queue-url --queue-name lab1-orders-queue --query 'QueueUrl' --output text) \
  --attribute-names ApproximateNumberOfMessages \
  --region us-east-1
```

**Expected Final State:**
- ✅ `lab1-orders-queue` exists and is empty
- ✅ `lab1-orders-queue.fifo` exists
- ✅ You successfully sent, received, and deleted messages
- ✅ You understand the FIFO queue naming requirement and `MessageGroupId`

---

### Cleanup

```bash
# Delete the Standard queue
aws sqs delete-queue \
  --queue-url $(aws sqs get-queue-url \
    --queue-name lab1-orders-queue \
    --query 'QueueUrl' \
    --output text) \
  --region us-east-1

# Delete the FIFO queue
aws sqs delete-queue \
  --queue-url $(aws sqs get-queue-url \
    --queue-name lab1-orders-queue.fifo \
    --query 'QueueUrl' \
    --output text) \
  --region us-east-1

echo "Cleanup complete. Both queues deleted."
```

> ⚠️ **Note:** After deleting a queue, you must wait **60 seconds** before creating a queue with the same name.

---

---

## Lab 2: Intermediate SQS Configuration

### Objective
In this lab, you will build a realistic order-processing pipeline using SQS with a **Dead-Letter Queue (DLQ)**, configure **Long Polling** for efficient message retrieval, set **message attributes** for filtering, use **batch operations** for efficiency, and monitor queue health using **Amazon CloudWatch**. You will simulate message processing failures to observe DLQ behavior.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account with console access |
| IAM Permissions | All Lab 1 permissions + `cloudwatch:GetMetricStatistics`, `cloudwatch:PutMetricAlarm`, `sns:CreateTopic`, `sns:Subscribe` |
| Tools | AWS CLI v2, `jq` for JSON parsing |
| Completed | Lab 1 (familiarity with basic SQS operations) |
| Region | `us-east-1` |

---

### Steps

#### Step 1: Create the Dead-Letter Queue (DLQ)

The DLQ receives messages that fail processing after a configured number of attempts.

**Console:**
1. Navigate to **SQS → Create queue**.
2. Select **Standard**.
3. Name it `lab2-orders-dlq`.
4. Set **Message retention period** to **14 days** (maximum — useful for debugging).
5. Click **Create queue**.

**CLI:**
```bash
# Create the DLQ with maximum retention
aws sqs create-queue \
  --queue-name lab2-orders-dlq \
  --attributes MessageRetentionPeriod=1209600 \
  --region us-east-1

# Store DLQ ARN
DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url $(aws sqs get-queue-url \
    --queue-name lab2-orders-dlq \
    --query 'QueueUrl' \
    --output text) \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text \
  --region us-east-1)

echo "DLQ ARN: $DLQ_ARN"
```

> ✅ **Verify:** DLQ is created. Note the ARN — it is required in the next step.

---

#### Step 2: Create the Main Queue with DLQ Redrive Policy

**Console:**
1. Click **Create queue**.
2. Select **Standard**, name it `lab2-orders-main`.
3. Scroll to **Dead-letter queue** section.
4. Enable **Dead-letter queue**.
5. Select `lab2-orders-dlq` from the dropdown.
6. Set **Maximum receives** to `3` (message moves to DLQ after 3 failed receives).
7. Click **Create queue**.

**CLI:**
```bash
# Create redrive policy JSON
REDRIVE_POLICY=$(cat <<EOF
{
  "deadLetterTargetArn": "$DLQ_ARN",
  "maxReceiveCount": "3"
}
EOF
)

# Create the main queue with DLQ policy
MAIN_QUEUE_URL=$(aws sqs create-queue \
  --queue-name lab2-orders-main \
  --attributes \
    VisibilityTimeout=30 \
    MessageRetentionPeriod=86400 \
    RedrivePolicy=$(echo $REDRIVE_POLICY | jq -c . | python3 -c "import sys,urllib.parse; print(urllib.parse.quote(sys.stdin.read()))") \
  --query 'QueueUrl' \
  --output text \
  --region us-east-1)

# Alternative approach using a JSON file
cat > /tmp/queue-attributes.json <<EOF
{
  "VisibilityTimeout": "30",
  "MessageRetentionPeriod": "86400",
  "RedrivePolicy": "{\"deadLetterTargetArn\":\"$DLQ_ARN\",\"maxReceiveCount\":\"3\"}"
}
EOF

aws sqs create-queue \
  --queue-name lab2-orders-main \
  --attributes file:///tmp/queue-attributes.json \
  --region us-east-1

MAIN_QUEUE_URL=$(aws sqs get-queue-url \
  --queue-name lab2-orders-main \
  --query 'QueueUrl' \
  --output text \
  --region us-east-1)

echo "Main Queue URL: $MAIN_QUEUE_URL"
```

> ✅ **Verify:** Check the redrive policy is attached