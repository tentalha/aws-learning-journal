# CloudFront — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using CloudFront CLI commands, ensure the following are in place:

**AWS CLI Version**
```bash
aws --version
# Requires AWS CLI v2.x recommended (minimum v1.16+)
```

**Configure AWS CLI**
```bash
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

> **Note:** CloudFront is a global service. Although it's managed globally, the AWS CLI region should be set to `us-east-1` for most CloudFront API calls.

### Required IAM Permissions

Attach the following IAM policy to your user/role for full CloudFront access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudfront:*",
        "s3:GetBucketPolicy",
        "s3:PutBucketPolicy",
        "acm:ListCertificates",
        "acm:DescribeCertificate",
        "wafv2:ListWebACLs",
        "route53:ListHostedZones"
      ],
      "Resource": "*"
    }
  ]
}
```

**Minimum read-only permissions:**
```json
{
  "Effect": "Allow",
  "Action": [
    "cloudfront:Get*",
    "cloudfront:List*",
    "cloudfront:Describe*"
  ],
  "Resource": "*"
}
```

---

## Core Commands

### 1. List All CloudFront Distributions

```bash
aws cloudfront list-distributions \
  --output table
```

**What it does:** Returns all CloudFront distributions in your account, including their IDs, domain names, origins, and statuses.

**Example Output:**
```json
{
  "DistributionList": {
    "Marker": "",
    "MaxItems": 100,
    "IsTruncated": false,
    "Quantity": 2,
    "Items": [
      {
        "Id": "E1EXAMPLE12345",
        "ARN": "arn:aws:cloudfront::123456789012:distribution/E1EXAMPLE12345",
        "Status": "Deployed",
        "DomainName": "d1example.cloudfront.net",
        "Origins": {
          "Quantity": 1,
          "Items": [
            {
              "Id": "my-s3-origin",
              "DomainName": "my-bucket.s3.amazonaws.com"
            }
          ]
        },
        "Enabled": true
      }
    ]
  }
}
```

---

### 2. Get a Specific Distribution

```bash
aws cloudfront get-distribution \
  --id E1EXAMPLE12345
```

**What it does:** Retrieves the full configuration and metadata for a specific CloudFront distribution, including its ETag (needed for updates).

**Example Output:**
```json
{
  "ETag": "E2QWRUHEXAMPLE",
  "Distribution": {
    "Id": "E1EXAMPLE12345",
    "ARN": "arn:aws:cloudfront::123456789012:distribution/E1EXAMPLE12345",
    "Status": "Deployed",
    "DomainName": "d1example.cloudfront.net",
    "DistributionConfig": {
      "Comment": "My production distribution",
      "Enabled": true,
      "DefaultCacheBehavior": {
        "ViewerProtocolPolicy": "redirect-to-https"
      }
    }
  }
}
```

---

### 3. Get Distribution Configuration

```bash
aws cloudfront get-distribution-config \
  --id E1EXAMPLE12345
```

**What it does:** Returns only the configuration portion of a distribution (not the metadata). The returned ETag is required for subsequent update operations.

---

### 4. Create a Distribution

```bash
aws cloudfront create-distribution \
  --distribution-config file://distribution-config.json
```

**`distribution-config.json`:**
```json
{
  "CallerReference": "my-distribution-2024-001",
  "Comment": "Production CDN for my-app.com",
  "Enabled": true,
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "my-s3-origin",
        "DomainName": "my-bucket.s3.amazonaws.com",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "my-s3-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"]
    },
    "ForwardedValues": {
      "QueryString": false,
      "Cookies": { "Forward": "none" }
    },
    "MinTTL": 0,
    "DefaultTTL": 86400,
    "MaxTTL": 31536000,
    "Compress": true,
    "TrustedSigners": {
      "Enabled": false,
      "Quantity": 0
    }
  },
  "DefaultRootObject": "index.html",
  "PriceClass": "PriceClass_100",
  "ViewerCertificate": {
    "CloudFrontDefaultCertificate": true
  }
}
```

