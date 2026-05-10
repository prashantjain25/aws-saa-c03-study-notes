# SAP Section 15: VPC — Advanced Networking

> **SAP Depth Note**: This is not an introductory VPC guide. It assumes you know Security Groups, NACLs, route tables, and basic IGW/NAT concepts from SAA level. The focus here is on architecture patterns, hybrid connectivity trade-offs, multi-account networking, and the decision frameworks that appear on the SAP exam.

---

## Table of Contents

1. [VPC Basics — SAP-Level CIDR Planning & Design](#1-vpc-basics--sap-level-cidr-planning--design)
2. [VPC Peering — Non-Transitive, Full Mesh Problem](#2-vpc-peering--non-transitive-full-mesh-problem)
3. [Transit Gateway — Hub-and-Spoke at Scale](#3-transit-gateway--hub-and-spoke-at-scale)
4. [VPC Endpoints — Gateway vs Interface](#4-vpc-endpoints--gateway-vs-interface)
5. [VPC Endpoint Policies — Fine-Grained Access Control](#5-vpc-endpoint-policies--fine-grained-access-control)
6. [AWS PrivateLink — Expose Services Without Peering](#6-aws-privatelink--expose-services-without-peering)
7. [AWS Site-to-Site VPN — Hybrid Connectivity](#7-aws-site-to-site-vpn--hybrid-connectivity)
8. [AWS Client VPN — Remote Access](#8-aws-client-vpn--remote-access)
9. [AWS Direct Connect — Dedicated Physical Link](#9-aws-direct-connect--dedicated-physical-link)
10. [On-Premises Redundant Connections — DX + VPN Patterns](#10-on-premises-redundant-connections--dx--vpn-patterns)
11. [VPC Flow Logs — Capture, Query, Alert](#11-vpc-flow-logs--capture-query-alert)
12. [AWS Network Firewall — Stateful Deep Inspection](#12-aws-network-firewall--stateful-deep-inspection)
13. [Architecture Patterns — Egress, Inspection, Hybrid DNS](#13-architecture-patterns--egress-inspection-hybrid-dns)
14. [Decision Framework — Connectivity Options Comparison](#14-decision-framework--connectivity-options-comparison)

---

## 1. VPC Basics — SAP-Level CIDR Planning & Design

### Why CIDR Planning Matters at the SAP Level

At the SAP level, you are not just creating one VPC in one account. You are designing a network for dozens of accounts, multiple regions, and on-premises data centers — all of which must coexist without IP conflicts. Poor CIDR planning causes VPC peering failures, Transit Gateway routing conflicts, and forces subnet re-IPs that break running workloads.

The SAP exam will give you scenarios where you must identify whether two VPCs CAN be peered (no CIDR overlap), or design a CIDR scheme for a multi-account environment.

### CIDR Planning Rules

**Rule 1: No overlapping CIDRs between any two VPCs you want to connect.**

```
VPC-A: 10.0.0.0/16    ✓
VPC-B: 10.1.0.0/16    ✓  (no overlap — can peer)
VPC-C: 10.0.0.0/24    ✗  (overlaps with VPC-A — cannot peer with A)
```

**Rule 2: Reserve enough address space. Subnets cannot be resized.**

A VPC CIDR can be extended (add secondary CIDRs) but cannot shrink or change the primary. Subnet CIDRs are permanent after creation.

**Rule 3: Design with Transit Gateway in mind.**

If you know you will use a Transit Gateway, each VPC needs its own unique RFC-1918 block. A common enterprise allocation strategy:

```
AWS Region: 10.0.0.0/8 (entire RFC-1918 class A for AWS)
├── Account: Production    → 10.0.0.0/12  (10.0.x.x to 10.15.x.x)
├── Account: Staging       → 10.16.0.0/12 (10.16.x.x to 10.31.x.x)
├── Account: Development   → 10.32.0.0/12 (10.32.x.x to 10.47.x.x)
└── On-Premises            → 192.168.0.0/16 (separate RFC-1918 block)
```

Within each account, allocate /16 per VPC:
```
Production VPC-1: 10.0.0.0/16   → US-East-1
Production VPC-2: 10.1.0.0/16   → US-West-2
Production VPC-3: 10.2.0.0/16   → EU-West-1
```

### Subnet Design — Public / Private / Data Tier

**Canonical 3-Tier VPC layout (per AZ):**

```
VPC: 10.0.0.0/16
├── AZ-a
│   ├── Public  subnet: 10.0.0.0/24   (ALB, NAT GW, Bastion)
│   ├── Private subnet: 10.0.1.0/24   (App servers, Lambda)
│   └── Data    subnet: 10.0.2.0/24   (RDS, ElastiCache — no outbound needed)
├── AZ-b
│   ├── Public  subnet: 10.0.10.0/24
│   ├── Private subnet: 10.0.11.0/24
│   └── Data    subnet: 10.0.12.0/24
└── AZ-c
    ├── Public  subnet: 10.0.20.0/24
    ├── Private subnet: 10.0.21.0/24
    └── Data    subnet: 10.0.22.0/24
```

**Why keep data subnets separate from private?** Isolation for compliance (PCI-DSS, HIPAA). If app servers are compromised, data-tier subnet NACLs still require an explicit allow. It also makes VPC Flow Log filtering trivial — all DB traffic is visibly separated.

### NAT Gateway vs NAT Instance

| Feature | NAT Gateway | NAT Instance |
|---------|-------------|--------------|
| Managed | Yes — AWS manages HA | No — EC2 you manage |
| Throughput | 5 Gbps auto-scales to 100 Gbps | Instance-type bounded (e.g., t3.micro = ~300 Mbps) |
| High Availability | Redundant within AZ (not cross-AZ) | Must configure manually with scripts |
| Security Group | Cannot attach | Can attach SG |
| Can serve as Bastion | No | Yes |
| Cost | ~$0.045/hour + $0.045/GB | EC2 cost only |
| Source/Dest Check | N/A (managed) | Must disable (`--no-source-dest-check`) |

🎯 **EXAM TIP**: The SAP exam tests NAT Instance in edge cases: "Cheapest NAT for dev environment with low traffic" → NAT Instance. "Need outbound internet AND SSH into private instances from same host" → NAT Instance (NAT Gateway cannot do SSH). "Need 50 Gbps sustained throughput" → NAT Gateway.

**One NAT Gateway per AZ for High Availability:**

```
                   AZ-a                    AZ-b
Private Subnet-a ──→ NAT-GW-a (EIP-a) ──→ IGW
Private Subnet-b ──→ NAT-GW-b (EIP-b) ──→ IGW
```

If you put ONE NAT Gateway in AZ-a and AZ-b's private subnet routes through it: when AZ-a fails, AZ-b's instances lose internet access. Cost vs resilience trade-off for non-critical workloads.

---

## 2. VPC Peering — Non-Transitive, Full Mesh Problem

### The Core Rule: Not Transitive

VPC Peering creates a direct private link between exactly two VPCs. There is no transit allowed through a peered VPC.

```
VPC-A ←──peering──→ VPC-B ←──peering──→ VPC-C

VPC-A CANNOT reach VPC-C through VPC-B.
Traffic from A destined for C is dropped at B.
```

This is by design. AWS does not route packets through intermediate VPCs. Even if VPC-B has routes to both A and C, those are separate peering connections and B does not forward between them.

### Full Mesh: The N(N-1)/2 Problem

For N VPCs to all communicate with each other via peering:

| VPCs | Peering Connections Required |
|------|------------------------------|
| 2    | 1                            |
| 5    | 10                           |
| 10   | 45                           |
| 20   | 190                          |
| 50   | 1,225                        |

Each peering connection also requires route table entries in BOTH VPCs. Managing 190 peering connections for 20 VPCs is operationally infeasible. This is exactly why Transit Gateway exists.

### When to Use VPC Peering (vs Transit Gateway)

VPC Peering still makes sense when:
- You have 2–5 VPCs that need to communicate
- You need lowest possible latency (peering is a direct path, no TGW hop)
- You want no data transfer charges within the same AZ
- You want connection-level isolation — each peering is explicitly authorized

🎯 **EXAM TIP**: VPC Peering is the answer when the scenario mentions "2 VPCs", "no transitive routing needed", or "minimize cost for same-region communication". Transit Gateway is the answer when there are many VPCs or shared services.

### Cross-Account and Cross-Region Peering

**Cross-Account:** VPC-A (Account A) initiates peering request → VPC-B (Account B) accepts. Route table entries must be added in both VPCs. Security group referencing across accounts requires CIDR ranges (you cannot reference a foreign account SG by ID across peering).

**Cross-Region:** Supported. Uses AWS backbone, not public internet. Data transfer fees apply (~$0.02/GB). Bandwidth cap of ~10 Gbps per peering connection. Cross-region peering CIDR cannot be edited after creation.

**Security Group referencing:**
- Same-region peering: Can reference remote SG by ID in rules
- Cross-region peering: Must use CIDR ranges only

🎯 **EXAM TIP**: If a question says "cross-region VPC communication with security group referencing by ID" — that is NOT possible. Must use CIDR or switch to same-region architecture.

---

## 3. Transit Gateway — Hub-and-Spoke at Scale

### Why Transit Gateway Exists

Transit Gateway (TGW) is a regional network transit hub. Instead of N(N-1)/2 peering connections, every VPC, VPN, and Direct Connect attaches to the TGW. The TGW then routes between attachments.

```
                    ┌─────────────────────────────────────┐
                    │          Transit Gateway             │
                    │   (Regional — us-east-1)             │
                    └──┬──────┬──────┬──────┬──────┬──────┘
                       │      │      │      │      │
                    VPC-A  VPC-B  VPC-C  VPN-1  DX-GW
```

### TGW Attachments

| Attachment Type | What Connects |
|-----------------|---------------|
| VPC attachment | One or more subnets from a VPC (AWS creates ENI per subnet) |
| VPN attachment | Site-to-Site VPN connection |
| Direct Connect Gateway | Via Transit VIF |
| TGW Peering | Another Transit Gateway (cross-region) |
| AWS RAM share | Share TGW to other accounts in your AWS Organization |

### TGW Route Tables — Traffic Segmentation

This is where TGW becomes powerful. You can create multiple TGW route tables and associate different attachments to different tables.

**Example: Prod/Dev Isolation + Shared Services**

```
TGW Route Table: "prod-rt"
  Associations: VPC-Prod, VPC-ProdDB
  Propagations: VPC-Prod → 10.0.0.0/16, VPC-ProdDB → 10.1.0.0/16
  Static Route: 0.0.0.0/0 → Inspection-VPC (all egress goes to firewall)

TGW Route Table: "dev-rt"
  Associations: VPC-Dev
  Propagations: VPC-Dev → 10.32.0.0/16
  Static Route: 0.0.0.0/0 → Inspection-VPC

TGW Route Table: "shared-rt"
  Associations: VPC-Shared (DNS, Security tooling)
  Propagations: all VPC CIDRs
```

Result: Prod can reach Shared. Dev can reach Shared. Prod CANNOT reach Dev (no route in prod-rt to 10.32.0.0/16).

### Cross-Region TGW Peering

Two Transit Gateways in different regions can be peered. This is NOT the same as VPC peering — you are peering the TGWs themselves.

```
TGW-US-East (us-east-1) ←── TGW Peering ──→ TGW-EU-West (eu-west-1)

Traffic from VPC-A (us-east-1) → TGW-US-East → TGW Peering → TGW-EU-West → VPC-B (eu-west-1)
```

**Key SAP details for cross-region TGW peering:**
- Static routes only (no BGP between TGWs)
- You must manually add routes in each TGW's route table pointing to the peering attachment
- Traffic uses AWS global backbone (not public internet)
- Data transfer charges apply (inter-region)
- Bandwidth: depends on region pair, typically up to 50 Gbps aggregate

🎯 **EXAM TIP**: Cross-region TGW peering uses STATIC routes. VPN over TGW uses BGP for dynamic route learning. Know this distinction — the SAP exam tests it.

### TGW with AWS RAM (Resource Access Manager) — Multi-Account

The most important SAP pattern: share a TGW across accounts using AWS RAM.

```
Network Account (Hub)
└── Creates Transit Gateway
└── Shares TGW via RAM to Production OU, Dev OU

Production Account
└── Creates TGW attachment (VPC → shared TGW)
└── Does NOT own the TGW, just attaches to it

Development Account
└── Creates TGW attachment (VPC → shared TGW)
```

**Why this is the correct enterprise pattern:**
- Centralized network team owns the TGW and route tables
- Application teams attach their VPCs without network team involvement
- All routing policy (which VPC can reach what) is enforced centrally
- One TGW supports up to 5,000 VPC attachments

🎯 **EXAM TIP**: Questions about "centralized network management for 200 AWS accounts" → TGW shared via AWS RAM. Questions about "application teams need to connect VPCs without network team bottleneck" → TGW with RAM sharing, with route propagation.

### TGW Bandwidth and Limits

| Metric | Limit |
|--------|-------|
| Max VPC attachments | 5,000 per TGW |
| Max route tables | 20 per TGW (soft limit) |
| Bandwidth per VPC attachment | Up to 50 Gbps |
| VPN attachment bandwidth | 1.25 Gbps per tunnel (ECMP for more) |
| Cross-region peering attachments | 50 per TGW |

### ECMP — Aggregating VPN Bandwidth Through TGW

By default, a single VPN tunnel provides 1.25 Gbps. With Transit Gateway and ECMP (Equal-Cost Multi-Path):

```
VPN Connection 1 → Tunnel A (1.25 Gbps) ─┐
                 → Tunnel B (1.25 Gbps) ─┤
VPN Connection 2 → Tunnel C (1.25 Gbps) ─┤→ TGW (ECMP active-active) → up to 5+ Gbps
                 → Tunnel D (1.25 Gbps) ─┘
```

🎯 **EXAM TIP**: ECMP only works when attached to Transit Gateway, NOT to a Virtual Private Gateway (VGW). If you need >1.25 Gbps VPN throughput → must use TGW with multiple VPN connections and ECMP enabled.

### Multicast Support

TGW supports multicast traffic replication. You create a multicast domain, add subnets, and the TGW replicates multicast packets to all receivers. Use case: financial market data feeds, media distribution. This is rarely tested but appears as a differentiator ("which AWS service supports multicast?").

---

## 4. VPC Endpoints — Gateway vs Interface

### The Problem VPC Endpoints Solve

Without endpoints, private resources that need to access AWS services (S3, KMS, SQS, etc.) must route through NAT Gateway → IGW → public internet → AWS service endpoint. This:
- Exposes traffic to public internet routing
- Incurs NAT Gateway data processing fees ($0.045/GB)
- Adds latency
- Fails compliance for "data must not traverse the internet"

With VPC endpoints, traffic stays entirely within the AWS network.

### Gateway Endpoints — S3 and DynamoDB Only

Gateway endpoints modify the VPC route table to direct traffic to the service via AWS backbone.

```
Route table entry added automatically:
Destination: pl-63a5400a (S3 prefix list)  →  Target: vpce-xxxxxxxx
```

**Key characteristics:**
- Free (no hourly or data charges)
- Works only for S3 and DynamoDB
- Not powered by PrivateLink — uses route table prefix list injection
- Cannot be accessed from on-premises (via VPN or DX) — the route exists only inside the VPC
- Cannot be accessed from a peered VPC — routes are not transitive through peering
- Endpoint policies can restrict which S3 buckets/DynamoDB tables are accessible

🎯 **EXAM TIP**: "On-premises servers need to access S3 privately via Direct Connect" — Gateway endpoint does NOT work here. You need an Interface endpoint for S3, which has a DNS name accessible from on-premises.

### Interface Endpoints — All Other Services (Powered by PrivateLink)

Interface endpoints create an ENI in your subnet with a private IP address. The service DNS name resolves to this private IP.

```
Without endpoint:
  kms.us-east-1.amazonaws.com → 52.119.x.x (public IP)

With interface endpoint:
  kms.us-east-1.amazonaws.com → 10.0.2.50  (your ENI private IP)
  (or use endpoint-specific DNS: vpce-xxx.kms.us-east-1.vpce.amazonaws.com → 10.0.2.50)
```

**Key characteristics:**
- Costs ~$0.01/hour/AZ + $0.01/GB data processed
- Place in multiple AZs for HA (separate ENI per AZ)
- Security Group attached to the ENI — controls who can send traffic to the endpoint
- Works from on-premises (via DX or VPN) because it has a real private IP
- Works from peered VPCs IF private DNS is not enabled (use the endpoint-specific DNS name)

### Interface Endpoint — DNS Resolution

When you enable "Private DNS" on an interface endpoint:
- The default service DNS name resolves to the endpoint ENI IP within that VPC
- Other VPCs peered to this VPC cannot use the private DNS (they resolve to public IP)
- On-premises servers can use the endpoint-specific DNS name (not the default one)

🎯 **EXAM TIP**: "Centralize VPC Endpoints and share across many VPCs" → Create endpoints in a central VPC, peer all VPCs to it, use endpoint-specific DNS (not private DNS). Alternatively, use AWS PrivateLink-based endpoints with Route 53 Private Hosted Zones forwarding.

### Endpoint for S3 — Gateway vs Interface Decision

| Scenario | Use |
|----------|-----|
| EC2 in same VPC accessing S3 | Gateway endpoint (free) |
| Lambda in same VPC accessing S3 | Gateway endpoint (free) |
| On-premises accessing S3 privately via DX | Interface endpoint |
| Peered VPC needs S3 access via central endpoint | Interface endpoint |
| Compliance: S3 traffic must never leave AWS network, on-premises | Interface endpoint |

---

## 5. VPC Endpoint Policies — Fine-Grained Access Control

### What Endpoint Policies Do

An endpoint policy is an IAM resource policy attached to the VPC endpoint itself. It controls which principals can use the endpoint and which actions/resources they can access through it.

Endpoint policies are in ADDITION to IAM policies on the principal — both must allow the action.

### S3 Gateway Endpoint Policy — Restrict to Own Buckets Only

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-company-bucket",
        "arn:aws:s3:::my-company-bucket/*"
      ]
    }
  ]
}
```

This prevents EC2 instances from exfiltrating data to attacker-controlled S3 buckets through the endpoint, even if the EC2 IAM role has broad S3 permissions.

### S3 Bucket Policy — Restrict Access to VPC Endpoint Only

Flip side: enforce that S3 is ONLY accessible from your VPC endpoint (deny all other access):

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
  "Condition": {
    "StringNotEquals": {
      "aws:sourceVpce": "vpce-xxxxxxxxxxxxxxxxx"
    }
  }
}
```

🎯 **EXAM TIP**: The combination of (1) S3 bucket policy requiring `aws:sourceVpce` and (2) endpoint policy limiting to specific buckets is the standard pattern for "data exfiltration prevention via S3". This appears frequently in SAP security scenarios.

### Endpoint Policy Scope

- Endpoint policies apply only to traffic flowing through that specific endpoint
- They do NOT override IAM policies — both must allow
- Default endpoint policy: `Allow *` (all principals, all actions)
- Custom policies can restrict by principal, action, resource, or condition

---

## 6. AWS PrivateLink — Expose Services Without Peering

### The Problem PrivateLink Solves

You have a service (an API, a SaaS product) that you want to expose to consumers in other VPCs or accounts — WITHOUT giving them VPC peering access (which would expose your entire VPC), without using the internet, and without them needing to manage complex routing.

### How PrivateLink Works

**Provider side:**
1. Deploy your service behind a **Network Load Balancer** (NLB)
2. Create a **VPC Endpoint Service** backed by the NLB
3. Grant access to specific AWS accounts or IAM principals

**Consumer side:**
1. Create an **Interface VPC Endpoint** pointing to the provider's endpoint service
2. An ENI is created in the consumer's subnet with a private IP
3. Traffic flows: Consumer ENI → PrivateLink → Provider NLB → Backend service

```
Consumer VPC (10.1.0.0/16)               Provider VPC (10.0.0.0/16)
┌─────────────────────────┐              ┌─────────────────────────┐
│  EC2: 10.1.2.5          │              │  NLB: 10.0.1.10         │
│    ↓                    │              │    ↓                    │
│  ENI: 10.1.3.10         │──PrivateLink─│  API Instances          │
│  (Interface Endpoint)   │              │                         │
└─────────────────────────┘              └─────────────────────────┘
```

**The consumer's 10.x address space and provider's address space can OVERLAP** — there is no peering, no route table, no CIDR conflict issue. PrivateLink is connection-oriented, not route-oriented.

### PrivateLink vs VPC Peering — When to Use Which

| Criteria | VPC Peering | PrivateLink |
|----------|-------------|-------------|
| Expose entire VPC | Yes (all traffic flows) | No (only specific service) |
| CIDR overlap allowed | No | Yes |
| Transitive routing | No | N/A (service-based) |
| Consumer sees provider's private IPs | Yes | No (sees only endpoint IP) |
| Number of consumers | Limited by mesh complexity | Millions (no limit) |
| Use case | Internal VPC-to-VPC communication | SaaS, shared service exposure |
| Route table changes | Required in both VPCs | Not required |

🎯 **EXAM TIP**: "Company wants to expose an internal API to 1000 customer AWS accounts without giving network access to their VPC" → PrivateLink. "Two internal teams need VPC-to-VPC communication with full network access" → VPC Peering or TGW.

### PrivateLink — Accepted Service Names

AWS itself uses PrivateLink for all Interface VPC Endpoints (when you create an interface endpoint for KMS, SQS, etc., you are connecting to AWS's PrivateLink-powered endpoint service).

You can check available endpoint services:
```
com.amazonaws.us-east-1.kms
com.amazonaws.us-east-1.sqs
com.amazonaws.us-east-1.secretsmanager
...
```

Custom (your own services) show as:
```
com.amazonaws.vpce.us-east-1.vpce-svc-xxxxxxxxxxxxxxxxx
```

---

## 7. AWS Site-to-Site VPN — Hybrid Connectivity

### Architecture

```
On-Premises Data Center                      AWS
┌─────────────────────────┐     IPSec/IKE   ┌─────────────────────────┐
│  Customer Gateway (CGW) │←──Encrypted──→  │ Virtual Private Gateway  │
│  (Physical/SW appliance)│   over Internet  │ (VGW) — attached to VPC │
│  Public IP: 203.0.113.1 │                  └─────────────────────────┘
└─────────────────────────┘
```

Every VPN connection creates TWO tunnels (active/standby or active/active with BGP). This provides redundancy within the same connection.

### BGP vs Static Routing

**BGP (dynamic routing):**
- On-premises router advertises prefixes to AWS VGW
- AWS VGW advertises VPC CIDRs back
- Automatic failover between tunnels
- Required for Transit Gateway VPN ECMP
- Requires BGP ASN on Customer Gateway

**Static routing:**
- You manually configure which on-premises prefixes to advertise
- Simpler but no automatic route learning
- Fine for simple single-site connectivity

🎯 **EXAM TIP**: TGW VPN with ECMP requires BGP. If the scenario mentions "BGP not available on on-premises router" and "high throughput needed" — Direct Connect is the answer, not VPN.

### VPN Throughput and Limits

| Metric | Value |
|--------|-------|
| Max per tunnel | 1.25 Gbps |
| Tunnels per VPN connection | 2 (active/standby with VGW; active/active with TGW+BGP) |
| VPN connections per VGW | 10 |
| Max throughput with TGW ECMP | ~50+ Gbps (multiple VPN connections, each with 2 tunnels) |

**With VGW:** Maximum 1.25 Gbps (one active tunnel at a time)
**With TGW + ECMP:** Multiple VPN connections, all tunnels active simultaneously

### VPN CloudHub Pattern

Multiple Customer Gateways connecting to one VGW — branch offices can communicate through the VGW hub:

```
Branch A (CGW-A) ─┐
Branch B (CGW-B) ─┤──→ VGW ──→ VPC
Branch C (CGW-C) ─┘
                  Branch-to-Branch traffic routes through VGW
```

🎯 **EXAM TIP**: VPN CloudHub allows BRANCH-TO-BRANCH communication via the VGW. This is transitive routing through VGW (not through a VPC). The VGW acts as the hub.

### Accelerated VPN

A feature that uses AWS Global Accelerator for VPN traffic — instead of routing over the public internet to the nearest AWS edge, your VPN traffic enters the AWS network at the nearest AWS Global Accelerator point of presence.

Use case: VPN connections from geographically remote locations with poor internet routing to the target region. Reduces latency and jitter.

---

## 8. AWS Client VPN — Remote Access

### What Client VPN Provides

AWS Client VPN is a managed OpenVPN-compatible service that lets individual users (laptops, mobile devices) connect to AWS VPCs and on-premises networks remotely over TLS.

```
Remote User (laptop)
  ↓ OpenVPN client
Client VPN Endpoint (in your VPC)
  ↓ splits into:
  ├── VPC resources (private IPs)
  └── (with split tunnel OFF) → all internet traffic via VPC too
```

### Authentication Options

| Method | Description |
|--------|-------------|
| Active Directory (AWS Managed AD or self-managed AD via AD Connector) | Username/password + MFA |
| SAML 2.0 (Federated identity — Okta, Azure AD, etc.) | SSO-based |
| Mutual Certificate (client + server certificates via ACM) | Certificate-based, no directory needed |

🎯 **EXAM TIP**: Client VPN authentication type determines complexity. "Enforce MFA for VPN access" → use Active Directory or SAML (not mutual certificate). "No directory, just developer laptops" → mutual certificate authentication.

### Split Tunneling

**Split tunnel ON (recommended):**
- Only traffic destined for VPC/on-premises CIDRs goes through VPN
- Internet traffic goes directly from user's ISP
- Lower VPN bandwidth usage, lower cost

**Split tunnel OFF (full tunnel):**
- ALL traffic goes through VPN
- Internet access routes through VPC (need NAT GW or internet access in VPC)
- Compliance requirement: log and inspect all user internet traffic

### Authorization Rules

Client VPN uses authorization rules to control which users (by Active Directory group or certificate) can access which network CIDRs.

```
Rule: AD group "developers" → allowed to reach 10.0.0.0/16 (dev VPC)
Rule: AD group "admins"     → allowed to reach 10.0.0.0/8  (all VPCs)
```

---

## 9. AWS Direct Connect — Dedicated Physical Link

### Why Direct Connect Exists

Site-to-Site VPN sends encrypted traffic over the public internet — bandwidth is variable, latency fluctuates, and data transfer costs are high. Direct Connect is a dedicated physical fiber connection from your data center to an AWS Direct Connect location (colocation facility). Traffic never traverses the public internet.

### Connection Types

**Dedicated Connection:**
- You (or your provider) physically runs fiber to an AWS DX location
- Port speeds: 1 Gbps, 10 Gbps, 100 Gbps
- Lead time: weeks to months (physical provisioning)
- You own the port and can create multiple VIFs (virtual interfaces) on it

**Hosted Connection:**
- An AWS Direct Connect Partner already has fiber to DX location
- You order bandwidth from the partner: 50 Mbps, 100 Mbps, 200 Mbps, 500 Mbps, 1 Gbps, 2 Gbps, 5 Gbps, 10 Gbps
- Faster lead time (weeks, not months)
- One hosted connection = one VIF only

### Virtual Interfaces (VIFs) — 802.1Q VLANs

A Direct Connect connection carries multiple logical channels via VLAN tagging (802.1Q):

| VIF Type | Purpose | Connects to |
|----------|---------|-------------|
| Private VIF | Access VPC private resources | VGW or DX Gateway |
| Public VIF | Access AWS public service endpoints (S3, CloudFront, etc.) | AWS public IP space |
| Transit VIF | Access multiple VPCs via Transit Gateway | DX Gateway + TGW |

🎯 **EXAM TIP**: You can only create ONE Transit VIF per DX connection (this is a hard limit). Multiple Private VIFs can be created up to the port limit. Public VIFs allow access to all AWS public endpoints globally (not just the region the DX is in).

### Direct Connect Gateway (DX Gateway)

A DX Gateway allows one DX connection to reach multiple VPCs across different regions and accounts.

```
DX Connection (us-east-1 DX Location)
  ↓ Private VIF
DX Gateway (global resource)
  ├── VPC-A attachment (us-east-1)    via VGW or TGW
  ├── VPC-B attachment (us-west-2)    via VGW or TGW
  └── VPC-C attachment (eu-west-1)    via VGW or TGW
```

**Important constraint:** VPCs connected to the SAME DX Gateway cannot communicate with each other through it. The DX Gateway is for on-premises to VPC, not VPC to VPC.

### DX Resilience Models and SLA

AWS defines two DX SLA tiers:

| Resilience Model | Architecture | SLA |
|-----------------|--------------|-----|
| **Non-redundant** (development) | Single connection, single location | No SLA |
| **High Resilience** (99.9%) | Two connections at same DX location | 99.9% |
| **Maximum Resilience** (99.99%) | Two connections at TWO DIFFERENT DX locations | 99.99% |

🎯 **EXAM TIP**: To achieve 99.99% SLA, you MUST have DX connections at physically separate DX locations (different cities or facilities). Two connections at the same location give 99.9%. The exam will describe a failure scenario and ask which architecture would have prevented it.

### LAG — Link Aggregation Group

Multiple physical DX connections bundled together to appear as a single connection with aggregated bandwidth:

```
2 × 10 Gbps DX connections → LAG → 20 Gbps logical connection
```

Rules:
- All connections in a LAG must be at the same DX location
- All must be the same port speed
- Minimum active connections configurable (e.g., if only 1 of 2 is up, LAG goes down)

### DX Encryption

Direct Connect is NOT encrypted by default. The fiber connection is private (no internet) but not encrypted.

Options to add encryption:
1. **VPN over DX**: Run IPSec VPN tunnel over the DX connection. DX handles the bandwidth, VPN handles encryption. This requires a Public VIF (VPN endpoint is a public IP).
2. **MACsec (Layer 2 encryption)**: Available on dedicated 10 Gbps and 100 Gbps connections. Encrypts at the Ethernet frame level. Requires MACsec-capable hardware on both ends.
3. **Application-level TLS**: HTTPS, TLS 1.2/1.3 at the application layer.

🎯 **EXAM TIP**: "DX with encryption" → VPN over DX (uses Public VIF) or MACsec. "DX without encryption but compliance requires private network" → DX alone satisfies "private network" but NOT encryption. These are different requirements.

---

## 10. On-Premises Redundant Connections — DX + VPN Patterns

### Pattern 1: Single DX with VPN Backup

Most cost-effective redundancy for non-critical but important workloads:

```
Primary:  On-Premises → DX Connection → VGW → VPC       (1–10 Gbps, low latency)
Backup:   On-Premises → Site-to-Site VPN → VGW → VPC    (1.25 Gbps, higher latency)
```

**How failover works:**
- With BGP: DX routes have lower MED (metric), preferred by default. When DX fails, BGP withdraws routes → VPN routes take over automatically.
- With static: You cannot automate this without scripting. BGP is strongly preferred for hybrid setups.

**Cost trade-off:** Cheapest HA option but VPN is the backup path. VPN throughput (1.25 Gbps) may not match DX bandwidth. If DX is 10 Gbps, failover to VPN causes significant degradation.

### Pattern 2: Dual DX Connections (99.9% SLA)

```
Primary:   On-Premises → DX-Connection-1 (DX Location A) → DX GW → VPC
Secondary: On-Premises → DX-Connection-2 (DX Location A) → DX GW → VPC
```

Two connections at same location. Protects against port/device failure, not against location failure.

### Pattern 3: Dual DX at Different Locations (99.99% SLA)

```
Primary:   On-Premises → DX-Connection-1 (DX Location A: Chicago) → DX GW → VPC
Secondary: On-Premises → DX-Connection-2 (DX Location B: Dallas)  → DX GW → VPC
```

Protects against location-level failure. Requires your on-premises to have connectivity paths to both DX locations (typically via different ISPs/carriers).

### Pattern 4: Dual DX + VPN (Maximum Resilience)

```
Tier 1: DX-Connection-1 (Location A) → 10 Gbps primary
Tier 2: DX-Connection-2 (Location B) → 10 Gbps secondary
Tier 3: Site-to-Site VPN → 1.25 Gbps emergency backup
```

BGP metric configuration:
- DX: lowest metric (most preferred)
- VPN: highest metric (least preferred, only used if both DX fail)

### ECMP with Multiple VPN Connections to TGW

For higher VPN throughput without DX:

```
On-Premises BGP Router
  ├── VPN Connection 1 (Tunnel A: 1.25 Gbps, Tunnel B: 1.25 Gbps)
  ├── VPN Connection 2 (Tunnel C: 1.25 Gbps, Tunnel D: 1.25 Gbps)
  └── VPN Connection 3 (Tunnel E: 1.25 Gbps, Tunnel F: 1.25 Gbps)
           ↓ ECMP (all active simultaneously)
      Transit Gateway
```

Total throughput: up to 6 × 1.25 Gbps = 7.5 Gbps (limited by TGW and on-premises router)

🎯 **EXAM TIP**: ECMP on TGW requires BGP and Equal-Cost routes. All VPN connections must advertise the same prefixes with equal BGP attributes for ECMP to load balance.

---

## 11. VPC Flow Logs — Capture, Query, Alert

### What Flow Logs Capture

Flow logs capture IP traffic METADATA — not the packet payload. They record connection-level information:

```
Default log record fields:
version | account-id | interface-id | srcaddr | dstaddr | srcport | dstport |
protocol | packets | bytes | start | end | action | log-status
```

**Custom fields (newer format, specify when creating):**
```
vpc-id | subnet-id | instance-id | tcp-flags | type | pkt-srcaddr |
pkt-dstaddr | region | az-id | sublocation-type | sublocation-id |
pkt-src-aws-service | pkt-dst-aws-service | flow-direction | traffic-path
```

The `traffic-path` field tells you HOW traffic left the VPC:
- `1` = Through an IGW
- `4` = Through a VGW
- `8` = Through a local gateway
- `17` = Through a Gateway VPC Endpoint
- `18` = Through an Interface VPC Endpoint

🎯 **EXAM TIP**: Flow logs do NOT capture: DNS queries to Route53 resolver, DHCP traffic, instance metadata (169.254.169.254), Windows license activation traffic, or traffic between NLB and targets (captured separately via NLB access logs).

### Flow Log Destinations

| Destination | Latency | Use Case |
|-------------|---------|----------|
| CloudWatch Logs | ~1 min (ENI), ~10 min (VPC/subnet) | Real-time alerting via CloudWatch Alarms or Metric Filters |
| S3 | ~5–15 min | Long-term storage, Athena analysis, cost-effective |
| Kinesis Data Firehose | Near real-time | Streaming to Splunk, OpenSearch, Datadog, etc. |

### Querying with Athena

Store flow logs in S3, create Athena table, query with SQL:

```sql
-- Find all rejected traffic to port 22 from outside VPC
SELECT srcaddr, COUNT(*) as attempts
FROM vpc_flow_logs
WHERE dstport = 22
  AND action = 'REJECT'
  AND srcaddr NOT LIKE '10.%'
GROUP BY srcaddr
ORDER BY attempts DESC
LIMIT 20;
```

```sql
-- Top data consumers (most bytes sent from VPC)
SELECT srcaddr, SUM(bytes) as total_bytes
FROM vpc_flow_logs
WHERE action = 'ACCEPT'
  AND start BETWEEN 1700000000 AND 1700086400
GROUP BY srcaddr
ORDER BY total_bytes DESC;
```

### Flow Log Filtering — ACCEPT, REJECT, ALL

- `ACCEPT`: Only log accepted traffic (smaller volume, misses security events)
- `REJECT`: Only log rejected traffic (useful for security monitoring, finding port scans)
- `ALL`: Log everything (most complete, highest cost)

🎯 **EXAM TIP**: "Detect unauthorized access attempts to RDS" → Flow Logs with REJECT filter to CloudWatch → Metric Filter for port 3306 rejected → CloudWatch Alarm → SNS notification. This is the canonical security monitoring pattern.

---

## 12. AWS Network Firewall — Stateful Deep Inspection

### What Network Firewall Provides

Network Firewall is a managed, stateful network firewall service that you deploy INTO a VPC subnet. It goes beyond Security Groups and NACLs by supporting:

- **Layer 7 inspection**: Domain name filtering, HTTP header inspection
- **Stateful rule groups**: Suricata-compatible IDS/IPS rules
- **Stateless rule groups**: Simple 5-tuple ACL rules (faster, no state tracking)
- **Managed threat intelligence**: AWS-managed rule groups (IPs, domains known to be malicious)

### Network Firewall Architecture Components

```
Firewall Endpoint (ENI in firewall subnet)
  ↑ traffic routed to it
Firewall Policy
  ├── Stateless Rule Groups (evaluated first, in priority order)
  │   ├── Pass → skip stateful inspection
  │   ├── Drop → discard packet
  │   └── Forward to stateful → continue to stateful rules
  └── Stateful Rule Groups (evaluated for forwarded traffic)
      ├── Suricata-compatible rules
      ├── Domain list rules (allow/deny by FQDN)
      └── Standard 5-tuple rules
```

### Deployment Models

**Distributed deployment:**
```
Each VPC has its own Network Firewall
Traffic inspected locally before leaving VPC
```
Pros: Simple routing, low latency, no cross-VPC dependencies
Cons: Higher cost (firewall per VPC), harder to centralize policy

**Centralized deployment (via Transit Gateway):**
```
All VPCs → TGW → Inspection VPC (with Network Firewall) → Egress VPC (with NAT GW + IGW)
```
See Architecture Patterns section for full diagram.

Pros: Single firewall policy for all VPCs, cost-effective at scale
Cons: More complex routing, TGW data transfer charges

### Suricata Rules — Examples

Network Firewall uses Suricata rule syntax for stateful rules:

```
# Block DNS queries to known malware domains
drop dns $HOME_NET any -> any 53 (dns.query; content:"malware-c2.com"; nocase; sid:1001;)

# Alert on outbound SSH to unexpected hosts
alert tcp $HOME_NET any -> !$HOME_NET 22 (msg:"Outbound SSH detected"; sid:1002;)

# Block HTTP to non-corporate domains
pass http $HOME_NET any -> $HTTP_SERVERS any (http.host; content:"internal.company.com"; sid:1003;)
drop http $HOME_NET any -> any 80 (msg:"Block non-internal HTTP"; sid:1004;)
```

🎯 **EXAM TIP**: The SAP exam does not require you to write Suricata rules, but does test: (1) when Network Firewall is the right choice vs Security Groups/NACLs/WAF, (2) centralized vs distributed deployment trade-offs, and (3) Network Firewall vs third-party firewall appliances.

---

## 13. Architecture Patterns — Egress, Inspection, Hybrid DNS

### Pattern 1: Centralized Egress Through TGW

**Problem:** 20 VPCs each need internet egress. You could put a NAT Gateway in each VPC (expensive), or centralize through a dedicated Egress VPC.

```
VPC-1 (10.0.0.0/16)  ─┐
VPC-2 (10.1.0.0/16)  ─┤
VPC-3 (10.2.0.0/16)  ─┤──→ TGW ──→ Egress VPC
...                    │              ├── NAT Gateway (per AZ)
VPC-N (10.N.0.0/16)  ─┘              └── Internet Gateway

TGW Route Table (spoke-rt):
  10.0.0.0/8  → local VPC attachments
  0.0.0.0/0   → Egress VPC attachment

Egress VPC Route Table (private):
  10.0.0.0/8  → TGW attachment
  0.0.0.0/0   → NAT Gateway

Egress VPC Route Table (public):
  10.0.0.0/8  → TGW attachment
  0.0.0.0/0   → IGW
```

**Cost savings:** 3 AZs × $0.045/hr = ~$99/month per VPC. With 20 VPCs, that's ~$1,980/month. Centralized: 3 NAT Gateways = ~$99/month total for NAT (plus TGW data transfer).

🎯 **EXAM TIP**: Centralized egress requires asymmetric routing awareness — return traffic must come back through the TGW to the originating VPC. Ensure TGW route tables have the correct VPC CIDR return routes.

### Pattern 2: Centralized Inspection (Security VPC)

**Problem:** Inspect all traffic (east-west between VPCs, and north-south to internet) through a central security appliance.

```
Spoke VPCs ──→ TGW ──→ Inspection VPC (Network Firewall / NGFW appliance)
                              ↓
                        TGW (back to destination VPC)
                              OR
                        Egress VPC (for internet-bound)

TGW Route Table Configuration:
  Spoke VPCs → all traffic (0.0.0.0/0 and inter-VPC) → Inspection VPC attachment
  Inspection VPC → post-inspection traffic → destination attachment
```

This requires "appliance mode" on the TGW VPC attachment to ensure symmetric routing (same Firewall endpoint processes both request and response).

🎯 **EXAM TIP**: Enable **Appliance Mode** on the TGW attachment to the Inspection VPC. Without it, asymmetric routing breaks stateful inspection (request goes through Firewall ENI in AZ-a, response goes through AZ-b ENI — stateful session is lost).

### Pattern 3: Hybrid DNS with Route 53 Resolver

**Problem:** On-premises DNS resolves `internal.company.com` using an on-premises DNS server. AWS resources need to resolve these names. AWS resources have `*.us-east-1.amazonaws.com` names that on-premises servers cannot resolve.

**Solution: Route 53 Resolver with Inbound and Outbound Endpoints**

```
On-Premises DNS Server (192.168.1.53)
  ├── Forwards "company.aws.internal" queries → Route53 Resolver Inbound Endpoint
  │   (10.0.2.10 — ENI in your VPC)
  │   → Route53 Private Hosted Zone resolves → returns private IP
  │
AWS Route53 Resolver Outbound Endpoint
  (10.0.2.20 — ENI in your VPC)
  ├── Forwards "internal.company.com" queries → On-Premises DNS (192.168.1.53)
  │   → On-Premises DNS resolves → returns on-premises IP
```

**Route 53 Resolver Forwarding Rules:**

```
Forwarding Rule: "internal.company.com" → 192.168.1.53 (on-premises DNS)
Forwarding Rule: "corp.local"           → 192.168.1.53 (on-premises DNS)
System Rule:     "amazonaws.com"        → Route53 (default AWS DNS)
```

🎯 **EXAM TIP**: Route 53 Resolver endpoints are ENIs in your VPC subnets. They consume IP addresses and must have security groups. Inbound endpoint = on-premises queries to AWS. Outbound endpoint = AWS queries forwarded to on-premises. Both can coexist.

**Sharing Resolver Rules via RAM:** Create Resolver forwarding rules in a central account, share via AWS RAM to all accounts in your Organization. All accounts then resolve on-premises hostnames without each needing their own outbound endpoints.

---

## 14. Decision Framework — VPC Connectivity Options Comparison

### Master Connectivity Decision Table

| Criteria | VPC Peering | Transit Gateway | PrivateLink | Site-to-Site VPN | Direct Connect |
|----------|-------------|-----------------|-------------|------------------|----------------|
| **Scale** | Small (2–5 VPCs) | Large (100s of VPCs) | Millions of consumers | Single site | Single site |
| **Transitive routing** | No | Yes | N/A (service-based) | No (per VGW) | No (per DX GW) |
| **CIDR overlap allowed** | No | No | Yes | No | No |
| **Cross-account** | Yes | Yes (via RAM) | Yes | Yes | Yes |
| **Cross-region** | Yes (peering) | Yes (TGW peering) | Yes (inter-region endpoint) | Yes | Yes (DX GW) |
| **On-premises connectivity** | No | Yes (VPN/DX attach) | No | Yes | Yes |
| **Bandwidth** | Up to 10 Gbps | 50 Gbps/VPC | NLB throughput | 1.25 Gbps/tunnel | 1–100 Gbps |
| **Latency** | Lowest (direct) | Low (one extra hop) | Low | Variable (internet) | Consistent (dedicated) |
| **Encryption** | Yes (AWS backbone) | Yes (AWS backbone) | Yes (AWS backbone) | Yes (IPSec) | No (add VPN or MACsec) |
| **Setup time** | Minutes | Minutes | Minutes | Minutes | Weeks–Months |
| **Cost (data transfer)** | Free same-region | TGW attachment fee | Endpoint hourly | None (internet costs) | Lower than internet |
| **Complexity** | Low | Medium | Low | Low | High |

### Connectivity Decision Tree

```
Need to connect...

├── Two VPCs privately?
│   ├── Same account, same region, small scale → VPC Peering
│   └── Many VPCs, multi-account, need routing control → Transit Gateway

├── Expose a service to other accounts without full network access?
│   ├── One service, controlled access, CIDR overlap OK → PrivateLink
│   └── Full network communication needed → VPC Peering or TGW

├── Connect on-premises to AWS?
│   ├── Quick setup, acceptable internet routing → Site-to-Site VPN
│   ├── Need >1.25 Gbps VPN throughput → TGW + multiple VPN + ECMP
│   ├── Mission-critical, consistent latency, high bandwidth → Direct Connect
│   ├── DX + encryption required → VPN over DX (Public VIF) or MACsec
│   └── Maximum resilience (99.99%) → Dual DX at different locations

├── Remote user access to VPC?
│   └── AWS Client VPN (OpenVPN-based, per-user authentication)

└── Access AWS services privately from VPC?
    ├── S3 or DynamoDB → Gateway Endpoint (free)
    └── Any other AWS service → Interface Endpoint (PrivateLink)
```

### SAP Scenario Quick Reference

| Scenario | Answer |
|----------|--------|
| 50 VPCs need full-mesh connectivity | Transit Gateway (not VPC peering) |
| Company A wants to sell API access to Company B without exposing VPC | PrivateLink |
| On-premises needs S3 access privately (no internet) | Interface Endpoint for S3 + Private VIF on DX or VPN |
| VPN throughput needs to exceed 1.25 Gbps | TGW + multiple VPN connections + ECMP + BGP |
| DX encryption in transit | VPN over DX (Public VIF) or MACsec (10/100 Gbps dedicated only) |
| 99.99% DX resilience | Two DX connections at different physical locations |
| Central network team manages routing for 200 accounts | TGW shared via AWS RAM |
| On-premises DNS must resolve AWS private hostnames | Route 53 Resolver Inbound Endpoint |
| AWS instances must resolve on-premises hostnames | Route 53 Resolver Outbound Endpoint + Forwarding Rule |
| Inspect all internet egress from 30 VPCs with one firewall | Centralized Egress VPC + TGW + Network Firewall |
| DX non-transitive VPC-to-VPC via DX GW | Not possible — use TGW instead |
| Detect port scan from external IP, block it | VPC Flow Logs (REJECT) + NACL DENY rule |
| Whitelist specific S3 buckets accessible via endpoint | VPC Endpoint Policy on S3 Gateway Endpoint |
| TGW cross-region routing | TGW Peering with static routes (no BGP between TGWs) |
| Multicast traffic routing in AWS | Transit Gateway (Multicast Domain) |

---

## Key SAP Exam Themes for VPC

1. **CIDR Planning**: Overlapping CIDRs prevent peering. Plan enterprise-wide allocation upfront.

2. **Transitive routing**: VPC Peering is NEVER transitive. TGW IS the routing hub. DX Gateway is NOT for VPC-to-VPC.

3. **TGW Route Table segmentation**: Multiple route tables on one TGW = network policy enforcement. Prod cannot reach Dev unless you explicitly add routes.

4. **DX resilience tiers**: Know the difference between 99.9% (single location, two connections) and 99.99% (two locations). Physical location separation is key.

5. **VPN ECMP**: Only works with TGW (not VGW), requires BGP, aggregates bandwidth across multiple VPN connections.

6. **PrivateLink vs Peering**: PrivateLink for service exposure at scale. Peering for network-level connectivity. CIDR overlap only acceptable with PrivateLink.

7. **Endpoint policies + bucket policies**: The dual-layer data exfiltration prevention model. Endpoint policy restricts which services are reachable; bucket policy restricts who can access (enforces endpoint usage).

8. **Hybrid DNS**: Route 53 Resolver endpoints bridge AWS and on-premises DNS. Inbound = on-prem queries to AWS. Outbound = AWS queries to on-prem. Share rules via RAM.

9. **Centralized architecture patterns**: Inspection VPC requires TGW Appliance Mode for symmetric routing. Egress VPC saves NAT GW costs at scale.

10. **Network Firewall deployment**: Distributed for simplicity, Centralized via TGW for cost and policy control at scale.
