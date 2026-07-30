# Inspector — Hands-On Labs

## Lab 1: Getting Started with Inspector

### Objective
In this lab, you will enable AWS Inspector v2 for your AWS account, configure it to scan EC2 instances and Amazon ECR container images for software vulnerabilities and unintended network exposure. By the end of this lab, you will understand how Inspector integrates with AWS Systems Manager (SSM Agent), how to navigate the Inspector dashboard, and how to interpret your first vulnerability findings.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account with billing enabled |
| IAM Permissions | `AmazonInspector2FullAccess`, `AmazonSSMManagedInstanceCore`, `AmazonEC2FullAccess`, `AmazonECR_FullAccess` |
| AWS CLI | Version 2.x installed and configured (`aws configure`) |
| Region | `us-east-1` (all commands use this region) |
| EC2 Key Pair | Optional — not required for SSM-based access |

**IAM Policy Check (CLI):**
```bash
aws iam get-user --query 'User.UserName' --output text
aws iam list-attached-user-policies \
  --user-name $(aws iam get-user --query 'User.UserName' --output text) \
  --output table
```

---

### Steps

#### Step 1: Launch a Test EC2 Instance with SSM Agent

**Console:**
1. Navigate to **EC2** → **Launch Instance**
2. Name: `inspector-lab1-target`
3. AMI: **Amazon Linux 2023** (latest)
4. Instance type: `t3.micro`
5. Key pair: **Proceed without a key pair** (we'll use SSM)
6. Under **Advanced details** → **IAM instance profile**, click **Create new IAM role**
   - Attach policy: `AmazonSSMManagedInstanceCore`
   - Role name: `inspector-lab1-ec2-role`
7. Leave storage at default (8 GiB gp3)
8. Click **Launch Instance**

**CLI:**
```bash
# Step 1a: Create the IAM role for SSM
aws iam create-role \
  --role-name inspector-lab1-ec2-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Step 1b: Attach SSM managed policy
aws iam attach-role-policy \
  --role-name inspector-lab1-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Step 1c: Create instance profile
aws iam create-instance-profile \
  --instance-profile-name inspector-lab1-ec2-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name inspector-lab1-ec2-profile \
  --role-name inspector-lab1-ec2-role

# Step 1d: Get default VPC and subnet
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text --region us-east-1)

SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[0].SubnetId' --output text --region us-east-1)

AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" \
             "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' \
  --output text --region us-east-1)

echo "VPC: $VPC_ID | Subnet: $SUBNET_ID | AMI: $AMI_ID"

# Step 1e: Launch the instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --iam-instance-profile Name=inspector-lab1-ec2-profile \
  --subnet-id $SUBNET_ID \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=inspector-lab1-target},{Key=Environment,Value=lab}]' \
  --region us-east-1 \
  --query 'Instances[0].InstanceId' --output text)

echo "Launched Instance ID: $INSTANCE_ID"
```

**Verify:**
```bash
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text --region us-east-1
```
> ✅ Expected output: `running`

---

#### Step 2: Enable AWS Inspector v2

**Console:**
1. Navigate to **AWS Inspector** (search in the console search bar)
2. Click **Get started**
3. On the **Welcome to Inspector** screen, ensure both scan types are checked:
   - ✅ **Amazon EC2 scanning**
   - ✅ **Amazon ECR container image scanning**
4. Click **Enable Inspector**
5. Wait approximately 60 seconds for activation to complete

**CLI:**
```bash
# Enable Inspector v2 for EC2 and ECR
aws inspector2 enable \
  --resource-types EC2 ECR \
  --region us-east-1

# Verify activation status
aws inspector2 batch-get-account-status \
  --account-ids $(aws sts get-caller-identity --query Account --output text) \
  --region us-east-1
```

**Verify:**
```bash
aws inspector2 list-coverage \
  --region us-east-1 \
  --query 'coveredResources[?resourceType==`AWS_EC2_INSTANCE`].{ID:resourceId,Status:scanStatus.statusCode}' \
  --output table
```
> ✅ Expected output: Your EC2 instance listed with `ACTIVE` scan status (may take 2–5 minutes to appear)

---

#### Step 3: Verify SSM Agent Connectivity

**Console:**
1. Navigate to **Systems Manager** → **Fleet Manager**
2. Confirm `inspector-lab1-target` appears with status **Online**

**CLI:**
```bash
# Wait for SSM registration (up to 2 minutes)
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query 'InstanceInformationList[0].{ID:InstanceId,PingStatus:PingStatus,AgentVersion:AgentVersion}' \
  --output table --region us-east-1
```
> ✅ Expected output: `PingStatus: Online`

---

#### Step 4: Explore the Inspector Dashboard

**Console:**
1. Navigate to **AWS Inspector** → **Dashboard**
2. Review the **Risk-based remediations** panel — note the top vulnerable packages
3. Click **Findings** in the left navigation
4. Apply filter: **Severity** = `CRITICAL` or `HIGH`
5. Click any finding to review:
   - **Vulnerability details** (CVE ID, CVSS score)
   - **Affected packages**
   - **Remediation** (suggested fix)
   - **Inspector score**

**CLI:**
```bash
# List all findings for your account
aws inspector2 list-findings \
  --region us-east-1 \
  --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_EC2_INSTANCE"}]}' \
  --sort-criteria '{"field":"INSPECTOR_SCORE","sortOrder":"DESC"}' \
  --max-results 10 \
  --query 'findings[*].{Title:title,Severity:severity,Score:inspectorScore,CVE:packageVulnerabilityDetails.vulnerabilityId}' \
  --output table

# Count findings by severity
aws inspector2 list-findings \
  --region us-east-1 \
  --query 'findings[*].severity' \
  --output text | sort | uniq -c | sort -rn
```

> ✅ Expected output: A list of CVE findings with severity ratings. Even a freshly launched Amazon Linux 2023 instance may show some findings.

---

#### Step 5: Review Finding Details

**CLI:**
```bash
# Get the ARN of the first critical or high finding
FINDING_ARN=$(aws inspector2 list-findings \
  --region us-east-1 \
  --filter-criteria '{
    "severity":[
      {"comparison":"EQUALS","value":"CRITICAL"},
      {"comparison":"EQUALS","value":"HIGH"}
    ]
  }' \
  --max-results 1 \
  --query 'findings[0].findingArn' \
  --output text)

echo "Finding ARN: $FINDING_ARN"

# Get detailed information about this finding
aws inspector2 batch-get-findings \
  --finding-arns "$FINDING_ARN" \
  --region us-east-1 \
  --query 'findings[0].{
    Title:title,
    Description:description,
    Severity:severity,
    Score:inspectorScore,
    CVE:packageVulnerabilityDetails.vulnerabilityId,
    FixedVersion:packageVulnerabilityDetails.vulnerablePackages[0].fixedInVersion,
    Remediation:remediation.recommendation.text
  }' \
  --output json
```

> ✅ Expected output: Detailed JSON with CVE information, CVSS score, affected package version, and remediation recommendation.

---

### Verification

Run this verification script to confirm Lab 1 is complete:

```bash
#!/bin/bash
echo "=== Lab 1 Verification ==="

# Check 1: Inspector is enabled
STATUS=$(aws inspector2 batch-get-account-status \
  --account-ids $(aws sts get-caller-identity --query Account --output text) \
  --region us-east-1 \
  --query 'accounts[0].state.status' --output text)
echo "Inspector Status: $STATUS"
[ "$STATUS" == "ENABLED" ] && echo "✅ Inspector is enabled" || echo "❌ Inspector is NOT enabled"

# Check 2: EC2 scanning is active
EC2_STATUS=$(aws inspector2 batch-get-account-status \
  --account-ids $(aws sts get-caller-identity --query Account --output text) \
  --region us-east-1 \
  --query 'accounts[0].resourceState.ec2.status' --output text)
echo "EC2 Scan Status: $EC2_STATUS"
[ "$EC2_STATUS" == "ENABLED" ] && echo "✅ EC2 scanning enabled" || echo "❌ EC2 scanning NOT enabled"

# Check 3: Instance is covered
COVERED=$(aws inspector2 list-coverage \
  --region us-east-1 \
  --query "length(coveredResources[?resourceId=='$INSTANCE_ID'])" \
  --output text)
echo "Instance covered: $COVERED resource(s)"
[ "$COVERED" -ge "1" ] && echo "✅ Instance is being scanned" || echo "❌ Instance NOT found in coverage"

# Check 4: Findings exist
FINDING_COUNT=$(aws inspector2 list-findings \
  --region us-east-1 \
  --query 'length(findings)' --output text)
echo "Total findings: $FINDING_COUNT"
[ "$FINDING_COUNT" -gt "0" ] && echo "✅ Findings detected" || echo "⚠️  No findings yet — wait 5-10 minutes"

echo "=== Verification Complete ==="
```

---

### Cleanup

> ⚠️ **Important:** AWS Inspector charges approximately $0.11 per EC2 instance-month. Always disable Inspector when not in use.

```bash
# Step 1: Terminate the EC2 instance
aws ec2 terminate-instances \
  --instance-ids $INSTANCE_ID \
  --region us-east-1

# Step 2: Disable Inspector v2
aws inspector2 disable \
  --resource-types EC2 ECR \
  --region us-east-1

# Step 3: Remove IAM role and profile
aws iam remove-role-from-instance-profile \
  --instance-profile-name inspector-lab1-ec2-profile \
  --role-name inspector-lab1-ec2-role

aws iam detach-role-policy \
  --role-name inspector-lab1-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam delete-instance-profile \
  --instance-profile-name inspector-lab1-ec2-profile

aws iam delete-role \
  --role-name inspector-lab1-ec2-role

# Step 4: Verify Inspector is disabled
aws inspector2 batch-get-account-status \
  --account-ids $(aws sts get-caller-identity --query Account --output text) \
  --region us-east-1 \
  --query 'accounts[0].state.status' --output text
```
> ✅ Expected: `DISABLED`

---

## Lab 2: Intermediate Inspector Configuration

### Objective
In this lab, you will configure AWS Inspector v2 for container image scanning with Amazon ECR, set up custom suppression rules to manage false positives, create automated finding exports to Amazon S3 via EventBridge, and configure SNS notifications for critical vulnerabilities. You will also explore Inspector's integration with AWS Security Hub.

---

### Prerequisites

| Requirement | Details |
|---|---|
| Lab 1 Completed | Inspector v2 must be enabled |
| IAM Permissions | `AmazonInspector2FullAccess`, `AmazonECR_FullAccess`, `AmazonS3FullAccess`, `AmazonSNSFullAccess`, `AmazonEventBridgeFullAccess`, `SecurityHubFullAccess` |
| Docker | Installed locally OR use AWS Cloud9/CloudShell |
| AWS CLI | v2.x with ECR helper: `amazon-ecr-credential-helper` |
| Region | `us-east-1` |

---

### Steps

#### Step 1: Create an ECR Repository and Enable Enhanced Scanning

**Console:**
1. Navigate to **Amazon ECR** → **Repositories** → **Create repository**
2. Repository name: `inspector-lab2-app`
3. Under **Image scan settings**:
   - Select **Enhanced scanning** (powered by Inspector)
4. Click **Create repository**

**CLI:**
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"
REPO_NAME="inspector-lab2-app"

# Create ECR repository with enhanced scanning
aws ecr create-repository \
  --repository-name $REPO_NAME \
  --image-scanning-configuration scanOnPush=true \
  --region $REGION

# Set repository scanning to enhanced (Inspector-powered)
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{
    "repositoryFilters": [{
      "filter": "inspector-lab2-*",
      "filterType": "WILDCARD"
    }],
    "scanFrequency": "CONTINUOUS_SCAN"
  }]' \
  --region $REGION

# Verify scanning configuration
aws ecr get-registry-scanning-configuration \
  --region $REGION \
  --query 'scanningConfiguration.{ScanType:scanType,Rules:rules}' \
  --output json
```

> ✅ Expected: `scanType: ENHANCED` with your filter rule applied.

---

#### Step 2: Build and Push a Vulnerable Container Image

We'll intentionally use an older base image to generate findings.

```bash
# Step 2a: Create a simple Dockerfile with an older base image
mkdir -p ~/inspector-lab2 && cd ~/inspector-lab2

cat > Dockerfile << 'EOF'
FROM python: