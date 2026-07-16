# EFS — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using the AWS CLI with Amazon EFS, ensure the following are in place:

**AWS CLI Installation & Configuration:**
```bash
# Install or update AWS CLI
pip install --upgrade awscli

# Configure credentials and default region
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

**Required IAM Permissions:**

Attach the following managed policies or equivalent inline policies to your IAM user/role:

- `AmazonElasticFileSystemFullAccess` — Full EFS management
- `AmazonVPCReadOnlyAccess` — Needed to reference subnets and security groups
- `AmazonEC2ReadOnlyAccess` — Needed to inspect network interfaces

**Minimal IAM Policy for EFS Operations:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticfilesystem:CreateFileSystem",
        "elasticfilesystem:DeleteFileSystem",
        "elasticfilesystem:DescribeFileSystems",
        "elasticfilesystem:CreateMountTarget",
        "elasticfilesystem:DeleteMountTarget",
        "elasticfilesystem:DescribeMountTargets",
        "elasticfilesystem:CreateAccessPoint",
        "elasticfilesystem:DeleteAccessPoint",
        "elasticfilesystem:DescribeAccessPoints",
        "elasticfilesystem:PutFileSystemPolicy",
        "elasticfilesystem:DescribeFileSystemPolicy",
        "elasticfilesystem:PutLifecycleConfiguration",
        "elasticfilesystem:TagResource",
        "elasticfilesystem:UntagResource",
        "elasticfilesystem:ListTagsForResource"
      ],
      "Resource": "*"
    }
  ]
}
```

**Verify CLI Access:**
```bash
aws efs describe-file-systems --region us-east-1
```

---

## Core Commands

### 1. Create a File System

```bash
aws efs create-file-system \
  --creation-token "my-efs-token-$(date +%s)" \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123def456 \
  --tags Key=Name,Value=my-efs-filesystem Key=Environment,Value=production \
  --region us-east-1
```

**What it does:** Creates a new EFS file system with encryption at rest using a KMS key, general purpose performance mode, and bursting throughput. The creation token ensures idempotency.

**Example Output:**
```json
{
    "OwnerId": "123456789012",
    "CreationToken": "my-efs-token-1698765432",
    "FileSystemId": "fs-0abc1234def567890",
    "FileSystemArn": "arn:aws:elasticfilesystem:us-east-1:123456789012:file-system/fs-0abc1234def567890",
    "CreationTime": "2024-01-15T10:30:00+00:00",
    "LifeCycleState": "creating",
    "NumberOfMountTargets": 0,
    "SizeInBytes": {
        "Value": 0,
        "Timestamp": "2024-01-15T10:30:00+00:00",
        "ValueInIA": 0,
        "ValueInStandard": 0
    },
    "PerformanceMode": "generalPurpose",
    "Encrypted": true,
    "KmsKeyId": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123def456",
    "ThroughputMode": "bursting",
    "Tags": [
        {"Key": "Name", "Value": "my-efs-filesystem"},
        {"Key": "Environment", "Value": "production"}
    ]
}
```

---

### 2. Describe File Systems

```bash
aws efs describe-file-systems \
  --region us-east-1 \
  --output table
```

**What it does:** Lists all EFS file systems in the specified region with their current state, size, and configuration details.

**Example Output:**
```
-----------------------------------------------------------------------------------------------
|                                    DescribeFileSystems                                      |
+-----------------------------+-------------------+------------------+------------------------+
|       FileSystemId          | LifeCycleState    | PerformanceMode  |   ThroughputMode       |
+-----------------------------+-------------------+------------------+------------------------+
|  fs-0abc1234def567890       |  available        |  generalPurpose  |  bursting              |
|  fs-0def5678abc901234       |  available        |  maxIO           |  provisioned           |
+-----------------------------+-------------------+------------------+------------------------+
```

---

### 3. Create a Mount Target

```bash
aws efs create-mount-target \
  --file-system-id fs-0abc1234def567890 \
  --subnet-id subnet-0a1b2c3d4e5f67890 \
  --security-groups sg-0123456789abcdef0 \
  --region us-east-1
```

**What it does:** Creates a mount target in a specific subnet, which provides a network endpoint (IP address) for mounting the EFS file system on EC2 instances within that subnet.

**Example Output:**
```json
{
    "OwnerId": "123456789012",
    "MountTargetId": "fsmt-0abc123456def7890",
    "FileSystemId": "fs-0abc1234def567890",
    "SubnetId": "subnet-0a1b2c3d4e5f67890",
    "LifeCycleState": "creating",
    "IpAddress": "10.0.1.45",
    "NetworkInterfaceId": "eni-0abc123456789def0",
    "AvailabilityZoneId": "use1-az1",
    "AvailabilityZoneName": "us-east-1a",
    "VpcId": "vpc-0abc123456789def0",
    "OwnerId": "123456789012"
}
```

---

### 4. Describe Mount Targets

```bash
aws efs describe-mount-targets \
  --file-system-id fs-0abc1234def567890 \
  --region us-east-1
```

**What it does:** Returns information about all mount targets associated with a specific EFS file system, including their IP addresses, subnet locations, and lifecycle states.

**Example Output:**
```json
{
    "MountTargets": [
        {
            "MountTargetId": "fsmt-0abc123456def7890",
            "FileSystemId": "fs-0abc1234def567890",
            "SubnetId": "subnet-0a1b2c3d4e5f67890",
            "LifeCycleState": "available",
            "IpAddress": "10.0.1.45",
            "AvailabilityZoneName": "us-east-1a"
        },
        {
            "MountTargetId": "fsmt-0def456789abc1234",
            "FileSystemId": "fs-0abc1234def567890",
            "SubnetId": "subnet-0b2c3d4e5f6789012",
            "LifeCycleState": "available",
            "IpAddress": "10.0.2.67",
            "AvailabilityZoneName": "us-east-1b"
        }
    ]
}
```

---

### 5. Create an Access Point

```bash
aws efs create-access-point \
  --file-system-id fs-0abc1234def567890 \
  --posix-user Uid=1001,Gid=1001 \
  --root-directory "Path=/app/data,CreationInfo={OwnerUid=1001,OwnerGid=1001,Permissions=755}" \
  --tags Key=Name,Value=my-app-access-point Key=Team,Value=backend \
  --client-token "ap-token-$(date +%s)" \
  --region us-east-1
```

**What it does:** Creates an EFS access point that enforces a specific POSIX user identity and a root directory for all file system requests. Useful for multi-tenant applications or Kubernetes pods.

**Example Output:**
```json
{
    "ClientToken": "ap-token-1698765432",
    "Name": "my-app-access-point",
    "Tags": [
        {"Key": "Name", "Value": "my-app-access-point"},
        {"Key": "Team", "Value": "backend"}
    ],
    "AccessPointId": "fsap-0abc123456789def0",
    "AccessPointArn": "arn:aws:elasticfilesystem:us-east-1:123456789012:access-point/fsap-0abc123456789def0",
    "FileSystemId": "fs-0abc1234def567890",
    "PosixUser": {
        "Uid": 1001,
        "Gid": 1001
    },
    "RootDirectory": {
        "Path": "/app/data",
        "CreationInfo": {
            "OwnerUid": 1001,
            "OwnerGid": 1001,
            "Permissions": "755"
        }
    },
    "OwnerId": "123456789012",
    "LifeCycleState": "creating"
}
```

---

### 6. Describe Access Points

```bash
aws efs describe-access-points \
  --file-system-id fs-0abc1234def567890 \
  --region us-east-1
```

**What it does:** Lists all access points for a given EFS file system, showing their POSIX configurations, root directories, and lifecycle states.

---

### 7. Put Lifecycle Configuration

```bash
aws efs put-lifecycle-configuration \
  --file-system-id fs-0abc1234def567890 \
  --lifecycle-policies \
    '[
      {"TransitionToIA": "AFTER_30_DAYS"},
      {"TransitionToPrimaryStorageClass": "AFTER_1_ACCESS"}
    ]' \
  --region us-east-1
```

**What it does:** Configures lifecycle management policies. Files not accessed for 30 days are moved to the cheaper Infrequent Access (IA) storage class. Files accessed again are automatically moved back to the standard storage class.

