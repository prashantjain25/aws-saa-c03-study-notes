# RDS, Aurora & ElastiCache
> 📚 Official Docs: https://docs.aws.amazon.com/rds/ | https://docs.aws.amazon.com/AmazonElastiCache/
> 🎯 SAA-C03 Exam Weight: Very High — databases are central to all architectures

---

## 🗄️ Amazon RDS — Managed Relational Database

### Why RDS Instead of Running Your Own DB on EC2?

You *could* install MySQL on an EC2 instance. But then YOU are responsible for:
- OS patching
- DB software upgrades
- Taking backups
- Setting up replication
- Monitoring disk space
- Failover when the instance crashes

**RDS handles all of that for you.** You just connect and query.

```
Self-managed DB on EC2:          Amazon RDS:
┌─────────────────────┐          ┌─────────────────────┐
│ EC2 Instance         │          │ RDS Managed Service  │
│ ├── OS patches (you) │          │ ├── OS patches (AWS)  │
│ ├── DB upgrades (you)│          │ ├── DB upgrades (AWS) │
│ ├── Backups (you)    │          │ ├── Auto backups ✅   │
│ ├── Replication (you)│          │ ├── Multi-AZ replica ✅│
│ └── Monitoring (you) │          │ └── Auto monitoring ✅ │
│ Can SSH in           │          │ CANNOT SSH in ❌       │
└─────────────────────┘          └─────────────────────┘
```

> ⚠️ **You CANNOT SSH into an RDS instance** — it's a managed service. This is intentional and a common exam gotcha.

**Supported Engines:** PostgreSQL, MySQL, MariaDB, Oracle, Microsoft SQL Server, IBM DB2, Aurora (AWS proprietary)

### Creating an RDS Instance

When you create an RDS database, you define the instance class, storage, engine, and security configuration. Here's the CLI approach:

```bash
# Create RDS MySQL instance with Multi-AZ and backups enabled
aws rds create-db-instance \
  --db-instance-identifier my-mysql-db \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password MySecurePass123 \
  --allocated-storage 20 \
  --vpc-security-group-ids sg-0abc123def456 \
  --db-subnet-group-name my-db-subnet-group \
  --backup-retention-period 7 \
  --multi-az \
  --no-publicly-accessible

# Describe your RDS instance to see current state
aws rds describe-db-instances \
  --db-instance-identifier my-mysql-db
```

### RDS Storage Auto Scaling

RDS can automatically expand your storage when it runs low — you set a **Maximum Storage Threshold** and RDS scales up automatically when:
- Free storage < 10% of allocated
- Low storage condition has lasted > 5 minutes  
- 6+ hours since last storage modification

```bash
# Enable storage auto-scaling on existing instance
aws rds modify-db-instance \
  --db-instance-identifier my-mysql-db \
  --storage-type gp2 \
  --allocated-storage 100 \
  --apply-immediately
```

---

## 📖 RDS Read Replicas vs Multi-AZ — The Critical Distinction

This is **one of the most tested concepts** in the exam. Let's be very clear:

```
READ REPLICA — Purpose: PERFORMANCE (more reads)
────────────────────────────────────────────────────────────
Primary DB (write + read)
     │
     │ ASYNC replication (slight lag possible)
     │
     ├──▶ Read Replica 1 (read-only)  ← Analytics queries go here
     ├──▶ Read Replica 2 (read-only)  ← Reporting goes here
     └──▶ Read Replica 3 (read-only, different region) ← Global reads
     
- Up to 15 read replicas
- Eventual consistency (ASYNC = small replication lag)
- Can be in same AZ, different AZ, or different region
- Cross-region replication costs $ for network transfer
- Replica can be PROMOTED to standalone DB (becomes read-write)
- Use case: Offload analytics/reporting from primary

MULTI-AZ — Purpose: DISASTER RECOVERY (high availability)
───────────────────────────────────────────────────────────────
Primary DB (read + write) ← Applications always connect here
     │
     │ SYNC replication (every write confirmed on both)
     │
     └──▶ Standby DB (SAME DATA, different AZ) ← NO traffic, just waiting
     
- Only 1 standby (not readable, not usable for queries)
- Synchronous = zero data loss on failover  
- Automatic failover via DNS (same endpoint, ~1-2 min to switch)
- Use case: Production databases needing HA
```

> 💡 **Key insight**: Multi-AZ standby **CANNOT serve reads** — it's purely for failover. If an interviewer asks "can you read from Multi-AZ standby?" — the answer is **NO** (for RDS — Aurora is different!).

### Creating Read Replicas

```bash
# Create a read replica in the same region
aws rds create-db-instance-read-replica \
  --db-instance-identifier my-mysql-db-replica \
  --source-db-instance-identifier my-mysql-db \
  --db-instance-class db.t3.micro

# Create a cross-region read replica (for disaster recovery)
aws rds create-db-instance-read-replica \
  --db-instance-identifier my-mysql-db-replica-eu \
  --source-db-instance-identifier my-mysql-db \
  --source-region us-east-1 \
  --region eu-west-1 \
  --db-instance-class db.t3.micro

# Promote a read replica to a standalone database
# (useful for cross-region migration or making it read-write)
aws rds promote-read-replica \
  --db-instance-identifier my-mysql-db-replica
```

### Enabling Multi-AZ

```bash
# Modify instance to enable Multi-AZ
aws rds modify-db-instance \
  --db-instance-identifier my-mysql-db \
  --multi-az \
  --apply-immediately

# Note: During initial enablement, there's a brief outage
# as the standby is created and synchronized
```

### RDS Backups

**Automated Backups (Managed by AWS):**
- Daily full backup during maintenance window
- Transaction logs backed up every 5 minutes
- **Point-in-Time Recovery (PITR)**: restore to any second within retention window
- Retention period: 1–35 days (set to 0 to disable)

**Manual Snapshots:**
- You trigger these manually
- **Retained indefinitely** (even after you delete the DB)
- Use for long-term archival or before major schema changes

```bash
# Create a manual snapshot (good before risky operations)
aws rds create-db-snapshot \
  --db-instance-identifier my-mysql-db \
  --db-snapshot-identifier my-snapshot-20260306

# Restore RDS instance from a snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier my-restored-db \
  --db-snapshot-identifier my-snapshot-20260306

# Configure automated backup settings
aws rds modify-db-instance \
  --db-instance-identifier my-mysql-db \
  --preferred-backup-window "03:00-04:00" \
  --backup-retention-period 7 \
  --apply-immediately
```

### RDS Subnet Groups

RDS instances must run in a VPC, and you need to specify which subnets (from which AZs) the instance can use.

```bash
# Create a DB subnet group (must include 2+ AZs)
aws rds create-db-subnet-group \
  --db-subnet-group-name my-db-subnet-group \
  --db-subnet-group-description "DB Subnet Group for RDS" \
  --subnet-ids subnet-0abc123def456 subnet-0def456abc123
```

### RDS Deletion

```bash
# Delete RDS instance with a final snapshot for recovery
aws rds delete-db-instance \
  --db-instance-identifier my-mysql-db \
  --final-db-snapshot-identifier my-final-snapshot

# Delete without final snapshot (be careful!)
aws rds delete-db-instance \
  --db-instance-identifier my-mysql-db \
  --skip-final-snapshot
```

---

## 🌟 Amazon Aurora — The Cloud-Native Database

Aurora is AWS's own relational database, designed specifically for the cloud. It's **not open source** but is **compatible with MySQL and PostgreSQL** drivers — meaning most apps can switch to Aurora with minimal changes.

### Aurora vs Standard RDS

| Feature | Aurora | RDS MySQL |
|---------|--------|-----------|
| Performance | 5x MySQL on RDS | Baseline |
| Storage | Auto-grows to 128 TB | Manual provisioning |
| Read Replicas | Up to 15, <10ms lag | Up to 5, higher lag |
| Failover | < 30 seconds | 1-2 minutes |
| Cross-region | Global Database | Manual setup |
| Cost | ~20% more | Lower |

### Aurora Storage Architecture — How It's Different

Standard RDS stores data on EBS volumes attached to the instance. Aurora uses a completely different, distributed storage layer:

```
Aurora Storage Layer (6 copies, 3 AZs):
┌──────────────────────────────────────────────────────────────┐
│  Aurora Instance (master)                                    │
│        │                                                     │
│        ▼  (writes to all storage nodes in parallel)          │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┐                     │
│  │ S1  │ S2  │ S3  │ S4  │ S5  │ S6  │  ← 6 storage nodes  │
│  │ AZ1 │ AZ1 │ AZ2 │ AZ2 │ AZ3 │ AZ3 │     across 3 AZs    │
│  └─────┴─────┴─────┴─────┴─────┴─────┘                     │
│  Write quorum: 4/6 needed                                   │
│  Read quorum: 3/6 needed                                    │
│  Can lose 2 nodes and still WRITE!                          │
│  Can lose 3 nodes and still READ!                           │
└──────────────────────────────────────────────────────────────┘
```

