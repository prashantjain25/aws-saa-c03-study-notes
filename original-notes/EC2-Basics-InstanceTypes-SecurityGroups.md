# Amazon EC2 – Basics
> 📚 Official Docs: https://docs.aws.amazon.com/ec2/  
> 🎯 SAA-C03 Exam Weight: Very High — most referenced service in the exam

---

## 🤔 What is EC2 and Why Does It Matter?

Before cloud, if your startup suddenly went viral, your physical servers would crash and you'd be offline for days waiting for new hardware to arrive. EC2 (Elastic Compute Cloud) solves this — it lets you rent virtual servers in the cloud and scale up in minutes.

**EC2 = IaaS (Infrastructure as a Service)** — You manage the OS and everything above; AWS manages the physical hardware underneath.

Think of EC2 like renting an apartment. AWS owns the building (hardware). You rent a unit (virtual machine), decide what furniture goes inside (OS, software), and control who has a key (Security Groups). You can upgrade to a bigger apartment (vertical scaling) or rent more apartments (horizontal scaling) any time.

> 🔗 EC2 Overview: https://aws.amazon.com/ec2/

---

## ⚙️ What Can You Configure on an EC2 Instance?

When you launch an EC2 instance, you choose:

| Component | Options |
|-----------|---------|
| **Operating System** | Amazon Linux, Ubuntu, Windows, macOS, Red Hat, SUSE |
| **CPU & Memory (RAM)** | Determined by instance type (e.g., t3.medium = 2 vCPU, 4 GB RAM) |
| **Storage** | Network-attached (EBS/EFS) or local hardware (Instance Store) |
| **Network** | VPC, subnet, public/private IP, network card speed |
| **Firewall** | Security Groups |
| **Bootstrap Script** | EC2 User Data — commands that run ONCE on first launch |

### EC2 User Data — What Is It?

User Data is a **bootstrap script** — a set of commands that AWS runs automatically the first time your instance starts. This is perfect for:
- Installing software (e.g., `yum install -y httpd`)
- Pulling your app code from S3
- Starting services at launch

```
Instance First Boot Flow:
┌──────────────────────────────────────────────────────┐
│  1. AWS launches EC2 instance                        │
│  2. EC2 reads User Data script                       │
│  3. Runs script AS ROOT (once only on first start)   │
│  4. Your app is ready! ✅                            │
└──────────────────────────────────────────────────────┘
```

> ⚠️ **Common misconception**: User Data runs ONLY on the FIRST launch, not on every restart. If you want it to run every reboot, you need to use a different mechanism (like rc.local or a cron job inside the script).

---

## 🖥️ EC2 Instance Types — Choosing the Right "Size"

AWS has hundreds of instance types. Rather than memorizing them all, understand the **families** and their use cases.

### Instance Naming Convention

```
        m  5  .  2xlarge
        │  │     │
        │  │     └── Size within family (nano, micro, small, medium, large, xlarge, 2xlarge...)
        │  └──────── Generation (higher = newer hardware, better price/performance)
        └─────────── Family (defines the hardware optimisation)
```

### Instance Families

| Family | Letter(s) | Optimized For | Real-World Use Cases |
|--------|-----------|---------------|---------------------|
| **General Purpose** | t, m | Balanced CPU/Memory/Network | Web servers, code repos, small DBs |
| **Compute Optimized** | c | High-performance CPU | Batch processing, HPC, ML inference, gaming servers |
| **Memory Optimized** | r, x, z | Large RAM | In-memory DBs (Redis), real-time analytics, SAP HANA |
| **Storage Optimized** | i, d, h | High IOPS local storage | OLTP databases, Cassandra, HDFS, data warehousing |
| **Accelerated Computing** | p, g, trn | GPU / custom chips | Deep learning training, video rendering |

> 💡 **t2.micro** is the free tier instance — 1 vCPU, 1 GB RAM, good for testing

### T-Series Burstable Performance — How Burst Credits Work

The `t` family (t2, t3, t4g) uses a **credit system**:
- Instance earns CPU credits over time when CPU is idle
- When you need more CPU, credits are "spent"
- When credits run out, CPU is throttled back to baseline

```
CPU Credit Model:
Idle time   → Credits accumulate  📈
Spike load  → Credits depleted    📉
No credits  → Throttled to baseline (e.g., 10% CPU for t2.micro)
```

> 💡 **T Unlimited** mode: allows bursting beyond credit balance but you pay for the extra CPU

> 🔗 Instance type comparison: https://instances.vantage.sh/

---

## 🔥 EC2 Purchasing Options — Optimize Your Cost

This is heavily tested. Understand the trade-off between commitment level and discount.

```
Cost (high → low)         Flexibility (high → low)
────────────────────────────────────────────────────
On-Demand    $$$$          ████████████████  Max
Spot         $             ████████          Moderate (can be terminated!)
Reserved     $$            ████              Low (1 or 3 year commit)
Savings Plan $$            █████             Moderate ($ commit, flexible type)
Dedicated    $$$$$         ██                Minimal (physical isolation)
```

### On-Demand Instances
- **Pay per second** (Linux/Windows) or per hour (other OS)
- No upfront cost, no commitment
- Highest cost per hour
- **Best for**: short-term, unpredictable workloads, testing, first deployments

**Launch an On-Demand EC2 instance:**

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name my-key-pair \
  --security-group-ids sg-0abc123def456 \
  --subnet-id subnet-0abc123def456 \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyServer}]'
```

**Describe running instances:**

```bash
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running"
```

**Stop / Start / Terminate instances:**

```bash
# Stop instance (you still pay for EBS storage)
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Start a stopped instance
aws ec2 start-instances --instance-ids i-1234567890abcdef0

# Terminate instance (permanent deletion, stop paying)
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0
```

### Reserved Instances (RIs)
- 1 or 3 year commitment → **up to 72% discount** vs On-Demand
- Payment options: All Upfront (max discount), Partial Upfront, No Upfront
- **Standard RI**: locked to specific instance type/region
- **Convertible RI**: can change instance family/OS/tenancy — less discount but flexible
- **Best for**: steady-state workloads (e.g., production database that runs 24/7)

### Savings Plans
- Commit to a **$ amount per hour** for 1 or 3 years → up to 72% discount
- More flexible than Reserved — applies across instance families, sizes, OS
- Compute Savings Plan: covers EC2, Lambda, Fargate
- EC2 Instance Savings Plan: more discount but locked to one instance family in one region
- **Best for**: consistent compute usage but uncertain about future instance needs

### Spot Instances
- Use AWS's **spare capacity** → up to **90% cheaper** than On-Demand
- The catch: **AWS can reclaim the instance with 2-minute warning** if they need the capacity back
- You set a max price; if spot price exceeds it, instance is terminated
- **Best for**: fault-tolerant batch jobs, data analysis, image rendering, CI/CD
- **Not for**: databases, critical workloads, anything that can't tolerate interruption

```
Spot Instance Termination Flow:
┌────────────────────────────────────────────────────────┐
│  Spot price rises above your max price                 │
│                ↓                                       │
│  AWS sends 2-minute interruption notice                │
│                ↓                                       │
│  Your instance is terminated (or hibernated/stopped)   │
│                ↓                                       │
│  Design your app to handle this gracefully!            │
└────────────────────────────────────────────────────────┘
```

### Dedicated Hosts
- **You get the entire physical server** — you can see socket, core counts
- Allows you to use existing per-socket, per-core licenses (BYOL — Bring Your Own License)
- Use case: Compliance requirements, regulatory needs (e.g., "no shared hardware")
- Most expensive option
- Can be purchased On-Demand or Reserved

### Dedicated Instances
- Your instance runs on hardware dedicated to you (not shared with other AWS customers)
- BUT — you don't get visibility into the physical server
- May share hardware with other instances in your own account
- Less expensive than Dedicated Hosts

### Capacity Reservations
- Reserve capacity in a **specific AZ** for any duration
- No billing discount — you pay On-Demand rate whether you use it or not
- **Best for**: critical events where you MUST have capacity available (e.g., Black Friday)

> 🔗 Pricing comparison: https://aws.amazon.com/ec2/pricing/

---

## 🛡️ Security Groups — EC2's Firewall

Security Groups are **stateful virtual firewalls** that control traffic to/from your EC2 instances.

**"Stateful" means**: If you allow inbound HTTP traffic, the response traffic is automatically allowed out — you don't need a separate outbound rule.

### How Security Groups Work

```
Internet
   │
   │  Port 80 (HTTP) ──── Inbound rule allows ──▶  EC2 Instance
   │                                                     │
   │◀─── Response auto-allowed (stateful) ──────────────┘
   │
   │  Port 22 (SSH) ──── No inbound rule ──▶  BLOCKED ❌
