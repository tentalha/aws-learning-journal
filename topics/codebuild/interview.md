# CodeBuild — Interview Questions

---

## Easy

### Q1. What is AWS CodeBuild, and what problem does it solve?

**Answer:**
AWS CodeBuild is a fully managed continuous integration (CI) service that compiles source code, runs tests, and produces software packages ready for deployment. It eliminates the need to provision, manage, and scale your own build servers. CodeBuild scales continuously and processes multiple builds concurrently, so your builds are not left waiting in a queue. You pay only for the compute time you consume.

---

### Q2. What is a `buildspec.yml` file, and where does it live?

**Answer:**
A `buildspec.yml` is a YAML-formatted file that defines the build commands and settings that CodeBuild uses to run a build. It contains a series of **phases** (install, pre_build, build, post_build), environment variables, artifacts, and cache configurations.

By default, CodeBuild looks for `buildspec.yml` in the **root directory** of the source code. Alternatively, you can specify a different name or path in the CodeBuild project configuration, or you can define the buildspec inline within the project itself.

---

### Q3. What are the four main phases in a standard `buildspec.yml`?

**Answer:**

| Phase | Purpose |
|---|---|
| `install` | Install dependencies required for the build (e.g., runtime versions, packages) |
| `pre_build` | Commands to run before the build (e.g., log in to ECR, run linters) |
| `build` | The actual build commands (e.g., `mvn package`, `npm run build`) |
| `post_build` | Commands to run after the build (e.g., push Docker images, send notifications) |

Each phase contains a `commands` block listing shell commands to execute sequentially.

---

### Q4. What compute types are available in CodeBuild, and how do they differ?

**Answer:**
CodeBuild offers several compute types that vary by CPU and memory:

| Compute Type | vCPUs | Memory | Disk |
|---|---|---|---|
| `BUILD_GENERAL1_SMALL` | 3 | 4 GB | 64 GB |
| `BUILD_GENERAL1_MEDIUM` | 7 | 16 GB | 128 GB |
| `BUILD_GENERAL1_LARGE` | 36 | 72 GB | 128 GB |
| `BUILD_GENERAL1_2XLARGE` | 72 | 144 GB | 128 GB |

There are also **Lambda compute** types (for very fast, lightweight builds) and **ARM-based** compute options. The choice affects both build performance and cost.

---

### Q5. How does CodeBuild integrate with other AWS developer tools?

**Answer:**
CodeBuild integrates natively with:
- **AWS CodePipeline** — as a build or test action within a pipeline stage
- **AWS CodeCommit** — as a source repository trigger
- **Amazon ECR** — to pull base images and push built Docker images
- **Amazon S3** — for source artifacts and build output artifacts
- **AWS CodeStar Notifications / Amazon SNS** — for build notifications
- **AWS Secrets Manager / Parameter Store** — for secure environment variable injection
- **Amazon CloudWatch Logs** — for streaming and storing build logs
- **GitHub / GitLab / Bitbucket** — via webhooks for source integration

---

## Medium

### Q1. Explain how CodeBuild caching works and the different cache types available.

**Answer:**
CodeBuild caching improves build performance by reusing previously downloaded dependencies or build outputs, reducing redundant work across builds.

**Cache Types:**

1. **Amazon S3 Cache**
   - Build cache is stored in an S3 bucket.
   - Slower to restore/save compared to local cache (network I/O).
   - Useful for sharing cache across multiple build projects or when local cache is unavailable.
   - Configured with a `cache` block in `buildspec.yml` specifying `paths`.

2. **Local Cache**
   - Cache is stored on the build host itself.
   - Much faster than S3 cache (no network transfer).
   - Three sub-modes:
     - **LOCAL_SOURCE_CACHE**: Caches the source code checkout.
     - **LOCAL_DOCKER_LAYER_CACHE**: Caches Docker image layers (requires privileged mode).
     - **LOCAL_CUSTOM_CACHE**: Caches arbitrary paths you define (e.g., `node_modules`, Maven `.m2`).
   - Only available when using the same build fleet (not guaranteed across different build hosts).

**Example buildspec cache configuration:**
```yaml
cache:
  paths:
    - '/root/.m2/**/*'
    - 'node_modules/**/*'
```

**Best Practice:** Combine S3 cache for cross-build persistence with local cache for speed within a project.

---

### Q2. How do you securely inject sensitive values (e.g., API keys, passwords) into a CodeBuild build?

**Answer:**
Never hardcode secrets in `buildspec.yml` or environment variables in plaintext. CodeBuild provides two secure mechanisms:

**1. AWS Systems Manager Parameter Store**
- Store secrets as `SecureString` parameters.
- Reference them in `buildspec.yml` under the `env.parameter-store` block:
```yaml
env:
  parameter-store:
    DB_PASSWORD: /myapp/prod/db_password
    API_KEY: /myapp/prod/api_key
```
- CodeBuild's IAM role must have `ssm:GetParameters` permission on those paths.

**2. AWS Secrets Manager**
- Store secrets as JSON key-value pairs.
- Reference them under `env.secrets-manager`:
```yaml
env:
  secrets-manager:
    MY_SECRET: arn:aws:secretsmanager:us-east-1:123456789:secret:myapp/secret:key
```
- CodeBuild's IAM role must have `secretsmanager:GetSecretValue` permission.

**Additional Security Practices:**
- Use `exported-variables` carefully — only export what downstream stages need.
- Enable **CloudTrail** to audit secret access.
- Avoid printing secrets with `set -x` or `echo` commands in build scripts.
- Use `parameter-store` or `secrets-manager` over plaintext `env.variables` for any sensitive value.

---

### Q3. What is a CodeBuild VPC configuration, and when would you use it?

**Answer:**
By default, CodeBuild builds run in a managed AWS network environment without direct access to resources in your VPC. VPC configuration allows CodeBuild build containers to run inside your VPC, enabling access to private resources.

**When to Use VPC Configuration:**
- Accessing **private RDS databases** for integration tests.
- Connecting to **private Elasticsearch/OpenSearch** clusters.
- Reaching **internal APIs** not exposed to the internet.
- Accessing resources in a **private subnet** (e.g., internal Nexus artifact repositories).
- Ensuring build traffic routes through your **NAT Gateway** for auditing.

