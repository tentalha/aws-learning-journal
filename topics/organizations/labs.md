# Organizations — Hands-On Labs

## Lab 1: Getting Started with Organizations

### Objective
In this lab, you will create an AWS Organization, invite a member account, and explore the organizational structure. By the end of this lab, you will understand how to create an Organization, navigate the management console, create Organizational Units (OUs), and move accounts into OUs.

### Prerequisites

**Required AWS Services:**
- AWS Organizations
- AWS IAM
- At least two AWS accounts (one management/master account, one member account)

**Required IAM Permissions (Management Account):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "organizations:*",
        "iam:CreateServiceLinkedRole"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools:**
- AWS CLI v2 installed and configured
- A web browser for AWS Console access
- AWS CLI profile configured for the management account (`aws configure --profile mgmt`)
- (Optional) A second AWS account email address for invitation

**Estimated Time:** 45–60 minutes  
**Estimated Cost:** $0 (AWS Organizations is free)

---

### Steps

#### Step 1: Create the AWS Organization

**Console:**
1. Sign in to the AWS Management Console with your management account.
2. Navigate to **AWS Organizations** (search "Organizations" in the top search bar).
3. Click **Create an organization**.
4. On the confirmation page, click **Create organization**.
5. Check your email — AWS will send a verification email to the management account's root email address. Click the verification link.

**CLI:**
```bash
# Create the organization with ALL features enabled
aws organizations create-organization \
  --feature-set ALL \
  --profile mgmt

# Expected output:
# {
#   "Organization": {
#     "Id": "o-exampleorgid11",
#     "Arn": "arn:aws:organizations::123456789012:organization/o-exampleorgid11",
#     "FeatureSet": "ALL",
#     "MasterAccountArn": "arn:aws:organizations::123456789012:account/o-exampleorgid11/123456789012",
#     "MasterAccountId": "123456789012",
#     "MasterAccountEmail": "admin@example.com",
#     ...
#   }
# }
```

**Verify:**
```bash
# Describe the organization to confirm creation
aws organizations describe-organization --profile mgmt
```
✅ **Expected Result:** You should see the Organization details including `Id`, `FeatureSet: ALL`, and `MasterAccountId`.

---

#### Step 2: Explore the Root and Default Structure

**Console:**
1. In the Organizations console, click **AWS accounts** in the left navigation.
2. Observe the **Root** at the top of the hierarchy.
3. Note that your management account is already listed under Root.

**CLI:**
```bash
# List the roots in your organization
aws organizations list-roots --profile mgmt

# Save the Root ID for later use
ROOT_ID=$(aws organizations list-roots \
  --profile mgmt \
  --query 'Roots[0].Id' \
  --output text)

echo "Root ID: $ROOT_ID"
# Expected: r-xxxx (e.g., r-ab12)

# List accounts in the organization
aws organizations list-accounts --profile mgmt
```

✅ **Expected Result:** One root (e.g., `r-ab12`) and one account (your management account) listed.

---

#### Step 3: Create Organizational Units (OUs)

**Console:**
1. In the Organizations console, select the **Root**.
2. Click **Actions** → **Create new**.
3. Enter the name `Production` and click **Create organizational unit**.
4. Repeat to create two more OUs: `Development` and `Shared-Services`.

**CLI:**
```bash
# Create the Production OU under Root
aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name Production \
  --profile mgmt

# Create the Development OU
aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name Development \
  --profile mgmt

# Create the Shared-Services OU
aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name Shared-Services \
  --profile mgmt

# List all OUs under Root
aws organizations list-organizational-units-for-parent \
  --parent-id $ROOT_ID \
  --profile mgmt
```

✅ **Expected Result:** Three OUs listed: `Production`, `Development`, and `Shared-Services`, each with a unique `Id` like `ou-ab12-xxxxxxxx`.

---

#### Step 4: Invite a Member Account

**Console:**
1. In the Organizations console, click **AWS accounts** → **Add an AWS account**.
2. Select **Invite an existing AWS account**.
3. Enter the Account ID or email address of your second AWS account.
4. Add a message: `"Joining our AWS Organization for lab purposes"`.
5. Click **Send invitation**.

**CLI:**
```bash
# Replace with your second account's ID or email
MEMBER_ACCOUNT_EMAIL="member-account@example.com"

aws organizations invite-account-to-organization \
  --target '{"Type": "EMAIL", "Id": "'"$MEMBER_ACCOUNT_EMAIL"'"}' \
  --notes "Joining our AWS Organization for lab purposes" \
  --profile mgmt

# List open handshakes to confirm the invitation was sent
aws organizations list-handshakes-for-organization \
  --profile mgmt
```

**Accept the invitation (from the member account):**
```bash
# Configure a second CLI profile for the member account
# aws configure --profile member

# List incoming handshakes
aws organizations list-handshakes-for-account \
  --profile member

# Get the handshake ID
HANDSHAKE_ID=$(aws organizations list-handshakes-for-account \
  --profile member \
  --query 'Handshakes[0].Id' \
  --output text)

# Accept the invitation
aws organizations accept-handshake \
  --handshake-id $HANDSHAKE_ID \
  --profile member
```

✅ **Expected Result:** The member account now appears in the **AWS accounts** list with status `ACTIVE`.

---

#### Step 5: Move the Member Account into an OU

**Console:**
1. In the Organizations console, select the member account.
2. Click **Actions** → **Move**.
3. Select the `Development` OU and click **Move AWS account**.

**CLI:**
```bash
# Get the member account ID
MEMBER_ACCOUNT_ID=$(aws organizations list-accounts \
  --profile mgmt \
  --query 'Accounts[?Name!=`management`].Id' \
  --output text | head -1)

# Get the Development OU ID
DEV_OU_ID=$(aws organizations list-organizational-units-for-parent \
  --parent-id $ROOT_ID \
  --profile mgmt \
  --query 'OrganizationalUnits[?Name==`Development`].Id' \
  --output text)

echo "Member Account ID: $MEMBER_ACCOUNT_ID"
echo "Development OU ID: $DEV_OU_ID"

# Move the account from Root to Development OU
aws organizations move-account \
  --account-id $MEMBER_ACCOUNT_ID \
  --source-parent-id $ROOT_ID \
  --destination-parent-id $DEV_OU_ID \
  --profile mgmt

# Verify the account is now in the Development OU
aws organizations list-accounts-for-parent \
  --parent-id $DEV_OU_ID \
  --profile mgmt
```

✅ **Expected Result:** The member account appears under the `Development` OU.

---

### Verification

Run the following commands to confirm successful lab completion:

```bash
# 1. Verify organization exists with ALL features
aws organizations describe-organization \
  --profile mgmt \
  --query 'Organization.{Id:Id, Features:FeatureSet}' \
  --output table

# 2. Verify three OUs exist under Root
aws organizations list-organizational-units-for-parent \
  --parent-id $ROOT_ID \
  --profile mgmt \
  --query 'OrganizationalUnits[*].Name' \
  --output table

# 3. Verify member account is in Development OU
aws organizations list-accounts-for-parent \
  --parent-id $DEV_OU_ID \
  --profile mgmt \
  --query 'Accounts[*].{Id:Id, Name:Name, Status:Status}' \
  --output table
```

**Expected Final State:**
| Check | Expected Value |
|-------|---------------|
| Organization FeatureSet | `ALL` |
| Number of OUs | 3 (Production, Development, Shared-Services) |
| Member Account Status | `ACTIVE` |
| Member Account Location | `Development` OU |

---

### Cleanup

> ⚠️ **Warning:** Removing accounts from an organization is irreversible unless you re-invite them. Proceed carefully.

```bash
# Step 1: Move member account back to Root before removal
aws organizations move-account \
  --account-id $MEMBER_ACCOUNT_ID \
  --source-parent-id $DEV_OU_ID \
  --destination-parent-id $ROOT_ID \
  --profile mgmt

# Step 2: Remove the member account from the organization
# NOTE: The member account must have a valid payment method configured first
aws organizations remove-account-from-organization \
  --account-id $MEMBER_ACCOUNT_ID \
  --profile mgmt

# Step 3: Delete the OUs (must be empty first)
for OU_NAME in Production Development Shared-Services; do
  OU_ID=$(aws organizations list-organizational-units-for-parent \
    --parent-id $ROOT_ID \
    --profile mgmt \
    --query "OrganizationalUnits[?Name=='$OU_NAME'].Id" \
    --output text)
  
  if [ -n "$OU_ID" ]; then
    aws organizations delete-organizational-unit \
      --organizational-unit-id $OU_ID \
      --profile mgmt
    echo "Deleted OU: $OU_NAME ($OU_ID)"
  fi
done

# Step 4: Delete the organization (optional — only if you want to fully remove it)
# WARNING: This is irreversible
# aws organizations delete-organization --profile mgmt

# Step 5: Verify cleanup
aws organizations list-organizational-units-for-parent \
  --parent-id $ROOT_ID \
  --profile mgmt
```

---

## Lab 2: Intermediate Organizations Configuration

### Objective
In this lab, you will implement **Service Control Policies (SCPs)** to enforce governance guardrails across your organization. You will create SCPs that deny access to specific AWS regions, prevent disabling of AWS CloudTrail, and restrict the creation of IAM users in member accounts. You will also learn how to attach SCPs to OUs and test policy enforcement.

### Prerequisites

**Required AWS Services:**
- AWS Organizations (with ALL features enabled — from Lab 1)
- AWS IAM
- AWS CloudTrail
- AWS STS

**Required IAM Permissions:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "organizations:*",
        "cloudtrail:*",
        "iam:*",
        "sts:AssumeRole"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools:**
- AWS CLI v2 with management account profile (`mgmt`)
- Member account CLI profile (`member`)
- `jq` installed for JSON parsing (optional but recommended)
- Completed Lab 1 (Organization with OUs and member account)

**Estimated Time:** 60–90 minutes  
**Estimated Cost:** $0

---

### Steps

#### Step 1: Enable SCP Policy Type on the Root

**Console:**
1. Navigate to **AWS Organizations** → **Policies** in the left menu.
2. Click **Service control policies**.
3. Click **Enable service control policies**.

**CLI:**
```bash
# Get Root ID
ROOT_ID=$(aws organizations list-roots \
  --profile mgmt \
  --query 'Roots[0].Id' \
  --output text)

# Enable SCP policy type on the Root
aws organizations enable-policy-type \
  --root-id $ROOT_ID \
  --policy-type SERVICE_CONTROL_POLICY \
  --profile mgmt

# Verify SCPs are enabled
aws organizations describe-effective-policy \
  --policy-type SERVICE_CONTROL_POLICY \
  --profile mgmt 2>/dev/null || \
aws organizations list-roots \
  --profile mgmt \
  --query 'Roots[0].PolicyTypes'
```

✅ **Expected Result:** `PolicyTypes` shows `{"Type": "SERVICE_CONTROL_POLICY", "Status": "ENABLED"}`.

---

#### Step 2: Create a Region Restriction SCP

**Console:**
1. Go to **Organizations** → **Policies** → **Service control policies**.
2. Click **Create policy**.
3. Name it `DenyNonApprovedRegions`.
4. Paste the policy JSON below and click **Create policy**.

**Create the policy file:**
```bash
cat > /tmp/deny-non-approved-regions.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "a4b:*",
        "acm:*",
        "aws-marketplace-management:*",
        "aws-marketplace:*",
        "budgets:*",
        "ce:*",
        "chime:*",
        "cloudfront:*",
        "config:*",
        "cur:*",
        "directconnect:*",
        "ec2:DescribeRegions",
        "ec2:DescribeTransitGateways",
        "fms:*",
        "globalaccelerator:*",
        "health:*",
        "iam:*",
        "importexport:*",
        "kms:*",
        "mobileanalytics:*",
        "networkmanager:*",
        "organizations:*",
        "pricing:*",
        "route53:*",
        "route53domains:*",
        "s3:GetAccountPublic*",
        "s3:ListAllMyBuckets",
        "s3:PutAccountPublic*",
        "shield:*",
        "sts:*",
        "support:*",
        "trustedadvisor:*",
        "waf-regional:*",
        "waf:*",
        "wafv2:*",
        "wellarchitected:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2",
            "eu-west-1"
          ]
        }
      }
    }
  ]
}
EOF
```

```bash
# Create the SCP
aws organizations create-policy \
  --name DenyNonApprovedRegions \
  --description "Denies all actions outside of approved AWS regions" \
  --content file:///tmp/deny-non-approved-regions.json \
  --type SERVICE_CONTROL_POLICY \
  --profile mgmt

# Save the policy ID
REGION_SCP_ID=$(aws organizations list-policies \
  --filter SERVICE_CONTROL_POLICY \
  --profile mgmt \
  --query 'Policies[?Name==`DenyNonApprovedRegions`].Id' \
  --output text)

echo "Region SCP ID: $REGION_SCP_ID"
```

✅ **Expected Result:** Policy created with an ID like `p-xxxxxxxx`.

---

#### Step 3: Create a CloudTrail Protection SCP

```bash
cat > /tmp/protect-cloudtrail.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyCloudTrailModification",