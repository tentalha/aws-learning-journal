# DynamoDB

## What is it?

**Amazon DynamoDB** is a fully managed, serverless, NoSQL key-value and document database service provided by AWS. It is part of the **AWS Database** service category and is designed to deliver single-digit millisecond performance at any scale.

DynamoDB is a **wide-column store** that supports both key-value and document data models. It automatically manages the underlying infrastructure — including hardware provisioning, software patching, setup, configuration, replication, and backups — so developers can focus on application logic rather than database operations.

**Key identifiers:**
- **Service Category:** AWS Database Services
- **Data Model:** Key-Value + Document (JSON)
- **Consistency Models:** Eventually Consistent and Strongly Consistent reads
- **Replication:** Multi-AZ by default (synchronous replication across 3 AZs)
- **Global Tables:** Multi-region, multi-active replication

---

## Why do we need it?

### Problems It Solves

Traditional relational databases (RDBMS) like MySQL or PostgreSQL struggle with:
- **Unpredictable traffic spikes** — scaling requires manual intervention and causes downtime
- **Rigid schemas** — changing table structure is costly and complex
- **Horizontal scaling limits** — RDBMS scales vertically, which has physical limits
- **Operational overhead** — patching, clustering, and failover management are complex

DynamoDB addresses these by offering:
- **Schema-less design** — each item can have different attributes
- **Automatic horizontal scaling** — partitions data across nodes transparently
- **Managed operations** — zero DBA effort for patching, backups, or failover
- **Predictable low latency** — consistent single-digit millisecond reads and writes

### When to Use DynamoDB

| Scenario | Why DynamoDB |
|---|---|
| High-traffic web applications | Handles millions of requests/sec with auto-scaling |
| Session management | Fast key-value lookups for user sessions |
| Gaming leaderboards | Sorted sets via sort keys and GSIs |
| IoT data ingestion | High write throughput with TTL for data expiry |
| E-commerce shopping carts | Flexible schema for varied product attributes |
| Real-time recommendation engines | Low-latency reads for serving recommendations |
| Event-driven microservices | DynamoDB Streams trigger Lambda functions |

### Real Business Scenarios

- **Amazon.com** uses DynamoDB for shopping cart and order management at massive scale
- **Lyft** uses it for real-time ride-matching data
- **Airbnb** uses it for session storage and feature flags
- **Duolingo** stores user progress and streaks in DynamoDB to handle 500M+ users

---

## Internal Working

### Data Storage and Partitioning

DynamoDB stores data in **partitions** — solid-state drive (SSD) storage units that are replicated across **three Availability Zones** within an AWS Region.

**Partition Key Hashing:**
1. When you write an item, DynamoDB applies a **hash function** (internal, not MD5) to the partition key value
2. The hash output determines which partition stores the item
3. Items with the same partition key (and different sort keys) are stored together on the same partition, sorted by sort key

**Partition Allocation Formula:**
```
Number of Partitions = MAX(
  ceil(RCU / 3000) + ceil(WCU / 1000),  -- throughput-based
  ceil(Storage / 10 GB)                  -- storage-based
)
```

### Read/Write Path

**Write Path:**
1. Client sends a `PutItem` request to DynamoDB endpoint
2. Request is routed to the **request router**
3. Router hashes the partition key to determine the target partition
4. Data is written to the **leader node** of that partition
5. Leader synchronously replicates to **two follower nodes** in other AZs
6. Once two of three nodes acknowledge, the write is confirmed (quorum write)

**Read Path (Eventually Consistent):**
1. Request routed to any of the three replica nodes
2. Returns data immediately — may not reflect the most recent write

**Read Path (Strongly Consistent):**
1. Request routed specifically to the **leader node**
2. Returns the most up-to-date data
3. Costs 2x read capacity units compared to eventually consistent reads

### Adaptive Capacity

DynamoDB uses **adaptive capacity** to automatically redistribute throughput to hot partitions. If one partition receives more traffic than its allocated RCUs/WCUs, DynamoDB borrows capacity from underutilized partitions in real time.

### Request Routing Architecture

```
Client
  │
  ▼
DynamoDB Endpoint (Regional)
  │
  ▼
Request Router (Stateless, load-balanced fleet)
  │
  ├─── Hash(PartitionKey) → Partition Location Map
  │
  ▼
Storage Node (Leader) ──── Sync Replication ──── Storage Node (Follower AZ-b)
                      └─── Sync Replication ──── Storage Node (Follower AZ-c)
```

---

## Architecture

### Core Components

#### 1. Tables
The top-level container for data. Unlike RDBMS, DynamoDB tables do not have a fixed schema (except for the primary key attributes).

#### 2. Items
Equivalent to rows in RDBMS. Each item is a collection of attributes and can have up to **400 KB** of data.

#### 3. Attributes
Name-value pairs within an item. Supported types:
- **Scalar:** String (S), Number (N), Binary (B), Boolean (BOOL), Null (NULL)
- **Document:** List (L), Map (M)
- **Set:** String Set (SS), Number Set (NS), Binary Set (BS)

#### 4. Primary Key
Every item must have a unique primary key. Two types:

| Type | Components | Use Case |
|---|---|---|
| Simple (Hash) | Partition Key only | Unique items like user profiles |
| Composite (Hash + Range) | Partition Key + Sort Key | Time-series, hierarchical data |

#### 5. Secondary Indexes

**Local Secondary Index (LSI):**
- Same partition key, different sort key
- Must be created at table creation time
- Shares throughput with the base table
- Max 5 LSIs per table
- Supports strongly consistent reads

**Global Secondary Index (GSI):**
- Different partition key and/or sort key
- Can be created or deleted at any time
- Has its own provisioned throughput (or on-demand)
- Max 20 GSIs per table (soft limit)
- Only eventually consistent reads

#### 6. DynamoDB Streams
An ordered log of all item-level changes (insert, update, delete) in a DynamoDB table. Retention period: **24 hours**.

Stream view types:
- `KEYS_ONLY` — Only key attributes
- `NEW_IMAGE` — Entire item after modification
- `OLD_IMAGE` — Entire item before modification
- `NEW_AND_OLD_IMAGES` — Both before and after states

#### 7. Global Tables
Multi-region, multi-active (active-active) replication. Any replica can accept writes and they are propagated to all other replicas asynchronously.

#### 8. DAX (DynamoDB Accelerator)
An in-memory cache cluster that sits in front of DynamoDB, reducing read latency from milliseconds to **microseconds** (up to 10x faster).

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS Region                           │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌────────────────────────┐│
│  │  Client  │───▶│   DAX    │───▶│     DynamoDB Table     ││
│  │  (App)   │    │  Cache   │    │                        ││
│  └──────────┘    └──────────┘    │  ┌──────────────────┐  ││
│                                  │  │  Partition 1     │  ││
│  ┌──────────┐                    │  │  (AZ-a,b,c)      │  ││
│  │ Lambda   │───────────────────▶│  └──────────────────┘  ││
│  │ Function │                    │  ┌──────────────────┐  ││
│  └──────────┘                    │  │  Partition 2     │  ││
│       ▲                          │  │  (AZ-a,b,c)      │  ││
│       │                          │  └──────────────────┘  ││
│  ┌────┴─────┐                    │  ┌──────────────────┐  ││
│  │ DynamoDB │                    │  │  Partition N     │  ││
│  │ Streams  │◀───────────────────│  │  (AZ-a,b,c)      │  ││
│  └──────────┘                    │  └──────────────────┘  ││
│                                  └────────────────────────┘│
│                                           │                 │
│                              ┌────────────▼──────────────┐ │
│                              │        GSI / LSI           │ │
│                              └───────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                                    │ (Global Tables)
                              ┌─────▼──────┐
                              │ AWS Region │
                              │  (Replica) │
                              └────────────┘
```

---

## Real World Example

### E-Commerce Order Management System

**Scenario:** An e-commerce platform needs to store and query customer orders efficiently. Requirements:
- Look up orders by `customerId`
- Query orders by status (pending, shipped, delivered)
- Retrieve the most recent orders for a customer
- Find all orders placed on a specific date

#### Step 1: Table Design

```
Table: Orders
  Partition Key: customerId (String)
  Sort Key: orderTimestamp (String, ISO 8601)

Attributes per item:
  - orderId (String)
  - status (String): "PENDING" | "SHIPPED" | "DELIVERED" | "CANCELLED"
  - totalAmount (Number)
  - items (List of Maps)
  - shippingAddress (Map)
  - TTL (Number): Unix timestamp for archiving old orders
```

#### Step 2: GSI for Status Queries

```
GSI: StatusIndex
  Partition Key: status (String)
  Sort Key: orderTimestamp (String)
  Projection: ALL
