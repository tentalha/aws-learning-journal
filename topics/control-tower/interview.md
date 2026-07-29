# Control Tower — Interview Questions

## Easy

---

### 1. What is AWS Control Tower, and what problem does it solve?

**Answer:**
AWS Control Tower is a managed service that provides a pre-configured, secure, multi-account AWS environment based on AWS best practices. It automates the setup of a well-architected, multi-account AWS environment called a **Landing Zone**.

It solves the problem of manually setting up and governing multiple AWS accounts at scale. Without Control Tower, organizations must manually configure IAM policies, SCPs, CloudTrail, Config, and cross-account access — a process that is error-prone and time-consuming. Control Tower automates all of this and provides ongoing governance through **Guardrails (Controls)**.

---

### 2. What is a Landing Zone in the context of AWS Control Tower?

**Answer:**
A **Landing Zone** is a well-architected, multi-account AWS environment that Control Tower sets up automatically. It includes:

- A **Management Account** (root of the AWS Organization)
- A **Log Archive Account** — centralized S3 bucket for all CloudTrail and Config logs
- An **Audit Account** — used for security tooling and cross-account access for auditors
- A default **Organizational Unit (OU)** structure (Security OU, Sandbox OU, etc.)
- Preconfigured **Guardrails (Controls)** applied at the OU level
- **AWS SSO (IAM Identity Center)** for centralized identity management

The Landing Zone is the baseline environment from which new accounts are vended.

---

### 3. What are Guardrails (Controls) in AWS Control Tower?

**Answer:**
Guardrails are high-level rules that provide ongoing governance for your AWS environment. They are categorized in two ways:

**By behavior:**
- **Preventive Guardrails** — Use **Service Control Policies (SCPs)** to prevent non-compliant actions (e.g., "Disallow deletion of log archive").
- **Detective Guardrails** — Use **AWS Config Rules** to detect non-compliant configurations and report them (e.g., "Detect whether MFA is enabled for the root user").
- **Proactive Guardrails** — Use **AWS CloudFormation hooks** to check resources before they are provisioned.

**By guidance:**
- **Mandatory** — Always enforced (cannot be disabled).
- **Strongly Recommended** — Based on AWS best practices.
- **Elective** — Optional, for specific use cases.

---

### 4. What is Account Factory in AWS Control Tower?

**Answer:**
**Account Factory** is a feature within AWS Control Tower that automates the provisioning of new AWS accounts with pre-approved configurations. It is built on top of **AWS Service Catalog**.

Key capabilities:
- Provisions new accounts with baseline configurations (network settings, Guardrails, SSO access)
- Enrolls accounts into the correct OU automatically
- Supports customization via **Account Factory for Terraform (AFT)** or **Account Factory Customizations (AFC)**
- Self-service for end users through the Service Catalog portal
- Ensures every new account adheres to organizational standards from day one

---

### 5. What AWS services does Control Tower integrate with under the hood?

**Answer:**
AWS Control Tower orchestrates and integrates with several AWS services:

| Service | Role in Control Tower |
|---|---|
| **AWS Organizations** | Creates and manages the OU and account hierarchy |
| **AWS IAM Identity Center (SSO)** | Centralized user authentication and permission sets |
| **AWS Service Catalog** | Powers Account Factory for account vending |
| **AWS CloudTrail** | Centralized activity logging to the Log Archive account |
| **AWS Config** | Powers detective guardrails and compliance monitoring |
| **AWS CloudFormation** | Deploys baseline resources (StackSets) across accounts |
| **Amazon SNS** | Sends notifications for guardrail violations |
| **Amazon S3** | Stores centralized logs in the Log Archive account |

---

## Medium

---

### 1. How does AWS Control Tower use AWS CloudFormation StackSets, and what is their significance?

**Answer:**
AWS Control Tower uses **AWS CloudFormation StackSets** to deploy baseline infrastructure resources consistently across all enrolled accounts and OUs. When Control Tower sets up a Landing Zone or enrolls a new account, it automatically creates and manages StackSets that provision resources such as:

- CloudTrail configurations
- AWS Config recorders and delivery channels
- IAM roles required for cross-account access (e.g., `AWSControlTowerExecution` role)
- SNS topics for notifications
- Config aggregators

**Significance:**
- **Consistency** — Every account gets the same baseline configuration without manual intervention.
- **Drift detection** — If someone manually modifies resources deployed by Control Tower StackSets, it creates **drift**, which Control Tower can detect and report.
- **Automated remediation** — Re-registering an OU or running "Repair" can re-deploy StackSets to fix drift.

**Important caveat:** These StackSets are managed by Control Tower and should **not** be manually modified. Doing so can break Control Tower's ability to manage the account.

---

### 2. What is the difference between a Preventive and a Detective Guardrail? Give a real-world example of each.

**Answer:**

