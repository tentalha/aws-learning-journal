# EventBridge — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using the AWS CLI with EventBridge, ensure the following are in place:

**Install and configure the AWS CLI:**
```bash
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

**Verify CLI version (EventBridge requires AWS CLI v2 for full feature support):**
```bash
aws --version
# aws-cli/2.13.0 Python/3.11.4 Linux/5.15.0 exe/x86_64.ubuntu.22
```

### Required IAM Permissions

Attach the following IAM policy to your user or role for full EventBridge access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "events:PutRule",
        "events:PutTargets",
        "events:RemoveTargets",
        "events:DeleteRule",
        "events:DescribeRule",
        "events:ListRules",
        "events:ListTargetsByRule",
        "events:EnableRule",
        "events:DisableRule",
        "events:PutEvents",
        "events:CreateEventBus",
        "events:DeleteEventBus",
        "events:DescribeEventBus",
        "events:ListEventBuses",
        "events:CreateArchive",
        "events:UpdateArchive",
        "events:DeleteArchive",
        "events:DescribeArchive",
        "events:ListArchives",
        "events:StartReplay",
        "events:DescribeReplay",
        "events:ListReplays",
        "events:CreateConnection",
        "events:CreateApiDestination",
        "events:TagResource",
        "events:UntagResource",
        "events:ListTagsForResource",
        "schemas:*"
      ],
      "Resource": "*"
    }
  ]
}
```

**Set a default region for all commands:**
```bash
export AWS_DEFAULT_REGION=us-east-1
export AWS_PROFILE=my-devops-profile
```

---

## Core Commands

### 1. Create an Event Bus

```bash
aws events create-event-bus \
  --name my-custom-event-bus \
  --tags Key=Environment,Value=Production Key=Team,Value=Platform
```

**What it does:** Creates a custom event bus. The default event bus already exists in every account; custom buses are used to isolate event flows between teams or applications.

**Example Output:**
```json
{
    "EventBusArn": "arn:aws:events:us-east-1:123456789012:event-bus/my-custom-event-bus"
}
```

---

### 2. List All Event Buses

```bash
aws events list-event-buses \
  --name-prefix my- \
  --limit 20
```

**What it does:** Lists all event buses in the current account and region, optionally filtered by a name prefix.

**Example Output:**
```json
{
    "EventBuses": [
        {
            "Name": "default",
            "Arn": "arn:aws:events:us-east-1:123456789012:event-bus/default"
        },
        {
            "Name": "my-custom-event-bus",
            "Arn": "arn:aws:events:us-east-1:123456789012:event-bus/my-custom-event-bus",
            "Tags": [
                { "Key": "Environment", "Value": "Production" }
            ]
        }
    ]
}
```

---

### 3. Create an EventBridge Rule (Schedule-Based)

```bash
aws events put-rule \
  --name my-scheduled-rule \
  --schedule-expression "rate(5 minutes)" \
  --state ENABLED \
  --description "Triggers every 5 minutes" \
  --event-bus-name default
```

**What it does:** Creates a scheduled rule that fires on a fixed rate. Supports `rate()` and `cron()` expressions.

**Example Output:**
```json
{
    "RuleArn": "arn:aws:events:us-east-1:123456789012:rule/my-scheduled-rule"
}
```

---

### 4. Create an EventBridge Rule (Event Pattern-Based)

```bash
aws events put-rule \
  --name my-s3-event-rule \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["Object Created"],
    "detail": {
      "bucket": {
        "name": ["my-source-bucket"]
      }
    }
  }' \
  --state ENABLED \
  --description "Triggers on S3 object creation in my-source-bucket" \
  --event-bus-name default
```

**What it does:** Creates a rule that matches events based on a JSON event pattern. This rule triggers when an object is created in `my-source-bucket`.

---

### 5. Add a Target to a Rule

```bash
aws events put-targets \
  --rule my-scheduled-rule \
  --event-bus-name default \
  --targets '[
    {
      "Id": "LambdaTarget1",
      "Arn": "arn:aws:lambda:us-east-1:123456789012:function:my-processor-function",
      "Input": "{\"key\": \"value\", \"environment\": \"production\"}"
    }
  ]'
```

