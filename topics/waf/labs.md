# WAF — Hands-On Labs

## Lab 1: Getting Started with WAF

### Objective
In this lab, you will create an AWS WAF Web ACL and attach it to an Application Load Balancer. You will configure AWS Managed Rule Groups to protect a sample web application from common threats including SQL injection, cross-site scripting (XSS), and known bad inputs. By the end of this lab, you will understand the fundamentals of WAF rule evaluation, rule priorities, and how to test WAF rules in count mode before enforcing them.

### Prerequisites

**AWS Services Required:**
- AWS WAF v2
- Application Load Balancer (ALB)
- Amazon EC2 (for a simple backend target)
- Amazon CloudWatch (for logging and metrics)

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "wafv2:*",
        "elasticloadbalancing:*",
        "ec2:*",
        "cloudwatch:GetMetricData",
        "cloudwatch:ListMetrics",
        "logs:CreateLogGroup",
        "logs:CreateLogDelivery",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- A web browser
- `curl` command-line tool
- AWS region: `us-east-1` (used throughout this lab)

**Estimated Cost:** < $1.00 USD (assuming lab is completed within 2 hours)

**Estimated Time:** 45–60 minutes

---

### Steps

#### Step 1: Deploy a Simple Web Application Backend

First, create a security group and launch an EC2 instance that will serve as your web application backend behind an ALB.

**Console:**
1. Navigate to **EC2 → Security Groups → Create security group**
2. Name: `waf-lab-ec2-sg`
3. Description: `Security group for WAF lab EC2 instances`
4. VPC: Select your default VPC
5. Add inbound rule: **HTTP (port 80)** from source `0.0.0.0/0`
6. Click **Create security group**

**CLI:**
```bash
# Set variables
export AWS_REGION="us-east-1"
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text \
  --region $AWS_REGION)

echo "Using VPC: $VPC_ID"

# Create EC2 security group
EC2_SG_ID=$(aws ec2 create-security-group \
  --group-name "waf-lab-ec2-sg" \
  --description "Security group for WAF lab EC2 instances" \
  --vpc-id $VPC_ID \
  --region $AWS_REGION \
  --query "GroupId" \
  --output text)

echo "EC2 Security Group ID: $EC2_SG_ID"

# Allow HTTP from anywhere (ALB will restrict access in production)
aws ec2 authorize-security-group-ingress \
  --group-id $EC2_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region $AWS_REGION
```

**Expected Output:**
```
Using VPC: vpc-0abc1234def56789a
EC2 Security Group ID: sg-0123456789abcdef0
```

Now launch an EC2 instance with a simple web server:

**CLI:**
```bash
# Get the latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-2023*-x86_64" \
    "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text \
  --region $AWS_REGION)

echo "Using AMI: $AMI_ID"

# User data script to install a simple web server
cat > /tmp/userdata.sh << 'EOF'
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl start httpd
systemctl enable httpd

# Create a simple web page
cat > /var/www/html/index.html << 'HTML'
<!DOCTYPE html>
<html>
<head><title>WAF Lab Application</title></head>
<body>
  <h1>WAF Lab - Web Application</h1>
  <p>Instance ID: INSTANCE_ID</p>
  <p>This application is protected by AWS WAF.</p>
</body>
</html>
HTML

# Replace placeholder with actual instance ID
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
sed -i "s/INSTANCE_ID/$INSTANCE_ID/" /var/www/html/index.html

# Create a login page to test SQL injection rules
cat > /var/www/html/login.html << 'HTML'
<!DOCTYPE html>
<html>
<head><title>Login Page</title></head>
<body>
  <h2>Login</h2>
  <form method="GET" action="/login.html">
    <label>Username: <input type="text" name="username"></label><br>
    <label>Password: <input type="password" name="password"></label><br>
    <input type="submit" value="Login">
  </form>
</body>
</html>
HTML
EOF

# Get a subnet for the instance
SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=defaultForAz,Values=true" \
  --query "Subnets[0].SubnetId" \
  --output text \
  --region $AWS_REGION)

# Launch EC2 instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --security-group-ids $EC2_SG_ID \
  --subnet-id $SUBNET_ID \
  --user-data file:///tmp/userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=waf-lab-web-server}]' \
  --query "Instances[0].InstanceId" \
  --output text \
  --region $AWS_REGION)

echo "Instance ID: $INSTANCE_ID"

# Wait for instance to be running
echo "Waiting for instance to be in running state..."
aws ec2 wait instance-running \
  --instance-ids $INSTANCE_ID \
  --region $AWS_REGION

echo "Instance is running!"
```

**Verify Step 1:**
```bash
# Check instance state
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].{State:State.Name,IP:PublicIpAddress}" \
  --output table \
  --region $AWS_REGION
```

**Expected Output:**
```
------------------------------
|    DescribeInstances       |
+------+---------------------+
|  IP  |  State              |
+------+---------------------+
|  3.x.x.x  |  running      |
+------+---------------------+
```

---

#### Step 2: Create an Application Load Balancer

**Console:**
1. Navigate to **EC2 → Load Balancers → Create load balancer**
2. Select **Application Load Balancer**
3. Name: `waf-lab-alb`
4. Scheme: **Internet-facing**
5. IP address type: **IPv4**
6. Select at least 2 Availability Zones
7. Create a new security group `waf-lab-alb-sg` allowing HTTP (80) from `0.0.0.0/0`
8. Create a target group `waf-lab-tg` with HTTP on port 80, register your EC2 instance
9. Click **Create load balancer**

**CLI:**
```bash
# Create ALB security group
ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name "waf-lab-alb-sg" \
  --description "Security group for WAF lab ALB" \
  --vpc-id $VPC_ID \
  --region $AWS_REGION \
  --query "GroupId" \
  --output text)

echo "ALB Security Group ID: $ALB_SG_ID"

# Allow HTTP inbound on ALB
aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region $AWS_REGION

# Get all default subnets (need at least 2 AZs for ALB)
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=defaultForAz,Values=true" \
  --query "Subnets[*].SubnetId" \
  --output text \
  --region $AWS_REGION | tr '\t' ' ')

echo "Subnets: $SUBNET_IDS"

# Create the ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name waf-lab-alb \
  --subnets $SUBNET_IDS \
  --security-groups $ALB_SG_ID \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --tags Key=Name,Value=waf-lab-alb \
  --query "LoadBalancers[0].LoadBalancerArn" \
  --output text \
  --region $AWS_REGION)

echo "ALB ARN: $ALB_ARN"

# Get instance private IP
INSTANCE_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].PrivateIpAddress" \
  --output text \
  --region $AWS_REGION)

# Create target group
TG_ARN=$(aws elbv2 create-target-group \
  --name waf-lab-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --health-check-path "/" \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --target-type instance \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text \
  --region $AWS_REGION)

echo "Target Group ARN: $TG_ARN"

# Register EC2 instance with target group
aws elbv2 register-targets \
  --target-group-arn $TG_ARN \
  --targets Id=$INSTANCE_ID \
  --region $AWS_REGION

# Create listener
LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --query "Listeners[0].ListenerArn" \
  --output text \
  --region $AWS_REGION)

echo "Listener ARN: $LISTENER_ARN"

# Get ALB DNS name
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query "LoadBalancers[0].DNSName" \
  --output text \
  --region $AWS_REGION)

echo "ALB DNS Name: $ALB_DNS"
```

**Verify Step 2:**
```bash
# Wait for ALB to be active
echo "Waiting for ALB to become active..."
aws elbv2 wait load-balancer-available \
  --load-balancer-arns $ALB_ARN \
  --region $AWS_REGION

# Test the application (may take 2-3 minutes for targets to be healthy)
echo "Testing application..."
sleep 60
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://$ALB_DNS/
```

**Expected Output:**
```
HTTP Status: 200
```

---

#### Step 3: Create a WAF Web ACL with Managed Rules

**Console:**
1. Navigate to **WAF & Shield → Web ACLs → Create web ACL**
2. Resource type: **Regional resources (ALB, API Gateway, AppSync)**
3. Region: **US East (N. Virginia)**
4. Name: `waf-lab-web-acl`
5. Description: `WAF Lab Web ACL with managed rules`
6. Add AWS Managed Rule Groups:
   - **AWSManagedRulesCommonRuleSet** (Core rule set)
   - **AWSManagedRulesSQLiRuleSet** (SQL injection)
7. Set default action to **Allow**
8. Associate with your ALB `waf-lab-alb`

**CLI:**
```bash
# Create the Web ACL
WEB_ACL_ARN=$(aws wafv2 create-web-acl \
  --name "waf-lab-web-acl" \
  --scope REGIONAL \
  --default-action Allow={} \
  --description "WAF Lab Web ACL with managed rules" \
  --rules '[
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 10,
      "OverrideAction": {
        "Count": {}
      },
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "AWSManagedRulesCommonRuleSet"
      }
    },
    {
      "Name": "AWSManagedRulesSQLiRuleSet",
      "Priority": 20,
      "OverrideAction": {
        "Count": {}
      },
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesSQLiRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "AWSManagedRulesSQLiRuleSet"
      }
    }
  ]' \
  --visibility-config '{
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "waf-lab-web-acl"
  }' \
  --region $AWS_REGION \
  --query "Summary.ARN" \
  --output text)

echo "Web ACL ARN: $WEB_ACL_ARN"

# Get the Web ACL ID (needed for association)
WEB_ACL_ID=$(aws wafv2 list-web-acls \
  --scope REGIONAL \
  --region $AWS_REGION \
  --query "WebACLs[?Name=='waf-lab-web-acl'].Id" \
  --output text)

echo "Web ACL ID: $WEB_ACL_ID"

# Associate the Web ACL with the ALB
aws wafv2 associate-web-acl \
  --web-acl-arn $WEB_ACL_ARN \
  --resource-arn $ALB