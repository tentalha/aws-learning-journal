# Elastic Beanstalk — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using the AWS CLI with Elastic Beanstalk, ensure the following are in place:

**AWS CLI Installation & Configuration**
```bash
# Verify AWS CLI version (v2 recommended)
aws --version

# Configure AWS CLI with credentials
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

**Required IAM Permissions**

The IAM user or role must have the following policies attached:

| Policy | Purpose |
|--------|---------|
| `AWSElasticBeanstalkFullAccess` | Full EB management |
| `AmazonEC2FullAccess` | EC2 instance management |
| `AmazonS3FullAccess` | Application version storage |
| `AmazonRDSFullAccess` | Database environment resources |
| `CloudWatchLogsFullAccess` | Log streaming and monitoring |
| `IAMReadOnlyAccess` | Service role lookups |

**Minimum Inline Policy for Read-Only Access**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticbeanstalk:Describe*",
        "elasticbeanstalk:List*",
        "elasticbeanstalk:Check*",
        "elasticbeanstalk:RequestEnvironmentInfo",
        "elasticbeanstalk:RetrieveEnvironmentInfo"
      ],
      "Resource": "*"
    }
  ]
}
```

**Service Role & Instance Profile**
```bash
# Create the Elastic Beanstalk service role (one-time setup)
aws iam create-role \
  --role-name aws-elasticbeanstalk-service-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Principal":{"Service":"elasticbeanstalk.amazonaws.com"},
      "Action":"sts:AssumeRole"
    }]
  }'

# Attach managed policies to the service role
aws iam attach-role-policy \
  --role-name aws-elasticbeanstalk-service-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSElasticBeanstalkEnhancedHealth

aws iam attach-role-policy \
  --role-name aws-elasticbeanstalk-service-role \
  --policy-arn arn:aws:iam::aws:policy/AWSElasticBeanstalkManagedUpdatesCustomerRolePolicy

# Create EC2 instance profile for EB environments
aws iam create-instance-profile \
  --instance-profile-name aws-elasticbeanstalk-ec2-role

aws iam add-role-to-instance-profile \
  --instance-profile-name aws-elasticbeanstalk-ec2-role \
  --role-name aws-elasticbeanstalk-ec2-role
```

**Set Default Region**
```bash
export AWS_DEFAULT_REGION=us-east-1
export AWS_DEFAULT_OUTPUT=json
```

---

## Core Commands

### 1. List Available Solution Stacks (Platforms)
```bash
aws elasticbeanstalk list-available-solution-stacks \
  --query 'SolutionStacks[?contains(@, `Python`)]' \
  --output table
```
**What it does:** Returns all supported platform solution stacks. Useful for finding the correct platform string when creating applications or environments. Filtered here to show only Python stacks.

**Example Output:**
```
-------------------------------------------------------------
|              ListAvailableSolutionStacks                  |
+-----------------------------------------------------------+
| 64bit Amazon Linux 2023 v4.1.0 running Python 3.11       |
| 64bit Amazon Linux 2023 v4.0.11 running Python 3.11      |
| 64bit Amazon Linux 2 v3.6.1 running Python 3.8           |
+-----------------------------------------------------------+
```

---

### 2. Create an Application
```bash
aws elasticbeanstalk create-application \
  --application-name my-web-app \
  --description "My production web application" \
  --resource-lifecycle-config '{
    "ServiceRole": "arn:aws:iam::123456789012:role/aws-elasticbeanstalk-service-role",
    "VersionLifecycleConfig": {
      "MaxCountRule": {
        "Enabled": true,
        "MaxCount": 10,
        "DeleteSourceFromS3": true
      }
    }
  }'
```
**What it does:** Creates a new Elastic Beanstalk application with a lifecycle policy that automatically deletes old application versions (keeping only the 10 most recent), freeing up S3 storage.

**Example Output:**
```json
{
    "Application": {
        "ApplicationArn": "arn:aws:elasticbeanstalk:us-east-1:123456789012:application/my-web-app",
        "ApplicationName": "my-web-app",
        "Description": "My production web application",
        "DateCreated": "2024-01-15T10:30:00.000Z",
        "DateUpdated": "2024-01-15T10:30:00.000Z",
        "ConfigurationTemplates": [],
        "ResourceLifecycleConfig": {
            "ServiceRole": "arn:aws:iam::123456789012:role/aws-elasticbeanstalk-service-role",
            "VersionLifecycleConfig": {
                "MaxCountRule": {
                    "Enabled": true,
                    "MaxCount": 10,
                    "DeleteSourceFromS3": true
                }
            }
        }
    }
}
```