```

This allows querying all orders with status = "PENDING" sorted by time.

#### Step 3: Access Patterns

| Access Pattern | DynamoDB Operation |
|---|---|
| Get all orders for a customer | `Query` on base table with `customerId` |
| Get orders after a date | `Query` with `KeyConditionExpression: customerId = :cid AND orderTimestamp > :date` |
| Get all pending orders | `Query` on `StatusIndex` with `status = PENDING` |
| Get a specific order | `GetItem` with `customerId` + `orderTimestamp` |
| Update order status | `UpdateItem` with `UpdateExpression` |

#### Step 4: Sample Item Structure

```json
{
  "customerId": "CUST-12345",
  "orderTimestamp": "2024-01-15T10:30:00Z",
  "orderId": "ORD-789012",
  "status": "SHIPPED",
  "totalAmount": 149.99,
  "items": [
    { "productId": "PROD-001", "name": "Laptop Stand", "qty": 1, "price": 49.99 },
    { "productId": "PROD-002", "name": "USB Hub", "qty": 2, "price": 50.00 }
  ],
  "shippingAddress": {
    "street": "123 Main St",
    "city": "Seattle",
    "state": "WA",
    "zip": "98101"
  },
  "TTL": 1767225600
}
```

#### Step 5: Event-Driven Processing with Streams

When an order status changes to "SHIPPED":
1. `UpdateItem` is called on the Orders table
2. DynamoDB Streams captures the change (`NEW_AND_OLD_IMAGES`)
3. Lambda function triggered by the stream
4. Lambda sends a shipping notification email via Amazon SES
5. Lambda updates an analytics table in another DynamoDB table

---

## Advantages

1. **Fully Managed:** Zero operational overhead — no patching, hardware management, or manual failover
2. **Predictable Performance at Scale:** Single-digit millisecond latency regardless of data volume (terabytes to petabytes)
3. **Automatic Multi-AZ Replication:** Data is synchronously replicated across 3 AZs by default — no configuration needed
4. **Serverless with On-Demand Mode:** Pay per request with no capacity planning required
5. **Infinite Horizontal Scaling:** Automatically partitions data as throughput and storage grow
6. **Event-Driven Integration:** DynamoDB Streams + Lambda enables real-time reactive architectures
7. **ACID Transactions:** Supports `TransactWriteItems` and `TransactGetItems` for multi-item, multi-table atomic operations
8. **Point-in-Time Recovery (PITR):** Continuous backups with restore to any second within the last 35 days
9. **Global Tables:** Multi-region active-active replication for global applications with low latency worldwide
10. **Fine-Grained Access Control:** IAM condition keys allow item-level and attribute-level security
11. **TTL (Time to Live):** Automatically delete expired items without consuming write capacity
12. **Flexible Data Model:** Schema-less design allows evolving data structures without migrations
13. **DAX Integration:** In-memory cache for microsecond read latency without application changes
14. **VPC Endpoint Support:** Private connectivity without traversing the public internet

---

## Limitations

### Hard Limits

| Constraint | Limit |
|---|---|
| Maximum item size | 400 KB |
| Maximum partition key length | 2048 bytes |
| Maximum sort key length | 1024 bytes |
| Maximum attribute name length | 64 KB (but 255 bytes recommended for indexes) |
| Maximum number of LSIs per table | 5 |
| Maximum number of GSIs per table | 20 (soft limit, can be increased) |
| Maximum items per `BatchGetItem` | 100 items or 16 MB |
| Maximum items per `BatchWriteItem` | 25 items or 16 MB |
| Maximum items per `TransactGetItems` | 100 items |
| Maximum items per `TransactWriteItems` | 100 items |
| Maximum results per `Query`/`Scan` | 1 MB (before filter expressions) |
| DynamoDB Streams retention | 24 hours |
| Maximum tables per account per region | 2500 (soft limit) |
| Maximum concurrent table operations | 500 |

### Functional Limitations

- **No complex JOIN operations** — must denormalize or handle joins in application code
- **No rich SQL querying** — limited query patterns; must design access patterns upfront
- **No referential integrity** — no foreign key constraints
- **No stored procedures or triggers** (Streams + Lambda is the equivalent)
- **Scan is expensive** — full table scans consume all read capacity; avoid in production
- **GSI eventual consistency** — GSIs are always eventually consistent (no strongly consistent reads)
- **Hot partition problem** — poor partition key design leads to throttling
- **No in-place schema changes** — cannot rename attributes or change primary key after creation
- **Limited aggregation** — no native SUM, COUNT, AVG across items (must use application logic or DynamoDB Streams)
- **Maximum WCU per partition:** 1,000 WCU; **Maximum RCU per partition:** 3,000 RCU

---

## Best Practices

### Data Modeling

1. **Design for access patterns first** — Identify all read/write patterns before designing the table schema (unlike RDBMS where you normalize first)
2. **Use single-table design** — Combine multiple entity types into one table using generic attribute names (PK, SK) and type prefixes (e.g., `USER#123`, `ORDER#456`) to minimize cross-table operations
3. **Choose high-cardinality partition keys** — Use attributes with many distinct values (user ID, order ID)