# Inspector — Interview Questions

## Easy

### Q1: What is AWS Inspector and what is its primary purpose?

**Answer:**
AWS Inspector is an automated vulnerability management service that continuously scans AWS workloads for software vulnerabilities and unintended network exposure. Its primary purpose is to automatically discover and report security findings across EC2 instances, container images in Amazon ECR, and AWS Lambda functions. Inspector assigns a risk score to each finding, prioritizes them, and integrates with AWS Security Hub and EventBridge for centralized management and automated remediation workflows.

---

### Q2: What types of resources does AWS Inspector v2 support scanning?

**Answer:**
AWS Inspector v2 supports scanning the following resource types:
- **Amazon EC2 instances** — scans for OS-level package vulnerabilities and network reachability issues
- **Amazon ECR container images** — scans container images for OS and programming language package vulnerabilities
- **AWS Lambda functions** — scans Lambda function code and application dependencies for software vulnerabilities

Note: Inspector v2 (launched 2021) is a significant redesign from Inspector Classic (v1), which only supported EC2 instances and required manual assessment runs.

---

### Q3: What is a "finding" in AWS Inspector, and how is it scored?

**Answer:**
A **finding** in AWS Inspector is a potential security issue or vulnerability discovered during a scan. Each finding includes:
- A description of the vulnerability
- The affected resource
- The severity level (Critical, High, Medium, Low, Informational)
- An **Amazon Inspector Score** — a contextual risk score (0.0–10.0) derived from the CVSS base score adjusted for environmental context such as network reachability and exploitability
- Remediation recommendations

The scoring helps teams prioritize which vulnerabilities to address first rather than treating all findings equally.

---

### Q4: What is the difference between AWS Inspector v1 (Classic) and AWS Inspector v2?

**Answer:**

| Feature | Inspector v1 (Classic) | Inspector v2 |
|---|---|---|
| Scan trigger | Manual assessment runs | Continuous, automated |
| Resources | EC2 only | EC2, ECR, Lambda |
| Agent requirement | Required (SSM or Inspector agent) | Uses SSM Agent (no separate agent) |
| CVE database | Limited | Continuously updated CVE feeds |
| Multi-account | Limited | Native AWS Organizations integration |
| Pricing | Per assessment | Per instance/image scanned |

Inspector v2 is the recommended service and Inspector Classic is no longer accepting new customers.

---

### Q5: How does AWS Inspector integrate with AWS Organizations?

**Answer:**
AWS Inspector v2 supports centralized management through AWS Organizations. A **delegated administrator account** can be designated to manage Inspector across all member accounts in the organization. This allows:
- Enabling Inspector across all existing and new member accounts automatically
- Viewing aggregated findings from a central account
- Managing suppression rules and filters centrally
- Generating organization-wide reports

This eliminates the need to configure Inspector individually in each account, making it suitable for enterprise-scale deployments.

---

## Medium

### Q1: How does AWS Inspector perform EC2 instance scanning, and what are the prerequisites?

**Answer:**
AWS Inspector v2 scans EC2 instances using **agentless scanning** (for network reachability) and **agent-based scanning** (for package vulnerabilities via AWS Systems Manager).

**Prerequisites for EC2 scanning:**
1. **AWS SSM Agent** must be installed and running on the EC2 instance (version 3.1.1767.0 or later recommended)
2. The instance must have an **IAM instance profile** with the `AmazonSSMManagedInstanceCore` managed policy
3. The instance must be able to communicate with SSM endpoints (either via internet or VPC endpoints)
4. The instance must be in a **managed instance** state within SSM

**What Inspector scans:**
- **Software vulnerability scanning:** Inspector queries the SSM agent to inventory installed packages and compares them against CVE databases (NVD, vendor advisories)
- **Network reachability:** Inspector analyzes VPC configurations, security groups, NACLs, route tables, and internet gateways to determine if ports are reachable from the internet or other VPCs — without sending actual network traffic

**Scan frequency:**
- Initial scan occurs when Inspector is enabled
- Re-scans are triggered by new CVE releases or changes to the instance (new packages installed, security group changes)

---

### Q2: Explain how AWS Inspector handles container image scanning in ECR and what the scan-on-push vs. continuous scanning difference means.

**Answer:**
AWS Inspector v2 integrates with Amazon ECR to provide enhanced container image scanning.

**Two scanning modes:**

**1. Scan on Push (Basic Scanning — native ECR):**
- Scans an image once when it is pushed to the repository
- Uses the open-source Clair scanner
- Results are available in the ECR console
- Does NOT re-scan for newly discovered CVEs after the initial push

**2. Enhanced Scanning (Inspector v2):**
- Continuously monitors images for new vulnerabilities even after they are pushed
- Uses Inspector's continuously updated CVE intelligence feeds
- Covers both OS-level packages (e.g., apt, yum packages) and programming language packages (e.g., npm, pip, Maven, NuGet, Composer)
- Findings appear in both the ECR console and the Inspector console
- Re-scans are triggered automatically when new CVEs are published that affect packages in the image

**Configuration:**
- Enhanced scanning is configured at the ECR registry or repository level
- Inspector must be enabled in the account
- You can set scanning frequency: `CONTINUOUS_SCAN` or `SCAN_ON_PUSH`

**Key advantage:** Enhanced scanning catches vulnerabilities discovered after an image was built, which is critical because images may sit in a registry for months before deployment.

---

### Q3: What is the Amazon Inspector Score and how does it differ from the CVSS score?

**Answer:**
Both scores measure vulnerability severity, but they serve different purposes:

**CVSS Score (Common Vulnerability Scoring System):**
- Industry-standard score (0.0–10.0) from NVD or vendor advisories
- Based purely on the vulnerability's intrinsic characteristics: attack vector, complexity, privileges required, user interaction, scope, and impact
- Does NOT consider your specific environment
- A CVSS 9.8 vulnerability on an isolated internal instance is treated the same as one on an internet-facing server

**Amazon Inspector Score:**
- Also 0.0–10.0, contextually adjusted
- Starts with the CVSS base score
- **Adjusts upward** if the affected resource is network-reachable from the internet
- **Adjusts upward** if there is known exploit code available (EPSS — Exploit Prediction Scoring System integration)
- **Adjusts downward** if the resource is isolated or the vulnerability's attack vector doesn't apply in your environment

**Practical example:**
- A CVE with CVSS 7.5 on an EC2 instance with a public IP and port 22 open might get an Inspector Score of 9.2
- The same CVE on an instance in a private subnet with no internet route might score 5.1

This contextual scoring helps security teams focus remediation efforts on the vulnerabilities that pose the greatest **actual** risk in their environment, rather than treating all high-CVSS vulnerabilities equally.

---

### Q4: How do suppression rules work in AWS Inspector, and when should you use them?

**Answer:**
**Suppression rules** (also called filter rules) allow you to automatically suppress findings that match specific criteria, preventing them from appearing in active findings reports. Suppressed findings are retained but marked as suppressed.

**Rule criteria you can filter on:**
- Finding title or description
- Severity level
- Resource tags
- Resource type (EC2, ECR, Lambda)
- CVE ID
- AWS account ID
- Inspector score range
- First/last observed date

**When to use suppression rules:**

1. **Accepted risk:** A vulnerability exists in a library you use, but the specific vulnerable code path is not reachable in your application
2. **Compensating controls:** You have other security controls that mitigate the risk (e.g., WAF rules blocking the exploit)
3. **False positives:** Inspector flags a finding that doesn't apply to your configuration
4. **Temporary suppression:** A patch isn't available yet and you've accepted the risk until a fix is released
5. **Dev/test environments:** Suppress lower-severity findings in non-production environments to focus on production

**Important considerations:**
- Suppression rules should be documented with justification (for audit purposes)
- Rules can be scoped to specific accounts or tags to avoid over-suppression
- In Organizations mode, the delegated admin can create organization-wide suppression rules
- Suppressed findings still count toward your overall finding inventory but are excluded from active dashboards

