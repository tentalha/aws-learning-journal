# CloudFormation — AWS CLI Commands

## Setup & Configuration

### Prerequisites

- **AWS CLI version**: 2.x recommended (`aws --version`)
- **Authentication**: Configured via `aws configure` or environment variables
- **Region**: Set via `--region` flag or `AWS_DEFAULT_REGION` environment variable

### Required IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:*",
        "s3:GetObject",
        "s3:PutObject",
        "iam:PassRole",
        "iam:GetRole"
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

# Verify identity and active region
aws sts get-caller-identity

# Set a default region for the session
export AWS_DEFAULT_REGION=us-east-1

# Verify CloudFormation access
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE
```

---

## Core Commands

### 1. Create a Stack

```bash
aws cloudformation create-stack \
  --stack-name my-app-stack \
  --template-body file://template.yaml \
  --parameters \
      ParameterKey=Environment,ParameterValue=production \
      ParameterKey=InstanceType,ParameterValue=t3.medium \
  --capabilities CAPABILITY_NAMED_IAM \
  --tags Key=Project,Value=MyApp Key=Owner,Value=DevTeam \
  --region us-east-1
```

**What it does**: Creates a new CloudFormation stack from a local template file. The `--capabilities` flag is required when the template creates IAM resources.

**Example Output**:
```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:123456789012:stack/my-app-stack/a1b2c3d4-1234-5678-abcd-ef1234567890"
}
```

---

### 2. Describe a Stack

```bash
aws cloudformation describe-stacks \
  --stack-name my-app-stack \
  --region us-east-1
```

**What it does**: Returns detailed information about the specified stack, including status, parameters, outputs, and tags.

**Example Output**:
```json
{
    "Stacks": [
        {
            "StackId": "arn:aws:cloudformation:us-east-1:123456789012:stack/my-app-stack/a1b2c3d4",
            "StackName": "my-app-stack",
            "StackStatus": "CREATE_COMPLETE",
            "Parameters": [
                {
                    "ParameterKey": "Environment",
                    "ParameterValue": "production"
                }
            ],
            "Outputs": [
                {
                    "OutputKey": "WebsiteURL",
                    "OutputValue": "https://my-app.example.com",
                    "Description": "Application URL"
                }
            ]
        }
    ]
}
```

---

### 3. Update a Stack

```bash
aws cloudformation update-stack \
  --stack-name my-app-stack \
  --template-body file://template-updated.yaml \
  --parameters \
      ParameterKey=Environment,ParameterValue=production \
      ParameterKey=InstanceType,ParameterValue=t3.large \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

**What it does**: Updates an existing stack with a new template or changed parameters. CloudFormation calculates a diff and applies only the necessary changes.

---

### 4. Delete a Stack

```bash
aws cloudformation delete-stack \
  --stack-name my-app-stack \
  --region us-east-1
```

**What it does**: Initiates deletion of the specified stack and all its resources. This is asynchronous — use `wait` or `describe-stacks` to monitor progress.

---

### 5. List All Stacks

```bash
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE ROLLBACK_COMPLETE \
  --region us-east-1
```

**What it does**: Lists stacks filtered by one or more status values. Useful for getting an overview of all active or failed stacks.

**Example Output**:
```json
{
    "StackSummaries": [
        {
            "StackId": "arn:aws:cloudformation:us-east-1:123456789012:stack/my-app-stack/a1b2c3d4",
            "StackName": "my-app-stack",
            "StackStatus": "CREATE_COMPLETE",
            "CreationTime": "2024-01-15T10:30:00.000Z"
        }
    ]
}
```

---

### 6. Validate a Template

```bash
aws cloudformation validate-template \
  --template-body file://template.yaml \
  --region us-east-1
```

**What it does**: Validates the syntax of a CloudFormation template without creating any resources. Essential step before deploying.

**Example Output**:
```json
{
    "Parameters": [
        {
            "ParameterKey": "Environment",
            "NoEcho": false,
            "Description": "Deployment environment"
        }
    ],
    "Description": "My Application Stack",
    "Capabilities": ["CAPABILITY_NAMED_IAM"],
    "CapabilitiesReason": "The following resource(s) require capabilities: [AWS::IAM::Role]"
}
```

---

### 7. Describe Stack Events

```bash
aws cloudformation describe-stack-events \
  --stack-name my-app-stack \
  --region us-east-1
```

