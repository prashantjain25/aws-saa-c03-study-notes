# Amazon CloudFront & AWS Global Accelerator

> **Exam**: AWS SAA-C03 | **Topic Weight**: Medium-High — expect 3–5 questions on CDN, caching, Global Accelerator comparisons

## Official AWS Documentation

- [Amazon CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)
- [AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html)
- [CloudFront Edge Locations](https://aws.amazon.com/cloudfront/features/)
- [CloudFront Origin Access Control](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)

---

## Why CloudFront Exists — The Problem of Distance

Imagine your application runs in `ap-south-1` (Mumbai). A user in São Paulo, Brazil opens your website. Their request travels ~17,000 km across the public internet — dozens of hops through routers, undersea cables, and ISP networks. Each hop adds latency, and the public internet is unpredictable.

**The solution**: Place copies of your content physically close to every user in the world. That's exactly what a **Content Delivery Network (CDN)** does.

```
WITHOUT CloudFront:
User (São Paulo) ────────────── 17,000 km, many hops ──────────────► Origin (Mumbai)
                                  latency: 300–400ms

WITH CloudFront:
User (São Paulo) ──► Edge Location (São Paulo)  ← content cached here!
                     latency: < 5ms (local cache hit)
                     │
                     │ (only on cache MISS)
                     └──────────────────────────────► Origin (Mumbai)
```

CloudFront is AWS's global CDN. It operates 216+ Points of Presence (edge locations) spread across the world. Users connect to the closest edge location; content is served from cache, dramatically reducing latency and origin load.

**Extra benefits**:
- Integrated **DDoS protection** via AWS Shield (Standard, always-on)
- **WAF** (Web Application Firewall) integration to filter malicious requests at the edge
- HTTPS termination at edge (reduces encryption overhead on origin)
- Cost savings — you pay for less data transfer from origin

---

## Section 1 — CloudFront Origins

An **Origin** is where CloudFront fetches content when it's not in the edge cache. CloudFront supports three origin types:

### 1. Amazon S3 Bucket (with OAC)

The most common use case: host a static website or file distribution on S3, let CloudFront cache and serve it globally.

**Origin Access Control (OAC)** is the critical security piece here. Without it, your S3 bucket would need to be public to serve content — anyone who finds the bucket URL bypasses CloudFront and accesses content for free (no caching, no DDoS protection). With OAC, the bucket stays **private**; only CloudFront can read from it.

```
User
  │
  ▼
CloudFront Distribution
  │
  │ (via OAC — signed requests only CloudFront can make)
  ▼
S3 Bucket (private, bucket policy allows only CloudFront OAC)
```

**OAC vs legacy OAI**: OAC (Origin Access Control) is the modern replacement for OAI (Origin Access Identity). Prefer OAC — it supports all S3 regions, SSE-KMS encryption, and PUT/POST operations.

**Bonus feature**: CloudFront can also be used to UPLOAD to S3 (ingress), not just download. Files go through the nearest edge location and are forwarded to S3 over AWS's internal network, which is faster and more reliable than uploading directly over the public internet.

### 2. VPC Origins (Private ALB / NLB / EC2)

**New capability**: CloudFront can deliver content from resources running in **private subnets** in your VPC — no need to expose them to the internet at all.

This is a powerful security pattern: your backend (ALB, NLB, or EC2 instances) stays completely private. CloudFront reaches them through AWS-internal networking.

```
Users
  │
  ▼
CloudFront Distribution (Edge Location)
  │
  ▼ (via VPC Origin — private AWS network)
┌──────────────── VPC ─────────────────────┐
│                                           │
│  ┌──────────────── Private Subnet ──────┐ │
│  │                                      │ │
│  │  [VPC Origin]──► Application LB      │ │
│  │              ──► Network LB          │ │
│  │              ──► EC2 Instance        │ │
│  └──────────────────────────────────────┘ │
└───────────────────────────────────────────┘
```

### 3. Custom HTTP Origin (Public Network)

When your origin must be reachable over the public network — for example, a public ALB or public EC2 instances.

**Security requirement**: CloudFront edge locations have known public IPs. Your EC2 or ALB security groups must explicitly **allow traffic from CloudFront edge location IPs**.

AWS publishes the full list of CloudFront IP ranges at:
`http://d7uri8nf7uskq.cloudfront.net/tools/list-cloudfront-ips`

**Two common sub-patterns:**

```
PATTERN A — EC2 as Origin (must be public):

Users ◄──► Edge Location ────(public internet)────► EC2 Instance
                                                    [Security Group:
                                                     Allow CloudFront IPs]
                                                    [Must have public IP]

PATTERN B — ALB as Origin (EC2 can be private):

Users ◄──► Edge Location ────(public internet)────► ALB (must be public)
                                                     │ [SG: Allow CF IPs]
                                                     │
                                                     ▼
                                                  EC2 Instances
                                                  [Can be private!]
                                                  [SG: Allow ALB SG]
```

Pattern B is better for security: only the ALB is exposed; EC2 instances stay private behind the ALB's security group.

---

## Section 2 — CloudFront vs S3 Cross-Region Replication

These two are frequently compared on the exam. They sound similar (both spread content globally) but serve entirely different purposes.

| Feature | Amazon CloudFront | S3 Cross-Region Replication (CRR) |
|---------|------------------|------------------------------------|
| Mechanism | **CDN** — caches copies at edge locations | **Replication** — copies objects to other S3 buckets |
| Scope | **216+ edge locations** globally | Only the specific regions you configure |
| Latency | Near-zero (served from cache) | Near real-time sync (~minutes), read from regional S3 |
| Content freshness | Files cached for a **TTL** (e.g., 1 day) | Files updated in **near real-time** |
| Content type | **Static content** (images, videos, JS, CSS) that changes rarely | **Dynamic content** that changes often |
| Read vs write | Read-optimised caching | Read-only in destination |
| Setup | Applies globally automatically | Must configure per destination region |
| Best for | **Global** distribution, static assets | **Low-latency reads in a few specific regions** |

**The exam heuristic**:
- "Must be available globally, changes infrequently" → **CloudFront**
- "Must be available in 2–3 specific regions, updates in near real-time" → **S3 CRR**

---

## Section 3 — CloudFront Geo Restriction

Sometimes you legally cannot serve content in certain countries (copyright laws, export controls, government regulations). CloudFront's **Geo Restriction** feature lets you control access by country.

The country is determined using a **3rd-party Geo-IP database** that maps IP addresses to countries.

**Two modes:**

- **Allowlist**: Only users in the listed approved countries can access your distribution. Everyone else gets a 403 Forbidden.
- **Blocklist**: Users in banned countries are blocked. Everyone else can access.

**Use cases:**
- A film studio licensing content only in specific countries (allowlist)
- Complying with sanctions by blocking certain countries (blocklist)
- Enforcing regional licensing agreements

**Configuration**: Set at the CloudFront distribution level under "Geo restrictions" — select mode (allowlist/blocklist) and specify countries using ISO country codes.

---

## Section 4 — CloudFront Cache Invalidations

### The Stale Cache Problem

Here's the dilemma with caching: CloudFront caches your content at edge locations based on a **TTL (Time-to-Live)**. If your TTL is 1 day and you update a file on the origin, users will still see the OLD version for up to 24 hours.

This is fine for stable assets, but what about urgent updates — a critical bug fix in JavaScript, correcting wrong pricing on a product page, or a security patch?

**CloudFront Invalidation** solves this: you force CloudFront to discard its cached copies immediately, regardless of the TTL. The next request to any edge location will be a cache miss — CloudFront fetches the fresh version from origin.

### How Invalidation Works

```
Admin updates index.html in S3 bucket (origin)
                │
                │ Create CloudFront Invalidation:
                │   /index.html
                │   /images/*
                │
                ▼
CloudFront propagates invalidation to ALL Edge Locations
                │
  ┌─────────────┼──────────────┐
  │             │              │
  ▼             ▼              ▼
Edge Loc 1  Edge Loc 2   Edge Loc 3
[index.html  [index.html  [index.html
 DELETED]     DELETED]     DELETED]
                │
Next user request → cache MISS → fresh fetch from S3
```

**Invalidation paths:**
- `/index.html` — invalidate a single specific file
- `/images/*` — invalidate everything under the `/images/` path (wildcard)
- `/*` — invalidate ALL cached files in the distribution

**Cost**: You get the first 1,000 invalidation paths per month free, then you pay per path. Using wildcards (`/images/*`) counts as ONE path — much more cost-effective than listing every file.

**Best practice**: Use versioned file names in production (`main.v2.js` instead of `main.js`) so you never need to invalidate — the new URL means a fresh cache. Reserve invalidations for emergencies.

---

## Section 5 — AWS Global Accelerator

### The Problem — Public Internet Hops

Even when your origin is on AWS, users around the world still reach it over the **public internet**. The public internet is not optimized for latency — packets bounce through many hops, each introducing delay and potential packet loss.

```
WITHOUT Global Accelerator:
User (Australia) ──(many hops over public internet)──► ALB (India)
                  ← unreliable, high latency, many hops →

WITH Global Accelerator:
User (Australia) ──► Edge Location (Australia)
                         │
                         │ AWS Private Network
                         │ (optimized, reliable, low latency)
                         ▼
                      ALB (India)  ← short final hop to origin
```

Global Accelerator gets traffic onto the **AWS private backbone network** as quickly as possible (at the nearest edge location to the user). AWS's internal network is purpose-built for low latency and high reliability — it avoids the unpredictable public internet for most of the journey.

### Anycast IP — The Magic Behind It

Understanding Anycast is key to understanding Global Accelerator.

**Unicast IP** (normal networking): one server, one IP address. Traffic from any location goes to that one server.

**Anycast IP**: multiple servers worldwide share the **same IP address**. The internet's routing protocol (BGP) automatically sends traffic to the nearest server holding that IP.

```
UNICAST — One IP, One Server:
Client → 12.34.56.78 → specific server in us-east-1
         (same IP always goes to same place)

ANYCAST — Same IP, Many Servers (routed to nearest):
Client (US)        → 12.34.56.78 → Edge Location (US)
Client (Australia) → 12.34.56.78 → Edge Location (Australia)
Client (Europe)    → 12.34.56.78 → Edge Location (Europe)
(same IP, different physical destination — routed to nearest)
```

Global Accelerator creates **2 Anycast IPs** for your application. Users worldwide all use these same 2 IPs. BGP routing automatically sends them to the nearest AWS edge location, where traffic enters the AWS private network.

### Global Accelerator Architecture

```
Users (anywhere in the world)
        │
        │ connects to same 2 Anycast IPs
        │ BGP routes to nearest edge location
        ▼
┌──── AWS Edge Locations ────────────────────────────────┐
│                                                         │
│   Edge (US)    Edge (Europe)    Edge (Australia)        │
│      │               │                │                 │
└──────┼───────────────┼────────────────┼─────────────────┘
       │               │                │
       └───────────────┼────────────────┘
                       │ AWS Private Backbone Network
                       │ (optimized, low latency)
                       ▼
                  Your Application
               (Public ALB in India)
                  [any AWS region]
```

### Global Accelerator Features

**Endpoint types supported**: Elastic IPs, EC2 instances, ALB, NLB — public or private.

**Consistent Performance:**
- Intelligent routing to the lowest-latency healthy endpoint
- Fast regional failover (sub-minute) when an endpoint becomes unhealthy
- **No DNS caching issues** — because the Anycast IPs never change, there's no TTL problem like with DNS-based routing. A DNS change for failover can take minutes/hours to propagate; Anycast routing changes instantly.
- Traffic stays on AWS's internal network for the long haul

**Health Checks:**
- Global Accelerator continuously monitors your application endpoints
- Unhealthy endpoints are automatically removed from routing in under 1 minute
- Great for **disaster recovery** across regions — automatic failover

**Security:**
- Only 2 external IPs need to be whitelisted by firewalls (simplifies security rules)
- DDoS protection via **AWS Shield** (always-on)

---

## Section 6 — CloudFront vs Global Accelerator

Both services use the AWS global edge network. Both integrate with Shield for DDoS protection. But they solve fundamentally different problems.

| Feature | Amazon CloudFront | AWS Global Accelerator |
|---------|------------------|-----------------------|
| Purpose | **Content caching** at the edge | **Network routing** via AWS backbone |
| What gets served | **Content** (files, HTML, images) served from edge | **Packets proxied** to your application |
| Protocol | HTTP/HTTPS | TCP, UDP (and HTTP) |
| Caching | **Yes** — core feature | **No** — not a CDN |
| Static IPs | No (uses domain names) | **Yes — 2 static Anycast IPs** |
| Best for (HTTP) | Cacheable content, API acceleration, static sites | Deterministic failover, static IP requirements |
| Best for (non-HTTP) | — | **Gaming (UDP), IoT (MQTT), VoIP** |
| DNS dependency | Yes (CNAME to CloudFront domain) | **No** — Anycast IPs, no DNS issues |
| Regional failover | Via Route 53 health checks + CloudFront origins | **Built-in, < 1 minute** |

**The exam decision rule:**

- "Serve images/videos/HTML faster globally" → **CloudFront**
- "Low latency for dynamic, non-cacheable traffic" → **Global Accelerator**
- "Need static IPs for my application" → **Global Accelerator** (Anycast IPs)
- "Gaming server, IoT, MQTT, UDP traffic" → **Global Accelerator**
- "Fast regional failover with static IP" → **Global Accelerator**
- "Need HTTP caching to reduce origin load" → **CloudFront**

---

## Section 7 — CloudFront Pricing & Price Classes

Edge location pricing varies by geographic region. Regions in North America and Europe are cheapest; South America and Australia are most expensive.

**Price Classes** let you limit which edge locations CloudFront uses to control costs:

| Price Class | Edge Locations | Cost | Performance |
|-------------|---------------|------|-------------|
| **All** | All 216+ worldwide | Highest | Best (lowest latency globally) |
| **Price Class 200** | All except most expensive regions | Medium | Good |
| **Price Class 100** | US, Mexico, Canada, Europe, Israel only | Lowest | Limited — high latency in Asia/Pacific/SA |

**Exam tip**: Price Class 100 = cheapest but sacrifices performance for users in Asia, South America, Australia. Choose "All" when global performance matters most.

---

## CloudFront Decision Flowchart

```
Need to deliver content faster globally?
│
├─ Static assets (JS, CSS, images, videos)?
│   └─ Amazon CloudFront (with S3 origin + OAC)
│
├─ Dynamic API responses (personalized, non-cacheable)?
│   └─ CloudFront (with ALB/EC2 origin) — still reduces latency
│      OR Global Accelerator (if non-HTTP, or need static IPs)
│
├─ Restrict access by country?
│   └─ CloudFront Geo Restriction (allowlist or blocklist)
│
├─ Keep backend private (not public internet)?
│   └─ CloudFront with VPC Origin (new feature)
│
├─ Need static IP addresses for your application?
│   └─ AWS Global Accelerator (2 Anycast IPs)
│
├─ Gaming / UDP / MQTT / non-HTTP workloads?
│   └─ AWS Global Accelerator
│
└─ Fast disaster recovery across regions?
    └─ AWS Global Accelerator (health checks, < 1 min failover)
```

---

## Points to Remember (Exam Focus)

1. **CloudFront = CDN**. Cache content at 216+ edge locations. Reduces origin load and user latency.

2. **OAC (Origin Access Control)** → S3 bucket stays private; only CloudFront can access it via signed requests. Always prefer OAC over legacy OAI.

3. **VPC Origins** → CloudFront can deliver from private ALB/NLB/EC2 in private subnets — no public exposure needed.

4. **CloudFront + public EC2** → EC2 must have a public IP; security group must allow CloudFront edge IPs.

5. **CloudFront + ALB** → ALB must be public; EC2 behind ALB can be private (ALB SG allows CloudFront IPs, EC2 SG allows ALB SG).

6. **CloudFront vs S3 CRR**: CloudFront = global static caching (TTL-based). S3 CRR = near real-time sync to specific regions (read-only).

7. **Geo Restriction** → Allowlist (approve countries) or Blocklist (ban countries). Uses 3rd-party Geo-IP database.

8. **Cache Invalidation** → Force cache refresh before TTL expires. Wildcard `/images/*` = 1 path (cost-efficient). Best practice: use versioned filenames instead.

9. **Global Accelerator** → Routes traffic over AWS private backbone using 2 Anycast IPs. No caching. Sub-minute failover.

10. **Anycast IP** → Same IP advertised from multiple edge locations; BGP routes user to nearest. This is how Global Accelerator's static IPs work globally.

11. **CloudFront = HTTP(S) caching / Global Accelerator = TCP/UDP routing.** Global Accelerator for UDP, gaming, MQTT, static IPs, deterministic failover.

12. **DDoS protection**: Both CloudFront and Global Accelerator integrate with AWS Shield Standard (always-on, free).

---

## Interview Tips

| Scenario | Answer |
|----------|--------|
| "Serve static website content with low latency globally" | Amazon CloudFront with S3 origin + OAC |
| "Restrict content to only US and UK users" | CloudFront Geo Restriction (Allowlist: US, UK) |
| "Update deployed JavaScript immediately, don't wait for TTL" | CloudFront Cache Invalidation (`/app.js`) |
| "Keep backend EC2 instances private, serve via CloudFront" | CloudFront VPC Origin |
| "Need a static IP that works globally for my ALB" | AWS Global Accelerator (2 Anycast IPs) |
| "Gaming platform needs UDP routing with global reach" | AWS Global Accelerator |
| "Regional failover in under 1 minute" | AWS Global Accelerator (health checks) |
| "Content licensed only for Europe, block all others" | CloudFront Geo Restriction (Allowlist: European countries) |
| "S3 bucket getting direct access bypassing CloudFront" | Enable OAC on CloudFront + bucket policy denies public access |
| "Both CloudFront and GA use edge locations — what's the difference?" | CloudFront caches content AT the edge; GA routes packets THROUGH the edge to the origin |

## Quick Reference — AWS CLI Commands

### CloudFront Commands

```bash
# Create an Origin Access Control (OAC) for S3
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "my-oac",
    "Description": "OAC for S3 bucket",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'

# Create a CloudFront distribution (S3 origin with OAC)
aws cloudfront create-distribution \
  --distribution-config '{
    "Origins": {
      "Quantity": 1,
      "Items": [{
        "Id": "my-s3-origin",
        "DomainName": "my-bucket.s3.us-east-1.amazonaws.com",
        "S3OriginConfig": {"OriginAccessIdentity": ""},
        "OriginAccessControlId": "E1234567890ABC"
      }]
    },
    "DefaultCacheBehavior": {
      "TargetOriginId": "my-s3-origin",
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
    },
    "Comment": "My CDN",
    "DefaultRootObject": "index.html",
    "Enabled": true,
    "CallerReference": "my-distribution-20260306"
  }'

# List CloudFront distributions
aws cloudfront list-distributions

# Create a cache invalidation (force refresh)
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths '/index.html' '/images/*'

# Invalidate everything
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths '/*'

# Check invalidation status
aws cloudfront list-invalidations \
  --distribution-id E1234567890ABC

# Enable CloudFront distribution
aws cloudfront update-distribution \
  --id E1234567890ABC \
  --distribution-config file://updated-config.json \
  --if-match ETAG123ABC
```

### Global Accelerator Commands

```bash
# Create a Global Accelerator
aws globalaccelerator create-accelerator \
  --name my-global-app \
  --ip-address-type IPV4 \
  --enabled

# Create a listener on the accelerator
aws globalaccelerator create-listener \
  --accelerator-arn arn:aws:globalaccelerator::123456789012:accelerator/abc123 \
  --protocol TCP \
  --port-ranges '[{"FromPort": 80, "ToPort": 80}, {"FromPort": 443, "ToPort": 443}]'

# Create an endpoint group (points to ALB in a region)
aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::123456789012:accelerator/abc123/listener/def456 \
  --endpoint-group-region us-east-1 \
  --traffic-dial-percentage 100 \
  --endpoint-configurations '[{
    "EndpointId": "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/abc123",
    "Weight": 100,
    "ClientIPPreservationEnabled": true
  }]'

# List accelerators
aws globalaccelerator list-accelerators

# Describe accelerator (get Anycast IPs)
aws globalaccelerator describe-accelerator \
  --accelerator-arn arn:aws:globalaccelerator::123456789012:accelerator/abc123
```

---

