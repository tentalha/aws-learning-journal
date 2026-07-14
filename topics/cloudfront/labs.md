# CloudFront — Hands-On Labs

## Lab 1: Getting Started with CloudFront

### Objective

In this lab, you will create an Amazon CloudFront distribution that serves static website content from an Amazon S3 bucket. You will learn how to configure an origin, set default cache behaviors, and verify that content is delivered through CloudFront's global edge network. By the end of this lab, you will have a working CDN setup that accelerates delivery of a static HTML website.

---

### Prerequisites

**AWS Services Required:**
- Amazon S3
- Amazon CloudFront

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
        "s3:PutBucketPolicy",
        "s3:GetObject",
        "cloudfront:CreateDistribution",
        "cloudfront:GetDistribution",
        "cloudfront:ListDistributions",
        "cloudfront:CreateInvalidation"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- A web browser
- `curl` (for verification)

**Estimated Cost:** < $0.50 for the duration of this lab  
**Estimated Time:** 30–45 minutes

---

### Steps

#### Step 1: Create an S3 Bucket for Static Website Content

**Console:**
1. Navigate to **S3** in the AWS Management Console.
2. Click **Create bucket**.
3. Set **Bucket name** to `cf-lab1-static-<your-account-id>` (must be globally unique).
4. Set **AWS Region** to `us-east-1`.
5. Under **Block Public Access settings**, leave **all options enabled** (CloudFront will access it privately via OAC).
6. Leave all other settings as default and click **Create bucket**.

**CLI:**
```bash
# Set variables
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="cf-lab1-static-${ACCOUNT_ID}"
REGION="us-east-1"

# Create the bucket
aws s3api create-bucket \
  --bucket "$BUCKET_NAME" \
  --region "$REGION"

echo "Bucket created: $BUCKET_NAME"
```

**Verify:** You should see the bucket listed in the S3 console or via:
```bash
aws s3api head-bucket --bucket "$BUCKET_NAME" && echo "Bucket exists"
```

---

#### Step 2: Upload Static Website Content

**Console:**
1. Open the bucket you just created.
2. Click **Upload** → **Add files**.
3. Create and upload an `index.html` file with the content below.
4. Also upload an `error.html` file.
5. Click **Upload**.

**CLI:**
```bash
# Create index.html
cat > /tmp/index.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>CloudFront Lab 1</title>
  <style>
    body { font-family: Arial, sans-serif; text-align: center; padding: 50px; background: #f0f4f8; }
    h1 { color: #232f3e; }
    .badge { background: #ff9900; color: white; padding: 10px 20px; border-radius: 5px; }
  </style>
</head>
<body>
  <h1>🚀 Hello from Amazon CloudFront!</h1>
  <p>This page is served from an S3 origin via CloudFront.</p>
  <span class="badge">Lab 1 — Static Website</span>
</body>
</html>
EOF

# Create error.html
cat > /tmp/error.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head><title>Error</title></head>
<body>
  <h1>404 — Page Not Found</h1>
  <p>The requested resource does not exist.</p>
</body>
</html>
EOF

# Upload files to S3
aws s3 cp /tmp/index.html "s3://${BUCKET_NAME}/index.html" \
  --content-type "text/html"

aws s3 cp /tmp/error.html "s3://${BUCKET_NAME}/error.html" \
  --content-type "text/html"

echo "Files uploaded successfully"
```

**Verify:**
```bash
aws s3 ls "s3://${BUCKET_NAME}/"
```
**Expected output:**
```
2024-01-15 10:00:00        512 error.html
2024-01-15 10:00:00        680 index.html
```

---

#### Step 3: Create a CloudFront Origin Access Control (OAC)

OAC allows CloudFront to securely access your private S3 bucket without making the bucket public.

**Console:**
1. Navigate to **CloudFront** in the AWS Console.
2. In the left panel, click **Origin access**.
3. Click **Create control setting**.
4. Set **Name** to `cf-lab1-oac`.
5. Set **Origin type** to **S3**.
6. Leave **Sign requests** as **Always**.
7. Click **Create**.

