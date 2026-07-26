# CodePipeline — Interview Questions

---

## Easy

### Q1. What is AWS CodePipeline and what problem does it solve?

**Answer:**
AWS CodePipeline is a fully managed continuous delivery (CD) service that automates the build, test, and deploy phases of your release process every time there is a code change. It solves the problem of manual, error-prone software release processes by orchestrating the steps required to release software changes continuously. It integrates with AWS services like CodeBuild, CodeDeploy, Lambda, and also third-party tools like GitHub, Jenkins, and Jira.

---

### Q2. What are the core components of a CodePipeline pipeline?

**Answer:**
A CodePipeline pipeline consists of the following core components:

- **Pipeline**: The top-level construct that defines the overall workflow.
- **Stages**: Logical divisions of the pipeline (e.g., Source, Build, Test, Deploy). A pipeline must have at least two stages.
- **Actions**: Individual tasks within a stage (e.g., pulling source code, running a build, deploying to an environment). Actions can run sequentially or in parallel.
- **Transitions**: The connections between stages that control the flow of artifacts. Transitions can be enabled or disabled manually.
- **Artifacts**: Files or data passed between stages, stored in an S3 bucket (the artifact store).

---

### Q3. What is an artifact store in CodePipeline?

**Answer:**
An artifact store is an Amazon S3 bucket used by CodePipeline to store input and output artifacts for pipeline actions. When a stage produces output (e.g., a compiled binary from CodeBuild), it is stored as a ZIP file in this S3 bucket. The next stage then retrieves the artifact from this bucket as its input. By default, CodePipeline creates and manages this bucket automatically, but you can specify a custom S3 bucket. The bucket must be in the same AWS region as the pipeline. For cross-region pipelines, each region requires its own artifact store.

---

### Q4. What source providers does CodePipeline support?

**Answer:**
CodePipeline supports the following source providers:

- **AWS CodeCommit** — Git-based repository hosted on AWS
- **Amazon S3** — Trigger pipeline from object changes in a bucket
- **Amazon ECR** — Trigger pipeline from container image changes
- **GitHub (via GitHub App or OAuth)** — Versions 1 and 2 connections
- **GitHub Enterprise Server** — Self-hosted GitHub
- **Bitbucket Cloud** — Via AWS CodeStar Connections
- **GitLab.com and GitLab Self-Managed** — Via AWS CodeStar Connections

---

### Q5. How does CodePipeline detect changes in source repositories?

**Answer:**
CodePipeline uses two primary mechanisms to detect source changes:

1. **Amazon CloudWatch Events (EventBridge)** — The recommended method. An EventBridge rule is created that monitors the source repository (e.g., CodeCommit push, ECR image push) and automatically triggers the pipeline. This is near real-time and the default behavior for CodeCommit and ECR sources.

2. **Periodic Polling** — CodePipeline polls the source repository at regular intervals (every minute). This is used for some S3 and older GitHub (OAuth) integrations. It is less efficient than event-based detection and is generally discouraged in favor of EventBridge.

---

## Medium

### Q1. Explain the difference between sequential and parallel actions in CodePipeline, and when you would use each.

**Answer:**
Within a single stage in CodePipeline, actions can be configured to run either **sequentially** or **in parallel**, controlled by the `runOrder` property.

- **Sequential Actions (`runOrder: 1, 2, 3...`)**: Actions with different `runOrder` values execute one after another. The next action only starts when the previous one completes successfully. Use this when there is a dependency between actions — for example, a security scan must pass before a deployment begins.

- **Parallel Actions (`runOrder: 1, 1, 1...`)**: Actions with the same `runOrder` value execute simultaneously. Use this to optimize pipeline execution time — for example, running unit tests, integration tests, and a static code analysis scan at the same time in a Test stage.

**Key constraint**: Parallel actions must be within the same stage. You cannot run actions from different stages in parallel — stages always execute sequentially.

**Example use case**: In a Deploy stage, you might deploy to `us-east-1` and `eu-west-1` simultaneously using two parallel CodeDeploy actions with `runOrder: 1`, then run a smoke test action with `runOrder: 2` after both deployments complete.

---

### Q2. What is the role of IAM in CodePipeline, and what are the key IAM entities involved?

**Answer:**
IAM plays a critical role in CodePipeline security and authorization. The key IAM entities are:

1. **CodePipeline Service Role**: An IAM role assumed by the CodePipeline service itself. It needs permissions to:
   - Read/write to the artifact store S3 bucket
   - Invoke CodeBuild projects
   - Trigger CodeDeploy deployments
   - Invoke Lambda functions
   - Interact with CodeCommit, ECR, CloudFormation, etc.
   - Pass roles to downstream services (`iam:PassRole`)

2. **Action-Specific Roles**: Some actions (like CloudFormation Deploy) require a separate IAM role that CodePipeline "passes" to the service. For example, a CloudFormation action needs a role with permissions to create/update AWS resources.

3. **Cross-Account Roles**: For cross-account pipelines, you need a role in the target account that the pipeline's service role can assume via `sts:AssumeRole`.

