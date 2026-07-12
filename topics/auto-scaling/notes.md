# Auto Scaling

## What is it?

**AWS Auto Scaling** is a fully managed service that automatically adjusts the capacity of AWS resources to maintain steady, predictable performance at the lowest possible cost. It monitors your applications and automatically scales resources up or down in response to demand changes.

| Attribute | Details |
|---|---|
| **Official Name** | AWS Auto Scaling |
| **Category** | Management & Governance / Compute |
| **Service Type** | Fully Managed |
| **Primary Resource** | Amazon EC2 Auto Scaling Groups (ASG) |
| **Console Path** | EC2 → Auto Scaling → Auto Scaling Groups |

AWS Auto Scaling is an **umbrella service** that encompasses:

- **Amazon EC2 Auto Scaling** — Scales EC2 instances within Auto Scaling Groups
- **Application Auto Scaling** — Scales other AWS resources (ECS tasks, DynamoDB tables, Aurora replicas, Lambda concurrency, etc.)
- **AWS Auto Scaling (Unified Console)** — Provides a single interface to manage scaling plans across multiple resource types

> **Key Distinction:** "Amazon EC2 Auto Scaling" and "AWS Auto Scaling" are related but distinct. EC2 Auto Scaling focuses exclusively on EC2 instances, while AWS Auto Scaling provides a broader, unified scaling experience across services.

---

## Why do we need it?

### The Core Problem

Without Auto Scaling, you must manually provision infrastructure based on peak demand estimates. This leads to two costly failure modes:

1. **Over-provisioning** — You pay for idle capacity 24/7 to handle occasional traffic spikes
2. **Under-provisioning** — Your application degrades or fails during unexpected traffic surges

### Business Scenarios

**Scenario 1: E-Commerce Flash Sale**
An online retailer runs on 10 EC2 instances during normal hours. During a Black Friday sale, traffic spikes 20x within minutes. Without Auto Scaling, the site crashes. With Auto Scaling, the fleet automatically expands to 200 instances during the sale and contracts back afterward.

**Scenario 2: SaaS Application with Business Hours Pattern**
A B2B SaaS tool sees heavy usage 9 AM–6 PM on weekdays and near-zero usage at night and weekends. Auto Scaling with scheduled actions reduces the fleet from 50 instances to 5 during off-hours, cutting compute costs by up to 70%.

**Scenario 3: Batch Processing Workloads**
A data pipeline processes files uploaded throughout the day. As the upload queue grows, Auto Scaling adds workers automatically. When the queue empties, workers are terminated, and you pay only for actual processing time.

**Scenario 4: Microservices on ECS**
A containerized application has variable load across its services. Application Auto Scaling adjusts ECS task counts per service independently, ensuring each microservice has appropriate capacity without over-provisioning the entire cluster.

### When to Use Auto Scaling

| Use Case | Recommended |
|---|---|
| Variable or unpredictable traffic | ✅ Yes |
| Stateless application tiers | ✅ Yes |
| Cost optimization for non-peak workloads | ✅ Yes |
| High availability requirements | ✅ Yes |
| Stateful single-instance databases | ❌ Not directly |
| Workloads requiring persistent local storage | ⚠️ With caution |

---

## Internal Working

### Core Mechanism

AWS Auto Scaling operates through a **continuous feedback loop**:

```
[CloudWatch Metrics] → [Alarm Threshold] → [Scaling Policy] → [Scaling Action] → [New Capacity]
        ↑                                                                               |
        └───────────────────────── Metrics Update ──────────────────────────────────────┘
```

### Step-by-Step Internal Flow

**1. Health Monitoring**
The Auto Scaling service continuously polls health check endpoints and CloudWatch metrics. EC2 Auto Scaling checks instance health via:
- EC2 system status checks
- ELB health checks (if attached)
- Custom health checks via `SetInstanceHealth` API

**2. Metric Evaluation**
CloudWatch alarms evaluate metrics over a defined period (e.g., average CPU > 70% for 2 consecutive 5-minute periods). The alarm state transitions: `OK → ALARM → OK`.

**3. Scaling Decision**
When an alarm fires, the scaling policy calculates the desired capacity change:

```
# Simple Scaling
New Desired = Current Desired + Adjustment (e.g., +2 instances)

# Target Tracking
New Desired = Calculated to achieve target metric value (e.g., CPU = 50%)

# Step Scaling
New Desired = Based on breach magnitude (e.g., CPU 70-80% → +1, CPU 80-90% → +2, CPU >90% → +4)
```

**4. Launch/Terminate Decision**
The Auto Scaling Group compares:
- `DesiredCapacity` vs `CurrentRunningInstances`
- If Desired > Current → **Scale Out** (launch new instances)
- If Desired < Current → **Scale In** (terminate instances)

**5. Launch Template / Launch Configuration Execution**
New instances are launched using the **Launch Template** (recommended) or Launch Configuration, which defines:
- AMI ID
- Instance type
- Security groups
- IAM instance profile
- User data scripts
- EBS volumes

**6. Instance Lifecycle**

```
Pending → Pending:Wait (lifecycle hook) → Pending:Proceed → InService
    ↓
Terminating → Terminating:Wait (lifecycle hook) → Terminating:Proceed → Terminated
```

**7. Cooldown Period**
After a scaling action, a cooldown period (default: 300 seconds) prevents additional scaling actions, allowing metrics to stabilize before making further decisions.

**8. Rebalancing**
EC2 Auto Scaling continuously monitors AZ distribution. If instances are unbalanced (due to AZ capacity issues or manual termination), it automatically rebalances by launching in under-represented AZs before terminating in over-represented ones.

### Predictive Scaling Internal Flow

```
[Historical CloudWatch Data (14 days)] → [ML Forecast Model] → [Predicted Load Curve]
                                                                         ↓
                                                          [Pre-emptive Scaling Schedule]
                                                                         ↓
                                                          [Instances Ready Before Load Arrives]
```

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AWS Auto Scaling Group                          │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Launch Template                               │   │
│  │  AMI | Instance Type | Key Pair | SGs | IAM Role | User Data    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
│  │  AZ: us-east │  │  AZ: us-east │  │  AZ: us-east │                 │
│  │       1a     │  │       1b     │  │       1c     │                 │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │                 │
│  │  │  EC2   │  │  │  │  EC2   │  │  │  │  EC2   │  │                 │
│  │  │  i-001 │  │  │  │  i-002 │  │  │  │  i-003 │  │                 │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │                 │
│  └──────────────┘  └──────────────┘  └──────────────┘                 │
│                                                                         │
│  Min: 2  |  Desired: 3  |  Max: 20                                     │
└─────────────────────────────────────────────────────────────────────────┘
         │                          │                          │
         ▼                          ▼                          ▼
┌─────────────────┐    ┌─────────────────────┐    ┌──────────────────────┐
│   CloudWatch    │    │  Application Load   │    │   Scaling Policies   │
│   Alarms &      │───▶│     Balancer        │    │  - Target Tracking   │
│   Metrics       │    │  (Health Checks)    │    │  - Step Scaling      │
└─────────────────┘    └─────────────────────┘    │  - Scheduled         │
                                                   │  - Predictive        │
                                                   └──────────────────────┘
```

### Key Architectural Components

#### 1. Auto Scaling Group (ASG)
The fundamental unit. Defines:
- **Min capacity** — Floor; ASG will never go below this
- **Max capacity** — Ceiling; ASG will never exceed this
- **Desired capacity** — Current target instance count
- **VPC and Subnets** — Where to launch instances (multi-AZ recommended)
- **Health check type** — EC2 or ELB
- **Health check grace period** — Time after launch before health checks begin

#### 2. Launch Template (Recommended over Launch Configuration)

```
Launch Template v1 (current)
├── AMI: ami-0abcdef1234567890
├── Instance Type: t3.medium
├── Key Pair: my-key-pair
├── Security Groups: [sg-web-tier]
├── IAM Instance Profile: EC2-SSM-Role
├── User Data: bootstrap.sh
├── EBS: /dev/xvda 20GB gp3
└── Tags: Environment=Production
```

**Launch Templates support:**
- Multiple instance types (mixed instances policy)
- Spot + On-Demand combinations
- Versioning (v1, v2, v3...)
- Inheritance from base templates

#### 3. Scaling Policies

| Policy Type | Trigger | Use Case | Cooldown |
|---|---|---|---|
| **Target Tracking** | Metric deviates from target | General purpose, simplest | Automatic |
| **Step Scaling** | Alarm breach magnitude | Fine-grained control | Manual |
| **Simple Scaling** | Single alarm | Legacy, basic use | Manual |
| **Scheduled Scaling** | Time/cron expression | Known patterns | N/A |
| **Predictive Scaling** | ML forecast | Cyclical workloads | N/A |

#### 4. Lifecycle Hooks

```
Instance Launch                    Instance Terminate
      │                                   │
      ▼                                   ▼
 Pending:Wait ──────────────────▶  Terminating:Wait
      │                                   │
      │  (Run custom actions:             │  (Run custom actions:
      │   - Install software              │   - Drain connections
      │   - Register with config          │   - Backup logs
      │   - Run health checks)            │   - Deregister from tools)
      │                                   │
      ▼                                   ▼
 Pending:Proceed              Terminating:Proceed
      │                                   │
      ▼                                   ▼
  InService                          Terminated
```

#### 5. Instance Refresh
Allows rolling replacement of instances when the Launch Template is updated:

```
Rolling Update Strategy:
├── Min Healthy Percentage: 90%
├── Instance Warmup: 300 seconds
└── Checkpoints: [20%, 50%, 100%]
```

#### 6. Mixed Instances Policy

```json
{
  "MixedInstancesPolicy": {
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 30,
      "SpotAllocationStrategy": "capacity-optimized"
    },
    "LaunchTemplate": {
      "Overrides": [
        {"InstanceType": "m5.large"},
        {"InstanceType": "m5a.large"},
        {"InstanceType": "m4.large"}
      ]
    }
  }
}
```

---

## Real World Example

### Scenario: High-Traffic News Website

**Context:** A news website experiences unpredictable traffic spikes when breaking news stories publish. The baseline is 5 EC2 instances but during major events, they need up to 100.

#### Step 1: Create a Launch Template

```bash
aws ec2 create-launch-template \
  --launch-template-name news-site-lt \
  --version-description "v1-production" \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "c5.xlarge",
    "SecurityGroupIds": ["sg-0abc123def456"],
    "IamInstanceProfile": {"Name": "EC2-NewsApp-Role"},
    "UserData": "base64-encoded-bootstrap-script",
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [{"Key": "Application", "Value": "NewsSite"}]
    }]
  }'
```

#### Step 2: Create the Auto Scaling Group

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name news-site-asg \
  --launch-template LaunchTemplateName=news-site-lt,Version='$Latest' \
  --min-size 5 \
  --max-size 100 \
  --desired-capacity 5 \
  --vpc-zone-identifier "subnet-aaa,subnet-bbb,subnet-ccc" \
  --target-group-arns "arn:aws:elasticloadbalancing:..." \
  --health-check-type ELB \
  --health-check-grace-period 300
```

#### Step 3: Attach Target Tracking Policy

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name news-site-asg \
  --policy-name news-site-cpu-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

#### Step 4: Add Scheduled Scaling for Known Events

```bash
# Scale up before scheduled press conferences (weekdays at 2 PM EST)
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name news-site-asg \
  --scheduled-action-name pre-conference-scaleup \
  --recurrence "0 19 * * 1-5" \
  --min-size 20 \
  --desired-capacity 20
```

#### Step 5: Configure Lifecycle Hook for Log Draining

```bash
aws autoscaling put-lifecycle-hook \
  --lifecycle-hook-name drain-logs-on-terminate \
  --auto-scaling-group-name news-site-asg \
  --lifecycle-transition autoscaling:EC2_INSTANCE_TERMINATING \
  --heartbeat-timeout 120 \
  --default-result CONTINUE \
  --notification-target-arn "arn:aws:sns:us-east-1:123456789:drain-hook-topic" \
  --role-arn "arn:aws:iam::123456789:role/asg-lifecycle-role"
```

#### Step 6: Traffic Flow During a Breaking News Event

```
Breaking News Published
        │
        ▼
Traffic Spikes: 10,000 → 150,000 req/min
        │
        ▼
ALB: Requests distributed across 5 instances
        │
        ▼
CPU Utilization: 60% → 95% (threshold: 60%)
        │
        ▼
CloudWatch Alarm: ALARM state triggered
        │
        ▼
Target Tracking Policy: Calculates needed instances
        │
        ▼
ASG: Desired