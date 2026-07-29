# GuardDuty

## What is it?

**Amazon GuardDuty** is a fully managed, intelligent threat detection service that continuously monitors AWS accounts, workloads, and data stored in Amazon S3 for malicious activity and unauthorized behavior. It falls under the **AWS Security, Identity, and Compliance** category.

GuardDuty uses machine learning, anomaly detection, and integrated threat intelligence (from AWS, CrowdStrike, and Proofpoint) to identify threats such as compromised EC2 instances, unusual API calls, potential account compromises, and malicious reconnaissance. It operates without requiring any software agents, sensors, or network appliances, analyzing data sources like AWS CloudTrail event logs, VPC Flow Logs, and DNS logs automatically.

**Key identifiers:**
- Service type: Managed Security / Threat Detection (IDS/IPS-like)
- Region-scoped but with multi-account/multi-region aggregation capability
- Produces **findings** — structured security alerts with severity levels (Low, Medium, High)
- Integrates natively with AWS Security Hub, Amazon EventBridge, and AWS Organizations

---

## Why do we need it?

### The Problem
Modern AWS environments are complex, distributed, and constantly changing. Traditional security tools require:
- Manual log analysis across CloudTrail, VPC Flow Logs, and DNS logs (terabytes of data)
- Constant signature updates for known threats
- Dedicated security engineers to monitor 24/7
- Complex SIEM integrations with high operational overhead

Without automated threat detection, organizations face:
- **Delayed incident response** — attackers may operate for weeks before detection
- **Credential compromise** — stolen IAM keys used for crypto-mining, data exfiltration
- **Insider threats** — legitimate users performing unauthorized actions
- **Compliance failures** — inability to demonstrate continuous monitoring (PCI DSS, HIPAA, SOC2)

### When to Use It
| Scenario | Why GuardDuty |
|---|---|
| Multi-account AWS Organizations | Centralized threat detection across all accounts |
| Regulated industries (Finance, Healthcare) | Continuous compliance monitoring |
| EC2/EKS/ECS workloads | Runtime threat detection without agents |
| S3 data lakes | Detect unusual data access patterns |
| Serverless architectures | Lambda function threat detection |
| DevOps/CI-CD pipelines | Detect compromised build credentials |

### Real Business Scenarios
1. **Crypto-jacking detection**: An attacker compromises an EC2 instance and starts mining cryptocurrency. GuardDuty detects the unusual outbound traffic to known crypto-mining domains and triggers an alert.
2. **Credential theft**: A developer accidentally commits IAM access keys to a public GitHub repository. GuardDuty detects API calls from an unusual geographic location and flags the finding.
3. **Data exfiltration**: An insider starts downloading massive amounts of S3 objects. GuardDuty's S3 protection detects the anomalous access pattern.
4. **Ransomware preparation**: An attacker scans EC2 instances for open ports. GuardDuty detects the port scanning activity.

---

## Internal Working

GuardDuty operates as a **passive, out-of-band threat detection system**. It does not sit in the data path — it analyzes copies of log data and metadata rather than intercepting traffic.

### Data Ingestion Pipeline

```
AWS Data Sources                    GuardDuty Processing Engine
─────────────────                   ────────────────────────────
CloudTrail Events  ──────────────►  Log Normalization & Parsing
VPC Flow Logs      ──────────────►  Feature Extraction
DNS Logs           ──────────────►  ML Model Inference
EKS Audit Logs     ──────────────►  Threat Intel Matching
S3 Data Events     ──────────────►  Anomaly Scoring
Lambda Logs        ──────────────►  Finding Generation
RDS Login Events   ──────────────►  Deduplication
EBS Volume Data    ──────────────►  Severity Assignment
Runtime Activity   ──────────────►  Finding Publishing
```

### Detection Mechanisms

**1. Threat Intelligence Feeds**
GuardDuty maintains continuously updated lists of:
- Known malicious IP addresses
- Malicious domains (C2 servers, phishing, crypto-mining pools)
- Tor exit nodes
- Anonymizing proxies
- Sources: AWS Security, CrowdStrike, Proofpoint

**2. Machine Learning Models**
- **Behavioral baselines**: GuardDuty learns normal behavior patterns for each AWS account over 2 weeks
- **Anomaly detection**: Deviations from baseline trigger findings (e.g., API calls from new geography)
- **Sequence models**: Detect multi-step attack patterns (reconnaissance → exploitation → exfiltration)
- **Neural networks**: Used for malware detection in EBS volume scanning

