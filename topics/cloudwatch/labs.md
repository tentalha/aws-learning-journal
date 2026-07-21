# CloudWatch — Hands-On Labs

## Lab 1: Getting Started with CloudWatch

### Objective
In this lab, you will explore Amazon CloudWatch fundamentals by publishing custom metrics from an EC2 instance, creating a basic dashboard, and setting up your first alarm. By the end, you will have a working CloudWatch dashboard displaying CPU utilization and a custom application metric, with an alarm that triggers when CPU exceeds a threshold.

---

### Prerequisites

**AWS Services Required:**
- Amazon EC2 (t2.micro or t3.micro — Free Tier eligible)
- Amazon CloudWatch
- Amazon SNS (for alarm notifications)
- AWS IAM

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "cloudwatch:PutMetricAlarm",
        "cloudwatch:DescribeAlarms",
        "cloudwatch:DeleteAlarms",
        "cloudwatch:PutDashboard",
        "cloudwatch:DeleteDashboards",
        "sns:CreateTopic",
        "sns:Subscribe",
        "sns:DeleteTopic",
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- SSH client (for EC2 access)
- A valid email address (for SNS alarm notifications)
- AWS Console access via a modern browser

**Estimated Cost:** < $0.50 (if cleaned up within 2 hours)
**Estimated Duration:** 45–60 minutes

---

### Steps

#### Step 1: Launch an EC2 Instance with CloudWatch Role

**Console:**
1. Navigate to **EC2 → Instances → Launch Instances**
2. Name: `cw-lab1-instance`
3. AMI: **Amazon Linux 2023** (free tier)
4. Instance type: `t2.micro`
5. Key pair: Create or select an existing key pair
6. Under **Advanced Details → IAM instance profile**, click **Create new IAM role**:
   - Role name: `EC2CloudWatchRole`
   - Attach policy: `CloudWatchAgentServerPolicy`
   - Attach policy: `AmazonSSMManagedInstanceCore`
7. Launch the instance

**CLI:**
```bash
# Step 1a: Create the IAM role trust policy
cat > /tmp/ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Step 1b: Create the IAM role
aws iam create-role \
  --role-name EC2CloudWatchRole \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json

# Step 1c: Attach required policies
aws iam attach-role-policy \
  --role-name EC2CloudWatchRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam attach-role-policy \
  --role-name EC2CloudWatchRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Step 1d: Create instance profile and add role
aws iam create-instance-profile \
  --instance-profile-name EC2CloudWatchProfile

aws iam add-role-to-instance-profile \
  --instance-profile-name EC2CloudWatchProfile \
  --role-name EC2CloudWatchRole

# Step 1e: Get the latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" \
            "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

echo "Using AMI: $AMI_ID"

# Step 1f: Launch the EC2 instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t2.micro \
  --iam-instance-profile Name=EC2CloudWatchProfile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=cw-lab1-instance},{Key=Lab,Value=CloudWatch-Lab1}]' \
  --query "Instances[0].InstanceId" \
  --output text)

echo "Launched Instance ID: $INSTANCE_ID"

# Save for later steps
export INSTANCE_ID
```

**Verify:**
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].State.Name" \
  --output text
```
**Expected Output:** `running` (may take 1–2 minutes)

---

#### Step 2: Create an SNS Topic for Alarm Notifications

**Console:**
1. Navigate to **SNS → Topics → Create topic**
2. Type: **Standard**
3. Name: `cw-lab1-alerts`
4. Click **Create topic**
5. Click **Create subscription**:
   - Protocol: **Email**
   - Endpoint: your email address
6. Check your email and click **Confirm subscription**

**CLI:**
```bash
# Create the SNS topic
SNS_TOPIC_ARN=$(aws sns create-topic \
  --name cw-lab1-alerts \
  --query TopicArn \
  --output text)

echo "SNS Topic ARN: $SNS_TOPIC_ARN"
export SNS_TOPIC_ARN

