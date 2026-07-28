# Organizations

## What is it?

**AWS Organizations** is a global account management service that enables you to centrally govern and manage multiple AWS accounts from a single location. It is categorized under **Management & Governance** in the AWS service portfolio.

AWS Organizations allows you to consolidate billing, apply governance policies across accounts, automate account creation, and enforce security guardrails at scale. It is the foundational service for building a **multi-account AWS strategy** and is tightly integrated with AWS Control Tower, AWS SSO (IAM Identity Center), AWS Config, and many other services.

Key identifiers:
- **Service Name:** AWS Organizations
- **Category:** Management & Governance
- **Scope:** Global (not region-specific)
- **Root entity:** Management Account (formerly Master Account)
- **Policy types supported:** SCPs, Tag Policies, Backup Policies, AI Services Opt-Out Policies, Chatbot Policies

---

## Why do we need it?

### The Problem Without Organizations

As organizations grow, they naturally accumulate multiple AWS accounts for different teams, environments, or projects. Without a centralized management layer:

- **Billing chaos**: Each account has a separate bill; no consolidated view of spend.
- **Security drift**: Each account may have different IAM policies, leaving security gaps.
- **No guardrails**: Developers in one account can spin up any resource without restriction.
- **Manual overhead**: Creating and configuring each account individually is time-consuming and error-prone.
- **No shared services**: Sharing resources like VPCs, Route 53, or S3 across accounts is complex.

### Business Scenarios

| Scenario | How Organizations Helps |
|---|---|
| A startup scaling from 1 to 20 accounts | Automates account provisioning and applies consistent policies |
| Enterprise with Dev/Staging/Prod isolation | Enforces environment-specific SCPs to prevent accidental prod changes |
| Financial services firm needing compliance | Applies Tag Policies and Backup Policies uniformly across all accounts |
| SaaS company with per-customer isolation | Creates separate accounts per customer for blast-radius reduction |
| Mergers & Acquisitions | Onboards acquired company's accounts under existing Organization |
| Cost optimization team | Enables consolidated billing with volume discounts and Reserved Instance sharing |

### Core Use Cases
1. **Centralized billing and cost management** — single invoice, RI/Savings Plan sharing
2. **Policy-based governance** — SCPs to restrict or allow specific AWS services
3. **Automated account vending** — programmatic account creation via APIs or Control Tower
4. **Security baseline enforcement** — ensure CloudTrail, Config, GuardDuty are enabled everywhere
5. **Resource sharing** — AWS RAM integration for sharing VPCs, Transit Gateways, etc.

---

## Internal Working

### Organization Hierarchy

AWS Organizations uses a **tree-based hierarchical structure**:

```
Root
├── Management Account (Payer Account)
├── Organizational Unit (OU) - Production
│   ├── Account A (prod-app)
│   ├── Account B (prod-db)
│   └── Organizational Unit (OU) - PCI
│       └── Account C (prod-pci)
├── Organizational Unit (OU) - Development
│   ├── Account D (dev-app)
│   └── Account E (dev-db)
└── Organizational Unit (OU) - Shared Services
    ├── Account F (network-hub)
    └── Account G (security-tooling)
```

### How Policies Flow

1. **Policy Attachment**: Policies (e.g., SCPs) are attached to the Root, OUs, or individual accounts.
2. **Inheritance**: Policies flow **downward** through the hierarchy. A policy attached to an OU applies to all accounts and nested OUs beneath it.
3. **Effective Policy**: The effective policy for any account is the **intersection** of all policies inherited from parent nodes.
4. **Implicit Deny vs. Allow**: SCPs act as a **permission boundary** — they do not grant permissions but restrict what IAM policies can allow.

### Service Control Policy (SCP) Evaluation Logic

```
Effective Permissions = 
  (IAM Identity Policies ∩ Resource Policies) 
  FILTERED BY 
  (All SCPs in the hierarchy chain)
```

- If an SCP at the Root **denies** an action, no account in the organization can perform it, regardless of IAM policies.
- The Management Account is **exempt from SCPs** — it always has full access.
- SCPs affect **all principals** (IAM users, roles, including root user of member accounts).

### Account Creation Flow

1. You call `CreateAccount` API on the Organizations service endpoint.
2. AWS provisions a new AWS account with a unique Account ID.
3. The account is placed in the Root by default (or specified OU).
4. A cross-account IAM role (`OrganizationAccountAccessRole` by default) is created in the new account, allowing the Management Account to assume it.
5. The account is automatically enrolled in consolidated billing.

