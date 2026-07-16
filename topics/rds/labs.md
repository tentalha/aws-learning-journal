# RDS — Hands-On Labs

## Lab 1: Getting Started with RDS

### Objective
In this lab, you will launch your first Amazon RDS MySQL instance, connect to it using a MySQL client, create a database schema, insert sample data, and run basic queries. By the end, you will understand the core concepts of RDS instance creation, security group configuration, and database connectivity.

---

### Prerequisites

**AWS Services Required:**
- Amazon RDS
- Amazon VPC (default VPC is acceptable)
- Amazon EC2 (for bastion host / client connection)

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:CreateDBInstance",
        "rds:DescribeDBInstances",
        "rds:DeleteDBInstance",
        "rds:ModifyDBInstance",
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- MySQL client (`mysql`) or MySQL Workbench
- A terminal / shell environment
- An existing default VPC (or a custom VPC with at least one public subnet)

**Estimated Cost:** ~$0.02–$0.05 for a `db.t3.micro` instance running for 1–2 hours (Free Tier eligible if applicable)

**Estimated Duration:** 45–60 minutes

---

### Steps

#### Step 1: Create a Security Group for RDS

**Console:**
1. Navigate to **EC2 → Security Groups → Create security group**.
2. Set the following values:
   - **Security group name:** `rds-lab1-sg`
   - **Description:** `Security group for Lab 1 RDS instance`
   - **VPC:** Select your default VPC
3. Under **Inbound rules**, click **Add rule**:
   - **Type:** MySQL/Aurora
   - **Port:** 3306
   - **Source:** My IP (or `0.0.0.0/0` for lab purposes only — **never use in production**)
4. Click **Create security group**.

**CLI:**
```bash
# Get your default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text)

echo "Default VPC ID: $VPC_ID"

# Create the security group
SG_ID=$(aws ec2 create-security-group \
  --group-name "rds-lab1-sg" \
  --description "Security group for Lab 1 RDS instance" \
  --vpc-id $VPC_ID \
  --query "GroupId" \
  --output text)

echo "Security Group ID: $SG_ID"

# Get your current public IP
MY_IP=$(curl -s https://checkip.amazonaws.com)/32
echo "Your IP: $MY_IP"

# Add inbound rule for MySQL on port 3306
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 3306 \
  --cidr $MY_IP

echo "Inbound rule added for MySQL access."
```

**✅ Verify:** The security group `rds-lab1-sg` appears in the EC2 console with an inbound rule allowing TCP port 3306.

---

#### Step 2: Create an RDS MySQL Instance

**Console:**
1. Navigate to **RDS → Databases → Create database**.
2. Choose **Standard create**.
3. Configure the following:
   - **Engine:** MySQL
   - **Version:** MySQL 8.0.x (latest)
   - **Templates:** Free tier
   - **DB instance identifier:** `rds-lab1-mysql`
   - **Master username:** `admin`
   - **Master password:** `Lab1Password!` (store this securely)
   - **DB instance class:** `db.t3.micro`
   - **Storage type:** gp2, 20 GiB
   - **Multi-AZ:** Disabled (Free Tier)
   - **VPC:** Default VPC
   - **Public access:** Yes (for this lab only)
   - **VPC security groups:** Select `rds-lab1-sg`
   - **Initial database name:** `labdb`
4. Leave all other settings as default and click **Create database**.

**CLI:**
```bash
# Create the RDS MySQL instance
aws rds create-db-instance \
  --db-instance-identifier "rds-lab1-mysql" \
  --db-instance-class "db.t3.micro" \
  --engine "mysql" \
  --engine-version "8.0.35" \
  --master-username "admin" \
  --master-user-password "Lab1Password!" \
  --allocated-storage 20 \
  --storage-type "gp2" \
  --db-name "labdb" \
  --vpc-security-group-ids $SG_ID \
  --publicly-accessible \
  --no-multi-az \
  --backup-retention-period 1 \
  --tags Key=Lab,Value=RDS-Lab1

echo "RDS instance creation initiated. This takes ~5-10 minutes..."
```

**Wait for the instance to become available:**
```bash
# Poll until the instance status is 'available'
aws rds wait db-instance-available \
  --db-instance-identifier "rds-lab1-mysql"

echo "RDS instance is now available!"
```

**✅ Verify:** The RDS instance status shows **Available** in the console or via:
```bash
aws rds describe-db-instances \
  --db-instance-identifier "rds-lab1-mysql" \
  --query "DBInstances[0].DBInstanceStatus" \
  --output text
# Expected output: available
```

---

#### Step 3: Retrieve the RDS Endpoint

**Console:**
1. Navigate to **RDS → Databases → rds-lab1-mysql**.
2. Under the **Connectivity & security** tab, note the **Endpoint** value.

**CLI:**
```bash
# Get the RDS endpoint
RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier "rds-lab1-mysql" \
  --query "DBInstances[0].Endpoint.Address" \
  --output text)

echo "RDS Endpoint: $RDS_ENDPOINT"
```

**Expected output:**
```
RDS Endpoint: rds-lab1-mysql.xxxxxxxxxxxxxxxx.us-east-1.rds.amazonaws.com
```

**✅ Verify:** The endpoint is a valid DNS hostname ending in `.rds.amazonaws.com`.

---

#### Step 4: Connect to the RDS Instance

**Using MySQL CLI:**
```bash
# Connect to the RDS instance
mysql -h $RDS_ENDPOINT \
      -P 3306 \
      -u admin \
      -p'Lab1Password!'

# You should see the MySQL prompt:
# mysql>
```

**Using MySQL Workbench (Console UI):**
1. Open MySQL Workbench.
2. Click **+** to create a new connection.
3. Fill in:
   - **Hostname:** `<RDS_ENDPOINT>`
   - **Port:** `3306`
   - **Username:** `admin`
   - **Password:** `Lab1Password!`
4. Click **Test Connection** → should return "Successfully made the MySQL connection".

**✅ Verify:** You see the `mysql>` prompt or a successful Workbench connection.

---

#### Step 5: Create Schema, Insert Data, and Query

```sql
-- Use the initial database
USE labdb;

-- Create a sample table
CREATE TABLE employees (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    first_name  VARCHAR(50) NOT NULL,
    last_name   VARCHAR(50) NOT NULL,
    department  VARCHAR(50),
    salary      DECIMAL(10,2),
    hire_date   DATE,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO employees (first_name, last_name, department, salary, hire_date) VALUES
  ('Alice',   'Johnson',  'Engineering',  95000.00, '2021-03-15'),
  ('Bob',     'Smith',    'Marketing',    72000.00, '2020-07-01'),
  ('Carol',   'Williams', 'Engineering',  105000.00,'2019-11-20'),
  ('David',   'Brown',    'HR',           68000.00, '2022-01-10'),
  ('Eve',     'Davis',    'Engineering',  98000.00, '2021-09-05');

-- Verify the data
SELECT * FROM employees;

-- Run aggregate queries
SELECT 
    department,
    COUNT(*)          AS headcount,
    AVG(salary)       AS avg_salary,
    MAX(salary)       AS max_salary
FROM employees
GROUP BY department
ORDER BY avg_salary DESC;

-- Check RDS version
SELECT VERSION();
```

**Expected output from the SELECT:**
```
+----+------------+-----------+-------------+-----------+------------+---------------------+
| id | first_name | last_name | department  | salary    | hire_date  | created_at          |
+----+------------+-----------+-------------+-----------+------------+---------------------+
|  1 | Alice      | Johnson   | Engineering | 95000.00  | 2021-03-15 | 2024-01-01 00:00:00 |
|  2 | Bob        | Smith     | Marketing   | 72000.00  | 2020-07-01 | 2024-01-01 00:00:00 |
...
+----+------------+-----------+-------------+-----------+------------+---------------------+
5 rows in set (0.01 sec)
```

**✅ Verify:** All 5 rows are inserted and the aggregate query returns correct department summaries.

---

### Verification

Run the following checklist to confirm successful lab completion:

```bash
# 1. Confirm instance is available
aws rds describe-db-instances \
  --db-instance-identifier "rds-lab1-mysql" \
  --query "DBInstances[0].{Status:DBInstanceStatus,Engine:Engine,Class:DBInstanceClass}" \
  --output table

# Expected:
# -----------------------------------------
# |        DescribeDBInstances            |
# +----------+-----------+----------------+
# |  Class   |  Engine   |    Status      |
# +----------+-----------+----------------+
# | db.t3.micro | mysql  |  available     |
# +----------+-----------+----------------+

# 2. Confirm database exists
mysql -h $RDS_ENDPOINT -u admin -p'Lab1Password!' \
  -e "SHOW DATABASES;" 2>/dev/null
# Expected output includes: labdb

# 3. Confirm table and row count
mysql -h $RDS_ENDPOINT -u admin -p'Lab1Password!' labdb \
  -e "SELECT COUNT(*) AS row_count FROM employees;" 2>/dev/null
# Expected output: row_count = 5
```

---

### Cleanup

> ⚠️ **Important:** Always clean up resources to avoid ongoing charges.

**Console:**
1. Navigate to **RDS → Databases → rds-lab1-mysql → Actions → Delete**.
2. Uncheck **Create final snapshot** (for lab cleanup).
3. Check **I acknowledge...** and type `delete me`.
4. Click **Delete**.
5. Navigate to **EC2 → Security Groups**, select `rds-lab1-sg`, and click **Delete security group**.

**CLI:**
```bash
# 1. Delete the RDS instance (no final snapshot for lab cleanup)
aws rds delete-db-instance \
  --db-instance-identifier "rds-lab1-mysql" \
  --skip-final-snapshot \
  --delete-automated-backups

echo "Waiting for RDS instance deletion..."

# 2. Wait for deletion to complete
aws rds wait db-instance-deleted \
  --db-instance-identifier "rds-lab1-mysql"

echo "RDS instance deleted."

# 3. Delete the security group
aws ec2 delete-security-group --group-id $SG_ID

echo "Security group deleted. Cleanup complete."
```

**✅ Verify Cleanup:**
```bash
# Confirm instance is gone
aws rds describe-db-instances \
  --db-instance-identifier "rds-lab1-mysql" 2>&1
# Expected: An error occurred (DBInstanceNotFound)
```

---

## Lab 2: Intermediate RDS Configuration

### Objective
In this lab, you will configure an RDS MySQL instance with custom parameter groups, enable automated backups and snapshots, set up CloudWatch alarms for key metrics (CPU, storage, connections), enable Performance Insights, and practice point-in-time recovery (PITR). You will also configure RDS Enhanced Monitoring and explore the RDS Event Subscriptions feature.

---

### Prerequisites

**AWS Services Required:**
- Amazon RDS
- Amazon CloudWatch
- Amazon SNS
- Amazon VPC
- IAM

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:*",
        "cloudwatch:PutMetricAlarm",
        "cloudwatch:DescribeAlarms",
        "sns:CreateTopic",
        "sns:Subscribe",
        "sns:ListTopics",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 configured
- MySQL client
- A valid email address (for SNS notifications)

**Estimated Cost:** ~$0.10–$0.20 for a `db.t3.micro` running 2–3 hours

**Estimated Duration:** 90–120 minutes

---

### Steps

#### Step 1: Create a Custom DB Parameter Group

**Console:**
1. Navigate to **RDS → Parameter groups → Create parameter group**.
2. Fill in:
   - **Parameter group family:** `mysql8.0`
   - **Type:** DB Parameter Group
   - **Group name:** `rds-lab2-params`
   - **Description:** `Custom params for Lab 2`
3. Click **Create**.
4. Select `rds-lab2-params` → **Edit parameters**.
5. Search for and modify:
   - `max_connections` → `200`
   - `slow_query_log` → `1`
   - `long_query_time` → `2`
   - `general_log` → `0`
   - `innodb_buffer_pool_size` → `{DBInstanceClassMemory*3/4}`
6. Click **Save changes**.

**CLI:**
```bash
# Create a custom parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name "rds-lab2-params" \
  --db-parameter-group-family "mysql8.0" \
  --description "Custom parameter group for Lab 2"

# Modify key parameters
aws rds modify-db-parameter-group \
  --db-parameter-group-name "rds-lab2-params" \
  --parameters \
    "ParameterName=max_connections,ParameterValue=200,ApplyMethod=immediate" \
    "ParameterName=slow_query_log,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=long_query_time,ParameterValue=2,ApplyMethod=immediate" \
    "ParameterName=general_log,ParameterValue=0,ApplyMethod=immediate"

echo "Parameter group created and configured."

# Verify parameters
aws rds describe-db-parameters \
  --db-parameter-group-name "rds-lab2-params" \
  --query "Parameters[?ParameterName=='slow_query_log' || ParameterName=='max_connections']" \
  --output table
```

**✅ Verify:** The parameter group exists