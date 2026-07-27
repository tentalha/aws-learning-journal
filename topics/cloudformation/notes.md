# CloudFormation

## What is it?

**AWS CloudFormation** is a fully managed **Infrastructure as Code (IaC)** service that enables you to model, provision, and manage AWS and third-party resources by declaring them in template files. It falls under the **Management & Governance** category of AWS services.

CloudFormation allows you to define your entire infrastructure — from VPCs and EC2 instances to RDS databases and IAM roles — using either **JSON** or **YAML** template files. These templates describe the desired state of your infrastructure, and CloudFormation handles the orchestration of API calls, dependency resolution, and lifecycle management to achieve that state.

**Key identifiers:**
- **Service Name:** AWS CloudFormation
- **Category:** Management & Governance / Infrastructure as Code
- **Core Concept:** Declarative infrastructure provisioning via templates → stacks
- **Supported Template Formats:** JSON, YAML
- **Current Template Version:** `2010-09-09` (the only valid value for `AWSTemplateFormatVersion`)

---

## Why do we need it?

### The Problem Without CloudFormation

Without IaC tooling, infrastructure management suffers from:

- **Manual drift:** Clicking through the AWS Console creates undocumented, unreproducible environments.
- **Snowflake servers:** Each environment (dev, staging, prod) diverges over time due to ad-hoc changes.
- **Slow provisioning:** Manually creating 50+ resources for a new microservice takes hours or days.
- **No audit trail:** There's no version history of infrastructure changes.
- **Human error:** Misconfigured security groups, missing IAM permissions, wrong instance types.
- **Disaster recovery gaps:** Rebuilding infrastructure from scratch after a failure is extremely difficult.

### When to Use CloudFormation

| Scenario | Why CloudFormation Helps |
|---|---|
| Multi-environment deployments (dev/staging/prod) | Parameterized templates ensure consistency |
| Disaster recovery / region failover | Recreate entire infrastructure in minutes |
| Compliance-driven infrastructure | Enforced, auditable, version-controlled configs |
| Microservices with many AWS resources | Manage complex dependency graphs automatically |
| Team-based infrastructure development | Treat infra like code — PRs, reviews, version control |
| SaaS multi-tenancy | Deploy isolated stacks per customer |

### Real Business Scenarios

1. **FinTech Startup:** A payment processing company needs identical, auditable environments for dev, QA, and production. CloudFormation templates stored in Git ensure every environment is provisioned identically.

2. **E-commerce Platform:** During Black Friday, a retail company needs to spin up additional infrastructure in a second AWS region within minutes. CloudFormation enables one-click cross-region replication.

3. **Enterprise Compliance:** A healthcare company (HIPAA) needs to prove to auditors that all infrastructure changes go through an approved change management process. CloudFormation change sets + Git-based workflows provide this audit trail.

4. **SaaS Provider:** A B2B software company needs to provision an isolated AWS environment per enterprise customer. CloudFormation stacks with customer-specific parameters automate this onboarding.

---

## Internal Working

### Template Processing Pipeline

When you submit a CloudFormation template, the service executes the following internal workflow:

```
Template Submission
       │
       ▼
┌─────────────────────┐
│  Template Parsing   │  ← JSON/YAML parsing, syntax validation
│  & Validation       │  ← Schema validation against resource specs
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│  Dependency Graph   │  ← Builds a DAG (Directed Acyclic Graph)
│  Construction       │  ← Resolves !Ref, !GetAtt, DependsOn
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│  Change Set         │  ← Computes diff between desired and current state
│  Computation        │  ← Identifies Add / Modify / Remove operations
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│  Resource           │  ← Calls AWS service APIs (EC2, RDS, IAM, etc.)
│  Provisioning       │  ← Parallel execution where dependencies allow
│  Engine             │  ← Sequential execution for dependent resources
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│  State Management   │  ← Stores resource metadata in CloudFormation's
│  & Stack State      │    internal datastore (not S3, not visible to you)
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│  Rollback Engine    │  ← On failure: reverses completed operations
│                     │  ← Respects deletion policies
└─────────────────────┘
```

### Dependency Resolution

CloudFormation builds a **Directed Acyclic Graph (DAG)** of resource dependencies:

