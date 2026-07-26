# CodePipeline

## What is it?

**AWS CodePipeline** is a fully managed, continuous delivery (CD) service that automates the build, test, and deploy phases of your release process every time there is a code change, based on the release model you define. It falls under the **AWS Developer Tools** category and is a core component of the AWS CI/CD ecosystem.

CodePipeline enables you to model, visualize, and automate the steps required to release your software. A **pipeline** is the workflow construct that describes how software changes go through a release process. Each pipeline is composed of a series of **stages** (e.g., Source, Build, Test, Deploy), and each stage contains one or more **actions** that are performed on the application artifacts.

- **Service Category:** Developer Tools / Continuous Delivery
- **Type:** Fully managed, serverless orchestration service
- **Current Version Context:** Supports both **V1** (JSON-based) and **V2** (YAML-based) pipeline types, with V2 introducing enhanced features like pipeline-level variables, triggers, and Git tags.

---

## Why do we need it?

### The Problem It Solves

Before CI/CD automation, software releases were manual, error-prone, and slow. Teams had to:
- Manually coordinate builds, tests, and deployments across environments.
- Rely on scripts that were inconsistent across team members.
- Struggle with long feedback loops between code commit and production deployment.
- Risk human error during deployments, especially under pressure.

CodePipeline addresses these challenges by providing a **centralized, automated, and auditable** workflow that orchestrates every step of the software delivery lifecycle.

### When to Use It

| Scenario | Why CodePipeline Fits |
|---|---|
| Microservices on ECS/EKS | Automate container builds and blue/green deployments |
| Serverless Lambda Functions | Trigger SAM/CDK deployments on every commit |
| Multi-environment promotion | Automate dev → staging → production promotions with approvals |
| Infrastructure as Code | Automatically apply Terraform/CloudFormation changes |
| Compliance-driven releases | Enforce mandatory test gates before production deployments |

### Real Business Scenarios

1. **E-commerce Platform:** A retail company needs to deploy code changes to their checkout service multiple times per day without downtime. CodePipeline orchestrates automated tests and blue/green deployments to ECS.

2. **Banking Application:** A financial institution requires that every production deployment passes security scanning (SAST/DAST) and has a manual approval from a senior architect. CodePipeline enforces this gate automatically.

3. **SaaS Product:** A startup ships a multi-tenant SaaS application. CodePipeline deploys to a staging environment, runs integration tests, and only promotes to production if all tests pass.

---

## Internal Working

### Core Execution Model

CodePipeline operates as an **event-driven orchestration engine**. Here is how it works internally:

1. **Trigger Detection:** CodePipeline monitors source repositories (CodeCommit, GitHub, S3, ECR) for changes. With V2 pipelines, it uses native webhook/polling mechanisms or EventBridge rules to detect changes.

2. **Pipeline Execution:** When a trigger fires, CodePipeline creates a **pipeline execution** — a unique run of the entire pipeline with a specific artifact set. Each execution is assigned a unique ID.

3. **Stage Execution:** Stages execute **sequentially** by default. A stage begins only after all actions in the previous stage succeed. Within a stage, actions can run **in parallel** (same `runOrder`) or **sequentially** (different `runOrder` values).

4. **Artifact Store:** CodePipeline uses an **Amazon S3 bucket** (the artifact store) to pass data between stages. When a source stage completes, it zips the source code and stores it in S3. Subsequent stages retrieve and potentially transform these artifacts.

5. **Action Workers:** Each action type has an underlying worker:
   - **AWS-owned actions** (CodeBuild, CodeDeploy, CloudFormation) are executed by AWS-managed workers.
   - **Custom actions** use a polling mechanism where an external worker calls `PollForJobs` API.
   - **Third-party actions** (Jenkins, GitHub Actions) integrate via webhooks or polling.

6. **State Machine:** Internally, CodePipeline maintains a state machine for each execution. States include: `InProgress`, `Succeeded`, `Failed`, `Stopped`, `Superseded`.

7. **Superseding:** If a new execution starts while a previous one is still running, the older execution can be **superseded** (cancelled) depending on pipeline configuration. This is configurable in V2 pipelines.

### Artifact Flow

```
Source Stage → [S3 Artifact Store] → Build Stage → [S3 Artifact Store] → Deploy Stage
     ↓                                      ↓                                  ↓
  SourceArtifact                      BuildArtifact                    (Consumed by deployer)
```

### Token-Based Action Coordination

CodePipeline uses **continuation tokens** to manage long-running actions. When an action (like a CodeDeploy deployment) starts, CodePipeline issues a job with a token. The action worker uses this token to report progress back to CodePipeline via `PutJobSuccessResult` or `PutJobFailureResult`.

---

## Architecture

### Pipeline Anatomy

```
┌─────────────────────────────────────────────────────────────────┐
│                         CODEPIPELINE                            │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  STAGE 1     │  │  STAGE 2     │  │  STAGE 3     │         │
│  │  Source      │→ │  Build       │→ │  Deploy      │         │
│  │              │  │              │  │              │         │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │         │
│  │ │ Action 1 │ │  │ │ Action 1 │ │  │ │ Action 1 │ │         │
│  │ │(CodeCommit│ │  │ │(CodeBuild)│ │  │ │(CodeDeploy│ │        │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                 │
│  Artifact Store: ┌─────────────────────────────────────────┐   │
│                  │         Amazon S3 Bucket                 │   │
│                  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

| Component | Description |
|---|---|
| **Pipeline** | Top-level workflow container; defines stages, transitions, and artifact store |
| **Stage** | A logical grouping of actions; stages run sequentially |
| **Action** | The atomic unit of work within a stage (source, build, test, deploy, approval, invoke) |
| **Artifact** | A file or set of files passed between stages, stored in S3 |
| **Transition** | The link between stages; can be enabled/disabled manually |
| **Execution** | A single run of the pipeline triggered by a source change or manual trigger |
| **Trigger** | The event that starts a pipeline execution (push, tag, schedule, manual) |

### Action Categories and Types

```
Action Category    | Action Providers
─────────────────────────────────────────────────────────────
Source             | CodeCommit, GitHub (v1/v2), S3, ECR, Bitbucket
Build              | CodeBuild, Jenkins
Test               | CodeBuild, AWS Device Farm, Ghost Inspector
Deploy             | CodeDeploy, CloudFormation, ECS, EKS, Elastic Beanstalk, S3, AppConfig
Approval           | Manual (SNS notification)
Invoke             | Lambda, Step Functions
```

### Multi-Region Pipeline Architecture

CodePipeline supports **cross-region actions**, allowing you to deploy to resources in different AWS regions from a single pipeline. Each region used requires its own artifact store (S3 bucket).

```
us-east-1 Pipeline
    ├── Source (us-east-1 CodeCommit)
    ├── Build  (us-east-1 CodeBuild)
    ├── Deploy to us-east-1 (ECS)
    └── Deploy to eu-west-1 (ECS) ← Cross-region action
         └── Requires S3 bucket in eu-west-1
```

### Cross-Account Pipeline Architecture

Pipelines can deploy across AWS accounts using cross-account IAM roles, enabling centralized CI/CD management:

```
Tooling Account (Pipeline)
    └── Assumes Role in → Dev Account (Deploy)
    └── Assumes Role in → Staging Account (Deploy)
    └── Assumes Role in → Prod Account (Deploy) + Manual Approval
