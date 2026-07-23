# KMS — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Ensure the AWS CLI is installed and configured before using KMS commands:

```bash
# Verify AWS CLI version (v2 recommended)
aws --version

# Configure CLI with credentials and region
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json
```

### Required IAM Permissions

Attach the following IAM policy to your user or role. KMS uses a **dual authorization model** — both IAM policies and KMS key policies must allow access.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:CreateKey",
        "kms:DescribeKey",
        "kms:ListKeys",
        "kms:ListAliases",
        "kms:EnableKey",
        "kms:DisableKey",
        "kms:ScheduleKeyDeletion",
        "kms:CancelKeyDeletion",
        "kms:CreateAlias",
        "kms:DeleteAlias",
        "kms:UpdateAlias",
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:GenerateDataKeyWithoutPlaintext",
        "kms:ReEncrypt*",
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant",
        "kms:GetKeyPolicy",
        "kms:PutKeyPolicy",
        "kms:GetKeyRotationStatus",
        "kms:EnableKeyRotation",
        "kms:DisableKeyRotation",
        "kms:TagResource",
        "kms:UntagResource",
        "kms:ListResourceTags"
      ],
      "Resource": "*"
    }
  ]
}
```

### Environment Variables (Optional)

```bash
# Set a default region for all KMS commands
export AWS_DEFAULT_REGION=us-east-1

# Use a named profile
export AWS_PROFILE=my-devops-profile

# Store frequently used Key ID in a variable
export KMS_KEY_ID="1234abcd-12ab-34cd-56ef-1234567890ab"
```

---

## Core Commands

### 1. Create a Symmetric KMS Key

```bash
aws kms create-key \
  --description "My application encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS \
  --tags TagKey=Environment,TagValue=Production TagKey=Application,TagValue=MyApp
```

**What it does:** Creates a new customer-managed KMS key (CMK) for symmetric encryption/decryption. AWS generates and manages the key material.

**Example Output:**
```json
{
    "KeyMetadata": {
        "AWSAccountId": "123456789012",
        "KeyId": "1234abcd-12ab-34cd-56ef-1234567890ab",
        "Arn": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab",
        "CreationDate": "2024-01-15T10:30:00.000000+00:00",
        "Enabled": true,
        "Description": "My application encryption key",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "KeyState": "Enabled",
        "Origin": "AWS_KMS",
        "KeyManager": "CUSTOMER",
        "CustomerMasterKeySpec": "SYMMETRIC_DEFAULT",
        "KeySpec": "SYMMETRIC_DEFAULT",
        "EncryptionAlgorithms": ["SYMMETRIC_DEFAULT"],
        "MultiRegion": false
    }
}
```

---

### 2. Create a Key Alias

```bash
aws kms create-alias \
  --alias-name alias/my-app-key \
  --target-key-id 1234abcd-12ab-34cd-56ef-1234567890ab
```

**What it does:** Creates a human-readable alias for a KMS key ID. Aliases must begin with `alias/` and cannot start with `alias/aws/` (reserved for AWS managed keys).

---

### 3. List All KMS Keys

```bash
aws kms list-keys \
  --output table
```

**What it does:** Returns a paginated list of all KMS key IDs and ARNs in the current account and region.

**Example Output:**
```
-------------------------------------------------------------------------------------------------------
|                                             ListKeys                                                |
+-----------------------------------------------------------------------------------------------------+
||                                              Keys                                                 ||
|+--------------------------------------+------------------------------------------------------------+|
||               KeyId                 |                          KeyArn                            ||
|+--------------------------------------+------------------------------------------------------------+|
||  1234abcd-12ab-34cd-56ef-1234567890ab|  arn:aws:kms:us-east-1:123456789012:key/1234abcd-...      ||
||  5678efgh-56ef-78gh-90ij-5678901234kl|  arn:aws:kms:us-east-1:123456789012:key/5678efgh-...      ||
|+--------------------------------------+------------------------------------------------------------+|
```

---

### 4. Describe a KMS Key

```bash
aws kms describe-key \
  --key-id alias/my-app-key
```

**What it does:** Returns detailed metadata about a KMS key, including its state, key spec, usage, and rotation status. You can reference a key by its ID, ARN, or alias.

**Example Output:**
```json
{
    "KeyMetadata": {
        "KeyId": "1234abcd-12ab-34cd-56ef-1234567890ab",
        "Arn": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab",
        "Enabled": true,
        "Description": "My application encryption key",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "KeyState": "Enabled",
        "KeySpec": "SYMMETRIC_DEFAULT",
        "MultiRegion": false
    }
}
```

---

### 5. Encrypt Plaintext Data

```bash
aws kms encrypt \
  --key-id alias/my-app-key \
  --plaintext "MySuperSecretPassword" \
  --output text \
  --query CiphertextBlob | base64 --decode > encrypted.bin
```

**What it does:** Encrypts up to 4 KB of plaintext data directly using the specified KMS key. The output is a Base64-encoded ciphertext blob. For larger data, use `generate-data-key` instead.

---

### 6. Decrypt Ciphertext

```bash
aws kms decrypt \
  --ciphertext-blob fileb://encrypted.bin \
  --output text \
  --query Plaintext | base64 --decode
```

**What it does:** Decrypts a ciphertext blob that was encrypted with a KMS key. KMS automatically identifies which key was used for encryption from the ciphertext metadata.

**Example Output:**
```
MySuperSecretPassword
```

---

### 7. Generate a Data Key

```bash
aws kms generate-data-key \
  --key-id alias/my-app-key \
  --key-spec AES_256 \
  --output json
```

**What it does:** Generates a unique symmetric data key for envelope encryption. Returns both the plaintext key (for immediate use) and the encrypted copy of the key. Store the encrypted key alongside your data; discard the plaintext key after use.

**Example Output:**
```json
{
    "CiphertextBlob": "AQIDAHh...base64encodedblob...==",
    "Plaintext": "base64encodedplaintextkey==",
    "KeyId": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab"
}
```

---

### 8. Enable Automatic Key Rotation

```bash
aws kms enable-key-rotation \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab
```

**What it does:** Enables automatic annual rotation of the key material for a customer-managed symmetric key. AWS KMS creates new key material every year while retaining old material to decrypt existing ciphertext.

---

### 9. Get Key Rotation Status

```bash
aws kms get-key-rotation-status \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab
```

**Example Output:**
```json
{
    "KeyRotationEnabled": true,
    "KeyId": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab",
    "RotationPeriodInDays": 365,
    "NextRotationDate": "2025-01-15T10:30:00.000000+00:00"
}
```

---

### 10. Get the Key Policy

```bash
aws kms get-key-policy \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
  --policy-name default \
  --output text | python3 -m json.tool
```

**What it does:** Retrieves the key policy document for a KMS key. `default` is the only valid policy name for customer-managed keys. Piping to `python3 -m json.tool` pretty-prints the JSON.

---

### 11. Put (Update) a Key Policy

```bash
aws kms put-key-policy \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
  --policy-name default \
  --policy file://key-policy.json
```

**What it does:** Replaces the key policy for a KMS key with the contents of `key-policy.json`. Be careful — an incorrect policy can lock you out of the key.

**Example `key-policy.json`:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow specific role to use the key",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/MyAppRole"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

### 12. Schedule Key Deletion

```bash
aws kms schedule-key-deletion \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
  --pending-window-in-days 30
```

**What it does:** Schedules a KMS key for deletion after a waiting period of 7–30 days. During the waiting period, the key is disabled and cannot be used. You can cancel deletion during this window.

**Example Output:**
```json
{
    "KeyId": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab",
    "DeletionDate": "2024-02-14T10:30:00.000000+00:00",
    "KeyState": "PendingDeletion",
    "PendingWindowInDays": 30
}
```

---

### 13. Cancel Key Deletion

```bash
aws kms cancel-key-deletion \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab
```

**What it does:** Cancels a scheduled key deletion, re-enabling the key. The key returns to the `Disabled` state and must be manually re-enabled.

---

### 14. List All Key Aliases

```bash
aws kms list-aliases \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab
```

**What it does:** Lists all aliases associated with a specific key. Omit `--key-id` to list all aliases in the account, including AWS managed key aliases.

**Example Output:**
```json
{
    "Aliases": [
        {
            "AliasName": "alias/my-app-key",
            "AliasArn": "arn:aws:kms:us-east-1:123456789012:alias/my-app-key",
            "TargetKeyId": "1234abcd-12ab-34cd-56ef-1234567890ab",
            "CreationDate": "2024-01-15T10:35:00.000000+00:00",
            "LastUpdatedDate": "2024-01-15T10:35:00.000000+00:00"
        }
    ]
}
```

---

### 15. Tag a KMS Key

```bash
aws kms tag-resource \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
  --tags TagKey=CostCenter,TagValue=CC-1234 \
         TagKey=Owner,TagValue=platform-team \
         TagKey=DataClassification,TagValue=Confidential
```

**What it does:** Adds or updates tags on a KMS key for cost allocation, access control, and organizational purposes.

---

## Common Operations

### Create Operations

```bash
# Create an asymmetric RSA key for signing
aws kms create-key \
  --key-spec RSA_4096 \
  --key-usage SIGN_VERIFY \
  --description "RSA 4096 signing key for document verification"

# Create an asymmetric RSA key for encryption
aws kms create-key \
  --key-spec RSA_2048 \
  --key-usage ENCRYPT_DECRYPT \
  --description "RSA 2048 key for asymmetric encryption"

# Create an ECC key for signing (e.g., for JWT or code signing)
aws kms create-key \
  --key-spec ECC_NIST_P256 \
  --key-usage SIGN_VERIFY \
  --description "ECC P-256 key for JWT signing"

# Create a multi-region primary key
aws kms create-key \
  --description "Multi-region primary key" \
  --multi-region \
  --key-usage ENCRYPT_DECRYPT

# Create a key with a custom key policy inline
aws kms create-key \
  --description "Key with inline policy" \
  --policy file://key-policy.json \
  --bypass-policy-lockout-safety-check

# Create an alias pointing to a key
aws kms create-alias \
  --alias-name alias/production-db-key \
  --target-key-id 1234abcd-12ab-34cd-56ef-1234567890ab
```

---

### Read / Describe Operations

```bash
# Describe a key by alias
aws kms describe-key --key-id alias/my-app-key

# Describe a key by ARN
aws kms describe-key \
  --key-id arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab

# List resource tags on a key
aws kms list-resource-tags \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab

# Get the public key for an asymmetric KMS key
aws kms get-public-key \
  --key-id alias/my-rsa-signing-