# Subscribe your email (replace with your actual email)
aws sns subscribe \
  --topic-arn "$SNS_TOPIC_ARN" \
  --protocol email \
  --notification-endpoint "your-email@example.com"
```

**Verify:**
```bash
aws sns list-subscriptions-by-topic \
  --topic-arn "$SNS_TOPIC_ARN" \
  --query "Subscriptions[0].SubscriptionArn" \
  --output text
```
**Expected Output:** `PendingConfirmation` (until you confirm via email)

> ⚠️ **Important:** Check your email inbox and click the confirmation link before proceeding.

---

#### Step 3: Publish a Custom Metric to CloudWatch

**Console:**
1. Navigate to **CloudWatch → Metrics → All metrics**
2. Note that you cannot publish metrics from the console directly — we use the CLI or SDK

**CLI (run from your local machine or EC2 via SSM):**
```bash
# Publish a custom metric simulating an application queue depth
aws cloudwatch put-metric-data \
  --namespace "MyApp/Production" \
  --metric-name "QueueDepth" \
  --dimensions "Environment=Production,Service=OrderProcessor" \
  --value 42 \
  --unit Count

# Publish several data points to make the metric visible
for i in {1..5}; do
  VALUE=$((RANDOM % 100))
  echo "Publishing QueueDepth=$VALUE"
  aws cloudwatch put-metric-data \
    --namespace "MyApp/Production" \
    --metric-name "QueueDepth" \
    --dimensions "Environment=Production,Service=OrderProcessor" \
    --value "$VALUE" \
    --unit Count
  sleep 5
done

echo "Custom metrics published successfully."
```

**Verify:**
```bash
# Query the metric data back (wait ~60 seconds for data to appear)
aws cloudwatch get-metric-statistics \
  --namespace "MyApp/Production" \
  --metric-name "QueueDepth" \
  --dimensions "Name=Environment,Value=Production" "Name=Service,Value=OrderProcessor" \
  --start-time "$(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v-10M +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 60 \
  --statistics Average Maximum Minimum \
  --output table
```

**Expected Output:**
```
----------------------------------------------------------
|              GetMetricStatistics                       |
+------------------+-------+-------+-------+------------+
|    Average       | Max   | Min   | Sum   | Timestamp  |
+------------------+-------+-------+-------+------------+
|  54.2            | 99.0  | 12.0  | ...   | 2024-...   |
+------------------+-------+-------+-------+------------+
```

> In the Console: Navigate to **CloudWatch → Metrics → All metrics → Custom namespaces → MyApp/Production** to see your metric graphed.

---

#### Step 4: Create a CloudWatch Alarm on CPU Utilization

**Console:**
1. Navigate to **CloudWatch → Alarms → Create alarm**
2. Click **Select metric → EC2 → Per-Instance Metrics**
3. Find your instance ID and select **CPUUtilization**
4. Configure:
   - Period: **1 minute**
   - Statistic: **Average**
   - Threshold: **Greater than 70** (percent)
   - Datapoints to alarm: **2 out of 3**
5. Under **Notification**:
   - Alarm state trigger: **In alarm**
   - SNS topic: `cw-lab1-alerts`
6. Alarm name: `cw-lab1-high-cpu`
7. Click **Create alarm**

**CLI:**
```bash
# Create the CPU utilization alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "cw-lab1-high-cpu" \
  --alarm-description "Alert when EC2 CPU exceeds 70% for 2 consecutive minutes" \
  --namespace "AWS/EC2" \
  --metric-name "CPUUtilization" \
  --dimensions "Name=InstanceId,Value=$INSTANCE_ID" \
  --statistic Average \
  --period 60 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions "$SNS_TOPIC_ARN" \
  --ok-actions "$SNS_TOPIC_ARN"

echo "Alarm created successfully."
```

**Verify:**
```bash
aws cloudwatch describe-alarms \
  --alarm-names "cw-lab1-high-cpu" \
  --query "MetricAlarms[0].{Name:AlarmName,State:StateValue,Threshold:Threshold}" \
  --output table
