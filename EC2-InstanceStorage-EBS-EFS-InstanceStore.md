# Amazon EC2 – Instance Storage (EBS, EFS, Instance Store)
> 📚 Official Docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Storage.html  
> 🎯 SAA-C03 Exam Weight: High — many scenario questions about storage choice

---

## 🗂️ Storage Options Overview

When you run an EC2 instance, you have three fundamentally different ways to store data. Understanding the difference is crucial:

```
Storage Options for EC2:
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  1. EBS Volume  ─── Network-attached block storage         │
│     (like plugging in an external hard drive over network) │
│                                                            │
│  2. Instance Store ─── Physical disk on the host server    │
│     (like the internal SSD of your laptop)                 │
│                                                            │
│  3. EFS ─── Shared network file system                     │
│     (like a NAS drive that multiple servers can mount)     │
└────────────────────────────────────────────────────────────┘
```

---

## 💾 EBS — Elastic Block Store

### What Is It?
EBS is a **network-attached block storage device** — like an external hard drive connected over a very fast network. Your instance reads and writes to it over the network (slight latency, but very durable and flexible).

### Key Characteristics

```
EC2 Instance (AZ: us-east-1a)
        │
        │ (network connection — very fast, but network)
        │
   ┌────┴────────────────────────────────┐
   │  EBS Volume (also in us-east-1a)    │
   │  - 100 GB gp3                       │
   │  - 3000 IOPS                        │
   │  - Data survives instance stop!     │
   └─────────────────────────────────────┘
```

- **AZ-locked**: EBS volumes are tied to a specific AZ — to use in another AZ, snapshot + restore
- **One-to-one**: Generally attached to ONE instance at a time (except Multi-Attach on io1/io2)
- **Persistent**: Data survives instance stop/start and termination (unless "Delete on Termination" is enabled)
- **Provisioned**: You choose size (GBs) and IOPS upfront

### EBS Volume Types — Deep Dive

#### SSD-based (for random I/O workloads like databases):

**gp3 (General Purpose SSD v3)** — Recommended default
- Baseline: 3,000 IOPS and 125 MB/s throughput — included in price
- Can scale up to 16,000 IOPS and 1,000 MB/s **independently** (IOPS and throughput are separate knobs)
- Size: 1 GB – 16 TB
- ✅ Use for: Boot volumes, dev/test, small-medium databases

**Create a gp3 EBS volume:**
```bash
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 100 \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125 \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123
```

**gp2 (General Purpose SSD v2)** — Older, still available
- IOPS **linked to size**: 3 IOPS per GB (so 100 GB = 300 IOPS, max 16,000 IOPS at 5,333 GB)
- If you need more IOPS, you must increase volume size (wasteful!)
- ✅ Use for: Same as gp3, but gp3 is better and cheaper

**io2 Block Express / io1 (Provisioned IOPS SSD)** — For demanding workloads
- io2 Block Express: up to **256,000 IOPS**, sub-millisecond latency
- io1: up to 64,000 IOPS (on Nitro instances), 32,000 on others
- IOPS:GB ratio: io1 = 50:1 max; io2 = 500:1 max
- Supports **Multi-Attach** (mount to multiple EC2 in same AZ)
- ✅ Use for: Large databases (Oracle, SQL Server), I/O-intensive apps

**Create an io2 EBS volume (high-performance):**
```bash
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 500 \
  --volume-type io2 \
  --iops 20000 \
  --encrypted
```

#### HDD-based (for sequential I/O workloads like streaming, analytics):

**st1 (Throughput Optimized HDD)**
- Max 500 MB/s throughput, 500 IOPS
- ✅ Use for: Big data (Kafka, Hadoop), data warehouses, log processing, ETL

**sc1 (Cold HDD)** — Cheapest EBS
- Max 250 MB/s, 250 IOPS
- ✅ Use for: Infrequently accessed data, archival, cost-sensitive workloads

> ⚠️ **Critical rule**: Only gp2, gp3, io1, io2 can be used as **BOOT volumes**. HDD types (st1, sc1) cannot be boot volumes.

**Attach EBS volume to instance:**
```bash
aws ec2 attach-volume \
  --volume-id vol-0abc123def456 \
  --instance-id i-1234567890abcdef0 \
  --device /dev/sdf
```

**Detach EBS volume:**
```bash
aws ec2 detach-volume --volume-id vol-0abc123def456
```

**Describe volumes:**
```bash
aws ec2 describe-volumes \
  --filters "Name=status,Values=available"
```

**Modify volume (increase size, change type — no downtime):**
```bash
aws ec2 modify-volume \
  --volume-id vol-0abc123def456 \
  --size 200 \
  --volume-type gp3 \
  --iops 6000
```

**Delete volume (must be detached first):**
```bash
aws ec2 delete-volume --volume-id vol-0abc123def456
```

