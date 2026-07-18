# API Gateway — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using the AWS CLI with API Gateway, ensure the following are in place:

**Install & Configure AWS CLI**
```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Configure credentials and default region
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

**Verify CLI Version**
```bash
aws --version
# aws-cli/2.13.0 Python/3.11.4 Linux/5.15.0 exe/x86_64.ubuntu.22
```

### Required IAM Permissions

Attach the following IAM policy to your user or role for full API Gateway access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "apigateway:GET",
        "apigateway:POST",
        "apigateway:PUT",
        "apigateway:PATCH",
        "apigateway:DELETE",
        "lambda:AddPermission",
        "lambda:RemovePermission",
        "iam:PassRole",
        "logs:CreateLogGroup",
        "logs:CreateLogDelivery",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

> **Note:** API Gateway has two major versions:
> - **REST APIs** — use `aws apigateway` (v1)
> - **HTTP APIs & WebSocket APIs** — use `aws apigatewayv2` (v2)
>
> This reference covers both. Commands are clearly labeled.

---

## Core Commands

### 1. Create a REST API

```bash
aws apigateway create-rest-api \
  --name "my-rest-api" \
  --description "My production REST API" \
  --endpoint-configuration '{"types": ["REGIONAL"]}' \
  --region us-east-1
```

**What it does:** Creates a new REST API in API Gateway. The `REGIONAL` endpoint type routes traffic through the regional endpoint (alternatives: `EDGE`, `PRIVATE`).

**Example Output:**
```json
{
    "id": "abc123def4",
    "name": "my-rest-api",
    "description": "My production REST API",
    "createdDate": "2024-01-15T10:30:00+00:00",
    "endpointConfiguration": {
        "types": ["REGIONAL"]
    },
    "apiKeySource": "HEADER",
    "disableExecuteApiEndpoint": false
}
```

---

### 2. List All REST APIs

```bash
aws apigateway get-rest-apis \
  --region us-east-1
```

**What it does:** Retrieves all REST APIs in the specified region.

**Example Output:**
```json
{
    "items": [
        {
            "id": "abc123def4",
            "name": "my-rest-api",
            "createdDate": "2024-01-15T10:30:00+00:00",
            "endpointConfiguration": {
                "types": ["REGIONAL"]
            }
        },
        {
            "id": "xyz789ghi0",
            "name": "my-legacy-api",
            "createdDate": "2023-06-01T08:00:00+00:00",
            "endpointConfiguration": {
                "types": ["EDGE"]
            }
        }
    ]
}
```

---

### 3. Get Resources (Endpoints) of a REST API

```bash
aws apigateway get-resources \
  --rest-api-id abc123def4 \
  --region us-east-1
```

**What it does:** Lists all resources (path segments) defined in a REST API, showing the resource tree.

**Example Output:**
```json
{
    "items": [
        {
            "id": "rootResourceId",
            "path": "/",
            "resourceMethods": {}
        },
        {
            "id": "childResource1",
            "parentId": "rootResourceId",
            "pathPart": "users",
            "path": "/users",
            "resourceMethods": {
                "GET": {},
                "POST": {}
            }
        }
    ]
}
```

---

### 4. Create a Resource (Path)

```bash
aws apigateway create-resource \
  --rest-api-id abc123def4 \
  --parent-id rootResourceId \
  --path-part "users" \
  --region us-east-1
```

**What it does:** Creates a new resource (path segment) under a parent resource. Use `{proxy+}` as `--path-part` for a greedy proxy resource.

**Example Output:**
```json
{
    "id": "newResource1",
    "parentId": "rootResourceId",
    "pathPart": "users",
    "path": "/users"
}
```

---

### 5. Put (Create) a Method on a Resource

```bash
aws apigateway put-method \
  --rest-api-id abc123def4 \
  --resource-id newResource1 \
  --http-method GET \
  --authorization-type NONE \
  --region us-east-1