```

### Key Rules to Know

- Security Groups contain **ALLOW rules only** — you can't write DENY rules in SGs (use NACLs for DENY)
- **Default inbound**: all traffic BLOCKED (no rules = blocked)
- **Default outbound**: all traffic ALLOWED
- Can reference **IP addresses OR other Security Groups** as source/destination
- One SG can be attached to multiple instances; one instance can have multiple SGs
- **Locked to a region + VPC** — can't use an SG from us-east-1 in eu-west-1

**Create a security group:**

```bash
aws ec2 create-security-group \
  --group-name my-app-sg \
  --description "Security group for web app" \
  --vpc-id vpc-1234567890abcdef0
```

**Allow inbound HTTP (port 80) from anywhere:**

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123def456 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0  # Allow from any IP address
```

**Allow inbound HTTPS (port 443):**

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123def456 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

**Allow inbound SSH (port 22) from specific IP:**

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123def456 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.10/32  # Allow only from this IP
```

### Referencing Security Groups (Power Feature!)

```
[Load Balancer SG]───allows──▶[EC2 App SG]───allows──▶[RDS DB SG]

Instead of hardcoding IPs:
  - EC2's SG allows inbound from "Load Balancer SG" (not an IP)
  - RDS's SG allows inbound from "EC2 App SG" (not an IP)
  
Now when you add/remove EC2 instances, security automatically follows!
```

**Allow ingress from another security group (SG-to-SG rule):**

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-backend123 \
  --protocol tcp \
  --port 3306 \
  --source-group sg-frontend456  # Allow from another SG
```

**Revoke an inbound rule:**

```bash
aws ec2 revoke-security-group-ingress \
  --group-id sg-0abc123def456 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.10/32
```

**Describe security groups:**

```bash
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=my-app-sg"
```

**Delete security group:**

```bash
aws ec2 delete-security-group --group-id sg-0abc123def456
```

### Must-Know Ports

| Port | Protocol | Used For |
|------|----------|----------|
| 22 | SSH | Secure shell into Linux instances |
| 21 | FTP | File Transfer Protocol |
| 22 | SFTP | Secure FTP (via SSH) |
| 80 | HTTP | Unsecured web traffic |
| 443 | HTTPS | Secured web traffic (TLS) |
| 3389 | RDP | Remote Desktop (Windows instances) |
| 3306 | MySQL/Aurora | Database connections |
| 5432 | PostgreSQL | Database connections |

> 🔗 Security Groups docs: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html

---

## 🔑 Key Pairs & SSH Access

**Create a key pair (for SSH access):**

```bash
aws ec2 create-key-pair \
  --key-name my-key-pair \
  --query 'KeyMaterial' \
  --output text > my-key-pair.pem

# Change permissions so only you can read it
chmod 400 my-key-pair.pem
```

---

## 💾 Elastic IP — Static Public IP Address

An **Elastic IP** is a static public IPv4 address that persists even after you stop/start an instance.

**Allocate an Elastic IP:**

```bash
aws ec2 allocate-address --domain vpc
# Returns AllocationId (e.g., eipalloc-0abc123def456)
```

**Associate Elastic IP with an instance:**

```bash
aws ec2 associate-address \
  --instance-id i-1234567890abcdef0 \
  --allocation-id eipalloc-0abc123def456
```

---

## 🔍 EC2 Instance Metadata

**From inside an EC2 instance** (IMDSv2 is more secure):

```bash
# Get temporary token (valid for 6 hours)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Use token to retrieve metadata
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

---

## 🔑 SSH, EC2 Instance Connect & SSM Session Manager

**Three ways to connect to your EC2 instance:**

```
Method 1: SSH (Traditional)
  Your laptop ──(port 22, SSH key)──▶ EC2 public IP
  Requires: key pair (.pem file), open SG port 22

Method 2: EC2 Instance Connect (Browser-based SSH)
  AWS Console ──(temporary SSH key, port 22)──▶ EC2
  Requires: open SG port 22, Amazon Linux 2 or Ubuntu

Method 3: SSM Session Manager (Most Secure — NO port 22 needed!)
  AWS Console ──(SSM Agent on EC2, HTTPS outbound only)──▶ EC2
  Requires: IAM role with SSM permissions, SSM Agent installed
  Best practice: no open ports at all!
```

> 💡 **Interview tip**: If asked "how do you connect to EC2 without opening any ports?" → **SSM Session Manager**

---

## ⭐ Interview Tips & Key Points to Remember

- **EC2 = IaaS** — you manage OS and up; AWS manages physical hardware
- **User Data runs ONCE only**, at first launch, as root user — not on reboot
- **Security Groups are STATEFUL** — inbound allow automatically allows outbound response
- **SGs have ALLOW rules only** — for DENY rules, use NACLs (different concept)
- **Security Group can reference another SG** — not just IP ranges (very powerful)
- **Spot instances can be reclaimed with 2-minute warning** — not for critical workloads
- **Reserved vs Savings Plan**: Reserved = specific instance type/AZ; Savings Plan = $ commitment, more flexible
- **Dedicated Host vs Dedicated Instance**: Host = full physical server visibility (BYOL, compliance); Instance = isolated hardware, no visibility
- **t-family = burstable** — earns/spends CPU credits; throttled when credits exhausted
- **Best cost for long-running steady workload**: Reserved (1 or 3 year) — up to 72% off
- **Best cost for fault-tolerant batch jobs**: Spot — up to 90% off
- **SSM Session Manager = no open ports needed** — best practice for secure EC2 access
- **On-Demand is billed per second** (Linux/Windows) — not per hour (common misconception)

---

## Quick Reference — AWS CLI Commands

### Instance Management
```bash
# Launch an EC2 instance
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name my-key-pair \
  --security-group-ids sg-0abc123def456 \
  --subnet-id subnet-0abc123def456 \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyServer}]'

# Describe instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running"

# Stop instance (you still pay for EBS)
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Start a stopped instance
aws ec2 start-instances --instance-ids i-1234567890abcdef0

# Terminate instance (permanent deletion)
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0
```

### Key Pairs
```bash
# Create a key pair
aws ec2 create-key-pair \
  --key-name my-key-pair \
  --query 'KeyMaterial' \
  --output text > my-key-pair.pem
```

### Security Groups
```bash
# Create a security group
aws ec2 create-security-group \
  --group-name my-app-sg \
  --description "Security group for web app" \
  --vpc-id vpc-1234567890abcdef0

# Allow inbound HTTP (port 80) from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123def456 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow inbound HTTPS (port 443)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123def456 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Allow inbound SSH (port 22) from specific IP
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123def456 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.10/32

# Allow ingress from another security group (SG-to-SG)
aws ec2 authorize-security-group-ingress \
  --group-id sg-backend123 \
  --protocol tcp \
  --port 3306 \
  --source-group sg-frontend456

# Revoke an inbound rule
aws ec2 revoke-security-group-ingress \
  --group-id sg-0abc123def456 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.10/32

# Describe security groups
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=my-app-sg"

# Delete security group
aws ec2 delete-security-group --group-id sg-0abc123def456
```

### Elastic IP
```bash
# Allocate an Elastic IP
aws ec2 allocate-address --domain vpc

# Associate Elastic IP with instance
aws ec2 associate-address \
  --instance-id i-1234567890abcdef0 \
  --allocation-id eipalloc-0abc123def456
```

### Instance Metadata (from inside EC2)
```bash
# Get IMDSv2 token
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Retrieve instance ID
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

