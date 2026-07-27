# CodeDeploy

## What is it?

**AWS CodeDeploy** is a fully managed, automated deployment service that belongs to the **AWS Developer Tools** category (part of the broader **AWS CodeSuite / CI/CD ecosystem**). It automates the deployment of application content — including code, serverless functions, web and configuration files, executables, packages, scripts, and multimedia files — to a variety of compute targets.

CodeDeploy supports deployments to:
- **Amazon EC2 instances**
- **On-premises servers** (physical or virtual machines running outside AWS)
- **AWS Lambda functions** (serverless deployments)
- **Amazon ECS services** (container deployments via Blue/Green)

It is compute-platform agnostic, meaning it can orchestrate deployments across hybrid environments with a unified workflow. CodeDeploy is often used as the **CD (Continuous Delivery/Deployment)** stage in a CI/CD pipeline alongside AWS CodePipeline, CodeBuild, and CodeCommit.

---

## Why do we need it?

### The Problem

Deploying application updates manually or with ad-hoc scripts introduces significant risk:
- **Human error** during manual deployments causes downtime.
- **No rollback strategy** — recovering from a bad deployment is slow and painful.
- **Inconsistent environments** — different servers may end up in different states.
- **Zero-downtime deployments** are difficult to implement without a framework.
- **Coordinating deployments** across hundreds or thousands of instances is operationally complex.

### What CodeDeploy Solves

| Problem | CodeDeploy Solution |
|---|---|
| Manual deployment risk | Automated, repeatable deployment process |
| Downtime during deployments | Rolling, Blue/Green, and Canary strategies |
| No rollback | Automatic rollback on failure or alarms |
| Multi-server coordination | Fleet-wide deployment with configurable batch sizes |
| Hybrid environments | Unified deployment to EC2 and on-premises |

### Real Business Scenarios

1. **E-commerce Platform**: A retail company deploys new checkout code to 500 EC2 instances every Friday evening. CodeDeploy rolls out the change to 10% of servers at a time, monitors error rates, and automatically rolls back if CloudWatch alarms trigger.

2. **Microservices on ECS**: A SaaS company runs 20 containerized microservices on ECS Fargate. CodeDeploy orchestrates Blue/Green deployments, shifting traffic gradually from old task sets to new ones with zero downtime.

3. **Lambda API Updates**: A fintech company updates a payment processing Lambda function using CodeDeploy's Canary deployment, routing 10% of traffic to the new version for 10 minutes before full cutover, with automatic rollback if errors spike.

4. **Hybrid Enterprise**: A bank runs critical workloads on-premises alongside EC2. CodeDeploy manages deployments across both environments from a single control plane.

---

## Internal Working

### High-Level Flow

```
Developer → Pushes Revision → S3 / GitHub / Bitbucket / ECR
                                        ↓
                              CodeDeploy Application
                                        ↓
                              Deployment Group (targets)
                                        ↓
                         CodeDeploy Agent (on EC2/on-prem)
                                        ↓
                              AppSpec File (instructions)
                                        ↓
                              Lifecycle Hooks (scripts)
                                        ↓
                              Deployed Application
```

### Core Concepts

#### 1. Revision
A **revision** is the deployable content — the application artifacts plus the `appspec.yml` file. It is stored in:
- **Amazon S3** (zip, tar, tar.gz bundles)
- **GitHub** or **Bitbucket** (repository reference)
- **Amazon ECR** (for ECS deployments)

#### 2. AppSpec File (`appspec.yml`)
The AppSpec (Application Specification) file is the **heart of CodeDeploy**. It defines:
- **Where** files should be copied on the target
- **What scripts** to run at each lifecycle event
- **Permissions** to apply to deployed files

```yaml
# EC2/On-Premises AppSpec example
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/myapp
hooks:
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/change_permissions.sh
      timeout: 180
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 60
  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 60
```

#### 3. CodeDeploy Agent
A **lightweight daemon** installed on EC2 or on-premises instances. It:
- Polls the CodeDeploy service endpoint via HTTPS
- Downloads the revision from S3 or GitHub
- Executes lifecycle hooks defined in `appspec.yml`
- Reports deployment status back to CodeDeploy
- Runs as a system service (`codedeploy-agent`)
- Communicates on **port 443 outbound** (no inbound ports required)

#### 4. Deployment Lifecycle Events (EC2/On-Premises)

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

For **Blue/Green deployments**, additional events exist:
- `BeforeBlockTraffic` / `BlockTraffic` / `AfterBlockTraffic`
- `BeforeAllowTraffic` / `AllowTraffic` / `AfterAllowTraffic`

#### 5. Deployment Configurations
Define the **speed and safety** of deployments:

| Configuration | Description |
|---|---|
| `CodeDeployDefault.AllAtOnce` | Deploy to all instances simultaneously |
| `CodeDeployDefault.HalfAtATime` | Deploy to 50% of instances at a time |
| `CodeDeployDefault.OneAtATime` | Deploy to one instance at a time (safest) |
| Custom | Define minimum healthy hosts (count or percentage) |

---

## Architecture

### Components Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     AWS CodeDeploy                          │
│                                                             │
│  ┌─────────────┐    ┌──────────────────┐                   │
│  │ Application │───▶│ Deployment Group │                   │
│  │  (logical   │    │  (target set +   │                   │
│  │  container) │    │   config + IAM)  │                   │
│  └─────────────┘    └────────┬─────────┘                   │
│                              │                              │
│                    ┌─────────▼──────────┐                  │
│                    │    Deployment      │                   │
│                    │ (specific revision │                   │
│                    │  + configuration) │                   │
│                    └─────────┬──────────┘                  │
└──────────────────────────────┼──────────────────────────────┘
                               │
           ┌───────────────────┼───────────────────┐
           ▼                   ▼                   ▼
    ┌─────────────┐   ┌─────────────┐   ┌──────────────┐
    │  EC2 Fleet  │   │  On-Prem    │   │  Lambda /    │
    │  (with      │   │  Servers    │   │  ECS Service │
    │   Agent)    │   │  (with      │   │              │
    └─────────────┘   │   Agent)    │   └──────────────┘
                      └─────────────┘
```

### Deployment Strategies

#### 1. In-Place Deployment (EC2/On-Premises only)
```
[Instance 1] → Stop App → Deploy → Start App
[Instance 2] → Stop App → Deploy → Start App  (rolling)
[Instance 3] → Stop App → Deploy → Start App
```
- The same instances are updated
- Load balancer can deregister instances during update
- No additional infrastructure cost

#### 2. Blue/Green Deployment
```
BLUE (current)          GREEN (new)
┌──────────┐           ┌──────────┐
│ v1 fleet │           │ v2 fleet │
│          │           │  (new    │
│  Active  │    ──▶    │instances)│
└──────────┘           └──────────┘
     │                      │
     └──────── ALB ──────────┘
               │
        Traffic shifted from
        Blue → Green
```
- New instances (Green) are provisioned with the new version
- Traffic is shifted via the load balancer
- Blue instances remain available for rollback
- After validation period, Blue instances are terminated (configurable)

#### 3. Lambda Deployment Types
| Type | Traffic Shift |
|---|---|
| **Canary** | X% → wait N minutes → 100% |
| **Linear** | X% every N minutes until 100% |
| **All-at-once** | Immediate 100% shift |

Example configurations:
- `CodeDeployDefault.LambdaCanary10Percent5Minutes`
- `CodeDeployDefault.LambdaLinear10PercentEvery1Minute`
- `CodeDeployDefault.LambdaAllAtOnce`

#### 4. ECS Deployment
- Creates a new **task set** (Green)
- Shifts traffic via ALB listener rules
- Supports same Canary/Linear/All-at-once strategies as Lambda
- Old task set (Blue) remains for rollback window

---

## Real World Example

### Scenario: Zero-Downtime Web Application Deployment

**Context**: A Node.js Express API running on 10 EC2 instances behind an Application Load Balancer. The team deploys updates multiple times per day and cannot afford downtime.

#### Step 1: Setup IAM Roles

```
CodeDeploy Service Role → Allows CodeDeploy to call EC2, ALB, ASG APIs
EC2 Instance Profile   → Allows instances to pull from S3, report to CodeDeploy
```

#### Step 2: Install CodeDeploy Agent on EC2

```bash
# On Amazon Linux 2
sudo yum update -y
sudo yum install -y ruby wget
wget https://aws-codedeploy-us-east-1.s3.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
sudo systemctl start codedeploy-agent
sudo systemctl enable codedeploy-agent
```

#### Step 3: Application Structure

```
my-node-api/
├── appspec.yml
├── scripts/
│   ├── install_dependencies.sh
│   ├── start_server.sh
│   ├── stop_server.sh
│   └── validate_service.sh
├── src/
│   └── app.js
└── package.json
```

#### Step 4: AppSpec File

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /opt/my-node-api
    overwrite: true
permissions:
  - object: /opt/my-node-api
    owner: ec2-user
    group: ec2-user
    mode: 755
hooks:
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 30
      runas: ec2-user
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 60
      runas: ec2-user
  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 60
      runas: ec2-user
```

#### Step 5: Deployment Scripts

```bash
# stop_server.sh
#!/bin/bash
if pgrep -f "node.*app.js" > /dev/null; then
  pkill -f "node.*app.js"
  sleep 2
fi

# start_server.sh
#!/bin/bash
cd /opt/my-node-api
npm install --production
pm2 start src/app.js --name "my-api" --restart-delay=1000
pm2 save

# validate_service.sh
#!/bin/bash
sleep 5
curl -sf http://localhost:3000/health || exit 1
echo "Service validation passed"
```

#### Step 6: Create CodeDeploy Resources

```bash
# Create Application
aws deploy create-application \
  --application-name MyNodeAPI \
  --compute-platform Server

# Create Deployment Group
aws deploy create-deployment-group \
  --application-name MyNodeAPI \
  --deployment-group-name Production \
  --deployment-config-name CodeDeployDefault.HalfAtATime \
  --ec2-tag-filters Key=Environment,Value=production,Type=KEY_AND_VALUE \
  --load-balancer-info elbInfoList=[{name=my-alb}] \
  --auto-rollback-configuration enabled=true,events="DEPLOYMENT_FAILURE,DEPLOYMENT_STOP_ON_ALARM" \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployServiceRole
```

#### Step 7: Package and Deploy

```bash
# Package revision to S3
aws deploy push \
  --application-name MyNodeAPI \
  --s3-location s3://my-deploy-bucket/MyNodeAPI/revision.zip \
  --source .

# Create deployment
aws deploy create-deployment \
  --application-name MyNodeAPI \
  --deployment-group-name Production \
  --s3-location bucket=my-deploy-bucket,key=MyNodeAPI/revision.zip,bundleType=zip \
  --description "Release v2.3.1 - Payment bug fix"
```

#### Step 8: Monitor

```bash
# Watch deployment progress
aws deploy get-deployment --deployment-id d-EXAMPLE123
aws deploy list-deployment-instances --deployment-id d-EXAMPLE123
```

**Result**: The deployment rolls out to 5 instances at a time. The ALB deregisters each batch before deployment and re-registers after validation. If any instance fails the `ValidateService` hook, CodeDeploy automatically rolls back all instances to the previous version.

---

## Advantages

1. **Zero-downtime deployments**: Rolling, Blue/Green, and Canary strategies minimize or eliminate user impact during updates.

2. **Automatic rollback**: Integrates with CloudWatch Alarms to trigger rollback automatically when error rates spike — no human intervention needed.

3. **Compute platform agnostic**: Single service handles EC2, on-premises, Lambda, and ECS deployments with a consistent model.

4. **Centralized deployment tracking**: Full audit trail of every deployment — who triggered it, what revision, which instances succeeded/failed, and timestamps.

5. **Integration with CI/CD**: Native integration with CodePipeline, CodeBuild, Jenkins, GitHub Actions, and Bitbucket Pipelines.

6. **Lifecycle hooks**: Fine-grained control over every phase of the deployment process with custom scripts.

7. **No infrastructure to manage**: Fully managed service — no deployment servers to patch or maintain.

8. **Cost-effective**: No additional charge for EC2 and on-premises deployments; only Lambda and ECS deployments have a per-update charge.

9. **Scalability**: Handles deployments to fleets of thousands of instances with configurable concurrency.

10. **Health-aware deployments**: Integrates with load balancers and auto-scaling groups to ensure only healthy instances receive traffic.

---

## Limitations

### Hard Limits

| Limit | Value |
|---|---|
| Applications per region per account | 1,000 |
| Deployment groups per application | 1,000 |
| Deployments per deployment group (concurrent) | 1 |
| Instances per deployment group | 500 (default, can be increased) |
| Maximum revision bundle size (S3) | 3 GB |
| Maximum AppSpec file size | 4 MB |
| Deployment lifecycle event timeout | 3,600 seconds (1 hour) max |
| Concurrent deployments per account per region | 1,000 |
| GitHub tokens per account | 25 |

### Functional Limitations

- **