---

### 3. Create an Application Version
```bash
# First, upload your application bundle to S3
aws s3 cp ./my-app.zip s3://my-eb-artifacts-bucket/my-web-app/my-app-v1.0.0.zip

# Create the application version pointing to the S3 artifact
aws elasticbeanstalk create-application-version \
  --application-name my-web-app \
  --version-label v1.0.0 \
  --description "Initial release - January 2024" \
  --source-bundle S3Bucket=my-eb-artifacts-bucket,S3Key=my-web-app/my-app-v1.0.0.zip \
  --auto-create-application \
  --process
```
**What it does:** Creates a versioned application bundle from an S3 artifact. The `--process` flag validates the source bundle before creating the version, and `--auto-create-application` creates the application if it doesn't already exist.

**Example Output:**
```json
{
    "ApplicationVersion": {
        "ApplicationVersionArn": "arn:aws:elasticbeanstalk:us-east-1:123456789012:applicationversion/my-web-app/v1.0.0",
        "ApplicationName": "my-web-app",
        "Description": "Initial release - January 2024",
        "VersionLabel": "v1.0.0",
        "SourceBundle": {
            "S3Bucket": "my-eb-artifacts-bucket",
            "S3Key": "my-web-app/my-app-v1.0.0.zip"
        },
        "DateCreated": "2024-01-15T10:35:00.000Z",
        "DateUpdated": "2024-01-15T10:35:00.000Z",
        "Status": "Processing"
    }
}
```

---

### 4. Create an Environment
```bash
aws elasticbeanstalk create-environment \
  --application-name my-web-app \
  --environment-name my-web-app-prod \
  --description "Production environment" \
  --solution-stack-name "64bit Amazon Linux 2023 v4.1.0 running Python 3.11" \
  --version-label v1.0.0 \
  --tier Name=WebServer,Type=Standard,Version="1.0" \
  --option-settings \
    Namespace=aws:autoscaling:launchconfiguration,OptionName=InstanceType,Value=t3.medium \
    Namespace=aws:autoscaling:launchconfiguration,OptionName=IamInstanceProfile,Value=aws-elasticbeanstalk-ec2-role \
    Namespace=aws:autoscaling:asg,OptionName=MinSize,Value=2 \
    Namespace=aws:autoscaling:asg,OptionName=MaxSize,Value=6 \
    Namespace=aws:elasticbeanstalk:environment,OptionName=ServiceRole,Value=aws-elasticbeanstalk-service-role \
    Namespace=aws:elasticbeanstalk:environment,OptionName=EnvironmentType,Value=LoadBalanced \
    Namespace=aws:elasticbeanstalk:healthreporting:system,OptionName=SystemType,Value=enhanced
```
**What it does:** Creates a production-grade load-balanced environment with enhanced health reporting, auto-scaling between 2–6 instances, and a specific application version deployed.

---

### 5. Describe Environments
```bash
aws elasticbeanstalk describe-environments \
  --application-name my-web-app \
  --environment-names my-web-app-prod \
  --query 'Environments[*].{Name:EnvironmentName,Status:Status,Health:Health,URL:CNAME,Version:VersionLabel}' \
  --output table
```
**What it does:** Retrieves detailed status information for one or more environments, including health, CNAME endpoint, and deployed version.

**Example Output:**
```
---------------------------------------------------------------------------
|                         DescribeEnvironments                            |
+--------------------+--------+--------+---------------------+-----------+
|        Name        | Health | Status |         URL         |  Version  |
+--------------------+--------+--------+---------------------+-----------+
| my-web-app-prod    | Green  | Ready  | my-web-app-prod.us- | v1.0.0    |
|                    |        |        | east-1.elasticbean  |           |
|                    |        |        | stalk.com           |           |
+--------------------+--------+--------+---------------------+-----------+
```

---

### 6. Update an Environment (Deploy New Version)
```bash
aws elasticbeanstalk update-environment \
  --application-name my-web-app \
  --environment-name my-web-app-prod \
  --version-label v1.1.0 \
  --option-settings \
    Namespace=aws:elasticbeanstalk:command,OptionName=DeploymentPolicy,Value=RollingWithAdditionalBatch \
    Namespace=aws:elasticbeanstalk:command,OptionName=BatchSizeType,Value=Percentage \
    Namespace=aws:elasticbeanstalk:command,OptionName=BatchSize,Value=25
```
**What it does:** Deploys a new application version to an existing environment using a `RollingWithAdditionalBatch` deployment policy, which maintains full capacity during the deployment by launching an extra batch of instances.

