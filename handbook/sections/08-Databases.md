# Section 08 — Databases

> **Purpose**: Database selection is one of the most consequential architectural decisions. The wrong choice creates a drag on performance, cost, and operational overhead for years. A senior architect does not think "SQL vs NoSQL" — they think in terms of access patterns, consistency requirements, transaction semantics, scaling axes, and team expertise. This section covers RDS, Aurora, DynamoDB, and DAX with the depth needed to defend a database choice in an architecture review.
>
> **Official Documentation**: [RDS](https://docs.aws.amazon.com/rds/) | [Aurora](https://docs.aws.amazon.com/aurora/) | [DynamoDB](https://docs.aws.amazon.com/dynamodb/) | [DAX](https://docs.aws.amazon.com/dynamodb/dax/)

---

## 1. Amazon RDS: Managed Relational Databases

### 1.1 RDS's Operational Contract

RDS manages the undifferentiated heavy lifting of database administration:
- Automated provisioning, patching, and backups
- Multi-AZ failover
- Read replica creation
- Automated snapshots and point-in-time recovery
- Monitoring via CloudWatch

**What RDS does NOT manage**:
- Query optimization (you write slow queries, you get slow performance)
- Index design
- Schema migrations
- Connection management (connection pooling is your responsibility)
- Application-level caching

### 1.2 RDS Engines and Selection

| Engine | Best For | AWS-Specific Features | Migration Path |
|--------|----------|---------------------|----------------|
| **MySQL** | Open-source LAMP stacks, web apps | MariaDB compatibility, Performance Insights | Direct lift-and-shift |
| **PostgreSQL** | Complex queries, JSON features, GIS | Babelfish (SQL Server compatibility), Aurora integration | Direct lift-and-shift |
| **MariaDB** | MySQL-compatible with additional features | Drop-in replacement for MySQL | Direct lift-and-shift |
| **SQL Server** | Windows/.NET enterprise apps | License Included or BYOL, Windows Authentication via AD | Always backup/restore or native backup to S3 |
| **Oracle** | Enterprise Oracle workloads | BYOL only, RDS Custom for OS-level access | Data Pump, RMAN backups to S3 |

> **Architectural Guidance**: Default to PostgreSQL for new relational workloads unless there is a specific engine dependency. PostgreSQL has the richest feature set, best Aurora integration, and strongest community momentum.

### 1.3 Multi-AZ vs Read Replicas

These are fundamentally different features that solve different problems:

| Feature | Multi-AZ | Read Replica |
|---------|----------|-------------|
| **Purpose** | High Availability | Scaling read capacity |
| **Replication** | Synchronous (wait for standby confirmation) | Asynchronous (lag typically milliseconds to seconds) |
| **Data consistency** | Identical to primary (synchronous) | Eventually consistent (replica lag) |
| **Failover** | Automatic (DNS updated to standby) | Manual promotion (or automatic with Aurora) |
| **Can serve reads?** | No (standby is for failover only) | Yes (up to 15 replicas) |
| **Can be in different region?** | No (same region only) | Yes (cross-region replicas for DR) |
| **Can be different instance type?** | No (same type for failover compatibility) | Yes (can be larger for analytics) |

**Critical distinction**: Multi-AZ is NOT for read scaling. The standby in Multi-AZ cannot serve read traffic (except for SQL Server using Database Mirroring, which is legacy). If you need to offload read traffic, create Read Replicas.

> **RDS Proxy Connection Pooling**: RDS Proxy sits between your application and RDS. It maintains a pool of database connections and multiplexes application connections onto it. This prevents connection exhaustion when using Lambda or high-concurrency applications. RDS Proxy supports MySQL and PostgreSQL.

### 1.4 RDS Backup Architecture

| Backup Type | Retention | Recovery Point | Use Case |
|-------------|-----------|---------------|----------|
| **Automated Backups** | 1-35 days | Point-in-time (5-minute granularity) | Operational recovery, human error |
| **Manual Snapshots** | Unlimited (until deleted) | Snapshot moment only | Long-term retention, pre-migration backup |
| **Snapshot export to S3** | Unlimited | Snapshot moment only | Analytics (Athena), cross-account sharing |

**Backup window**: AWS performs backups during a configurable maintenance window. I/O may be briefly suspended for single-AZ instances (seconds). Multi-AZ backups are taken from the standby — no I/O suspension.

### 1.5 RDS Parameter Groups and Tuning

RDS exposes database parameters via **parameter groups**. Key tunables:
- `max_connections`: Total concurrent connections (default often too low for high-concurrency apps)
- `innodb_buffer_pool_size` (MySQL) / `shared_buffers` (PostgreSQL): Memory for caching data and indexes
- `log_min_duration_statement` (PostgreSQL): Log slow queries for optimization

> **RDS Limitation**: You cannot access the underlying OS (except RDS Custom, which provides OS-level access for specific use cases). This means you cannot install OS monitoring agents, custom plugins, or modify kernel parameters.

---

## 2. Amazon Aurora: Cloud-Native Relational Database

### 2.1 Aurora Architecture

Aurora is a reimagined relational database that separates compute from storage:

```
┌─────────────────────────────────────────────────────────────┐
│                     Aurora Cluster                            │
│                                                             │
│  ┌─────────────┐        ┌─────────────┐                    │
│  │   Writer    │◄──────►│   Reader    │  (up to 15)       │
│  │  (Primary)  │  shared │  (Replicas) │                    │
│  └──────┬──────┘ storage └─────────────┘                    │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │            Aurora Distributed Storage                 │    │
│  │  (6 copies across 3 AZs, 10 GB segments, SSD)       │    │
│  │  - Self-healing: auto-detects and repairs bad blocks│    │
│  │  - 99.99999% durability (6 nines)                   │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**Key Aurora innovations**:
- **Shared storage**: All instances (writer + readers) read from the same storage volume. No replication lag for storage.
- **Compute replication lag**: Reader instances may lag slightly behind the writer (typically < 20ms) because they apply redo logs from the writer.
- **Storage auto-scaling**: Grows automatically from 10 GB to 128 TB. No provisioning required.
- **Instantaneous failover**: Reader promotion to writer is typically < 30 seconds.

### 2.2 Aurora MySQL vs Aurora PostgreSQL

| Dimension | Aurora MySQL | Aurora PostgreSQL |
|-----------|-------------|-------------------|
| **Performance** | Up to 5x throughput vs MySQL RDS | Up to 3x throughput vs PostgreSQL RDS |
| **Features** | MySQL-compatible | PostgreSQL-compatible + Babelfish (SQL Server T-SQL) |
| **Global Database** | Yes (cross-region replication) | Yes |
| **Serverless v2** | Yes | Yes |
| **Use case** | High-throughput MySQL workloads | Complex queries, JSON, geospatial |

### 2.3 Aurora Global Database

Aurora Global Database replicates data across regions with **< 1 second lag**:
- **Primary region**: Handles writes.
- **Secondary regions**: Read-only replicas that can be promoted to primary for DR.
- **Storage replication**: Uses dedicated infrastructure, not database replication logs. Minimal impact on primary performance.
- **Failover**: Promote secondary to primary in < 1 minute (RTO). RPO near zero.

> **Use case**: Global applications with read-local requirements. Users in EU read from EU Aurora cluster (low latency), while writes go to US primary.

### 2.4 Aurora Serverless v2

Aurora Serverless v2 scales compute capacity automatically:
- **Minimum and maximum ACU (Aurora Capacity Units)** configurable
- **Scale in seconds** (not minutes like Serverless v1)
- **Pay per ACU-second**
- **Same underlying engine** as provisioned Aurora — no compatibility issues

**When to use Serverless v2**:
- Variable or unpredictable workloads (dev/test, seasonal apps)
- Workloads with long idle periods
- Microservices with small databases
- When you don't want to manage instance types

**When NOT to use**:
- Steady-state high-throughput workloads (provisioned is cheaper)
- Workloads requiring specific instance types or features

---

## 3. Amazon DynamoDB: Managed NoSQL

### 3.1 DynamoDB's Data Model

DynamoDB is a **key-value and document database** with a strictly defined access model:

| Concept | Description | Constraint |
|---------|-------------|------------|
| **Table** | Collection of items | No limit on number of items |
| **Item** | A record (max 400 KB) | All items in a table need not have the same attributes |
| **Primary Key** | Uniquely identifies an item | REQUIRED. Two types: simple (partition key only) or composite (partition + sort key) |
| **Partition Key** | Determines which partition stores the item | Must be high-cardinality. Hot keys = throttling. |
| **Sort Key** | Enables range queries within a partition | Optional. Enables `Query` (not just `GetItem`) |
| **Attributes** | Key-value pairs within an item | Up to 400 KB total item size |
| **Secondary Indexes** | Alternate query patterns | GSI (global, any attributes) or LSI (local, same partition key) |

### 3.2 Partition Behavior and Hot Keys

DynamoDB distributes data across partitions based on partition key hash. Each partition:
- Handles up to 3,000 RCU (read capacity units) and 1,000 WCU (write capacity units)
- Can store unlimited data but has throughput limits

**Hot key problem**: If one partition key receives disproportionate traffic, that partition throttles despite unused capacity on other partitions.

**Example**: A table with `UserID` as partition key. Celebrity user `@elonmusk` gets 10,000x more reads than average. All those reads hit ONE partition → throttling.

**Mitigations**:
1. **Write sharding**: Append random suffix to hot partition keys for writes. Query all shards in parallel.
2. **DAX**: In-memory cache for DynamoDB. Absorbs read traffic.
3. **Read replicas**: DynamoDB Global Tables or read replica regions.
4. **Caching layer**: ElastiCache in front of DynamoDB for hot data.
5. **Adaptive capacity**: DynamoDB automatically redistributes unused capacity to hot partitions, but this takes time and has limits.

### 3.3 Capacity Modes

| Mode | Billing | Scaling | Best For |
|------|---------|---------|----------|
| **On-Demand** | Per-request (read/write units) | Automatic, instant | Unpredictable traffic, new applications, sporadic workloads |
| **Provisioned** | Per capacity unit (RCU/WCU) | Manual or auto-scaling | Predictable traffic, cost optimization |

**On-Demand pricing**: Approximately 5x more expensive than well-tuned provisioned capacity. Use it when traffic is unknown or spiky. Switch to provisioned once patterns stabilize.

**Provisioned Auto-Scaling**: DynamoDB can automatically adjust RCU/WCU within configured bounds. Target utilization (default 70%) triggers scaling. Scale-up is fast; scale-down is limited to a few times per day.

### 3.4 DynamoDB Streams and Change Data Capture

DynamoDB Streams capture a time-ordered sequence of item-level modifications:

```
DynamoDB Table ──► Stream (INSERT, MODIFY, REMOVE events)
                        │
                        ├──► Lambda (trigger function per record)
                        ├──► Kinesis Data Streams (fan-out to multiple consumers)
                        └──► EventBridge (event routing)
```

**Stream record views**:
- `NEW_IMAGE`: Only the new item state
- `OLD_IMAGE`: Only the old item state
- `NEW_AND_OLD_IMAGES`: Both states
- `KEYS_ONLY`: Only the key attributes (smallest, cheapest)

**Use cases**:
- Cross-region replication (DynamoDB Global Tables use Streams internally)
- Materialized view updates
- Audit logging
- Event-driven architectures (order placed → inventory updated → notification sent)

### 3.5 Global Tables

DynamoDB Global Tables provide **multi-region, active-active replication**:
- Writes in any region replicate to all other regions (typically < 1 second)
- Conflict resolution: **last-writer-wins** (timestamp-based)
- Each region has its own RCU/WCU capacity
- Streams must be enabled

> **Conflict Resolution Limitation**: If the same item is modified simultaneously in two regions, the one with the later timestamp wins. The other write is lost. Applications requiring conflict-free concurrent writes need application-level coordination.

### 3.6 DynamoDB Transactions

DynamoDB supports ACID transactions across multiple items (up to 100 items or 4 MB):

```python
# TransactWrite: All-or-nothing writes across multiple items
transact_write_items([
    {'Update': {'TableName': 'Accounts', 'Key': {'ID': 'A'}, 'UpdateExpression': 'SET Balance = Balance - :amount'}},
    {'Update': {'TableName': 'Accounts', 'Key': {'ID': 'B'}, 'UpdateExpression': 'SET Balance = Balance + :amount'}}
])
```

**Transactional costs**: 2x the standard write cost (write is applied to each item + a transaction record). Use only when atomicity across items is truly required.

### 3.7 Single-Table Design

A DynamoDB best practice is modeling multiple entity types in a single table using composite keys and overloaded indexes:

```
Partition Key    Sort Key                          Attributes
─────────────────────────────────────────────────────────────────
USER#123         PROFILE#METADATA                  name, email, created
USER#123         ORDER#2024-01-01#001              total, status
USER#123         ORDER#2024-01-15#002              total, status
ORDER#001        ORDERMETA#DATA                    user_id, total, status
```

**Benefits**:
- Single query retrieves a user and all their orders (GSIs enable reverse lookups)
- Fewer tables to manage
- Efficient access patterns with minimal round trips

**Tradeoff**: Complex to design. Requires upfront access pattern analysis. Poorly designed single-table schemas are painful to refactor.

> **Single-table design is NOT mandatory** — it is an optimization. Start with multiple tables if your team lacks DynamoDB modeling experience. Migrate to single-table when access patterns are well understood.

---

## 4. DynamoDB Accelerator (DAX)

DAX is a **managed, in-memory cache for DynamoDB**:
- **Microsecond latency** for cached reads (vs millisecond for DynamoDB)
- **Seamless integration**: Change your DynamoDB client to DAX client — same API
- **Write-through**: DAX automatically invalidates on writes
- **Multi-AZ**: Cluster spans multiple AZs for HA

**When to use DAX**:
- Read-heavy workloads with hot keys
- Need sub-millisecond read latency
- Want cache without application code changes

**When NOT to use DAX**:
- Write-heavy workloads (DAX adds latency to writes)
- Need flexible caching logic (use ElastiCache Redis instead)
- Cost-sensitive (DAX clusters are expensive — r5.2xlarge nodes and up)

> **DAX vs ElastiCache**: DAX is simpler (no cache invalidation code) but less flexible (only caches DynamoDB). ElastiCache requires explicit cache management but works with any data source and supports more data structures.

---

## 5. Database Selection Framework

```
What are your requirements?
├── Need ACID transactions across multiple tables?
│   ├── Yes → RDS (PostgreSQL/MySQL) or Aurora
│   └── No → Continue...
├── Need complex joins, aggregations, window functions?
│   ├── Yes → RDS PostgreSQL or Aurora PostgreSQL
│   └── No → Continue...
├── Need massive scale (millions of TPS, PB of data)?
│   ├── Yes → DynamoDB (with careful key design)
│   └── No → Continue...
├── Need sub-millisecond latency with simple key-value access?
│   ├── Yes → DynamoDB (+ DAX if needed)
│   └── No → Continue...
├── Need flexible schema per item?
│   ├── Yes → DynamoDB or DocumentDB (MongoDB-compatible)
│   └── No → Continue...
├── Need full-text search?
│   ├── Yes → OpenSearch (with RDS/DynamoDB as source)
│   └── No → Continue...
├── Need graph queries?
│   ├── Yes → Neptune
│   └── No → RDS/Aurora likely fits
└── Multi-region active-active writes?
    ├── Yes → DynamoDB Global Tables or Aurora Global Database
    └── No → Single-region RDS/Aurora with cross-region read replicas
```

---

## 6. Interview Challenges

* **Scenario:** A customer wants to scale read traffic for an RDS database.
  * **Design:** Use Read Replicas, potentially combined with Multi-AZ. Because Multi-AZ is for High Availability and its standby cannot serve read traffic (except in SQL Server legacy mirroring), while Read Replicas are explicitly designed to handle read scaling (up to 15 replicas per instance). Combining both is the standard production pattern for HA and read scaling.

* **Scenario:** A DynamoDB table with 10,000 WCU and 1,000 items is experiencing throttling during writes.
  * **Design:** Mitigate hot partition issues by redesigning the partition key for higher cardinality or using write sharding. Because DynamoDB distributes capacity per partition, not globally, and a single partition is limited to 1,000 WCU. If all writes hit the same key, it throttles regardless of the table's total provisioned capacity.

* **Scenario:** Choosing between Aurora PostgreSQL and RDS PostgreSQL for a new OLTP application.
  * **Design:** Default to Aurora PostgreSQL. Because Aurora offers better performance (up to 3x throughput), auto-scaling storage without volume management, instant failover, and Serverless v2 options for variable workloads. Only choose RDS if you need specific unsupported extensions, OS-level access (RDS Custom), or if you are already on RDS and migration costs outweigh the benefits.

* **Scenario:** Designing a session store for a global web application with 10 million daily active users.
  * **Design:** Use DynamoDB with Global Tables and enable TTL on an `ExpiresAt` attribute. Because DynamoDB provides single-digit millisecond latency, automatic scaling, and TTL for automatic cleanup at massive scale, while Global Tables ensure low-latency local reads/writes across regions. Add DAX if sub-millisecond latency is needed, or use ElastiCache Redis if complex data structures are preferred.

---

## 7. Points to Remember

- **Multi-AZ is for HA, not read scaling** — the standby cannot serve reads (except SQL Server legacy mirroring).
- **Read Replicas are asynchronous** — expect replica lag (typically milliseconds, but can spike during heavy write load).
- **Aurora storage is separate from compute** — storage auto-scales to 128 TB. Compute instances can be added/removed independently.
- **Aurora Global Database provides cross-region replication with < 1 second lag** — use for DR and read-local workloads.
- **RDS Proxy prevents connection exhaustion** — essential for Lambda-to-RDS or high-concurrency applications.
- **DynamoDB partition keys must have high cardinality** — low-cardinality keys create hot partitions and throttling.
- **DynamoDB item limit is 400 KB** — for larger objects, store metadata in DynamoDB and the payload in S3.
- **DynamoDB Streams enable event-driven architectures** — but streams have a 24-hour retention. Process promptly.
- **Global Tables use last-writer-wins conflict resolution** — simultaneous writes to the same item in different regions can result in data loss.
- **On-Demand DynamoDB is ~5x the cost of provisioned** — switch to provisioned with auto-scaling once traffic patterns stabilize.
- **Single-table design is powerful but complex** — only adopt when access patterns are well understood.
- **DAX is simple but expensive** — evaluate whether ElastiCache Redis provides better flexibility for the cost.
- **PostgreSQL is the default choice for new relational workloads** — richest features, best Aurora integration, strongest community.

---

## 13. Further Reading

For deeper coverage, CLI commands, and exam-specific trivia, see the original notes:

- **RDS, Aurora, ElastiCache**: [`RDS-Aurora-ElastiCache.md`](../../original-notes/RDS-Aurora-ElastiCache.md)
- **DynamoDB, API Gateway, Step Functions, Cognito**: [`DynamoDB-APIGateway-Serverless.md`](../../original-notes/DynamoDB-APIGateway-Serverless.md)

---

*Section 08 — Databases | Last Validated: 2026-05-10*
