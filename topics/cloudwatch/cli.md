# CloudWatch — AWS CLI Commands

## Setup & Configuration

### Prerequisites

- AWS CLI v2 installed (`aws --version`)
- AWS credentials configured (`aws configure` or environment variables)
- Appropriate IAM permissions attached to your user/role

### Required IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "cloudwatch:PutMetricAlarm",
        "cloudwatch:DescribeAlarms",
        "cloudwatch:DeleteAlarms",
        "cloudwatch:EnableAlarmActions",
        "cloudwatch:DisableAlarmActions",
        "cloudwatch:PutDashboard",
        "cloudwatch:GetDashboard",
        "cloudwatch:ListDashboards",
        "cloudwatch:DeleteDashboards",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:GetLogEvents",
        "logs:FilterLogEvents",
        "logs:PutMetricFilter",
        "logs:DeleteLogGroup"
      ],
      "Resource": "*"
    }
  ]
}
```

### CLI Configuration

```bash
# Configure default region
aws configure set region us-east-1

# Verify identity
aws sts get-caller-identity

# Set output format to table for readable output
aws configure set output table

# Use a named profile for CloudWatch operations
aws configure --profile cloudwatch-admin
export AWS_PROFILE=cloudwatch-admin
```

---

## Core Commands

### 1. List All Available Metrics

```bash
aws cloudwatch list-metrics \
  --namespace "AWS/EC2" \
  --metric-name "CPUUtilization"
```

**What it does:** Lists all metrics available in the specified namespace. Useful for discovering what metrics exist before querying or creating alarms.

**Example Output:**
```json
{
    "Metrics": [
        {
            "Namespace": "AWS/EC2",
            "MetricName": "CPUUtilization",
            "Dimensions": [
                {
                    "Name": "InstanceId",
                    "Value": "i-0abc123def456789"
                }
            ]
        }
    ]
}
```

---

### 2. Get Metric Statistics

```bash
aws cloudwatch get-metric-statistics \
  --namespace "AWS/EC2" \
  --metric-name "CPUUtilization" \
  --dimensions Name=InstanceId,Value=i-0abc123def456789 \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z \
  --period 3600 \
  --statistics Average Maximum Minimum
```

**What it does:** Retrieves statistical data points for a specific metric over a defined time range. `--period` is in seconds (3600 = 1 hour).

**Example Output:**
```json
{
    "Label": "CPUUtilization",
    "Datapoints": [
        {
            "Timestamp": "2024-01-15T12:00:00+00:00",
            "Average": 23.45,
            "Maximum": 87.32,
            "Minimum": 2.10,
            "Unit": "Percent"
        }
    ]
}
```

---

### 3. Put Custom Metric Data

```bash
aws cloudwatch put-metric-data \
  --namespace "MyApp/Performance" \
  --metric-name "RequestLatency" \
  --value 245.5 \
  --unit Milliseconds \
  --dimensions Name=Environment,Value=production Name=Service,Value=api-gateway
```

**What it does:** Publishes custom metric data to CloudWatch. Useful for application-level metrics not automatically collected by AWS.

---

### 4. Create a Metric Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPUUtilization-prod-web-01" \
  --alarm-description "Alarm when CPU exceeds 80% for 5 minutes" \
  --metric-name "CPUUtilization" \
  --namespace "AWS/EC2" \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=i-0abc123def456789 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-alert-topic \
  --ok-actions arn:aws:sns:us-east-1:123456789012:my-alert-topic \
  --treat-missing-data breaching
```

**What it does:** Creates or updates a CloudWatch alarm that triggers an SNS notification when CPU exceeds 80% for two consecutive 5-minute periods.

---

### 5. Describe Alarms

```bash
aws cloudwatch describe-alarms \
  --alarm-names "HighCPUUtilization-prod-web-01" \
  --state-value ALARM
```

**What it does:** Returns detailed configuration and state information for one or more alarms. Filter by state: `OK`, `ALARM`, or `INSUFFICIENT_DATA`.

**Example Output:**
```json
{
    "MetricAlarms": [
        {
            "AlarmName": "HighCPUUtilization-prod-web-01",
            "AlarmArn": "arn:aws:cloudwatch:us-east-1:123456789012:alarm:HighCPUUtilization-prod-web-01",
            "StateValue": "ALARM",
            "StateReason": "Threshold Crossed: 2 datapoints [85.3, 82.1] were greater than the threshold (80.0).",
            "MetricName": "CPUUtilization",
            "Threshold": 80.0
        }
    ]
}
```

---

### 6. Delete an Alarm

```bash
aws cloudwatch delete-alarms \
  --alarm-names "HighCPUUtilization-prod-web-01" "LowDiskSpace-prod-db-01"
```

**What it does:** Permanently deletes one or more CloudWatch alarms. Accepts multiple alarm names in a single call.

---

### 7. Create a CloudWatch Log Group

```bash
aws logs create-log-group \
  --log-group-name "/myapp/production/application" \
  --tags Environment=production,Application=myapp,Team=platform
```

**What it does:** Creates a new CloudWatch Logs log group with optional tags for cost allocation and organization.

---

### 8. Create a Log Stream

```bash
aws logs create-log-stream \
  --log-group-name "/myapp/production/application" \
  --log-stream-name "web-server-01/2024/01/15"
```

**What it does:** Creates a log stream within an existing log group. Log streams typically represent individual sources (e.g., a specific EC2 instance or container).

---

### 9. Put Log Events

```bash
aws logs put-log-events \
  --log-group-name "/myapp/production/application" \
  --log-stream-name "web-server-01/2024/01/15" \
  --log-events \
    timestamp=1705276800000,message="Application started successfully" \
    timestamp=1705276860000,message="Received 1500 requests in last 60 seconds"
```

**What it does:** Sends log events to a CloudWatch log stream. Timestamps must be in milliseconds since Unix epoch.

---

### 10. Filter Log Events

```bash
aws logs filter-log-events \
  --log-group-name "/myapp/production/application" \
  --filter-pattern "ERROR" \
  --start-time 1705276800000 \
  --end-time 1705363200000 \
  --limit 50
```

**What it does:** Searches log events across all streams in a log group matching a filter pattern. Supports CloudWatch Logs filter syntax including `ERROR`, `[level=ERROR]`, or `{ $.level = "ERROR" }` for JSON logs.

**Example Output:**
```json
{
    "events": [
        {
            "logStreamName": "web-server-01/2024/01/15",
            "timestamp": 1705280400000,
            "message": "ERROR: Database connection timeout after 30s",
            "ingestionTime": 1705280401234,
            "eventId": "37167989060827136"
        }
    ]
}
```

---

### 11. Describe Log Groups

```bash
aws logs describe-log-groups \
  --log-group-name-prefix "/myapp/production" \
  --limit 20
```

**What it does:** Lists log groups matching the specified prefix. Useful for auditing log group configurations and retention settings.

---

### 12. Set Log Group Retention Policy

```bash
aws logs put-retention-policy \
  --log-group-name "/myapp/production/application" \
  --retention-in-days 90
```

**What it does:** Sets the number of days log events are retained. Valid values: 1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1096, 1827, 2192, 2557, 2922, 3288, 3653.

---

### 13. Create a Dashboard

```bash
aws cloudwatch put-dashboard \
  --dashboard-name "ProductionOverview" \
  --dashboard-body file://dashboard-body.json
```

**dashboard-body.json:**
```json
{
    "widgets": [
        {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "metrics": [
                    ["AWS/EC2", "CPUUtilization", "InstanceId", "i-0abc123def456789"]
                ],
                "period": 300,
                "stat": "Average",
                "region": "us-east-1",
                "title": "EC2 CPU Utilization"
            }
        }
    ]
}
```

**What it does:** Creates or updates a CloudWatch dashboard using a JSON body definition. Dashboards can contain metric graphs, alarms, text widgets, and more.

---

### 14. Get Metric Data (Advanced Query)