```

**What it does:** Adds an HTTP method (GET, POST, PUT, DELETE, etc.) to a resource. Set `--authorization-type` to `AWS_IAM`, `COGNITO_USER_POOLS`, or `CUSTOM` for protected endpoints.

---

### 6. Put an Integration (Connect to Lambda)

```bash
aws apigateway put-integration \
  --rest-api-id abc123def4 \
  --resource-id newResource1 \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:123456789012:function:my-lambda-function/invocations" \
  --region us-east-1
```

**What it does:** Connects a method to a backend integration. `AWS_PROXY` passes the full request to Lambda. Note: the integration HTTP method for Lambda is always `POST` regardless of the API method.

---

### 7. Deploy an API to a Stage

```bash
aws apigateway create-deployment \
  --rest-api-id abc123def4 \
  --stage-name prod \
  --stage-description "Production deployment" \
  --description "Initial production release v1.0" \
  --region us-east-1
```

**What it does:** Deploys the current API configuration to a named stage, making it publicly accessible. This is required after any changes to the API.

**Example Output:**
```json
{
    "id": "deploy123",
    "description": "Initial production release v1.0",
    "createdDate": "2024-01-15T11:00:00+00:00"
}
```

---

### 8. Get Stage Details

```bash
aws apigateway get-stage \
  --rest-api-id abc123def4 \
  --stage-name prod \
  --region us-east-1
```

**What it does:** Retrieves configuration details for a specific stage, including throttling settings, logging, and caching configuration.

**Example Output:**
```json
{
    "deploymentId": "deploy123",
    "stageName": "prod",
    "description": "Production deployment",
    "cacheClusterEnabled": false,
    "defaultRouteSettings": {},
    "methodSettings": {},
    "variables": {},
    "createdDate": "2024-01-15T11:00:00+00:00",
    "lastUpdatedDate": "2024-01-15T11:00:00+00:00"
}
```

---

### 9. Create an HTTP API (v2)

```bash
aws apigatewayv2 create-api \
  --name "my-http-api" \
  --protocol-type HTTP \
  --description "My HTTP API with auto-deploy" \
  --cors-configuration AllowOrigins="https://my-frontend.example.com",AllowMethods="GET,POST,PUT,DELETE",AllowHeaders="Content-Type,Authorization" \
  --region us-east-1
```

**What it does:** Creates a new HTTP API (v2), which is faster and cheaper than REST APIs. Supports CORS configuration natively.

**Example Output:**
```json
{
    "ApiEndpoint": "https://abcdef1234.execute-api.us-east-1.amazonaws.com",
    "ApiId": "abcdef1234",
    "ApiKeySelectionExpression": "$request.header.x-api-key",
    "CreatedDate": "2024-01-15T10:30:00+00:00",
    "Name": "my-http-api",
    "ProtocolType": "HTTP"
}
```

---

### 10. Create a Route for an HTTP API (v2)

```bash
aws apigatewayv2 create-route \
  --api-id abcdef1234 \
  --route-key "GET /users" \
  --target "integrations/integration123" \
  --region us-east-1
```

**What it does:** Creates a route that maps an HTTP method + path combination to an integration target.

---

### 11. Create a Lambda Integration for HTTP API (v2)

```bash
aws apigatewayv2 create-integration \
  --api-id abcdef1234 \
  --integration-type AWS_PROXY \
  --integration-uri "arn:aws:lambda:us-east-1:123456789012:function:my-lambda-function" \
  --payload-format-version "2.0" \
  --region us-east-1
```

**What it does:** Creates a Lambda proxy integration for an HTTP API. Payload format version `2.0` is the modern, simplified format recommended for HTTP APIs.

---

### 12. Create an API Key

```bash
aws apigateway create-api-key \
  --name "my-partner-api-key" \
  --description "API key for partner integration" \
  --enabled \
  --region us-east-1
