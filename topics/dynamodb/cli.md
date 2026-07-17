# DynamoDB — AWS CLI Commands

## Setup & Configuration

### Prerequisites

- AWS CLI v2 installed (`aws --version`)
- AWS credentials configured (`aws configure` or environment variables)
- Appropriate IAM permissions attached to your user/role

### Required IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:CreateTable",
        "dynamodb:DeleteTable",
        "dynamodb:DescribeTable",
        "dynamodb:ListTables",
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:BatchWriteItem",
        "dynamodb:BatchGetItem",
        "dynamodb:UpdateTable",
        "dynamodb:TagResource",
        "dynamodb:ListTagsOfResource",
        "dynamodb:DescribeTimeToLive",
        "dynamodb:UpdateTimeToLive",
        "dynamodb:CreateBackup",
        "dynamodb:DescribeBackup",
        "dynamodb:ListBackups",
        "dynamodb:RestoreTableFromBackup",
        "dynamodb:DescribeContinuousBackups",
        "dynamodb:UpdateContinuousBackups",
        "dynamodb:ExportTableToPointInTime",
        "dynamodb:DescribeExport",
        "dynamodb:ListExports"
      ],
      "Resource": "*"
    }
  ]
}
```

### Configure Default Region

```bash
# Set a default region for all CLI commands
aws configure set region us-east-1

# Or use the --region flag per command
aws dynamodb list-tables --region us-east-1

# Set output format (json | text | table | yaml)
aws configure set output json
```

### Verify Access

```bash
# Quick connectivity test — list all tables in the current region
aws dynamodb list-tables
```

---

## Core Commands

### 1. Create a Table

```bash
aws dynamodb create-table \
  --table-name my-orders-table \
  --attribute-definitions \
      AttributeName=OrderId,AttributeType=S \
      AttributeName=CustomerId,AttributeType=S \
  --key-schema \
      AttributeName=OrderId,KeyType=HASH \
      AttributeName=CustomerId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

**What it does:** Creates a new DynamoDB table named `my-orders-table` with a composite primary key (partition key `OrderId` + sort key `CustomerId`) and on-demand billing mode.

**Example Output:**
```json
{
    "TableDescription": {
        "AttributeDefinitions": [
            { "AttributeName": "OrderId", "AttributeType": "S" },
            { "AttributeName": "CustomerId", "AttributeType": "S" }
        ],
        "TableName": "my-orders-table",
        "KeySchema": [
            { "AttributeName": "OrderId", "KeyType": "HASH" },
            { "AttributeName": "CustomerId", "KeyType": "RANGE" }
        ],
        "TableStatus": "CREATING",
        "CreationDateTime": "2024-01-15T10:30:00.000Z",
        "TableArn": "arn:aws:dynamodb:us-east-1:123456789012:table/my-orders-table",
        "TableId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "BillingModeSummary": {
            "BillingMode": "PAY_PER_REQUEST"
        }
    }
}
```

---

### 2. Describe a Table

```bash
aws dynamodb describe-table \
  --table-name my-orders-table \
  --region us-east-1
```

**What it does:** Returns detailed metadata about a table including its schema, status, provisioned throughput, indexes, and ARN.

**Example Output:**
```json
{
    "Table": {
        "TableName": "my-orders-table",
        "TableStatus": "ACTIVE",
        "ItemCount": 4821,
        "TableSizeBytes": 983040,
        "ProvisionedThroughput": {
            "ReadCapacityUnits": 0,
            "WriteCapacityUnits": 0
        },
        "BillingModeSummary": {
            "BillingMode": "PAY_PER_REQUEST"
        },
        "TableArn": "arn:aws:dynamodb:us-east-1:123456789012:table/my-orders-table"
    }
}
```

---

### 3. List All Tables

```bash
aws dynamodb list-tables \
  --region us-east-1
```

**What it does:** Returns a list of all table names in the current region for the authenticated account.

**Example Output:**
```json
{
    "TableNames": [
        "my-orders-table",
        "my-products-table",
        "my-users-table",
        "my-sessions-table"
    ]
}
```

---

### 4. Put Item (Insert / Overwrite)

