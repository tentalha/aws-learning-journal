# Aurora — Hands-On Labs

## Lab 1: Getting Started with Aurora

### Objective
In this lab, you will create an Amazon Aurora MySQL-Compatible cluster, connect to it using a MySQL client, create a sample database schema, and insert and query data. By the end, you will understand Aurora's cluster architecture (writer and reader endpoints), how to create a DB cluster via the AWS Console and CLI, and how to perform basic database operations.

### Prerequisites

**AWS Services Required:**
- Amazon Aurora (MySQL 8.0-compatible)
- Amazon EC2 (bastion host for database access)
- Amazon VPC (default VPC is acceptable)
- AWS Secrets Manager (optional but recommended)

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:CreateDBCluster",
        "rds:CreateDBInstance",
        "rds:DescribeDBClusters",
        "rds:DescribeDBInstances",
        "rds:DeleteDBCluster",
        "rds:DeleteDBInstance",
        "rds:ModifyDBCluster",
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
- MySQL client (`mysql`) installed locally or on a bastion EC2 instance
- A terminal / shell environment
- Default VPC with at least 2 subnets in different Availability Zones

**Environment Variables (set these before starting):**
```bash
export AWS_REGION="us-east-1"
export DB_CLUSTER_ID="aurora-lab1-cluster"
export DB_INSTANCE_ID="aurora-lab1-instance-1"
export DB_NAME="labdb"
export DB_USER="labadmin"
export DB_PASSWORD="Lab@Password123!"   # Change this!
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" \
  --output text \
  --region $AWS_REGION)
echo "Using VPC: $VPC_ID"
```

---

### Steps

#### Step 1: Create a DB Subnet Group

A DB subnet group tells Aurora which subnets it may use. Aurora requires subnets in **at least two** different Availability Zones.

