# Organizations — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using AWS CLI with AWS Organizations, ensure the following:

1. **AWS CLI installed and configured** (v2 recommended):
   ```bash
   aws --version
   # aws-cli/2.x.x Python/3.x.x ...
   ```

2. **Configure CLI with management account credentials:**
   ```bash
   aws configure
   # AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
   # AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   # Default region name: us-east-1
   # Default output format: json
   ```

3. **Required IAM Permissions** — Attach one or more of the following managed policies to your IAM user/role:

   | Policy | Use Case |
   |--------|----------|
   | `AWSOrganizationsFullAccess` | Full management access |
   | `AWSOrganizationsReadOnlyAccess` | Read-only access |
   | `AdministratorAccess` | Full AWS access (management account only) |

4. **Key IAM actions required:**
   ```
   organizations:CreateOrganization
   organizations:DescribeOrganization
   organizations:ListAccounts
   organizations:CreateAccount
   organizations:InviteAccountToOrganization
   organizations:MoveAccount
   organizations:CreateOrganizationalUnit
   organizations:AttachPolicy
   organizations:CreatePolicy
   organizations:EnablePolicyType
   ```

5. **Important notes:**
   - Most write operations must be performed from the **management (master) account**
   - Some operations require **all features** to be enabled in the organization
   - Service Control Policies (SCPs) require `FEATURE_SET: ALL`

---

## Core Commands

### 1. Create an Organization

```bash
aws organizations create-organization \
  --feature-set ALL
```

**What it does:** Creates a new AWS Organization from the current account, which becomes the management account. `ALL` enables all features including SCPs; use `CONSOLIDATED_BILLING` for billing only.

**Example output:**
```json
{
  "Organization": {
    "Id": "o-exampleorgid11",
    "Arn": "arn:aws:organizations::123456789012:organization/o-exampleorgid11",
    "FeatureSet": "ALL",
    "MasterAccountArn": "arn:aws:organizations::123456789012:account/o-exampleorgid11/123456789012",
    "MasterAccountId": "123456789012",
    "MasterAccountEmail": "admin@example.com",
    "AvailablePolicyTypes": [
      {
        "Type": "SERVICE_CONTROL_POLICY",
        "Status": "ENABLED"
      }
    ]
  }
}
```

---

### 2. Describe the Organization

```bash
aws organizations describe-organization
```

**What it does:** Returns metadata about the current organization, including its ID, ARN, feature set, and management account details.

**Example output:**
```json
{
  "Organization": {
    "Id": "o-exampleorgid11",
    "Arn": "arn:aws:organizations::123456789012:organization/o-exampleorgid11",
    "FeatureSet": "ALL",
    "MasterAccountId": "123456789012",
    "MasterAccountEmail": "admin@example.com"
  }
}
```

---

### 3. Create a Member Account

```bash
aws organizations create-account \
  --email dev-team@example.com \
  --account-name "Dev Team Account" \
  --iam-user-access-to-billing ALLOW \
  --role-name OrganizationAccountAccessRole
```

**What it does:** Creates a new AWS account and automatically adds it to the organization. The `--role-name` specifies the IAM role that the management account can assume in the new account.

**Example output:**
```json
{
  "CreateAccountStatus": {
    "Id": "car-examplecreateaccountrequestid",
    "AccountName": "Dev Team Account",
    "State": "IN_PROGRESS",
    "RequestedTimestamp": "2024-01-15T10:30:00.000Z",
    "CompletedTimestamp": null,
    "AccountId": null
  }
}
```

---

### 4. Check Account Creation Status

```bash
aws organizations describe-create-account-status \
  --create-account-request-id car-examplecreateaccountrequestid
```

**What it does:** Polls the status of an account creation request. State will be `IN_PROGRESS`, `SUCCEEDED`, or `FAILED`.

**Example output:**
```json
{
  "CreateAccountStatus": {
    "Id": "car-examplecreateaccountrequestid",
    "AccountName": "Dev Team Account",
    "State": "SUCCEEDED",
    "RequestedTimestamp": "2024-01-15T10:30:00.000Z",
    "CompletedTimestamp": "2024-01-15T10:35:42.000Z",
    "AccountId": "987654321098"
  }
}
```

---

### 5. List All Accounts in the Organization

```bash
aws organizations list-accounts \
  --output table
```

**What it does:** Returns a list of all AWS accounts in the organization with their IDs, names, emails, status, and join timestamps.

