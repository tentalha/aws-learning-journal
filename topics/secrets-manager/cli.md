# Secrets Manager — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using AWS CLI with Secrets Manager, ensure the following are in place:

**Install/Update AWS CLI:**
```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify version
aws --version
```

**Configure CLI credentials:**
```bash
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

### Required IAM Permissions

Attach a policy with the following minimum permissions to your IAM user or role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:CreateSecret",
        "secretsmanager:GetSecretValue",
        "secretsmanager:PutSecretValue",
        "secretsmanager:UpdateSecret",
        "secretsmanager:DeleteSecret",
        "secretsmanager:ListSecrets",
        "secretsmanager:DescribeSecret",
        "secretsmanager:RotateSecret",
        "secretsmanager:TagResource",
        "secretsmanager:UntagResource",
        "secretsmanager:RestoreSecret",
        "secretsmanager:ListSecretVersionIds"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:*"
    }
  ]
}
```

**If using KMS Customer Managed Keys (CMK), also add:**
```json
{
  "Effect": "Allow",
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey",
    "kms:DescribeKey"
  ],
  "Resource": "arn:aws:kms:us-east-1:123456789012:key/mrk-1234abcd12ab34cd56ef1234567890ab"
}
```

---

## Core Commands

### 1. Create a New Secret (String Value)

```bash
aws secretsmanager create-secret \
  --name "prod/myapp/database" \
  --description "Production database credentials for MyApp" \
  --secret-string '{"username":"dbadmin","password":"S3cur3P@ssw0rd!"}' \
  --region us-east-1
```

**What it does:** Creates a new secret in Secrets Manager with a JSON key-value string. The secret is encrypted using the default AWS managed key (`aws/secretsmanager`) unless a CMK is specified.

**Example Output:**
```json
{
    "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/database-aBcDeF",
    "Name": "prod/myapp/database",
    "VersionId": "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111"
}
```

---

### 2. Create a Secret with a KMS Key

```bash
aws secretsmanager create-secret \
  --name "prod/myapp/api-key" \
  --description "Third-party API key for MyApp production" \
  --secret-string "sk-prod-abc123XYZexampleApiKeyValue9876" \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/mrk-1234abcd12ab34cd56ef1234567890ab" \
  --tags Key=Environment,Value=Production Key=Team,Value=Backend \
  --region us-east-1
```

**What it does:** Creates a secret encrypted with a specific Customer Managed Key (CMK) in KMS, and applies resource tags for cost allocation and access control.

---

### 3. Retrieve a Secret Value

```bash
aws secretsmanager get-secret-value \
  --secret-id "prod/myapp/database" \
  --region us-east-1
```

**What it does:** Retrieves the current (AWSCURRENT) version of a secret's value. Returns both the ARN and the decrypted secret string or binary.

**Example Output:**
```json
{
    "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/database-aBcDeF",
    "Name": "prod/myapp/database",
    "VersionId": "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
    "SecretString": "{\"username\":\"dbadmin\",\"password\":\"S3cur3P@ssw0rd!\"}",
    "VersionStages": ["AWSCURRENT"],
    "CreatedDate": "2024-01-15T10:30:00.000000+00:00"
}
```

---

### 4. Retrieve a Specific Secret Version

```bash
aws secretsmanager get-secret-value \
  --secret-id "prod/myapp/database" \
  --version-stage "AWSPREVIOUS" \
  --region us-east-1
```

**What it does:** Retrieves a specific version of the secret using a version stage label (`AWSCURRENT`, `AWSPREVIOUS`, or a custom label). Useful for rollback scenarios.

---

### 5. Update a Secret's Value

```bash
aws secretsmanager put-secret-value \
  --secret-id "prod/myapp/database" \
  --secret-string '{"username":"dbadmin","password":"N3wS3cur3P@ss2024!"}' \
  --region us-east-1
```

**What it does:** Adds a new version of the secret value. The old version is labeled `AWSPREVIOUS` and the new version becomes `AWSCURRENT`. Does not modify secret metadata.

**Example Output:**
```json
{
    "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/database-aBcDeF",
    "Name": "prod/myapp/database",
    "VersionId": "b2c3d4e5-6789-01bc-defg-EXAMPLE22222",
    "VersionStages": ["AWSCURRENT"]
}
```

---

### 6. Describe a Secret

```bash
aws secretsmanager describe-secret \
  --secret-id "prod/myapp/database" \
  --region us-east-1
```

**What it does:** Returns metadata about a secret (ARN, name, description, rotation configuration, tags, version IDs) without revealing the secret value.

**Example Output:**
```json
{
    "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/database-aBcDeF",
    "Name": "prod/myapp/database",
    "Description": "Production database credentials for MyApp",
    "KmsKeyId": "arn:aws:kms:us-east-1:123456789012:key/mrk-1234abcd12ab34cd56ef1234567890ab",
    "RotationEnabled": false,
    "LastChangedDate": "2024-01-15T10:30:00.000000+00:00",
    "LastAccessedDate": "2024-01-16T00:00:00.000000+00:00",
    "Tags": [
        {"Key": "Environment", "Value": "Production"},
        {"Key": "Team", "Value": "Backend"}
    ],
    "VersionIdsToStages": {
        "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111": ["AWSPREVIOUS"],
        "b2c3d4e5-6789-01bc-defg-EXAMPLE22222": ["AWSCURRENT"]
    }
}
```

---

### 7. List All Secrets

```bash
aws secretsmanager list-secrets \
  --max-results 50 \
  --region us-east-1
```

**What it does:** Returns a paginated list of all secrets in the current account and region. Does not return secret values, only metadata.

