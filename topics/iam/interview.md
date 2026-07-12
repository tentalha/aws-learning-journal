# IAM — Interview Questions

---

## Easy

### 1. What is AWS IAM and what problem does it solve?

**Answer:**
AWS Identity and Access Management (IAM) is a global AWS service that enables you to securely control access to AWS services and resources. It solves the problem of managing *who* (authentication) can do *what* (authorization) within your AWS account.

Without IAM, every user would need the root account credentials, creating massive security risks. IAM lets you create individual identities (users, groups, roles) and attach fine-grained permissions to them, following the principle of least privilege.

---

### 2. What is the difference between an IAM User, Group, and Role?

**Answer:**

| Entity | Description |
|--------|-------------|
| **IAM User** | A permanent identity representing a person or service. Has long-term credentials (password, access keys). |
| **IAM Group** | A collection of IAM users. Policies attached to a group apply to all members. Groups cannot be nested. |
| **IAM Role** | A temporary identity assumed by trusted entities (users, services, applications). Uses short-term credentials via STS. Has no permanent credentials. |

**Key distinction:** Roles are preferred over users for applications and services because they use temporary credentials, reducing the risk of credential exposure.

---

### 3. What is an IAM Policy and what are the main types?

**Answer:**
An IAM Policy is a JSON document that defines permissions. It specifies what actions are allowed or denied on which resources under what conditions.

**Main types:**
- **Identity-based policies** — Attached to users, groups, or roles. Can be *managed* (AWS-managed or customer-managed) or *inline*.
- **Resource-based policies** — Attached directly to a resource (e.g., S3 bucket policy, SQS queue policy). Support cross-account access without assuming a role.
- **Permission boundaries** — Set the maximum permissions an identity-based policy can grant.
- **Service Control Policies (SCPs)** — Applied at the AWS Organizations level; define guardrails across accounts.
- **Session policies** — Passed inline when assuming a role via STS.
- **ACLs (Access Control Lists)** — Legacy mechanism for cross-account access on resources like S3.

---

### 4. What is the AWS root account and why should it not be used for day-to-day operations?

**Answer:**
The root account is the account created when you first sign up for AWS. It has **unrestricted access** to all AWS services and resources, including billing.

**Reasons to avoid daily use:**
- Cannot be restricted by IAM policies or SCPs.
- If compromised, the attacker has full control over everything.
- No audit trail separation — all actions appear under the root identity.
- AWS best practices explicitly recommend locking it away.

**Best practices for root account:**
- Enable MFA immediately.
- Do not create access keys for root.
- Use root only for tasks that *require* it (e.g., changing account email, enabling AWS Support plans, closing the account).

---

### 5. What does the principle of least privilege mean in the context of IAM?

**Answer:**
The principle of least privilege means granting an identity **only the permissions it needs** to perform its specific task — nothing more.

**In IAM practice:**
- Start with zero permissions and add only what is required.
- Avoid wildcard actions (`"Action": "*"`) or wildcard resources (`"Resource": "*"`) unless absolutely necessary.
- Use IAM Access Analyzer and AWS Trusted Advisor to identify overly permissive policies.
- Regularly review and revoke unused permissions using IAM Access Advisor (last accessed information).
- Use permission boundaries to limit maximum permissions for delegated administrators.

---

## Medium

### 1. Explain how IAM policy evaluation logic works when multiple policies are attached.

**Answer:**
AWS evaluates all applicable policies using a specific order of precedence. The evaluation logic is:

1. **Explicit Deny wins** — If *any* policy (identity-based, resource-based, SCP, permissions boundary) explicitly denies an action, it is denied regardless of any allow.
2. **SCPs** — If AWS Organizations SCPs are in place, the action must be allowed by the SCP.
3. **Resource-based policy** — If a resource-based policy grants access (same account), access may be allowed even without an identity-based policy.
4. **Permissions boundary** — If a boundary is set, the effective permissions are the intersection of the boundary and identity-based policies.
5. **Identity-based policy** — The action must be explicitly allowed.
6. **Default Deny** — If no explicit allow is found, the request is implicitly denied.

**Simplified formula:**
```
Effective Permissions = (Identity-based ∩ Permissions Boundary) ∩ SCP — Explicit Denies
```

**Cross-account access** requires an explicit allow in *both* the identity-based policy of the caller AND the resource-based policy of the target resource (or the caller must assume a role in the target account).

---

### 2. What are IAM Roles and how does the assume-role process work?

**Answer:**
An IAM Role is an identity with a trust policy and permission policies. Instead of being associated with a person, it is assumed temporarily by trusted entities.

**The assume-role flow:**

```
1. Principal (user/service/account) calls STS:AssumeRole
2. STS validates:
   - The caller is listed in the role's trust policy
   - The caller has sts:AssumeRole permission in their identity policy
3. STS returns temporary credentials:
   - AccessKeyId
   - SecretAccessKey
   - SessionToken
   - Expiration (15 min to 12 hours)
4. Principal uses these credentials to make API calls
5. Credentials expire; principal must re-assume the role
```

**Trust policy example (allows EC2 to assume the role):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

**Common use cases:**
- EC2 instance profiles (application assumes role automatically via metadata service)
- Cross-account access
- Identity federation (web identity, SAML)
- Lambda execution roles

---

### 3. What is the difference between AWS-managed policies and customer-managed policies? When would you use each?

**Answer:**

| Aspect | AWS-Managed Policies | Customer-Managed Policies |
|--------|---------------------|--------------------------|
| **Owner** | AWS creates and maintains | You create and maintain |
| **Updates** | AWS updates automatically | You control updates |
| **Versioning** | Managed by AWS | Up to 5 versions stored |
| **Reusability** | Shared across accounts | Reusable within your account |
| **Granularity** | Broad (e.g., `AmazonS3ReadOnlyAccess`) | Tailored to your exact needs |
| **Visibility** | Available in all accounts | Only in your account |

**When to use AWS-managed:**
- Quick prototyping or development environments.
- When the policy aligns exactly with your needs (e.g., `ReadOnlyAccess` for auditors).
- When you want AWS to handle policy updates as services evolve.

**When to use customer-managed:**
- Production environments requiring least privilege.
- When you need to scope down to specific resources (e.g., specific S3 bucket ARNs).
- When compliance requires you to audit and control every permission change.
- When building permission boundaries or delegated admin patterns.

**Inline policies** should generally be avoided — they cannot be reused, are harder to audit, and are deleted when the identity is deleted.

---

### 4. How does IAM handle cross-account access? Describe two methods.

**Answer:**

**Method 1: Cross-Account Role Assumption**

This is the recommended approach.

```
Account A (Trusting) — contains the Role
Account B (Trusted) — contains the User/Role that assumes

Steps:
1. In Account A: Create a role with a trust policy allowing Account B's principal
2. In Account B: Attach a policy to the user/role allowing sts:AssumeRole on Account A's role ARN
3. User in Account B calls sts:AssumeRole, gets temporary credentials for Account A's role
4. User makes API calls in Account A using those credentials
```

**Trust policy in Account A:**
```json
{
  "Principal": { "AWS": "arn:aws:iam::ACCOUNT_B_ID:role/MyRole" },
  "Action": "sts:AssumeRole"
}
```

**Method 2: Resource-Based Policies**

Some services (S3, SQS, SNS, KMS, Lambda) support resource-based policies that can directly grant access to principals in other accounts without role assumption.

```json
{
  "Principal": { "AWS": "arn:aws:iam::ACCOUNT_B_ID:root" },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```

**Key differences:**
- Role assumption: Credentials change, CloudTrail shows the assumed role identity.
- Resource-based: No credential change; the caller retains their original identity. Simpler but less auditable for complex scenarios.

---

### 5. What is IAM Access Analyzer and how does it help with security?

**Answer:**
IAM Access Analyzer is a service that continuously analyzes resource-based policies to identify resources that are accessible from **outside your AWS account or organization** — called "external access findings."

**How it works:**
- You create an **analyzer** scoped to an account or AWS Organization.
- Access Analyzer uses automated reasoning (Zelkova, a formal verification engine) to mathematically prove whether a policy grants external access.
- It generates **findings** for resources like S3 buckets, KMS keys, IAM roles, Lambda functions, SQS queues, and Secrets Manager secrets.

**Key capabilities:**
1. **External access analysis** — Identifies unintended public or cross-account access.
2. **Unused access analysis** — Identifies unused roles, users, and permissions (helps right-size permissions).
3. **Policy validation** — Checks policies for syntax errors, security warnings, and best practice violations before deployment.
4. **Policy generation** — Analyzes CloudTrail logs and generates least-privilege policies based on actual usage.

**Workflow:**
```
Finding Generated → Review Finding → Archive (expected) or Remediate (unexpected)
```

**Integration points:**
- EventBridge for automated remediation workflows.
- Security Hub for centralized finding management.
- AWS Config for compliance tracking.

---

## Hard

### 1. Explain the complete IAM policy evaluation for a cross-account request involving SCPs, permission boundaries, and resource-based policies.

**Answer:**
This is one of the most complex scenarios in IAM. Let's walk through the full evaluation chain.

**Scenario:** User in Account A (member of AWS Org) tries to access an S3 bucket in Account B.

**Complete evaluation steps:**

