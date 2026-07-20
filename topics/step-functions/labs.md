# Step Functions — Hands-On Labs

## Lab 1: Getting Started with Step Functions

### Objective
In this lab, you will create your first AWS Step Functions state machine using the Standard workflow type. You will build a simple order processing workflow that chains together Lambda functions to validate an order, process payment, and send a confirmation notification. By the end of this lab, you will understand the core concepts of state machines, states, transitions, and how to execute and monitor workflows.

### Prerequisites

**AWS Services Required:**
- AWS Step Functions
- AWS Lambda
- Amazon SNS
- AWS IAM
- Amazon CloudWatch

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "states:*",
        "lambda:CreateFunction",
        "lambda:InvokeFunction",
        "lambda:DeleteFunction",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole",
        "iam:DeleteRole",
        "iam:DetachRolePolicy",
        "sns:CreateTopic",
        "sns:DeleteTopic",
        "logs:CreateLogGroup",
        "logs:DeleteLogGroup"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- Python 3.9+ (for Lambda function code)
- A text editor or IDE
- AWS Console access in a supported browser

**Region:** `us-east-1` (all steps assume this region; adjust as needed)

---

### Steps

#### Step 1: Create IAM Roles

**Why:** Step Functions needs permission to invoke Lambda, and Lambda needs a basic execution role.

**Console:**
1. Navigate to **IAM → Roles → Create role**
2. Select **AWS service** → **Lambda** → Click **Next**
3. Attach policy: `AWSLambdaBasicExecutionRole`
4. Name the role: `lab1-lambda-execution-role`
5. Click **Create role**

Repeat for the Step Functions role:
1. **IAM → Roles → Create role**
2. Select **AWS service** → **Step Functions**
3. Attach policy: `AWSLambdaRole`
4. Name the role: `lab1-stepfunctions-execution-role`
5. Click **Create role**

**CLI:**
```bash
# Create Lambda execution role
aws iam create-role \
  --role-name lab1-lambda-execution-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name lab1-lambda-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Create Step Functions execution role
aws iam create-role \
  --role-name lab1-stepfunctions-execution-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "states.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name lab1-stepfunctions-execution-role \
  --policy-arn arn:aws:iam::aws:policy/AWSLambdaRole
```

**Verify:**
```bash
aws iam get-role --role-name lab1-lambda-execution-role \
  --query 'Role.RoleName' --output text
# Expected: lab1-lambda-execution-role

aws iam get-role --role-name lab1-stepfunctions-execution-role \
  --query 'Role.RoleName' --output text
# Expected: lab1-stepfunctions-execution-role
```

---

#### Step 2: Create Lambda Functions

**Why:** These functions simulate the steps in an order processing pipeline.

**Create three Lambda function source files:**

```bash
mkdir -p lab1-functions && cd lab1-functions
```

**File: `validate_order.py`**
```python
import json

def lambda_handler(event, context):
    print(f"Validating order: {json.dumps(event)}")
    
    order_id = event.get("order_id")
    amount = event.get("amount", 0)
    
    if not order_id:
        raise ValueError("Missing order_id")
    
    if amount <= 0:
        raise ValueError("Invalid order amount")
    
    return {
        "order_id": order_id,
        "amount": amount,
        "status": "VALIDATED",
        "customer_email": event.get("customer_email", "customer@example.com")
    }
```

**File: `process_payment.py`**
```python
import json
import random

def lambda_handler(event, context):
    print(f"Processing payment for order: {event['order_id']}")
    
    # Simulate payment processing
    transaction_id = f"TXN-{random.randint(100000, 999999)}"
    
    return {
        "order_id": event["order_id"],
        "amount": event["amount"],
        "status": "PAYMENT_PROCESSED",
        "transaction_id": transaction_id,
        "customer_email": event["customer_email"]
    }
```

**File: `send_confirmation.py`**
```python
import json

def lambda_handler(event, context):
    print(f"Sending confirmation for order: {event['order_id']}")
    
    # Simulate sending email confirmation
    message = (
        f"Order {event['order_id']} confirmed. "
        f"Transaction: {event['transaction_id']}. "
        f"Amount: ${event['amount']}"
    )
    
    return {
        "order_id": event["order_id"],
        "status": "CONFIRMED",
        "message": message,
        "notification_sent": True
    }
```

**Package and deploy Lambda functions via CLI:**
```bash
# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/lab1-lambda-execution-role"

# Package validate_order
zip validate_order.zip validate_order.py
aws lambda create-function \
  --function-name lab1-validate-order \
  --runtime python3.11 \
  --role "${ROLE_ARN}" \
  --handler validate_order.lambda_handler \
  --zip-file fileb://validate_order.zip \
  --timeout 10

# Package process_payment
zip process_payment.zip process_payment.py
aws lambda create-function \
  --function-name lab1-process-payment \
  --runtime python3.11 \
  --role "${ROLE_ARN}" \
  --handler process_payment.lambda_handler \
  --zip-file fileb://process_payment.zip \
  --timeout 10

# Package send_confirmation
zip send_confirmation.zip send_confirmation.py
aws lambda create-function \
  --function-name lab1-send-confirmation \
  --runtime python3.11 \
  --role "${ROLE_ARN}" \
  --handler send_confirmation.lambda_handler \
  --zip-file fileb://send_confirmation.zip \
  --timeout 10
```

**Console (alternative):**
1. Navigate to **Lambda → Create function**
2. Select **Author from scratch**
3. Function name: `lab1-validate-order`
4. Runtime: **Python 3.11**
5. Execution role: **Use an existing role** → `lab1-lambda-execution-role`
6. Click **Create function**
7. In the code editor, paste the `validate_order.py` content
8. Click **Deploy**
9. Repeat for `lab1-process-payment` and `lab1-send-confirmation`

