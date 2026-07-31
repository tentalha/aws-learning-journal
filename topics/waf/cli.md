# WAF — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using AWS WAF CLI commands, ensure the following are in place:

**AWS CLI Version**
```bash
aws --version
# Requires AWS CLI v2.x for full WAFv2 support
```

**Configure AWS CLI**
```bash
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

> **Important:** AWS WAF has two API versions:
> - `aws wafv2` — Current version (recommended for all new deployments)
> - `aws waf` — Classic/Legacy version (regional and global)
>
> This guide focuses on **WAFv2**, which is the current standard.

### Required IAM Permissions

Attach the following IAM policy to your user or role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "wafv2:*",
        "cloudfront:GetDistribution",
        "elasticloadbalancing:DescribeLoadBalancers",
        "apigateway:GET",
        "logs:CreateLogGroup",
        "logs:CreateLogDelivery",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

**Managed Policy Alternative:**
```bash
aws iam attach-user-policy \
  --user-name my-waf-admin \
  --policy-arn arn:aws:iam::aws:policy/AWSWAFFullAccess
```

### Scope Parameter

WAFv2 requires a `--scope` parameter for all commands:
- `REGIONAL` — For ALB, API Gateway, AppSync, Cognito User Pools
- `CLOUDFRONT` — For CloudFront distributions (must use `us-east-1` region)

```bash
# For CloudFront WAF, always specify us-east-1
export AWS_DEFAULT_REGION=us-east-1

# For regional resources
export AWS_DEFAULT_REGION=us-west-2
```

---

## Core Commands

### 1. List All Web ACLs

```bash
aws wafv2 list-web-acls \
  --scope REGIONAL \
  --region us-east-1
```

**What it does:** Lists all Web ACLs in the specified scope and region.

**Example Output:**
```json
{
    "WebACLs": [
        {
            "Name": "my-web-acl",
            "Id": "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
            "Description": "Main application WAF",
            "LockToken": "token-abc123",
            "ARN": "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/a1b2c3d4"
        }
    ]
}
```

---

### 2. Create a Web ACL

```bash
aws wafv2 create-web-acl \
  --name "my-web-acl" \
  --scope REGIONAL \
  --default-action '{"Allow": {}}' \
  --visibility-config '{
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "my-web-acl-metric"
  }' \
  --description "WAF for production ALB" \
  --region us-east-1
```

**What it does:** Creates a new Web ACL with a default "Allow" action and CloudWatch metrics enabled.

**Example Output:**
```json
{
    "Summary": {
        "Name": "my-web-acl",
        "Id": "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
        "Description": "WAF for production ALB",
        "LockToken": "token-xyz789",
        "ARN": "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/a1b2c3d4"
    }
}
```

---

### 3. Get Web ACL Details

```bash
aws wafv2 get-web-acl \
  --name "my-web-acl" \
  --scope REGIONAL \
  --id "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111" \
  --region us-east-1
```

**What it does:** Retrieves full configuration details of a specific Web ACL, including all rules.

**Example Output:**
```json
{
    "WebACL": {
        "Name": "my-web-acl",
        "Id": "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
        "ARN": "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/a1b2c3d4",
        "DefaultAction": { "Allow": {} },
        "Rules": [],
        "VisibilityConfig": {
            "SampledRequestsEnabled": true,
            "CloudWatchMetricsEnabled": true,
            "MetricName": "my-web-acl-metric"
        },
        "Capacity": 0
    },
    "LockToken": "token-xyz789"
}
```

---

### 4. Associate Web ACL with an ALB

```bash
aws wafv2 associate-web-acl \
  --web-acl-arn "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/a1b2c3d4" \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/50dc6c495c0c9188" \
  --region us-east-1
```

**What it does:** Associates a Web ACL with an Application Load Balancer (or API Gateway, AppSync, etc.).

---

### 5. Create an IP Set

```bash
aws wafv2 create-ip-set \
  --name "blocked-ips" \
  --scope REGIONAL \
  --ip-address-version IPV4 \
  --addresses "192.0.2.0/24" "198.51.100.0/24" "203.0.113.44/32" \
  --description "Blocked malicious IPs" \
  --region us-east-1
```

**What it does:** Creates an IP set that can be referenced in WAF rules to allow or block specific IP ranges.

**Example Output:**
```json
{
    "Summary": {
        "Name": "blocked-ips",
        "Id": "b2c3d4e5-6789-01bc-defa-EXAMPLE22222",
        "Description": "Blocked malicious IPs",
        "LockToken": "token-ip123",
        "ARN": "arn:aws:wafv2:us-east-1:123456789012:regional/ipset/blocked-ips/b2c3d4e5"
    }
}
```

---

### 6. Create a Managed Rule Group Association

```bash
aws wafv2 update-web-acl \
  --name "my-web-acl" \
  --scope REGIONAL \
  --id "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111" \
  --lock-token "token-xyz789" \
  --default-action '{"Allow": {}}' \
  --rules '[
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 1,
      "OverrideAction": {"None": {}},
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
    }
  ]' \
  --visibility-config '{
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "my-web-acl-metric"
  }' \
  --region us-east-1
```

**What it does:** Updates a Web ACL to include the AWS Managed Rules Common Rule Set, which protects against common web exploits.

---

### 7. List Available Managed Rule Groups

```bash
aws wafv2 list-available-managed-rule-groups \
  --scope REGIONAL \
  --region us-east-1
```

**What it does:** Lists all available managed rule groups from AWS and third-party vendors.

**Example Output:**
```json
{
    "ManagedRuleGroups": [
        {
            "VendorName": "AWS",
            "Name": "AWSManagedRulesCommonRuleSet",
            "VersioningSupported": true,
            "Description": "Contains rules that are generally applicable to web applications."
        },
        {
            "VendorName": "AWS",
            "Name": "AWSManagedRulesSQLiRuleSet",
            "VersioningSupported": true,
            "Description": "Rules to block SQL injection attacks."
        }
    ]
}
```

---

### 8. Get Sampled Requests

```bash
aws wafv2 get-sampled-requests \
  --web-acl-arn "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/a1b2c3d4" \
  --rule-metric-name "AWSManagedRulesCommonRuleSet" \
  --scope REGIONAL \
  --time-window '{
    "StartTime": "2024-01-15T00:00:00Z",
    "EndTime": "2024-01-15T23:59:59Z"
  }' \
  --max-items 100 \
  --region us-east-1
```

**What it does:** Retrieves a sample of web requests that matched a specific rule, useful for analyzing traffic patterns.

---

### 9. Create a Rule Group

```bash
aws wafv2 create-rule-group \
  --name "custom-rule-group" \
  --scope REGIONAL \
  --capacity 100 \
  --rules '[
    {
      "Name": "BlockSQLInjection",
      "Priority": 1,
      "Action": {"Block": {}},
      "Statement": {
        "SqliMatchStatement": {
          "FieldToMatch": {"Body": {}},
          "TextTransformations": [
            {"Priority": 1, "Type": "URL_DECODE"},
            {"Priority": 2, "Type": "HTML_ENTITY_DECODE"}
          ]
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "BlockSQLInjection"
      }
    }
  ]' \
  --visibility-config '{
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "custom-rule-group-metric"
  }' \
  --region us-east-1
```

**What it does:** Creates a reusable custom rule group with a SQL injection blocking rule.

---

### 10. Enable WAF Logging

```bash
aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/a1b2c3d4",
    "LogDestinationConfigs": [
      "arn:aws:firehose:us-east-1:123456789012:deliverystream/aws-waf-logs-my-stream"
    ],
    "RedactedFields": [
      {
        "SingleHeader": {
          "Name": "authorization"
        }
      }
    ]
  }' \
  --region us-east-1
```

**What it does:** Enables WAF logging to a Kinesis Data Firehose stream, with the Authorization header redacted for security.

> **Note:** The Firehose delivery stream name **must** begin with `aws-waf-logs-`.

---

### 11. Delete a Web ACL

```bash
aws wafv2 delete-web-acl \
  --name "my-web-acl" \
  --scope REGIONAL \
  --id "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111" \
  --lock-token "token-xyz789" \
  --region us-east-1
```

**What it does:** Deletes a Web ACL. You must first disassociate it from all resources and obtain the current lock token.

---

### 12. List Resources Associated with a Web ACL

```bash
aws wafv2 list-resources-for-web-acl \
  --web-acl-arn "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/a1b2c3d4" \
  --region us-east-1
```

**What it does:** Lists all AWS resources (ALBs, API Gateways, etc.) currently protected by the specified Web ACL.

**Example Output:**
```json
{
    "ResourceArns": [
        "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/50dc6c495c0c9188",
        "arn:aws:apigateway:us-east-1::/restapis/abc123def4/stages/prod"
    ]
}
```

---

### 13. Create a Regex Pattern Set

```bash
aws wafv2 create-regex-pattern-set \
  --name "bad-user-agents" \
  --scope REGIONAL \
  --regular-expression-list '[
    {"RegexString": "(?i)sqlmap"},
    {"RegexString": "(?i)nikto"},
    {"RegexString": "(?i)nessus"},
    {"RegexString": "(?i)masscan"}
  ]' \
  --description "Known scanner and attack tool user agents" \
  --region us-east-1
```

**What it does:** Creates a regex pattern set for matching malicious user agent strings in WAF rules.

---

### 14. Disassociate Web ACL from Resource

```bash
aws wafv2 disassociate-web-acl \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/50dc6c495c0c9188" \
  --region us-east-1
```

**What it does:** Removes the Web ACL association from the specified resource, stopping WAF protection.

---

### 15. Check WAF Capacity Units

```bash
aws wafv2 check-capacity \
  --scope REGIONAL \
  --rules '[
    {
      "Name": "TestRule",
      "Priority": 1,
      "Action": {"Block": {}},
      "Statement": {
        "IPSetReferenceStatement": {
          "ARN": "arn:aws:wafv2:us-east-1:123456789012:regional/ipset/blocked-ips/b2c3d4e5"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,