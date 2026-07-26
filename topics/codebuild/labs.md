# CodeBuild — Hands-On Labs

## Lab 1: Getting Started with CodeBuild

### Objective
In this lab, you will create your first AWS CodeBuild project from scratch. You will build a simple Node.js application, configure a `buildspec.yml` file, store build artifacts in Amazon S3, and review build logs in CloudWatch. By the end of this lab, you will understand the core components of CodeBuild: source, environment, buildspec, and artifacts.

---

### Prerequisites

**AWS Services Required:**
- AWS CodeBuild
- Amazon S3
- AWS IAM
- Amazon CloudWatch Logs

**IAM Permissions Required:**
- `codebuild:*`
- `s3:CreateBucket`, `s3:PutObject`, `s3:GetObject`
- `iam:CreateRole`, `iam:AttachRolePolicy`
- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`

**Tools:**
- AWS CLI v2 installed and configured (`aws configure`)
- A text editor (VS Code recommended)
- Git (optional but recommended)
- Node.js 18+ (to understand the sample app)

**Estimated Time:** 45–60 minutes  
**Estimated Cost:** < $0.10

---

### Steps

#### Step 1: Create the S3 Bucket for Source and Artifacts

**Console:**
1. Navigate to **S3** → **Create bucket**
2. Set **Bucket name**: `codebuild-lab1-<your-account-id>` (must be globally unique)
3. Set **Region**: `us-east-1` (or your preferred region)
4. Leave **Block all public access** enabled
5. Click **Create bucket**

**CLI:**
```bash
# Set your account ID as a variable
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"
BUCKET_NAME="codebuild-lab1-${ACCOUNT_ID}"

# Create the S3 bucket
aws s3 mb s3://${BUCKET_NAME} --region ${REGION}

# Enable versioning (best practice)
aws s3api put-bucket-versioning \
  --bucket ${BUCKET_NAME} \
  --versioning-configuration Status=Enabled
```

**Verify:**
```bash
aws s3 ls | grep codebuild-lab1
```
**Expected output:**
```
2024-01-15 10:00:00 codebuild-lab1-123456789012
```

---

#### Step 2: Create the Sample Node.js Application and buildspec.yml

Create a local project directory with the following structure:

```
my-node-app/
├── src/
│   └── app.js
├── test/
│   └── app.test.js
├── package.json
└── buildspec.yml
```

**Create the project files:**

```bash
mkdir -p my-node-app/src my-node-app/test
cd my-node-app
```

**`src/app.js`:**
```javascript
// src/app.js
function add(a, b) {
  return a + b;
}

function multiply(a, b) {
  return a * b;
}

function greet(name) {
  return `Hello, ${name}! Welcome to CodeBuild.`;
}

module.exports = { add, multiply, greet };

// Simple HTTP server entry point
const http = require('http');
const PORT = process.env.PORT || 3000;

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ message: greet('World'), version: '1.0.0' }));
});

if (require.main === module) {
  server.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
  });
}
```

**`test/app.test.js`:**
```javascript
// test/app.test.js
const { add, multiply, greet } = require('../src/app');

// Simple test runner (no external dependencies)
let passed = 0;
let failed = 0;

function test(description, fn) {
  try {
    fn();
    console.log(`  ✓ ${description}`);
    passed++;
  } catch (err) {
    console.error(`  ✗ ${description}: ${err.message}`);
    failed++;
  }
}

function assert(condition, message) {
  if (!condition) throw new Error(message || 'Assertion failed');
}

console.log('\nRunning tests...\n');

test('add() returns correct sum', () => {
  assert(add(2, 3) === 5, 'Expected 2 + 3 = 5');
});

test('add() handles negative numbers', () => {
  assert(add(-1, 1) === 0, 'Expected -1 + 1 = 0');
});

test('multiply() returns correct product', () => {
  assert(multiply(4, 5) === 20, 'Expected 4 * 5 = 20');
});

test('greet() returns correct greeting', () => {
  const result = greet('Alice');
  assert(result === 'Hello, Alice! Welcome to CodeBuild.', `Unexpected: ${result}`);
});

