# EventBridge — Hands-On Labs

## Lab 1: Getting Started with EventBridge

### Objective

In this lab, you will learn the fundamentals of Amazon EventBridge by creating a custom event bus, defining event rules with pattern matching, and routing events to an AWS Lambda function target. By the end of this lab, you will understand how to publish custom events, write event patterns, and verify end-to-end event delivery through CloudWatch Logs.

---

### Prerequisites

**AWS Services Required:**
- Amazon EventBridge
- AWS Lambda
- Amazon CloudWatch Logs
- AWS IAM

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "events:CreateEventBus",
        "events:PutRule",
        "events:PutTargets",
        "events:PutEvents",
        "events:DescribeRule",
        "events:ListRules",
        "events:ListTargetsByRule",
        "lambda:CreateFunction",
        "lambda:AddPermission",
        "lambda:InvokeFunction",
        "lambda:GetFunction",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:DescribeLogGroups",
        "logs:GetLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- AWS Management Console access
- `curl` or a REST client (optional)
- Basic familiarity with JSON

**Environment Variables (set these before starting):**
```bash
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export LAB1_PREFIX="eb-lab1"
```

---

### Steps

#### Step 1: Create a Custom Event Bus

A custom event bus isolates your application events from AWS service events on the default bus.

**Console:**
1. Navigate to **Amazon EventBridge** in the AWS Console.
2. In the left navigation pane, click **Event buses**.
3. Click **Create event bus**.
4. Enter the name: `eb-lab1-orders-bus`
5. Leave **Event archive** and **Schema discovery** disabled for now.
6. Click **Create**.

**CLI:**
```bash
aws events create-event-bus \
  --name "eb-lab1-orders-bus" \
  --region $AWS_REGION
```

**Verify:**
```bash
aws events describe-event-bus \
  --name "eb-lab1-orders-bus" \
  --region $AWS_REGION
```

**Expected Output:**
```json
{
    "Name": "eb-lab1-orders-bus",
    "Arn": "arn:aws:events:us-east-1:123456789012:event-bus/eb-lab1-orders-bus",
    "CreationTime": "2024-01-15T10:00:00.000Z"
}
```

---

#### Step 2: Create the Lambda Execution Role

**CLI:**
```bash
# Create the trust policy
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

# Create the IAM role
aws iam create-role \
  --role-name "eb-lab1-lambda-role" \
  --assume-role-policy-document file:///tmp/lambda-trust-policy.json

# Attach the basic Lambda execution policy
aws iam attach-role-policy \
  --role-name "eb-lab1-lambda-role" \
  --policy-arn "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"

echo "Waiting for role propagation..."
sleep 10
```

**Verify:**
```bash
aws iam get-role --role-name "eb-lab1-lambda-role" \
  --query 'Role.Arn' --output text
```

**Expected Output:**
```
arn:aws:iam::123456789012:role/eb-lab1-lambda-role
```

---

#### Step 3: Create the Lambda Function

This Lambda function will log all incoming EventBridge events to CloudWatch Logs.

**CLI:**
```bash
# Create the Lambda function code
cat > /tmp/event_handler.py << 'EOF'
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info("=== EventBridge Event Received ===")
    logger.info(f"Source: {event.get('source', 'unknown')}")
    logger.info(f"Detail Type: {event.get('detail-type', 'unknown')}")
    logger.info(f"Detail: {json.dumps(event.get('detail', {}), indent=2)}")
    logger.info(f"Full Event: {json.dumps(event, indent=2)}")
    
    return {
        "statusCode": 200,
        "message": f"Processed event from {event.get('source', 'unknown')}"
    }
EOF

# Package the function
cd /tmp && zip event_handler.zip event_handler.py

# Get the role ARN
LAMBDA_ROLE_ARN=$(aws iam get-role \
  --role-name "eb-lab1-lambda-role" \
  --query 'Role.Arn' --output text)

# Create the Lambda function
aws lambda create-function \
  --function-name "eb-lab1-order-processor" \
  --runtime "python3.12" \
  --role "$LAMBDA_ROLE_ARN" \
  --handler "event_handler.lambda_handler" \
  --zip-file "fileb:///tmp/event_handler.zip" \
  --description "Processes EventBridge order events" \
  --timeout 30 \
  --region $AWS_REGION
```

