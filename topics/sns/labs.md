# SNS — Hands-On Labs

## Lab 1: Getting Started with SNS

### Objective
In this lab, you will create your first Amazon Simple Notification Service (SNS) topic, subscribe multiple endpoints (email and AWS Lambda), and publish messages to verify end-to-end delivery. By the end, you will understand the publisher-subscriber model, topic types (Standard vs. FIFO), and how to confirm subscriptions.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account with billing enabled |
| IAM Permissions | `sns:*`, `lambda:CreateFunction`, `lambda:AddPermission`, `iam:CreateRole`, `iam:AttachRolePolicy` |
| Tools | AWS CLI v2 installed and configured (`aws configure`) |
| Region | `us-east-1` (all commands use this region) |
| Email Address | A valid email you can access to confirm subscriptions |

---

### Steps

#### Step 1: Create a Standard SNS Topic

**Console:**
1. Open the [SNS Console](https://console.aws.amazon.com/sns/v3/home).
2. In the left navigation, click **Topics**, then click **Create topic**.
3. Select **Standard** as the type.
4. Enter the following:
   - **Name:** `lab1-notifications`
   - **Display name:** `Lab1 Alerts`
5. Leave all other settings as default.
6. Click **Create topic**.

**CLI:**
```bash
aws sns create-topic \
  --name lab1-notifications \
  --attributes DisplayName="Lab1 Alerts" \
  --region us-east-1
```

**Expected Output:**
```json
{
    "TopicArn": "arn:aws:sns:us-east-1:123456789012:lab1-notifications"
}
```

> **Save the TopicArn** — you will need it throughout this lab. Set it as a variable:
```bash
TOPIC_ARN=$(aws sns create-topic \
  --name lab1-notifications \
  --query 'TopicArn' \
  --output text \
  --region us-east-1)
echo "Topic ARN: $TOPIC_ARN"
```

**Verify:**
```bash
aws sns get-topic-attributes \
  --topic-arn $TOPIC_ARN \
  --region us-east-1
```
Confirm `TopicArn` appears in the output.

---

#### Step 2: Subscribe an Email Endpoint

**Console:**
1. On the topic detail page, click **Create subscription**.
2. Set:
   - **Protocol:** `Email`
   - **Endpoint:** `your-email@example.com`
3. Click **Create subscription**.
4. Check your inbox and click the **Confirm subscription** link.

**CLI:**
```bash
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint your-email@example.com \
  --region us-east-1
```

**Expected Output:**
```json
{
    "SubscriptionArn": "pending confirmation"
}
```

> ✅ Check your email and click **Confirm subscription**. This step is required before messages are delivered.

**Verify:**
```bash
aws sns list-subscriptions-by-topic \
  --topic-arn $TOPIC_ARN \
  --region us-east-1
```
After confirmation, `SubscriptionArn` should show a full ARN (not `PendingConfirmation`).

---

#### Step 3: Create an IAM Role and Lambda Function for SNS

**Create the IAM execution role:**
```bash
# Create trust policy document
cat > /tmp/lambda-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name lab1-lambda-sns-role \
  --assume-role-policy-document file:///tmp/lambda-trust-policy.json

# Attach basic Lambda execution policy
aws iam attach-role-policy \
  --role-name lab1-lambda-sns-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**Create the Lambda function:**
```bash
# Create function code
cat > /tmp/lambda_function.py << 'EOF'
import json

def lambda_handler(event, context):
    for record in event['Records']:
        sns_message = record['Sns']
        print(f"Subject: {sns_message.get('Subject', 'No Subject')}")
        print(f"Message: {sns_message['Message']}")
        print(f"Timestamp: {sns_message['Timestamp']}")
    return {"statusCode": 200, "body": "SNS message processed"}
EOF

# Package it
cd /tmp && zip lambda_function.zip lambda_function.py

# Get your account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Deploy the function
aws lambda create-function \
  --function-name lab1-sns-processor \
  --runtime python3.12 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/lab1-lambda-sns-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb:///tmp/lambda_function.zip \
  --region us-east-1
```

**Console:**
1. Open the [Lambda Console](https://console.aws.amazon.com/lambda).
2. Click **Create function** → **Author from scratch**.
3. Set:
   - **Function name:** `lab1-sns-processor`
   - **Runtime:** Python 3.12
   - **Execution role:** Use the role created above (`lab1-lambda-sns-role`)
4. Paste the Python code above into the inline editor.
5. Click **Deploy**.

---

#### Step 4: Subscribe Lambda to the SNS Topic

**Console:**
1. Return to the SNS topic `lab1-notifications`.
2. Click **Create subscription**.
3. Set:
   - **Protocol:** `AWS Lambda`
   - **Endpoint:** Select `lab1-sns-processor` from the dropdown.
4. Click **Create subscription**.

**CLI:**
```bash
LAMBDA_ARN=$(aws lambda get-function \
  --function-name lab1-sns-processor \
  --query 'Configuration.FunctionArn' \
  --output text \
  --region us-east-1)

aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol lambda \
  --notification-endpoint $LAMBDA_ARN \
  --region us-east-1

# Grant SNS permission to invoke Lambda
aws lambda add-permission \
  --function-name lab1-sns-processor \
  --statement-id sns-invoke-permission \
  --action lambda:InvokeFunction \
  --principal sns.amazonaws.com \
  --source-arn $TOPIC_ARN \
  --region us-east-1
```

**Verify:**
```bash
aws sns list-subscriptions-by-topic \
  --topic-arn $TOPIC_ARN \
  --region us-east-1
```
You should see **two** subscriptions: one `email` and one `lambda`.

---

#### Step 5: Publish a Test Message

**Console:**
1. On the topic page, click **Publish message**.
2. Set:
   - **Subject:** `Test Alert from Lab 1`
   - **Message body:** `Hello from SNS! This is a test notification.`
3. Click **Publish message**.

**CLI:**
```bash
aws sns publish \
  --topic-arn $TOPIC_ARN \
  --subject "Test Alert from Lab 1" \
  --message "Hello from SNS! This is a test notification." \
  --region us-east-1
```

**Expected Output:**
```json
{
    "MessageId": "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111"
}
```

---

### Verification

Run the following checks to confirm the lab is complete:

```bash
# 1. Verify topic exists
aws sns list-topics --region us-east-1 | grep lab1-notifications

# 2. Verify subscriptions (should show 2 confirmed)
aws sns list-subscriptions-by-topic \
  --topic-arn $TOPIC_ARN \
  --region us-east-1

# 3. Check Lambda logs for received message
aws logs filter-log-events \
  --log-group-name /aws/lambda/lab1-sns-processor \
  --filter-pattern "Message" \
  --region us-east-1
```

✅ **Success Criteria:**
- [ ] Topic `lab1-notifications` exists with type `Standard`
- [ ] Email subscription is in `Confirmed` state
- [ ] Lambda subscription is in `Confirmed` state
- [ ] CloudWatch Logs show the SNS message was received by Lambda
- [ ] You received the test email

---

### Cleanup

```bash
# 1. List and delete all subscriptions
aws sns list-subscriptions-by-topic \
  --topic-arn $TOPIC_ARN \
  --query 'Subscriptions[].SubscriptionArn' \
  --output text \
  --region us-east-1 | tr '\t' '\n' | while read ARN; do
    echo "Deleting subscription: $ARN"
    aws sns unsubscribe --subscription-arn $ARN --region us-east-1
done

# 2. Delete the SNS topic
aws sns delete-topic \
  --topic-arn $TOPIC_ARN \
  --region us-east-1

# 3. Delete the Lambda function
aws lambda delete-function \
  --function-name lab1-sns-processor \
  --region us-east-1

# 4. Detach policy and delete IAM role
aws iam detach-role-policy \
  --role-name lab1-lambda-sns-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role \
  --role-name lab1-lambda-sns-role

# 5. Delete CloudWatch Log Group
aws logs delete-log-group \
  --log-group-name /aws/lambda/lab1-sns-processor \
  --region us-east-1

echo "Cleanup complete."
```

---

## Lab 2: Intermediate SNS Configuration

### Objective
In this lab, you will implement advanced SNS features including **message filtering** (subscription filter policies), **message attributes**, **dead-letter queues (DLQ)** with SQS, and **fan-out architecture**. You will build a multi-subscriber system where different consumers receive only relevant messages based on filter criteria — a common pattern in event-driven microservices.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account |
| IAM Permissions | `sns:*`, `sqs:*`, `lambda:*`, `iam:*`, `logs:*` |
| Tools | AWS CLI v2, Python 3.x (for local testing scripts) |
| Completed | Lab 1 concepts understood (not required to be running) |
| Region | `us-east-1` |

---

### Steps

#### Step 1: Create an SNS Standard Topic with SQS Fan-Out

**Architecture Overview:**
```
                    ┌─────────────────────────────┐
                    │      SNS Topic               │
                    │   lab2-order-events          │
                    └──────────┬──────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │  SQS Queue   │  │  SQS Queue   │  │  SQS DLQ     │
    │  (orders)    │  │  (inventory) │  │  (failures)  │
    └──────────────┘  └──────────────┘  └──────────────┘
    Filter: NEW_ORDER  Filter: SHIPPED   Undeliverable msgs
```

**Create the SNS Topic:**
```bash
# Create the main topic
TOPIC_ARN=$(aws sns create-topic \
  --name lab2-order-events \
  --attributes DisplayName="Order Events" \
  --query 'TopicArn' \
  --output text \
  --region us-east-1)

echo "Topic ARN: $TOPIC_ARN"
```

---

#### Step 2: Create SQS Queues (Fan-Out Targets)

**Console:**
1. Open the [SQS Console](https://console.aws.amazon.com/sqs).
2. Create three queues:
   - `lab2-orders-queue` (Standard)
   - `lab2-inventory-queue` (Standard)
   - `lab2-dlq` (Standard, for dead letters)

**CLI:**
```bash
# Create Orders queue
ORDERS_QUEUE_URL=$(aws sqs create-queue \
  --queue-name lab2-orders-queue \
  --region us-east-1 \
  --query 'QueueUrl' \
  --output text)

# Create Inventory queue
INVENTORY_QUEUE_URL=$(aws sqs create-queue \
  --queue-name lab2-inventory-queue \
  --region us-east-1 \
  --query 'QueueUrl' \
  --output text)

# Create Dead Letter Queue
DLQ_URL=$(aws sqs create-queue \
  --queue-name lab2-dlq \
  --region us-east-1 \
  --query 'QueueUrl' \
  --output text)

# Get Queue ARNs
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ORDERS_QUEUE_ARN="arn:aws:sqs:us-east-1:${ACCOUNT_ID}:lab2-orders-queue"
INVENTORY_QUEUE_ARN="arn:aws:sqs:us-east-1:${ACCOUNT_ID}:lab2-inventory-queue"
DLQ_ARN="arn:aws:sqs:us-east-1:${ACCOUNT_ID}:lab2-dlq"

echo "Orders Queue: $ORDERS_QUEUE_ARN"
echo "Inventory Queue: $INVENTORY_QUEUE_ARN"
echo "DLQ: $DLQ_ARN"
```

---

#### Step 3: Set SQS Access Policies to Allow SNS

Each SQS queue needs a resource policy allowing SNS to send messages.

```bash
# Policy template function
create_sqs_policy() {
  local QUEUE_ARN=$1
  cat << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSNSPublish",
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "${QUEUE_ARN}",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "${TOPIC_ARN}"
        }
      }
    }
  ]
}
EOF
}

# Apply policy to Orders queue
aws sqs set-queue-attributes \
  --queue-url $ORDERS_QUEUE_URL \
  --attributes Policy="$(create_sqs_policy $ORDERS_QUEUE_ARN)" \
  --region us-east-1

# Apply policy to Inventory queue
aws sqs set-queue-attributes \
  --queue-url $INVENTORY_QUEUE_URL \
  --attributes Policy="$(create_sqs_policy $INVENTORY_QUEUE_ARN)" \
  --region us-east-1

echo "SQS policies applied."
```

---

#### Step 4: Subscribe SQS Queues with Filter Policies

This is the core of the lab — **subscription filter policies** ensure each queue only receives relevant messages.

**Subscribe