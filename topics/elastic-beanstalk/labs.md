# Elastic Beanstalk — Hands-On Labs

## Lab 1: Getting Started with Elastic Beanstalk

### Objective

In this lab, you will deploy a sample Node.js web application to AWS Elastic Beanstalk using both the AWS Management Console and the Elastic Beanstalk CLI (EB CLI). You will learn how Elastic Beanstalk automatically provisions and manages the underlying infrastructure — including EC2 instances, load balancers, Auto Scaling groups, and security groups — so you can focus on your application code. By the end of this lab, you will have a publicly accessible web application running on Elastic Beanstalk and understand the core concepts of environments, applications, and application versions.

---

### Prerequisites

**AWS Services Used:**
- AWS Elastic Beanstalk
- Amazon EC2
- Amazon S3 (for application bundles)
- AWS IAM

**Required IAM Permissions:**
- `AWSElasticBeanstalkFullAccess` managed policy (or equivalent)
- `IAMFullAccess` or the ability to create roles (for the Beanstalk service role)

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- Python 3.7+ (required for EB CLI)
- EB CLI installed:

```bash
pip install awsebcli --upgrade
eb --version
# Expected: EB CLI 3.x.x (Python 3.x.x)
```

- Node.js 18.x or later (for the sample app)
- A text editor (VS Code recommended)
- An AWS account with a default VPC in your chosen region

**Estimated Cost:** < $0.50 if cleaned up within 2 hours (t3.micro instance)

**Region:** `us-east-1` (N. Virginia) — used throughout this lab

---

### Steps

#### Step 1: Create the Sample Node.js Application

**1.1 — Create the project directory and application files:**

```bash
mkdir eb-lab1-nodejs && cd eb-lab1-nodejs
```

**1.2 — Create the application entry point:**

```bash
cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.send(`
    <html>
      <body style="font-family: Arial; text-align: center; padding: 50px;">
        <h1>🚀 Hello from Elastic Beanstalk!</h1>
        <p>Environment: ${process.env.NODE_ENV || 'development'}</p>
        <p>App Version: ${process.env.APP_VERSION || '1.0.0'}</p>
        <p>Instance ID: ${process.env.EC2_INSTANCE_ID || 'local'}</p>
        <p>Timestamp: ${new Date().toISOString()}</p>
      </body>
    </html>
  `);
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'OK', timestamp: new Date().toISOString() });
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
EOF
```

**1.3 — Create the `package.json`:**

```bash
cat > package.json << 'EOF'
{
  "name": "eb-lab1-nodejs",
  "version": "1.0.0",
  "description": "Elastic Beanstalk Lab 1 - Node.js Sample App",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
EOF
```

**1.4 — Install dependencies:**

```bash
npm install
```

**✅ Verify:** You should see a `node_modules` directory and `package-lock.json` created.

```bash
ls -la
# Expected output:
# app.js  node_modules/  package.json  package-lock.json
```

---

#### Step 2: Initialize the Elastic Beanstalk Application (EB CLI)

**2.1 — Initialize the EB application:**

```bash
eb init eb-lab1-nodejs --platform "node.js-18" --region us-east-1
```

> **Note:** If prompted interactively, select:
> - Region: `us-east-1`
> - Application name: `eb-lab1-nodejs`
> - Platform: `Node.js 18`
> - CodeCommit: `No`
> - SSH keypair: Select an existing key pair or create a new one

**2.2 — Verify the `.elasticbeanstalk/config.yml` was created:**

```bash
cat .elasticbeanstalk/config.yml
```

**Expected output:**

```yaml
branch-defaults:
  default:
    environment: eb-lab1-env
global:
  application_name: eb-lab1-nodejs
  default_ec2_keyname: your-key-pair
  default_platform: Node.js 18 running on 64bit Amazon Linux 2023
  default_region: us-east-1
  sc: null
```

---

#### Step 3: Create the Elastic Beanstalk Environment

**Option A — Using EB CLI (Recommended for this lab):**

```bash
eb create eb-lab1-env \
  --instance-type t3.micro \
  --single-instance \
  --envvars NODE_ENV=production,APP_VERSION=1.0.0
```

> `--single-instance` deploys without a load balancer to minimize cost for this lab.

**Option B — Using AWS Console:**

1. Open the [Elastic Beanstalk Console](https://console.aws.amazon.com/elasticbeanstalk)
2. Click **"Create application"**
3. Fill in:
   - **Application name:** `eb-lab1-nodejs`
   - **Platform:** `Node.js`
   - **Platform branch:** `Node.js 18 running on 64bit Amazon Linux 2023`
4. Under **Application code**, select **"Upload your code"**
5. Create a ZIP of your application:
   ```bash
   zip -r eb-lab1-v1.zip . -x "*.git*" -x "node_modules/*"
   ```
6. Upload `eb-lab1-v1.zip`
7. Click **"Configure more options"** → Set instance type to `t3.micro`
8. Click **"Create environment"**

**✅ Verify (CLI):** Monitor the environment creation:

```bash
eb status
```

Wait approximately 3–5 minutes. Expected final output:

```
Environment details for: eb-lab1-env
  Application name: eb-lab1-nodejs
  Region: us-east-1
  Deployed Version: app-1-...
  Environment ID: e-xxxxxxxxxx
  Platform: arn:aws:elasticbeanstalk:us-east-1::platform/Node.js 18...
  Tier: WebServer-Standard-1.0
  CNAME: eb-lab1-env.xxxxxx.us-east-1.elasticbeanstalk.com
  Updated: 2024-01-15 10:30:00+00:00
  Status: Ready
  Health: Green
```

---

#### Step 4: Access and Verify the Application

**4.1 — Open the application in your browser:**

```bash
eb open
```

This automatically opens your default browser to the Elastic Beanstalk URL.

**4.2 — Get the application URL via CLI:**

```bash
eb status | grep CNAME
# Example: CNAME: eb-lab1-env.us-east-1.elasticbeanstalk.com
```

**4.3 — Test the health endpoint:**

```bash
# Replace with your actual CNAME
BEANSTALK_URL=$(eb status | grep CNAME | awk '{print $2}')
curl -s http://$BEANSTALK_URL/health | python3 -m json.tool
```

**Expected output:**

```json
{
    "status": "OK",
    "timestamp": "2024-01-15T10:35:00.000Z"
}
```

**4.4 — View application logs:**

```bash
eb logs --all
```

---

#### Step 5: Deploy an Application Update

**5.1 — Modify the application to version 2.0:**

```bash
# Update app.js to show v2.0
sed -i "s/App Version: \${process.env.APP_VERSION || '1.0.0'}/App Version: \${process.env.APP_VERSION || '2.0.0'} - UPDATED/" app.js
```

**5.2 — Update the environment variable and deploy:**

```bash
eb setenv APP_VERSION=2.0.0
eb deploy
```

**5.3 — Monitor the deployment:**

```bash
eb events --follow
# Press Ctrl+C once deployment completes
```

**✅ Verify:** Refresh your browser. The page should now show `App Version: 2.0.0 - UPDATED`.

---

### Verification

Run all the following checks to confirm successful lab completion:

```bash
# 1. Check environment health
eb status | grep "Health: Green"

# 2. Verify application version
eb appversion list

# 3. Confirm HTTP 200 response
BEANSTALK_URL=$(eb status | grep CNAME | awk '{print $2}')
curl -o /dev/null -s -w "HTTP Status: %{http_code}\n" http://$BEANSTALK_URL/

# 4. Check environment configuration
eb config

# 5. Verify via AWS CLI
aws elasticbeanstalk describe-environments \
  --application-name eb-lab1-nodejs \
  --environment-names eb-lab1-env \
  --query "Environments[0].{Status:Status,Health:Health,CNAME:CNAME}" \
  --output table
```

**Expected final output:**

```
---------------------------------------------------------------------
|                      DescribeEnvironments                         |
+-------+------+----------------------------------------------------+
| CNAME | Health | Status                                          |
+-------+------+----------------------------------------------------+
| eb-lab1-env.xxx.us-east-1.elasticbeanstalk.com | Green | Ready  |
+-------+------+----------------------------------------------------+
```

---

### Cleanup

> ⚠️ **Important:** Always clean up resources to avoid unexpected charges.

**Step 1 — Terminate the Elastic Beanstalk environment:**

```bash
eb terminate eb-lab1-env --force
```

Wait for the termination to complete (2–3 minutes).

**Step 2 — Delete the Elastic Beanstalk application:**

```bash
aws elasticbeanstalk delete-application \
  --application-name eb-lab1-nodejs \
  --terminate-env-by-force
```

**Step 3 — Remove the S3 bucket created by Elastic Beanstalk:**

```bash
# Find the EB S3 bucket
EB_BUCKET=$(aws s3 ls | grep "elasticbeanstalk-us-east-1" | awk '{print $3}')
echo "Found bucket: $EB_BUCKET"

# Empty and delete (only if this is a dedicated lab account)
# aws s3 rm s3://$EB_BUCKET --recursive
# aws s3 rb s3://$EB_BUCKET
```

> ⚠️ Only delete the S3 bucket if it was created solely for this lab.

**Step 4 — Verify cleanup via Console:**

```bash
aws elasticbeanstalk describe-applications \
  --application-names eb-lab1-nodejs \
  --query "Applications[*].ApplicationName"
# Expected: [] (empty array)
```

**Step 5 — Remove local project files (optional):**

```bash
cd .. && rm -rf eb-lab1-nodejs
```

---

## Lab 2: Intermediate Elastic Beanstalk Configuration

### Objective

In this lab, you will explore advanced Elastic Beanstalk configuration techniques using `.ebextensions` and `.platform` hooks. You will deploy a Python Flask application with a **load-balanced, auto-scaling environment**, configure custom environment properties, set up **Amazon RDS** (PostgreSQL) as a decoupled database, implement **rolling deployments** to achieve zero-downtime updates, and configure **CloudWatch custom metrics and alarms**. By the end, you will understand how to customize Elastic Beanstalk environments beyond the defaults using configuration files.

---

### Prerequisites

**AWS Services Used:**
- AWS Elastic Beanstalk
- Amazon EC2 (Auto Scaling)
- Elastic Load Balancing (ALB)
- Amazon RDS (PostgreSQL)
- Amazon CloudWatch
- AWS IAM

**Required IAM Permissions:**
- `AWSElasticBeanstalkFullAccess`
- `AmazonRDSFullAccess`
- `CloudWatchFullAccess`

**Tools Required:**
- EB CLI (from Lab 1)
- Python 3.9+
- `pip` package manager
- AWS CLI v2

**Estimated Cost:** ~$0.20–$0.50/hour (t3.small EC2 + db.t3.micro RDS). Clean up within 4 hours.

---

### Steps

#### Step 1: Create the Flask Application

**1.1 — Set up the project structure:**

```bash
mkdir eb-lab2-flask && cd eb-lab2-flask

# Create the directory structure
mkdir -p .ebextensions .platform/hooks/postdeploy
```

**1.2 — Create the Flask application:**

```bash
cat > application.py << 'EOF'
import os
import json
from datetime import datetime
from flask import Flask, jsonify, request
import psycopg2

application = Flask(__name__)

def get_db_connection():
    """Connect to RDS PostgreSQL using Beanstalk environment variables."""
    try:
        conn = psycopg2.connect(
            host=os.environ.get('RDS_HOSTNAME', 'localhost'),
            database=os.environ.get('RDS_DB_NAME', 'eblab2db'),
            user=os.environ.get('RDS_USERNAME', 'postgres'),
            password=os.environ.get('RDS_PASSWORD', ''),
            port=os.environ.get('RDS_PORT', '5432'),
            connect_timeout=5
        )
        return conn
    except Exception as e:
        return None

@application.route('/')
def index():
    db_status = "Connected" if get_db_connection() else "Unavailable"
    return jsonify({
        "application": "EB Lab 2 - Flask",
        "version": os.environ.get('APP_VERSION', '1.0.0'),
        "environment": os.environ.get('FLASK_ENV', 'production'),
        "database_status": db_status,
        "instance_id": os.environ.get('EC2_INSTANCE_ID', 'unknown'),
        "timestamp": datetime.utcnow().isoformat()
    })

@application.route('/health')
def health():
    return jsonify({"status": "healthy", "timestamp": datetime.utcnow().isoformat()}), 200

@application.route('/api/items', methods=['GET'])
def get_items():
    conn = get_db_connection()
    if not conn:
        return jsonify({"error": "Database unavailable", "items": []}), 503
    
    try:
        cur = conn.cursor()
        cur.execute('SELECT id, name, created_at FROM items ORDER BY created_at DESC LIMIT 10;')
        rows = cur.fetchall()
        items = [{"id": r[0], "name": r[1], "created_at": str(r[2])} for r in rows]
        cur.close()
        conn.close()
        return jsonify({"items": items, "count": len(items)})
    except Exception as e:
        return jsonify({"error": str(e), "items": []}), 500

@application.route('/api/items', methods=['POST'])
def create_item():
    conn = get_db_connection()
    if not conn:
        return jsonify({"error": "Database unavailable"}