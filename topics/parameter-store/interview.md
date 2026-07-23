# Parameter Store — Interview Questions

## Easy

---

### Q1. What is AWS Systems Manager Parameter Store, and what is its primary purpose?

**Answer:**
AWS Systems Manager Parameter Store is a secure, hierarchical storage service for configuration data management and secrets management. Its primary purpose is to provide a centralized place to store configuration values such as database connection strings, API keys, license codes, passwords, and other application settings. It allows you to separate your configuration data from your application code, making applications more portable, secure, and easier to manage across environments.

---

### Q2. What are the two main types of parameters in Parameter Store?

**Answer:**
Parameter Store offers two tiers:

- **Standard Parameters:** Free to use, support up to 10,000 parameters per account per region, with a maximum value size of 4 KB. No additional charge for storage or API calls (standard throughput).
- **Advanced Parameters:** Support up to 100,000 parameters per account per region, with a maximum value size of 8 KB. They also support parameter policies (e.g., expiration, notification). Advanced parameters incur a charge per parameter per month and per API interaction.

---

### Q3. What are the three parameter types supported by Parameter Store?

**Answer:**

| Type | Description |
|------|-------------|
| `String` | Any block of text — plain text values like environment names, AMI IDs, etc. |
| `StringList` | A comma-separated list of values stored as a single parameter. |
| `SecureString` | An encrypted value using AWS KMS. Used for sensitive data like passwords, API keys, and secrets. |

---

### Q4. How does Parameter Store integrate with AWS KMS?

**Answer:**
When you create a `SecureString` parameter, Parameter Store uses AWS Key Management Service (KMS) to encrypt the value at rest. You can use:

- The **AWS-managed default key** (`aws/ssm`) — no additional KMS cost for the key itself, but API call charges apply.
- A **customer-managed KMS key (CMK)** — gives you full control over key rotation, access policies, and auditing.

When retrieving a `SecureString`, you must have `kms:Decrypt` permission on the relevant KMS key. The `--with-decryption` flag must be used in the CLI to retrieve the plaintext value.

---

### Q5. How do you retrieve a parameter value using the AWS CLI?

**Answer:**

```bash
# Retrieve a plain String parameter
aws ssm get-parameter --name "/myapp/dev/db-host"

# Retrieve a SecureString parameter with decryption
aws ssm get-parameter \
  --name "/myapp/dev/db-password" \
  --with-decryption

# Retrieve multiple parameters by path
aws ssm get-parameters-by-path \
  --path "/myapp/dev/" \
  --with-decryption \
  --recursive
```

The response includes the `Name`, `Type`, `Value`, `Version`, `LastModifiedDate`, and `ARN` of the parameter.

---

## Medium

---

### Q1. Explain the hierarchical naming convention in Parameter Store and why it is beneficial.

**Answer:**
Parameter Store supports a hierarchical namespace using forward slashes (`/`) as delimiters, similar to a file system path. For example:

```
/application/environment/component/parameter-name

/myapp/prod/database/host
/myapp/prod/database/port
/myapp/prod/database/password
/myapp/dev/database/host
/myapp/dev/database/port
```

**Benefits:**

1. **Bulk retrieval:** Using `GetParametersByPath`, you can retrieve all parameters under a given path (e.g., `/myapp/prod/`) in a single API call, reducing the number of calls your application needs to make at startup.
2. **IAM scoping:** You can write IAM policies that grant access to specific paths, enforcing least-privilege access. For example, a dev Lambda function can only access `/myapp/dev/*` while a prod function accesses `/myapp/prod/*`.
3. **Environment separation:** Clearly separates parameters by environment, application, or team without naming collisions.
4. **Organizational clarity:** Makes it easy to audit and manage parameters at scale.
5. **CloudFormation and CDK integration:** Stacks can reference parameters by predictable path patterns.

---

### Q2. What are Parameter Policies, and what types are available?

**Answer:**
Parameter Policies are available only for **Advanced Parameters** and allow you to define lifecycle rules on individual parameters. There are three types:

1. **Expiration Policy:**
   - Deletes the parameter at a specified date/time (ISO 8601 format).
   - Useful for temporary credentials or time-limited configuration values.
   ```json
   {
     "Type": "Expiration",
     "Version": "1.0",
     "Attributes": {
       "Timestamp": "2024-12-31T23:59:59.000Z"
     }
   }
   ```

2. **ExpirationNotification Policy:**
   - Sends an Amazon EventBridge event a specified number of days/hours before a parameter expires.
   - Allows teams to rotate secrets proactively before expiration.
   ```json
   {
     "Type": "ExpirationNotification",
     "Version": "1.0",
     "Attributes": {
       "Before": "15",
       "Unit": "Days"
     }
   }
   ```

