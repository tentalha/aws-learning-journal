# AWS Config — Interview Questions

---

## Easy

### 1. What is AWS Config and what is its primary purpose?

**Answer:**
AWS Config is a fully managed service that continuously monitors and records the configuration state of your AWS resources. Its primary purpose is to:
- Provide a detailed inventory of AWS resources and their configurations
- Record configuration changes over time (configuration history)
- Evaluate resource configurations against desired settings using **Config Rules**
- Enable compliance auditing, security analysis, and operational troubleshooting

It answers the question: *"What did my AWS environment look like at any point in time?"*

---

### 2. What is a Configuration Item (CI) in AWS Config?

**Answer:**
A **Configuration Item (CI)** is a point-in-time snapshot of a single AWS resource. It includes:
- **Metadata** — version, creation time, state ID
- **Attributes** — resource type, ID, ARN, region, availability zone
- **Relationships** — how the resource relates to other AWS resources (e.g., an EC2 instance is associated with a security group)
- **Current configuration** — the full JSON configuration of the resource
- **Related events** — the CloudTrail event that triggered the change

A CI is created every time a resource's configuration changes.

---

### 3. What is an AWS Config Rule?

**Answer:**
An AWS Config Rule represents a desired configuration state for your AWS resources. Config evaluates resources against these rules and marks them as **COMPLIANT** or **NON_COMPLIANT**.

There are two types of rules:
- **AWS Managed Rules** — pre-built rules provided by AWS (e.g., `s3-bucket-public-read-prohibited`, `encrypted-volumes`, `mfa-enabled-for-iam-console-access`). There are 150+ managed rules available.
- **Custom Rules** — rules you write yourself using **AWS Lambda** (Guard-based rules or Lambda-based rules) to implement organization-specific compliance logic.

Rules can be triggered by **configuration changes** or on a **periodic schedule** (e.g., every 24 hours).

---

### 4. What is the AWS Config Configuration Recorder?

**Answer:**
The **Configuration Recorder** is the component within AWS Config that discovers and records the configurations of supported AWS resources. Key points:
- By default, it records **all supported resource types** in the region where it is enabled
- You can customize it to record only specific resource types
- It must be **started** to begin recording (it can be stopped and restarted)
- Only **one Configuration Recorder** can exist per AWS region per account
- Recorded data is delivered to an **S3 bucket** (called the delivery channel) as configuration history and configuration snapshots

---

### 5. What is the difference between AWS Config and AWS CloudTrail?

**Answer:**

| Feature | AWS Config | AWS CloudTrail |
|---|---|---|
| **Focus** | *What* is the resource configuration? | *Who* did *what* action and *when*? |
| **Data Type** | Resource configuration state | API call logs (events) |
| **Primary Use** | Compliance, drift detection, inventory | Audit trail, security investigation |
| **Question Answered** | "What does my S3 bucket look like now vs. 30 days ago?" | "Who changed the S3 bucket policy at 3pm?" |
| **Storage** | S3 (configuration history & snapshots) | S3, CloudWatch Logs |
| **Relationships** | Tracks resource relationships | No relationship tracking |

They are **complementary** — CloudTrail tells you *who* made a change, Config tells you *what* changed.

---

## Medium

### 1. How does AWS Config handle multi-account and multi-region aggregation?

**Answer:**
AWS Config supports aggregating configuration and compliance data across multiple accounts and regions using an **Aggregator**.

**Key Components:**
- **Aggregator** — a resource in a central (aggregator) account that collects Config data from source accounts/regions
- **Source accounts** — individual AWS accounts that authorize the aggregator account to collect their data
- **Authorization** — source accounts must explicitly authorize the aggregator (or you can use AWS Organizations for automatic authorization)

**Two methods of setting up aggregation:**
1. **Individual account authorization** — each source account manually authorizes the aggregator account via the Config console, CLI, or API
2. **AWS Organizations integration** — if you use AWS Organizations, the management account or a delegated admin account can automatically aggregate all member accounts without individual authorization

**What is aggregated:**
- Configuration items from all source accounts and regions
- Compliance status of Config rules (note: rules must be deployed independently in each account/region — the aggregator only *reads* compliance data, it does not deploy rules)
- Conformance pack compliance data

**Limitation:** The aggregator provides a **read-only** view. You cannot remediate or modify resources from the aggregator account directly.

---

### 2. What are Conformance Packs and how do they differ from individual Config Rules?

**Answer:**
A **Conformance Pack** is a collection of AWS Config rules and remediation actions packaged together as a single deployable unit using a YAML template (similar to CloudFormation).

**Key Differences from Individual Rules:**

| Aspect | Individual Config Rules | Conformance Packs |
|---|---|---|
| **Deployment** | One rule at a time | Bundle of rules deployed together |
| **Scope** | Single account/region | Can deploy across entire AWS Organization |
| **Template** | N/A | YAML/JSON template |
| **Use Case** | Specific compliance check | Full compliance framework (e.g., PCI-DSS, HIPAA, CIS) |
| **Remediation** | Configured separately | Can include remediation actions in the pack |

