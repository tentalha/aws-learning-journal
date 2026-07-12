# Auto Scaling — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using AWS CLI with Auto Scaling, ensure the following are in place:

**Install/Update AWS CLI**
```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify version
aws --version
```

**Configure CLI credentials**
```bash
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

### Required IAM Permissions

Attach the following managed policies or create a custom policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:*",
        "ec2:DescribeInstances",
        "ec2:DescribeLaunchTemplates",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "elasticloadbalancing:DescribeLoadBalancers",
        "elasticloadbalancing:DescribeTargetGroups",
        "cloudwatch:PutMetricAlarm",
        "cloudwatch:DescribeAlarms",
        "iam:PassRole",
        "iam:CreateServiceLinkedRole"
      ],
      "Resource": "*"
    }
  ]
}
```

**AWS Managed Policies (quick setup)**
```bash
# Attach full Auto Scaling access
aws iam attach-user-policy \
  --user-name my-devops-user \
  --policy-arn arn:aws:iam::aws:policy/AutoScalingFullAccess
```

**Set default output format for readability**
```bash
export AWS_DEFAULT_OUTPUT=json
export AWS_DEFAULT_REGION=us-east-1
```

---

## Core Commands

### 1. Create a Launch Template (prerequisite for ASG)
```bash
aws ec2 create-launch-template \
  --launch-template-name my-app-launch-template \
  --version-description "Initial version" \
  --launch-template-data '{
    "ImageId": "ami-0c02fb55956c7d316",
    "InstanceType": "t3.medium",
    "KeyName": "my-keypair",
    "SecurityGroupIds": ["sg-0abc123def456789a"],
    "UserData": "IyEvYmluL2Jhc2gKeXVtIHVwZGF0ZSAteQ==",
    "IamInstanceProfile": {
      "Arn": "arn:aws:iam::123456789012:instance-profile/my-ec2-profile"
    },
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [{"Key": "Name", "Value": "my-app-instance"}]
    }]
  }'
```
**What it does:** Creates a versioned launch template that defines instance configuration for the Auto Scaling Group. This is the modern replacement for Launch Configurations.

**Example output:**
```json
{
    "LaunchTemplate": {
        "LaunchTemplateId": "lt-0abcdef1234567890",
        "LaunchTemplateName": "my-app-launch-template",
        "CreateTime": "2024-01-15T10:30:00.000Z",
        "CreatedBy": "arn:aws:iam::123456789012:user/my-devops-user",
        "DefaultVersionNumber": 1,
        "LatestVersionNumber": 1
    }
}
```

---

### 2. Create an Auto Scaling Group
```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-app-asg \
  --launch-template "LaunchTemplateId=lt-0abcdef1234567890,Version=\$Latest" \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 4 \
  --vpc-zone-identifier "subnet-0abc123def456789a,subnet-0def456abc123789b,subnet-0ghi789def456123c" \
  --target-group-arns "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-tg/1234567890abcdef" \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --tags "Key=Environment,Value=production,PropagateAtLaunch=true" \
         "Key=Application,Value=my-app,PropagateAtLaunch=true"
```
**What it does:** Creates an Auto Scaling Group with defined capacity limits, spread across multiple subnets (availability zones), and attached to an ALB target group with ELB-based health checks.

---

### 3. Describe Auto Scaling Groups
```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-app-asg \
  --query 'AutoScalingGroups[*].{Name:AutoScalingGroupName,Min:MinSize,Max:MaxSize,Desired:DesiredCapacity,Instances:length(Instances)}' \
  --output table
```
**What it does:** Retrieves detailed configuration and current state of one or more ASGs, including instance counts, capacity settings, and health status.

**Example output:**
```
----------------------------------------------------------
|              DescribeAutoScalingGroups                 |
+----------+-------+------+---------+--------------------+
| Desired  |  Max  | Min  |  Name   |     Instances      |
+----------+-------+------+---------+--------------------+
|  4       |  10   |  2   | my-app-asg |   4            |
+----------+-------+------+---------+--------------------+
```

---

### 4. Set Desired Capacity (Manual Scaling)
```bash
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name my-app-asg \
  --desired-capacity 6 \
  --honor-cooldown
```
**What it does:** Manually adjusts the desired number of running instances. The `--honor-cooldown` flag respects the cooldown period to prevent rapid scaling oscillations.

---

### 5. Create a Target Tracking Scaling Policy
```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-app-asg \
  --policy-name my-cpu-target-tracking-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 60.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60,
    "DisableScaleIn": false
  }'
```
**What it does:** Creates a target tracking policy that automatically adjusts capacity to keep average CPU utilization at 60%. AWS manages the scale-in/out alarms automatically.

**Example output:**
```json
{
    "PolicyARN": "arn:aws:autoscaling:us-east-1:123456789012:scalingPolicy:abc123:autoScalingGroupName/my-app-asg:policyName/my-cpu-target-tracking-policy",
    "Alarms": [
        {
            "AlarmName": "TargetTracking-my-app-asg-AlarmHigh-abc123",
            "AlarmARN": "arn:aws:cloudwatch:us-east-1:123456789012:alarm:TargetTracking-my-app-asg-AlarmHigh-abc123"
        },
        {
            "AlarmName": "TargetTracking-my-app-asg-AlarmLow-abc123",
            "AlarmARN": "arn:aws:cloudwatch:us-east-1:123456789012:alarm:TargetTracking-my-app-asg-AlarmLow-abc123"
        }
    ]
}
```

---

### 6. Create a Scheduled Scaling Action
```bash
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name my-app-asg \
  --scheduled-action-name scale-up-business-hours \
  --recurrence "0 8 * * MON-FRI" \
  --min-size 4 \
  --max-size 20 \
  --desired-capacity 8 \
  --time-zone "America/New_York"
```
**What it does:** Schedules a recurring scaling action using cron syntax. This example scales up every weekday at 8 AM Eastern time to handle business-hours traffic.

---

### 7. Describe Scaling Activities
```bash
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name my-app-asg \
  --max-items 10 \
  --query 'Activities[*].{Status:StatusCode,Cause:Cause,Start:StartTime,End:EndTime}' \
  --output table
```
**What it does:** Shows the history of scaling events for an ASG, including the reason each scaling action was triggered and its outcome. Essential for auditing and debugging.

---

### 8. Attach Instances to an ASG
```bash
aws autoscaling attach-instances \
  --instance-ids i-0abcdef1234567890 i-0fedcba0987654321 \
  --auto-scaling-group-name my-app-asg
```
**What it does:** Adds existing EC2 instances to an Auto Scaling Group. The instances must be in a running state and in the same Availability Zone as the ASG.

---

### 9. Detach Instances from an ASG
```bash
aws autoscaling detach-instances \
  --instance-ids i-0abcdef1234567890 \
  --auto-scaling-group-name my-app-asg \
  --should-decrement-desired-capacity
```
**What it does:** Removes specific instances from an ASG. Using `--should-decrement-desired-capacity` reduces the desired count so a replacement is not launched automatically.

---

### 10. Suspend Scaling Processes
```bash
aws autoscaling suspend-processes \
  --auto-scaling-group-name my-app-asg \
  --scaling-processes Launch Terminate HealthCheck ReplaceUnhealthy
```
**What it does:** Temporarily pauses specific Auto Scaling processes. Useful during deployments, maintenance windows, or debugging. Available processes include: `Launch`, `Terminate`, `HealthCheck`, `ReplaceUnhealthy`, `AZRebalance`, `AlarmNotification`, `ScheduledActions`, `AddToLoadBalancer`, `InstanceRefresh`.

---

### 11. Resume Scaling Processes
```bash
aws autoscaling resume-processes \
  --auto-scaling-group-name my-app-asg \
  --scaling-processes Launch Terminate HealthCheck ReplaceUnhealthy
```
**What it does:** Re-enables previously suspended Auto Scaling processes, restoring normal operation after maintenance.

---

### 12. Update an Auto Scaling Group
```bash
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-app-asg \
  --min-size 3 \
  --max-size 15 \
  --desired-capacity 5 \
  --default-cooldown 180 \
  --health-check-grace-period 600
```
**What it does:** Modifies the configuration of an existing ASG without interrupting running instances. Changes take effect immediately for new scaling decisions.

---

### 13. Delete an Auto Scaling Group
```bash
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name my-app-asg \
  --force-delete
```
**What it does:** Deletes the ASG and terminates all running instances. The `--force-delete` flag skips the normal graceful termination process and deletes even if instances are in a protected state.

---

### 14. Describe Auto Scaling Instances
```bash
aws autoscaling describe-auto-scaling-instances \
  --query 'AutoScalingInstances[?AutoScalingGroupName==`my-app-asg`].{ID:InstanceId,AZ:AvailabilityZone,State:LifecycleState,Health:HealthStatus}' \
  --output table
```
**What it does:** Lists all instances managed by Auto Scaling with their current lifecycle state and health status.

**Example output:**
```
--------------------------------------------------------------
|           DescribeAutoScalingInstances                     |
+---------------------+------------+-----------+------------+
|          AZ         |   Health   |    ID     |   State    |
+---------------------+------------+-----------+------------+
|  us-east-1a         |  Healthy   | i-0abc123 |  InService |
|  us-east-1b         |  Healthy   | i-0def456 |  InService |
|  us-east-1c         |  Healthy   | i-0ghi789 |  InService |
|  us-east-1a         |  Healthy   | i-0jkl012 |  InService |
+---------------------+------------+-----------+------------+
```

---

### 15. Start an Instance Refresh
```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name my-app-asg \
  --strategy Rolling \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 300,
    "CheckpointPercentages": [20, 50, 100],
    "CheckpointDelay": 600
  }'
```
**What it does:** Initiates a rolling replacement of all instances in the ASG, replacing them with the latest launch template version. Checkpoints pause the refresh for manual validation at defined thresholds.

---

## Common Operations

### Create Operations

**Create a Launch Configuration (legacy, prefer Launch Templates)**
```bash
aws autoscaling create-launch-configuration \
  --launch-configuration-name my-app-lc \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.medium \
  --key-name my-keypair \
  --security-groups sg-0abc123def456789a \
  --iam-instance-profile my-ec2-profile \
  --user-data file://userdata.sh
```

**Create a Step Scaling Policy**
```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-app-asg \
  --policy-name my-step-scale-out-policy \
  --policy-type StepScaling \
  --adjustment-type ChangeInCapacity \
  --step-adjustments '[
    {"MetricIntervalLowerBound": 0, "MetricIntervalUpperBound": 20, "ScalingAdjustment": 1},
    {"MetricIntervalLowerBound": 20, "MetricIntervalUpperBound": 40, "ScalingAdjustment": 2},
    {"MetricIntervalLowerBound": 40, "ScalingAdjustment": 4}
  ]' \
  --estimated-instance-warmup 120
```

**Create a Lifecycle Hook**
```bash
aws autoscaling put-lifecycle-hook \
  --lifecycle-hook-name my-launch-hook \
  --auto-scaling-group-name my-app-asg \
  --lifecycle-transition autoscaling:EC2_INSTANCE_LAUNCHING \
  --notification-target-arn arn:aws:sqs:us-east-1:123456789012:my-lifecycle-queue \
  --role-arn arn:aws:iam::123456