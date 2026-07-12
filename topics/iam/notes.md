# IAM

## What is it?

**AWS Identity and Access Management (IAM)** is a foundational AWS security service that enables you to manage access to AWS services and resources securely. It falls under the **Security, Identity, and Compliance** category of AWS services.

IAM allows you to:
- Create and manage **AWS users, groups, roles, and policies**
- Control **who** (authentication) can do **what** (authorization) on **which** AWS resources
- Enforce **least-privilege access** across your entire AWS environment

IAM is a **global service** — it is not region-specific, and identities created in IAM are available across all AWS regions within an account. It is provided at **no additional cost**.

---

## Why do we need it?

### The Problem It Solves

Without IAM, every person and application would need to use the root account credentials to interact with AWS — a catastrophic security risk. IAM solves the challenge of **multi-user, multi-service access control** in a cloud environment.

### Core Problems Addressed

| Problem | IAM Solution |
|---|---|
| Multiple developers need different access levels | Users and Groups with scoped policies |
| Applications need to call AWS APIs securely | IAM Roles with temporary credentials |
| Third-party tools need limited AWS access | Cross-account roles and permission boundaries |
| Compliance requires audit trails | CloudTrail + IAM activity logging |
| Prevent accidental resource deletion | Deny policies and SCPs |

### Real Business Scenarios

1. **Startup with a Dev Team**: A company has developers, QA engineers, and DevOps staff. Developers need read/write to S3 and Lambda, QA needs read-only to CloudWatch logs, and DevOps needs full EC2 access. IAM groups and policies enforce this cleanly.

2. **CI/CD Pipeline**: A Jenkins or GitHub Actions pipeline needs to deploy Lambda functions and update S3 buckets. Instead of embedding credentials in code, an IAM Role is assumed by the pipeline runner.

3. **SaaS Multi-Tenant Application**: A SaaS provider needs to access customer AWS accounts. They create IAM Roles in customer accounts that their service assumes via cross-account role assumption.

4. **Regulatory Compliance (HIPAA/PCI-DSS)**: Organizations must demonstrate that only authorized personnel access sensitive data. IAM policies, MFA enforcement, and CloudTrail logs satisfy audit requirements.

5. **Microservices Architecture**: An ECS task running a payment service needs to write to DynamoDB. An IAM Task Role provides scoped, temporary credentials without hardcoding secrets.

---

## Internal Working

### Authentication and Authorization Flow

IAM operates on a **request evaluation model**. Every API call to AWS goes through a multi-step evaluation process:

```
API Request
    │
    ▼
┌─────────────────────────────────────────────┐
│  1. Authentication                          │
│     - Verify identity (Access Key / Token)  │
│     - Validate signature (SigV4)            │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│  2. Context Building                        │
│     - Principal (who is making the call?)   │
│     - Action (what are they doing?)         │
│     - Resource (what resource is targeted?) │
│     - Conditions (IP, MFA, time, tags?)     │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│  3. Policy Evaluation Logic                 │
│     - Collect all applicable policies       │
│     - Evaluate in order (see below)         │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│  4. Allow or Deny Decision                  │
└─────────────────────────────────────────────┘
```

### Policy Evaluation Order

AWS evaluates policies in a **strict priority order**:

```
1. Explicit DENY (in any policy)      → DENY immediately
2. AWS Organizations SCPs             → Must Allow
3. Resource-based policies            → Can Allow
4. IAM Permission Boundaries          → Must Allow
5. Session Policies                   → Must Allow
6. Identity-based policies            → Must Allow
7. Default                            → DENY (implicit)
```

> **Key Rule**: An explicit `Deny` always overrides any `Allow`. If no explicit `Allow` exists, the default is `Deny`.

### Credential Types

| Credential Type | Use Case | Expiry |
|---|---|---|
| Username + Password | AWS Console login | Until changed |
| Access Key ID + Secret Access Key | Programmatic access | Until rotated/deleted |
| Temporary Security Credentials (STS) | Roles, federated users | 15 min – 36 hours |
| X.509 Certificates | SOAP API calls (legacy) | Until expired |

