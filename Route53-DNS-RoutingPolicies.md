# Amazon Route 53 — DNS & Routing Policies
> 📚 Official Docs: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/
> 🎯 SAA-C03 Exam Weight: High — routing policies appear in many architecture scenarios

---

## 🌐 DNS Fundamentals — How the Internet Finds Things

Before Route 53, let's understand DNS itself. When you type `www.google.com` in your browser:

```
DNS Resolution Flow:
┌──────────────────────────────────────────────────────────────┐
│  1. Browser checks local cache → not found                   │
│  2. OS asks your ISP's DNS resolver (Recursive DNS)          │
│  3. Recursive DNS asks Root DNS → "Who handles .com?"        │
│  4. Root DNS → "Ask the .com TLD servers"                    │
│  5. TLD (.com) → "Ask google.com's name servers"             │
│  6. google.com's name server → "216.58.210.46"               │
│  7. Browser connects to 216.58.210.46                        │
│                                                              │
│  This whole thing takes ~100ms but is cached (TTL)           │
└──────────────────────────────────────────────────────────────┘
```

### Key DNS Record Types

| Record | Maps | Example |
|--------|------|---------|
| **A** | Hostname → IPv4 | `api.example.com` → `1.2.3.4` |
| **AAAA** | Hostname → IPv6 | `api.example.com` → `2001:db8::1` |
| **CNAME** | Hostname → Hostname | `www.example.com` → `example.com` |
| **MX** | Hostname → Mail server | email routing |
| **NS** | Zone → Name server | Delegates DNS for a domain |
| **TXT** | Text records | Domain verification, SPF |

---

## 📍 What is Route 53?

Route 53 is AWS's managed DNS service. The name comes from **port 53** — the standard DNS port.

**Key capabilities:**
1. **DNS service** — Resolve domain names to IPs/AWS resources
2. **Domain registrar** — Buy and manage domain names (like GoDaddy but inside AWS)
3. **Health checking** — Monitor endpoint health, auto-failover
4. **Traffic routing** — 8 different routing policies (more on this below)

> 🏆 **Route 53 is the ONLY AWS service with a 100% uptime SLA** — AWS guarantees it will never be down.

---

## 🔍 Hosted Zones — Containers for DNS Records

A **Hosted Zone** is a container for DNS records for a domain. Two types:

```
Public Hosted Zone:                  Private Hosted Zone:
example.com → 1.2.3.4               app.internal → 10.0.1.50
Accessible from the INTERNET         Accessible only within your VPC(s)
For public websites/APIs             For internal microservices
```

**Cost:** $0.50/month per hosted zone

### Creating Hosted Zones

```bash
# Create a public hosted zone
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "$(date +%s)" \
  --hosted-zone-config Comment="My public zone"

# Create a private hosted zone (for VPC internal DNS)
aws route53 create-hosted-zone \
  --name internal.example.com \
  --caller-reference "$(date +%s)" \
  --hosted-zone-config PrivateZone=true,Comment="Internal zone" \
  --vpc VPCRegion=us-east-1,VPCId=vpc-1234567890abcdef0

# List all hosted zones
aws route53 list-hosted-zones

# Get hosted zone details
aws route53 get-hosted-zone --id /hostedzone/Z1234567890ABC

# Delete hosted zone (must delete all records first)
aws route53 delete-hosted-zone --id /hostedzone/Z1234567890ABC
```

---

## ⭐ CNAME vs Alias — A Critical Difference

This is one of the most tested Route 53 concepts:

```
CNAME:
- Maps hostname to ANOTHER hostname
- example.com (root/apex) → CANNOT use CNAME ❌
- www.example.com → CAN use CNAME ✅
- Costs per query ($0.40/million)
- Can point to any hostname (even non-AWS)

Alias:
- Maps hostname to AWS RESOURCE directly
- example.com (root/apex) → CAN use Alias ✅ (this is the key advantage!)
- www.example.com → CAN use Alias ✅  
- FREE of charge!
- Native health check support
- TTL is set by AWS (you can't change it)
- Can point to: ALB, NLB, CloudFront, S3 website, API Gateway, VPC endpoints...
- CANNOT point to EC2 DNS names
```

> 💡 **The rule**: Use **CNAME** for subdomain-to-hostname mapping. Use **Alias** for domain apex (naked domain like `example.com`) pointing to AWS resources — CNAME doesn't work here.

### Creating A Records and CNAME Records

