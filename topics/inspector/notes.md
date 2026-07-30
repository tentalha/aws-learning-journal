# Inspector

## What is it?

**AWS Inspector** is a fully managed **automated security assessment service** that continuously scans AWS workloads for software vulnerabilities and unintended network exposure. It falls under the **Security, Identity & Compliance** category of AWS services.

AWS Inspector automatically discovers and scans:
- **Amazon EC2 instances** for software vulnerabilities and network reachability issues
- **Amazon ECR container images** for known software vulnerabilities
- **AWS Lambda functions** for software vulnerabilities in application code and package dependencies

> **Note:** AWS Inspector v2 (launched in 2021) is the current generation, replacing the original Inspector v1. Inspector v2 is fundamentally re-architected with continuous scanning, automatic discovery, and deeper AWS integrations. Inspector v1 is deprecated and should not be used for new implementations.

Inspector uses the **Common Vulnerabilities and Exposures (CVE)** database along with other threat intelligence feeds to identify vulnerabilities and assigns risk scores using the **Amazon Inspector Risk Score**, which is a contextualized version of the CVSS (Common Vulnerability Scoring System) score.

---

## Why do we need it?

### The Problem

Modern cloud environments face several security challenges:

1. **Rapidly changing vulnerability landscape** — New CVEs are published daily. Manually tracking and patching vulnerabilities across hundreds of EC2 instances, container images, and Lambda functions is operationally infeasible.
2. **Unintended network exposure** — Misconfigured security groups or network ACLs can inadvertently expose sensitive workloads to the internet.
3. **Compliance requirements** — Regulations like PCI-DSS, HIPAA, SOC 2, and FedRAMP require continuous vulnerability management and evidence of remediation.
4. **Shift-left security** — Teams need to catch vulnerabilities early in the CI/CD pipeline, not after production deployment.
5. **Container sprawl** — With the proliferation of container images in ECR, ensuring each image is free of critical vulnerabilities before deployment is challenging.

### When to Use It

| Scenario | Why Inspector Helps |
|---|---|
| **DevSecOps pipeline** | Scan ECR images before deployment; block critical CVEs |
| **Compliance auditing** | Continuous evidence of vulnerability scanning for regulators |
| **EC2 fleet management** | Automatic discovery and scanning of all EC2 instances |
| **Serverless security** | Scan Lambda function packages for vulnerable dependencies |
| **Multi-account organizations** | Centralized vulnerability management across all AWS accounts |

### Real Business Scenarios

- **E-commerce platform**: A retail company running hundreds of EC2 instances needs to ensure PCI-DSS compliance. Inspector continuously scans all instances and generates findings that feed into their compliance reporting dashboard.
- **SaaS company**: A startup builds container-based microservices. Inspector scans every image pushed to ECR, blocking deployments of images with critical vulnerabilities via an EventBridge + CodePipeline integration.
- **Healthcare provider**: A HIPAA-covered entity uses Inspector to maintain continuous visibility into software vulnerabilities across their AWS environment, generating audit-ready reports.

---

## Internal Working

### Discovery Mechanism

AWS Inspector v2 uses the **AWS Systems Manager (SSM) Agent** for EC2 scanning. When Inspector is enabled:

1. **Automatic Discovery**: Inspector uses AWS APIs to discover all EC2 instances, ECR repositories, and Lambda functions within enabled accounts.
2. **SSM Agent Communication**: For EC2 instances, Inspector communicates with the SSM Agent (must be installed and running) to collect software inventory — installed packages, OS version, kernel version.
3. **Agentless Scanning (EC2)**: Inspector v2 also supports **agentless scanning** for EC2 instances that don't have SSM Agent, using EBS snapshot-based scanning.

### Vulnerability Matching Engine

```
Software Inventory → CVE Database Lookup → Risk Scoring → Finding Generation
```

1. **Software Inventory Collection**: The SSM Agent reports all installed packages (RPM, DEB, etc.) and their versions.
2. **CVE Matching**: Inspector matches the collected inventory against:
   - **NVD (National Vulnerability Database)**
   - **Vendor-specific advisories** (Amazon Linux, Ubuntu, Red Hat, etc.)
   - **GitHub Security Advisories** (for Lambda/container dependencies)
3. **Risk Score Calculation**: Inspector calculates an **Amazon Inspector Risk Score** (0.0–10.0) by combining:
   - Base CVSS score
   - Exploitability data (CISA KEV, exploit maturity)
   - Network reachability (is the instance internet-facing?)
   - Environmental context (is the instance in a production environment?)

### Scanning Types