### EBS Multi-Attach (io1/io2 only)

```
                    ┌──── EC2 Instance A ────┐
                    │    (AZ: us-east-1a)    │
io2 Volume ─────────┤                        │
(us-east-1a)        └──── EC2 Instance B ────┘
                         (AZ: us-east-1a)

⚠️ SAME AZ only! Multi-Attach doesn't work across AZs.
⚠️ Must use cluster-aware file system (not standard ext4 or XFS)
```

Use case: Achieve higher application availability in clustered Linux apps (e.g., Oracle RAC, Teradata)

### EBS Snapshots

A snapshot is a **point-in-time backup** of your EBS volume, stored in S3.

```
Snapshot Workflow:
EBS Volume ──▶ Snapshot (stored in S3, incremental) ──▶ Restore to new EBS in any AZ/Region
                    │
                    ├── Copy to another region (for DR)
                    ├── Share with other AWS accounts
                    └── Create AMI from snapshot
```

**Snapshot Features:**
- **Incremental**: Only changed blocks since last snapshot are saved (efficient!)
- **EBS Snapshot Archive**: Move snapshot to archive tier → 75% cheaper, but 24–72h to restore
- **Recycle Bin**: Protect snapshots from accidental deletion; set retention 1 day–1 year
- **Fast Snapshot Restore (FSR)**: Pay extra to eliminate latency on first use of restored volume

**Create a snapshot:**
```bash
aws ec2 create-snapshot \
  --volume-id vol-0abc123def456 \
  --description "Backup before upgrade"
```

**Create snapshot with tags:**
```bash
aws ec2 create-snapshot \
  --volume-id vol-0abc123def456 \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=db-backup-2026-03-06}]'
```

**List snapshots (owned by me):**
```bash
aws ec2 describe-snapshots \
  --owner-ids self
```

**Copy snapshot to another region (for DR):**
```bash
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-0abc123def456 \
  --destination-region eu-west-1 \
  --description "DR copy"
```

**Restore volume from snapshot:**
```bash
aws ec2 create-volume \
  --snapshot-id snap-0abc123def456 \
  --availability-zone us-east-1a \
  --volume-type gp3
```

**Delete snapshot:**
```bash
aws ec2 delete-snapshot --snapshot-id snap-0abc123def456
```

**Enable snapshot recycle bin (protect from accidental deletion):**
```bash
aws rbin create-rule \
  --retention-period Value=7,Unit=DAYS \
  --resource-type EBS_SNAPSHOT \
  --description "7-day EBS snapshot retention"
```

**Fast Snapshot Restore (FSR) for immediate performance:**
```bash
aws ec2 enable-fast-snapshot-restores \
  --availability-zones us-east-1a us-east-1b \
  --source-snapshot-ids snap-0abc123def456
```

### EBS Encryption

**Enable EBS encryption by default for account/region:**
```bash
aws ec2 enable-ebs-encryption-by-default --region us-east-1
```

**Check if default encryption is enabled:**
```bash
aws ec2 get-ebs-encryption-by-default
```

**Create encrypted copy of unencrypted snapshot:**
```bash
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-UNENCRYPTED \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123
```

> 🔗 EBS docs: https://docs.aws.amazon.com/ebs/latest/userguide/what-is-ebs.html

---

## ⚡ EC2 Instance Store — Blazing Fast Temporary Storage

**What is it?** A physical disk directly attached to the host server — the hardware the EC2 instance runs on. No network involved, so performance is phenomenal.

```
Server Hardware (Physical)
┌───────────────────────────────────────── ─┐
│  CPU    RAM    [NVMe SSD — Instance Store]│
│                    │                      │
│              EC2 Instance                 │
└──────────────────────────────────────── ──┘
```

**The trade-off:**
| Aspect | Instance Store |
|--------|---------------|
| **Performance** | Millions of IOPS — fastest possible |
| **Durability** | ⚠️ **EPHEMERAL** — data lost on stop, termination, or hardware failure |
| **Size** | Fixed (depends on instance type) |
| **Cost** | Included in instance price |

**When to use it:**
- Buffer or cache (you can rebuild the data)
- Temporary scratch space for processing (e.g., Spark intermediate results)
- When you need the absolute highest IOPS (databases with custom replication)

> ⚠️ **Never use Instance Store for anything you can't afford to lose!** Backup to S3 or EBS if you need persistence.

---

## 📂 EFS — Elastic File System

### What Is It?

EFS is a **managed NFS (Network File System)**. Unlike EBS (one-to-one), EFS can be mounted by **hundreds of EC2 instances simultaneously, across multiple AZs**.

