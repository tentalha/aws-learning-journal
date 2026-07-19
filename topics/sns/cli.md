# SNS — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using AWS CLI with SNS, ensure the following:

1. **AWS CLI installed and configured**
```bash
aws --version
aws configure
```

2. **Required IAM Permissions**

Attach the following managed policies or equivalent inline permissions:

| Policy | Use Case |
|---|---|
| `AmazonSNSFullAccess` | Full SNS management |
| `AmazonSNSReadOnlyAccess` | Read-only access |
| `AmazonSNS_SMSSandbox` | SMS sandbox management |

3. **Minimum IAM Policy for Common Operations**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sns:CreateTopic",
        "sns:DeleteTopic",
        "sns:Subscribe",
        "sns:Unsubscribe",
        "sns:Publish",
        "sns:ListTopics",
        "sns:ListSubscriptions",
        "sns:GetTopicAttributes",
        "sns:SetTopicAttributes",
        "sns:GetSubscriptionAttributes",
        "sns:SetSubscriptionAttributes",
        "sns:ListSubscriptionsByTopic",
        "sns:ConfirmSubscription",
        "sns:AddPermission",
        "sns:RemovePermission"
      ],
      "Resource": "*"
    }
  ]
}
```

4. **Set default region and output format**
```bash
export AWS_DEFAULT_REGION=us-east-1
export AWS_DEFAULT_OUTPUT=json
```

5. **Verify access**
```bash
aws sns list-topics
```

---

## Core Commands

### 1. Create a Standard SNS Topic
```bash
aws sns create-topic \
  --name my-alerts-topic \
  --tags Key=Environment,Value=production Key=Team,Value=devops
```

**What it does:** Creates a new SNS topic and returns its Amazon Resource Name (ARN).

**Example Output:**
```json
{
    "TopicArn": "arn:aws:sns:us-east-1:123456789012:my-alerts-topic"
}
```

---

### 2. Create a FIFO SNS Topic
```bash
aws sns create-topic \
  --name my-ordered-topic.fifo \
  --attributes FifoTopic=true,ContentBasedDeduplication=true
```

**What it does:** Creates a FIFO (First-In-First-Out) topic that preserves message ordering and deduplicates messages automatically. FIFO topic names must end in `.fifo`.

**Example Output:**
```json
{
    "TopicArn": "arn:aws:sns:us-east-1:123456789012:my-ordered-topic.fifo"
}
```

---

### 3. List All SNS Topics
```bash
aws sns list-topics
```

**What it does:** Returns a list of all SNS topics in the current account and region.

**Example Output:**
```json
{
    "Topics": [
        {
            "TopicArn": "arn:aws:sns:us-east-1:123456789012:my-alerts-topic"
        },
        {
            "TopicArn": "arn:aws:sns:us-east-1:123456789012:my-ordered-topic.fifo"
        }
    ]
}
```

---

### 4. Get Topic Attributes
```bash
aws sns get-topic-attributes \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic
```

**What it does:** Retrieves all attributes for a specified topic, including owner, subscriptions count, delivery policy, and KMS key info.

**Example Output:**
```json
{
    "Attributes": {
        "SubscriptionsConfirmed": "2",
        "DisplayName": "",
        "SubscriptionsDeleted": "0",
        "EffectiveDeliveryPolicy": "{\"http\":{\"defaultHealthyRetryPolicy\":{\"minDelayTarget\":20,\"maxDelayTarget\":20,\"numRetries\":3}}}",
        "Owner": "123456789012",
        "Policy": "{...}",
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:my-alerts-topic",
        "SubscriptionsPending": "0"
    }
}
```

---

### 5. Subscribe an Email Endpoint to a Topic
```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --protocol email \
  --notification-endpoint alerts@mycompany.com
```

**What it does:** Creates a subscription for an email endpoint. The subscriber receives a confirmation email and must confirm before messages are delivered.

**Example Output:**
```json
{
    "SubscriptionArn": "pending confirmation"
}
```

---

### 6. Subscribe an SQS Queue to a Topic
```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789012:my-processing-queue
```

**What it does:** Subscribes an SQS queue to receive messages from the topic. No confirmation is required for AWS service endpoints.

**Example Output:**
```json
{
    "SubscriptionArn": "arn:aws:sns:us-east-1:123456789012:my-alerts-topic:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111"
}
```

---

### 7. Subscribe a Lambda Function to a Topic
```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:us-east-1:123456789012:function:my-processor-function
```

**What it does:** Subscribes an AWS Lambda function to the topic so it's invoked whenever a message is published.

---

### 8. Publish a Message to a Topic
```bash
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --subject "System Alert" \
  --message "CPU utilization exceeded 90% on instance i-0abc123def456789."
```

**What it does:** Sends a message to all confirmed subscribers of the specified topic.

**Example Output:**
```json
{
    "MessageId": "a1b2c3d4-5678-90ab-cdef-EXAMPLE22222"
}
```

---

### 9. Publish a Message with Message Attributes
```bash
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --message "Deployment completed successfully." \
  --message-attributes '{
    "environment": {
      "DataType": "String",
      "StringValue": "production"
    },
    "severity": {
      "DataType": "String",
      "StringValue": "info"
    }
  }'
```

**What it does:** Publishes a message with custom attributes that can be used for subscription filter policies.

---

### 10. List Subscriptions for a Topic
```bash
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic
```

**What it does:** Returns all subscriptions associated with a specific topic.

**Example Output:**
```json
{
    "Subscriptions": [
        {
            "SubscriptionArn": "arn:aws:sns:us-east-1:123456789012:my-alerts-topic:a1b2c3d4-...",
            "Owner": "123456789012",
            "Protocol": "email",
            "Endpoint": "alerts@mycompany.com",
            "TopicArn": "arn:aws:sns:us-east-1:123456789012:my-alerts-topic"
        }
    ]
}
```

---

### 11. Get Subscription Attributes
```bash
aws sns get-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111
```

**What it does:** Returns all attributes for a specific subscription, including filter policy, delivery policy, and raw message delivery setting.

---

### 12. Set a Subscription Filter Policy
```bash
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111 \
  --attribute-name FilterPolicy \
  --attribute-value '{"severity": ["critical", "high"], "environment": ["production"]}'
```

**What it does:** Applies a filter policy to a subscription so it only receives messages matching the specified attribute conditions.

---

### 13. Unsubscribe from a Topic
```bash
aws sns unsubscribe \
  --subscription-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111
```

**What it does:** Removes the specified subscription. The endpoint will no longer receive messages from the topic.

---

### 14. Delete a Topic
```bash
aws sns delete-topic \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic
```

**What it does:** Permanently deletes the topic and all its subscriptions. This action is irreversible.

---

### 15. Send an SMS Message Directly
```bash
aws sns publish \
  --phone-number +15551234567 \
  --message "Your verification code is: 483920" \
  --message-attributes '{
    "AWS.SNS.SMS.SMSType": {
      "DataType": "String",
      "StringValue": "Transactional"
    },
    "AWS.SNS.SMS.SenderID": {
      "DataType": "String",
      "StringValue": "MyApp"
    }
  }'
```

**What it does:** Publishes an SMS message directly to a phone number without requiring a topic subscription.

---

## Common Operations

### Create

**Create a topic with encryption (KMS)**
```bash
aws sns create-topic \
  --name my-secure-topic \
  --attributes KmsMasterKeyId=arn:aws:kms:us-east-1:123456789012:key/my-key-id
```

**Create a topic with a display name**
```bash
aws sns set-topic-attributes \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --attribute-name DisplayName \
  --attribute-value "Production Alerts"
```

**Subscribe an HTTPS endpoint**
```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --protocol https \
  --notification-endpoint https://webhook.mycompany.com/sns-receiver
```

---

### Read / Describe

**List all subscriptions across all topics**
```bash
aws sns list-subscriptions
```

**Get platform application attributes (for mobile push)**
```bash
aws sns get-platform-application-attributes \
  --platform-application-arn arn:aws:sns:us-east-1:123456789012:app/GCM/my-android-app
```

**Get endpoint attributes (mobile push endpoint)**
```bash
aws sns get-endpoint-attributes \
  --endpoint-arn arn:aws:sns:us-east-1:123456789012:endpoint/GCM/my-android-app/a1b2c3d4-5678-90ab-cdef-EXAMPLE33333
```

**List platform applications**
```bash
aws sns list-platform-applications
```

---

### Update

**Enable raw message delivery on a subscription**
```bash
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111 \
  --attribute-name RawMessageDelivery \
  --attribute-value true
```

**Update topic delivery retry policy**
```bash
aws sns set-topic-attributes \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --attribute-name DeliveryPolicy \
  --attribute-value '{
    "http": {
      "defaultHealthyRetryPolicy": {
        "minDelayTarget": 20,
        "maxDelayTarget": 20,
        "numRetries": 10,
        "numMaxDelayRetries": 0,
        "backoffFunction": "linear"
      },
      "disableSubscriptionOverrides": false
    }
  }'
```

**Update platform endpoint token (mobile push)**
```bash
aws sns set-endpoint-attributes \
  --endpoint-arn arn:aws:sns:us-east-1:123456789012:endpoint/GCM/my-android-app/a1b2c3d4-5678-90ab-cdef-EXAMPLE33333 \
  --attributes Token=new-device-token-from-fcm,Enabled=true
```

---

### Delete

**Delete a subscription**
```bash
aws sns unsubscribe \
  --subscription-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111
```

**Delete a platform application**
```bash
aws sns delete-platform-application \
  --platform-application-arn arn:aws:sns:us-east-1:123456789012:app/GCM/my-android-app
```

**Delete a platform endpoint**
```bash
aws sns delete-endpoint \
  --endpoint-arn arn:aws:sns:us-east-1:123456789012:endpoint/GCM/my-android-app/a1b2c3d4-5678-90ab-cdef-EXAMPLE33333
```

---

### List

**List all topics with pagination**
```bash
aws sns list-topics \
  --output table \
  --query 'Topics[*].TopicArn'
```

**List all subscriptions in table format**
```bash
aws sns list-subscriptions \
  --output table \
  --query 'Subscriptions[*].{Protocol:Protocol,Endpoint:Endpoint,Topic:TopicArn}'
```

**List subscriptions by protocol using JMESPath**
```bash
aws sns list-subscriptions \
  --query "Subscriptions[?Protocol=='email']" \
  --output table
```

---

## Advanced Commands

### 1. Publish a Message with Per-Protocol Custom Payloads
```bash
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-alerts-topic \
  --message-structure json \
  --subject "Deployment Alert" \
  --message '{
    "default": "Deployment completed on production.",
    "email": "Dear Team,\n\nDeployment v2.4.1 completed successfully on production at 14:35 UTC.\n\nRegards,\nDevOps Bot",
    "sqs": "{\"event\":\"deployment\",\"version\":\"v2.4.1\",\"status\":\"success\",\"timestamp\":\"2024-01-15T14:35:00Z\"}",
    "lambda": "{\"event\":\"deployment\",\"version\":\"v2.4.1\",\"status\":\"success\"}",
    "http": "{\"event\":\"deployment\",\"version\":\"v2.4.1\"}"
  }'
```