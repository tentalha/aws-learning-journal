# Aurora — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using AWS CLI with Aurora, ensure the following are in place:

- **AWS CLI version**: 2.x recommended (`aws --version`)
- **Service namespace**: Aurora uses the `rds` CLI namespace (`aws rds ...`)
- **Default region**: Set via `aws configure` or `--region` flag

```bash
# Configure AWS CLI
aws configure set region us-east-1
aws configure set output json

# Verify identity and permissions
aws sts get-caller-identity
```

### Required IAM Permissions

Attach the following managed policies or inline equivalents:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:CreateDBCluster",
        "rds:CreateDBInstance",
        "rds:DeleteDBCluster",
        "rds:DeleteDBInstance",
        "rds:DescribeDBClusters",
        "rds:DescribeDBInstances",
        "rds:ModifyDBCluster",
        "rds:ModifyDBInstance",
        "rds:RestoreDBClusterFromSnapshot",
        "rds:CreateDBClusterSnapshot",
        "rds:DescribeDBClusterSnapshots",
        "rds:FailoverDBCluster",
        "rds:AddTagsToResource",
        "rds:ListTagsForResource",
        "rds:StartDBCluster",
        "rds:StopDBCluster",
        "secretsmanager:CreateSecret",
        "secretsmanager:GetSecretValue",
        "kms:CreateKey",
        "kms:DescribeKey",
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups"
      ],
      "Resource": "*"
    }
  ]
}
```

### Useful Environment Variables

```bash
# Set commonly reused values as environment variables
export AWS_REGION="us-east-1"
export CLUSTER_ID="my-aurora-cluster"
export DB_ENGINE="aurora-mysql"       # or aurora-postgresql
export DB_ENGINE_VERSION="8.0.mysql_aurora.3.04.0"
export MASTER_USER="admin"
export MASTER_PASSWORD="MySecureP@ssw0rd!"
export SUBNET_GROUP="my-db-subnet-group"
export SECURITY_GROUP="sg-0abc123def456789"
```

---

## Core Commands

### 1. Create an Aurora DB Cluster

```bash
aws rds create-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.04.0 \
  --master-username admin \
  --master-user-password MySecureP@ssw0rd! \
  --db-subnet-group-name my-db-subnet-group \
  --vpc-security-group-ids sg-0abc123def456789 \
  --backup-retention-period 7 \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/my-kms-key-id \
  --tags Key=Environment,Value=production Key=Project,Value=myapp \
  --region us-east-1
```

**What it does**: Creates a new Aurora DB cluster with encryption, a 7-day backup retention, and resource tags. This is the foundational step — instances are added separately.

**Example output**:
```json
{
  "DBCluster": {
    "DBClusterIdentifier": "my-aurora-cluster",
    "Status": "creating",
    "Engine": "aurora-mysql",
    "EngineVersion": "8.0.mysql_aurora.3.04.0",
    "Endpoint": "my-aurora-cluster.cluster-cxyz1234.us-east-1.rds.amazonaws.com",
    "ReaderEndpoint": "my-aurora-cluster.cluster-ro-cxyz1234.us-east-1.rds.amazonaws.com",
    "MultiAZ": false,
    "StorageEncrypted": true
  }
}
```

---

### 2. Add a DB Instance to the Cluster

```bash
aws rds create-db-instance \
  --db-instance-identifier my-aurora-instance-1 \
  --db-cluster-identifier my-aurora-cluster \
  --db-instance-class db.r6g.large \
  --engine aurora-mysql \
  --publicly-accessible false \
  --tags Key=Environment,Value=production
```

**What it does**: Adds a writer or reader instance to an existing Aurora cluster. The first instance added becomes the primary (writer).

---

### 3. Describe DB Clusters

```bash
aws rds describe-db-clusters \
  --db-cluster-identifier my-aurora-cluster \
  --query 'DBClusters[*].{ID:DBClusterIdentifier,Status:Status,Endpoint:Endpoint,Engine:Engine,Version:EngineVersion}' \
  --output table