console.log(`\nResults: ${passed} passed, ${failed} failed\n`);

if (failed > 0) {
  process.exit(1);
}
```

**`package.json`:**
```json
{
  "name": "my-node-app",
  "version": "1.0.0",
  "description": "Sample app for CodeBuild lab",
  "main": "src/app.js",
  "scripts": {
    "test": "node test/app.test.js",
    "start": "node src/app.js"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

**`buildspec.yml`:**
```yaml
# buildspec.yml — CodeBuild build specification
version: 0.2

env:
  variables:
    APP_NAME: "my-node-app"
    NODE_ENV: "test"

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - echo "=== INSTALL PHASE ==="
      - echo "Node version: $(node --version)"
      - echo "NPM version: $(npm --version)"
      - npm install --production=false

  pre_build:
    commands:
      - echo "=== PRE_BUILD PHASE ==="
      - echo "Build started on $(date)"
      - echo "Source version: $CODEBUILD_SOURCE_VERSION"

  build:
    commands:
      - echo "=== BUILD PHASE ==="
      - echo "Running tests..."
      - npm test
      - echo "Creating deployment package..."
      - mkdir -p dist
      - cp -r src dist/
      - cp package.json dist/
      - echo "Build completed on $(date)"

  post_build:
    commands:
      - echo "=== POST_BUILD PHASE ==="
      - echo "Build status: $CODEBUILD_BUILD_SUCCEEDING"
      - |
        if [ "$CODEBUILD_BUILD_SUCCEEDING" = "1" ]; then
          echo "Build succeeded! Packaging artifacts..."
        else
          echo "Build failed. Check logs above."
        fi

artifacts:
  files:
    - dist/**/*
    - package.json
    - buildspec.yml
  name: "$APP_NAME-$(date +%Y%m%d-%H%M%S)"
  discard-paths: no

cache:
  paths:
    - node_modules/**/*

reports:
  build-report:
    files:
      - '**/*'
    base-directory: 'dist'

logs:
  cloudwatch:
    group-name: /aws/codebuild/my-node-app
    stream-name: build-log
  s3:
    location: codebuild-lab1-ACCOUNT_ID/logs
    encryption-disabled: false
```

> **Note:** Replace `ACCOUNT_ID` in `buildspec.yml` with your actual account ID, or leave the S3 log section out for now.

---

#### Step 3: Upload Source Code to S3

```bash
# From inside the my-node-app directory
cd my-node-app

# Create a zip archive of the source
zip -r ../my-node-app-source.zip .

# Upload to S3
aws s3 cp ../my-node-app-source.zip \
  s3://${BUCKET_NAME}/source/my-node-app-source.zip

# Verify upload
aws s3 ls s3://${BUCKET_NAME}/source/
```

**Expected output:**
```
2024-01-15 10:05:00       1234 my-node-app-source.zip
```

---

#### Step 4: Create the CodeBuild IAM Service Role

**Console:**
1. Navigate to **IAM** → **Roles** → **Create role**
2. Select **AWS service** → **CodeBuild**
3. Click **Next: Permissions**
4. Attach policies: `AmazonS3FullAccess`, `CloudWatchLogsFullAccess`
5. Name the role: `CodeBuildLab1ServiceRole`
6. Click **Create role**

**CLI:**
```bash
# Create the trust policy document
cat > /tmp/codebuild-trust-policy.json << 'EOF'
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

# Create the IAM role
aws iam create-role \
  --role-name CodeBuildLab1ServiceRole \
  --assume-role-policy-document file:///tmp/codebuild-trust-policy.json \
  --description "Service role for CodeBuild Lab 1"

# Create a custom policy for S3 and CloudWatch access
cat > /tmp/codebuild-permissions.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:PutObject",
        "s3:GetBucketAcl",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::${BUCKET_NAME}",
        "arn:aws:s3:::${BUCKET_NAME}/*"
      ]
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# Create and attach the policy
aws iam create-policy \
  --policy-name CodeBuildLab1Policy \
  --policy-document file:///tmp/codebuild-permissions.json

aws iam attach-role-policy \
  --role-name CodeBuildLab1ServiceRole \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/CodeBuildLab1Policy

# Verify role creation
aws iam get-role --role-name CodeBuildLab1ServiceRole \
  --query 'Role.RoleName' --output text
```

**Expected output:**
```
CodeBuildLab1ServiceRole
```

---

#### Step 5: Create the CodeBuild Project

**Console:**
1. Navigate to **CodeBuild** → **Build projects** → **Create build project**
2. **Project configuration:**
   - Project name: `my-node-app-build`
   - Description: `Lab 1 - First CodeBuild project`
3. **Source:**
   - Source provider: **Amazon S3**
   - Bucket: `codebuild-lab1-<your-account-id>`
   - S3 object key: `source/my-node-app-source.zip`
4. **Environment:**
   - Environment image: **Managed image**
   - Operating system: **Amazon Linux 2023**
   - Runtime: **Standard**
   - Image: `aws/codebuild/amazonlinux-x86_64-standard:5.0`
   - Service role: **Existing service role** → `CodeBuildLab1ServiceRole`
5. **Buildspec:**
   - Select **Use a buildspec file**
   - Buildspec name: `buildspec.yml`
6. **Artifacts:**
   - Type: **Amazon S3**
   - Bucket name: `codebuild-lab1-<your-account-id>`
   - Path: `artifacts/`
   - Packaging: **Zip**
7. **Logs:**
   - CloudWatch: Enabled
   - Group name: `/aws/codebuild/my-node-app`
8. Click **Create build project**

**CLI:**
```bash
# Create the CodeBuild project using JSON configuration
cat > /tmp/codebuild-project.json << EOF
{
  "name": "my-node-app-build",
  "description": "Lab 1 - First CodeBuild project",
  "source": {
    "type": "S3",
    "location": "${BUCKET_NAME}/source/my-node-app-source.zip",
    "buildspec": "buildspec.yml"
  },
  "artifacts": {
    "type": "S3",
    "location": "${BUCKET_NAME}",
    "path": "artifacts/",
    "packaging": "ZIP",
    "name": "my-node-app-build"
  },
  "environment": {
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/amazonlinux-x86_64-standard:5.0",
    "computeType": "BUILD_GENERAL1_SMALL",
    "environmentVariables": [
      {
        "name": "NODE_ENV",
        "value": "test",
        "type": "PLAINTEXT"
      }
    ],
    "privilegedMode": false,
    "imagePullCredentialsType": "CODEBUILD"
  },
  "serviceRole": "arn:aws:iam::${ACCOUNT_ID}:role/CodeBuildLab1ServiceRole",
  "logsConfig": {
    "cloudWatchLogs": {
      "status": "ENABLED",
      "groupName": "/aws/codebuild/my-node-app",
      "streamName": "build-log"
    },
    "s3Logs": {
      "status": "DISABLED"
    }
  },
  "timeoutInMinutes": 15,
  "queuedTimeoutInMinutes": 60
}
EOF

aws codebuild create-project \
  --cli-input-json file:///tmp/codebuild-project.json \
  --region ${REGION}

# Verify project creation
aws codebuild batch-get-projects \
  --names my-node-app-build \
  --query 'projects[0].name' \
  --output text
```

**Expected output:**
```
my-node-app-build
```

---

#### Step 6: Start a Build and Monitor It

**Console:**
1. Navigate to **CodeBuild** → **Build projects** → `my-node-app-build`
2. Click **Start build**
3. Leave all defaults and click **Start build**
4. Watch the **Build status** and **Phase details** tabs

**CLI:**
```bash
# Start the build
BUILD_ID=$(aws codebuild start-build \
  --project-name my-node-app-build \
  --query 'build.id' \
  --output text)

echo "Build started: ${BUILD_ID}"

# Poll build status every 10 seconds
echo "Waiting for build to complete..."
while true; do
  STATUS=$(aws codebuild batch-get-builds \
    --ids "${BUILD_ID}" \
    --query 'builds[0].buildStatus' \
    --output text