```
              ┌──────────────────────────────┐
              │          EFS (NFS)           │
              │  Scales automatically        │
              │  Multi-AZ, highly available  │
              └──────┬──────┬──────┬─────────┘
                     │      │      │
              EC2 A  │   EC2 B  EC2 C
            (AZ-1a)  │  (AZ-1b)  (AZ-1c)
                     │
            All share the same files!
```

### Key Characteristics
- **Linux only** (not Windows) — uses NFS v4 protocol
- **Pay per GB used** (not provisioned) — scales automatically from GB to petabytes
- Highly available (multi-AZ), highly durable (11 nines)
- ~3x more expensive than gp2 EBS
- Uses a Security Group to control access

**Create an EFS file system:**
```bash
aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=my-shared-fs
```

**Create an EFS mount target (one per AZ):**
```bash
aws efs create-mount-target \
  --file-system-id fs-0abc123def456 \
  --subnet-id subnet-0abc123def456 \
  --security-groups sg-0abc123def456
```

**Create a second mount target in another AZ:**
```bash
aws efs create-mount-target \
  --file-system-id fs-0abc123def456 \
  --subnet-id subnet-0def456abc123 \
  --security-groups sg-0abc123def456
```

**Describe file systems:**
```bash
aws efs describe-file-systems
```

**Describe mount targets:**
```bash
aws efs describe-mount-targets \
  --file-system-id fs-0abc123def456
```

**Mount EFS on EC2 (from inside the instance, using NFS):**
```bash
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
  fs-0abc123def456.efs.us-east-1.amazonaws.com:/ /mnt/efs
```

**Mount EFS on EC2 (using EFS mount helper — amazon-efs-utils):**
```bash
sudo mount -t efs -o tls fs-0abc123def456:/ /mnt/efs
```

**Create EFS access point (for container workloads):**
```bash
aws efs create-access-point \
  --file-system-id fs-0abc123def456 \
  --posix-user Uid=1001,Gid=1001 \
  --root-directory Path=/app-data,CreationInfo={OwnerUid=1001,OwnerGid=1001,Permissions=755}
```

**Enable lifecycle management (move to IA after 30 days):**
```bash
aws efs put-lifecycle-configuration \
  --file-system-id fs-0abc123def456 \
  --lifecycle-policies TransitionToIA=AFTER_30_DAYS
```

**Delete EFS file system (delete mount targets first):**
```bash
aws efs delete-mount-target --mount-target-id fsmt-0abc123def456
aws efs delete-file-system --file-system-id fs-0abc123def456
```

### EFS Storage Classes

EFS has tiers, similar to S3:

| Tier | Access | Cost |
|------|--------|------|
| EFS Standard | Frequent access | $$$ |
| EFS Standard-IA | Infrequent access | $ (up to 92% cheaper) |
| EFS One Zone | Frequent, single AZ | $$ |
| EFS One Zone-IA | Infrequent, single AZ | Cheapest |

**EFS Lifecycle Management**: Automatically moves files to IA tier after N days of no access (configurable: 7, 14, 30, 60, 90 days).

### EFS Performance Modes

| Mode | When to Use |
|------|-------------|
| General Purpose (default) | Latency-sensitive: web serving, CMS |
| Max I/O | Scale to higher levels of aggregate throughput: big data, media processing |

### EFS Throughput Modes

| Mode | Description |
|------|-------------|
| Bursting | Throughput scales with file system size (like t-class EC2) |
| Provisioned | Set throughput regardless of storage size |
| Elastic | Auto-scales throughput based on workload (recommended) |

> 🔗 EFS docs: https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html

---