**Example Output:**
```json
{
    "SecretList": [
        {
            "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/database-aBcDeF",
            "Name": "prod/myapp/database",
            "Description": "Production database credentials for MyApp",
            "LastChangedDate": "2024-01-15T10:30:00.000000+00:00",
            "Tags": [{"Key": "Environment", "Value": "Production"}]
        }
    ]
}
```

---

### 8. Update Secret Metadata

```bash
aws secretsmanager update-secret \
  --secret-id "prod/myapp/database" \
  --description "Updated: Production RDS credentials for MyApp v2" \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/mrk-newkey5678abcd12ab34cd56ef123456" \
  --region us-east-1
```

**What it does:** Updates secret metadata such as description or KMS key. Use `update-secret` for metadata changes and `put-secret-value` for value changes.

---

### 9. Delete a Secret

```bash
aws secretsmanager delete-secret \
  --secret-id "prod/myapp/database" \
  --recovery-window-in-days 30 \
  --region us-east-1
```

**What it does:** Schedules a secret for deletion after a recovery window (7–30 days). During this window, the secret can be restored. The secret is inaccessible immediately upon deletion scheduling.

**Example Output:**
```json
{
    "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/database-aBcDeF",
    "Name": "prod/myapp/database",
    "DeletionDate": "2024-02-15T10:30:00.000000+00:00"
}
```

---

### 10. Force Delete a Secret Immediately

```bash
aws secretsmanager delete-secret \
  --secret-id "prod/myapp/temp-token" \
  --force-delete-without-recovery \
  --region us-east-1
```

**What it does:** Immediately and permanently deletes a secret with no recovery window. Use with extreme caution — this action is irreversible.

---

### 11. Restore a Deleted Secret

```bash
aws secretsmanager restore-secret \
  --secret-id "prod/myapp/database" \
  --region us-east-1
```

**What it does:** Cancels a scheduled deletion and restores the secret to an active state. Only works within the recovery window.

---

### 12. List Secret Version IDs

```bash
aws secretsmanager list-secret-version-ids \
  --secret-id "prod/myapp/database" \
  --include-deprecated \
  --region us-east-1
```

**What it does:** Lists all version IDs for a secret, including their staging labels and creation dates. The `--include-deprecated` flag shows versions no longer associated with any staging label.

**Example Output:**
```json
{
    "Versions": [
        {
            "VersionId": "b2c3d4e5-6789-01bc-defg-EXAMPLE22222",
            "VersionStages": ["AWSCURRENT"],
            "CreatedDate": "2024-01-16T09:00:00.000000+00:00"
        },
        {
            "VersionId": "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
            "VersionStages": ["AWSPREVIOUS"],
            "CreatedDate": "2024-01-15T10:30:00.000000+00:00"
        }
    ],
    "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/database-aBcDeF",
    "Name": "prod/myapp/database"
}
```

---

### 13. Tag a Secret

```bash
aws secretsmanager tag-resource \
  --secret-id "prod/myapp/database" \
  --tags \
    Key=CostCenter,Value=CC-12345 \
    Key=Owner,Value=backend-team@example.com \
    Key=Compliance,Value=PCI-DSS \
  --region us-east-1
```

**What it does:** Adds or overwrites tags on a secret. Tags are useful for cost allocation, access control via ABAC, and resource organization.

---

### 14. Remove Tags from a Secret

```bash
aws secretsmanager untag-resource \
  --secret-id "prod/myapp/database" \
  --tag-keys "CostCenter" "Owner" \
  --region us-east-1
```

**What it does:** Removes specified tags from a secret by their keys. Does not affect the secret value or other metadata.

---

### 15. Get the Resource Policy of a Secret

```bash
aws secretsmanager get-resource-policy \
  --secret-id "prod/myapp/database" \
  --region us-east-1
```

**What it does:** Retrieves the JSON resource-based policy attached to a secret. Returns an empty response if no policy is attached.

**Example Output:**
```json
{
    "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/database-aBcDeF",
    "Name": "prod/myapp/database",
    "ResourcePolicy": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"AWS\":\"arn:aws:iam::123456789012:role/MyAppRole\"},\"Action\":\"secretsmanager:GetSecretValue\",\"Resource\":\"*\"}]}"
}
```

---

## Common Operations

### Create Operations

**Create a secret from a file:**
```bash
# First, create the secret file
cat > /tmp/db-credentials.json << 'EOF'
{
  "host": "mydb.cluster-abc123.us-east-1.rds.amazonaws.com",
  "port": 5432,
  "dbname": "myappdb",
  "username": "dbadmin",
  "password": "S3cur3P@ssw0rd!"
}
EOF

aws secretsmanager create-secret \
  --name "prod/myapp/rds-credentials" \
  --description "Full RDS connection details for MyApp production" \
  --secret-string file:///tmp/db-credentials.json \
  --region us-east-1
```

**Create a binary secret (e.g., a certificate):**
```bash
aws secretsmanager create-secret \
  --name "prod/myapp/ssl-certificate" \
  --description "SSL certificate for MyApp" \
  --secret-binary fileb:///path/to/certificate.p12 \
  --region us-east-1
```

---

### Read Operations

**Extract just the secret value using `--query`:**
```bash
aws secretsmanager get-secret-value \
  --secret-id "prod/myapp/database" \
  --query "SecretString" \
  --output text \
  --region us-east-1
```

**Parse a JSON secret and extract a specific field:**
```bash
aws secretsmanager get-secret-value \
  --secret-id "prod/myapp/database" \
  --query "SecretString" \
  --output text \
  --region us-east-1 | python3 -c "import sys, json; print(json.load(sys.stdin)['password'])"
```

**Retrieve a secret