**What it does:** Associates one or more targets with a rule. When the rule matches, EventBridge sends the event to all configured targets. Up to 5 targets per rule are supported.

**Example Output:**
```json
{
    "FailedEntryCount": 0,
    "FailedEntries": []
}
```

---

### 6. Describe a Rule

```bash
aws events describe-rule \
  --name my-scheduled-rule \
  --event-bus-name default
```

**What it does:** Returns detailed configuration information about a specific rule including its ARN, state, schedule/pattern, and role.

**Example Output:**
```json
{
    "Name": "my-scheduled-rule",
    "Arn": "arn:aws:events:us-east-1:123456789012:rule/my-scheduled-rule",
    "ScheduleExpression": "rate(5 minutes)",
    "State": "ENABLED",
    "Description": "Triggers every 5 minutes",
    "EventBusName": "default",
    "CreatedBy": "123456789012"
}
```

---

### 7. List Targets for a Rule

```bash
aws events list-targets-by-rule \
  --rule my-scheduled-rule \
  --event-bus-name default
```

**What it does:** Lists all targets associated with a specific rule, including their IDs, ARNs, and any input transformations.

**Example Output:**
```json
{
    "Targets": [
        {
            "Id": "LambdaTarget1",
            "Arn": "arn:aws:lambda:us-east-1:123456789012:function:my-processor-function",
            "Input": "{\"key\": \"value\"}"
        }
    ]
}
```

---

### 8. Send Custom Events to an Event Bus

```bash
aws events put-events \
  --entries '[
    {
      "Source": "com.mycompany.orders",
      "DetailType": "OrderPlaced",
      "Detail": "{\"orderId\": \"ORD-12345\", \"customerId\": \"CUST-67890\", \"amount\": 199.99}",
      "EventBusName": "my-custom-event-bus",
      "Time": "2024-01-15T10:30:00Z",
      "Resources": ["arn:aws:dynamodb:us-east-1:123456789012:table/orders-table"]
    }
  ]'
```

**What it does:** Publishes one or more custom events to an event bus. This is the primary way to inject application-generated events into EventBridge.

**Example Output:**
```json
{
    "FailedEntryCount": 0,
    "Entries": [
        {
            "EventId": "11710aed-b79e-4468-a20b-bb3c0c3b4860"
        }
    ]
}
```

---

### 9. Disable a Rule

```bash
aws events disable-rule \
  --name my-scheduled-rule \
  --event-bus-name default
```

**What it does:** Disables a rule, preventing it from triggering targets. The rule configuration is preserved and can be re-enabled at any time.

---

### 10. Enable a Rule

```bash
aws events enable-rule \
  --name my-scheduled-rule \
  --event-bus-name default
```

**What it does:** Re-enables a previously disabled rule, allowing it to match and route events again.

---

### 11. List All Rules

```bash
aws events list-rules \
  --name-prefix my- \
  --event-bus-name default \
  --limit 50
```

**What it does:** Lists all rules on an event bus, optionally filtered by a name prefix. Supports pagination.

**Example Output:**
```json
{
    "Rules": [
        {
            "Name": "my-scheduled-rule",
            "Arn": "arn:aws:events:us-east-1:123456789012:rule/my-scheduled-rule",
            "State": "ENABLED",
            "ScheduleExpression": "rate(5 minutes)",
            "EventBusName": "default"
        },
        {
            "Name": "my-s3-event-rule",
            "Arn": "arn:aws:events:us-east-1:123456789012:rule/my-s3-event-rule",
            "State": "ENABLED",
            "EventBusName": "default"
        }
    ]
}
```

---

### 12. Remove a Target from a Rule

```bash
aws events remove-targets \
  --rule my-scheduled-rule \
  --event-bus-name default \
  --ids LambdaTarget1
```

**What it does:** Removes one or more targets from a rule. The rule itself remains; only the target association is removed.

---

### 13. Delete a Rule

```bash
aws events delete-rule \
  --name my-scheduled-rule \
  --event-bus-name default \
  --force
```

**What it does:** Deletes a rule. The `--force` flag removes the rule even if it still has targets attached (otherwise you must remove targets first).

---

### 14. Create an Archive

