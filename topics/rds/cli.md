# RDS — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Ensure the AWS CLI is installed and configured before running any RDS commands.

```bash
# Install or update AWS CLI
pip install --upgrade awscli

# Configure credentials and default region
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json

# Verify configuration
aws sts get-caller-identity
```

### Required IAM Permissions

Attach the following managed policies or create a custom policy for RDS operations:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:CreateDBInstance",
        "rds:DeleteDBInstance",
        "rds:ModifyDBInstance",
        "rds:DescribeDBInstances",
        "rds:StartDBInstance",
        "rds:StopDBInstance",
        "rds:RebootDBInstance",
        "rds:CreateDBSnapshot",
        "rds:DeleteDBSnapshot",
        "rds:DescribeDBSnapshots",
        "rds:RestoreDBInstanceFromDBSnapshot",
        "rds:CreateDBSubnetGroup",
        "rds:DescribeDBSubnetGroups",
        "rds:CreateDBParameterGroup",
        "rds:ModifyDBParameterGroup",
        "rds:DescribeDBParameterGroups",
        "rds:ListTagsForResource",
        "rds:AddTagsToResource",
        "rds:CreateDBCluster",
        "rds:DeleteDBCluster",
        "rds:DescribeDBClusters"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
# Attach the AWS managed policy for full RDS access
aws iam attach-user-policy \
  --user-name my-rds-admin \
  --policy-arn arn:aws:iam::aws:policy/AmazonRDSFullAccess

# Verify the RDS service is accessible
aws rds describe-db-instances --query 'DBInstances[].DBInstanceIdentifier'
```

---

## Core Commands

### 1. Create a DB Instance

```bash
aws rds create-db-instance \
  --db-instance-identifier my-mysql-instance \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0.35 \
  --master-username admin \
  --master-user-password MySecurePass123! \
  --allocated-storage 20 \
  --storage-type gp2 \
  --db-name mydatabase \
  --vpc-security-group-ids sg-0a1b2c3d4e5f67890 \
  --db-subnet-group-name my-db-subnet-group \
  --backup-retention-period 7 \
  --multi-az \
  --publicly-accessible false \
  --tags Key=Environment,Value=production Key=Owner,Value=devops-team
```

**What it does:** Provisions a new RDS DB instance with MySQL 8.0, Multi-AZ enabled, 20 GB gp2 storage, 7-day automated backups, and private networking.

**Example Output:**
```json
{
  "DBInstance": {
    "DBInstanceIdentifier": "my-mysql-instance",
    "DBInstanceClass": "db.t3.micro",
    "Engine": "mysql",
    "DBInstanceStatus": "creating",
    "MasterUsername": "admin",
    "DBName": "mydatabase",
    "AllocatedStorage": 20,
    "MultiAZ": true,
    "EngineVersion": "8.0.35",
    "PubliclyAccessible": false
  }
}
```

---

### 2. Describe DB Instances

```bash
aws rds describe-db-instances \
  --db-instance-identifier my-mysql-instance \
  --query 'DBInstances[0].{ID:DBInstanceIdentifier,Status:DBInstanceStatus,Endpoint:Endpoint.Address,Class:DBInstanceClass}'
```

**What it does:** Retrieves metadata and current status of a specific RDS instance. The `--query` filter returns only the most useful fields.

**Example Output:**
```json
{
  "ID": "my-mysql-instance",
  "Status": "available",
  "Endpoint": "my-mysql-instance.cabcdefghijk.us-east-1.rds.amazonaws.com",
  "Class": "db.t3.micro"
}
```

---

### 3. List All DB Instances

```bash
aws rds describe-db-instances \
  --query 'DBInstances[*].{ID:DBInstanceIdentifier,Engine:Engine,Status:DBInstanceStatus,Class:DBInstanceClass,MultiAZ:MultiAZ}' \
  --output table
