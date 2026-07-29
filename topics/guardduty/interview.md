# GuardDuty — Interview Questions

---

## Easy

---

**Q1. What is Amazon GuardDuty, and what is its primary purpose?**

**Answer:**
Amazon GuardDuty is a managed threat detection service that continuously monitors AWS accounts and workloads for malicious activity and unauthorized behavior. It uses machine learning, anomaly detection, and integrated threat intelligence (from AWS, CrowdStrike, and Proofpoint) to identify threats such as compromised credentials, reconnaissance activity, and data exfiltration. Its primary purpose is to provide intelligent and continuous security monitoring without requiring you to deploy or manage additional security infrastructure.

---

**Q2. What are the primary data sources that GuardDuty analyzes?**

**Answer:**
GuardDuty analyzes the following core data sources:

- **AWS CloudTrail Event Logs** – Management events and, optionally, data events (S3 and Lambda)
- **VPC Flow Logs** – Network traffic metadata within your VPC
- **DNS Logs** – DNS query logs from Route 53 Resolver (only available in GuardDuty; not directly accessible by customers)
- **AWS CloudTrail S3 Data Events** – Object-level API activity on S3 buckets
- **Kubernetes Audit Logs** – API server activity for EKS clusters (EKS Protection)
- **Amazon EBS Volumes** – Malware scanning for EC2 and container workloads (Malware Protection)
- **RDS Login Activity** – Authentication attempts for Aurora databases (RDS Protection)
- **Lambda Network Activity** – Network communications made by Lambda functions (Lambda Protection)

---

**Q3. How does GuardDuty classify its findings?**

**Answer:**
GuardDuty classifies findings into three severity levels:

| Severity | Score Range | Description |
|---|---|---|
| **High** | 7.0 – 8.9 | Immediate action required; resource is likely compromised |
| **Medium** | 4.0 – 6.9 | Suspicious activity that should be investigated |
| **Low** | 0.1 – 3.9 | Unusual activity with low risk; monitor and review |

Findings are also categorized by **finding type**, which follows the format:
`ThreatPurpose:ResourceTypeAffected/ThreatFamilyName.DetectionMechanism!Artifact`

Example: `UnauthorizedAccess:EC2/SSHBruteForce`

---

**Q4. Does GuardDuty require you to enable VPC Flow Logs or CloudTrail manually before using it?**

**Answer:**
No. GuardDuty does not require you to independently enable or configure VPC Flow Logs, DNS Logs, or CloudTrail. It processes these logs independently through its own mechanisms without affecting your existing logging configurations or incurring additional costs for those log sources. GuardDuty accesses the data directly from AWS infrastructure. However, for optional protection plans like EKS Protection, Malware Protection, RDS Protection, and Lambda Protection, you may need to enable specific features within GuardDuty itself.

---

**Q5. What is the difference between GuardDuty and AWS Security Hub?**

**Answer:**

| Feature | GuardDuty | Security Hub |
|---|---|---|
| **Primary Role** | Threat detection and finding generation | Security posture management and finding aggregation |
| **Finding Source** | Generates its own findings | Aggregates findings from multiple services (GuardDuty, Inspector, Macie, etc.) |
| **Standards/Benchmarks** | Not applicable | Supports CIS, PCI DSS, AWS Foundational Security Best Practices |
| **Remediation** | Does not remediate | Provides a centralized view; integrates with automation |
| **Relationship** | Can send findings TO Security Hub | Acts as a central dashboard |

In short, GuardDuty **detects threats**, while Security Hub **consolidates and manages** findings from GuardDuty and other services.

---

## Medium

---

**Q1. How does GuardDuty handle multi-account environments, and what is the recommended setup?**

**Answer:**
GuardDuty supports multi-account management through two models:

**1. AWS Organizations Integration (Recommended):**
- A **Delegated Administrator** account (typically a dedicated security account) manages GuardDuty for all member accounts in the organization.
- The delegated administrator can auto-enable GuardDuty for new accounts as they join the organization.
- It can centrally view and manage all findings across all member accounts.
- Member accounts cannot disable GuardDuty if the administrator has enabled it.

**2. Invitation-Based (Legacy):**
- A **Master account** sends invitations to member accounts.
- Member accounts must accept the invitation.
- Less scalable and harder to manage than Organizations integration.

**Best Practices:**
- Use AWS Organizations with a dedicated security/audit account as the delegated administrator.
- Enable `Auto-enable` for new accounts to ensure consistent coverage.
- Aggregate findings in the administrator account for centralized analysis.
- Use Security Hub to further aggregate and normalize findings across regions.

---

**Q2. Explain GuardDuty's Malware Protection feature. How does it work, and what are its limitations?**

**Answer:**
**Malware Protection** enables GuardDuty to scan Amazon EBS volumes attached to EC2 instances and container workloads for malware without deploying an agent.

