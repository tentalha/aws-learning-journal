# DynamoDB — Hands-On Labs

## Lab 1: Getting Started with DynamoDB

### Objective

In this lab, you will create your first DynamoDB table, insert items, query data, and explore the DynamoDB console. By the end, you will understand DynamoDB's core concepts: tables, items, attributes, partition keys, and sort keys. You will build a simple **product catalog** table that stores e-commerce product data.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account with billing enabled |
| IAM Permissions | `AmazonDynamoDBFullAccess` or equivalent |
| AWS CLI | Version 2.x installed and configured (`aws configure`) |
| AWS Region | `us-east-1` (used throughout this lab) |
| Tools | AWS Management Console, Terminal / Command Prompt |

Verify your CLI is configured correctly:

```bash
aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "AIDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

---

### Steps

#### Step 1: Create a DynamoDB Table

**Console:**
1. Navigate to the [DynamoDB Console](https://console.aws.amazon.com/dynamodb).
2. Click **Create table**.
3. Fill in the following:
   - **Table name:** `lab1-product-catalog`
   - **Partition key:** `ProductId` (type: `String`)
   - **Sort key:** `Category` (type: `String`)
4. Under **Table settings**, select **Customize settings**.
5. Set **Read/Write capacity mode** to **On-demand**.
6. Leave all other settings as default.
7. Click **Create table**.

**CLI:**
```bash
aws dynamodb create-table \
  --table-name lab1-product-catalog \
  --attribute-definitions \
      AttributeName=ProductId,AttributeType=S \
      AttributeName=Category,AttributeType=S \
  --key-schema \
      AttributeName=ProductId,KeyType=HASH \
      AttributeName=Category,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

**Verify:**
```bash
aws dynamodb describe-table \
  --table-name lab1-product-catalog \
  --region us-east-1 \
  --query "Table.TableStatus"
```

✅ **Expected output:** `"ACTIVE"`

---

#### Step 2: Insert Items into the Table

**Console:**
1. In the DynamoDB console, click your table `lab1-product-catalog`.
2. Click the **Explore table items** tab.
3. Click **Create item**.
4. Switch to **JSON view** and paste the following:
```json
{
  "ProductId": {"S": "PROD-001"},
  "Category": {"S": "Electronics"},
  "Name": {"S": "Wireless Headphones"},
  "Price": {"N": "79.99"},
  "Stock": {"N": "150"},
  "InStock": {"BOOL": true}
}
```
5. Click **Create item**.
6. Repeat for two more items using the JSON below.

**CLI — Insert three items:**
```bash
# Item 1
aws dynamodb put-item \
  --table-name lab1-product-catalog \
  --item '{
    "ProductId": {"S": "PROD-001"},
    "Category": {"S": "Electronics"},
    "Name": {"S": "Wireless Headphones"},
    "Price": {"N": "79.99"},
    "Stock": {"N": "150"},
    "InStock": {"BOOL": true}
  }' \
  --region us-east-1

# Item 2
aws dynamodb put-item \
  --table-name lab1-product-catalog \
  --item '{
    "ProductId": {"S": "PROD-002"},
    "Category": {"S": "Electronics"},
    "Name": {"S": "USB-C Hub"},
    "Price": {"N": "34.99"},
    "Stock": {"N": "300"},
    "InStock": {"BOOL": true}
  }' \
  --region us-east-1

# Item 3
aws dynamodb put-item \
  --table-name lab1-product-catalog \
  --item '{
    "ProductId": {"S": "PROD-003"},
    "Category": {"S": "Books"},
    "Name": {"S": "AWS Solutions Architect Guide"},
    "Price": {"N": "49.99"},
    "Stock": {"N": "75"},
    "InStock": {"BOOL": true}
  }' \
  --region us-east-1
```

**Verify:**
```bash
aws dynamodb scan \
  --table-name lab1-product-catalog \
  --region us-east-1 \
  --query "Count"
```

✅ **Expected output:** `3`

---

#### Step 3: Retrieve a Single Item with GetItem

**Console:**
1. In **Explore table items**, click **Run** to see all items.
2. Click on item `PROD-001` to view its full attributes.

**CLI:**
```bash
aws dynamodb get-item \
  --table-name lab1-product-catalog \
  --key '{
    "ProductId": {"S": "PROD-001"},
    "Category": {"S": "Electronics"}
  }' \
  --region us-east-1
```