**What it does**: Returns all events for a stack in reverse chronological order. Critical for diagnosing failed deployments.

---

### 8. Describe Stack Resources

```bash
aws cloudformation describe-stack-resources \
  --stack-name my-app-stack \
  --region us-east-1
```

**What it does**: Lists all resources in a stack with their physical IDs, logical IDs, resource types, and current status.

**Example Output**:
```json
{
    "StackResources": [
        {
            "StackName": "my-app-stack",
            "LogicalResourceId": "MyEC2Instance",
            "PhysicalResourceId": "i-0abc123def456789",
            "ResourceType": "AWS::EC2::Instance",
            "ResourceStatus": "CREATE_COMPLETE"
        }
    ]
}
```

---

### 9. Get a Stack Template

```bash
aws cloudformation get-template \
  --stack-name my-app-stack \
  --template-stage Original \
  --region us-east-1
```

**What it does**: Retrieves the template body for a deployed stack. Use `--template-stage Processed` to get the fully resolved template after transforms.

---

### 10. Create a Change Set

```bash
aws cloudformation create-change-set \
  --stack-name my-app-stack \
  --change-set-name my-changeset-v2 \
  --template-body file://template-updated.yaml \
  --parameters \
      ParameterKey=InstanceType,ParameterValue=t3.large \
  --capabilities CAPABILITY_NAMED_IAM \
  --description "Upgrade instance type to t3.large" \
  --region us-east-1
```

**What it does**: Creates a change set to preview the impact of a stack update before applying it. No resources are modified at this stage.

---

### 11. Describe a Change Set

```bash
aws cloudformation describe-change-set \
  --stack-name my-app-stack \
  --change-set-name my-changeset-v2 \
  --region us-east-1
```

**What it does**: Shows all proposed changes in a change set, including which resources will be added, modified, or removed.

**Example Output**:
```json
{
    "Changes": [
        {
            "Type": "Resource",
            "ResourceChange": {
                "Action": "Modify",
                "LogicalResourceId": "MyEC2Instance",
                "ResourceType": "AWS::EC2::Instance",
                "Replacement": "False",
                "Details": [
                    {
                        "Target": {
                            "Attribute": "Properties",
                            "Name": "InstanceType"
                        },
                        "ChangeSource": "DirectModification"
                    }
                ]
            }
        }
    ],
    "Status": "CREATE_COMPLETE"
}
```

---

### 12. Execute a Change Set

```bash
aws cloudformation execute-change-set \
  --stack-name my-app-stack \
  --change-set-name my-changeset-v2 \
  --region us-east-1
```

**What it does**: Applies the changes described in a change set to the live stack. This triggers the actual resource modifications.

---

### 13. Wait for Stack Completion

```bash
aws cloudformation wait stack-create-complete \
  --stack-name my-app-stack \
  --region us-east-1
```

**What it does**: Polls the stack status every 30 seconds and returns only when the stack reaches `CREATE_COMPLETE` or fails. Useful in CI/CD pipelines.

---

### 14. Deploy (Create or Update)

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-app-stack \
  --parameter-overrides \
      Environment=production \
      InstanceType=t3.medium \
  --capabilities CAPABILITY_NAMED_IAM \
  --no-fail-on-empty-changeset \
  --tags Project=MyApp Owner=DevTeam \
  --region us-east-1
```

**What it does**: High-level command that creates the stack if it doesn't exist, or updates it if it does. Automatically uses change sets internally. Ideal for CI/CD pipelines.

---

### 15. List Stack Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name my-app-stack \
  --query "Stacks[0].Outputs" \
  --output table \
  --region us-east-1
```

**What it does**: Retrieves and displays the output values exported by a stack in a readable table format.

**Example Output**:
```
---------------------------------------------------------
|                    DescribeStacks                     |
+---------------+---------------------+-----------------+
|  Description  |     OutputKey       |   OutputValue   |
+---------------+---------------------+-----------------+
|  App URL      |  WebsiteURL         |  https://...    |
|  DB Endpoint  |  DatabaseEndpoint   |  db.example.com |
+---------------+---------------------+-----------------+
```

---

## Common Operations

### Create Operations