**What it does:** Creates a new CloudFront distribution using the configuration defined in the JSON file. Returns the new distribution's ID and domain name.

---

### 5. Update a Distribution

```bash
# Step 1: Get current config and ETag
aws cloudfront get-distribution-config \
  --id E1EXAMPLE12345 \
  --output json > current-config.json

# Step 2: Extract ETag
ETAG=$(aws cloudfront get-distribution-config \
  --id E1EXAMPLE12345 \
  --query 'ETag' \
  --output text)

# Step 3: Apply updated config
aws cloudfront update-distribution \
  --id E1EXAMPLE12345 \
  --distribution-config file://updated-config.json \
  --if-match "$ETAG"
```

**What it does:** Updates an existing distribution. The `--if-match` flag with the current ETag prevents conflicting concurrent updates.

---

### 6. Disable a Distribution

```bash
# Get current ETag
ETAG=$(aws cloudfront get-distribution-config \
  --id E1EXAMPLE12345 \
  --query 'ETag' \
  --output text)

# Disable the distribution (set Enabled to false in config first)
aws cloudfront update-distribution \
  --id E1EXAMPLE12345 \
  --distribution-config file://disabled-config.json \
  --if-match "$ETAG"
```

**What it does:** Disables a distribution. A distribution must be disabled before it can be deleted. Disabling can take several minutes to propagate.

---

### 7. Delete a Distribution

```bash
# Get ETag of the disabled distribution
ETAG=$(aws cloudfront get-distribution \
  --id E1EXAMPLE12345 \
  --query 'ETag' \
  --output text)

aws cloudfront delete-distribution \
  --id E1EXAMPLE12345 \
  --if-match "$ETAG"
```

**What it does:** Permanently deletes a CloudFront distribution. The distribution must be in a `Disabled` state with `Status: Deployed` before deletion.

---

### 8. Create an Invalidation

```bash
aws cloudfront create-invalidation \
  --distribution-id E1EXAMPLE12345 \
  --paths "/*"
```

**Invalidate specific paths:**
```bash
aws cloudfront create-invalidation \
  --distribution-id E1EXAMPLE12345 \
  --paths "/index.html" "/assets/app.js" "/assets/styles.css"
```

**What it does:** Removes cached content from CloudFront edge locations, forcing the CDN to fetch fresh content from the origin on the next request.

**Example Output:**
```json
{
  "Location": "https://cloudfront.amazonaws.com/2020-05-31/distribution/E1EXAMPLE12345/invalidation/IEXAMPLE12345",
  "Invalidation": {
    "Id": "IEXAMPLE12345",
    "Status": "InProgress",
    "CreateTime": "2024-01-15T10:30:00.000Z",
    "InvalidationBatch": {
      "Paths": {
        "Quantity": 1,
        "Items": ["/*"]
      },
      "CallerReference": "cli-1705312200"
    }
  }
}
```

---

### 9. List Invalidations

```bash
aws cloudfront list-invalidations \
  --distribution-id E1EXAMPLE12345
```

**What it does:** Lists all invalidations for a distribution, showing their IDs, statuses (`InProgress` or `Completed`), and creation times.

---

### 10. Get Invalidation Status

```bash
aws cloudfront get-invalidation \
  --distribution-id E1EXAMPLE12345 \
  --id IEXAMPLE12345
```

**What it does:** Returns the current status of a specific invalidation request. Use this to check if an invalidation has completed.

**Example Output:**
```json
{
  "Invalidation": {
    "Id": "IEXAMPLE12345",
    "Status": "Completed",
    "CreateTime": "2024-01-15T10:30:00.000Z"
  }
}
```

---

### 11. Create an Origin Access Control (OAC)

```bash
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "my-s3-oac",
    "Description": "OAC for my production S3 bucket",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'
```

**What it does:** Creates an Origin Access Control, the modern replacement for Origin Access Identity (OAI). OAC restricts direct S3 bucket access, ensuring content is only served through CloudFront.

---

### 12. List Origin Access Controls

```bash
aws cloudfront list-origin-access-controls
```

**What it does:** Lists all Origin Access Controls in your account with their IDs and configurations.

---

### 13. Create a Cache Policy

```bash
aws cloudfront create-cache-policy \
  --cache-policy-config file://cache-policy.json
```

**`cache-policy.json`:**
```json
{
  "Name": "my-custom-cache-policy",
  "Comment": "Cache policy for API responses",
  "DefaultTTL": 3600,
  "MaxTTL": 86400,
  "MinTTL": 0,
  "ParametersInCacheKeyAndForwardedToOrigin": {
    "EnableAcceptEncodingGzip": true,
    "EnableAcceptEncodingBrotli": true,
    "HeadersConfig": {
      "HeaderBehavior": "none"
    },
    "CookiesConfig": {
      "CookieBehavior": "none"
    },
    "QueryStringsConfig": {
      "QueryStringBehavior": "whitelist",
      "QueryStrings": {
        "Quantity": 1,
        "Items": ["version"]
      }
    }
  }
}
```

**What it does:** Creates a reusable cache policy that defines TTL settings and which request attributes are included in the cache key.

---

### 14. List Cache Policies

```bash
# List managed (AWS) cache policies
aws cloudfront list-cache-policies \
  --type managed

# List custom cache policies
aws cloudfront list-cache-policies \
  --type custom
```

**What it does:** Lists available cache policies. `managed` returns AWS-provided policies; `custom` returns your own policies.

---

### 15. Tag a Distribution

```bash
aws cloudfront tag-resource \
  --resource "arn:aws:cloudfront::123456789012:distribution/E1EXAMPLE12345" \
  --tags 'Items=[{Key=Environment,Value=production},{Key=Project,Value=my-app},{Key=Team,Value=platform}]'
```

**What it does:** Adds or updates tags on a CloudFront distribution for cost allocation, resource organization, and access control.

---

## Common Operations

### Create Operations

**Create a distribution with HTTPS and custom domain:**
```bash
aws cloudfront create-distribution \
  --distribution-config file://https-distribution-config.json
```

**`https-distribution-config.json`:**
```json
{
  "CallerReference": "my-https-dist-2024",
  "Comment": "HTTPS distribution for www.my-app.com",
  "Enabled": true,
  "Aliases": {
    "Quantity": 2,
    "Items": ["www.my-app.com", "my-app.com"]
  },
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "alb-origin",
        "DomainName": "my-alb-1234567890.us-east-1.elb.amazonaws.com",
        "CustomOriginConfig": {
          "HTTPPort": 80,
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "https-only",
          "OriginSSLProtocols": {
            "Quantity": 1,
            "Items": ["TLSv1.2"]
          }
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "alb-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 7,
      "Items": ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
    },
    "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
    "OriginRequestPolicyId": "216adef6-5c7f-47e4-b989-5492eafa07d3",
    "Compress": true,
    "TrustedSigners": {
      "Enabled": false,
      "Quantity": 0
    }
  },
  "ViewerCertificate": {
    "ACMCertificateArn": "arn:aws:acm:us-east-1:123456789012:certificate/abc12345-1234-1234-1234-abc123456789",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "PriceClass": "PriceClass_All"
}
```

**Create a CloudFront function:**
```bash
aws cloudfront create-function \
  --name my-url-rewrite-function \
  --function-config '{"Comment":"URL rewrite for SPA","Runtime":"cloudfront-js-2.0"}' \
  --function-code fileb://function-code.js
```

**`function-code.js`:**
```javascript
function handler(event) {
  var request = event.request;
  var uri = request.uri;
  // Rewrite requests to index.html for SPA routing
  if (!uri.includes('.')) {
    request.uri = '/index.html';
  }
  return request;
}
```

**Create an origin request policy:**
```bash
aws cloudfront create-origin-request-policy \
  --origin-request-policy-config '{
    "Name": "my-origin-request-policy",
    "Comment": "Forward auth headers to origin",
    "HeadersConfig":