4. **Customer-Managed CMK**: If using a custom KMS key for artifact encryption, the service role must have `kms:GenerateDataKey` and `kms:Decrypt` permissions.

**Best practice**: Apply least-privilege principles. Scope S3 permissions to the specific artifact bucket and prefix, not `*`.

---

### Q3. How do manual approval actions work in CodePipeline, and what are the configuration options?

**Answer:**
A **Manual Approval action** is a built-in action type (`ActionTypeId` category: `Approval`, provider: `Manual`) that pauses pipeline execution and waits for a human to approve or reject before proceeding.

**Configuration options:**
- **SNS Topic ARN**: When specified, CodePipeline publishes a notification to the SNS topic when the approval is needed. Subscribers (email, Lambda, etc.) receive the notification.
- **Approval URL**: A URL included in the notification — typically a link to a staging environment, test report, or change management ticket.
- **Comments**: A custom message providing context to the approver.

**Approval process:**
1. Pipeline reaches the approval action and pauses.
2. SNS notification is sent (if configured).
3. An authorized IAM user/role navigates to the CodePipeline console (or uses the CLI/SDK) to approve or reject.
4. If **approved**, the pipeline continues to the next stage.
5. If **rejected**, the pipeline stops and the stage is marked as failed.

**Timeout**: By default, manual approval actions expire after **7 days** if no action is taken, causing the pipeline to fail.

**IAM permission required**: The approver needs `codepipeline:PutApprovalResult` permission.

---

### Q4. Describe how cross-region actions work in CodePipeline.

**Answer:**
CodePipeline supports deploying to multiple AWS regions from a single pipeline using **cross-region actions**.

**How it works:**
1. Each target region requires its own **artifact store** (S3 bucket) in that region. These are defined in the pipeline's `artifactStores` map (plural), as opposed to `artifactStore` (singular) for single-region pipelines.
2. When a cross-region action is executed, CodePipeline automatically copies the required input artifacts from the primary region's artifact store to the target region's artifact store.
3. The action then executes in the target region (e.g., a CodeDeploy deployment in `ap-southeast-1`).

**Configuration requirements:**
- Define `artifactStores` with a bucket for each region used.
- Set the `region` property on the action to the target region.
- The pipeline's service role must have permissions to write to S3 buckets in all target regions.
- If using customer-managed KMS keys, each region needs its own CMK.

**Use case**: A global SaaS application that needs to deploy the same application version to `us-east-1`, `eu-west-1`, and `ap-southeast-1` simultaneously, with the same pipeline managing all deployments.

---

### Q5. What are CodePipeline pipeline variables and how are they used?

**Answer:**
Pipeline variables allow you to pass dynamic values between pipeline actions and stages, enabling more flexible and parameterized pipelines.

**Types of variables:**

1. **Pipeline-level variables**: Defined at the pipeline level with a default value. They can be overridden when starting a pipeline execution via the `StartPipelineExecution` API or CLI. Example: `#{variables.Environment}` could be `staging` or `production`.

2. **Action output variables**: Many actions produce output variables that can be consumed by downstream actions. Examples:
   - **CodeCommit source action**: `#{SourceVariables.CommitId}`, `#{SourceVariables.BranchName}`
   - **CodeBuild action**: Variables exported from the build using `exported-variables` in `buildspec.yml`
   - **GitHub source**: `#{SourceVariables.CommitMessage}`, `#{SourceVariables.AuthorDate}`
   - **CloudFormation**: Output values from stacks

**Variable syntax**: `#{namespace.variableName}` — each action that produces variables has a namespace (e.g., `SourceVariables`, `BuildVariables`).

**Use case example**: Pass the Git commit ID from the source action to a CodeBuild action as an environment variable to tag Docker images with the commit hash, then pass the image tag to a CloudFormation deploy action to update an ECS task definition.

---

## Hard

### Q1. Explain how you would design a multi-account, multi-region CodePipeline architecture with proper security controls.

**Answer:**
A robust multi-account, multi-region CodePipeline setup typically follows AWS's recommended multi-account strategy (e.g., using AWS Organizations with separate accounts for tooling, staging, and production).

**Architecture:**

```
Tooling Account (us-east-1)
  └── CodePipeline (pipeline definition, artifact store S3)
        ├── Stage: Source (CodeCommit in tooling account)
        ├── Stage: Build (CodeBuild in tooling account)
        ├── Stage: Deploy-Staging (cross-account to Staging account, us-east-1)
        ├── Stage: Approval (manual)
        └── Stage: Deploy-Prod (cross-account to Prod account, multi-region)
```

**Key security controls:**

1. **Cross-Account Role Assumption**:
   - In each target account (staging, prod), create an IAM role (e.g., `CodePipelineCrossAccountRole`) with a trust policy allowing the tooling account's pipeline service role to assume it.
   - The cross-account role has permissions to execute deployments (CodeDeploy, CloudFormation, ECS, etc.) in that account.
   - The pipeline service role needs `sts:AssumeRole` permission for each cross-account role.

