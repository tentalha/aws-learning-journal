# DynamoDB — Interview Questions

---

## Easy

### 1. What is Amazon DynamoDB and what type of database is it?

**Answer:**
Amazon DynamoDB is a fully managed, serverless, NoSQL database service provided by AWS. It is a key-value and document database that delivers single-digit millisecond performance at any scale. AWS handles all the underlying infrastructure, including hardware provisioning, setup, configuration, replication, software patching, and cluster scaling. DynamoDB supports both key-value and document data models and is designed for applications that require consistent, low-latency data access at massive scale.

---

### 2. What are the primary components of a DynamoDB table?

**Answer:**
The primary components of a DynamoDB table are:

- **Table**: The top-level entity, analogous to a table in a relational database.
- **Items**: Individual records in a table, analogous to rows. Each item is uniquely identifiable.
- **Attributes**: The data elements within an item, analogous to columns. Unlike relational databases, items in the same table can have different attributes.
- **Primary Key**: Uniquely identifies each item in a table. It can be:
  - **Partition Key (Simple Primary Key)**: A single attribute that DynamoDB uses to determine the partition where the item is stored.
  - **Composite Primary Key (Partition Key + Sort Key)**: A combination of two attributes that together uniquely identify an item.

---

### 3. What is the difference between a Partition Key and a Sort Key in DynamoDB?

**Answer:**

| Feature | Partition Key | Sort Key |
|---|---|---|
| **Purpose** | Determines the physical partition where data is stored | Orders items within the same partition |
| **Uniqueness** | Must be unique across all items (when used alone) | Does not need to be unique; combined with partition key must be unique |
| **Usage** | Required for all tables | Optional; used to create a composite primary key |
| **Queries** | Can only query by exact match | Supports range queries (`begins_with`, `between`, `<`, `>`, etc.) |

When a table has both a partition key and a sort key, multiple items can share the same partition key value as long as their sort keys are different. This enables powerful query patterns such as retrieving all orders for a specific customer sorted by date.

---

### 4. What are DynamoDB's two capacity modes?

**Answer:**
DynamoDB offers two read/write capacity modes:

1. **On-Demand Mode**:
   - DynamoDB automatically scales to accommodate your workload.
   - You pay per request (per Read Request Unit and Write Request Unit).
   - No capacity planning required.
   - Best for unpredictable or spiky workloads.
   - More expensive per request compared to provisioned mode at consistent load.

2. **Provisioned Mode**:
   - You specify the number of reads and writes per second (Read Capacity Units and Write Capacity Units).
   - You can use **Auto Scaling** to automatically adjust provisioned capacity.
   - More cost-effective for predictable, consistent workloads.
   - Risk of throttling if traffic exceeds provisioned capacity.

You can switch between modes once every 24 hours.

---

### 5. What is a DynamoDB Global Secondary Index (GSI) and why would you use it?

**Answer:**
A Global Secondary Index (GSI) is an index that has a partition key and an optional sort key that are **different** from the base table's primary key. It is called "global" because queries on the index can span all partitions of the base table.

**Why use a GSI:**
- To support additional query patterns beyond the base table's primary key.
- For example, if your table uses `UserID` as the partition key, but you also need to query users by `Email`, you would create a GSI with `Email` as the partition key.
- GSIs have their own provisioned throughput (or inherit on-demand mode) and are maintained asynchronously by DynamoDB.
- A table can have up to **20 GSIs** by default.

---

## Medium

### 1. Explain DynamoDB Read Consistency models. What is the difference between Eventually Consistent and Strongly Consistent reads?

**Answer:**
DynamoDB replicates data across multiple Availability Zones within an AWS Region. This replication introduces a window where data may not yet be consistent across all replicas.

**Eventually Consistent Reads:**
- Default read behavior.
- When you read data, DynamoDB may return data from a replica that hasn't yet received the latest write.
- The data will become consistent within a second or less.
- Costs **0.5 Read Capacity Units (RCU)** per 4 KB of data read.
- Best for workloads where slightly stale data is acceptable (e.g., product catalog browsing).

**Strongly Consistent Reads:**
- DynamoDB returns a result that reflects all writes that received a successful response before the read.
- Reads are served from the primary replica.
- Costs **1 RCU** per 4 KB of data read (twice the cost of eventually consistent).
- Has higher latency and lower availability than eventually consistent reads.
- Not available for GSIs.
- Best for use cases requiring the most up-to-date data (e.g., financial balances, inventory counts).

**Transactional Reads:**
- Part of DynamoDB Transactions (TransactGetItems).
- Costs **2 RCUs** per 4 KB.
- Provides ACID guarantees across multiple items.

**Practical consideration:** For most applications, eventually consistent reads are sufficient and more cost-effective. Use strongly consistent reads only when your business logic demands it.

---

### 2. What is DynamoDB Streams and what are its common use cases?

**Answer:**
DynamoDB Streams is an optional feature that captures a time-ordered sequence of item-level modifications (inserts, updates, deletes) in a DynamoDB table. The stream records are stored for **24 hours**.

**Stream Record View Types:**
- `KEYS_ONLY`: Only the key attributes of the modified item.
- `NEW_IMAGE`: The entire item as it appears after the modification.
- `OLD_IMAGE`: The entire item as it appeared before the modification.
- `NEW_AND_OLD_IMAGES`: Both the new and old images of the item.

**Common Use Cases:**

1. **Event-Driven Architectures**: Trigger AWS Lambda functions in response to table changes (e.g., send a welcome email when a new user item is inserted).
2. **Cross-Region Replication**: DynamoDB Global Tables uses Streams internally to replicate data across regions.
3. **Audit and Compliance**: Maintain a complete audit trail of all changes to sensitive data.
4. **Materialized Views / Aggregations**: Maintain aggregate counts or derived data in another table by processing stream events.
5. **Search Index Synchronization**: Propagate changes to Amazon OpenSearch Service to enable full-text search.
6. **Cache Invalidation**: Invalidate or update ElastiCache entries when underlying DynamoDB data changes.

**Integration with Lambda:**
DynamoDB Streams integrates natively with AWS Lambda as an event source mapping. Lambda polls the stream shard and invokes your function with a batch of records. You can configure batch size, starting position, and error handling behaviors.

---

### 3. How does DynamoDB handle partitioning, and what is a "hot partition" problem?

**Answer:**

**DynamoDB Partitioning:**
DynamoDB distributes data across multiple partitions (physical storage nodes) based on the partition key. It uses a consistent hashing algorithm to map partition key values to specific partitions. Each partition:
- Stores up to **10 GB** of data.
- Supports up to **3,000 RCUs** and **1,000 WCUs** of throughput.

When a table grows beyond these limits, DynamoDB automatically splits partitions and redistributes data.

**The Hot Partition Problem:**
A hot partition occurs when a disproportionate amount of traffic is directed to a single partition, causing it to exceed its throughput limits and resulting in `ProvisionedThroughputExceededException` errors (throttling).

**Causes:**
- Poor partition key selection (e.g., using a boolean field, a status field with few values, or a date field where all today's writes go to one partition).
- Time-series data where the current time period receives all writes.

**Solutions:**

1. **Choose High-Cardinality Partition Keys**: Use attributes with many distinct values (e.g., `UserID`, `OrderID`, `UUID`).
2. **Write Sharding**: Add a random suffix (e.g., 1–N) to the partition key to distribute writes across N logical partitions. Reads must then query all N shards and aggregate results.
3. **Adaptive Capacity**: DynamoDB automatically shifts capacity to frequently accessed partitions within seconds. However, this doesn't eliminate the problem for sustained hot partitions.
4. **DAX (DynamoDB Accelerator)**: Use DAX as a caching layer to absorb read traffic, reducing pressure on hot partitions.
5. **Caching**: Cache frequently read items in ElastiCache or application-level caches.

---

### 4. What are DynamoDB Transactions and when should you use them?

**Answer:**
DynamoDB Transactions (introduced in 2018) provide ACID (Atomicity, Consistency, Isolation, Durability) guarantees across multiple items and multiple tables within a single AWS account and region.

**Two Transaction APIs:**

1. **TransactWriteItems**: Atomically writes up to **100 items** across multiple tables. Supports `Put`, `Update`, `Delete`, and `ConditionCheck` operations.
2. **TransactGetItems**: Atomically reads up to **100 items** across multiple tables.

**ACID Guarantees:**
- **Atomicity**: Either all operations succeed or all fail. No partial writes.
- **Consistency**: Data remains in a valid state before and after the transaction.
- **Isolation**: Concurrent transactions don't interfere with each other (serializable isolation).
- **Durability**: Committed transactions are persisted even in the event of system failures.

**Cost:**
- Transactional writes cost **2 WCUs** per KB (vs. 1 WCU for standard writes).
- Transactional reads cost **2 RCUs** per 4 KB (vs. 1 RCU for strongly consistent reads).

**When to Use Transactions:**
- **Financial operations**: Debit one account and credit another atomically.
- **Inventory management**: Reserve inventory and create an order record simultaneously.
- **Conditional multi-item updates**: Update multiple related items only if certain conditions are met.
- **Game state management**: Update player stats, inventory, and leaderboard in a single atomic operation.

**When NOT to Use Transactions:**
- Simple single-item operations (use conditional writes with `ConditionExpression` instead — cheaper and faster).
- High-throughput scenarios where the 2x cost is prohibitive.
- Cross-region operations (transactions are region-scoped).

---

### 5. Explain DynamoDB's Local Secondary Index (LSI) vs. Global Secondary Index (GSI). What are the key differences and limitations?

**Answer:**

| Feature | Local Secondary Index (LSI) | Global Secondary Index (GSI) |
|---|---|---|
| **Partition Key** | Same as base table | Can be any attribute |
| **Sort Key** | Different from base table | Optional, can be any attribute |
| **Creation** | Must be created at table creation time | Can be added or deleted at any time |
| **Throughput** | Shares the base table's provisioned throughput | Has its own separate provisioned throughput (or on-demand) |
| **Read Consistency** | Supports both eventually and strongly consistent reads | Eventually consistent reads only |
| **Item Size Limit** | 10 GB per partition key value (shared with base table) | No partition-level size limit |
| **Scope** | Local to a single partition | Global across all partitions |
| **Maximum per Table** | 5 LSIs | 20 GSIs (default, can be increased) |
| **Projection** | Can project any attributes | Can project any attributes |

**Key Limitations:**

**LSI Limitations:**
- Cannot be added after table creation — you must plan ahead.
- The 10 GB per partition key value limit applies to the combined size of the base table and all LSI data for that partition key value. Exceeding this causes write rejections.
- Not suitable for tables where a single partition key value will accumulate more than 10 GB.

**GSI Limitations:**
- Only eventually consistent reads — cannot use strongly consistent reads.
- GSI updates are asynchronous; there can be a brief lag between base table writes and GSI updates.
- If the GSI write throughput is insufficient, writes to the base table can be throttled (GSI throttling affects the base table).
- Items that don't have the GSI partition key attribute are not included in the index.

**Best Practice:** In most modern designs, GSIs are preferred over LSIs due to their flexibility. LSIs are only warranted when you specifically need strongly consistent reads on an alternate sort key within the same partition.

---

## Hard

### 1. Deep dive into DynamoDB's internal architecture: How does consistent hashing, partition management, and the Paxos-based replication work?

**Answer:**

**Consistent Hashing for Data Distribution:**
DynamoDB uses a variant of consistent hashing to map partition key values to storage partitions. The hash function (internal, not exposed) maps each partition key to a position on a virtual hash ring. Partitions own ranges of this ring. This approach ensures:
- Relatively even data distribution when partition keys have high cardinality.
- Minimal data movement when partitions are added or split.
- Deterministic routing — the same partition key always maps to the same partition.

**Partition Splitting:**
A partition is split when it exceeds either:
- **10 GB** of storage, or
- **3,000 RCUs** or **1,000 WCUs** of throughput

When a split occurs, the hash key range is divided, and data is redistributed. Importantly, once a partition is split, it is never merged back together, even if the data is deleted. This means provisioned throughput is also divided between the new partitions, which can sometimes cause unexpected throttling (the "diluted throughput" problem).

**Replication with Multi-Paxos:**
DynamoDB stores each partition across **three replicas** in different Availability Zones. Replication uses a variant of the **Paxos consensus protocol** (specifically, Multi-Paxos) to ensure that writes are acknowledged only after a quorum (at least 2 of 3 replicas) confirms the write. This provides:
- **Durability**: Data is not acknowledged until written to a quorum.
- **Strong Consistency**: Strongly consistent reads are served from the leader replica, which has the most up-to-date data.
- **Automatic Failover**: If a replica fails, the remaining replicas continue to serve traffic. A new replica is provisioned automatically.

**Storage Engine:**
DynamoDB uses a **B-tree** based storage engine (similar to log-structured merge trees in some configurations) for efficient range queries on sort keys. The actual storage is on SSDs, providing fast I/O.

**Request Routing:**
- Client requests hit the DynamoDB **Request Router** (a fleet of stateless servers).
- The router uses a metadata service to determine which partition owns the requested key.
- The request is forwarded to the appropriate partition leader.
- For eventually consistent reads, the router may send the request to any replica.

**Write-Ahead Log (WAL):**
All writes are first recorded in a Write-Ahead Log for durability before being applied to the B-tree. This ensures that even if a node crashes mid-write, the data can be recovered.

**Adaptive Capacity:**
DynamoDB's Adaptive Capacity feature monitors access patterns and automatically redistributes throughput at the item-collection level (not just the partition level), allowing "hot" partitions to temporarily borrow capacity from underutilized partitions. This operates within seconds and is transparent to the application.

---

### 2. Explain advanced DynamoDB data modeling patterns: Single-Table Design, its benefits, trade-offs, and when to avoid it.

**Answer:**

**Single-Table Design (STD) Philosophy:**
Single-Table Design is a DynamoDB data modeling approach where all entities in an application are stored in a single DynamoDB table. This pattern leverages DynamoDB's flexible schema and composite primary keys to co-locate related data, enabling efficient access patterns with minimal API calls.

**Core Techniques:**

1. **Generic Primary Key Naming**: Use generic attribute names like `PK` (partition key) and `SK` (sort key) rather than entity-specific names, allowing different entity types to coexist.

2. **Key Overloading**: Use different formats for the same PK/SK attributes depending on the entity type:
   ```
   PK: USER#12345    SK: USER#12345          → User profile item
   PK: USER#12345    SK: ORDER#2023-01-15#001 → Order item for that user
   PK: USER#12345    SK: ADDRESS