✅ **Expected output:**
```json
{
    "Item": {
        "Stock": {"N": "150"},
        "Price": {"N": "79.99"},
        "Name": {"S": "Wireless Headphones"},
        "InStock": {"BOOL": true},
        "Category": {"S": "Electronics"},
        "ProductId": {"S": "PROD-001"}
    }
}
```

---

#### Step 4: Update an Item

**Console:**
1. Click on item `PROD-001`.
2. Click **Edit**.
3. Change the `Price` attribute value to `69.99`.
4. Click **Save and close**.

**CLI:**
```bash
aws dynamodb update-item \
  --table-name lab1-product-catalog \
  --key '{
    "ProductId": {"S": "PROD-001"},
    "Category": {"S": "Electronics"}
  }' \
  --update-expression "SET Price = :newprice, Stock = Stock - :decrement" \
  --expression-attribute-values '{
    ":newprice": {"N": "69.99"},
    ":decrement": {"N": "1"}
  }' \
  --return-values UPDATED_NEW \
  --region us-east-1
```

✅ **Expected output:**
```json
{
    "Attributes": {
        "Price": {"N": "69.99"},
        "Stock": {"N": "149"}
    }
}
```

---

#### Step 5: Query and Scan the Table

**Query** (uses the partition key — efficient):
```bash
aws dynamodb query \
  --table-name lab1-product-catalog \
  --key-condition-expression "ProductId = :pid" \
  --expression-attribute-values '{":pid": {"S": "PROD-001"}}' \
  --region us-east-1
```

**Scan with Filter** (reads entire table — use sparingly):
```bash
aws dynamodb scan \
  --table-name lab1-product-catalog \
  --filter-expression "Price < :maxprice" \
  --expression-attribute-values '{":maxprice": {"N": "50.00"}}' \
  --projection-expression "ProductId, #n, Price" \
  --expression-attribute-names '{"#n": "Name"}' \
  --region us-east-1
```

> 💡 **Note:** `Name` is a reserved word in DynamoDB, so we use an expression attribute name `#n` as an alias.

✅ **Expected output:** Items with `Price` less than `50.00` (PROD-002 and PROD-003).

---

#### Step 6: Delete an Item

**Console:**
1. Select the checkbox next to `PROD-003`.
2. Click **Actions → Delete items**.
3. Confirm deletion.

**CLI:**
```bash
aws dynamodb delete-item \
  --table-name lab1-product-catalog \
  --key '{
    "ProductId": {"S": "PROD-003"},
    "Category": {"S": "Books"}
  }' \
  --region us-east-1
```

**Verify:**
```bash
aws dynamodb scan \
  --table-name lab1-product-catalog \
  --region us-east-1 \
  --query "Count"
```

✅ **Expected output:** `2`

---

### Verification

Run the following checklist to confirm successful lab completion:

```bash
# 1. Table exists and is ACTIVE
aws dynamodb describe-table \
  --table-name lab1-product-catalog \
  --query "Table.[TableStatus, BillingModeSummary.BillingMode]" \
  --region us-east-1

# 2. Correct number of items remain
aws dynamodb scan \
  --table-name lab1-product-catalog \
  --select COUNT \
  --region us-east-1

# 3. Updated price reflects correctly
aws dynamodb get-item \
  --table-name lab1-product-catalog \
  --key '{"ProductId":{"S":"PROD-001"},"Category":{"S":"Electronics"}}' \
  --query "Item.Price" \
  --region us-east-1
```

✅ Expected results:
- Table status: `["ACTIVE", "PAY_PER_REQUEST"]`
- Item count: `2`
- Price: `{"N":"69.99"}`

---

### Cleanup

> ⚠️ **Always clean up resources to avoid unexpected charges.**

**Console:**
1. Go to DynamoDB → Tables.
2. Select `lab1-product-catalog`.
3. Click **Delete** → Type `confirm` → Click **Delete table**.

**CLI:**
```bash
aws dynamodb delete-table \
  --table-name lab1-product-catalog \
  --region us-east-1

# Verify deletion
aws dynamodb describe-table \
  --table-name lab1-product-catalog \
  --region us-east-1
```

✅ **Expected:** `ResourceNotFoundException` error confirms the table no longer exists.

---

## Lab 2: Intermediate DynamoDB Configuration

### Objective

