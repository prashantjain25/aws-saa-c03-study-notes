# AWS – Getting Started & Global Infrastructure
> 📚 Official Docs: https://aws.amazon.com/about-aws/global-infrastructure/  
> 🎯 SAA-C03 Exam Weight: Foundational (appears in every domain)

---

## 🌐 What is Cloud Computing & Why AWS?

Before AWS existed, companies had to buy physical servers, set up data centers, hire staff to maintain them, and predict how much hardware they'd need 3–5 years in advance. If you guessed wrong, you either had too much (wasted money) or too little (your site crashes on Black Friday).

**AWS solves this** by letting you rent computing resources on-demand — you pay only for what you use, and you can scale up or down in minutes.

### The 6 Advantages of Cloud (AWS Whitepaper)
These are directly from AWS and frequently appear in exams:

1. **Trade capital expense for variable expense** — No upfront hardware purchase; pay as you go
2. **Benefit from massive economies of scale** — AWS buys hardware at huge volumes, passing savings to you
3. **Stop guessing capacity** — Scale up/down based on actual demand
4. **Increase speed and agility** — Go from idea to deployment in minutes, not months
5. **Stop spending money on data center operations** — Focus on your product, not the rack
6. **Go global in minutes** — Deploy in multiple regions with a few clicks

> 🔗 Reference: https://docs.aws.amazon.com/whitepapers/latest/aws-overview/six-advantages-of-cloud-computing.html

---

## 🏗️ AWS Global Infrastructure — The Big Picture

Think of AWS infrastructure as a **tree of nested layers**:

```
🌍 AWS Global Network
 └── Regions (geographic areas)
      └── Availability Zones (isolated data centers)
           └── Data Centers (physical buildings)
 └── Edge Locations / Points of Presence (CDN cache nodes)
```

> 🔗 Explore the live map: https://infrastructure.aws/

---

## 📍 AWS Regions

A **Region** is a **cluster of data centers** in a specific geographic area, identified by a code like `us-east-1` (N. Virginia) or `eu-west-3` (Paris).

**Key facts:**
- AWS has **30+ regions** worldwide (growing)
- Most AWS services are **region-scoped** (you create resources in a specific region)
- Data in one region **never leaves that region** without your explicit permission (crucial for compliance)

### How to Choose a Region? (The CAPS Framework)

This is a classic interview question. Remember **CAPS**:

| Factor | Why It Matters |
|--------|---------------|
| **C**ompliance | Legal requirements — e.g., GDPR requires EU data to stay in EU |
| **A**vailable Services | Not all regions have all AWS services (new services launch in us-east-1 first) |
| **P**roximity | Closer region = lower latency for your users |
| **S**aving (Pricing) | Same service can cost 20–30% more in some regions |

> 💡 **Example**: A fintech company in Germany serving German customers should use `eu-central-1` (Frankfurt) — for both GDPR compliance and proximity.

---

## 🏢 Availability Zones (AZs)

Each region has **multiple AZs** (usually 3, minimum 3, maximum 6).

**What is an AZ?** It is one or more **discrete physical data centers** with:
- Independent power supply
- Independent networking
- Independent cooling
- **Physically separated** from other AZs (often miles apart to avoid simultaneous disasters like floods or fires)
- **Connected to each other** via private, high-bandwidth, ultra-low-latency fiber links

### Visual — Region with 3 AZs:

```
┌─────────────────────── AWS Region: ap-southeast-2 (Sydney) ───────────────────────┐
│                                                                                   │
│   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐             │
│   │  ap-southeast-2a │    │  ap-southeast-2b│     │  ap-southeast-2c│             │
│   │                 │     │                 │     │                 │             │
│   │  [Data Center]  │────▶│  [Data Center]  │────▶│  [Data Center]  │             │
│   │  [Data Center]  │◀────│  [Data Center]  │◀────│  [Data Center]  │             │
│   └─────────────────┘     └─────────────────┘     └─────────────────┘             │
│          │                        │                        │                      │
│          └────────── High-Bandwidth, Low-Latency Network ──┘                      │
└───────────────────────────────────────────────────────────────────────────────────┘
```

> 💡 **Why design apps across multiple AZs?** If one AZ has a power outage or fire, your app keeps running in the other AZs. This is the foundation of **High Availability** on AWS.

---

## 🌐 Edge Locations (Points of Presence)

Edge Locations are **not full AWS regions** — they are **smaller caching nodes** placed in cities worldwide to bring content closer to end users.

**Numbers to know:**
- **400+ Edge Locations** (and growing)
- **10+ Regional Edge Caches** (larger intermediate caches)
- Spread across **90+ cities in 40+ countries**

**Used by:** Amazon CloudFront (CDN), Route 53 (DNS), AWS Shield, Lambda@Edge

```
User in Mumbai
     │
     ▼
[Edge Location Mumbai]  ◄── Cache hit? Serve instantly ✅
     │
     │ Cache miss? Fetch from...
     ▼
[Regional Edge Cache]
     │
     ▼
[S3 Bucket in us-east-1]  ◄── Origin (actual data)
```

> 💡 **Analogy**: Think of Edge Locations as local branches of a bank. Instead of everyone traveling to HQ (origin), they go to the nearest branch (Edge Location) for fast service.

---

## 🗂️ Global vs Regional Services — Critical for Exam

This catches many people out. Some AWS services are **global** (they exist once across all regions), while most are **regional** (you deploy them per-region).

### ✅ Global Services (NOT tied to a region):
| Service | What It Does |
|---------|-------------|
| **IAM** | Identity & Access Management |
| **Route 53** | DNS service |
| **CloudFront** | CDN (uses Edge Locations) |
| **WAF** | Web Application Firewall |

### 🗺️ Regional Services (examples):
| Service | Model |
|---------|-------|
| Amazon EC2 | IaaS (Infrastructure as a Service) |
| Elastic Beanstalk | PaaS (Platform as a Service) |
| AWS Lambda | FaaS (Function as a Service) |
| Amazon Rekognition | SaaS (Software as a Service) |

> 🔗 Check which services are available per region: https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/

---

## ☁️ Cloud Computing Models — Know the 3 Types

These appear in exams under "What model does EC2 represent?":

| Model | Who Manages What | AWS Example | You Manage |
|-------|-----------------|-------------|------------|
| **IaaS** | You manage OS and up | EC2 | OS, runtime, app, data |
| **PaaS** | Provider manages OS & runtime | Elastic Beanstalk | App code & data only |
| **SaaS** | Provider manages everything | Gmail, Dropbox (or Rekognition) | Just use it |

---

## ⭐ Interview Tips & Key Points to Remember

> These are the facts most frequently tested in AWS interviews and the SAA-C03 exam.

- **IAM, Route 53, CloudFront, WAF are GLOBAL** — no region selection needed
- **Regions have min 3 AZs, max 6** — know this number!
- **Data never leaves a region** without explicit permission — critical for compliance questions
- **Edge Locations far outnumber Regions** (~400+ vs ~30+) — used for CDN caching
- **AZs are physically separate** (disaster isolation) but connected (low latency)
- **CAPS framework** for region selection: Compliance, Availability, Proximity, Savings
- **IaaS vs PaaS vs FaaS**: EC2=IaaS, Beanstalk=PaaS, Lambda=FaaS — common interview distinction
- **AWS launched SQS in 2004** (first public service); re-launched with SQS+S3+EC2 in 2006
- When exam says "low latency global users" → think Edge Locations / CloudFront
- When exam says "compliance/data residency" → think Region selection + data sovereignty
