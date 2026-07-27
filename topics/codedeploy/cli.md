# CodeDeploy — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using CodeDeploy CLI commands, ensure the following are in place:

**AWS CLI Installation & Configuration**
```bash
# Verify AWS CLI version (v2 recommended)
aws --version

# Configure default profile
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region: us-east-1
# Default output format: json
```

**Required IAM Permissions**

Attach the following managed policies or equivalent inline policies to your IAM user/role:

| Policy | Purpose |
|--------|---------|
| `AWSCodeDeployFullAccess` | Full access to CodeDeploy resources |
| `AWSCodeDeployDeployerAccess` | Trigger deployments only |
| `AWSCodeDeployReadOnlyAccess` | Read-only access |

```bash
# Attach CodeDeploy full access to an IAM user
aws iam attach-user-policy \
  --user-name my-devops-user \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeDeployFullAccess

# Attach CodeDeploy full access to an IAM role
aws iam attach-role-policy \
  --role-name my-devops-role \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeDeployFullAccess
```

**CodeDeploy Service Role**

CodeDeploy requires a service role to interact with other AWS services:

```bash
# Create trust policy document
cat > codedeploy-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codedeploy.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the service role
aws iam create-role \
  --role-name CodeDeployServiceRole \
  --assume-role-policy-document file://codedeploy-trust-policy.json

# Attach the managed policy
aws iam attach-role-policy \
  --role-name CodeDeployServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole
```

---

## Core Commands

### 1. Create an Application

```bash
aws deploy create-application \
  --application-name my-web-app \
  --compute-platform Server
```

**What it does:** Creates a new CodeDeploy application. The `--compute-platform` can be `Server` (EC2/on-premises), `Lambda`, or `ECS`.

**Example Output:**
```json
{
    "applicationId": "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111"
}
```

---

### 2. Create a Deployment Group

```bash
aws deploy create-deployment-group \
  --application-name my-web-app \
  --deployment-group-name my-deployment-group \
  --deployment-config-name CodeDeployDefault.OneAtATime \
  --ec2-tag-filters Key=Environment,Value=Production,Type=KEY_AND_VALUE \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployServiceRole
```

**What it does:** Creates a deployment group that defines which EC2 instances to deploy to, the deployment configuration, and associated settings.

**Example Output:**
```json
{
    "deploymentGroupId": "b2c3d4e5-6789-01bc-defg-EXAMPLE22222"
}
```

---

### 3. Create a Deployment

```bash
aws deploy create-deployment \
  --application-name my-web-app \
  --deployment-group-name my-deployment-group \
  --deployment-config-name CodeDeployDefault.OneAtATime \
  --s3-location bucket=my-codedeploy-bucket,bundleType=zip,key=deployments/my-app-v1.0.zip \
  --description "Production deployment v1.0"
```

**What it does:** Initiates a deployment using an application revision stored in S3. Returns a deployment ID for tracking.

**Example Output:**
```json
{
    "deploymentId": "d-EXAMPLE123"
}
```

---

### 4. Get Deployment Status

```bash
aws deploy get-deployment \
  --deployment-id d-EXAMPLE123
```

**What it does:** Retrieves detailed information about a specific deployment, including its status, start/end times, and instance summary.

**Example Output:**
```json
{
    "deploymentInfo": {
        "applicationName": "my-web-app",
        "deploymentGroupName": "my-deployment-group",
        "deploymentId": "d-EXAMPLE123",
        "status": "Succeeded",
        "deploymentOverview": {
            "Pending": 0,
            "InProgress": 0,
            "Succeeded": 3,
            "Failed": 0,
            "Skipped": 0,
            "Ready": 0
        },
        "createTime": "2024-01-15T10:30:00.000Z",
        "completeTime": "2024-01-15T10:45:00.000Z"
    }
}
```

---

### 5. List Deployments

```bash
aws deploy list-deployments \
  --application-name my-web-app \
  --deployment-group-name my-deployment-group \
  --include-only-statuses Succeeded Failed InProgress \
  --create-time-range start=2024-01-01T00:00:00,end=2024-12-31T23:59:59
```

**What it does:** Lists deployment IDs for a given application and deployment group, optionally filtered by status and time range.

**Example Output:**
```json
{
    "deployments": [
        "d-EXAMPLE123",
        "d-EXAMPLE456",
        "d-EXAMPLE789"
    ]
}
```

---

### 6. Push a Revision to S3

```bash
aws deploy push \
  --application-name my-web-app \
  --s3-location s3://my-codedeploy-bucket/deployments/my-app-v1.0.zip \
  --ignore-hidden-files \
  --source ./my-app-directory
```

**What it does:** Bundles and uploads application content to S3 and registers the revision with CodeDeploy. Requires the CodeDeploy agent to be installed locally.

---

### 7. Register an Application Revision

```bash
aws deploy register-application-revision \
  --application-name my-web-app \
  --s3-location bucket=my-codedeploy-bucket,key=deployments/my-app-v1.0.zip,bundleType=zip,eTag=abc123def456 \
  --description "Version 1.0 release candidate"
```

**What it does:** Registers a specific S3 object as a revision for a CodeDeploy application without triggering a deployment.

---

### 8. List Application Revisions

```bash
aws deploy list-application-revisions \
  --application-name my-web-app \
  --s3-bucket my-codedeploy-bucket \
  --sort-by registerTime \
  --sort-order descending \
  --deployed exclude
```

**What it does:** Lists all registered revisions for an application, optionally filtered by S3 bucket and sorted by registration time.

**Example Output:**
```json
{
    "revisions": [
        {
            "revisionType": "S3",
            "s3Location": {
                "bucket": "my-codedeploy-bucket",
                "key": "deployments/my-app-v1.0.zip",
                "bundleType": "zip",
                "eTag": "abc123def456"
            }
        }
    ]
}
```

---

### 9. Get Deployment Instance Details

```bash
aws deploy get-deployment-instance \
  --deployment-id d-EXAMPLE123 \
  --instance-id i-0a1b2c3d4e5f67890
```

**What it does:** Returns lifecycle event details for a specific instance within a deployment, useful for debugging failed deployments.

**Example Output:**
```json
{
    "instanceSummary": {
        "deploymentId": "d-EXAMPLE123",
        "instanceId": "arn:aws:ec2:us-east-1:123456789012:instance/i-0a1b2c3d4e5f67890",
        "status": "Failed",
        "lifecycleEvents": [
            {
                "lifecycleEventName": "ApplicationStop",
                "status": "Succeeded"
            },
            {
                "lifecycleEventName": "BeforeInstall",
                "status": "Failed",
                "diagnostics": {
                    "errorCode": "ScriptFailed",
                    "message": "Script at specified location: scripts/before_install.sh failed to complete"
                }
            }
        ]
    }
}
```

---

### 10. Stop a Deployment

```bash
aws deploy stop-deployment \
  --deployment-id d-EXAMPLE123 \
  --auto-rollback-enabled
```

**What it does:** Stops an in-progress deployment. The `--auto-rollback-enabled` flag triggers an automatic rollback to the last successful revision.

**Example Output:**
```json
{
    "status": "Pending",
    "statusMessage": "Deployment stop operation is pending."
}
```

---

### 11. List Deployment Configurations

```bash
aws deploy list-deployment-configs
```

**What it does:** Lists all available deployment configurations, including AWS-predefined ones and any custom configurations you have created.

**Example Output:**
```json
{
    "deploymentConfigsList": [
        "CodeDeployDefault.OneAtATime",
        "CodeDeployDefault.HalfAtATime",
        "CodeDeployDefault.AllAtOnce",
        "CodeDeployDefault.LambdaCanary10Percent5Minutes",
        "CodeDeployDefault.LambdaLinear10PercentEvery1Minute",
        "my-custom-deployment-config"
    ]
}
```

---

### 12. Create a Custom Deployment Configuration

```bash
aws deploy create-deployment-config \
  --deployment-config-name my-custom-deployment-config \
  --minimum-healthy-hosts type=FLEET_PERCENT,value=75 \
  --compute-platform Server
```

**What it does:** Creates a custom deployment configuration specifying the minimum number or percentage of healthy instances required during deployment.

---

### 13. List Applications

```bash
aws deploy list-applications \
  --output table
```

**What it does:** Lists all CodeDeploy applications in the current region and account.

**Example Output:**
```
------------------------------
|      ListApplications      |
+----------------------------+
||       applications       ||
|+---------------------------+|
||  my-web-app              ||
||  my-lambda-app           ||
||  my-ecs-service          ||
|+---------------------------+|
```

---

### 14. Delete a Deployment Group

```bash
aws deploy delete-deployment-group \
  --application-name my-web-app \
  --deployment-group-name my-deployment-group
```

**What it does:** Deletes a deployment group from an application. Does not delete the application itself or any associated instances.

---

### 15. Delete an Application

```bash
aws deploy delete-application \
  --application-name my-web-app
```

**What it does:** Permanently deletes a CodeDeploy application and all its associated deployment groups and revision history.

---

## Common Operations

### Create Operations

```bash
# Create an application for Lambda
aws deploy create-application \
  --application-name my-lambda-app \
  --compute-platform Lambda

# Create an application for ECS
aws deploy create-application \
  --application-name my-ecs-app \
  --compute-platform ECS

# Create a deployment group for Lambda with canary config
aws deploy create-deployment-group \
  --application-name my-lambda-app \
  --deployment-group-name my-lambda-dg \
  --deployment-config-name CodeDeployDefault.LambdaCanary10Percent5Minutes \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployServiceRole \
  --deployment-style deploymentType=BLUE_GREEN,deploymentOption=WITH_TRAFFIC_CONTROL

# Create a deployment group for ECS
aws deploy create-deployment-group \
  --application-name my-ecs-app \
  --deployment-group-name my-ecs-dg \
  --deployment-config-name CodeDeployDefault.ECSCanary10Percent5Minutes \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployServiceRole \
  --ecs-services serviceName=my-ecs-service,clusterName=my-ecs-cluster \
  --load-balancer-info targetGroupPairInfoList=[{targetGroups=[{name=my-target-group-1},{name=my-target-group-2}],prodTrafficRoute={listenerArns=[arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/my-alb/1234567890abcdef/abcdef1234567890]}}]

# Create a deployment from GitHub
aws deploy create-deployment \
  --application-name my-web-app \
  --deployment-group-name my-deployment-group \
  --github-location repository=my-org/my-repo,commitId=abc123def456789
```

---

### Read / Describe Operations

```bash
# Get application details
aws deploy get-application \
  --application-name my-web-app

# Get deployment group details
aws deploy get-deployment-group \
  --application-name my-web-app \
  --deployment-group-name my-deployment-group

# Get deployment configuration details
aws deploy get-deployment-config \
  --deployment-config-name CodeDeployDefault.OneAtATime

# Get on-premises instance details
aws deploy get-on-premises-instance \
  --instance-name my-on-prem-server

# Batch get applications
aws deploy batch-get-applications \
  --application-names my-web-app my-lambda-app my-ecs-app

# Batch get deployments
aws deploy batch-get-deployments \
  --deployment-ids d-EXAMPLE123 d-EXAMPLE456 d-EXAMPLE789

# Get revision details
aws deploy get-application-revision \
  --application-name my-web-app \
  --s3-location bucket=my-codedeploy-bucket,key=deployments/my-app-v1.0.zip,bundleType=zip
```

---

### Update Operations

```bash
# Update an application name
aws deploy update-application \
  --application-name my-web-app \
  --new-application-name my-web-app-v2

# Update a deployment group
aws deploy update-deployment-group \
  --application-name my-web-app \
  --current-deployment-group-name my-deployment-group \
  --new-deployment-group-name my-deployment-group-v2 \
  --deployment-config-name CodeDeployDefault.HalfAtATime \
  --ec2-tag-filters Key=Environment,Value=Production,Type=KEY_AND_VALUE \
  --auto-rollback-configuration enabled=true,events=DEPLOYMENT_FAILURE,DEPLOYMENT_STOP_ON_ALARM \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployServiceRole

# Update deployment group to add auto-scaling groups
aws deploy update-deployment-group \
  --application-name my-web-app \
  --current-deployment-group-name my-deployment-group \
  --auto-scaling-groups my-auto-scaling-group \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployServiceRole
```

---

### Delete Operations

```bash
# Delete a deployment configuration