**Configuration Requirements:**
- Specify VPC ID, subnet IDs (private subnets recommended), and security group IDs in the CodeBuild project.
- The build container gets an ENI in the specified subnets.
- For internet access from within VPC, a **NAT Gateway** in a public subnet is required (CodeBuild containers don't get public IPs in VPC mode).
- The IAM role needs `ec2:CreateNetworkInterface`, `ec2:DescribeNetworkInterfaces`, `ec2:DeleteNetworkInterface` permissions.

**Trade-off:** VPC builds have slightly longer startup times due to ENI provisioning.

---

### Q4. How does CodeBuild handle Docker image builds, and what is "privileged mode"?

**Answer:**
CodeBuild can build Docker images within its build environment, but this requires special configuration because it involves running Docker-in-Docker (DinD).

**Privileged Mode:**
- By default, CodeBuild build containers run in **non-privileged mode** for security isolation.
- To run Docker commands (`docker build`, `docker push`) inside a CodeBuild build, you must enable **privileged mode** in the project settings.
- Privileged mode grants the build container elevated Linux capabilities, allowing it to run the Docker daemon.

**Typical Docker Build Workflow in buildspec.yml:**
```yaml
phases:
  pre_build:
    commands:
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
  build:
    commands:
      - docker build -t $IMAGE_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker tag $IMAGE_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION $ECR_REGISTRY/$IMAGE_NAME:latest
  post_build:
    commands:
      - docker push $ECR_REGISTRY/$IMAGE_NAME:latest
```

**Security Considerations:**
- Only enable privileged mode when necessary.
- Use **LOCAL_DOCKER_LAYER_CACHE** to speed up repeated Docker builds.
- Use multi-stage Docker builds to minimize image size and attack surface.
- The CodeBuild IAM role needs `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, etc.

---

### Q5. What are CodeBuild environment variables, and what built-in variables does AWS provide?

**Answer:**
CodeBuild supports environment variables at multiple levels:

**Types of Environment Variables:**
1. **Plaintext** — Defined in project settings or `buildspec.yml`; visible in logs and console. Use only for non-sensitive values.
2. **Parameter Store** — Fetched from SSM at build time.
3. **Secrets Manager** — Fetched from Secrets Manager at build time.

**Built-in CodeBuild Environment Variables:**

| Variable | Description |
|---|---|
| `CODEBUILD_BUILD_ID` | Unique build ID (e.g., `myproject:abc123`) |
| `CODEBUILD_BUILD_NUMBER` | Sequential build number within the project |
| `CODEBUILD_SOURCE_VERSION` | Source version (commit ID, branch, tag) |
| `CODEBUILD_RESOLVED_SOURCE_VERSION` | Resolved commit SHA (useful for tagging Docker images) |
| `CODEBUILD_SOURCE_REPO_URL` | URL of the source repository |
| `CODEBUILD_BUILD_SUCCEEDING` | `1` if build is succeeding, `0` if failing |
| `CODEBUILD_INITIATOR` | Entity that started the build (user or pipeline) |
| `AWS_DEFAULT_REGION` | AWS region of the build |
| `AWS_ACCOUNT_ID` | AWS account ID |
| `CODEBUILD_LOG_PATH` | CloudWatch Logs path for the build |

**Variable Precedence (highest to lowest):**
1. Start build override variables
2. Project-level environment variables
3. `buildspec.yml` environment variables

---

## Hard

### Q1. Deep dive: How does CodeBuild's IAM permission model work, and how would you design a least-privilege IAM role for a complex build?

**Answer:**
CodeBuild uses an **IAM service role** that is assumed by the build container during execution. Every API call made during the build (to S3, ECR, SSM, etc.) is made using the credentials from this role.

**Understanding the Permission Layers:**

1. **CodeBuild Service Principal Trust Policy:**
```json
{
  "Effect": "Allow",
  "Principal": { "Service": "codebuild.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

2. **Resource-Based Policies:** S3 bucket policies, ECR repository policies, and KMS key policies must also allow the CodeBuild role.

3. **VPC Permissions:** If using VPC mode, the role needs EC2 network interface permissions.

**Designing a Least-Privilege Role for a Complex Build (e.g., Java app → ECR → S3 artifacts):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:us-east-1:123456789012:log-group:/aws/codebuild/myproject*"
    },
    {
      "Sid": "S3ArtifactsBucket",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:GetBucketAcl", "s3:GetBucketLocation"],
      "Resource": [
        "arn:aws:s3:::my-artifacts-bucket",
        "arn:aws:s3:::my-artifacts-bucket/*"
      ]
    },
    {
      "Sid": "ECRAccess",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SSMParameterStore",
      "Effect": "Allow",
      "Action": ["ssm:GetParameters", "ssm:GetParameter"],
      "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/prod/*"
    },
    {
      "Sid": "KMSForEncryptedParams",
      "Effect": "Allow",
      "Action": ["kms:Decrypt"],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/your-kms-key-id"
    }
  ]
}
```

**Advanced Considerations:**
- Use **IAM permission boundaries** to limit what the CodeBuild role can do even if policies are misconfigured.
- Use **SCPs (Service Control Policies)** at the AWS Organizations level to enforce guardrails.
- Audit with **IAM Access Analyzer** to identify overly permissive policies.
- Use **condition keys** like `aws:SourceAccount` and `aws:SourceArn` in trust policies to prevent confused deputy attacks.
- Rotate credentials by scoping SSM parameters with time-based conditions where possible.

---

### Q2. Explain CodeBuild's batch build feature in depth, including use cases, configuration, and limitations.

**Answer:**
AWS CodeBuild Batch Builds allow you to run multiple related builds as a single unit, with the ability to define dependencies between them. This is ideal for monorepos, matrix testing, and parallel test execution.

**Batch Build Types:**

**1. Build Graph (`build-graph`)**
- Defines a directed acyclic graph (DAG) of builds where some builds depend on others completing first.
- Use case: Build a base Docker image, then in parallel build multiple microservices that depend on it.

```yaml
batch:
  build-graph:
    - identifier: build_base
      env:
        compute-type: BUILD_GENERAL1_SMALL
    - identifier: build_service_a
      depend-on:
        - build_base
    - identifier: build_service_b
      depend-on:
        - build_base
```

**2. Build List (`build-list`)**
- Runs a list of builds in parallel with no dependencies.
- Use case: Run the same build against multiple environments or configurations simultaneously.

```yaml
batch:
  build-list:
    - identifier: linux_build
      env:
        type: LINUX_CONTAINER
        image: aws/codebuild/standard:6.0
    - identifier: arm_build
      env:
        type: ARM_CONTAINER
        image: aws/codebuild/amazonlinux2-aarch64-standard:2.0
```

**3. Build Matrix (`build-matrix`)**
- Runs builds for every combination of environment variables you define.
- Use case: Test a library against multiple language runtime versions.

```yaml
batch:
  build-matrix:
    static:
      env:
        type: LINUX_CONTAINER
    dynamic:
      env:
        variables:
          RUNTIME_VERSION: ["3.8", "3.9", "3.10"]
          FRAMEWORK: ["django", "flask"]
```
This generates 6 builds (3 × 2 combinations).

**Batch Configuration:**
- Must