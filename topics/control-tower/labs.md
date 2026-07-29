# Control Tower — Hands-On Labs

## Lab 1: Getting Started with Control Tower

### Objective
In this lab, you will set up AWS Control Tower for the first time in a management account, create a Landing Zone, and explore the Account Factory. You will learn how Control Tower automates multi-account governance by provisioning baseline guardrails (Service Control Policies and AWS Config rules), creating the Log Archive and Audit accounts, and enabling centralized logging. By the end of this lab, you will have a functional Landing Zone with organizational units (OUs) and understand how Control Tower enforces baseline security policies across your AWS Organization.

### Prerequisites

**AWS Services Required:**
- AWS Control Tower (management account)
- AWS Organizations
- AWS IAM Identity Center (formerly SSO)
- AWS CloudTrail
- AWS Config
- Amazon S3
- AWS SNS

**IAM Permissions:**
- You must be signed in as the **root user** or an IAM user/role with `AdministratorAccess` on the management account
- The management account must **not** already have AWS Organizations enabled (or must be the master account of an existing organization)

**Account Requirements:**
- A dedicated AWS management account (do not use a production account)
- At least **two additional email addresses** not associated with any existing AWS account (for Log Archive and Audit accounts)
- Service limits: Ensure your account can create at least 3 additional accounts

**Tools:**
- AWS Management Console access
- AWS CLI v2 installed and configured
- `jq` (optional, for JSON parsing)

**Estimated Time:** 60–90 minutes  
**Estimated Cost:** < $5 USD (primarily from Config rules and CloudTrail)

> ⚠️ **Warning:** Control Tower setup takes approximately 45–60 minutes to complete. Do not navigate away or close your browser during setup.

---

### Steps

#### Step 1: Verify Management Account Prerequisites

Before enabling Control Tower, verify your account meets all prerequisites.

**Console:**
1. Sign in to the AWS Management Console at `https://console.aws.amazon.com`
2. Navigate to **AWS Organizations** → Check if an organization already exists
3. Navigate to **IAM** → **Account Settings** → Verify no conflicting SCPs exist
4. Navigate to **AWS Config** → Verify Config is **not** already enabled in the management account (or be prepared to reconcile)

**CLI:**
```bash
# Check current account ID
aws sts get-caller-identity

# Expected output:
# {
#     "UserId": "AIDAXXXXXXXXXXXXXXXXX",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/admin"
# }

# Check if Organizations is already configured
aws organizations describe-organization 2>&1 || echo "No organization found - OK to proceed"

# Check AWS Config status
aws configservice describe-configuration-recorders \
  --query 'ConfigurationRecorders[*].name' \
  --output text

# Check current region
aws configure get region
```

**Verify:**
- You are authenticated as an admin user in the management account
- Note your 12-digit Account ID
- Confirm the region you want to use as your Control Tower home region (e.g., `us-east-1`)

---

#### Step 2: Enable AWS Control Tower

**Console:**
1. Navigate to the **AWS Control Tower** console: `https://console.aws.amazon.com/controltower`
2. Click **"Set up landing zone"**
3. On the **Review pricing and select Regions** page:
   - **Home Region:** Select `us-east-1` (US East - N. Virginia)
   - **Additional Regions:** Leave unchecked for now
   - Check the acknowledgment box for pricing
   - Click **Next**

4. On the **Configure organizational units** page:
   - **Security OU name:** `Security` (keep default)
   - **Sandbox OU name:** `Sandbox` (keep default)
   - Click **Next**

5. On the **Configure shared accounts** page:
   - **Log Archive account email:** Enter a unique email (e.g., `aws-log-archive-lab@yourdomain.com`)
   - **Log Archive account name:** `Log Archive`
   - **Audit account email:** Enter a unique email (e.g., `aws-audit-lab@yourdomain.com`)
   - **Audit account name:** `Audit`
   - Click **Next**

6. On the **Get access to AWS IAM Identity Center** page:
   - Select **"Enable and configure IAM Identity Center"**
   - Click **Next**

7. On the **Review and set up landing zone** page:
   - Review all settings carefully
   - Check both acknowledgment boxes
   - Click **"Set up landing zone"**

> ⏳ **Note:** Setup will take 45–60 minutes. You will see a progress tracker in the console.

**CLI (Verification only — Control Tower setup must be done via Console or API):**
```bash
# Monitor Control Tower setup status
aws controltower get-landing-zone-operation \
  --operation-identifier $(aws controltower list-landing-zone-operations \
    --query 'landingZoneOperations[0].operationIdentifier' \
    --output text) \
  --query 'operationDetails.status' \
  --output text

# List landing zones once setup is complete
aws controltower list-landing-zones \
  --query 'landingZones[*].{Arn:arn,Status:status}' \
  --output table
```

