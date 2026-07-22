# Secrets Manager — Hands-On Labs

## Lab 1: Getting Started with Secrets Manager

### Objective
In this lab, you will learn the fundamentals of AWS Secrets Manager by creating, retrieving, and rotating a simple database credential secret. By the end of this lab, you will understand how to store sensitive configuration data securely, retrieve secrets programmatically, and understand the basic secret lifecycle.

**What you will build:**
- A manually managed secret containing database credentials
- An IAM policy granting least-privilege access to the secret
- A simple Python script that retrieves and uses the secret at runtime

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account with billing enabled |
| IAM Permissions | `secretsmanager:*`, `iam:CreatePolicy`, `iam:AttachUserPolicy` |
| AWS CLI | Version 2.x installed and configured (`aws configure`) |
| Python | Version 3.8+ with `boto3` installed (`pip install boto3`) |
| Region | `us-east-1` (all examples use this region) |

**Verify your CLI is configured:**
```bash
aws sts get-caller-identity
```
Expected output:
```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

---

### Steps

#### Step 1: Create Your First Secret via the AWS Console

**Console:**
1. Navigate to **AWS Secrets Manager** in the AWS Console.
2. Click **Store a new secret**.
3. On the **Choose secret type** page:
   - Select **Other type of secret**.
   - Under **Key/value pairs**, add the following pairs:

     | Key | Value |
     |---|---|
     | `username` | `db_admin` |
     | `password` | `MyS3cur3P@ssw0rd!` |
     | `host` | `mydb.example.com` |
     | `port` | `5432` |
     | `dbname` | `production_db` |

4. Leave **Encryption key** as `aws/secretsmanager` (default AWS managed key).
5. Click **Next**.
6. For **Secret name**, enter: `lab1/myapp/database-credentials`
7. For **Description**, enter: `Primary database credentials for myapp`
8. Add a tag: Key = `Environment`, Value = `Lab`
9. Click **Next**, skip rotation for now, click **Next** again, then **Store**.

**CLI equivalent:**
```bash
aws secretsmanager create-secret \
  --name "lab1/myapp/database-credentials" \
  --description "Primary database credentials for myapp" \
  --secret-string '{
    "username": "db_admin",
    "password": "MyS3cur3P@ssw0rd!",
    "host": "mydb.example.com",
    "port": "5432",
    "dbname": "production_db"
  }' \
  --tags '[{"Key":"Environment","Value":"Lab"}]' \
  --region us-east-1
```

**Verify after this step:**
```bash
aws secretsmanager describe-secret \
  --secret-id "lab1/myapp/database-credentials" \
  --region us-east-1
```

Expected output (abbreviated):
```json
{
    "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:lab1/myapp/database-credentials-AbCdEf",
    "Name": "lab1/myapp/database-credentials",
    "Description": "Primary database credentials for myapp",
    "LastChangedDate": "2024-01-15T10:00:00.000000+00:00",
    "Tags": [{"Key": "Environment", "Value": "Lab"}]
}
```

---

#### Step 2: Retrieve the Secret Value

**Console:**
1. In Secrets Manager, click on `lab1/myapp/database-credentials`.
2. Scroll to **Secret value** and click **Retrieve secret value**.
3. Observe that the secret is displayed in **Key/value** format.
4. Click **Plaintext** to see the raw JSON.

**CLI — Retrieve the full secret:**
```bash
aws secretsmanager get-secret-value \
  --secret-id "lab1/myapp/database-credentials" \
  --region us-east-1