---

### 7. Describe Application Versions
```bash
aws elasticbeanstalk describe-application-versions \
  --application-name my-web-app \
  --query 'ApplicationVersions[*].{Label:VersionLabel,Status:Status,Created:DateCreated,Description:Description}' \
  --output table
```
**What it does:** Lists all application versions for a given application, showing their labels, processing status, and creation dates.

**Example Output:**
```
--------------------------------------------------------------------------------
|                        DescribeApplicationVersions                           |
+----------+---------+---------------------------+------------------------------+
|  Label   | Status  |          Created          |         Description          |
+----------+---------+---------------------------+------------------------------+
| v1.1.0   | Processed| 2024-01-20T14:00:00.000Z | Hotfix - bug fix release     |
| v1.0.0   | Processed| 2024-01-15T10:35:00.000Z | Initial release - January    |
+----------+---------+---------------------------+------------------------------+
```

---

### 8. Describe Environment Resources
```bash
aws elasticbeanstalk describe-environment-resources \
  --environment-name my-web-app-prod \
  --query 'EnvironmentResources.{
    LoadBalancers:LoadBalancers[*].Name,
    AutoScalingGroups:AutoScalingGroups[*].Name,
    Instances:Instances[*].Id
  }'
```
**What it does:** Returns all AWS resources provisioned for an environment, including load balancers, Auto Scaling groups, EC2 instances, and more. Extremely useful for cross-service troubleshooting.

**Example Output:**
```json
{
    "LoadBalancers": ["awseb-e-abc123def456-stack-AWSEBLB-XYZ789"],
    "AutoScalingGroups": ["awseb-e-abc123def456-stack-AWSEBAutoScalingGroup-ABC123"],
    "Instances": ["i-0abc123def456789a", "i-0def456abc789012b"]
}
```

---

### 9. Retrieve Environment Logs
```bash
# Request log retrieval (triggers log bundling on instances)
aws elasticbeanstalk request-environment-info \
  --environment-name my-web-app-prod \
  --info-type tail

# Wait a few seconds for logs to be collected, then retrieve them
sleep 15

aws elasticbeanstalk retrieve-environment-info \
  --environment-name my-web-app-prod \
  --info-type tail \
  --query 'EnvironmentInfo[*].{Instance:Ec2InstanceId,Message:Message}' \
  --output text
```
**What it does:** Two-step process to fetch tail logs (last 100 lines of each log file) from all instances in an environment. Useful for quick debugging without SSH access.

---

### 10. Terminate an Environment
```bash
aws elasticbeanstalk terminate-environment \
  --environment-name my-web-app-staging \
  --terminate-resources \
  --force-terminate
```
**What it does:** Terminates an environment and all its provisioned AWS resources (EC2 instances, load balancer, Auto Scaling group, etc.). The `--force-terminate` flag bypasses the termination protection check.

> ⚠️ **Warning:** This action is irreversible. The environment and all associated resources will be permanently deleted.

---

### 11. Swap Environment CNAMEs (Blue/Green Deploy)
```bash
aws elasticbeanstalk swap-environment-cnames \
  --source-environment-name my-web-app-blue \
  --destination-environment-name my-web-app-green
```
**What it does:** Atomically swaps the CNAME DNS records between two environments, enabling zero-downtime blue/green deployments. Traffic is instantly redirected from the old environment to the new one.

---

### 12. Rebuild an Environment
```bash
aws elasticbeanstalk rebuild-environment \
  --environment-name my-web-app-prod
```
**What it does:** Terminates and re-launches all EC2 instances in an environment without changing the application version or configuration. Useful for recovering from instance-level corruption or drift.

---

### 13. Describe Configuration Options
```bash
aws elasticbeanstalk describe-configuration-options \
  --application-name my-web-app \
  --environment-name my-web-app-prod \
  --query 'Options[?Namespace==`aws:autoscaling:launchconfiguration`].{Name:Name,Default:DefaultValue,Type:ValueType}' \
  --output table
```
**What it does:** Lists all available configuration options for an environment namespace, showing names, default values, and value types. Essential for