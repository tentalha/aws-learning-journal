# Load Balancer — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using the AWS CLI with Elastic Load Balancing, ensure the following:

**AWS CLI Version**
```bash
aws --version
# Requires AWS CLI v2.x (recommended) or v1.x >= 1.18
```

**Configure AWS CLI**
```bash
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

**Required IAM Permissions**

Attach the following managed policies or equivalent inline permissions to your IAM user/role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:*",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeVpcs",
        "ec2:DescribeInstances",
        "acm:ListCertificates",
        "acm:DescribeCertificate",
        "iam:CreateServiceLinkedRole",
        "cognito-idp:DescribeUserPoolClient",
        "wafv2:GetWebACL",
        "wafv2:AssociateWebACL"
      ],
      "Resource": "*"
    }
  ]
}
```

> **AWS Managed Policies:** `ElasticLoadBalancingFullAccess` or `ElasticLoadBalancingReadOnly` for read-only access.

### CLI Service Namespaces

| Load Balancer Type | CLI Namespace | Notes |
|---|---|---|
| Application (ALB) | `aws elbv2` | HTTP/HTTPS, Layer 7 |
| Network (NLB) | `aws elbv2` | TCP/UDP/TLS, Layer 4 |
| Gateway (GWLB) | `aws elbv2` | Layer 3, appliance traffic |
| Classic (CLB) | `aws elb` | Legacy, avoid for new workloads |

---

## Core Commands

### 1. Create an Application Load Balancer

```bash
aws elbv2 create-load-balancer \
  --name my-app-load-balancer \
  --subnets subnet-0abc12345def67890 subnet-0def67890abc12345 \
  --security-groups sg-0123456789abcdef0 \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --tags Key=Environment,Value=Production Key=Team,Value=Platform
```

**What it does:** Creates an internet-facing Application Load Balancer across two Availability Zones with the specified security group. Tags help with cost allocation and resource management.

**Example Output:**
```json
{
    "LoadBalancers": [
        {
            "LoadBalancerArn": "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-app-load-balancer/1234567890abcdef",
            "DNSName": "my-app-load-balancer-1234567890.us-east-1.elb.amazonaws.com",
            "CanonicalHostedZoneId": "Z35SXDOTRQ7X7K",
            "CreatedTime": "2024-01-15T10:30:00.000Z",
            "LoadBalancerName": "my-app-load-balancer",
            "Scheme": "internet-facing",
            "VpcId": "vpc-0abc123456def7890",
            "State": {
                "Code": "provisioning"
            },
            "Type": "application",
            "IpAddressType": "ipv4"
        }
    ]
}
```

---

### 2. Create a Target Group

```bash
aws elbv2 create-target-group \
  --name my-app-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-0abc123456def7890 \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3
```

**What it does:** Creates a target group that routes traffic to EC2 instances on port 80. Health checks are performed at `/health` every 30 seconds.

**Example Output:**
```json
{
    "TargetGroups": [
        {
            "TargetGroupArn": "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-targets/abcdef1234567890",
            "TargetGroupName": "my-app-targets",
            "Protocol": "HTTP",
            "Port": 80,
            "VpcId": "vpc-0abc123456def7890",
            "HealthCheckProtocol": "HTTP",
            "HealthCheckPath": "/health",
            "HealthCheckIntervalSeconds": 30,
            "HealthCheckTimeoutSeconds": 5,
            "HealthyThresholdCount": 2,
            "UnhealthyThresholdCount": 3,
            "Matcher": {
                "HttpCode": "200"
            },
            "TargetType": "instance"
        }
    ]
}
```

---

### 3. Register Targets with a Target Group

```bash
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-targets/abcdef1234567890 \
  --targets Id=i-0abc1234567890def,Port=80 Id=i-0def1234567890abc,Port=80
```

**What it does:** Registers two EC2 instances as targets in the specified target group. Traffic will be load balanced across these instances once they pass health checks.

---

### 4. Create an HTTP Listener

```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-app-load-balancer/1234567890abcdef \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-targets/abcdef1234567890
```

**What it does:** Creates a listener on port 80 that forwards all incoming HTTP traffic to the `my-app-targets` target group.

---

### 5. Create an HTTPS Listener with Certificate

```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-app-load-balancer/1234567890abcdef \
  --protocol HTTPS \
  --port 443 \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --certificates CertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/abcd1234-ef56-7890-abcd-ef1234567890 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-targets/abcdef1234567890
```

**What it does:** Creates an HTTPS listener on port 443 using a TLS 1.3 security policy and an ACM-managed certificate.

---

### 6. Describe Load Balancers

```bash
aws elbv2 describe-load-balancers \
  --names my-app-load-balancer \
  --query 'LoadBalancers[*].{Name:LoadBalancerName,DNS:DNSName,State:State.Code,Type:Type}' \
  --output table
```

**What it does:** Retrieves details about the specified load balancer and formats the output as a human-readable table.

**Example Output:**
```
--------------------------------------------------------------------------------------------------
|                                    DescribeLoadBalancers                                       |
+------------------------+----------------------------------------------------------+-------+----+
|  DNS                   |  Name                    | State      | Type              |
+------------------------+----------------------------------------------------------+-------+----+
|  my-app-lb-123.elb.amazonaws.com  |  my-app-load-balancer  |  active  |  application  |
+------------------------+----------------------------------------------------------+-------+----+
```

---

### 7. Describe Target Groups

```bash
aws elbv2 describe-target-groups \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-app-load-balancer/1234567890abcdef \
  --query 'TargetGroups[*].{Name:TargetGroupName,Protocol:Protocol,Port:Port,HealthPath:HealthCheckPath}' \
  --output table
```

**What it does:** Lists all target groups associated with the specified load balancer, showing key configuration details.

---

### 8. Describe Target Health

```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-targets/abcdef1234567890 \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,Port:Target.Port,State:TargetHealth.State,Reason:TargetHealth.Reason}' \
  --output table
```

**What it does:** Checks the health status of all registered targets in a target group. Essential for debugging unhealthy instances.

**Example Output:**
```json
{
    "TargetHealthDescriptions": [
        {
            "Target": {
                "Id": "i-0abc1234567890def",
                "Port": 80
            },
            "TargetHealth": {
                "State": "healthy"
            }
        },
        {
            "Target": {
                "Id": "i-0def1234567890abc",
                "Port": 80
            },
            "TargetHealth": {
                "State": "unhealthy",
                "Reason": "Target.FailedHealthChecks",
                "Description": "Health checks failed"
            }
        }
    ]
}
```

---

### 9. Create a Listener Rule (Path-Based Routing)

```bash
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/my-app-load-balancer/1234567890abcdef/abcdef1234567890 \
  --priority 10 \
  --conditions Field=path-pattern,Values='/api/*' \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-api-targets/fedcba0987654321
```

**What it does:** Creates a routing rule that sends all requests matching `/api/*` to a dedicated API target group, enabling path-based routing.

---

### 10. Modify Load Balancer Attributes

```bash
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-app-load-balancer/1234567890abcdef \
  --attributes \
    Key=idle_timeout.timeout_seconds,Value=120 \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=my-alb-access-logs-bucket \
    Key=access_logs.s3.prefix,Value=my-app-load-balancer \
    Key=deletion_protection.enabled,Value=true
```

**What it does:** Enables S3 access logging, sets idle timeout to 120 seconds, and enables deletion protection to prevent accidental removal.

---

### 11. Modify Target Group Attributes

```bash
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-targets/abcdef1234567890 \
  --attributes \
    Key=deregistration_delay.timeout_seconds,Value=60 \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=86400
```

**What it does:** Enables sticky sessions (session affinity) with a 24-hour cookie duration and reduces deregistration delay to 60 seconds for faster deployments.

---

### 12. Delete a Load Balancer

```bash
aws elbv2 delete-load-balancer \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-app-load-balancer/1234567890abcdef
```

**What it does:** Deletes the specified load balancer. Note: Deletion protection must be disabled first if enabled. This action is irreversible.

---

### 13. Deregister Targets

```bash
aws elbv2 deregister-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-targets/abcdef1234567890 \
  --targets Id=i-0abc1234567890def Id=i-0def1234567890abc
```

**What it does:** Removes EC2 instances from the target group. In-flight connections are drained during the deregistration delay period.

---

### 14. Create a Network Load Balancer

```bash
aws elbv2 create-load-balancer \
  --name my-network-load-balancer \
  --subnets subnet-0abc12345def67890 subnet-0def67890abc12345 \
  --scheme internet-facing \
  --type network \
  --ip-address-type ipv4 \
  --tags Key=Environment,Value=Production
```

**What it does:** Creates a Network Load Balancer for TCP/UDP traffic. NLBs do not require security groups and support static IP addresses and Elastic IPs.

---

### 15. Describe Listeners

```bash
aws elbv2 describe-listeners \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-app-load-balancer/1234567890abcdef \
  --query 'Listeners[*].{Port:Port,Protocol:Protocol,ARN:ListenerArn}' \
  --output table
```

**What it does:** Lists all listeners configured on the specified load balancer with their protocols and ports.

---

## Common Operations

### Create Operations

**Create a Lambda Target Group (for serverless backends)**
```bash
aws elbv2 create-target-group \
  --name my-lambda-targets \
  --target-type lambda \
  --health-check-enabled \
  --health-check-path /health
```

**Create an IP-type Target Group (for containers/ECS)**
```bash
aws elbv2 create-target-group \
  --name my-ecs-targets \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-0abc123456def7890 \
  --target-type ip