# AWS Solutions Architect Associate (SAA-C03) — Classic Solutions Architecture
## Comprehensive Study Guide: Building Scalable, Resilient Applications

**Last Updated**: March 2026 | **Target Audience**: SAA-C03 Exam Candidates  
**Study Style**: Tutor-led (emphasizing WHY, not just WHAT) with real-world analogies

---

## Official AWS Documentation References

- **Elastic Load Balancing & Sticky Sessions**: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/sticky-sessions.html
- **ElastiCache Deep Dive**: https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/WhatIs.html
- **EFS (Elastic File System)**: https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html
- **AMI Fundamentals**: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html

---

## Table of Contents

1. [Section 1: WhatIsTheTime.com — Stateless Web App Evolution](#section-1)
2. [Section 2: MyClothes.com — Stateful Web App (Shopping Cart)](#section-2)
3. [Section 3: MyWordPress.com — Stateful + Media Storage](#section-3)
4. [Section 4: Instantiating Applications Quickly](#section-4)
5. [Key Exam Tips & Interview Points](#exam-tips)

---

<a name="section-1"></a>

# SECTION 1: WhatIsTheTime.com — Stateless Web App Evolution

## The Problem Statement

Imagine you're building **WhatIsTheTime.com** — a simple website that displays the current time. On day one, maybe 10 users visit. On day 30, you're viral and have 1 million users hitting your site simultaneously.

**The Challenge**: How do you scale from handling 10 users to 1 million users without:
- Downtime
- Your infrastructure costing a fortune
- Single points of failure
- Users experiencing errors

This is the journey we take through this section — each step represents a real architectural decision, and each step solves a specific problem while introducing new ones.

---

## Step 1: Single Instance (The Starting Point)

### Architecture
```
                    Internet Users
                           |
                        (DNS)
                           |
                    ┌──────▼──────┐
                    │  EC2 Instance│
                    │  10.0.1.5    │
                    │              │
                    │ Running App  │
                    └──────────────┘
```

### How It Works
- User connects to instance via public IP
- Application processes requests
- Simple, fast, cheap (~$0.01/hour)

### Problems
- **Single Point of Failure**: If instance crashes or network is down, entire app is down
- **No Scaling**: If traffic spikes, the instance might run out of CPU/RAM
- **No High Availability**: No redundancy across AZs

---

## Step 2: Add Vertical Scaling (Bigger Instance)

```
                        Internet
                           |
                    ┌──────▼──────────┐
                    │ EC2 (c5.2xlarge) │
                    │ (More CPU/RAM)   │
                    │                  │
                    │  Running App     │
                    └──────────────────┘
```

### What Changed
- Upgraded from `t2.micro` → `c5.2xlarge`
- Can handle more concurrent users
- More expensive

### Problems Still Exist
- **Still a SPOF**: One big machine can still crash
- **Expensive ceiling**: Can't upgrade forever (max instance size)
- **No true redundancy**: All eggs in one basket

---

## Step 3: Add Horizontal Scaling (Multiple Instances + Load Balancer)

```
         ┌─────────────────────────────────────────┐
         │                 ALB                     │
         │  (Distributes traffic across instances) │
         └──────────┬──────────────────────────────┘
                    │ (3 backends)
        ┌───────────┼───────────┐
        │           │           │
    ┌───▼───┐   ┌───▼───┐   ┌───▼───┐
    │ EC2-1 │   │ EC2-2 │   │ EC2-3 │
    │(10.0) │   │(10.1) │   │(10.2) │
    └───────┘   └───────┘   └───────┘
    (us-east-1a) (us-east-1b) (us-east-1c)
```

### How It Works
- **Application Load Balancer (ALB)**: Listens on port 80/443, sends requests to healthy instances
- Health checks every 30 seconds
- If instance fails, ALB stops sending traffic to it
- New requests go to healthy instances

### Benefits
- **Resilience**: If EC2-1 crashes, traffic flows to EC2-2 and EC2-3
- **Scalability**: Easy to add more instances as traffic grows
- **Multi-AZ**: Instances in different AZs = resilience to datacenter failures

**Create ALB and enable sticky sessions (legacy pattern):**
```bash
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/50dc6c495c0c9188 \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=86400
```

### Problems Introduced
- **Session Loss (Stateful Apps)**: If user uploads a file to EC2-1, but next request goes to EC2-2, EC2-2 doesn't have the file
- **Manual Scaling**: Still have to manually add instances when traffic spikes
- **EBS Limitation**: Each instance has its own EBS volume, no sharing

---

## Step 4: Add Auto Scaling Group (Auto-Responders)

```
┌─────────────────────────────────────────────────────────┐
│           Auto Scaling Group (min=2, max=10)             │
│                                                          │
│   ┌──────────────────────────────────────────┐           │
│   │ Auto Scaling Policy:                      │           │
│   │ - If CPU > 70% → add 1 instance          │           │
│   │ - If CPU < 30% → remove 1 instance       │           │
│   │ - Min 2 running, Max 10 allowed          │           │
│   └──────────────────────────────────────────┘           │
│                                                          │
│        ┌────────┐      ┌────────┐                        │
│        │ EC2 #1 │      │ EC2 #2 │                        │
│        │ us-1a  │      │ us-1b  │                        │
│        └────────┘      └────────┘                        │
│        (scales up/down automatically)                    │
└─────────────────────────────────────────────────────────┘
```

### How It Works
- **Launch Template**: Defines how new instances should be created (AMI, instance type, security groups, etc.)
- **Desired Capacity**: How many instances to run (usually = current load)
- **Scaling Policy**: "If metric X happens, adjust desired capacity"
- **Health Checks**: ASG replaces unhealthy instances automatically

### Benefits
- **Cost Efficient**: Run fewer instances during low traffic, many during spikes
- **Automatic Recovery**: If instance crashes, ASG launches a replacement
- **Flexibility**: Quickly adjust capacity to match demand

---

<a name="section-2"></a>

# SECTION 2: MyClothes.com — Stateful Web App (Shopping Cart)

## The New Problem: Session State

**WhatIsTheTime.com** is stateless — every request is independent. But **MyClothes.com** is different:

```
User Flow:
1. Browse products
2. Add item to shopping cart (state!)
3. Keep shopping, cart persists
4. Checkout
```

### The Problem: Load Balancing With State

```
SCENARIO: User adds item to cart, request goes to EC2-1

Request 1 (goes to EC2-1):
  User: "Add shirt to cart"
  EC2-1 RAM: cart = {shirt}

Request 2 (ALB decides to send to EC2-2 for load balancing):
  User: "Show my cart"
  EC2-2 RAM: cart = {} (EMPTY! EC2-1's cart not visible here)

User sees empty cart → BUG!
```

### Naive Solution 1: Sticky Sessions (Bad Idea)

```
ALB with "Sticky Sessions" enabled:
  Same user always routed to same EC2 instance
  
User connects to EC2-1 ──────────────────────
                              ↓
                   ALL requests to EC2-1
                   (even if EC2-2 is idle)

Problems:
  - If EC2-1 crashes, user loses cart (session in RAM)
  - Uneven load distribution (some instances loaded, some idle)
  - Scaling becomes difficult
```

**Enable sticky sessions on target group:**
```bash
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/50dc6c495c0c9188 \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=86400
```

### Better Solution: ElastiCache (Session Store)

```
┌─────────────────────────────────────────────┐
│           ALB (sticky sessions OFF)         │
└──────────┬──────────────────────────────────┘
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
  EC2-1  EC2-2  EC2-3  (stateless, no local cart)
    │      │      │
    └──────┼──────┘
           │ (all write/read session here)
           ▼
   ┌──────────────────┐
   │   ElastiCache    │
   │   Redis Cluster  │
   │                  │
   │ cart-user123 =   │
   │ {shirt, pants}   │
   │                  │
   │ cart-user456 =   │
   │ {socks}          │
   └──────────────────┘
```

### How It Works
1. User login → app stores session ID in browser cookie
2. User adds to cart → EC2-1 updates ElastiCache
3. EC2-2 gets next request → queries ElastiCache (finds cart)
4. Cart visible on EC2-2, EC2-3, etc.

### Benefits
- **Resilient**: ElastiCache replicas in other AZs
- **Fast**: Sub-millisecond response times
- **Scalable**: Don't care which EC2 handles request
- **Session survives instance failure**: Data in cache, not instance RAM

**Create ElastiCache Redis cluster for session storage:**
```bash
aws elasticache create-cache-cluster \
  --cache-cluster-id session-store \
  --cache-node-type cache.t3.small \
  --engine redis \
  --num-cache-nodes 1 \
  --cache-subnet-group-name my-cache-subnet-group \
  --security-group-ids sg-cache-only-from-ec2
```

---

<a name="section-3"></a>

# SECTION 3: MyWordPress.com — Stateful + Media Storage

## The Third Problem: Shared Media

**MyWordPress.com** requires:
1. User accounts (sessions) → ElastiCache ✓
2. Blog posts & images (database) → ?
3. All users see same media → shared storage?

### The EBS Problem (Wrong Solution)

```
NAIVE APPROACH: Attach single EBS to one instance
┌──────────────────────────────────────────┐
│               ALB                        │
└──────────┬───────────────────────────────┘
           │
    ┌──────┼───────┐
    ▼      ▼       ▼
  EC2-1  EC2-2   EC2-3
    ▲
    │ (only connected here!)
    │
  ┌─────────────────────┐
  │  EBS Volume         │
  │  /var/www/wordpress │
  │  user images here   │
  └─────────────────────┘

Problem:
  EC2-2 uploads image ──▶ /tmp/wordpress  (local storage)
  EC2-3 can't see it (EBS not attached!)
  Image disappears
```

### The Better Solution: EFS (Shared File System)

```
┌──────────────────────────────────────────┐
│               ALB                        │
└──────────┬───────────────────────────────┘
           │
    ┌──────┼────────┐
    ▼      ▼        ▼
  EC2-1  EC2-2   EC2-3 (all can mount same EFS!)
    │      │        │
    └──────┼────────┘
           │
    ┌──────▼────────────┐
    │   EFS             │
    │   (Multi-AZ NFS)  │
    │                   │
    │ /wordpress/media/ │
    │  ├─ image1.jpg    │
    │  ├─ image2.jpg    │
    │  └─ image3.jpg    │
    │ (all instances see) │
    └───────────────────┘
```

### How It Works
1. EC2-1 mounts EFS at `/mnt/efs`
2. EC2-2 mounts EFS at `/mnt/efs`
3. EC2-3 mounts EFS at `/mnt/efs`
4. Any file written to `/mnt/efs` visible to all three

**Create EFS for WordPress images:**
```bash
aws efs create-file-system \
  --performance-mode generalPurpose \
  --encrypted \
  --tags Key=Name,Value=wordpress-images
```

**Create mount targets in each AZ:**
```bash
# Mount target in AZ-1
aws efs create-mount-target \
  --file-system-id fs-0abc123def456 \
  --subnet-id subnet-az1 \
  --security-groups sg-efs-from-ec2

# Mount target in AZ-2
aws efs create-mount-target \
  --file-system-id fs-0abc123def456 \
  --subnet-id subnet-az2 \
  --security-groups sg-efs-from-ec2
```

### Complete WordPress Architecture

```
┌────────────────────────────────────────────────────────┐
│                     ROUTE 53 (DNS)                      │
│              mywordpress.com → ALB IP                   │
└────────────┬───────────────────────────────────────────┘
             │
    ┌────────▼─────────────────────────────┐
    │    Application Load Balancer (ALB)    │
    │    Port 80/443 → instance port 8080   │
    └────────┬─────────────────────────────┘
             │ (health checks every 30s)
      ┌──────┼──────┐
      ▼      ▼      ▼
    EC2-1  EC2-2  EC2-3 (ASG: min 2, max 6)
      │      │      │ (WordPress app + PHP)
      └──────┼──────┘
             │ (all write to EFS)
      ┌──────▼──────────────────┐
      │     EFS                  │
      │ /wp-content/uploads/     │
      └──────────────────────────┘
             │
             │ (app also queries database)
             │
      ┌──────▼──────────────────────────┐
      │ RDS MySQL                        │
      │ - Multi-AZ for failover          │
      │ - Automated backups              │
      │ - Read replica for reporting     │
      └──────────────────────────────────┘
             │
      ┌──────▼──────────────────┐
      │ ElastiCache Redis        │
      │ (session storage)        │
      │ (user login state)       │
      └──────────────────────────┘
```

**Create RDS with Multi-AZ for high availability:**
```bash
aws rds create-db-instance \
  --db-instance-identifier user-data-db \
  --db-instance-class db.t3.medium \
  --engine mysql \
  --master-username admin \
  --master-user-password MySecurePass123 \
  --allocated-storage 100 \
  --multi-az \
  --backup-retention-period 7 \
  --no-publicly-accessible \
  --vpc-security-group-ids sg-rds-from-ec2-only
```

**Create RDS read replica for read scaling:**
```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier user-data-db-read-1 \
  --source-db-instance-identifier user-data-db \
  --db-instance-class db.t3.medium
```

---

## The Security Groups Pattern

This architecture uses **layered security groups**:

```
Layer 1: Internet-Facing (Public)
┌─────────────────────────────────┐
│ ALB Security Group (elb-sg)     │
│ Inbound:                        │
│  - 0.0.0.0/0 port 80 (HTTP)    │
│  - 0.0.0.0/0 port 443 (HTTPS)  │
└─────────────────────────────────┘

Layer 2: Application Servers
┌─────────────────────────────────┐
│ EC2 Security Group (app-sg)     │
│ Inbound:                        │
│  - elb-sg port 8080 (from ALB) │
│  - NO direct internet access    │
└─────────────────────────────────┘

Layer 3: Database (Private)
┌─────────────────────────────────┐
│ RDS Security Group (rds-sg)     │
│ Inbound:                        │
│  - app-sg port 3306 (MySQL)    │
│  - NO internet access           │
└─────────────────────────────────┘

Layer 4: Cache (Private)
┌─────────────────────────────────┐
│ ElastiCache SG (cache-sg)       │
│ Inbound:                        │
│  - app-sg port 6379 (Redis)    │
│  - NO internet access           │
└─────────────────────────────────┘
```

**Create security groups with SG-to-SG rules:**
```bash
# Create ELB security group
aws ec2 create-security-group \
  --group-name elb-sg \
  --description "ELB accepts all HTTP/HTTPS" \
  --vpc-id vpc-1234567890abcdef0

# Create app security group
aws ec2 create-security-group \
  --group-name app-sg \
  --description "EC2 accepts only from ELB" \
  --vpc-id vpc-1234567890abcdef0

# Allow ALB → EC2 (SG-to-SG reference)
aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp \
  --port 8080 \
  --source-group sg-elb

# Create RDS security group
aws ec2 create-security-group \
  --group-name rds-sg \
  --description "RDS accepts only from App SG" \
  --vpc-id vpc-1234567890abcdef0

# Allow EC2 → RDS
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds \
  --protocol tcp \
  --port 3306 \
  --source-group sg-app

# Create ElastiCache security group
aws ec2 create-security-group \
  --group-name cache-sg \
  --description "Cache accepts only from App SG" \
  --vpc-id vpc-1234567890abcdef0

# Allow EC2 → ElastiCache
aws ec2 authorize-security-group-ingress \
  --group-id sg-cache \
  --protocol tcp \
  --port 6379 \
  --source-group sg-app
```

---

<a name="section-4"></a>

# SECTION 4: Instantiating Applications Quickly

## The Speed Problem

When traffic spikes and ASG launches new instances:

```
SCENARIO: Traffic jumps from 10 instances → 30 instances

ASG launches new instance ──────────────────────────┐
                                                    │
If using User Data (script):                       │
  1. OS boots (1 min)                              │
  2. User Data runs:                               │
     - Download application code (2 min)           │
     - Install dependencies (Java, Node, etc.) (3 min)
     - Start services (1 min)                      │
                                                    │
  TOTAL TIME: ~7 minutes before app is ready ──┐   │
                              │                │   │
                              ▼                ▼   ▼
                        Users see latency during scale-up
```

### Solution: Golden AMI

A **Golden AMI** is a pre-baked EC2 image with everything pre-installed and configured.

**Workflow:**
1. Launch instance from standard Ubuntu AMI
2. Install Java, Node, dependencies
3. Install application code and config
4. Create snapshot image → Golden AMI
5. Use Golden AMI in ASG Launch Template

**Benefits:**
- Instance ready in 1-2 minutes (not 7+)
- Consistent deployments (same everywhere)
- Faster auto-scaling response

**Create a Golden AMI from a running instance:**
```bash
aws ec2 create-image \
  --instance-id i-1234567890abcdef0 \
  --name "my-app-golden-ami-2026-03-06" \
  --description "Pre-baked AMI with app dependencies" \
  --no-reboot
```

**List AMIs you own:**
```bash
aws ec2 describe-images --owners self
```

**Share AMI with another AWS account:**
```bash
aws ec2 modify-image-attribute \
  --image-id ami-0abcdef1234567890 \
  --launch-permission "Add=[{UserId=987654321098}]"
```

### Naming Convention for AMIs

Approach 1: Semantic Versioning
```
ami-myapp-v1.0.0 (Java 11, Node 16)
ami-myapp-v1.1.0 (Node 18 update)
ami-myapp-v2.0.0 (Java 17 upgrade)
```

Approach 2: Build Timestamp
```
ami-myapp-2026-03-05-10-30
ami-myapp-2026-03-05-14-22
```

Approach 3: Git Commit Hash
```
ami-myapp-a3f9e2b
ami-myapp-f4e2d1c
```

Best practice: Semantic versioning (clear meaning)

---

## Common Exam Scenarios

### Scenario 1: Site Down During Traffic Spike

**Situation**: Traffic spikes 10x. Site becomes unresponsive.

**Problem**:
- Single instance (no ASG)
- Or ASG but instances take 10 minutes to launch

**Solution**:
1. Implement ASG
2. Use Golden AMI (1-2 min startup)
3. Pre-warm ASG (keep buffer instances)
4. Use Kinesis/SQS to buffer requests during spike

---

### Scenario 2: Losing User State

**Situation**: Shopping cart disappears, user session lost.

**Problem**:
- Using ELB stickiness with single instance
- Instance failure = lost cart

**Solution**:
1. Store sessions in ElastiCache
2. All instances read from cache
3. Enable Multi-AZ ElastiCache
4. Stateless instances, distributed sessions

---

### Scenario 3: Images Not Visible on All Instances

**Situation**: Upload image on EC2-A, can't see on EC2-B.

**Problem**:
- EBS volume attached to EC2-A only
- EC2-B can't access

**Solution**:
1. Use EFS instead of EBS
2. Mount same EFS on all instances
3. Shared storage, everyone sees uploads

---

### Scenario 4: Database Becoming Bottleneck

**Situation**: Database CPU at 100%, application slow.

**Problem**:
- All traffic (reads + writes) hitting one database
- Can't scale vertically more

**Solution**:
1. RDS Read Replicas for read traffic
2. ElastiCache Lazy Loading for repeated reads
3. Multi-AZ for resilience (separate concern)
4. Eventually: database sharding (advanced)

---

### Scenario 5: Disaster Recovery Takes Too Long

**Situation**: Database crashes. RDS restore takes 30 minutes.

**Problem**:
- Relying on daily snapshots
- Restores slowly

**Solution**:
1. Enable RDS Multi-AZ (automatic failover in 1-2 min)
2. Regular snapshots for cross-region DR
3. Eventually: Aurora Global Database (read-only replica in another region)

---

## Interview Question Patterns

### Pattern 1: "Tell me about [architecture] and when you'd use it"

**Setup**: Describe WhatIsTheTime stateless pattern
**Why**: Explain each component (ASG, ALB, AZ redundancy, Golden AMI)
**Trade-off**: Discuss limitations (no per-user state)
**Alternative**: When you'd use ElastiCache instead

### Pattern 2: "We have problem X. What AWS service fixes it?"

**Problem**: User data lost when instance fails  
**Answer**: ElastiCache or DynamoDB (external storage)

**Problem**: Scaling takes too long  
**Answer**: Golden AMI or Docker containers

**Problem**: One AZ going down breaks app  
**Answer**: Deploy across multiple AZs, use Multi-AZ RDS/ElastiCache

### Pattern 3: "What are the trade-offs of..."

**Golden AMI vs User Data**:
- Golden: fast (1 min), static, must rebuild
- User Data: slow (7+ min), flexible, simple

**EBS vs EFS**:
- EBS: single instance, fast, good for OS
- EFS: multi-instance, network latency, shared storage

**ElastiCache vs DynamoDB for sessions**:
- ElastiCache: fast (~1ms), cost per instance
- DynamoDB: slower (~50ms), serverless, auto-scaling

---

## Final Checklist: What You MUST Know for SAA-C03

### Architecture Concepts
- [ ] Horizontal vs vertical scaling
- [ ] Stateless vs stateful applications
- [ ] When to use each pattern
- [ ] Single points of failure and how to eliminate them

### Specific Technologies
- [ ] **EC2**: instances, AMIs, security groups
- [ ] **Auto Scaling Group**: min/max/desired, launch templates, scaling policies
- [ ] **Load Balancers**: ALB (application-level), NLB (network-level), health checks
- [ ] **Route 53**: DNS, alias records, TTL
- [ ] **RDS**: Multi-AZ, read replicas, snapshots, Aurora
- [ ] **ElastiCache**: Redis vs Memcached, lazy loading, multi-AZ
- [ ] **EBS**: block storage, snapshots, single-instance
- [ ] **EFS**: shared storage, multi-AZ, pricing model

### Design Patterns
- [ ] WhatIsTheTime (stateless evolution)
- [ ] MyClothes (sessions + caching)
- [ ] MyWordPress (database + shared storage)
- [ ] Golden AMI (fast startup)

### Exam Tricks
- [ ] Always think about: single points of failure, scalability, disaster recovery
- [ ] Multi-AZ = resilience (not read scaling)
- [ ] Read Replicas = read scaling (not failover)
- [ ] EBS = single instance (not shared)
- [ ] EFS = multi-instance (not single fast access)
- [ ] Stickiness = bad pattern (use ElastiCache instead)

---

## Quick Reference — AWS CLI Commands

### Elastic Load Balancer Stickiness
```bash
# Enable sticky sessions on target group
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/50dc6c495c0c9188 \
  --attributes Key=stickiness.enabled,Value=true Key=stickiness.type,Value=lb_cookie Key=stickiness.lb_cookie.duration_seconds,Value=86400
```

### ElastiCache Session Store
```bash
# Create ElastiCache Redis for session storage
aws elasticache create-cache-cluster \
  --cache-cluster-id session-store \
  --cache-node-type cache.t3.small \
  --engine redis \
  --num-cache-nodes 1 \
  --cache-subnet-group-name my-cache-subnet-group \
  --security-group-ids sg-cache-only-from-ec2
```

### RDS
```bash
# Create RDS with Multi-AZ for high availability
aws rds create-db-instance \
  --db-instance-identifier user-data-db \
  --db-instance-class db.t3.medium \
  --engine mysql \
  --master-username admin \
  --master-user-password MySecurePass123 \
  --allocated-storage 100 \
  --multi-az \
  --backup-retention-period 7 \
  --no-publicly-accessible \
  --vpc-security-group-ids sg-rds-from-ec2-only

# Create RDS read replica for read scaling
aws rds create-db-instance-read-replica \
  --db-instance-identifier user-data-db-read-1 \
  --source-db-instance-identifier user-data-db \
  --db-instance-class db.t3.medium
```

### Amazon EFS
```bash
# Create EFS for shared image storage
aws efs create-file-system \
  --performance-mode generalPurpose \
  --encrypted \
  --tags Key=Name,Value=wordpress-images

# Create mount target in AZ-1
aws efs create-mount-target \
  --file-system-id fs-0abc123def456 \
  --subnet-id subnet-az1 \
  --security-groups sg-efs-from-ec2

# Create mount target in AZ-2
aws efs create-mount-target \
  --file-system-id fs-0abc123def456 \
  --subnet-id subnet-az2 \
  --security-groups sg-efs-from-ec2
```

### Golden AMI
```bash
# Create AMI from running instance
aws ec2 create-image \
  --instance-id i-1234567890abcdef0 \
  --name "my-app-golden-ami-2026-03-06" \
  --description "Pre-baked AMI with app dependencies" \
  --no-reboot

# List AMIs you own
aws ec2 describe-images --owners self

# Share AMI with another account
aws ec2 modify-image-attribute \
  --image-id ami-0abcdef1234567890 \
  --launch-permission "Add=[{UserId=987654321098}]"
```

### Security Groups Pattern
```bash
# Create ELB security group
aws ec2 create-security-group \
  --group-name elb-sg \
  --description "ELB accepts all HTTP/HTTPS" \
  --vpc-id vpc-1234567890abcdef0

# Create app security group
aws ec2 create-security-group \
  --group-name app-sg \
  --description "EC2 accepts only from ELB" \
  --vpc-id vpc-1234567890abcdef0

# Allow ALB to EC2 (SG-to-SG)
aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp \
  --port 8080 \
  --source-group sg-elb

# Create RDS security group
aws ec2 create-security-group \
  --group-name rds-sg \
  --description "RDS accepts only from App SG" \
  --vpc-id vpc-1234567890abcdef0

# Allow EC2 to RDS
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds \
  --protocol tcp \
  --port 3306 \
  --source-group sg-app

# Create ElastiCache security group
aws ec2 create-security-group \
  --group-name cache-sg \
  --description "Cache accepts only from App SG" \
  --vpc-id vpc-1234567890abcdef0

# Allow EC2 to ElastiCache
aws ec2 authorize-security-group-ingress \
  --group-id sg-cache \
  --protocol tcp \
  --port 6379 \
  --source-group sg-app
```

---

## Parting Wisdom

The AWS Solutions Architect exam is testing **architectural thinking**, not just knowledge of services.

**Core principle**: Understand the *why* behind each design decision.

- Why multi-AZ? → Resilience to AZ failures
- Why ElastiCache? → Distributed sessions, cache frequently accessed data
- Why Golden AMI? → Fast scaling during spikes
- Why EFS? → Shared storage across instances

Master these three solutions architectures (stateless, stateful with sessions, stateful with media), and you'll handle 70% of the exam questions.

**Good luck with your exam!**

---

## Reference Links

- **Elastic Load Balancing**: https://docs.aws.amazon.com/elasticloadbalancing/
- **ElastiCache Documentation**: https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/WhatIs.html
- **EFS Guide**: https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html
- **EC2 AMIs**: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html
- **RDS Best Practices**: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_BestPractices.html
- **Auto Scaling**: https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html

---

**Document Version**: 1.0  
**Last Updated**: March 5, 2026  
**Status**: Ready for SAA-C03 exam preparation
