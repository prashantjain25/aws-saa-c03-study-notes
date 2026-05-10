# AWS SAA-C03: DynamoDB, API Gateway & Serverless Architecture
## Comprehensive Study Guide with Real-World Analogies

**Last Updated:** March 6, 2026  
**Tutor Style:** Expert-led Approach — WHY not just WHAT  
**Exam Focus:** Solutions Architect Associate (SAA-C03)

---

## Table of Contents
1. [Amazon DynamoDB — Overview](#1-amazon-dynamodb--overview)
2. [DynamoDB Basics: Tables, Keys & Items](#2-dynamodb-basics)
3. [Read/Write Capacity Modes](#3-readwrite-capacity-modes)
4. [DynamoDB Accelerator (DAX)](#4-dynamodb-accelerator-dax)
5. [DynamoDB Streams & Change Data Capture](#5-dynamodb-streams--change-data-capture)
6. [DynamoDB Global Tables](#6-dynamodb-global-tables)
7. [DynamoDB TTL & Backup Strategies](#7-dynamodb-ttl--backup-strategies)
8. [API Gateway — Foundations](#8-api-gateway--foundations)
9. [API Gateway Integrations & Endpoints](#9-api-gateway-integrations--endpoints)
10. [API Gateway Security & Auth](#10-api-gateway-security--auth)
11. [AWS Step Functions](#11-aws-step-functions)
12. [Amazon Cognito](#12-amazon-cognito)
13. [Architecture Diagrams](#13-architecture-diagrams)
14. [AWS CLI Quick Reference](#14-aws-cli-quick-reference)
15. [Interview Tips & Exam Hacks](#15-interview-tips--exam-hacks)
16. [Official AWS Documentation](#16-official-aws-documentation)

---

## 1. Amazon DynamoDB — Overview

### Why DynamoDB? Real-World Context

Imagine you're building a mobile gaming platform where millions of players score points across hundreds of games simultaneously. You need a database that:
- **Scales horizontally** without you thinking about it (Netflix has trillions of rows)
- **Never locks** — multiple players can write simultaneously (DynamoDB uses optimistic locking)
- **Is always on** — built-in replication across availability zones (99.99% uptime)
- **Costs predictably** — pay only for what you use (no surprise bills)

That's DynamoDB.

### What Makes DynamoDB Different from SQL Databases?

| Aspect | DynamoDB | RDS (SQL) |
|--------|----------|-----------|
| **Schema** | Schema-free (add attributes anytime) | Fixed schema (migrations needed) |
| **Scaling** | Horizontal (automatic sharding) | Vertical (bigger server) + Read replicas |
| **Query Model** | Key-value + document | SQL with JOINs |
| **Transactions** | Limited (all-or-nothing on items) | Full ACID support |
| **Consistency** | Eventually/Strongly consistent (configurable) | Always strongly consistent |
| **Cost Model** | Request-based or provisioned | Time-based (monthly bill) |

### Key Characteristics

- **Fully Managed:** AWS handles patching, replication, backups — you just use it
- **Replication:** Automatic across 3+ AZs in a region (synchronous writes)
- **Performance:** Single-digit millisecond latency (p99 < 10ms)
- **Scale:** Millions of requests/second, trillions of rows, 100s of terabytes
- **Table Classes:**
  - **Standard:** Default, for hot/warm data
  - **Infrequent Access (IA):** 25-60% cheaper storage, for rarely accessed data (backups, archives)

### Points to Remember: DynamoDB Overview
✓ Fully managed, serverless NoSQL database  
✓ Multi-AZ by default (no replication configuration needed)  
✓ Sub-millisecond latency on properly designed queries  
✓ Schema-flexible (add attributes dynamically)  
✓ No JOINs — design for access patterns upfront  
✓ Auto-scales or use fixed provisioned throughput  

---

## 2. DynamoDB Basics: Tables, Keys & Items

### Real-World Analogy: Library Card System

Think of DynamoDB like a library:
- **Table** = Library section (e.g., Fiction books)
- **Partition Key** = Library card number (identifies which person)
- **Sort Key** = Book ISBN (which book they checked out)
- **Item** = A checkout record (library card + ISBN + due date, condition, etc.)
- **Attribute** = Each piece of information (due date is an attribute)

### Table Structure

```
┌──────────────────────────────────────────────────────────┐
│ Users_Games (Table)                                       │
├────────────────┬─────────────┬──────────┬────────────────┤
│ UserId (HASH)  │ GameId (SK) │  Score   │ PlayedAt       │
├────────────────┼─────────────┼──────────┼────────────────┤
│ user_001       │ game_A      │  9500    │ 2026-03-01     │
├────────────────┼─────────────┼──────────┼────────────────┤
│ user_001       │ game_B      │  15200   │ 2026-03-02     │
├────────────────┼─────────────┼──────────┼────────────────┤
│ user_002       │ game_A      │  8900    │ 2026-02-28     │
└────────────────┴─────────────┴──────────┴────────────────┘
```

### Primary Key Design (THE MOST IMPORTANT CONCEPT)

#### Option 1: Partition Key ONLY (Simple Primary Key)
- Uniquely identifies each item
- Must be unique across entire table
- Example: `UserEmail` in a Users table
- **Use when:** Each item is independent, no range queries needed

```
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions AttributeName=Email,AttributeType=S \
  --key-schema AttributeName=Email,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

#### Option 2: Partition Key + Sort Key (Composite Primary Key)
- **Partition Key:** Determines which partition (shard) the item lives in
- **Sort Key:** Sorts items within that partition
- **Combination** must be unique
- **Enables range queries:** "Get all game scores for user_001 where GameId >= 'game_A'"

```
aws dynamodb create-table \
  --table-name Users_Games \
  --attribute-definitions \
    AttributeName=UserId,AttributeType=S \
    AttributeName=GameId,AttributeType=S \
  --key-schema \
    AttributeName=UserId,KeyType=HASH \
    AttributeName=GameId,KeyType=RANGE \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

### Why Partition Key Matters

DynamoDB hashes the Partition Key to decide which server stores the item. If you always query by partition key → **fast, predictable performance**. If you always query by sort key alone → **table scan (slow)**.

```
User Data Distribution:
hash(user_001) → Partition #1
hash(user_002) → Partition #5
hash(user_003) → Partition #12
(Each partition is a separate server, auto-balanced)
```

### Data Types

| Category | Types | Example |
|----------|-------|---------|
| **Scalar** | String (S), Number (N), Binary (B), Boolean (BOOL), Null | "John", 42, true |
| **Document** | Map (M), List (L) | {name: "John", scores: [100, 200]} |
| **Set** | String Set (SS), Number Set (NS), Binary Set (BS) | ["red", "blue", "green"] |

### Maximum Item Size: 400 KB
- If your item exceeds 400 KB, split into multiple items or use S3

```bash
# Put item example
aws dynamodb put-item \
  --table-name Users_Games \
  --item '{
    "UserId": {"S": "user_001"},
    "GameId": {"S": "game_A"},
    "Score": {"N": "9500"},
    "Badges": {"SS": ["speedrun", "perfect-score"]},
    "Metadata": {"M": {
      "Platform": {"S": "mobile"},
      "Version": {"N": "2"}
    }}
  }'
```

### Points to Remember: Keys & Items
✓ Partition Key determines data distribution (which server)  
✓ Sort Key enables range queries and sorting  
✓ Max item size is 400 KB (use S3 for larger data)  
✓ Schema is flexible — add attributes later anytime  
✓ Partition Key + Sort Key combination must be unique  
✓ No JOINs in DynamoDB — design table for access patterns  

---

## 3. Read/Write Capacity Modes

### Why Two Modes? Cost Optimization

Different workloads need different billing models:
- **Predictable traffic (e.g., daily report generation at noon)** → Provisioned mode (cheaper)
- **Spiky traffic (e.g., flash sale, viral content)** → On-Demand mode (scales automatically)

### Provisioned Mode (Default)

**You specify upfront:** How many reads & writes per second you expect.

#### Read Capacity Units (RCU)
- **1 RCU** = 1 strongly consistent read of 4 KB per second
  - OR 2 eventually consistent reads of 4 KB per second
- **Formula:** If you need 100 KB/sec of reads:
  - Strongly consistent: 100 ÷ 4 = 25 RCU
  - Eventually consistent: 100 ÷ (4 × 2) = 12.5 RCU (round up to 13)

#### Write Capacity Units (WCU)
- **1 WCU** = 1 write of 1 KB per second
- **Formula:** If you write 50 KB/sec:
  - 50 ÷ 1 = 50 WCU

#### Why Provisioned?
- **Cheaper for predictable workloads** (~$0.47/WCU/month, ~$0.09/RCU/month)
- **Predictable costs** (no surprise scaling)
- **Supports auto-scaling** (adjust RCU/WCU automatically when traffic spikes)

```bash
# Create table with provisioned throughput
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions AttributeName=UserId,AttributeType=S \
  --key-schema AttributeName=UserId,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=100,WriteCapacityUnits=50

# Enable auto-scaling
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Users" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --min-capacity 10 \
  --max-capacity 200
```

### On-Demand Mode (PAY_PER_REQUEST)

**No capacity planning:** You pay per read/write request, requests scale automatically.

- **Cost:** ~$1.25 per 1 million reads, ~$6.25 per 1 million writes (2-3x more expensive per request)
- **No throttling:** Automatically scales to handle any traffic
- **Perfect for:** New applications (unpredictable), spiky workloads, minimal traffic

```bash
# Create table with on-demand billing
aws dynamodb create-table \
  --table-name Events \
  --attribute-definitions AttributeName=EventId,AttributeType=S \
  --key-schema AttributeName=EventId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### Provisioned vs On-Demand Decision Tree

```
Is your traffic PREDICTABLE?
├─ YES → Use Provisioned mode + auto-scaling
│        (Save money, predictable bills)
│
└─ NO → Use On-Demand mode
         (Pay more per request, zero capacity planning)

Do you have UNPREDICTABLE SPIKES?
├─ YES → On-Demand (automatically scales)
│
└─ NO → Provisioned with auto-scaling
```

### Points to Remember: Capacity Modes
✓ Provisioned = cheaper for predictable traffic  
✓ On-Demand = more expensive per request, but auto-scales  
✓ Can switch between modes every 24 hours  
✓ RCU = 4 KB strongly consistent OR 8 KB eventually consistent  
✓ WCU = 1 KB per second  
✓ Auto-scaling works ONLY with provisioned mode  
✓ On-Demand = no throttling, guaranteed to scale  

---

## 4. DynamoDB Accelerator (DAX)

### The Problem: Read Bottleneck

You have a DynamoDB table with hot items (e.g., leaderboard top 10). Every read still takes ~5-10 milliseconds from DynamoDB. Can we go faster?

**YES: Use DAX (DynamoDB Accelerator).**

### What is DAX?

DAX is a **fully managed, in-memory cache specifically for DynamoDB** that sits between your application and DynamoDB.

```
Application → DAX (microseconds) → DynamoDB (milliseconds)
```

- **Latency:** < 1 millisecond (microsecond range) vs 5+ ms from DynamoDB
- **Transparent:** Your app code doesn't change — DAX API is 100% compatible with DynamoDB SDK
- **Default TTL:** 5 minutes (items automatically expire)
- **Use case:** Solves the "hot partition" problem (one item that everyone reads)

### DAX Architecture

```
┌─────────────┐
│ Application │
└──────┬──────┘
       │ (DynamoDB SDK call)
       ▼
┌─────────────────────┐
│   DAX Cluster       │
│  (In-Memory Cache)  │
└──────┬──────────────┘
       │ (Cache miss → fetch)
       ▼
┌──────────────────────┐
│    DynamoDB Table    │
└──────────────────────┘
```

### DAX vs ElastiCache: The Key Distinction

This is a COMMON EXAM QUESTION. Know the difference:

| Aspect | DAX | ElastiCache |
|--------|-----|-------------|
| **Purpose** | Cache DynamoDB items & queries | General application cache |
| **What it caches** | DynamoDB objects, query/scan results | Any application data (session, leaderboard, computed values) |
| **Integration** | Works ONLY with DynamoDB | Works with anything (app business logic, RDS, etc.) |
| **Latency** | Microseconds | Milliseconds |
| **Example use case** | Leaderboard reads (hot items) | Computed leaderboard ranking, session data |

**EXAM TIP:** If the question says "DynamoDB read performance problem" → **DAX**. If it says "we compute the leaderboard and want to cache it" → **ElastiCache**.

### Points to Remember: DAX
✓ In-memory cache for DynamoDB only  
✓ Microsecond latency (faster than DynamoDB)  
✓ Transparent to application (API-compatible)  
✓ Solves hot partition/read congestion  
✓ Default TTL = 5 minutes  
✓ DAX ≠ ElastiCache (DAX is DynamoDB-specific)  

---

## 5. DynamoDB Streams & Change Data Capture

### The Problem: React to Data Changes

You insert an order into DynamoDB. You need to:
1. Validate inventory
2. Create shipping label
3. Send confirmation email
4. Update analytics dashboard

How do you trigger these actions?

**Answer: DynamoDB Streams**

### What is DynamoDB Streams?

A **time-ordered stream of item modifications** (create, update, delete) that happened in your table.

- **Retention:** 24 hours
- **View Types:**
  - `KEYS_ONLY`: Just the key attributes
  - `NEW_IMAGE`: Item after the update
  - `OLD_IMAGE`: Item before the update
  - `NEW_AND_OLD_IMAGES`: Both (most useful)
- **Limited consumers:** Few concurrent applications can read streams
- **Processing:** Lambda (via event mapping), KCL Adapter, Kinesis Data Streams

```bash
# Enable DynamoDB Streams on a table
aws dynamodb update-table \
  --table-name Orders \
  --stream-specification \
    StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

### Architecture: DynamoDB Streams

```
┌──────────────┐
│ Orders Table │ (item inserted)
└───────┬──────┘
        │
        ▼
┌──────────────────────────────┐
│   DynamoDB Streams           │ (24 hour retention)
│   {KEYS_ONLY, NEW_AND_OLD}   │
└───────┬──────────────────────┘
        │
        ├──► Lambda (Event Mapping) ──► SNS ──► Email
        │
        ├──► KCL Adapter ──► Another DDB Table
        │
        └──► Kinesis Adapter ──► Redshift
```

### Kinesis Data Streams for DynamoDB (Better Alternative)

If 24 hours isn't enough, use **Kinesis Data Streams** instead:

- **Retention:** Up to 1 year (configurable)
- **More consumers:** Scales to many concurrent readers
- **Processing options:**
  - Lambda
  - Kinesis Data Analytics (SQL queries on stream)
  - Kinesis Firehose (to S3, Redshift, etc.)
  - AWS Glue

```
DynamoDB → Kinesis Data Streams → Kinesis Firehose → S3 (for analytics)
                                                   → Redshift (data warehouse)
                                                   → OpenSearch (indexing)
```

### Points to Remember: Streams
✓ DynamoDB Streams = 24-hour retention (default)  
✓ Kinesis Streams = up to 1-year retention  
✓ Enable StreamViewType=NEW_AND_OLD_IMAGES for full audit trail  
✓ Lambda can auto-trigger from DDB Streams (event mapping)  
✓ Limited consumers for DDB Streams (use Kinesis for many readers)  
✓ Streams must be enabled to use Global Tables  

---

## 6. DynamoDB Global Tables

### The Problem: Global Users, Local Latency

Your gaming app has users in US, Europe, and Asia. You could replicate to multiple regions, but:
- How do you keep data synchronized?
- Can you write in both regions?
- What if a region goes down?

**Answer: DynamoDB Global Tables (Active-Active Replication)**

### What is Global Tables?

A **multi-region, fully replicated table where you can READ AND WRITE in any region**.

```
Region: us-east-1 (Primary)
  Users_Games Table (Writable)
        │ (replicates automatically)
        ▼
Region: eu-west-1 (Replica)
  Users_Games Table (ALSO Writable)
        │ (replicates automatically)
        ▼
Region: ap-southeast-1 (Replica)
  Users_Games Table (ALSO Writable)
```

### Key Features

- **Active-Active:** Write to ANY region, read from ANY region
- **Low latency:** Users in EU read from eu-west-1 (not from us-east-1)
- **Auto-sync:** Changes replicate within seconds
- **Conflicts resolved:** Last-write-wins (timestamp-based)
- **Prerequisite:** DynamoDB Streams MUST be enabled

### Use Cases

1. **Global gaming platform:** Players score points in their region, results visible globally
2. **Disaster recovery:** If us-east-1 fails, app continues in eu-west-1
3. **Compliance:** GDPR data stays in EU, other data in US

```bash
# Enable Streams first (prerequisite)
aws dynamodb update-table \
  --table-name Users \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

# Create Global Table
aws dynamodb create-global-table \
  --global-table-name Users \
  --replication-group RegionName=us-east-1 RegionName=eu-west-1
```

### Points to Remember: Global Tables
✓ Active-Active replication (write anywhere)  
✓ Low latency for global users  
✓ DynamoDB Streams must be enabled  
✓ Conflict resolution = last-write-wins  
✓ Changes replicate within seconds  
✓ Unlike RDS (which is Active-Passive for reads only)  

---

## 7. DynamoDB TTL & Backup Strategies

### TTL: Automatic Expiration

Imagine a shopping cart table. Abandoned carts from 30 days ago are useless — why store them?

**TTL (Time To Live)** automatically deletes items after expiry.

```bash
# Enable TTL on Sessions table
aws dynamodb update-time-to-live \
  --table-name Sessions \
  --time-to-live-specification "Enabled=true, AttributeName=ExpiresAt"
```

#### How it Works

1. Create a `Number` attribute with Unix epoch timestamp
2. Set TTL to that attribute name
3. DynamoDB deletes items when current time > attribute value
4. Deletion happens **within 48 hours** (eventually consistent)

#### Use Cases

- **Sessions:** Expire old user sessions
- **Temp data:** Caches, tokens, OTPs
- **Compliance:** GDPR (right to be forgotten)
- **Cost optimization:** Don't pay storage for old data

```
Item created at 2026-03-01
ExpiresAt = 1743465600 (2026-03-30 in Unix time)
Current time: 2026-04-01
DynamoDB deletes item ✓
No storage charges for deleted items ✓
```

### Backup Strategies for Disaster Recovery

#### Option 1: PITR (Point-in-Time Recovery)

**Continuous backups** — recover to ANY point in the last 35 days.

```bash
# Enable PITR
aws dynamodb update-continuous-backups \
  --table-name Users \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true

# Restore to a specific point (creates new table)
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Users \
  --target-table-name Users-Restored \
  --restore-date-time 2026-03-05T14:30:00
```

**Pros:**
- Fine-grained recovery (second-level precision)
- No manual backup needed

**Cons:**
- Only 35 days
- Creates new table (can't restore in-place)

#### Option 2: On-Demand Backups

**Full backups** that you manually trigger and keep indefinitely.

```bash
# Create on-demand backup
aws dynamodb create-backup \
  --table-name Users \
  --backup-name users-backup-20260306

# Restore from backup
aws dynamodb restore-table-from-backup \
  --target-table-name Users-Restored \
  --backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users/backup/01234567890123
```

**Use for:** Compliance (keep 7 years), major migrations, regulatory requirements

#### Option 3: AWS Backup Service

**Cross-region backups** with lifecycle management.

- Centralized backup management
- Backup to another region (DR)
- Retention policies (e.g., delete after 1 year)
- Compliance automation

### DynamoDB Export to S3

Export table data WITHOUT using Read Capacity Units (non-blocking).

```bash
# Export to S3 (supports DynamoDB JSON, ION format)
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users \
  --s3-bucket my-export-bucket \
  --s3-prefix users-export/ \
  --export-time 1646433000
```

**Use for:**
- Data analysis with Athena
- ETL pipelines
- Historical audits
- No performance impact

### Points to Remember: TTL & Backups
✓ TTL attribute must be Number type (Unix timestamp)  
✓ Deletion within 48 hours (eventually consistent)  
✓ PITR = last 35 days, second-level precision  
✓ On-Demand Backups = unlimited retention  
✓ Export to S3 = no RCU consumption  
✓ Restore always creates NEW table (not in-place)  

---

## 8. API Gateway — Foundations

### Why API Gateway?

You built a Lambda function. Now you need to expose it to mobile apps, web browsers, and third-party developers. You need:
- **HTTPS endpoint** (mobile apps won't talk to plain HTTP)
- **Rate limiting** (prevent abuse)
- **API versioning** (v1, v2 without breaking old clients)
- **Request transformation** (convert JSON to XML)
- **Caching** (popular endpoints shouldn't hit Lambda every time)
- **Authentication** (only valid users access)

That's **API Gateway.**

### What is API Gateway?

A **fully managed, serverless API layer** that sits between clients and backend (Lambda, HTTP endpoints, AWS services).

```
┌──────────────┐
│  Mobile App  │
│  Web Browser │
│  Third-party │
└──────┬───────┘
       │ HTTPS
       ▼
┌──────────────────────┐
│   API Gateway        │ (CORS, auth, throttling, caching)
└──────┬───────────────┘
       │
       ├──► Lambda Function
       ├──► HTTP Endpoint (ALB, on-prem)
       ├──► AWS Service (Kinesis, SQS, Step Functions)
       └──► Mock Response
```

### Three API Types

| Type | Use Case | Protocol |
|------|----------|----------|
| **REST API** | Standard APIs (CRUD operations) | HTTP/REST |
| **HTTP API** | Modern, faster REST APIs | HTTP/REST + gRPC |
| **WebSocket API** | Real-time bi-directional communication | WebSocket (ws://, wss://) |

### Features

- **CORS:** Enable cross-origin requests (JavaScript from other domains)
- **API Keys:** Track & throttle by client
- **Stages:** Separate environments (dev, test, prod)
- **Documentation:** Auto-generated Swagger/OpenAPI specs
- **SDK Generation:** Auto-generate SDKs for Python, JavaScript, Java
- **Caching:** Cache responses per stage (default 300 seconds)
- **Custom Domain:** Use your own domain with ACM certificate

```bash
# Create REST API
aws apigateway create-rest-api \
  --name MyGameAPI \
  --description "Game scores API" \
  --endpoint-configuration types=REGIONAL

# Create stage
aws apigateway create-deployment \
  --rest-api-id abc123xyz \
  --stage-name prod \
  --stage-description "Production"
```

### Points to Remember: API Gateway Basics
✓ Fully managed, serverless API layer  
✓ Three types: REST, HTTP, WebSocket  
✓ No infrastructure to manage (no EC2, ALB needed)  
✓ Supports multiple backends (Lambda, HTTP, AWS services)  
✓ Built-in caching, CORS, throttling  
✓ Auto-scales to millions of requests/second  

---

## 9. API Gateway Integrations & Endpoints

### Integration Types: How API Gateway Calls Backends

#### Type 1: Lambda Function (Most Common)

API Gateway invokes Lambda synchronously, returns response to client.

```
Client → API Gateway → Lambda → Response
         (invoke)      (execute) (return)
```

**Best for:** Serverless applications, microservices, business logic

```bash
# AWS CLI doesn't directly create integrations—use CloudFormation
# But here's conceptually how it works:
# PUT {api-id}/resources/{resource-id}/methods/GET/integration
# Integration Type: AWS_PROXY (Lambda Proxy Integration)
# URI: arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:123456789012:function:MyFunction/invocations
```

**Pros:**
- Simple, serverless
- Full control over logic
- Scales automatically

**Cons:**
- Adds Lambda invocation latency
- Cold starts possible

#### Type 2: HTTP (HTTP Endpoints)

API Gateway calls any HTTP endpoint — on-prem servers, ALB, third-party APIs.

```
Client → API Gateway → HTTP Endpoint (ALB / On-Prem / API)
                     → Response
```

**Best for:** Exposing legacy systems, load balancers, hybrid architectures

```bash
# HTTP Integration Example:
# URI: https://internal-api.company.com/orders
# HTTP Method: POST
# Request transformer: map API Gateway request → HTTP endpoint format
```

**Pros:**
- Works with existing infrastructure
- No refactoring needed

**Cons:**
- Depends on HTTP endpoint availability
- More latency than direct Lambda

#### Type 3: AWS Service Integration (Direct)

API Gateway directly calls AWS services — **no Lambda needed**.

```
Client → API Gateway → AWS Service (SQS, Kinesis, DynamoDB, Step Functions)
```

**Examples:**

```
1. Client → API GW → Kinesis Data Streams → Firehose → S3
   (Store events without Lambda)

2. Client → API GW → SQS Queue → Lambda consumer
   (Async processing without API invoking Lambda)

3. Client → API GW → Step Functions → Start workflow
   (Orchestrate workflow directly from API)

4. Client → API GW → DynamoDB → Get item
   (Simple CRUD without business logic layer)
```

**Best for:** Simple operations, decoupling client from Lambda, high throughput

```bash
# AWS Service Integration Example:
# Service: DynamoDB
# Action: GetItem
# Request mapping: Transform API request → DynamoDB GetItem format
# Response mapping: Transform DynamoDB response → JSON
```

**Pros:**
- Zero Lambda overhead
- Lower latency
- Direct to AWS service

**Cons:**
- Limited to what the service supports
- No custom logic

### Endpoint Types: Where API Gateway Runs

#### Type 1: Edge-Optimized (Default)

API Gateway deployed in **ONE region**, but requests routed globally through **CloudFront edge locations**.

```
┌────────────────────────────────────────┐
│         CloudFront Edge Network        │
│  (London, Tokyo, Sydney, São Paulo)    │
└─────────────────┬──────────────────────┘
                  │
                  ▼ (traffic concentrated here)
         ┌────────────────────┐
         │  API Gateway       │
         │  (us-east-1 only)  │
         └────────────────────┘
```

**Best for:** Global users, low latency from anywhere

**Note:** Custom HTTPS certificate MUST be in `us-east-1` (CloudFront requirement)

```bash
aws apigateway create-domain-name \
  --domain-name api.example.com \
  --certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/abc123 \
  --endpoint-configuration types=EDGE_OPTIMIZED
```

#### Type 2: Regional

API Gateway deployed in the **same region as clients**. You manage your own CloudFront if needed.

```
┌────────────────────────────┐
│  API Gateway               │
│  (eu-west-1)              │
└────────────────────────────┘
```

**Best for:** Regional applications, lower costs, more control

#### Type 3: Private

Only accessible from **within a VPC** using VPC Endpoint (ENI).

```
┌─────────────────────────────┐
│ VPC                         │
│ ┌─────────────────────────┐ │
│ │  EC2 / Lambda (in VPC)  │ │
│ └────────┬────────────────┘ │
│          │ (private)        │
│ ┌────────▼────────────────┐ │
│ │  API Gateway Endpoint   │ │
│ └────────────────────────┘ │
└─────────────────────────────┘
```

**Best for:** Internal microservices, intranet APIs, security

### Points to Remember: Integrations & Endpoints
✓ Lambda = most common, best for business logic  
✓ HTTP = legacy systems, ALBs, hybrid  
✓ AWS Service = direct access, no Lambda overhead  
✓ Edge-Optimized = global CDN via CloudFront  
✓ Regional = control over CDN, lower cost  
✓ Private = VPC-only, intranet  
✓ Custom domain cert: us-east-1 for Edge-Optimized, same region for Regional  

---

## 10. API Gateway Security & Auth

### Four Authentication Methods

#### Method 1: IAM Roles (Internal Apps)

For **AWS resources** calling your API (EC2, Lambda, on-prem with IAM role).

```
EC2 Instance → Signs request with IAM Creds → API Gateway → validates Sig V4 → Lambda
```

**How it works:**
- EC2/Lambda signs request with AWS Signature Version 4
- API Gateway verifies signature = signature is valid

**Cost:** None (no additional charges)

```bash
# EC2/Lambda code example (AWS SDK handles signing automatically):
# import boto3
# client = boto3.client('apigateway')
# response = client.make_request(...)  # SDK automatically signs request
```

**Best for:** Microservices, internal APIs, AWS-to-AWS

#### Method 2: Amazon Cognito (External Users)

For **mobile/web users** signing in with username/password, Facebook, Google, etc.

```
Mobile App → User Pool (sign-in) → Returns JWT → API Gateway → validates JWT → Lambda
```

**How it works:**
1. User signs in via Cognito User Pool (username/password or social)
2. Cognito returns JWT token
3. App sends JWT in Authorization header to API Gateway
4. API Gateway validates JWT with User Pool
5. Request reaches Lambda

**Zero custom auth code!** API Gateway handles validation.

```bash
# Create User Pool
aws cognito-idp create-user-pool \
  --pool-name MyAppUsers \
  --policies '{"PasswordPolicy":{"MinimumLength":8,"RequireNumbers":true}}'

# Attach to API Gateway:
# Authorization Type: Cognito User Pool
# User Pool: <select your pool>
```

**Best for:** Mobile/web apps, social login, user management

#### Method 3: Custom Authorizer (Lambda Authorizer)

For **custom authentication** (OAuth 2.0, custom tokens, JWT, external IdP).

```
Client → API Gateway → Lambda Authorizer → validates token → returns IAM policy → request proceeds
```

**How it works:**
1. Client sends request with custom token (header, query param)
2. API Gateway calls Lambda Authorizer function
3. Lambda validates token (JWT decode, database lookup, external call)
4. Lambda returns IAM policy (allow/deny + resource ARN)
5. API Gateway either allows or denies request

**Highly flexible** — you control validation logic.

```bash
# Lambda Authorizer Example (Python)
def lambda_handler(event, context):
    token = event['authorizationToken']
    
    # Validate token (JWT decode, DB lookup, etc.)
    if is_valid(token):
        policy = {
            "principalId": user_id,
            "policyDocument": {
                "Version": "2012-10-17",
                "Statement": [{
                    "Action": "execute-api:Invoke",
                    "Effect": "Allow",
                    "Resource": event['methodArn']
                }]
            }
        }
    else:
        policy = {"principalId": "user", "policyDocument": {"Statement": []}}
    
    return policy
```

**Best for:** OAuth 2.0, custom auth flows, third-party IdP integration

#### Method 4: API Keys (NOT for authentication)

API Keys track usage & throttle by client — **not security**.

```bash
# Create API Key (for rate limiting, not auth)
aws apigateway create-api-key \
  --name Client1Key \
  --enabled

# Create Usage Plan (throttling + quota)
aws apigateway create-usage-plan \
  --name BasicPlan \
  --throttle burstLimit=5000,rateLimit=1000 \
  --quota limit=1000000,period=MONTH
```

**Use for:** Tracking client usage, rate limiting, NOT replacing real auth

### HTTPS Custom Domain

```bash
# Step 1: Get certificate from ACM
aws acm request-certificate \
  --domain-name api.example.com \
  --region us-east-1  # MUST be us-east-1 for Edge-Optimized

# Step 2: Create custom domain
aws apigateway create-domain-name \
  --domain-name api.example.com \
  --certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/abc123 \
  --endpoint-configuration types=EDGE_OPTIMIZED

# Step 3: Create Route 53 alias record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "d1234.cloudfront.net",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

### Points to Remember: API Gateway Security
✓ IAM = internal AWS resources (no extra cost)  
✓ Cognito = external users, social login, zero auth code  
✓ Custom Authorizer = custom auth logic, OAuth, external IdP  
✓ API Keys = usage tracking, NOT authentication  
✓ Custom domain cert: us-east-1 for Edge-Optimized  
✓ JWT validation happens automatically (Cognito)  
✓ Custom Authorizer = flexible, but you code the validation  

---

## 11. AWS Step Functions

### The Problem: Orchestrating Multi-Step Workflows

You need to process an order:
1. Validate order (Lambda)
2. Reserve inventory (DynamoDB)
3. Process payment (external API call)
4. Wait for shipping label (3-5 minutes)
5. Send confirmation email
6. If anything fails, refund and notify

You could write Lambda → call another Lambda → if error, do this → etc. **That's fragile, hard to debug, error-prone.**

### What is Step Functions?

A **serverless workflow orchestration service** that manages the flow of tasks, retries, error handling, and state.

```
┌─────────────────────────────────────────┐
│  AWS Step Functions (Visual Workflow)    │
│                                         │
│  ┌──────────┐                          │
│  │ Validate │                          │
│  └─────┬────┘                          │
│        │                               │
│  ┌─────▼──────────┐                   │
│  │ Reserve Inventory                  │
│  └─────┬──────────┘                   │
│        │                               │
│  ┌─────▼──────────┐                   │
│  │ Process Payment│                   │
│  └─────┬──────────┘                   │
│        │                               │
│  ┌─────▼──────────────┐               │
│  │ Wait 3-5 min (Shipping)            │
│  └─────┬──────────────┘               │
│        │                               │
│  ┌─────▼──────────────┐               │
│  │ Send Confirmation  │               │
│  └────────────────────┘               │
│                                         │
│  On Error: Refund & Notify             │
│                                         │
└─────────────────────────────────────────┘
```

### Key Features

- **Serverless:** No infrastructure to manage
- **Visual:** Design workflow in UI or JSON (Amazon States Language)
- **Retries & Error Handling:** Exponential backoff, catch-and-handle errors
- **Parallel Execution:** Run tasks in parallel, join results
- **Wait States:** Delay, wait for external event, scheduled time
- **Human Approval:** Pause workflow, wait for human approval
- **Max Duration:** 1 year for Standard workflows
- **Integrations:** Lambda, EC2, ECS, SQS, SNS, DynamoDB, API Gateway, on-prem servers

### State Machine Types

```json
{
  "Comment": "Order Processing Workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ValidateOrder",
      "Next": "CheckInventory"
    },
    "CheckInventory": {
      "Type": "Task",
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Inventory",
      "Next": "ProcessPayment"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:states:us-east-1:123456789012:stateMachine:PaymentProcessor",
      "Retry": [{
        "ErrorEquals": ["States.TaskFailed"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3,
        "BackoffRate": 2.0
      }],
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "RefundAndNotify"
      }],
      "Next": "SendConfirmation"
    },
    "SendConfirmation": {
      "Type": "Task",
      "Resource": "arn:aws:sns:us-east-1:123456789012:OrderConfirmed",
      "End": true
    },
    "RefundAndNotify": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:RefundAndNotify",
      "End": true
    }
  }
}
```

### Creating & Running State Machine

```bash
# Create state machine
aws stepfunctions create-state-machine \
  --name order-workflow \
  --definition file://state-machine.json \
  --role-arn arn:aws:iam::123456789012:role/StepFunctionsRole

# Start execution
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:123456789012:stateMachine:order-workflow \
  --name order-001-execution \
  --input '{"orderId": "order-001", "amount": 99.99}'

# List executions
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:us-east-1:123456789012:stateMachine:order-workflow \
  --status-filter RUNNING
```

### Real-World Use Cases

1. **Order Processing:** Validate → Reserve → Pay → Ship
2. **Data Pipeline:** Extract → Transform → Load (ETL)
3. **Content Approval:** Submit → Review → Approve → Publish
4. **Microservices Orchestration:** Chain multiple microservices
5. **Machine Learning Workflow:** Prepare data → Train → Evaluate → Deploy

### Points to Remember: Step Functions
✓ Serverless workflow orchestrator  
✓ Handles retries, error handling, parallel execution  
✓ Max duration = 1 year (Standard workflows)  
✓ Integrates with Lambda, EC2, ECS, SQS, DynamoDB, etc.  
✓ Visual designer or JSON state machine  
✓ Exponential backoff for retries  
✓ Human approval workflow support  

---

## 12. Amazon Cognito

### What is Cognito? Two Completely Different Services

**Cognito = two separate services with confusing names. Know the difference on the exam!**

### Service 1: Cognito User Pools (Authentication)

A **serverless user database** for managing sign-ups, sign-ins, and user identity.

#### What Problems Does It Solve?

- Don't want to build user sign-up page (password hashing, email verification)
- Need social login (Facebook, Google, SAML)
- Need MFA support
- Want managed user directory

#### Features

- **Sign-up/Sign-in:** Self-service user registration
- **Email/Phone Verification:** Automatic verification flows
- **MFA:** Multi-factor authentication (TOTP, SMS)
- **Password Policies:** Enforce complexity requirements
- **Federated Identity:** Facebook, Google, Twitter, SAML 2.0, OpenID Connect
- **User Attributes:** Store custom user data (profile, preferences)
- **JWT Tokens:** Returns tokens for authenticated sessions

```bash
# Create User Pool
aws cognito-idp create-user-pool \
  --pool-name GameAppUsers \
  --policies '{
    "PasswordPolicy": {
      "MinimumLength": 8,
      "RequireUppercase": true,
      "RequireLowercase": true,
      "RequireNumbers": true,
      "RequireSymbols": false
    }
  }' \
  --mfa-configuration OPTIONAL \
  --account-recovery-setting RecoveryMechanisms='[{Name=verified_email,Priority=1},{Name=verified_phone_number,Priority=2}]'

# Create User Pool Client (your app)
aws cognito-idp create-user-pool-client \
  --user-pool-id us-east-1_abc123xyz \
  --client-name MyGameApp \
  --no-generate-secret \
  --explicit-auth-flows "ALLOW_USER_PASSWORD_AUTH" "ALLOW_USER_SRP_AUTH" "ALLOW_REFRESH_TOKEN_AUTH"

# Sign up a user
aws cognito-idp sign-up \
  --client-id xyz789 \
  --username prashant@example.com \
  --password Temp@1234Password

# Confirm user email
aws cognito-idp admin-confirm-sign-up \
  --user-pool-id us-east-1_abc123xyz \
  --username prashant@example.com

# User Pool Integration with API Gateway:
# Authorization Type: Cognito User Pool
# User Pool: <select your pool>
# API Gateway validates JWT automatically ✓
```

### Service 2: Cognito Identity Pools (Authorization)

A **credential provider** that gives users temporary AWS credentials to access AWS services directly.

#### What Problems Does It Solve?

- Mobile app needs to upload photo to S3 directly (not through Lambda)
- Web app needs to read DynamoDB directly
- Want federated access (Google users get credentials)
- Need different permissions for authenticated vs anonymous users

#### How It Works

```
Mobile App (user logs in via Google)
    │
    ├──► Cognito Identity Pool
    │    (Maps Google token → AWS credentials)
    │
    ├──► Returns temporary credentials
    │    (Access Key, Secret Key, Session Token)
    │
    └──► App uses credentials to call S3, DynamoDB directly
```

#### Authenticated vs Unauthenticated Roles

```bash
# Unauthenticated Role (any visitor)
# Permissions: READ from public S3 bucket, READ from public DynamoDB

# Authenticated Role (logged-in user)
# Permissions: WRITE to their user folder in S3,
#             UPDATE their own DynamoDB items
```

### Cognito User Pools vs Identity Pools: The Key Distinction

| Aspect | User Pools | Identity Pools |
|--------|-----------|----------------|
| **Purpose** | Sign-in / Sign-up | AWS credentials |
| **Returns** | JWT token | Temporary AWS credentials |
| **Best for** | User authentication | AWS service access |
| **Example** | "Login to game app" | "Upload photo to S3 directly" |
| **Integration** | API Gateway, ALB | Mobile SDK, Web SDK |

**EXAM TIP:** "Hundreds of users need to sign in" → **User Pools**. "Mobile users need to upload to S3 directly" → **Identity Pools**.

### Complete Cognito Architecture

```bash
# Step 1: User creates account in User Pool
aws cognito-idp sign-up \
  --client-id xyz789 \
  --username user@example.com \
  --password Secure@123

# Step 2: User logs in, receives JWT
# (User Pool returns ID token, Access token, Refresh token)

# Step 3: Use ID token to call Identity Pool
aws cognito-identity get-id \
  --identity-pool-id us-east-1:abc123xyz \
  --logins cognito-idp.us-east-1.amazonaws.com/us-east-1_abc123xyz:token

# Step 4: Get temporary AWS credentials
aws cognito-identity get-credentials-for-identity \
  --identity-id us-east-1:xyz123abc \
  --logins cognito-idp.us-east-1.amazonaws.com/us-east-1_abc123xyz:token

# Step 5: Use credentials to call S3 directly
# (App now has Access Key, Secret Key, Session Token)
# aws s3 cp photo.jpg s3://my-bucket/users/user-001/photo.jpg
```

### Points to Remember: Cognito
✓ User Pools = sign-in/sign-up (authentication)  
✓ Identity Pools = AWS credentials (authorization)  
✓ User Pools integrate with API Gateway (validate JWT)  
✓ Identity Pools allow direct AWS service access  
✓ Supports social login (Facebook, Google, SAML, OIDC)  
✓ Federated identity = merge multiple identity providers  
✓ Exam distinction: "sign-in" → User Pools, "S3 access" → Identity Pools  

---

## 13. Architecture Diagrams

### Diagram 1: DynamoDB with DAX and Streams for Real-Time Processing

<p align="center"><img src="diagrams/icons/named/user.jpg" width="52" title="Mobile App"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/dynamodb.jpg" width="52" title="DAX Cache"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/dynamodb.jpg" width="52" title="DynamoDB Table"/></p>
<p align="center"><sub>Read path: App → DAX → DynamoDB</sub></p>
<p align="center"><img src="diagrams/icons/named/dynamodb.jpg" width="52" title="DynamoDB Streams"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/lambda.jpg" width="52" title="Lambda Consumer"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/kinesis.jpg" width="52" title="Kinesis Firehose"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/s3.jpg" width="52" title="S3 Analytics"/></p>
<p align="center"><sub>Stream path: DDB Streams → Lambda → Kinesis Firehose → S3</sub></p>

### Diagram 2: Complete Serverless API with Cognito & DynamoDB

<p align="center"><img src="diagrams/icons/named/user.jpg" width="52" title="Mobile / Web Client"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/cognito.jpg" width="52" title="Cognito User Pool (Sign-in)"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/api-gateway.jpg" width="52" title="API Gateway (REST)"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/lambda.jpg" width="52" title="Lambda Function"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/dynamodb.jpg" width="52" title="DynamoDB Games Table"/></p>
<p align="center"><sub>Client → Cognito (auth) → API Gateway → Lambda → DynamoDB / S3</sub></p>
<p align="center"><img src="diagrams/icons/named/lambda.jpg" width="52" title="Lambda also writes"/> &nbsp;→&nbsp; <img src="diagrams/icons/named/s3-purple.png" width="52" title="S3 Bucket (Leaderboards)"/></p>

### Diagram 3: API Gateway AWS Service Integration (No Lambda)

<p align="center"><img src="diagrams/icons/named/user-bw.png" width="52" title="Client"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/api-gateway.jpg" width="52" title="API Gateway (direct, no Lambda)"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/kinesis.jpg" width="52" title="Kinesis Data Streams"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/kinesis-bw.jpg" width="52" title="Kinesis Firehose"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/s3.jpg" width="52" title="S3 Bucket"/></p>
<p align="center"><sub>Client → API Gateway (direct AWS integration, no Lambda!) → Kinesis → Firehose → S3</sub></p>

### Diagram 4: Cognito User Pools vs Identity Pools

<p align="center"><img src="diagrams/icons/named/user.jpg" width="52" title="App User"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/cognito.jpg" width="52" title="Cognito User Pool (Auth/JWT)"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/cognito-bw.png" width="52" title="Identity Pool (STS/AWS Creds)"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/s3-purple.png" width="52" title="S3"/>&nbsp;/&nbsp;<img src="diagrams/icons/named/dynamodb.jpg" width="52" title="DynamoDB"/>&nbsp;/&nbsp;<img src="diagrams/icons/named/lambda.jpg" width="52" title="Lambda"/></p>
<p align="center"><sub>User → Cognito User Pool (sign-in) → Identity Pool (STS) → AWS services</sub></p>

### Diagram 5: DynamoDB Global Tables — Active-Active Replication

<p align="center">
  <table align="center" cellspacing="10" cellpadding="6" border="0">
    <tr>
      <td align="center"><img src="diagrams/icons/named/user-bw.png" width="44" title="App in US"/><br/><sub><b>US App</b></sub></td>
      <td></td>
      <td align="center"><img src="diagrams/icons/named/user-bw.png" width="44" title="App in EU"/><br/><sub><b>EU App</b></sub></td>
      <td></td>
      <td align="center"><img src="diagrams/icons/named/user-bw.png" width="44" title="App in Asia"/><br/><sub><b>Asia App</b></sub></td>
    </tr>
    <tr><td align="center">↕ read/write</td><td></td><td align="center">↕ read/write</td><td></td><td align="center">↕ read/write</td></tr>
    <tr>
      <td align="center"><img src="diagrams/icons/named/dynamodb.jpg" width="44" title="DynamoDB us-east-1"/><br/><sub>us-east-1</sub></td>
      <td align="center">&nbsp;&nbsp;⇄&nbsp;&nbsp;</td>
      <td align="center"><img src="diagrams/icons/named/dynamodb.jpg" width="44" title="DynamoDB eu-west-1"/><br/><sub>eu-west-1</sub></td>
      <td align="center">&nbsp;&nbsp;⇄&nbsp;&nbsp;</td>
      <td align="center"><img src="diagrams/icons/named/dynamodb.jpg" width="44" title="DynamoDB ap-southeast-1"/><br/><sub>ap-southeast-1</sub></td>
    </tr>
  </table>
</p>
<p align="center"><sub>Active-Active Global Tables: writes in any region replicate globally (sub-second latency)</sub></p>

### Diagram 6: Step Functions Order Processing Workflow

<p align="center"><img src="diagrams/icons/named/api-gateway.jpg" width="52" title="API Gateway"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/lambda-bw.png" width="52" title="Validate Order"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/lambda-bw.png" width="52" title="Check Inventory"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/lambda-bw.png" width="52" title="Process Payment"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/lambda-bw.png" width="52" title="Send Confirmation"/></p>
<p align="center"><sub>API Gateway → Step Functions workflow (each step is a Lambda or service task) — failure path: Process Payment → Refund & Notify</sub></p>

---

## 14. AWS CLI Quick Reference

### DynamoDB Commands

```bash
# ========== CREATE & MANAGE TABLES ==========

# Create table (Provisioned mode)
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions \
    AttributeName=UserId,AttributeType=S \
    AttributeName=GameId,AttributeType=S \
  --key-schema \
    AttributeName=UserId,KeyType=HASH \
    AttributeName=GameId,KeyType=RANGE \
  --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10 \
  --region us-east-1

# Create table (On-Demand mode)
aws dynamodb create-table \
  --table-name Events \
  --attribute-definitions AttributeName=EventId,AttributeType=S \
  --key-schema AttributeName=EventId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# List tables
aws dynamodb list-tables

# Describe table (schema, capacity, status)
aws dynamodb describe-table --table-name Users

# ========== READ/WRITE DATA ==========

# Put item (insert or overwrite)
aws dynamodb put-item \
  --table-name Users \
  --item '{
    "UserId": {"S": "user123"},
    "GameId": {"S": "game_A"},
    "Score": {"N": "9500"},
    "Badges": {"SS": ["speedrun", "perfect"]},
    "Timestamp": {"N": "1646433000"}
  }'

# Get item (retrieve by partition key + sort key)
aws dynamodb get-item \
  --table-name Users \
  --key '{
    "UserId": {"S": "user123"},
    "GameId": {"S": "game_A"}
  }'

# Query items (by partition key, with sort key filter)
aws dynamodb query \
  --table-name Users \
  --key-condition-expression "UserId = :uid" \
  --expression-attribute-values '{":uid": {"S": "user123"}}' \
  --projection-expression "GameId, Score"

# Query with sort key range
aws dynamodb query \
  --table-name Users \
  --key-condition-expression "UserId = :uid AND GameId BETWEEN :g1 AND :g2" \
  --expression-attribute-values '{
    ":uid": {"S": "user123"},
    ":g1": {"S": "game_A"},
    ":g2": {"S": "game_M"}
  }'

# Scan entire table (slow — last resort)
aws dynamodb scan --table-name Users --limit 100

# Update item
aws dynamodb update-item \
  --table-name Users \
  --key '{"UserId": {"S": "user123"}, "GameId": {"S": "game_A"}}' \
  --update-expression "SET Score = :score" \
  --expression-attribute-values '{":score": {"N": "10000"}}'

# Delete item
aws dynamodb delete-item \
  --table-name Users \
  --key '{"UserId": {"S": "user123"}, "GameId": {"S": "game_A"}}'

# ========== CAPACITY & SCALING ==========

# Update provisioned throughput
aws dynamodb update-table \
  --table-name Users \
  --provisioned-throughput ReadCapacityUnits=20,WriteCapacityUnits=20

# Set up auto-scaling (write capacity)
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Users" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --min-capacity 5 \
  --max-capacity 100

# Create auto-scaling policy
aws application-autoscaling put-scaling-policy \
  --policy-name UserWriteScaling \
  --policy-type TargetTrackingScaling \
  --service-namespace dynamodb \
  --resource-id "table/Users" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --target-tracking-scaling-policy-configuration \
    'TargetValue=70.0,PredefinedMetricSpecification={PredefinedMetricType=DynamoDBWriteCapacityUtilization},ScaleOutCooldown=60,ScaleInCooldown=300'

# ========== STREAMS & REPLICATION ==========

# Enable DynamoDB Streams
aws dynamodb update-table \
  --table-name Users \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

# List streams
aws dynamodb list-streams --table-name Users

# Create Global Table
aws dynamodb create-global-table \
  --global-table-name Users \
  --replication-group RegionName=us-east-1 RegionName=eu-west-1 RegionName=ap-southeast-1

# ========== TTL & BACKUPS ==========

# Enable TTL
aws dynamodb update-time-to-live \
  --table-name Sessions \
  --time-to-live-specification "Enabled=true, AttributeName=ExpiresAt"

# Enable PITR (Point-in-Time Recovery)
aws dynamodb update-continuous-backups \
  --table-name Users \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true

# Create on-demand backup
aws dynamodb create-backup \
  --table-name Users \
  --backup-name users-backup-20260306

# List backups
aws dynamodb list-backups --table-name Users

# Restore from backup
aws dynamodb restore-table-from-backup \
  --target-table-name Users-Restored \
  --backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users/backup/01234567890123

# Restore to point-in-time
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Users \
  --target-table-name Users-Restored-PiT \
  --restore-date-time 2026-03-05T14:30:00

# Export to S3
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users \
  --s3-bucket my-export-bucket \
  --s3-prefix users-export/ \
  --export-format DYNAMODB_JSON
```

### API Gateway Commands

```bash
# ========== CREATE API ==========

# Create REST API
aws apigateway create-rest-api \
  --name MyGameAPI \
  --description "Game scores and leaderboard API" \
  --endpoint-configuration types=REGIONAL

# Create HTTP API (newer, faster)
aws apigatewayv2 create-api \
  --name MyGameAPI \
  --protocol-type HTTP

# ========== STAGES & DEPLOYMENTS ==========

# Create deployment
aws apigateway create-deployment \
  --rest-api-id abc123xyz \
  --stage-name prod \
  --stage-description "Production"

# Update stage
aws apigateway update-stage \
  --rest-api-id abc123xyz \
  --stage-name prod \
  --patch-operations op=replace,path=/variables/environment,value=prod

# Enable caching
aws apigateway update-stage \
  --rest-api-id abc123xyz \
  --stage-name prod \
  --patch-operations op=replace,path=/cacheClusterEnabled,value=true

# ========== API KEYS & THROTTLING ==========

# Create API key
aws apigateway create-api-key \
  --name Client1Key \
  --description "API key for client 1" \
  --enabled

# Create usage plan (throttling + quota)
aws apigateway create-usage-plan \
  --name BasicPlan \
  --description "Basic tier with throttling" \
  --throttle burstLimit=5000,rateLimit=1000 \
  --quota limit=1000000,period=MONTH

# Associate API key with usage plan
aws apigateway create-usage-plan-key \
  --usage-plan-id abc123 \
  --key-id xyz789 \
  --key-type API_KEY

# ========== CUSTOM DOMAIN ==========

# Create custom domain
aws apigateway create-domain-name \
  --domain-name api.example.com \
  --certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/abc123 \
  --endpoint-configuration types=EDGE_OPTIMIZED

# Create Route 53 alias (manual after creating domain)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "d1234.cloudfront.net",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

### Step Functions Commands

```bash
# ========== STATE MACHINES ==========

# Create state machine
aws stepfunctions create-state-machine \
  --name order-workflow \
  --definition file://state-machine.json \
  --role-arn arn:aws:iam::123456789012:role/StepFunctionsRole

# List state machines
aws stepfunctions list-state-machines

# Describe state machine
aws stepfunctions describe-state-machine \
  --state-machine-arn arn:aws:states:us-east-1:123456789012:stateMachine:order-workflow

# ========== EXECUTIONS ==========

# Start execution
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:123456789012:stateMachine:order-workflow \
  --name order-001-exec \
  --input '{"orderId": "order-001", "amount": 99.99, "customerId": "cust123"}'

# List executions
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:us-east-1:123456789012:stateMachine:order-workflow \
  --status-filter RUNNING

# Describe execution (see status & output)
aws stepfunctions describe-execution \
  --execution-arn arn:aws:states:us-east-1:123456789012:execution:order-workflow:order-001-exec

# Stop execution
aws stepfunctions stop-execution \
  --execution-arn arn:aws:states:us-east-1:123456789012:execution:order-workflow:order-001-exec
```

### Cognito Commands

```bash
# ========== USER POOLS ==========

# Create user pool
aws cognito-idp create-user-pool \
  --pool-name GameAppUsers \
  --policies '{
    "PasswordPolicy": {
      "MinimumLength": 8,
      "RequireUppercase": true,
      "RequireLowercase": true,
      "RequireNumbers": true,
      "RequireSymbols": false
    }
  }' \
  --mfa-configuration OPTIONAL

# Create user pool client
aws cognito-idp create-user-pool-client \
  --user-pool-id us-east-1_abc123xyz \
  --client-name GameApp \
  --no-generate-secret \
  --explicit-auth-flows "ALLOW_USER_PASSWORD_AUTH" "ALLOW_REFRESH_TOKEN_AUTH"

# ========== USER MANAGEMENT ==========

# Sign up user
aws cognito-idp sign-up \
  --client-id xyz789 \
  --username user@example.com \
  --password Secure@Password123

# Confirm user (admin)
aws cognito-idp admin-confirm-sign-up \
  --user-pool-id us-east-1_abc123xyz \
  --username user@example.com

# Authenticate user
aws cognito-idp admin-initiate-auth \
  --user-pool-id us-east-1_abc123xyz \
  --client-id xyz789 \
  --auth-flow ADMIN_USER_PASSWORD_AUTH \
  --auth-parameters USERNAME=user@example.com,PASSWORD=Secure@Password123

# List users
aws cognito-idp list-users --user-pool-id us-east-1_abc123xyz

# ========== IDENTITY POOLS ==========

# Create identity pool
aws cognito-identity create-identity-pool \
  --identity-pool-name GameAppIdentity \
  --allow-unauthenticated-identities true \
  --cognito-identity-providers ProviderName=cognito-idp.us-east-1.amazonaws.com/us-east-1_abc123xyz,ClientId=xyz789

# Get identity ID
aws cognito-identity get-id \
  --identity-pool-id us-east-1:abc123xyz \
  --logins cognito-idp.us-east-1.amazonaws.com/us-east-1_abc123xyz:token

# Get credentials
aws cognito-identity get-credentials-for-identity \
  --identity-id us-east-1:xyz123abc \
  --logins cognito-idp.us-east-1.amazonaws.com/us-east-1_abc123xyz:token
```

---

## 15. Interview Tips & Exam Hacks

### DynamoDB Interview Tips

**Q: "Your DynamoDB table is slow. What do you check?"**

**Answer:**
1. **Is it a read problem or write problem?** (CloudWatch metrics)
2. **Is there a hot partition?** (One user's data everyone queries)
   - Solution: Add a random suffix to partition key to distribute load
3. **Are you querying by sort key alone?** (That's a table scan — slow)
   - Solution: Always query by partition key
4. **RCU/WCU throttled?** (Requests return 400 ThrottlingException)
   - Solution: Increase capacity OR enable auto-scaling
5. **Use DAX if it's a read problem** (cache hot items)

**Q: "Should we use Provisioned or On-Demand mode?"**

**Answer:**
- **Provisioned:** Predictable traffic, cost-conscious (cheaper per request)
- **On-Demand:** Spiky/unpredictable traffic, new product (scales automatically)
- **Exam trick:** "New startup launching in beta" → On-Demand (unknown traffic)

**Q: "How do we replicate DynamoDB to another region?"**

**Answer:**
- **Global Tables:** Active-Active (write anywhere)
- **DynamoDB Streams → Lambda:** Active-Passive (read-only replicas)
- **Backup & Restore:** Manual cross-region copy

**Q: "Our app needs user sessions. How do we expire them?"**

**Answer:**
- **TTL:** Automatic deletion when timestamp expires
- **Attribute:** Number type, Unix epoch
- **Use case:** Sessions, temp data, compliance

### API Gateway Interview Tips

**Q: "Should we use REST API or HTTP API?"**

**Answer:**
- **REST API:** Full features (stages, models, validators), features-rich
- **HTTP API:** Newer, 60% cheaper, faster, simpler
- **Default:** REST unless you need cost optimization

**Q: "How do we authenticate API requests?"**

**Answer:**
1. **Internal (EC2 → API):** IAM roles + Signature V4
2. **External users:** Cognito User Pool (zero auth code)
3. **Custom auth:** Lambda Authorizer (OAuth, custom tokens)
4. **Simple throttling:** API keys (not real auth)

**Q: "Can we call AWS services directly from API Gateway?"**

**Answer:**
- **YES:** AWS Service Integration (Kinesis, SQS, DynamoDB, Step Functions)
- **Benefit:** No Lambda, zero latency overhead
- **Example:** Client → API GW → SQS → Lambda consumer

### Cognito Interview Tips

**Q: "What's the difference between User Pools and Identity Pools?"**

**Answer (critical distinction):**
- **User Pools:** "How do I sign in?" → Authentication
- **Identity Pools:** "How do I access S3?" → Authorization
- **Common exam trap:** "Mobile users need to upload to S3 directly" → Identity Pools

**Q: "Can we integrate Cognito with third-party IdP?"**

**Answer:**
- **YES:** SAML 2.0, OpenID Connect, Facebook, Google
- **User Pool redirects** to external IdP, user logs in there, returns token
- **Use case:** Enterprise SSO, social login

### Step Functions Interview Tips

**Q: "How long can a Step Functions workflow run?"**

**Answer:**
- **Standard workflows:** Up to 1 year (use for long-running processes)
- **Express workflows:** Up to 5 minutes (for real-time processing)
- **Exam trick:** "Process takes 2 hours" → Standard workflow

**Q: "How do we handle errors in workflows?"**

**Answer:**
- **Retry:** Exponential backoff (2^n seconds), max attempts
- **Catch:** Handle specific errors, continue or fail
- **Fallback tasks:** On error, run alternative task (refund, notify)

### General Serverless Architecture Tips

**Q: "Design a serverless API for a gaming leaderboard."**

**Answer:**
```
Clients
  ├─ Cognito User Pool (sign-in)
  ├─ API Gateway (REST API)
  ├─ Lambda (business logic)
  ├─ DynamoDB (game scores)
  │  ├─ DAX (cache leaderboard top 10 — hot data)
  │  └─ Streams (for real-time updates)
  ├─ S3 (leaderboard images)
  └─ CloudFront (CDN for static assets)
```

**Q: "How do we make data available across regions?"**

**Answer:**
- **DynamoDB Global Tables:** For database
- **CloudFront:** For static assets (S3 + distribution)
- **Multi-region Lambda:** For compute (SNS/SQS for async)

**Q: "How do we monitor a serverless system?"**

**Answer:**
- **CloudWatch Logs:** Lambda, API Gateway logs
- **X-Ray:** Trace requests end-to-end
- **CloudWatch Metrics:** RCU/WCU, Lambda duration, API latency
- **Alarms:** DynamoDB throttling, Lambda errors, cold starts

### Exam Hacks

1. **Read the ENTIRE question** before answering (exam tricks hide details)
2. **Look for keywords:**
   - "Sign-in/Sign-up" → **Cognito User Pool**
   - "Upload to S3 directly" → **Cognito Identity Pool**
   - "DynamoDB is slow" → **DAX** (if reads) or **increase capacity**
   - "Multi-region" → **Global Tables**
   - "Spiky traffic" → **On-Demand mode or Auto-Scaling**
   - "Orchestrate workflow" → **Step Functions**
   - "Expire data automatically" → **TTL**

3. **Elimination strategy:**
   - "Can't use IAM in this scenario" → eliminate IAM auth
   - "Needs custom validation" → Lambda Authorizer
   - "Needs OAuth" → Custom Authorizer or Cognito Federated

4. **Time management:** Skip ambiguous questions, mark for review

---

## 16. Official AWS Documentation

### DynamoDB
- **Main Guide:** https://docs.aws.amazon.com/dynamodb/latest/developerguide/
- **Best Practices:** https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html
- **API Reference:** https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/
- **Global Tables:** https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html
- **Streams:** https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html

### DynamoDB Accelerator (DAX)
- **Guide:** https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.html
- **Getting Started:** https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAXGettingStarted.html

### API Gateway
- **Developer Guide:** https://docs.aws.amazon.com/apigateway/latest/developerguide/
- **API Reference:** https://docs.aws.amazon.com/apigateway/latest/api/
- **REST API:** https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html
- **WebSocket:** https://docs.aws.amazon.com/apigateway/latest/developerguide/websocket-api.html
- **Integrations:** https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-integration-settings.html

### Step Functions
- **Developer Guide:** https://docs.aws.amazon.com/step-functions/latest/dg/
- **State Machine Language:** https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html
- **Best Practices:** https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language-state-machine-structure.html

### Amazon Cognito
- **User Pools Guide:** https://docs.aws.amazon.com/cognito/latest/developerguide/user-pools.html
- **Identity Pools Guide:** https://docs.aws.amazon.com/cognito/latest/developerguide/identity-pools.html
- **API Reference:** https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/
- **Federated Identities:** https://docs.aws.amazon.com/cognito/latest/developerguide/external-identity-providers.html

### Serverless Architecture
- **Well-Architected Framework:** https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/
- **Whitepapers:** https://aws.amazon.com/whitepapers/serverless-architecture/

---

## Final Checklist Before SAA-C03 Exam

- [ ] **DynamoDB:** Understand partition key distribution, RCU/WCU calculations, hot partitions
- [ ] **DAX vs ElastiCache:** Know when to use each (DynamoDB reads vs app-level caching)
- [ ] **Capacity Modes:** Provisioned (predictable) vs On-Demand (spiky), auto-scaling
- [ ] **Replication:** Global Tables (Active-Active) vs Streams vs Backups
- [ ] **TTL:** Automatic expiration, Unix timestamps, eventually consistent deletion
- [ ] **API Gateway:** Integration types (Lambda, HTTP, AWS Service), endpoint types (Edge, Regional, Private)
- [ ] **Authentication:** IAM (internal), Cognito (external), Custom Authorizer (custom)
- [ ] **Cognito:** User Pools (sign-in) vs Identity Pools (AWS creds) — CRITICAL distinction
- [ ] **Step Functions:** Workflows, retries, error handling, max 1 year duration
- [ ] **Serverless Design:** Lambda → API Gateway → DynamoDB → DAX/CloudFront

---

**Study Tips:**
1. Build projects, don't just read — hands-on experience matters
2. Practice CLI commands until they're muscle memory
3. Review AWS architecture diagrams regularly
4. Simulate exam questions under time pressure
5. Focus on **WHY** not **WHAT** — expert-led approach works!

**Good luck on SAA-C03! You've got this.**

---

*Last Reviewed: March 6, 2026*  
*Author: Claude Sonnet (AWS SAA-C03 Study Guide)*