**Benefits of Conformance Packs:**
- AWS provides **sample conformance packs** for common frameworks (PCI DSS, HIPAA, NIST, CIS AWS Foundations Benchmark)
- Simplifies deploying a compliance framework across an organization
- Provides an **aggregate compliance score** for the entire pack
- Deployed via CloudFormation StackSets for organization-wide rollout

**Example:** The `Operational-Best-Practices-for-PCI-DSS` conformance pack contains 60+ rules covering PCI DSS requirements.

---

### 3. Explain AWS Config Remediation. What are the two types and how do they work?

**Answer:**
AWS Config **Remediation** allows you to automatically or manually fix non-compliant resources detected by Config Rules.

**Two Types:**

**1. Manual Remediation:**
- You trigger remediation on demand from the AWS Config console or via API
- Useful for reviewing non-compliant resources before taking action
- You select the non-compliant resource and click "Remediate"

**2. Automatic Remediation:**
- Config automatically triggers the remediation action when a resource is found non-compliant
- Configured with a **retry count** and **retry interval** to handle transient failures
- Uses **AWS Systems Manager Automation documents (SSM Automation runbooks)** to execute the remediation logic

**How it works:**
1. Config Rule evaluates a resource → marks it `NON_COMPLIANT`
2. Config triggers the associated **SSM Automation document**
3. The SSM document executes the remediation (e.g., enables encryption, removes public access, adds tags)
4. Config re-evaluates the resource after remediation

**Important considerations:**
- The IAM role used for remediation must have permissions to both invoke SSM Automation and modify the target resource
- AWS provides **pre-built SSM Automation documents** for common remediations (e.g., `AWS-EnableS3BucketEncryption`, `AWSConfigRemediation-DeleteUnusedIAMPolicy`)
- You can also create **custom SSM Automation documents** for complex remediations
- Remediation has a **throttling limit** — be careful with automatic remediation on rules that evaluate many resources

---

### 4. What is the AWS Config delivery channel and what does it deliver?

**Answer:**
The **Delivery Channel** is the mechanism through which AWS Config delivers configuration data to an S3 bucket and optionally to an SNS topic.

**What is delivered:**

**1. Configuration History Files (S3):**
- Delivered every **6 hours** (or on demand)
- Contains all configuration items for a specific resource type during the time period
- Format: `{account-id}_Config_{region}_ConfigHistory_{resource-type}_{timestamp}.json.gz`

**2. Configuration Snapshots (S3):**
- A point-in-time snapshot of all configuration items for all recorded resources
- Triggered on demand via API/CLI or scheduled
- Useful for full environment audits

**3. Configuration Stream (SNS):**
- Real-time notifications of configuration changes
- Delivers individual configuration items as they change
- Can be used to trigger Lambda functions, SQS queues, or other SNS subscribers for real-time processing

**S3 Bucket Requirements:**
- Must exist before enabling Config (or Config can create it)
- Bucket policy must allow Config service to write to it
- Can be in a different account (for centralized logging)
- Server-side encryption is supported

**SNS Topic (optional):**
- Notifies about configuration changes, compliance state changes, and delivery status

---

### 5. How does AWS Config track resource relationships, and why is this important?

**Answer:**
AWS Config automatically discovers and records **relationships** between AWS resources as part of each Configuration Item.

**How Relationships Are Tracked:**
- Each CI contains a `relationships` field listing related resources
- Relationships are bidirectional — if Resource A relates to Resource B, both CIs will reference each other
- Relationships are updated whenever either resource changes

**Examples of Tracked Relationships:**
- EC2 Instance → VPC, Subnet, Security Groups, EBS Volumes, IAM Instance Profile, Elastic IP
- S3 Bucket → S3 Bucket Policy, ACL
- RDS Instance → VPC, Subnet Group, Security Groups, Parameter Group
- Lambda Function → IAM Role, VPC, Security Groups

**Why This Is Important:**

1. **Impact Analysis** — Before changing a security group, you can see all EC2 instances using it
2. **Troubleshooting** — Trace configuration issues across related resources
3. **Compliance** — Verify that resources are associated with compliant configurations (e.g., EC2 instances must be in an approved VPC)
4. **Security Investigation** — Understand the blast radius of a misconfigured resource
5. **Timeline Correlation** — View how relationships changed over time alongside configuration changes

**Practical Example:** If an EC2 instance was compromised, you can use Config to see exactly which security groups, VPCs, and IAM roles were attached at the time of the incident — even if those resources have since changed.

---

## Hard

### 1. Describe the internal architecture of AWS Config — how does it discover, record, and evaluate resources?

**Answer:**
AWS Config operates through a multi-stage pipeline:

**Stage 1: Resource Discovery**
- Config uses **AWS service APIs** to perform an initial discovery of all existing resources when first enabled
- After initial discovery, it listens to **CloudTrail events** (specifically, the Config service subscribes to resource-level API events) to detect changes
- For resources that don't generate CloudTrail events on every change (e.g., some networking resources), Config uses **periodic polling**
- Supported resource types are explicitly defined — not all AWS resources are supported (though coverage is extensive)