**How it works:**
1. GuardDuty detects a suspicious finding related to an EC2 instance or container (e.g., `CryptoCurrency:EC2/BitcoinTool.B!DNS`).
2. It automatically triggers a malware scan by creating a snapshot of the EBS volume.
3. The snapshot is replicated into a GuardDuty-managed AWS account.
4. GuardDuty mounts and scans the volume using threat intelligence and signature-based detection.
5. Results are published as a new finding type: `Malware:EC2/MaliciousFile` or similar.
6. The snapshot in the GuardDuty account is deleted after scanning.

**On-Demand Scanning:**
You can also initiate malware scans manually without a GuardDuty finding trigger.

**Limitations:**
- Only scans EBS volumes; does not scan instance memory or network traffic in real time.
- Encrypted volumes using customer-managed KMS keys require explicit key policy grants to GuardDuty.
- Maximum EBS volume size supported is **1 TB** per volume.
- Scan frequency is limited (one scan per finding per 24-hour period for auto-triggered scans).
- Does not replace a host-based IDS/EDR solution for real-time file system monitoring.

---

**Q3. What are GuardDuty Suppression Rules, and when should you use them?**

**Answer:**
**Suppression Rules** allow you to automatically filter out findings that match specific criteria, preventing them from appearing in the active findings list. Suppressed findings are still logged to S3 (if configured) but are marked as `ARCHIVED` and do not trigger EventBridge events.

**Use Cases:**
- **Known good activity:** A security scanner or penetration testing tool generates `Recon:EC2/PortProbeUnprotectedPort` findings on a known IP range.
- **Bastion host SSH activity:** An EC2 bastion host generates `UnauthorizedAccess:EC2/SSHBruteForce` because it receives many SSH attempts, which is expected.
- **Internal services:** Internal microservices making expected API calls that trigger low-severity findings.

**How to create a Suppression Rule:**
1. Navigate to GuardDuty → Findings → Create Suppression Rule.
2. Define filter criteria (finding type, resource tags, IP address, account ID, etc.).
3. Save the rule; matching future findings are automatically archived.

**Important Considerations:**
- Use suppression rules carefully — suppressing too broadly can create blind spots.
- Prefer using **trusted IP lists** (for known safe IPs) over suppression rules when possible.
- Regularly audit suppression rules to ensure they remain valid.
- Suppressed findings still count toward your GuardDuty usage/billing.

---

**Q4. How does GuardDuty EKS Protection work, and what threats does it detect?**

**Answer:**
**EKS Protection** extends GuardDuty's threat detection to Amazon EKS clusters by analyzing **Kubernetes audit logs** and **EKS runtime activity**.

**Two Components:**

**1. EKS Audit Log Monitoring:**
- Analyzes Kubernetes control plane audit logs.
- Detects threats such as:
  - `Policy:Kubernetes/AdminAccessToDefaultServiceAccount` – Binding admin privileges to default service accounts.
  - `CredentialAccess:Kubernetes/MaliciousIPCaller` – API calls from known malicious IPs.
  - `Execution:Kubernetes/ExecInKubeSystemPod` – Exec commands into kube-system namespace pods.
  - `Persistence:Kubernetes/ContainerWithSensitiveMount` – Containers mounting sensitive host paths.

**2. EKS Runtime Monitoring:**
- Deploys a lightweight **GuardDuty security agent** (as a DaemonSet) onto EKS nodes.
- Monitors runtime behavior at the OS level (system calls, file access, network connections).
- Detects threats such as:
  - Cryptomining activity within containers.
  - Reverse shell connections from containers.
  - Privilege escalation attempts.
  - Malicious file execution.

**Setup:**
- Enable EKS Protection in GuardDuty console or via API.
- For Runtime Monitoring, you can use **managed agent deployment** (GuardDuty auto-deploys the DaemonSet) or manual deployment.
- Ensure EKS cluster has appropriate IAM permissions.

---

**Q5. How can you automate remediation of GuardDuty findings? Describe a complete workflow.**

**Answer:**
GuardDuty integrates with Amazon EventBridge to enable automated remediation workflows.

**Complete Workflow:**

```
GuardDuty Finding → EventBridge Rule → Target (Lambda/SNS/Step Functions) → Remediation Action
```

**Step-by-Step Implementation:**

1. **GuardDuty generates a finding** (e.g., `UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B` — login from unusual location).

2. **EventBridge Rule** is configured to match:
   ```json
   {
     "source": ["aws.guardduty"],
     "detail-type": ["GuardDuty Finding"],
     "detail": {
       "severity": [{ "numeric": [">=", 7] }]
     }
   }
   ```

3. **Lambda Function** is triggered and performs:
   - Parses the finding details (affected IAM user, source IP, resource ARN).
   - Calls `iam:AttachUserPolicy` to attach a **deny-all policy** to the compromised IAM user.
   - Creates a **WAF IP block rule** if the source IP is identified.
   - Sends a notification via **SNS** to the security team.
   - Creates a ticket in **Jira/ServiceNow** via API.
   - Logs the action to **DynamoDB** for audit trail.

4. **Step Functions** (for complex workflows):
   - Orchestrates multi-step remediation with approval gates.
   - Human approval via SNS before taking destructive actions.