```

**What it does:** Lists all RDS instances in the current account and region in a readable table format.

**Example Output:**
```
--------------------------------------------------------------------------------------------
|                                  DescribeDBInstances                                     |
+------------------+----------+-----------+------------+--------+
|     Class        | Engine   |    ID     | MultiAZ    | Status |
+------------------+----------+-----------+------------+--------+
|  db.t3.micro     | mysql    | my-mysql  | True       | available |
|  db.t3.small     | postgres | my-pg-db  | False      | available |
+------------------+----------+-----------+------------+--------+
```

---

### 4. Create a DB Snapshot

```bash
aws rds create-db-snapshot \
  --db-instance-identifier my-mysql-instance \
  --db-snapshot-identifier my-mysql-snapshot-2024-01-15 \
  --tags Key=CreatedBy,Value=cli Key=Date,Value=2024-01-15
```

**What it does:** Creates a manual point-in-time snapshot of a DB instance. Manual snapshots persist until explicitly deleted, unlike automated snapshots.

**Example Output:**
```json
{
  "DBSnapshot": {
    "DBSnapshotIdentifier": "my-mysql-snapshot-2024-01-15",
    "DBInstanceIdentifier": "my-mysql-instance",
    "SnapshotCreateTime": "2024-01-15T10:30:00.000Z",
    "Engine": "mysql",
    "AllocatedStorage": 20,
    "Status": "creating",
    "SnapshotType": "manual"
  }
}
```

---

### 5. Restore DB Instance from Snapshot

```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier my-mysql-restored \
  --db-snapshot-identifier my-mysql-snapshot-2024-01-15 \
  --db-instance-class db.t3.micro \
  --vpc-security-group-ids sg-0a1b2c3d4e5f67890 \
  --db-subnet-group-name my-db-subnet-group \
  --no-publicly-accessible \
  --multi-az
```

**What it does:** Creates a new DB instance from a previously taken snapshot. The restored instance is fully independent of the source.

---

### 6. Modify a DB Instance

```bash
aws rds modify-db-instance \
  --db-instance-identifier my-mysql-instance \
  --db-instance-class db.t3.small \
  --allocated-storage 50 \
  --backup-retention-period 14 \
  --preferred-backup-window "02:00-03:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --apply-immediately
```

**What it does:** Modifies an existing RDS instance — here upgrading the instance class, increasing storage, and adjusting backup windows. `--apply-immediately` applies changes now rather than during the next maintenance window.

---

### 7. Stop a DB Instance

```bash
aws rds stop-db-instance \
  --db-instance-identifier my-mysql-instance
```

**What it does:** Stops a running DB instance to reduce costs. Stopped instances are automatically restarted after 7 days by AWS.

> **Note:** Multi-AZ instances and read replicas can be stopped. Aurora clusters use `stop-db-cluster` instead.

---

### 8. Start a DB Instance

```bash
aws rds start-db-instance \
  --db-instance-identifier my-mysql-instance
```

**What it does:** Starts a previously stopped RDS instance. The instance returns to `available` state within a few minutes.

---

### 9. Reboot a DB Instance

```bash
aws rds reboot-db-instance \
  --db-instance-identifier my-mysql-instance \
  --force-failover
```

**What it does:** Reboots the DB instance. `--force-failover` forces a failover to the standby in a Multi-AZ deployment, useful for testing HA failover scenarios.

---

### 10. Delete a DB Instance

```bash
aws rds delete-db-instance \
  --db-instance-identifier my-mysql-instance \
  --skip-final-snapshot \
  --delete-automated-backups
```

**What it does:** Permanently deletes an RDS instance. Use `--final-db-snapshot-identifier` instead of `--skip-final-snapshot` in production to preserve a final backup.

```bash
# Production-safe deletion with final snapshot
aws rds delete-db-instance \
  --db-instance-identifier my-mysql-instance \
  --final-db-snapshot-identifier my-mysql-final-snapshot-before-delete \
  --delete-automated-backups
```

---

### 11. Create a DB Subnet Group

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name my-db-subnet-group \
  --db-subnet-group-description "Subnet group for production RDS instances" \
  --subnet-ids subnet-0a1b2c3d4e5f67890 subnet-0b2c3d4e5f6789012 subnet-0c3d4e5f678901234 \
  --tags Key=Environment,Value=production
```

**What it does:** Creates a DB subnet group spanning multiple AZs, which is required for placing RDS instances in a VPC and enabling Multi-AZ deployments.

---

### 12. Create a DB Parameter Group

```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name my-mysql8-params \
  --db-parameter-group-family mysql8.0 \
  --description "Custom parameters for MySQL 8.0 production instances"
```