| Aspect | Preventive Guardrail | Detective Guardrail |
|---|---|---|
| **Mechanism** | Service Control Policy (SCP) | AWS Config Rule |
| **Timing** | Blocks action before it happens | Reports non-compliance after the fact |
| **Effect** | Hard stop — API call returns `AccessDenied` | Soft alert — marks resource as non-compliant |
| **Scope** | Applied at OU level via Organizations | Applied per account via Config |

**Real-world examples:**

- **Preventive:** *"Disallow changes to AWS CloudTrail configuration"* — An SCP is attached to the OU that explicitly denies `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail`, and related API calls. Even an account administrator cannot disable CloudTrail.

- **Detective:** *"Detect whether Amazon EBS volumes are attached to Amazon EC2 instances"* — An AWS Config rule checks whether EBS volumes exist in an unattached state (a cost and security concern). It marks them as non-compliant and reports to the Audit account dashboard, but does not block the creation.

**Key insight:** Preventive guardrails are stronger but less flexible. Detective guardrails allow teams to move fast while still maintaining visibility.

---

### 3. How does Control Tower handle account enrollment for existing AWS accounts?

**Answer:**
Enrolling an **existing account** (one not originally created by Control Tower) is more complex than vending a new account. The process involves:

**Prerequisites:**
1. The account must already be part of the same **AWS Organization** as the Control Tower management account.
2. The account must have an IAM role named **`AWSControlTowerExecution`** with a trust policy allowing the management account to assume it. This role must have `AdministratorAccess`.
3. The account must be moved into an OU that is **registered with Control Tower**.

**Enrollment steps:**
1. Ensure the `AWSControlTowerExecution` role exists in the target account.
2. Move the account to a Control Tower-managed OU in AWS Organizations.
3. In the Control Tower console, navigate to **Account Factory** → **Enroll Account**.
4. Provide account details (email, OU, SSO user).
5. Control Tower deploys StackSets to baseline the account.

**Common pitfalls:**
- Missing or incorrectly named IAM role
- Account in a non-registered OU
- Conflicting Config recorders (must delete existing Config recorder before enrollment)
- CloudTrail conflicts if a trail already exists with the same name

---

### 4. What is Control Tower Drift, and how do you detect and remediate it?

**Answer:**
**Drift** occurs when the actual state of your Control Tower environment deviates from the expected configuration. This can happen when:

- Someone manually modifies or deletes a StackSet resource (e.g., deletes a CloudTrail trail)
- An SCP is manually modified outside of Control Tower
- An account is moved to a non-registered OU
- The `AWSControlTowerExecution` role is deleted or modified
- A new account is created directly in AWS Organizations (bypassing Account Factory)

**Types of drift:**
- **Account drift** — Account configuration deviates from baseline
- **OU drift** — OU-level SCPs or StackSets have been modified
- **Guardrail drift** — A guardrail's underlying SCP or Config rule has been changed

**Detection:**
- Control Tower console shows a **"Drift detected"** status on the affected account or OU
- The **Landing Zone** page shows drift status
- Amazon SNS notifications can alert on drift events
- AWS CloudTrail logs show who made the change

**Remediation:**
- For **Landing Zone drift**: Go to Control Tower → Landing Zone → **Update** (re-runs setup)
- For **OU drift**: Re-register the OU via **Control Tower → OUs → Re-register OU**
- For **account drift**: Update the account via Account Factory
- For **guardrail drift**: Disable and re-enable the guardrail

---

### 5. What is Account Factory for Terraform (AFT), and how does it differ from the standard Account Factory?

**Answer:**
**Account Factory for Terraform (AFT)** is an open-source solution (maintained by AWS) that extends Control Tower's Account Factory to support **Terraform-based account vending**. It is hosted in a dedicated **AFT Management Account**.

**Architecture of AFT:**
```
Git Repository (Account Request)
        ↓
AWS CodePipeline / CodeBuild
        ↓
AFT Management Account (Terraform)
        ↓
Control Tower Account Factory (API)
        ↓
New/Enrolled AWS Account
        ↓
Post-provisioning customizations (Terraform)
```

**Key differences:**

| Feature | Standard Account Factory | Account Factory for Terraform (AFT) |
|---|---|---|
| **Interface** | AWS Service Catalog (console/API) | Terraform + Git workflow |
| **Customization** | Limited (Account Customizations) | Full Terraform customization |
| **Pipeline** | None | CodePipeline + CodeBuild |
| **Version control** | No native Git integration | Git-native (GitOps) |
| **Account customizations** | AWS-native only | Any Terraform resource |
| **Global customizations** | Not supported | Supported (applied to all accounts) |

**Use AFT when:**
- Your organization uses Terraform as the IaC standard
- You need complex, repeatable account customizations
- You want GitOps-based account vending workflows
- You need to apply global customizations to all accounts

---

