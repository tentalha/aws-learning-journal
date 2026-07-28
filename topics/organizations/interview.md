# Organizations — Interview Questions

## Easy

---

**Q1. What is AWS Organizations, and what is its primary purpose?**

**Answer:**
AWS Organizations is a global AWS service that enables centralized management of multiple AWS accounts. Its primary purposes are:
- **Consolidated Billing**: Aggregate charges across all member accounts into a single payer account, enabling volume discounts and simplified cost tracking.
- **Policy-Based Governance**: Apply Service Control Policies (SCPs) to enforce permission guardrails across accounts.
- **Account Management**: Programmatically create, invite, and organize AWS accounts into a hierarchical structure.

The service is free to use — you pay only for the AWS resources consumed within each member account.

---

**Q2. What is the difference between a Management Account (formerly Master Account) and a Member Account in AWS Organizations?**

**Answer:**

| Feature | Management Account | Member Account |
|---|---|---|
| Role | Owns the Organization; pays all bills | Belongs to the Organization |
| SCPs applied to it | SCPs do NOT restrict the management account | SCPs fully apply |
| Can invite accounts | Yes | No |
| Can leave the org | Cannot leave (it IS the org) | Can leave (with conditions) |
| Billing | Receives consolidated bill | Costs rolled up to management account |

The management account has implicit full access to all AWS services and cannot be restricted by SCPs — this is a critical security consideration.

---

**Q3. What is an Organizational Unit (OU), and why would you use one?**

**Answer:**
An Organizational Unit (OU) is a logical grouping of AWS accounts within an AWS Organization. OUs are nested within the Root or within other OUs, forming a hierarchical tree structure.

**Why use OUs:**
- Apply SCPs to a group of accounts with a single policy attachment rather than attaching to each account individually.
- Mirror your organizational structure (e.g., Business Units, Environments, Teams).
- Delegate administration — AWS accounts in the same OU inherit the same policy guardrails.

**Example hierarchy:**
```
Root
├── Security OU
│   └── Security Tooling Account
├── Infrastructure OU
│   ├── Networking Account
│   └── Shared Services Account
├── Workloads OU
│   ├── Production OU
│   │   └── App-Prod Account
│   └── Development OU
│       └── App-Dev Account
└── Sandbox OU
    └── Developer Sandbox Accounts
```

---

**Q4. What is a Service Control Policy (SCP), and how does it differ from an IAM policy?**

**Answer:**
A **Service Control Policy (SCP)** is an Organizations policy type that sets the maximum permissions (permission boundaries) for accounts within the Organization.

**Key differences:**

| Aspect | SCP | IAM Policy |
|---|---|---|
| Scope | Applies to entire AWS accounts | Applies to IAM principals (users, roles, groups) |
| Who it restricts | All principals in the account, including root user | Specific principals only |
| Grants permissions | No — it only restricts | Yes — it grants permissions |
| Inheritance | Inherited down the OU hierarchy | Not hierarchical |
| Effective permissions | Intersection of SCP + IAM policy | IAM policy alone (within SCP boundary) |

**Critical concept:** An action is only allowed if it is permitted by **both** the SCP and the IAM policy. SCPs do not grant permissions; they define the ceiling.

---

**Q5. What are the two feature sets available in AWS Organizations, and what is the key difference?**

**Answer:**

| Feature Set | Capabilities |
|---|---|
| **Consolidated Billing Only** | Consolidated billing, cost aggregation, volume discounts. No SCPs, no advanced policy features. |
| **All Features** | Everything in Consolidated Billing PLUS SCPs, tag policies, backup policies, AI services opt-out policies, delegated administrators, and trusted access. |

**Key points:**
- You can upgrade from Consolidated Billing to All Features, but **you cannot downgrade**.
- Upgrading requires acceptance from all member accounts.
- AWS recommends always enabling All Features to take full advantage of governance capabilities.
- Most modern Organizations use All Features mode.

---

## Medium

---

**Q6. Explain how SCP inheritance works in an AWS Organizations hierarchy. What happens when multiple SCPs are applied at different levels?**

**Answer:**
SCP inheritance follows an **implicit deny** model with hierarchical evaluation:

**Inheritance Rules:**
1. SCPs are evaluated from the **Root → OU → Account** level.
2. An account's effective SCP is the **intersection** of all SCPs applied at every level above and including the account.
3. If any level in the hierarchy **denies** an action, it is denied regardless of what lower-level SCPs say.
4. The default SCP `FullAWSAccess` is attached to the Root when you enable SCPs. Removing it from any level will deny all access at that level.

**Example:**
```
Root (SCP: Allow all AWS services)
└── Production OU (SCP: Deny S3 Delete)
    └── App-Prod Account (SCP: Allow EC2 only)
```

For a principal in `App-Prod Account`:
- Root allows everything → Production OU denies S3 Delete → Account allows EC2 only
- Effective permission: **EC2 actions only, and S3 Delete explicitly denied**

**Evaluation logic (per action):**
```
Is there an explicit Deny in any SCP in the hierarchy?
  YES → DENY (stop evaluation)
  NO → Is there an explicit Allow in ALL levels of the hierarchy?
         YES → Proceed to IAM evaluation
         NO → DENY
```

**Common pitfall:** Removing `FullAWSAccess` from the Root without adding explicit Allow SCPs will lock out all accounts. Always test SCP changes in a non-production OU first.

**SCP does NOT affect:**
- The management account
- Service-linked roles
- Actions performed by AWS services on your behalf (e.g., CloudFormation service role)

---

**Q7. What is Delegated Administration in AWS Organizations, and why is it important for security?**

**Answer:**
**Delegated Administration** allows you to designate a member account (not the management account) as an administrator for specific AWS services that integrate with Organizations.

**Why it matters for security:**
The management account has unrestricted access to the entire organization. AWS best practice is to use the management account **only** for organizational management tasks and avoid deploying workloads there. Delegated administration enables this by moving service-specific administration to a dedicated member account.

**How it works:**
```bash
# Enable trusted access for the service
aws organizations enable-aws-service-access \
  --service-principal guardduty.amazonaws.com

# Register a delegated administrator
aws organizations register-delegated-administrator \
  --account-id 123456789012 \
  --service-principal guardduty.amazonaws.com
```

**Services supporting delegated administration (common examples):**

| Service | Delegated Admin Use Case |
|---|---|
| AWS Security Hub | Centralized security findings management |
| Amazon GuardDuty | Centralized threat detection |
| AWS Config | Centralized compliance evaluation |
| AWS Firewall Manager | Centralized firewall policy management |
| Amazon Macie | Centralized data sensitivity discovery |
| AWS IAM Identity Center | SSO administration |
| AWS Backup | Centralized backup policies |

**Best practice:** Create a dedicated "Security Tooling" account and register it as the delegated administrator for security services. This limits blast radius if the management account is compromised.

---

**Q8. Describe the different policy types available in AWS Organizations and their use cases.**

**Answer:**
AWS Organizations supports five policy types (all require All Features mode except where noted):

**1. Service Control Policies (SCPs)**
- **Purpose:** Set maximum permission guardrails for accounts.
- **Use case:** Prevent member accounts from disabling CloudTrail, restrict workloads to specific AWS Regions, enforce encryption requirements.
- **Example:** Deny any action that disables GuardDuty.

**2. Tag Policies**
- **Purpose:** Enforce standardized tag keys and values across resources.
- **Use case:** Ensure all EC2 instances have `Environment`, `CostCenter`, and `Owner` tags with approved values.
- **Enforcement:** Tag policies can report non-compliance or prevent non-compliant tagging (enforcement mode).

**3. Backup Policies**
- **Purpose:** Automatically deploy AWS Backup plans to member accounts.
- **Use case:** Ensure all RDS instances in Production OU are backed up daily with 30-day retention, without requiring each account to configure this manually.
- **Result:** Creates a "governed backup plan" in each member account.

**4. AI Services Opt-Out Policies**
- **Purpose:** Opt accounts out of AWS using their content to improve AI/ML services.
- **Use case:** Compliance-sensitive organizations that cannot allow AWS to process customer data for model training (e.g., healthcare, finance).
- **Services covered:** Rekognition, Comprehend, Lex, Polly, Transcribe, etc.

**5. Chatbot Policies** *(newer addition)*
- **Purpose:** Control which AWS Chatbot configurations can be used in member accounts.
- **Use case:** Standardize notification channels and restrict unauthorized Chatbot integrations.

**Policy evaluation order:**
```
Backup/Tag/AI Policies → Applied as configurations (not permission-based)
SCPs → Permission boundary evaluation
IAM Policies → Actual permission grant
```

---

**Q9. How does consolidated billing work in AWS Organizations, and what are the pricing benefits?**

**Answer:**
Consolidated billing aggregates usage and costs from all member accounts into the management account's single monthly bill.

**How it works:**
1. Each member account accrues its own charges independently.
2. At billing cycle end, all charges are combined into one bill paid by the management account.
3. Individual account cost data is still available via Cost Explorer and Cost and Usage Reports (CUR).

**Pricing benefits:**

**1. Volume Discounts (Tiered Pricing)**
AWS services like S3, data transfer, and DynamoDB use tiered pricing. Usage from all accounts is **pooled** to reach higher discount tiers faster.

```
Example (S3 Standard):
- Account A uses 40 TB → billed at Tier 1 rate
- Account B uses 40 TB → billed at Tier 1 rate
- Combined in Org: 80 TB → 50 TB at Tier 1 + 30 TB at Tier 2 (cheaper)
```

**2. Reserved Instance (RI) and Savings Plans Sharing**
- RIs purchased in one account can be applied to matching usage in any other account in the organization.
- Savings Plans work the same way.
- You can disable RI sharing per account if isolation is required.

**3. Spot Instance Diversity**
- Spot Fleet requests can draw from capacity across accounts (less common benefit).

**Cost allocation:**
- Tags are used for cost allocation within consolidated billing.
- AWS Cost Explorer shows costs broken down by account, service, tag, and region.
- Cost and Usage Reports (CUR) provide the most granular billing data.

**Important:** Even with consolidated billing, each account's resource limits (service quotas) are independent. Billing consolidation does not share quotas.

---

**Q10. What is AWS Control Tower, and how does it relate to AWS Organizations?**

**Answer:**
**AWS Control Tower** is a higher-level service that automates the setup and governance of a multi-account AWS environment following AWS best practices. It is **built on top of** AWS Organizations.

**Relationship:**
```
AWS Control Tower
    ↓ uses and manages
AWS Organizations
    ↓ contains
AWS Accounts (organized into OUs)
    ↓ governed by
Guardrails (implemented as SCPs + AWS Config Rules)
```

**What Control Tower provides on top of Organizations:**

| Capability | Organizations Alone | With Control Tower |
|---|---|---|
| Account vending | Manual API calls | Automated via Account Factory |
| Guardrails | Manual SCP authoring | Pre-built preventive + detective guardrails |
| Baseline configuration | Manual | Automated (CloudTrail, Config, IAM Identity Center) |
| Landing zone | DIY | Pre-built, AWS-recommended structure |
| Drift detection | Not available | Built-in |

**Control Tower components:**
- **Landing Zone**: The overall multi-account environment setup.
- **Account Factory**: Self-service portal (or API) to provision new accounts with baseline configurations.
- **Guardrails**: 
  - *Preventive*: SCPs that block non-compliant actions.
  - *Detective*: AWS Config rules that detect and report non-compliance.
- **Log Archive Account**: Centralized, immutable log storage.
- **Audit Account**: Security tooling and cross-account access for auditors.

**When to use Control Tower vs. raw Organizations:**
- **Control Tower**: Greenfield environments, teams wanting AWS best practices out-of-the-box, faster time-to-value.
- **Raw Organizations**: Existing complex environments, organizations needing custom OU structures, teams with dedicated cloud platform engineering teams.

---

## Hard

---

**Q11. Explain the exact permission evaluation logic when a principal in a member account attempts an action. Walk through every layer of the evaluation chain.**

**Answer:**
AWS uses a multi-layer evaluation model. Here is the complete evaluation chain for a principal in a member account:

**Complete Evaluation Order:**

```
Step 1: Explicit Deny in any policy?
    → If YES: DENY (stop)

Step 2: Is it an Organizations Service Control Policy (SCP) evaluation?
    → Does an SCP at Root, any parent OU, or the account level ALLOW the action?
    → If NO Allow exists anywhere in the SCP chain: DENY (stop)
    → Note: The default FullAWSAccess SCP counts as an Allow

Step 3: Is there a Resource-Based Policy that allows the principal?
    → If YES, and the principal is in the same account: ALLOW (stop)
    → If YES, and the principal is cross-account: must also pass IAM identity policy check

Step 4: Is there a Permissions Boundary on the IAM principal?
    → If YES: the action must be allowed by the boundary
    → If boundary DENIES or does not ALLOW: DENY (stop)

Step 5: Is this a Session Policy (assumed role with policy)?
    → If YES: action must be allowed by the session policy
    → If session policy DENIES or does not ALLOW: DENY (stop)

Step 6: Identity-Based Policy (IAM policy attached to user/role)
    → Does the IAM policy explicitly ALLOW the action?
    → If YES: ALLOW
    → If NO: DENY (implicit deny)
```

**Visual representation:**
```
Request
  │
  ▼
[Explicit Deny in ANY policy] ──YES──► DENY
  │ NO
  ▼
[SCP allows action?] ──NO──► DENY
  │ YES
  ▼
[Resource-based policy allows (same account)?] ──YES──► ALLOW
  │ NO
  ▼
[Permissions Boundary allows?] ──NO──► DENY
  │ YES
  ▼
[Session Policy allows?] ──NO──► DENY
  │ YES
  ▼
[Identity-based policy allows?] ──NO──► DENY
  │ YES
  ▼
ALLOW
```

**Critical nuances for Organizations specifically:**

1. **Management account bypass**: SCPs never apply to the management account. Steps 1 and 2 above skip SCP evaluation entirely for the management account.

2. **Service-linked roles**: SCPs do NOT restrict service-linked roles. AWS services that use service-linked roles (e.g., AWS Config, GuardDuty) can always perform their required actions.

3. **Root user in member account**: SCPs DO apply to the root user of member accounts. This is a key security feature — you can use SCPs to prevent even the root user from disabling security services.

4. **Cross-account access**: When Account A's role is assumed by Account B's principal:
   - Account B's SCP must allow the action (for the calling principal)
   - Account A's SCP must allow the action (for the target resource/account)
   - Account A's resource policy or role trust policy must allow Account B
   - Account B's IAM policy must allow the assume-role action

---

**Q12. Design a comprehensive SCP strategy for a large enterprise with strict compliance requirements. Include deny-list vs. allow-list approaches and explain the trade-offs.**

**Answer:**

**Two fundamental SCP strategies:**

**Strategy 1: Deny-List (Default Allow)**
- Start with `FullAWSAccess` attached everywhere (default).
- Add explicit Deny statements for prohibited actions.
- **Philosophy:** Allow everything except what is explicitly forbidden.

```json
{
  "Version": "2012-10-17",