```bash
aws events create-archive \
  --archive-name my-orders-archive \
  --event-source-arn "arn:aws:events:us-east-1:123456789012:event-bus/my-custom-event-bus" \
  --description "Archive all order events for 90 days" \
  --event-pattern '{"source": ["com.mycompany.orders"]}' \
  --retention-days 90
```

**What it does:** Creates an archive that stores events matching the specified pattern from the source event bus. Archived events can be replayed later.

**Example Output:**
```json
{
    "ArchiveArn": "arn:aws:events:us-east-1:123456789012:archive/my-orders-archive",
    "State": "ENABLED",
    "CreationTime": "2024-01-15T10:00:00+00:00"
}
```

---

### 15. Start an Event Replay

```bash
aws events start-replay \
  --replay-name my-orders-replay-20240115 \
  --description "Replay January 2024 order events" \
  --event-source-arn "arn:aws:events:us-east-1:123456789012:archive/my-orders-archive" \
  --event-start-time "2024-01-01T00:00:00Z" \
  --event-end-time "2024-01-15T23:59:59Z" \
  --destination '{
    "Arn": "arn:aws:events:us-east-1:123456789012:event-bus/my-custom-event-bus",
    "FilterArns": [
      "arn:aws:events:us-east-1:123456789012:rule/my-custom-event-bus/my-s3-event-rule"
    ]
  }'
```

**What it does:** Replays archived events back to an event bus within a specified time range. Useful for reprocessing events after a bug fix or new consumer deployment.

---

## Common Operations

### Create Operations

**Create a custom event bus with a resource policy:**
```bash
# Create the event bus
aws events create-event-bus \
  --name my-shared-event-bus

# Attach a resource-based policy to allow cross-account publishing
aws events put-permission \
  --event-bus-name my-shared-event-bus \
  --action events:PutEvents \
  --principal "111122223333" \
  --statement-id AllowAccountPublish
```

**Create a rule with a cron schedule:**
```bash
aws events put-rule \
  --name my-daily-report-rule \
  --schedule-expression "cron(0 9 * * ? *)" \
  --state ENABLED \
  --description "Runs every day at 9:00 AM UTC" \
  --event-bus-name default
```

**Create a rule targeting an SQS queue with a dead-letter queue:**
```bash
aws events put-targets \
  --rule my-s3-event-rule \
  --event-bus-name default \
  --targets '[
    {
      "Id": "SQSTarget1",
      "Arn": "arn:aws:sqs:us-east-1:123456789012:my-processing-queue",
      "DeadLetterConfig": {
        "Arn": "arn:aws:sqs:us-east-1:123456789012:my-dlq"
      },
      "RetryPolicy": {
        "MaximumRetryAttempts": 3,
        "MaximumEventAgeInSeconds": 3600
      }
    }
  ]'
```

**Create a connection for API destinations:**
```bash
aws events create-connection \
  --name my-api-connection \
  --description "Connection to external order API" \
  --authorization-type API_KEY \
  --auth-parameters '{
    "ApiKeyAuthParameters": {
      "ApiKeyName": "x-api-key",
      "ApiKeyValue": "my-secret-api-key-value"
    }
  }'
```

**Create an API destination:**
```bash
aws events create-api-destination \
  --name my-order-api-destination \
  --description "Sends events to external order management system" \
  --invocation-endpoint "https://api.mycompany.com/v1/orders/events" \
  --http-method POST \
  --connection-arn "arn:aws:events:us-east-1:123456789012:connection/my-api-connection/abc12345-1234-1234-1234-abc123456789" \
  --invocation-rate-limit-per-second 300
```

---

### Read / Describe Operations

**Describe an event bus:**
```bash
aws events describe-event-bus \
  --name my-custom-event-bus
```

**Describe an archive:**
```bash
aws events describe-archive \
  --archive-name my-orders-archive
```

**Describe a replay:**
```bash
aws events describe-replay \
  --replay-name my-orders-replay-20240115
```

**Describe a connection:**
```bash
aws events describe-connection \
  --name my-api-connection
```

**Describe an API destination:**
```bash
aws events describe-api-destination \
  --name my-order-api-destination
```

---

### List Operations

**List all rules across all buses