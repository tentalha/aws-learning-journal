# Auto Scaling — Hands-On Labs

## Lab 1: Getting Started with Auto Scaling

### Objective
In this lab, you will learn the foundational components of AWS Auto Scaling by creating a Launch Template, configuring an Auto Scaling Group (ASG), and observing how EC2 instances are automatically launched and terminated based on desired capacity settings. By the end of this lab, you will understand the relationship between Launch Templates, Auto Scaling Groups, and EC2 instances.

### Prerequisites

**AWS Services Required:**
- Amazon EC2
- AWS Auto Scaling
- Amazon VPC (default VPC is acceptable)

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "autoscaling:*",
        "elasticloadbalancing:*",
        "cloudwatch:*",
        "iam:CreateServiceLinkedRole",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS Management Console access
- AWS CLI v2 installed and configured (`aws configure`)
- A key pair created in your target region (e.g., `us-east-1`)
- Basic familiarity with EC2 instances

**Estimated Cost:** < $0.50 (assuming cleanup within 2 hours)  
**Estimated Duration:** 45–60 minutes  
**Region:** `us-east-1` (N. Virginia)

---

### Steps

#### Step 1: Create a Security Group for the Auto Scaling Group

**Console:**
1. Navigate to **EC2 → Security Groups → Create security group**
2. Configure:
   - **Security group name:** `asg-lab1-sg`
   - **Description:** `Security group for ASG Lab 1`
   - **VPC:** Select your default VPC
3. Add **Inbound Rules:**
   - Type: `HTTP`, Port: `80`, Source: `0.0.0.0/0`
   - Type: `SSH`, Port: `22`, Source: `My IP`
4. Click **Create security group**

**CLI:**
```bash
# Get the default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text)

echo "Default VPC ID: $VPC_ID"

# Create the security group
SG_ID=$(aws ec2 create-security-group \
  --group-name "asg-lab1-sg" \
  --description "Security group for ASG Lab 1" \
  --vpc-id $VPC_ID \
  --query "GroupId" \
  --output text)

echo "Security Group ID: $SG_ID"

# Get your current public IP
MY_IP=$(curl -s https://checkip.amazonaws.com)/32

# Add inbound rules
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr $MY_IP

echo "Security group rules added successfully"
```

**✅ Verify:** The security group `asg-lab1-sg` appears in the EC2 console with the correct inbound rules.

---

#### Step 2: Create a Launch Template

The Launch Template defines the configuration for instances that the ASG will launch.

**Console:**
1. Navigate to **EC2 → Launch Templates → Create launch template**
2. Configure:
   - **Launch template name:** `asg-lab1-lt`
   - **Template version description:** `v1 - Basic web server`
   - **Auto Scaling guidance:** ✅ Check the box
3. **AMI:** Search for `Amazon Linux 2023 AMI` (64-bit x86)
4. **Instance type:** `t3.micro`
5. **Key pair:** Select your existing key pair
6. **Security groups:** Select `asg-lab1-sg`
7. Expand **Advanced details → User data**, paste the following:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
cat > /var/www/html/index.html << EOF
<html>
<body style="font-family: Arial; text-align: center; padding: 50px;">
  <h1>Auto Scaling Lab 1</h1>
  <p><strong>Instance ID:</strong> ${INSTANCE_ID}</p>
  <p><strong>Availability Zone:</strong> ${AZ}</p>
  <p><strong>Launched at:</strong> $(date)</p>
</body>
</html>
EOF
```

8. Click **Create launch template**

**CLI:**
```bash
# Create user data script
cat > /tmp/userdata.sh << 'USERDATA'
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
cat > /var/www/html/index.html << EOF
<html>
<body style="font-family: Arial; text-align: center; padding: 50px;">
  <h1>Auto Scaling Lab 1</h1>
  <p><strong>Instance ID:</strong> ${INSTANCE_ID}</p>
  <p><strong>Availability Zone:</strong> ${AZ}</p>
  <p><strong>Launched at:</strong> $(date)</p>
