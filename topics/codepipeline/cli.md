# CodePipeline — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Ensure the AWS CLI is installed and configured before using CodePipeline commands.

```bash
# Install or update AWS CLI
pip install --upgrade awscli

# Configure default profile
aws configure

# Verify identity
aws sts get-caller-identity
```

### Required IAM Permissions

Attach the following managed policies or equivalent inline permissions to your IAM user/role:

- `AWSCodePipelineFullAccess` — Full pipeline management
- `AWSCodePipelineReadOnlyAccess` — Read-only access
- `AWSCodeBuildAdminAccess` — If using CodeBuild actions
- `AmazonS3FullAccess` — For artifact bucket access

```bash
# Attach managed policy to an IAM role
aws iam attach-role-policy \
  --role-name my-devops-role \
  --policy-arn arn:aws:iam::aws:policy/AWSCodePipelineFullAccess

# Verify attached policies
aws iam list-attached-role-policies \
  --role-name my-devops-role
```

### Minimum IAM Policy (Least Privilege)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codepipeline:*",
        "iam:PassRole",
        "s3:GetObject",
        "s3:PutObject",
        "s3:GetBucketVersioning",
        "codestar-connections:UseConnection"
      ],
      "Resource": "*"
    }
  ]
}
```

### Set Default Region and Output Format

```bash
# Set environment variables for convenience
export AWS_DEFAULT_REGION=us-east-1
export AWS_DEFAULT_OUTPUT=json

# Or use a named profile
aws configure --profile my-pipeline-profile
export AWS_PROFILE=my-pipeline-profile
```

---

## Core Commands

### 1. `create-pipeline` — Create a New Pipeline

```bash
aws codepipeline create-pipeline \
  --cli-input-json file://pipeline-definition.json
```

**Example `pipeline-definition.json`:**

```json
{
  "pipeline": {
    "name": "my-app-pipeline",
    "roleArn": "arn:aws:iam::123456789012:role/AWSCodePipelineServiceRole",
    "artifactStore": {
      "type": "S3",
      "location": "my-pipeline-artifacts-bucket"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "SourceAction",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "CodeStarSourceConnection",
              "version": "1"
            },
            "configuration": {
              "ConnectionArn": "arn:aws:codestar-connections:us-east-1:123456789012:connection/abc12345",
              "FullRepositoryId": "my-org/my-app-repo",
              "BranchName": "main"
            },
            "outputArtifacts": [{ "name": "SourceOutput" }]
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "BuildAction",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "my-codebuild-project"
            },
            "inputArtifacts": [{ "name": "SourceOutput" }],
            "outputArtifacts": [{ "name": "BuildOutput" }]
          }
        ]
      }
    ],
    "version": 1
  }
}
```

**Example Output:**

```json
{
  "pipeline": {
    "name": "my-app-pipeline",
    "roleArn": "arn:aws:iam::123456789012:role/AWSCodePipelineServiceRole",
    "artifactStore": {
      "type": "S3",
      "location": "my-pipeline-artifacts-bucket"
    },
    "version": 1,
    "created": "2024-01-15T10:30:00.000Z",
    "updated": "2024-01-15T10:30:00.000Z"
  }
}
```

---

### 2. `get-pipeline` — Retrieve Pipeline Configuration

```bash
aws codepipeline get-pipeline \
  --name my-app-pipeline
```

**Example Output:**

```json
{
  "pipeline": {
    "name": "my-app-pipeline",
    "roleArn": "arn:aws:iam::123456789012:role/AWSCodePipelineServiceRole",
    "artifactStore": {
      "type": "S3",
      "location": "my-pipeline-artifacts-bucket"
    },
    "stages": [...],
    "version": 3
  },
  "metadata": {
    "pipelineArn": "arn:aws:codepipeline:us-east-1:123456789012:my-app-pipeline",
    "created": "2024-01-15T10:30:00.000Z",
    "updated": "2024-01-20T14:22:00.000Z"
  }
}
```

---

### 3. `list-pipelines` — List All Pipelines

```bash
aws codepipeline list-pipelines
```

**Example Output:**

```json
{
  "pipelines": [
    {
      "name": "my-app-pipeline",
      "version": 3,
      "created": "2024-01-15T10:30:00.000Z",
      "updated": "2024-01-20T14:22:00.000Z"
    },
    {
      "name": "my-infra-pipeline",
      "version": 1,
      "created": "2024-01-18T09:00:00.000Z",
      "updated": "2024-01-18T09:00:00.000Z"
    }
  ]
}
```

---

### 4. `get-pipeline-state` — Get Current Pipeline Execution State

```bash
aws codepipeline get-pipeline-state \
  --name my-app-pipeline
```

**Example Output:**

```json
{
  "pipelineName": "my-app-pipeline",
  "pipelineVersion": 3,
  "stageStates": [
    {
      "stageName": "Source",
      "inboundTransitionState": {
        "enabled": true
      },
      "actionStates": [
        {
          "actionName": "SourceAction",
          "latestExecution": {
            "status": "Succeeded",
            "lastStatusChange": "2024-01-20T14:22:00.000Z",
            "externalExecutionId": "abc12345"
          }
        }
      ]
    },
    {
      "stageName": "Build",
      "actionStates": [
        {
          "actionName": "BuildAction",
          "latestExecution": {
            "status": "InProgress",
            "lastStatusChange": "2024-01-20T14:23:00.000Z"
          }
        }
      ]
    }
  ]
}
```

---

### 5. `start-pipeline-execution` — Manually Trigger a Pipeline

```bash
aws codepipeline start-pipeline-execution \
  --name my-app-pipeline \
  --client-request-token unique-token-$(date +%s)
```

**Example Output:**

```json
{
  "pipelineExecutionId": "d-EXAMPLE1"
}
```

---

### 6. `get-pipeline-execution` — Get Details of a Specific Execution

```bash
aws codepipeline get-pipeline-execution \
  --pipeline-name my-app-pipeline \
  --pipeline-execution-id d-EXAMPLE1
```

**Example Output:**

```json
{
  "pipelineExecution": {
    "pipelineName": "my-app-pipeline",
    "pipelineVersion": 3,
    "pipelineExecutionId": "d-EXAMPLE1",
    "status": "Succeeded",
    "artifactRevisions": [
      {
        "name": "SourceOutput",
        "revisionId": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
        "revisionSummary": "Updated deployment configuration",
        "revisionUrl": "https://github.com/my-org/my-app-repo/commit/a1b2c3d4"
      }
    ]
  }
}
```

---

### 7. `list-pipeline-executions` — List Execution History

```bash
aws codepipeline list-pipeline-executions \
  --pipeline-name my-app-pipeline \
  --max-results 10
```

**Example Output:**

```json
{
  "pipelineExecutionSummaries": [
    {
      "pipelineExecutionId": "d-EXAMPLE1",
      "status": "Succeeded",
      "startTime": "2024-01-20T14:22:00.000Z",
      "lastUpdateTime": "2024-01-20T14:35:00.000Z",
      "sourceRevisions": [
        {
          "actionName": "SourceAction",
          "revisionId": "a1b2c3d4e5f6",
          "revisionSummary": "Updated deployment configuration"
        }
      ]
    },
    {
      "pipelineExecutionId": "d-EXAMPLE2",
      "status": "Failed",
      "startTime": "2024-01-19T10:00:00.000Z",
      "lastUpdateTime": "2024-01-19T10:05:00.000Z"
    }
  ]
}
```

---

### 8. `stop-pipeline-execution` — Stop a Running Execution

```bash
# Stop with abandon (immediately stops)
aws codepipeline stop-pipeline-execution \
  --pipeline-name my-app-pipeline \
  --pipeline-execution-id d-EXAMPLE1 \
  --abandon \
  --reason "Emergency stop - hotfix required"

# Stop gracefully (waits for in-progress actions)
aws codepipeline stop-pipeline-execution \
  --pipeline-name my-app-pipeline \
  --pipeline-execution-id d-EXAMPLE1 \
  --reason "Stopping for maintenance"
```

---

### 9. `update-pipeline` — Update Pipeline Configuration

```bash
aws codepipeline update-pipeline \
  --cli-input-json file://updated-pipeline-definition.json
```

> **Tip:** Always retrieve the current pipeline first with `get-pipeline`, modify the JSON, then pass it to `update-pipeline`. The version number increments automatically.

---

### 10. `delete-pipeline` — Delete a Pipeline

```bash
aws codepipeline delete-pipeline \
  --name my-app-pipeline