```

**What it does**: Retrieves metadata about a specific Aurora cluster, including endpoints, status, and engine version. The `--query` flag narrows output to key fields.

**Example output**:
```
-----------------------------------------------------------------------------------------------------
|                                        DescribeDBClusters                                         |
+----------+-----------------------------------------------+--------+---------------+--------------+
|  Engine  |                  Endpoint                     |   ID   |    Status     |   Version    |
+----------+-----------------------------------------------+--------+---------------+--------------+
| aurora-  | my-aurora-cluster.cluster-cxyz1234.us-east-1  | my-    |  available    | 8.0.mysql_   |
| mysql    | .rds.amazonaws.com                            | aurora |               | aurora.3.04.0|
+----------+-----------------------------------------------+--------+---------------+--------------+
```

---

### 4. Describe DB Instances

```bash
aws rds describe-db-instances \
  --filters Name=db-cluster-id,Values=my-aurora-cluster \
  --query 'DBInstances[*].{ID:DBInstanceIdentifier,Class:DBInstanceClass,Status:DBInstanceStatus,AZ:AvailabilityZone,Role:ReadReplicaSourceDBInstanceIdentifier}' \
  --output table
```

**What it does**: Lists all instances belonging to a specific Aurora cluster, showing their class, status, and availability zone.

---

### 5. Create a Cluster Snapshot

```bash
aws rds create-db-cluster-snapshot \
  --db-cluster-identifier my-aurora-cluster \
  --db-cluster-snapshot-identifier my-aurora-cluster-snapshot-2024-01-15 \
  --tags Key=CreatedBy,Value=cli Key=Date,Value=2024-01-15
```

**What it does**: Creates a manual snapshot of the entire Aurora cluster. Useful before major changes or as a long-term backup.

**Example output**:
```json
{
  "DBClusterSnapshot": {
    "DBClusterSnapshotIdentifier": "my-aurora-cluster-snapshot-2024-01-15",
    "DBClusterIdentifier": "my-aurora-cluster",
    "Status": "creating",
    "Engine": "aurora-mysql",
    "AllocatedStorage": 0,
    "SnapshotCreateTime": "2024-01-15T10:30:00.000Z"
  }
}
```

---

### 6. Restore a Cluster from Snapshot

```bash
aws rds restore-db-cluster-from-snapshot \
  --db-cluster-identifier my-aurora-cluster-restored \
  --snapshot-identifier my-aurora-cluster-snapshot-2024-01-15 \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.04.0 \
  --db-subnet-group-name my-db-subnet-group \
  --vpc-security-group-ids sg-0abc123def456789 \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/my-kms-key-id
```

**What it does**: Creates a new Aurora cluster from an existing snapshot. Commonly used for disaster recovery or creating test environments from production data.

---

### 7. Modify a DB Cluster

```bash
aws rds modify-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --backup-retention-period 14 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "sun:05:00-sun:06:00" \
  --deletion-protection \
  --apply-immediately
```

**What it does**: Updates cluster-level settings such as backup retention, maintenance windows, and deletion protection. `--apply-immediately` bypasses the next maintenance window.

---

### 8. Failover a DB Cluster

```bash
aws rds failover-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --target-db-instance-identifier my-aurora-instance-2
```

**What it does**: Initiates a manual failover, promoting a specified reader instance to become the new writer. Used for testing HA or planned maintenance.

---

### 9. Start a Stopped DB Cluster

```bash
aws rds start-db-cluster \
  --db-cluster-identifier my-aurora-cluster
```

**What it does**: Starts a previously stopped Aurora cluster. Useful for dev/test environments that don't need to run 24/7.

---

### 10. Stop a DB Cluster

```bash
aws rds stop-db-cluster \
  --db-cluster-identifier my-aurora-cluster
```

**What it does**: Stops all instances in an Aurora cluster to save costs. The cluster can be stopped for up to 7 days before AWS automatically restarts it.

---

### 11. Delete a DB Instance

```bash
aws rds delete-db-instance \
  --db-instance-identifier my-aurora-instance-1 \
  --skip-final-snapshot
```

**What it does**: Removes a specific instance from the cluster. Use `--final-db-snapshot-identifier` instead of `--skip-final-snapshot` in production environments.

---

### 12. Delete a DB Cluster

```bash
aws rds delete-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --final-db-cluster-snapshot-identifier my-aurora-cluster-final-snapshot
```

**What it does**: Deletes the Aurora cluster and creates a final snapshot. All instances must be deleted before the cluster can be removed.

---

### 13. List All Aurora Clusters

```bash
aws rds describe-db-clusters \
  --query 'DBClusters[?Engine==`aurora-mysql` || Engine==`aurora-postgresql`].{ID:DBClusterIdentifier,Engine:Engine,Status:Status,Endpoint:Endpoint}' \
  --output table
```

**What it does**: Lists all Aurora clusters (both MySQL and PostgreSQL-compatible) in the current region with key details.

---

### 14. Add a Reader Instance (Read Replica)

```bash
aws rds create-db-instance \
  --db-instance-identifier my-aurora-reader-1 \
  --db-cluster-identifier my-aurora-cluster \
  --db-instance-class db.r6g.large \
  --engine aurora-mysql \
  --promotion-tier 1 \
  --tags Key=Role,Value=reader
```

**What it does**: Adds a dedicated reader instance to the Aurora cluster. `--promotion-tier` (0–15) determines failover priority; lower numbers are promoted first.

---

### 15. Wait for Cluster to Become Available

```bash
aws rds wait db-cluster-available \
  --db-cluster-identifier my-aurora-cluster
```

**What it does**: Blocks script execution until the specified cluster reaches the `available` state. Essential for automation pipelines that depend on cluster readiness.

---

## Common Operations

### Create Operations

```bash
# Create a DB subnet group (prerequisite)
aws rds create-db-subnet-group \
  --db-subnet-group-name my-db-subnet-group \
  --db-subnet-group-description "Aurora subnet group for production" \
  --subnet-ids subnet-0abc123 subnet-0def456 subnet-0ghi789 \
  --tags Key=Environment,Value=production

# Create a DB cluster parameter group
aws rds create-db-cluster-parameter-group \
  --db-cluster-parameter-group-name my-aurora-cluster-params \
  --db-parameter-group-family aurora-mysql8.0 \
  --description "Custom parameter group for Aurora MySQL 8.0"

# Create a DB parameter group (instance-level)
aws rds create-db-parameter-group \
  --db-parameter-group-name my-aurora-instance-params \
  --db-parameter-group-family aurora-mysql8.0 \
  --description "Custom instance parameter group for Aurora MySQL 8.0"

# Create Aurora cluster with custom parameter groups
aws rds create-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.04.0 \
  --master-username admin \
  --master-user-password MySecureP@ssw0rd! \
  --db-subnet-group-name my-db-subnet-group \
  --vpc-security-group-ids sg-0abc123def456789 \
  --db-cluster-parameter-group-name my-aurora-cluster-params \
  --enable-cloudwatch-logs-exports '["audit","error","general","slowquery"]'
```

---

### Read / Describe Operations

```bash
# Describe a specific cluster with full details
aws rds describe-db-clusters \
  --db-cluster-identifier my-aurora-cluster

# Describe cluster endpoints
aws rds describe-db-cluster-endpoints \
  --db-cluster-identifier my-aurora-cluster \
  --output table

# Describe cluster parameter groups
aws rds describe-db-cluster-parameter-groups \
  --db-cluster-parameter-group-name my-aurora-cluster-params

# Get specific parameters from a parameter group
aws rds describe-db-cluster-parameters \
  --db-cluster-parameter-group-name my-aurora-cluster-params \
  --query 'Parameters[?ParameterName==`max_connections` || ParameterName==`innodb_buffer_pool_size`]'

# Describe available engine versions
aws rds describe-db-engine-versions \
  --engine aurora-mysql \
  --query 'DBEngineVersions[*].{Engine:Engine,Version:EngineVersion,Status:Status}' \
  --output table

# List snapshots for a cluster
aws rds describe-db-cluster-snapshots \
  --db-cluster-identifier my-aurora-cluster \
  --snapshot-type manual \
  --output table

# Get cluster tags
aws rds list-tags-for-resource \
  --resource-name arn:aws:rds:us-east-1:123456789012:cluster:my-aurora-cluster
```

---

### Update / Modify Operations

```bash
# Modify cluster parameter group settings
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name my-aurora-cluster-params \
  --parameters ParameterName=max_connections,ParameterValue=500,ApplyMethod=immediate \
               ParameterName=slow_query_log,ParameterValue=1,ApplyMethod=immediate

# Modify an instance class (scale up/down)
aws rds modify-db-instance \
  --db-instance-identifier my-aurora-instance-1 \
  --db-instance-class db.r6g.2xlarge \
  --apply-immediately

# Enable Performance Insights on an instance
aws rds modify-db-instance \
  --db-instance-identifier my-aurora-instance-1 \
  --enable-performance-insights \
  --performance-insights-retention-period 7 \
  --apply-immediately

# Enable Enhanced Monitoring
aws rds modify-db-instance \
  --db-instance-identifier my-aurora-instance-1 \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds