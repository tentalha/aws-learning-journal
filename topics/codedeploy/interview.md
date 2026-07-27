# CodeDeploy — Interview Questions

---

## Easy

### Q1. What is AWS CodeDeploy and what problem does it solve?

**Answer:**
AWS CodeDeploy is a fully managed deployment service that automates software deployments to a variety of compute services, including Amazon EC2 instances, AWS Lambda functions, Amazon ECS services, and on-premises servers. It solves the problem of manual, error-prone deployments by providing a consistent, repeatable deployment process that minimizes downtime, reduces human error, and enables rapid release of new features. It handles the complexity of updating applications across fleets of instances while maintaining availability.

---

### Q2. What are the three deployment compute platforms supported by CodeDeploy?

**Answer:**
CodeDeploy supports three deployment compute platforms:

1. **EC2/On-Premises** — Deploys application content to EC2 instances or on-premises servers. Requires the CodeDeploy agent to be installed on each target instance.
2. **AWS Lambda** — Deploys updates to Lambda function versions by shifting traffic between an old and new version using aliases.
3. **Amazon ECS** — Deploys containerized applications to ECS services, supporting blue/green deployments by shifting traffic between task sets.

---

### Q3. What is an AppSpec file and what is its purpose?

**Answer:**
An AppSpec (Application Specification) file is a YAML or JSON configuration file that CodeDeploy uses to manage a deployment. It defines:

- **Files** (for EC2/On-Premises): Which source files should be copied to which destination on the target instance.
- **Hooks**: Lifecycle event hooks that specify scripts to run at different stages of the deployment (e.g., `BeforeInstall`, `AfterInstall`, `ApplicationStart`, `ValidateService`).
- **Resources** (for Lambda/ECS): Specifies the Lambda function or ECS task definition and container configuration.

The AppSpec file must be placed at the root of the application revision and is named `appspec.yml` (or `appspec.yaml`).

---

### Q4. What is the difference between an in-place deployment and a blue/green deployment in CodeDeploy?

**Answer:**

| Feature | In-Place Deployment | Blue/Green Deployment |
|---|---|---|
| **How it works** | The application is stopped, updated, and restarted on existing instances | A new set of instances (green) is provisioned, traffic is shifted from old (blue) to new (green) |
| **Downtime risk** | Higher; instances are taken offline during update | Minimal; traffic shifts only after green is healthy |
| **Rollback speed** | Slower; requires redeployment | Fast; redirect traffic back to blue environment |
| **Cost** | Lower; reuses existing instances | Higher; temporarily runs two environments |
| **Platform support** | EC2/On-Premises | EC2, Lambda, ECS |

---

### Q5. What is the CodeDeploy agent and when is it required?

**Answer:**
The CodeDeploy agent is a software package that runs on EC2 instances or on-premises servers. It is responsible for:

- Polling the CodeDeploy service for deployment instructions.
- Downloading the application revision from Amazon S3 or GitHub.
- Executing the deployment lifecycle event hooks defined in the AppSpec file.
- Reporting deployment status back to CodeDeploy.

The agent is **required** for **EC2/On-Premises** deployments. It is **not required** for Lambda or ECS deployments, as those platforms are managed directly by AWS APIs without a local agent. The agent can be installed manually, via AWS Systems Manager, or via EC2 user data scripts.

---

## Medium

### Q1. Explain the CodeDeploy deployment lifecycle events for an EC2/On-Premises deployment. What order do they run in?

**Answer:**
CodeDeploy defines a series of lifecycle events that execute in a specific order during an EC2/On-Premises deployment. These events allow you to hook scripts at critical points in the deployment process:

**In-Place Deployment Order:**

```
ApplicationStop
  ↓
DownloadBundle
  ↓
BeforeInstall
  ↓
Install
  ↓
AfterInstall
  ↓
ApplicationStart
  ↓
ValidateService
```

**Additional events for blue/green deployments:**
- `BeforeBlockTraffic` / `BlockTraffic` / `AfterBlockTraffic` — Run on the original (blue) instances before traffic is blocked.
- `BeforeAllowTraffic` / `AllowTraffic` / `AfterAllowTraffic` — Run on the replacement (green) instances before/after traffic is allowed.

**Key points:**
- `DownloadBundle` and `Install` are reserved by CodeDeploy and **cannot** have custom scripts attached.
- `ApplicationStop` runs using the AppSpec from the **previous** successful deployment, not the current one.
- If any hook script exits with a non-zero exit code, the deployment is marked as failed and CodeDeploy can trigger a rollback.
- Scripts have a default timeout of 30 minutes (configurable up to 3600 seconds).

---

### Q2. What are CodeDeploy deployment configurations, and what built-in configurations are available?

**Answer:**
A deployment configuration defines how CodeDeploy routes and deploys traffic during a deployment. It specifies the minimum number of healthy instances (for EC2/On-Premises) or the traffic shifting behavior (for Lambda/ECS).

**Built-in EC2/On-Premises configurations:**