**Verify:**
```bash
aws lambda get-function \
  --function-name "eb-lab1-order-processor" \
  --query 'Configuration.{State:State,Runtime:Runtime,Handler:Handler}' \
  --output table \
  --region $AWS_REGION
```

**Expected Output:**
```
----------------------------------------------
|              GetFunction                   |
+----------+-------------------+-------------+
| Handler  |    Runtime        |   State     |
+----------+-------------------+-------------+
|event_handler.lambda_handler|python3.12|Active|
+----------+-------------------+-------------+
```

---

#### Step 4: Create an EventBridge Rule with Event Pattern

The rule will match all order-related events from your application.

**Console:**
1. In EventBridge, click **Rules** in the left navigation.
2. Select **eb-lab1-orders-bus** from the Event bus dropdown.
3. Click **Create rule**.
4. Name: `eb-lab1-order-events-rule`
5. Description: `Matches all order events from the orders service`
6. Rule type: **Rule with an event pattern**
7. Click **Next**.
8. Event source: **Other**
9. Paste the following event pattern:
   ```json
   {
     "source": ["com.myapp.orders"],
     "detail-type": ["OrderPlaced", "OrderUpdated", "OrderCancelled"]
   }
   ```
10. Click **Next**, then select **Lambda function** as the target.
11. Select `eb-lab1-order-processor`.
12. Click **Next**, then **Create rule**.

**CLI:**
```bash
# Create the event pattern file
cat > /tmp/event-pattern.json << 'EOF'
{
  "source": ["com.myapp.orders"],
  "detail-type": ["OrderPlaced", "OrderUpdated", "OrderCancelled"]
}
EOF

# Create the rule
aws events put-rule \
  --name "eb-lab1-order-events-rule" \
  --event-bus-name "eb-lab1-orders-bus" \
  --event-pattern file:///tmp/event-pattern.json \
  --description "Matches all order events from the orders service" \
  --state "ENABLED" \
  --region $AWS_REGION
```

**Verify:**
```bash
aws events describe-rule \
  --name "eb-lab1-order-events-rule" \
  --event-bus-name "eb-lab1-orders-bus" \
  --region $AWS_REGION
```

**Expected Output:**
```json
{
    "Name": "eb-lab1-order-events-rule",
    "Arn": "arn:aws:events:us-east-1:123456789012:rule/eb-lab1-orders-bus/eb-lab1-order-events-rule",
    "EventPattern": "{\"source\":[\"com.myapp.orders\"],\"detail-type\":[\"OrderPlaced\",\"OrderUpdated\",\"OrderCancelled\"]}",
    "State": "ENABLED",
    "EventBusName": "eb-lab1-orders-bus"
}
```

---

#### Step 5: Add Lambda as the Rule Target

**Console:** *(Completed in Step 4 if using Console)*

**CLI:**
```bash
# Get Lambda ARN
LAMBDA_ARN=$(aws lambda get-function \
  --function-name "eb-lab1-order-processor" \
  --query 'Configuration.FunctionArn' \
  --output text \
  --region $AWS_REGION)

# Add Lambda as target
aws events put-targets \
  --rule "eb-lab1-order-events-rule" \
  --event-bus-name "eb-lab1-orders-bus" \
  --targets "[{\"Id\":\"OrderProcessorLambda\",\"Arn\":\"$LAMBDA_ARN\"}]" \
  --region $AWS_REGION

# Grant EventBridge permission to invoke Lambda
aws lambda add-permission \
  --function-name "eb-lab1-order-processor" \
  --statement-id "eb-lab1-invoke-permission" \
  --action "lambda:InvokeFunction" \
  --principal "events.amazonaws.com" \
  --source-arn "arn:aws:events:$AWS_REGION:$AWS_ACCOUNT_ID:rule/eb-lab1-orders-bus/eb-lab1-order-events-rule" \
  --region $AWS_REGION
```

**Verify:**
```bash
aws events list-targets-by-rule \
  --rule "eb-lab1-order-events-rule" \
  --event-bus-name "eb-lab1-orders-bus" \
  --region $AWS_REGION
```

