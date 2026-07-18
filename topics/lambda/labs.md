# Lambda — Hands-On Labs

## Lab 1: Getting Started with Lambda

### Objective

In this lab, you will create your first AWS Lambda function from scratch. You will learn how to write a simple Python function, configure its execution role, invoke it manually, and inspect logs in CloudWatch. By the end, you will understand the core Lambda workflow: write → deploy → invoke → observe.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account with console and CLI access |
| IAM Permissions | `AWSLambdaFullAccess`, `IAMFullAccess`, `CloudWatchLogsReadOnlyAccess` |
| AWS CLI | Version 2.x installed and configured (`aws configure`) |
| Runtime | Python 3.12 (no local install needed for console) |
| Region | `us-east-1` (all labs use this region unless noted) |

---

### Steps

#### Step 1: Create an IAM Execution Role for Lambda

Lambda needs an IAM role to interact with AWS services. This role is assumed by the Lambda service at runtime.

**Console:**

1. Navigate to **IAM → Roles → Create role**.
2. Select **Trusted entity type**: `AWS service`.
3. Select **Use case**: `Lambda`. Click **Next**.
4. Search for and attach the policy: `AWSLambdaBasicExecutionRole`.
5. Click **Next**.
6. Set **Role name**: `lambda-lab1-basic-role`.
7. Click **Create role**.
8. Copy the **Role ARN** — you will need it shortly.

**CLI:**

```bash
# Create the trust policy document
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
  --role-name lambda-lab1-basic-role \
  --assume-role-policy-document file:///tmp/lambda-trust-policy.json \
  --region us-east-1

# Attach the basic execution policy
aws iam attach-role-policy \
  --role-name lambda-lab1-basic-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Capture the Role ARN
ROLE_ARN=$(aws iam get-role \
  --role-name lambda-lab1-basic-role \
  --query 'Role.Arn' \
  --output text)

echo "Role ARN: $ROLE_ARN"
```

**✅ Verify:** In IAM console, confirm `lambda-lab1-basic-role` exists with the `AWSLambdaBasicExecutionRole` policy attached. CLI output should show `"RoleName": "lambda-lab1-basic-role"`.

---

#### Step 2: Write the Lambda Function Code

**Console:** You will write the code directly in the inline editor in Step 3.

**CLI — Create the deployment package:**

```bash
# Create the function source file
mkdir -p /tmp/lambda-lab1
cat > /tmp/lambda-lab1/lambda_function.py << 'EOF'
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """
    Simple Lambda function that echoes the input event
    and returns a greeting message.
    """
    logger.info(f"Received event: {json.dumps(event)}")
    
    name = event.get("name", "World")
    message = f"Hello, {name}! This is your first Lambda function."
    
    logger.info(f"Returning message: {message}")
    
    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": message,
            "input_event": event
        })
    }
EOF

# Package the function into a ZIP file
cd /tmp/lambda-lab1
zip -j /tmp/lambda-lab1.zip lambda_function.py

echo "Package created: /tmp/lambda-lab1.zip"
ls -lh /tmp/lambda-lab1.zip
```

**Expected output:**
```
Package created: /tmp/lambda-lab1.zip
-rw-r--r-- 1 user group 512 Jan 01 00:00 /tmp/lambda-lab1.zip
```

---

#### Step 3: Create the Lambda Function

**Console:**

1. Navigate to **Lambda → Functions → Create function**.
2. Select **Author from scratch**.
3. Configure:
   - **Function name**: `lab1-hello-world`
   - **Runtime**: `Python 3.12`
   - **Architecture**: `x86_64`
4. Under **Permissions → Change default execution role**:
   - Select **Use an existing role**
   - Choose `lambda-lab1-basic-role`
5. Click **Create function**.
6. In the **Code source** editor, replace the default code with the Python code from Step 2.
7. Click **Deploy**.

**CLI:**

```bash
# Wait a few seconds for the IAM role to propagate
sleep 10

# Create the Lambda function
aws lambda create-function \
  --function-name lab1-hello-world \
  --runtime python3.12 \
  --role $ROLE_ARN \
  --handler lambda_function.lambda_handler \
  --zip-file fileb:///tmp/lambda-lab1.zip \
  --description "Lab 1: Hello World Lambda function" \
  --timeout 30 \
  --memory-size 128 \
  --region us-east-1

# Wait for the function to become active
aws lambda wait function-active \
  --function-name lab1-hello-world \
  --region us-east-1

echo "Function is active and ready."
```

