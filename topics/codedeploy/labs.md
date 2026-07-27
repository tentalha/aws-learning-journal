# CodeDeploy — Hands-On Labs

## Lab 1: Getting Started with CodeDeploy

### Objective

In this lab, you will deploy a simple Node.js web application to an Amazon EC2 instance using AWS CodeDeploy. You will learn how to:

- Create and configure a CodeDeploy application and deployment group
- Install and configure the CodeDeploy agent on an EC2 instance
- Structure an application revision with an `appspec.yml` file
- Trigger a deployment from Amazon S3
- Monitor deployment progress and troubleshoot common issues

By the end of this lab, you will have a working CodeDeploy pipeline that deploys a "Hello World" Node.js application to a single EC2 instance.

---

### Prerequisites

**AWS Services Required:**
- Amazon EC2 (t2.micro or t3.micro)
- AWS CodeDeploy
- Amazon S3
- AWS IAM

**IAM Permissions:**
Your IAM user or role must have the following permissions:
- `codedeploy:*`
- `ec2:*`
- `s3:*`
- `iam:CreateRole`, `iam:AttachRolePolicy`, `iam:PassRole`

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- Git installed locally
- `zip` utility installed locally
- A terminal / shell environment (bash, zsh, or PowerShell)

**Estimated Cost:** < $0.05 (assuming cleanup within 2 hours)

**Estimated Duration:** 45–60 minutes

---

### Steps

#### Step 1: Create Required IAM Roles

You need two IAM roles:
1. **CodeDeploy Service Role** — allows CodeDeploy to call AWS services on your behalf
2. **EC2 Instance Profile** — allows the EC2 instance to communicate with CodeDeploy and S3

##### 1a. Create the CodeDeploy Service Role

**Console:**
1. Open the [IAM Console](https://console.aws.amazon.com/iam/)
2. Click **Roles** → **Create role**
3. Select **AWS service** → **CodeDeploy**
4. Select use case **CodeDeploy** → click **Next**
5. The `AWSCodeDeployRole` policy is pre-selected → click **Next**
6. Name the role: `CodeDeployServiceRole`
7. Click **Create role**

**CLI:**
```bash
# Create the trust policy document
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

# Create the role
aws iam create-role \
  --role-name CodeDeployServiceRole \
  --assume-role-policy-document file://codedeploy-trust-policy.json

# Attach the managed policy
aws iam attach-role-policy \
  --role-name CodeDeployServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole
```

**Verify:** Run the following and confirm the role exists:
```bash
aws iam get-role --role-name CodeDeployServiceRole \
  --query 'Role.RoleName' --output text
# Expected output: CodeDeployServiceRole
```

##### 1b. Create the EC2 Instance Profile

**CLI:**
```bash
# Create trust policy for EC2
cat > ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the EC2 role
aws iam create-role \
  --role-name CodeDeployEC2InstanceRole \
  --assume-role-policy-document file://ec2-trust-policy.json

# Attach S3 read access (for pulling revision from S3)
aws iam attach-role-policy \
  --role-name CodeDeployEC2InstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Attach SSM policy for Session Manager access (optional but recommended)
aws iam attach-role-policy \
  --role-name CodeDeployEC2InstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Create the instance profile
aws iam create-instance-profile \
  --instance-profile-name CodeDeployEC2Profile

# Add role to the instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name CodeDeployEC2Profile \
  --role-name CodeDeployEC2InstanceRole
```

**Verify:**
```bash
aws iam get-instance-profile \
  --instance-profile-name CodeDeployEC2Profile \
  --query 'InstanceProfile.InstanceProfileName' \
  --output text
# Expected output: CodeDeployEC2Profile
```

---

#### Step 2: Launch an EC2 Instance with the CodeDeploy Agent

##### 2a. Find the Latest Amazon Linux 2023 AMI

**CLI:**
```bash
export AWS_REGION=us-east-1

AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-2023*-x86_64" \
    "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text \
  --region $AWS_REGION)

echo "Using AMI: $AMI_ID"
```

##### 2b. Create a Security Group

**CLI:**
```bash
# Get the default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)

# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name CodeDeployLabSG \
  --description "Security group for CodeDeploy Lab 1" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "Security Group ID: $SG_ID"

# Allow HTTP inbound (port 80)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow SSH inbound (port 22) — restrict to your IP in production!
MY_IP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr ${MY_IP}/32
```

##### 2c. Create User Data Script to Install CodeDeploy Agent

```bash
cat > user-data.sh << 'EOF'
#!/bin/bash
yum update -y
yum install -y ruby wget nodejs npm

# Install CodeDeploy agent
cd /home/ec2-user
wget https://aws-codedeploy-us-east-1.s3.us-east-1.amazonaws.com/latest/install
chmod +x ./install
./install auto

# Start and enable the agent
systemctl start codedeploy-agent
systemctl enable codedeploy-agent
EOF
```

##### 2d. Launch the EC2 Instance

**Console:**
1. Open the [EC2 Console](https://console.aws.amazon.com/ec2/)
2. Click **Launch instance**
3. Name: `CodeDeployLab1-Instance`
4. AMI: Amazon Linux 2023
5. Instance type: `t2.micro`
6. Key pair: Select existing or create new
7. Security group: Select `CodeDeployLabSG`
8. Expand **Advanced details** → IAM instance profile → select `CodeDeployEC2Profile`
9. Paste the user data script from above into the **User data** field
10. Click **Launch instance**

**CLI:**
```bash
# Create or use an existing key pair name
KEY_NAME="codedeploy-lab-key"

# Create a new key pair (skip if you have one)
aws ec2 create-key-pair \
  --key-name $KEY_NAME \
  --query 'KeyMaterial' \
  --output text > ${KEY_NAME}.pem
chmod 400 ${KEY_NAME}.pem

# Launch the instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID \
  --iam-instance-profile Name=CodeDeployEC2Profile \
  --user-data file://user-data.sh \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=CodeDeployLab1-Instance},{Key=Environment,Value=Lab}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance ID: $INSTANCE_ID"

# Wait for the instance to be running
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
echo "Instance is running!"
```

**Verify:**
```bash
# Get the public IP
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "Instance Public IP: $PUBLIC_IP"

# Wait ~2 minutes, then SSH in and check the agent
ssh -i ${KEY_NAME}.pem ec2-user@$PUBLIC_IP \
  "sudo systemctl status codedeploy-agent --no-pager"
```

**Expected Output:**
```
● codedeploy-agent.service - AWS CodeDeploy Host Agent
   Loaded: loaded (/usr/lib/systemd/system/codedeploy-agent.service; enabled)
   Active: active (running) since ...
```

---

#### Step 3: Create an S3 Bucket for Application Revisions

**Console:**
1. Open the [S3 Console](https://console.aws.amazon.com/s3/)
2. Click **Create bucket**
3. Name: `codedeploy-lab1-revisions-<your-account-id>` (must be globally unique)
4. Region: `us-east-1`
5. Leave all other defaults → click **Create bucket**

**CLI:**
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="codedeploy-lab1-revisions-${ACCOUNT_ID}"

aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region us-east-1

echo "Bucket created: $BUCKET_NAME"
```

**Verify:**
```bash
aws s3 ls | grep codedeploy-lab1
# Expected: The bucket name appears in the output
```

---

#### Step 4: Create the Application Revision

The revision includes your application code and the `appspec.yml` file that tells CodeDeploy how to deploy it.

##### 4a. Create the Application Directory Structure

```bash
mkdir -p codedeploy-lab1-app/scripts
cd codedeploy-lab1-app
```

##### 4b. Create the Application Files

```bash
# Create the main application file
cat > app.js << 'EOF'
const http = require('http');
const os = require('os');

const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end(`
    <!DOCTYPE html>
    <html>
      <head><title>CodeDeploy Lab 1</title></head>
      <body style="font-family: Arial; text-align: center; padding: 50px;">
        <h1>🚀 Hello from CodeDeploy!</h1>
        <p>Deployed successfully to: <strong>${os.hostname()}</strong></p>
        <p>Version: <strong>1.0.0</strong></p>
        <p>Deployment time: <strong>${new Date().toISOString()}</strong></p>
      </body>
    </html>
  `);
});

server.listen(80, () => {
  console.log('Server running on port 80');
});
EOF

# Create package.json
cat > package.json << 'EOF'
{
  "name": "codedeploy-lab1-app",
  "version": "1.0.0",
  "description": "CodeDeploy Lab 1 Application",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  }
}
EOF
```

##### 4c. Create Lifecycle Hook Scripts

```bash
# Install dependencies script
cat > scripts/install_dependencies.sh << 'EOF'
#!/bin/bash
cd /home/ec2-user/app
npm install
EOF

# Start application script
cat > scripts/start_server.sh << 'EOF'
#!/bin/bash
# Stop any existing node processes
pkill -f "node app.js" || true

# Start the application
cd /home/ec2-user/app
nohup node app.js > /var/log/app.log 2>&1 &

# Wait and verify
sleep 2
if pgrep -f "node app.js" > /dev/null; then
    echo "Application started successfully"
else
    echo "ERROR: Application failed to start"
    exit 1
fi
EOF

# Stop application script
cat > scripts/stop_server.sh << 'EOF'
#!/bin/bash
pkill -f "node app.js" || true
echo "Application stopped"
EOF

# Make scripts executable
chmod +x scripts/*.sh
```

##### 4d. Create the AppSpec File

```bash
cat > appspec.yml << 'EOF'
version: 0.0
os: linux
files:
  - source: /app.js
    destination: /home/ec2-user/app
  - source: /package.json
    destination: /home/ec2-user/app

permissions:
  - object: /home/ec2-user/app
    owner: ec2-user
    group: ec2-user
    mode: 755
    type:
      - directory
      - file

hooks:
  BeforeInstall:
    - location: scripts/stop_server.sh
      timeout: 30
      runas: root
  AfterInstall:
    - location: scripts/install_dependencies.sh
      timeout: 60
      runas: ec2-user
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 30
      runas: root
  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 30
      runas: root
EOF
```

##### 4e. Create the Validation Script

```bash
cat > scripts/validate_service.sh << 'EOF'
#!/bin/bash
# Wait for the service to be ready
sleep 3

# Check if the process is running
if ! pgrep -f "node app.js" > /dev/null; then
    echo "ERROR: Node process not running"
    exit 1
fi

# Check if the HTTP endpoint responds
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:80)
if [ "$HTTP_STATUS" -eq 200 ]; then
    echo "Validation PASSED: HTTP 200 received"
    exit 0
else
    echo "Validation FAILED: HTTP status $HTTP_STATUS"
    exit 1
fi
EOF

chmod +x scripts/validate_service.sh
```

##### 4f. Package and Upload the Revision

```bash
# Go back to parent directory
cd ..

# Create