In this lab, you will implement **Global Secondary Indexes (GSIs)**, enable **DynamoDB Streams**, configure **Time to Live (TTL)**, and integrate DynamoDB with **AWS Lambda** for stream processing. You will build a **session management system** that automatically expires old sessions and triggers downstream processing when records change.

---

### Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Active account with billing enabled |
| IAM Permissions | `AmazonDynamoDBFullAccess`, `AWSLambdaFullAccess`, `IAMFullAccess` |
| AWS CLI | Version 2.x configured |
| AWS Region | `us-east-1` |
| Runtime | Python 3.11 (for Lambda function) |
| Lab 1 | Not required — independent lab |

---

### Steps

#### Step 1: Create a Sessions Table with a GSI

**Console:**
1. Navigate to DynamoDB → **Create table**.
2. Configure:
   - **Table name:** `lab2-user-sessions`
   - **Partition key:** `SessionId` (String)
   - **Sort key:** `UserId` (String)
3. Under **Customize settings** → **Secondary indexes** → **Create global index**:
   - **Partition key:** `UserId` (String)
   - **Sort key:** `CreatedAt` (String)
   - **Index name:** `UserId-CreatedAt-index`
   - **Projection:** All
4. Set billing mode to **On-demand**.
5. Click **Create table**.

**CLI:**
```bash
aws dynamodb create-table \
  --table-name lab2-user-sessions \
  --attribute-definitions \
      AttributeName=SessionId,AttributeType=S \
      AttributeName=UserId,AttributeType=S \
      AttributeName=CreatedAt,AttributeType=S \
  --key-schema \
      AttributeName=SessionId,KeyType=HASH \
      AttributeName=UserId,KeyType=RANGE \
  --global-secondary-indexes '[
    {
      "IndexName": "UserId-CreatedAt-index",
      "KeySchema": [
        {"AttributeName": "UserId", "KeyType": "HASH"},
        {"AttributeName": "CreatedAt", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }
  ]' \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

**Verify:**
```bash
aws dynamodb describe-table \
  --table-name lab2-user-sessions \
  --query "Table.GlobalSecondaryIndexes[0].IndexStatus" \
  --region us-east-1
```

✅ **Expected output:** `"ACTIVE"`

---

#### Step 2: Enable DynamoDB Streams

**Console:**
1. Open `lab2-user-sessions` table.
2. Click the **Exports and streams** tab.
3. Under **DynamoDB stream details**, click **Enable**.
4. Select **New and old images**.
5. Click **Enable stream**.

**CLI:**
```bash
aws dynamodb update-table \
  --table-name lab2-user-sessions \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --region us-east-1
```

**Capture the Stream ARN:**
```bash
STREAM_ARN=$(aws dynamodb describe-table \
  --table-name lab2-user-sessions \
  --query "Table.LatestStreamArn" \
  --output text \
  --region us-east-1)

echo "Stream ARN: $STREAM_ARN"
```

✅ **Expected output:** An ARN like `arn:aws:dynamodb:us-east-1:123456789012:table/lab2-user-sessions/stream/2024-01-01T00:00:00.000`

---

#### Step 3: Enable Time to Live (TTL)

**Console:**
1. In the table view, click the **Additional settings** tab.
2. Under **Time to Live (TTL)**, click **Enable**.
3. Set **TTL attribute** to `ExpiresAt`.
4. Click **Enable TTL**.

**CLI:**
```bash
aws dynamodb update-time-to-live \
  --table-name lab2-user-sessions \
  --time-to-live-specification \
      Enabled=true,AttributeName=ExpiresAt \
  --region us-east-1
```

**Verify TTL is enabled:**
```bash
aws dynamodb describe-time-to-live \
  --table-name lab2-user-sessions \
  --region us-east-1
```

✅ **Expected output:**
```json
{
    "TimeToLiveDescription": {
        "TimeToLiveStatus": "ENABLED",
        "AttributeName": "ExpiresAt"
    }
}
```

---

#### Step 4: Create an IAM Role for Lambda

```bash
# Create the trust policy
cat > /tmp/lambda-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the IAM role
aws iam create-role \
  --role-name lab2-dynamodb-stream-role \
  --assume-role-policy-document file:///tmp/lambda-trust-policy.json

# Attach managed policies
aws iam attach-role-policy \
  --role-name lab2-dynamodb-stream-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambd