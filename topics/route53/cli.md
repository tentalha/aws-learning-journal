# Route53 — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Ensure the AWS CLI is installed and configured with appropriate credentials:

```bash
# Install/update AWS CLI
pip install --upgrade awscli

# Configure credentials and default region
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

### Required IAM Permissions

Attach the following managed policies or create a custom policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:*",
        "route53domains:*",
        "route53resolver:*",
        "cloudwatch:GetMetricStatistics",
        "elasticloadbalancing:DescribeLoadBalancers",
        "elasticbeanstalk:DescribeEnvironments",
        "s3:GetBucketLocation",
        "s3:GetBucketWebsite"
      ],
      "Resource": "*"
    }
  ]
}
```

**Managed Policy Alternatives:**
- `AmazonRoute53FullAccess` — Full access to Route 53
- `AmazonRoute53ReadOnlyAccess` — Read-only access
- `AmazonRoute53DomainsFullAccess` — Domain registration management
- `AmazonRoute53ResolverFullAccess` — DNS Resolver management

### Verify Setup

```bash
# Verify CLI access to Route53
aws route53 list-hosted-zones

# Check current caller identity
aws sts get-caller-identity
```

> **Note:** Route53 is a global service. Most Route53 API calls do not require a `--region` flag, but `route53domains` commands must target `us-east-1`.

---

## Core Commands

### 1. List All Hosted Zones

```bash
aws route53 list-hosted-zones
```

**What it does:** Returns all hosted zones associated with the current AWS account.

**Example Output:**
```json
{
    "HostedZones": [
        {
            "Id": "/hostedzone/Z1D633PJN98FT9",
            "Name": "example.com.",
            "CallerReference": "cli-1609459200",
            "Config": {
                "Comment": "Production hosted zone",
                "PrivateZone": false
            },
            "ResourceRecordSetCount": 12
        }
    ],
    "IsTruncated": false,
    "MaxItems": "100"
}
```

---

### 2. Get a Specific Hosted Zone

```bash
aws route53 get-hosted-zone \
  --id Z1D633PJN98FT9
```

**What it does:** Retrieves detailed information about a specific hosted zone, including its delegation set (name servers).

**Example Output:**
```json
{
    "HostedZone": {
        "Id": "/hostedzone/Z1D633PJN98FT9",
        "Name": "example.com.",
        "Config": {
            "Comment": "Production hosted zone",
            "PrivateZone": false
        },
        "ResourceRecordSetCount": 12
    },
    "DelegationSet": {
        "NameServers": [
            "ns-1234.awsdns-12.org",
            "ns-567.awsdns-34.co.uk",
            "ns-890.awsdns-56.net",
            "ns-1111.awsdns-78.com"
        ]
    }
}
```

---

### 3. Create a Public Hosted Zone

```bash
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "cli-$(date +%s)" \
  --hosted-zone-config Comment="Production zone for example.com",PrivateZone=false
```

**What it does:** Creates a new public hosted zone for the specified domain. The `--caller-reference` must be unique per request to ensure idempotency.

---

### 4. Create a Private Hosted Zone

```bash
aws route53 create-hosted-zone \
  --name internal.example.com \
  --caller-reference "private-$(date +%s)" \
  --hosted-zone-config Comment="Internal VPC zone",PrivateZone=true \
  --vpc VPCRegion=us-east-1,VPCId=vpc-0abc12345def67890
```

**What it does:** Creates a private hosted zone associated with a specific VPC, enabling internal DNS resolution.

---

### 5. List Resource Record Sets in a Hosted Zone

```bash
aws route53 list-resource-record-sets \
  --hosted-zone-id Z1D633PJN98FT9
```

**What it does:** Lists all DNS records within a hosted zone.

**Example Output:**
```json
{
    "ResourceRecordSets": [
        {
            "Name": "example.com.",
            "Type": "A",
            "TTL": 300,
            "ResourceRecords": [
                { "Value": "203.0.113.10" }
            ]
        },
        {
            "Name": "www.example.com.",
            "Type": "CNAME",
            "TTL": 300,
            "ResourceRecords": [
                { "Value": "example.com" }
            ]
        }
    ],
    "IsTruncated": false,
    "MaxItems": "300"
}
```

---

### 6. Create or Update a DNS Record (Change Resource Record Sets)

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1D633PJN98FT9 \
  --change-batch '{
    "Comment": "Add A record for web server",
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "www.example.com",
          "Type": "A",
          "TTL": 300,
          "ResourceRecords": [
            { "Value": "203.0.113.10" }
          ]
        }
      }
    ]
  }'
```

**What it does:** Creates, updates, or deletes DNS records. The `Action` can be `CREATE`, `UPSERT`, or `DELETE`.

---

### 7. Create an Alias Record (Pointing to AWS Resource)

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1D633PJN98FT9 \
  --change-batch '{
    "Comment": "Alias record pointing to ALB",
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "app.example.com",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "my-alb-1234567890.us-east-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'
```

**What it does:** Creates an Alias record that points to an AWS resource (ALB, CloudFront, S3, etc.) without incurring additional DNS query charges.

---

### 8. Get Change Status

```bash
aws route53 get-change \
  --id C2682N5HXP0BZ4
```

**What it does:** Checks the status of a DNS change request. Status can be `PENDING` or `INSYNC`.

**Example Output:**
```json
{
    "ChangeInfo": {
        "Id": "/change/C2682N5HXP0BZ4",
        "Status": "INSYNC",
        "SubmittedAt": "2024-01-15T10:30:00.000Z",
        "Comment": "Add A record for web server"
    }
}
```

---

### 9. Delete a DNS Record

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1D633PJN98FT9 \
  --change-batch '{
    "Comment": "Delete A record for old server",
    "Changes": [
      {
        "Action": "DELETE",
        "ResourceRecordSet": {
          "Name": "old-server.example.com",
          "Type": "A",
          "TTL": 300,
          "ResourceRecords": [
            { "Value": "203.0.113.20" }
          ]
        }
      }
    ]
  }'
```

**What it does:** Deletes a specific DNS record. The record details must exactly match what is currently in Route53.

---

### 10. Delete a Hosted Zone

```bash
aws route53 delete-hosted-zone \
  --id Z1D633PJN98FT9
```

**What it does:** Deletes a hosted zone. The zone must be empty (only SOA and NS records may remain) before deletion.

---

### 11. Create a Health Check

```bash
aws route53 create-health-check \
  --caller-reference "hc-$(date +%s)" \
  --health-check-config '{
    "IPAddress": "203.0.113.10",
    "Port": 443,
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "FullyQualifiedDomainName": "www.example.com",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'
```

**What it does:** Creates a health check that monitors an endpoint. Can be used with routing policies for failover.

---

### 12. List Health Checks

```bash
aws route53 list-health-checks
```

**What it does:** Returns all health checks configured in the account.

---

### 13. Create a Traffic Policy

```bash
aws route53 create-traffic-policy \
  --name "weighted-routing-policy" \
  --document file://traffic-policy.json \
  --comment "Weighted routing between two regions"
```

**What it does:** Creates a reusable traffic policy that defines complex routing logic using a JSON document.

---

### 14. List Reusable Delegation Sets

```bash
aws route53 list-reusable-delegation-sets
```

**What it does:** Lists all reusable delegation sets, which allow consistent name server assignments across multiple hosted zones.

---

### 15. Test DNS Query (Test DNS Answer)

```bash
aws route53 test-dns-answer \
  --hosted-zone-id Z1D633PJN98FT9 \
  --record-name www.example.com \
  --record-type A
```

**What it does:** Simulates a DNS query against Route53 to verify how it would respond, without making an actual DNS query.

**Example Output:**
```json
{
    "Nameserver": "ns-1234.awsdns-12.org",
    "RecordName": "www.example.com.",
    "RecordType": "A",
    "RecordData": ["203.0.113.10"],
    "ResponseCode": "NOERROR",
    "Protocol": "UDP"
}
```

---

## Common Operations

### Create Operations

#### Create a Hosted Zone with Tags

```bash
# Step 1: Create the hosted zone
ZONE_ID=$(aws route53 create-hosted-zone \
  --name staging.example.com \
  --caller-reference "staging-$(date +%s)" \
  --hosted-zone-config Comment="Staging environment",PrivateZone=false \
  --query 'HostedZone.Id' \
  --output text | cut -d'/' -f3)

# Step 2: Tag the hosted zone
aws route53 change-tags-for-resource \
  --resource-type hostedzone \
  --resource-id "$ZONE_ID" \
  --add-tags \
    Key=Environment,Value=Staging \
    Key=Team,Value=DevOps \
    Key=CostCenter,Value=CC-1234
```

#### Create Multiple DNS Records in One Batch

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1D633PJN98FT9 \
  --change-batch '{
    "Comment": "Initial DNS setup for example.com",
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "example.com",
          "Type": "MX",
          "TTL": 3600,
          "ResourceRecords": [
            { "Value": "10 mail.example.com" },
            { "Value": "20 mail2.example.com" }
          ]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "example.com",
          "Type": "TXT",
          "TTL": 300,
          "ResourceRecords": [
            { "Value": "\"v=spf1 include:_spf.google.com ~all\"" }
          ]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "_dmarc.example.com",
          "Type": "TXT",
          "TTL": 300,
          "ResourceRecords": [
            { "Value": "\"v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com\"" }
          ]
        }
      }
    ]
  }'
```

#### Create a Weighted Routing Record

```bash
# Primary record (70% traffic)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1D633PJN98FT9 \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.example.com",
          "Type": "A",
          "SetIdentifier": "primary-us-east-1",
          "Weight": 70,
          "TTL": 60,
          "ResourceRecords": [
            { "Value": "203.0.113.10" }
          ]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.example.com",
          "Type": "A",
          "SetIdentifier": "secondary-us-west-2",
          "Weight": 30,
          "TTL": 60,
          "ResourceRecords": [
            { "Value": "203.0.113.20" }
          ]
        }
      }
    ]
  }'
```

#### Create a Failover Routing Record

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1D633PJN98FT9 \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "app.example.com",
          "Type": "A",
          "SetIdentifier": "primary-failover",
          "Failover": "PRIMARY",
          "TTL": 60,
          "ResourceRecords": [
            { "Value": "203.0.113.10" }
          ],
          "HealthCheckId": "abcdef12-1234-1234-1234-abcdef123456"
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "app.example.com",
          "Type": "A",
          "SetIdentifier": "secondary-failover",
          "Failover": "SECONDARY",
          "TTL": 60,
          "ResourceRecords": [
            { "Value": "203.0.113.20" }
          ]
        }
      }
    ]
  }'
```

---

### Read / Describe Operations

#### Get Hosted Zone Details

```bash
aws route53 get-hosted-zone \
  --id Z1D633PJ