**Verify:**
```bash
aws lambda list-functions \
  --query 'Functions[?starts_with(FunctionName, `lab1`)].FunctionName' \
  --output table
```

**Expected output:**
```
------------------------------
|       ListFunctions        |
+----------------------------+
|  lab1-validate-order       |
|  lab1-process-payment      |
|  lab1-send-confirmation    |
+----------------------------+
```

---

#### Step 3: Create the State Machine Definition

**Why:** The Amazon States Language (ASL) definition describes the workflow logic.

**Create file: `order-workflow.json`**
```json
{
  "Comment": "Lab 1 - Simple Order Processing Workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:lab1-validate-order",
      "Next": "ProcessPayment",
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 2,
          "BackoffRate": 1.5
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "OrderFailed",
          "ResultPath": "$.error"
        }
      ]
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:lab1-process-payment",
      "Next": "SendConfirmation",
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "OrderFailed",
          "ResultPath": "$.error"
        }
      ]
    },
    "SendConfirmation": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:lab1-send-confirmation",
      "Next": "OrderSuccess"
    },
    "OrderSuccess": {
      "Type": "Succeed"
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "OrderProcessingFailed",
      "Cause": "An error occurred during order processing"
    }
  }
}
```

**Replace `ACCOUNT_ID` with your actual account ID:**
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/ACCOUNT_ID/${ACCOUNT_ID}/g" order-workflow.json

# Verify the substitution
grep "Resource" order-workflow.json
```

---

#### Step 4: Create the State Machine

**Console:**
1. Navigate to **Step Functions → State machines → Create state machine**
2. Select **Write your workflow in code**
3. Type: **Standard**
4. Paste the content of `order-workflow.json` into the editor
5. Click **Next**
6. Name: `lab1-order-processing`
7. Execution role: **Choose an existing role** → `lab1-stepfunctions-execution-role`
8. Click **Create state machine**

**CLI:**
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SF_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/lab1-stepfunctions-execution-role"

aws stepfunctions create-state-machine \
  --name lab1-order-processing \
  --definition file://order-workflow.json \
  --role-arn "${SF_ROLE_ARN}" \
  --type STANDARD
```

**Expected output:**
```json
{
    "stateMachineArn": "arn:aws:states:us-east-1:123456789012:stateMachine:lab1-order-processing",
    "creationDate": "2024-01-15T10:00:00.000Z"
}
```

**Verify:**
```bash
aws stepfunctions describe-state-machine \
  --state-machine-arn "arn:aws:states:us-east-1:${ACCOUNT_ID}:stateMachine:lab1-order-processing" \
  --query '{Name: name, Status: status, Type: type}' \
  --output table
```

---

#### Step 5: Execute the State Machine

**Console:**
1. In **Step Functions → State machines**, click `lab1-order-processing`
2. Click **Start execution**
3. In the input box, enter:
```json
{
  "order_id": "ORD-001",
  "amount": 99.99,
  "customer_email": "alice@example.com"
}
```
4. Click **Start execution**
5. Observe the visual workflow graph — each state should turn green as it completes

**CLI:**
```bash
STATE_MACHINE_ARN="arn:aws:states:us-east-1:${ACCOUNT_ID}:stateMachine:lab1-order-processing"

EXECUTION_ARN=$(aws stepfunctions start-execution \
  --state-machine-arn "${STATE_MACHINE_ARN}" \
  --name "test-execution-001" \
  --input '{"order_id": "ORD-001", "amount": 99.99, "customer_email": "alice@example.com"}' \
  --query 'executionArn' \
  --output text)

echo "Execution ARN: ${EXECUTION_ARN}"

# Wait for completion and check status
sleep 10

aws stepfunctions describe-execution \
  --execution-arn "${EXECUTION_ARN}" \
  --query '{Status: status, StartDate: startDate, StopDate: stopDate}' \
  --output table
```

**Expected output:**
```
---------------------------------------------
|          DescribeExecution                |
+------------+------------------------------+
|  Status    |  SUCCEEDED                   |
|  StartDate |  2024-01-15T10:05:00.000Z    |
|  StopDate  |  2024-01-15T10:05:03.500Z    |
+------------+------------------------------+
```

**View execution output:**
```bash
aws stepfunctions get-execution-history \
  --execution-arn "${EXECUTION_ARN}" \
  --query 'events[-1].executionSucceededEventDetails.output' \
  --output text | python3 -m json.tool
```

---

#### Step 6: Test Error Handling

**Console:**
1. Click **Start execution** again on the state machine
2. Enter invalid input (missing `order_id`):
```json
{
  "amount": 99.99,
  "customer_email": "bob@example.com"
}
```
3. Observe the workflow — `ValidateOrder` should fail and transition to `OrderFailed`

**CLI:**
```bash
FAILED_EXECUTION_ARN=$(aws stepfunctions start-execution \
  --state-machine-arn "${STATE_MACHINE_ARN}" \
  --name "test-execution-fail-001" \
  --input '{"amount": 99.99, "customer_email": "bob@example.com"}' \
  --query 'executionArn' \
  --output text)

sleep 10

aws stepfunctions describe-execution \
  --execution-arn "${FAILED_EXECUTION_ARN}" \
  --query '{Status: status, Cause: cause}' \
  --output table
```

**Expected output:**
```
-----------------------------------------------
|          DescribeExecution                  |
+---------+-----------------------------------+
|  Cause  |  OrderProcessingFailed            |
|  Status |  FAILED                           |
+---------+-----------------------------------+
```

---

### Verification

Run the following commands to confirm successful