</body>
</html>
EOF
USERDATA

# Encode user data to base64
USERDATA_B64=$(base64 -w 0 /tmp/userdata.sh)

# Get the latest Amazon Linux 2023 AMI ID
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-*-x86_64" \
    "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

echo "AMI ID: $AMI_ID"

# Create the launch template
LT_ID=$(aws ec2 create-launch-template \
  --launch-template-name "asg-lab1-lt" \
  --version-description "v1 - Basic web server" \
  --launch-template-data "{
    \"ImageId\": \"$AMI_ID\",
    \"InstanceType\": \"t3.micro\",
    \"SecurityGroupIds\": [\"$SG_ID\"],
    \"UserData\": \"$USERDATA_B64\",
    \"TagSpecifications\": [
      {
        \"ResourceType\": \"instance\",
        \"Tags\": [
          {\"Key\": \"Name\", \"Value\": \"asg-lab1-instance\"},
          {\"Key\": \"Lab\", \"Value\": \"AutoScaling-Lab1\"}
        ]
      }
    ]
  }" \
  --query "LaunchTemplate.LaunchTemplateId" \
  --output text)

echo "Launch Template ID: $LT_ID"
```

**✅ Verify:** Run the following command and confirm the launch template exists:
```bash
aws ec2 describe-launch-templates \
  --launch-template-names "asg-lab1-lt" \
  --query "LaunchTemplates[0].{Name:LaunchTemplateName,ID:LaunchTemplateId,Version:LatestVersionNumber}" \
  --output table
```

Expected output:
```
-------------------------------------------------
|         DescribeLaunchTemplates               |
+------+------------------+--------------------+
|  ID  |       Name       |      Version       |
+------+------------------+--------------------+
|lt-xxx| asg-lab1-lt      |         1          |
+------+------------------+--------------------+
```

---

#### Step 3: Get Subnet IDs from the Default VPC

**CLI:**
```bash
# Get all subnet IDs in the default VPC
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[*].SubnetId" \
  --output text | tr '\t' ',')

echo "Subnet IDs: $SUBNET_IDS"

# Store individual subnets for reference
SUBNET_1=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" \
  --output text)

SUBNET_2=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[1].SubnetId" \
  --output text)
```

---

#### Step 4: Create the Auto Scaling Group

**Console:**
1. Navigate to **EC2 → Auto Scaling Groups → Create Auto Scaling group**
2. **Step 1 — Name and launch template:**
   - **Name:** `asg-lab1-asg`
   - **Launch template:** `asg-lab1-lt`
   - Click **Next**
3. **Step 2 — Instance launch options:**
   - **VPC:** Default VPC
   - **Availability Zones and subnets:** Select at least 2 subnets (different AZs)
   - Click **Next**
4. **Step 3 — Advanced options:** Skip (no load balancer for this lab), click **Next**
5. **Step 4 — Group size and scaling:**
   - **Desired capacity:** `2`
   - **Minimum capacity:** `1`
   - **Maximum capacity:** `4`
   - **Automatic scaling:** Leave as `No scaling policies`
   - Click **Next**
6. **Step 5 — Notifications:** Skip, click **Next**
7. **Step 6 — Tags:**
   - Key: `Environment`, Value: `Lab`
   - Key: `Project`, Value: `AutoScalingLab1`
   - Click **Next**
8. **Step 7 — Review:** Click **Create Auto Scaling group**

**CLI:**
```bash
# Create the Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "asg-lab1-asg" \
  --launch-template "LaunchTemplateId=$LT_ID,Version=1" \
  --min-size 1 \
  --max-size 4 \
  --desired-capacity 2 \
  --vpc-zone-identifier "$SUBNET_IDS" \
  --tags \
    "Key=Environment,Value=Lab,PropagateAtLaunch=true" \
    "Key=Project,Value=AutoScalingLab1,PropagateAtLaunch=true" \
    "Key=Name,Value=asg-lab1-instance,PropagateAtLaunch=true"