| Scan Type | Target | Method |
|---|---|---|
| **Package vulnerability scan** | EC2, ECR, Lambda | CVE matching against installed packages |
| **Network reachability scan** | EC2 | VPC configuration analysis (SGs, NACLs, routing) |
| **Code vulnerability scan** | Lambda | Static code analysis (Python, Java) |
| **ECR image scan** | Container images | Layer-by-layer OS and library scanning |

### Continuous vs. On-Demand Scanning

- **Continuous scanning**: Inspector rescans when new CVEs are published, when packages change, or when network configurations change — without manual intervention.
- **On-demand scanning** (ECR): Can be triggered manually or via pipeline for specific images.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Organization                          │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Delegated Administrator Account              │   │
│  │                                                           │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │           AWS Inspector v2 Console               │    │   │
│  │  │  - Aggregated Findings Dashboard                 │    │   │
│  │  │  - Coverage Summary                              │    │   │
│  │  │  - Risk-based Prioritization                     │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │   Member Account │  │   Member Account │  │Member Account│  │
│  │                  │  │                  │  │              │  │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │ ┌──────────┐│  │
│  │  │ EC2 + SSM  │  │  │  │    ECR     │  │  │ │ Lambda   ││  │
│  │  │   Agent    │  │  │  │  Images    │  │  │ │Functions ││  │
│  │  └────────────┘  │  │  └────────────┘  │  │ └──────────┘│  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     ┌──────────────┐ ┌─────────────┐ ┌────────────────┐
     │  EventBridge │ │  Security   │ │  S3 (Reports)  │
     │   (Findings) │ │    Hub      │ │                │
     └──────────────┘ └─────────────┘ └────────────────┘
              │               │
     ┌────────▼───────┐ ┌─────▼──────────────┐
     │ Lambda/SNS/SQS │ │  SIEM / Ticketing  │
     │  (Automation)  │ │  (Jira, Splunk)    │
     └────────────────┘ └────────────────────┘
```

### Key Architectural Components

| Component | Role |
|---|---|
| **Inspector Service** | Core scanning engine; manages discovery, scanning, and finding generation |
| **SSM Agent** | Agent on EC2 instances that provides software inventory data |
| **ECR Integration** | Hooks into image push events to trigger scans automatically |
| **Lambda Integration** | Scans function deployment packages and layer dependencies |
| **Findings Database** | Stores all findings with full context, filterable via API |
| **Delegated Administrator** | Single account (usually Security account) managing Inspector across the org |
| **EventBridge Integration** | Publishes finding events for automated remediation workflows |
| **Security Hub Integration** | Aggregates findings alongside GuardDuty, Macie, etc. |

### Multi-Account Architecture Pattern

```
Management Account
    └── Delegates Inspector Admin to → Security Account
                                            └── Enables Inspector in → All Member Accounts
                                                                         (via Organizations)
```

---

## Real World Example

### Scenario: Secure Container Deployment Pipeline for a FinTech Application

**Context**: A FinTech company runs microservices on Amazon ECS. They need to ensure no container image with critical CVEs reaches production and that all EC2 instances (for legacy services) are continuously monitored.

#### Step-by-Step Walkthrough

**Step 1: Enable Inspector v2**
```bash
# Enable Inspector for the account (or via Organizations for all accounts)
aws inspector2 enable --resource-types EC2 ECR LAMBDA
```

**Step 2: Configure ECR Enhanced Scanning**
```bash
# Set ECR repository to use enhanced scanning (Inspector-powered)
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{"repositoryFilters": [{"filter": "*", "filterType": "WILDCARD"}], "scanFrequency": "CONTINUOUS_SCAN"}]'
```

**Step 3: Developer Pushes Image to ECR**
```bash
docker build -t my-fintech-app:v1.2.3 .
docker tag my-fintech-app:v1.2.3 123456789.dkr.ecr.us-east-1.amazonaws.com/fintech-app:v1.2.3
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/fintech-app:v1.2.3
# Inspector automatically scans the pushed image
```

**Step 4: EventBridge Rule Captures Critical Findings**
```json
{
  "source": ["aws.inspector2"],
  "detail-type": ["Inspector2 Finding"],
  "detail": {
    "severity": ["CRITICAL", "HIGH"],
    "status": ["ACTIVE"],
    "type": ["PACKAGE_VULNERABILITY"]
  }
}
```

**Step 5: Lambda Function Blocks Deployment**
```javascript
// Lambda triggered by EventBridge
exports.handler = async (event) => {
  const finding = event.detail;
  const imageDigest = finding.resources[0].details.awsEcrContainerImage.imageDigest;
  
  // Add ECR lifecycle policy or tag to block deployment
  await ecrClient.send(new PutImageCommand({
    repositoryName: 'fintech-app',
    imageTag: 'BLOCKED-CRITICAL-CVE',
    imageManifest: imageDigest
  }));
  
  // Notify security team via SNS
  await snsClient.send(new PublishCommand({
    TopicArn: process.env.SECURITY_TOPIC_ARN,
    Subject: `CRITICAL CVE found in ${finding.resources[0].id}`,
    Message: JSON.stringify(finding, null, 2)
  }));
};
```

**Step 6: Security Team Reviews Finding**
- Inspector finding shows: `CVE-2023-XXXX` in `log4j-core:2.14.0` with Inspector Risk Score: **9.8**
- Finding includes: affected packages, fix version, exploit availability, network exposure context

**Step 7: Remediation**
```dockerfile
# Developer updates Dockerfile
FROM amazoncorretto:17
# Update vulnerable dependency
RUN pip install log4j-core==2.17.1
```

**Step 8: Compliance Report Generation**
```bash
# Generate CSV report of all findings for audit
aws inspector2 list-findings \
  --filter-criteria '{"severity": [{"comparison": "EQUALS", "value": "CRITICAL"}]}' \
  --query 'findings[*].{CVE:packageVulnerabilityDetails.vulnerabilityId,Score:inspectorScore,Resource:resources[0].id,Status:status}' \
  --output table
