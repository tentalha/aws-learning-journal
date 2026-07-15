# EBS — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Ensure you have the AWS CLI installed and configured before running any EBS commands.

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure credentials and default region
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

### Required IAM Permissions

Attach the following IAM policy to allow full EBS management:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume",
        "ec2:DeleteVolume",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumeStatus",
        "ec2:DescribeVolumeAttribute",
        "ec2:ModifyVolume",
        "ec2:CreateSnapshot",
        "ec2:DeleteSnapshot",
        "ec2:DescribeSnapshots",
        "ec2:CopySnapshot",
        "ec2:CreateTags",
        "ec2:EnableFastSnapshot",
        "ec2:ListSnapshotBlocks",
        "ec2:GetSnapshotBlock",
        "ec2:EnableVolumeIO",
        "ec2:DescribeAvailabilityZones"
      ],
      "Resource": "*"
    }
  ]
}
```

### Verify CLI Access

```bash
# Verify CLI version and connectivity
aws --version
aws ec2 describe-availability-zones --query 'AvailabilityZones[0].ZoneName' --output text
```

---

## Core Commands

### 1. Create an EBS Volume

```bash
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 100 \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125 \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123def456 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=my-data-volume},{Key=Environment,Value=production}]'
```

**What it does:** Creates a new encrypted GP3 EBS volume of 100 GiB in `us-east-1a` with custom IOPS and throughput settings, tagged for easy identification.

**Example Output:**
```json
{
    "VolumeId": "vol-0a1b2c3d4e5f67890",
    "Size": 100,
    "SnapshotId": "",
    "AvailabilityZone": "us-east-1a",
    "State": "creating",
    "CreateTime": "2024-01-15T10:30:00.000Z",
    "Attachments": [],
    "Tags": [
        {"Key": "Name", "Value": "my-data-volume"},
        {"Key": "Environment", "Value": "production"}
    ],
    "VolumeType": "gp3",
    "Iops": 3000,
    "Encrypted": true,
    "KmsKeyId": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123def456",
    "Throughput": 125
}
```

---

### 2. Describe Volumes

```bash
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].{ID:VolumeId,Size:Size,Type:VolumeType,AZ:AvailabilityZone,State:State}' \
  --output table
```

**What it does:** Lists all available (unattached) EBS volumes with a formatted table showing key attributes.

**Example Output:**
```
--------------------------------------------------------------
|                      DescribeVolumes                       |
+-----+------------------+------+------------+--------+------+
| AZ  |       ID         | Size |   State    | Type   |      |
+-----+------------------+------+------------+--------+------+
| us-east-1a | vol-0a1b2c3d4e5f67890 | 100 | available | gp3 |
| us-east-1b | vol-0b2c3d4e5f678901 | 50  | available | gp2 |
+-----+------------------+------+------------+--------+------+
```

---

### 3. Attach a Volume to an EC2 Instance

```bash
aws ec2 attach-volume \
  --volume-id vol-0a1b2c3d4e5f67890 \
  --instance-id i-0abc123def456789 \
  --device /dev/sdf
```

**What it does:** Attaches the specified EBS volume to an EC2 instance at the given device path.

**Example Output:**
```json
{
    "AttachTime": "2024-01-15T10:35:00.000Z",
    "Device": "/dev/sdf",
    "InstanceId": "i-0abc123def456789",
    "State": "attaching",
    "VolumeId": "vol-0a1b2c3d4e5f67890"
}
```

---

### 4. Detach a Volume from an EC2 Instance

```bash
aws ec2 detach-volume \
  --volume-id vol-0a1b2c3d4e5f67890 \
  --instance-id i-0abc123def456789 \
  --device /dev/sdf \
  --force
```

**What it does:** Detaches an EBS volume from an EC2 instance. The `--force` flag forces detachment even if the volume is busy (use with caution — may cause data corruption).

---

### 5. Create a Snapshot

```bash
aws ec2 create-snapshot \
  --volume-id vol-0a1b2c3d4e5f67890 \
  --description "Daily backup of production data volume - $(date +%Y-%m-%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=prod-data-backup},{Key=RetentionDays,Value=30}]'
```

**What it does:** Creates a point-in-time snapshot of the specified EBS volume with a descriptive name and retention tag.

**Example Output:**
```json
{
    "SnapshotId": "snap-0abc123def456789a",
    "VolumeId": "vol-0a1b2c3d4e5f67890",
    "State": "pending",
    "StartTime": "2024-01-15T10:40:00.000Z",
    "Progress": "0%",
    "OwnerId": "123456789012",
    "Description": "Daily backup of production data volume - 2024-01-15",
    "VolumeSize": 100,
    "Encrypted": true
}
```

---

### 6. Describe Snapshots

```bash
aws ec2 describe-snapshots \
  --owner-ids self \
  --filters Name=volume-id,Values=vol-0a1b2c3d4e5f67890 \
  --query 'Snapshots[*].{ID:SnapshotId,State:State,Progress:Progress,StartTime:StartTime,Size:VolumeSize}' \
  --output table
```

**What it does:** Lists all snapshots owned by your account for a specific volume, showing their completion state and progress.

---

### 7. Delete a Snapshot

```bash
aws ec2 delete-snapshot \
  --snapshot-id snap-0abc123def456789a
```

**What it does:** Permanently deletes the specified snapshot. This action is irreversible.

---

### 8. Delete a Volume

```bash
aws ec2 delete-volume \
  --volume-id vol-0a1b2c3d4e5f67890
```

**What it does:** Permanently deletes an EBS volume. The volume must be in `available` state (not attached) before deletion.

---

### 9. Modify a Volume (Resize / Change Type)

```bash
aws ec2 modify-volume \
  --volume-id vol-0a1b2c3d4e5f67890 \
  --size 200 \
  --volume-type gp3 \
  --iops 6000 \
  --throughput 250
