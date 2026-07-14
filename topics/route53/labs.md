# Route53 — Hands-On Labs

## Lab 1: Getting Started with Route53

### Objective
In this lab, you will create a Route53 hosted zone, register a test domain (or use an existing one), and configure basic DNS record types including A, CNAME, MX, and TXT records. By the end of this lab, you will understand how Route53 manages DNS and how to route traffic to AWS resources such as an EC2 instance and an S3 static website.

### Prerequisites

**AWS Services Required:**
- Amazon Route53
- Amazon EC2 (t2.micro or t3.micro — Free Tier eligible)
- Amazon S3
- AWS IAM

**IAM Permissions Required:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:*",
        "route53domains:*",
        "ec2:DescribeInstances",
        "s3:CreateBucket",
        "s3:PutBucketWebsite",
        "s3:PutObject",
        "s3:PutBucketPolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- A registered domain name (you can use Route53 domain registration, or use a free subdomain from a provider like `nip.io` for testing)
- `dig` or `nslookup` installed locally (for DNS verification)
- A web browser

**Estimated Cost:** < $1.00 (Hosted zone: $0.50/month prorated; EC2 t3.micro: ~$0.01/hour)

**Estimated Time:** 45–60 minutes

---

### Steps

#### Step 1: Create a Public Hosted Zone

**Console:**
1. Navigate to the [Route53 Console](https://console.aws.amazon.com/route53/).
2. In the left navigation pane, click **Hosted zones**.
3. Click **Create hosted zone**.
4. Fill in the form:
   - **Domain name:** `lab-example.com` *(replace with your actual domain)*
   - **Type:** Public hosted zone
   - **Comment:** `Route53 Lab 1 - Getting Started`
5. Click **Create hosted zone**.

**CLI:**
```bash
# Create a public hosted zone
aws route53 create-hosted-zone \
  --name "lab-example.com" \
  --caller-reference "lab1-$(date +%s)" \
  --hosted-zone-config Comment="Route53 Lab 1 - Getting Started",PrivateZone=false

# Save the Hosted Zone ID for later steps
HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name \
  --dns-name "lab-example.com" \
  --query "HostedZones[0].Id" \
  --output text | cut -d'/' -f3)

echo "Hosted Zone ID: $HOSTED_ZONE_ID"
```

**Verify:**
- In the console, you should see your hosted zone listed with two default records: an **NS record** and an **SOA record**.
- CLI: Run `aws route53 get-hosted-zone --id $HOSTED_ZONE_ID` and confirm the zone details.

**Expected Output (CLI):**
```json
{
    "HostedZone": {
        "Id": "/hostedzone/Z0123456789ABCDEFGHIJ",
        "Name": "lab-example.com.",
        "CallerReference": "lab1-1700000000",
        "Config": {
            "Comment": "Route53 Lab 1 - Getting Started",
            "PrivateZone": false
        },
        "ResourceRecordSetCount": 2
    }
}
```

---

#### Step 2: Launch an EC2 Instance to Use as a Target

**Console:**
1. Navigate to **EC2 > Instances > Launch instances**.
2. Configure:
   - **Name:** `route53-lab-web-server`
   - **AMI:** Amazon Linux 2023
   - **Instance type:** t3.micro
   - **Key pair:** Select or create one
   - **Security group:** Allow inbound HTTP (port 80) and SSH (port 22)
3. In **User data**, paste the following bootstrap script:
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Route53 Lab - Web Server</h1><p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" > /var/www/html/index.html
```
4. Click **Launch instance**.
5. Note the **Public IPv4 address** once the instance is running.

**CLI:**
```bash
# Get the latest Amazon Linux 2023 AMI ID
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" \
              "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

echo "Using AMI: $AMI_ID"

# Create a security group
SG_ID=$(aws ec2 create-security-group \
  --group-name "route53-lab-sg" \
  --description "Security group for Route53 lab" \
  --query "GroupId" --output text)

# Allow HTTP and SSH
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

# Create user data script
cat > /tmp/userdata.sh << 'EOF'
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Route53 Lab - Web Server</h1><p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" > /var/www/html/index.html
EOF

# Launch the instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --security-group-ids $SG_ID \
  --user-data file:///tmp/userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=route53-lab-web-server}]' \
  --query "Instances[0].InstanceId" \
  --output text)

echo "Instance ID: $INSTANCE_ID"

# Wait for the instance to be running
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# Get the public IP
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)

echo "Public IP: $PUBLIC_IP"
```

**Verify:**
- Instance state shows **Running** in the EC2 console.
- Navigate to `http://<PUBLIC_IP>` in your browser — you should see the "Route53 Lab - Web Server" page.

---

#### Step 3: Create an A Record Pointing to the EC2 Instance

**Console:**
1. Go to **Route53 > Hosted zones > lab-example.com**.
2. Click **Create record**.
3. Configure:
   - **Record name:** `www`
   - **Record type:** A
   - **Value:** `<EC2-PUBLIC-IP>` (e.g., `54.123.45.67`)
   - **TTL:** `300`
4. Click **Create records**.

**CLI:**
```bash
# Set your EC2 public IP
PUBLIC_IP="54.123.45.67"   # Replace with your actual EC2 public IP

# Create the A record using a change batch
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Comment": "Create A record for www",
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "www.lab-example.com",
          "Type": "A",
          "TTL": 300,
          "ResourceRecords": [
            {
              "Value": "'"$PUBLIC_IP"'"
            }
          ]
        }
      }
    ]
  }'
```

**Verify:**
```bash
# Verify the record was created
aws route53 list-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --query "ResourceRecordSets[?Name=='www.lab-example.com.']"

# Test DNS resolution (after updating your domain's NS records)
dig www.lab-example.com A
nslookup www.lab-example.com
```

**Expected Output:**
```
;; ANSWER SECTION:
www.lab-example.com.    300    IN    A    54.123.45.67
```

---

#### Step 4: Create a CNAME Record

**Console:**
1. In your hosted zone, click **Create record**.
2. Configure:
   - **Record name:** `blog`
   - **Record type:** CNAME
   - **Value:** `www.lab-example.com`
   - **TTL:** `300`
3. Click **Create records**.

**CLI:**
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Comment": "Create CNAME record for blog",
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "blog.lab-example.com",
          "Type": "CNAME",
          "TTL": 300,
          "ResourceRecords": [
            {
              "Value": "www.lab-example.com"
            }
          ]
        }
      }
    ]
  }'
```

**Verify:**
```bash
dig blog.lab-example.com CNAME
```

**Expected Output:**
```
;; ANSWER SECTION:
blog.lab-example.com.   300    IN    CNAME    www.lab-example.com.
www.lab-example.com.    300    IN    A        54.123.45.67
```

---

#### Step 5: Create TXT and MX Records

**Console:**
1. Create a **TXT record** for domain verification:
   - **Record name:** *(leave blank for apex)*
   - **Record type:** TXT
   - **Value:** `"v=spf1 include:amazonses.com ~all"`
   - **TTL:** `300`

2. Create an **MX record**:
   - **Record name:** *(leave blank for apex)*
   - **Record type:** MX
   - **Value:** `10 mail.lab-example.com`
   - **TTL:** `300`

**CLI:**
```bash
# Create both records in a single change batch
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Comment": "Create TXT and MX records",
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "lab-example.com",
          "Type": "TXT",
          "TTL": 300,
          "ResourceRecords": [
            {
              "Value": "\"v=spf1 include:amazonses.com ~all\""
            }
          ]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "lab-example.com",
          "Type": "MX",
          "TTL": 300,
          "ResourceRecords": [
            {
              "Value": "10 mail.lab-example.com"
            }
          ]
        }
      }
    ]
  }'
```

**Verify:**
```bash
# List all records in the hosted zone
aws route53 list-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --output table

dig lab-example.com TXT
dig lab-example.com MX
```

---

### Verification

Run the following checklist to confirm Lab 1 is complete:

```bash
echo "=== Route53 Lab 1 Verification ==="

echo "1. Checking hosted zone exists..."
aws route53 list-hosted-zones-by-name \
  --dns-name "lab-example.com" \
  --query "HostedZones[0].Name" --output text

echo "2. Listing all DNS records..."
aws route53 list-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --query "ResourceRecordSets[*].{Name:Name,Type:Type}" \
  --output table

echo "3. Verifying EC2 web server is accessible..."
curl -s -o /dev/null -w "%{http_code}" http://$PUBLIC_IP

echo ""
echo "Expected: 6 records (NS, SOA, A, CNAME, TXT, MX)"
```

**✅ Success Criteria:**
- [ ] Hosted zone `lab-example.com` exists and is public
- [ ] NS and SOA records are present (auto-created)
- [ ] `www.lab-example.com` A record points to EC2 public IP
- [ ] `blog.lab-example.com` CNAME points to `www.lab-example.com`
- [ ] TXT record with SPF value exists at apex
- [ ] MX record exists at apex
- [ ] EC2 web server returns HTTP 200

---

### Cleanup

> ⚠️ **Important:** Run cleanup in order to avoid ongoing charges ($0.50/month per hosted zone + EC2 costs).

```bash
echo "=== Starting Lab 1 Cleanup ==="

# Step 1: Delete all non-NS, non-SOA records first
# (Route53 won't let you delete a hosted zone with active records)
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "DELETE",
        "ResourceRecordSet": {
          "Name": "www.lab-example.com",
          "Type": "A",
          "TTL": 300,
          "ResourceRecords": [{"Value": "'"$PUBLIC_IP"'"}]
        }
      },
      {
        "Action": "DELETE",
        "ResourceRecordSet": {
          "Name": "blog.lab-example.com",
          "Type": "CNAME",
          "TTL": 300,
          "ResourceRecords": [{"Value": "www.lab-example.com"}]
        }
      },
      {
        "Action": "DELETE",
        "ResourceRecordSet": {
          "Name": "lab-example.com",
          "Type": "TXT",
          "TTL": 300,
          "ResourceRecords": [{"Value": "\"v=spf1 include:amazonses.com ~all\""}]
        }
      },
      {
        "Action": "DELETE",
        "ResourceRecordSet": {
          "Name": "lab-example.com",
          "Type": "MX",
          "TTL": 300,
          "ResourceRecords": [{"Value": "10 mail.lab-example.com"}]
        }
      }
    ]
  }'

# Step 2: Delete the hosted zone
aws route53 delete-hosted-zone --id $HOSTED_ZONE_ID

# Step 3: Terminate the EC2 instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-terminated --instance-