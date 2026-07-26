# CodeBuild — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using AWS CLI with CodeBuild, ensure the following:

1. **AWS CLI installed and configured:**
```bash
aws --version
aws configure
```

2. **Set your default region and output format:**
```bash
aws configure set region us-east-1
aws configure set output json
```

3. **Required IAM Permissions:** Attach the following managed policies or equivalent inline policies to your IAM user/role:

| Policy | Purpose |
|---|---|
| `AWSCodeBuildAdminAccess` | Full CodeBuild access |
| `AWSCodeBuildDeveloperAccess` | Build management without project deletion |
| `AWSCodeBuildReadOnlyAccess` | Read-only access to CodeBuild resources |

4. **Minimum IAM permissions for common operations:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codebuild:CreateProject",
        "codebuild:UpdateProject",
        "codebuild:DeleteProject",
        "codebuild:StartBuild",
        "codebuild:StopBuild",
        "codebuild:BatchGetBuilds",
        "codebuild:ListBuildsForProject",
        "codebuild:ListProjects",
        "codebuild:BatchGetProjects",
        "logs:GetLogEvents",
        "s3:GetObject",
        "s3:PutObject",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

5. **Service role for CodeBuild projects** — CodeBuild needs its own IAM role to access resources:
```bash
# Create a trust policy for CodeBuild
cat > codebuild-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codebuild.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name CodeBuildServiceRole \
  --assume-role-policy-document file://codebuild-trust-policy.json

aws iam attach-role-policy \
  --role-name CodeBuildServiceRole \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeBuildAdminAccess
```

---

## Core Commands

### 1. Create a CodeBuild Project

```bash
aws codebuild create-project \
  --name my-build-project \
  --source '{
    "type": "CODECOMMIT",
    "location": "https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-app-repo"
  }' \
  --artifacts '{
    "type": "S3",
    "location": "my-artifacts-bucket",
    "path": "builds/",
    "name": "my-build-project",
    "packaging": "ZIP"
  }' \
  --environment '{
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/standard:7.0",
    "computeType": "BUILD_GENERAL1_SMALL"
  }' \
  --service-role arn:aws:iam::123456789012:role/CodeBuildServiceRole
```

**What it does:** Creates a new CodeBuild project with a CodeCommit source, S3 artifact output, and a standard Linux build environment.

**Example Output:**
```json
{
  "project": {
    "name": "my-build-project",
    "arn": "arn:aws:codebuild:us-east-1:123456789012:project/my-build-project",
    "source": {
      "type": "CODECOMMIT",
      "location": "https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-app-repo"
    },
    "artifacts": {
      "type": "S3",
      "location": "my-artifacts-bucket"
    },
    "environment": {
      "type": "LINUX_CONTAINER",
      "image": "aws/codebuild/standard:7.0",
      "computeType": "BUILD_GENERAL1_SMALL"
    },
    "serviceRole": "arn:aws:iam::123456789012:role/CodeBuildServiceRole",
    "created": "2024-01-15T10:00:00.000Z",
    "lastModified": "2024-01-15T10:00:00.000Z"
  }
}
```

---

### 2. Start a Build

```bash
aws codebuild start-build \
  --project-name my-build-project
```

**What it does:** Triggers a new build run for the specified project using its default configuration.

**Example Output:**
```json
{
  "build": {
    "id": "my-build-project:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
    "arn": "arn:aws:codebuild:us-east-1:123456789012:build/my-build-project:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
    "buildNumber": 1,
    "startTime": "2024-01-15T10:05:00.000Z",
    "currentPhase": "QUEUED",
    "buildStatus": "IN_PROGRESS",
    "projectName": "my-build-project"
  }
}
```

---

### 3. Start a Build with Source Override

```bash
aws codebuild start-build \
  --project-name my-build-project \
  --source-version "refs/heads/feature/my-new-feature" \
  --environment-variables-override '[
    {"name": "ENV", "value": "staging", "type": "PLAINTEXT"},
    {"name": "DB_PASSWORD", "value": "/myapp/db/password", "type": "PARAMETER_STORE"}
  ]'
```

**What it does:** Starts a build on a specific branch with environment variable overrides, including a reference to AWS Systems Manager Parameter Store for secrets.

---

### 4. Get Build Details

```bash
aws codebuild batch-get-builds \
  --ids "my-build-project:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111"
```

**What it does:** Retrieves detailed information about one or more specific builds by their IDs.

**Example Output:**
```json
{
  "builds": [
    {
      "id": "my-build-project:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
      "buildStatus": "SUCCEEDED",
      "startTime": "2024-01-15T10:05:00.000Z",
      "endTime": "2024-01-15T10:12:30.000Z",
      "currentPhase": "COMPLETED",
      "phases": [
        {
          "phaseType": "INSTALL",
          "phaseStatus": "SUCCEEDED",
          "durationInSeconds": 15
        },
        {
          "phaseType": "BUILD",
          "phaseStatus": "SUCCEEDED",
          "durationInSeconds": 120
        }
      ],
      "logs": {
        "groupName": "/aws/codebuild/my-build-project",
        "streamName": "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111"
      }
    }
  ]
}
```

---

### 5. List Builds for a Project

```bash
aws codebuild list-builds-for-project \
  --project-name my-build-project \
  --sort-order DESCENDING
```

**What it does:** Returns a list of build IDs for a specific project, sorted in descending order (most recent first).

**Example Output:**
```json
{
  "ids": [
    "my-build-project:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
    "my-build-project:b2c3d4e5-6789-01bc-defg-EXAMPLE22222",
    "my-build-project:c3d4e5f6-7890-12cd-efgh-EXAMPLE33333"
  ]
}
```

---

### 6. List All Projects

```bash
aws codebuild list-projects \
  --sort-by NAME \
  --sort-order ASCENDING
```

**What it does:** Lists all CodeBuild projects in the current account and region.

**Example Output:**
```json
{
  "projects": [
    "my-build-project",
    "my-frontend-build",
    "my-backend-build",
    "my-integration-tests"
  ]
}
```

---

### 7. Get Project Details

```bash
aws codebuild batch-get-projects \
  --names my-build-project my-frontend-build
```

**What it does:** Retrieves detailed configuration for one or more CodeBuild projects by name.

---

### 8. Stop a Running Build

```bash
aws codebuild stop-build \
  --id "my-build-project:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111"
```

**What it does:** Attempts to stop a build that is currently in progress.

**Example Output:**
```json
{
  "build": {
    "id": "my-build-project:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
    "buildStatus": "STOPPED",
    "currentPhase": "COMPLETED"
  }
}
```

---

### 9. Update a Project

```bash
aws codebuild update-project \
  --name my-build-project \
  --environment '{
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/standard:7.0",
    "computeType": "BUILD_GENERAL1_MEDIUM",
    "environmentVariables": [
      {
        "name": "APP_ENV",
        "value": "production",
        "type": "PLAINTEXT"
      }
    ]
  }' \
  --timeout-in-minutes 30
```

**What it does:** Updates an existing project's configuration — in this example, upgrading compute size and setting a build timeout.

---

### 10. Delete a Project

```bash
aws codebuild delete-project \
  --name my-build-project
```

**What it does:** Permanently deletes a CodeBuild project. This does **not** delete associated build history or artifacts.

---

### 11. List Build Batches for a Project

```bash
aws codebuild list-build-batches-for-project \
  --project-name my-build-project \
  --sort-order DESCENDING
```

**What it does:** Lists all build batch IDs associated with a specific project.

---

### 12. Create a Webhook

```bash
aws codebuild create-webhook \
  --project-name my-build-project \
  --filter-groups '[[
    {
      "type": "EVENT",
      "pattern": "PUSH"
    },
    {
      "type": "HEAD_REF",
      "pattern": "^refs/heads/main$"
    }
  ]]'
```

**What it does:** Creates a webhook that triggers builds automatically on push events to the `main` branch.

**Example Output:**
```json
{
  "webhook": {
    "url": "https://codebuild.us-east-1.amazonaws.com/webhooks/...",
    "payloadUrl": "https://codebuild.us-east-1.amazonaws.com/webhooks/...",
    "secret": "EXAMPLE_SECRET_VALUE",
    "filterGroups": [...]
  }
}
```

---

### 13. List Source Credentials

```bash
aws codebuild list-source-credentials
```

**What it does:** Lists source credential configurations (GitHub, GitLab, Bitbucket) connected to CodeBuild.

**Example Output:**
```json
{
  "sourceCredentialsInfos": [
    {
      "arn": "arn:aws:codebuild:us-east-1:123456789012:token/github",
      "serverType": "GITHUB",
      "authType": "PERSONAL_ACCESS_TOKEN"
    }
  ]
}
```

---

### 14. Import Source Credentials

```bash
aws codebuild import-source-credentials \
  --server-type GITHUB \
  --auth-type PERSONAL_ACCESS_TOKEN \
  --token ghp_EXAMPLE_GITHUB_TOKEN_VALUE_HERE
```

**What it does:** Connects a GitHub (or other provider) personal access token to CodeBuild for source authentication.

---

### 15. Invalidate Project Cache

```bash
aws codebuild invalidate-project-cache \
  --project-name my-build-project
```

**What it does:** Forces the next build to ignore the existing cache and rebuild from scratch.

---

## Common Operations

### Create Operations

```bash
# Create a project from a JSON file (recommended for complex configs)
aws codebuild create-project \
  --cli-input-json file://my-project-config.json

# Create a report group
aws codebuild create-report-group \
  --name my-test-report-group \
  --type TEST \
  --export-config '{
    "exportConfigType": "S3",
    "s3Destination": {
      "bucket": "my-reports-bucket",
      "path": "test-reports/",
      "packaging": "NONE",
      "encryptionDisabled": false
    }
  }'

# Create a webhook for GitHub source
aws codebuild create-webhook \
  --project-name my-build-project \
  --build-type BUILD \
  --filter-groups '[[{"type": "EVENT", "pattern": "PULL_REQUEST_CREATED, PULL_REQUEST_UPDATED"}]]'
```

---

### Read / Describe Operations

```bash
# Get full project configuration
aws codebuild batch-get-projects \
  --names my-build-project

# Get detailed build information
aws codebuild batch-get-builds \
  --ids "my-build-project:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111"

# Get report group details
aws codebuild batch-get-report-groups \
  --report-group-arns arn:aws:codebuild:us-east-1:123456789012:report-group/my-test-report-group

# Get reports within a report group
aws codebuild list-reports-for-report-group \
  --report-group-arn arn:aws:codebuild:us-east-1:123456789012:report-group/my-test-report-group

# Get details for specific reports
aws codebuild batch-get-reports \
  --report-arns arn:aws:codebuild:us-east-1:123456789012:report/my-test-report-group:abc123
```

---

### Update Operations

```bash
# Update project source
aws codebuild update-project \
  --name my-build-project \
  --source '{
    "type": "GITHUB",
    "location": "https://github.com/my-org/my-app-repo.git",
    "buildspec": "buildspec-prod.yml"
  }'

# Update project artifacts
aws codebuild update-project \
  --name my-build-project \
  --artifacts '{
    "type": "S3",
    "location": "my-new-artifacts-bucket",
    "packaging": "ZIP"
  }'

# Update webhook filter groups
aws codebuild update-webhook \
  --project-name my-build-project \
  --filter-groups '[[
    {"type": "EVENT", "pattern": "PUSH"},
    {"type": "HEAD_REF", "pattern": "^refs/heads/(main|develop)$"}
  ]]'

# Update a report group
aws codebuild update-report-group \
  --arn arn:aws:codebuild:us-east-1:123456789012:report-