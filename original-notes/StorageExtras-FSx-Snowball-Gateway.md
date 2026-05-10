# AWS Storage Extras — FSx, Snow Family, Storage Gateway & DataSync

> **Exam**: AWS SAA-C03 | **Topic Weight**: Medium-High (8–12% of questions touch hybrid storage and migration)

## Official AWS Documentation

- [Amazon FSx for Windows File Server](https://docs.aws.amazon.com/fsx/latest/WindowsGuide/what-is.html)
- [Amazon FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html)
- [AWS Snowball / Snow Family](https://docs.aws.amazon.com/snowball/latest/developer-guide/whatissnowball.html)
- [AWS DataSync](https://docs.aws.amazon.com/datasync/latest/userguide/what-is-datasync.html)
- [AWS Storage Gateway](https://docs.aws.amazon.com/storagegateway/latest/userguide/WhatIsStorageGateway.html)

---

## The Big Picture — Why These Services Exist

By now you know the core AWS storage trio: **S3** (objects), **EBS** (block, single-instance), **EFS** (shared file, Linux). But enterprises don't live in an ideal world. They have:

- Windows servers that speak **SMB**, not NFS
- HPC clusters running **Lustre** for petabyte-scale parallelism
- Petabytes of on-premises data they simply can't upload over a slow internet pipe
- Legacy backup systems they can't afford to rip-and-replace overnight

This is where the "extras" come in. Think of this section as **"AWS for the real enterprise world"** — hybrid, heterogeneous, and often constrained by physical reality.

---

## Section 1 — Amazon FSx Overview

**What is FSx?** Launch 3rd-party, high-performance file systems on AWS as fully managed services. No patching, no hardware, no capacity planning — just pick your protocol.

**Mental model**: EFS is Amazon's own generic NFS (Linux, POSIX). FSx is the "bring your own enterprise file system" option. AWS manages the infrastructure; you get the familiar protocol.

**The four FSx flavours:**

```
Amazon FSx
├── FSx for Windows File Server   → SMB / NTFS   (Windows ecosystem)
├── FSx for Lustre                → Lustre API    (HPC / ML)
├── FSx for NetApp ONTAP          → Multi-protocol (NFS, SMB, iSCSI)
└── FSx for OpenZFS               → NFS / ZFS     (ZFS workloads)
```

For the SAA-C03 exam, focus deepens on **Windows** and **Lustre**. NetApp ONTAP and OpenZFS appear occasionally but at a shallower level.

---

## Section 2 — Amazon FSx for Windows File Server

### The Problem It Solves

Your company has a fleet of Windows EC2 instances and on-premises Windows servers. They need shared file storage — a classic Windows file share. You might think "use EFS" — but EFS speaks NFS, a Linux protocol. Windows doesn't natively mount NFS. The solution is **FSx for Windows File Server**, which speaks SMB and NTFS, the language Windows has always used.

**Analogy**: EFS is to Linux what FSx for Windows is to Windows — but FSx for Windows also works for Linux clients that have SMB support.

### Key Features

| Feature | Detail |
|---------|--------|
| Protocols | **SMB** (Server Message Block) + **Windows NTFS** |
| Authentication | **Microsoft Active Directory** integration — domain-joined, ACLs, user quotas |
| Linux support | Can be mounted on Linux EC2 instances (via SMB client) |
| DFS Namespaces | Group files across multiple file systems under a single unified namespace |
| Performance | Scales to 10s GB/s, millions of IOPS, 100s PB |
| Storage tiers | **SSD** (latency-sensitive: databases, media) / **HDD** (broad: home dirs, CMS) |
| On-prem access | Via **VPN** or **Direct Connect** |
| High Availability | **Multi-AZ** deployment (primary + standby with automatic failover) |
| Backup | Daily automated backups to **Amazon S3** |

### Architecture Diagram

```
┌─────────────────────────────── AWS Cloud ──────────────────────────────┐
│                                                                         │
│   Windows EC2 instances (AD-joined)                                     │
│           │ SMB + NTFS                                                  │
│           │                                                             │
│   Linux EC2 instances                                                   │
│           │ SMB (cross-platform)                                        │
│           │                                                             │
│           ▼                                                             │
│   ┌──────────────────────────────────────┐                             │
│   │   FSx for Windows File Server        │                             │
│   │   ┌────────────────────────────────┐ │                             │
│   │   │ Primary (AZ 1) ↔ Standby (AZ 2)│ │  ← Multi-AZ HA             │
│   │   └────────────────────────────────┘ │                             │
│   │   Active Directory integration       │                             │
│   │   Daily backup ──────────────────────┼──► Amazon S3               │
│   └──────────────────────────────────────┘                             │
└─────────────────────────────────────────────────────────────────────────┘
          ▲
          │ VPN or Direct Connect
          │
  On-premises Windows PCs / Servers
```

### When to choose FSx for Windows vs EFS

- **EFS** → Linux workloads, POSIX file system, NFS protocol, general-purpose shared storage
- **FSx for Windows** → Windows workloads, SMB protocol, Active Directory integration, Windows NTFS ACLs

---

## Section 3 — Amazon FSx for Lustre

### What Is Lustre?

**Lustre** = **"Linux"** + **"cluster"**. It's an open-source, parallel distributed file system born in the world of supercomputing. The name itself tells you the target audience: Linux clusters doing intense parallel I/O.

**Why does this matter for cloud?** When you're training a deep learning model with 100 GPUs, each GPU needs to read training data simultaneously. A normal NFS file system becomes the bottleneck. Lustre parallelises reads across many storage nodes — all 100 GPUs get full speed at the same time.

### Use Cases

- Machine Learning (training at scale)
- **High Performance Computing (HPC)** — genomics, physics simulations, computational fluid dynamics
- Video processing and rendering pipelines
- Financial modelling, Electronic Design Automation (EDA)

### Performance Numbers

| Metric | Value |
|--------|-------|
| Throughput | 100s of GB/s |
| IOPS | Millions |
| Latency | **Sub-millisecond** |

### Storage Options

| Tier | Best For |
|------|---------|
| **SSD** | Low-latency, IOPS-intensive, small & random file operations |
| **HDD** | Throughput-intensive, large & sequential file operations (cheaper per TB) |

### Key Feature — Seamless S3 Integration

This is one of the most powerful things about FSx for Lustre: it can treat an S3 bucket as a data repository.

- **Read S3 as a file system**: Data is lazily loaded from S3 on first access — FSx fetches only what you actually read, just-in-time
- **Write results back to S3**: Computation output is automatically synced back to S3 through FSx

**Practical example** — ML training pipeline:
```
ML Training Data
(stored in S3)
        │
        │ lazy-load on first access
        ▼
FSx for Lustre
(ultra-fast parallel access)
        │
        │ GPU cluster reads at 100s GB/s
        ▼
Trained Model + Results
        │
        │ write-back
        ▼
Amazon S3 (stored for long-term)
```

**On-premises access**: HPC clusters on-prem can access FSx for Lustre via VPN or Direct Connect — you get cloud-scale storage without migrating your compute cluster.

---

### FSx for Lustre — Deployment Options

This is frequently tested. You pick the deployment type based on whether your data needs to survive a hardware failure.

#### Scratch File System (Temporary)

- Data is **NOT replicated** — no redundancy at the storage node level
- If the file server fails → data is lost
- **Trade-off**: No replication = more storage allocated to actual data = 6x higher burst throughput (200 MB/s per TiB)
- **Use case**: Short-term processing jobs where speed matters more than durability (you can re-run if it fails), cost optimisation

```
SCRATCH — Fast, Temporary:
┌──────────── Region ────────────────────────────────────────────┐
│                                                                 │
│  Availability Zone 1                    Availability Zone 2   │
│  ┌─────────────────────────────────┐                           │
│  │ [Compute] ─┐                   │                           │
│  │ [Compute] ─┤── ENI ──► FSx for Lustre                      │
│  │ [Compute] ─┘           (NOT replicated)        [S3 bucket] │
│  └─────────────────────────────────┘    ◄──────── (optional)  │
│                               ↑                                │
│                     HIGH BURST PERFORMANCE                     │
│                       200 MB/s per TiB                         │
└─────────────────────────────────────────────────────────────────┘
```

#### Persistent File System (Durable)

- Data **IS replicated within the same AZ** (protects against hardware failure)
- Failed files are automatically replaced within minutes
- **Use case**: Long-term processing workloads, sensitive data that must survive individual disk/server failures

```
PERSISTENT — Durable, Long-Term:
┌──────────── Region ────────────────────────────────────────────┐
│                                                                 │
│  Availability Zone 1                    Availability Zone 2   │
│  ┌─────────────────────────────────┐                           │
│  │ [Compute] ─┐                   │                           │
│  │ [Compute] ─┤── ENI ──► FSx for Lustre                      │
│  │ [Compute] ─┘     [data replicated  ]          [S3 bucket]  │
│  └─────────────────────────────────┘    ◄──────── (optional)  │
│                               ↑                                │
│              Replication within AZ = survives hardware failure │
└─────────────────────────────────────────────────────────────────┘
```

---

## Section 4 — FSx Service Comparison

| Feature | FSx for Windows | FSx for Lustre | Amazon EFS |
|---------|----------------|----------------|------------|
| Protocol | SMB, NTFS | POSIX, Lustre API | NFS v4 |
| OS | Windows (+ Linux SMB) | Linux | Linux |
| Use case | Windows apps, AD auth | HPC, ML, analytics | General Linux shared storage |
| Throughput | 10s GB/s | **100s GB/s** | General purpose |
| S3 integration | No | **Yes** (lazy load + write-back) | No |
| On-prem access | VPN / Direct Connect | VPN / Direct Connect | Via DataSync |
| Persistence | Multi-AZ HA | Scratch (no HA) or Persistent (AZ-level HA) | Multi-AZ |
| Auth | Active Directory | POSIX permissions | POSIX + IAM |

---

## Section 5 — AWS Snow Family: Offline Data Migration

### The Uncomfortable Truth About Internet Uploads

Everyone's first instinct is "just upload to S3." But let's look at the math:

| Data Size | 100 Mbps line | 1 Gbps line | 10 Gbps line |
|-----------|--------------|-------------|--------------|
| 10 TB | 12 days | 30 hours | 3 hours |
| 100 TB | **124 days** | 12 days | 30 hours |
| **1 PB** | **3 years** | **124 days** | 12 days |

And that's under ideal conditions. Real-world challenges compound this:
- Limited or shared bandwidth
- High network transfer costs
- Connection instability over long periods
- You can't take the production database offline for weeks

**The golden rule**: If transferring over the network would take **more than 1 week** → use Snowball devices instead.

**Analogy**: Sometimes it's faster to physically drive a truck full of hard drives than to upload over the wire. AWS literally does this — they ship you a ruggedised device, you load it, ship it back, and AWS imports the data. This is faster and cheaper than paying for a 10 Gbps line for months.

### AWS Snowball Edge

AWS ships a physical, ruggedised device to your location. You attach it to your local network, copy data using the Snowball client, and ship it back. AWS imports the data into Amazon S3.

**Two variants:**

| Device | Compute | Memory | Storage (SSD) | Best For |
|--------|---------|--------|---------------|----------|
| Snowball Edge **Storage** Optimized | 104 vCPUs | 416 GB | **210 TB** | Pure data migration |
| Snowball Edge **Compute** Optimized | 104 vCPUs | 416 GB | **28 TB** | Edge computing + migration |

### Data Migration Workflow

```
YOUR DATA CENTER
       │
       │ 1. Copy data over local network
       ▼
 ┌─────────────┐
 │ Snowball    │  ← AWS ships this to you
 │ Edge Device │
 └─────────────┘
       │
       │ 2. Ship device back to AWS
       ▼
  AWS Facility
       │
       │ 3. AWS imports data
       ▼
  Amazon S3 bucket  ✓
```

---

## Section 6 — Edge Computing with Snowball

### What Is Edge Computing?

**The concept**: Process data WHERE IT'S CREATED, rather than shipping it to a central cloud region for processing.

Why does this matter? Some locations have:
- No reliable internet connection
- No connectivity to cloud regions at all
- Real-time processing requirements that can't tolerate upload latency

**Real-world examples:**
- A truck on a remote highway monitoring road conditions
- A ship crossing the ocean with satellite-only connectivity
- A mining station underground with no network infrastructure
- An oil rig in the middle of the ocean

```
Remote Location (no internet)
┌──────────────────────────────────┐
│  Sensors / Cameras               │
│         │ raw data               │
│         ▼                        │
│  ┌─────────────────┐             │
│  │ Snowball Edge   │             │
│  │ Compute Optimized            │
│  │                 │             │
│  │ ┌─────────────┐ │             │
│  │ │ EC2 Instance│ │             │  ← Run ML inference here
│  │ │ Lambda fn   │ │             │  ← Process video here
│  │ └─────────────┘ │             │  ← Aggregate data here
│  └─────────────────┘             │
│         │ processed results      │
│         ▼                        │
│  Local storage (28 TB SSD)       │
└──────────────────────────────────┘
         │
         │ Eventually: ship device to AWS
         ▼
    Amazon S3 (ingest results)
```

**Snowball Edge Compute Optimized use cases:**
- Machine learning inference at the edge (classify images before connectivity)
- Real-time video transcoding
- Preprocessing IoT sensor data to reduce upload volume
- Running applications in completely disconnected environments

**Snowball Edge Storage Optimized use cases:**
- Bulk data collection in remote locations (less compute-intensive)
- Large-volume migration with minimal on-device processing

---

## Section 7 — Snowball to Glacier: The Gotcha

One of the most commonly tested tricky questions:

> "Can Snowball import data directly to S3 Glacier?"

**Answer: NO.**

**Why?** S3 Glacier is a storage class within S3, not a separate service with its own import endpoint. You can't point a Snowball import job directly at Glacier.

**The correct architecture:**

```
Snowball Edge
     │
     │ import
     ▼
Amazon S3 (Standard or Standard-IA)
     │
     │ S3 Lifecycle Policy
     │ (Transition after N days)
     ▼
Amazon S3 Glacier (or Glacier Deep Archive)
```

Configure an S3 Lifecycle Policy on the bucket to automatically transition objects to Glacier after they arrive. This is a two-step process but it's the only supported path.

---

## Section 8 — AWS Storage Gateway

### The Hybrid Storage Problem

Most enterprises don't move to cloud all at once. They have on-premises servers, legacy applications, and existing workflows that can't change overnight. But they want to use cloud storage for cost, durability, and scalability. **Storage Gateway** is the bridge.

**Mental model**: Storage Gateway is a VM (or hardware appliance) you install on-premises. It presents familiar local storage interfaces (NFS, SMB, iSCSI) to your on-prem apps, but transparently stores the actual data in AWS behind the scenes.

```
On-Premises Environment
┌──────────────────────────────────┐
│  Application Server              │
│       │ NFS / SMB / iSCSI        │
│       ▼                          │
│  ┌───────────────────────────┐   │
│  │  Storage Gateway          │   │   ← VM on VMware/Hyper-V
│  │  (local cache)            │   │     or hardware appliance
│  └───────────────────────────┘   │
│       │ HTTPS (encrypted)        │
└───────┼──────────────────────────┘
        │
        ▼
   AWS Cloud Storage
   (S3 / Glacier / EBS / FSx)
```

### The Four Gateway Types

#### 1. S3 File Gateway

Exposes an S3 bucket as an NFS or SMB file share to on-premises applications. Files written to the share are stored as S3 objects. A local cache stores recently accessed data for low-latency reads.

```
On-prem app → NFS/SMB → S3 File Gateway → S3 bucket
                         (local cache)      │
                                           S3 Lifecycle
                                            │
                                           Glacier
```

- Good for: Backup, archiving, file shares with cloud tiering
- AD integration available for SMB access control
- Works with S3 Intelligent-Tiering and lifecycle policies

#### 2. Volume Gateway

Presents iSCSI block devices to on-premises servers — works just like a SAN.

Two modes:

| Mode | Primary Storage | Backup |
|------|----------------|--------|
| **Stored** volumes | On-premises (entire dataset local) | Async EBS snapshots to S3 |
| **Cached** volumes | Amazon S3 (cloud-primary) | Frequently accessed data cached on-prem |

- **Stored**: Low-latency for all data (it's all local), backed up asynchronously
- **Cached**: Reduced on-prem storage footprint, slightly higher latency for non-cached data

#### 3. Tape Gateway

Replaces physical tape backup infrastructure with virtual tapes stored in S3 and Glacier, without changing your backup software.

```
Backup Software (Veeam, Veritas, Backup Exec)
         │ iSCSI VTL (Virtual Tape Library)
         ▼
  Tape Gateway (VM on-prem)
         │
         ▼
  Virtual tapes in S3 (active)
         │ archived
         ▼
  Virtual tapes in Glacier / Glacier Deep Archive
```

- No changes to existing backup workflows
- Unlimited virtual tape capacity
- Dramatically lower cost than physical tape infrastructure

#### 4. FSx File Gateway

A local cache specifically for **FSx for Windows File Server**.

When on-premises Windows clients access your FSx file system directly over the WAN, every read involves a round-trip to AWS — latency adds up. FSx File Gateway maintains a local cache of frequently accessed files, giving on-prem clients near-local-disk performance while the authoritative data lives in FSx.

### Hardware Appliance

What if your data centre doesn't have VMware or Hyper-V to run the Storage Gateway VM? AWS sells a **Storage Gateway Hardware Appliance** — a physical rack-mountable device that runs the gateway software. Just plug it in, activate it, and it works.

---

## Section 9 — AWS DataSync

### The Problem DataSync Solves

You have large amounts of data on-premises (NFS server, Windows file server, HDFS cluster) and you need to migrate it to AWS — but you DO have sufficient internet bandwidth or a Direct Connect link. You don't want to ship physical devices. But you also don't want to write custom scripts to handle retries, metadata preservation, bandwidth throttling, scheduling, and error handling.

**DataSync** is a managed data transfer service that handles all of this for you.

### How It Works

```
On-Premises (or Other Cloud)                   AWS Cloud
┌────────────────────────────┐           ┌────────────────────────────┐
│  NFS Server                │           │                            │
│  SMB Server                │           │  Amazon S3 (any class)     │
│  HDFS Cluster              ├──────────►│  Amazon EFS                │
│                            │ HTTPS     │  Amazon FSx (all types)    │
│  ┌──────────────────────┐  │ encrypted │                            │
│  │ DataSync Agent (VM)  │  │           └────────────────────────────┘
│  └──────────────────────┘  │
└────────────────────────────┘

Also supports cloud-to-cloud:
Azure Blob / Google Cloud Storage → Amazon S3
```

### Key Characteristics

| Feature | Detail |
|---------|--------|
| Agent | Lightweight VM deployed on-premises or in another cloud |
| Scheduling | Hourly, daily, weekly, or on-demand |
| Metadata | Preserves file permissions, timestamps, ownership |
| Encryption | TLS in transit; data encrypted at rest in destination |
| Bandwidth | Configurable throttling, up to 10 Gbps per task |
| Sources | NFS, SMB, HDFS, S3-compatible APIs, Azure Blob, GCS |
| Destinations | S3 (any class), EFS, FSx for Windows/Lustre/OpenZFS/NetApp |

### DataSync vs Snow Family — When to Use Each

| Scenario | Tool |
|----------|------|
| Good internet bandwidth, < 1 week transfer | **AWS DataSync** |
| Petabyte scale, or no internet connectivity | **Snowball Edge** |
| Ongoing incremental sync (recurring) | **AWS DataSync** |
| One-time bulk migration | **Snowball Edge** |
| Need to process data at remote location | **Snowball Edge Compute Optimized** |

---

## Section 10 — Storage Decision Tree

Use this flowchart to answer scenario questions on the exam:

```
What do you need to do?
│
├─ Share files across EC2 instances / containers
│   ├─ Linux, general purpose → Amazon EFS (NFS)
│   ├─ Windows, SMB, Active Directory → FSx for Windows File Server
│   └─ HPC / ML, maximum throughput → FSx for Lustre
│
├─ Migrate large data to AWS
│   ├─ Good internet, < 1 week → AWS DataSync
│   ├─ Petabyte scale OR no internet → AWS Snowball Edge (Storage Optimized)
│   └─ Process data first at remote location → Snowball Edge (Compute Optimized)
│
├─ Extend on-prem storage to cloud (hybrid)
│   ├─ File share (NFS/SMB) → Storage Gateway: S3 File Gateway
│   ├─ Block storage (iSCSI/SAN) → Storage Gateway: Volume Gateway
│   ├─ Tape backup replacement → Storage Gateway: Tape Gateway
│   └─ Windows file server with local cache → Storage Gateway: FSx File Gateway
│
└─ Archive data long-term from Snowball
    └─ Snowball → S3 → S3 Lifecycle Policy → Glacier
                  (CANNOT go to Glacier directly!)
```

---

## Points to Remember (Exam Focus)

1. **FSx for Windows** = SMB + NTFS + Active Directory. Multi-AZ. Can also be mounted on Linux. Daily S3 backups.

2. **FSx for Lustre** = "Linux" + "cluster". HPC and ML. Integrates with S3 (lazy load + write-back). Sub-millisecond latency.

3. **Scratch vs Persistent Lustre**: Scratch = no replication, max burst speed, temporary. Persistent = replicated within AZ, survives hardware failures.

4. **Snowball Rule**: If data transfer over internet takes more than 1 week → use Snowball.

5. **Snowball cannot import to Glacier directly** — must land in S3 first, then use a lifecycle policy to transition to Glacier.

6. **Snowball Edge Compute Optimized** = run EC2 instances and Lambda functions on the device at remote locations with no internet.

7. **DataSync** = online transfer with an agent VM. Preserves file metadata, permissions, timestamps. Supports scheduling and throttling.

8. **Storage Gateway types**: S3 File GW (NFS/SMB→S3), Volume GW (iSCSI→EBS snapshots), Tape GW (VTL→Glacier), FSx File GW (local cache for FSx Windows).

9. **EFS vs FSx for Windows**: EFS = Linux + NFS. FSx for Windows = Windows + SMB. Never swap these on the exam.

10. **S3 is object storage** — it cannot be mounted as a POSIX file system. Applications that need a file system must use EFS or FSx.

---

## Interview Tips

| Scenario | Answer |
|----------|--------|
| "Windows apps need shared file storage in AWS" | FSx for Windows File Server (SMB + AD) |
| "ML training at HPC scale needs fast shared storage" | FSx for Lustre (parallel FS, S3 integration) |
| "Ship 200 TB to AWS, no fast internet" | AWS Snowball Edge (Storage Optimized) |
| "Run ML inference on a ship with no internet" | Snowball Edge Compute Optimized (edge computing) |
| "Migrate NFS server to AWS online" | AWS DataSync |
| "Replace physical tape backup system" | AWS Storage Gateway — Tape Gateway |
| "On-prem app needs to write to S3 using NFS" | AWS Storage Gateway — S3 File Gateway |
| "Windows clients need low-latency access to FSx" | FSx File Gateway (local cache) |
| "Need Glacier archival from Snowball import" | Snowball → S3 → Lifecycle policy → Glacier |
| "HPC cluster needs fast temp storage for a batch job" | FSx for Lustre — Scratch File System |

## Quick Reference — AWS CLI Commands

### Amazon FSx Commands

```bash
# Create FSx for Windows File Server
aws fsx create-file-system \
  --file-system-type WINDOWS \
  --storage-capacity 300 \
  --subnet-ids subnet-0abc123def456 \
  --windows-configuration '{
    "ThroughputCapacity": 8,
    "ActiveDirectoryId": "d-1234567890",
    "DeploymentType": "MULTI_AZ_1",
    "PreferredSubnetId": "subnet-0abc123def456",
    "AutomaticBackupRetentionDays": 7
  }'

# Create FSx for Lustre (Scratch — for temp HPC jobs)
aws fsx create-file-system \
  --file-system-type LUSTRE \
  --storage-capacity 1200 \
  --subnet-ids subnet-0abc123def456 \
  --lustre-configuration '{
    "DeploymentType": "SCRATCH_2",
    "ImportPath": "s3://my-ml-training-data",
    "ExportPath": "s3://my-ml-training-data/output"
  }'

# Create FSx for Lustre (Persistent)
aws fsx create-file-system \
  --file-system-type LUSTRE \
  --storage-capacity 1200 \
  --subnet-ids subnet-0abc123def456 \
  --lustre-configuration '{
    "DeploymentType": "PERSISTENT_2",
    "PerUnitStorageThroughput": 125
  }'

# Describe FSx file systems
aws fsx describe-file-systems

# Delete FSx file system
aws fsx delete-file-system --file-system-id fs-0abc123def456
```

### DataSync Commands

```bash
# Create DataSync S3 location (destination)
aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::my-destination-bucket \
  --s3-config BucketAccessRoleArn=arn:aws:iam::123456789012:role/DataSyncRole

# Create DataSync NFS location (source on-premises)
aws datasync create-location-nfs \
  --server-hostname 192.168.1.100 \
  --subdirectory /data/exports \
  --on-prem-config AgentArns=arn:aws:datasync:us-east-1:123456789012:agent/agent-abc123

# Create DataSync task (NFS → S3)
aws datasync create-task \
  --name "Migrate NFS to S3" \
  --source-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-source \
  --destination-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-dest \
  --options '{
    "PreserveDeletedFiles": "PRESERVE",
    "PreserveDevices": "NONE",
    "Uid": "INT_VALUE",
    "Gid": "INT_VALUE",
    "LogLevel": "TRANSFER"
  }'

# Start DataSync task execution
aws datasync start-task-execution \
  --task-arn arn:aws:datasync:us-east-1:123456789012:task/task-abc123

# Check task execution status
aws datasync describe-task-execution \
  --task-execution-arn arn:aws:datasync:us-east-1:123456789012:task/task-abc123/execution/exec-abc123
```

### Storage Gateway Commands

```bash
# Activate a gateway (after deploying the VM)
aws storagegateway activate-gateway \
  --activation-key "ABCDE-12345-FGHIJ-67890-KLMNO" \
  --gateway-name my-file-gateway \
  --gateway-timezone "GMT" \
  --gateway-region us-east-1 \
  --gateway-type FILE_S3

# Create NFS file share (S3 File Gateway)
aws storagegateway create-nfs-file-share \
  --client-token "token-$(date +%s)" \
  --gateway-arn arn:aws:storagegateway:us-east-1:123456789012:gateway/sgw-ABC12DEF \
  --role arn:aws:iam::123456789012:role/StorageGatewayRole \
  --location-arn arn:aws:s3:::my-bucket \
  --nfs-file-share-defaults FileMode=0666,DirectoryMode=0777

# List gateways
aws storagegateway list-gateways

# List file shares
aws storagegateway list-file-shares \
  --gateway-arn arn:aws:storagegateway:us-east-1:123456789012:gateway/sgw-ABC12DEF
```

---

