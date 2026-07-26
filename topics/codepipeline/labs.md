# CodePipeline — Hands-On Labs

## Lab 1: Getting Started with CodePipeline

### Objective

In this lab, you will build a basic CI/CD pipeline using AWS CodePipeline that automatically deploys a static web application. You will connect a CodeCommit repository as the source, use CodeBuild to build the application, and deploy it to an S3 bucket configured for static website hosting. By the end of this lab, you will understand the core concepts of pipeline stages, actions, and transitions.

### Prerequisites

**AWS Services Required:**
- AWS CodePipeline
- AWS CodeCommit
- AWS CodeBuild
- Amazon S3
- AWS IAM

**IAM Permissions Required:**
- `AWSCodePipelineFullAccess`
- `AWSCodeCommitFullAccess`
- `AWSCodeBuildAdminAccess`
- `AmazonS3FullAccess`
- `IAMFullAccess` (to create service roles)

**Tools Required:**
- AWS CLI v2 installed and configured
- Git client installed
- A text editor (VS Code recommended)

**Estimated Time:** 45–60 minutes  
**Estimated Cost:** < $1.00 USD

---

### Steps

#### Step 1: Create the S3 Artifact Bucket

CodePipeline requires an S3 bucket to store pipeline artifacts between stages.

**Console:**
1. Navigate to **S3** → **Create bucket**
2. Set **Bucket name**: `codepipeline-artifacts-<your-account-id>-us-east-1`
3. Set **Region**: `us-east-1`
4. Under **Versioning**, enable **Bucket Versioning**
5. Leave all other settings as default → **Create bucket**

**CLI:**
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"
ARTIFACT_BUCKET="codepipeline-artifacts-${ACCOUNT_ID}-${REGION}"

aws s3api create-bucket \
  --bucket "${ARTIFACT_BUCKET}" \
  --region "${REGION}"

aws s3api put-bucket-versioning \
  --bucket "${ARTIFACT_BUCKET}" \
  --versioning-configuration Status=Enabled

echo "Artifact bucket created: ${ARTIFACT_BUCKET}"
```

**Verify:**
```bash
aws s3api get-bucket-versioning --bucket "${ARTIFACT_BUCKET}"
```

**Expected Output:**
```json
{
    "Status": "Enabled"
}
```

---

#### Step 2: Create the S3 Deployment Bucket (Static Website)

**Console:**
1. Navigate to **S3** → **Create bucket**
2. Set **Bucket name**: `my-webapp-deploy-<your-account-id>`
3. Under **Block Public Access**, **uncheck** "Block all public access"
4. Acknowledge the warning → **Create bucket**
5. Go to the bucket → **Properties** → **Static website hosting** → **Enable**
6. Set **Index document**: `index.html`
7. Set **Error document**: `error.html`
8. Save changes
9. Go to **Permissions** → **Bucket Policy** → Add the following policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-webapp-deploy-<your-account-id>/*"
    }
  ]
}
```

**CLI:**
```bash
DEPLOY_BUCKET="my-webapp-deploy-${ACCOUNT_ID}"

aws s3api create-bucket \
  --bucket "${DEPLOY_BUCKET}" \
  --region "${REGION}"

aws s3api delete-public-access-block \
  --bucket "${DEPLOY_BUCKET}"

aws s3api put-bucket-website \
  --bucket "${DEPLOY_BUCKET}" \
  --website-configuration '{
    "IndexDocument": {"Suffix": "index.html"},
    "ErrorDocument": {"Key": "error.html"}
  }'

aws s3api put-bucket-policy \
  --bucket "${DEPLOY_BUCKET}" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Sid\": \"PublicReadGetObject\",
      \"Effect\": \"Allow\",
      \"Principal\": \"*\",
      \"Action\": \"s3:GetObject\",
      \"Resource\": \"arn:aws:s3:::${DEPLOY_BUCKET}/*\"
    }]
  }"

echo "Deploy bucket created: ${DEPLOY_BUCKET}"
```

