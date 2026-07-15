# S3 — Hands-On Labs

## Lab 1: Getting Started with S3

### Objective
In this lab, you will create your first Amazon S3 bucket, upload objects, configure basic permissions, and learn how to access objects via pre-signed URLs. By the end, you will understand the core S3 concepts: buckets, objects, keys, and basic access control.

### Prerequisites

**AWS Services Required:**
- Amazon S3

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket",
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:DeleteBucket",
        "s3:ListBucket",
        "s3:GetPresignedUrl"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS Management Console access
- AWS CLI v2 installed and configured (`aws configure`)
- A text editor
- `curl` or a web browser for testing URLs

**Environment Setup:**
```bash
# Verify AWS CLI is configured
aws sts get-caller-identity

# Set a unique bucket name variable (replace YOUR_ALIAS with your name/initials)
export BUCKET_NAME="s3-lab-intro-YOUR_ALIAS-$(date +%s)"
export AWS_REGION="us-east-1"
echo "Bucket name: $BUCKET_NAME"
```

---

### Steps

#### Step 1: Create an S3 Bucket

**Console:**
1. Navigate to the [S3 Console](https://s3.console.aws.amazon.com/s3/)
2. Click **Create bucket**
3. Set **Bucket name** to `s3-lab-intro-YOUR_ALIAS-TIMESTAMP` (must be globally unique)
4. Set **AWS Region** to `us-east-1`
5. Under **Block Public Access settings**, ensure **Block all public access** is checked (default)
6. Leave **Bucket Versioning** as `Disabled`
7. Leave **Default encryption** as `Amazon S3 managed keys (SSE-S3)`
8. Click **Create bucket**

**CLI:**
```bash
aws s3api create-bucket \
  --bucket "$BUCKET_NAME" \
  --region "$AWS_REGION"
```

> **Note:** For regions other than `us-east-1`, add `--create-bucket-configuration LocationConstraint=YOUR_REGION`

**Verify:**
```bash
# Confirm bucket exists
aws s3api head-bucket --bucket "$BUCKET_NAME"
echo "Exit code: $? (0 = success)"

# List all your buckets
aws s3 ls | grep "$BUCKET_NAME"
```

**Expected Output:**
```
2024-01-15 10:30:00 s3-lab-intro-jdoe-1705312200
```

---

#### Step 2: Upload Objects to the Bucket

**Console:**
1. Click on your newly created bucket
2. Click **Upload**
3. Click **Add files**
4. Create and upload a simple text file (see CLI instructions to create sample files)
5. Click **Upload**
6. Wait for the success message

**CLI — Create sample files and upload them:**
```bash
# Create sample files locally
echo "Hello from S3 Lab! This is file 1." > file1.txt
echo "This is a second test file for the S3 lab." > file2.txt
mkdir -p images
echo "<html><body><h1>Hello S3!</h1></body></html>" > images/index.html

# Upload a single file
aws s3 cp file1.txt s3://$BUCKET_NAME/file1.txt

# Upload with metadata
aws s3 cp file2.txt s3://$BUCKET_NAME/documents/file2.txt \
  --metadata "author=student,lab=s3-intro"

# Upload an entire directory (sync)
aws s3 sync images/ s3://$BUCKET_NAME/images/

# List objects in the bucket
aws s3 ls s3://$BUCKET_NAME --recursive
```

**Verify:**
```bash
aws s3api list-objects-v2 \
  --bucket "$BUCKET_NAME" \
  --query "Contents[].{Key:Key, Size:Size, LastModified:LastModified}" \
  --output table
```

**Expected Output:**
```
-----------------------------------------------------------------
|                       ListObjectsV2                           |
+---------------------+--------------------------+--------------+
|    LastModified     |           Key            |    Size      |
+---------------------+--------------------------+--------------+
|  2024-01-15T10:31:00|  documents/file2.txt     |  43          |
|  2024-01-15T10:31:01|  file1.txt               |  36          |
|  2024-01-15T10:31:02|  images/index.html       |  45          |
+---------------------+--------------------------+--------------+
```

---

#### Step 3: Retrieve Object Metadata and Content

**Console:**
1. Click on `file1.txt` in the S3 console
2. Review the **Properties** tab — note the Object URL, size, and storage class
3. Click **Open** to view the file content in your browser

**CLI:**
```bash
# Get object metadata (HEAD request — no download)
aws s3api head-object \
  --bucket "$BUCKET_NAME" \
  --key "file1.txt"

# Download a single file
aws s3 cp s3://$BUCKET_NAME/file1.txt ./downloaded_file1.txt
cat downloaded_file1.txt

# Read object content directly to stdout
aws s3 cp s3://$BUCKET_NAME/file1.txt -
```

**Verify:**
```bash
# Confirm downloaded file matches original
diff file1.txt downloaded_file1.txt && echo "Files match!" || echo "Files differ!"
```

**Expected Output:**
```json
{
    "AcceptRanges": "bytes",
    "LastModified": "2024-01-15T10:31:00+00:00",
    "ContentLength": 36,
    "ETag": "\"a1b2c3d4e5f6...\"",
    "ContentType": "text/plain",
    "Metadata": {},
    "StorageClass": "STANDARD"
}
```

---

#### Step 4: Generate a Pre-Signed URL

Pre-signed URLs allow temporary, secure access to private S3 objects without making them public.

**Console:**
1. Click on `file1.txt` in the S3 console
2. Click **Object actions** → **Share with a presigned URL**
3. Set the expiration to **5 minutes**
4. Click **Create presigned URL**
5. Copy the URL and open it in a browser or incognito window

**CLI:**
```bash
# Generate a pre-signed URL valid for 300 seconds (5 minutes)
PRESIGNED_URL=$(aws s3 presign s3://$BUCKET_NAME/file1.txt \
  --expires-in 300)

echo "Pre-signed URL:"
echo "$PRESIGNED_URL"

# Test the URL with curl
curl -s "$PRESIGNED_URL"
```

**Verify:**
```bash
# The curl command should return the file content
curl -s "$PRESIGNED_URL" | grep "Hello from S3"
```

**Expected Output:**
```
Hello from S3 Lab! This is file 1.
```

---

#### Step 5: Configure a Bucket Policy for Read Access

**Console:**
1. Go to your bucket → **Permissions** tab
2. Under **Block public access**, click **Edit**
3. **Uncheck** "Block all public access" and confirm
4. Go to **Bucket policy** and click **Edit**
5. Paste the policy below (replace `BUCKET_NAME`)
6. Click **Save changes**

**CLI:**
```bash
# First, disable Block Public Access (required before applying public policy)
aws s3api put-public-access-block \
  --bucket "$BUCKET_NAME" \
  --public-access-block-configuration \
    "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Create and apply a bucket policy for public read on the /images/ prefix only
cat > bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForImages",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/images/*"
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket "$BUCKET_NAME" \
  --policy file://bucket-policy.json
```

**Verify:**
```bash
# Get the public URL for the image file
echo "Public URL: https://${BUCKET_NAME}.s3.amazonaws.com/images/index.html"

# Test public access to the images prefix
curl -s "https://${BUCKET_NAME}.s3.amazonaws.com/images/index.html"

# Confirm non-images prefix is still private (should return 403)
curl -s -o /dev/null -w "%{http_code}" \
  "https://${BUCKET_NAME}.s3.amazonaws.com/file1.txt"
```

**Expected Output:**
```
<html><body><h1>Hello S3!</h1></body></html>
403
```

---

### Verification

Run the following verification script to confirm all lab tasks were completed:

```bash
#!/bin/bash
echo "=== Lab 1 Verification ==="

# Check 1: Bucket exists
echo -n "1. Bucket exists: "
aws s3api head-bucket --bucket "$BUCKET_NAME" 2>/dev/null && echo "PASS" || echo "FAIL"

# Check 2: Objects uploaded
echo -n "2. Objects in bucket: "
COUNT=$(aws s3api list-objects-v2 --bucket "$BUCKET_NAME" --query "length(Contents)" --output text)
[ "$COUNT" -ge 3 ] && echo "PASS ($COUNT objects)" || echo "FAIL (found $COUNT objects)"

# Check 3: Bucket policy exists
echo -n "3. Bucket policy configured: "
aws s3api get-bucket-policy --bucket "$BUCKET_NAME" 2>/dev/null | grep -q "PublicReadForImages" && echo "PASS" || echo "FAIL"

# Check 4: Public image accessible
echo -n "4. Public image accessible: "
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
  "https://${BUCKET_NAME}.s3.amazonaws.com/images/index.html")
[ "$HTTP_CODE" == "200" ] && echo "PASS" || echo "FAIL (HTTP $HTTP_CODE)"

echo "=== Verification Complete ==="
```

---

### Cleanup

> ⚠️ **Important:** S3 charges for storage. Always clean up lab resources.

```bash
# Step 1: Remove the bucket policy
aws s3api delete-bucket-policy --bucket "$BUCKET_NAME"

# Step 2: Delete all objects in the bucket
aws s3 rm s3://$BUCKET_NAME --recursive

# Step 3: Verify bucket is empty
aws s3 ls s3://$BUCKET_NAME

# Step 4: Delete the bucket
aws s3api delete-bucket --bucket "$BUCKET_NAME"

# Step 5: Clean up local files
rm -f file1.txt file2.txt downloaded_file1.txt bucket-policy.json
rm -rf images/

# Step 6: Verify deletion
aws s3 ls | grep "$BUCKET_NAME" && echo "WARNING: Bucket still exists!" || echo "Cleanup complete!"
```

---

## Lab 2: Intermediate S3 Configuration

### Objective
In this lab, you will configure advanced S3 features including **versioning**, **lifecycle policies**, **S3 Event Notifications** to Lambda, and **Cross-Region Replication (CRR)**. You will simulate a real-world backup pipeline where objects are automatically tiered to cheaper storage and replicated to a secondary region for disaster recovery.

### Prerequisites

**AWS Services Required:**
- Amazon S3 (two buckets in different regions)
- AWS Lambda
- Amazon SNS
- AWS IAM

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "lambda:CreateFunction",
        "lambda:AddPermission",
        "lambda:DeleteFunction",
        "lambda:GetFunction",
        "sns:CreateTopic",
        "sns:Subscribe",
        "sns:DeleteTopic",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole",
        "iam:DeleteRole",
        "iam:DetachRolePolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2
- Python 3.x (for Lambda function code)
- `zip` utility

**Environment Setup:**
```bash
export PRIMARY_REGION="us-east-1"
export REPLICA_REGION="us-west-2"
export ALIAS="student01"  # Replace with your alias
export PRIMARY_BUCKET="s3-lab-primary-${ALIAS}-$(date +%s)"
export REPLICA_BUCKET="s3-lab-replica-${ALIAS}-$(date +%s)"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "Primary Bucket:  $PRIMARY_BUCKET"
echo "Replica Bucket:  $REPLICA_BUCKET"
echo "Account ID:      $ACCOUNT_ID"
```

---

### Steps

#### Step 1: Create Primary and Replica Buckets with Versioning

Versioning is **required** for Cross-Region Replication and is a best practice for any production bucket.

**Console:**
1. Create a bucket in `us-east-1` named `s3-lab-primary-ALIAS-TIMESTAMP`
2. Under **Bucket Versioning**, select **Enable**
3. Create a second bucket in `us-west-2` named `s3-lab-replica-ALIAS-TIMESTAMP`
4. Enable versioning on the replica bucket too

**CLI:**
```bash
# Create primary bucket (us-east-1)
aws s3api create-bucket \
  --bucket "$PRIMARY_BUCKET" \
  --region "$PRIMARY_REGION"

# Enable versioning on primary bucket
aws s3api put-bucket-versioning \
  --bucket "$PRIMARY_BUCKET" \
  --versioning-configuration Status=Enabled

# Create replica bucket (us-west-2)
aws s3api create-bucket \
  --bucket "$REPLICA_BUCKET" \
  --region "$REPLICA_REGION" \
  --create-bucket-configuration LocationConstraint="$REPLICA_REGION"

# Enable versioning on replica bucket
aws s3api put-bucket-versioning \
  --bucket "$REPLICA_BUCKET" \
  --versioning-configuration Status=Enabled
```

**Verify:**
```bash
# Check versioning status on both buckets
for BUCKET in "$PRIMARY_BUCKET" "$REPLICA_BUCKET"; do
  echo -n "Versioning on $BUCKET: "
  aws s3api get-bucket-versioning --bucket "$BUCKET" --query "Status" --output text
done
```

**Expected Output:**
```
Versioning on s3-lab-primary-student01-...: Enabled
Versioning on s3-lab-replica-student01-...: Enabled
```

---

#### Step 2: Test Object Versioning

**Console:**
1. Go to the primary bucket → **Upload** `file1.txt`
2. Modify the file locally and upload again with the same name
3. Click **Show versions** toggle in the Objects tab to see both versions

**CLI:**
```bash
# Create and upload initial version
echo "Version 1 - Original content" > versioned-file.txt
aws s3 cp versioned-file.txt s3://$PRIMARY_BUCKET/versioned-file.txt

# Capture version ID
V1_ID=$(aws s3api