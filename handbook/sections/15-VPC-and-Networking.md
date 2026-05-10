# Section 15 — VPC & Networking

> **Purpose**: Networking is the foundation upon which all AWS architecture rests. A misconfigured route table, an overly permissive security group, or a missing VPC endpoint can render an otherwise perfect application unreachable or insecure. This section covers VPC design, subnetting, routing, hybrid connectivity, and DNS with the precision needed for architecture reviews and production troubleshooting.
>
> **Official Documentation**: [VPC](https://docs.aws.amazon.com/vpc/) | [Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/) | [Direct Connect](https://docs.aws.amazon.com/directconnect/) | [Route53](https://docs.aws.amazon.com/route53/)

---

## 1. VPC Fundamentals

### 1.1 What a VPC Actually Is

A VPC is a **logically isolated network** within an AWS region. It is not a physical network — it is software-defined networking (SDN) implemented by the AWS Nitro hypervisor and custom network hardware.

**Key properties**:
- A VPC spans **all Availability Zones** in a region
- Subnets are **AZ-specific** — a subnet cannot span multiple AZs
- The default VPC is pre-configured with public subnets, an Internet Gateway, and DNS resolution
- Custom VPCs start with nothing — you must explicitly configure every component

### 1.2 CIDR Planning

CIDR block selection is a long-term decision. You cannot change a VPC's primary CIDR without significant disruption.

| VPC CIDR | Total IPs | Typical Use |
|----------|-----------|-------------|
| `10.0.0.0/16` | 65,536 | Small-medium organization |
| `10.0.0.0/12` | 1,048,576 | Large organization |
| `172.16.0.0/12` | 1,048,576 | When 10.x.x.x conflicts with on-prem |
| `192.168.0.0/16` | 65,536 | When both 10.x and 172.x conflict |

**Subnet reservation per subnet**:
- Network address (first IP)
- VPC router (second IP — `10.0.1.1`)
- DNS server (third IP — `10.0.1.2`)
- Future use (fourth IP — `10.0.1.3`)
- Network broadcast (last IP — `10.0.1.255`)

**Usable IPs = 2^(32-prefix) - 5**

A `/28` subnet has 16 IPs, 11 usable. A `/24` has 256 IPs, 251 usable.

> **Exam/Interview Trap**: "How many EC2 instances in a /24 subnet?" 251 — NOT 256. The 5 reserved IPs plus the network and broadcast addresses reduce the count.

### 1.3 Public vs Private vs Isolated Subnets

The distinction is purely about **routing**, not an inherent subnet property:

| Subnet Type | Route Table | Can reach internet? | Can be reached from internet? |
|-------------|-------------|---------------------|------------------------------|
| **Public** | `0.0.0.0/0 → IGW` | Yes (if instance has public/elastic IP) | Yes (if instance has public/elastic IP and security group allows) |
| **Private** | `0.0.0.0/0 → NAT Gateway` | Yes (outbound only) | No (no direct internet route) |
| **Isolated** | No `0.0.0.0/0` route | No | No |

```
Internet
    │
    ▼
┌──────────┐
│   IGW    │──► Public Subnet (ALB, Bastion, NAT GW)
└──────────┘
    │
    ▼
┌──────────┐
│ NAT GW   │──► Private Subnet (App servers, workers)
└──────────┘
    │
    ▼
Isolated Subnet (Databases, no internet at all)
```

> **NAT Gateway Behavior**: NAT Gateway allows **outbound** connections from private subnets. It does NOT allow inbound connections from the internet. Responses to outbound connections are allowed because NAT is stateful — return traffic is matched to the original outbound flow.

---

## 2. VPC Routing Deep Dive

### 2.1 Route Tables and Route Evaluation

A route table is a **longest-prefix match** routing table:

```
Destination          Target
─────────────────────────────
10.0.0.0/16        local        (VPC local traffic — implicit, cannot be deleted)
10.0.1.0/24        local        (same AZ traffic)
0.0.0.0/0          nat-xxx      (default route to NAT Gateway)
172.16.0.0/12      pcx-xxx      (VPC Peering connection)
192.168.0.0/16     tgw-xxx      (Transit Gateway)
```

**Route evaluation order**:
1. Most specific prefix match wins
2. `local` routes (VPC CIDR) are always most specific for intra-VPC traffic
3. If no match, packet is dropped (blackholed)

> **Common routing error**: A VPC peering route `172.16.0.0/12` conflicts with a subnet's local route if the peered VPC CIDR overlaps. **VPC peering requires non-overlapping CIDRs.**

### 2.2 NAT Gateway Semantics

**NAT Gateway characteristics**:
- **AZ-specific**: Created in a specific subnet (must be public). Provides NAT for that AZ only.
- **Highly available**: Managed by AWS within the AZ. Not a single instance.
- **Outbound only**: No inbound internet connectivity. Return traffic is allowed because NAT Gateway is stateful.
- **Bandwidth**: Up to 45 Gbps per NAT Gateway. Can create multiple NAT Gateways for higher bandwidth (per AZ).
- **Elastic IP**: NAT Gateway requires one Elastic IP per gateway.
- **Cost**: $0.045/hour per gateway + $0.045/GB data processing.

**HA NAT pattern**: One NAT Gateway per AZ. Route tables in private subnets route `0.0.0.0/0` to their AZ's NAT Gateway. If one NAT Gateway fails, only that AZ loses outbound internet — other AZs unaffected.

```
AZ-a Public Subnet: NAT GW-a ──► Private Subnet-a RT: 0.0.0.0/0 → NAT GW-a
AZ-b Public Subnet: NAT GW-b ──► Private Subnet-b RT: 0.0.0.0/0 → NAT GW-b
```

> **NAT Gateway vs NAT Instance**: NAT Instances (self-managed EC2) are legacy. NAT Gateway is managed, higher bandwidth, HA, and simpler. Only use NAT Instance if you need custom routing, logging, or third-party firewall integration.

---

## 3. Security Groups and Network ACLs

### 3.1 Security Groups: Stateful Firewalls

Security Groups are **stateful** — if inbound traffic is allowed, the return traffic is automatically allowed, regardless of outbound rules.

**Key properties**:
- **Instance-level** (attach to ENI, not subnet)
- **Stateful**: Return traffic automatically permitted
- **Only ALLOW rules** — no explicit DENY
- **Evaluate as a union**: All rules are evaluated; if any rule allows, traffic is allowed
- **Can reference other security groups** (dynamic, IP-independent rules)

```json
// Example: App SG allows inbound from ALB SG only
{
  "Type": "ingress",
  "Protocol": "tcp",
  "FromPort": 8080,
  "ToPort": 8080,
  "SourceSecurityGroupId": "sg-alb"
}
```

> **Cross-SG Reference**: Referencing another SG is more maintainable than IP addresses. When the ALB scales to new instances (different IPs), the App SG automatically permits them because the ALB SG membership is what matters.

### 3.2 Network ACLs: Stateless Subnet Filters

NACLs are **stateless** — you must explicitly allow BOTH inbound and outbound traffic directions.

**Key properties**:
- **Subnet-level** (applies to all instances in the subnet)
- **Stateless**: Return traffic must be explicitly allowed by a matching outbound rule
- **ALLOW and DENY rules** — can explicitly block
- **Rule order matters**: Rules evaluated in number order; first match wins
- **Default**: Allow all inbound/outbound

**Ephemeral port requirement**: When a client connects to a server's port 80, the server responds from an ephemeral port (1024-65535). With stateless NACLs, you must allow outbound ephemeral ports.

```
Inbound: Allow source ANY to destination port 80
Outbound: Allow source port 80 to destination ephemeral ports (1024-65535)
```

> **SG vs NACL Decision Framework**: Use Security Groups for almost everything. Use NACLs only for:
> 1. Explicit subnet-level blocking (e.g., block a known bad IP range)
> 2. Defense in depth (block SSH at subnet level even if SG allows it)
> 3. Compliance requirements for layered security controls

### 3.3 VPC Flow Logs

VPC Flow Logs capture IP traffic metadata:
- **Source/destination IP and port**
- **Protocol, packets, bytes**
- **Action** (ACCEPT or REJECT — but note: NACL REJECT is logged, SG implicit deny is NOT logged because SG is stateful and the return packet is what gets evaluated)

**Important limitation**: Flow Logs do NOT log:
- Traffic to/from the VPC router (169.254.169.254 — instance metadata)
- DNS traffic to Amazon DNS (Route53 Resolver)
- Traffic within the same subnet that doesn't traverse the VPC router

**Destination options**: CloudWatch Logs (real-time, small volumes), S3 (long-term, large volumes), Kinesis Data Firehose (streaming analytics).

---

## 4. VPC Connectivity Options

### 4.1 VPC Peering

VPC Peering connects two VPCs with full mesh routing between them.

**Constraints**:
- **Non-transitive**: If VPC A peers with B, and B peers with C, A cannot reach C through B. You need a separate peering between A and C, or use Transit Gateway.
- **No overlapping CIDRs**: Peered VPCs must have non-overlapping CIDR blocks.
- **Cross-region supported**: But adds latency and data transfer costs.
- **Cross-account supported**: Requires acceptance from the peer VPC owner.

**Route table requirement**: Each VPC must add a route for the peer VPC's CIDR pointing to the peering connection (`pcx-xxx`).

> **Peering is NOT transitive** — this is the most tested VPC networking concept. For transitive routing (hub-and-spoke), use Transit Gateway.

### 4.2 VPC Endpoints

VPC Endpoints allow private access to AWS services without traversing the public internet.

| Type | Services | Cost | Characteristics |
|------|----------|------|-----------------|
| **Gateway Endpoint** | S3, DynamoDB | Free | Route table entry. Uses AWS private network. No security group. |
| **Interface Endpoint** | Most AWS services (EC2 API, SNS, SQS, etc.) | $0.01/hour per AZ + data transfer | ENI in your subnet. Security group controlled. DNS resolution. |

**Gateway Endpoint architecture**:
```
Private Subnet ──► Route Table: 0.0.0.0/0 → NAT GW
                     pl-xxx (S3 prefix) → vpce-s3 (Gateway Endpoint)
```

Traffic to S3 (`s3.us-east-1.amazonaws.com`) matching the prefix list routes through the Gateway Endpoint instead of NAT Gateway. This is free and reduces NAT Gateway data processing charges.

**Interface Endpoint architecture**:
```
Private Subnet ──► ENI (vpce-xxxx) ──► AWS Service
                     │
                     └── Security Group controls access
                     └── Private DNS: sqs.us-east-1.vpce.amazonaws.com
```

Interface endpoints create ENIs in your subnets. You control access via security groups. They support most AWS services.

> **Endpoint Policy**: Both Gateway and Interface endpoints support policies restricting which buckets/queues can be accessed. Use this for defense in depth — even if IAM allows S3 access, the endpoint policy can restrict which buckets are reachable from the VPC.

### 4.3 Transit Gateway

Transit Gateway (TGW) is a **centralized hub** for VPC connectivity.

**Architecture**:
```
                          ┌─────────────┐
                          │             │
        VPC-A ───────────►│   Transit   │◄─────────── VPC-B
        (10.1.0.0/16)     │  Gateway    │            (10.2.0.0/16)
                          │             │
        VPC-C ───────────►│             │◄─────────── On-Prem
        (10.3.0.0/16)     │             │   (DX/VPN)
                          └─────────────┘
```

**Transit Gateway features**:
- **Transitive routing**: VPC A can reach VPC B, C, and on-prem through TGW
- **Route tables per attachment**: Control which VPCs can talk to which others (segmentation)
- **Cross-region peering**: TGWs in different regions can peer
- **ECMP**: Equal-cost multi-path routing for multiple VPN/Direct Connect attachments (load balance across tunnels)
- **Scale**: Up to 5,000 VPC attachments per TGW

**TGW Route Tables**: You can create multiple route tables in a TGW and associate different attachments with different tables. This enables network segmentation:

```
TGW Route Table: "Production"
├── Attachment: VPC-Prod-A (10.1.0.0/16)
├── Attachment: VPC-Prod-B (10.2.0.0/16)
└── Attachment: DX-Gateway (192.168.0.0/16)

TGW Route Table: "Development"
├── Attachment: VPC-Dev-A (10.10.0.0/16)
└── Attachment: VPC-Dev-B (10.11.0.0/16)

TGW Route Table: "Shared Services"
├── Attachment: VPC-Shared (10.100.0.0/16)
├── Propagated: All VPC CIDRs (shared services need to reach everything)
```

> **Cost**: TGW charges per attachment per hour ($0.05/attachment/hour) + per GB processed ($0.02/GB). For 50 VPCs, this is $1,800/month in attachment fees alone. Plan TGW usage carefully.

### 4.4 Site-to-Site VPN

**Architecture**:
```
On-Prem Router ──► Internet ──► [Customer Gateway] ◄──► [Virtual Private Gateway] ──► VPC
                                    (your device)          (AWS managed)
                                    IPsec tunnel
```

**VPN Characteristics**:
- **Quick to establish**: Hours vs weeks for Direct Connect
- **Internet-based**: Traffic traverses public internet (encrypted via IPsec)
- **Throughput**: Up to 1.25 Gbps per tunnel (two tunnels per connection for HA)
- **Latency**: Variable (internet-dependent)
- **Cost**: $0.05/hour per connection + data transfer out

**HA Pattern**: Two tunnels to two different Virtual Private Gateway endpoints in different AZs. On-prem router uses BGP for dynamic routing and automatic failover.

### 4.5 AWS Direct Connect

Direct Connect provides a **dedicated physical connection** from your on-premises data center to AWS.

| Feature | Direct Connect | Site-to-Site VPN |
|---------|---------------|------------------|
| **Setup time** | Weeks to months | Hours |
| **Bandwidth** | 1 Gbps to 100 Gbps | Up to 1.25 Gbps per tunnel |
| **Latency** | Consistent, low | Variable |
| **Cost** | Port hourly + per-GB outbound | Hourly tunnel + data transfer |
| **Route** | Private (not internet) | Public internet (IPsec encrypted) |
| **Resilience** | SLA available | No SLA |

**Direct Connect Components**:
- **DX Location**: AWS facility where you connect. You can use a partner (APN) if you don't have equipment at the DX location.
- **Virtual Interface (VIF)**: Private VIF (connects to one VPC via VGW) or Public VIF (connects to AWS public services like S3, DynamoDB)
- **Transit VIF**: Connects to Transit Gateway (preferred for multi-VPC)
- **DX Gateway**: Enables cross-region connectivity. Connect DX to DX Gateway, which connects to VPCs in multiple regions.

**DX + VPN Backup Pattern**:
```
Primary:   Direct Connect (1-10 Gbps, consistent, low latency)
Backup:    Site-to-Site VPN (automatic failover if DX fails)
           Both connect to Transit Gateway for unified routing
```

> **DX Resilience**: For critical workloads, use two DX connections to two different DX locations (diverse physical paths). A single DX location is a single point of failure.

---

## 5. AWS PrivateLink

PrivateLink provides **private connectivity to AWS services and SaaS applications** without traversing the public internet.

**How it works**:
1. Service provider creates a **VPC Endpoint Service** (powered by NLB)
2. Consumer creates a **VPC Endpoint** (Interface type) in their VPC
3. Traffic flows privately over AWS's network between consumer VPC and provider VPC

**Use cases**:
- Accessing AWS services from on-prem without public internet
- Third-party SaaS providers offering private connectivity to their APIs
- Cross-account service access without peering or Transit Gateway

> **PrivateLink vs VPC Peering**: PrivateLink is unidirectional (consumer → provider). The provider cannot initiate connections to the consumer. Peering is bidirectional. PrivateLink is simpler for service consumption; peering is better for full network integration.

---

## 6. Route 53: DNS Architecture

### 6.1 Route 53 Is Global

Route 53 is a **global service**, not regional. DNS records are propagated to edge locations worldwide. Do NOT draw Route 53 inside a VPC — it exists outside the VPC boundary.

### 6.2 Routing Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Simple** | Single record, no health checks | Static resources, single endpoint |
| **Failover** | Primary + Secondary. Health check on primary; route to secondary if primary fails | DR patterns |
| **Geolocation** | Route based on user's geographic location | Compliance (EU data stays in EU), localized content |
| **Geoproximity** | Route based on geographic proximity with optional bias | Direct users to nearest healthy endpoint |
| **Latency** | Route to region with lowest latency for the user | Multi-region active-active applications |
| **Weighted** | Distribute traffic by percentage | Canary deployments, A/B testing |
| **Round Robin** | Rotate through multiple records | Simple load distribution |
| **Multivalue Answer** | Return up to 8 healthy records | Client-side load balancing |

### 6.3 Alias Records vs CNAME

**Alias records** are Route 53-specific:
- Map to AWS resources (ALB, CloudFront, S3 website, API Gateway)
- **Free**: No additional Route 53 charges for Alias queries
- **APEX support**: Can be used at the zone apex (`example.com`), unlike CNAME
- **Health check integration**: Automatically reflects target health

**CNAME**:
- Maps to any DNS name
- **Not free**: Standard Route 53 query charges apply
- **Cannot be used at zone apex**

> **Always use Alias records for AWS resources** — they are free, support apex domains, and integrate with health checks.

### 6.4 Health Checks

Route 53 health checks monitor endpoints and influence routing decisions:
- **Endpoint checks**: HTTP/HTTPS/TCP checks against an IP or domain
- **Calculated checks**: Combine multiple checks with AND/OR logic
- **CloudWatch alarm checks**: Route based on CloudWatch metrics (e.g., ALB 5xx rate)
- **Recovery time**: Health check failure must persist for the configured interval + failure threshold (e.g., 30-second interval × 3 failures = 90 seconds to failover)

---

## 7. Interview Challenges

### Q1: "A private subnet instance cannot reach S3. Security groups and NACLs look correct. What's wrong?"

**Answer**: Check the route table. Private subnets need either:
1. A VPC Gateway Endpoint for S3 (preferred — free, private, no NAT needed)
2. A route to a NAT Gateway (costs money, less efficient)

If the route table has no route to S3 and no NAT Gateway route, S3 traffic has no path. Gateway Endpoints are the correct solution for S3 and DynamoDB access from private subnets.

### Q2: "Design VPC networking for a 3-tier application across 3 AZs with on-prem connectivity."

**Answer**:
```
VPC: 10.0.0.0/16
├── AZ-a
│   ├── Public:  10.0.1.0/24  (ALB, NAT GW-a)
│   ├── Private: 10.0.2.0/24  (App tier)
│   └── Data:    10.0.3.0/24  (RDS, ElastiCache)
├── AZ-b
│   ├── Public:  10.0.4.0/24  (ALB, NAT GW-b)
│   ├── Private: 10.0.5.0/24  (App tier)
│   └── Data:    10.0.6.0/24  (RDS, ElastiCache)
├── AZ-c
│   ├── Public:  10.0.7.0/24  (ALB, NAT GW-c)
│   ├── Private: 10.0.8.0/24  (App tier)
│   └── Data:    10.0.9.0/24  (RDS, ElastiCache)
└── Transit Gateway Attachment
    └── Routes: 192.168.0.0/16 (on-prem) via DX
```

Security:
- ALB SG: inbound 443 from `0.0.0.0/0`
- App SG: inbound 8080 from ALB SG only
- Data SG: inbound 3306/6379 from App SG only
- NACL: allow ephemeral ports outbound, specific ports inbound

### Q3: "VPC Peering vs Transit Gateway for 20 VPCs that all need to communicate."

**Answer**: Transit Gateway. Peering requires n(n-1)/2 connections = 190 peering relationships for 20 VPCs. Transit Gateway requires 20 attachments + TGW route table propagation. TGW also provides:
- Centralized routing control
- Network segmentation via multiple route tables
- Cross-region peering
- Better visibility (CloudWatch metrics per attachment)

The only reason to use peering is if TGW cost ($0.05/hour per attachment) is prohibitive for a very large number of VPCs with minimal traffic. Even then, TGW usually wins on operational simplicity.

### Q4: "A security group allows inbound SSH from 0.0.0.0/0. A developer cannot SSH to the instance. Why?"

**Answer**: Check these in order:
1. Is the instance in a public subnet with a route to IGW?
2. Does the instance have a public IP or Elastic IP?
3. Is the NACL blocking inbound port 22 or outbound ephemeral ports?
4. Is there a host-level firewall (iptables, Windows Firewall) blocking SSH?
5. Is the SSH key correct and the sshd service running?

Security group rules are necessary but not sufficient. Routing, IP assignment, and host-level configuration must also be correct.

---

## 8. Points to Remember

- **VPC spans all AZs; subnets are AZ-specific** — a subnet cannot span AZs.
- **5 IPs are reserved per subnet** — usable IPs = total - 5.
- **Public vs Private is determined by route table**, not a subnet property.
- **NAT Gateway allows outbound-only internet access** — it is stateful and does not allow unsolicited inbound connections.
- **Security Groups are stateful** — return traffic is automatic. NACLs are stateless — both directions must be explicitly allowed.
- **NACLs require ephemeral port rules** for return traffic (1024-65535).
- **VPC Peering is non-transitive** — A→B and B→C does not give A→C. Use Transit Gateway for transitive routing.
- **VPC Peering requires non-overlapping CIDRs** — plan CIDRs before creating VPCs.
- **Gateway Endpoints are free** for S3 and DynamoDB. Interface Endpoints cost $0.01/hour per AZ.
- **Transit Gateway enables hub-and-spoke networking** — up to 5,000 VPC attachments.
- **Site-to-Site VPN is quick but uses the public internet** — use Direct Connect for consistent, low-latency, high-bandwidth connectivity.
- **Direct Connect + VPN backup** is the standard resilient hybrid pattern.
- **Route 53 is a global service** — not inside any VPC. Alias records are free and support apex domains.
- **Route 53 health checks influence routing** — failover routing requires health checks on the primary endpoint.
- **VPC Flow Logs do NOT log security group implicit deny** — they log NACL REJECTs and ACCEPTs.
- **PrivateLink provides unidirectional private service access** — simpler than peering for service consumption patterns.

---

## 9. Further Reading

For deeper coverage, CLI commands, and exam-specific trivia, see the original notes:

- **VPC deep dive, subnets, NAT, NACLs, peering, VPN, DX, Transit Gateway**: [`VPC-Networking-DeepDive.md`](../../original-notes/VPC-Networking-DeepDive.md)
- **Route53 DNS, routing policies, health checks**: [`Route53-DNS-RoutingPolicies.md`](../../original-notes/Route53-DNS-RoutingPolicies.md)
- **Advanced VPC (SAP level)**: [`SAP-15-VPC.md`](../../original-notes/SAP-15-VPC.md)

---

*Section 15 — VPC & Networking | Last Validated: 2026-05-10*