**CLI:**
```bash
# Create OAC
OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config \
    "Name=cf-lab1-oac,Description=OAC for Lab 1,SigningProtocol=sigv4,SigningBehavior=always,OriginAccessControlOriginType=s3" \
  --query 'OriginAccessControl.Id' \
  --output text)

echo "OAC ID: $OAC_ID"
```

**Verify:**
```bash
aws cloudfront get-origin-access-control \
  --id "$OAC_ID" \
  --query 'OriginAccessControl.OriginAccessControlConfig.Name' \
  --output text
```
**Expected output:** `cf-lab1-oac`

---

#### Step 4: Create a CloudFront Distribution

**Console:**
1. In **CloudFront**, click **Create distribution**.
2. Under **Origin**:
   - **Origin domain**: Select your S3 bucket from the dropdown (`cf-lab1-static-<account-id>.s3.us-east-1.amazonaws.com`).
   - **Origin access**: Select **Origin access control settings (recommended)**.
   - Select the `cf-lab1-oac` control setting you created.
   - Click **Copy policy** when prompted (you'll use this in Step 5).
3. Under **Default cache behavior**:
   - **Viewer protocol policy**: `Redirect HTTP to HTTPS`
   - **Cache policy**: `CachingOptimized`
4. Under **Settings**:
   - **Default root object**: `index.html`
   - **Price class**: `Use only North America and Europe` (to reduce cost)
5. Click **Create distribution**.
6. Note the **Distribution ID** and **Domain name** (e.g., `d1234abcd.cloudfront.net`).

**CLI:**
```bash
# Create distribution configuration file
cat > /tmp/cf-distribution.json <<EOF
{
  "CallerReference": "cf-lab1-$(date +%s)",
  "DefaultRootObject": "index.html",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "S3-cf-lab1-origin",
        "DomainName": "${BUCKET_NAME}.s3.${REGION}.amazonaws.com",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        },
        "OriginAccessControlId": "${OAC_ID}"
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-cf-lab1-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"]
    }
  },
  "PriceClass": "PriceClass_100",
  "Enabled": true,
  "Comment": "CloudFront Lab 1 Distribution"
}
EOF

# Create the distribution
DISTRIBUTION=$(aws cloudfront create-distribution \
  --distribution-config file:///tmp/cf-distribution.json)

DISTRIBUTION_ID=$(echo "$DISTRIBUTION" | jq -r '.Distribution.Id')
CF_DOMAIN=$(echo "$DISTRIBUTION" | jq -r '.Distribution.DomainName')

echo "Distribution ID: $DISTRIBUTION_ID"
echo "CloudFront Domain: $CF_DOMAIN"
```

**Verify:**
```bash
aws cloudfront get-distribution \
  --id "$DISTRIBUTION_ID" \
  --query 'Distribution.Status' \
  --output text
```
**Expected output:** `InProgress` initially, then `Deployed` (takes 5–10 minutes).

---

#### Step 5: Update the S3 Bucket Policy to Allow CloudFront Access

**Console:**
1. Go back to **S3** → your bucket → **Permissions** tab.
2. Under **Bucket policy**, click **Edit**.
3. Paste the policy that CloudFront generated in Step 4.
4. Click **Save changes**.

**CLI:**
```bash
# Create bucket policy allowing CloudFront OAC access
cat > /tmp/bucket-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAC",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::${ACCOUNT_ID}:distribution/${DISTRIBUTION_ID}"
        }
      }
    }
  ]
}
EOF

# Apply the bucket policy
aws s3api put-bucket-policy \
  --bucket "$BUCKET_NAME" \
  --policy file:///tmp/bucket-policy.json

echo "Bucket policy applied"
```

**Verify:**
```bash
aws s3api get-bucket-policy \
  --bucket "$BUCKET_NAME" \
  --query Policy \
  --output text | python3 -m json.tool
```

---

#### Step 6: Wait for Distribution Deployment and Test

**CLI — Wait for deployment:**
```bash
echo "Waiting for distribution to deploy (this may take 5-10 minutes)..."
aws cloudfront wait distribution-deployed --id "$DISTRIBUTION_ID"
echo "Distribution is deployed!"
```

**Test the distribution:**
```bash
# Test via curl
curl -I "https://${CF_DOMAIN}/index.html"
```

**Expected output:**
```
HTTP/2 200
content-type: text/html
x-cache: Miss from cloudfront        ← First request (cache miss)
via: 1.1 <hash>.cloudfront.net (CloudFront)
x-amz-cf-pop: IAD79-C1
```

**Test caching (second request):**
```bash
curl -I "https://${CF_DOMAIN}/index.html"
```
**Expected:** `x-cache: Hit from cloudfront` on subsequent requests.

**Open in browser:**
```
https://<your-cf-domain>.cloudfront.net/index.html
```

---

#### Step 7: Create a Cache Invalidation

When content changes, you must invalidate the CloudFront cache.

**Console:**
1. In CloudFront, select your distribution.
2. Click the **Invalidations** tab.
3. Click **Create invalidation**.
4. Enter `/*` to invalidate all files.
5. Click **Create invalidation**.

**CLI:**
```bash
# Update the index.html
cat > /tmp/index-v2.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>CloudFront Lab 1 — Updated</title>
  <style>
    body { font-family: Arial, sans-serif; text-align: center; padding: 50px; background: #e8f5e9; }
    h1 { color: #1b5e20; }
    .badge { background: #4caf50; color: white; padding: 10px 20px; border-radius: 5px; }
  </style>
</head>
<body>
  <h1>✅ Content Updated!</h1>
  <p>This is version 2 of the page, served after cache invalidation.</p>
  <span class="badge">Lab 1 — Cache Invalidation Test</span>
</body>
</html>
EOF

# Upload updated content
aws s3 cp /tmp/index-v2.html "s3://${BUCKET_NAME}/index.html" \
  --content-type "text/html"

# Create invalidation
INVALIDATION_ID=$(aws cloudfront create-invalidation \
  --distribution-id "$DISTRIBUTION_ID" \
  --paths "/*" \
  --query 'Invalidation.Id' \
  --output text)

echo "Invalidation created: $INVALIDATION_ID"

# Wait for invalidation to complete
aws cloudfront wait invalidation-completed \
  --distribution-id "$DISTRIBUTION_ID" \
  --id "$INVALIDATION_ID"

echo "Invalidation complete! Test the URL again."
curl "https://${CF_DOMAIN}/index.html" | grep -o '<h1>.*</h1>'
```

---

### Verification

Run the following checks to confirm the lab was completed successfully:

```bash
# 1. Verify distribution is deployed
STATUS=$(aws cloudfront get-distribution \
  --id "$DISTRIBUTION_ID" \
  --query 'Distribution.Status' \
  --output text)
echo "Distribution Status: $STATUS"   # Expected: Deployed

# 2. Verify HTTPS redirect works
curl -I "http://${CF_DOMAIN}/index.html" 2>/dev/null | grep -E "HTTP|Location"
# Expected: HTTP/1.1 301 and Location: https://...

# 3. Verify cache hit
curl -I "https://${CF_DOMAIN}/index.html" 2>/dev/null | grep "x-cache"
# Expected: x-cache: Hit from cloudfront

# 4. Verify S3 is NOT directly accessible (should return 403)
curl -I "https://${BUCKET_NAME}.s3.amazonaws.com/index.html" 2>/dev/null | grep "HTTP"
# Expected: HTTP/1.1 403 Forbidden
```

---

### Cleanup

Run the following steps **in order** to avoid ongoing charges:

```bash
# Step 1: Disable the distribution (required before deletion)
ETAG=$(aws cloudfront get-distribution-config \
  --id "$DISTRIBUTION_ID" \
  --query 'ETag' \
  --output text)

# Get current config and disable it
aws cloudfront get-distribution-config \
  --id "$DISTRIBUTION_ID" \
  --query 'DistributionConfig' \
  > /tmp/dist-config.json

# Set Enabled to false using Python
python3 -c "
import json
with open('/tmp/dist-config.json', 'r') as f:
    config = json.load(f)
config['Enabled'] = False
with open('/tmp/dist-config-disabled.json', 'w') as f:
    json.dump(config, f)
"

aws cloudfront update-distribution \
  --id "$DISTRIBUTION_ID" \
  --distribution-config file:///tmp/dist-config-disabled.json \
  --if-match "$ETAG"

# Step 2: Wait for distribution to be disabled
echo "Waiting for distribution to be disabled..."
aws cloudfront wait distribution-deployed --id "$