```

---

## Real World Example

### Scenario: Node.js Microservice Deployment to ECS with Blue/Green

**Context:** A fintech company has a Node.js REST API deployed on Amazon ECS (Fargate). They want to automate the entire release process from code commit to production with a mandatory approval gate.

**Pipeline Stages:**

#### Stage 1: Source
- **Trigger:** Developer pushes to the `main` branch of a CodeCommit repository.
- **Action:** `Source` action pulls the latest code and stores it as `SourceArtifact` in S3.

#### Stage 2: Build
- **Action:** CodeBuild runs `buildspec.yml`:
  ```yaml
  version: 0.2
  phases:
    install:
      runtime-versions:
        nodejs: 18
    pre_build:
      commands:
        - npm ci
        - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI
    build:
      commands:
        - npm test
        - docker build -t $IMAGE_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION .
        - docker push $IMAGE_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
        - printf '[{"name":"api","imageUri":"%s"}]' $IMAGE_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION > imagedefinitions.json
  artifacts:
    files:
      - imagedefinitions.json
      - appspec.yaml
      - taskdef.json
  ```
- **Output:** `BuildArtifact` containing `imagedefinitions.json`, `appspec.yaml`, `taskdef.json`.

#### Stage 3: Deploy to Staging
- **Action:** ECS (Blue/Green) deploy action using CodeDeploy.
- CodeDeploy shifts traffic to the new task definition on the staging cluster.
- Integration tests run automatically via a Lambda invoke action.

#### Stage 4: Manual Approval
- **Action:** Approval action sends an SNS notification to the engineering manager's email.
- Pipeline pauses for up to 7 days waiting for approval.
- Manager reviews staging, then approves or rejects in the CodePipeline console.

#### Stage 5: Deploy to Production
- **Action:** ECS (Blue/Green) deploy action targeting the production ECS cluster.
- CodeDeploy performs a canary deployment: 10% traffic for 5 minutes, then 100%.
- CloudWatch alarms monitor error rates; if triggered, CodeDeploy automatically rolls back.

**Step-by-Step Flow:**
```
1. Developer: git push origin main
2. CodePipeline detects change → creates Execution #42
3. Source stage: Downloads code → stores SourceArtifact in S3
4. Build stage: CodeBuild builds Docker image → pushes to ECR → stores BuildArtifact
5. Staging Deploy: CodeDeploy blue/green to staging ECS cluster
6. Lambda Invoke: Runs integration test suite → returns pass/fail
7. Manual Approval: SNS email sent → manager approves after 2 hours
8. Production Deploy: CodeDeploy canary (10% → 100%) with CloudWatch rollback guard
9. Pipeline Execution #42: SUCCEEDED
```

---

## Advantages

1. **Fully Managed:** No infrastructure to provision or maintain. AWS handles availability, patching, and scaling of the pipeline service itself.

2. **Visual Workflow:** The AWS Console provides a real-time visual representation of pipeline execution, making it easy to identify bottlenecks and failures.

3. **Rich Integration Ecosystem:** Native integrations with 20+ AWS services and support for third-party tools (GitHub, Jenkins, Jira, Slack via Lambda).

4. **Fast Feedback Loop:** Developers get immediate feedback on code quality through automated build and test stages.

5. **Parallel Execution:** Multiple actions within a stage can run in parallel using `runOrder`, reducing total pipeline execution time.

6. **Audit Trail:** Every pipeline execution is logged with start/end times, action results, and artifact versions, supporting compliance requirements.

7. **Cross-Region and Cross-Account:** Supports complex enterprise deployment topologies without requiring multiple separate tools.

8. **Pipeline as Code:** Pipelines can be defined in CloudFormation, CDK, or Terraform, enabling version-controlled infrastructure.

9. **Extensible:** Custom action types and Lambda invoke actions allow integration with any tool or workflow.

10. **Manual Approval Gates:** Built-in support for human-in-the-loop approvals with SNS notifications and configurable timeout periods.

11. **V2 Pipeline Enhancements:** Pipeline-level variables, Git tag triggers, custom trigger filters, and improved execution modes (QUEUED, PARALLEL, SUPERSEDED).

---

## Limitations

### Hard Limits (Default Quotas)

| Limit | Value |
|---|---|
| Pipelines per region per account | 1,000 |
| Stages per pipeline | 10 (minimum 2) |
| Actions per stage | 50 |
| Actions per pipeline | 500 |
| Maximum artifacts per action (input or output) | 5 |
| Maximum artifact size | 3 GB (S3 limit) |
| Pipeline name length | 100 characters |
| Action timeout | 1 hour (default), configurable up to 8 hours |
| Manual approval timeout | 7 days |
| Parallel pipelines per account | 300 concurrent executions |

> Most of these are **soft limits** that can be increased via AWS Support, except where noted.

### Functional Limitations

1. **No Native Rollback Orchestration:** CodePipeline itself does not have a "rollback" button. Rollbacks must be implemented via re-running a previous pipeline execution or using CodeDeploy's rollback features.

2. **Limited Native Scheduling:** V1 pipelines cannot be scheduled (cron-based). V2 pipelines support scheduled triggers via EventBridge integration.

3. **No Built-in Secret Management in Pipeline Definition:** Secrets must be referenced via Parameter Store or Secrets Manager in action configurations.

4. **Sequential Stage Execution:** Stages always execute sequentially; you cannot have parallel stages (only parallel actions within a stage).

5. **Artifact Store Limitations:** The S3 artifact store bucket must be in the same region as the pipeline (except for cross-region actions which need separate buckets).

6. **Custom Action Polling Latency:** Custom actions using the polling model can have up to 30-second latency before a worker picks up the job.

7. **No Native Docker Layer Caching:** CodePipeline itself doesn't cache; caching must be configured in CodeBuild.

8. **Pipeline Execution History:** Console shows the last 500 executions; older executions require CloudTrail or CloudWatch Logs queries.

---

## Best Practices

### Pipeline Design

1. **Use Pipeline as Code:** Define all pipelines in CloudFormation, CDK, or Terraform. Store pipeline definitions in version control alongside application code.

2. **Implement the Promotion Model:** Use separate pipelines or stages for each environment (dev → staging → prod). Never deploy directly to production from a feature branch.

3. **Fail Fast:** Order your stages so that the fastest and most likely-to-fail checks (linting, unit tests) run first. Save expensive integration tests for later stages.

4. **Keep Pipelines Focused:** Create one pipeline per microservice or deployable unit. Avoid monolithic pipelines that deploy multiple unrelated services.

5. **Use Pipeline Variables (V2):** Leverage pipeline-level and stage-level variables to parameterize pipelines, reducing duplication across environments.

### Security (Well-Architected: Security Pillar)

6. **Least Privilege IAM:** The CodePipeline service role should only have permissions required for its specific actions. Avoid