```bash
# Create a simple A record (maps hostname to IP)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "54.123.45.67"}]
      }
    }]
  }'

# Create a CNAME record (subdomain → another hostname)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "blog.example.com",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "example.com"}]
      }
    }]
  }'

# Create an ALIAS record (for ALB — no TTL, free, zone apex compatible!)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "my-alb-1234567890.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# Create ALIAS at zone apex (naked domain)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "my-alb-1234567890.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

---

## 🗺️ Route 53 Routing Policies — 8 Types

### 1. 📝 Simple Routing
One record, one or multiple values. No health checks.

```
DNS Query: api.example.com
Route 53 returns: [1.2.3.4, 5.6.7.8, 9.10.11.12]
Client randomly picks one (client-side load balancing)
```
Use for: Single resources, no HA needed.

```bash
# Simple routing record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [
          {"Value": "1.2.3.4"},
          {"Value": "5.6.7.8"}
        ]
      }
    }]
  }'
```

---

### 2. ⚖️ Weighted Routing
Split traffic by percentage across multiple resources.

```
example.com
├── 70% → EC2 in us-east-1 (main prod)
├── 20% → EC2 in eu-west-1 (secondary)
└── 10% → EC2 new version (A/B testing)

Weight 0 = no traffic sent to that resource (useful for maintenance!)
```
Use for: A/B testing, blue/green deployments, gradual traffic shifting.

```bash
# Weighted routing: 70% to primary, 30% to secondary
# Record 1: 70% weight
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Primary",
        "Weight": 70,
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.123.45.67"}]
      }
    }]
  }'

# Record 2: 30% weight
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Secondary",
        "Weight": 30,
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.234.56.78"}]
      }
    }]
  }'

# Gradual traffic shift: 0% → 10% → 50% → 100% by updating weights
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "NewVersion",
        "Weight": 10,
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.111.22.33"}]
      }
    }]
  }'
```

---

### 3. ⚡ Latency-Based Routing
Routes user to the AWS region with lowest latency FROM that user's location.

```
User in London:
Route 53 checks: latency to us-east-1 = 90ms | eu-west-1 = 10ms
→ Routes to eu-west-1 ✅

User in New York:
Route 53 checks: latency to us-east-1 = 5ms | eu-west-1 = 80ms
→ Routes to us-east-1 ✅
```

> ⚠️ **Latency ≠ Geography**: A user in Brazil might have lower latency to us-east-1 than sa-east-1 depending on routing.

```bash
# Latency routing: us-east-1
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "us-east-1",
        "Region": "us-east-1",
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.123.45.67"}]
      }
    }]
  }'

# Latency routing: eu-west-1
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "eu-west-1",
        "Region": "eu-west-1",
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.234.56.78"}]
      }
    }]
  }'
```

---

### 4. 🔄 Failover Routing (Active-Passive)
Primary record + secondary record. If primary fails health check → Route 53 automatically returns the secondary.

```
Primary: api-primary.example.com (EC2 in us-east-1)
         ↓ Health check every 30 seconds
         FAILS health check!
         ↓
Secondary: api-backup.example.com (EC2 in eu-west-1)
         ← Route 53 now returns THIS record
```

> 💡 This is **Active-Passive** HA. Not to be confused with Active-Active where all resources receive traffic simultaneously (use Weighted or Latency for that).

```bash
# Failover routing: primary record (with health check)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Primary",
        "Failover": "PRIMARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.123.45.67"}],
        "HealthCheckId": "healthcheck-id-here"
      }
    }]
  }'

# Failover routing: secondary record (backup)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Secondary",
        "Failover": "SECONDARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.234.56.78"}]
      }
    }]
  }'
```

---

### 5. 🌍 Geolocation Routing
Routes based on **where the user is physically located** (continent, country, US state).

```
User in Germany → Route 53 returns German server (GDPR compliance!)
User in Japan → Route 53 returns Tokyo server
User from unknown/unmatched location → Default record (always set one!)
```

**Geolocation vs Latency:**
- Geolocation = "WHERE are you from?" (geography/compliance)
- Latency = "WHERE is fastest for you?" (performance)

Use for: Content localization, legal restrictions, GDPR compliance.

```bash
# Geolocation routing: Europe (EU users)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Europe",
        "GeoLocation": {
          "ContinentCode": "EU"
        },
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.123.45.67"}]
      }
    }]
  }'

# Geolocation routing: Japan
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Japan",
        "GeoLocation": {
          "CountryCode": "JP"
        },
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.234.56.78"}]
      }
    }]
  }'

# Geolocation routing: Default (catch-all for unknown/unmatched)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Default",
        "GeoLocation": {
          "CountryCode": "*"
        },
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.111.22.33"}]
      }
    }]
  }'
