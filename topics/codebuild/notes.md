# CodeBuild

## What is it?

**AWS CodeBuild** is a fully managed, serverless continuous integration (CI) service in the **AWS Developer Tools** category. It compiles source code, runs tests, and produces software packages (build artifacts) that are ready to deploy — without requiring you to provision, manage, or scale your own build servers.

CodeBuild scales continuously and processes multiple builds concurrently, charging only for the compute time consumed during the build. It supports popular build tools and runtimes including Maven, Gradle, npm, pip, Docker, and many others through curated managed images or custom Docker images.

**Service Category:** Developer Tools → CI/CD  
**Type:** Fully Managed, Serverless Build Service  
**Official Name:** AWS CodeBuild

---

## Why do we need it?

### The Problem It Solves

Before managed CI services, teams had to:
- Provision and maintain dedicated build servers (Jenkins, TeamCity, etc.)
- Handle OS patching, scaling, and infrastructure reliability
- Pay for idle compute capacity between builds
- Manage plugin ecosystems and version compatibility

CodeBuild eliminates this operational burden by providing on-demand, ephemeral build environments.

### When to Use It

| Scenario | Why CodeBuild |
|---|---|
| Microservices with many repos | Parallel, isolated builds per service |
| Dockerized applications | Native Docker build support |
| Compliance-sensitive workloads | Ephemeral environments, no state leakage |
| Variable build frequency | Pay-per-minute, no idle cost |
| Multi-language monorepos | Custom images with multiple runtimes |

### Real Business Scenarios

1. **E-commerce platform:** A retail company runs 50+ microservices. CodeBuild triggers a build for each service independently when a PR is merged, running unit tests, SAST scans, and producing ECR-ready Docker images.

2. **Financial services firm:** Requires audit trails for every build artifact. CodeBuild integrates with CloudTrail and S3 for immutable logs, satisfying SOC 2 compliance requirements.

3. **Game studio:** Build times spike before major releases. CodeBuild auto-scales to handle 100 concurrent builds during peak periods without pre-provisioning hardware.

4. **SaaS startup:** Uses CodeBuild as the build stage in a CodePipeline to compile a React frontend, run Jest tests, and deploy to CloudFront — all without managing a single server.

---

## Internal Working

### Build Lifecycle

When a build is triggered, CodeBuild follows this internal sequence:

```
Trigger → Queue → Provision Build Container → Source Download
    → Cache Restore → Build Phases → Artifact Upload → Cache Save → Terminate
```

### Step-by-Step Internal Flow

1. **Trigger:** A build is initiated via API call, CodePipeline, S3 event, GitHub webhook, or scheduled EventBridge rule.

2. **Queue:** The build request enters a regional queue. CodeBuild manages fleet capacity internally — there is no user-visible queue management.

3. **Container Provisioning:** CodeBuild spins up a fresh, isolated Docker container (the "build environment") on AWS-managed compute. The container is based on either:
   - An AWS-managed image (Amazon Linux 2, Ubuntu, Windows Server)
   - A custom Docker image from ECR or Docker Hub

4. **Source Download:** CodeBuild pulls source code from the configured source provider (CodeCommit, S3, GitHub, Bitbucket, GitHub Enterprise). Source credentials are resolved via IAM or Secrets Manager.

5. **Cache Restore:** If caching is enabled (S3 or local cache), CodeBuild downloads and extracts the cache layer to speed up dependency installation.

6. **Build Phases (from `buildspec.yml`):**
   - `install` — Install runtime versions, tools
   - `pre_build` — Login to ECR, run linters
   - `build` — Compile code, run tests
   - `post_build` — Push Docker images, send notifications

7. **Artifact Upload:** Build outputs are uploaded to S3 or passed to CodePipeline as artifacts.

8. **Cache Save:** Updated cache is compressed and uploaded to S3.

9. **Container Termination:** The build container is destroyed. No state persists between builds (unless explicitly stored externally).

### The `buildspec.yml` Engine

