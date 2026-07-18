# Lambda — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using the AWS CLI with Lambda, ensure the following are in place:

- **AWS CLI v2** installed and configured (`aws configure`)
- **Default region** set (or use `--region` flag per command)
- **IAM permissions** attached to your user/role

### Required IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:CreateFunction",
        "lambda:UpdateFunctionCode",
        "lambda:UpdateFunctionConfiguration",
        "lambda:InvokeFunction",
        "lambda:GetFunction",
        "lambda:ListFunctions",
        "lambda:DeleteFunction",
        "lambda:AddPermission",
        "lambda:RemovePermission",
        "lambda:CreateAlias",
        "lambda:PublishVersion",
        "lambda:GetPolicy",
        "lambda:ListVersionsByFunction",
        "lambda:PutFunctionConcurrency",
        "lambda:TagResource",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

### Initial Configuration

```bash
# Configure AWS CLI with your credentials
aws configure

# Verify identity and active credentials
aws sts get-caller-identity

# Set a default region for all commands (optional)
export AWS_DEFAULT_REGION=us-east-1

# Confirm Lambda service is accessible
aws lambda list-functions --max-items 1
```

---

## Core Commands

### 1. Create a Lambda Function

```bash
aws lambda create-function \
  --function-name my-function \
  --runtime python3.11 \
  --role arn:aws:iam::123456789012:role/my-lambda-execution-role \
  --handler app.lambda_handler \
  --zip-file fileb://function.zip \
  --description "My first Lambda function" \
  --timeout 30 \
  --memory-size 256 \
  --environment "Variables={ENV=production,LOG_LEVEL=INFO}" \
  --tags "Project=MyApp,Team=Backend"
```

**What it does:** Creates a new Lambda function from a local deployment package (`.zip`). Specifies the runtime, IAM execution role, entry point handler, and environment variables.

**Example Output:**
```json
{
    "FunctionName": "my-function",
    "FunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:my-function",
    "Runtime": "python3.11",
    "Role": "arn:aws:iam::123456789012:role/my-lambda-execution-role",
    "Handler": "app.lambda_handler",
    "CodeSize": 1024,
    "Description": "My first Lambda function",
    "Timeout": 30,
    "MemorySize": 256,
    "State": "Pending",
    "StateReason": "The function is being created.",
    "StateReasonCode": "Creating"
}
```

---

### 2. Invoke a Lambda Function

```bash
aws lambda invoke \
  --function-name my-function \
  --payload '{"key1": "value1", "key2": "value2"}' \
  --cli-binary-format raw-in-base64-out \
  --log-type Tail \
  output.json
```

**What it does:** Synchronously invokes the Lambda function with a JSON payload and writes the response to `output.json`. The `--log-type Tail` flag returns the last 4 KB of execution logs in the response metadata.

**Example Output (to terminal):**
```json
{
    "StatusCode": 200,
    "LogResult": "U1RBUlQgUmVxdWVzdElkOi...(base64 encoded logs)",
    "ExecutedVersion": "$LATEST"
}
```

---

### 3. Get Function Details

```bash
aws lambda get-function \
  --function-name my-function
```

**What it does:** Returns metadata about the function including configuration, code location (S3 presigned URL), concurrency settings, and tags.

**Example Output:**
```json
{
    "Configuration": {
        "FunctionName": "my-function",
        "FunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:my-function",
        "Runtime": "python3.11",
        "MemorySize": 256,
        "Timeout": 30,
        "LastModified": "2024-01-15T10:30:00.000+0000",
        "State": "Active"
    },
    "Code": {
        "RepositoryType": "S3",
        "Location": "https://awslambda-us-east-1-tasks.s3.amazonaws.com/..."
    },
    "Tags": {
        "Project": "MyApp",
        "Team": "Backend"
    }
}
```

---

### 4. Update Function Code

```bash
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip \
  --publish
```

**What it does:** Deploys new code to an existing Lambda function from a local `.zip` file. The `--publish` flag creates a new numbered version after the update.

---

### 5. Update Function Configuration

```bash
aws lambda update-function-configuration \
  --function-name my-function \
  --timeout 60 \
  --memory-size 512 \
  --environment "Variables={ENV=production,LOG_LEVEL=DEBUG,DB_HOST=db.example.com}" \
  --description "Updated configuration for production"
```

**What it does:** Modifies the runtime settings of an existing function — timeout, memory, environment variables — without touching the code.

---

### 6. List All Lambda Functions

```bash
aws lambda list-functions \
  --max-items 50 \
  --query 'Functions[*].{Name:FunctionName,Runtime:Runtime,Memory:MemorySize,Timeout:Timeout,Modified:LastModified}' \
  --output table
```

**What it does:** Lists all Lambda functions in the current region with a formatted table output showing key properties.

**Example Output:**
```
-----------------------------------------------------------------------
|                          ListFunctions                              |
+------------------+-----------+--------+---------+------------------+
|     Modified     |  Memory   |  Name  | Runtime | Timeout          |
+------------------+-----------+--------+---------+------------------+
|  2024-01-15T...  |  256      | my-fn  | python3 | 30               |
+------------------+-----------+--------+---------+------------------+
```

---

### 7. Delete a Lambda Function

```bash
aws lambda delete-function \
  --function-name my-function
```

**What it does:** Permanently deletes the Lambda function and all its versions and aliases. This action is irreversible.

---

### 8. Publish a New Version

```bash
aws lambda publish-version \
  --function-name my-function \
  --description "Release v1.2.0 - Added retry logic"
```

**What it does:** Creates an immutable, numbered snapshot of the current `$LATEST` code and configuration. Useful for stable production deployments.

**Example Output:**
```json
{
    "FunctionName": "my-function",
    "FunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:my-function:3",
    "Version": "3",
    "Description": "Release v1.2.0 - Added retry logic",
    "State": "Active"
}
```

---

### 9. Create an Alias

```bash
aws lambda create-alias \
  --function-name my-function \
  --name production \
  --function-version 3 \
  --description "Production stable alias"
```

**What it does:** Creates a named pointer (alias) to a specific Lambda version. Aliases allow you to decouple callers from version numbers and support traffic shifting.

---

### 10. Add a Resource-Based Permission

```bash
aws lambda add-permission \
  --function-name my-function \
  --statement-id allow-s3-invoke \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::my-source-bucket \
  --source-account 123456789012
```

**What it does:** Grants an AWS service (S3 in this case) permission to invoke the Lambda function. Required for event source integrations like S3, SNS, API Gateway, etc.

---

### 11. Create an Event Source Mapping (SQS Trigger)

```bash
aws lambda create-event-source-mapping \
  --function-name my-function \
  --event-source-arn arn:aws:sqs:us-east-1:123456789012:my-queue \
  --batch-size 10 \
  --maximum-batching-window-in-seconds 5 \
  --enabled
```

**What it does:** Connects an SQS queue to the Lambda function as a trigger. Lambda will poll the queue and invoke the function with batches of up to 10 messages.

---

### 12. Set Reserved Concurrency

```bash
aws lambda put-function-concurrency \
  --function-name my-function \
  --reserved-concurrent-executions 100
```

**What it does:** Limits the maximum number of concurrent executions for the function. Setting this to `0` effectively throttles (disables) the function.

---

### 13. Get Function Logs (via CloudWatch)

```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/my-function \
  --start-time $(date -d '1 hour ago' +%s000) \
  --filter-pattern "ERROR" \
  --query 'events[*].message' \
  --output text
```

**What it does:** Queries CloudWatch Logs for ERROR-level messages from the Lambda function in the past hour.

---

### 14. List Function Versions

```bash
aws lambda list-versions-by-function \
  --function-name my-function \
  --query 'Versions[*].{Version:Version,Description:Description,Modified:LastModified}' \
  --output table
```

**What it does:** Lists all published versions of a Lambda function along with their descriptions and modification timestamps.

---

### 15. Get the Resource-Based Policy

```bash
aws lambda get-policy \
  --function-name my-function \
  --query 'Policy' \
  --output text | python3 -m json.tool
```

**What it does:** Retrieves and pretty-prints the resource-based policy attached to the function, showing all principals that have been granted invoke permissions.

---

## Common Operations

### Create

```bash
# Create function from S3 bucket
aws lambda create-function \
  --function-name my-function \
  --runtime nodejs20.x \
  --role arn:aws:iam::123456789012:role/my-lambda-execution-role \
  --handler index.handler \
  --code S3Bucket=my-deployment-bucket,S3Key=functions/my-function.zip \
  --timeout 15 \
  --memory-size 128

# Create a Lambda layer
aws lambda publish-layer-version \
  --layer-name my-dependencies-layer \
  --description "Shared Python dependencies" \
  --zip-file fileb://layer.zip \
  --compatible-runtimes python3.11 python3.12 \
  --compatible-architectures x86_64 arm64

# Create an alias with traffic shifting (canary deployment)
aws lambda create-alias \
  --function-name my-function \
  --name production \
  --function-version 5 \
  --routing-config AdditionalVersionWeights={"4"=0.1}
```

---

### Read / Describe

```bash
# Get full function configuration only
aws lambda get-function-configuration \
  --function-name my-function

# Get function configuration for a specific version
aws lambda get-function-configuration \
  --function-name my-function \
  --qualifier 3

# Get function URL configuration
aws lambda get-function-url-config \
  --function-name my-function

# List all aliases for a function
aws lambda list-aliases \
  --function-name my-function \
  --output table

# Get a specific alias
aws lambda get-alias \
  --function-name my-function \
  --name production

# List all event source mappings
aws lambda list-event-source-mappings \
  --function-name my-function

# Get concurrency settings
aws lambda get-function-concurrency \
  --function-name my-function

# List layers available in your account/region
aws lambda list-layers \
  --compatible-runtime python3.11 \
  --query 'Layers[*].{Name:LayerName,ARN:LatestMatchingVersion.LayerVersionArn}' \
  --output table
```

---

### Update

```bash
# Update function code from S3
aws lambda update-function-code \
  --function-name my-function \
  --s3-bucket my-deployment-bucket \
  --s3-key functions/my-function-v2.zip

# Update function code for a container image
aws lambda update-function-code \
  --function-name my-function \
  --image-uri 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-repo:latest

# Update alias to point to new version
aws lambda update-alias \
  --function-name my-function \
  --name production \
  --function-version 6

# Update alias traffic shifting (10% to new version)
aws lambda update-alias \
  --function-name my-function \
  --name production \
  --function-version 6 \
  --routing-config AdditionalVersionWeights={"5"=0.10}

# Update event source mapping (e.g., change batch size)
aws lambda update-event-source-mapping \
  --uuid a1b2c3d4-e5f6-7890-abcd-ef1234567890 \
  --batch-size 25

# Tag a Lambda function
aws lambda tag-resource \
  --resource arn:aws:lambda:us-east-1:123456789012:function:my-function \
  --tags "CostCenter=engineering,Environment=production"
```

---

### Delete

```bash
# Delete a specific function version
aws lambda delete-function \
  --function-name my-function \
  --qualifier 2

# Delete an alias
aws lambda delete-alias \
  --function-name my-function \
  --name staging

# Delete an event source mapping
aws lambda delete-event-source-mapping \
  --uuid a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Remove a resource-based permission statement
aws lambda remove-permission \
  --function-name my-function \
  --statement-id allow-s3-invoke

# Delete a layer version
aws lambda delete-layer-version \
  --layer-name my-dependencies-layer \
  --version-number 3

# Remove reserved concurrency (restores unreserved pool)
aws lambda delete-function-concurrency \
  --function-name my-function

# Remove tags from a function
aws lambda untag-resource \
  --resource arn:aws:lambda:us-east-1:123456789012:function:my-function \
  --tag-keys "CostCenter" "Environment"
```

---

### List

```bash
# List all functions with pagination
aws lambda list-functions \
  --max-items 100

# List functions filtered by runtime
aws lambda list-functions \
  --query 'Functions[?Runtime==`python3.11`].FunctionName' \
  --output text

# List all layer versions for a specific layer
aws lambda list-layer-versions \
  --layer-name my-dependencies-layer

# List tags on a function
aws lambda list-tags \
  --resource arn:aws:lambda:us-east-1:123456789012:function:my-function

# List function URL configs
aws lambda list-function-url-configs \
  --function-name my-function

# List provisioned concurrency configs
aws lambda list-provisioned-concurrency-configs \
  --function-name my-function
```

---

## Advanced Commands

### 1. Configure