## 🆚 EBS vs EFS vs Instance Store — The Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│              EC2 Storage Decision Guide                                  │
│                                                                          │
│  Need shared storage across instances/AZs?                               │
│       YES ──▶ EFS (NFS, Linux, multi-AZ, auto-scale)                    │
│       NO  ──▶ Continue below                                             │
│                                                                          │
│  Need MAXIMUM performance (millions of IOPS)?                            │
│       YES ──▶ Instance Store (but data is ephemeral!)                   │
│       NO  ──▶ Continue below                                             │
│                                                                          │
│  Need persistent block storage?                                          │
│       Single instance, random I/O (DB, boot) ──▶ gp3 or io2 (EBS)      │
│       Sequential I/O (streaming, ETL)         ──▶ st1 (EBS)             │
│       Rarely accessed archive                 ──▶ sc1 (EBS)             │
└─────────────────────────────────────────────────────────────────────────┘
```

| Feature | EBS | EFS | Instance Store |
|---------|-----|-----|----------------|
| **Attach to** | 1 instance (io1/io2: multi) | Many instances, multi-AZ | 1 instance only |
| **AZ scope** | 1 AZ | Multi-AZ | Physical host |
| **Persistence** | ✅ Persistent | ✅ Persistent | ❌ Ephemeral |
| **Performance** | Good (network) | Good (network) | Best (physical) |
| **Pricing** | Per GB provisioned | Per GB used | Included |
| **OS support** | Linux + Windows | Linux only | Linux + Windows |
| **Use case** | Boot volumes, DBs | Shared web content, home dirs | Cache, temp |

---

## ⭐ Interview Tips & Key Points to Remember

- **EBS is AZ-locked** — snapshot it first if you want to move to another AZ/region
- **Only SSD types (gp2, gp3, io1, io2) can be boot volumes** — HDD (st1, sc1) cannot
- **gp3 > gp2**: gp3 lets you tune IOPS and throughput independently; gp3 is cheaper too
- **io2 Block Express**: up to 256,000 IOPS — for the most demanding databases
- **Multi-Attach (io1/io2 only)**: same AZ, cluster-aware file system required
- **Instance Store = ephemeral** — fastest possible, but data gone on stop/terminate
- **EFS = shared NFS, Linux only, pay per use** (not provisioned), auto-scales
- **EFS ~3x more expensive than EBS gp2** but needed for shared/multi-AZ scenarios
- **EFS-IA** can save up to 92% for infrequently accessed files (automatic tiering)
- **EBS snapshots are incremental** (only changed blocks saved after first snapshot)
- **Snapshot Archive** = 75% cheaper, but takes 24–72h to restore — plan ahead
- Scenario: "need 100 EC2 instances to share files across AZs" → **EFS**, not EBS

---

## Quick Reference — AWS CLI Commands

### EBS Volumes
```bash
# Create gp3 EBS volume
aws ec2 create-volume --availability-zone us-east-1a --size 100 --volume-type gp3 --iops 3000 --throughput 125 --encrypted

# Create io2 EBS volume
aws ec2 create-volume --availability-zone us-east-1a --size 500 --volume-type io2 --iops 20000 --encrypted

# Attach EBS volume
aws ec2 attach-volume --volume-id vol-0abc123def456 --instance-id i-1234567890abcdef0 --device /dev/sdf

# Detach EBS volume
aws ec2 detach-volume --volume-id vol-0abc123def456

# Describe volumes
aws ec2 describe-volumes --filters "Name=status,Values=available"

# Modify volume
aws ec2 modify-volume --volume-id vol-0abc123def456 --size 200 --volume-type gp3 --iops 6000

# Delete volume
aws ec2 delete-volume --volume-id vol-0abc123def456
```

### EBS Snapshots
```bash
# Create snapshot
aws ec2 create-snapshot --volume-id vol-0abc123def456 --description "Backup before upgrade"

# Create snapshot with tags
aws ec2 create-snapshot --volume-id vol-0abc123def456 --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=db-backup}]'

# List snapshots
aws ec2 describe-snapshots --owner-ids self

# Copy snapshot to another region
aws ec2 copy-snapshot --source-region us-east-1 --source-snapshot-id snap-0abc123def456 --destination-region eu-west-1

# Restore volume from snapshot
aws ec2 create-volume --snapshot-id snap-0abc123def456 --availability-zone us-east-1a --volume-type gp3

# Delete snapshot
aws ec2 delete-snapshot --snapshot-id snap-0abc123def456

# Enable snapshot recycle bin
aws rbin create-rule --retention-period Value=7,Unit=DAYS --resource-type EBS_SNAPSHOT

# Enable Fast Snapshot Restore
aws ec2 enable-fast-snapshot-restores --availability-zones us-east-1a us-east-1b --source-snapshot-ids snap-0abc123def456
```

### EBS Encryption
```bash
# Enable default encryption
aws ec2 enable-ebs-encryption-by-default --region us-east-1

# Check default encryption status
aws ec2 get-ebs-encryption-by-default

# Copy and encrypt snapshot
aws ec2 copy-snapshot --source-region us-east-1 --source-snapshot-id snap-UNENCRYPTED --encrypted
```

### EFS File System
```bash
# Create EFS
aws efs create-file-system --performance-mode generalPurpose --throughput-mode bursting --encrypted

# Create mount target
aws efs create-mount-target --file-system-id fs-0abc123def456 --subnet-id subnet-0abc123def456 --security-groups sg-0abc123def456

# Describe file systems
aws efs describe-file-systems

# Describe mount targets
aws efs describe-mount-targets --file-system-id fs-0abc123def456

# Create access point
aws efs create-access-point --file-system-id fs-0abc123def456 --posix-user Uid=1001,Gid=1001

# Enable lifecycle management
aws efs put-lifecycle-configuration --file-system-id fs-0abc123def456 --lifecycle-policies TransitionToIA=AFTER_30_DAYS

# Delete mount target
aws efs delete-mount-target --mount-target-id fsmt-0abc123def456

# Delete EFS
aws efs delete-file-system --file-system-id fs-0abc123def456
```