```

---

## Advantages

1. **Continuous, Automated Scanning**: Unlike scheduled or point-in-time scans, Inspector v2 continuously monitors for new vulnerabilities as CVEs are published — zero manual intervention required.

2. **Agentless EC2 Scanning**: No SSM Agent? No problem. Inspector can scan EC2 instances via EBS snapshots, removing the dependency on agent installation.

3. **Contextualized Risk Scoring**: The Amazon Inspector Risk Score goes beyond raw CVSS by incorporating network reachability and exploit maturity, enabling true risk-based prioritization.

4. **Native AWS Integration**: Deep integration with ECR (image push triggers scan), Lambda (automatic function scanning), Organizations (multi-account), Security Hub, and EventBridge.

5. **Unified Findings View**: A single pane of glass for vulnerabilities across EC2, containers, and Lambda — eliminating tool sprawl.

6. **Shift-Left Security**: ECR scanning enables vulnerability detection before deployment, integrating naturally into CI/CD pipelines.

7. **No Infrastructure to Manage**: Fully managed service — no scanning servers, no database maintenance, no agent fleet management (for agentless mode).

8. **Compliance-Ready Reporting**: Built-in reporting (CSV, JSON) and Security Hub integration provide audit-ready evidence for PCI-DSS, HIPAA, SOC 2, and more.

9. **Cost-Effective**: Pay-per-scan model with no upfront costs; much cheaper than maintaining a commercial vulnerability scanner like Qualys or Tenable.

10. **Suppression Rules**: Ability to suppress known false positives or accepted risks, reducing alert fatigue.

---

## Limitations

### Functional Limitations

| Limitation | Detail |
|---|---|
| **EC2 OS Support** | Limited to supported operating systems: Amazon Linux, Ubuntu, Debian, RHEL, CentOS, Windows Server. Non-standard OSes may have incomplete scanning. |
| **SSM Agent Requirement** | Agent-based EC2 scanning requires SSM Agent v3.1.1400+ and SSM managed instance. Instances without SSM require agentless mode. |
| **Lambda Runtime Support** | Code vulnerability scanning supports only Python and Java runtimes. Node.js, Go, .NET not supported for code scanning (package scanning is supported). |
| **Network Reachability Scope** | Network reachability analysis only covers VPC-level configurations (SGs, NACLs, IGW, NAT). Does not analyze WAF, CloudFront, or application-layer controls. |
| **No Penetration Testing** | Inspector is a vulnerability assessment tool, not a penetration testing tool. It identifies known CVEs but doesn't exploit them. |
| **ECR Private Only** | Inspector scans only private ECR repositories, not public ones. |

### Service Quotas (Default, Adjustable)

| Quota | Default Limit |
|---|---|
| Maximum suppression rules per account | 200 |
| Maximum filter criteria per API call | 10 |
| Maximum findings returned per API call | 100 |
| Maximum accounts per delegated administrator | 5,000 |
| CIS scan configurations per account | 10 |

### Regional Limitations
- Inspector v2 is available in most AWS regions but not all. Always verify regional availability before architecture decisions.
- Findings are regional — cross-region aggregation requires Security Hub or custom solutions.

---

## Best Practices

### 1. Enable via AWS Organizations
```bash
# Enable Inspector organization-wide from the delegated admin account
aws inspector2 enable-organization-admin-account \
  --admin-account-id 123456789012
```
Ensures all current and future member accounts are automatically enrolled.

### 2. Designate a Delegated Administrator
Assign a dedicated **Security** or **Audit** account as the Inspector delegated administrator. Never use