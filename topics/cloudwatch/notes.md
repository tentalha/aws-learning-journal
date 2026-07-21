# CloudWatch

## What is it?

**Amazon CloudWatch** is a fully managed monitoring and observability service provided by AWS under the **Management & Governance** category. It collects, aggregates, and visualizes operational data — including metrics, logs, events, and traces — from AWS resources, applications, and on-premises infrastructure in real time.

CloudWatch acts as the **central nervous system** for AWS operational intelligence, providing:

- **Metrics**: Numerical time-series data points from AWS services and custom applications
- **Logs**: Structured and unstructured log ingestion, storage, and querying via CloudWatch Logs Insights
- **Alarms**: Threshold-based or anomaly-driven notifications and automated actions
- **Dashboards**: Customizable, shareable visual panels for operational data
- **Events / EventBridge**: Near-real-time stream of system events describing changes in AWS resources
- **Container Insights**: Enhanced monitoring for ECS, EKS, and Kubernetes
- **Application Insights**: Automated problem detection for .NET and SQL Server applications
- **Synthetics**: Canary scripts to monitor endpoints and APIs
- **RUM (Real User Monitoring)**: Client-side performance monitoring for web applications
- **Evidently**: Feature flagging and A/B testing with built-in metrics

> **Official Name**: Amazon CloudWatch  
> **Category**: Management & Governance → Monitoring & Observability

---

## Why do we need it?

### The Problem It Solves

In distributed cloud architectures, dozens or hundreds of services run simultaneously. Without centralized observability, teams face:

- **Blind spots**: No visibility into resource utilization, errors, or latency spikes
- **Reactive operations**: Issues are discovered only after customers report them
- **Siloed data**: Logs in one place, metrics in another, traces elsewhere
- **Manual scaling**: No automated response to changing load patterns
- **Compliance gaps**: No audit trail of system behavior over time

### When to Use It

| Scenario | CloudWatch Feature |
|---|---|
| CPU spike causes application slowdown | Metrics + Alarms + Auto Scaling |
| Lambda function throwing errors | Logs + Metric Filters + Alarms |
| API Gateway latency regression | Dashboards + Anomaly Detection |
| Security group change audit | CloudTrail → CloudWatch Logs |
| Scheduled batch job failure | Events/EventBridge Rules |
| End-user page load performance | RUM |
| Synthetic uptime monitoring | Synthetics Canaries |

### Real Business Scenarios

1. **E-commerce platform**: A retail company monitors order processing latency during Black Friday. CloudWatch alarms trigger Auto Scaling when queue depth exceeds thresholds, preventing customer-facing slowdowns.

2. **Healthcare SaaS**: A HIPAA-compliant application centralizes audit logs from 50 microservices into CloudWatch Logs, enabling compliance reporting and anomaly detection.

3. **FinTech startup**: A payment processing API uses CloudWatch Synthetics to run every-minute canary tests against critical payment endpoints, alerting on-call engineers within 60 seconds of degradation.

4. **Gaming company**: Real-time dashboards display active player counts, matchmaking latency, and server health across 12 AWS regions simultaneously.

---

## Internal Working

### Data Collection Pipeline

```
AWS Services / Applications / On-Premises
        │
        ▼
[CloudWatch Agent / API / SDK]
        │
        ▼
[Data Plane - Regional Ingestion Endpoints]
        │
   ┌────┴────┐
   │         │
[Metrics   [Logs
 Store]     Store]
   │         │
   ▼         ▼
[Aggregation & Processing Layer]
        │
        ▼
[Alarms Engine] → [SNS/Lambda/EC2 Auto Scaling/SSM]
        │
        ▼
[Query Engine (Metrics Insights / Logs Insights)]
        │
        ▼
[Dashboards / API / Console]
```

### Metrics Internals

- **Data Points**: Each metric data point consists of a `Namespace`, `MetricName`, `Dimensions`, `Timestamp`, `Value`, and `Unit`.
- **Resolution**: 
  - **Standard resolution**: 1-minute granularity (default for most AWS services)
  - **High resolution**: 1-second granularity (requires custom metrics with `StorageResolution=1`)
- **Aggregation**: CloudWatch uses **statistical aggregation** — when multiple data points arrive in the same period, they are stored as `Sum`, `SampleCount`, `Min`, `Max`, and optionally percentiles.
- **Retention Policy**:
  - Data points with period < 60 seconds: retained **3 hours**
  - Data points with period = 60 seconds (1 minute): retained **15 days**
  - Data points with period = 300 seconds (5 minutes): retained **63 days**
  - Data points with period = 3600 seconds (1 hour): retained **455 days (15 months)**

### Logs Internals

- Logs are organized into **Log Groups** (logical containers) and **Log Streams** (sequences of log events from a single source).
- Ingested log events are **indexed by timestamp** and stored durably in S3-backed storage.
- **Subscription Filters** allow real-time streaming to Kinesis Data Streams, Kinesis Firehose, or Lambda.
- **Metric Filters** parse log data and publish custom metrics — bridging unstructured logs and the metrics system.
- **Logs Insights** uses a purpose-built query language (similar to SQL) with columnar storage for fast analytical queries.

### Alarms Engine

- Alarms evaluate metrics against a **threshold** over a **evaluation period** (number of data points × period length).
- States: `OK`, `ALARM`, `INSUFFICIENT_DATA`
- **M out of N** evaluation: An alarm triggers when M of the last N data points breach the threshold, reducing noise from transient spikes.
- **Anomaly Detection**: Uses ML to create a dynamic band model; alarms fire when metrics fall outside the expected range.
- **Composite Alarms**: Combine multiple alarms using Boolean logic (`AND`, `OR`, `NOT`) to reduce alert fatigue.

---

## Architecture

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                             │
│  EC2 │ Lambda │ RDS │ ECS │ EKS │ API GW │ Custom Apps │ On-Prem│
└──────────────────────┬──────────────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
    [CW Agent]   [AWS SDK/API]  [Service-Native
    [SSM Agent]  [PutMetricData]  Auto-Publishing]
          │            │            │
          └────────────┼────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────────┐
│                   CLOUDWATCH CORE                               │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────────────────┐  │
│  │   METRICS   │  │    LOGS     │  │        EVENTS          │  │
│  │             │  │             │  │    (EventBridge)        │  │
│  │ Namespaces  │  │ Log Groups  │  │                        │  │
│  │ Dimensions  │  │ Log Streams │  │  Rules → Targets       │  │
│  │ Statistics  │  │ Metric      │  │                        │  │
│  │ Math        │  │ Filters     │  │                        │  │
│  └──────┬──────┘  └──────┬──────┘  └───────────────────────┘  │
│         │                │                                      │
│  ┌──────▼────────────────▼──────────────────────────────────┐  │
│  │              ALARMS ENGINE                               │  │
│  │  Static Threshold │ Anomaly Detection │ Composite Alarms │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                              │                                  │
│  ┌───────────────────────────▼───────────────────────────────┐ │
│  │              DASHBOARDS & INSIGHTS                        │ │
│  │  Custom Dashboards │ Logs Insights │ Metrics Insights     │ │
│  │  Container Insights │ Application Insights │ RUM          │ │
│  └───────────────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────────────────┘
                       │
          ┌────────────┼────────────────────┐
          │            │                    │
        [SNS]      [Lambda]          [Auto Scaling]
        [SQS]      [EC2 Actions]     [Systems Manager]
        [PagerDuty] [Step Functions] [OpsCenter]
```

### Key Architectural Components

| Component | Description |
|---|---|
| **Namespaces** | Containers for metrics (e.g., `AWS/EC2`, `AWS/Lambda`, `MyApp/Orders`) |
| **Dimensions** | Key-value pairs that uniquely identify a metric (e.g., `InstanceId=i-1234`) |
| **Log Groups** | Logical grouping of log streams with shared retention and access policies |
| **Log Streams** | Ordered sequence of log events from a single source |
| **Metric Filters** | Patterns applied to log data to extract and publish metrics |
| **Subscription Filters** | Real-time log streaming to external destinations |
| **Alarms** | Watches a metric and transitions states based on rules |
| **Composite Alarms** | Logical combination of multiple alarms |
| **Dashboards** | Cross-account, cross-region visualization panels |
| **Synthetics** | Headless Chromium-based canary scripts |
| **RUM** | JavaScript snippet embedded in web apps |

---

## Real World Example

### Scenario: Auto-Scaling a Web Application Based on Request Latency

**Context**: An e-commerce company runs a Node.js API behind an Application Load Balancer on EC2. During peak hours, response latency degrades. The team wants to automatically scale out EC2 instances when P99 latency exceeds 500ms and scale in when it drops below 200ms.

#### Step-by-Step Walkthrough

**Step 1: Enable Detailed Monitoring on EC2**
```bash
aws ec2 monitor-instances --instance-ids i-1234567890abcdef0
```
This enables 1-minute granularity metrics (vs. 5-minute default).

**Step 2: Configure CloudWatch Agent for Custom Application Metrics**

The Node.js app publishes a custom metric for request latency:
```javascript
// Application code publishes P99 latency every minute
await cloudwatch.putMetricData({
  Namespace: 'MyApp/API',
  MetricData: [{
    MetricName: 'P99Latency',
    Value: p99LatencyMs,
    Unit: 'Milliseconds',
    Dimensions: [{ Name: 'Service', Value: 'OrderAPI' }]
  }]
});
```

**Step 3: Create a Scale-Out Alarm**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "OrderAPI-HighLatency-ScaleOut" \
  --metric-name P99Latency \
  --namespace MyApp/API \
  --dimensions Name=Service,Value=OrderAPI \
  --statistic p99 \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 500 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:autoscaling:us-east-1:123456789:scalingPolicy:policy-id
```

**Step 4: Create a Scale-In Alarm**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "OrderAPI-LowLatency-ScaleIn" \
  --metric-name P99Latency \
  --namespace MyApp/API \
  --dimensions Name=Service,Value=OrderAPI \
  --statistic p99 \
  --period 300 \
  --evaluation-periods 5 \
  --threshold 200 \
  --comparison-operator LessThanThreshold \
  --alarm-actions arn:aws:autoscaling:us-east-1:123456789:scalingPolicy:scale-in-policy-id
```

**Step 5: Create a CloudWatch Dashboard**
- Add widgets for: EC2 instance count, P99 latency, ALB request count, error rate
- Share the dashboard URL with the operations team

**Step 6: Set Up Anomaly Detection (Optional Enhancement)**
```bash
aws cloudwatch put-anomaly-detector \
  --namespace MyApp/API \
  --metric-name P99Latency \
  --dimensions Name=Service,Value=OrderAPI \
  --stat p99
```

**Result**: The system now automatically scales out when latency spikes and scales in during quiet periods, maintaining SLA without manual intervention.

---

## Advantages

1. **Native AWS Integration**: Zero-configuration metrics from 70+ AWS services. EC2, Lambda, RDS, and others publish metrics automatically.

2. **Unified Observability**: Single pane of glass for metrics, logs, traces (via X-Ray integration), events, and synthetics — reducing tool sprawl.

3. **Anomaly Detection**: ML-powered dynamic thresholds eliminate the need to manually tune static alarm thresholds for seasonal or irregular workloads.

4. **Cross-Account and Cross-Region Dashboards**: Consolidate monitoring across multi-account AWS Organizations environments from a single dashboard.

5. **Serverless and Scalable**: No infrastructure to manage. CloudWatch scales automatically with your data volume.

6. **Logs Insights**: Powerful query language with sub-second query performance on terabytes of log data.

7. **Flexible Alarm Actions**: Alarms can trigger SNS, Lambda, EC2 Auto Scaling, EC2 instance actions (stop/start/reboot/terminate), Systems Manager OpsCenter, and more.

8. **Composite Alarms**: Reduce alert fatigue by combining multiple alarms with Boolean logic.

9. **Embedded Metric Format (EMF)**: Structured JSON logs that CloudWatch automatically parses into metrics — ideal for Lambda and containerized workloads.

10. **EventBridge Integration**: Near-real-time event routing to 20+ AWS services and SaaS partners.

11. **CloudWatch Synthetics**: Proactive monitoring with canary scripts before real users are affected.

12. **Container and Application Insights**: Pre-built dashboards and automated problem detection for common application stacks.

---

## Limitations

### Metrics Limitations

| Limit | Value |
|---|---|
| Metrics per namespace | Unlimited (practical: tens of thousands) |
| Custom metrics per account | 10,000 (soft limit, can be increased) |
| Dimensions per metric | 30 |
| API call rate: `PutMetricData` | 1,000 transactions/second per account |
| Data points per `PutMetricData` call | 1,000 |
| Metric name length | 256 characters |
| Dimension name/value length | 256 characters |
| Minimum alarm evaluation period | 10 seconds (high-resolution alarms) |

### Logs Limitations

| Limit | Value |
|---|---|
| Log event size | 256 KB |
| Log group name length | 512 characters |
| Batch size (`PutLogEvents`) | 1 MB or 10,000 events |
| Subscription filters per log group | 2 (can be increased) |
| Metric filters per log group | 100 |
| Maximum log retention | Unlimited (never expire) or 1 day to 10 years |
| Logs Insights query timeout | 15 minutes |
| Logs Insights concurrent queries | 10 |

### Alarm Limitations

| Limit | Value |
|---|---|
| Alarms per account (standard) | 10 alarms (Free Tier), then paid |
| Composite alarm depth | 4 levels of nesting |
| Alarm name length | 255 characters |

### General Limitations

- **Latency**: Metric data can have up to