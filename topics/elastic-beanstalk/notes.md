# Elastic Beanstalk

## What is it?

**AWS Elastic Beanstalk** is a fully managed Platform as a Service (PaaS) offering from Amazon Web Services that enables developers to deploy, manage, and scale web applications and services without needing to manually provision or configure the underlying infrastructure. It falls under the **Compute** category of AWS services.

Elastic Beanstalk supports applications built with popular programming languages and platforms including:

- **Java** (Tomcat)
- **Node.js**
- **Python**
- **Ruby**
- **PHP**
- **.NET** (Windows Server with IIS)
- **Go**
- **Docker** (single and multi-container)
- **Packer Builder**

You simply upload your application code (as a ZIP, WAR, or Docker image), and Elastic Beanstalk automatically handles:

- Capacity provisioning
- Load balancing
- Auto Scaling
- Application health monitoring
- Platform patching and updates

> **Key distinction:** Unlike fully managed services (e.g., Lambda), Elastic Beanstalk gives you full access to the underlying EC2 instances, RDS databases, and other AWS resources. You retain control while AWS manages the orchestration.

---

## Why do we need it?

### The Problem It Solves

Deploying a web application on AWS from scratch requires provisioning EC2 instances, configuring security groups, setting up load balancers, configuring Auto Scaling groups, managing IAM roles, setting up CloudWatch alarms, and more. This creates significant operational overhead, especially for small teams or developers who want to focus on business logic rather than infrastructure management.

### When to Use Elastic Beanstalk

| Scenario | Reason |
|---|---|
| Startups or small teams | Minimal DevOps expertise required |
| Rapid prototyping | Fast deployment without infrastructure setup |
| Standard web applications | Well-supported language runtimes |
| Lift-and-shift migrations | Minimal code changes required |
| Teams adopting AWS | Learning AWS with guardrails |

### Real Business Scenarios

1. **E-commerce startup:** A 5-person team building a Node.js storefront needs to go live in 2 weeks. They use Elastic Beanstalk to deploy without hiring a DevOps engineer, focusing entirely on product features.

2. **Enterprise migration:** A company running a legacy Java/Tomcat application on-premises wants to migrate to AWS with minimal refactoring. Elastic Beanstalk's Tomcat platform makes this nearly seamless.

3. **SaaS product:** A B2B SaaS company needs to deploy multiple environments (dev, staging, production) for their Python Django application with consistent configuration across environments.

4. **Agency development:** A digital agency managing dozens of client websites uses Elastic Beanstalk to standardize deployments across PHP/WordPress and Python/Django applications.

---

## Internal Working

### Deployment Flow

When you deploy an application to Elastic Beanstalk, the following sequence occurs internally:

```
Developer uploads code (ZIP/WAR/Docker)
         │
         ▼
  Elastic Beanstalk API
         │
         ▼
  Application Version stored in S3
         │
         ▼
  Environment Update Triggered
         │
         ├── EC2 Instance(s) provisioned via Auto Scaling Group
         ├── Load Balancer configured (ALB/NLB/Classic)
         ├── Security Groups applied
         ├── IAM Instance Profile attached
         └── Application deployed via cfn-hup / eb-agent
```

### Core Internal Components

#### 1. Application Version Management
Every deployment creates an **Application Version** — a labeled iteration of deployable code stored as a ZIP/WAR in an S3 bucket managed by Elastic Beanstalk. Versions are immutable and can be redeployed at any time.

#### 2. Environment Configuration
Elastic Beanstalk uses **saved configurations** (`.ebextensions`) and environment properties to configure the platform. These are processed during environment creation and updates.

#### 3. CloudFormation Under the Hood
Elastic Beanstalk internally uses **AWS CloudFormation** to provision and manage resources. Each environment corresponds to a CloudFormation stack. This means:
- You can inspect the CloudFormation stack in the console
- Resource changes trigger CloudFormation stack updates
- Stack drift can occur if you manually modify resources

#### 4. EC2 Instance Bootstrapping
When an EC2 instance is launched within an Elastic Beanstalk environment:
1. The **eb-agent** (Elastic Beanstalk agent) is pre-installed on the AMI
2. The agent downloads the application version from S3
3. Container commands and `.ebextensions` hooks are executed
4. The application server (Nginx/Apache/IIS) is configured and started
5. Health checks begin reporting to Elastic Beanstalk

#### 5. Health Monitoring System
Elastic Beanstalk has a two-tier health reporting system:
- **Basic health:** Based on ELB health checks and EC2 instance status
- **Enhanced health:** Uses a dedicated health agent on each instance that reports CPU, memory, load, and application-level metrics every 10 seconds

---

## Architecture

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    AWS Account                           │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Elastic Beanstalk Application       │    │
│  │                                                  │    │
│  │  ┌──────────────┐    ┌──────────────────────┐   │    │
│  │  │  Environment  │    │    Environment        │   │    │
│  │  │  (Production) │    │    (Staging)          │   │    │
│  │  │               │    │                      │   │    │
│  │  │  ┌─────────┐  │    │  ┌─────────┐         │   │    │
│  │  │  │   ALB   │  │    │  │   ALB   │         │   │    │
│  │  │  └────┬────┘  │    │  └────┬────┘         │   │    │
│  │  │       │       │    │       │               │   │    │
│  │  │  ┌────▼────┐  │    │  ┌────▼────┐         │   │    │
│  │  │  │  ASG    │  │    │  │  ASG    │         │   │    │
│  │  │  │ ┌─────┐ │  │    │  │ ┌─────┐ │         │   │    │
│  │  │  │ │ EC2 │ │  │    │  │ │ EC2 │ │         │   │    │
│  │  │  │ └─────┘ │  │    │  │ └─────┘ │         │   │    │
│  │  │  │ ┌─────┐ │  │    │  └─────────┘         │   │    │
│  │  │  │ │ EC2 │ │  │    │                      │   │    │
│  │  │  │ └─────┘ │  │    └──────────────────────┘   │    │
│  │  │  └─────────┘  │                                │    │
│  │  └───────────────┘                                │    │
│  │                                                   │    │
│  │  S3 Bucket (Application Versions)                 │    │
│  └───────────────────────────────────────────────────┘    │
│                                                           │
│  Supporting Services: RDS, ElastiCache, SQS, CloudWatch   │
└───────────────────────────────────────────────────────────┘
```

### Key Architectural Components

#### 1. Application
The top-level logical container. An application holds:
- Multiple **environments** (e.g., prod, staging, dev)
- All **application versions**
- **Saved configurations**

#### 2. Environment
An environment is a running version of your application. Two types exist:

| Type | Use Case |
|---|---|
| **Web Server Environment** | Handles HTTP/HTTPS requests; includes ALB |
| **Worker Environment** | Processes background tasks from SQS queue |

#### 3. Environment Tier Architecture

**Web Server Tier:**
```
Internet → Route 53 → ALB → Auto Scaling Group (EC2 instances) → RDS/Cache
```

**Worker Tier:**
```
SQS Queue → sqsd daemon (on EC2) → Application → Response back to SQS
```

#### 4. Platform Versions
Each environment runs on a specific **platform version** (e.g., `Node.js 18 running on 64bit Amazon Linux 2023`). Platforms are versioned and updated by AWS regularly.

#### 5. Deployment Policies

| Policy | Description | Downtime |
|---|---|---|
| **All at once** | Deploy to all instances simultaneously | Yes (brief) |
| **Rolling** | Deploy in batches | No (reduced capacity) |
| **Rolling with additional batch** | Adds new instances before deploying | No |
| **Immutable** | New ASG with new instances; swap on success | No |
| **Traffic splitting** | Canary deployment with configurable traffic % | No |
| **Blue/Green** | Swap environment URLs | No |

---

## Real World Example

### Scenario: Deploying a Node.js REST API for a FinTech Application

A FinTech startup needs to deploy a Node.js Express API that handles payment processing. Requirements:
- Zero-downtime deployments
- Auto Scaling based on load
- Secure environment with no public EC2 access
- Separate environments for dev and production

#### Step-by-Step Walkthrough

**Step 1: Initialize the Elastic Beanstalk Application**

```bash
# Install EB CLI
pip install awsebcli

# Navigate to project directory
cd payment-api

# Initialize EB application
eb init payment-api \
  --platform "Node.js 18 running on 64bit Amazon Linux 2023" \
  --region us-east-1
```

**Step 2: Configure the Application**

Create `.elasticbeanstalk/config.yml`:
```yaml
branch-defaults:
  main:
    environment: payment-api-prod
  develop:
    environment: payment-api-dev

global:
  application_name: payment-api
  default_ec2_keyname: fintech-keypair
  default_platform: Node.js 18 running on 64bit Amazon Linux 2023
  default_region: us-east-1
  sc: git
```

**Step 3: Add `.ebextensions` Configuration**

Create `.ebextensions/01_environment.config`:
```yaml
option_settings:
  aws:elasticbeanstalk:application:environment:
    NODE_ENV: production
    PORT: 8080
  aws:elasticbeanstalk:environment:process:default:
    HealthCheckPath: /health
    Port: 8080
    Protocol: HTTP
  aws:autoscaling:launchconfiguration:
    InstanceType: t3.medium
    SecurityGroups: sg-0abc12345
  aws:elasticbeanstalk:cloudwatch:logs:
    StreamLogs: true
    DeleteOnTerminate: false
    RetentionInDays: 30
```

Create `.ebextensions/02_nginx.config`:
```yaml
files:
  "/etc/nginx/conf.d/proxy.conf":
    mode: "000644"
    owner: root
    group: root
    content: |
      client_max_body_size 10M;
      
container_commands:
  01_reload_nginx:
    command: "service nginx reload"
    ignoreErrors: true
```

**Step 4: Add Procfile for Node.js**

```
web: node server.js
```

**Step 5: Create Production Environment**

```bash
eb create payment-api-prod \
  --instance-type t3.medium \
  --min-instances 2 \
  --max-instances 10 \
  --elb-type application \
  --vpc.id vpc-0abc123 \
  --vpc.elbsubnets subnet-pub1,subnet-pub2 \
  --vpc.ec2subnets subnet-priv1,subnet-priv2 \
  --vpc.elbpublic \
  --envvars "DB_HOST=mydb.cluster.us-east-1.rds.amazonaws.com,DB_PORT=5432"
```

**Step 6: Configure Immutable Deployment Policy**

```bash
eb config payment-api-prod
```

Update deployment policy in the config:
```yaml
aws:elasticbeanstalk:command:
  DeploymentPolicy: Immutable
  HealthCheckSuccessThreshold: Ok
  IgnoreHealthCheck: false
  Timeout: 600
```

**Step 7: Deploy Application**

```bash
# Deploy to production
eb deploy payment-api-prod --staged

# Monitor deployment
eb events payment-api-prod --follow
```

**Step 8: Verify and Monitor**

```bash
# Check environment health
eb health payment-api-prod --refresh

# View application logs
eb logs payment-api-prod

# Open application in browser
eb open payment-api-prod
```

**Result:** The API is deployed across 2 AZs with an ALB, Auto Scaling between 2-10 instances, CloudWatch logging enabled, and zero-downtime immutable deployments configured.

---

## Advantages

### 1. **Speed of Deployment**
Deploy a production-ready application in minutes without manual infrastructure setup. The learning curve for AWS infrastructure is dramatically reduced.

### 2. **Full Infrastructure Control**
Unlike fully managed PaaS solutions, you retain SSH access to EC2 instances, can customize Nginx/Apache configurations, and can inspect all underlying resources.

### 3. **No Additional Cost**
Elastic Beanstalk itself is free. You only pay for the underlying AWS resources (EC2, ALB, RDS, etc.) that it provisions.

### 4. **Multiple Deployment Strategies**
Built-in support for rolling, immutable, and blue/green deployments enables zero-downtime releases without custom tooling.

### 5. **Managed Platform Updates**
AWS regularly releases updated platform versions with security patches. Elastic Beanstalk can apply managed platform updates automatically during maintenance windows.

### 6. **Environment Cloning**
Clone an existing environment (including all configuration) with a single command — ideal for spinning up staging environments that mirror production.

### 7. **Built-in Health Monitoring**
Enhanced health reporting provides detailed instance-level metrics without requiring custom monitoring setup.

### 8. **Multi-Environment Support**
Easily manage development, staging, and production environments under a single application umbrella with environment-specific configurations.

### 9. **Integration with Developer Tools**
Native integration with AWS CodePipeline, CodeBuild, and CodeDeploy enables full CI/CD pipelines.

### 10. **`.ebextensions` Customization**
Powerful configuration-as-code system allows customizing EC2 instances, installing packages, running commands, and modifying platform behavior.

---

## Limitations

### Hard Limits and Quotas

| Resource | Default Limit |
|---|---|
| Applications per region | 75 |
| Application versions per application | 1,000 |
| Environments per application | 200 |
| Saved configurations per application | 2,000 |
| Custom platforms per region | 25 |
| Environment name length | 40 characters |

### Technical Limitations

1. **Platform Lock-in:** You are constrained to AWS-supported platform versions. Custom runtimes require Docker or custom platforms.

2. **Slow Environment Creation:** Creating a new environment can take 5-15 minutes, making it unsuitable for rapid scaling scenarios.

3. **Limited Kubernetes Support:** Elastic Beanstalk's multi-container Docker uses ECS under the hood, not Kubernetes. For Kubernetes, EKS is preferred.

4. **CloudFormation Dependency:** Since Elastic Beanstalk uses CloudFormation internally, CloudFormation service limits apply. Stack updates can fail if CloudFormation is experiencing issues.

5. **Manual Resource Management Risks:** Manually modifying resources created by Elastic Beanstalk (e.g., changing ASG settings directly) can cause conflicts during the next deployment.

6. **No Native Serverless Support:** Elastic Beanstalk cannot scale to zero. Minimum instance count is always 1 (unless using scheduled scaling).

7