### Trust Relationship for Cross-Account Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::MANAGEMENT_ACCOUNT_ID:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## Architecture

### Core Architectural Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Organizations                         │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Management Account                     │   │
│  │  • Payer for all accounts                                 │   │
│  │  • Full API access to Organizations                       │   │
│  │  • Not affected by SCPs                                   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                           [Root]                                 │
│                              │                                   │
│              ┌───────────────┼───────────────┐                  │
│              │               │               │                  │
│           [OU: Prod]    [OU: Dev]    [OU: Shared Svcs]          │
│              │               │               │                  │
│         [Accounts]      [Accounts]      [Accounts]              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Policy Types                           │   │
│  │  • Service Control Policies (SCPs)                       │   │
│  │  • Tag Policies                                           │   │
│  │  • Backup Policies                                        │   │
│  │  • AI Services Opt-Out Policies                          │   │
│  │  • Chatbot Policies                                       │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-Account Architecture Patterns

#### Pattern 1: Environment-Based OU Structure
```
Root
├── Infrastructure OU
│   └── Network Account (Transit Gateway, DNS)
├── Security OU
│   ├── Log Archive Account
│   └── Security Tooling Account (GuardDuty Delegated Admin)
├── Workloads OU
│   ├── Production OU
│   │   └── App Accounts (prod-*)
│   └── Non-Production OU
│       └── App Accounts (dev-*, staging-*)
└── Sandbox OU
    └── Individual Developer Accounts
```

#### Pattern 2: Business Unit-Based Structure
```
Root
├── Corporate OU
│   ├── Finance OU
│   ├── HR OU
│   └── IT OU
├── Product Line A OU
│   ├── Prod Account
│   └── Dev Account
└── Product Line B OU
    ├── Prod Account
    └── Dev Account
```

### SCP Architecture — Deny List vs. Allow List

**Deny List Strategy (Default — Recommended)**
- Start with `FullAWSAccess` SCP at Root
- Add specific Deny SCPs at OU/Account level
- Easier to manage, less restrictive by default

**Allow List Strategy**
- Remove `FullAWSAccess` SCP
- Add explicit Allow SCPs for only approved services
- More restrictive, requires careful management

---

## Real World Example

### Scenario: E-Commerce Company Multi-Account Setup

**Company:** RetailCo — an e-commerce platform with 200 engineers, multiple product teams, and strict PCI-DSS compliance requirements.

#### Step 1: Create the Organization

```bash
# Create the organization with all features enabled
aws organizations create-organization --feature-set ALL
```

#### Step 2: Design the OU Structure

```
Root
├── Security OU (SCPs: Deny disable CloudTrail, Deny leave org)
│   ├── Log Archive Account (centralized S3 logs)
│   └── Security Tooling Account (GuardDuty, SecurityHub admin)
├── Infrastructure OU
│   └── Network Account (Transit Gateway, Route 53 Resolver)
├── Workloads OU
│   ├── PCI OU (Additional SCPs: restrict to us-east-1 only)
│   │   ├── pci-prod account
│   │   └── pci-staging account
│   ├── Production OU (SCP: deny delete on critical resources)
│   │   ├── storefront-prod
│   │   ├── payments-prod
│   │   └── inventory-prod
│   └── Non-Production OU (SCP: limit expensive instance types)
│       ├── storefront-dev
│       └── storefront-staging
└── Sandbox OU (SCP: deny resources outside us-east-1, budget limit)
    └── developer-sandbox-* accounts
```

#### Step 3: Create Organizational Units

```bash
# Get Root ID
ROOT_ID=$(aws organizations list-roots --query 'Roots[0].Id' --output text)

# Create Security OU
aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Security"

# Create Workloads OU
aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Workloads"
```

#### Step 4: Create and Attach a Region-Restriction SCP for PCI OU

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonUSEast1",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": "us-east-1"
        },
        "StringNotLike": {
          "aws:PrincipalARN": [
            "arn:aws:iam::*:role/PlatformAdmin"
          ]
        }
      }
    }
  ]
}
```

#### Step 5: Create Member Accounts Programmatically

```bash
# Create production account for storefront
aws organizations create-account \
  --email "aws+storefront-prod@retailco.com" \
  --account-name "storefront-prod" \
  --iam-user-access-to-billing ALLOW \
  --role-name OrganizationAccountAccessRole
```

#### Step 6: Enable Trusted Access for AWS Services

```bash
# Enable CloudTrail organization-wide
aws organizations enable-aws-service-access \
  --service-principal cloudtrail.amazonaws.com

# Enable Config organization-wide
aws organizations enable-aws-service-access \
  --service-principal config.amazonaws.com

# Enable GuardDuty organization-wide
aws organizations enable-aws-service-access \
  --service-principal guardduty.amazonaws.com
```

#### Step 7: Set Up Consolidated Billing Benefits

- Reserved Instances purchased in the Management Account automatically shared across all member accounts.
- Savings Plans applied organization-wide.
- Single consolidated invoice at month end.

#### Outcome

- RetailCo reduced AWS bill by 23% through RI sharing and volume discounts.
- Security team enforces 14 SCPs that prevent common misconfigurations.
- New developer accounts are provisioned in under 5 minutes via automation.
- PCI auditors have a clear, documented account boundary for compliance scope.

---

## Advantages

### 1. Centralized Governance at Scale
- Single pane of glass to manage hundreds of accounts
- Policies applied once propagate automatically to all child OUs/accounts

### 2. Consolidated Billing
- Single invoice for all accounts
- Volume pricing discounts apply across the entire organization
- Reserved Instances and Savings Plans shared across accounts
- Cost allocation tags work across all accounts

### 3. Security Guardrails via SCPs
- Prevent even the account root user from performing restricted actions
- Enforce compliance requirements (e.g., deny non-compliant regions)
- Cannot be overridden by any IAM policy in member accounts

### 4. Blast Radius Reduction
- Isolate workloads in separate accounts
- A security incident in one account doesn't automatically compromise others
- IAM permission boundaries are naturally enforced at account level

### 5. Automated Account Provisioning
- `CreateAccount` API enables account vending machines
- Integrates with AWS Control Tower for fully automated, compliant account creation
- Consistent baseline configuration via CloudFormation StackSets

### 6. AWS Service Integration
- Delegated administrator model allows security/compliance teams to manage services without Management Account access
- Organization-wide CloudTrail, Config, GuardDuty, SecurityHub from a single account

### 7. No Additional Cost
- AWS Organizations itself is **free** — no charge for using the service

### 8. Tag Policy Enforcement
- Define standardized tag keys and values
- Prevent tag drift across the organization
- Enables accurate cost allocation and resource tracking

---

## Limitations

### Hard Limits (Default Quotas)

| Limit | Default Value | Can Increase? |
|---|---|---|
| Maximum accounts in an organization | 10 (new org) → request increase | Yes (up to thousands) |
| Maximum OUs per organization | 1,000 | No |
| Maximum nesting levels for OUs | 5 | No |
| Maximum SCPs per organization | 1,000 | No |
| Maximum SCPs attached per entity | 5 | No |
| Maximum size of an SCP | 5,120 characters | No |
| Maximum Tag Policies per org | 1,000 | No |
| Maximum Backup Policies per org | 1,000 | No |
| Maximum delegated administrators | 3 per service | No |
| Maximum invitations per 24 hours | 20 | No |

### Functional Limitations

1. **Management Account SCPs**: The Management Account is **never** subject to SCPs — this is a known design constraint and security consideration.

2. **Account Removal**: Removing an account from an organization requires the account to have standalone billing information (credit card). Accounts created via Organizations may not have this.

3. **Account Deletion**: You cannot delete an AWS account directly; it must be closed through the Billing console, and there is a 90-day waiting period.

4. **SCP Character Limit**: The 5,120-character limit per SCP can be restrictive for complex policies. You may need to split policies across multiple SCPs.

5. **No Retroactive Policy Application**: Attaching an SCP doesn't affect resources already created — it only restricts future API calls.

6. **Feature Set Upgrade is One-Way**: Upgrading from "Consolidated Billing only" to "All Features" requires all member accounts to accept an invitation and **cannot be reversed**.

7. **Region Availability**: While Organizations is a global service, some integrated services (e.g., CloudFormation StackSets) have region-specific considerations.

8. **Delegated Admin Limits**: Only 3 delegated administrators per AWS service.

9. **No Cross-Organization Trust**: Organizations from different AWS Organization IDs cannot be merged natively; accounts must be migrated individually.

---

## Best Practices

### 1. Use a Dedicated Management Account
> **Well-Architected Pillar: Security**

- The Management Account should be used **exclusively** for Organizations management.
- Do not deploy workloads in the Management Account.
- Enable MFA on the Management Account root user and lock away the credentials.
- Limit access to the Management Account to a