```

Expected output:
```json
{
    "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:lab1/myapp/database-credentials-AbCdEf",
    "Name": "lab1/myapp/database-credentials",
    "VersionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "SecretString": "{\"username\":\"db_admin\",\"password\":\"MyS3cur3P@ssw0rd!\",\"host\":\"mydb.example.com\",\"port\":\"5432\",\"dbname\":\"production_db\"}",
    "VersionStages": ["AWSCURRENT"],
    "CreatedDate": "2024-01-15T10:00:00.000000+00:00"
}
```

**CLI — Extract a specific field using jq:**
```bash
aws secretsmanager get-secret-value \
  --secret-id "lab1/myapp/database-credentials" \
  --region us-east-1 \
  --query 'SecretString' \
  --output text | jq -r '.username'
```

Expected output:
```
db_admin
```

---

#### Step 3: Retrieve the Secret Programmatically with Python

Create a file named `retrieve_secret.py`:

```python
import boto3
import json
from botocore.exceptions import ClientError


def get_database_credentials(secret_name: str, region_name: str = "us-east-1") -> dict:
    """
    Retrieve database credentials from AWS Secrets Manager.

    Args:
        secret_name: The name or ARN of the secret.
        region_name: AWS region where the secret is stored.

    Returns:
        Dictionary containing the database credentials.
    """
    session = boto3.session.Session()
    client = session.client(
        service_name="secretsmanager",
        region_name=region_name
    )

    try:
        response = client.get_secret_value(SecretId=secret_name)
    except ClientError as e:
        error_code = e.response["Error"]["Code"]
        if error_code == "DecryptionFailureException":
            print("ERROR: Secrets Manager cannot decrypt using the provided KMS key.")
        elif error_code == "InternalServiceErrorException":
            print("ERROR: An internal service error occurred.")
        elif error_code == "InvalidParameterException":
            print("ERROR: Invalid parameter provided.")
        elif error_code == "InvalidRequestException":
            print("ERROR: Invalid request for current state of resource.")
        elif error_code == "ResourceNotFoundException":
            print(f"ERROR: Secret '{secret_name}' was not found.")
        raise e

    # Parse the secret string into a dictionary
    secret = json.loads(response["SecretString"])
    return secret


def main():
    secret_name = "lab1/myapp/database-credentials"
    region = "us-east-1"

    print(f"Retrieving secret: {secret_name}")
    credentials = get_database_credentials(secret_name, region)

    # Use credentials (never print passwords in production!)
    print(f"✅ Successfully retrieved credentials for user: {credentials['username']}")
    print(f"   Host: {credentials['host']}")
    print(f"   Port: {credentials['port']}")
    print(f"   Database: {credentials['dbname']}")
    print(f"   Password retrieved: {'*' * len(credentials['password'])}")

    # Simulate using the credentials to connect
    connection_string = (
        f"postgresql://{credentials['username']}:****"
        f"@{credentials['host']}:{credentials['port']}/{credentials['dbname']}"
    )
    print(f"\n   Connection string (masked): {connection_string}")


if __name__ == "__main__":
    main()
```

**Run the script:**
```bash
python retrieve_secret.py
```

Expected output:
```
Retrieving secret: lab1/myapp/database-credentials
✅ Successfully retrieved credentials for user: db_admin
   Host: mydb.example.com
   Port: 5432
   Database: production_db
   Password retrieved: ******************

   Connection string (masked): postgresql://db_admin:****@mydb.example.com:5432/production_db
```

---

#### Step 4: Update the Secret Value

**Console:**
1. In Secrets Manager, select your secret.
2. Under **Secret value**, click **Edit**.
3. Change the `password` value to `NewS3cur3P@ssw0rd!`
4. Click **Save**.

**CLI:**
```bash
aws secretsmanager put-secret-value \
  --secret-id "lab1/myapp/database-credentials" \
  --secret-string '{
    "username": "db_admin",
    "password": "NewS3cur3P@ssw0rd!",
    "host": "mydb.example.com",
    "port": "5432",
    "dbname": "production_db"
  }' \
  --region us-east-1
```

**Verify the update — check version history:**
```bash
aws secretsmanager list-secret-version-ids \
  --secret-id "lab1/myapp/database-credentials" \
  --region us-east-1
```

Expected output (two versions now exist):
```json
{
    "Versions": [
        {
            "VersionId": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
            "VersionStages": ["AWSCURRENT"],
            "LastAccessedDate": "2024-01-15T00:00:00+00:00"
        },
        {
            "VersionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
            "VersionStages": ["AWSPREVIOUS"],
            "LastAccessedDate": "2024-01-15T00:00:00+00:00"
        }
    ]
}
```

---

#### Step 5: Create a Least-Privilege IAM Policy for Secret Access

**Create the policy document:**
```bash
# First, get your secret ARN
SECRET_ARN=$(aws secretsmanager describe-secret \
  --secret-id "lab1/myapp/database-credentials" \
  --query 'ARN' \
  --output text \
  --region us-east-1)

