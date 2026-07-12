# Auto Scaling — Interview Questions

---

## Easy

### Q1. What is AWS Auto Scaling and what problem does it solve?

**Answer:**
AWS Auto Scaling is a service that automatically adjusts the number of compute resources (EC2 instances, ECS tasks, DynamoDB capacity, etc.) in response to changing demand. It solves the core problem of **capacity management** — ensuring you have enough resources to handle peak load without paying for idle capacity during low-traffic periods. It provides both **high availability** (by replacing unhealthy instances) and **cost optimization** (by scaling in when demand drops).

---

### Q2. What are the three types of Auto Scaling policies?

**Answer:**
1. **Simple Scaling** — Triggers a scaling action based on a single CloudWatch alarm. After the action completes, a cooldown period must expire before another action can occur.
2. **Step Scaling** — Triggers scaling actions based on the *magnitude* of the alarm breach. Different step adjustments can be applied depending on how far the metric exceeds the threshold (e.g., add 1 instance if CPU > 60%, add 3 instances if CPU > 80%).
3. **Target Tracking Scaling** — Automatically adjusts capacity to keep a specific metric (e.g., CPU utilization, ALB request count per target) at a defined target value. AWS manages the CloudWatch alarms automatically.

> **Bonus:** There is also **Scheduled Scaling**, which scales based on a known time pattern (e.g., scale up every weekday at 8 AM).

---

### Q3. What is a Launch Template and how does it differ from a Launch Configuration?

**Answer:**

| Feature | Launch Configuration | Launch Template |
|---|---|---|
| Versioning | Not supported | Supported (multiple versions) |
| Spot + On-Demand mix | Not supported | Supported |
| T2/T3 Unlimited | Not supported | Supported |
| AWS recommendation | Legacy | **Recommended** |
| Modification | Immutable | New versions can be created |

A **Launch Template** is the modern, recommended way to define the configuration for instances launched by an Auto Scaling group (AMI ID, instance type, key pair, security groups, user data, etc.). It supports versioning and a richer feature set. Launch Configurations are legacy and AWS no longer adds new features to them.

---

### Q4. What is a Cooldown Period in Auto Scaling?

**Answer:**
A **cooldown period** is a configurable wait time (default: **300 seconds**) after a scaling activity completes, during which Auto Scaling does not launch or terminate additional instances. This prevents Auto Scaling from launching or terminating instances before the effects of a previous scaling action have had time to take effect (e.g., waiting for a new instance to fully start and begin handling traffic before deciding if more instances are needed). Cooldown periods apply to **Simple Scaling** policies. Step Scaling and Target Tracking policies use **instance warmup** instead.

---

### Q5. What is the difference between horizontal and vertical scaling, and which does Auto Scaling primarily use?

**Answer:**
- **Horizontal Scaling (Scale Out/In):** Adding or removing instances/nodes. This is what AWS Auto Scaling primarily implements. It is preferred for cloud-native architectures because it provides better fault tolerance and near-linear capacity increases.
- **Vertical Scaling (Scale Up/Down):** Increasing or decreasing the size/power of an existing instance (e.g., moving from `t3.medium` to `t3.xlarge`). This typically requires downtime and has an upper limit.

AWS Auto Scaling primarily uses **horizontal scaling**. Vertical scaling on AWS is typically done manually or through automation scripts and requires stopping/starting instances.

---

## Medium

### Q1. Explain the difference between the `DesiredCapacity`, `MinSize`, and `MaxSize` parameters in an Auto Scaling Group.

**Answer:**
These three parameters define the capacity boundaries of an Auto Scaling Group (ASG):

- **`MinSize`**: The minimum number of instances that must always be running. Auto Scaling will never scale below this value, even if a scale-in policy triggers. This ensures baseline availability.
- **`MaxSize`**: The maximum number of instances the ASG can launch. This acts as a cost control guardrail, preventing runaway scaling. Auto Scaling will never exceed this value.
- **`DesiredCapacity`**: The number of instances Auto Scaling tries to maintain at any given time. It must always satisfy `MinSize ≤ DesiredCapacity ≤ MaxSize`. When a scaling policy fires, it changes the `DesiredCapacity`, and the ASG reconciles the actual number of running instances to match it.

**Example:** If `Min=2`, `Desired=4`, `Max=10`:
- Auto Scaling maintains 4 instances.
- If 2 instances fail health checks, they are replaced (not scaled below 2).
- A scale-out policy can increase `Desired` up to 10.
- A scale-in policy can decrease `Desired` down to 2.

---

### Q2. How does Auto Scaling handle instance health checks, and what are the two types?

**Answer:**
Auto Scaling continuously monitors instance health and replaces unhealthy instances automatically. There are two types of health checks:

1. **EC2 Health Checks (Default):** Auto Scaling checks the EC2 instance status (system status checks and instance status checks). An instance is considered unhealthy if it is in any state other than `running` (e.g., `stopped`, `terminated`, `impaired`).

2. **ELB Health Checks:** When an ASG is attached to a load balancer, you can enable ELB health checks. These check whether the load balancer considers the instance healthy (i.e., passing the target group health check). This is more application-aware — an instance could be running but failing health checks because the application is unresponsive.

**Health Check Grace Period:** A configurable period (default: 0 seconds, recommended: set to match your application startup time) during which Auto Scaling does not perform health checks on a newly launched instance. This prevents premature termination of instances that are still initializing.

**Best Practice:** Always enable ELB health checks when using a load balancer, and set an appropriate grace period to match your application's startup time.

---

### Q3. What is a Lifecycle Hook in Auto Scaling, and provide a practical use case?

**Answer:**
A **Lifecycle Hook** pauses an instance during an Auto Scaling scale-out or scale-in event and puts it into a wait state (`Pending:Wait` or `Terminating:Wait`). This allows you to perform custom actions before the instance is put into service or terminated.

**How it works:**
1. Auto Scaling triggers a scale-out event.
2. Instance enters `Pending:Wait` state.
3. Auto Scaling sends a notification to an SNS topic, SQS queue, or EventBridge.
4. Your custom process performs its action (e.g., install software, drain connections).
5. Your process calls `CompleteLifecycleAction` (success) or the timeout expires (configurable, default 1 hour, max 48 hours).
6. Instance transitions to `InService`.

**Practical Use Cases:**
- **Scale-Out:** Install configuration management agents (Chef, Puppet), pull secrets from AWS Secrets Manager, run custom bootstrap scripts, register the instance with a service discovery system, warm up application caches.
- **Scale-In:** Gracefully drain in-flight requests, deregister from service discovery, flush logs to S3 or CloudWatch, complete ongoing database transactions, push final metrics.

**Example Architecture:**
```
ASG Scale-In → Lifecycle Hook → SQS → Lambda → Drain connections → CompleteLifecycleAction → Instance Terminated
```

---

### Q4. Explain the Termination Policy in Auto Scaling. What is the default policy and how can it be customized?

**Answer:**
When Auto Scaling needs to scale in, it must decide which instance(s) to terminate. The **termination policy** defines this logic.

**Default Termination Policy (evaluated in order):**
1. Select the Availability Zone with the most instances (for balance).
2. Within that AZ, find instances using the oldest Launch Template or Launch Configuration.
3. If there's a tie, terminate the instance closest to the next billing hour (legacy; less relevant with per-second billing).
4. If still tied, select randomly.

**Available Termination Policies:**
| Policy | Behavior |
|---|---|
| `Default` | As described above |
| `OldestInstance` | Terminates the oldest instance (good for rolling updates) |
| `NewestInstance` | Terminates the newest instance (good for testing new configurations) |
| `OldestLaunchTemplate` | Terminates instances using the oldest launch template version |
| `OldestLaunchConfiguration` | Legacy equivalent |
| `ClosestToNextInstanceHour` | Minimizes cost |
| `AllocationStrategy` | Optimizes for Spot instance pools |

**Custom Termination Policies:** You can specify multiple policies in a list; they are evaluated in order. You can also use **instance scale-in protection** to mark specific instances as protected from scale-in, which is useful for instances running stateful workloads.

---

### Q5. What is the difference between Dynamic Scaling and Predictive Scaling?

**Answer:**

| Aspect | Dynamic Scaling | Predictive Scaling |
|---|---|---|
| **Trigger** | Reacts to real-time metric changes | Proactively scales based on ML-predicted future load |
| **Latency** | Has inherent lag (metric alarm → scale → instance boot) | Scales *before* load arrives |
| **Best For** | Unpredictable, reactive workloads | Workloads with recurring, predictable patterns |
| **Data Required** | Real-time CloudWatch metrics | At least 14 days of historical metric data |
| **Modes** | Simple, Step, Target Tracking | Forecast Only, Forecast and Scale |

**Dynamic Scaling** responds to current conditions. There is always a delay between load increasing and new instances becoming available (typically 5–15 minutes including alarm evaluation, instance launch, and warmup).

**Predictive Scaling** uses machine learning to analyze historical CloudWatch metrics and forecasts future capacity needs. It proactively adjusts `DesiredCapacity` up to 60 minutes before predicted load spikes. It works best for workloads with daily or weekly patterns (e.g., business-hours traffic, Monday morning spikes).

**Best Practice:** Combine both — use Predictive Scaling to handle known patterns proactively, and Dynamic Scaling (Target Tracking) as a safety net for unexpected spikes.

---

## Hard

### Q1. How does Auto Scaling handle Spot Instances, and what strategies can minimize interruption impact?

**Answer:**
Auto Scaling integrates deeply with the Spot Instance model through **Mixed Instances Policy** and **EC2 Fleet** strategies.

**Mixed Instances Policy Configuration:**
```json
{
  "MixedInstancesPolicy": {
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 20,
      "SpotAllocationStrategy": "capacity-optimized",
      "SpotInstancePools": 4
    },
    "LaunchTemplate": {
      "Overrides": [
        {"InstanceType": "m5.large"},
        {"InstanceType": "m5a.large"},
        {"InstanceType": "m4.large"},
        {"InstanceType": "c5.large"}
      ]
    }
  }
}
```

**Spot Allocation Strategies:**
- **`lowest-price`**: Launches from the cheapest pool. Higher interruption risk.
- **`capacity-optimized`**: Launches from the pool with the most available capacity. **Recommended** — reduces interruption rates significantly.
- **`capacity-optimized-prioritized`**: Like `capacity-optimized` but respects instance type priority order.
- **`price-capacity-optimized`**: Balances between lowest price and highest capacity (newest recommended strategy).

**Strategies to Minimize Interruption Impact:**
1. **Diversify across instance types and AZs:** Use 6–10 different instance types. Spot interruptions are pool-specific.
2. **Spot Instance Interruption Notices:** AWS provides a 2-minute warning via EC2 instance metadata (`/latest/meta-data/spot/termination-time`) and EventBridge. Use Lifecycle Hooks to gracefully drain.
3. **Maintain On-Demand baseline:** Set `OnDemandBaseCapacity` to ensure critical minimum capacity is never interrupted.
4. **Use `capacity-optimized` strategy:** Statistically proven to reduce interruptions by up to 50%.
5. **Rebalance Recommendations:** Enable **Capacity Rebalancing** — Auto Scaling proactively replaces Spot instances at elevated interruption risk *before* they are interrupted, launching a replacement first.
6. **Application-level resilience:** Design stateless applications, use SQS for work queuing, and implement graceful shutdown handlers.

---

### Q2. Describe the internals of Target Tracking Scaling and explain how AWS manages the CloudWatch alarms behind the scenes.

**Answer:**
Target Tracking Scaling is the most sophisticated and recommended dynamic scaling policy. Understanding its internals is critical for advanced usage.

**How It Works Internally:**

When you create a Target Tracking policy targeting, for example, 50% average CPU utilization, AWS automatically creates **two CloudWatch alarms**:

1. **Scale-Out Alarm:** Triggers when the metric is *above* the target for a sustained period. The threshold is set slightly above the target to avoid oscillation.
2. **Scale-In Alarm:** Triggers when the metric is *below* the target. Scale-in has a **longer evaluation period** (typically 15 minutes vs. ~3 minutes for scale-out) to prevent aggressive scale-in that would then require immediate scale-out.

**The Control Algorithm:**
Auto Scaling uses a proportional scaling algorithm:
```
Required Capacity = Ceil(Current Capacity × (Current Metric Value / Target Value))
```
Example: 10 instances, CPU at 75%, target 50%:
```
Required = Ceil(10 × (75/50)) = Ceil(15) = 15 instances
```

**Key Behaviors:**
- You **cannot manually modify** the auto-created CloudWatch alarms. Doing so will cause undefined behavior.
- If multiple Target Tracking policies exist on an ASG, Auto Scaling uses the policy that provides the **largest** desired capacity (scale-out) or the **smallest** (scale-in). This ensures all targets are respected.
- **Scale-in is disabled** when the metric is unavailable (e.g., no healthy instances to report metrics) to prevent accidental scale-in during outages.
- **Instance Warmup:** New instances don't contribute their metrics to the aggregate until the warmup period expires. This prevents premature scale-out triggered by partially-warmed instances.

**Predefined Metrics Available:**
- `ASGAverageCPUUtilization`
- `ASGAverageNetworkIn` / `ASGAverageNetworkOut`
- `ALBRequestCountPerTarget`

**Custom Metrics:** You can use any CloudWatch metric with Target Tracking, but the metric must scale proportionally with instance count (e.g., "requests per instance" — not total requests, which wouldn't decrease as you scale out).

---

### Q3. How does Auto Scaling integrate with Application Load Balancers for zero-downtime deployments, and what are the failure modes to watch for?

**Answer:**
The ASG–ALB integration is fundamental for production deployments. Understanding the full request lifecycle and failure modes is critical.

**Registration/Deregistration Flow:**

**Scale-Out:**
```
Instance Launched → Registered with Target Group → Health Checks Begin →
[Grace Period] → Health Check Passes → Instance enters InService →
ALB begins routing traffic
```

**Scale-In:**
```
Scale-In Triggered → ALB begins Connection Draining (Deregistration Delay) →
[Default 300s] → Existing connections complete or timeout →
Instance deregistered → ASG terminates instance
```

**Critical Configuration Points:**

1. **Deregistration Delay (Connection Draining):** Default 300 seconds. During this period, the ALB stops sending *new* requests to the deregistering instance but allows *in-flight* requests to complete. Set this to match your longest expected request duration. For microservices with short requests, reduce to 30–60 seconds to speed up deployments.

2. **Health Check Grace Period vs. ALB Health Check Interval:**
   - Grace Period must be long enough for the application to start.
   - ALB health check interval + unhealthy threshold determines how quickly a failing instance is detected.
   - If Grace Period < application startup time, healthy instances will be terminated prematurely.

3. **Slow Start Mode:** ALB feature that gradually increases traffic to newly registered targets over a configurable duration (30–900 seconds). Prevents overwhelming new instances that are still warming up caches or JIT-compiling code.

**Failure