```
Step 1: SCP Evaluation (Account A)
  → Is the action allowed by the SCP attached to Account A's OU?
  → If SCP denies → REQUEST DENIED

Step 2: SCP Evaluation (Account B)  
  → Is the action allowed by the SCP attached to Account B's OU?
  → If SCP denies → REQUEST DENIED

Step 3: Permissions Boundary (Account A user)
  → Does a permissions boundary exist on the user?
  → If boundary exists, action must be allowed by boundary
  → If boundary denies → REQUEST DENIED

Step 4: Identity-based Policy (Account A user)
  → Does the user's identity policy allow the action on the S3 ARN in Account B?
  → If no allow → REQUEST DENIED

Step 5: Resource-based Policy (Account B S3 bucket)
  → Does the bucket policy explicitly allow the principal from Account A?
  → If no allow → REQUEST DENIED

Step 6: Explicit Deny Check
  → Any explicit deny in any policy at any step → REQUEST DENIED

Step 7: If all checks pass → REQUEST ALLOWED
```

**Critical insight for cross-account:**
Both the identity-based policy in Account A AND the resource-based policy in Account B must grant access. This is unlike same-account access where *either* can grant access.

**Mathematical representation:**
```
Cross-Account Access = 
  (SCPs_AccountA ∩ SCPs_AccountB) ∩ 
  (IdentityPolicy ∩ PermissionsBoundary) ∩ 
  ResourcePolicy — ExplicitDenies
```

**Common pitfall:** Developers often grant the identity-based policy in Account A but forget the bucket policy in Account B, resulting in access denied errors that are hard to debug.

---

### 2. How would you implement a secure, scalable IAM strategy for a large enterprise with 50+ AWS accounts using AWS Organizations?

**Answer:**
This requires a multi-layered approach combining Organizations, SCPs, permission boundaries, and centralized identity management.

**Architecture layers:**

**Layer 1: Account Structure**
```
Root (Management Account)
├── Security OU
│   ├── Log Archive Account
│   └── Security Tooling Account
├── Infrastructure OU
│   └── Shared Services Account
├── Workload OU
│   ├── Production OU
│   │   ├── Prod Account 1
│   │   └── Prod Account 2
│   └── Non-Production OU
│       ├── Dev Account
│       └── Staging Account
└── Sandbox OU
    └── Sandbox Accounts
```

**Layer 2: SCPs as Guardrails**
```json
// Deny leaving the organization
{
  "Effect": "Deny",
  "Action": "organizations:LeaveOrganization",
  "Resource": "*"
}

// Restrict to approved regions
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
    }
  }
}

// Protect security services
{
  "Effect": "Deny",
  "Action": [
    "cloudtrail:StopLogging",
    "guardduty:DeleteDetector",
    "config:DeleteConfigRule"
  ],
  "Resource": "*"
}
```

**Layer 3: Centralized Identity with IAM Identity Center (SSO)**
- Single sign-on from corporate IdP (Okta, Azure AD) via SAML/OIDC.
- Permission Sets defined centrally, deployed to accounts.
- Users never have long-term IAM users — always federated sessions.

**Layer 4: Permission Boundaries for Delegation**
```json
// Boundary for developer-created roles
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:*", "dynamodb:*", "lambda:*"],
    "Resource": "*"
  },
  {
    "Effect": "Deny",
    "Action": ["iam:*", "organizations:*", "account:*"],
    "Resource": "*"
  }]
}
```

**Layer 5: Automated Compliance**
- AWS Config Rules to detect IAM policy violations.
- Access Analyzer at Organization level for external access findings.
- CloudTrail + Security Hub for centralized audit.
- Automated remediation via EventBridge + Lambda.

**Key principles:**
- No IAM users with long-term credentials in workload accounts.
- Break-glass root access with hardware MFA, stored in vault.
- Regular permission reviews using IAM Access Advisor data.
- Infrastructure-as-Code (CDK/Terraform) for all IAM resources.

---

### 3. Explain IAM Roles Anywhere and how it enables non-AWS workloads to use IAM roles. What are the security considerations?

**Answer:**
IAM Roles Anywhere extends the IAM role assumption model to workloads running **outside of AWS** (on-premises servers, containers, CI/CD systems) by using X.509 certificates as the identity mechanism instead of AWS credentials.

**How it works:**

```
Architecture:
[On-Premises Server] → [IAM Roles Anywhere] → [STS] → [Temporary Credentials]

Components:
1. Trust Anchor — References your Certificate Authority (CA)
   - Can be AWS Private CA or external CA (self-signed, enterprise CA)
   
2. Profile — Maps certificate attributes to IAM roles
   - Defines which roles can be assumed
   - Sets session duration and managed policy overrides
   
3. Subject — The workload with the X.509 certificate
```

**Authentication flow:**
```