```

---

### 6. 🎯 Geoproximity Routing (Traffic Flow only)
Routes based on location of resources AND users, with a **bias** to attract or repel traffic.

```
Bias +50 on us-east-1 → Attracts users from wider geographic area
Bias -50 on eu-west-1 → Pushes users away to other regions
```

Only available in Route 53 **Traffic Flow** (visual editor). Use for: gradually shifting traffic between regions.

---

### 7. 🌐 IP-Based Routing
Route traffic based on the **client's IP address** (CIDR block).

```
CIDR 192.0.2.0/24 → Route to Server A (known office IP → low-latency server)
CIDR 203.0.113.0/24 → Route to Server B (known ISP IP → their nearest server)
```

Use for: Optimizing performance based on known ISP/corporate IP ranges.

```bash
# IP-based routing: Route office IPs to low-latency server
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "OfficeNetwork",
        "GeoProximityLocation": {
          "Latitude": "40.7128",
          "Longitude": "-74.0060"
        },
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.123.45.67"}]
      }
    }]
  }'
```

---

### 8. 📊 Multi-Value Routing
Return multiple healthy records (up to 8). Different from Simple — it has **health checks**.

```
Route 53 returns up to 8 records, all healthy:
[1.2.3.4 ✅, 5.6.7.8 ✅, 9.10.11.12 ❌ (unhealthy — excluded)]
→ Returns: [1.2.3.4, 5.6.7.8]
Client picks one.
```

> ⚠️ **Multi-Value is NOT a load balancer** — it's client-side selection from healthy IPs. For proper load balancing, use ELB.

```bash
# Multi-value routing: return up to 8 healthy IPs
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.example.com",
          "Type": "A",
          "SetIdentifier": "Instance1",
          "MultiValueAnswer": true,
          "TTL": 60,
          "ResourceRecords": [{"Value": "1.2.3.4"}],
          "HealthCheckId": "healthcheck-1"
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.example.com",
          "Type": "A",
          "SetIdentifier": "Instance2",
          "MultiValueAnswer": true,
          "TTL": 60,
          "ResourceRecords": [{"Value": "5.6.7.8"}],
          "HealthCheckId": "healthcheck-2"
        }
      }
    ]
  }'
```

---

## ❤️ Health Checks in Route 53

Health checks allow Route 53 to automatically stop routing to unhealthy endpoints.

```
Types of Health Checks:
1. Endpoint health check → HTTP/HTTPS/TCP to your server, checks for 2xx/3xx response
2. Calculated health check → Combines multiple child health checks (AND/OR logic)
3. CloudWatch Alarm health check → Monitors a CloudWatch metric (e.g., SQS queue depth)

For PRIVATE resources (in VPC):
→ Can't health check directly (Route 53 checkers are on internet)
→ Solution: Create CloudWatch Metric + Alarm → health check monitors the Alarm
```

Route 53 has 15+ health checkers globally. Default interval: 30 seconds.

### Creating Health Checks

```bash
# Create HTTP health check
aws route53 create-health-check \
  --caller-reference "$(date +%s)" \
  --health-check-config '{
    "Type": "HTTP",
    "IPAddress": "54.123.45.67",
    "Port": 80,
    "ResourcePath": "/health",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'

# Create HTTPS health check
aws route53 create-health-check \
  --caller-reference "$(date +%s)" \
  --health-check-config '{
    "Type": "HTTPS",
    "FullyQualifiedDomainName": "api.example.com",
    "Port": 443,
    "ResourcePath": "/api/health",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'

# Create TCP health check (for databases)
aws route53 create-health-check \
  --caller-reference "$(date +%s)" \
  --health-check-config '{
    "Type": "TCP",
    "IPAddress": "10.0.0.100",
    "Port": 3306,
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'

# Create CloudWatch alarm health check (for private resources)
aws route53 create-health-check \
  --caller-reference "$(date +%s)" \
  --health-check-config '{
    "Type": "CLOUDWATCH_METRIC",
    "AlarmIdentifier": {
      "Region": "us-east-1",
      "Name": "my-app-health-alarm"
    },
    "InsufficientDataHealthStatus": "Healthy"
  }'

# List health checks
aws route53 list-health-checks

# Describe specific health check
aws route53 get-health-check --health-check-id healthcheck-id-here

# Get health check status
aws route53 get-health-check-status --health-check-id healthcheck-id-here
```

### Listing and Testing Records

```bash
# List all record sets in a hosted zone
aws route53 list-resource-record-sets \
  --hosted-zone-id Z1234567890ABC

# Test DNS resolution (what Route 53 would return)
aws route53 test-dns-answer \
  --hosted-zone-id Z1234567890ABC \
  --record-name www.example.com \
  --record-type A

# Verify record change status
aws route53 list-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --filters Name=name,Values=api.example.com
```

### Testing DNS from Terminal

```bash
# Use dig (best for DNS queries)
dig www.example.com
dig www.example.com @8.8.8.8  # Query Google's DNS specifically

# Use nslookup
nslookup www.example.com
nslookup www.example.com 8.8.8.8  # Query Google's DNS

# Check specific record type
dig www.example.com A
dig api.example.com CNAME
dig mail.example.com MX
```

---

## ⭐ Interview Tips & Key Points to Remember

- **Route 53 = 100% SLA** — only AWS service with this guarantee
- **CNAME cannot be Zone Apex** (naked domain like `example.com`) — use Alias instead
- **Alias is free; CNAME costs per query** — always prefer Alias for AWS resources
- **Alias cannot point to EC2 DNS names** — only ELB, CloudFront, S3, API GW, etc.
- **Failover = Active-Passive**; health check required on primary
- **Geolocation = WHERE you ARE**; Latency = WHERE is FASTEST for you
- **Multi-Value ≠ ELB** — it's just returning multiple healthy IPs to the client
- **Geoproximity requires Traffic Flow** — visual policy editor
- **Default record in Geolocation** is critical — without it, unmatched users get NXDOMAIN
- **Weight 0 = no traffic** — use this for maintenance windows without deleting the record
- **Private hosted zones** require VPC association and `enableDnsHostnames` + `enableDnsSupport` on VPC
- Scenario "GDPR — EU users must use EU servers" → **Geolocation routing**
- Scenario "gradual traffic shift to new version" → **Weighted routing (0 → 10 → 50 → 100)**
- Scenario "failover to backup site automatically" → **Failover routing + health check**

---

## Quick Reference — AWS CLI Commands

### Hosted Zones

```bash
# Create public hosted zone
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "$(date +%s)" \
  --hosted-zone-config Comment="My public zone"

# Create private hosted zone
aws route53 create-hosted-zone \
  --name internal.example.com \
  --caller-reference "$(date +%s)" \
  --hosted-zone-config PrivateZone=true \
  --vpc VPCRegion=us-east-1,VPCId=vpc-1234567890abcdef0

# List hosted zones
aws route53 list-hosted-zones

# Get hosted zone details
aws route53 get-hosted-zone --id /hostedzone/Z1234567890ABC

# Delete hosted zone
aws route53 delete-hosted-zone --id /hostedzone/Z1234567890ABC
```

### Simple Records

```bash
# Create A record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "54.123.45.67"}]
      }
    }]
  }'

# Create ALIAS record (for ALB)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "my-alb-1234567890.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Weighted Routing

```bash
# Record with 70% weight
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Primary",
        "Weight": 70,
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.123.45.67"}]
      }
    }]
  }'

# Record with 30% weight
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Secondary",
        "Weight": 30,
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.234.56.78"}]
      }
    }]
  }'
```

### Latency Routing

```bash
# Record for us-east-1
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "us-east-1",
        "Region": "us-east-1",
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.123.45.67"}]
      }
    }]
  }'

# Record for eu-west-1
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "eu-west-1",
        "Region": "eu-west-1",
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.234.56.78"}]
      }
    }]
  }'
```

### Failover Routing

```bash
# Primary record (with health check)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Primary",
        "Failover": "PRIMARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.123.45.67"}],
        "HealthCheckId": "healthcheck-id-here"
      }
    }]
  }'

# Secondary record (backup)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Secondary",
        "Failover": "SECONDARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.234.56.78"}]
      }
    }]
  }'
```

### Geolocation Routing

```bash
# Europe
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Europe",
        "GeoLocation": {"ContinentCode": "EU"},
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.123.45.67"}]
      }
    }]
  }'

# Default (catch-all)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "Default",
        "GeoLocation": {"CountryCode": "*"},
        "TTL": 60,
        "ResourceRecords": [{"Value": "54.111.22.33"}]
      }
    }]
  }'
```

### Health Checks

```bash
# Create HTTP health check
aws route53 create-health-check \
  --caller-reference "$(date +%s)" \
  --health-check-config '{
    "Type": "HTTP",
    "IPAddress": "54.123.45.67",
    "Port": 80,
    "ResourcePath": "/health",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'

# List health checks
aws route53 list-health-checks

# Test DNS answer (what Route 53 will return)
aws route53 test-dns-answer \
  --hosted-zone-id Z1234567890ABC \
  --record-name www.example.com \
  --record-type A

# List all records in zone
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890ABC
```

