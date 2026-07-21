# CloudWatch — Interview Questions

---

## Easy

### 1. What is Amazon CloudWatch and what is its primary purpose?

**Answer:**
Amazon CloudWatch is a monitoring and observability service built for DevOps engineers, developers, site reliability engineers (SREs), and IT managers. Its primary purpose is to collect and track metrics, collect and monitor log files, set alarms, and automatically react to changes in your AWS resources. CloudWatch provides a unified view of operational health across AWS services, on-premises servers, and hybrid environments. It enables you to detect anomalous behavior, set alarms, visualize logs and metrics side by side, take automated actions, troubleshoot issues, and discover insights to keep your applications running smoothly.

---

### 2. What is a CloudWatch Metric and what are its key components?

**Answer:**
A CloudWatch Metric is a time-ordered set of data points published to CloudWatch. It represents a variable to monitor over time, such as CPU utilization or disk I/O.

Key components include:
- **Namespace:** A container for metrics (e.g., `AWS/EC2`, `AWS/RDS`). Custom metrics use custom namespaces.
- **Metric Name:** The name of the metric (e.g., `CPUUtilization`).
- **Dimensions:** Name/value pairs that uniquely identify a metric (e.g., `InstanceId=i-1234567890abcdef0`). Up to 30 dimensions per metric.
- **Timestamp:** The time the data point was recorded.
- **Value:** The actual numeric data point.
- **Unit:** The unit of measurement (e.g., Percent, Bytes, Count).
- **Resolution:** Standard (1-minute granularity) or High-Resolution (1-second granularity).

---

### 3. What is a CloudWatch Alarm and what states can it be in?

**Answer:**
A CloudWatch Alarm watches a single metric or the result of a math expression based on CloudWatch metrics over a specified time period. When the metric breaches a defined threshold, the alarm takes an action.

An alarm can be in one of three states:
- **OK:** The metric is within the defined threshold.
- **ALARM:** The metric has breached the defined threshold.
- **INSUFFICIENT_DATA:** The alarm has just started, the metric is not available, or there is not enough data to determine the alarm state.

Common actions triggered by alarms include:
- Sending notifications via Amazon SNS
- Performing EC2 actions (stop, terminate, reboot, recover)
- Triggering Auto Scaling actions
- Invoking Systems Manager OpsCenter OpsItems

---

### 4. What is the difference between CloudWatch Logs and CloudWatch Metrics?

**Answer:**
| Feature | CloudWatch Logs | CloudWatch Metrics |
|---|---|---|
| **Data Type** | Text-based log events (unstructured/structured) | Numeric time-series data points |
| **Purpose** | Store, monitor, and analyze log files | Track performance and operational data |
| **Examples** | Application logs, VPC Flow Logs, Lambda logs | CPU utilization, request count, latency |
| **Retention** | Configurable (1 day to indefinite) | 15 months by default |
| **Querying** | CloudWatch Logs Insights (SQL-like syntax) | Metric math, statistics |
| **Granularity** | Per log event with timestamp | 1-second to 1-minute intervals |

Logs are the raw textual output of your systems, while Metrics are numerical aggregations derived from system behavior. You can create **Metric Filters** on log groups to extract metric data from log events.

---

### 5. What is the default metric retention period in CloudWatch?

**Answer:**
CloudWatch retains metric data according to the following schedule based on the metric's resolution:

- **High-resolution metrics (< 60 seconds):** Retained for **3 hours**
- **1-minute data points:** Retained for **15 days**
- **5-minute data points:** Retained for **63 days**
- **1-hour data points:** Retained for **15 months (455 days)**

As data ages, it is aggregated and stored at coarser resolutions. For example, data available at 1-minute resolution becomes available only at 5-minute resolution after 15 days. This means you always have up to 15 months of historical data, but older data has lower granularity. For longer retention, you should export metrics to Amazon S3 or use a third-party solution.

---

## Medium

### 1. Explain CloudWatch Logs Insights and how it differs from basic log filtering.

**Answer:**
**CloudWatch Logs Insights** is an interactive, pay-per-query log analytics tool that allows you to search and analyze log data stored in CloudWatch Logs using a purpose-built query language.

**Key capabilities:**
- **Query Language:** Supports commands like `fields`, `filter`, `stats`, `sort`, `limit`, `parse`, and `pattern`. Example:
  ```
  fields @timestamp, @message
  | filter @message like /ERROR/
  | stats count(*) as errorCount by bin(5m)
  | sort errorCount desc
  ```
- **Multi-log group queries:** Query up to 50 log groups simultaneously.
- **Automatic field discovery:** Automatically discovers fields in JSON log events.
- **Visualization:** Results can be visualized as bar charts or line graphs.
- **Saved queries:** Frequently used queries can be saved.

**Differences from basic log filtering:**
| Feature | Basic Log Filtering | Logs Insights |
|---|---|---|
| Aggregation | No | Yes (count, sum, avg, percentiles) |
| Multi-group | No | Yes (up to 50 groups) |
| Visualization | No | Yes |
| Performance | Slow on large datasets | Optimized, parallel execution |
| Syntax | Simple pattern matching | Rich query language |

**Pricing:** Charged per GB of log data scanned, so query optimization (time range, specific log groups) reduces cost.

---

### 2. What are CloudWatch Dashboards and what are their limitations?

**Answer:**
**CloudWatch Dashboards** are customizable home pages in the CloudWatch console that you can use to monitor your resources in a single view, even across different AWS Regions and accounts.

**Features:**
- Support multiple widget types: Line graphs, stacked area charts, number widgets, bar charts, text widgets (Markdown), alarm status widgets, and logs tables.
- **Cross-account and cross-region:** A single dashboard can display metrics from multiple AWS accounts and regions.
- **Automatic dashboards:** Pre-built dashboards for common AWS services (EC2, Lambda, etc.).
- **Sharing:** Dashboards can be shared publicly or with specific users without requiring AWS account access.
- **Animated:** Playback historical data using the time range selector.

**Limitations:**
- **Maximum 500 dashboards** per AWS account.
- **Maximum 2,500 metrics** per dashboard.
- Up to **100 widgets** per dashboard.
- Dashboard refresh rate minimum is **10 seconds** (for high-resolution metrics) or **1 minute** for standard.
- Cross-account dashboards require **CloudWatch cross-account observability** to be configured.
- No built-in alerting from dashboards — alarms must be configured separately.
- No native drill-down capability (cannot click a metric spike to automatically see correlated logs).

---

### 3. Explain the concept of CloudWatch Metric Math and provide a practical use case.

**Answer:**
**CloudWatch Metric Math** allows you to perform mathematical operations on CloudWatch metrics to create new time-series data. You can use these derived metrics in graphs and alarms without storing them as separate metrics.

**Supported operations:**
- Arithmetic: `+`, `-`, `*`, `/`
- Statistical functions: `AVG()`, `SUM()`, `MIN()`, `MAX()`, `STDDEV()`
- Time-series functions: `RATE()`, `DIFF()`, `FILL()`, `RUNNING_SUM()`
- Anomaly detection: `ANOMALY_DETECTION_BAND()`
- Search functions: `SEARCH()`, `METRICS()`

**Practical Use Case — Calculating Error Rate:**
Suppose you have an API Gateway with metrics for total requests (`RequestCount`) and `5XXError` count. You want to alert when the error rate exceeds 5%:

```
# In CloudWatch Alarms with Metric Math:
m1 = AWS/ApiGateway RequestCount (Sum, 1 min)
m2 = AWS/ApiGateway 5XXError (Sum, 1 min)
e1 = (m2 / m1) * 100   # Error rate as percentage
```

Create an alarm on `e1` with threshold > 5.

**Another Use Case — Aggregating across instances:**
```
# Total CPU across all EC2 instances in Auto Scaling Group
SEARCH('{AWS/EC2,AutoScalingGroupName} MetricName="CPUUtilization"', 'Average', 300)
```

Then use `AVG(METRICS())` to get the average CPU across all returned metrics.

Metric Math expressions are evaluated at query time and do not incur additional storage costs.

---

### 4. What is CloudWatch Container Insights and how does it work?

**Answer:**
**CloudWatch Container Insights** is a monitoring and observability solution for containerized applications and microservices running on Amazon ECS, Amazon EKS, Kubernetes on EC2, and AWS Fargate. It collects, aggregates, and summarizes metrics and logs from your containerized applications.

**How it works:**

1. **Data Collection Agent:**
   - For **ECS:** Uses the CloudWatch Agent as a sidecar container or daemon service.
   - For **EKS/Kubernetes:** Deploys the **CloudWatch Agent** and **Fluent Bit** as DaemonSets. The agent collects metrics; Fluent Bit forwards logs.
   - For **Fargate:** Uses a sidecar container pattern since DaemonSets aren't supported.

2. **Metrics collected:**
   - CPU and memory utilization at cluster, node, pod, task, and service levels
   - Network I/O, disk I/O
   - Container restart counts
   - Kubernetes-specific: pending pod counts, failed node counts

3. **Log collection:**
   - Application logs, performance logs, and Kubernetes control plane logs (API server, scheduler, etc.)

4. **Embedded Metric Format (EMF):**
   - Container Insights uses EMF — a JSON-structured log format that CloudWatch automatically extracts into metrics. This enables high-cardinality metrics without pre-aggregation.

5. **Pre-built dashboards:**
   - Automatic dashboards at cluster, node, pod, and service levels.

**Enabling on EKS:**
```bash
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability \
  --service-account-role-arn arn:aws:iam::ACCOUNT_ID:role/CloudWatchAgentRole
```

**Cost consideration:** Container Insights generates a high volume of custom metrics and logs, which can significantly increase CloudWatch costs in large clusters.

---

### 5. Explain CloudWatch Anomaly Detection and how it can be used in alarms.

**Answer:**
**CloudWatch Anomaly Detection** applies machine learning algorithms to continuously analyze a metric's historical data, determine a normal baseline, and surface anomalies with minimal user configuration.

**How it works:**
1. CloudWatch trains a model using up to **two weeks** of historical metric data.
2. The model accounts for **time-of-day patterns**, **day-of-week patterns**, and **seasonal trends**.
3. It generates a band (upper and lower bounds) representing expected values. The width of the band is controlled by a **standard deviation multiplier** you specify.
4. Data points outside the band are considered anomalies.

**Using Anomaly Detection in Alarms:**
Instead of setting a static threshold, you configure the alarm to trigger when the metric falls outside the anomaly detection band:

```
Alarm Condition: 
  Metric: AWS/ApplicationELB TargetResponseTime
  Anomaly Detection Model: ANOMALY_DETECTION_BAND(m1, 2)
  Alarm when: m1 > ANOMALY_DETECTION_BAND upper bound
  For: 3 consecutive 1-minute periods
```

The `2` is the number of standard deviations — higher values produce a wider band (fewer false alarms).

**Excluding periods from training:**
You can exclude known anomalous periods (deployments, maintenance windows) so they don't skew the model:
```bash
aws cloudwatch put-anomaly-detector \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=my-function \
  --configuration ExcludedTimeRanges=[{StartTime=2024-01-01T00:00:00Z,EndTime=2024-01-01T06:00:00Z}]
```

**Benefits over static thresholds:**
- No need to know "normal" upfront
- Automatically adjusts to changing traffic patterns
- Reduces false positives during predictable load changes (e.g., business hours vs. nights)

**Limitation:** Requires sufficient historical data; new metrics may need up to 15 minutes of training before the model is usable.

---

## Hard

### 1. Deep-dive into CloudWatch Logs architecture: Log Groups, Log Streams, Metric Filters, and Subscription Filters. How do they interact at scale?

**Answer:**
**Hierarchical Architecture:**

```
CloudWatch Logs
└── Log Group (e.g., /aws/lambda/my-function)
    ├── Log Stream (e.g., 2024/01/15/[$LATEST]abc123)
    │   ├── Log Event { timestamp, message }
    │   └── Log Event { timestamp, message }
    ├── Metric Filter → CloudWatch Metric
    └── Subscription Filter → Kinesis / Lambda / Firehose
```

**Log Groups:**
- Logical container for log streams sharing the same retention, metric filters, and subscription filters.
- Retention policies: 1 day to 10 years, or never expire (default).
- **Resource policies** control cross-account access.
- KMS encryption can be applied at the log group level.
- **Log class:** Standard (full indexing) or **Infrequent Access** (50% cheaper, limited features — no Logs Insights, no metric filters).

**Log Streams:**
- Sequence of log events from the same source (e.g., one Lambda container, one EC2 instance).
- No size limit per stream, but individual log events are limited to **256 KB**.
- `PutLogEvents` API requires a **sequence token** (except with IAM `logs:PutLogEvents` with `aws:RequestedRegion` condition). Concurrent writes from multiple threads require separate streams to avoid `InvalidSequenceTokenException`.
- **Batch limits:** Each `PutLogEvents` call can contain up to **10,000 log events** or **1 MB** of data.

**Metric Filters:**
- Scan incoming log events in real-time using filter patterns.
- **Filter patterns:** Simple keyword matching (`ERROR`), JSON patterns (`{ $.level = "ERROR" }`), or space-delimited patterns.
- Extract up to **3 dimensions** and a **metric value** from matched events.
- **Important limitation:** Metric filters only process new log events after the filter is created — they are **not retroactive**.
- Maximum **100 metric filters per log group**.
- Metric filters have a **default value** (usually 0) that is published when no matching events occur, preventing gaps in metric data.

**Subscription Filters:**
- Forward matching log events in near real-time to a destination.
- Supported destinations:
  - **Amazon Kinesis Data Streams** (cross-account supported)
  - **Amazon Kinesis Data Firehose** (cross-account supported)
  - **AWS Lambda** (same account only, unless using resource-based policy)
  - **Amazon OpenSearch Service** (via Lambda)
- Maximum **2 subscription filters per log group** (soft limit, can be increased).
- **Cross-account log sharing:** Uses a destination ARN with a destination policy granting `PutSubscriptionFilter` permission.

**Scale Considerations:**
- **Throttling:** `PutLogEvents` is throttled at **5 requests/second per log stream** and **1,500 transactions/second per account per region**.
- **Kinesis fan-out:** For high-throughput scenarios, use multiple Kinesis shards. Each shard handles 1 MB/s or 1,000 records/s.
- **CloudWatch Logs Agent vs. CloudWatch Agent:** The newer unified CloudWatch Agent supports structured log collection, custom namespaces, and buffering, making it more efficient at scale.
- **Embedded Metric Format (EMF):** For Lambda and containers, EMF allows writing metrics as structured JSON logs, bypassing the `PutMetricData` API limits (150