**What it does:** Creates a custom parameter group for MySQL 8.0. Parameter groups allow you to tune engine-level settings like `max_connections`, `slow_query_log`, etc.

---

### 13. Modify DB Parameter Group

```bash
aws rds modify-db-parameter-group \
  --db-parameter-group-name my-mysql8-params \
  --parameters \
    "ParameterName=max_connections,ParameterValue=500,ApplyMethod=pending-reboot" \
    "ParameterName=slow_query_log,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=long_query_time,ParameterValue=2,ApplyMethod=immediate" \
    "ParameterName=innodb_buffer_pool_size,ParameterValue={DBInstanceClassMemory*3/4},ApplyMethod=pending-reboot"
```

**What it does:** Updates specific MySQL engine parameters. `ApplyMethod=immediate` takes effect right away; `pending-reboot` requires an instance restart.

---

### 14. Create a Read Replica

```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier my-mysql-read-replica \
  --source-db-instance-identifier my-mysql-instance \
  --db-instance-class db.t3.micro \
  --availability-zone us-east-1b \
  --publicly-accessible false \
  --tags Key=Role,Value=read-replica
```

**What it does:** Creates a read replica of an existing RDS instance. Read replicas offload read traffic and can be promoted to standalone instances for disaster recovery.

---

### 15. Create an Aurora DB Cluster

```bash
aws rds create-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.04.0 \
  --master-username admin \
  --master-user-password MyAuroraPass123! \
  --db-subnet-group-name my-db-subnet-group \
  --vpc-security-group-ids sg-0a1b2c3d4e5f67890 \
  --backup-retention-period 7 \
  --database-name myauroradb \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-1234abcd12ab34cd56ef1234567890ab \
  --tags Key=Environment,Value=production
```

**What it does:** Creates an Aurora MySQL cluster with encryption at rest. You must then add instances to the cluster using `create-db-instance` with `--db-cluster-identifier`.

---

## Common Operations

### Create Operations

```bash
# Create a PostgreSQL instance
aws rds create-db-instance \
  --db-instance-identifier my-postgres-instance \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 15.4 \
  --master-username pgadmin \
  --master-user-password PgSecurePass123! \
  --allocated-storage 100 \
  --storage-type gp3 \
  --storage-encrypted \
  --db-subnet-group-name my-db-subnet-group \
  --vpc-security-group-ids sg-0a1b2c3d4e5f67890 \
  --backup-retention-period 7 \
  --multi-az \
  --no-publicly-accessible

# Create a DB option group
aws rds create-option-group \
  --option-group-name my-mysql-option-group \
  --engine-name mysql \
  --major-engine-version 8.0 \
  --option-group-description "Custom option group for MySQL 8.0"

# Create an Aurora writer instance
aws rds create-db-instance \
  --db-instance-identifier my-aurora-writer \
  --db-instance-class db.r6g.large \
  --engine aurora-mysql \
  --db-cluster-identifier my-aurora-cluster \
  --publicly-accessible false
```

---

### Read / Describe Operations

```bash
# Describe all DB instances with full output
aws rds describe-db-instances

# Get endpoint and port for a specific instance
aws rds describe-db-instances \
  --db-instance-identifier my-mysql-instance \
  --query 'DBInstances[0].Endpoint.{Host:Address,Port:Port}'

# List all snapshots for a specific instance
aws rds describe-db-snapshots \
  --db-instance-identifier my-mysql-instance \
  --query 'DBSnapshots[*].{ID:DBSnapshotIdentifier,Status:Status,Created:SnapshotCreateTime}' \
  --output table

# Describe DB parameter groups
aws rds describe-db-parameter-groups \
  --query 'DBParameterGroups[*].{Name:DBParameterGroupName,Family:DBParameterGroupFamily}'

# View parameters in a parameter group
aws rds describe-db-parameters \
  --db-parameter-group-name my-mysql8-params \
  --query 'Parameters[?IsModifiable==`true`].[ParameterName,ParameterValue,ApplyType]' \
  --output table

# Describe DB subnet groups
aws rds describe-db-subnet-groups \
  --db-subnet-group-name my-db-subnet-group

# Describe Aurora clusters
aws rds describe-db-clusters \
  --db-cluster-identifier