3. **NoChangeNotification Policy:**
   - Triggers an EventBridge event if a parameter has not been updated within a specified timeframe.
   - Useful for enforcing secret rotation policies.
   ```json
   {
     "Type": "NoChangeNotification",
     "Version": "1.0",
     "Attributes": {
       "After": "30",
       "Unit": "Days"
     }
   }
   ```

These policies enable automated governance and compliance workflows when combined with EventBridge rules and Lambda functions.

---

### Q3. How does Parameter Store versioning work, and how can you reference a specific version?

**Answer:**
Every time a parameter value is updated, Parameter Store automatically increments a version number starting at `1`. The current version is always returned by default when you call `GetParameter`.

**Key points:**

- Versions are immutable — you cannot edit a historical version.
- Up to **100 previous versions** are retained for Standard Parameters; Advanced Parameters also retain 100 versions.
- You can reference a specific version by appending `:version-number` to the parameter name.

**CLI example:**
```bash
# Get the current (latest) version
aws ssm get-parameter --name "/myapp/prod/api-key"

# Get a specific historical version
aws ssm get-parameter --name "/myapp/prod/api-key:3"

# Get the latest version explicitly
aws ssm get-parameter --name "/myapp/prod/api-key:$LATEST"
```

**Use cases for version pinning:**
- Rollback: If a new parameter value causes issues, you can reference the previous version during debugging.
- Audit: Verify what value was in use at a specific point in time.
- CloudFormation: You can pin a CloudFormation template to a specific parameter version to prevent unintended drift.

**CloudFormation dynamic reference:**
```yaml
MyParameter: !Sub "{{resolve:ssm:/myapp/prod/api-key:3}}"
```

---

### Q4. Compare Parameter Store and AWS Secrets Manager. When would you choose one over the other?

**Answer:**

| Feature | Parameter Store | Secrets Manager |
|---|---|---|
| **Cost** | Free (Standard); ~$0.05/advanced param/month | ~$0.40/secret/month |
| **Max value size** | 4 KB (Standard), 8 KB (Advanced) | 65 KB |
| **Automatic rotation** | No (manual via Lambda + EventBridge) | Yes (native rotation via Lambda) |
| **Cross-account access** | Limited | Native support via resource policies |
| **Secret types** | Generic key-value | Structured JSON (RDS, Redshift, etc.) |
| **Audit** | CloudTrail | CloudTrail |
| **Parameter policies** | Advanced only | Native expiration/rotation |
| **Versioning** | Yes (100 versions) | Yes (with staging labels) |
| **Multi-region replication** | No | Yes |

**Choose Parameter Store when:**
- Storing non-sensitive configuration (feature flags, environment names, AMI IDs).
- Cost is a concern and you have many parameters.
- You need hierarchical path-based access control.
- You're already using SSM and want a unified toolset.

**Choose Secrets Manager when:**
- You need **automatic secret rotation** (especially for RDS, Redshift, DocumentDB).
- You need **cross-account secret sharing**.
- The secret is a structured JSON object (e.g., RDS credentials with host, port, username, password).
- You need **multi-region replication** of secrets.

---

### Q5. How do you use Parameter Store with Lambda functions? What are the best practices for caching?

**Answer:**
Lambda functions commonly retrieve parameters at initialization time. There are several approaches:

**Approach 1: Direct SDK call at cold start**
```python
import boto3

ssm = boto3.client('ssm')

def get_parameter(name):
    response = ssm.get_parameter(Name=name, WithDecryption=True)
    return response['Parameter']['Value']

# Called during cold start (outside handler)
DB_HOST = get_parameter('/myapp/prod/db-host')
DB_PASS = get_parameter('/myapp/prod/db-password')

def handler(event, context):
    # Use DB_HOST and DB_PASS
    pass
```

**Approach 2: AWS Parameters and Secrets Lambda Extension (recommended)**
AWS provides a Lambda layer (`AWS-Parameters-and-Secrets-Lambda-Extension`) that:
- Caches parameter values in-memory within the Lambda execution environment.
- Serves values via a local HTTP endpoint (`localhost:2773`).
- Configurable TTL for cache refresh.
- Reduces SSM API calls and latency.

```python
import urllib.request
import os

def get_parameter(name):
    url = f"http://localhost:2773/systemsmanager/parameters/get?name={name}&withDecryption=true"
    req = urllib.request.Request(url)
    req.add_header('X-Aws-Parameters-Secrets-Token', os.environ['AWS_SESSION_TOKEN'])
    with urllib.request.urlopen(req) as response:
        return response.read()
```

**Best practices:**
1. **Cache outside the handler** to reuse across warm invocations.
2. **Use the Lambda Extension** for automatic caching with configurable TTL.
3. **Use `GetParametersByPath`** to batch-load all parameters for a path in one call.
4. **Avoid calling SSM inside the handler** on every invocation — it adds latency and cost.
5. **Set appropriate IAM permissions** using least-privilege paths.

---

## Hard

---

### Q1. Describe the IAM permission model for Parameter Store in detail, including how to implement least-privilege access across multiple teams and environments.

**Answer:**
Parameter Store integrates deeply with IAM, and a well-designed permission model is critical for security at scale.

**Core IAM Actions:**

| Action | Description |
|--------|-------------|
| `ssm:GetParameter` | Retrieve a single parameter |
| `ssm:GetParameters` | Retrieve multiple parameters by name |
| `ssm:GetParametersByPath` | Retrieve parameters under a path |
| `ssm:PutParameter` | Create or update a parameter |
| `ssm:DeleteParameter` | Delete a parameter |
| `ssm:DescribeParameters` | List parameters (metadata only) |
| `ssm:GetParameterHistory` | View version history |
| `ssm:LabelParameterVersion` | Add labels to parameter versions |

**Least-Privilege Path-Based Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadDevParameters",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/dev/*"
    },
    {
      "Sid": "DenyProdParameters",
      "Effect": "Deny",
      "Action": "ssm:*",
      "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/prod/*"
    }
  ]
}
```

**Multi-team strategy:**

```
/team-alpha/prod/*   → IAM role: team-alpha-prod-role
/team-alpha/dev/*    → IAM role: team-alpha-dev-role
/team-beta/prod/*    → IAM role: team-beta-prod-role
/shared/global/*     → IAM role: all roles (read-only)
```

**SecureString KMS considerations:**
- Reading a `SecureString` requires both `ssm:GetParameter` AND `kms:Decrypt` on the KMS key.
- Writing a `SecureString` requires `ssm:PutParameter` AND `kms:GenerateDataKey`.
- Use separate CMKs per environment to enforce hard boundaries.

**SCP (Service Control Policy) guardrails:**
In AWS Organizations, you can use SCPs to prevent production accounts from ever accessing dev parameters and vice versa, adding an organization-level layer of defense.

**Conditions for additional security:**
```json
{
  "Condition": {
    "StringEquals": {
      "aws:RequestedRegion": "us-east-1"
    },
    "Bool": {
      "aws:SecureTransport": "true"
    }
  }
}
```

---

### Q2. How does Parameter Store handle high-throughput scenarios, and what are the throttling limits and mitigation strategies?

**Answer:**
Parameter Store enforces API throughput limits that can become a bottleneck in high-scale architectures.

**Default Throughput Limits:**

| Tier | Default TPS | Maximum TPS (with higher throughput) |
|------|------------|--------------------------------------|
| Standard | 40 TPS | 40 TPS (cannot be increased without upgrading) |
| Advanced | 40 TPS | 1,000 TPS (configurable, additional cost) |

**Higher Throughput Mode:**
You can enable higher throughput for Advanced Parameters at the account/region level, allowing up to 1,000 TPS for `GetParameter` and `GetParameters` operations. This incurs additional charges per API interaction.

```bash
aws ssm update-service-setting \
  --setting-id arn:aws:ssm:us-east-1:123456789012:servicesetting/ssm/parameter-store/high-throughput-enabled \
  --setting-value true
```

**Throttling error:** `ThrottlingException` — HTTP 400

**Mitigation Strategies:**

1. **Caching at the application layer:**
   - Use in-memory caches (e.g., Python `functools.lru_cache`, Redis, ElastiCache) with a TTL.
   - Refresh only when the TTL expires, not on every request.

2. **Lambda Extension caching:**
   - The AWS Parameters and Secrets Lambda Extension caches values locally per execution environment, dramatically reducing SSM API calls.

3. **Batch retrieval:**
   - Use `GetParametersByPath` (returns up to 10 parameters per page) or `GetParameters` (up to 10 names per call) instead of individual `GetParameter` calls.

4. **Parameter Store → Environment Variables pattern:**
   - At deployment time (e.g., in CI/CD), resolve parameters and inject them as Lambda environment variables. The application never calls SSM at runtime.
   - Trade-off: Values are static until redeployment.

5. **AppConfig integration:**
   - For feature flags and frequently changing configuration, use AWS AppConfig (which has its own caching mechanism) backed by Parameter Store as the source of truth.

6. **Exponential backoff with jitter:**
   - Implement retry logic with exponential backoff for `ThrottlingException` errors.

7. **Regional distribution:**
   - For globally distributed applications, replicate critical parameters to multiple regions and read from the local region.

---

### Q3. Explain how Parameter Store integrates with CloudFormation, including dynamic references, and what are their limitations?

**Answer:**
CloudFormation provides two mechanisms to use Parameter Store values: **stack parameters** and **dynamic references**.

**Mechanism 1: Stack Parameters (SSM Parameter Type)**
```yaml
Parameters:
  LatestAmi