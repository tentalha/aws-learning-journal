# Secrets Manager — Interview Questions

## Easy

---

**Q1. What is AWS Secrets Manager, and what problem does it solve?**

**Answer:**
AWS Secrets Manager is a fully managed service that enables you to securely store, retrieve, rotate, and manage sensitive information such as database credentials, API keys, OAuth tokens, and other secrets. It solves the problem of hardcoding secrets in application source code, configuration files, or environment variables — practices that expose credentials to security risks. Instead, applications call the Secrets Manager API at runtime to retrieve the secret value, ensuring credentials are never exposed in plaintext in code repositories or deployment artifacts.

---

**Q2. How does Secrets Manager differ from AWS Systems Manager Parameter Store?**

**Answer:**
Both services can store secrets, but they differ in key ways:

| Feature | Secrets Manager | Parameter Store |
|---|---|---|
| **Cost** | Paid per secret per month | Free tier available (Standard); Advanced tier costs extra |
| **Automatic Rotation** | Built-in, native rotation with Lambda | Manual rotation required |
| **Cross-account access** | Supported natively | Supported but more complex |
| **Secret versioning** | Full versioning with staging labels | Basic versioning |
| **Encryption** | Always KMS-encrypted | Standard tier uses SSM-managed key; Advanced uses KMS |
| **Primary use case** | Secrets/credentials with rotation | Configuration data and secrets |

Secrets Manager is purpose-built for secrets with rotation requirements, while Parameter Store is often used for broader configuration management including non-sensitive data.

---

**Q3. What is automatic secret rotation in Secrets Manager, and how does it work at a high level?**

**Answer:**
Automatic secret rotation is a feature that periodically and automatically updates a secret's value without requiring manual intervention. It works through an AWS Lambda function that Secrets Manager invokes on a defined schedule. The rotation Lambda function follows a four-step lifecycle:

1. **createSecret** — Creates a new version of the secret with a new credential value.
2. **setSecret** — Updates the target resource (e.g., RDS database) with the new credential.
3. **testSecret** — Verifies the new credential works against the target resource.
4. **finishSecret** — Marks the new version as `AWSCURRENT` and the old version as `AWSPREVIOUS`.

AWS provides managed rotation Lambda functions for supported services like Amazon RDS, Redshift, and DocumentDB.

---

**Q4. How do you retrieve a secret value from Secrets Manager in an application?**

**Answer:**
You retrieve a secret using the AWS SDK or CLI. The primary API call is `GetSecretValue`. Example using the AWS CLI:

```bash
aws secretsmanager get-secret-value \
  --secret-id MyDatabaseSecret \
  --version-stage AWSCURRENT
```

In Python using boto3:

```python
import boto3
import json

client = boto3.client('secretsmanager', region_name='us-east-1')

response = client.get_secret_value(SecretId='MyDatabaseSecret')
secret = json.loads(response['SecretString'])

db_password = secret['password']
```

Best practice is to cache the secret locally for a period (e.g., 5 minutes) to reduce API call frequency and latency, and to handle the case where a cached secret fails (indicating rotation occurred) by re-fetching from Secrets Manager.

---

**Q5. What encryption mechanism does Secrets Manager use to protect secrets?**

**Answer:**
Secrets Manager always encrypts secrets at rest using AWS Key Management Service (KMS). By default, it uses the AWS-managed key `aws/secretsmanager`. You can optionally specify a customer-managed KMS key (CMK) for greater control, including the ability to:

- Audit key usage in AWS CloudTrail.
- Define granular key policies.
- Rotate the KMS key independently.
- Revoke access by disabling the KMS key.

In transit, all communication with Secrets Manager is encrypted using TLS. The secret value is encrypted before being stored in Secrets Manager's underlying storage, and decryption happens only when an authorized principal calls `GetSecretValue`.

---

## Medium

---

**Q1. Explain secret versioning and staging labels in Secrets Manager. How are they used during rotation?**

**Answer:**
Secrets Manager maintains multiple versions of a secret simultaneously. Each version is identified by a unique **Version ID** (UUID) and one or more **staging labels** — string identifiers that mark the state of a version.

**Key built-in staging labels:**
- `AWSCURRENT` — The current, active version used by applications.
- `AWSPENDING` — The newly created version during rotation, not yet validated.
- `AWSPREVIOUS` — The previous version, retained temporarily after rotation completes.

**How rotation uses them:**

```
Before Rotation:
  Version A → AWSCURRENT
  Version Z → AWSPREVIOUS

During Rotation:
  Version A → AWSCURRENT
  Version B → AWSPENDING  (new credentials created, not yet tested)

After Rotation Completes:
  Version B → AWSCURRENT  (new credentials validated and promoted)
  Version A → AWSPREVIOUS (retained for rollback)
  Version Z → (no label, eligible for deletion)
```