**Console:**
1. Open the [Amazon RDS Console](https://console.aws.amazon.com/rds).
2. In the left navigation pane, choose **Subnet groups**.
3. Click **Create DB subnet group**.
4. Fill in:
   - **Name:** `aurora-lab1-subnet-group`
   - **Description:** `Subnet group for Aurora Lab 1`
   - **VPC:** Select your default VPC
5. Under **Add subnets**, select all available Availability Zones and their corresponding subnets.
6. Click **Create**.

**CLI:**
```bash
# Retrieve subnet IDs from the default VPC
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[*].SubnetId" \
  --output text \
  --region $AWS_REGION | tr '\t' ' ')

echo "Subnets found: $SUBNET_IDS"

# Create the subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name "aurora-lab1-subnet-group" \
  --db-subnet-group-description "Subnet group for Aurora Lab 1" \
  --subnet-ids $SUBNET_IDS \
  --region $AWS_REGION
```

**Verify:**
```bash
aws rds describe-db-subnet-groups \
  --db-subnet-group-name "aurora-lab1-subnet-group" \
  --query "DBSubnetGroups[0].SubnetGroupStatus" \
  --output text \
  --region $AWS_REGION
```
**Expected output:** `Complete`

---

#### Step 2: Create a Security Group for Aurora

**Console:**
1. Open the [VPC Console](https://console.aws.amazon.com/vpc).
2. Choose **Security Groups** → **Create security group**.
3. Fill in:
   - **Name:** `aurora-lab1-sg`
   - **Description:** `Security group for Aurora Lab 1`
   - **VPC:** Select your default VPC
4. Under **Inbound rules**, add:
   - **Type:** MySQL/Aurora | **Port:** 3306 | **Source:** Your IP (`My IP`)
5. Click **Create security group**.

**CLI:**
```bash
# Create the security group
SG_ID=$(aws ec2 create-security-group \
  --group-name "aurora-lab1-sg" \
  --description "Security group for Aurora Lab 1" \
  --vpc-id $VPC_ID \
  --query "GroupId" \
  --output text \
  --region $AWS_REGION)

echo "Security Group ID: $SG_ID"

# Get your current public IP
MY_IP=$(curl -s https://checkip.amazonaws.com)/32
echo "Your IP: $MY_IP"

# Allow MySQL traffic from your IP
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 3306 \
  --cidr $MY_IP \
  --region $AWS_REGION
```

**Verify:**
```bash
aws ec2 describe-security-groups \
  --group-ids $SG_ID \
  --query "SecurityGroups[0].IpPermissions" \
  --output table \
  --region $AWS_REGION
```
**Expected output:** A table showing port 3306 open to your IP CIDR.

---

#### Step 3: Create the Aurora DB Cluster

**Console:**
1. In the RDS Console, click **Create database**.
2. Choose:
   - **Standard create**
   - **Engine type:** Amazon Aurora
   - **Edition:** Amazon Aurora MySQL-Compatible Edition
   - **Engine version:** Aurora MySQL 8.0 (latest available)
3. Under **Templates**, select **Dev/Test**.
4. Under **Settings**:
   - **DB cluster identifier:** `aurora-lab1-cluster`
   - **Master username:** `labadmin`
   - **Master password:** `Lab@Password123!`
5. Under **Instance configuration**:
   - **DB instance class:** `db.t3.medium`
6. Under **Availability & durability**:
   - **Multi-AZ deployment:** Do not create an Aurora Replica (for Lab 1)
7. Under **Connectivity**:
   - **VPC:** Default VPC
   - **DB subnet group:** `aurora-lab1-subnet-group`
   - **Public access:** Yes (for this lab only)
   - **VPC security group:** Choose existing → `aurora-lab1-sg`
8. Under **Additional configuration**:
   - **Initial database name:** `labdb`
9. Click **Create database**.

**CLI:**
```bash
# Create the Aurora cluster
aws rds create-db-cluster \
  --db-cluster-identifier $DB_CLUSTER_ID \
  --engine aurora-mysql \
  --engine-version "8.0.mysql_aurora.3.04.1" \
  --master-username $DB_USER \
  --master-user-password $DB_PASSWORD \
  --database-name $DB_NAME \
  --db-subnet-group-name "aurora-lab1-subnet-group" \
  --vpc-security-group-ids $SG_ID \
  --backup-retention-period 1 \
  --region $AWS_REGION

# Create the primary (writer) DB instance
aws rds create-db-instance \
  --db-instance-identifier $DB_INSTANCE_ID \
  --db-cluster-identifier $DB_CLUSTER_ID \
  --engine aurora-mysql \
  --db-instance-class db.t3.medium \
  --publicly-accessible \
  --region $AWS_REGION
```

**Verify (wait for cluster to become available — typically 5–10 minutes):**
```bash
# Poll cluster status
aws rds wait db-cluster-available \
  --db-cluster-identifier $DB_CLUSTER_ID \
  --region $AWS_REGION

# Check cluster status
aws rds describe-db-clusters \
  --db-cluster-identifier $DB_CLUSTER_ID \
  --query "DBClusters[0].{Status:Status,Endpoint:Endpoint,ReaderEndpoint:ReaderEndpoint}" \
  --output table \
  --region $AWS_REGION
```
**Expected output:**
```
----------------------------------------------------------
|                   DescribeDBClusters                   |
+---------------+----------------------------------------+
|  Endpoint     | aurora-lab1-cluster.cluster-xxxx...    |
|  ReaderEndpoint| aurora-lab1-cluster.cluster-ro-xxxx...|
|  Status       | available                              |
+---------------+----------------------------------------+
```

---

#### Step 4: Retrieve Connection Endpoints

**CLI:**
```bash
# Get the writer endpoint
WRITER_ENDPOINT=$(aws rds describe-db-clusters \
  --db-cluster-identifier $DB_CLUSTER_ID \
  --query "DBClusters[0].Endpoint" \
  --output text \
  --region $AWS_REGION)

# Get the reader endpoint
READER_ENDPOINT=$(aws rds describe-db-clusters \
  --db-cluster-identifier $DB_CLUSTER_ID \
  --query "DBClusters[0].ReaderEndpoint" \
  --output text \
  --region $AWS_REGION)

echo "Writer Endpoint: $WRITER_ENDPOINT"
echo "Reader Endpoint: $READER_ENDPOINT"
```

---

#### Step 5: Connect to Aurora and Create a Schema

```bash
# Connect to the writer endpoint
mysql -h $WRITER_ENDPOINT \
      -P 3306 \
      -u $DB_USER \
      -p$DB_PASSWORD \
      $DB_NAME
```

Once connected, run the following SQL:

```sql
-- Verify connection
SELECT VERSION(), @@aurora_version\G

-- Create a sample schema
CREATE TABLE IF NOT EXISTS products (
    product_id   INT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    category     VARCHAR(50),
    price        DECIMAL(10,2),
    stock_qty    INT DEFAULT 0,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO products (product_name, category, price, stock_qty) VALUES
  ('Wireless Mouse',     'Electronics', 29.99,  150),
  ('Mechanical Keyboard','Electronics', 89.99,   75),
  ('USB-C Hub',          'Electronics', 49.99,  200),
  ('Standing Desk Mat',  'Office',      39.99,  300),
  ('Monitor Stand',      'Office',      59.99,  100);

-- Verify the data
SELECT * FROM products;

-- Show Aurora-specific status
SHOW STATUS LIKE 'Aurora%';
```

**Expected output:**
```
+------------+-----------------------+-------------+-------+-----------+
| product_id | product_name          | category    | price | stock_qty |
+------------+-----------------------+-------------+-------+-----------+
|          1 | Wireless Mouse        | Electronics | 29.99 |       150 |
|          2 | Mechanical Keyboard   | Electronics | 89.99 |        75 |
|          3 | USB-C Hub             | Electronics | 49.99 |       200 |
|          4 | Standing Desk Mat     | Office      | 39.99 |       300 |
|          5 | Monitor Stand         | Office      | 59.99 |       100 |
+------------+-----------------------+-------------+-------+-----------+
5 rows in set (0.01 sec)
```

---

#### Step 6: Test the Reader Endpoint

```bash
# Connect via the reader endpoint (read-only)
mysql -h $READER_ENDPOINT \
      -P 3306 \
      -u $DB_USER \
      -p$DB_PASSWORD \
      $DB_NAME
```

```sql
-- This should succeed (read)
SELECT product_name, price FROM products WHERE category = 'Electronics';

-- Attempt a write — this should FAIL on the reader endpoint
-- (when a replica is present; with only one instance, reader = writer)
INSERT INTO products (product_name, category, price) VALUES ('Test', 'Test', 0.01);

-- Check which instance you're connected to
SELECT @@aurora_server_id;
```

---

### Verification

Run the following commands to confirm successful lab completion:

```bash
# 1. Cluster is available
aws rds describe-db-clusters \
  --db-cluster-identifier $DB_CLUSTER_ID \
  --query "DBClusters[0].Status" \
  --output text \
  --region $AWS_REGION
# Expected: available

# 2. Instance is available
aws rds describe-db-instances \
  --db-instance-identifier $DB_INSTANCE_ID \
  --query "DBInstances[0].DBInstanceStatus" \
  --output text \
  --region $AWS_REGION
# Expected: available

# 3. Verify data from CLI using a query (requires mysql client)
mysql -h $WRITER_ENDPOINT -u $DB_USER -p$DB_PASSWORD $DB_NAME \
  -e "SELECT COUNT(*) AS total_products FROM products;"
# Expected: total_products = 5
```

---

### Cleanup

> ⚠️ **Important:** Aurora charges accrue per instance-hour. Always clean up after labs.

```bash
# Step 1: Delete the DB instance first
aws rds delete-db-instance \
  --db-instance-identifier $DB_INSTANCE_ID \
  --skip-final-snapshot \
  --region $AWS_REGION

echo "Waiting for instance deletion..."
aws rds wait db-instance-deleted \
  --db-instance-identifier $DB_INSTANCE_ID \
  --region $AWS_REGION

# Step 2: Delete the DB cluster
aws rds delete-db-cluster \
  --db-cluster-identifier $DB_CLUSTER_ID \
  --skip-final-snapshot \
  --region $AWS_REGION

echo "Waiting for cluster deletion..."
aws rds wait db-cluster-deleted \
  --db-cluster-identifier $DB_CLUSTER_ID \
  --region $AWS_REGION

# Step 3: Delete the DB subnet group
aws rds delete-db-subnet-group \
  --db-subnet-group-name "aurora-lab1-subnet-group" \
  --region $AWS_REGION

# Step 4: Delete the security group
aws ec2 delete-security-group \
  --group-id $SG_ID \
  --region $AWS_REGION

echo "Cleanup complete!"
```

**Console verification:** Navigate to RDS → Databases and confirm no clusters remain.

---

## Lab 2: Intermediate Aurora Configuration

### Objective
In this lab, you will configure an Aurora MySQL cluster with a **read replica**, implement **Aurora Auto Scaling** for read replicas, enable **Performance Insights**, configure **Enhanced Monitoring**, set up a **custom parameter group**, and use **Aurora Global Database** concepts. You will also simulate a failover and observe Aurora's fast failover behavior (typically under 30 seconds).

### Prerequisites

**Builds on Lab 1 concepts. Additional requirements:**
- Completion of Lab 1 (or equivalent Aurora cluster knowledge)
- IAM permissions for CloudWatch, Auto Scaling, and RDS Parameter Groups
- `sysbench` or `mysqlslap` installed for load generation
- At least 2 Availability Zones available in your region

**Additional IAM Permissions:**
```json
{
  "Effect": "Allow",
  "Action": [
    "application-autoscaling:RegisterScalableTarget",
    "application-autoscaling:PutScalingPolicy",
    "application-autoscaling:DescribeScalableTargets",
    "cloudwatch:GetMetricStatistics",
    "cloudwatch:DescribeAlarms",
    