**Example Output:**
```json
{
    "LifecyclePolicies": [
        {
            "TransitionToIA": "AFTER_30_DAYS"
        },
        {
            "TransitionToPrimaryStorageClass": "AFTER_1_ACCESS"
        }
    ]
}
```

---

### 8. Put File System Policy (Resource-Based Policy)

```bash
aws efs put-file-system-policy \
  --file-system-id fs-0abc1234def567890 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::123456789012:role/my-ec2-role"
        },
        "Action": [
          "elasticfilesystem:ClientMount",
          "elasticfilesystem:ClientWrite"
        ],
        "Condition": {
          "Bool": {
            "elasticfilesystem:AccessedViaMountTarget": "true"
          }
        }
      }
    ]
  }' \
  --region us-east-1
```

**What it does:** Attaches a resource-based policy to the EFS file system to control which IAM principals can mount and write to the file system, and under what conditions.

---

### 9. Describe File System Policy

```bash
aws efs describe-file-system-policy \
  --file-system-id fs-0abc1234def567890 \
  --region us-east-1
```

**What it does:** Retrieves the current resource-based policy attached to an EFS file system.

---

### 10. Tag a Resource

```bash
aws efs tag-resource \
  --resource-id fs-0abc1234def567890 \
  --tags Key=CostCenter,Value=CC-12345 Key=Owner,Value=platform-team \
  --region us-east-1
```

**What it does:** Adds or updates tags on an EFS resource (file system, mount target, or access point). Tags are key-value pairs used for cost allocation, access control, and organization.

---

### 11. List Tags for Resource

```bash
aws efs list-tags-for-resource \
  --resource-id fs-0abc1234def567890 \
  --region us-east-1
```

**Example Output:**
```json
{
    "Tags": [
        {"Key": "Name", "Value": "my-efs-filesystem"},
        {"Key": "Environment", "Value": "production"},
        {"Key": "CostCenter", "Value": "CC-12345"},
        {"Key": "Owner", "Value": "platform-team"}
    ]
}
```

---

### 12. Update File System (Change Throughput Mode)

```bash
aws efs update-file-system \
  --file-system-id fs-0abc1234def567890 \
  --throughput-mode provisioned \
  --provisioned-throughput-in-mibps 256 \
  --region us-east-1
```

**What it does:** Updates the throughput configuration of an existing EFS file system. Here, it switches from bursting to provisioned throughput at 256 MiB/s.

---

### 13. Delete a Mount Target

```bash
aws efs delete-mount-target \
  --mount-target-id fsmt-0abc123456def7890 \
  --region us-east-1
```

**What it does:** Deletes a specific mount target. The file system must have its mount targets deleted before the file system itself can be deleted. This operation is irreversible.

---

### 14. Delete an Access Point

```bash
aws efs delete-access-point \
  --access-point-id fsap-0abc123456789def0 \
  --region us-east-1
```

**What it does:** Permanently deletes an EFS access point. Applications using this access point will lose access immediately.

---

### 15. Delete a File System

```bash
aws efs delete-file-system \
  --file-system-id fs-0abc1234def567890 \
  --region us-east-1
```

**What it does:** Permanently deletes an EFS file system and all data within it. All mount targets and access points must be deleted first.

---

## Common Operations

### Create Operations

```bash
# Create a file system with provisioned throughput
aws efs create-file-system \
  --creation-token "prod-efs-$(uuidgen)" \
  --performance-mode generalPurpose \
  --throughput-mode provisioned \
  --provisioned-throughput-in-mibps 128 \
  --encrypted \
  --tags Key=Name,Value=prod-shared-storage \
  --region us-east-1

# Create a mount target in each AZ (repeat for each subnet)
aws efs create-mount-target \
  --file-system-id fs-0abc1234def567890 \
  --subnet-id subnet-0a1b2c3d4e5f67890 \
  --security-groups sg-0123456789abcdef0 \
  --region us-east-1

# Create an access point for a specific application
aws efs create-access-point \
  --file-system-id fs-0abc1234def567890 \
  --posix-user Uid=2000,