| Configuration | Description |
|---|---|
| `CodeDeployDefault.AllAtOnce` | Deploys to all instances simultaneously. Fastest but highest risk. |
| `CodeDeployDefault.HalfAtATime` | Deploys to at most half the instances at a time. |
| `CodeDeployDefault.OneAtATime` | Deploys to one instance at a time. Slowest but safest. |

**Built-in Lambda/ECS configurations:**

| Configuration | Description |
|---|---|
| `CodeDeployDefault.LambdaAllAtOnce` | Shifts 100% of traffic immediately. |
| `CodeDeployDefault.LambdaLinear10PercentEvery1Minute` | Shifts 10% every minute until 100%. |
| `CodeDeployDefault.LambdaLinear10PercentEvery3Minutes` | Shifts 10% every 3 minutes. |
| `CodeDeployDefault.LambdaCanary10Percent5Minutes` | Shifts 10% first, then 90% after 5 minutes. |
| `CodeDeployDefault.LambdaCanary10Percent30Minutes` | Shifts 10% first, then 90% after 30 minutes. |

**Custom configurations** can be created to specify exact minimum healthy host counts or percentages. This is valuable when you need fine-grained control over deployment velocity.

---

### Q3. How does CodeDeploy integrate with Auto Scaling groups? What happens when new instances are launched during a deployment?

**Answer:**
CodeDeploy integrates with Auto Scaling groups through a feature called **deployment group association**. When you associate a CodeDeploy deployment group with an Auto Scaling group:

**Normal operation:**
- When a new EC2 instance launches (scale-out event), CodeDeploy automatically detects it via an Auto Scaling lifecycle hook.
- CodeDeploy deploys the **last successful deployment** to the new instance before it is put into service.
- This ensures all instances in the fleet always run the same application version.

**During an active deployment:**
If Auto Scaling launches a new instance while a deployment is in progress, CodeDeploy handles it as follows:
- The new instance receives the **in-progress deployment** (the current one being deployed).
- This can sometimes cause issues if the deployment is failing, as new instances may also fail.
- CodeDeploy uses a **pending** status for such instances and retries the deployment.

**Important considerations:**
- You must ensure the CodeDeploy agent is installed on new instances, typically via an EC2 launch template user data script or a golden AMI.
- For blue/green deployments with Auto Scaling, CodeDeploy creates a **new Auto Scaling group** for the replacement (green) environment.
- The original Auto Scaling group (blue) is kept for a configurable retention period before termination.
- Scale-in events during deployment can cause failures if the instance being terminated was part of the active deployment.

---

### Q4. Explain how CodeDeploy rollbacks work. What are the different rollback triggers?

**Answer:**
CodeDeploy supports automatic and manual rollbacks to restore a previous application version when a deployment fails or degrades service quality.

**How rollbacks work mechanically:**
A rollback is not an "undo" operation. Instead, CodeDeploy creates a **new deployment** using the last known good revision. This means the rollback itself goes through the full deployment lifecycle, including all hooks.

**Automatic rollback triggers:**

1. **Deployment failure** — If a deployment fails (e.g., a lifecycle hook script exits with a non-zero code, or the minimum healthy host threshold is not met), CodeDeploy automatically rolls back if configured to do so.

2. **Alarm-based rollback** — You can configure CloudWatch Alarms in the deployment group. If a specified alarm enters the `ALARM` state during or after a deployment (within a monitoring window), CodeDeploy triggers an automatic rollback. This is powerful for catching post-deployment regressions like elevated error rates or latency.

**Manual rollback:**
- Navigate to the deployment in the CodeDeploy console and choose "Stop and Roll Back Deployment."
- Alternatively, use the AWS CLI: `aws deploy stop-deployment --deployment-id <id> --auto-rollback-enabled`

**Rollback configuration options in deployment group:**
```yaml
autoRollbackConfiguration:
  enabled: true
  events:
    - DEPLOYMENT_FAILURE
    - DEPLOYMENT_STOP_ON_ALARM
    - DEPLOYMENT_STOP_ON_REQUEST
```

**Limitation:** If the rollback itself fails, CodeDeploy does not attempt further automatic rollbacks. Manual intervention is required.

---

### Q5. What is the difference between a CodeDeploy deployment group and a deployment configuration? How do they relate to each other?

**Answer:**

**Deployment Group:**
A deployment group is a logical grouping of deployment targets (EC2 instances, Lambda functions, ECS services, or on-premises servers). It defines **where** to deploy and **how** to manage those targets. Key properties include:
- Target instances (via EC2 tags, Auto Scaling group names, or on-premises instance tags)
- IAM service role for CodeDeploy
- Deployment type (in-place vs. blue/green)
- Load balancer configuration
- Rollback settings
- CloudWatch alarm configuration
- Trigger notifications (SNS)

**Deployment Configuration:**
A deployment configuration defines **how fast** and **how safely** to deploy across the targets. It specifies:
- The minimum number or percentage of healthy instances during deployment
- Traffic shifting strategy (for Lambda/ECS)

**Relationship:**
When you create a deployment, you specify:
1. An **Application** (logical grouping)
2. A **Deployment Group** (where to deploy + target management rules)
3. A **Deployment Configuration** (speed/safety of rollout)
4. A **Revision** (what to deploy — S3 location or GitHub reference)

Think of it this way:
- Deployment Group = "Deploy to production EC2 instances tagged `env:prod` behind this ALB"
- Deployment Configuration = "Deploy to only 25% of instances at a time, requiring 75% healthy"

A single deployment group can use different deployment configurations for different deployments, giving you flexibility to deploy slowly during business hours and faster during maintenance windows.

---

## Hard

### Q1. Deep dive into CodeDeploy's blue/green deployment mechanism for ECS. How does traffic shifting work at the load balancer level?

**Answer:**
CodeDeploy's blue/green deployment for ECS is tightly integrated with Application Load Balancer (ALB) or Network Load Balancer (NLB) listener rules and target groups. Understanding the mechanics requires knowledge of how ECS task sets and ALB weighted routing work together.

**Architecture components:**
- **Blue task set**: The currently running ECS tasks serving production traffic.
- **Green task set**: The newly deployed ECS tasks with the updated container image/configuration.
- **Production listener**: The ALB listener (e.g., port 443) serving live traffic.
- **Test listener** (optional): A secondary ALB listener (e.g., port 8080) used to test the green environment before traffic shift.
- **Two target groups**: One for blue tasks, one for green tasks.

**Deployment flow:**

```
1. CodeDeploy creates a new ECS task set (green) using the new task definition
2. Green tasks register with the green target group
3. CodeDeploy runs BeforeAllowTraffic hook (Lambda function)
4. If a test listener is configured, 100% of test listener traffic routes to green
5. Traffic shifting begins per deployment configuration:
   - AllAtOnce: 100% immediately to green
   - Canary: e.g., 10% to green, wait, then 100%
   - Linear: e.g., 10% every minute
6. ALB listener rule weights are adjusted: blue weight decreases, green weight increases
7. AfterAllowTraffic hook runs (Lambda function)
8. After the termination wait time, blue task set is terminated
```

**AppSpec file for ECS:**
```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:us-east-1:123456789:task-definition/myapp:5"
        LoadBalancerInfo:
          ContainerName: "myapp-container"
          ContainerPort: 8080
        PlatformVersion: "LATEST"
Hooks:
  - BeforeAllowTraffic: "arn:aws:lambda:us-east-1:123456789:function:pre-traffic-check"
  - AfterAllowTraffic: "arn:aws:lambda:us-east-1:123456789:function:post-traffic-check"
```

**Critical nuances:**
- The ECS service must be created with `deploymentController: CODE_DEPLOY` — this cannot be changed after service creation.
- During the termination wait period (configurable 0–2880 minutes), both task sets run simultaneously, doubling compute costs.
- If a CloudWatch alarm fires during traffic shifting, CodeDeploy halts the shift and rolls back by redirecting 100% traffic to the blue target group.
- The `BeforeAllowTraffic` and `AfterAllowTraffic` hooks must be Lambda functions (not shell scripts), and they must call back to CodeDeploy using `PutLifecycleEventHookExecutionStatus` API within the timeout period.
- Connection draining on the blue target group ensures in-flight requests complete before the blue task set is terminated.

---

### Q2. Explain how CodeDeploy handles deployments to large fleets (thousands of instances). What are the performance implications and how would you optimize for speed and safety?

**Answer:**
Deploying to large fleets introduces challenges around parallelism, failure handling, and deployment velocity. CodeDeploy has specific behaviors and limitations that must be understood at scale.

**How CodeDeploy handles large fleets:**

**Agent polling model:**
- CodeDeploy agents on instances poll the CodeDeploy service endpoint every 15 seconds (configurable).
- At scale, this creates significant API call volume. AWS throttles CodeDeploy API calls, and you may need to request limit increases.
- Each agent downloads the revision independently from S3, creating potential S3 bandwidth bottlenecks.

**Deployment batching:**
- CodeDeploy processes instances in batches based on the deployment configuration.
- For `OneAtATime` on 10,000 instances, this would take an extremely long time.
- For large fleets, custom deployment configurations with higher percentages (e.g., 25% at a time) are more practical.

**Optimization strategies:**

1. **Use a custom deployment configuration:**
```
Minimum healthy hosts: 75% (deploy to 25% at a time)
This balances speed with safety for a 10,000-instance fleet:
- Batch 1: 2,500 instances simultaneously
- If successful, Batch 2: 2,500 instances
- Total: ~4 batches instead of 10,000 sequential deployments
```

2. **Pre-bake AMIs (Golden AMIs):**
   - Include the application in the AMI to reduce deployment time to configuration changes only.
   - Reduces the `Install` phase significantly.

3. **S3 Transfer Acceleration or regional S3 buckets:**
   - Store revision artifacts in the same region as your instances.
   - Use S3 Transfer Acceleration for cross-region deployments.
   - Enable S3 bucket versioning to avoid race conditions.

4. **Minimize