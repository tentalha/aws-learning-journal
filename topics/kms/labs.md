# KMS — Hands-On Labs

## Lab 1: Getting Started with KMS

### Objective
In this lab, you will create your first AWS Key Management Service (KMS) Customer Managed Key (CMK), configure its key policy, and use it to encrypt and decrypt data using both the AWS Console and the AWS CLI. You will also explore key metadata, understand key rotation, and perform basic envelope encryption using the `GenerateDataKey` API.

### Prerequisites

**AWS Services Required:**
- AWS KMS
- AWS IAM
- AWS CloudShell (or local AWS CLI v2)

**IAM Permissions Required:**
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
        "kms:CreateAlias",
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:EnableKeyRotation",
        "kms:GetKeyRotationStatus",
        "kms:PutKeyPolicy",
        "kms:GetKeyPolicy",
        "kms:ScheduleKeyDeletion",
        "kms:CancelKeyDeletion"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tools Required:**
- AWS CLI v2 installed and configured (`aws configure`)
- `base64` command-line utility (pre-installed on Linux/macOS)
- A text editor
- AWS account with billing enabled

**Estimated Cost:** < $0.01 (KMS charges $1/month per CMK, prorated; lab cleanup prevents charges)

**Estimated Time:** 45–60 minutes

---

### Steps

#### Step 1: Create a Customer Managed Key (CMK) via the Console

**Console Instructions:**
1. Navigate to the [AWS KMS Console](https://console.aws.amazon.com/kms).
2. In the left navigation pane, click **Customer managed keys**.
3. Click **Create key**.
4. Configure the key:
   - **Key type:** Symmetric
   - **Key usage:** Encrypt and decrypt
   - **Key material origin:** KMS
   - Click **Next**.
5. Add key labels:
   - **Alias:** `lab1-demo-key`
   - **Description:** `KMS Lab 1 - Demo symmetric encryption key`
   - **Tags:** Key = `Environment`, Value = `Lab`
   - Click **Next**.
6. Define key administrators:
   - Select your current IAM user or role.
   - Click **Next**.
7. Define key usage permissions:
   - Select your current IAM user or role.
   - Click **Next**.
8. Review the key policy (note the JSON structure), then click **Finish**.

**CLI Instructions:**
```bash
# Step 1a: Create the CMK
KEY_ID=$(aws kms create-key \
  --description "KMS Lab 1 - Demo symmetric encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --tags TagKey=Environment,TagValue=Lab \
  --query 'KeyMetadata.KeyId' \
  --output text)

echo "Created Key ID: $KEY_ID"

# Step 1b: Create an alias for the key
aws kms create-alias \
  --alias-name alias/lab1-demo-key \
  --target-key-id $KEY_ID

echo "Alias 'alias/lab1-demo-key' created for key: $KEY_ID"
```

**Verify Step 1:**
```bash
# Describe the key to verify it was created
aws kms describe-key \
  --key-id alias/lab1-demo-key \
  --query 'KeyMetadata.{KeyId:KeyId,KeyState:KeyState,KeyUsage:KeyUsage,Description:Description}' \
  --output table
```

**Expected Output:**
```
------------------------------------------------------------------
|                         DescribeKey                            |
+-------------+----------------------------------------------+---+
|  Description| KMS Lab 1 - Demo symmetric encryption key        |
|  KeyId      | xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx             |
|  KeyState   | Enabled                                          |
|  KeyUsage   | ENCRYPT_DECRYPT                                  |
+-------------+--------------------------------------------------+
```

---

#### Step 2: Encrypt Plaintext Data

**Console Instructions:**
1. In the KMS Console, click on your key `lab1-demo-key`.
2. Note the **Key ARN** — copy it for reference.
3. (Encryption via console is limited; use CLI for full encrypt/decrypt experience.)

**CLI Instructions:**
```bash
# Step 2a: Encrypt a simple plaintext string
PLAINTEXT="Hello, KMS World! This is my secret message."

CIPHERTEXT=$(aws kms encrypt \
  --key-id alias/lab1-demo-key \
  --plaintext "$PLAINTEXT" \
  --query 'CiphertextBlob' \
  --output text)

echo "Ciphertext (Base64): $CIPHERTEXT"

# Step 2b: Save the ciphertext to a file (simulating storage)
echo $CIPHERTEXT > /tmp/encrypted_message.b64
echo "Ciphertext saved to /tmp/encrypted_message.b64"

# Step 2c: Encrypt a binary file
echo "This is a secret configuration file." > /tmp/secret_config.txt

aws kms encrypt \
  --key-id alias/lab1-demo-key \
  --plaintext fileb:///tmp/secret_config.txt \
  --output text \
  --query CiphertextBlob | base64 --decode > /tmp/secret_config.enc

echo "Encrypted file saved to /tmp/secret_config.enc"
ls -lh /tmp/secret_config.enc
```

**Verify Step 2:**
```bash
# Confirm the ciphertext is non-empty and base64-encoded
echo "Ciphertext length: ${#CIPHERTEXT} characters"
echo "First 50 chars: ${CIPHERTEXT:0:50}"

# Verify the encrypted file exists and has content
file /tmp/secret_config.enc
```

**Expected Output:**
```
Ciphertext length: 344 characters (approximately)
First 50 chars: AQICAHi...
```

---

#### Step 3: Decrypt the Encrypted Data

**CLI Instructions:**
```bash
# Step 3a: Decrypt the ciphertext blob
DECRYPTED=$(aws kms decrypt \
  --ciphertext-blob fileb://<(echo $CIPHERTEXT | base64 --decode) \
  --query 'Plaintext' \
  --output text | base64 --decode)

echo "Decrypted message: $DECRYPTED"

# Step 3b: Decrypt using the saved file
DECRYPTED_FROM_FILE=$(aws kms decrypt \
  --ciphertext-blob fileb:///tmp/secret_config.enc \
  --query 'Plaintext' \
  --output text | base64 --decode)

echo "Decrypted file contents: $DECRYPTED_FROM_FILE"

# Step 3c: Verify the key used for decryption
aws kms decrypt \
  --ciphertext-blob fileb:///tmp/secret_config.enc \
  --query '{KeyId:KeyId,EncryptionAlgorithm:EncryptionAlgorithm}' \
  --output table
```

**Verify Step 3:**
```bash
# Confirm decrypted text matches original
if [ "$DECRYPTED" = "$PLAINTEXT" ]; then
  echo "✅ SUCCESS: Decrypted text matches original plaintext!"
else
  echo "❌ FAILURE: Decrypted text does not match."
  echo "Original:  $PLAINTEXT"
  echo "Decrypted: $DECRYPTED"
fi
```

**Expected Output:**
```
Decrypted message: Hello, KMS World! This is my secret message.
✅ SUCCESS: Decrypted text matches original plaintext!
```

---

#### Step 4: Perform Envelope Encryption with GenerateDataKey

**CLI Instructions:**
```bash
# Step 4a: Generate a data key (returns plaintext + encrypted data key)
DATA_KEY_OUTPUT=$(aws kms generate-data-key \
  --key-id alias/lab1-demo-key \
  --key-spec AES_256 \
  --output json)

# Step 4b: Extract the plaintext data key (use immediately, never store!)
PLAINTEXT_DATA_KEY=$(echo $DATA_KEY_OUTPUT | python3 -c \
  "import json,sys; print(json.load(sys.stdin)['Plaintext'])")

# Step 4c: Extract the encrypted data key (safe to store)
ENCRYPTED_DATA_KEY=$(echo $DATA_KEY_OUTPUT | python3 -c \
  "import json,sys; print(json.load(sys.stdin)['CiphertextBlob'])")

echo "Plaintext Data Key (Base64): ${PLAINTEXT_DATA_KEY:0:20}..."
echo "Encrypted Data Key (Base64): ${ENCRYPTED_DATA_KEY:0:20}..."

# Step 4d: Save the encrypted data key to disk (this is safe to store)
echo $ENCRYPTED_DATA_KEY > /tmp/encrypted_data_key.b64
echo "Encrypted data key saved. Plaintext key used in memory only."

# Step 4e: Use the plaintext data key to encrypt data locally (simulated)
# In production, you'd use this with a local crypto library (e.g., OpenSSL)
echo "Simulating local encryption with plaintext data key..."
echo "Data key would be used to encrypt large files locally"
echo "Only the encrypted data key is persisted with the data"

# Step 4f: To decrypt later, first decrypt the data key via KMS
RECOVERED_DATA_KEY=$(aws kms decrypt \
  --ciphertext-blob fileb://<(cat /tmp/encrypted_data_key.b64 | base64 --decode) \
  --query 'Plaintext' \
  --output text)

echo "Recovered Data Key: ${RECOVERED_DATA_KEY:0:20}..."
echo "✅ Envelope encryption cycle complete!"
```

**Verify Step 4:**
```bash
# Confirm both data keys match
if [ "$PLAINTEXT_DATA_KEY" = "$RECOVERED_DATA_KEY" ]; then
  echo "✅ SUCCESS: Data key successfully recovered via KMS!"
else
  echo "❌ Data key mismatch"
fi
```

---

#### Step 5: Enable Automatic Key Rotation

**Console Instructions:**
1. In KMS Console, click on `lab1-demo-key`.
2. Click the **Key rotation** tab.
3. Check **Automatically rotate this KMS key every year**.
4. Click **Save**.

**CLI Instructions:**
```bash
# Step 5a: Enable automatic key rotation (annual rotation)
aws kms enable-key-rotation \
  --key-id alias/lab1-demo-key

echo "Key rotation enabled."

# Step 5b: Verify rotation status
aws kms get-key-rotation-status \
  --key-id alias/lab1-demo-key

# Step 5c: List key rotation settings with more detail
aws kms describe-key \
  --key-id alias/lab1-demo-key \
  --query 'KeyMetadata.{KeyId:KeyId,KeyState:KeyState,Origin:Origin}' \
  --output json
```

**Expected Output:**
```json
{
    "KeyRotationEnabled": true
}
```

---

### Verification

Run the following verification script to confirm all lab objectives were completed:

```bash
#!/bin/bash
echo "=== KMS Lab 1 Verification ==="

# Check 1: Key exists
KEY_STATE=$(aws kms describe-key \
  --key-id alias/lab1-demo-key \
  --query 'KeyMetadata.KeyState' \
  --output text 2>/dev/null)
[ "$KEY_STATE" = "Enabled" ] && echo "✅ CMK exists and is Enabled" || echo "❌ CMK not found"

# Check 2: Alias exists
ALIAS_EXISTS=$(aws kms list-aliases \
  --query "Aliases[?AliasName=='alias/lab1-demo-key'].AliasName" \
  --output text)
[ -n "$ALIAS_EXISTS" ] && echo "✅ Alias 'lab1-demo-key' exists" || echo "❌ Alias not found"

# Check 3: Key rotation enabled
ROTATION=$(aws kms get-key-rotation-status \
  --key-id alias/lab1-demo-key \
  --query 'KeyRotationEnabled' \
  --output text)
[ "$ROTATION" = "True" ] && echo "✅ Key rotation is enabled" || echo "❌ Key rotation not enabled"

# Check 4: Encrypt/Decrypt round-trip
TEST_MSG="verification-test"
CT=$(aws kms encrypt --key-id alias/lab1-demo-key \
  --plaintext "$TEST_MSG" --query CiphertextBlob --output text)
PT=$(aws kms decrypt \
  --ciphertext-blob fileb://<(echo $CT | base64 --decode) \
  --query Plaintext --output text | base64 --decode)
[ "$PT" = "$TEST_MSG" ] && echo "✅ Encrypt/Decrypt round-trip successful" || echo "❌ Round-trip failed"

echo "=== Verification Complete ==="
```

---

### Cleanup

> ⚠️ **Important:** KMS keys cannot be deleted immediately. The minimum waiting period is 7 days. Schedule deletion now to avoid the $1/month charge.

```bash
# Step 1: Get the Key ID
KEY_ID=$(aws kms describe-key \
  --key-id alias/lab1-demo-key \
  --query 'KeyMetadata.KeyId' \
  --output text)

echo "Scheduling deletion for Key ID: $KEY_ID"

# Step 2: Delete the alias first
aws kms delete-alias \
  --alias-name alias/lab1-demo-key
echo "✅ Alias deleted"

# Step 3: Schedule key deletion (minimum 7-day waiting period)
aws kms schedule-key-deletion \
  --key-id $KEY_ID \
  --pending-window-in-days 7

echo "✅ Key scheduled for deletion in 7 days"
echo "Key ID: $KEY_ID will be permanently deleted after the waiting period."

# Step 4: Clean up local files
rm -f /tmp/encrypted_message.b64 \
       /tmp/secret_config.txt \
       /tmp/secret_config.enc \
       /tmp/encrypted_data_key.b64
echo "✅ Local temp files cleaned up"

# Step 5: Verify scheduled deletion
aws kms describe-key \
  --key-id $KEY_ID \
  --query 'KeyMetadata.{KeyState:KeyState,DeletionDate:DeletionDate}' \
  --output table
```

**Expected Cleanup Output:**
```
-----------------------------------------------
|              DescribeKey                    |
+--------------+------------------------------+
|  DeletionDate| 2024-XX-XXTXX:XX:XX+00:00   |
|  KeyState    | PendingDeletion              |
+--------------+------------------------------+
```

> 💡 **Tip:** If you want to cancel the deletion within the 7-day window, run:
> ```bash
> aws kms cancel-key-deletion --key-id $KEY_ID
> aws kms enable-key --key-id $KEY_ID
> ```

---

## Lab 2: Intermediate KMS Configuration

### Objective
In this lab, you will implement fine-grained KMS key policies, use KMS with Amazon S3 Server-Side Encryption (SSE-KMS), configure KMS grants for cross-service access, and set up AWS CloudTrail logging to audit all KMS API calls. You will also explore encryption context to add an additional layer of security to your encrypted data.

### Prerequisites

**AWS Services Required:**
- AWS KMS
- Amazon S3
- AWS CloudTrail
- AWS IAM
- Amazon CloudWatch Logs

**IAM Permissions Required (in addition to Lab 