### STS (Security Token Service) Internals

When a role is assumed, AWS STS generates **temporary credentials** consisting of:
- `AccessKeyId`
- `SecretAccessKey`
- `SessionToken`
- `Expiration`

These are signed using AWS's internal key management and validated on every API call without a round-trip to IAM (credentials are cryptographically verifiable).

---

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Account                              │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  Users   │    │  Groups  │    │  Roles   │    │ Policies │  │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘  │
│       │               │               │               │        │
│       └───────────────┴───────────────┘               │        │
│                       │                               │        │
│                       ▼                               │        │
│              ┌─────────────────┐                      │        │
│              │   Principals    │◄─────────────────────┘        │
│              └────────┬────────┘                               │
│                       │                                        │
│                       ▼                                        │
│              ┌─────────────────┐                               │
│              │  AWS STS        │                               │
│              │  (Token Service)│                               │
│              └────────┬────────┘                               │
│                       │                                        │
│                       ▼                                        │
│              ┌─────────────────┐                               │
│              │  AWS Resources  │                               │
│              │  (S3, EC2, etc.)│                               │
│              └─────────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```

### Component Breakdown

#### 1. **Users**
- Represent a **human or application** identity
- Have long-term credentials (password, access keys)
- Can belong to multiple groups
- Best practice: Use roles instead of users for applications

#### 2. **Groups**
- Logical collection of users
- Policies attached to groups apply to all members
- Groups **cannot be nested** (no groups within groups)
- A user can belong to **up to 10 groups**

#### 3. **Roles**
- An identity with **no long-term credentials**
- Assumed by trusted entities (users, services, accounts)
- Issues temporary credentials via STS
- Types:
  - **Service Roles**: Assumed by AWS services (EC2, Lambda, ECS)
  - **Cross-Account Roles**: Assumed by identities in other AWS accounts
  - **Identity Provider Roles**: Assumed by federated users (SAML, OIDC)

#### 4. **Policies**
JSON documents defining permissions. Types:

| Policy Type | Attached To | Use Case |
|---|---|---|
| **Identity-based** | Users, Groups, Roles | Grant permissions to principals |
| **Resource-based** | Resources (S3, SQS) | Grant cross-account or service access |
| **Permission Boundaries** | Users, Roles | Cap maximum permissions |
| **Service Control Policies (SCPs)** | AWS Org OUs/Accounts | Organization-wide guardrails |
| **Session Policies** | AssumeRole API calls | Restrict temporary session permissions |
| **Access Control Lists (ACLs)** | S3, VPC | Legacy cross-account resource access |

#### 5. **Identity Providers (IdPs)**
- **SAML 2.0**: Corporate directory federation (Active Directory, Okta)
- **OpenID Connect (OIDC)**: Web identity federation (Google, GitHub Actions, Cognito)
- **AWS SSO / IAM Identity Center**: Centralized multi-account access

### Policy Document Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadAccess",
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

---

## Real World Example

### Scenario: E-Commerce Platform with Multiple Teams

**Company**: An e-commerce company with 50 engineers across Dev, QA, DevOps, and Data teams needs a secure, auditable IAM structure.

#### Step 1: Design the Group Structure

```
IAM Groups:
├── Developers
│   ├── Policy: AmazonS3FullAccess (scoped to dev-*)
│   ├── Policy: AWSLambda_FullAccess
│   └── Policy: AmazonDynamoDBFullAccess
├── QA
│   ├── Policy: ReadOnlyAccess (CloudWatch, S3)
│   └── Policy: Custom: QATestingPolicy
├── DevOps
│   ├── Policy: AdministratorAccess (with Permission Boundary)
│   └── Policy: Custom: InfraDeployPolicy
└── DataEngineers
    ├── Policy: AmazonAthenaFullAccess
    └── Policy: AmazonS3ReadOnlyAccess (scoped to data-lake-*)