```bash
# Create stack from a local template
aws cloudformation create-stack \
  --stack-name my-app-stack \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_IAM

# Create stack from a template stored in S3
aws cloudformation create-stack \
  --stack-name my-app-stack \
  --template-url https://s3.amazonaws.com/my-cfn-bucket/template.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# Create stack with a role ARN (CloudFormation assumes this role)
aws cloudformation create-stack \
  --stack-name my-app-stack \
  --template-body file://template.yaml \
  --role-arn arn:aws:iam::123456789012:role/CloudFormationDeployRole \
  --capabilities CAPABILITY_NAMED_IAM

# Create stack with a stack policy to protect resources
aws cloudformation create-stack \
  --stack-name my-app-stack \
  --template-body file://template.yaml \
  --stack-policy-body file://stack-policy.json

# Create stack with termination protection enabled
aws cloudformation create-stack \
  --stack-name my-production-stack \
  --template-body file://template.yaml \
  --enable-termination-protection \
  --capabilities CAPABILITY_NAMED_IAM
```

---

### Read / Describe Operations

```bash
# Describe a specific stack
aws cloudformation describe-stacks \
  --stack-name my-app-stack

# Describe a specific resource within a stack
aws cloudformation describe-stack-resource \
  --stack-name my-app-stack \
  --logical-resource-id MyEC2Instance

# List all resources in a stack
aws cloudformation list-stack-resources \
  --stack-name my-app-stack

# Get the template of a deployed stack
aws cloudformation get-template \
  --stack-name my-app-stack

# List all change sets for a stack
aws cloudformation list-change-sets \
  --stack-name my-app-stack

# Get a specific stack policy
aws cloudformation get-stack-policy \
  --stack-name my-app-stack
```

---

### Update Operations

```bash
# Update stack with a new template
aws cloudformation update-stack \
  --stack-name my-app-stack \
  --template-body file://template-v2.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# Update stack using existing template, only change parameters
aws cloudformation update-stack \
  --stack-name my-app-stack \
  --use-previous-template \
  --parameters \
      ParameterKey=InstanceType,ParameterValue=t3.large \
      ParameterKey=Environment,UsePreviousValue=true

# Update termination protection setting
aws cloudformation update-termination-protection \
  --stack-name my-app-stack \
  --enable-termination-protection

# Update stack policy
aws cloudformation set-stack-policy \
  --stack-name my-app-stack \
  --stack-policy-body file://updated-stack-policy.json

# Cancel an in-progress update
aws cloudformation cancel-update-stack \
  --stack-name my-app-stack
```

---

### Delete Operations

```bash
# Delete a stack
aws cloudformation delete-stack \
  --stack-name my-app-stack

# Delete a stack and retain specific resources
aws cloudformation delete-stack \
  --stack-name my-app-stack \
  --retain-resources MyS3Bucket MyRDSInstance

# Delete a change set without applying it
aws cloudformation delete-change-set \
  --stack-name my-app-stack \
  --change-set-name my-changeset-v2

# Wait for stack deletion to complete
aws cloudformation wait stack-delete-complete \
  --stack-name my-app-stack
```

---

### List Operations

```bash
# List stacks by status
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE

# List ALL stacks including deleted ones
aws cloudformation list-stacks \
  --stack-status-filter \
    CREATE_COMPLETE UPDATE_COMPLETE DELETE_COMPLETE \
    ROLLBACK_COMPLETE CREATE_FAILED DELETE_FAILED

# List stack exports (cross-stack references)
aws cloudformation list-exports \
  --region us-east-1

# List stacks importing a specific export
aws cloudformation list-imports \
  --export-name MyVPCId

# List all stack sets
aws cloudformation list-stack-sets \
  --status ACTIVE
```

---

## Advanced Commands

### 1. Package a Template (Upload Local Artifacts to S3)

```bash
aws cloudformation package \
  --template-file template.yaml \
  --s3-bucket my-cfn-artifacts-bucket \
  --s3-prefix cloudformation/my-app \
  --output-template-file packaged-template.yaml \
  --region us-east-1
```

**What it does**: Scans a template for local file references (Lambda code, nested templates, etc.), uploads them to S3, and outputs a new template with S3 URIs substituted in. This is a prerequisite for deploying templates with local artifacts.

---

### 2. Detect Stack Drift

```bash
# Initiate drift detection
aws cloudformation detect-stack-drift \
  --stack-name my-app-stack \
  --region us-east-1

# Check drift detection status (use the DetectionId from above)
aws