**3. Rule-based Detection**
- Known attack signatures and patterns
- AWS-curated detection logic updated continuously
- Examples: `UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B` (console login from unusual location)

### Finding Lifecycle

```
1. Data Source Event Occurs
         │
         ▼
2. GuardDuty Ingests & Analyzes (near real-time, typically < 5 minutes)
         │
         ▼
3. Detection Engine Evaluates Against All Methods
         │
         ▼
4. Finding Generated with:
   - Finding Type (e.g., Backdoor:EC2/C&CActivity.B)
   - Severity (1-10 numeric, mapped to Low/Medium/High/Critical)
   - Affected Resource Details
   - Evidence (IPs, domains, API calls)
   - MITRE ATT&CK Mapping
         │
         ▼
5. Finding Stored in GuardDuty (90-day retention)
         │
         ▼
6. Published to EventBridge (near real-time)
         │
         ▼
7. Automated Response / Notification
```

### Finding Deduplication
GuardDuty aggregates similar findings within a **6-hour window** by default to prevent alert fatigue. The same finding type for the same resource is updated rather than creating duplicate findings.

---

## Architecture

### Core Architectural Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Organization                          │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Management/Delegated Admin Account          │   │
│  │                                                          │   │
│  │   ┌──────────────────────────────────────────────────┐   │   │
│  │   │           GuardDuty Administrator                │   │   │
│  │   │                                                  │   │   │
│  │   │  ┌─────────────┐    ┌────────────────────────┐  │   │   │
│  │   │  │  Findings   │    │   Threat Intel Lists   │  │   │   │
│  │   │  │  Aggregator │    │   (Custom + Managed)   │  │   │   │
│  │   │  └─────────────┘    └────────────────────────┘  │   │   │
│  │   └──────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │  Member Account │  │  Member Account │  │ Member Account │  │
│  │  (Region A)     │  │  (Region B)     │  │  (Region C)    │  │
│  │                 │  │                 │  │                │  │
│  │  GuardDuty      │  │  GuardDuty      │  │  GuardDuty     │  │
│  │  Detector       │  │  Detector       │  │  Detector      │  │
│  └─────────────────┘  └─────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Protection Plans (Feature Modules)

| Protection Plan | Data Source | What It Detects |
|---|---|---|
| **Foundational** (default) | CloudTrail Mgmt Events, VPC Flow Logs, DNS Logs | Account compromise, EC2 threats, network anomalies |
| **S3 Protection** | CloudTrail S3 Data Events | Unusual S3 access, data exfiltration |
| **EKS Protection** | EKS Audit Logs + Runtime Activity | Container escape, cryptomining in pods |
| **Malware Protection** | EBS Volume Snapshots | Malware, trojans, ransomware on EC2/ECS |
| **RDS Protection** | RDS Login Activity | Brute force, credential stuffing on databases |
| **Lambda Protection** | Lambda Network Activity | Compromised functions, C2 communication |
| **Runtime Monitoring** | OS-level events (agent-based) | Process injection, privilege escalation |

### Multi-Account Architecture

```
                    ┌─────────────────────────┐
                    │   Security/Audit Account  │
                    │   (Delegated Admin)       │
                    │                           │
                    │  ┌─────────────────────┐  │
                    │  │  GuardDuty Admin    │  │
                    │  │  - All findings     │  │
                    │  │  - Suppression rules│  │
                    │  │  - Trusted IPs      │  │
                    │  │  - Threat intel     │  │
                    │  └─────────────────────┘  │
                    └──────────┬──────────────┘
                               │ (Invites/Auto-enable)
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
    │ Member Acct 1  │ │Member Acct 2│ │Member Acct 3│
    │ Dev            │ │ Staging     │ │ Production  │
    │ GuardDuty      │ │ GuardDuty   │ │ GuardDuty   │
    │ Detector       │ │ Detector    │ │ Detector    │
    └────────────────┘ └─────────────┘ └─────────────┘
```

### Finding Response Architecture

```
GuardDuty Finding
      │
      ▼
Amazon EventBridge
      │
      ├──► SNS Topic ──► Email/PagerDuty/Slack
      │
      ├──► Lambda Function ──► Auto-remediation
      │         │               (Block IP, Isolate EC2,
      │         │                Revoke IAM credentials)
      │         ▼
      │    AWS Systems Manager
      │    (Run remediation playbooks)
      │
      ├──► AWS Security Hub ──► Centralized findings
      │
      ├──► S3 Bucket ──► Long-term storage/SIEM
      │
      └──► Kinesis Data Firehose ──► Splunk/Datadog
```

---

## Real World Example

### Scenario: Detecting and Responding to a Compromised EC2 Instance

**Context**: A fintech company runs a payment processing application on EC2 instances. An attacker exploits a vulnerability in a web application and gains shell access to an EC2 instance.

#### Step 1: GuardDuty Detects Malicious Activity

The compromised EC2 instance starts communicating with a known C2 (Command and Control) server.

GuardDuty generates finding:
```json
{
  "schemaVersion": "2.0",
  "accountId": "123456789012",
  "region": "us-east-1",
  "type": "Backdoor:EC2/C&CActivity.B!DNS",
  "severity": 8.0,
  "title": "EC2 instance is communicating with a known command and control server",
  "description": "EC2 instance i-0abc123def456789 is communicating on port 443 with a known command and control server.",
  "resource": {
    "resourceType": "Instance",
    "instanceDetails": {
      "instanceId": "i-0abc123def456789",
      "instanceType": "t3.medium",
      "tags": [{"key": "Environment", "value": "Production"}]
    }
  },
  "service": {
    "action": {
      "actionType": "DNS_REQUEST",
      "dnsRequestAction": {
        "domain": "malicious-c2.badactor.com",
        "protocol": "UDP",
        "blocked": false
      }
    },
    "additionalInfo": {
      "threatListName": "ProofPoint",
      "threatName": "KnownThreatList"
    }
  }
}
```

#### Step 2: EventBridge Rule Triggers Automated Response

```json
// EventBridge Rule
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{"numeric": [">=", 7]}],
    "type": ["Backdoor:EC2/C&CActivity.B!DNS"]
  }
}
```

#### Step 3: Lambda Remediation Function Executes

```javascript
// Auto-remediation Lambda
async function isolateInstance(instanceId, accountId) {
  // 1. Create isolation security group (no inbound/outbound)
  // 2. Attach isolation SG to instance
  // 3. Detach from Auto Scaling Group
  // 4. Create EBS snapshot for forensics
  // 5. Send notification to security team
  // 6. Create JIRA ticket
}
```

#### Step 4: Security Team Notified

- **PagerDuty alert** fires with finding details
- **Slack message** posted to `#security-incidents` channel
- **JIRA ticket** created with severity P1

#### Step 5: Forensic Investigation

- EBS snapshot analyzed using Malware Protection scan
- GuardDuty finding timeline reviewed
- VPC Flow Logs correlated with finding timestamp
- Incident report generated

#### Step 6: Remediation Verified

- EC2 instance terminated after forensic copy
- New instance launched from clean AMI
- Security group rules tightened
- GuardDuty finding archived/resolved

---

## Advantages

1. **Zero Operational Overhead**: No agents, sensors, or network taps to deploy or maintain. Fully managed by AWS.

2. **Near Real-Time Detection**: Findings generated within minutes of suspicious activity, not hours or days.

3. **Intelligent Baselines**: ML models learn account-specific normal behavior, reducing false positives compared to static rules.

4. **Continuous Threat Intel Updates**: AWS automatically updates threat intelligence feeds — no manual signature updates required.

5. **Multi-Account Centralization**: Native AWS Organizations integration allows a single security team to monitor hundreds of accounts from one pane of glass.

6. **Broad Coverage**: Detects threats across compute (EC2, EKS, Lambda), storage (S3), database (RDS), and account-level activities.

7. **MITRE ATT&CK Mapping**: Findings are mapped to the MITRE ATT&CK framework, enabling security teams to understand attack stages.

8. **Cost-Effective at Scale**: Pay-per-use model with no upfront costs; often cheaper than building equivalent SIEM capabilities.

9. **Native Integration**: Deep integration with AWS Security Hub, EventBridge, and Organizations enables automated response workflows.

10. **Compliance Support**: Helps meet continuous monitoring requirements for PCI DSS, HIPAA, SOC 2, ISO 27001, and FedRAMP.

11. **Runtime Threat Detection**: With the GuardDuty agent, provides OS-level visibility without managing security tooling.

12. **Malware Scanning**: Automated EBS volume malware scanning without impacting running workloads (uses snapshots).

---

## Limitations

### Functional Limitations

| Limitation | Detail |
|---|---|
| **Not