```bash
aws cloudwatch get-metric-data \
  --metric-data-queries '[
    {
      "Id": "cpu_avg",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/EC2",
          "MetricName": "CPUUtilization",
          "Dimensions": [{"Name": "InstanceId", "Value": "i-0abc123def456789"}]
        },
        "Period": 300,
        "Stat": "Average"
      },
      "Label": "CPU Average"
    }
  ]' \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z
```

**What it does:** More powerful than `get-metric-statistics` — supports multiple metrics, math expressions, and cross-account queries in a single API call.

---

### 15. Describe Alarm History

```bash
aws cloudwatch describe-alarm-history \
  --alarm-name "HighCPUUtilization-prod-web-01" \
  --history-item-type StateUpdate \
  --start-date 2024-01-01 \
  --end-date 2024-01-31
```

**What it does:** Returns historical state changes, configuration updates, and action executions for an alarm. Essential for post-incident analysis.

---

## Common Operations

### Create Operations

```bash
# Create a composite alarm (alarm of alarms)
aws cloudwatch put-composite-alarm \
  --alarm-name "ProductionCritical-Composite" \
  --alarm-description "Fires when any critical production alarm triggers" \
  --alarm-rule "ALARM(\"HighCPUUtilization-prod-web-01\") OR ALARM(\"LowDiskSpace-prod-db-01\")" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:pagerduty-critical

# Create a metric filter on a log group
aws logs put-metric-filter \
  --log-group-name "/myapp/production/application" \
  --filter-name "ErrorCount" \
  --filter-pattern "ERROR" \
  --metric-transformations \
    metricName=ApplicationErrorCount,metricNamespace=MyApp/Errors,metricValue=1,defaultValue=0

# Create an anomaly detector
aws cloudwatch put-anomaly-detector \
  --namespace "AWS/EC2" \
  --metric-name "CPUUtilization" \
  --dimensions Name=InstanceId,Value=i-0abc123def456789 \
  --stat "Average" \
  --configuration '{"ExcludedTimeRanges": [], "MetricTimezone": "UTC"}'
```

---

### Read / Describe Operations

```bash
# Describe all alarms in ALARM state
aws cloudwatch describe-alarms \
  --state-value ALARM \
  --query 'MetricAlarms[*].{Name:AlarmName,State:StateValue,Reason:StateReason}' \
  --output table

# Get a specific dashboard
aws cloudwatch get-dashboard \
  --dashboard-name "ProductionOverview"

# Describe log streams in a log group
aws logs describe-log-streams \
  --log-group-name "/myapp/production/application" \
  --order-by LastEventTime \
  --descending \
  --limit 10

# Get log events from a specific stream
aws logs get-log-events \
  --log-group-name "/myapp/production/application" \
  --log-stream-name "web-server-01/2024/01/15" \
  --start-from-head \
  --limit 100

# Describe metric filters
aws logs describe-metric-filters \
  --log-group-name "/myapp/production/application"
```

---

### Update Operations

```bash
# Update alarm threshold
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPUUtilization-prod-web-01" \
  --metric-name "CPUUtilization" \
  --namespace "AWS/EC2" \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 3 \
  --dimensions Name=InstanceId,Value=i-0abc123def456789

# Update log group retention
aws logs put-retention-policy \
  --log-group-name "/myapp/production/application" \
  --retention-in-days 180

# Tag a CloudWatch alarm
aws cloudwatch tag-resource \
  --resource-arn "arn:aws:cloudwatch:us-east-1:123456789012:alarm:HighCPUUtilization-prod-web-01" \
  --tags Key=Environment,Value=production Key=CostCenter,Value=engineering
```

---

### Delete Operations

```bash
# Delete multiple alarms at once
aws cloudwatch delete-alarms \
  --alarm-names "OldAlarm-1" "OldAlarm-2" "OldAlarm-3"

# Delete a dashboard
aws cloudwatch delete-dashboards \
  --dashboard-names "OldDashboard" "TestDashboard"

# Delete a log group (WARNING: deletes all log streams and events)
aws logs delete-log-group \
  --log-group-name "/myapp/staging/application"

# Delete a metric filter
aws logs delete-metric-filter \
  --log-group-name "/myapp/production/application" \
  --