- **Implicit dependencies** are detected via `!Ref` and `!GetAtt` intrinsic functions
- **Explicit dependencies** are declared via `DependsOn`
- Resources with no dependencies on each other are provisioned **in parallel**
- Resources that depend on others wait for their dependencies to reach `CREATE_COMPLETE`

**Example DAG:**
```
VPC ──► Subnet ──► EC2Instance
  │                    ▲
  └──► SecurityGroup ──┘
```

In this graph, `Subnet` and `SecurityGroup` can be created in parallel after `VPC` is ready, and `EC2Instance` waits for both.

### Stack State Machine

Each stack resource transitions through states:

```
CREATE_IN_PROGRESS → CREATE_COMPLETE
                  → CREATE_FAILED → ROLLBACK_IN_PROGRESS → ROLLBACK_COMPLETE
UPDATE_IN_PROGRESS → UPDATE_COMPLETE
                  → UPDATE_FAILED → UPDATE_ROLLBACK_IN_PROGRESS → UPDATE_ROLLBACK_COMPLETE
DELETE_IN_PROGRESS → DELETE_COMPLETE
                  → DELETE_FAILED
```

### Custom Resources & Lambda Integration

For resources not natively supported, CloudFormation invokes **Lambda functions** or **SNS topics** as **Custom Resources**. The flow:

1. CloudFormation sends a `CREATE`/`UPDATE`/`DELETE` event to the Lambda function
2. Lambda performs the custom logic (e.g., calls a third-party API)
3. Lambda sends a response (success/failure) to a pre-signed S3 URL provided by CloudFormation
4. CloudFormation reads the response and continues or rolls back

---

## Architecture

### Core Architectural Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        Developer / CI-CD                         │
│              (Git, CodePipeline, Console, CLI, SDK)              │
└───────────────────────────┬─────────────────────────────────────┘
                            │ Template (YAML/JSON)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CloudFormation Service                         │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  Template    │  │  Stack       │  │  StackSet              │ │
│  │  Registry   │  │  Management  │  │  Management            │ │
│  └──────────────┘  └──────────────┘  └────────────────────────┘ │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  Change Set  │  │  Drift       │  │  CloudFormation        │ │
│  │  Engine      │  │  Detection   │  │  Registry              │ │
│  └──────────────┘  └──────────────┘  └────────────────────────┘ │
└───────────────────────────┬─────────────────────────────────────┘
                            │ AWS API Calls
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     AWS Resource Layer                            │
│   EC2 │ VPC │ RDS │ S3 │ Lambda │ IAM │ ECS │ EKS │ ...        │
└─────────────────────────────────────────────────────────────────┘
```

### Key Architectural Concepts

#### 1. Templates
The source of truth. A CloudFormation template has the following top-level sections:

```yaml
AWSTemplateFormatVersion: "2010-09-09"  # Optional, always this value
Description: "Human-readable description"  # Optional
Metadata: {}          # Optional - template metadata
Parameters: {}        # Optional - input values
Rules: {}             # Optional - parameter validation rules
Mappings: {}          # Optional - static lookup tables
Conditions: {}        # Optional - conditional resource creation
Transform: {}         # Optional - macros (e.g., SAM transform)
Resources: {}         # REQUIRED - resource definitions
Outputs: {}           # Optional - exported values
```

#### 2. Stacks
A **stack** is a single unit of related AWS resources managed as a group. Operations (create, update, delete) apply to the entire stack.

#### 3. StackSets
**StackSets** extend stacks to deploy across **multiple AWS accounts and regions** from a single template. Uses a **administrator account** → **target accounts** model.

```
Administrator Account
       │
       ├──► Account A, us-east-1
       ├──► Account A, eu-west-1
       ├──► Account B, us-east-1
       └──► Account C, ap-southeast-1
```

#### 4. Nested Stacks
Break large templates into smaller, reusable components using `AWS::CloudFormation::Stack`:

```
Root Stack
    ├── Network Stack (VPC, Subnets, IGW)
    ├── Security Stack (IAM Roles, Security Groups)
    ├── Database Stack (RDS, ElastiCache)
    └── Application Stack (ECS, ALB, Auto Scaling)
```

#### 5. CloudFormation Registry
Allows you to register **private** and **public** resource types (including third-party resources like Datadog monitors or MongoDB Atlas clusters) and use them in templates.

#### 6. Change Sets
Preview the impact of a stack update before executing it. Shows which resources will be **Added**, **Modified**, or **Removed** — and whether the modification requires **replacement** (new resource) or is done **in-place**.

---

## Real World Example

### Scenario: Three-Tier Web Application on AWS

**Goal:** Deploy a production-ready three-tier web application (ALB → ECS Fargate → RDS Aurora) using CloudFormation with separate stacks per tier.

#### Step 1: Network Stack (network.yaml)

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Network infrastructure - VPC, Subnets, IGW, NAT"

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-vpc"

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs ""]
      MapPublicIpOnLaunch: true

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs ""]
      MapPublicIpOnLaunch: true

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.3.0/24
      AvailabilityZone: !Select [0, !GetAZs ""]

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.4.0/24
      AvailabilityZone: !Select [1, !GetAZs ""]

Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: !Sub "${Environment}-VPCId"

  PublicSubnets:
    Value: !Join [",", [!Ref PublicSubnet1, !Ref PublicSubnet2]]
    Export:
      Name: !Sub "${Environment}-PublicSubnets"

  PrivateSubnets:
    Value: !Join [",", [!Ref PrivateSubnet1, !Ref PrivateSubnet2]]
    Export:
      Name: !Sub "${Environment}-PrivateSubnets"
```

#### Step 2: Deploy Network Stack

```bash
aws cloudformation deploy \
  --template-file network.yaml \
  --stack-name prod-network \
  --parameter-overrides Environment=prod \
  --region us-east-1
```

#### Step 3: Application Stack References Network Outputs

```yaml
# In the application stack, reference the network stack outputs:
VpcId:
  Fn::ImportValue: !Sub "${Environment}-VPCId"

Subnets:
  Fn::Split:
    - ","
    - Fn::ImportValue: !Sub "${Environment}-PrivateSubnets"
```

#### Step 4: Create a Change Set Before Updating

```bash
# Create change set
aws cloudformation create-change-set \
  --stack-name prod-network \
  --template-body file://network-v2.yaml \
  --change-set-name update-add-nat-gateway

# Review the change set
aws cloudformation describe-change-set \
  --stack-name prod-network \
  --change-set-name update-add-nat-gateway

# Execute after review
aws cloudformation execute-change-set \
  --stack-name prod-network \
  --change-set-name update-add-nat-gateway
```

#### Step 5: CI/CD Pipeline Integration

```
Git Push → CodePipeline → 
  CodeBuild (cfn-lint validate) → 
  CloudFormation (Create Change Set) → 
  Manual Approval Gate → 
  CloudFormation (Execute Change Set)
```

---

## Advantages

### 1. **Infrastructure as Code (IaC) Benefits**
- Templates are version-controlled in Git — full history of infrastructure changes
- Code reviews via pull requests catch misconfigurations before deployment
- Reproducible environments eliminate "works on my machine" infrastructure issues

### 2. **Automatic Dependency Management**
- No need to manually sequence resource creation — CloudFormation resolves the DAG automatically
- Parallel provisioning of independent resources reduces deployment time

### 3. **Rollback on Failure**
- Automatic rollback to the last known good state on stack creation/update failure
- Prevents partially-provisioned, broken infrastructure

### 4. **Drift Detection**
- Detects when resources have been manually modified outside of CloudFormation
- Helps maintain infrastructure integrity and compliance

### 5. **Native AWS Integration**
- Deep integration with all AWS services — no plugins or providers needed
- IAM-native access control for who can deploy what

### 6. **No Additional Cost for CloudFormation Itself**
- The service itself is free; you only pay for the resources it provisions

### 7. **StackSets for Multi-Account/Region Governance**
- Deploy and manage infrastructure across hundreds of accounts from a single template
- Essential for AWS Organizations-based governance

### 8. **CloudFormation Registry**
- Extend CloudFormation with custom and third-party resource types
- Manage non-AWS resources (Datadog, MongoDB, GitHub) in the same workflow

### 9. **Intrinsic Functions**
- Powerful built-in functions (`!Sub`, `!If