5. **Security Hub** aggregates the finding and tracks remediation status.

**Best Practices:**
- Test automation in non-production environments first.
- Implement safeguards to avoid remediating false positives (e.g., require human approval for account-level actions).
- Use dead-letter queues (DLQ) on Lambda to handle failures.
- Tag resources with `do-not-auto-remediate` for known exceptions.

---

## Hard

---

**Q1. Explain the machine learning models GuardDuty uses and how it establishes behavioral baselines. What are the implications for newly enabled accounts?**

**Answer:**
GuardDuty employs multiple ML and statistical techniques to detect anomalies:

**Machine Learning Techniques:**

1. **Anomaly Detection (Unsupervised ML):**
   - Builds behavioral baselines for IAM entities (users, roles) based on historical CloudTrail activity.
   - Tracks dimensions like: API calls made, source IPs, geographic locations, time of day, user agents, and services accessed.
   - Flags deviations from established baselines (e.g., `Anomalous:IAMUser/AnomalousBehavior`).

2. **Neural Networks for DNS Analysis:**
   - Identifies domain generation algorithm (DGA) patterns used by malware for C2 communication.
   - Detects DNS tunneling by analyzing query patterns, response sizes, and entropy.

3. **Threat Intelligence Integration:**
   - Compares observed IPs, domains, and hashes against curated threat intel feeds (AWS, CrowdStrike, Proofpoint).
   - Rule-based matching for known-bad indicators.

4. **Sequence Models:**
   - Detects multi-step attack patterns (e.g., reconnaissance followed by exploitation).

**Baseline Establishment Period:**
- GuardDuty requires approximately **7–14 days** to establish behavioral baselines for ML-based findings.
- During this period, anomaly-based findings may not fire or may have lower accuracy.
- Rule-based and threat-intelligence-based findings are active **immediately** upon enablement.

**Implications for Newly Enabled Accounts:**
- **False Negatives:** Anomaly-based detections won't fire during the learning period, creating a temporary blind spot.
- **False Positives:** After the baseline is established, any unusual activity (even legitimate) may trigger findings until the model adapts.
- **Mitigation:**
  - Enable GuardDuty proactively, well before a security incident.
  - Use trusted IP lists to reduce noise during the baseline period.
  - Supplement with rule-based tools (CloudTrail Insights, Config Rules) during the learning period.
  - In AWS Organizations, enable GuardDuty for new accounts immediately upon account creation using auto-enable.

**Important Note:** Each AWS Region maintains its own independent baseline, so enabling GuardDuty in a new region restarts the learning period for that region.

---

**Q2. A GuardDuty finding `Stealth:S3/ServerAccessLoggingDisabled` fires repeatedly despite a valid business reason. How would you architect a comprehensive false positive management strategy without creating security blind spots?**

**Answer:**
This is a nuanced problem requiring a layered approach that balances noise reduction with security integrity.

**Root Cause Analysis First:**
- Identify which S3 buckets are triggering the finding.
- Determine if disabling server access logging is truly intentional (cost optimization, compliance, etc.) or a misconfiguration.
- Validate with the bucket owners and document the business justification.

**Strategy 1: Trusted IP Lists (Not Applicable Here)**
- Trusted IP lists apply to network-based findings, not S3 configuration findings. Not suitable for this case.

**Strategy 2: Suppression Rules (Targeted)**
```
Finding Type: Stealth:S3/ServerAccessLoggingDisabled
AND
Resource Tag: logging-exempt = true
AND
Resource Tag: exception-approved-by = security-team
```
- Only suppress findings for explicitly tagged, approved buckets.
- Require a tagging governance process (SCP to enforce tag presence before suppression applies).

**Strategy 3: Compensating Controls (Preferred)**
Instead of suppressing, implement compensating controls to maintain security visibility:

1. **Enable S3 Data Events in CloudTrail** – Ensures object-level activity is still captured even without server access logging.
2. **Enable AWS Config Rule** `s3-bucket-logging-enabled` – Continuously evaluates and alerts on compliance drift.
3. **Enable Macie** – Provides data classification and anomaly detection independent of server access logs.
4. **VPC Flow Logs** – Captures network-level access to S3 from within VPC (via VPC endpoints).

**Strategy 4: Finding Workflow Integration**
1. Route GuardDuty findings to Security Hub.
2. In Security Hub, create a custom workflow status: `SUPPRESSED` with a note referencing the approved exception ticket.
3. Set a **review cadence** (e.g., quarterly) to re-evaluate all suppressed findings.
4. Store exception approvals in a DynamoDB table with TTL, automatically expiring exceptions after 90 days.

**Strategy 5: Governance Controls**
- Use AWS Config Conformance Packs to enforce S3 logging as a baseline.
- Use SCPs to prevent disabling logging on buckets tagged `data-classification = confidential`.

**Anti-Pattern to Avoid:**
Never create a broad suppression rule like `Finding Type: Stealth:S3/*` — this would suppress legitimate stealth findings including `Stealth:S3/BucketPolicyChange`.

---