**Verify:**
```bash
aws s3api get-bucket-website --bucket "${DEPLOY_BUCKET}"
```

**Expected Output:**
```json
{
    "IndexDocument": { "Suffix": "index.html" },
    "ErrorDocument": { "Key": "error.html" }
}
```

---

#### Step 3: Create the CodeCommit Repository and Push Sample Application

**Console:**
1. Navigate to **CodeCommit** → **Create repository**
2. Set **Repository name**: `my-webapp`
3. Set **Description**: `Sample web application for CodePipeline lab`
4. Click **Create**

**CLI:**
```bash
aws codecommit create-repository \
  --repository-name my-webapp \
  --repository-description "Sample web application for CodePipeline lab"
```

**Expected Output:**
```json
{
    "repositoryMetadata": {
        "repositoryName": "my-webapp",
        "cloneUrlHttp": "https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-webapp",
        ...
    }
}
```

Now clone the repository and add application files:

```bash
# Configure git credentials helper
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true

# Clone the repository
git clone https://git-codecommit.${REGION}.amazonaws.com/v1/repos/my-webapp
cd my-webapp

# Create index.html
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My CodePipeline App</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 50px; background: #f0f8ff; }
        h1 { color: #232f3e; }
        .badge { background: #ff9900; color: white; padding: 10px 20px; border-radius: 5px; }
    </style>
</head>
<body>
    <h1>🚀 Deployed via AWS CodePipeline!</h1>
    <p>Version: <span class="badge">1.0.0</span></p>
    <p>Build Time: AUTO_REPLACED_BY_BUILD</p>
</body>
</html>
EOF

# Create error.html
cat > error.html << 'EOF'
<!DOCTYPE html>
<html><body><h1>404 - Page Not Found</h1></body></html>
EOF

# Create buildspec.yml for CodeBuild
cat > buildspec.yml << 'EOF'
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 18
  pre_build:
    commands:
      - echo "Starting build at $(date)"
      - BUILD_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  build:
    commands:
      - echo "Running build phase"
      - sed -i "s/AUTO_REPLACED_BY_BUILD/${BUILD_TIME}/g" index.html
      - echo "Build completed successfully"
  post_build:
    commands:
      - echo "Post-build phase complete"

artifacts:
  files:
    - index.html
    - error.html
  discard-paths: yes
EOF

# Commit and push
git add .
git commit -m "Initial commit: Add web app and buildspec"
git push origin main
```

**Verify:**
```bash
aws codecommit get-branch \
  --repository-name my-webapp \
  --branch-name main
```

**Expected Output:**
```json
{
    "branch": {
        "branchName": "main",
        "commitId": "a1b2c3d4..."
    }
}
```

---

#### Step 4: Create the CodeBuild Project

**Console:**
1. Navigate to **CodeBuild** → **Create build project**
2. Set **Project name**: `my-webapp-build`
3. Under **Source**: Select **AWS CodeCommit**, Repository: `my-webapp`, Branch: `main`
4. Under **Environment**:
   - Managed image: **Amazon Linux 2**
   - Runtime: **Standard**
   - Image: `aws/codebuild/amazonlinux2-x86_64-standard:5.0`
   - Service role: **New service role** → Name: `codebuild-my-webapp-service-role`
5. Under **Buildspec**: Use a buildspec file (default)
6. Under **Artifacts**: No artifacts (CodePipeline will handle this)
7. Click **Create build project**

**CLI:**
```bash
# Create CodeBuild service role
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
  --role-name codebuild-my-webapp-service-role \
  --assume-role-policy-document file://codebuild-trust-policy.json

aws iam attach-role-policy \
  --role-name codebuild-my-webapp-service-role \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeBuildAdminAccess

aws iam attach-role-policy \
  --role-name codebuild-my-webapp-service-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

aws iam attach-role-policy \
  --role-name codebuild-my-webapp-service-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Create CodeBuild project
aws codebuild create-project \
  --name my-webapp-build \
  --source '{
    "type": "CODECOMMIT",
    "location": "https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-webapp",
    "buildspec": "buildspec.yml"
  }' \
  --artifacts '{"type": "NO_ARTIFACTS"}' \
  --environment '{
    "type": "LINUX_CONTAINER",
    "computeType": "BUILD_GENERAL1_SMALL",
    "image": "aws/codebuild/amazonlinux2-x86_64-standard:5.0"
  }' \
  --service-role "codebuild-my-webapp-service-role"
```

**Verify:**
```bash
aws codebuild batch-get-projects --names my-webapp-build \
  --query 'projects[0].name'
```

**Expected Output:**
```
"my-webapp-build"
```

---

#### Step 5: Create the CodePipeline Service Role

**Console:**
1. Navigate to **IAM** → **Roles** → **Create role**
2. Select **AWS service** → **CodePipeline**
3. Attach policy: `AWSCodePipelineFullAccess`
4. Role name: `AWSCodePipelineServiceRole-us-east-1-my-webapp`
5. Create role

**CLI:**
```bash
cat > codepipeline-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codepipeline.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name AWSCodePipelineServiceRole-us-east-1-my-webapp \
  --assume-role-policy-document file://codepipeline-trust-policy.json

aws iam attach-role-policy \
  --role-name AWSCodePipelineServiceRole-us-east-1-my-webapp \
  --policy-arn arn:aws:iam::aws:policy/AWSCodePipelineFullAccess

aws iam attach-role-policy \
  --role-name AWSCodePipelineServiceRole-us-east-1-my-webapp \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-role-policy \
  --role-name AWSCodePipelineServiceRole-us-east-1-my-webapp \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeCommitFullAccess

aws iam attach-role-policy \
  --role-name AWSCodePipelineServiceRole-us-east-1-my-webapp \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeBuildAdminAccess
```

---

#### Step 6: Create the CodePipeline

**Console:**
1. Navigate to **CodePipeline** → **Create pipeline**
2. Set **Pipeline name**: `my-webapp-pipeline`
3. Set **Service role**: Existing service role → `AWSCodePipelineServiceRole-us-east-1-my-webapp`
4. Under **Artifact store**: Custom location → Select `codepipeline-artifacts-<account-id>-us-east-1`
5. Click **Next**

**Source Stage:**
- Provider: **AWS CodeCommit**
- Repository name: `my-webapp`
- Branch name: `main`
- Detection option: **Amazon CloudWatch Events** (recommended)
- Click **Next**

**Build Stage:**
- Provider: **AWS CodeBuild**
- Region: `us-east-1`
- Project name: `my-webapp-build`
- Click **Next**

**Deploy Stage:**
- Provider: **Amazon S3**
- Region: `us-east-1`
- Bucket: `my-webapp-deploy-<account-id>`
- Check **Extract file before deploy**
- Click **Next** → **Create pipeline**

**CLI:**
```bash
PIPELINE_ROLE_ARN=$(aws iam get-role \
  --role-name AWSCodePipelineServiceRole-us-east-1-my-webapp \
  --query 'Role.Arn' --output text)

cat > pipeline-definition.json << EOF
{
  "pipeline": {
    "name": "my-webapp-pipeline",
    "roleArn": "${PIPELINE_ROLE_ARN}",
    "artifactStore": {
      "type": "S3",
      "location": "${ARTIFACT_BUCKET}"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "Source",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "CodeCommit",
              "version": "1"
            },
            "configuration": {
              "RepositoryName": "my-webapp",
              "BranchName": "main",
              "PollForSourceChanges": "false"
            },
            "outputArtifacts": [{"name": "SourceOutput"}],
            "runOrder": 1
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "Build",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider