# Step Functions — AWS CLI Commands

## Setup & Configuration

### Prerequisites

- AWS CLI v2 installed (`aws --version`)
- Valid AWS credentials configured (`aws configure`)
- Appropriate IAM permissions

### Required IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "states:CreateStateMachine",
        "states:DeleteStateMachine",
        "states:DescribeStateMachine",
        "states:ListStateMachines",
        "states:StartExecution",
        "states:StopExecution",
        "states:DescribeExecution",
        "states:ListExecutions",
        "states:GetExecutionHistory",
        "states:UpdateStateMachine",
        "states:TagResource",
        "states:UntagResource",
        "states:ListTagsForResource",
        "states:CreateActivity",
        "states:DeleteActivity",
        "states:DescribeActivity",
        "states:ListActivities",
        "states:SendTaskSuccess",
        "states:SendTaskFailure",
        "states:SendTaskHeartbeat",
        "states:GetActivityTask"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": "arn:aws:iam::123456789012:role/StepFunctionsRole"
    }
  ]
}
```

### Environment Setup

```bash
# Configure default region
aws configure set region us-east-1

# Set output format to JSON
aws configure set output json

# Verify CLI version (v2 recommended)
aws --version

# Set environment variables for convenience
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SF_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/StepFunctionsExecutionRole"
```

### Trust Policy for Step Functions IAM Role

```bash
# Create the trust policy file
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "states.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the IAM role
aws iam create-role \
  --role-name StepFunctionsExecutionRole \
  --assume-role-policy-document file://trust-policy.json
```

---

## Core Commands

### 1. Create a State Machine

```bash
aws stepfunctions create-state-machine \
  --name "my-order-processing-workflow" \
  --definition file://state-machine-definition.json \
  --role-arn "arn:aws:iam::123456789012:role/StepFunctionsExecutionRole" \
  --type STANDARD \
  --tags key=Environment,value=Production key=Team,value=Backend
```

**Sample `state-machine-definition.json`:**

```json
{
  "Comment": "Order processing workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:validate-order",
      "Next": "ProcessPayment",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "OrderFailed"
        }
      ]
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:process-payment",
      "Next": "ShipOrder",
      "Retry": [
        {
          "ErrorEquals": ["PaymentError"],
          "IntervalSeconds": 5,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ]
    },
    "ShipOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ship-order",
      "End": true
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "OrderProcessingFailed",
      "Cause": "Order validation or payment failed"
    }
  }
}
```

**Example Output:**

```json
{
  "stateMachineArn": "arn:aws:states:us-east-1:123456789012:stateMachine:my-order-processing-workflow",
  "creationDate": "2024-01-15T10:30:00.000Z"
}
```

---

### 2. List State Machines

```bash
aws stepfunctions list-state-machines \
  --max-results 20
```

**Example Output:**

```json
{
  "stateMachines": [
    {
      "stateMachineArn": "arn:aws:states:us-east-1:123456789012:stateMachine:my-order-processing-workflow",
      "name": "my-order-processing-workflow",
      "type": "STANDARD",
      "creationDate": "2024-01-15T10:30:00.000Z"
    },
    {
      "stateMachineArn": "arn:aws:states:us-east-1:123456789012:stateMachine:my-data-pipeline",
      "name": "my-data-pipeline",
      "type": "EXPRESS",
      "creationDate": "2024-01-10T08:00:00.000Z"
    }
  ],
  "nextToken": null
}
```

---

### 3. Describe a State Machine

```bash
aws stepfunctions describe-state-machine \
  --state-machine-arn "arn:aws:states:us-east-1:123456789012:stateMachine:my-order-processing-workflow"
```

**Example Output:**

```json
{
  "stateMachineArn": "arn:aws:states:us-east-1:123456789012:stateMachine:my-order-processing-workflow",
  "name": "my-order-processing-workflow",
  "status": "ACTIVE",
  "definition": "{\"Comment\":\"Order processing workflow\", ...}",
  "roleArn": "arn:aws:iam::123456789012:role/StepFunctionsExecutionRole",
  "type": "STANDARD",
  "creationDate": "2024-01-15T10:30:00.000Z",
  "loggingConfiguration": {
    "level": "OFF",
    "includeExecutionData": false,
    "destinations": []
  },
  "tracingConfiguration": {
    "enabled": false
  }
}
```

---

### 4. Start an Execution

```bash
aws stepfunctions start-execution \
  --state-machine-arn "arn:aws:states:us-east-1:123456789012:stateMachine:my-order-processing-workflow" \
  --name "order-exec-$(date +%Y%m%d%H%M%S)" \
  --input '{"orderId": "ORD-12345", "customerId": "CUST-67890", "amount": 99.99}'
```

**Example Output:**

```json
{
  "executionArn": "arn:aws:states:us-east-1:123456789012:execution:my-order-processing-workflow:order-exec-20240115103045",
  "startDate": "2024-01-15T10:30:45.000Z"
}
```

---

### 5. Describe an Execution

```bash
aws stepfunctions describe-execution \
  --execution-arn "arn:aws:states:us-east-1:123456789012:execution:my-order-processing-workflow:order-exec-20240115103045"
```

**Example Output:**

```json
{
  "executionArn": "arn:aws:states:us-east-1:123456789012:execution:my-order-processing-workflow:order-exec-20240115103045",
  "stateMachineArn": "arn:aws:states:us-east-1:123456789012:stateMachine:my-order-processing-workflow",
  "name": "order-exec-20240115103045",
  "status": "SUCCEEDED",
  "startDate": "2024-01-15T10:30:45.000Z",
  "stopDate": "2024-01-15T10:31:02.000Z",
  "input": "{\"orderId\": \"ORD-12345\", \"customerId\": \"CUST-67890\", \"amount\": 99.99}",
  "output": "{\"status\": \"shipped\", \"trackingId\": \"TRACK-999\"}",
  "inputDetails": {
    "included": true
  },
  "outputDetails": {
    "included": true
  }
}
```

---

### 6. List Executions

```bash
aws stepfunctions list-executions \
  --state-machine-arn "arn:aws:states:us-east-1:123456789012:stateMachine:my-order-processing-workflow" \
  --status-filter RUNNING \
  --max-results 50
```

**Available status filters:** `RUNNING`, `SUCCEEDED`, `FAILED`, `TIMED_OUT`, `ABORTED`, `PENDING_REDRIVE`

**Example Output:**

```json
{
  "executions": [
    {
      "executionArn": "arn:aws:states:us-east-1:123456789012:execution:my-order-processing-workflow:order-exec-20240115103045",
      "stateMachineArn": "arn:aws:states:us-east-1:123456789012:stateMachine:my-order-processing-workflow",
      "name": "order-exec-20240115103045",
      "status": "RUNNING",
      "startDate": "2024-01-15T10:30:45.000Z"
    }
  ]
}
```

---

### 7. Get Execution History

```bash
aws stepfunctions get-execution-history \
  --execution-arn "arn:aws:states:us-east-1:123456789012:execution:my-order-processing-workflow:order-exec-20240115103045" \
  --max-results 100 \
  --reverse-order
```

**Example Output:**

```json
{
  "events": [
    {
      "timestamp": "2024-01-15T10:31:02.000Z",
      "type": "ExecutionSucceeded",
      "id": 8,
      "previousEventId": 7,
      "executionSucceededEventDetails": {
        "output": "{\"status\": \"shipped\", \"trackingId\": \"TRACK-999\"}",
        "outputDetails": {
          "included": true
        }
      }
    },
    {
      "timestamp": "2024-01-15T10:30:58.000Z",
      "type": "TaskSucceeded",
      "id": 7,
      "previousEventId": 6,
      "taskSucceededEventDetails": {
        "resourceType": "lambda",
        "resource": "invoke",
        "output": "{\"status\": \"shipped\"}",
        "outputDetails": {
          "included": true
        }
      }
    }
  ]
}
```

---

### 8. Stop an Execution

```bash
aws stepfunctions stop-execution \
  --execution-arn "arn:aws:states:us-east-1:123456789012:execution:my-order-processing-workflow:order-exec-20240115103045" \
  --error "ManualStop" \
  --cause "Stopped manually due to incorrect input data"
```

**Example Output:**

```json
{
  "stopDate": "2024-01-15T10:35:00.000Z"
}
```

---

### 9. Update a State Machine

```bash
aws stepfunctions update-state-machine \
  --state-machine-arn "arn:aws:states:us-east-1:123456789012:stateMachine:my-order-processing-workflow" \
  --definition file://updated-state-machine-definition.json \
  --role-arn "arn:aws:iam::123456789012:role/StepFunctionsExecutionRole" \
  --logging-configuration '{
    "level": "ALL",
    "includeExecutionData": true,
    "destinations": [
      {
        "cloudWatchLogsLogGroup": {
          "logGroupArn": "arn:aws:logs:us-east-1:123456789012:log-group:/aws/states/my-order-processing-workflow:*"
        }
      }
    ]
  }'
```

**Example Output:**

```json
{
  "updateDate": "2024-01-15T11:00:00.000Z",
  "revisionId": "2a3b4c5d-6e7f-8a9b-0c1d-2e3f4a5b6c7d"
}
```

---

### 10. Delete a State Machine

```bash
aws stepfunctions delete-state-machine \
  --state-machine-arn "arn:aws:states:us-east-1:123456789012:stateMachine:my-order-processing-workflow"
```

> ⚠️ This command returns no output on success. Running executions continue until completion.

---

### 11. Create an Activity

```bash
aws stepfunctions create-activity \
  --name "my-manual-approval-activity" \
  --tags key=Environment,value=Production
```

**Example Output:**

```json
{
  "activityArn": "arn:aws:states:us-east-1:123456789012:activity:my-manual-approval-activity",
  "creationDate": "2024-01-15T10:00:00.000Z"
}
```

---

### 12. Get Activity Task (Poll for Work)

```bash
aws stepfunctions get-activity-task \
  --activity-arn "arn:aws:states:us-east-1:123456789012:activity:my-manual-approval-activity" \
  --worker-name "approval-worker-01"
```

**Example Output:**

```json
{
  "taskToken": "AAAAKgAAAAIAAAAAAAAAAX...<long-token>...",
  "input": "{\"orderId\": \"ORD-12345\", \"approvalRequired\": true}"
}
```

---

### 13. Send Task Success

```bash
aws stepfunctions send-task-success \
  --task-token "AAAAKgAAAAIAAAAAAAAAAX...<long-token>..." \
  --task-output '{"approved": true, "approvedBy": "manager@example.com", "timestamp": "2024-01-15T11:00:00Z"}'
```

---

### 14. Send Task Failure

```bash
aws stepfunctions send-task-failure \
  --task-token "AAAAKgAAAAIAAAAAAAAAAX...<long-token>..." \
  --error "ApprovalRejected" \
  --cause "Order amount exceeds approval threshold"
```

---

### 15. List Tags for a Resource

```bash
aws stepfunctions list-tags-for-resource \
  --resource-arn "arn:aws:states:us-east-1:123456789012:stateMachine:my-order-processing-workflow"
```

**Example Output:**

```json
{
  "tags": [
    {
      "key": "Environment",
      "value": "Production"
    },
    {
      "key": "Team",
      "value": "Backend"
    }
  ]
}
```

---

## Common Operations

### Create Operations

```bash
# Create a STANDARD state machine
aws stepfunctions create