2. **KMS Encryption**:
   - Use a customer-managed KMS key (CMK) in the tooling account for artifact encryption.
   - The CMK key policy must allow the cross-account roles in staging and production accounts to use the key for `kms:Decrypt` and `kms:GenerateDataKey`.
   - This is critical because artifacts stored in S3 need to be decrypted by the target account's role.

3. **S3 Artifact Store Policy**:
   - The artifact store bucket policy must allow cross-account access from the staging and production account role ARNs.

4. **Cross-Region Artifact Stores**:
   - For multi-region deployments, provision S3 buckets in each target region (in the tooling account or target accounts), with appropriate bucket policies and KMS keys.

5. **SCPs and Permission Boundaries**:
   - Use Service Control Policies at the Organization level to prevent production account roles from being used outside of approved deployment patterns.
   - Apply permission boundaries to the cross-account roles to limit blast radius.

6. **Audit and Compliance**:
   - Enable CloudTrail in all accounts to capture `StartPipelineExecution`, `PutApprovalResult`, and other API calls.
   - Use AWS Config rules to detect pipeline configuration drift.
   - Send pipeline state change events to a centralized EventBridge event bus for monitoring.

---

### Q2. How does CodePipeline handle pipeline execution modes, and what are the implications of each mode for high-frequency deployments?

**Answer:**
CodePipeline supports three **execution modes** that control how concurrent pipeline executions are handled:

**1. SUPERSEDED (Default)**
- If a new execution is triggered while a previous one is in progress, the in-progress execution continues, but any executions waiting in queue are **superseded** (dropped) by the newer one.
- Only one execution can be "waiting" at any time — the most recent one.
- **Implication**: In high-frequency commit environments, intermediate commits may never be deployed. Only the latest queued commit will eventually deploy after the current one finishes.

**2. QUEUED**
- All executions are queued and processed one at a time, in order.
- No execution is dropped — every commit triggers a deployment.
- **Implication**: In high-frequency environments, a large backlog can build up. This is appropriate for environments requiring every commit to be deployed (e.g., audit requirements), but can cause significant delays.

**3. PARALLEL**
- Multiple executions can run simultaneously without waiting for each other.
- Each execution is independent.
- **Implication**: Risk of race conditions — a slower execution for an older commit could deploy after a faster execution for a newer commit, resulting in a rollback to an older version. Requires careful design (e.g., deployment strategies that handle ordering, or using execution IDs to validate ordering before deploying).

**Choosing the right mode:**
- **SUPERSEDED**: Best for most CI/CD scenarios where you only care about the latest state.
- **QUEUED**: Best for compliance-driven environments or when every artifact must be validated and deployed.
- **PARALLEL**: Best for feature branch deployments to isolated environments or when executions are truly independent.

**Advanced consideration**: For PARALLEL mode, implement a Lambda function in a pre-deployment stage that checks the current deployed version tag against the incoming execution's version and aborts if the incoming version is older, preventing accidental rollbacks.

---

### Q3. Deep dive into CodePipeline's integration with CloudFormation. What action modes are available and how do you manage stack drift and rollbacks?

**Answer:**
CodePipeline's CloudFormation integration is one of its most powerful features for infrastructure-as-code deployments. The CloudFormation action provider supports **eight action modes**:

**Stack Action Modes:**
1. **CREATE_UPDATE**: Creates a new stack or updates an existing one. Most commonly used.
2. **DELETE_ONLY**: Deletes an existing stack.
3. **REPLACE_ON_FAILURE**: If stack creation fails, deletes and recreates it. Useful for development environments.

**Change Set Action Modes:**
4. **CHANGE_SET_REPLACE**: Creates or replaces a change set without executing it. Used for previewing changes.
5. **CHANGE_SET_EXECUTE**: Executes a previously created change set.

**Stack Set Action Modes (for AWS Organizations):**
6. **CHANGE_SET_REPLACE** (StackSet)
7. **CHANGE_SET_EXECUTE** (StackSet)

**Best practice pattern — Change Set Review:**
```
Stage: Deploy-Staging
  Action 1: CHANGE_SET_REPLACE (creates change set, runOrder: 1)
  Action 2: CHANGE_SET_EXECUTE (executes change set, runOrder: 2)

Stage: Approval
  Action: Manual approval with link to CloudFormation change set console

Stage: Deploy-Prod
  Action 1: CHANGE_SET_REPLACE (runOrder: 1)
  Action 2: Manual Approval (runOrder: 2) — review change set
  Action 3: CHANGE_SET_EXECUTE (runOrder: 3)
```

**Handling Drift:**
- CloudFormation drift detection is not natively integrated into CodePipeline actions.
- Solution: Add a Lambda action before the deployment that calls `DetectStackDrift` and `DescribeStackDriftDetectionStatus`, then fails the pipeline if drift is detected, forcing a review before overwriting manual changes.

**Handling Rollbacks:**
- CloudFormation automatically rolls back on stack update failure (configurable).
- For CodePipeline, a failed CloudFormation action marks the stage as failed and stops the pipeline.
- Implement a Lambda action in