```

#### Step 2: Create Service Roles for Applications

```
IAM Roles:
├── ecommerce-api-role (for ECS Tasks)
│   ├── DynamoDB: PutItem, GetItem, Query on orders-table
│   ├── S3: GetObject on product-images-bucket
│   └── SQS: SendMessage on order-processing-queue
├── ecommerce-lambda-role (for Lambda Functions)
│   ├── DynamoDB: GetItem on inventory-table
│   ├── SNS: Publish on notifications-topic
│   └── CloudWatch Logs: CreateLogGroup, PutLogEvents
└── ecommerce-cicd-role (for GitHub Actions OIDC)
    ├── ECR: BatchCheckLayerAvailability, PutImage
    ├── ECS: UpdateService, RegisterTaskDefinition
    └── S3: PutObject on deployment-artifacts-bucket
```

#### Step 3: Configure GitHub Actions OIDC (No Static Credentials)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:myorg/ecommerce-app:*"
        }
      }
    }
  ]
}
```

#### Step 4: Enforce MFA for Console Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyWithoutMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ListVirtualMFADevices",
        "iam:ResyncMFADevice",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

#### Step 5: Set Up Cross-Account Role for Monitoring

```
Production Account (123456789012)
    └── Role: monitoring-read-role
        ├── Trust: Security Account (987654321098)
        └── Policy: ReadOnlyAccess + CloudTrail:GetTrailStatus

Security Account (987654321098)
    └── Users assume monitoring-read-role to audit production
```

---

## Advantages

1. **Zero Additional Cost**: IAM is completely free — no charges for users, roles, policies, or API calls.

2. **Granular Permission Control**: Policies can be scoped to specific resources, actions, and conditions — down to individual S3 objects or DynamoDB items.

3. **Temporary Credentials via STS**: Roles eliminate the need for long-lived credentials, dramatically reducing the attack surface.

4. **Centralized Access Management**: Single control plane for all AWS services and resources across an account.

5. **Federation Support**: Seamlessly integrates with corporate directories (Active Directory via SAML), social IdPs, and OIDC providers.

6. **Attribute-Based Access Control (ABAC)**: Use tags on resources and principals to dynamically control access without modifying policies.

7. **Permission Boundaries**: Delegate IAM management safely — developers can create roles but cannot exceed their own permission boundary.

8. **Global Availability**: IAM is a global service with high availability built-in; no regional configuration needed.

9. **Audit and Compliance**: Every IAM action is logged in CloudTrail, enabling full audit trails for compliance (SOC2, HIPAA, PCI-DSS).

10. **AWS Organizations Integration**: Service Control Policies (SCPs) provide account-level guardrails across hundreds of accounts.

11. **Condition Keys**: Fine-grained conditions based on IP address, VPC, MFA status, request time, tags, and more.

12. **IAM Access Analyzer**: Automatically identifies overly permissive policies and external access.

---

## Limitations

### Hard Limits (Service Quotas)

| Resource | Default Limit |
|---|---|
| IAM users per account | 5,000 |
| Groups per account | 300 |
| Roles per account | 1,000 |
| Managed policies per account | 1,500 |
| Managed policies per user/role/group | 10 |
| Inline policy size per entity | 2,048 characters |
| Managed policy size | 6,144 characters |
| Groups a user can belong to | 10 |
| Versions per managed policy | 5 |
| MFA devices per user | 8 |
| Role session duration (max) | 12 hours |
| STS temporary credentials minimum | 15 minutes |
| Access keys per user | 2 |
| Signing certificates per user | 2 |

> Most of these limits can be increased via AWS Support request, **except** the 2 access keys per user limit.

### Functional Limitations

1. **No Group Nesting**: Groups cannot contain other groups — only users.
2. **No Cross-Account Group Membership**: Users from Account A cannot be added to a group in Account