echo "Secret ARN: $SECRET_ARN"
```

**Create the IAM policy file `secretsmanager-readonly-policy.json`:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowGetSecretValue",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:lab1/myapp/database-credentials-*"
    }
  ]
}
```

> ⚠️ **Note:** Replace `123456789012` with your actual AWS account ID. The wildcard `*` at the end accounts for the random suffix Secrets Manager appends.

**Apply the policy using CLI:**
```bash
aws iam create-policy \
  --policy-name "Lab1-SecretsManager-ReadOnly" \
  --policy-document file://secretsmanager-readonly-policy.json \
  --description "Read-only access to lab1 database credentials secret"
```

Expected output:
```json
{
    "Policy": {
        "PolicyName": "Lab1-SecretsManager-ReadOnly",
        "PolicyId": "ANPAXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::123456789012:policy/Lab1-SecretsManager-ReadOnly",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "IsAttachable": true,
        "CreateDate": "2024-01-15T10:00:00+00:00",
        "UpdateDate": "2024-01-15T10:00:00+00:00"
    }
}
```

---

### Verification

Run the following checklist to confirm successful lab completion:

```bash
#!/bin/bash
echo "=== Lab 1 Verification ==="

# 1. Verify secret exists
echo -n "1. Secret exists: "
aws secretsmanager describe-secret \
  --secret-id "lab1/myapp/database-credentials" \
  --region us-east-1 \
  --query 'Name' \
  --output text 2>/dev/null && echo "✅ PASS" || echo "❌ FAIL"

# 2. Verify secret has two versions
echo -n "2. Secret has multiple versions: "
VERSION_COUNT=$(aws secretsmanager list-secret-version-ids \
  --secret-id "lab1/myapp/database-credentials" \
  --region us-east-1 \
  --query 'length(Versions)' \
  --output text 2>/dev/null)
[ "$VERSION_COUNT" -ge 2 ] && echo "✅ PASS ($VERSION_COUNT versions)" || echo "❌ FAIL"

# 3. Verify IAM policy exists
echo -n "3. IAM policy created: "
aws iam get-policy \
  --policy-arn "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/Lab1-SecretsManager-ReadOnly" \
  --query 'Policy.PolicyName' \
  --output text 2>/dev/null && echo "✅ PASS" || echo "❌ FAIL"

# 4. Verify secret retrieval works
echo -n "4. Secret retrieval works: "
aws secretsmanager get-secret-value \
  --secret-id "lab1/myapp/database-credentials" \
  --region us-east-1 \
  --query 'SecretString' \
  --output text 2>/dev/null | jq -e '.username' > /dev/null && echo "✅ PASS" || echo "❌ FAIL"

echo "=== Verification Complete ==="
```

---

### Cleanup

> ⚠️ **Important:** AWS Secrets Manager charges approximately $0.40/secret/month. Run cleanup immediately after completing the lab.

```bash
# Step 1: Delete the secret (with 7-day recovery window — default)
aws secretsmanager delete-secret \
  --secret-id "lab1/myapp/database-credentials" \
  --recovery-window-in-days 7 \
  --region us-east-1

# Step 2: To delete immediately (no recovery window) — use with caution!
aws secretsmanager delete-secret \
  --secret-id "lab1/myapp/database-credentials" \
  --force-delete-without-recovery \
  --region us-east-1

# Step 3: Delete the IAM policy
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws iam delete-policy \
  --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/Lab1-SecretsManager-ReadOnly"

# Step 4: Verify cleanup
echo "Verifying cleanup..."
aws secretsmanager list-secrets \
  --filter Key=name,Values=lab1 \
  --region us-east-1 \
  --query 'SecretList[].Name'
```

Expected output after cleanup:
```json
[]
```

---

## Lab 2: Intermediate Secrets Manager Configuration

### Objective
In this lab, you will configure automatic secret rotation using AWS Lambda, integrate Secrets Manager with Amazon RDS, set up CloudWatch monitoring and alerting for