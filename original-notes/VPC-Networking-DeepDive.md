# AWS SAA-C03: VPC & Networking Deep Dive

## Table of Contents
1. [VPC Fundamentals](#1-vpc-fundamentals)
2. [Subnets](#2-subnets)
3. [Internet Gateway (IGW)](#3-internet-gateway-igw)
4. [Route Tables](#4-route-tables)
5. [NAT Gateway](#5-nat-gateway)
6. [Network ACLs (NACLs)](#6-network-acls-nacls)
7. [Security Groups](#7-security-groups)
8. [VPC Peering](#8-vpc-peering)
9. [VPC Endpoints](#9-vpc-endpoints)
10. [VPC Flow Logs](#10-vpc-flow-logs)
11. [Bastion Hosts](#11-bastion-hosts)
12. [Site-to-Site VPN](#12-site-to-site-vpn)
13. [AWS Direct Connect (DX)](#13-aws-direct-connect-dx)
14. [Transit Gateway](#14-transit-gateway)
15. [VPC Traffic Mirroring](#15-vpc-traffic-mirroring)
16. [IPv6 in VPC](#16-ipv6-in-vpc)
17. [AWS PrivateLink](#17-aws-privatelink)
18. [Points to Remember (Exam Focus)](#points-to-remember-exam-focus)
19. [AWS CLI Quick Reference](#aws-cli-quick-reference)

---

## 1. VPC Fundamentals

### What is a VPC?

A **Virtual Private Cloud (VPC)** is your own logically isolated network inside AWS. Think of it like this: Instead of sharing a public building with other companies, you rent an entire private floor in AWS's data center. You decide the layout, security, access points—everything.

**Key Facts:**
- One VPC per region by default (soft limit, can request increase)
- A VPC spans ALL Availability Zones in that region
- Default VPC: automatically created with a /16 CIDR (65,536 IPs), public subnets in each AZ, IGW attached, DNS enabled
- Custom VPC: you start from scratch—no subnets, no routing, no internet access until you configure it

### CIDR Notation & IP Ranges

**Why CIDR matters?** The `/XX` notation tells you how many IP addresses you get:

| CIDR | IPs | Use Case |
|------|-----|----------|
| `/16` | 65,536 | Entire VPC |
| `/24` | 256 | Standard subnet |
| `/28` | 16 | **Exam trick:** Only 11 usable (AWS reserves 5) |

**The 5 Reserved IPs per Subnet:**
```
10.0.1.0   → Network address (AWS reserves)
10.0.1.1   → VPC router (AWS reserves)
10.0.1.2   → DNS server (AWS reserves)
10.0.1.3   → Future use (AWS reserves)
10.0.1.255 → Broadcast address (AWS reserves)
```

> **Tutor Tip:** On the exam, if they give you a /28 subnet and ask "how many instances can you launch?", the answer is 11, not 16. This is a classic exam gotcha.

### VPC Settings

**Tenancy Options:**
- `default`: Shared hardware with other AWS customers (cheaper, default)
- `dedicated`: Your instances run on dedicated physical hardware (expensive, ~50% more cost)

> **When to use dedicated tenancy?** Only when you have strict compliance requirements (HIPAA, PCI-DSS). For 99% of workloads, stay with default.

### VPC Peering Connection Analogy

Imagine your company has two floors in the building. VPC Peering is like opening a private corridor between floors so employees can walk directly from Floor A to Floor B without using the public elevator (internet).

---

## 2. Subnets

### Subnet Basics

A **subnet** is a range of IP addresses within ONE Availability Zone. This is critical: **subnets CANNOT span multiple AZs**. If you need resources in two AZs, you need two subnets.

**Visualization:**
```
VPC: 10.0.0.0/16 (spans all AZs)
├── Subnet A: 10.0.1.0/24 (AZ-A, 256 IPs)
├── Subnet B: 10.0.2.0/24 (AZ-B, 256 IPs)
└── Subnet C: 10.0.3.0/24 (AZ-A, 256 IPs) ← Another subnet in same AZ is OK
```

### Public vs Private vs Isolated Subnets

**Public Subnet:**
- Has a route table entry: `0.0.0.0/0 → igw-xxx` (Internet Gateway)
- Instances with public IPs can reach internet
- Example: web servers, load balancers, NAT Gateways

**Private Subnet:**
- Has a route table entry: `0.0.0.0/0 → nat-xxx` (NAT Gateway in public subnet)
- Instances can reach internet OUTBOUND ONLY
- Inbound internet requests are blocked
- Example: application servers, databases, workers

**Isolated Subnet:**
- No route to internet (route table has only local routes)
- Cannot reach internet at all
- Used for: databases, sensitive data storage

> **Exam Alert:** You CANNOT connect a private subnet directly to an IGW. The instance won't have a public IP, so it's useless. Use NAT Gateway instead.

### Auto-Assign Public IPv4

Each subnet has a setting: "Auto-assign Public IPv4 address". If enabled, every new EC2 instance automatically gets a public IP (at no extra cost, but consumes your subnet's IPs). Default VPC has this enabled; custom VPCs have it disabled.

---

## 3. Internet Gateway (IGW)

### What Does an IGW Do?

An **Internet Gateway** is the door between your VPC and the internet. It performs two functions:

1. **Allows communication** between VPC instances and internet
2. **Performs 1:1 NAT** for instances with public IPs (translates private IP → public IP)

### Critical Insight: Attachment + Routes

Here's what confuses everyone: **Attaching an IGW is NOT enough**. Two steps required:

```
Step 1: Create and attach IGW to VPC
Step 2: Update route table: 0.0.0.0/0 → igw-xxx
```

Without Step 2, the IGW is attached but invisible to your instances. It's like having a door to the street but no sign pointing to it.

### IGW Characteristics

- **One IGW per VPC**: You cannot attach two IGWs to the same VPC
- **Highly Available**: Horizontally scaled, redundant, AWS manages it
- **Bandwidth**: Unlimited, no throttling
- **State**: Must be in "available" state to work

> **Tutor Tip:** On practice exams, if you see "instances can't reach internet but IGW is attached," the answer is almost always "route table missing the 0.0.0.0/0 → IGW route."

---

## 4. Route Tables

### Understanding Route Tables

A **route table** is a set of rules (called routes) that determine where network traffic from your subnet or instance is directed. Every subnet must have one (default is main route table).

**Route Table Structure:**
```
Destination       Target
10.0.0.0/16       local         ← AWS adds this automatically (can't remove)
0.0.0.0/0         igw-xxxxx     ← For internet access
172.31.0.0/16     pcx-xxxxx     ← For VPC peering
```

### Most Specific Route Wins (Longest Prefix Match)

If your packet matches multiple routes, AWS uses the MOST SPECIFIC (highest /XX number):

```
Instance tries to reach 8.8.8.8

Route Table:
0.0.0.0/0         → IGW           (matches, /0)
8.8.8.0/24        → VPN           (matches, /24) ← This wins! More specific
8.8.8.8/32        → VPN           (hypothetical /32, even more specific)
```

> **Exam Question Type:** "Traffic to 192.168.1.0/24 should go via VPN. What route do you add?" Answer: Add route `192.168.1.0/24 → VPN` with higher specificity than any default routes.

### Main Route Table vs Custom Route Tables

- **Main Route Table**: Automatically created with VPC, auto-associated to subnets that don't have explicit associations
- **Custom Route Table**: You create and explicitly associate to subnets
- Best Practice: Create separate custom route tables for public and private subnets (easier to manage)

---

## 5. NAT Gateway

### The NAT Gateway Problem & Solution

**Problem:** Your EC2 instance in a private subnet needs to download security updates from the internet, but you can't expose it with a public IP.

**Solution:** Use a **NAT Gateway**. It's like a mail filter: your instance can send outbound requests, and responses come back, but unsolicited inbound traffic is blocked.

### How NAT Gateway Works

```
Private EC2 Instance (10.0.2.10)
         ↓ (outbound traffic: 10.0.2.10:12345 → 1.2.3.4:443)
    NAT Gateway (public IP: 54.1.2.3)
         ↓ (translates to: 54.1.2.3:54321 → 1.2.3.4:443)
    Internet
         ↓ (response: 1.2.3.4:443 → 54.1.2.3:54321)
    NAT Gateway (translates back)
         ↓ (translates to: 1.2.3.4:443 → 10.0.2.10:12345)
Private EC2 Instance receives response
```

### NAT Gateway Requirements

**Location:** MUST be in a PUBLIC subnet (it needs a public IP)

**Elastic IP:** MUST have an Elastic IP attached (persistent public IP)

**Route Table Update:** Private subnet's route table needs:
```
0.0.0.0/0 → nat-xxxxx (not igw)
```

### Scaling & Availability

- **Bandwidth**: 5 Gbps per NAT Gateway, auto-scales to 100 Gbps
- **AZ-Scoped**: One NAT Gateway serves only instances in its AZ
- **High Availability Strategy**: One NAT Gateway PER AZ (if AZ-A NAT fails, AZ-B instances lose internet)

### Cost Considerations

NAT Gateway is NOT free:
- **Hourly cost**: ~$0.045/hour per NAT Gateway (~$32/month)
- **Data processing**: ~$0.045/GB

> **Cost Optimization Tip:** If you have only a dev VPC with light traffic, consider a NAT Instance (EC2) instead. It's cheaper but requires manual configuration and doesn't auto-scale.

### NAT Gateway vs NAT Instance Comparison

| Feature | NAT Gateway | NAT Instance |
|---------|-------------|--------------|
| Managed Service | Yes | No (EC2) |
| Auto-Scale | Yes | No (manual) |
| Security Group | No | Yes (configure) |
| Bandwidth | Up to 100 Gbps | Depends on instance type |
| Cost | $$$$ | $$ |
| Can be Bastion | No | Yes |
| Source/Dest Check | N/A | Must disable |

> **Exam Secret:** If they ask "how to SSH into private instances," the answer is bastion host (public subnet), not NAT Gateway. NAT Gateways are one-way (outbound only).

---

## 6. Network ACLs (NACLs)

### What is a NACL?

A **Network Access Control List (NACL)** is a stateless firewall at the SUBNET level. Think of it as the security gate at your building's entrance—everyone entering or leaving must pass through.

**Key: STATELESS** means return traffic MUST be explicitly allowed. If you allow inbound port 80, you must also allow outbound ephemeral ports.

### NACL Rules

Rules are numbered 1-32766 and evaluated in order (lowest first):
- First matching rule wins (stop evaluating)
- Default NACL: Rule #100 allows ALL traffic (both inbound and outbound)
- Custom NACL: Rule #100 denies everything by default (must add allow rules)

**Example Rule Structure:**
```
Rule #  Type       Protocol Port Range Source      Action
100     Inbound    TCP      80         0.0.0.0/0   Allow
110     Inbound    TCP      443        0.0.0.0/0   Allow
120     Inbound    TCP      22         10.0.0.0/8  Allow
130     Inbound    TCP      1024-65535 0.0.0.0/0   Allow ← Ephemeral!
```

### Stateful vs Stateless (Critical Concept)

**Stateful (Security Groups):**
```
Client → Server (port 80): inbound rule allows
Server → Client (port 54321): automatically allowed (return traffic)
```

**Stateless (NACLs):**
```
Client → Server (port 80): inbound rule allows
Server → Client (port 54321): NEED explicit outbound rule (1024-65535)
```

### Ephemeral Ports

When your EC2 instance makes an outbound HTTP request:
```
Source: 10.0.1.10:12345 (random high port)
Destination: 1.2.3.4:80

Response comes back:
Source: 1.2.3.4:80
Destination: 10.0.1.10:12345

Your NACL outbound rules MUST allow 1024-65535 range!
```

> **Tutor Tip:** Ephemeral ports always mess up NACL exam questions. Remember: **inbound traffic uses destination ports (80, 443), return traffic uses source ports (1024-65535)**.

### When to Use NACLs

**Use NACL when you need to:** DENY specific IP addresses (Security Groups can only allow)

```
Example: A hacker's IP 203.0.113.45 keeps scanning your VPC
NACL Rule #100: Inbound, All Protocol, All Ports, 203.0.113.45 → DENY
NACL Rule #110: Inbound, All Protocol, All Ports, 0.0.0.0/0 → ALLOW
```

Security Group can't do this (no DENY rules).

---

## 7. Security Groups

### Stateful Firewall at Instance Level

A **Security Group (SG)** is a stateful firewall that controls traffic at the **ENI (Elastic Network Interface)** level. Multiple instances can share the same SG.

**Key: STATEFUL** means if you allow inbound traffic, return traffic is automatically allowed without any outbound rule.

### Inbound vs Outbound Rules

**Default SG (any VPC):**
- Inbound: Allow from same SG
- Outbound: Allow all

**Example Custom SG (Web Server):**
```
Inbound Rules:
- HTTP from 0.0.0.0/0 (port 80)
- HTTPS from 0.0.0.0/0 (port 443)
- SSH from 10.0.0.0/16 (port 22, only from VPC)

Outbound Rules:
- All traffic (default)
```

### Referencing Other Security Groups

This is powerful: instead of specifying CIDR ranges, reference another SG:

```
DB Security Group Inbound Rule:
- MySQL (port 3306) from app-sg

This means: Allow traffic from ANY instance with app-sg attached
```

Why is this useful? Instances can join/leave app-sg dynamically, and DB access automatically grants/revokes without touching NACL or route table rules.

> **Exam Alert:** You can reference SGs from SAME REGION ONLY. Cross-region SG references don't work. Use CIDR ranges instead.

### Multiple Security Groups

An EC2 instance can have UP TO 5 security groups attached. All rules from all SGs are COMBINED (union, not intersection).

```
Instance with sg-web and sg-app:
- SG-web allows port 80 from 0.0.0.0/0
- SG-app allows port 3000 from 10.0.0.0/16

Result: Instance allows BOTH 80 (from anywhere) and 3000 (from VPC)
```

### Security Groups Applied To

- EC2 instances
- ELB (load balancers)
- RDS databases
- Lambda functions (in VPC)
- ECS tasks

---

## 8. VPC Peering

### Direct Private Connection Between VPCs

**VPC Peering** creates a private, encrypted connection between two VPCs using AWS backbone (not the internet). Like the private corridor between office floors.

**Supports:**
- Cross-account peering (even different AWS accounts)
- Cross-region peering (but with slight latency)
- Same-region peering (usually preferred)

### Critical Limitation: NOT Transitive

This trips up everyone:

```
VPC-A ←peering→ VPC-B ←peering→ VPC-C

Result: A and C CANNOT communicate directly!

You need: A ↔ B, A ↔ C, B ↔ C (mesh topology)
```

If you have 10 VPCs and need them all to communicate, VPC peering requires 45 connections. **Solution: Transit Gateway** (hub-and-spoke).

### Setup Steps

1. **Create Peering Connection**
   - VPC-A initiates request to VPC-B
   - Check: CIDR ranges don't overlap!

2. **Accept Peering Connection**
   - VPC-B account accepts request
   - Status: `pcx-xxxxx` (available)

3. **Update Route Tables (BOTH VPCs)**
   - VPC-A route table: `10.1.0.0/16 (VPC-B CIDR) → pcx-xxxxx`
   - VPC-B route table: `10.0.0.0/16 (VPC-A CIDR) → pcx-xxxxx`
   - If you skip this, peering is active but traffic doesn't flow!

4. **Update Security Groups (BOTH VPCs)**
   - Optional but recommended
   - SG-A allows traffic from SG-B (and vice versa)
   - In same region: can reference SG by ID; in different regions: must use CIDR

### Cross-Region Peering Costs

- **Same region**: Free
- **Cross-region**: Charged per GB transferred ($0.02/GB out in most cases)

---

## 9. VPC Endpoints

### Problem Solved by VPC Endpoints

**Without Endpoint:**
```
Private Instance → IGW → Internet → AWS API → Response
(exposed, uses internet bandwidth)
```

**With VPC Endpoint:**
```
Private Instance → VPC Endpoint → AWS Service (via AWS backbone)
(private, no internet exposure, cheaper data transfer)
```

### Two Types of Endpoints

#### Gateway Endpoint (Route Table-based)

**Available for:** S3 and DynamoDB ONLY

**How it works:**
- Creates a route table entry: `S3 CIDR → vpce-xxxxx`
- Traffic routes directly to S3/DynamoDB via AWS backbone
- FREE!
- No DNS changes needed (though you can customize)

**Example Use Case:**
```
Private EC2 Instance (10.0.2.10)
  ↓ (wants to access s3://my-bucket/file.txt)
Route Table: s3-endpoint → vpce-xxxxx (Gateway Endpoint)
  ↓
S3 (no internet traversal)
```

#### Interface Endpoint (PrivateLink-based)

**Available for:** All other AWS services (SQS, SNS, Secrets Manager, CloudWatch Logs, KMS, SSM, DynamoDB stream, etc.)

**How it works:**
- Creates an ENI (Elastic Network Interface) with private IP in your subnet
- Service DNS resolves to private IP (e.g., kms.us-east-1.amazonaws.com → 10.0.2.50)
- You access the service using existing SDK/CLI (no code changes)
- Costs: ~$0.01/hour/AZ + $0.01/GB data processed

**Example Use Case:**
```
Private Lambda function needs to decrypt data using KMS
├─ Without Endpoint: Lambda → IGW → Internet → KMS (exposed, slow)
└─ With Interface Endpoint: Lambda → ENI (private IP) → KMS (secure, fast)
```

### VPC Endpoint Policies

Both gateway and interface endpoints support **endpoint policies** (IAM-style):

```json
{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringEquals": {
          "aws:username": "production"
        }
      }
    }
  ]
}
```

> **Exam Question:** "Private Lambda can't access DynamoDB. What's missing?" Answer: Interface VPC Endpoint for DynamoDB + endpoint policy + route table entry.

### Gateway vs Interface Quick Reference

| Feature | Gateway | Interface |
|---------|---------|-----------|
| Services | S3, DynamoDB only | All others |
| Cost | FREE | $0.01/hour + $0.01/GB |
| DNS Changed | No (optional) | Yes |
| Route Table | Requires entry | Doesn't require |
| ENI Created | No | Yes |
| Source IP | Preserved | Via ENI IP |

---

## 10. VPC Flow Logs

### Capture Network Traffic Metadata

**VPC Flow Logs** capture metadata about IP traffic flowing through your VPC, subnet, or ENI. It's like a network packet recorder.

**Captured Data:**
```
source IP, dest IP, source port, dest port, protocol (6=TCP, 17=UDP),
action (ACCEPT or REJECT), bytes, packets, timestamp
```

**NOT Captured:**
- Instance metadata queries (169.254.169.254)
- Amazon DNS queries (reserved.amazonaws.com)
- DHCP traffic
- Windows license activation
- Link-local traffic

### Log Destinations

1. **CloudWatch Logs**: Real-time monitoring, Log Insights queries
2. **S3**: Long-term storage, Athena queries
3. **Kinesis Data Firehose**: Streaming to third-party analytics

### Log Delay

- **VPC or Subnet level**: ~10 minutes delay
- **ENI level**: ~1 minute delay

### Use Cases

**Troubleshooting Connectivity:**
```
Query Flow Logs: "Who is sending traffic to my private RDS on port 3306?"
→ Reveals unexpected connections, helps with security group debugging
```

**Security & Compliance:**
```
"Detect all traffic from suspicious IP 203.0.113.45"
→ Forensics, incident response
```

**Traffic Analysis:**
```
"Which subnets generate most data transfer?"
→ Optimization, capacity planning
```

### Format Examples

**Default Format:**
```
2 12345678 10.0.1.10 10.0.2.20 443 50123 6 1250 20000 1693920340 1693920400 ACCEPT OK
```

**Parseable:**
```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes windowstart windowend action log-status
```

---

## 11. Bastion Hosts

### Access Private Instances Securely

A **Bastion Host** (or Jump Host) is an EC2 instance in a PUBLIC subnet that acts as a secure gateway to instances in PRIVATE subnets.

**Use Case:**
```
Admin (corporate IP: 203.0.113.1)
  ↓ SSH to Bastion (port 22)
Bastion Host (public subnet, public IP: 54.1.2.3)
  ↓ SSH to Private Instance (port 22)
Private Instance (10.0.2.10)
```

### Bastion Host Security Configuration

**Bastion Host Security Group:**
```
Inbound:
- SSH (port 22) from corporate IP range (203.0.113.0/24)

Outbound:
- SSH (port 22) to private subnets (10.0.0.0/16)
```

**Private Instance Security Group:**
```
Inbound:
- SSH (port 22) from bastion-sg (not from 0.0.0.0/0!)

Outbound:
- Allow all (default)
```

### Bastion vs NAT Gateway

| Feature | Bastion | NAT Gateway |
|---------|---------|-------------|
| Type | EC2 instance | Managed service |
| Purpose | SSH access | Internet access |
| Inbound | Allows SSH from corp | Denies unsolicited |
| Outbound | Allows outbound SSH | Allows all outbound |
| Cost | EC2 cost | Higher than NAT |
| Can Act as NAT | Yes (with config) | No |

> **Exam Alert:** If asked "how to SSH into private instance," answer is **Bastion Host**. If asked "how to access internet from private instance," answer is **NAT Gateway**.

### Alternatives to Bastion

**AWS Systems Manager Session Manager:**
- No port 22 needed
- Uses IAM roles + SSM Agent
- Auditing built-in
- Preferred in modern AWS deployments

```bash
aws ssm start-session --target i-0123456789abcdef0
```

---

## 12. Site-to-Site VPN

### Connect On-Premises to AWS

**Site-to-Site VPN** creates an encrypted connection from your on-premises data center to your VPC over the public internet.

**Use Case:**
```
Your Data Center (On-Premises)
  ↓ Encrypted over internet
Site-to-Site VPN Tunnel
  ↓
AWS VPC
```

### Components

**Virtual Private Gateway (VGW):**
- AWS-side VPN endpoint
- Attached to your VPC
- Routes traffic to Customer Gateway

**Customer Gateway (CGW):**
- Your side (on-premises)
- Software (e.g., OpenVPN) or hardware appliance
- Public IP address (static)

**VPN Connection:**
- Two redundant tunnels (tunnel 1 and tunnel 2)
- If tunnel 1 fails, tunnel 2 takes over automatically
- Each tunnel can handle max 1.25 Gbps

### Setup Steps

1. **Create Virtual Private Gateway (VGW)**
   ```bash
   aws ec2 create-vpn-gateway --type ipsec.1
   ```

2. **Attach VGW to VPC**
   ```bash
   aws ec2 attach-vpn-gateway --vpn-gateway-id vgw-xxxxx --vpc-id vpc-xxxxx
   ```

3. **Create Customer Gateway (CGW)**
   ```bash
   aws ec2 create-customer-gateway --type ipsec.1 --public-ip-address 203.0.113.50 --bgp-asn 65000
   ```

4. **Create VPN Connection**
   ```bash
   aws ec2 create-vpn-connection --type ipsec.1 --customer-gateway-id cgw-xxxxx --vpn-gateway-id vgw-xxxxx
   ```

5. **Enable Route Propagation**
   - Route table: Enable route propagation from VGW
   - Automatically learns routes from your on-premises network

### VPN CloudHub

If you have multiple on-premises sites (branch offices), connect them all to one VGW:

```
Branch A (Office 1)
  ↓
VGW (in VPC) ← Branches communicate through VPC
  ↓
Branch B (Office 2)
```

### VPN vs Direct Connect

| Feature | Site-to-Site VPN | Direct Connect |
|---------|------------------|----------------|
| Setup Time | Minutes | Weeks/months |
| Bandwidth | 1.25 Gbps per tunnel | 1 Gbps to 100 Gbps |
| Reliability | Redundant tunnels | Single connection (add 2nd for HA) |
| Latency | Variable (internet) | Consistent (dedicated) |
| Cost | Lower | Higher |
| Encryption | Built-in | Not encrypted (add VPN for security) |

> **Exam Strategy:** If exam says "quick setup" → VPN. If exam says "mission-critical, must be reliable" → Direct Connect. If both required → use Direct Connect with VPN backup.

---

## 13. AWS Direct Connect (DX)

### Dedicated Physical Connection

**AWS Direct Connect** is a dedicated private physical connection from your on-premises data center directly to AWS (NOT over the public internet).

### Connection Types & Bandwidth

**Dedicated Connection:**
- Dedicated 1 Gbps, 10 Gbps, or 100 Gbps port
- Physical fiber-optic connection
- Lead time: weeks to months (AWS provision and carrier provision fiber)
- Cost: port hours + data transfer

**Hosted Connection:**
- Partner-provisioned (AWS Direct Connect partner)
- Smaller bandwidth: 50 Mbps to 10 Gbps
- Faster lead time
- Cost: hourly + data transfer

### Benefits Over VPN

1. **Consistent Latency**: Dedicated connection, no internet congestion
2. **Dedicated Bandwidth**: Full 100 Gbps if you purchase it
3. **Lower Data Transfer Cost**: DX data transfer is cheaper than internet
4. **Private Connectivity**: Traffic never leaves AWS network
5. **Compliance**: Some regulations require private connections

### Virtual Interfaces (VIFs)

A DX connection carries data via Virtual Interfaces (VIFs):

**Private VIF:**
- Access VPC resources (private IPs)
- Routes to VGW
- Example: EC2, RDS, S3 (private endpoint)

**Public VIF:**
- Access AWS public services
- Routes to AWS public endpoints (S3, CloudFront, DynamoDB, etc.)
- Source IP is your DX connection IP (publicly routable)

**Transit VIF:**
- Used with Direct Connect Gateway + Transit Gateway
- Access multiple VPCs across regions

### Direct Connect Gateway

Single DX connection can connect to multiple VPCs across regions:

```
DX Connection → Direct Connect Gateway → VPC-A (us-east-1)
                                      → VPC-B (us-west-2)
                                      → VPC-C (eu-west-1)
```

Benefits:
- One DX connection serves multiple VPCs
- Simplifies on-premises to multi-VPC connectivity

### Resiliency & Failover

**Single DX Connection:**
- Single point of failure
- If DX connection fails, no connectivity to AWS

**Recommended for Critical Workloads:**
- Two DX connections from different locations
- VPN as backup for DX

```
Primary DX → AWS VPC
Backup DX → AWS VPC (from different location)
VPN → AWS VPC (if both DX connections fail)
```

### DX Encryption

> **Important:** Direct Connect is NOT encrypted by default. Data travels in clear text over the dedicated fiber. If you need encryption:

1. **Use VPN over DX**: IPSec encryption tunneled over DX
2. **Use TLS/SSL**: Application-level encryption
3. **Use MACsec**: Link-layer encryption for DX

---

## 14. Transit Gateway

### Hub-and-Spoke Network Topology

**Transit Gateway (TGW)** is a regional hub that connects VPCs, VPNs, and Direct Connect connections in a hub-and-spoke model.

**Problem Solved:**
```
Without TGW (Mesh):
10 VPCs need 10×9÷2 = 45 peering connections (unmanageable)

With TGW (Hub-and-Spoke):
10 VPCs need 10 TGW attachments (simple)
```

### Attachments

You attach to Transit Gateway:
- **VPC Attachments**: One or more subnets from each VPC (AWS creates ENI in subnet)
- **VPN Attachments**: Site-to-Site VPN connections
- **Direct Connect Attachments**: Direct Connect Gateway
- **Peering Attachments**: Other Transit Gateways (cross-region)

### Transit Gateway Route Tables

TGW has its own route tables controlling which attachments can communicate:

```
TGW Route Table:

Destination         Target
10.0.0.0/16        vpc-a-attachment    (VPC-A)
10.1.0.0/16        vpc-b-attachment    (VPC-B)
10.2.0.0/16        vpn-attachment      (Site-to-Site VPN)
192.168.0.0/16     tgw-peering         (Remote TGW)
```

**Segmentation Example:**
```
Prod VPC can reach: Prod RDS VPC, VPN
Dev VPC can reach: Dev RDS VPC, VPN
No cross-env traffic (prod→dev blocked)
```

### Advanced Features

**ECMP (Equal Cost Multi-Path):**
- Multiple VPN tunnels to same TGW
- Bandwidth aggregation: 4 VPN tunnels × 1.25 Gbps = 5 Gbps
- Active-active load balancing

**Multicast Support:**
- TGW can replicate multicast traffic to multiple attachments

**Network Manager:**
- Global visibility of your network
- Visualize VPCs, VPNs, Direct Connect connections

### Cross-Region TGW Peering

Two Transit Gateways in different regions can peer:

```
TGW-US-East (us-east-1)
  ↓ Peering
TGW-EU-West (eu-west-1)

Traffic from VPC-A (us-east-1) can reach VPC-B (eu-west-1)
```

> **Exam Alert:** Transit Gateway is the answer to "connect 100 VPCs" or "enable transitive VPC routing" or "shared network infrastructure across accounts."

---

## 15. VPC Traffic Mirroring

### Copy Traffic for Deep Inspection

**VPC Traffic Mirroring** copies inbound or outbound traffic from source ENI(s) to a monitoring appliance (IDS/IPS, packet analyzer).

### Components

**Source (ENI to mirror):**
- Specific EC2 instance ENI
- Captures inbound, outbound, or both

**Target:**
- ENI of monitoring appliance (e.g., Suricata IDS instance)
- NLB (Network Load Balancer) for distributed monitoring

**Filter:**
- Optional: limit which traffic to mirror
- Specify: protocol, port range, CIDR ranges

### Use Cases

1. **Intrusion Detection (IDS):**
   - Mirror all traffic to IDS appliance
   - Real-time threat detection

2. **Content Analysis:**
   - Deep packet inspection
   - DLP (Data Loss Prevention)

3. **Troubleshooting:**
   - Capture problematic traffic
   - Packet-level analysis (vs Flow Logs metadata)

### Traffic Mirroring vs VPC Flow Logs

| Feature | Traffic Mirroring | VPC Flow Logs |
|---------|-------------------|---------------|
| Data Type | Full packets | Metadata only |
| Use Case | Deep DPI, IDS | Troubleshooting, auditing |
| Performance Impact | Higher | Minimal |
| Cost | Monitoring appliance | Flow Logs cost |
| Real-time | Yes | ~1-10 min delay |

---

## 16. IPv6 in VPC

### Dual-Stack VPC (IPv4 + IPv6)

VPCs can support both IPv4 and IPv6 (dual-stack). IPv6 enables future-proofing and provides publicly routable addresses for all instances (no NAT needed).

### IPv6 CIDR Allocation

**VPC:** AWS assigns /56 CIDR (example: `2600:1f16:a2e:d100::/56`)
**Subnet:** /64 CIDR (example: `2600:1f16:a2e:d100::/64`)

IPv6 addresses are always PUBLIC (no private IPv6), so:
- No need for NAT Gateway with IPv6
- All instances can reach internet directly (if Egress-Only IGW attached)

### Egress-Only Internet Gateway

**Purpose:** Allow IPv6 outbound traffic from private subnets (equivalent of NAT for IPv6)

**Setup:**
1. Create Egress-Only IGW
2. Attach to VPC
3. Update private subnet route table: `::/0 → eigw-xxxxx`

Now instances can:
- Send outbound IPv6 traffic to internet
- Receive inbound responses only (stateful)

### Inbound IPv6 Traffic

To allow inbound IPv6:
1. Attach regular IGW (for IPv4) or Egress-Only IGW won't help
2. Update route table: `::/0 → igw-xxxxx`
3. Update SG: Allow inbound on port (e.g., port 80 from `::/0`)

### IPv6 Considerations

> **Exam Alert:** If exam mentions "EC2 can't reach internet over IPv6," check:
> 1. VPC has IPv6 CIDR assigned
> 2. Subnet has IPv6 CIDR assigned
> 3. Instance has IPv6 address
> 4. Route table has `::/0 → igw` or `::/0 → eigw`
> 5. SG allows IPv6 traffic (`::/0` in rules)

---

## 17. AWS PrivateLink

### Share SaaS Services Privately

**AWS PrivateLink** allows you to expose YOUR VPC services to other VPCs/accounts WITHOUT requiring VPC peering, route tables, or internet exposure.

**Use Case:**
```
Your Company provides API service internally
└─ Expose to other departments' VPCs
   using PrivateLink (no peering, no internet)

   or

   Expose to customers
   (customers access via Interface VPC Endpoint)
```

### Architecture

**Provider Side:**
1. Create Application Load Balancer (ALB) or Network Load Balancer (NLB)
2. Create VPC Endpoint Service backed by NLB
3. Share endpoint service with consumer account(s)

**Consumer Side:**
1. Create Interface VPC Endpoint
2. Point to provider's endpoint service
3. Instances connect via private IP to consumer's ENI (created in endpoint)

### Traffic Flow

```
Consumer VPC
└─ EC2 Instance (10.1.2.5)
   ↓ Connects to Interface Endpoint IP (10.1.3.10)
   ↓
PrivateLink (AWS manages the path)
   ↓
Provider VPC
└─ Network Load Balancer (10.0.1.10)
   └─ Backend Target (API Service)
```

### Benefits

1. **Security**: No internet exposure
2. **Simplicity**: No VPC peering, no route tables
3. **Scale**: Supports millions of customers (doesn't have peering transitive limits)
4. **Multi-account**: Works cross-account without sharing credentials

### Costs

**Provider:** No charges for PrivateLink (pay for ALB/NLB)

**Consumer:** ~$0.01/hour/AZ + data processing

---

## Points to Remember (Exam Focus)

### Critical Concepts

1. **VPC CIDR Planning**
   - /16 for VPC (65,536 IPs)
   - /24 for typical subnet (256 IPs)
   - Always reserve 5 IPs per subnet
   - /28 gives only 11 usable IPs (exam gotcha!)

2. **Public vs Private**
   - Public subnet: 0.0.0.0/0 → IGW
   - Private subnet: 0.0.0.0/0 → NAT GW
   - Update route table AFTER creating gateway

3. **Stateful vs Stateless**
   - Security Group: STATEFUL (no outbound rule for return traffic)
   - NACL: STATELESS (must allow ephemeral ports 1024-65535 outbound)

4. **Internet Access**
   - VPC needs IGW for bidirectional
   - Private subnet needs NAT GW for outbound-only
   - NAT Instance can act as bastion; NAT GW cannot

5. **Peering & Transitive Routing**
   - VPC peering NOT transitive
   - 10 VPCs = 45 peering connections (mesh)
   - Solution: Transit Gateway (10 attachments)

6. **VPC Endpoints**
   - S3 & DynamoDB: **Gateway** endpoints (free, route table)
   - All others: **Interface** endpoints (PrivateLink, costs money)
   - If private service can't access S3 → add gateway endpoint

7. **NAT Gateway Scaling**
   - One NAT GW per AZ for HA
   - Not free (~$32/month + data)
   - Max 100 Gbps, auto-scales from 5 Gbps

8. **Security Group Tricks**
   - Can reference other SG (same region only)
   - Multiple SGs on instance = union of rules
   - No DENY rules (only allow)

9. **NACLs for Denying**
   - Only way to BLOCK specific IP
   - Rules numbered, lowest first
   - Stateless: must allow return ports

10. **Bastion Hosts**
    - Public subnet, SSH access to private
    - Not for NAT (that's NAT Gateway)
    - Alternative: Systems Manager Session Manager

11. **VPN vs Direct Connect**
    - VPN: quick setup, encrypted, 1.25 Gbps per tunnel
    - DX: dedicated, high performance, weeks to setup, not encrypted by default
    - DX + VPN: VPN as backup for DX

12. **Transit Gateway**
    - Hub-and-spoke topology
    - Solves N×(N-1)/2 peering problem
    - Supports VPC, VPN, DX, peering

13. **Flow Logs**
    - VPC/subnet level: 10 min delay
    - ENI level: 1 min delay
    - Use to detect intrusion, troubleshoot connectivity

14. **IPv6 in VPC**
    - All addresses PUBLIC
    - Egress-Only IGW for outbound from private
    - No NAT needed for IPv6

15. **PrivateLink**
    - Provider: NLB → VPC Endpoint Service
    - Consumer: Interface VPC Endpoint
    - No peering, no route tables, million-customer scale

### Common Exam Scenarios

**Scenario 1:** "EC2 in private subnet needs to download updates from internet"
→ NAT Gateway in public subnet + route table

**Scenario 2:** "Need to SSH into private EC2 from internet"
→ Bastion host in public subnet

**Scenario 3:** "Two VPCs need to communicate privately"
→ VPC peering (update route tables + SG)

**Scenario 4:** "One VPC needs to connect 100 other VPCs"
→ Transit Gateway

**Scenario 5:** "Hacker IP 203.0.113.1 keeps scanning. Block it."
→ NACL DENY rule (SG can't deny)

**Scenario 6:** "Private Lambda can't access S3"
→ Add S3 Gateway VPC Endpoint

**Scenario 7:** "On-premises office needs AWS access, quick setup"
→ Site-to-Site VPN

**Scenario 8:** "On-premises office needs AWS access, mission-critical, high performance"
→ Direct Connect (add VPN backup)

**Scenario 9:** "Need to detect intrusions in VPC traffic"
→ VPC Traffic Mirroring to IDS

**Scenario 10:** "Share internal API service with other accounts securely"
→ PrivateLink (NLB → VPC Endpoint Service)

---

## AWS CLI Quick Reference

### VPC Management

```bash
# Create VPC with /16 CIDR
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=MyVPC}]'

# List VPCs
aws ec2 describe-vpcs --query 'Vpcs[*].[VpcId,CidrBlock,IsDefault]' --output table

# Delete VPC (and all dependencies)
aws ec2 delete-vpc --vpc-id vpc-xxxxx

# Enable DNS resolution in VPC
aws ec2 modify-vpc-attribute --vpc-id vpc-xxxxx --enable-dns-resolution

# Enable DNS hostnames in VPC
aws ec2 modify-vpc-attribute --vpc-id vpc-xxxxx --enable-dns-hostnames
```

### Subnet Management

```bash
# Create public subnet
aws ec2 create-subnet --vpc-id vpc-xxxxx --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PublicSubnet}]'

# Create private subnet
aws ec2 create-subnet --vpc-id vpc-xxxxx --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PrivateSubnet}]'

# List subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxxxx" --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone]' --output table

# Enable auto-assign public IPv4 for subnet
aws ec2 modify-subnet-attribute --subnet-id subnet-xxxxx --map-public-ip-on-launch

# Disable auto-assign public IPv4 for subnet
aws ec2 modify-subnet-attribute --subnet-id subnet-xxxxx --no-map-public-ip-on-launch

# Delete subnet
aws ec2 delete-subnet --subnet-id subnet-xxxxx
```

### Internet Gateway

```bash
# Create Internet Gateway
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=MyIGW}]'

# Attach IGW to VPC
aws ec2 attach-internet-gateway --internet-gateway-id igw-xxxxx --vpc-id vpc-xxxxx

# Detach IGW from VPC
aws ec2 detach-internet-gateway --internet-gateway-id igw-xxxxx --vpc-id vpc-xxxxx

# List Internet Gateways
aws ec2 describe-internet-gateways --query 'InternetGateways[*].[InternetGatewayId,Tags[0].Value]' --output table

# Delete Internet Gateway
aws ec2 delete-internet-gateway --internet-gateway-id igw-xxxxx
```

### Route Tables

```bash
# Create route table
aws ec2 create-route-table --vpc-id vpc-xxxxx --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PublicRT}]'

# Add route to IGW
aws ec2 create-route --route-table-id rtb-xxxxx --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxxxx

# Add route to NAT Gateway
aws ec2 create-route --route-table-id rtb-xxxxx --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-xxxxx

# Add route for VPC Peering
aws ec2 create-route --route-table-id rtb-xxxxx --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id pcx-xxxxx

# List route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-xxxxx" --query 'RouteTables[*].[RouteTableId,Routes[*].[DestinationCidrBlock,GatewayId,NatGatewayId]]' --output table

# Associate subnet with route table
aws ec2 associate-route-table --subnet-id subnet-xxxxx --route-table-id rtb-xxxxx

# Disassociate subnet from route table
aws ec2 disassociate-route-table --association-id rtbassoc-xxxxx

# Delete route table
aws ec2 delete-route-table --route-table-id rtb-xxxxx

# Delete specific route
aws ec2 delete-route --route-table-id rtb-xxxxx --destination-cidr-block 0.0.0.0/0
```

### NAT Gateway

```bash
# Allocate Elastic IP for NAT Gateway
aws ec2 allocate-address --domain vpc --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=NATIP}]'

# Create NAT Gateway in public subnet
aws ec2 create-nat-gateway --subnet-id subnet-xxxxx (public) --allocation-id eipalloc-xxxxx --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=MyNAT}]'

# List NAT Gateways
aws ec2 describe-nat-gateways --query 'NatGateways[*].[NatGatewayId,SubnetId,State,NatGatewayAddresses[0].PublicIp]' --output table

# Delete NAT Gateway
aws ec2 delete-nat-gateway --nat-gateway-id nat-xxxxx

# Release Elastic IP
aws ec2 release-address --allocation-id eipalloc-xxxxx

# Monitor NAT Gateway bandwidth (CloudWatch)
aws cloudwatch get-metric-statistics --namespace AWS/NatGateway --metric-name BytesOutToDestination --dimensions Name=NatGatewayId,Value=nat-xxxxx --start-time 2024-01-01T00:00:00Z --end-time 2024-01-01T01:00:00Z --period 300 --statistics Sum
```

### Security Groups

```bash
# Create Security Group
aws ec2 create-security-group --group-name web-sg --description "Web server SG" --vpc-id vpc-xxxxx --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=web-sg}]'

# Add inbound HTTP rule
aws ec2 authorize-security-group-ingress --group-id sg-xxxxx --protocol tcp --port 80 --cidr 0.0.0.0/0

# Add inbound SSH rule from specific CIDR
aws ec2 authorize-security-group-ingress --group-id sg-xxxxx --protocol tcp --port 22 --cidr 203.0.113.0/24

# Add inbound rule from another SG
aws ec2 authorize-security-group-ingress --group-id sg-xxxxx --protocol tcp --port 3306 --source-group sg-yyyyy

# Add outbound rule
aws ec2 authorize-security-group-egress --group-id sg-xxxxx --protocol tcp --port 443 --cidr 0.0.0.0/0

# Remove inbound rule
aws ec2 revoke-security-group-ingress --group-id sg-xxxxx --protocol tcp --port 80 --cidr 0.0.0.0/0

# List Security Groups
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=vpc-xxxxx" --query 'SecurityGroups[*].[GroupId,GroupName,IpPermissions[*].[FromPort,ToPort,IpProtocol]]' --output table

# Delete Security Group
aws ec2 delete-security-group --group-id sg-xxxxx
```

### Network ACLs

```bash
# Create custom NACL
aws ec2 create-network-acl --vpc-id vpc-xxxxx --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=MyNACL}]'

# Add inbound allow rule
aws ec2 create-network-acl-entry --network-acl-id acl-xxxxx --rule-number 100 --protocol tcp --port-range FromPort=80,ToPort=80 --cidr-block 0.0.0.0/0 --ingress

# Add inbound deny rule
aws ec2 create-network-acl-entry --network-acl-id acl-xxxxx --rule-number 110 --protocol tcp --port-range FromPort=22,ToPort=22 --cidr-block 203.0.113.45/32 --egress false --no-egress

# Add outbound allow ephemeral ports
aws ec2 create-network-acl-entry --network-acl-id acl-xxxxx --rule-number 120 --protocol tcp --port-range FromPort=1024,ToPort=65535 --cidr-block 0.0.0.0/0 --egress

# List NACL rules
aws ec2 describe-network-acls --network-acl-ids acl-xxxxx --query 'NetworkAcls[*].Entries[*].[RuleNumber,Protocol,PortRange,CidrBlock,RuleAction]' --output table

# Delete NACL entry
aws ec2 delete-network-acl-entry --network-acl-id acl-xxxxx --rule-number 100 --egress false

# Associate NACL with subnet
aws ec2 associate-network-acl --network-acl-id acl-xxxxx --subnet-id subnet-xxxxx
```

### VPC Peering

```bash
# Create VPC Peering Connection
aws ec2 create-vpc-peering-connection --vpc-id vpc-xxxxx --peer-vpc-id vpc-yyyyy --peer-owner-id 123456789012 --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=vpc-a-to-b}]'

# Accept peering connection (in peer account)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-xxxxx

# Add route in VPC-A to reach VPC-B CIDR
aws ec2 create-route --route-table-id rtb-a --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id pcx-xxxxx

# Add route in VPC-B to reach VPC-A CIDR
aws ec2 create-route --route-table-id rtb-b --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id pcx-xxxxx

# List VPC Peering Connections
aws ec2 describe-vpc-peering-connections --filters "Name=status-code,Values=active" --query 'VpcPeeringConnections[*].[VpcPeeringConnectionId,RequesterVpcInfo.VpcId,AccepterVpcInfo.VpcId,Status.Code]' --output table

# Reject peering connection
aws ec2 reject-vpc-peering-connection --vpc-peering-connection-id pcx-xxxxx

# Delete peering connection
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id pcx-xxxxx
```

### VPC Endpoints

```bash
# Create Gateway Endpoint for S3
aws ec2 create-vpc-endpoint --vpc-id vpc-xxxxx --service-name com.amazonaws.us-east-1.s3 --route-table-ids rtb-xxxxx --vpc-endpoint-type Gateway

# Create Interface Endpoint for Secrets Manager
aws ec2 create-vpc-endpoint --vpc-id vpc-xxxxx --service-name com.amazonaws.us-east-1.secretsmanager --vpc-endpoint-type Interface --subnet-ids subnet-xxxxx --security-group-ids sg-xxxxx

# List VPC Endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=vpc-xxxxx" --query 'VpcEndpoints[*].[VpcEndpointId,ServiceName,VpcEndpointType,State]' --output table

# Modify endpoint policy
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-xxxxx --policy-document file://endpoint-policy.json

# Delete VPC Endpoint
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-xxxxx
```

### VPC Flow Logs

```bash
# Create Flow Logs to CloudWatch
aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-xxxxx --traffic-type ALL --log-destination-type cloud-watch-logs --log-group-name /aws/vpc/flowlogs --deliver-logs-permission-role-arn arn:aws:iam::123456789012:role/flowlogsRole

# Create Flow Logs to S3
aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-xxxxx --traffic-type ALL --log-destination-type s3 --log-destination arn:aws:s3:::my-bucket/flowlogs/

# List Flow Logs
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=vpc-xxxxx" --query 'FlowLogs[*].[FlowLogId,ResourceId,FlowLogStatus,LogDestinationType]' --output table

# Delete Flow Logs
aws ec2 delete-flow-logs --flow-log-ids fl-xxxxx

# Query Flow Logs in CloudWatch Logs Insights
aws logs start-query --log-group-name /aws/vpc/flowlogs --start-time $(date -d '1 hour ago' +%s) --end-time $(date +%s) --query-string 'fields @timestamp, srcaddr, dstaddr, action | filter action = "REJECT" | stats count() by srcaddr'
```

### Network Interfaces

```bash
# Describe ENIs
aws ec2 describe-network-interfaces --filters "Name=vpc-id,Values=vpc-xxxxx" --query 'NetworkInterfaces[*].[NetworkInterfaceId,PrivateIpAddress,SubnetId,Status]' --output table

# Create secondary private IP on ENI
aws ec2 assign-private-ip-addresses --network-interface-id eni-xxxxx --private-ip-addresses 10.0.1.50

# Attach Elastic IP to ENI
aws ec2 associate-address --network-interface-id eni-xxxxx --allocation-id eipalloc-xxxxx

# Detach Elastic IP
aws ec2 disassociate-address --association-id eipassoc-xxxxx
```

### Transit Gateway

```bash
# Create Transit Gateway
aws ec2 create-transit-gateway --description "Central Hub" --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=MyTGW}]'

# Create TGW attachment to VPC
aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id tgw-xxxxx --vpc-id vpc-xxxxx --subnet-ids subnet-a subnet-b --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=vpc-a-attachment}]'

# Create TGW route table
aws ec2 create-transit-gateway-route-table --transit-gateway-id tgw-xxxxx --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=prod-rt}]'

# Create route in TGW RT
aws ec2 create-transit-gateway-route --transit-gateway-route-table-id tgw-rtb-xxxxx --destination-cidr-block 10.0.0.0/16 --transit-gateway-attachment-id tgw-attach-xxxxx

# List Transit Gateways
aws ec2 describe-transit-gateways --query 'TransitGateways[*].[TransitGatewayId,State,AmazonSideAsn,TransitGatewayArn]' --output table

# List TGW attachments
aws ec2 describe-transit-gateway-attachments --filters "Name=transit-gateway-id,Values=tgw-xxxxx" --query 'TransitGatewayAttachments[*].[TransitGatewayAttachmentId,ResourceType,State]' --output table
```

### Site-to-Site VPN

```bash
# Create VPN Gateway
aws ec2 create-vpn-gateway --type ipsec.1 --tag-specifications 'ResourceType=vpn-gateway,Tags=[{Key=Name,Value=MyVGW}]'

# Attach VGW to VPC
aws ec2 attach-vpn-gateway --vpn-gateway-id vgw-xxxxx --vpc-id vpc-xxxxx

# Create Customer Gateway (on-premises end)
aws ec2 create-customer-gateway --type ipsec.1 --public-ip-address 203.0.113.50 --bgp-asn 65000 --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=corp-office}]'

# Create VPN Connection (two redundant tunnels created automatically)
aws ec2 create-vpn-connection --type ipsec.1 --customer-gateway-id cgw-xxxxx --vpn-gateway-id vgw-xxxxx --options "StaticRoutesOnly=true" --tag-specifications 'ResourceType=vpn-connection,Tags=[{Key=Name,Value=corp-vpn}]'

# Enable route propagation (auto-learn routes from on-premises)
aws ec2 enable-vgw-route-propagation --route-table-id rtb-xxxxx --gateway-id vgw-xxxxx

# List VPN Connections
aws ec2 describe-vpn-connections --query 'VpnConnections[*].[VpnConnectionId,State,CustomerGatewayId,VpnGatewayId]' --output table

# Download VPN configuration (for on-premises router)
aws ec2 describe-vpn-connections --vpn-connection-ids vpn-xxxxx --query 'VpnConnections[*].CustomerGatewayConfiguration' --output text > vpn-config.xml
```

### Direct Connect

```bash
# Create Connection (request dedicated DX connection)
aws directconnect create-connection --location "Your City" --bandwidth 10Gbps --connection-name "corp-dx" --tags Key=Name,Value=corp-dx

# Create Private Virtual Interface (for VPC access)
aws directconnect create-private-virtual-interface --connection-id dxcon-xxxxx --new-private-virtual-interface virtualInterfaceName=corp-vif,vlan=100,asn=65001,authKey=secret,addressFamily=ipv4,amazonAddress=169.254.10.1/30,customerAddress=169.254.10.2/30,virtualGatewayId=vgw-xxxxx

# List Connections
aws directconnect describe-connections --query 'connections[*].[connectionId,connectionName,bandwidth,connectionState]' --output table

# List Virtual Interfaces
aws directconnect describe-virtual-interfaces --query 'virtualInterfaces[*].[virtualInterfaceId,virtualInterfaceName,interfaceType,virtualInterfaceState]' --output table
```

### Bastion / Tunneling

```bash
# SSH through bastion (proxy command)
ssh -i /path/to/private-key -o ProxyCommand="ssh -i /path/to/bastion-key -W %h:%p ec2-user@bastion-public-ip" ec2-user@private-instance-ip

# Start Systems Manager Session (alternative to bastion)
aws ssm start-session --target i-0123456789abcdef0 --document-name AWS-StartInteractiveCommand

# Port forward through bastion
ssh -i /path/to/key -L 3306:private-rds-endpoint:3306 ec2-user@bastion-public-ip
```

### Traffic Mirroring

```bash
# Create traffic mirror target (ENI)
aws ec2 create-traffic-mirror-target --network-interface-id eni-xxxxx (of monitoring appliance) --description "IDS appliance"

# Create traffic mirror filter (specify what to mirror)
aws ec2 create-traffic-mirror-filter --description "Mirror HTTP traffic"

# Add rule to filter (mirror HTTP)
aws ec2 create-traffic-mirror-filter-rule --traffic-mirror-filter-id tmf-xxxxx --traffic-direction ingress --rule-number 100 --rule-action accept --source-cidr-block 0.0.0.0/0 --destination-cidr-block 0.0.0.0/0 --protocol 6 --destination-port-range FromPort=80,ToPort=80

# Create traffic mirror session (attach to source)
aws ec2 create-traffic-mirror-session --network-interface-id eni-yyyyy (source) --traffic-mirror-target-id tmt-xxxxx --traffic-mirror-filter-id tmf-xxxxx --session-number 1

# List mirror sessions
aws ec2 describe-traffic-mirror-sessions --query 'TrafficMirrorSessions[*].[TrafficMirrorSessionId,NetworkInterfaceId,Status]' --output table
```

---

## Summary

This comprehensive guide covers all VPC networking concepts required for AWS SAA-C03 exam. Key takeaways:

1. **Understand the layers**: VPC → Subnets → Route Tables → Security Groups/NACLs
2. **Know when to use**: Gateway vs Interface endpoints, VPN vs Direct Connect, VPC Peering vs Transit Gateway
3. **Master the stateful/stateless concept**: Critical differentiator between SG and NACL
4. **Practice real-world scenarios**: Bastion hosts, multi-VPC architectures, hybrid connectivity
5. **CLI fluency**: Practice all commands with your own VPC setup

Good luck on your SAA-C03 exam!