**Expected output:**
```json
{
    "FunctionName": "lab1-hello-world",
    "FunctionArn": "arn:aws:lambda:us-east-1:123456789012:functions/lab1-hello-world",
    "Runtime": "python3.12",
    "State": "Active",
    ...
}
```

**✅ Verify:** In the Lambda console, confirm `lab1-hello-world` shows **State: Active**.

---

#### Step 4: Invoke the Lambda Function

**Console:**

1. Open `lab1-hello-world` in the Lambda console.
2. Click the **Test** tab.
3. Click **Create new test event**.
4. Set **Event name**: `TestGreeting`.
5. Replace the default JSON with:
   ```json
   {
     "name": "AWS Learner"
   }
   ```
6. Click **Save**, then click **Test**.
7. Observe the **Execution result** panel.

**CLI:**

```bash
# Invoke the function synchronously
aws lambda invoke \
  --function-name lab1-hello-world \
  --payload '{"name": "AWS Learner"}' \
  --cli-binary-format raw-in-base64-out \
  --log-type Tail \
  --region us-east-1 \
  /tmp/lambda-lab1-response.json

# Display the response
echo "=== Response Body ==="
cat /tmp/lambda-lab1-response.json

# Decode and display the logs
echo ""
echo "=== Execution Logs ==="
aws lambda invoke \
  --function-name lab1-hello-world \
  --payload '{"name": "AWS Learner"}' \
  --cli-binary-format raw-in-base64-out \
  --log-type Tail \
  --query 'LogResult' \
  --output text \
  --region us-east-1 \
  /tmp/lambda-lab1-response2.json | base64 --decode
```

**Expected output:**
```json
{
  "statusCode": 200,
  "body": "{\"message\": \"Hello, AWS Learner! This is your first Lambda function.\", \"input_event\": {\"name\": \"AWS Learner\"}}"
}
```

**✅ Verify:** The response contains `statusCode: 200` and the greeting message with the name you provided.

---

#### Step 5: View CloudWatch Logs

**Console:**

1. In the Lambda console, click the **Monitor** tab.
2. Click **View CloudWatch logs**.
3. Click the most recent **Log stream**.
4. Expand log entries to see `START`, `INFO`, and `END` records.

**CLI:**

```bash
# Get the log group name
LOG_GROUP="/aws/lambda/lab1-hello-world"

# List log streams (most recent first)
aws logs describe-log-streams \
  --log-group-name $LOG_GROUP \
  --order-by LastEventTime \
  --descending \
  --max-items 1 \
  --region us-east-1 \
  --query 'logStreams[0].logStreamName' \
  --output text

# Store the stream name
STREAM_NAME=$(aws logs describe-log-streams \
  --log-group-name $LOG_GROUP \
  --order-by LastEventTime \
  --descending \
  --max-items 1 \
  --region us-east-1 \
  --query 'logStreams[0].logStreamName' \
  --output text)

# Retrieve log events
aws logs get-log-events \
  --log-group-name $LOG_GROUP \
  --log-stream-name "$STREAM_NAME" \
  --region us-east-1 \
  --query 'events[*].message' \
  --output text
```

**Expected log output:**
```
START RequestId: abc-123 Version: $LATEST
Received event: {"name": "AWS Learner"}
Returning message: Hello, AWS Learner! This is your first Lambda function.
END RequestId: abc-123
REPORT RequestId: abc-123  Duration: 2.45 ms  Billed Duration: 3 ms  Memory Size: 128 MB  Max Memory Used: 36 MB
```

**✅ Verify:** You can see the `INFO` log lines from your function in CloudWatch.

---

### Verification

Run the following checklist to confirm Lab 1 is complete:

```bash
# Check function exists and is active
aws lambda get-function \
  --function-name lab1-hello-world \
  --region us-east-1 \
  --query '{Name: Configuration.FunctionName, State: Configuration.State, Runtime: Configuration.Runtime}'

# Confirm IAM role exists
aws iam get-role \
  --role-name lambda-lab1-basic-role \
  --query 'Role.RoleName'

# Confirm CloudWatch log group exists
aws logs describe-log-groups \
  --log-group-name-prefix /aws/lambda/lab1-hello-world \
  --region us-east-1 \
  --query 'logGroups[0].logGroupName'
```

All three commands should return expected values without errors. ✅

---

### Cleanup

```bash
# Delete the Lambda function
aws lambda delete-function \
  --function-name lab1-hello-world \
  --region us-east-1

# Detach the policy from the role
aws iam detach-role-policy \
  --role-name lambda-lab1-basic-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Delete the IAM role
aws iam delete-role \
  --role-name lambda-lab1-basic-role

# Delete the CloudWatch log group
aws logs delete-log-group \
  --log-group-name /aws/lambda/lab1-hello-world \
  --region us-east-1

# Remove local temp files
rm -rf /tmp/lambda-lab1 /tmp/lambda-lab1.zip /tmp/lambda-lab1-response*.json /tmp/lambda-trust-policy.json

echo "✅ Lab 1 cleanup complete."
```

---

## Lab 2: Intermediate Lambda Configuration

### Objective

In this lab, you will build a more realistic Lambda function that integrates with **Amazon S3** and **Amazon DynamoDB**. The function will be triggered by an S3 `PUT` event, read the uploaded object's metadata, write a record to DynamoDB, and use environment variables for configuration. You will also configure Lambda **layers**, **reserved concurrency**, and **dead-letter queues (DLQ)** using SQS. You will monitor execution with **Lambda Insights** and structured JSON logging.

---

### Prerequisites

| Requirement | Details |
|---|---|
| IAM Permissions | `AWSLambdaFullAccess`, `AmazonS3FullAccess`, `AmazonDynamoDBFullAccess`, `AmazonSQSFullAccess`, `CloudWatchLambdaInsightsExecutionRolePolicy` |
| AWS CLI | v2.x configured for `us-east-1` |
| Python | 3.12 (local, for building the layer) |
| pip | Installed locally |
| Lab 1 | Not required — this is independent |

---

### Steps

#### Step 1: Create Supporting Infrastructure

**Create the S3 bucket:**

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="lambda-lab2-uploads-${ACCOUNT_ID}"

aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region us-east-1

# Block all public access
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

echo "S3 Bucket: $BUCKET_NAME"
```

**Create the DynamoDB table:**

```bash
aws dynamodb create-table \
  --table-name lab2-file-metadata \
  --attribute-definitions \
    AttributeName=file_key,AttributeType=S \
  --key-schema \
    AttributeName=file_key,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

# Wait for table to become active
aws dynamodb wait table-exists \
  --table-name lab2-file-metadata \
  --region us-east-1

echo "✅ DynamoDB table 'lab2-file-metadata' is active."
```

**Create the SQS Dead Letter Queue:**

```bash
DLQ_URL=$(aws sqs create-queue \
  --queue-name lambda-lab2-dlq \
  --region us-east-1 \
  --query 'QueueUrl' \
  --output text)

DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url $DLQ_URL \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)

echo "DLQ ARN: $DLQ_ARN"
```

**✅ Verify:** Confirm all three resources exist in the console: S3 bucket, DynamoDB table (status: Active), and SQS queue.

---

#### Step 2: Build a Lambda Layer with Third-Party Dependencies

Lambda Layers allow you to package shared dependencies separately from your function code, keeping deployment packages small.

```bash
# Create layer directory structure
mkdir -p /tmp/lambda-lab2-layer/python

# Install the 'aws-lambda-powertools' library into the layer
pip install \
  aws-lambda-powertools \
  --target /tmp/lambda-lab2-layer/python \
  --quiet

# Package the layer
cd /tmp/lambda-lab2-layer
zip -r /tmp/lambda-lab2-layer.zip python/

echo "Layer package size:"
ls -lh /tmp/lambda-lab2-layer.zip

# Publish the layer
LAYER_ARN=$(aws lambda publish-layer-version \
  --layer-name lab2-powertools-layer \
  --description "AWS Lambda Powertools for Python" \
  --zip-file fileb:///tmp/lambda-lab2-layer.