```

> ⚠️ This action is irreversible. The pipeline is immediately deleted. Artifacts in S3 are not removed.

---

### 11. `enable-stage-transition` — Re-enable a Disabled Stage

```bash
aws codepipeline enable-stage-transition \
  --pipeline-name my-app-pipeline \
  --stage-name Deploy \
  --transition-type Inbound
```

---

### 12. `disable-stage-transition` — Block a Stage from Executing

```bash
aws codepipeline disable-stage-transition \
  --pipeline-name my-app-pipeline \
  --stage-name Deploy \
  --transition-type Inbound \
  --reason "Freezing deployments for end-of-quarter code freeze"
```

---

### 13. `put-approval-result` — Approve or Reject a Manual Approval Action

```bash
# Approve
aws codepipeline put-approval-result \
  --pipeline-name my-app-pipeline \
  --stage-name Approve \
  --action-name ManualApproval \
  --result "summary=Approved by SRE team after review,status=Approved" \
  --token "approval-token-abc123"

# Reject
aws codepipeline put-approval-result \
  --pipeline-name my-app-pipeline \
  --stage-name Approve \
  --action-name ManualApproval \
  --result "summary=Rejected - tests failed in staging,status=Rejected" \
  --token "approval-token-abc123"
```

> **Note:** The `--token` value is found in the `get-pipeline-state` output under `latestExecution.token`.

---

### 14. `retry-stage-execution` — Retry a Failed Stage

```bash
aws codepipeline retry-stage-execution \
  --pipeline-name my-app-pipeline \
  --stage-name Build \
  --pipeline-execution-id d-EXAMPLE2 \
  --retry-mode FAILED_ACTIONS
```

---

### 15. `list-action-types` — List Available Action Types

```bash
# List all action types
aws codepipeline list-action-types

# Filter by owner
aws codepipeline list-action-types \
  --action-owner-filter AWS

# Filter by category
aws codepipeline list-action-types \
  --action-owner-filter AWS \
  --query "actionTypes[?id.category=='Deploy']"
```

---

## Common Operations

### Create Operations

```bash
# Create a pipeline from JSON definition
aws codepipeline create-pipeline \
  --cli-input-json file://pipeline-definition.json

# Create a custom action type (e.g., for Jenkins)
aws codepipeline create-custom-action-type \
  --category Build \
  --provider MyJenkinsProvider \
  --version 1 \
  --settings "entityUrlTemplate=http://my-jenkins.example.com/job/{Config:ProjectName}/,executionUrlTemplate=http://my-jenkins.example.com/job/{Config:ProjectName}/{ExternalExecutionId}/" \
  --configuration-properties "[{\"name\":\"ProjectName\",\"required\":true,\"key\":true,\"secret\":false,\"queryable\":false,\"description\":\"Jenkins project name\",\"type\":\"String\"}]" \
  --input-artifact-details "minimumCount=0,maximumCount=5" \
  --output-artifact-details "minimumCount=0,maximumCount=5"

# Create a webhook for GitHub integration
aws codepipeline put-webhook \
  --cli-input-json file://webhook-definition.json
```

---

### Read / Describe Operations

```bash
# Get full pipeline definition
aws codepipeline get-pipeline \
  --name my-app-pipeline

# Get pipeline execution state
aws codepipeline get-pipeline-state \
  --name my-app-pipeline

# Get a specific execution's details
aws codepipeline get-pipeline-execution \
  --pipeline-name my-app-pipeline \
  --pipeline-execution-id d-EXAMPLE1

# Get action execution details (individual action logs)
aws codepipeline list-action-executions \
  --pipeline-name my-app-pipeline \
  --filter "pipelineExecutionId=d-EXAMPLE1"

# Get a specific webhook
aws codepipeline get-webhook \
  --webhook-name my-github-webhook

# List all webhooks
aws codepipeline list-webhooks
```

---

### Update Operations

```bash
# Update pipeline (increment version automatically handled)
aws codepipeline update-pipeline \
  --cli-input-json file://updated-pipeline.json

# Tag a pipeline
aws codepipeline tag-resource \
  --resource-arn arn:aws:codepipeline:us-east-1:123456789012:my-app-pipeline \
  --tags key=Environment,value=Production key=Team,value=Platform

# Untag a pipeline
aws codepipeline untag-resource \
  --resource-arn arn:aws:cod