```

**What it does:** Creates an API key that can be used to control and meter access to your API.

**Example Output:**
```json
{
    "id": "apikey123abc",
    "name": "my-partner-api-key",
    "description": "API key for partner integration",
    "enabled": true,
    "createdDate": "2024-01-15T10:00:00+00:00",
    "value": "Xf3kL9mN2pQ7rS1tU4vW6yZ8aB0cD5eF"
}
```

---

### 13. Create a Usage Plan

```bash
aws apigateway create-usage-plan \
  --name "standard-plan" \
  --description "Standard tier - 1000 req/day, 100 req/sec burst" \
  --api-stages "apiId=abc123def4,stage=prod" \
  --throttle burstLimit=100,rateLimit=50 \
  --quota limit=1000,offset=0,period=DAY \
  --region us-east-1
```

**What it does:** Creates a usage plan that defines throttling and quota limits for API keys associated with specific API stages.

---

### 14. Delete a REST API

```bash
aws apigateway delete-rest-api \
  --rest-api-id abc123def4 \
  --region us-east-1
```

**What it does:** Permanently deletes a REST API and all its resources, methods, integrations, stages, and deployments. **This action is irreversible.**

---

### 15. Get the Invoke URL for a Stage

```bash
# Construct the invoke URL manually from API ID and stage
API_ID="abc123def4"
STAGE="prod"
REGION="us-east-1"
echo "https://${API_ID}.execute-api.${REGION}.amazonaws.com/${STAGE}"
```

**What it does:** Constructs the base invoke URL for your deployed API. Append your resource paths to form complete endpoint URLs.

---

## Common Operations

### Create Operations

```bash
# Create a REST API
aws apigateway create-rest-api \
  --name "my-rest-api" \
  --endpoint-configuration '{"types": ["REGIONAL"]}' \
  --region us-east-1

# Create a resource under the root
aws apigateway create-resource \
  --rest-api-id abc123def4 \
  --parent-id rootResourceId \
  --path-part "orders" \
  --region us-east-1

# Create a child resource with path parameter
aws apigateway create-resource \
  --rest-api-id abc123def4 \
  --parent-id ordersResourceId \
  --path-part "{orderId}" \
  --region us-east-1

# Create a method with API key required
aws apigateway put-method \
  --rest-api-id abc123def4 \
  --resource-id ordersResourceId \
  --http-method POST \
  --authorization-type NONE \
  --api-key-required \
  --region us-east-1

# Create a method response
aws apigateway put-method-response \
  --rest-api-id abc123def4 \
  --resource-id ordersResourceId \
  --http-method GET \
  --status-code 200 \
  --response-models '{"application/json": "Empty"}' \
  --region us-east-1

# Create an authorizer (Lambda-based)
aws apigateway create-authorizer \
  --rest-api-id abc123def4 \
  --name "my-lambda-authorizer" \
  --type TOKEN \
  --authorizer-uri "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:123456789012:function:my-authorizer-function/invocations" \
  --identity-source "method.request.header.Authorization" \
  --authorizer-result-ttl-in-seconds 300 \
  --region us-east-1

# Create a model (request/response schema)
aws apigateway create-model \
  --rest-api-id abc123def4 \
  --name "OrderModel" \
  --content-type "application/json" \
  --schema '{"$schema":"http://json-schema.org/draft-04/schema#","title":"OrderModel","type":"object","properties":{"orderId":{"type":"string"},"amount":{"type":"number"}}}' \
  --region us-east-1

# Create a VPC Link (for private integrations)
aws apigateway create-vpc-link \
  --name "my-vpc-link" \
  --target-arns "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/net/my-nlb/abc123" \
  --region us-east-1

# Create a stage with settings
aws apigateway create-stage \
  --rest-api-id abc123def4 \
  --stage-name staging \
  --deployment-id deploy123 \
  --description "Staging environment" \
  --variables '{"lambdaAlias":"staging","tableName":"orders-staging"}' \
  --region us-east-1
```

---

### Read / Describe Operations

```bash
# Get a specific REST API
aws apigateway get-rest-api \
  --rest-api-id abc123def4 \
  --region us-east-1

# Get a specific resource
aws apigateway get-resource \
  --rest-api-id abc123def4 \
  --resource-id newResource1 \
  --region us-east-1

# Get method details