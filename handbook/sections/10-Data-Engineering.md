# Section 10 — Data Engineering

> **Purpose**: Modern data architectures ingest data from hundreds of sources, process it at massive scale, and serve it to analysts, ML models, and applications with sub-second latency. This section covers ingestion, stream processing, batch analytics, and the AWS services that compose a production data platform.
>
> **Official Documentation**: [Kinesis](https://docs.aws.amazon.com/kinesis/) | [Glue](https://docs.aws.amazon.com/glue/) | [EMR](https://docs.aws.amazon.com/emr/) | [Athena](https://docs.aws.amazon.com/athena/) | [Redshift](https://docs.aws.amazon.com/redshift/)

---

## 1. Amazon Kinesis: Real-Time Data Streaming

### 1.1 Kinesis Data Streams

Kinesis Data Streams is a **distributed, ordered log** for real-time data ingestion.

| Dimension | Kinesis Data Streams | SQS FIFO |
|-----------|---------------------|----------|
| **Throughput** | MB/s per shard (configurable) | 300 msg/sec per queue |
| **Ordering** | Per shard (within a shard, records are ordered) | Per message group |
| **Retention** | 24 hours default, 365 days extended | 14 days max |
| **Replay** | Yes — consumers can re-read from any position | No |
| **Multiple consumers** | Yes — independent consumer applications | No — one consumer per message |
| **Use case** | Real-time analytics, log aggregation, event sourcing | Task queues, job processing |

**Shard mechanics**:
- Each shard provides 1 MB/s write and 2 MB/s read
- A stream has N shards; total throughput = N × shard capacity
- Records within a shard are ordered by sequence number
- Partition key determines which shard receives the record (hash of partition key modulo shard count)

**Resharding**: Split or merge shards to adjust capacity. Splitting doubles capacity for a key range; merging halves it. Resharding is online but takes time.

> **Hot shard problem**: If partition keys are poorly distributed (e.g., all records use `device_id = "default"`), one shard receives all traffic and throttles while others sit idle. Use high-cardinality partition keys.

### 1.2 Kinesis Data Firehose

Firehose is a **managed delivery service** that automatically loads streaming data to destinations:

```
Producers ──► Kinesis Data Firehose ──► S3 / Redshift / OpenSearch / Splunk / HTTP endpoint
                    │
                    └── Buffering (size + time)
                    └── Transformation (Lambda)
                    └── Format conversion (Parquet/ORC from JSON/CSV)
                    └── Compression (GZIP, Snappy)
```

**Buffering**: Firehose batches records before delivery. Configurable:
- Buffer size: 1-128 MB
- Buffer interval: 60-900 seconds

Firehose delivers when EITHER buffer size OR buffer interval is reached. Use smaller buffers for near-real-time; larger buffers for cost optimization.

### 1.3 Kinesis Data Analytics

Managed Apache Flink service for stream processing:
- SQL or Flink Java/Scala applications
- Real-time aggregations, joins, windowing
- Output to Kinesis Streams, Firehose, or Lambda

> **Modern alternative**: Managed Flink (successor to Kinesis Data Analytics). Also consider Amazon EMR with Flink or Spark Streaming for complex processing.

---

## 2. AWS Glue: Serverless Data Integration

### 2.1 Glue Components

| Component | Purpose |
|-----------|---------|
| **Glue Data Catalog** | Central metadata repository (tables, schemas, partitions). Hive-compatible. |
| **Glue ETL Jobs** | Spark-based serverless ETL. Python or Scala. |
| **Glue DataBrew** | Visual data preparation (no-code). |
| **Glue Crawlers** | Auto-discover schema from S3, RDS, DynamoDB. |
| **Glue Studio** | Visual ETL job authoring. |

### 2.2 Glue Data Catalog

The Glue Data Catalog is the **metadata layer** for your data lake:
- Athena queries use it to discover table schemas
- Redshift Spectrum uses it for external tables
- EMR Spark can use it instead of Hive metastore
- Lake Formation secures it with fine-grained access control

**Partitioning**: Store data as `s3://bucket/table/column=value/` and register partitions in the catalog. Athena and other query engines prune partitions, reading only relevant data.

> **Operational Reality**: Glue Crawlers take time and cost money. For well-understood schemas, create tables manually via CloudFormation or Terraform. Use crawlers only for exploratory data lakes.

### 2.3 Glue ETL

Glue ETL runs Apache Spark on serverless infrastructure:
- **Worker types**: G.1X (4 vCPU, 16 GB), G.2X (8 vCPU, 32 GB), G.4X, G.8X
- **DPU allocation**: Configure minimum and maximum workers. Glue scales within this range.
- **Bookmarking**: Track processed data to enable incremental ETL (process only new files).

---

## 3. Amazon EMR: Managed Hadoop/Spark

EMR provides managed clusters for big data frameworks:

| Framework | Use Case |
|-----------|----------|
| **Spark** | General-purpose big data processing, ML, streaming |
| **Hive** | SQL-like queries on HDFS/S3 data |
| **Presto/Trino** | Interactive SQL queries on federated data sources |
| **Flink** | Stream processing |
| **HBase** | NoSQL on HDFS |

**EMR deployment options**:
- **Transient clusters**: Spin up for a job, terminate after. Cost-efficient for batch workloads.
- **Persistent clusters**: Long-running for interactive analytics.
- **Serverless (EMR Serverless)**: Submit jobs without managing clusters. AWS provisions Spark/Hive workers on demand.

> **EMR vs Glue ETL**: EMR gives more control (custom Spark versions, libraries, instance types). Glue ETL is simpler and serverless. Choose EMR for complex, long-running, or custom Spark workloads. Choose Glue for standard ETL pipelines.

---

## 4. Amazon Athena: Serverless SQL Queries

Athena queries data directly in S3 using Presto/Trino. No infrastructure to manage.

**Key concepts**:
- **Tables**: Defined in Glue Data Catalog. Point to S3 locations.
- **Partitions**: Physical directory structure (`year=2026/month=05/`). Athena prunes irrelevant partitions.
- **File formats**: Parquet and ORC are fastest (columnar, compressed). CSV is slowest.
- **Cost**: $5 per TB scanned. Partitioning and columnar formats reduce scanned data dramatically.

**Athena Federated Query**: Query non-S3 sources (RDS, DynamoDB, Redshift, custom JDBC) via Lambda connectors.

> **Cost Optimization**: A 1 TB CSV scan costs $5. The same data in compressed Parquet with partitioning might scan 10 GB, costing $0.05. Format and partitioning are critical for Athena cost control.

---

## 5. Amazon Redshift: Data Warehouse

### 5.1 Redshift Architecture

Redshift is a **columnar, massively parallel processing (MPP)** data warehouse:
- **Leader node**: Receives queries, parses SQL, builds execution plan, aggregates results.
- **Compute nodes**: Store data (columnar), execute query plans in parallel.
- **Node slices**: Each compute node is divided into slices. Data is distributed across slices by distribution key.

### 5.2 Redshift Distribution Styles

| Style | Behavior | Use Case |
|-------|----------|----------|
| **AUTO** | Redshift chooses based on table size and statistics | Default. Good starting point. |
| **KEY** | Rows with same distribution key value go to same node | Large fact tables joined on a common key (e.g., `customer_id`) |
| **ALL** | Entire table copied to every node | Small dimension tables (< 3M rows) |
| **EVEN** | Round-robin distribution | No clear join key; staging tables |

### 5.3 Redshift Spectrum

Query data in S3 without loading into Redshift:
- Uses Glue Data Catalog for table definitions
- Queries execute on Redshift nodes but read from S3
- Ideal for infrequently accessed historical data
- Cost: Redshift compute + S3 GET requests + data scanned

### 5.4 Redshift Serverless

Redshift Serverless automatically provisions and scales compute:
- Pay per RPU (Redshift Processing Unit) hour
- Base capacity + max capacity configurable
- No cluster management, patching, or scaling decisions

---

## 6. Data Architecture Patterns

### Pattern: Modern Data Lake (Lake House)

```
Sources (RDS, S3, Kinesis, API) ──► Glue ETL / Spark ──► S3 (raw + curated zones)
                                                          │
                                                          ├──► Athena (ad-hoc SQL)
                                                          ├──► Redshift (BI, complex analytics)
                                                          ├──► SageMaker (ML training)
                                                          └──► QuickSight (dashboards)
```

**Zone structure**:
- **Raw**: Ingested data as-is. Never modified. Serves as source of truth.
- **Cleaned**: Deduplicated, validated, basic transformations.
- **Curated**: Business-ready tables. Optimized for query patterns (Parquet, partitioned).

### Pattern: Real-Time Analytics

```
IoT / App Events ──► Kinesis Data Streams ──► Kinesis Data Analytics (Flink)
                                                      │
                                                      ├──► Real-time dashboard (OpenSearch)
                                                      ├──► Alerts (Lambda → SNS)
                                                      └──► S3 (historical store)
```

---

## 7. Architectural Decision Challenges

* **Scenario:** Deciding between Athena and Redshift for a 50 TB data warehouse.
  * **Design:** Redshift. Because Redshift is a purpose-built MPP data warehouse with columnar storage, workload management (WLM), and predictable cluster-based costs, making it ideal for dedicated BI workloads and complex queries at scale. Use Athena only for ad-hoc exploration and smaller data lake queries since its costs scale linearly with data scanned.

* **Scenario:** Choosing between Kinesis Data Streams and Kafka for a real-time event pipeline.
  * **Design:** Kinesis Data Streams for fully managed infrastructure, a per-shard throughput model, and AWS-native integration (Lambda, Firehose, EMR). Kafka (MSK or self-managed) if you need higher throughput per partition, longer retention, have existing Kafka expertise, or require multi-cloud portability.

---

## 8. Points to Remember

- **Kinesis shard capacity**: 1 MB/s write, 2 MB/s read per shard. Scale by adding shards.
- **Kinesis partition key determines the shard** — use high-cardinality keys to avoid hot shards.
- **Firehose buffers before delivery** — configure buffer size and interval for latency vs cost tradeoff.
- **Glue Data Catalog is the metadata hub** — Athena, Redshift Spectrum, and EMR all use it.
- **Athena costs scale with data scanned** — use Parquet, Snappy compression, and partitioning to minimize costs.
- **Redshift distribution keys matter** — poor distribution causes skew and slow queries. Use KEY for large fact tables, ALL for small dimensions.
- **EMR transient clusters are cost-efficient** for batch jobs. EMR Serverless removes cluster management entirely.
- **Data lake zones: Raw → Cleaned → Curated** — never modify raw data. Build curated layers for consumption.

---

## 13. Further Reading

For deeper coverage, CLI commands, and exam-specific trivia, see the detailed reference:

- **Kinesis Data Streams, Firehose, Analytics**: [`Integration-Messaging-SQS-SNS-Kinesis.md`](../../detailed-reference/Integration-Messaging-SQS-SNS-Kinesis.md)
- **Glue, EMR, Athena, Redshift, QuickSight**: [`AdvancedIdentity-DR-OtherServices.md`](../../detailed-reference/AdvancedIdentity-DR-OtherServices.md)
- **Architecture patterns**: [`Classic-SolutionsArchitecture.md`](../../detailed-reference/Classic-SolutionsArchitecture.md)

---

*Section 10 — Data Engineering | Last Validated: 2026-05-10*