---

### Q5: How does AWS Inspector integrate with AWS Security Hub, and what are the benefits of this integration?

**Answer:**
AWS Inspector automatically sends findings to AWS Security Hub when both services are enabled in the same account/region. This integration provides several benefits:

**How the integration works:**
1. Inspector generates a finding
2. Finding is automatically forwarded to Security Hub in **ASFF (Amazon Security Finding Format)**
3. Security Hub normalizes and stores the finding alongside findings from other services (GuardDuty, Macie, Config, etc.)
4. No additional configuration required — it's automatic when both services are active

**Benefits:**

**1. Centralized visibility:**
- Single pane of glass for all security findings across Inspector, GuardDuty, Macie, and third-party tools
- Reduces context switching between consoles

**2. Cross-service correlation:**
- Security Hub can correlate Inspector vulnerability findings with GuardDuty threat detections
- Example: Inspector finds a critical CVE on an EC2 instance AND GuardDuty detects suspicious outbound traffic from the same instance → high-priority incident

**3. Automated workflows:**
- Security Hub custom actions can trigger Lambda functions or Step Functions for automated remediation
- EventBridge rules on Security Hub findings can route to ticketing systems (Jira, ServiceNow)

**4. Compliance standards:**
- Inspector findings contribute to Security Hub compliance checks (AWS Foundational Security Best Practices, CIS)

**5. Aggregation across regions:**
- Security Hub's finding aggregation feature can collect Inspector findings from multiple regions into a single region

**6. Suppression sync:**
- Findings suppressed in Inspector can also be updated in Security Hub

---

## Hard

### Q1: Describe the architecture of AWS Inspector v2's vulnerability intelligence pipeline and how it maintains up-to-date CVE coverage.

**Answer:**
AWS Inspector v2's vulnerability intelligence pipeline is a sophisticated, continuously operating system that aggregates, normalizes, and correlates vulnerability data from multiple sources.

**Data Sources:**
1. **National Vulnerability Database (NVD)** — NIST-maintained CVE repository
2. **Vendor Security Advisories** — OS vendor bulletins (Amazon Linux, Ubuntu, RHEL, Debian, Windows)
3. **Language ecosystem advisories** — GitHub Advisory Database, PyPI, npm, Maven Central, RubyGems, NuGet, Packagist
4. **EPSS (Exploit Prediction Scoring System)** — ML-based probability scores for exploitation likelihood
5. **CISA KEV (Known Exploited Vulnerabilities catalog)** — actively exploited vulnerabilities
6. **Proprietary threat intelligence** — AWS internal threat research

**Pipeline stages:**

```
[Source Feeds] → [Ingestion Layer] → [Normalization] → [Enrichment] → [CPE Matching] → [Finding Generation]
```

1. **Ingestion:** Continuous polling and webhook-based ingestion from all sources
2. **Normalization:** Converting disparate formats into a unified vulnerability schema
3. **Enrichment:** Adding EPSS scores, CISA KEV flags, vendor patch availability
4. **CPE (Common Platform Enumeration) Matching:** Mapping CVEs to specific software versions using CPE dictionaries
5. **Package inventory correlation:** Matching CPE data against the software inventory collected from your resources via SSM
6. **Finding generation:** Creating or updating findings when new CVEs match installed packages

**Continuous re-scanning trigger model:**
- **New CVE published** → Inspector re-evaluates all inventoried packages across all accounts
- **Package change detected** (new software installed/removed) → Re-scan triggered for that resource
- **Network configuration change** (security group modified) → Network reachability re-analysis triggered
- **Scheduled full re-scan** — periodic comprehensive re-scan as a safety net

**Scale considerations:**
- Inspector operates across millions of AWS customer resources simultaneously
- The CVE matching is done server-side using the package inventory (not by running scans on instances), which is why it's nearly real-time
- Package inventory is maintained by SSM, which polls instances on a configurable schedule (default: every 30 minutes)

**Gap:** Inspector's coverage depends on the quality of CPE data. Some vulnerabilities lack precise CPE identifiers, which can lead to missed detections or false positives. AWS continuously improves CPE mapping accuracy.

---

### Q2: How would you design a fully automated vulnerability remediation pipeline using AWS Inspector, and what are the key technical challenges?

**Answer:**
A fully automated vulnerability remediation pipeline requires careful design to balance speed with safety. Here's an enterprise-grade architecture:

**Architecture Components:**

```
Inspector Finding
      ↓
EventBridge Rule (filter by severity/resource type)
      ↓
Step Functions State Machine
      ├── Validate finding (not suppressed, not already patched)
      ├── Check change freeze calendar (SSM Change Calendar)
      ├── Identify patch (SSM Patch Manager baseline lookup)
      ├── Create change record (ServiceNow/Jira via Lambda)
      ├── Notify approver (SNS → Slack/Email) [for Critical]
      ├── Approval gate (wait for human approval for Critical)
      ├── Pre-patch snapshot (EC2 AMI snapshot via Lambda)
      ├── Apply patch (SSM Run Command → yum/apt update)
      ├── Verify patch applied (SSM inventory re-scan)
      ├── Validate Inspector finding resolved
      └── Rollback on failure (restore from snapshot)
```

**Detailed Flow:**

**1. Finding Ingestion:**
```json
// EventBridge pattern for critical EC2 findings
{
  "source": ["aws.inspector2"],
  "detail-type": ["Inspector2 Finding"],
  "detail": {
    "severity": ["CRITICAL", "HIGH"],
    "type": ["PACKAGE_VULNERABILITY"],
    "resources": {
      "type": ["AWS_EC2_INSTANCE"]
    }
  }
}
```

**2. Deduplication and enrichment (Lambda):**
- Check DynamoDB for existing remediation record for this CVE + instance combination
- Enrich with instance metadata (environment tag, criticality tag, owner tag)
- Determine automation eligibility (is auto-patching enabled for this instance?)

**3. Patch identification:**
- Query SSM Patch Manager to verify the patch is in the approved baseline
- Check if patch requires reboot (critical for production)
- Estimate patch window using instance tags (e.g., `patch-window: weekend`)

**4. Safety gates:**
- **Change freeze check:** SSM Change Calendar integration to block patching during freeze windows
- **Blast radius limit:** Maximum N instances patched simultaneously (configurable per environment)
- **Canary approach:** Patch one instance first, validate, then proceed with others
- **Human approval:** Required for Critical severity in production environments

**5. Patch execution:**
- SSM Run Command with `AWS-RunPatchBaseline` document
- Patch applied in maintenance window or immediately based on severity
- CloudWatch Logs capture all output

**6. Validation:**
- Wait for SSM inventory to update (15–30 minutes)
- Query Inspector API to confirm finding status changed to `CLOSED`
- If finding persists, escalate to human review

**Key Technical Challenges:**

1. **False closure:** Inspector may not immediately re-scan after patching. Solution: Force SSM inventory sync via `aws ssm send-command --document-name "AWS-GatherSoftwareInventory"`

2. **Reboot-required patches:** Some patches require instance restart. Solution: Tag-based scheduling with maintenance windows, pre-notify application owners

3. **Dependency conflicts:** Patching one package may break another. Solution: Test in staging first using AMI-based blue/green approach

4. **Container images:** Patching running containers is not possible; you must rebuild the image. Solution: Trigger CI/CD pipeline to rebuild and redeploy the image with updated base layer

5. **Lambda functions:** Require code deployment to update dependencies. Solution: Trigger CodePipeline to update the Lambda deployment package

6. **Rate limiting:** Inspector API has rate limits; use exponential backoff and SQS for buffering

7. **Patch availability:** Not all CVEs have available patches. Solution: Filter automation to only CVEs with `fixAvailable: true` in the Inspector finding

---

### Q3: Explain how AWS Inspector's network reachability analysis works technically, and what are its limitations compared to actual penetration testing?

**Answer