# EBS — Hands-On Labs

## Lab 1: Getting Started with EBS

### Objective
In this lab, you will learn the fundamentals of Amazon Elastic Block Store (EBS). You will create an EBS volume, attach it to an EC2 instance, format and mount the volume, write data to it, and then detach and reattach it to a different instance to demonstrate data persistence. By the end, you will understand the core lifecycle of an EBS volume.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account with billing enabled |
| IAM Permissions | `ec2:*` (or scoped: `ec2:CreateVolume`, `ec2:AttachVolume`, `ec2:DescribeVolumes`, `ec2:DetachVolume`, `ec2:DeleteVolume`) |
| EC2 Key Pair | An existing key pair in `us-east-1` for SSH access |
| AWS CLI | Version 2.x installed and configured (`aws configure`) |
| Tools | SSH client (Terminal / PuTTY), AWS Management Console access |
| Region | `us-east-1` (N. Virginia) — all steps use this region |

> **Cost Estimate:** This lab uses a `t3.micro` EC2 instance and a 10 GiB `gp3` volume. Total cost is under $0.10 if completed within 2 hours and cleaned up promptly.

---

### Steps

#### Step 1: Launch Two EC2 Instances

**Console:**
1. Navigate to **EC2 → Instances → Launch instances**
2. Set **Name** to `ebs-lab-instance-1`
3. Choose **Amazon Linux 2023 AMI** (free tier eligible)
4. Select **t3.micro** instance type
5. Select your existing key pair
6. Under **Network settings**, allow **SSH (port 22)** from your IP
7. Under **Configure storage**, keep the default 8 GiB root volume
8. Click **Launch instance**
9. Repeat steps 1–8 to create `ebs-lab-instance-2` in the **same Availability Zone** (e.g., `us-east-1a`)

> ⚠️ **Critical:** Both instances and your EBS volume **must be in the same Availability Zone**. EBS volumes can only be attached to instances in the same AZ.

**CLI:**
```bash
# Store your key pair name and AZ
KEY_NAME="your-key-pair-name"
AZ="us-east-1a"
SUBNET_ID="subnet-xxxxxxxxx"  # Replace with a subnet in us-east-1a

# Get the latest Amazon Linux 2023 AMI ID
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" \
            "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text \
  --region us-east-1)

echo "Using AMI: $AMI_ID"

# Create a security group for SSH access
SG_ID=$(aws ec2 create-security-group \
  --group-name ebs-lab-sg \
  --description "Security group for EBS lab" \
  --region us-east-1 \
  --query 'GroupId' \
  --output text)

# Allow SSH from your current IP
MY_IP=$(curl -s https://checkip.amazonaws.com)/32
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr $MY_IP \
  --region us-east-1

# Launch Instance 1
INSTANCE_1=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID \
  --placement AvailabilityZone=$AZ \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ebs-lab-instance-1}]' \
  --region us-east-1 \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance 1 ID: $INSTANCE_1"

# Launch Instance 2
INSTANCE_2=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID \
  --placement AvailabilityZone=$AZ \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ebs-lab-instance-2}]' \
  --region us-east-1 \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance 2 ID: $INSTANCE_2"

# Wait for instances to be running
aws ec2 wait instance-running \
  --instance-ids $INSTANCE_1 $INSTANCE_2 \
  --region us-east-1

echo "Both instances are running!"
```

**✅ Verify:** In the EC2 console, both instances show **Running** state. Note the Availability Zone — both must match.

---

#### Step 2: Create an EBS Volume

**Console:**
1. Navigate to **EC2 → Elastic Block Store → Volumes → Create volume**
2. Configure:
   - **Volume type:** `gp3`
   - **Size:** `10` GiB
   - **IOPS:** `3000` (default)
   - **Throughput:** `125` MiB/s (default)
   - **Availability Zone:** `us-east-1a` (must match your instances)
   - **Snapshot ID:** Leave empty
   - **Encryption:** Leave unchecked for now (covered in Lab 3)
3. Add a tag: **Key** = `Name`, **Value** = `ebs-lab-data-volume`
4. Click **Create volume**

**CLI:**
```bash
VOLUME_ID=$(aws ec2 create-volume \
  --volume-type gp3 \
  --size 10 \
  --availability-zone $AZ \
  --iops 3000 \
  --throughput 125 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=ebs-lab-data-volume},{Key=Lab,Value=ebs-lab-1}]' \
  --region us-east-1 \
  --query 'VolumeId' \
  --output text)

echo "Volume ID: $VOLUME_ID"

# Wait for volume to become available
aws ec2 wait volume-available \
  --volume-ids $VOLUME_ID \
  --region us-east-1

# Verify volume details
aws ec2 describe-volumes \
  --volume-ids $VOLUME_ID \
  --region us-east-1 \
  --query 'Volumes[0].{ID:VolumeId,State:State,Size:Size,Type:VolumeType,AZ:AvailabilityZone}'
```

**Expected Output:**
```json
{
    "ID": "vol-0abc123def456789",
    "State": "available",
    "Size": 10,
    "Type": "gp3",
    "AZ": "us-east-1a"
}
```

**✅ Verify:** Volume state shows `available` in the console under **Elastic Block Store → Volumes**.

---

#### Step 3: Attach the Volume to Instance 1

**Console:**
1. Select the `ebs-lab-data-volume` in the Volumes list
2. Click **Actions → Attach volume**
3. Select `ebs-lab-instance-1` from the instance dropdown
4. Set **Device name** to `/dev/sdf`
5. Click **Attach volume**

**CLI:**
```bash
aws ec2 attach-volume \
  --volume-id $VOLUME_ID \
  --instance-id $INSTANCE_1 \
  --device /dev/sdf \
  --region us-east-1

# Wait for attachment
aws ec2 wait volume-in-use \
  --volume-ids $VOLUME_ID \
  --region us-east-1

# Verify attachment state
aws ec2 describe-volumes \
  --volume-ids $VOLUME_ID \
  --region us-east-1 \
  --query 'Volumes[0].Attachments[0].{State:State,Device:Device,InstanceId:InstanceId}'
```

**Expected Output:**
```json
{
    "State": "attached",
    "Device": "/dev/sdf",
    "InstanceId": "i-0abc123def456789"
}
```

**✅ Verify:** Volume state changes to `in-use` in the console.

---

#### Step 4: Format and Mount the Volume on Instance 1

SSH into Instance 1 and prepare the volume for use.

```bash
# Get the public IP of Instance 1
INSTANCE_1_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_1 \
  --region us-east-1 \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "Instance 1 IP: $INSTANCE_1_IP"

# SSH into the instance
ssh -i ~/.ssh/$KEY_NAME.pem ec2-user@$INSTANCE_1_IP
```

Once connected to Instance 1, run these commands:

```bash
# List all block devices to confirm the new volume is visible
lsblk

# Expected output shows /dev/xvdf (AWS maps /dev/sdf to /dev/xvdf on Nitro instances)
# NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
# xvda    202:0    0   8G  0 disk
# └─xvda1 202:1    0   8G  0 part /
# xvdf    202:80   0  10G  0 disk    <-- your new volume

# Verify no filesystem exists yet
sudo file -s /dev/xvdf
# Expected: /dev/xvdf: data  (means no filesystem)

# Create an ext4 filesystem
sudo mkfs -t ext4 /dev/xvdf

# Create a mount point
sudo mkdir -p /data

# Mount the volume
sudo mount /dev/xvdf /data

# Verify the mount
df -h /data
# Expected:
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/xvdf       9.8G   24K  9.3G   1% /data

# Write some test data to the volume
sudo bash -c 'echo "Hello from ebs-lab-instance-1! Data persists across instances." > /data/testfile.txt'
sudo bash -c 'date >> /data/testfile.txt'

# Verify the file
cat /data/testfile.txt

# Exit the SSH session
exit
```

**✅ Verify:** The `df -h` command shows the 10 GiB volume mounted at `/data`, and the test file exists.

---

#### Step 5: Detach Volume and Reattach to Instance 2

```bash
# Detach the volume from Instance 1
# First, unmount it via SSH (best practice before detaching)
ssh -i ~/.ssh/$KEY_NAME.pem ec2-user@$INSTANCE_1_IP \
  "sudo umount /data && echo 'Volume unmounted successfully'"

# Detach the volume
aws ec2 detach-volume \
  --volume-id $VOLUME_ID \
  --region us-east-1

# Wait for the volume to become available again
aws ec2 wait volume-available \
  --volume-ids $VOLUME_ID \
  --region us-east-1

echo "Volume is available and ready to attach"

# Attach to Instance 2
aws ec2 attach-volume \
  --volume-id $VOLUME_ID \
  --instance-id $INSTANCE_2 \
  --device /dev/sdf \
  --region us-east-1

aws ec2 wait volume-in-use \
  --volume-ids $VOLUME_ID \
  --region us-east-1

echo "Volume attached to Instance 2"
```

---

#### Step 6: Verify Data Persistence on Instance 2

```bash
# Get Instance 2 public IP
INSTANCE_2_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_2 \
  --region us-east-1 \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

# SSH into Instance 2
ssh -i ~/.ssh/$KEY_NAME.pem ec2-user@$INSTANCE_2_IP
```

On Instance 2:
```bash
# List block devices
lsblk
# /dev/xvdf should appear as a 10G disk

# Mount the existing volume (DO NOT format — data already exists!)
sudo mkdir -p /data
sudo mount /dev/xvdf /data

# Verify the data from Instance 1 is still there
cat /data/testfile.txt

# Expected output:
# Hello from ebs-lab-instance-1! Data persists across instances.
# Mon Jan 15 14:23:45 UTC 2024

# Add a second line from Instance 2
sudo bash -c 'echo "Confirmed readable from ebs-lab-instance-2!" >> /data/testfile.txt'
cat /data/testfile.txt

exit
```

**✅ Verify:** The data written on Instance 1 is readable on Instance 2. This confirms EBS volume data persistence.

---

### Verification

Run these checks to confirm successful lab completion:

```bash
# 1. Verify volume is in-use and attached to Instance 2
aws ec2 describe-volumes \
  --volume-ids $VOLUME_ID \
  --region us-east-1 \
  --query 'Volumes[0].{State:State,AttachedTo:Attachments[0].InstanceId,Device:Attachments[0].Device}'

# 2. Verify both instances are running
aws ec2 describe-instances \
  --instance-ids $INSTANCE_1 $INSTANCE_2 \
  --region us-east-1 \
  --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,State:State.Name}'

# 3. Confirm volume type and size
aws ec2 describe-volumes \
  --volume-ids $VOLUME_ID \
  --region us-east-1 \
  --query 'Volumes[0].{Type:VolumeType,Size:Size,IOPS:Iops,Throughput:Throughput}'
```

**Expected final state:**
- ✅ Volume `ebs-lab-data-volume` is `in-use`, attached to `ebs-lab-instance-2`
- ✅ Both instances are in `running` state
- ✅ Data written on Instance 1 is readable on Instance 2
- ✅ Volume type is `gp3`, 10 GiB

---

### Cleanup

> ⚠️ Run cleanup in this order to avoid dependency errors.

```bash
# Step 1: Unmount volume on Instance 2
ssh -i ~/.ssh/$KEY_NAME.pem ec2-user@$INSTANCE_2_IP \
  "sudo umount /data && echo 'Unmounted'"

# Step 2: Detach the EBS volume
aws ec2 detach-volume \
  --volume-id $VOLUME_ID \
  --region us-east-1

aws ec2 wait volume-available \
  --volume-ids $VOLUME_ID \
  --region us-east-1

# Step 3: Delete the EBS volume
aws ec2 delete-volume \
  --volume-id $VOLUME_ID \
  --region us-east-1

echo "Volume deleted"

# Step 4: Terminate both EC2 instances
aws ec2 terminate-instances \
  --instance-ids $INSTANCE_1 $INSTANCE_2 \
  --region us-east-1

aws