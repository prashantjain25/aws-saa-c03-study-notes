# Amazon EC2 – Associate Level Concepts
> 📚 Official Docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/  
> 🎯 These topics separate associate-level understanding from beginner

---

## 📦 EC2 Placement Groups — Control Where Instances Live

By default, AWS places your instances wherever it finds capacity. But for certain workloads, you want control. Placement Groups let you influence how EC2 instances are physically placed on AWS hardware.

There are **3 types**, each optimized for a different goal:

### 1. 🔗 Cluster Placement Group — "All Together, Super Fast"

```
┌────────────────── Single AZ ──────────────────────┐
│  ┌──────────────── Same Rack ──────────────────┐  │
│  │  [EC2] ──10Gbps──[EC2] ──10Gbps──[EC2]      │  │
│  │  [EC2] ──10Gbps──[EC2] ──10Gbps──[EC2]      │  │
│  └─────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────┘
```

- All instances on the **same rack in the same AZ**
- Ultra-low network latency + high bandwidth (up to 10 Gbps between instances)
- **Risk**: If the rack fails, ALL instances fail simultaneously
- **Use cases**: HPC (High Performance Computing), big data jobs that need low-latency inter-node communication, scientific simulations

> 💡 Analogy: A cluster group is like putting all your servers in a single room — lightning fast communication between them, but if the room catches fire, everything's gone.

**Create a Cluster placement group:**
```bash
aws ec2 create-placement-group \
  --group-name my-cluster-pg \
  --strategy cluster
```

### 2. 📊 Spread Placement Group — "Isolated for Safety"

```
┌── AZ-1 ──┐  ┌── AZ-1 ──┐  ┌── AZ-2 ──┐  ┌── AZ-3 ──┐
│  Rack A  │  │  Rack B  │  │  Rack C  │  │  Rack D  │
│  [EC2-1] │  │  [EC2-2] │  │  [EC2-3] │  │  [EC2-4] │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
     └──different physical hardware──┘
```

- Each instance is on a **different physical rack** (different hardware, power, network)
- Maximum **7 instances per AZ** per placement group (hard limit!)
- Can span multiple AZs
- **Risk** if one rack fails: only 1 instance affected — others safe
- **Use cases**: Critical apps where you can't afford multiple failures (e.g., 3-node Zookeeper cluster, primary-secondary databases)

> ⚠️ **The 7-instance limit per AZ is a very common exam question!**

**Create a Spread placement group:**
```bash
aws ec2 create-placement-group \
  --group-name my-spread-pg \
  --strategy spread
```

### 3. 🗂️ Partition Placement Group — "Distributed Systems"

```
┌──────────────────── AZ (e.g., us-east-1a) ────────────────────┐
│                                                               │
│  Partition 1     Partition 2     Partition 3                  │
│  (Rack set A)    (Rack set B)    (Rack set C)                 │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                    │
│  │ EC2     │    │ EC2     │    │ EC2     │                    │
│  │ EC2     │    │ EC2     │    │ EC2     │                    │
│  │ EC2     │    │ EC2     │    │ EC2     │                    │
│  └─────────┘    └─────────┘    └─────────┘                    │
│  partitions don't share racks with each other                 │
└───────────────────────────────────────────────────────────────┘
```

- Up to **7 partitions per AZ**, each partition = its own set of racks
- **100s of EC2 instances** can be in one group
- If partition 1 fails, partitions 2 and 3 are unaffected
- EC2 instances know which partition they're in (via metadata API) — useful for rack-aware distributed systems
- **Use cases**: HDFS, HBase, Cassandra, Kafka — distributed systems that need rack awareness

**Create a Partition placement group:**
```bash
aws ec2 create-placement-group \
  --group-name my-partition-pg \
  --strategy partition \
  --partition-count 3
```

### Quick Comparison

| | Cluster | Spread | Partition |
|--|---------|--------|-----------|
| Goal | Performance | Safety | Distributed scale |
| Instances per AZ | No limit | **Max 7** | 100s |
| Spans AZs? | No (single AZ) | Yes | Yes |
| Failure impact | All at once | 1 at a time | 1 partition |
| Use case | HPC, low latency | Critical apps | Hadoop, Cassandra |

**Launch instance into a Cluster placement group:**
```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type c5n.18xlarge \
  --placement '{"GroupName": "my-cluster-pg"}' \
  --count 1
```

**Describe placement groups:**
```bash
aws ec2 describe-placement-groups
```

**Delete placement group (must be empty):**
```bash
aws ec2 delete-placement-group --group-name my-cluster-pg
```

