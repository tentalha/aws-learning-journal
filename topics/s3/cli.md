# S3 — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using the AWS CLI with S3, ensure the following are in place:

**Install and configure the AWS CLI:**
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

**Verify configuration:**
```bash
aws sts get-caller-identity
aws configure list
```

### Required IAM Permissions

The following IAM policy covers most S3 operations. Scope down as needed for production:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket",
        "s3:DeleteBucket",
        "s3:ListBucket",
        "s3:ListAllMyBuckets",
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:GetBucketPolicy",
        "s3:PutBucketPolicy",
        "s3:DeleteBucketPolicy",
        "s3:GetBucketVersioning",
        "s3:PutBucketVersioning",
        "s3:GetBucketEncryption",
        "s3:PutEncryptionConfiguration",
        "s3:GetBucketLogging",
        "s3:PutBucketLogging",
        "s3:GetBucketLocation",
        "s3:GetBucketAcl",
        "s3:PutBucketAcl",
        "s3:GetLifecycleConfiguration",
        "s3:PutLifecycleConfiguration",
        "s3:GetBucketNotification",
        "s3:PutBucketNotification",
        "s3:GetBucketTagging",
        "s3:PutBucketTagging",
        "s3:ReplicateObject",
        "s3:GetReplicationConfiguration",
        "s3:PutReplicationConfiguration"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

> **Note:** The AWS CLI provides two interfaces for S3:
> - `aws s3` — High-level commands (sync, cp, mv, rm, ls)
> - `aws s3api` — Low-level API commands with full parameter control

---

## Core Commands

### 1. Create a Bucket

```bash
aws s3api create-bucket \
  --bucket my-bucket \
  --region us-east-1
```

> For regions **other than** `us-east-1`, you must specify `--create-bucket-configuration`:

```bash
aws s3api create-bucket \
  --bucket my-bucket \
  --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1
```

**Example output:**
```json
{
    "Location": "/my-bucket"
}
```

---

### 2. List All Buckets

```bash
aws s3 ls
```

**Example output:**
```
2023-01-15 10:23:45 my-bucket
2023-03-22 08:14:30 my-logs-bucket
2024-06-01 14:55:10 my-static-website-bucket
```

---

### 3. List Objects in a Bucket

```bash
# High-level listing
aws s3 ls s3://my-bucket/

# Recursive listing with human-readable sizes
aws s3 ls s3://my-bucket/ --recursive --human-readable --summarize
```

**Example output:**
```
2024-09-10 12:00:01    1.2 MiB images/logo.png
2024-09-10 12:00:02   45.3 KiB docs/readme.md
2024-09-10 12:00:03  512.0 KiB videos/intro.mp4

Total Objects: 3
   Total Size: 1.7 MiB
```

---

### 4. Upload a File

```bash
# Upload a single file
aws s3 cp /local/path/myfile.txt s3://my-bucket/prefix/myfile.txt

# Upload with metadata and storage class
aws s3 cp /local/path/myfile.txt s3://my-bucket/prefix/myfile.txt \
  --storage-class STANDARD_IA \
  --metadata "author=john,project=demo" \
  --content-type "text/plain"
```

**Example output:**
```
upload: local/path/myfile.txt to s3://my-bucket/prefix/myfile.txt
```

---

### 5. Download a File

```bash
# Download a single file
aws s3 cp s3://my-bucket/prefix/myfile.txt /local/path/myfile.txt

# Download an entire prefix (folder)
aws s3 cp s3://my-bucket/prefix/ /local/path/ --recursive
```

**Example output:**
```
download: s3://my-bucket/prefix/myfile.txt to local/path/myfile.txt
```

---

### 6. Sync a Directory

```bash
# Sync local directory to S3
aws s3 sync /local/path/my-app/ s3://my-bucket/my-app/ \
  --delete \
  --exclude "*.tmp" \
  --exclude ".DS_Store"

# Sync S3 to local directory
aws s3 sync s3://my-bucket/my-app/ /local/path/my-app/ \
  --delete

# Dry-run sync (preview changes without executing)
aws s3 sync /local/path/my-app/ s3://my-bucket/my-app/ \
  --dryrun
```

**Example output:**
```
(dryrun) upload: local/path/my-app/index.html to s3://my-bucket/my-app/index.html
(dryrun) upload: local/path/my-app/style.css to s3://my-bucket/my-app/style.css
(dryrun) delete: s3://my-bucket/my-app/old-page.html
```

---

### 7. Delete an Object

```bash
# Delete a single object
aws s3 rm s3://my-bucket/prefix/myfile.txt

# Delete all objects under a prefix
aws s3 rm s3://my-bucket/prefix/ --recursive

# Delete objects matching a pattern
aws s3 rm s3://my-bucket/logs/ --recursive --exclude "*" --include "*.log"
```

**Example output:**
```
delete: s3://my-bucket/prefix/myfile.txt
```

---

### 8. Delete a Bucket

```bash
# Delete an empty bucket
aws s3api delete-bucket \
  --bucket my-empty-bucket \
  --region us-east-1

# Force delete a bucket and all its contents
aws s3 rb s3://my-bucket --force
```

**Example output:**
```
remove_bucket: my-bucket
```

---

### 9. Get Object Metadata (Head Object)

```bash
aws s3api head-object \
  --bucket my-bucket \
  --key prefix/myfile.txt
```

**Example output:**
```json
{
    "AcceptRanges": "bytes",
    "LastModified": "2024-09-10T12:00:01+00:00",
    "ContentLength": 1258291,
    "ETag": "\"d41d8cd98f00b204e9800998ecf8427e\"",
    "ContentType": "text/plain",
    "Metadata": {
        "author": "john",
        "project": "demo"
    },
    "StorageClass": "STANDARD_IA"
}
```

---

### 10. Enable Bucket Versioning

```bash
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled
```

**Check versioning status:**
```bash
aws s3api get-bucket-versioning \
  --bucket my-bucket
```

**Example output:**
```json
{
    "Status": "Enabled",
    "MFADelete": "Disabled"
}
```

---

### 11. Enable Server-Side Encryption (SSE)

```bash
# Enable SSE with AWS managed keys (SSE-S3)
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      },
      "BucketKeyEnabled": true
    }]
  }'

# Enable SSE with KMS key
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/my-key-id"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

---

### 12. Block Public Access

```bash
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration '{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }'
```

**Verify public access block:**
```bash
aws s3api get-public-access-block --bucket my-bucket
```

**Example output:**
```json
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}
```

---

### 13. Apply a Bucket Policy

```bash
# Write policy to a file first
cat > /tmp/bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontReadOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket my-bucket \
  --policy file:///tmp/bucket-policy.json
```

**Get existing policy:**
```bash
aws s3api get-bucket-policy \
  --bucket my-bucket \
  --output text | python3 -m json.tool
```

---

### 14. Generate a Presigned URL

```bash
# Generate a presigned URL valid for 1 hour (3600 seconds)
aws s3 presign s3://my-bucket/private/document.pdf \
  --expires-in 3600

# Generate a presigned URL valid for 7 days
aws s3 presign s3://my-bucket/private/document.pdf \
  --expires-in 604800
```

**Example output:**
```
https://my-bucket.s3.amazonaws.com/private/document.pdf?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIOSFODNN7EXAMPLE%2F20240910%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20240910T120000Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=examplesignature
```

---

### 15. Move (Rename) an Object

```bash
# Move within the same bucket
aws s3 mv s3://my-bucket/old-prefix/myfile.txt s3://my-bucket/new-prefix/myfile.txt

# Move between buckets
aws s3 mv s3://my-source-bucket/myfile.txt s3://my-destination-bucket/myfile.txt

# Move all objects matching a prefix
aws s3 mv s3://my-bucket/old-prefix/ s3://my-bucket/new-prefix/ --recursive
```

---

## Common Operations

### Create

**Create a bucket with full configuration:**
```bash
# 1. Create the bucket
aws s3api create-bucket \
  --bucket my-new-bucket \
  --region us-east-1

# 2. Block all public access
aws s3api put-public-access-block \
  --bucket my-new-bucket \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# 3. Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-new-bucket \
  --versioning-configuration Status=Enabled

# 4. Enable default encryption
aws s3api put-bucket-encryption \
  --bucket my-new-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'

# 5. Add tags
aws s3api put-bucket-tagging \
  --bucket my-new-bucket \
  --tagging 'TagSet=[{Key=Environment,Value=production},{Key=Project,Value=my-app},{Key=Owner,Value=devops-team}]'
```

**Create a multipart upload (for large files):**
```bash
# Initiate multipart upload
aws s3api create-multipart-upload \
  --bucket my-bucket \
  --key large-file/bigdata.csv \
  --storage-class STANDARD_IA
```

**Example output:**
```json
{
    "Bucket": "my-bucket",
    "Key": "large-file/bigdata.csv",
    "UploadId": "VXBsb2FkIElEIGZvciA2aWWpbmcncyBteS1tb3ZpZS5tMnRzIHVwbG9hZA"
}
```

---

### Read

**Get a specific object:**
```bash
# Download object to local file
aws s3api get-object \
  --bucket my-bucket \
  --key prefix/myfile.txt \
  /local/path/output.txt

# Download a specific version of an object
aws s3api get-object \
  --bucket my-bucket \
  --key prefix/myfile.txt \
  --version-id "3sL4kqtJlcpXroDTDmJ+rmSpXd3dIbrHY+MTRCxf3vjVBH40Nr8X8gdRQBpUMLUo" \
  /local/path/output.txt

# Read object content directly to stdout
aws s3 cp s3://my-bucket/config/settings.json - | jq .
```

**Get object ACL:**
```bash
aws s3api get-object-acl \
  --bucket my-bucket \
  --key prefix/myfile