```

**Expected Output:**
```
----------------------------------------------
|            DescribeAlarms                  |
+-------+-------------------+----------------+
| Name  | cw-lab1-high-cpu  |                |
| State | INSUFFICIENT_DATA |                |
| Thresh| 70.0              |                |
+-------+-------------------+----------------+
```

> The state will be `INSUFFICIENT_DATA` initially and transition to `OK` once enough data points are collected.

---

#### Step 5: Create a CloudWatch Dashboard

**Console:**
1. Navigate to **CloudWatch → Dashboards → Create dashboard**
2. Name: `cw-lab1-dashboard`
3. Add widget: **Line chart**
   - Metrics: EC2 → Per-Instance Metrics → CPUUtilization for your instance
4. Add another widget: **Number**
   - Metrics: MyApp/Production → QueueDepth
5. Add widget: **Alarm status** → select `cw-lab1-high-cpu`
6. Click **Save dashboard**

**CLI:**
```bash
# Create dashboard via JSON definition
cat > /tmp/dashboard-body.json << EOF
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "EC2 CPU Utilization",
        "metrics": [
          ["AWS/EC2", "CPUUtilization", "InstanceId", "$INSTANCE_ID",
           {"stat": "Average", "period": 60, "color": "#ff7f0e"}]
        ],
        "view": "timeSeries",
        "yAxis": {"left": {"min": 0, "max": 100}},
        "annotations": {
          "horizontal": [{"value": 70, "label": "Alarm Threshold", "color": "#d62728"}]
        },
        "period": 60,
        "region": "$(aws configure get region)"
      }
    },
    {
      "type": "metric",
      "x": 12, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "App Queue Depth",
        "metrics": [
          ["MyApp/Production", "QueueDepth",
           "Environment", "Production",
           "Service", "OrderProcessor",
           {"stat": "Average", "period": 60, "color": "#2ca02c"}]
        ],
        "view": "timeSeries",
        "period": 60,
        "region": "$(aws configure get region)"
      }
    },
    {
      "type": "alarm",
      "x": 0, "y": 6, "width": 6, "height": 3,
      "properties": {
        "title": "Alarm Status",
        "alarms": [
          "arn:aws:cloudwatch:$(aws configure get region):$(aws sts get-caller-identity --query Account --output text):alarm:cw-lab1-high-cpu"
        ]
      }
    }
  ]
}
EOF

aws cloudwatch put-dashboard \
  --dashboard-name "cw-lab1-dashboard" \
  --dashboard-body file:///tmp/dashboard-body.json

echo "Dashboard created."
```

**Verify:**
```bash
aws cloudwatch list-dashboards \
  --query "DashboardEntries[?DashboardName=='cw-lab1-dashboard']" \
  --output table
```

**Expected Output:**
```
---------------------------------------------
|           ListDashboards                  |
+-----------------------+-------------------+
|  DashboardName        | cw-lab1-dashboard |
|  Size                 | 1234              |
+-----------------------+-------------------+
```

---

### Verification

Run this verification script to confirm all lab resources are in place:

```bash
#!/bin/bash
echo "=== Lab 1 Verification ==="

echo ""
echo "1. Checking EC2 Instance..."
STATE=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=cw-lab1-instance" \
  --query "Reservations[0].Instances[0].State.Name" \
  --output text)
echo "   Instance State: $STATE"
[ "$STATE" == "running" ] && echo "   ✅ PASS" || echo "   ❌ FAIL"

echo ""
echo "2. Checking SNS Topic..."
TOPIC=$(aws sns list-topics \
  --query "Topics[?contains(TopicArn,'cw-lab1-alerts')].TopicArn" \
  --output text)
[ -n "$TOPIC" ] && echo "   ✅ PASS: $TOPIC" || echo "   ❌ FAIL: Topic not found"

echo ""
echo "3. Checking Custom Metric..."
METRIC=$(aws cloudwatch list-metrics \
  --namespace "MyApp/Production" \
  --metric-name "QueueDepth" \
  --query "Metrics[0].MetricName" \
  --output text)
[ "$METRIC" == "QueueDepth" ] &&