Applications that retrieve `AWSCURRENT` automatically get the latest valid secret after rotation. The `AWSPREVIOUS` version is retained so that in-flight connections using the old credential aren't immediately broken. Custom staging labels can also be defined for application-specific version management.

---

**Q2. How does resource-based policy work in Secrets Manager, and when would you use it over IAM identity-based policies?**

**Answer:**
Secrets Manager supports **resource-based policies** attached directly to a secret. These policies define who (which principals) can perform which actions on that specific secret.

**Example resource-based policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/ApplicationRole"
      },
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "*"
    }
  ]
}
```

**When to use resource-based policies vs. identity-based policies:**

| Use Case | Preferred Approach |
|---|---|
| Grant cross-account access to a secret | Resource-based policy (attach to secret, allow external account principal) |
| Grant multiple secrets to one role | Identity-based policy on the IAM role |
| Restrict access to a specific secret | Resource-based policy (more granular, closer to the resource) |
| Deny access regardless of IAM policy | Resource-based explicit Deny |
| Organizational-wide access control | Use both + SCPs |

Resource-based policies are essential for **cross-account secret sharing** because they allow you to grant access to principals in other AWS accounts without requiring them to assume a role in your account.

---

**Q3. How would you implement secret caching in an application to avoid throttling and reduce latency?**

**Answer:**
Secrets Manager has API rate limits (`GetSecretValue` is throttled at 10,000 requests per second per region, but per-account limits can be lower). Repeated calls on every request add latency (~10-50ms) and cost. The solution is **client-side caching**.

**AWS provides official caching libraries:**
- **Java:** `aws-secretsmanager-caching-java`
- **Python:** `aws-secretsmanager-caching-python`
- **.NET:** `AWSSDK.SecretsManager.Caching`

**Python caching example:**

```python
import botocore
import botocore.session
from aws_secretsmanager_caching import SecretCache, SecretCacheConfig

client = botocore.session.get_session().create_client('secretsmanager')
cache_config = SecretCacheConfig(
    max_cache_size=1000,
    exception_retry_delay_base=1,
    exception_retry_growth_factor=2,
    exception_retry_delay_max=3600,
    default_version_stage='AWSCURRENT',
    secret_refresh_interval=3600,  # Refresh every hour
    secret_version_stage_refresh_interval=3600
)
cache = SecretCache(config=cache_config, client=client)

secret = cache.get_secret_string('MySecret')
```

**Best practices for caching:**
1. Set a TTL (typically 5 minutes to 1 hour) to balance freshness with performance.
2. **Handle authentication failures** — if a cached credential fails, immediately refresh the cache (the secret may have rotated).
3. Use **exponential backoff** when Secrets Manager calls fail.
4. Store secrets in memory only, never write cached secrets to disk or logs.
5. In Lambda, use the **Lambda extension for Secrets Manager** which caches secrets within the execution environment.

---

**Q4. What is the Secrets Manager Lambda extension, and how does it improve performance in serverless architectures?**

**Answer:**
The **AWS Secrets Manager Lambda Extension** is a Lambda layer that runs as a sidecar process within the Lambda execution environment. It provides an **in-process HTTP endpoint** (localhost) that Lambda functions call to retrieve secrets, instead of making direct API calls to Secrets Manager.

**How it works:**

```
Lambda Function → localhost:2773/secretsmanager/get?secretId=MySecret
                → Lambda Extension (local cache)
                → Secrets Manager API (only on cache miss or TTL expiry)
```

**Benefits:**
1. **Reduced latency** — Local HTTP call (~1ms) vs. network call to Secrets Manager (~10-50ms).
2. **Reduced API calls** — Secrets are cached within the execution environment across warm invocations.
3. **No SDK dependency** — Works with any runtime (Python, Node.js, Go, etc.) using a simple HTTP call.
4. **Automatic refresh** — Respects `AWSCURRENT` staging label and refreshes on TTL expiry.

**Usage example (environment variable + HTTP call):**

```bash
# Set environment variable
PARAMETERS_SECRETS_EXTENSION_CACHE_ENABLED=true
SECRETS_MANAGER_TTL=300  # 5 minutes

# In function code (any language)
curl -H "X-Aws-Parameters-Secrets-Token: $AWS_SESSION_TOKEN" \
  http://localhost:2773/secretsmanager/get?secretId=MySecret
```

**Limitation:** The cache is per execution environment, not shared across concurrent Lambda instances.

---

**Q5. How do you audit access to secrets in Secrets Manager, and what CloudTrail events should you monitor?**

**Answer:**
AWS CloudTrail automatically records all Secrets Manager API calls. Every `GetSecretValue`, `CreateSecret`, `DeleteSecret`, `RotateSecret`, and policy change is logged as a CloudTrail event.

**Critical events to monitor:**

| Event | Risk Level | Description |
|---|---|---|
| `GetSecretValue` | High | Someone retrieved a secret value |
| `DeleteSecret` | Critical | Secret deletion initiated |
| `PutSecretValue` | High | Secret value was changed |
| `RotateSecret` | Medium | Rotation was triggered |
| `UpdateSecret` | Medium | Secret metadata changed |
| `CreateSecret` | Medium | New secret created |
| `RestoreSecret` | Medium | Deleted secret was restored |
| `RemoveRegionsFromReplication` | Medium | Replication configuration changed |

**Monitoring strategy:**

```json
// CloudWatch Metric Filter for GetSecretValue
{
  "filterPattern": "{ $.eventName = \"GetSecretValue\" }",
  "metricName": "SecretAccessCount",
  "metricNamespace": "SecurityMetrics"
}
```

**Best practices:**
1. Enable **CloudTrail data events** for Secrets Manager (they're not enabled by default for data plane operations in all setups).
2. Create **CloudWatch Alarms** for unexpected spikes in `GetSecretValue` calls.
3. Use **AWS Security Hub** and **Amazon GuardDuty** which can analyze Secrets Manager events for anomalous behavior.
4. Set up **EventBridge rules** to trigger alerts on `DeleteSecret` or policy changes.
5. Use **AWS Config** rules to enforce that secrets have rotation enabled and resource policies are compliant.

---

## Hard

---

**Q1. Describe the complete rotation Lambda function lifecycle in detail. What happens if one step fails, and how does Secrets Manager handle partial rotation failures?**

**Answer:**
The rotation Lambda function is invoked four times per rotation cycle, once for each step. Secrets Manager passes a `Step` parameter and the `SecretId` + `ClientRequestToken` (the new version ID).

**Detailed step breakdown:**

```python
def lambda_handler(event, context):
    secret_id = event['SecretId']
    token = event['ClientRequestToken']  # New version UUID
    step = event['Step']

    if step == "createSecret":
        create_secret(client, secret_id, token)
    elif step == "setSecret":
        set_secret(client, secret_id, token)
    elif step == "testSecret":
        test_secret(client, secret_id, token)
    elif step == "finishSecret":
        finish_secret(client, secret_id, token)
```

**Step 1 — createSecret:**
```python
def create_secret(client, secret_id, token):
    # Check if AWSPENDING already exists (idempotency)
    try:
        client.get_secret_value(SecretId=secret_id, 
                                VersionId=token, 
                                VersionStage="AWSPENDING")
        return  # Already created, skip
    except client.exceptions.ResourceNotFoundException:
        pass
    
    # Generate new credentials
    current = json.loads(client.get_secret_value(
        SecretId=secret_id, VersionStage="AWSCURRENT")['SecretString'])
    new_secret = current.copy()
    new_secret['password'] = generate_password()
    
    # Store as AWSPENDING
    client.put_secret_value(
        SecretId=secret_id,
        ClientRequestToken=token,
        SecretString=json.dumps(new_secret),
        VersionStages=['AWSPENDING']
    )
```

**Step 2 — setSecret:**
Connects to the target resource using `AWSCURRENT` credentials and creates/updates the user with `AWSPENDING` credentials. Must be **idempotent** — if the password was already set, this should not fail.

**Step 3 — testSecret:**
Connects to the target resource using `AWSPENDING` credentials to verify they work. If this fails, the rotation fails here.

**Step 4 — finishSecret:**
```python
def finish_secret(client, secret_id, token):
    # Move AWSCURRENT label to new version
    metadata = client.describe_secret(SecretId=secret_id)
    current_version = None
    for version, stages in metadata['VersionIdsToStages'].items():
        if 'AWSCURRENT' in stages:
            current_version = version
            break
    
    client.update_secret_version_stage(
        SecretId=secret_id,
        VersionStage='AWSCURRENT',
        MoveToVersionId=token,
        RemoveFromVersionId=current_version
    )
```

**Failure handling:**

| Failure Point | Behavior |
|---|---|
| `createSecret` fails | Rotation fails; `AWSPENDING` may or may not exist; next rotation attempt retries |
| `setSecret` fails | Rotation fails; `AWSPENDING` exists but target resource still has old password |
| `testSecret` fails | Rotation fails; target resource has new password but `AWSCURRENT` still points to old version — **dangerous state** |
| `finishSecret` fails | New password works on target but `AWSCURRENT` still points to old version |

**Critical concern — testSecret failure after setSecret:**
If `setSecret` succeeds (target DB has new password) but `testSecret` fails, the target resource now has a password that doesn't match `AWSCURRENT`. Applications will fail to authenticate. Resolution requires manual intervention to either roll back the DB password or manually call `finishSecret` logic.

**Idempotency is critical** — each step must be safe to call multiple times because Secrets Manager may retry on transient failures.

---

**Q2. How would you design a secrets management strategy for a multi-account, multi-region AWS organization with strict compliance requirements (e.g., PCI-DSS)?**

**Answer:**
This requires a layered approach addressing governance, access control, replication, rotation, and auditability.

**Architecture Overview:**

```
AWS Organizations