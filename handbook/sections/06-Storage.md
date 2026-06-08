# Section 06 — Storage

> **Purpose**: Storage architecture in AWS is not about choosing the "best" storage service — it is about matching access patterns, durability requirements, latency tolerances, and cost constraints to the right abstraction. A principal engineer designs storage systems that survive AZ failures, protect against operator error, and scale from gigabytes to petabytes without rearchitecture.
>
> **Official Documentation**: [S3](https://docs.aws.amazon.com/s3/) | [EBS](https://docs.aws.amazon.com/ebs/) | [EFS](https://docs.aws.amazon.com/efs/) | [FSx](https://docs.aws.amazon.com/fsx/) | [Storage Gateway](https://docs.aws.amazon.com/storagegateway/)

---

## 1. Amazon S3: The Universal Storage Layer

### 1.1 Why S3 Is Fundamental

S3 is not merely "cloud storage." It is the **default integration point** for AWS services. CloudTrail writes to S3. Athena queries S3. EMR processes S3 data. Lambda triggers on S3 events. If you understand S3 deeply, you understand how much of AWS works.

**S3's architectural contract**:
- **11 nines durability** (99.999999999%): For 10,000 objects, expect to lose 1 object every 10 million years. This is achieved via erasure coding across multiple devices and facilities.
- **Availability varies by tier**: 99.99% for Standard, lower for Glacier tiers.
- **Virtually unlimited capacity**: No provisioning required. Pay per GB stored + API calls + data transfer.
- **Regional service**: Data lives in a specific AWS region. Cross-region replication is opt-in.

### 1.2 S3 Consistency Model (Modern Behavior)

As of December 2020, S3 provides **strong read-after-write consistency** for all operations:
- After a successful `PUT` of a new object, any subsequent `GET` returns the latest version.
- After a successful `DELETE`, any subsequent `GET` returns `404`.
- For **LIST operations**: S3 is still eventually consistent. Objects may appear in LIST results before `PUT` fully propagates, or remain visible briefly after `DELETE`.

> **Architectural Note**: Do not design systems that depend on immediate LIST consistency for correctness. If you need to verify object existence, use `HEAD` or `GET` on the specific key, not LIST.

### 1.3 S3 Storage Classes: The Cost-Durability-Access Tradeoff

| Class | Durability | Availability | Min Storage | Retrieval | Use Case |
|-------|-----------|-------------|-------------|-----------|----------|
| **S3 Standard** | 11 nines | 99.99% | None | Milliseconds | Frequently accessed data, CDN origins, active workloads |
| **S3 Intelligent-Tiering** | 11 nines | 99.9% | None | Milliseconds | Unknown/variable access patterns. Auto-moves between tiers. Small monthly monitoring fee. |
| **S3 Standard-IA** | 11 nines | 99.9% | 30 days | Milliseconds | Infrequently accessed but needs immediate access when needed. Min size: 128 KB. |
| **S3 One Zone-IA** | 11 nines (single AZ) | 99.5% | 30 days | Milliseconds | Re-creatable data, secondary copies. 20% cheaper but AZ-loss risk. |
| **S3 Glacier Instant Retrieval** | 11 nines | 99.9% | 90 days | Milliseconds | Archive data with occasional immediate needs. Min size: 128 KB. |
| **S3 Glacier Flexible Retrieval** | 11 nines | 99.99% (after retrieval) | 90 days | 1-5 min (expedited), 3-5 hrs (standard), 5-12 hrs (bulk) | Backups, media archives, compliance. Cheapest storage with flexible retrieval. |
| **S3 Glacier Deep Archive** | 11 nines | 99.9% (after retrieval) | 180 days | 12 hrs (standard), 48 hrs (bulk) | Long-term retention (7-10 years). Cheapest AWS storage. |
| **S3 Express One Zone** | 11 nines (single AZ) | 99.95% | None | Single-digit milliseconds | High-performance, frequently accessed small objects. Directory bucket structure. |

> **Cost Trap**: Small objects in Standard-IA or Glacier incur a minimum 128 KB charge. A 10 KB object in Standard-IA is billed as 128 KB. For many small objects, Standard or Intelligent-Tiering may be cheaper despite higher per-GB rate.

### 1.4 S3 Lifecycle Policies

Lifecycle policies automate storage class transitions and deletion:

```
Day 0:    S3 Standard
Day 30:   → S3 Standard-IA (if accessed infrequently)
Day 90:   → S3 Glacier Flexible Retrieval
Day 365:  → S3 Glacier Deep Archive
Day 2555: → Delete (7-year retention met)
```

**Lifecycle rules can filter by**:
- Prefix (e.g., `logs/` vs `data/`)
- Object tags
- Object size (e.g., only objects > 128 KB)

**Important**: Transitions are charged per request. Transitioning 1 billion objects costs money. For massive datasets, consider batch operations or S3 Batch Operations.

### 1.5 S3 Replication

**Cross-Region Replication (CRR)**:
- Replicates objects to a bucket in a different region.
- Use case: Disaster recovery, latency reduction for global users, compliance (data must be in multiple regions).
- **What replicates**: New objects, object tags, metadata. **What does NOT**: Existing objects (must use S3 Batch Replication), DELETE markers (unless configured), ACLs (unless configured).
- **Replication Time Control (RTC)**: Guarantees replication within 15 minutes. Additional charge.

**Same-Region Replication (SRR)**:
- Replicates within the same region.
- Use case: Aggregate logs from multiple buckets into one central bucket for analysis.

**Important**: Replication requires versioning on both source and destination. The IAM role needs permissions on both buckets. If encryption is used, KMS permissions are required for both source and destination keys.

### 1.6 S3 Security Architecture

**Encryption at Rest**:

| Method | Key Management | Use Case |
|--------|---------------|----------|
| **SSE-S3** | AWS-owned key | Default. Transparent, no cost. |
| **SSE-KMS** | AWS-managed or customer-managed KMS key | Audit trail, key rotation, cross-account sharing. |
| **SSE-C** | Customer-provided key in HTTP header | External key management systems. Key sent with every request. |
| **DSSE-KMS** | Dual-layer SSE-KMS | Highest security — two layers of encryption with independent keys. |

**Access Control**:

| Mechanism | Scope | Best For |
|-----------|-------|----------|
| **IAM Policies** | User/role-level access control | Cross-service access, fine-grained control |
| **Bucket Policies** | Bucket-level rules | Public/private access, cross-account access, IP restrictions |
| **ACLs** | Object-level legacy access | Avoid for new buckets. Use bucket policies instead. |
| **S3 Access Points** | Named network endpoints with dedicated policies | Simplifying access for specific applications or VPCs |
| **Block Public Access** | Account/bucket-level overrides | Preventing accidental public exposure |

> **The Principle of Least Privilege in S3**: Never use `s3:*` in policies. Always scope to specific buckets and actions. A compromised credential with `s3:DeleteBucket` can destroy your entire data lake.

**S3 Block Public Access**: Four independent settings that override bucket policies and ACLs:
1. Block new public ACLs and uploading public objects
2. Remove public access granted through ACLs
3. Block new public bucket policies
4. Block public and cross-account access through bucket policies

Enable all four at the account level. This is a safety net that has prevented countless breaches.

### 1.7 S3 Event Notifications and Lambda Integration

```
S3 Object Created ──► [Event Notification] ──► [SQS / SNS / Lambda / EventBridge]
                                                          │
                                                          ▼
                                                   [Image Processing]
                                                   [Data Validation]
                                                   [ETL Trigger]
```

**Event types**: `s3:ObjectCreated:*`, `s3:ObjectRemoved:*`, `s3:ObjectRestore:*`, `s3:Replication:*`, `s3:LifecycleTransition`

**EventBridge integration** (modern, preferred): EventBridge provides more flexible filtering, cross-account routing, and replay capabilities than raw S3 notifications.

### 1.8 S3 Performance Optimization

| Technique | Effect | Use Case |
|-----------|--------|----------|
| **Multipart Upload** | Parallel upload of large files (5 MB – 5 TB) | Files > 100 MB. Required for files > 5 GB. |
| **S3 Transfer Acceleration** | Uses CloudFront edge locations to speed uploads from distant clients | Global user base uploading to central bucket |
| **Byte-Range Fetches** | Download specific byte ranges in parallel | Large file partial access, video streaming |
| **S3 Batch Operations** | Run operations on billions of objects | Bulk tagging, replication, Glacier restore |
| **S3 Select** | Query CSV/JSON/Parquet in-place | Retrieve subset of data without downloading full object |
| **S3 Express One Zone** | Single-digit millisecond latency | High-frequency small object access (ML training, gaming) |

> **S3 Performance Myth**: "S3 has a limit of 3,500 PUT/s or 5,500 GET/s per prefix." This was historically true but is no longer a hard limit for most workloads. S3 scales automatically. However, for extreme workloads (10,000+ TPS), distributing keys across multiple prefixes can still help with load distribution.

---

## 2. EBS: Block Storage for EC2

### 2.1 EBS Fundamentals

EBS provides **network-attached block storage** for EC2. Key characteristics:
- **AZ-bound**: An EBS volume can only attach to EC2 instances in the same AZ.
- **Persistent**: Survives instance stop/start and termination (if not configured to delete on termination).
- **Snapshot-based backup**: Point-in-time snapshots to S3. Incremental (only changed blocks).

### 2.2 EBS Volume Types (Detailed)

| Volume | IOPS | Throughput | Max Size | Durability | Cost | Best For |
|--------|------|-----------|----------|-----------|------|----------|
| **gp3** | 3,000-16,000 configurable | 125-1,000 MiB/s | 16 TB | 99.8%-99.9% | Low | General purpose. Boot volumes, dev environments, small-medium databases. |
| **io2** | 16,000-256,000 | 1,000 MiB/s | 64 TB | 99.999% | High | I/O-intensive databases (Oracle, SAP). Highest durability SLA. |
| **io2 Block Express** | 256,000-1,000,000 | 4,000 MiB/s | 64 TB | 99.999% | Highest | Sub-millisecond latency, highest IOPS. Enterprise databases. |
| **st1** | 500 IOPS/ TB baseline | 500 MiB/s | 16 TB | 99.8%-99.9% | Low | Big data, log processing, sequential I/O. |
| **sc1** | 250 IOPS/ TB baseline | 250 MiB/s | 16 TB | 99.8%-99.9% | Lowest | Cold data, infrequent access. |

> **gp3 IOPS/Throughput are independent**: With gp2, IOPS was tied to volume size (3 IOPS/GB). With gp3, a 100 GB volume can have 16,000 IOPS and 1,000 MiB/s. This is a game-changer for small, high-performance workloads.

### 2.3 EBS Snapshots and Recovery

- **Snapshots are incremental**: First snapshot is full; subsequent snapshots only store changed blocks.
- **Snapshot storage in S3**: Snapshots live in S3 (invisible to you, managed by AWS).
- **Cross-region copy**: Snapshots can be copied to other regions for DR.
- **Fast Snapshot Restore (FSR)**: Pre-warm snapshots so volumes created from them are fully performant immediately. Without FSR, new volumes from snapshots have lazy loading (data fetched on first access).

> **EBS Multi-Attach (io1/io2 only)**: A single EBS volume can attach to multiple EC2 instances simultaneously. Use case: Shared block storage for clustered databases (e.g., Oracle RAC). Requires cluster-aware file systems.

---

## 3. EFS: Managed NFS

### 3.1 EFS Architecture

EFS provides a **managed NFSv4 file system** that multiple EC2 instances can mount simultaneously.

**Performance Modes**:

| Mode | Throughput | Latency | Use Case |
|------|-----------|---------|----------|
| **General Purpose** (default) | Bursts to 3+ GB/s | Low | Web serving, content management, home directories, config sharing |
| **Max I/O** | Scales to 10+ GB/s, 500,000+ IOPS | Higher latency | Big data, media processing, high-aggregate-IOPS workloads |

**Throughput Modes**:
- **Bursting**: Scales with file system size. More data = more throughput baseline. Burst credits for spikes.
- **Provisioned**: Set throughput independently of storage size. Useful when data is small but access is frequent.
- **Elastic** (new): Automatically scales throughput based on demand. No provisioning needed.

**Storage Classes**:
- **Standard**: Frequently accessed files.
- **EFS IA** (Infrequent Access): Cheaper for files not accessed daily. Transparent via Lifecycle Management.
- **EFS Archive**: Cheapest for rarely accessed files. Similar to S3 Glacier for file systems.

> **EFS Limitation**: EFS is a network file system. It is NOT suitable for high-IOPS databases, boot volumes, or applications requiring local block storage. Use EBS for databases, EFS for shared file workloads.

---

## 4. FSx: Specialized File Systems

| Service | Protocol | Underlying OS | Use Case |
|---------|----------|--------------|----------|
| **FSx for Windows File Server** | SMB | Windows Server | Windows apps, user home directories, SQL Server failover clusters |
| **FSx for Lustre** | POSIX/Lustre | Lustre | HPC, ML training, video rendering. Scales to 100+ GB/s. |
| **FSx for NetApp ONTAP** | NFS, SMB, iSCSI | NetApp ONTAP | Enterprise NAS migration, multiprotocol access, snapshots, cloning |
| **FSx for OpenZFS** | NFS | OpenZFS | Linux workloads needing ZFS features (compression, snapshots, clones) |

**FSx for Lustre + S3 Integration**: Lustre can be linked to an S3 bucket as its backing store. Data in S3 appears as files in Lustre. Modified files are automatically written back to S3. This is the standard pattern for HPC workloads that process S3 data.

---

## 5. Storage Gateway: Hybrid Cloud Bridge

Storage Gateway connects on-premises environments to AWS storage.

| Gateway Type | Interface | Use Case | Data Location |
|-------------|-----------|----------|---------------|
| **File Gateway** | NFS/SMB | Extend on-prem file storage to S3. | Hot cache local, full data in S3 |
| **Volume Gateway (Cached)** | iSCSI | Primary data in S3, hot cache local. | S3 primary, local cache |
| **Volume Gateway (Stored)** | iSCSI | Full copy local, async backup to S3. | Full local + S3 backup |
| **Tape Gateway** | iSCSI (VTL) | Replace physical tape backup with virtual tapes in S3/Glacier. | S3 (virtual tape library) |

> **Storage Gateway is NOT a replacement for high-performance local storage** — it is a bridge. Latency to S3 is higher than local disk. Use it for tiering, backup, and DR, not primary production databases.

---

## 6. Storage Architecture Decision Framework

```
What is the access pattern?
├── Block storage needed?
│   ├── Single instance?           → EBS (gp3 for general, io2 for databases)
│   └── Multiple instances?        → EFS (NFS) or FSx (specialized protocols)
├── Object storage needed?
│   ├── Frequently accessed?       → S3 Standard or Intelligent-Tiering
│   ├── Infrequently accessed?   → S3 Standard-IA or Glacier Instant Retrieval
│   ├── Archive/backup?            → Glacier Flexible Retrieval or Deep Archive
│   └── Unknown pattern?         → S3 Intelligent-Tiering
├── File sharing?
│   ├── Linux workloads?         → EFS
│   ├── Windows workloads?       → FSx for Windows
│   ├── HPC/ML?                  → FSx for Lustre
│   └── Enterprise NAS?          → FSx for NetApp ONTAP
└── Hybrid/on-prem extension?    → Storage Gateway
```

---

## 7. Architectural Decision Challenges

* **Scenario:** Design a data lake that stores 10 PB of log data with 7-year retention, queryable by analysts.
  * **Design:** Ingest logs via Kinesis Data Firehose to S3, transform to Parquet using Glue/Lambda, and query directly with Athena. Because Parquet minimizes Athena scan costs, and S3 Lifecycle policies can automatically transition raw data to Glacier Deep Archive, providing cost-effective 10 PB storage with enforced 7-year retention.

* **Scenario:** An application needs shared storage across 50 EC2 instances in multiple AZs. EBS or EFS?
  * **Design:** EFS. Because EBS volumes are AZ-bound and typically attach to a single instance. EFS is designed for multi-AZ, multi-instance shared access via NFS (or FSx for Windows if Windows-based).

* **Scenario:** Why would you use S3 Intelligent-Tiering over manually managing lifecycle policies?
  * **Design:** Use S3 Intelligent-Tiering. Because it removes the operational burden of predicting access patterns by automatically moving infrequently accessed objects to cheaper tiers. This typically yields savings that outweigh its small monitoring fee, avoiding the overhead of manual lifecycle management.

---

## 8. Points to Remember

- **S3 durability is 11 nines across all classes** — the durability SLA does not decrease for cheaper tiers. Availability and retrieval time vary.
- **S3 is strongly consistent for read-after-write since Dec 2020** — LIST operations remain eventually consistent.
- **Minimum object size charges apply to IA and Glacier tiers** — 128 KB minimum billed per object.
- **S3 Block Public Access should be enabled at the account level** — it is the most effective accidental-exposure prevention.
- **EBS volumes are AZ-bound** — to move data across AZs, snapshot and restore, or use EFS.
- **gp3 decouples IOPS/throughput from volume size** — a 100 GB gp3 volume can have 16,000 IOPS.
- **EBS snapshots are incremental and stored in S3** — cross-region snapshot copies enable DR.
- **EFS is NOT a database storage solution** — use EBS (io2) or native database storage for transactional workloads.
- **FSx for Lustre integrates with S3** — use this for HPC workloads that need POSIX access to S3 data.
- **Intelligent-Tiering has a small monitoring fee** — but it eliminates the risk of misclassifying access patterns.
- **S3 Object Lock (WORM)** prevents object deletion/modification for a retention period — critical for compliance.
- **S3 Transfer Acceleration uses CloudFront edge locations** — beneficial for global upload patterns, not same-region traffic.

---

## 13. Further Reading

For deeper coverage, CLI commands, and exam-specific trivia, see the detailed reference:

- **EBS, EFS, Instance Store**: [`EC2-InstanceStorage-EBS-EFS-InstanceStore.md`](../../detailed-reference/EC2-InstanceStorage-EBS-EFS-InstanceStore.md)
- **S3 buckets, objects, storage classes, lifecycle**: [`S3-Basics-BucketsObjectsStorageClasses.md`](../../detailed-reference/S3-Basics-BucketsObjectsStorageClasses.md)
- **S3 encryption, policies, access points, Object Lambda**: [`S3-Advanced-Security-Encryption.md`](../../detailed-reference/S3-Advanced-Security-Encryption.md)
- **FSx, Snowball, Storage Gateway, DataSync**: [`StorageExtras-FSx-Snowball-Gateway.md`](../../detailed-reference/StorageExtras-FSx-Snowball-Gateway.md)

---

*Section 06 — Storage | Last Validated: 2026-05-10*
