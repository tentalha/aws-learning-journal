# API Gateway — Hands-On Labs

## Lab 1: Getting Started with API Gateway

### Objective
In this lab, you will build your first REST API using Amazon API Gateway. You will create a simple HTTP API that integrates with AWS Lambda to return a "Hello, World!" response. By the end of this lab, you will understand how to create API resources, configure HTTP methods, integrate with Lambda, deploy to a stage, and invoke your API via a public URL.

### Prerequisites

**AWS Services Required:**
- Amazon API Gateway (REST API)
- AWS Lambda
- AWS IAM

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "apigateway:*",
        "lambda:*",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole",
        "logs:CreateLogGroup",
        "logs:CreateLogDelivery"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS Management Console access
- AWS CLI v2 installed and configured (`aws configure`)
- `curl` or Postman for testing
- A text editor

**Estimated Cost:** < $0.01 (within Free Tier limits)  
**Estimated Duration:** 45–60 minutes

---

### Steps

#### Step 1: Create the Lambda Backend Function

**Console Approach:**
1. Navigate to the [AWS Lambda Console](https://console.aws.amazon.com/lambda).
2. Click **Create function**.
3. Select **Author from scratch**.
4. Configure the following:
   - **Function name:** `hello-world-function`
   - **Runtime:** Python 3.12
   - **Architecture:** x86_64
5. Under **Permissions**, select **Create a new role with basic Lambda permissions**.
6. Click **Create function**.
7. In the **Code source** editor, replace the default code with:

```python
import json

def lambda_handler(event, context):
    print(f"Received event: {json.dumps(event)}")
    
    name = "World"
    
    # Extract query string parameter if provided
    if event.get("queryStringParameters") and event["queryStringParameters"].get("name"):
        name = event["queryStringParameters"]["name"]
    
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": json.dumps({
            "message": f"Hello, {name}!",
            "source": "AWS Lambda via API Gateway"
        })
    }
```

8. Click **Deploy** to save the function.

**CLI Approach:**
```bash
# Set environment variables for reuse
export AWS_REGION="us-east-1"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create the Lambda execution role
aws iam create-role \
  --role-name hello-lambda-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach the basic Lambda execution policy
aws iam attach-role-policy \
  --role-name hello-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Wait for role propagation
sleep 10

# Create the deployment package
cat > lambda_function.py << 'EOF'
import json

def lambda_handler(event, context):
    print(f"Received event: {json.dumps(event)}")
    name = "World"
    if event.get("queryStringParameters") and event["queryStringParameters"].get("name"):
        name = event["queryStringParameters"]["name"]
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": json.dumps({
            "message": f"Hello, {name}!",
            "source": "AWS Lambda via API Gateway"
        })
    }
EOF

zip function.zip lambda_function.py

# Deploy the Lambda function
aws lambda create-function \
  --function-name hello-world-function \
  --runtime python3.12 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/hello-lambda-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --region ${AWS_REGION}
```

**✅ Verify Step 1:**
```bash
aws lambda invoke \
  --function-name hello-world-function \
  --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  response.json

cat response.json
```
**Expected Output:**
```json
{"statusCode": 200, "headers": {"Content-Type": "application/json", "Access-Control-Allow-Origin": "*"}, "body": "{\"message\": \"Hello, World!\", \"source\": \"AWS Lambda via API Gateway\"}"}
```

---

#### Step 2: Create the REST API in API Gateway

**Console Approach:**
1. Navigate to the [API Gateway Console](https://console.aws.amazon.com/apigateway).
2. Click **Create API**.
3. Under **REST API**, click **Build** (not "REST API Private").
4. Configure:
   - **Protocol:** REST
   - **Create new API:** New API
   - **API name:** `HelloWorldAPI`
   - **Description:** `My first API Gateway lab`
   - **Endpoint Type:** Regional
5. Click **Create API**.

**CLI Approach:**
```bash
# Create the REST API
API_ID=$(aws apigateway create-rest-api \
  --name "HelloWorldAPI" \
  --description "My first API Gateway lab" \
  --endpoint-configuration types=REGIONAL \
  --region ${AWS_REGION} \
  --query 'id' \
  --output text)

echo "API ID: ${API_ID}"

# Get the root resource ID
ROOT_RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id ${API_ID} \
  --region ${AWS_REGION} \
  --query 'items[0].id' \
  --output text)

echo "Root Resource ID: ${ROOT_RESOURCE_ID}"
```

**✅ Verify Step 2:**
```bash
aws apigateway get-rest-api \
  --rest-api-id ${API_ID} \
  --region ${AWS_REGION}
```
**Expected Output:** A JSON object showing the API with name `HelloWorldAPI` and status details.

---

#### Step 3: Create a Resource and Method

**Console Approach:**
1. In the API Gateway console, select your `HelloWorldAPI`.
2. In the left panel, click **Resources**.
3. With the root `/` resource selected, click **Actions** → **Create Resource**.
4. Configure:
   - **Resource Name:** `hello`
   - **Resource Path:** `hello`
   - **Enable API Gateway CORS:** ✅ (check this box)
5. Click **Create Resource**.
6. With `/hello` selected, click **Actions** → **Create Method**.
7. Select **GET** from the dropdown and click the ✓ checkmark.
8. Configure the GET method integration:
   - **Integration type:** Lambda Function
   - **Use Lambda Proxy integration:** ✅ (check this box)
   - **Lambda Region:** us-east-1
   - **Lambda Function:** `hello-world-function`
9. Click **Save**, then **OK** to grant API Gateway permission to invoke Lambda.

**CLI Approach:**
```bash
# Create the /hello resource
RESOURCE_ID=$(aws apigateway create-resource \
  --rest-api-id ${API_ID} \
  --parent-id ${ROOT_RESOURCE_ID} \
  --path-part "hello" \
  --region ${AWS_REGION} \
  --query 'id' \
  --output text)

echo "Resource ID: ${RESOURCE_ID}"

# Create the GET method (no authorization for now)
aws apigateway put-method \
  --rest-api-id ${API_ID} \
  --resource-id ${RESOURCE_ID} \
  --http-method GET \
  --authorization-type NONE \
  --region ${AWS_REGION}

# Set up Lambda proxy integration
LAMBDA_ARN="arn:aws:lambda:${AWS_REGION}:${ACCOUNT_ID}:function:hello-world-function"

aws apigateway put-integration \
  --rest-api-id ${API_ID} \
  --resource-id ${RESOURCE_ID} \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri "arn:aws:apigateway:${AWS_REGION}:lambda:path/2015-03-31/functions/${LAMBDA_ARN}/invocations" \
  --region ${AWS_REGION}

# Grant API Gateway permission to invoke Lambda
aws lambda add-permission \
  --function-name hello-world-function \
  --statement-id apigateway-invoke-permission \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:${AWS_REGION}:${ACCOUNT_ID}:${API_ID}/*/GET/hello" \
  --region ${AWS_REGION}
```

**✅ Verify Step 3:**
```bash
aws apigateway get-method \
  --rest-api-id ${API_ID} \
  --resource-id ${RESOURCE_ID} \
  --http-method GET \
  --region ${AWS_REGION}
```
**Expected Output:** JSON showing `httpMethod: GET`, `authorizationType: NONE`, and the Lambda integration details.

---

#### Step 4: Deploy the API to a Stage

**Console Approach:**
1. In the Resources view, click **Actions** → **Deploy API**.
2. Configure:
   - **Deployment stage:** [New Stage]
   - **Stage name:** `dev`
   - **Stage description:** `Development stage`
   - **Deployment description:** `Initial deployment`
3. Click **Deploy**.
4. Note the **Invoke URL** displayed at the top of the stage editor (e.g., `https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev`).

**CLI Approach:**
```bash
# Deploy the API
DEPLOYMENT_ID=$(aws apigateway create-deployment \
  --rest-api-id ${API_ID} \
  --stage-name dev \
  --stage-description "Development stage" \
  --description "Initial deployment" \
  --region ${AWS_REGION} \
  --query 'id' \
  --output text)

echo "Deployment ID: ${DEPLOYMENT_ID}"

# Construct the invoke URL
INVOKE_URL="https://${API_ID}.execute-api.${AWS_REGION}.amazonaws.com/dev"
echo "API Invoke URL: ${INVOKE_URL}"
```

**✅ Verify Step 4:**
```bash
aws apigateway get-stage \
  --rest-api-id ${API_ID} \
  --stage-name dev \
  --region ${AWS_REGION}
```
**Expected Output:** JSON showing stage details including `stageName: dev` and `deploymentId`.

---

#### Step 5: Test the API

**Console Approach:**
1. In the API Gateway console, navigate to **Resources** → **GET** method under `/hello`.
2. Click **TEST** (the lightning bolt icon).
3. Leave all fields empty and click **Test**.
4. Verify the response body shows `{"message": "Hello, World!", ...}`.
5. Try adding `name=Alice` in the **Query Strings** field and test again.

**CLI / curl Approach:**
```bash
# Basic invocation
curl -X GET "${INVOKE_URL}/hello"

# With query string parameter
curl -X GET "${INVOKE_URL}/hello?name=Alice"

# With verbose output to inspect headers
curl -v -X GET "${INVOKE_URL}/hello?name=Bob"
```

**Expected Output:**
```json
{
  "message": "Hello, Alice!",
  "source": "AWS Lambda via API Gateway"
}
```

---

### Verification

Run the following checklist to confirm successful lab completion:

```bash
# 1. Verify Lambda function exists
aws lambda get-function --function-name hello-world-function --region ${AWS_REGION}

# 2. Verify API exists
aws apigateway get-rest-api --rest-api-id ${API_ID} --region ${AWS_REGION}

# 3. Verify the stage is deployed
aws apigateway get-stage --rest-api-id ${API_ID} --stage-name dev --region ${AWS_REGION}

# 4. Verify end-to-end API call returns HTTP 200
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${INVOKE_URL}/hello")
echo "HTTP Status: ${HTTP_STATUS}"   # Should be 200

# 5. Verify query string parameter works
curl -s "${INVOKE_URL}/hello?name=TestUser" | python3 -m json.tool
```

**Success Criteria:**
- ✅ Lambda function returns a 200 response when invoked directly
- ✅ API Gateway endpoint returns HTTP 200
- ✅ Response body contains `"message": "Hello, World!"`
- ✅ Query string `?name=Alice` changes the greeting dynamically

---

### Cleanup

Run these commands to remove all resources and avoid charges:

```bash
# 1. Delete the API Gateway REST API (removes all stages and deployments)
aws apigateway delete-rest-api \
  --rest-api-id ${API_ID} \
  --region ${AWS_REGION}

echo "✅ API Gateway deleted"

# 2. Delete the Lambda function
aws lambda delete-function \
  --function-name hello-world-function \
  --region ${AWS_REGION}

echo "✅ Lambda function deleted"

# 3. Detach policy and delete IAM role
aws iam detach-role-policy \
  --role-name hello-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role --role-name hello-lambda-role

echo "✅ IAM role deleted"

# 4. Clean up local files
rm -f function.zip lambda_function.py response.json

echo "✅ Cleanup complete!"
```

**Console Cleanup:**
1. API Gateway → APIs → Select `HelloWorldAPI` → **Actions** → **Delete**
2. Lambda → Functions → Select `hello-world-function` → **Actions** → **Delete**
3. IAM → Roles → Search for `hello-lambda-role` → **Delete**

---

## Lab 2: Intermediate API Gateway Configuration

### Objective
In this lab, you will build a CRUD (Create, Read, Update, Delete) REST API backed by Amazon DynamoDB using Lambda functions. You will configure API keys and usage plans for rate limiting, enable request validation, implement Lambda authorizers for token-based authentication, and set up CloudWatch logging and X-Ray tracing for observability. By the end, you will have a functional, monitored, and secured items management API.

### Prerequisites

**AWS Services Required:**
- Amazon API Gateway (REST API)
- AWS Lambda (Python 3.12)
- Amazon DynamoDB
- AWS IAM
- Amazon CloudWatch
- AWS X-Ray

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "apigateway:*",
        "lambda:*",
        "dynamodb:*",
        "iam:*",
        "logs:*",
        "xray:*",
        "cloudwatch:*"
      ],
      "Resource": "*"
    }
  ]
}
```

**Prerequisites from Lab 1:**
- Familiarity with creating Lambda functions and REST APIs