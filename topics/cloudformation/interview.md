# CloudFormation — Interview Questions

---

## Easy

### 1. What is AWS CloudFormation, and what problem does it solve?

**Answer:**
AWS CloudFormation is an Infrastructure as Code (IaC) service that allows you to define and provision AWS infrastructure using declarative template files written in JSON or YAML. It solves the problem of manual, error-prone infrastructure provisioning by allowing teams to version-control, replicate, and automate the creation of AWS resources in a consistent and repeatable manner. Instead of clicking through the AWS Console or writing imperative scripts, you describe the desired end state and CloudFormation handles the provisioning and dependency management.

---

### 2. What are the core components of a CloudFormation template?

**Answer:**
A CloudFormation template has the following sections (only `Resources` is mandatory):

| Section | Required | Purpose |
|---|---|---|
| `AWSTemplateFormatVersion` | No | Declares the template format version |
| `Description` | No | Human-readable description of the template |
| `Parameters` | No | Dynamic inputs passed at stack creation/update time |
| `Mappings` | No | Static key-value lookup tables |
| `Conditions` | No | Logic to conditionally create resources |
| `Transform` | No | Macros (e.g., `AWS::Serverless-2016-10-31` for SAM) |
| `Resources` | **Yes** | Defines the AWS resources to be created |
| `Outputs` | No | Values exported from the stack for cross-stack reference |

---

### 3. What is a CloudFormation Stack?

**Answer:**
A CloudFormation Stack is a collection of AWS resources that are managed as a single unit. When you deploy a CloudFormation template, AWS creates a stack that tracks all the resources defined in that template. You can create, update, and delete all the resources in a stack together. Stacks also maintain the state of deployed resources, enabling CloudFormation to detect and apply only the necessary changes during updates.

---

### 4. What is the difference between `Parameters` and `Mappings` in CloudFormation?

**Answer:**
- **Parameters** are dynamic inputs provided by the user at stack creation or update time. They allow templates to be reusable across environments (e.g., passing a different instance type for dev vs. prod). Parameters support types like `String`, `Number`, `AWS::EC2::KeyPair::KeyName`, etc.
- **Mappings** are static, hardcoded lookup tables embedded in the template. They are used to map keys to values (e.g., mapping a region to an AMI ID). Mappings cannot be changed at runtime without modifying the template itself.

```yaml
Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-0abcdef1234567890
    eu-west-1:
      AMI: ami-0fedcba9876543210
```

---

### 5. What happens when a CloudFormation stack update fails?

**Answer:**
When a stack update fails, CloudFormation automatically performs a **rollback** to the last known stable state. This means any resources that were successfully created or modified during the failed update are reverted. The stack enters a `UPDATE_ROLLBACK_COMPLETE` state. If the rollback itself fails, the stack enters `UPDATE_ROLLBACK_FAILED`, which requires manual intervention (e.g., using `ContinueUpdateRollback` API). You can also configure a **stack policy** to protect specific resources from being replaced or deleted during updates.

---

## Medium

### 1. Explain the difference between CloudFormation Change Sets and direct stack updates. When would you use each?

**Answer:**
**Direct Stack Updates** immediately apply changes to a stack when you submit an updated template or new parameter values. CloudFormation calculates the diff and begins modifying resources right away without giving you a chance to review the impact.

**Change Sets** are a preview mechanism that shows you *what will change* before any changes are applied. A Change Set describes whether resources will be `Add`, `Modify`, or `Remove`, and for modifications, it indicates whether the change will cause resource **interruption** (`No Interruption`, `Some Interruption`, or `Replacement`).

**When to use Change Sets:**
- In production environments where accidental resource replacement (e.g., an RDS instance being deleted and recreated) would cause downtime or data loss.
- During peer review workflows where a second engineer reviews the impact before approval.
- When integrating CloudFormation into CI/CD pipelines — create the change set in one stage, approve it, then execute in the next stage.
- When you want to understand the blast radius of a template change before committing.

**Example workflow:**
```bash
# Create a change set
aws cloudformation create-change-set \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --change-set-name my-change-set

# Review the change set
aws cloudformation describe-change-set \
  --stack-name my-stack \
  --change-set-name my-change-set

# Execute after approval
aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name my-change-set
```

---

### 2. What are CloudFormation Outputs and Cross-Stack References? How do they work?

**Answer:**
**Outputs** are values that a CloudFormation stack exposes after creation. They can display useful information (like a load balancer URL) or be exported for use by other stacks.

**Cross-Stack References** allow one stack (consumer) to use values from another stack (producer) without hardcoding resource ARNs or IDs. This is achieved using `Export` in the producer stack and `Fn::ImportValue` in the consumer stack.