```

**What it does:** Modifies an existing EBS volume's size, type, IOPS, or throughput without detaching it. Changes take effect online (Elastic Volumes feature).

**Example Output:**
```json
{
    "VolumeModification": {
        "VolumeId": "vol-0a1b2c3d4e5f67890",
        "ModificationState": "modifying",
        "TargetSize": 200,
        "TargetIops": 6000,
        "TargetVolumeType": "gp3",
        "TargetThroughput": 250,
        "OriginalSize": 100,
        "OriginalIops": 3000,
        "OriginalVolumeType": "gp3",
        "OriginalThroughput": 125,
        "StartTime": "2024-01-15T10:45:00.000Z"
    }
}
```

---

### 10. Describe Volume Modification Status

```bash
aws ec2 describe-volumes-modifications \
  --volume-ids vol-0a1b2c3d4e5f67890 \
  --query 'VolumesModifications[*].{VolumeId:VolumeId,State:ModificationState,Progress:Progress,TargetSize:TargetSize}'
```

**What it does:** Checks the current status of a volume modification operation (modifying, optimizing, completed, or failed).

---

### 11. Copy a Snapshot to Another Region

```bash
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-0abc123def456789a \
  --description "Cross-region backup copy" \
  --destination-region us-west-2 \
  --encrypted \
  --kms-key-id arn:aws:kms:us-west-2:123456789012:key/mrk-xyz789uvw012 \
  --region us-west-2
```

**What it does:** Copies a snapshot from one AWS region to another, optionally re-encrypting with a different KMS key. Useful for DR strategies.

---

### 12. Create a Volume from a Snapshot

```bash
aws ec2 create-volume \
  --snapshot-id snap-0abc123def456789a \
  --availability-zone us-east-1a \
  --volume-type gp3 \
  --iops 3000 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=restored-volume},{Key=RestoredFrom,Value=snap-0abc123def456789a}]'
```

**What it does:** Creates a new EBS volume pre-populated with data from an existing snapshot. Commonly used for restoring backups or cloning volumes.

---

### 13. Enable Fast Snapshot Restore

```bash
aws ec2 enable-fast-snapshot-restores \
  --availability-zones us-east-1a us-east-1b \
  --source-snapshot-ids snap-0abc123def456789a
```

**What it does:** Enables Fast Snapshot Restore (FSR) for a snapshot in specified AZs, allowing volumes created from it to deliver full performance immediately without initialization.

---

### 14. Describe Volume Status

```bash
aws ec2 describe-volume-status \
  --volume-ids vol-0a1b2c3d4e5f67890 \
  --query 'VolumeStatuses[*].{VolumeId:VolumeId,Status:VolumeStatus.Status,Events:Events}'
```

**What it does:** Returns the operational status of a volume including health checks and any scheduled maintenance events.

**Example Output:**
```json
[
    {
        "VolumeId": "vol-0a1b2c3d4e5f67890",
        "Status": "ok",
        "Events": []
    }
]
```

---

### 15. Add Tags to a Volume

```bash
aws ec2 create-tags \
  --resources vol-0a1b2c3d4e5f67890 \
  --tags Key=CostCenter,Value=engineering Key=Owner,Value=devops-team Key=Backup,Value=true
```

**What it does:** Adds or overwrites tags on an existing EBS volume. Tags are key-value pairs used for organization, cost allocation, and automation.

---

## Common Operations

### Create Operations

```bash
# Create a standard GP3 volume
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 50 \
  --volume-type gp3

# Create a high-performance IO2 volume (for databases)
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 500 \
  --volume-type io2 \
  --iops 32000 \
  --encrypted

# Create a volume from a snapshot
aws ec2 create-volume \
  --snapshot-id snap-0abc123def456789a \
  --availability-zone us-east-1a \
  --volume-type gp3

# Create a snapshot of a volume
aws ec2 create-snapshot \
  --volume-id vol-0a1b2c3d4e5f67890 \
  --description "Manual snapshot before maintenance"

# Create multiple snapshots at once (snapshot set)
aws ec2 create-snapshots \
  --instance-specification InstanceId=i-0abc123def456789,ExcludeBootVolume=false \
  --description "Pre-upgrade snapshot set" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=pre-upgrade-snap}]'
```

---

### Read / Describe Operations

```bash
# Describe a specific volume
aws ec2 describe-volumes \
  --volume-ids vol-0a1b2c3d4e5f67890

# Describe all volumes with a specific tag
aws ec2 describe-volumes \
  --filters Name=tag:Environment,Values=production \
  --query 'Volumes[*].{ID:VolumeId,Size:Size,Type:VolumeType,State:State,AZ:AvailabilityZone}' \
  --output table

# Describe volume attributes
aws ec2 describe-volume-attribute \
  --volume-id vol-0a1b2c3d4e5f67890 \
  --attribute autoEnableIO

# Describe a specific snapshot
aws ec2 describe-snapshots \
  --snapshot-ids snap-0abc123def456789a

# Get snapshot details with progress
aws ec2 describe-snapshots \
  --owner-ids self \
  --filters Name=status,Values=pending \
  --query 'Snapshots[*].{ID:SnapshotId,Progress:Progress,VolumeSize:VolumeSize,Description:Description}'
```

---

### Update / Modify Operations

```bash
# Resize volume to larger size
aws ec2 modify-volume \
  --volume-id vol-0a1b2c3d4e5f67890 \
  --size 200

# Upgrade volume type from gp2 to gp3
aws ec2 modify-volume \
  --volume-id vol-0a1b2c3d4e5f67890 \
  --volume-type gp3 \
  --