## Hard

---

### 1. Explain the internal architecture of how Control Tower enforces Guardrails at the OU level using SCPs. What are the limitations of SCP-based enforcement?

**Answer:**

**SCP Enforcement Architecture:**

When a preventive guardrail is enabled on an OU, Control Tower attaches an **SCP** to that OU in AWS Organizations. The SCP uses an explicit **Deny** statement, which overrides any Allow policies in member accounts.

**Example SCP for "Disallow deletion of AWS CloudTrail":**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GRCLOUDTRAILENABLED",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalARN": "arn:aws:iam::*:role/AWSControlTowerExecution"
        }
      }
    }
  ]
}
```

**Key design pattern:** The `AWSControlTowerExecution` role is exempted from SCPs using `ArnNotLike` conditions, allowing Control Tower itself to manage resources without being blocked by its own guardrails.

**Inheritance:** SCPs are inherited hierarchically. An SCP on a parent OU applies to all child OUs and accounts. Control Tower leverages this by applying guardrails at the OU level.

**SCP Limitations:**
1. **Do not apply to the Management Account** — The root/management account is exempt from SCPs by design. This is why sensitive operations should never be performed from the management account.
2. **Do not affect resource-based policies** — SCPs only restrict identity-based actions. An S3 bucket policy can still allow cross-account access even if an SCP restricts it.
3. **Service-linked roles are not exempt** — SCPs can accidentally block AWS service-linked roles if not carefully crafted.
4. **No Allow effect for SCPs alone** — SCPs only restrict; they don't grant permissions. An SCP Allow just means "don't deny," but the identity still needs an IAM Allow.
5. **Maximum 5 SCPs per entity** — AWS Organizations limits to 5 SCPs per OU/account, which can be a constraint for organizations with many guardrails.
6. **No condition key for all services** — Some services don't support all condition keys, limiting fine-grained SCP conditions.
7. **Cannot restrict root user actions for certain operations** — Root user cannot be restricted from actions like closing an account or changing support plan.

---

### 2. How would you design a customized Control Tower Landing Zone for an enterprise with strict data residency requirements, multiple business units, and existing AWS accounts?

**Answer:**

**Requirements Analysis:**
- Data residency: Resources must stay in specific AWS Regions (e.g., EU-only)
- Multiple business units (BUs): Separate governance, billing, and autonomy
- Existing accounts: Must be enrolled without disruption

**Architectural Design:**

```
Root (Management Account)
├── Security OU (Control Tower Managed)
│   ├── Log Archive Account
│   └── Audit Account
├── Infrastructure OU
│   ├── Network Account (Transit Gateway, DNS)
│   └── Shared Services Account
├── Business Unit A OU
│   ├── BU-A Dev OU
│   │   └── Dev Accounts (x3)
│   ├── BU-A Staging OU
│   └── BU-A Prod OU
│       └── Prod Accounts (x5)
├── Business Unit B OU
│   └── (same structure)
└── Sandbox OU
    └── Individual developer accounts
```

**Data Residency Enforcement:**

1. **Region Deny SCP** applied at the root or BU OU level:
```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["eu-west-1", "eu-central-1"]
    },
    "ArnNotLike": {
      "aws:PrincipalARN": [
        "arn:aws:iam::*:role/AWSControlTowerExecution",
        "arn:aws:iam::*:role/aws-reserved/sso.amazonaws.com/*"
      ]
    }
  }
}
```

2. **Global services exemption** — IAM, Route 53, CloudFront, etc. are global and must be exempted from region deny SCPs.

**Multi-BU Design Considerations:**
- Each BU OU gets its own set of **elective guardrails** appropriate to their compliance requirements
- **Separate permission sets** in IAM Identity Center per BU
- **Separate billing alerts** using AWS Budgets per OU
- **Delegated Administrator** pattern: Each BU can have a delegated admin account for services like Security Hub, GuardDuty

**Existing Account Enrollment Strategy:**
1. Audit existing accounts for Config recorders, CloudTrail conflicts
2. Create `AWSControlTowerExecution` role via CloudFormation StackSet from the management account
3. Enroll accounts in batches, starting with non-production
4. Use **AFT** (Account Factory for Terraform) to codify the enrollment process
5. Validate compliance post-enrollment using the Control Tower dashboard

**Additional Customizations:**
- **Custom Lambda hooks** via Control Tower lifecycle events to apply additional tagging policies
- **AWS Config Conformance Packs** for industry-specific compliance (PCI-DSS, GDPR)
- **Centralized Security Hub** with delegated admin in the Audit account
- **AWS Organizations Tag Policies** for consistent tagging

---

### 3. How do Control Tower lifecycle events work, and how can you use them to implement custom automation?

**Answer:**

**Lifecycle Events Overview:**
Control Tower publishes **lifecycle events** to **Amazon EventBridge** in