### Creating an Aurora Cluster

```bash
# Create Aurora MySQL cluster (primary only)
aws rds create-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --engine aurora-mysql \
  --engine-version "8.0.mysql_aurora.3.02.0" \
  --master-username admin \
  --master-user-password MySecurePass123 \
  --vpc-security-group-ids sg-0abc123def456 \
  --db-subnet-group-name my-db-subnet-group \
  --backup-retention-period 7

# Add a reader instance to the cluster
aws rds create-db-instance \
  --db-instance-identifier my-aurora-reader-1 \
  --db-cluster-identifier my-aurora-cluster \
  --db-instance-class db.r5.large \
  --engine aurora-mysql

# Add a second reader for higher throughput
aws rds create-db-instance \
  --db-instance-identifier my-aurora-reader-2 \
  --db-cluster-identifier my-aurora-cluster \
  --db-instance-class db.r5.large \
  --engine aurora-mysql
```

### Aurora Endpoints

```
Aurora Cluster:
┌─────────────────────────────────────────────────────────┐
│  Writer Endpoint: cluster-xxx.cluster-ro-xxx.rds.aws    │
│       │                                                 │
│  Master Instance (read + write) ←─ point your app here  │
│                                                         │
│  Reader Endpoint: cluster-ro-xxx.cluster-ro-xxx.rds.aws │
│       │                                                 │
│  Read Replica 1 ┐                                       │
│  Read Replica 2 ├── Load balanced automatically          │
│  Read Replica 3 ┘                                       │
└─────────────────────────────────────────────────────────┘
```

### Aurora Global Database

For globally distributed applications, Aurora Global replicates to up to 5 secondary regions:

```
Primary Region (us-east-1) ──▶ Secondary Region (eu-west-1) [read-only]
        (read/write)            Secondary Region (ap-east-1) [read-only]
        
Replication lag: < 1 second
Recovery Point Objective (RPO): < 1 second
Recovery Time Objective (RTO): < 1 minute (promote secondary to primary)
```

```bash
# Create Aurora Global Database
aws rds create-global-cluster \
  --global-cluster-identifier my-global-aurora \
  --source-db-cluster-identifier arn:aws:rds:us-east-1:123456789012:cluster:my-aurora-cluster

# Add a secondary region
aws rds create-db-cluster \
  --db-cluster-identifier my-aurora-secondary \
  --global-cluster-identifier my-global-aurora \
  --engine aurora-mysql \
  --region eu-west-1
```

### Aurora Serverless v2

Instead of provisioning a fixed instance size, Aurora Serverless automatically adjusts compute capacity in real time.

```
Traffic:   ──────▄▄▄████████████▄▄────────▄▄▄▄████▄▄─────
ACU (Auto  ──────▄▄▄████████████▄▄────────▄▄▄▄████▄▄─────
  Scaling Units): matches traffic instantly
  
Min: 0.5 ACU | Max: 128 ACU (configurable)
Billing: per ACU-second (pay for what you use)
```

```bash
# Create Aurora Serverless v2 cluster
aws rds create-db-cluster \
  --db-cluster-identifier my-aurora-serverless \
  --engine aurora-mysql \
  --engine-version "8.0.mysql_aurora.3.02.0" \
  --master-username admin \
  --master-user-password MySecurePass123 \
  --db-subnet-group-name my-db-subnet-group \
  --serverlessv2-scaling-configuration MinCapacity=0.5,MaxCapacity=2.0
```

**Use when**: unpredictable, intermittent workloads — dev/test, multi-tenant SaaS, new apps

### Aurora Auto Scaling (for Reader Replicas)

```bash
# Register Aurora cluster for auto-scaling
aws application-autoscaling register-scalable-target \
  --service-namespace rds \
  --resource-id cluster:my-aurora-cluster \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --min-capacity 1 \
  --max-capacity 5

# Create target tracking scaling policy (keep CPU at 70%)
aws application-autoscaling put-scaling-policy \
  --service-namespace rds \
  --resource-id cluster:my-aurora-cluster \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --policy-name aurora-cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "RDSReaderAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

---

## ⚡ Amazon ElastiCache — In-Memory Caching

### Why Use a Cache?

Every time your app queries a DB, it takes 10–100ms (network + disk I/O). For a page that makes 20 DB queries, that's up to 2 seconds just in DB time. With caching, frequently-read data lives in memory — queries take microseconds.

```
Without Cache:                    With Cache (Cache-Aside pattern):
App ──▶ DB query (100ms) ──▶ Result   App ──▶ Cache check (1ms)
                                            ├── HIT: return result (1ms total) ✅
                                            └── MISS: query DB (100ms), store in cache, return
```

**What is the Cache-Aside (Lazy Loading) Pattern?**  
The application is responsible for loading data into the cache. It first checks the cache, and only fetches from the DB on a cache miss, then stores the result. Simple but data can be stale. See more at: https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/caching-patterns.html

ElastiCache supports two engines: **Redis** and **Memcached**.

### Redis vs Memcached — Choose Wisely

```
Redis:                              Memcached:
┌─────────────────────────────┐    ┌─────────────────────────────┐
│ ✅ Persistence (AOF/RDB)     │    │ ❌ No persistence            │
│ ✅ Multi-AZ with failover    │    │ ❌ No replication            │
│ ✅ Read replicas             │    │ ✅ Multi-threaded             │
│ ✅ Pub/Sub messaging         │    │ ✅ Simple, pure caching       │
│ ✅ Sorted Sets (leaderboards)│    │ ✅ Slightly simpler           │
│ ✅ Geospatial indexes        │    │                             │
│ ✅ Backup and restore        │    │                             │
│ Use: Sessions, leaderboards, │    │ Use: Simple shared cache,    │
│  real-time analytics, queues │    │  large data, multi-thread   │
└─────────────────────────────┘    └─────────────────────────────┘
```

> 💡 **Rule of thumb**: If you need the cache to survive failures → Redis. If you just need a fast throwaway cache → Memcached.

### Creating ElastiCache Clusters

```bash
# Create Redis cluster (cluster mode disabled — single shard)
aws elasticache create-cache-cluster \
  --cache-cluster-id my-redis-cluster \
  --cache-node-type cache.t3.micro \
  --engine redis \
  --num-cache-nodes 1 \
  --cache-subnet-group-name my-cache-subnet-group \
  --security-group-ids sg-0abc123def456

# Create Memcached cluster (multi-node)
aws elasticache create-cache-cluster \
  --cache-cluster-id my-memcached \
  --cache-node-type cache.t3.micro \
  --engine memcached \
  --num-cache-nodes 3  # Can have multiple nodes
```

### ElastiCache Subnet Groups

```bash
# Create cache subnet group (required for VPC)
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name my-cache-subnet-group \
  --cache-subnet-group-description "Cache Subnet Group" \
  --subnet-ids subnet-0abc123def456 subnet-0def456abc123
```

### Redis Replication Groups (Multi-AZ)

For production Redis with automatic failover:

```bash
# Create Redis replication group with Multi-AZ
aws elasticache create-replication-group \
  --replication-group-id my-redis-rg \
  --replication-group-description "Redis HA with failover" \
  --cache-node-type cache.r6g.large \
  --engine redis \
  --num-cache-clusters 2 \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --cache-subnet-group-name my-cache-subnet-group

# Check replication group status
aws elasticache describe-replication-groups \
  --replication-group-id my-redis-rg
```

### Caching Strategies

**Cache-Aside (Lazy Loading):**
```
Read: App checks cache → MISS → read from DB → write to cache → return
Write: App writes directly to DB (cache may become stale)
Pros: Only requested data is cached | Cons: Initial cache miss penalty, stale data possible
```

```bash
# Application code pattern (pseudo-code):
# GET /user/123
#   user = cache.get("user:123")
#   if not user:
#     user = db.query("SELECT * FROM users WHERE id=123")
#     cache.set("user:123", user, ttl=3600)  # Cache for 1 hour
#   return user
```

**Write-Through:**
```
Write: App writes to DB → also writes to cache (same time)
Read: Cache always has fresh data
Pros: No stale data | Cons: Write penalty (every write = 2 writes), wasted cache if data never re-read
```

**Session Store:**
```
User logs in → Store session data in Redis with TTL
Any app server can validate session from Redis (enables stateless app tier!)
```

> 🔗 Caching strategies deep-dive: https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/

### Describing ElastiCache Resources

```bash
# List all cache clusters
aws elasticache describe-cache-clusters \
  --show-cache-node-info

# Get details of specific cluster
aws elasticache describe-cache-clusters \
  --cache-cluster-id my-redis-cluster
```

---

## ⭐ Interview Tips & Key Points to Remember

- **Cannot SSH into RDS** — managed service; AWS handles the OS
- **Read Replicas = ASYNC** (slight lag, eventual consistency); **Multi-AZ = SYNC** (zero data loss)
- **Multi-AZ standby = PASSIVE** — not readable, not queryable, just for failover
- **RDS Proxy** = connection pooler for Lambda + RDS — prevents connection storms
- **RDS Proxy reduces failover time by 66%** — maintains connection pool during failover
- **Aurora = 6 copies across 3 AZs** — write quorum 4/6, read quorum 3/6
- **Aurora failover < 30 seconds** vs ~2 minutes for RDS
- **Aurora storage auto-grows** to 128 TB in 10 GB increments — no manual scaling
- **Aurora Serverless v2** = auto-scale compute; ideal for unpredictable workloads
- **Aurora Global**: replication lag < 1s, RTO < 1 minute when promoting secondary
- **Redis vs Memcached**: Redis = persistence + replication + rich data structures; Memcached = simple, fast, multi-thread
- **Cache-Aside pattern**: app manages cache population; lazy — only caches requested data
- **Write-Through pattern**: every write goes to both DB and cache — fresh data but write overhead
- Scenario "Lambda functions overloading RDS connections" → **RDS Proxy**
- Scenario "reduce DB load for read-heavy app" → **ElastiCache** + read replicas
- Scenario "global app, < 1 second replication to Europe" → **Aurora Global Database**

---

## Quick Reference — AWS CLI Commands

### RDS Management

```bash
# Create RDS instance
aws rds create-db-instance \
  --db-instance-identifier my-mysql-db \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password MySecurePass123 \
  --allocated-storage 20 \
  --backup-retention-period 7 \
  --multi-az

# Describe RDS instances
aws rds describe-db-instances --db-instance-identifier my-mysql-db

# Modify RDS (enable Multi-AZ, change size)
aws rds modify-db-instance \
  --db-instance-identifier my-mysql-db \
  --multi-az \
  --apply-immediately

# Create snapshot
aws rds create-db-snapshot \
  --db-instance-identifier my-mysql-db \
  --db-snapshot-identifier my-snapshot-20260306

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier my-restored-db \
  --db-snapshot-identifier my-snapshot-20260306

# Delete RDS instance
aws rds delete-db-instance \
  --db-instance-identifier my-mysql-db \
  --final-db-snapshot-identifier my-final-snapshot
```

### RDS Read Replicas

```bash
# Create read replica (same region)
aws rds create-db-instance-read-replica \
  --db-instance-identifier my-mysql-db-replica \
  --source-db-instance-identifier my-mysql-db

# Create cross-region read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier my-mysql-db-replica-eu \
  --source-db-instance-identifier my-mysql-db \
  --source-region us-east-1 \
  --region eu-west-1

# Promote read replica to standalone DB
aws rds promote-read-replica --db-instance-identifier my-mysql-db-replica
```

### Aurora Management

```bash
# Create Aurora cluster
aws rds create-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --engine aurora-mysql \
  --master-username admin \
  --master-user-password MySecurePass123

# Add reader instance to cluster
aws rds create-db-instance \
  --db-instance-identifier my-aurora-reader-1 \
  --db-cluster-identifier my-aurora-cluster \
  --db-instance-class db.r5.large \
  --engine aurora-mysql

# Create Global Database
aws rds create-global-cluster \
  --global-cluster-identifier my-global-aurora \
  --source-db-cluster-identifier arn:aws:rds:us-east-1:123456789012:cluster:my-aurora-cluster
```

### ElastiCache Management

```bash
# Create Redis cluster
aws elasticache create-cache-cluster \
  --cache-cluster-id my-redis-cluster \
  --cache-node-type cache.t3.micro \
  --engine redis

# Create Memcached cluster
aws elasticache create-cache-cluster \
  --cache-cluster-id my-memcached \
  --cache-node-type cache.t3.micro \
  --engine memcached \
  --num-cache-nodes 3

# Create Redis replication group (Multi-AZ)
aws elasticache create-replication-group \
  --replication-group-id my-redis-rg \
  --cache-node-type cache.r6g.large \
  --engine redis \
  --num-cache-clusters 2 \
  --automatic-failover-enabled \
  --multi-az-enabled

# Describe cache clusters
aws elasticache describe-cache-clusters --show-cache-node-info
```

### Subnet Groups

```bash
# Create RDS subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name my-db-subnet-group \
  --db-subnet-group-description "DB Subnet Group" \
  --subnet-ids subnet-0abc123def456 subnet-0def456abc123

# Create ElastiCache subnet group
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name my-cache-subnet-group \
  --cache-subnet-group-description "Cache Subnet Group" \
  --subnet-ids subnet-0abc123def456 subnet-0def456abc123
```