**Verify:**
- The Control Tower dashboard shows **"Your landing zone is ready"**
- Status shows **"Succeeded"** with a green checkmark
- You see 2 OUs: `Security` and `Sandbox`
- You see 3 accounts: Management, Log Archive, Audit

**Expected Output:**
```
Landing Zone Status: ACTIVE
Organizational Units: Security, Sandbox
Shared Accounts: Log Archive, Audit
Guardrails Enabled: ~20 mandatory guardrails
```

---

#### Step 3: Explore the Control Tower Dashboard

**Console:**
1. Navigate to **Control Tower** → **Dashboard**
2. Review the following sections:
   - **Guardrails summary** — Note the number of enabled/non-compliant guardrails
   - **Accounts** — Verify 3 accounts are shown
   - **OUs** — Verify Security and Sandbox OUs exist
3. Navigate to **Control Tower** → **Controls** (formerly Guardrails)
4. Filter by **"Mandatory"** to see all mandatory controls
5. Click on **"Disallow Changes to AWS CloudTrail Configuration"** to inspect its details

**CLI:**
```bash
# List all enabled controls (guardrails) in the landing zone
aws controltower list-enabled-controls \
  --target-identifier $(aws organizations list-roots \
    --query 'Roots[0].Arn' --output text) \
  --query 'enabledControls[*].{Control:controlIdentifier,Status:statusSummary.status}' \
  --output table

# List all OUs in the organization
aws organizations list-organizational-units-for-parent \
  --parent-id $(aws organizations list-roots --query 'Roots[0].Id' --output text) \
  --query 'OrganizationalUnits[*].{Name:Name,Id:Id}' \
  --output table

# List all accounts in the organization
aws organizations list-accounts \
  --query 'Accounts[*].{Name:Name,Id:Id,Status:Status,Email:Email}' \
  --output table
```

**Verify:**
- At least 20 mandatory guardrails are shown as **Enabled**
- All accounts show **ACTIVE** status
- OUs are correctly named

---

#### Step 4: Create a New Organizational Unit

**Console:**
1. Navigate to **Control Tower** → **Organization**
2. Click **"Create organizational unit"**
3. Enter:
   - **OU Name:** `Workloads`
   - **Parent OU:** Root
4. Click **"Create"**
5. Wait for the OU to be created (1–2 minutes)

**CLI:**
```bash
# Get the Root ID
ROOT_ID=$(aws organizations list-roots --query 'Roots[0].Id' --output text)
echo "Root ID: $ROOT_ID"

# Create a new OU
aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Workloads" \
  --query 'OrganizationalUnit.{Name:Name,Id:Id,Arn:Arn}' \
  --output table

# Verify the OU was created
aws organizations list-organizational-units-for-parent \
  --parent-id $ROOT_ID \
  --query 'OrganizationalUnits[*].{Name:Name,Id:Id}' \
  --output table
```

**Verify:**
- The `Workloads` OU appears in the Control Tower Organization view
- The OU shows **"Registered"** status in Control Tower

---

#### Step 5: Explore Account Factory

**Console:**
1. Navigate to **Control Tower** → **Account Factory**
2. Click **"Create account"**
3. Review the form fields (do **NOT** submit yet):
   - Account email
   - Display name
   - IAM Identity Center user email
   - Organizational Unit
   - Account Factory parameters
4. Click **Cancel** (we will provision accounts in Lab 2)
5. Navigate to **Account Factory** → **Parameters** to review default VPC settings

**CLI:**
```bash
# List available Account Factory products in Service Catalog
aws servicecatalog search-products \
  --filters Key=FullTextSearch,Value="AWS Control Tower Account Factory" \
  --query 'ProductViewSummaries[*].{Name:Name,ProductId:ProductId}' \
  --output table

# List Account Factory provisioned products
aws servicecatalog search-provisioned-products \
  --access-level-filter Key=Account,Value=self \
  --query 'ProvisionedProducts[*].{Name:Name,Status:Status,Type:Type}' \
  --output table
```

**Verify:**
- Account Factory is accessible
- Service Catalog shows the Account Factory product
- Existing accounts (Log Archive, Audit) appear as provisioned products

---

### Verification

Run the following verification checklist:

```bash
#!/bin/bash
# Control Tower Lab 1 Verification Script

echo "=== Control Tower Lab 1 Verification ==="
echo ""

# 1. Verify Landing Zone is Active
echo "1. Checking Landing Zone status..."
aws controltower list-landing-zones \
  --query 'landingZones[0].status' \
  --output text

# 2. Verify OUs exist
echo ""
echo "2. Checking Organizational Units..."
ROOT_ID=$(aws organizations list-roots --query 'Roots[0].Id' --output text)
aws organizations list-organizational-units-for-parent \
  --parent-id $ROOT_ID \
  --query 'OrganizationalUnits[*].Name' \
  --output text

# 3. Verify accounts
echo ""
echo "3. Checking AWS Accounts..."
aws organizations list-accounts \
  --query 'Accounts[*].{Name:Name,Status:Status}' \
  --output table

# 4. Verify CloudTrail is enabled
echo ""
echo "4. Checking CloudTrail..."
aws cloudtrail describe-trails \
  --query 'trailList[*].{Name:Name,IsOrganizationTrail:IsOrganizationTrail}' \
  --output table

# 5. Verify S3 log bucket exists
echo ""
echo "5. Checking S3 Log Buckets..."
aws s3 ls | grep -E "aws-controltower|log-archive"

echo ""
echo "=== Verification Complete ==="
```

**Expected Results:**
- Landing Zone status: `ACTIVE`
- OUs: `Security`, `Sandbox`, `Workloads` (minimum)
- Accounts: Management, Log Archive, Audit — all `ACTIVE`
- CloudTrail: Organization trail enabled
- S3: Log archive bucket exists

---

### Cleanup

> ⚠️ **Important:** Decommissioning Control Tower is a multi-step process. The Log Archive and Audit accounts will **persist** even after cleanup unless manually closed. Full cleanup of a Landing Zone is complex and irreversible.

```bash
# Step 1: Note all resources created for cleanup reference
echo "=== Resources to Clean Up ==="

# List S3 buckets created by Control Tower
aws s3 ls | grep controltower

# List CloudTrail trails
aws cloudtrail describe-trails --query 'trailList[*].Name' --output text

# List Config recorders
aws configservice describe-configuration-recorders \
  --query 'ConfigurationRecorders[*].name' --output text
```

**Console Cleanup Steps:**

1. Navigate to **Control Tower** → **Landing zone settings**
2. If you want to fully decommission (for lab purposes only):
   - Click **"Delete landing zone"**
   - Type the confirmation text
   - Click **"Delete"**
   - ⏳ This takes 20–30 minutes

3. **After decommission, manually clean up:**
   ```bash
   # Delete Control Tower S3 buckets (empty them first)
   # List buckets
   aws s3 ls | grep -E "aws-controltower|aws-logs"
   
   # Empty and delete each bucket (replace BUCKET_NAME)
   BUCKET_NAME="aws-controltower-logs-123456789012-us-east-1"
   aws s3 rm s3://$BUCKET_NAME --recursive
   aws s3 rb s3://$BUCKET_NAME
   ```

4. **Close member accounts** (Log Archive and Audit):
   - Navigate to **AWS Organizations** → **Accounts**
   - Select each account → **Close account**
   - ⚠️ Account closure takes 90 days to fully process

> 💡 **Lab Tip:** For training purposes, you may choose to keep the Landing Zone active and use it for Labs 2 and 3.

---

## Lab 2: Intermediate Control Tower Configuration

### Objective
In this lab, you will build upon the Landing Zone created in Lab 1 by provisioning a new member account using Account Factory, applying custom guardrails (Controls) to an Organizational Unit, configuring Account Factory for Terraform (AFT) concepts, and enabling additional preventive and detective controls. You will also integrate Control Tower with AWS Config Conformance Packs and explore drift detection. By the end, you will have a governed member account with custom controls enforced.

### Prerequisites

**From Lab 1:**
- Completed Lab 1 with an active Landing Zone
- `Security`, `Sandbox`, and `Workloads` OUs exist
- Management account credentials available

**Additional Requirements:**
- A new unique email address for the new member account
- AWS CLI v2 configured with management account credentials
- Python 3.8+ (for automation scripts)
- Terraform v1.5+ (optional, for AFT section)

**IAM Permissions:**
- `AdministratorAccess` on the management account
- Service Catalog: `servicecatalog:ProvisionProduct`
- Organizations: `organizations:*`

**Estimated Time:** 45–60 minutes  
**Estimated Cost:** < $3 USD

---

### Steps

#### Step 1: Provision a Member Account via Account Factory

**Console:**
1. Navigate to **Control Tower** → **Account Factory**
2. Click **"Create account"**
3. Fill in the form:
   - **Account email:** `aws-workload-dev-lab@yourdomain.com`
   - **Display name:** `Workload-Dev`
   - **IAM Identity Center user email:** Your admin email
   - **IAM Identity Center user first name:** `Lab`
   - **IAM Identity Center user last name:** `Admin`
   - **Organizational Unit:** `Workloads`
4. Under **Account Factory parameters:**
   - **Managed account email:** Same as account email
   - **Account name:** `Workload-Dev`
5. Click **"Create account"**
6. ⏳ Wait 15–20 minutes for provisioning to complete

**CLI (via Service Catalog):**
```bash
# Get the Account Factory product ID
PRODUCT_ID=$(aws servicecatalog search-products \
  --filters Key=FullTextSearch,Value="AWS Control Tower Account Factory" \
  --query 'ProductViewSummaries[0].ProductId' \
  --output text)

echo "Product ID: $PRODUCT_ID"

# Get the provis