The `buildspec.yml` is the heart of CodeBuild. It is a YAML file that defines:
- Runtime versions
- Environment variables
- Build commands per phase
- Artifact paths
- Cache paths
- Reports configuration (test reports)

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────┐
│                    AWS CodeBuild                         │
│                                                          │
│  ┌──────────────┐    ┌───────────────────────────────┐  │
│  │  Build       │    │      Build Environment         │  │
│  │  Project     │───▶│  ┌─────────────────────────┐  │  │
│  │              │    │  │   Docker Container       │  │  │
│  │ - Source     │    │  │   (Managed/Custom Image) │  │  │
│  │ - Env Vars   │    │  │                         │  │  │
│  │ - Artifacts  │    │  │  buildspec.yml phases:  │  │  │
│  │ - Cache      │    │  │  install → pre_build    │  │  │
│  │ - VPC Config │    │  │  build → post_build     │  │  │
│  └──────────────┘    │  └─────────────────────────┘  │  │
│                      └───────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
         │                          │
         ▼                          ▼
   ┌───────────┐            ┌──────────────┐
   │  Source   │            │   Artifacts  │
   │ Providers │            │   (S3, ECR)  │
   │ CodeCommit│            └──────────────┘
   │ GitHub    │
   │ S3        │
   │ Bitbucket │
   └───────────┘
```

### Key Architectural Components

| Component | Description |
|---|---|
| **Build Project** | The configuration unit — defines source, environment, buildspec, artifacts, and IAM role |
| **Build Environment** | The ephemeral Docker container where builds execute |
| **Buildspec** | YAML file defining build commands and phases |
| **Service Role** | IAM role assumed by CodeBuild to access AWS resources |
| **Artifacts** | Output files uploaded to S3 or passed to CodePipeline |
| **Cache** | S3 or local cache to speed up builds |
| **Reports** | Test result reports (JUnit, NUnit, Cucumber) stored in CodeBuild |
| **Batch Builds** | Run multiple builds in parallel within a single build project |

### Compute Types

| Compute Type | vCPU | Memory | Use Case |
|---|---|---|---|
| `BUILD_GENERAL1_SMALL` | 3 | 4 GB | Small projects, scripts |
| `BUILD_GENERAL1_MEDIUM` | 7 | 16 GB | Standard builds |
| `BUILD_GENERAL1_LARGE` | 36 | 72 GB | Large compilations, ML |
| `BUILD_GENERAL1_2XLARGE` | 72 | 144 GB | Very large builds |
| `BUILD_LAMBDA_1GB` | 1 | 1 GB | Lambda-based builds |
| `BUILD_LAMBDA_2GB` | 2 | 2 GB | Lambda-based builds |

### VPC Architecture Pattern

```
┌─────────────────────────────────────────┐
│              VPC                         │
│  ┌─────────────────┐  ┌───────────────┐ │
│  │  Private Subnet  │  │ Private Subnet│ │
│  │                  │  │               │ │
│  │  CodeBuild ENV   │  │  RDS/ElastiC  │ │
│  │  (Build Container│──│  ache/Private │ │
│  │   in ENI)        │  │  Resources    │ │
│  └─────────────────┘  └───────────────┘ │
│           │                              │
│    ┌──────▼──────┐                       │
│    │  NAT Gateway│ (for internet access) │
│    └─────────────┘                       │
└─────────────────────────────────────────┘
```

---

## Real World Example

### Scenario: Containerized Node.js API — Full CI Pipeline

**Context:** A team maintains a Node.js REST API deployed on ECS. Every push to `main` should trigger a build that runs tests, builds a Docker image, pushes to ECR, and outputs the image URI for deployment.

### Step 1: Project Structure

```
my-api/
├── src/
│   └── index.js
├── tests/
│   └── api.test.js
├── Dockerfile
├── package.json
└── buildspec.yml
```

### Step 2: `buildspec.yml`

```yaml
version: 0.2

env:
  variables:
    AWS_DEFAULT_REGION: "us-east-1"
    IMAGE_REPO_NAME: "my-api"
  parameter-store:
    DB_PASSWORD: "/myapp/db/password"

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - echo "Installing dependencies..."
      - npm ci

  pre_build:
    commands:
      - echo "Running linting and tests..."
      - npm run lint
      - npm test -- --reporter=jest-junit
      - echo "Logging into ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME

  build:
    commands:
      - echo "Building Docker image..."
      - docker build -t $REPOSITORY_URI:$IMAGE_TAG .
      - docker tag $REPOSITORY_URI:$IMAGE_TAG $REPOSITORY_URI:latest

  post_build:
    commands:
      - echo "Pushing Docker image to ECR..."
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - docker push $REPOSITORY_URI:latest
      - echo "Writing image definitions file..."
      - printf '[{"name":"my-api","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

reports:
  jest-test-report:
    files:
      - "junit.xml"
    file-format: JUNITXML

artifacts:
  files:
    - imagedefinitions.json

cache:
  paths:
    - "node_modules/**/*"