echo "Auto Scaling Group created successfully"
```

**✅ Verify:** Check the ASG status:
```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names "asg-lab1-asg" \
  --query "AutoScalingGroups[0].{Name:AutoScalingGroupName,Min:MinSize,Max:MaxSize,Desired:DesiredCapacity,Status:Status}" \
  --output table
```

---

#### Step 5: Observe Instance Launch Activity

**Console:**
1. Navigate to **EC2 → Auto Scaling Groups → asg-lab1-asg**
2. Click the **Activity** tab — observe `Successful` launch events
3. Click the **Instance management** tab — wait for instances to show `InService` status
4. Navigate to **EC2 → Instances** — confirm 2 instances tagged `asg-lab1-instance` are running

**CLI:**
```bash
# Watch the activity history
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name "asg-lab1-asg" \
  --query "Activities[*].{Status:StatusCode,Description:Description,Time:StartTime}" \
  --output table

# Check instance health
aws autoscaling describe-auto-scaling-instances \
  --query "AutoScalingInstances[?AutoScalingGroupName=='asg-lab1-asg'].{InstanceId:InstanceId,State:LifecycleState,Health:HealthStatus,AZ:AvailabilityZone}" \
  --output table
```

**✅ Verify:** You should see 2 instances in `InService` state.

---

#### Step 6: Manually Adjust Desired Capacity

Now observe how the ASG responds when you change the desired capacity.

**Console:**
1. Go to **EC2 → Auto Scaling Groups → asg-lab1-asg**
2. Click **Edit**
3. Change **Desired capacity** to `3`
4. Click **Update**
5. Watch the **Activity** tab for a new launch event

**CLI:**
```bash
# Scale out to 3 instances
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name "asg-lab1-asg" \
  --desired-capacity 3 \
  --honor-cooldown

echo "Desired capacity updated to 3"

# Wait and observe
sleep 30

# Check current instances
aws autoscaling describe-auto-scaling-instances \
  --query "AutoScalingInstances[?AutoScalingGroupName=='asg-lab1-asg'].{InstanceId:InstanceId,State:LifecycleState,AZ:AvailabilityZone}" \
  --output table
```

**✅ Verify:** A third instance launches and reaches `InService` state. The Activity tab shows a `Launch` event.

---

#### Step 7: Test the Web Server on Launched Instances

```bash
# Get public IPs of running instances
aws ec2 describe-instances \
  --filters \
    "Name=tag:Name,Values=asg-lab1-instance" \
    "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].{InstanceId:InstanceId,PublicIP:PublicIpAddress,AZ:Placement.AvailabilityZone}" \
  --output table

# Test HTTP response (replace with actual IP)
curl http://<PUBLIC_IP>
```

**✅ Verify:** Each instance returns an HTML page showing its unique Instance ID and Availability Zone.

---

### Verification

Run the following verification script to confirm the lab is complete:

```bash
#!/bin/bash
echo "=== Lab 1 Verification ==="

# Check Launch Template
LT_COUNT=$(aws ec2 describe-launch-templates \
  --launch-template-names "asg-lab1-lt" \
  --query "length(LaunchTemplates)" \
  --output text 2>/dev/null)
[ "$LT_COUNT" -eq 1 ] && echo "✅ Launch Template exists" || echo "❌ Launch Template missing"

# Check ASG
ASG_EXISTS=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names "asg-lab1-asg" \
  --query "length(AutoScalingGroups)" \
  --output text)
[ "$ASG_EXISTS" -eq 1 ] && echo "✅ Auto Scaling Group exists" || echo "❌ ASG missing"

# Check running instances
INSTANCE_COUNT=$(aws autoscaling describe-auto-scaling-instances \
  --query "length(AutoScalingInstances[?AutoScalingGroupName=='asg-lab1