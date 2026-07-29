# Control Tower

## What is it?

**AWS Control Tower** is a fully managed service that provides a prescriptive, automated way to set up and govern a secure, multi-account AWS environment (called a **Landing Zone**) based on AWS best practices. It falls under the **Cloud Governance and Management** category.

Control Tower sits at the top of the AWS Organizations hierarchy and orchestrates multiple AWS services — including AWS Organizations, AWS Service Catalog, AWS Config, AWS CloudTrail, AWS IAM Identity Center (formerly SSO), and more — to establish a well-architected, enterprise-grade multi-account environment with centralized governance, compliance guardrails, and account vending automation.

**Key identifiers:**
- **Service Category:** Cloud Governance / Landing Zone Management
- **Deployment Scope:** Multi-account, multi-region
- **Underlying Mechanism:** AWS Organizations + Service Catalog + Config Rules + CloudTrail + IAM Identity Center
- **Management Model:** Managed service (AWS handles the orchestration infrastructure)

---

## Why do we need it?

### The Problem

Managing a single AWS account is straightforward. But enterprise organizations typically need **dozens to thousands of AWS accounts** to achieve:
- Workload isolation (production vs. staging vs. dev)
- Team autonomy (each team owns their account)
- Blast radius reduction (a security incident in one account doesn't affect others)
- Regulatory compliance (PCI, HIPAA, SOC 2 require isolation)
- Cost attribution (per-team or per-product billing)

Without Control Tower, setting up this multi-account structure manually involves:
- Manually creating AWS Organizations structures
- Writing hundreds of SCPs (Service Control Policies)
- Configuring CloudTrail and Config in every account
- Setting up centralized logging manually
- Building an account provisioning pipeline from scratch
- Onboarding new accounts consistently and repeatedly

This is error-prone, time-consuming, and leads to **configuration drift** across accounts.

### Business Scenarios Where Control Tower Is Essential

| Scenario | Why Control Tower |
|---|---|
| **Enterprise cloud adoption** | Standardize the foundation before teams start building |
| **Mergers & Acquisitions** | Quickly onboard acquired company accounts under governance |
| **Regulated industries** (Healthcare, Finance) | Enforce compliance guardrails automatically |
| **ISV / SaaS companies** | Provision isolated tenant environments at scale |
| **Government/Public Sector** | Meet FedRAMP, NIST compliance baselines |
| **Rapid team scaling** | New teams get a compliant account in minutes, not weeks |

### The "Before vs. After" Story

**Before Control Tower:** A new team requests an AWS account. The cloud team manually creates it, configures logging, applies SCPs, sets up IAM Identity Center, and hands it over — taking 2–3 weeks.

**After Control Tower:** A new team submits an Account Factory request. Control Tower automatically provisions a fully compliant, logged, governed account in 30–45 minutes.

---

## Internal Working

### How Control Tower Bootstraps a Landing Zone

When you set up a Control Tower Landing Zone, it performs the following orchestration steps automatically:

```
Step 1: Validate Prerequisites
  └─ Checks that AWS Organizations is enabled or creates a new org
  └─ Verifies IAM permissions for the management account

Step 2: Create Foundational OUs
  └─ Security OU (contains Log Archive + Audit accounts)
  └─ Sandbox OU (optional, for experimentation)

Step 3: Create Shared Accounts
  └─ Log Archive Account (centralized S3 logging)
  └─ Audit Account (read-only access for security teams)

Step 4: Enable Baseline Services
  └─ AWS CloudTrail (organization-level trail)
  └─ AWS Config (aggregated across all accounts)
  └─ IAM Identity Center (centralized identity)
  └─ Amazon S3 (log buckets with KMS encryption)

Step 5: Apply Mandatory Guardrails
  └─ SCPs applied to Security OU
  └─ Config Rules deployed to all accounts

Step 6: Deploy Account Factory
  └─ AWS Service Catalog product for account provisioning
```

### Guardrail Evaluation Engine

Guardrails work through two mechanisms:

1. **Preventive Guardrails** → Implemented as **AWS Organizations Service Control Policies (SCPs)**
   - Evaluated at the IAM authorization layer
   - Block actions before they happen
   - Example: "Disallow deletion of CloudTrail logs"

2. **Detective Guardrails** → Implemented as **AWS Config Rules**
   - Evaluated continuously after resources are created
   - Report compliance violations without blocking
   - Example: "Detect if S3 buckets are publicly accessible"

3. **Proactive Guardrails** (newer feature) → Implemented as **AWS CloudFormation Hooks**
   - Evaluate resources before they are provisioned via CloudFormation
   - Block non-compliant infrastructure-as-code deployments

### Account Vending Machine (Account Factory)

Account Factory uses **AWS Service Catalog** under the hood:

```
User Request (via Console / API / AFT)
         │
         ▼
AWS Service Catalog Product
         │
         ▼
CloudFormation StackSet
         │
         ├─► Create AWS Account (via Organizations API)
         ├─► Apply Account Baseline (SCPs, Config, CloudTrail)
         ├─► Configure IAM Identity Center permissions
         ├─► Apply VPC baseline (optional)
         └─► Register in Control Tower
```

### Drift Detection

Control Tower continuously monitors for **configuration drift** — when someone manually changes a governed resource outside of Control Tower:

- Runs periodic checks on guardrail compliance
- Detects if SCPs are modified, CloudTrail is disabled, etc.
- Reports drift in the Control Tower dashboard
- Allows re-registration to remediate drift

---

## Architecture

### High-Level Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    AWS Organizations                              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  Root                                    │    │
│  │                    │                                     │    │
│  │         ┌──────────┴──────────┐                         │    │
│  │         │                     │                         │    │
│  │   ┌─────▼──────┐    ┌────────▼────────┐                │    │
│  │   │ Security OU│    │  Custom OUs      │                │    │
│  │   │            │    │  (Workloads)     │                │    │
│  │   │ ┌────────┐ │    │                  │                │    │
│  │   │ │  Log   │ │    │ ┌──────────────┐ │                │    │
│  │   │ │Archive │ │    │ │   Dev OU     │ │                │    │
│  │   │ │Account │ │    │ │  ┌────────┐  │ │                │    │
│  │   │ └────────┘ │    │ │  │Team A  │  │ │                │    │
│  │   │            │    │ │  │Account │  │ │                │    │
│  │   │ ┌────────┐ │    │ │  └────────┘  │ │                │    │
│  │   │ │ Audit  │ │    │ └──────────────┘ │                │    │
│  │   │ │Account │ │    │                  │                │    │
│  │   │ └────────┘ │    │ ┌──────────────┐ │                │    │
│  │   └────────────┘    │ │   Prod OU    │ │                │    │
│  │                     │ │  ┌────────┐  │ │                │    │
│  │                     │ │  │Team A  │  │ │                │    │
│  │                     │ │  │Account │  │ │                │    │
│  │                     │ │  └────────┘  │ │                │    │
│  │                     │ └──────────────┘ │                │    │
│  └─────────────────────┴──────────────────┘                │    │
└──────────────────────────────────────────────────────────────────┘

         Control Tower Management Layer
┌──────────────────────────────────────────────────────────────────┐
│  Management Account                                              │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  Account   │  │  Guardrail   │  │   IAM Identity Center    │ │
│  │  Factory   │  │  Engine      │  │   (Centralized SSO)      │ │
│  │ (Svc Cat.) │  │ (SCPs+Config)│  │                          │ │
│  └────────────┘  └──────────────┘  └──────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### Key Architectural Components

#### 1. Management Account (formerly Master Account)
- Hosts Control Tower itself
- Contains Account Factory (Service Catalog product)
- Holds the IAM Identity Center configuration
- Should have **minimal workloads** — used only for governance

#### 2. Security OU
- **Mandatory** OU created by Control Tower
- Contains exactly two accounts:
  - **Log Archive Account**: Centralized S3 bucket receiving CloudTrail and Config logs from all accounts
  - **Audit Account**: Read-only cross-account access for security teams; receives SNS notifications for compliance violations

#### 3. Sandbox OU
- Optional OU for experimentation
- Fewer guardrails, allowing developers to explore freely

#### 4. Custom OUs and Accounts
- Workload OUs (Dev, Staging, Prod, etc.)
- Each enrolled account gets baseline guardrails applied automatically

#### 5. Guardrails (Controls)
Three categories:
- **Mandatory**: Always enabled, cannot be disabled (e.g., disallow changes to CloudTrail)
- **Strongly Recommended**: Pre-configured best practices (e.g., detect public S3 buckets)
- **Elective**: Optional, organization-specific controls

Three behaviors:
- **Preventive**: SCP-based, blocks actions
- **Detective**: Config Rule-based, detects violations
- **Proactive**: CloudFormation Hook-based, validates before deployment

#### 6. Account Factory for Terraform (AFT)
- Infrastructure-as-code wrapper around Account Factory
- Uses Terraform to define account vending pipelines
- Supports customizations via Terraform modules

---

## Real World Example

### Scenario: Financial Services Company Adopting AWS at Scale

**Company:** FinanceCorpX — a mid-sized financial services company with 50+ development teams, subject to SOC 2 and PCI DSS compliance requirements.

**Goal:** Migrate 200+ applications to AWS over 18 months, with each team owning their own AWS accounts.

### Step-by-Step Walkthrough

#### Phase 1: Initial Setup (Day 1)

```
1. Cloud Platform Team logs into the Management Account
2. Navigates to AWS Control Tower → "Set up landing zone"
3. Configures:
   - Home Region: us-east-1
   - Additional governed regions: us-west-2, eu-west-1
   - Log retention: 365 days (compliance requirement)
   - Enables CloudTrail for all regions
4. Control Tower automatically creates:
   - Security OU with Log Archive + Audit accounts
   - Sandbox OU for experimentation
   - Organization-level CloudTrail
   - Config aggregator
   - IAM Identity Center with default permission sets
```

#### Phase 2: Guardrail Configuration (Day 2-3)

```
Mandatory Guardrails (auto-enabled):
  ✓ Disallow deletion of log archive
  ✓ Disallow changes to CloudTrail
  ✓ Disallow changes to Config rules

Strongly Recommended Guardrails (enabled by cloud team):
  ✓ Detect whether public read access is enabled for S3 buckets
  ✓ Detect whether MFA is enabled for IAM users
  ✓ Detect whether EBS volumes are encrypted

Elective Guardrails (PCI-specific):
  ✓ Disallow internet connectivity through RDP
  ✓ Detect whether unrestricted internet connectivity exists through SSH
  ✓ Disallow creation of access keys for the root user
```

#### Phase 3: OU Structure Design (Day 3-5)

```
Root
├── Security OU (Control Tower managed)
│   ├── Log Archive Account
│   └── Audit Account
├── Infrastructure OU
│   ├── Networking Account (Transit Gateway, Direct Connect)
│   └── Shared Services Account (Active Directory, DNS)
├── Workloads OU
│   ├── Development OU
│   │   └── [Team accounts provisioned here]
│   ├── Staging OU
│   │   └── [Team accounts provisioned here]
│   └── Production OU
│       └── [Team accounts provisioned here]
└── Sandbox OU
    └── [Individual developer accounts]
```

#### Phase 4: Account Factory Customization (Week 2)

The cloud team creates an Account Factory customization using AFT:

```hcl
# aft-account-request/terraform/main.tf
module "payment_team_prod" {
  source = "./modules/aft-account-request"

  control_tower_parameters = {
    AccountEmail              = "aws-payment-prod@financecorpx.com"
    AccountName               = "PaymentTeam-Production"
    ManagedOrganizationalUnit = "Workloads/Production"
    SSOUserEmail              = "platform-admin@financecorpx.com"
    SSOUserFirstName          = "Platform"
    SSOUserLastName           = "Admin"
  }

  account_tags = {
    Team        = "PaymentTeam"
    Environment = "Production"
    CostCenter  = "CC-1042"
    Compliance  = "PCI-DSS"
  }

  account_customizations_name = "pci-production-baseline"
}
```

#### Phase 5: Team Onboarding (Ongoing)

```
Payment Team requests a new account:
1. Submits request via internal Service Portal (backed by Service Catalog)
2. Control Tower Account Factory:
   a. Creates new AWS account via Organizations API
   b. Applies baseline SCPs from parent OU
   c. Configures CloudTrail → logs to Log Archive Account
   d. Deploys Config rules → reports to Audit Account
   e. Creates IAM Identity Center assignment
   f. Applies VPC baseline (3 AZs, public/private subnets)
3. Account ready in ~35 minutes
4. Payment Team developer logs in via IAM Identity Center SSO portal
5. Compliance team sees account in Config aggregator dashboard
```

#### Phase 6: Compliance Reporting (Ongoing)

```
Security Team workflow:
1. Opens Audit Account
2. Reviews AWS Config Aggregator → sees compliance across all 200 accounts
3. Receives SNS notification: "S3 bucket in PaymentTeam-Dev is publicly accessible"
4. Reviews CloudTrail logs in Log Archive Account
5. Raises ticket with Payment Team to remediate
6. Config rule shows compliant after remediation
```

---

## Advantages

### 1. **Rapid Landing Zone Deployment**
- Full multi-account foundation deployed in hours, not months
- Eliminates weeks of manual setup and configuration

### 2. **Automated Compliance Enforcement**
- 300+ built-in guardrails covering security, operations, and cost
- Guardrails automatically apply to new accounts as they're enrolled
- Proactive guardrails prevent non-compliant IaC from deploying

### 3. **Consistent Account Baseline**
- Every new account gets the same security baseline
- Eliminates "snowflake accounts" with inconsistent configurations
- Account vending in ~30-45 minutes

### 4. **Centralized Visibility**
- Single dashboard showing compliance status across all accounts
- Aggregated CloudTrail and Config data
- Drift detection alerts when governance is bypassed

### 5. **Scalability**
- Supports