```

### Step 3: CodeBuild Project Configuration

```bash
aws codebuild create-project \
  --name "my-api-build" \
  --source type=CODECOMMIT,location=https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-api \
  --artifacts type=S3,location=my-artifacts-bucket,name=my-api \
  --environment type=LINUX_CONTAINER,computeType=BUILD_GENERAL1_MEDIUM,\
image=aws/codebuild/standard:7.0,privilegedMode=true \
  --service-role arn:aws:iam::123456789012:role/CodeBuildServiceRole \
  --region us-east-1
```

> **Note:** `privilegedMode=true` is required for Docker-in-Docker builds.

### Step 4: IAM Service Role Permissions

The CodeBuild service role needs:
- `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage` (ECR push)
- `s3:PutObject`, `s3:GetObject` (artifacts and cache)
- `logs:CreateLogGroup`, `logs:PutLogEvents` (CloudWatch Logs)
- `ssm:GetParameters` (Parameter Store access)
- `codecommit:GitPull` (source access)

### Step 5: CodePipeline Integration

```
[CodeCommit] → [CodePipeline] → [CodeBuild Stage] → [ECS Deploy Stage]
                                      │
                                  Runs buildspec.yml
                                  Produces imagedefinitions.json
                                      │
                                  Passed to ECS Deploy
```

### Step 6: Outcome

- Build takes ~3 minutes
- Jest test results appear in CodeBuild Reports console
- Docker image tagged with Git commit SHA is in ECR
- ECS service updated with the new image URI
- Total cost: ~$0.005 per build (MEDIUM compute, 3 minutes)

---

## Advantages

1. **Fully Managed:** No servers to provision, patch, or maintain. AWS handles the underlying infrastructure entirely.

2. **True Serverless Scaling:** Scales to thousands of concurrent builds without configuration. No build queues caused by insufficient agents.

3. **Pay-Per-Use:** Billed per build minute — zero cost when not building. Ideal for teams with variable build frequency.

4. **Security Isolation:** Each build runs in a fresh, ephemeral container. No risk of build contamination or secret leakage between builds.

5. **Native AWS Integration:** Deep integration with CodePipeline, CodeCommit, ECR, S3, Secrets Manager, Parameter Store, and CloudWatch.

6. **Custom Environments:** Use any Docker image as a build environment, supporting virtually any language, tool, or framework.

7. **Privileged Mode:** Supports Docker-in-Docker for building container images natively.

8. **Test Reporting:** Native support for JUnit, NUnit, Cucumber, TestNG test report formats with trend visualization.

9. **Batch Builds:** Run multiple related builds in parallel or in a dependency graph within a single project invocation.

10. **VPC Support:** Build containers can be placed inside a VPC to access private resources (RDS, ElastiCache, internal APIs).

11. **Compliance:** Supports FIPS endpoints, VPC isolation, and integrates with AWS Config and CloudTrail for audit trails.

---

## Limitations

### Hard Limits (Default Quotas)

| Limit | Value |
|---|---|
| Maximum build timeout | 8 hours |
| Default build timeout | 60 minutes |
| Maximum concurrent builds per account (per region) | 60 (can be increased) |
| Maximum number of build projects | 5,000 |
| Maximum buildspec size | 16 KB (inline) |
| Maximum environment variables | 100 |
| Maximum artifact size | No explicit limit (S3 limit applies) |
| Maximum source file size | 1 GB (zipped from S3) |
| Maximum cache archive size | 5 GB |

### Functional Limitations

- **No persistent build agents:** Each build starts fresh — no warm agents. Cold start time adds latency (typically 30–90 seconds).
- **No native Windows ARM support:** ARM compute is Linux-only.
- **Limited local caching:** Local cache is tied to a specific build fleet host — not guaranteed across builds.
- **No interactive debugging:** Cannot SSH into a running build container (unlike self-hosted Jenkins). CodeBuild does offer a local build agent for local testing.
- **Docker layer caching:** Not natively supported for remote builds (workarounds exist using `--cache-from`).
- **No built-in secrets masking in logs:** Secrets printed via `echo` will appear in logs unless explicitly managed.
- **Batch build complexity:** Batch builds require additional configuration and have separate quotas.
- **VPC builds require NAT Gateway:** For internet access within VPC, a NAT Gateway is required (adds cost).

---

## Best Practices

### 1. Store `buildspec.yml` in Source Control
Always keep `buildspec.yml` in the root of your repository rather than defining it inline in the CodeBuild project. This enables versioning and peer review of build logic.

### 2. Use Least-Privilege IAM Roles
Create a dedicated IAM service role per project with only the permissions required. Avoid using `AdministratorAccess` or broad wildcards.

```json
{