---

## 🌐 Elastic Network Interfaces (ENI) — Virtual Network Cards

An ENI is a **virtual network card** you can attach to EC2 instances. When you launch an EC2 instance, it automatically gets a primary ENI. But you can create additional ENIs and manage them independently.

### What's Inside an ENI?

```
┌─────────────────── ENI (eth0) ──────────────────────┐
│                                                     │
│  Primary Private IPv4:    10.0.1.15                 │
│  Secondary Private IPv4:  10.0.1.16 (optional)      │
│  Elastic IP (public):     54.23.101.5 (optional)    │
│  Public IPv4:             3.92.155.47 (optional)    │
│  Security Groups:         [sg-web, sg-db]           │
│  MAC Address:             0a:12:34:56:78:9a         │
└─────────────────────────────────────────────────────┘
```

### Key Properties
- ENIs are **AZ-bound** — can't move an ENI to a different AZ
- Can be attached/detached from EC2 instances on-the-fly
- Moving an ENI carries its private IP, Elastic IP, and MAC address to the new instance

**Create an ENI in a specific subnet:**
```bash
aws ec2 create-network-interface \
  --subnet-id subnet-0abc123def456 \
  --description "Secondary ENI for app" \
  --groups sg-0abc123def456
```

**Attach ENI to EC2 instance (hot attach):**
```bash
aws ec2 attach-network-interface \
  --network-interface-id eni-0abc123def456 \
  --instance-id i-1234567890abcdef0 \
  --device-index 1
```

**Detach ENI from instance:**
```bash
aws ec2 detach-network-interface \
  --attachment-id eni-attach-0abc123def456
```

**Describe ENIs:**
```bash
aws ec2 describe-network-interfaces \
  --filters "Name=subnet-id,Values=subnet-0abc123def456"
```

### The Failover Use Case

```
BEFORE FAILURE:
EC2-Primary ← [ENI with 10.0.1.15] ← Application connects to 10.0.1.15

PRIMARY FAILS:
EC2-Primary ✗ (crashed)
                ↓
Detach ENI from EC2-Primary

AFTER FAILOVER:
EC2-Standby ← [Same ENI with 10.0.1.15] ← Application still connects to 10.0.1.15
```

The application doesn't notice — it's still talking to the same private IP! This is a cheap, simple failover mechanism.

**Move ENI to another instance (detach + reattach):**
```bash
# 1. Detach from failed instance
aws ec2 detach-network-interface --attachment-id eni-attach-OLD

# 2. Attach to new instance (carries same private IP, Elastic IP)
aws ec2 attach-network-interface \
  --network-interface-id eni-0abc123def456 \
  --instance-id i-NEW-INSTANCE-ID \
  --device-index 1
```

> 🔗 ENI docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html

---

## 😴 EC2 Hibernate — Pause and Resume Like a Laptop

**Normal stop vs hibernate:**

```
Normal STOP:
  RAM contents ──── LOST ────
  EBS volume ─── Preserved
  Next boot: OS starts fresh (slow)

HIBERNATE:
  RAM contents ──── Dumped to encrypted EBS root volume ────▶ Preserved!
  Next start: RAM restored from EBS, OS resumes instantly (fast!)
```

Think of it like closing your laptop lid vs turning it off completely.

### When to Use Hibernate?
- Long-running processes you don't want to restart
- Applications with slow initialization (loading ML models, caches, etc.)
- Services that need fast resume times

### Hibernate Requirements (Constraints for Exam)
| Constraint | Value |
|------------|-------|
| **Max RAM size** | 150 GB |
| **Max hibernate duration** | 60 days |
| **Root volume type** | Must be EBS (not Instance Store) |
| **Root volume encryption** | Must be ENABLED |
| **Supported purchase types** | On-Demand, Reserved, Spot |
| **Instance bare metal** | NOT supported |

> ⚠️ Hibernate must be enabled at LAUNCH time — you can't enable it after the instance is already running!

**Launch a hibernate-capable instance:**
```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type m5.large \
  --hibernation-options Configured=true \
  --block-device-mappings '[{
    "DeviceName": "/dev/xvda",
    "Ebs": {
      "VolumeSize": 30,
      "Encrypted": true
    }
  }]'
```

**Hibernate a running instance:**
```bash
aws ec2 stop-instances \
  --instance-ids i-1234567890abcdef0 \
  --hibernate
```

**Resume from hibernate (same as start-instances):**
```bash
aws ec2 start-instances --instance-ids i-1234567890abcdef0
```

---

## 🔍 EC2 Instance Metadata Service (IMDS)

**What is IMDS?**
It's a special HTTP endpoint (`169.254.169.254`) that every EC2 instance can call to learn about itself — its instance ID, public IP, IAM role credentials, security groups, AZ, etc.

```
Inside EC2:
curl http://169.254.169.254/latest/meta-data/instance-id
→ Returns: i-0123456789abcdef0

curl http://169.254.169.254/latest/meta-data/iam/security-credentials/my-role
→ Returns: temporary credentials (AccessKeyId, SecretAccessKey, Token)
```

**Why does this matter?** Your application code can fetch its own IAM credentials from IMDS without hardcoding them. AWS SDKs do this automatically.

### IMDSv1 vs IMDSv2 — Security Matters

| | IMDSv1 | IMDSv2 |
|-|--------|--------|
| How it works | Simple GET request | Session-oriented (PUT to get token first, then GET) |
| SSRF vulnerability | Vulnerable | Protected |
| Recommendation | Deprecated — avoid | **Use this** |

**SSRF (Server-Side Request Forgery)**: An attacker tricks your app into making HTTP requests to `169.254.169.254` and stealing IAM credentials. IMDSv2 prevents this because it requires a session token.

**Describe instance status (checks):**
```bash
aws ec2 describe-instance-status \
  --instance-ids i-1234567890abcdef0
```

**Get console output (for debugging boot issues):**
```bash
aws ec2 get-console-output \
  --instance-id i-1234567890abcdef0 \
  --latest
```

**Reboot instance:**
```bash
aws ec2 reboot-instances --instance-ids i-1234567890abcdef0
```

> 🔗 IMDS docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html

---

## ⭐ Interview Tips & Key Points to Remember

- **Cluster = performance (same rack)** → low latency, high bandwidth, high failure risk
- **Spread = availability (different racks)** → max **7 instances per AZ** (always remember this!)
- **Partition = distributed scale** → rack-aware systems (HDFS, Cassandra, Kafka)
- **ENI is AZ-bound** — can't cross AZs; commonly used for IP failover
- **Hibernate requires encrypted EBS root volume** — must be enabled at launch
- **Max hibernate: 150 GB RAM, 60 days** — know these numbers
- **IMDS endpoint: 169.254.169.254** — link-local address, only accessible from within the instance
- **IMDSv2 is more secure** — protects against SSRF attacks; prefer v2 over v1
- **Moving an ENI moves its private IP, Elastic IP, and MAC address** — application sees no change

---

## Quick Reference — AWS CLI Commands

### Placement Groups
```bash
# Create a Cluster placement group
aws ec2 create-placement-group --group-name my-cluster-pg --strategy cluster

# Create a Spread placement group
aws ec2 create-placement-group --group-name my-spread-pg --strategy spread

# Create a Partition placement group
aws ec2 create-placement-group --group-name my-partition-pg --strategy partition --partition-count 3

# Launch instance into a placement group
aws ec2 run-instances --image-id ami-0abcdef1234567890 --instance-type c5n.18xlarge --placement '{"GroupName": "my-cluster-pg"}' --count 1

# Describe placement groups
aws ec2 describe-placement-groups

# Delete placement group
aws ec2 delete-placement-group --group-name my-cluster-pg
```

### Elastic Network Interfaces
```bash
# Create an ENI
aws ec2 create-network-interface --subnet-id subnet-0abc123def456 --description "Secondary ENI" --groups sg-0abc123def456

# Attach ENI to instance
aws ec2 attach-network-interface --network-interface-id eni-0abc123def456 --instance-id i-1234567890abcdef0 --device-index 1

# Detach ENI
aws ec2 detach-network-interface --attachment-id eni-attach-0abc123def456

# Describe ENIs
aws ec2 describe-network-interfaces --filters "Name=subnet-id,Values=subnet-0abc123def456"
```

### EC2 Hibernation
```bash
# Launch instance with hibernation
aws ec2 run-instances --image-id ami-0abcdef1234567890 --instance-type m5.large --hibernation-options Configured=true

# Hibernation stop
aws ec2 stop-instances --instance-ids i-1234567890abcdef0 --hibernate

# Resume from hibernation
aws ec2 start-instances --instance-ids i-1234567890abcdef0
```

### EC2 Instance Status & Metadata
```bash
# Describe instance status
aws ec2 describe-instance-status --instance-ids i-1234567890abcdef0

# Get console output
aws ec2 get-console-output --instance-id i-1234567890abcdef0 --latest

# Reboot instance
aws ec2 reboot-instances --instance-ids i-1234567890abcdef0
```