**Producer Stack:**
```yaml
Outputs:
  VpcId:
    Description: The VPC ID
    Value: !Ref MyVPC
    Export:
      Name: !Sub "${AWS::StackName}-VpcId"
```

**Consumer Stack:**
```yaml
Resources:
  MySubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !ImportValue my-network-stack-VpcId
      CidrBlock: 10.0.1.0/24
```

**Important limitations:**
- Export names must be **unique within a region and account**.
- You **cannot delete** a stack whose outputs are being consumed by another stack — you must first delete or update the consumer stack.
- Cross-stack references only work **within the same region and account**. For cross-account/cross-region sharing, use SSM Parameter Store or AWS RAM instead.

---

### 3. What is the `DependsOn` attribute in CloudFormation, and when should you use it explicitly?

**Answer:**
CloudFormation automatically infers dependencies between resources when you use intrinsic functions like `!Ref` or `!GetAtt`. For example, if a security group references a VPC via `!Ref`, CloudFormation knows to create the VPC first.

However, there are cases where dependencies exist logically but are not expressed through direct references. In these cases, you use `DependsOn` to explicitly declare the creation order.

**Common use cases:**
1. **Internet Gateway Attachment before Route Table:** A route to the internet gateway requires the attachment to be complete, but the route resource may not directly reference the attachment resource.
2. **RDS instance waiting for a DB subnet group** when the subnet group is defined separately without a direct `!Ref` link.
3. **Lambda function waiting for an IAM role to fully propagate** before being invoked.

```yaml
Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    DependsOn: MyDBInstance
    Properties:
      ...

  MyDBInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      ...
```

**Caution:** Overusing `DependsOn` can slow down deployments because CloudFormation parallelizes resource creation by default. Only use it when there is a genuine implicit dependency.

---

### 4. Explain CloudFormation Drift Detection. What does it detect and what are its limitations?

**Answer:**
**Drift Detection** is the process of comparing the actual, live configuration of stack resources against the expected configuration defined in the CloudFormation template. When someone manually changes a resource outside of CloudFormation (e.g., modifying a security group rule via the Console), the resource is said to have **drifted**.

**How it works:**
1. You initiate drift detection on a stack or individual resources.
2. CloudFormation calls the relevant AWS service APIs to get the current resource configuration.
3. It compares the live configuration against the template-defined configuration.
4. Resources are marked as `IN_SYNC`, `DRIFTED`, `NOT_CHECKED`, or `DELETED`.

**What it detects:**
- Changes to properties that CloudFormation manages (e.g., security group rules, instance type, S3 bucket policies).
- Resources that have been deleted outside of CloudFormation.

**Limitations:**
- Not all resource types support drift detection (coverage is extensive but not 100%).
- Drift detection is **not real-time** — it is a point-in-time snapshot. You must trigger it manually or via automation.
- It only detects *what* has changed, not *who* changed it (use CloudTrail for that).
- Some properties are intentionally excluded from drift detection (e.g., write-only properties like passwords).
- Drift detection does **not automatically remediate** drift — you must decide whether to update the template or re-sync the resource.

---

### 5. What are CloudFormation Nested Stacks, and what are the trade-offs compared to using a single monolithic template?

**Answer:**
**Nested Stacks** allow you to reference other CloudFormation templates as resources within a parent template using the `AWS::CloudFormation::Stack` resource type. This enables modular, reusable infrastructure components.

```yaml
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/network.yaml
      Parameters:
        VpcCidr: 10.0.0.0/16

  AppStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/app.yaml
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
```

**Advantages of Nested Stacks:**
- **Modularity:** Separate concerns (networking, compute, database) into reusable modules.
- **Bypass template size limits:** A single template is limited to 1 MB (S3) or 51,200 bytes (direct upload). Nested stacks let you exceed this.
- **Team ownership:** Different teams can own and update their respective nested stack templates.
- **Reusability:** The same nested template can be used across multiple parent stacks.

**Trade-offs and disadvantages:**
- **Complexity:** Debugging failures across nested stacks is harder — you must trace errors through multiple stack event logs.
- **Tight coupling:** The parent stack must be updated to propagate changes to nested stacks, making independent deployments harder.
- **Slower deployments:** Nested stacks are created/updated sequentially unless parallelism is explicitly possible.
- **Compared to Cross-Stack References:** Nested stacks are tightly coupled (parent manages child lifecycle), while cross-stack references allow independent stack lifecycles. Choose nested stacks for components that are always deployed together.

---

## Hard

### 1. Deep dive into CloudFormation's resource provisioning model. How does CloudFormation handle resource creation, updates, and deletions internally, and what are the implications for idempotency?

**Answer:**
CloudFormation uses a **desired-state model** backed by a resource provider framework. Internally, the process works as follows:

**Provisioning Flow:**
1. CloudFormation parses the template and builds a **dependency graph** (DAG) of resources.
2. It determines the execution plan: which resources to create, modify, or delete.
3. For each resource, it invokes the corresponding **resource provider** (either AWS-native or a CloudFormation Extension/Custom Resource).
4. Resource providers implement handlers for `CREATE`, `READ`, `UPDATE`, `DELETE`, and `LIST` operations.
5. CloudFormation uses **stabilization polling** — after calling a handler, it polls the resource's status until it reaches a terminal state (`CREATE_COMPLETE`, `CREATE_FAILED`, etc.).

**Update Behaviors:**
CloudFormation categorizes property changes into three update behaviors:
- **No Interruption:** The property can be updated in-place (e.g., changing an EC2 instance's tags).
- **Some Interruption:** The resource is temporarily unavailable (e.g., changing an EC2 instance type requires a stop/start).
- **Replacement:** A new resource is created, references are updated, and the old resource is deleted. This is the most dangerous behavior (e.g., changing an RDS instance's `DBInstanceIdentifier`). CloudFormation creates the replacement first, then deletes the original — but the old resource gets a temporary logical ID suffix during this period.

**Idempotency Implications:**
CloudFormation achieves idempotency through **stack state tracking** in DynamoDB (internal). Each resource has a physical resource ID. If a `CREATE` operation partially succeeds and then fails, CloudFormation knows which resources were already created and will attempt to roll them back rather than create duplicates. However, true idempotency depends on the underlying resource provider — some AWS APIs are idempotent (e.g., S3 `CreateBucket`) while others are not. For custom resources, developers must implement their own idempotency logic using the `RequestId` passed by CloudFormation.

**Token-based Asynchronous Operations:**
For long-running operations, resource providers return a **stabilization token**. CloudFormation stores this token and polls with `GetResource` calls until the operation completes. This is why CloudFormation can handle operations like creating an Aurora cluster (which takes several minutes) without timing out.

---

### 2. Explain CloudFormation StackSets in depth. How does it handle deployment across multiple accounts and regions, and what are the failure tolerance mechanisms?

**Answer:**
**CloudFormation StackSets** extend the concept of stacks to allow deployment of a single template across multiple AWS accounts and regions from a single administrator account (or using AWS Organizations-based trusted access).

**Architecture:**
```
Administrator Account
└── StackSet (template + deployment targets)
    ├── Stack Instance → Account A, us-east-1
    ├── Stack Instance → Account A, eu-west-1
    ├── Stack Instance → Account B, us-east-1
    └── Stack Instance → Account B, ap-southeast-1
```

**Deployment Models:**
1. **Self-managed permissions:** You manually create `AWSCloudFormationStackSetAdministrationRole` in the admin account and `AWSCloudFormationStackSetExecutionRole` in each target account. The admin role assumes the execution role to deploy.
2. **Service-managed permissions (Organizations):** AWS manages the roles automatically. You can target entire OUs, and new accounts added to an OU can be auto-enrolled via **automatic deployment**.

**Failure Tolerance Mechanisms:**
- **MaxConcurrentCount / MaxConcurrentPercentage:** Controls how many accounts/regions are deployed to simultaneously. Lower values reduce blast radius.
- **FailureToleranceCount / FailureTolerancePercentage:** Defines how many stack instance failures are acceptable before the entire operation is stopped. E.g., `FailureTolerancePercentage: 20` means up to 20% of accounts can fail before the operation halts.
- **RegionConcurrencyType:** `SEQUENTIAL` (one region at a time, safer) or `PARALLEL` (all regions simultaneously, faster).
- **RetainStacksOnAccountRemoval:** When an account is removed from an OU, you can choose to retain the deployed stacks rather than delete them.

**Drift Detection on StackSets:**
StackSets support drift detection across all stack instances. Each stack instance is checked independently, and results are aggregated at the StackSet level.

**Limitations:**
- StackSets have a default limit of 2,000 stack instances per StackSet.
- Operations on StackSets are serialized — you cannot run two operations on the same StackSet simultaneously.
- Stack instances in a StackSet cannot be updated independently; all updates flow through the StackSet.
- Rollback behavior in StackSets is complex — a failed stack instance does not automatically roll back other successfully deployed instances.

---

### 3. How do CloudFormation Custom Resources work? Describe the complete lifecycle, error handling, and best practices for production-grade implementations.

**Answer:**
**Custom Resources** allow you to extend CloudFormation to manage resources that are not natively supported, execute arbitrary logic during stack operations, or integrate with third-party services. They are implemented using AWS Lambda or SNS.

**Complete Lifecycle:**

```
CloudFormation → Sends HTTPS POST (JSON) to ServiceToken (Lambda/SNS)
                 ↓
             Custom Resource Handler
                 ↓
             