```bash
aws dynamodb put-item \
  --table-name my-orders-table \
  --item '{
    "OrderId":    {"S": "ORD-20240115-001"},
    "CustomerId": {"S": "CUST-9876"},
    "Status":     {"S": "PENDING"},
    "TotalAmount":{"N": "149.99"},
    "CreatedAt":  {"S": "2024-01-15T10:30:00Z"},
    "Items":      {"L": [
      {"M": {"ProductId": {"S": "PROD-101"}, "Qty": {"N": "2"}}}
    ]}
  }' \
  --return-values ALL_OLD \
  --region us-east-1
```

**What it does:** Inserts a new item or completely replaces an existing item with the same primary key. `--return-values ALL_OLD` returns the previous item if it was overwritten.

---

### 5. Get Item (Point Lookup)

```bash
aws dynamodb get-item \
  --table-name my-orders-table \
  --key '{
    "OrderId":    {"S": "ORD-20240115-001"},
    "CustomerId": {"S": "CUST-9876"}
  }' \
  --projection-expression "OrderId, CustomerId, #st, TotalAmount" \
  --expression-attribute-names '{"#st": "Status"}' \
  --region us-east-1
```

**What it does:** Retrieves a single item by its exact primary key. Uses a projection expression to return only specific attributes and an expression attribute name to handle the reserved word `Status`.

**Example Output:**
```json
{
    "Item": {
        "OrderId":     {"S": "ORD-20240115-001"},
        "CustomerId":  {"S": "CUST-9876"},
        "Status":      {"S": "PENDING"},
        "TotalAmount": {"N": "149.99"}
    }
}
```

---

### 6. Update Item

```bash
aws dynamodb update-item \
  --table-name my-orders-table \
  --key '{
    "OrderId":    {"S": "ORD-20240115-001"},
    "CustomerId": {"S": "CUST-9876"}
  }' \
  --update-expression "SET #st = :newStatus, UpdatedAt = :ts ADD Version :inc" \
  --condition-expression "#st = :currentStatus" \
  --expression-attribute-names '{"#st": "Status"}' \
  --expression-attribute-values '{
    ":newStatus":     {"S": "SHIPPED"},
    ":currentStatus": {"S": "PENDING"},
    ":ts":            {"S": "2024-01-16T08:00:00Z"},
    ":inc":           {"N": "1"}
  }' \
  --return-values UPDATED_NEW \
  --region us-east-1
```

**What it does:** Updates specific attributes of an existing item. The condition expression ensures the update only succeeds if the current `Status` is `PENDING` (optimistic locking pattern).

---

### 7. Delete Item

```bash
aws dynamodb delete-item \
  --table-name my-orders-table \
  --key '{
    "OrderId":    {"S": "ORD-20240115-001"},
    "CustomerId": {"S": "CUST-9876"}
  }' \
  --condition-expression "#st = :cancelable" \
  --expression-attribute-names '{"#st": "Status"}' \
  --expression-attribute-values '{":cancelable": {"S": "PENDING"}}' \
  --return-values ALL_OLD \
  --region us-east-1
```

**What it does:** Deletes an item only if the condition expression is satisfied. Returns the deleted item's attributes via `ALL_OLD`.

---

### 8. Query Items

```bash
aws dynamodb query \
  --table-name my-orders-table \
  --key-condition-expression "CustomerId = :cid AND begins_with(OrderId, :prefix)" \
  --filter-expression "#st = :status" \
  --expression-attribute-names '{"#st": "Status"}' \
  --expression-attribute-values '{
    ":cid":    {"S": "CUST-9876"},
    ":prefix": {"S": "ORD-2024"},
    ":status": {"S": "SHIPPED"}
  }' \
  --index-name CustomerId-OrderId-index \
  --scan-index-forward false \
  --limit 20 \
  --region us-east-1
```

**What it does:** Queries a Global Secondary Index (GSI) for orders belonging to a specific customer, filtered to `SHIPPED` status, sorted in descending order, and limited to 20 results.

---

### 9. Scan Table

```bash
aws dynamodb scan \
  --table-name my-orders-table \
  --filter-expression "TotalAmount > :minAmount AND #st = :status" \
  --expression-attribute-names '{"#st": "Status"}' \
  --expression-attribute-values '{
    ":minAmount": {"N": "100"},
    ":status":    {"S": "PENDING"}
  }' \
  --projection-expression "OrderId, CustomerId, TotalAmount" \
  --page-size 100 \
  --max-items 500 \
  --region us-east-1
```

**What it does:** Scans the entire table and returns items matching the filter expression. Use sparingly on large tables — prefer `query` when possible.

---

### 10. Batch Write Items

```bash
aws dynamodb batch-write-item \
  --request-items '{
    "my-orders-table": [
      {
        "PutRequest": {
          "Item": {
            "OrderId":    {"S": "ORD-20240115-002"},
            "CustomerId": {"S": "CUST-1111"},
            "Status":     {"S": "PENDING"},
            "TotalAmount":{"N": "75.00"}
          }
        }
      },
      {
        "PutRequest": {
          "Item": {
            "OrderId":    {"S": "ORD-20240115-003"},
            "CustomerId": {"S": "CUST-2222"},
            "Status":     {"S": "PENDING"},
            "TotalAmount":{"N": "210.50"}
          }
        }
      },
      {
        "DeleteRequest": {
          "Key": {
            "OrderId":    {"S": "ORD-20231201-099"},
            "CustomerId": {"S": "CUST-3333"}
          }
        }
      }
    ]
  }' \
  --region us-east-1
```

**What it does:** Performs up to 25 put or delete operations in a single call. More efficient than individual API calls. Check `UnprocessedItems` in the response for any failed operations.

---

### 11. Batch Get Items

```bash
aws dynamodb batch-get-item \
  --request-items '{
    "my-orders-table": {
      "Keys": [
        {
          "OrderId":    {"S": "ORD-20240115-001"},
          "CustomerId": {"S": "CUST-9876"}
        },
        {
          "OrderId":    {"S": "ORD-20240115-002"},
          "CustomerId": {"S": "CUST-1111"}
        }
      ],
      "ProjectionExpression": "OrderId, CustomerId, TotalAmount, #st",
      "ExpressionAttributeNames": {"#st": "Status"}
    }
  }' \
  --region us-east-1
```

**What it does:** Retrieves up to 100 items (or 16 MB) from one or more tables in a single request.

---

### 12. Enable Point-in-Time Recovery (PITR)

```bash
aws dynamodb update-continuous-backups \
  --table-name my-orders-table \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true \
  --region us-east-1
```

**What it does:** Enables PITR on a table, allowing you to restore to any point within the last 35 days.

---

### 13. Create On-Demand Backup

```bash
aws dynamodb create-backup \
  --table-name my-orders-table \
  --backup-name my-orders-backup-20240115 \
  --region us-east-1
```

**What it does:** Creates a full, on-demand backup of the table. Backups are stored independently of the table and do not affect performance.

**Example Output:**
```json
{
    "BackupDetails": {
        "BackupArn": "arn:aws:dynamodb:us-east-1:123456789012:table/my-orders-table/backup/01705312200000-abc12345",
        "BackupName": "my-orders-backup-20240115",
        "BackupStatus": "CREATING",
        "BackupType": "USER",
        "BackupCreationDateTime": "2024-01-15T10:30:00.000Z"
    }
}
```

---

### 14. Update Table (Change Billing Mode / Capacity)

```bash
aws dynamodb update-table \
  --table-name my-orders-table \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=50,WriteCapacityUnits=25 \
  --region us-east-1
```

**What it does:** Switches the table from on-demand to provisioned capacity mode and sets specific read/write capacity units.

---

### 15. Delete Table

```bash
aws dynamodb delete-table \
  --table-name my-orders-table \
  --region us-east-1
```

**What it does:** Permanently deletes the table and all its data. This operation is irreversible. The table enters `DELETING` status and is removed asynchronously.

---

## Common Operations

### Create Operations

```bash
# Create a table with provisioned throughput and a Local Secondary Index (LSI)
aws dynamodb create-table \
  --table-name my-products-table \
  --attribute-definitions \
      AttributeName=ProductId,AttributeType=S \
      AttributeName=Category,AttributeType=S \
      AttributeName=Price,AttributeType=N \
  --key-schema \
      AttributeName=ProductId,KeyType=HASH \
      AttributeName=Category,KeyType=RANGE \
  --local-secondary-indexes '[
    {
      "IndexName": "Category-Price-index",
      "KeySchema": [
        {"AttributeName": "ProductId", "KeyType": "HASH"},
        {"AttributeName": "Price",     "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }
  ]' \
  