**Example output:**
```
------------------------------------------------------------------------------------------
|                                      ListAccounts                                      |
+----------------+--------------------+----------------------------+----------+----------+
|   AccountId    |       Email        |           Name             |  Status  |JoinedMethod|
+----------------+--------------------+----------------------------+----------+----------+
|  123456789012  | admin@example.com  | Management Account         | ACTIVE   | INVITED  |
|  987654321098  | dev@example.com    | Dev Team Account           | ACTIVE   | CREATED  |
|  111222333444  | prod@example.com   | Production Account         | ACTIVE   | CREATED  |
+----------------+--------------------+----------------------------+----------+----------+
```

---

### 6. Create an Organizational Unit (OU)

```bash
aws organizations create-organizational-unit \
  --parent-id r-examplerootid \
  --name "Production"
```

**What it does:** Creates a new Organizational Unit (OU) under a specified parent (root or another OU). OUs allow you to group accounts and apply policies hierarchically.

**Example output:**
```json
{
  "OrganizationalUnit": {
    "Id": "ou-examplerootid-exampleouid",
    "Arn": "arn:aws:organizations::123456789012:ou/o-exampleorgid11/ou-examplerootid-exampleouid",
    "Name": "Production"
  }
}
```

---

### 7. List Roots

```bash
aws organizations list-roots
```

**What it does:** Returns the root of the organization's hierarchy. Every organization has exactly one root, which is the starting point for the OU tree.

**Example output:**
```json
{
  "Roots": [
    {
      "Id": "r-examplerootid",
      "Arn": "arn:aws:organizations::123456789012:root/o-exampleorgid11/r-examplerootid",
      "Name": "Root",
      "PolicyTypes": [
        {
          "Type": "SERVICE_CONTROL_POLICY",
          "Status": "ENABLED"
        }
      ]
    }
  ]
}
```

---

### 8. Move an Account to an OU

```bash
aws organizations move-account \
  --account-id 987654321098 \
  --source-parent-id r-examplerootid \
  --destination-parent-id ou-examplerootid-exampleouid
```

**What it does:** Moves an account from one parent (root or OU) to another. This is how you organize accounts into OUs after creation.

---

### 9. Create a Service Control Policy (SCP)

```bash
aws organizations create-policy \
  --name "DenyRootUserActions" \
  --description "Prevents root user actions in member accounts" \
  --type SERVICE_CONTROL_POLICY \
  --content file://deny-root-actions.json
```

**`deny-root-actions.json`:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUser",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

**What it does:** Creates an SCP document that can be attached to OUs or accounts to restrict permissions. SCPs act as guardrails and override IAM policies.

**Example output:**
```json
{
  "Policy": {
    "PolicySummary": {
      "Id": "p-examplepolicyid",
      "Arn": "arn:aws:organizations::123456789012:policy/o-exampleorgid11/service_control_policy/p-examplepolicyid",
      "Name": "DenyRootUserActions",
      "Description": "Prevents root user actions in member accounts",
      "Type": "SERVICE_CONTROL_POLICY",
      "AwsManaged": false
    },
    "Content": "{...}"
  }
}
```

---

### 10. Attach a Policy to an OU or Account

```bash
# Attach to an OU
aws organizations attach-policy \
  --policy-id p-examplepolicyid \
  --target-id ou-examplerootid-exampleouid

# Attach to a specific account
aws organizations attach-policy \
  --policy-id p-examplepolicyid \
  --target-id 987654321098
```

**What it does:** Attaches an SCP (or other policy type) to a target OU or account. Policies are inherited by all children of the target.

---

### 11. List Policies

```bash
aws organizations list-policies \
  --filter SERVICE_CONTROL_POLICY
```

**What it does:** Lists all policies of the specified type in the organization. Filter options: `SERVICE_CONTROL_POLICY`, `TAG_POLICY`, `BACKUP_POLICY`, `AISERVICES_OPT_OUT_POLICY`.

**Example output:**
```json
{
  "Policies": [
    {
      "Id": "p-FullAWSAccess",
      "Arn": "arn:aws:organizations::aws:policy/service_control_policy/p-FullAWSAccess",
      "Name": "FullAWSAccess",
      "Description": "Allows access to every operation",
      "Type": "SERVICE_CONTROL_POLICY",
      "AwsManaged": true
    },
    {
      "Id": "p-examplepolicyid",
      "Name": "DenyRootUserActions",
      "Type": "SERVICE_CONTROL_POLICY",
      "AwsManaged": false
    }
  ]
}
```

---

### 12. Enable a Policy Type on Root

```bash
aws organizations enable-policy-type \
  --root-id r-examplerootid \
  --policy-type TAG_POLICY
```

**What it does:** Enables a policy type (e.g., Tag Policies, Backup Policies) on the organization root before you can create and attach policies of that type.

---

### 13. Invite an Existing Account to the Organization

```bash
aws organizations invite-account-to-organization \
  --target '{"Type": "ACCOUNT", "Id": "555666777888"}' \
  --notes "Please join our organization for centralized billing and governance."
```

**What it does:** Sends an invitation to an existing AWS account to join the organization. The account owner must accept the invitation.

---

### 14. List Children of an OU

```bash
# List child OUs
aws organizations list-children \
  --parent-id ou-examplerootid-exampleouid \
  --child-type ORGANIZATIONAL_UNIT

# List child accounts
aws organizations list-children \
  --parent-id ou-examplerootid-exampleouid \
  --child-type ACCOUNT
```

**What it does:** Lists direct children of a specified parent (root or OU), filtered by type (accounts or OUs).

---

### 15. Describe an Account

```bash
aws organizations describe-account \
  --account-id 987654321098
```

**What it does:** Returns detailed metadata for a specific account in the organization.

**Example output:**
```json
{
  "Account": {
    "Id": "987654321098",
    "Arn": "arn:aws:organizations::123456789012:account/o-exampleorgid11/987654321098",
    "Email": "dev-team@example.com",
    "Name": "Dev Team Account",
    "Status": "ACTIVE",
    "JoinedMethod": "CREATED",
    "JoinedTimestamp": "2024-01-15T10:35:42.000Z"
  }
}
```

---

## Common Operations

### Create Operations

```bash
# Create the organization
aws organizations create-organization --feature-set ALL

# Create an OU under root
aws organizations create-organizational-unit \
  --parent-id r-examplerootid \
  --name "Sandbox"

# Create a nested OU
aws organizations create-organizational-unit \
  --parent-id ou-examplerootid-exampleouid \
  --name "Team-Alpha"

# Create a new member account
aws organizations create-account \
  --email sandbox-01@example.com \
  --account-name "Sandbox-01" \
  --iam-user-access-to-billing ALLOW

# Create a Tag Policy
aws organizations create-policy \
  --name "RequiredTags" \
  --description "Enforces required resource tags" \
  --type TAG_POLICY \
  --content file://tag-policy.json

# Create a Backup Policy
aws organizations create-policy \
  --name "DailyBackupPolicy" \
  --description "Enforces daily backups on all accounts" \
  --type BACKUP_POLICY \
  --content file://backup-policy.json
```

---

### Read / Describe Operations

```bash
# Describe the organization
aws organizations describe-organization

# Describe a specific account
aws organizations describe-account --account-id 987654321098

# Describe a specific OU
aws organizations describe-organizational-unit \
  --organizational-unit-id ou-examplerootid-exampleouid

# Describe a specific policy
aws organizations describe-policy --policy-id p-examplepolicyid

# Describe a handshake (invitation)
aws organizations describe-handshake \
  --handshake-id h-examplehandshakeid
```

---

### List Operations

```bash
# List all accounts
aws organizations list-accounts

# List all OUs under a parent
aws organizations list-organizational-units-for-parent \
  --parent-id r-examplerootid

# List all policies by type
aws organizations list-policies --filter SERVICE_CONTROL_POLICY

# List policies attached to a target
aws organizations list-policies-for-target \
  --target-id ou-examplerootid-exampleouid \
  --filter SERVICE_CONTROL_POLICY

# List all targets a policy is attached to
aws organizations list-targets-for-policy \
  --policy-id p-examplepolicyid

# List parents of an account or OU
aws organizations list-parents \
  --child-id 987654321098

# List all delegated administrators
aws organizations list-delegated-administrators

# List all AWS services with delegated admin
aws organizations list-delegated-services-for-account \
  --account-id 987654321098

# List all handshakes (invitations) for the organization
aws organizations list-handshakes-for-organization

# List tags on a resource
aws organizations list-tags-for-resource \
  --resource-id ou-examplerootid-exampleouid
```

---