**Stage 2: Configuration Recording**
- When a change is detected, Config calls the relevant AWS service API to retrieve the full, current configuration of the resource
- This is assembled into a **Configuration Item (CI)** containing metadata, attributes, relationships, and the full configuration JSON
- The CI is assigned a unique **configurationItemVersion** and **configurationStateId**
- The CI is stored in Config's internal datastore and simultaneously streamed to the delivery channel

**Stage 3: Rule Evaluation**
- **Change-triggered rules:** When a CI is created, Config checks which rules apply to that resource type and triggers evaluation
- **Periodic rules:** A scheduled CloudWatch Events rule triggers Config to evaluate all in-scope resources against the rule
- For **Lambda-based custom rules:** Config invokes the Lambda function with the CI as the event payload; Lambda returns `COMPLIANT`, `NON_COMPLIANT`, or `NOT_APPLICABLE`
- For **AWS Guard (CloudFormation Guard) rules:** Config evaluates the Guard policy directly without Lambda
- Evaluation results are stored and made available via the Config API and console

**Stage 4: Compliance Aggregation**
- Compliance results roll up from individual resources → rules → conformance packs → aggregators
- The aggregator account polls source accounts' Config APIs to collect compliance data

**Key Internal Behaviors:**
- Config has a **recording frequency** setting: continuous (default, records every change) or daily
- There is a **configuration item delivery delay** — changes appear in Config typically within minutes but can take up to several minutes depending on the service
- Config maintains a **resource timeline** — an ordered list of all CIs for a resource, enabling historical queries

---

### 2. How would you design a custom AWS Config Rule using Lambda to enforce a complex organizational policy? Walk through the full implementation.

**Answer:**
**Scenario:** Enforce that all EC2 instances must have a `CostCenter` tag with a value matching the pattern `CC-[0-9]{4}` (e.g., `CC-1234`).

**Full Implementation:**

**Step 1: Lambda Function Design**

```python
import boto3
import json
import re

def evaluate_compliance(configuration_item, rule_parameters):
    """Evaluate a single EC2 instance for tag compliance."""
    
    # Skip resources being deleted
    if configuration_item['configurationItemStatus'] == 'ResourceDeleted':
        return 'NOT_APPLICABLE'
    
    # Only evaluate EC2 instances
    if configuration_item['resourceType'] != 'AWS::EC2::Instance':
        return 'NOT_APPLICABLE'
    
    # Get tags from configuration item
    tags = configuration_item.get('tags', {})
    
    # Check for CostCenter tag
    cost_center = tags.get('CostCenter', '')
    
    # Validate pattern
    pattern = r'^CC-\d{4}$'
    if re.match(pattern, cost_center):
        return 'COMPLIANT'
    else:
        return 'NON_COMPLIANT'

def lambda_handler(event, context):
    invoking_event = json.loads(event['invokingEvent'])
    rule_parameters = json.loads(event.get('ruleParameters', '{}'))
    
    config_client = boto3.client('config')
    
    evaluations = []
    
    # Handle both change-triggered and periodic invocations
    if invoking_event.get('messageType') == 'ConfigurationItemChangeNotification':
        configuration_item = invoking_event['configurationItem']
        compliance = evaluate_compliance(configuration_item, rule_parameters)
        evaluations.append({
            'ComplianceResourceType': configuration_item['resourceType'],
            'ComplianceResourceId': configuration_item['resourceId'],
            'ComplianceType': compliance,
            'OrderingTimestamp': configuration_item['configurationItemCaptureTime']
        })
    
    elif invoking_event.get('messageType') == 'ScheduledNotification':
        # For periodic evaluation, paginate through all EC2 instances
        paginator = config_client.get_paginator('list_discovered_resources')
        for page in paginator.paginate(resourceType='AWS::EC2::Instance'):
            for resource in page['resourceIdentifiers']:
                response = config_client.get_resource_config_history(
                    resourceType=resource['resourceType'],
                    resourceId=resource['resourceId'],
                    limit=1
                )
                if response['configurationItems']:
                    ci = response['configurationItems'][0]
                    compliance = evaluate_compliance(ci, rule_parameters)
                    evaluations.append({
                        'ComplianceResourceType': ci['resourceType'],
                        'ComplianceResourceId': ci['resourceId'],
                        'ComplianceType': compliance,
                        'OrderingTimestamp': ci['configurationItemCaptureTime']
                    })
    
    # Put evaluations in batches of 100 (API limit)
    for i in range(0, len(evaluations), 100):
        config_client.put_evaluations(
            Evaluations=evaluations[i:i+100],
            ResultToken=event['resultToken']
        )
    
    return {'evaluations': len(evaluations)}
```

**Step 2: IAM Role for Lambda**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "config:PutEvaluations",
        "config:GetResourceConfigHistory",
        "config:ListDiscoveredResources"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "