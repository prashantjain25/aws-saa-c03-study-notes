# Section 05 — Compute & Load Balancing

> **Purpose**: Compute is where architecture becomes concrete. A VPC design, an IAM policy, a security group — these are abstractions until a workload runs on them. This section covers the full spectrum of AWS compute: from EC2 instances you fully control, to Lambda functions where AWS manages everything, to container orchestrators where you trade control for operational efficiency. The architect's job is not to choose the "best" compute option but to match compute abstractions to operational requirements, team capabilities, and cost constraints.
>
> **Official Documentation**: [EC2](https://docs.aws.amazon.com/ec2/) | [Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) | [ECS](https://docs.aws.amazon.com/ecs/) | [EKS](https://docs.aws.amazon.com/eks/) | [ELB](https://docs.aws.amazon.com/elasticloadbalancing/)

---

## 1. EC2: The Foundation of AWS Compute

### 1.1 Why EC2 Still Matters

Despite the rise of serverless and containers, EC2 remains the foundation because it offers **maximum control**. When you need custom kernel modules, specific CPU architectures, GPU drivers, bare-metal performance, or legacy software that cannot run in containers, EC2 is the answer.

**EC2's architectural contract**: AWS manages the hardware, hypervisor, and network virtualization. You manage the operating system, runtime, application, and security patches.

### 1.2 Instance Families and Selection Logic

| Family | Optimized For | Characteristics | Example Use Cases |
|--------|--------------|-----------------|-------------------|
| **T** (Burstable) | General purpose, variable CPU | CPU credits accumulate during idle, burst during load. Baseline performance + burst capability. Cheapest option. | Microservices, dev/test, low-traffic web apps, CI/CD runners |
| **M** (General Purpose) | Balanced compute, memory, network | Consistent baseline performance. No bursting. The "safe default" choice. | Application servers, small-medium databases, backend services |
| **C** (Compute Optimized) | High CPU, lower memory per vCPU | Highest CPU performance per dollar. | Batch processing, HPC, gaming servers, video encoding |
| **R** (Memory Optimized) | High memory-to-CPU ratio | Up to 768 GB RAM per instance. | In-memory caches, large databases (MySQL, PostgreSQL), SAP HANA |
| **I** (Storage Optimized) | High IOPS, NVMe SSD | Local NVMe storage. Extremely high IOPS. | NoSQL databases (Cassandra, MongoDB), data warehouses, log processing |
| **G/P/Trn** (Accelerated) | GPU/ML inference | NVIDIA GPUs, AWS Trainium/Inferentia chips. | ML training, inference, graphics rendering, video transcoding |
| **Graviton** (ARM-based) | Cost-efficiency | AWS-designed ARM processors. Up to 40% better price-performance. | General workloads that compile on ARM. NOT for x86-only software. |

> **Architectural Decision**: Start with `M6i` or `M6g` (Graviton) for general workloads. Move to `C` or `R` only when profiling shows CPU or memory is the bottleneck. The default should be Graviton for new workloads unless there is a specific x86 dependency.

### 1.3 EC2 Purchasing Options: The Cost-Reliability Tradeoff

| Option | Cost Savings | Reliability | Use Case | Critical Caveat |
|--------|-------------|-------------|----------|----------------|
| **On-Demand** | None (baseline) | Maximum | Unpredictable workloads, dev/test, short-term | Most expensive; only use when flexibility is paramount |
| **Reserved Instances** | Up to 72% (1-3 year commitment) | Same as On-Demand | Steady-state production workloads | Pay regardless of usage. Modifying reservations has constraints. |
| **Savings Plans** | Similar to RIs (~72%) | Same as On-Demand | Flexible commitment across instance families/regions | Compute Savings Plans are more flexible than RIs. EC2 Instance Savings Plans are cheaper but family-specific. |
| **Spot Instances** | Up to 90% | Interrupted with 2-minute warning | Fault-tolerant, stateless batch jobs, CI/CD, rendering | NOT for stateful workloads without checkpointing. interruption can happen anytime. |
| **Dedicated Hosts** | N/A (more expensive) | Single-tenant hardware | Software licensing per-core (BYOL), compliance requirements | You pay for the entire host regardless of utilization. |

> **Hybrid Strategy for Production**: Run base capacity on 1-year Savings Plans (predictable, lower cost), burst with On-Demand, and run batch workloads on Spot. This gives cost optimization without sacrificing availability.

### 1.4 EC2 Networking: ENI, ENA, and Placement

**Elastic Network Interface (ENI)**:
- An ENI is a virtual network card. Each EC2 instance has at least one primary ENI.
- You can attach multiple ENIs to an instance (limit varies by instance type).
- ENIs can be detached from one instance and attached to another — useful for failover patterns.
- Each ENI has a primary private IP and can have multiple secondary private IPs.

**Enhanced Networking (ENA)**:
- ENA provides higher bandwidth, lower latency, and lower jitter compared to traditional virtual NICs.
- Required for instances above 10 Gbps networking.
- Automatically enabled on modern instance types (M5, C5, R5, and later).

**Placement Groups**:
| Type | Behavior | Use Case |
|------|----------|----------|
| **Cluster** | Instances in the same AZ, same rack, lowest network latency | HPC, tightly-coupled computing. Risk: rack failure affects all instances |
| **Spread** | Each instance on distinct underlying hardware (max 7 per AZ per group) | Critical singleton workloads where hardware failure must be isolated |
| **Partition** | Multiple partitions, each on isolated hardware. Many instances per partition. | Large distributed systems (Kafka, Hadoop) where you want failure isolation within the cluster |

> **Important**: Cluster placement groups cannot span AZs. Spread placement groups are limited to 7 instances per AZ. Partition placement groups support up to 7 partitions per AZ with hundreds of instances each.

### 1.5 EC2 Instance Store vs EBS vs EFS

| Storage | Persistence | Performance | Use Case |
|---------|------------|-------------|----------|
| **Instance Store** | Ephemeral (lost on stop/terminate) | Highest (NVMe, local to host) | Temporary cache, buffers, scratch space. NEVER for data you cannot lose. |
| **EBS** | Persistent, survives stop/start | Good (gp3: up to 16K IOPS, io2: up to 256K IOPS) | General-purpose persistent storage. Boot volumes, databases, file systems. |
| **EFS** | Persistent, shared across instances | Good (scales automatically, burst model) | Shared file systems, content management, home directories. |

**EBS Volume Types**:

| Type | IOPS | Throughput | Latency | Cost | Best For |
|------|------|-----------|---------|------|----------|
| **gp3** (SSD) | 3,000-16,000 (configurable) | 125-1,000 MiB/s | <1ms | Low | General purpose. Replace gp2 for new workloads. |
| **gp2** (SSD) | 3 IOPS/GB (burstable to 3,000) | 250 MiB/s max | <1ms | Medium | Legacy. Performance tied to volume size. |
| **io2/io2 Block Express** | 256,000-1,000,000 | 4,000 MiB/s | Sub-millisecond | High | Mission-critical databases requiring highest IOPS and durability (99.999% durability). |
| **st1** (HDD) | N/A (throughput-optimized) | 500 MiB/s | Higher | Low | Big data, sequential workloads, log processing. |
| **sc1** (HDD) | N/A (cold storage) | 250 MiB/s | Higher | Lowest | Infrequently accessed data. |

> **gp3 vs gp2**: gp3 decouples IOPS and throughput from volume size. A 100 GB gp3 volume can have 16,000 IOPS. A 100 GB gp2 volume has 300 IOPS (3 × 100). For low-size, high-IOPS needs, gp3 is dramatically better.

### 1.6 EC2 Hibernate and Fast Boot

**Hibernate** saves the contents of RAM to the root EBS volume and shuts down the instance. On start, RAM is restored from disk — the OS boots from hibernation, not from scratch.

- **Use case**: Long-running processes with large in-memory state that you want to pause overnight to save costs.
- **Requirements**: Root volume must be EBS, instance type must support hibernate, OS must support hibernate, sufficient root volume space for RAM dump.
- **Limitation**: Hibernate does NOT preserve instance store data. If you need both RAM and instance store preserved, you must handle that separately.

---

## 2. Elastic Load Balancing: Traffic Distribution Architecture

### 2.1 The Load Balancer Taxonomy

AWS offers four load balancers. Understanding their layer in the OSI model determines when to use each:

```
OSI Layer → Load Balancer → Key Characteristic
─────────────────────────────────────────────────
Layer 7   → ALB           → HTTP/HTTPS-aware, content-based routing
Layer 4   → NLB           → TCP/UDP, ultra-low latency, static IP
Layer 3   → GWLB          → Transparent security appliance insertion
Legacy    → CLB           → Deprecated. Do not use for new workloads.
```

### 2.2 Application Load Balancer (ALB)

**ALB's core capability**: It inspects HTTP requests and routes based on content.

**Routing Rules**:
- **Path-based**: `/api/*` → API target group, `/static/*` → S3 (via Lambda or EC2)
- **Host-based**: `api.example.com` → API targets, `www.example.com` → Web targets
- **HTTP header**: Route based on custom headers (e.g., `X-Canary: true` → canary deployment target group)
- **Query string**: `?version=v2` → v2 target group
- **Source IP**: Geographic or IP-based routing

**Target Group Types**:
- EC2 instances
- ECS tasks (IP-based registration)
- Lambda functions (HTTP request translated to Lambda event)
- IP addresses (on-premises servers, other cloud providers)

**ALB Operational Characteristics**:
- **Client IP preservation**: ALB does NOT preserve the original client IP to targets. Targets see the ALB's internal IP. The original client IP is in the `X-Forwarded-For` header.
- **Cross-zone load balancing**: Enabled by default (free). Traffic is distributed across all targets in all AZs equally.
- **Connection idle timeout**: 60 seconds default. Long-polling or WebSocket applications may need this increased.
- **Slow start mode**: Gradually ramp up traffic to new targets (useful for JVM applications with long warm-up).

### 2.3 Network Load Balancer (NLB)

NLB operates at Layer 4 (TCP/UDP). It does not inspect HTTP content. This makes it faster but less flexible.

**NLB Key Characteristics**:
- **Static IP per AZ**: Each AZ gets one static IP (or Elastic IP). Essential when clients need to whitelist specific IPs.
- **Client IP preservation**: NLB preserves the original client IP. Targets see the real source IP (via Proxy Protocol for TCP).
- **Ultra-low latency**: ~100 microseconds vs ~400 microseconds for ALB.
- **Protocol support**: TCP, UDP, TLS (pass-through or termination).
- **Cross-zone load balancing**: Disabled by default. Enabling it incurs inter-AZ data transfer charges.

**When to use NLB**:
- Static IP requirement (firewall whitelists, third-party integration)
- Ultra-high throughput (millions of requests per second)
- Non-HTTP protocols (TCP, UDP, MQTT, gaming protocols)
- TLS pass-through (terminate TLS on the target, not the load balancer)

> **NLB + ALB Hybrid Pattern**: Use NLB in front for static IP and DDoS resilience, then route to ALB for HTTP routing. This combines NLB's Layer 4 performance with ALB's Layer 7 intelligence.

### 2.4 Gateway Load Balancer (GWLB)

GWLB is not a traditional load balancer — it is a **transparent network appliance insertion mechanism**.

**Architecture**:
```
User Traffic ──► GWLB ──► [Third-Party Security Appliance Fleet] ──► GWLB ──► Application
                         (Palo Alto, Fortinet, Suricata, etc.)
                         GENEVE protocol on port 6081
```

- **Transparent**: The application doesn't know traffic was inspected.
- **Route table integration**: You add a route table entry sending traffic to the GWLB endpoint.
- **Bidirectional**: Both inbound and outbound traffic can be inspected.

**Use case**: Compliance requiring all VPC traffic to pass through a specific firewall or IDS/IPS before reaching applications.

> **GWLB Limitation**: The appliance fleet runs as EC2 instances or Gateway Load Balancer Endpoints. You manage the appliance software lifecycle. GWLB itself is just the plumbing.

### 2.5 SSL/TLS and SNI

**SSL Termination at the Load Balancer**:
```
Client ──HTTPS──► [ALB/NLB] ──HTTP──► EC2 Target
                 ↑ TLS cert (ACM)
```

- Reduces CPU load on targets (they handle HTTP, not TLS)
- Centralizes certificate management
- Simplifies certificate renewal (ACM auto-renewal)

**SNI (Server Name Indication)**:
- SNI allows multiple TLS certificates on a single IP/port.
- The client includes the hostname in the TLS handshake.
- ALB and NLB support SNI. CLB does NOT.
- **CloudFront + SNI**: CloudFront uses SNI by default. For clients that don't support SNI (legacy IoT, very old browsers), you need Dedicated IP Custom SSL (~$600/month).

---

## 3. Auto Scaling Groups: Elastic Capacity

### 3.1 ASG Architecture

An ASG is a **control loop**: it monitors metrics, compares to thresholds, and adjusts instance count.

```
CloudWatch Metrics (CPU, ALB request count, custom) ──┐
                                                      ▼
                                               [Scaling Policy]
                                                      │
                              ┌───────────────────────┴───────────────────────┐
                              ▼                                               ▼
                      Scale Out (add instances)                      Scale In (remove instances)
                              │                                               │
                              ▼                                               ▼
                    Launch Template / Launch Configuration          Terminate oldest / least healthy
                              │                                               │
                              ▼                                               ▼
                    New instances register with ELB                ELB stops sending traffic first
```

### 3.2 Scaling Policy Types

| Policy Type | Behavior | Best For | Caveat |
|-------------|----------|----------|--------|
| **Target Tracking** | Maintains a metric at a target value (e.g., CPU = 50%) | Most workloads. Simple, automatic. | Can oscillate if metric is noisy. Choose smooth metrics (CPU, request count per target). |
| **Step Scaling** | Add/remove N instances when alarm threshold breached | Predictable step patterns | Requires CloudWatch alarms. Manual threshold tuning. |
| **Scheduled Scaling** | Scale at specified times | Predictable traffic (business hours, batch windows) | No automatic reaction to unexpected demand. |
| **Predictive Scaling** | ML forecasts demand and pre-provisions | Regular cyclical patterns | Costly if pattern is irregular (over-provisions). |
| **Manual Scaling** | Directly set desired capacity | Emergencies, testing | Not automatic. |

> **Scaling Cooldown**: Default 300 seconds between scaling activities. Prevents thrashing. For applications with long boot times, increase cooldown. For fast-booting containers, you can decrease it.

### 3.3 Launch Templates vs Launch Configurations

Launch Configurations are **deprecated**. Always use Launch Templates.

**Launch Template capabilities**:
- Multiple versions (immutable history)
- Mixed instance policies (Spot + On-Demand in same ASG)
- CPU options (disable hyperthreading for licensing)
- Advanced EBS and network configurations
- T2/T3 unlimited CPU credits setting

**ASG with Mixed Instances Policy**:
```
ASG: Desired = 10
├── On-Demand Base: 2 (always On-Demand)
├── On-Demand Percentage: 50% of remaining = 4
└── Spot Instances: remaining 4
    ├── Instance type: M6g.large (priority 1)
    ├── Instance type: M5.large (priority 2, fallback)
    └── Instance type: C6g.large (priority 3, fallback)
```

This pattern gives cost optimization (Spot for 40% of capacity) with reliability (On-Demand for critical base + half of variable).

### 3.4 Health Checks and Termination

**Health Check Types**:
- **ELB health checks**: ASG uses ALB/NLB health check status. If ALB marks an instance unhealthy, ASG replaces it.
- **EC2 status checks**: ASG monitors EC2's built-in status checks (system and instance).

**Termination Policies** (control which instance to remove during scale-in):
- Default: AZ with most instances, then oldest launch template
- OldestInstance: Terminate the oldest (useful for immutable deployments)
- NewestInstance: Terminate the newest (useful for testing)
- OldestLaunchTemplate: Terminate instances with oldest configuration first
- AllocationStrategy: Consider Spot vs On-Demand (terminate Spot first to save costs)

> **Scale-In Protection**: Individual instances can be protected from termination. Useful for stateful instances, or during deployments when you want to ensure the new version isn't immediately terminated.

---

## 4. AWS Lambda: Serverless Compute

### 4.1 Lambda's Execution Model

Lambda is **event-driven, stateless, ephemeral compute**. Each invocation runs in a fresh execution environment (with caveats — see cold starts).

**Lambda's architectural contract**:
- You provide code and configuration.
- AWS provisions infrastructure, scales automatically, patches the OS, and monitors health.
- You pay per invocation + compute time (GB-seconds).
- Maximum execution time: 15 minutes.

### 4.2 Lambda Scaling Semantics

Lambda scaling is **concurrency-based**, not instance-based.

```
Incoming Events ──► Lambda Service ──► Execution Environments
                                        (concurrent invocations)

Initial burst: 1,000 concurrent executions per region (soft limit, can be increased)
Sustained rate: 500 new environments per minute after burst
```

**Scaling behavior**:
- Each concurrent invocation gets a separate execution environment.
- If 1,000 events arrive simultaneously, Lambda creates up to 1,000 environments (subject to burst limit).
- If events arrive faster than environments can be created, they are queued (for async invocations) or throttled (for sync invocations).

**Reserved Concurrency**: Guarantee a minimum number of concurrent environments for a function. Also acts as a **maximum** — if reserved concurrency is 50, the function can never exceed 50 concurrent executions.

**Provisioned Concurrency**: Pre-warm environments so they are ready to serve requests immediately. Eliminates cold start latency. You pay for provisioned capacity even when not used.

> **Critical Design Scenario**: "A Lambda function processes SQS messages. Messages start queueing up. CPU is low. What's happening?"
> **Answer**: Check `ConcurrentExecutions` metric. If it hits the reserved concurrency limit or account burst limit, Lambda cannot create more environments. Messages queue in SQS. CPU is low because the function isn't even running — it's waiting for concurrency capacity. Solution: increase reserved concurrency or account limit.

### 4.3 Cold Starts and Mitigation

**What causes cold starts**:
- First invocation of a function (environment creation)
- After a period of inactivity (environments are recycled)
- After a code or configuration update
- Scaling beyond currently warm environments

**Cold start mitigation strategies**:
1. **Provisioned Concurrency**: Keep environments warm. Most effective but costs money.
2. **Keep-alive pings**: CloudWatch Events rule triggering function every 5-15 minutes. Cheap but not guaranteed (Lambda can still recycle).
3. **Smaller deployment packages**: Faster download and initialization.
4. **Lambda SnapStart (Java only)**: Pre-initialize the function and snapshot the JVM. Restores from snapshot ~10x faster than cold start.
5. **Graviton2 (ARM)**: Generally faster initialization than x86.
6. **Avoid VPC for simple functions**: VPC-enabled Lambda requires ENI creation, adding 5-15 seconds to cold start. Use VPC only when the function MUST access VPC resources.

> **VPC Lambda Cold Start**: When a Lambda function is configured with VPC access, AWS creates an ENI in the specified subnets. ENI creation is the dominant cold start factor (often 5-15 seconds). Solutions: use VPC endpoints (S3, DynamoDB) to avoid VPC, use Hyperplane ENI sharing (modern, reduces ENI creation), or use provisioned concurrency.

### 4.4 Lambda Failure Modes and Retry Behavior

| Invocation Type | Retry Behavior | Dead Letter Queue |
|----------------|---------------|-------------------|
| **Synchronous** | No automatic retry. Client receives error immediately. | Not applicable (client handles retry) |
| **Asynchronous** (S3 events, SNS, EventBridge) | 2 retries with exponential backoff (1 min, 2 min). If all fail → DLQ or destination. | SQS DLQ or Lambda destination (SQS, SNS, Lambda, EventBridge) |
| **Polling** (SQS, Kinesis, DynamoDB Streams) | Built-in retry via message visibility timeout or stream checkpoint. Failed batches can be sent to DLQ. | SQS DLQ for SQS polling; Lambda destination for Kinesis/DynamoDB |

**Lambda Destinations (modern replacement for DLQ)**:
- On success: Send result to SQS, SNS, Lambda, or EventBridge
- On failure: Send error details to SQS, SNS, Lambda, or EventBridge
- More flexible than DLQ — you can route success and failure to different destinations.

### 4.5 Lambda Concurrency Limits and Throttling

**Account-level concurrency limit**: 1,000 concurrent executions per region (default, adjustable).

**Problem scenario**: One "noisy neighbor" function consumes all 1,000 concurrent executions, starving other functions.

**Solutions**:
1. **Reserved concurrency per function**: Guarantee minimums, cap maximums
2. **Provisioned concurrency**: Isolate capacity for critical functions
3. **Request concurrency increase**: AWS Support can raise the account limit (e.g., to 10,000+)

> **Lambda + RDS Connection Pooling**: Lambda's scale-out nature creates many database connections. RDS Proxy solves this by pooling connections between Lambda and RDS. Without RDS Proxy, a high-concurrency Lambda function can exhaust RDS `max_connections`.

---

## 5. Containers on AWS: ECS and EKS

### 5.1 The Container Orchestration Decision

| Dimension | Amazon ECS | Amazon EKS |
|-----------|-----------|------------|
| **Control plane** | AWS-managed (no Kubernetes API) | AWS-managed Kubernetes control plane |
| **Learning curve** | Lower (AWS-specific concepts) | Higher (Kubernetes expertise required) |
| **Portability** | AWS-only | Multi-cloud (Kubernetes is standard) |
| **Ecosystem** | AWS-native integrations | Rich ecosystem (Helm, operators, service mesh) |
| **Operational burden** | Lower (AWS handles more) | Higher (you manage add-ons, upgrades, networking) |
| **Cost** | No control plane fee | $0.10/hour per cluster (~$72/month) |
| **Best for** | Teams wanting simple container deployment, AWS-native tooling | Teams with Kubernetes expertise, multi-cloud strategy, complex requirements |

> **Architectural Decision**: Choose ECS if your team is AWS-focused and wants simplicity. Choose EKS if you need Kubernetes-native features (operators, CRDs, specific CNI plugins) or multi-cloud portability. Do NOT choose EKS just because "Kubernetes is industry standard" — if your use case is simple, EKS adds operational complexity with minimal benefit.

### 5.2 ECS Architecture Deep Dive

**ECS Components**:

```
Cluster
├── Task Definition (container specs: image, CPU, memory, ports, env vars)
├── Task (running instance of a task definition)
│   └── Container(s)
├── Service (maintains desired count of tasks, integrates with ALB)
└── Capacity Provider (ECS on EC2, Fargate, or External)
```

**ECS Launch Types**:

| Launch Type | Infrastructure | Control | Cost Model |
|-------------|---------------|---------|------------|
| **Fargate** | Serverless | Minimal (task CPU/memory only) | Per vCPU-hour + per GB-hour | 
| **EC2** | Your EC2 instances | Full (instance type, AMI, patching) | EC2 cost + no ECS surcharge |
| **External** | On-premises servers | Full | ECS Anywhere (per-instance fee) |

**ECS + Fargate**: The simplest path. You define task CPU and memory, AWS provisions infrastructure. No cluster management. Best for:
- Microservices with variable traffic
- Batch jobs
- Teams without container infrastructure expertise

**ECS + EC2**: You manage the EC2 instances. Best for:
- GPU workloads (Fargate doesn't support GPUs)
- Large tasks that need > 16 vCPU or > 120 GB memory (Fargate limits)
- Cost optimization via Reserved Instances or Spot
- Compliance requiring specific host-level controls

**ECS Task Networking**:
- **awsvpc mode**: Each task gets its own ENI. Most secure and scalable. Required for Fargate.
- **bridge mode**: Tasks share the host's network namespace. Legacy.
- **host mode**: Task uses host network directly. Limited.
- **none**: No external networking. Rare.

> **ECS Task IAM Roles**: Each task (not the EC2 host) can have its own IAM role. This is critical for security — the frontend service and backend service in the same cluster should have different permissions. Without task roles, all containers inherit the EC2 instance profile's permissions (violating least privilege).

### 5.3 EKS Architecture and Operational Realities

**EKS Control Plane**: AWS manages the Kubernetes API server, etcd, and control plane components. You pay $0.10/hour per cluster.

**EKS Data Plane**: You manage the worker nodes (EC2 or Fargate).

**EKS Networking (CNI)**:
- EKS uses the **Amazon VPC CNI** by default. Each pod gets an IP from the VPC CIDR.
- **Critical limitation**: The number of pods per node is limited by ENI attachments and available IPs in subnets. A large cluster can exhaust VPC IP addresses.
- **IPv6 mode**: Available to address IP exhaustion. Pods get IPv6 addresses; cluster communication uses IPv6.

**EKS Upgrades**:
- AWS manages control plane upgrades (1 version at a time, you initiate).
- YOU manage node upgrades. Common approaches: managed node group rolling update, blue/green node pools, or Karpenter auto-provisioning.
- **Breaking changes**: Kubernetes deprecates APIs between versions. A deployment using `extensions/v1beta1` will fail after upgrade. Always check deprecation notices.

**EKS Add-ons**:
- CoreDNS, kube-proxy, VPC CNI are required and AWS-managed.
- Optional: AWS Load Balancer Controller (replaces in-tree AWS cloud provider), EBS CSI driver, EFS CSI driver.

> **EKS vs Self-Managed Kubernetes on EC2**: Self-managed gives you full control but requires you to manage etcd backups, API server HA, and certificate rotation. EKS handles all of this. The $72/month cluster fee is trivial compared to the engineering cost of self-managing a production control plane.

---

## 6. Cross-Service Compute Patterns

### Pattern: Three-Tier Web Application
```
Internet ──► [Route53] ──► [CloudFront + WAF] ──► [ALB]
                                                     │
                    ┌──────────┬──────────┬──────────┘
                    ▼          ▼          ▼
                 [ECS/EKS]  [ECS/EKS]  [ECS/EKS]
                 Frontend   Frontend   Frontend
                    │
                    ▼
                 [ALB Internal]
                    │
                    ▼
                 [ECS/EKS]  [ECS/EKS]  [ECS/EKS]
                 Backend    Backend    Backend
                    │
                    ▼
                 [RDS Multi-AZ]    [ElastiCache]
                 Primary + Standby   Session/Cache
```

### Pattern: Event-Driven Microservices
```
S3 Upload Event ──► [Lambda] ──► [SQS] ──► [Lambda Worker]
                                              │
                                              ▼
                                         [DynamoDB]
```

### Pattern: Real-Time Streaming
```
IoT Devices ──► [Kinesis Data Streams] ──► [Lambda / KCL Consumer]
                                                  │
                                                  ▼
                                            [OpenSearch / S3]
```

---

## 7. Architectural Decision Challenges

* **Scenario:** Running production workloads on Spot Instances to save costs.
  * **Design:** Use Spot Fleet or ASG Mixed Instances Policy (Spot + On-Demand base) for workloads that are fault-tolerant, checkpoint-capable, diversified across AZs/instance types, and not latency-sensitive. Because Spot instances can be interrupted with a 2-minute warning, you should never run 100% Spot for critical production without failover.

* **Scenario:** A client needs to whitelist your load balancer IP in their firewall.
  * **Design:** Use a Network Load Balancer (NLB). Because NLB provides a static IP per AZ (via Elastic IP), whereas ALB provides a DNS name that resolves to dynamic, changing IPs over time.

* **Scenario:** A Lambda function in a VPC needs performant access to S3 and DynamoDB.
  * **Design:** Use VPC Gateway Endpoints for S3 and DynamoDB instead of a NAT Gateway. Because endpoints keep traffic on the AWS private backbone, are free for S3/DynamoDB, and avoid adding NAT Gateway latency to the Lambda VPC cold start. (Alternatively, if no other VPC resources are needed, remove the function from the VPC entirely to avoid ENI creation overhead).

* **Scenario:** A team of 5 developers is building a microservices platform and needs to choose between ECS and EKS.
  * **Design:** Use ECS with Fargate. Because the operational burden, learning curve, and cost (no $72/month cluster fee) of EKS are too high for a small team without existing Kubernetes expertise, whereas ECS provides sufficient capabilities for standard microservices and simpler management.

---

## 8. Points to Remember

- **Graviton (ARM) instances offer up to 40% better price-performance** — default to Graviton for new workloads unless x86-specific dependencies exist.
- **gp3 EBS outperforms gp2 at lower cost** for most workloads. Use gp2 only if already deployed.
- **Cluster placement groups cannot span AZs** — they sacrifice AZ-level HA for intra-rack latency.
- **ALB does not preserve client IP** — read `X-Forwarded-For`. NLB preserves client IP natively.
- **NLB cross-zone load balancing is disabled by default** and incurs inter-AZ charges when enabled.
- **CloudFront certificates MUST be in us-east-1** — always request them there.
- **Lambda burst concurrency = 1,000 per region** (default). Request increase for high-traffic workloads.
- **Lambda VPC cold start adds 5-15 seconds** due to ENI creation. Use VPC endpoints or remove VPC when possible.
- **Lambda 15-minute timeout** — design long-running processes as Step Functions workflows or ECS tasks.
- **ASG termination policies default to oldest launch configuration** — this can terminate newly deployed instances during rolling updates if not configured properly.
- **Spot Instances are viable for production** when workloads are fault-tolerant and use mixed instance policies.
- **ECS task roles provide least-privilege per container** — never rely on EC2 instance profiles for multi-tenant clusters.
- **EKS VPC CNI can exhaust VPC IPs** — plan subnet sizing carefully or use IPv6 mode.
- **EKS control plane upgrades are managed; node upgrades are your responsibility** — always test on a non-production cluster first.
- **Reserved Concurrency limits a function's maximum concurrency** — it both guarantees and caps capacity.

---

## 13. Further Reading

For deeper coverage, CLI commands, and exam-specific trivia, see the detailed reference:

- **EC2 fundamentals, instance types, security groups**: [`EC2-Basics-InstanceTypes-SecurityGroups.md`](../../detailed-reference/EC2-Basics-InstanceTypes-SecurityGroups.md)
- **Placement groups, ENI, hibernate**: [`EC2-Associate-PlacementGroups-ENI-Hibernate.md`](../../detailed-reference/EC2-Associate-PlacementGroups-ENI-Hibernate.md)
- **ELB, ASG, HA, scalability patterns**: [`HA-Scalability-ELB-ASG.md`](../../detailed-reference/HA-Scalability-ELB-ASG.md)
- **Lambda, concurrency, cold starts, RDS Proxy**: [`Serverless-Lambda-EdgeFunctions.md`](../../detailed-reference/Serverless-Lambda-EdgeFunctions.md)
- **ECS, EKS, Fargate, ECR, container orchestration**: [`Containers-ECS-EKS-ECR-Fargate.md`](../../detailed-reference/Containers-ECS-EKS-ECR-Fargate.md)

---

*Section 05 — Compute & Load Balancing | Last Validated: 2026-05-10*
