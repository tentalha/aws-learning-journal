# Elastic Beanstalk — Interview Questions

---

## Easy

### 1. What is AWS Elastic Beanstalk and what problem does it solve?

**Answer:**
AWS Elastic Beanstalk is a Platform as a Service (PaaS) offering that allows developers to deploy and manage web applications and services without needing to manually provision or manage the underlying infrastructure. You simply upload your application code, and Elastic Beanstalk automatically handles:

- Capacity provisioning
- Load balancing
- Auto Scaling
- Application health monitoring
- OS and platform patching

It solves the problem of infrastructure complexity for developers who want to focus on writing code rather than managing servers, networking, and deployment pipelines. You retain full control over the AWS resources powering your application and can access them at any time.

---

### 2. What programming languages and platforms does Elastic Beanstalk support?

**Answer:**
Elastic Beanstalk supports a wide range of platforms and languages, including:

| Platform | Notes |
|---|---|
| Java (Tomcat) | SE and EE support |
| .NET on Windows Server | IIS |
| .NET Core on Linux | Cross-platform |
| Node.js | |
| PHP | |
| Python | |
| Ruby | |
| Go | |
| Docker | Single and Multi-container |
| Packer Builder | Custom AMIs |

Elastic Beanstalk also supports **custom platforms** built using Packer, allowing teams to create their own platform definitions if the built-in ones do not meet their requirements.

---

### 3. What is the difference between an Elastic Beanstalk Application, Environment, and Application Version?

**Answer:**

| Concept | Description |
|---|---|
| **Application** | A logical collection of Elastic Beanstalk components, including environments, versions, and configurations. Think of it as the top-level container. |
| **Application Version** | A specific, labeled iteration of deployable code. It points to an S3 object (ZIP or WAR file) containing your application code. You can have multiple versions stored simultaneously. |
| **Environment** | A running instance of an application version. It provisions the actual AWS resources (EC2, ELB, Auto Scaling Group, etc.). A single application can have multiple environments (e.g., `production`, `staging`, `dev`). |

**Relationship:**
```
Application
  ├── Application Version v1.0
  ├── Application Version v1.1
  ├── Environment: Production (running v1.0)
  └── Environment: Staging (running v1.1)
```

---

### 4. What are the two environment tiers in Elastic Beanstalk?

**Answer:**

**1. Web Server Tier:**
- Designed for standard web applications that handle HTTP/HTTPS requests.
- Provisions an Elastic Load Balancer (ELB), Auto Scaling Group, and EC2 instances.
- Typically used for front-end web applications or REST APIs.
- Uses an `nginx` or `Apache` proxy by default.

**2. Worker Tier:**
- Designed for background processing tasks that are decoupled from the web tier.
- Provisions an SQS queue, Auto Scaling Group, and EC2 instances.
- A daemon on each EC2 instance polls the SQS queue and processes messages.
- Ideal for long-running tasks like video encoding, email sending, or report generation.
- Supports periodic tasks via a `cron.yaml` file.

The two tiers are commonly used together — the web tier accepts requests and pushes jobs to SQS, while the worker tier consumes and processes them.

---

### 5. How does Elastic Beanstalk store and manage configuration?

**Answer:**
Elastic Beanstalk manages configuration at multiple levels:

1. **Saved Configurations:** Snapshots of an environment's configuration stored in S3, which can be applied to new or existing environments.

2. **`.ebextensions` folder:** A directory in your application source bundle containing YAML or JSON configuration files (`.config` extension). These allow you to:
   - Customize the EC2 instance (install packages, run commands)
   - Configure environment variables
   - Set up additional AWS resources
   - Modify software settings

3. **Environment Properties:** Key-value pairs (environment variables) set directly in the Elastic Beanstalk console or CLI, accessible to the application at runtime.

4. **`ebextensions` precedence order** (highest to lowest):
   - Direct API/Console settings
   - Saved configurations
   - `.ebextensions` configuration files
   - Default platform values

---

## Medium

### 1. Explain the different deployment policies available in Elastic Beanstalk and when you would use each.

**Answer:**
Elastic Beanstalk offers five deployment policies, each with different tradeoffs between availability, speed, and cost:

**1. All at Once**
- Deploys to all instances simultaneously.
- **Downtime:** Yes — all instances are unavailable during deployment.
- **Speed:** Fastest.
- **Use case:** Development environments where downtime is acceptable.
- **Rollback:** Manual re-deployment of previous version.

**2. Rolling**
- Deploys in batches. A subset of instances is taken out of service, updated, and returned before moving to the next batch.
- **Downtime:** No, but reduced capacity during deployment.
- **Speed:** Slower than All at Once.
- **Use case:** Production environments where some capacity reduction is acceptable.
- **Rollback:** Manual re-deployment.

**3. Rolling with Additional Batch**
- Launches a new batch of instances first, then rolls the deployment across all instances. Maintains full capacity throughout.
- **Downtime:** No.
- **Capacity:** Full capacity maintained.
- **Cost:** Slightly higher due to additional instances.
- **Use case:** Production environments where full capacity must be maintained.
- **Rollback:** Manual re-deployment.

**4. Immutable**
- Launches a completely new Auto Scaling Group with new instances running the new version. Traffic is shifted only after the new instances pass health checks.
- **Downtime:** No.
- **Speed:** Slowest.
- **Safety:** Highest — old instances remain until new ones are healthy.
- **Cost:** Doubles instance count temporarily.
- **Rollback:** Fastest — terminate new ASG.
- **Use case:** Mission-critical production environments.

**5. Traffic Splitting (Canary)**
- Deploys to a new set of instances and routes a configurable percentage of traffic to them. Gradually shifts traffic if health checks pass.
- **Downtime:** No.
- **Use case:** A/B testing, canary releases, validating new versions with a small percentage of real traffic.
- **Rollback:** Automatically rolls back if health check failures exceed threshold.

```
Policy Comparison:
┌──────────────────────────┬──────────┬─────────────┬──────────┬──────────────┐
│ Policy                   │ Downtime │ Full Cap.   │ Speed    │ Rollback     │
├──────────────────────────┼──────────┼─────────────┼──────────┼──────────────┤
│ All at Once              │ Yes      │ No          │ Fast     │ Manual       │
│ Rolling                  │ No       │ No          │ Medium   │ Manual       │
│ Rolling + Extra Batch    │ No       │ Yes         │ Medium   │ Manual       │
│ Immutable                │ No       │ Yes         │ Slow     │ Fast (auto)  │
│ Traffic Splitting        │ No       │ Yes         │ Medium   │ Automatic    │
└──────────────────────────┴──────────┴─────────────┴──────────┴──────────────┘
```

---

### 2. What is `.ebextensions` and how would you use it to install a custom package and run a command on EC2 instances?

**Answer:**
`.ebextensions` is a directory placed at the root of your application source bundle. It contains one or more `.config` files written in YAML or JSON that customize your Elastic Beanstalk environment during instance provisioning and application deployment.

**Key sections in an `.ebextensions` config file:**

| Section | Purpose |
|---|---|
| `packages` | Install OS packages (yum, apt, rubygems, python) |
| `sources` | Download and extract archives |
| `files` | Create files on the instance |
| `commands` | Run shell commands before the application is set up |
| `container_commands` | Run commands after the application source is extracted but before deployment |
| `services` | Enable/disable OS services |
| `option_settings` | Configure Elastic Beanstalk options |

**Example: Install `htop` and `jq`, then run a script:**

```yaml
# .ebextensions/01_custom_setup.config

packages:
  yum:
    htop: []
    jq: []

files:
  "/etc/myapp/config.json":
    mode: "000644"
    owner: root
    group: root
    content: |
      {
        "environment": "production",
        "debug": false
      }

commands:
  01_create_log_dir:
    command: "mkdir -p /var/log/myapp"
    ignoreErrors: false

container_commands:
  01_run_migrations:
    command: "python manage.py migrate"
    leader_only: true

option_settings:
  aws:elasticbeanstalk:application:environment:
    MY_ENV_VAR: "my_value"
```

**Important distinction:**
- `commands` run **before** the application source is extracted.
- `container_commands` run **after** the source is extracted, giving access to your application code.
- `leader_only: true` ensures the command runs only on one instance (useful for DB migrations).

---

### 3. How does Elastic Beanstalk health monitoring work, and what is the difference between basic and enhanced health reporting?

**Answer:**

**Basic Health Reporting:**
- Available on all environments.
- Reports health as one of four colors: **Green** (OK), **Yellow** (Warning), **Red** (Degraded), **Grey** (No data).
- Based on ELB health checks (HTTP response codes) and EC2 instance status checks.
- Metrics are sent to CloudWatch at 5-minute intervals.
- Limited visibility — you know something is wrong but not always why.

**Enhanced Health Reporting:**
- Requires the Elastic Beanstalk health agent installed on EC2 instances (included in platform AMIs from 2014 onwards).
- Provides detailed health data at **10-second intervals**.
- Monitors:
  - HTTP response rates and latency (p50, p75, p85, p90, p95, p99)
  - Instance-level CPU, load average, memory, disk I/O
  - Application process health (is the app server running?)
  - Deployment status
  - Root cause identification
- Introduces more granular health statuses: `Ok`, `Info`, `Warning`, `Degraded`, `Severe`, `No Data`, `Pending`
- Publishes metrics to CloudWatch under the `AWS/ElasticBeanstalk` namespace.
- Enables **Managed Platform Updates** and **Immutable deployments** health gates.

**When to use Enhanced:**
Always use Enhanced Health in production. The additional visibility dramatically reduces mean time to resolution (MTTR) for incidents. It also enables Elastic Beanstalk to make smarter decisions during rolling deployments (e.g., halt a rolling update if new instances are unhealthy).

**Health Check URL:**
You can configure a custom health check URL (e.g., `/health`) that returns HTTP 200 to give Elastic Beanstalk a more accurate view of application health beyond just "is the process running."

---

### 4. What are Elastic Beanstalk Managed Platform Updates and how do you configure them?

**Answer:**
Managed Platform Updates is a feature that automatically applies platform updates (OS patches, runtime updates, web server updates) to your environment during a configurable maintenance window, without requiring manual intervention.

**How it works:**
1. You define a weekly **maintenance window** (day + time + duration).
2. Elastic Beanstalk checks for available platform updates.
3. If updates are available, it applies them using an **Immutable deployment** strategy to ensure zero downtime.
4. After new instances pass health checks, the old instances are terminated.

**Update levels you can configure:**

| Level | What Gets Updated |
|---|---|
| **Minor and Patch** | Both minor version updates (e.g., 2.3.x → 2.4.x) and patch updates |
| **Patch only** | Only patch-level updates (e.g., 2.3.1 → 2.3.2) — safer, smaller changes |

**Configuration via Console:**
```
Environment → Configuration → Updates, monitoring, and logging
→ Managed platform updates: Enabled
→ Weekly update window: Tuesday at 09:00 UTC (2 hours)
→ Update level: Patch
```

**Configuration via `.ebextensions`:**
```yaml
option_settings:
  aws:elasticbeanstalk:managedactions:
    ManagedActionsEnabled: true
    PreferredStartTime: "Tue:09:00"
  aws:elasticbeanstalk:managedactions:platformupdate:
    UpdateLevel: patch
    InstanceRefreshEnabled: true
```

**Key considerations:**
- Requires **Enhanced Health Reporting** to be enabled.
- `InstanceRefreshEnabled: true` replaces all instances even if no platform update is available, ensuring instances are always running fresh AMIs.
- Managed updates respect your environment's minimum healthy instance percentage settings.
- You can view the history of managed actions in the **Managed Actions** tab.

---

### 5. How do you manage environment variables and secrets in Elastic Beanstalk securely?

**Answer:**
Managing secrets in Elastic Beanstalk requires careful consideration to avoid storing sensitive data in plaintext in source code or `.ebextensions` files.

**Option 1: Environment Properties (Basic, not for secrets)**
```bash
eb setenv DB_HOST=mydb.cluster.rds.amazonaws.com DB_PORT=5432
```
These are stored in the Elastic Beanstalk configuration and visible in the console. **Do not use for passwords or API keys.**

**Option 2: AWS Systems Manager Parameter Store (Recommended)**
- Store secrets as `SecureString` parameters encrypted with KMS.
- Retrieve them at instance startup via `.ebextensions`:

```yaml
# .ebextensions/02_fetch_secrets.config
files:
  "/opt/elasticbeanstalk/hooks/appdeploy/pre/01_fetch_secrets.sh":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/bin/bash
      DB_PASSWORD=$(aws ssm get-parameter \
        --name "/myapp/production/db_password" \
        --with-decryption \
        --query "Parameter.Value" \
        --output text \
        --region us-east-1)
      echo "DB_PASSWORD=$DB_PASSWORD" >> /etc/environment
```

- The EC2 instance profile must have `ssm:GetParameter` and `kms:Decrypt` permissions.

**Option 3: AWS Secrets Manager**
- More feature-rich than SSM Parameter Store.
- Supports automatic secret rotation.
- Retrieve via SDK at application startup:

```python
import boto3
import json

def get_secret():
    client = boto3.client('secretsmanager', region_name='us-east-1')
    response = client.get_secret_value(SecretId='myapp/production/db')
    return json.loads(response['SecretString'])
```

**Option 4: Encrypted S3 with SSE-KMS**
- Store a config file in an S3 bucket encrypted with KMS.
- Download at instance startup via `.ebextensions`.

**IAM Instance Profile Setup:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "kms:Decrypt",
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "*"
    }
  ]
}
```

**Best practice:** Use SSM Parameter Store for configuration and Secrets Manager for credentials that require rotation. Never commit secrets to source control or hardcode them in `.ebextensions`.

---

## Hard

### 1. Explain how Elastic Beanstalk integrates with a CI/CD pipeline and what the trade-offs are between using the EB CLI, CodePipeline, and custom deployment scripts.

**Answer:**

**Architecture Overview:**
```
Source