**Expected Output:**
```json
{
    "Targets": [
        {
            "Id": "OrderProcessorLambda",
            "Arn": "arn:aws:events:us-east-1:123456789012:function:eb-lab1-order-processor"
        }
    ]
}
```

---

#### Step 6: Publish Test Events

Now send events to your custom bus and watch them flow through the system.

**Console:**
1. In EventBridge, click **Event buses**.
2. Select **eb-lab1-orders-bus**.
3. Click **Send events**.
4. Fill in:
   - **Event source:** `com.myapp.orders`
   - **Detail type:** `OrderPlaced`
   - **Event detail:**
     ```json
     {
       "orderId": "ORD-001",
       "customerId": "CUST-123",
       "amount": 99.99,
       "currency": "USD",
       "items": [
         {"sku": "WIDGET-A", "qty": 2, "price": 49.99}
       ]
     }
     ```
5. Click **Send**.

**CLI:**
```bash
# Send an OrderPlaced event
aws events put-events \
  --entries '[
    {
      "Source": "com.myapp.orders",
      "DetailType": "OrderPlaced",
      "Detail": "{\"orderId\":\"ORD-001\",\"customerId\":\"CUST-123\",\"amount\":99.99,\"currency\":\"USD\",\"items\":[{\"sku\":\"WIDGET-A\",\"qty\":2,\"price\":49.99}]}",
      "EventBusName": "eb-lab1-orders-bus"
    }
  ]' \
  --region $AWS_REGION

# Send an OrderCancelled event
aws events put-events \
  --entries '[
    {
      "Source": "com.myapp.orders",
      "DetailType": "OrderCancelled",
      "Detail": "{\"orderId\":\"ORD-001\",\"reason\":\"Customer request\",\"refundAmount\":99.99}",
      "EventBusName": "eb-lab1-orders-bus"
    }
  ]' \
  --region $AWS_REGION

# Send a non-matching event (should NOT trigger the rule)
aws events put-events \
  --entries '[
    {
      "Source": "com.myapp.inventory",
      "DetailType": "StockUpdated",
      "Detail": "{\"sku\":\"WIDGET-A\",\"newStock\":50}",
      "EventBusName": "eb-lab1-orders-bus"
    }
  ]' \
  --region $AWS_REGION
```

**Expected Output (for matching events):**
```json
{
    "FailedEntryCount": 0,
    "Entries": [
        {
            "EventId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
        }
    ]
}
```

---

#### Step 7: Verify Events in CloudWatch Logs

**Console:**
1. Navigate to **CloudWatch** → **Log groups**.
2. Find `/aws/lambda/eb-lab1-order-processor`.
3. Click the most recent log stream.
4. Look for log entries containing `EventBridge Event Received`.

**CLI:**
```bash
# Wait a few seconds for the events to process
sleep 5

# Get the log group name
LOG_GROUP="/aws/lambda/eb-lab1-order-processor"

# List log streams (most recent first)
aws logs describe-log-streams \
  --log-group-name "$LOG_GROUP" \
  --order-by LastEventTime \
  --descending \
  --max-items 1 \
  --region $AWS_REGION

# Get the latest stream name
STREAM_NAME=$(aws logs describe-log-streams \
  --log-group-name "$LOG_GROUP" \
  --order-by LastEventTime \
  --descending \
  --max-items 1 \
  --query 'logStreams[0].logStreamName' \
  --output text \
  --region $AWS_REGION)

# Retrieve log events
aws logs get-log-events \
  --log-group-name "$LOG_GROUP" \
  --log-stream-name "$STREAM_NAME" \
  --region $AWS_REGION \
  --query 'events[*].message' \
  --output text
```

**Expected Log Output:**
```
=== EventBridge Event Received ===
Source: com.myapp.orders
Detail Type: OrderPlaced
Detail: {
  "orderId": "ORD-001",
  "customerId": "CUST-123",
  "amount": 99.99,
  ...
}
```

---

### Verification

Run the following checklist to confirm successful lab completion:

```bash
echo "=== Lab 1 Verification ==="

# 1. Check event bus exists
echo -n "1. Custom event bus exists: "
aws events describe-event-bus --name "eb-lab1-orders-bus" \
  --query 'Name' --output text --region $AWS_REGION 2>/dev/null \
  && echo "✅ PASS" || echo "❌ FAIL"

# 