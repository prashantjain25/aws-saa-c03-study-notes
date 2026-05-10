# AWS SAP-C02 — Section 4: Security

**Solutions Architect Professional — Expert-Style Study Notes**  
*Understand the WHY, not just the WHAT. Depth over breadth.*

---

## Table of Contents

1. [CloudTrail — Multi-Region, Organization Trails, Event Types](#1-cloudtrail)
2. [CloudTrail — EventBridge Integration](#2-cloudtrail--eventbridge-integration)
3. [CloudTrail — SA Pro Patterns](#3-cloudtrail--sa-pro-patterns)
4. [KMS — Key Types, Envelope Encryption, Policies, Grants, Multi-Region](#4-kms)
5. [Parameter Store — Tiers, SecureString, Hierarchy, Rotation](#5-parameter-store)
6. [Secrets Manager — Rotation, Lambda, Cross-Account, vs Parameter Store](#6-secrets-manager)
7. [RDS Security — Encryption, IAM Auth, SSL, TDE](#7-rds-security)
8. [SSL Encryption, SNI & MITM](#8-ssl-encryption-sni--mitm)
9. [AWS Certificate Manager — ACM](#9-aws-certificate-manager--acm)
10. [CloudHSM — Dedicated Hardware, FIPS Level 3, Cluster Architecture](#10-cloudhsm)
11. [Solution Architecture — SSL on ELB](#11-solution-architecture--ssl-on-elb)
12. [S3 Security — Bucket Policies, ACLs, MFA Delete, Object Lock](#12-s3-security)
13. [S3 Access Points](#13-s3-access-points)
14. [S3 Multi-Region Access Points](#14-s3-multi-region-access-points)
15. [S3 Object Lambda](#15-s3-object-lambda)
16. [DDoS and AWS Shield](#16-ddos-and-aws-shield)
17. [AWS WAF — Web Application Firewall](#17-aws-waf)
18. [AWS Firewall Manager](#18-aws-firewall-manager)
19. [Blocking an IP Address](#19-blocking-an-ip-address)
20. [Amazon Inspector](#20-amazon-inspector)
21. [AWS Config — Rules, Conformance Packs, Remediation, Aggregation](#21-aws-config)
22. [AWS Managed Logs — Which Services Log Where](#22-aws-managed-logs)
23. [Amazon GuardDuty](#23-amazon-guardduty)
24. [IAM Advanced Policies — Condition Keys, NotAction, PassRole, Permission Boundaries](#24-iam-advanced-policies)
25. [EC2 Instance Connect](#25-ec2-instance-connect)
26. [AWS Security Hub](#26-aws-security-hub)
27. [Amazon Detective](#27-amazon-detective)
28. [Decision Framework — Which Security Service for Which Threat](#28-decision-framework)

---

## 1. CloudTrail

### Why CloudTrail Exists

Think of CloudTrail as the "security camera system" for your AWS account. Every API call — whether through the Console, CLI, SDK, or another AWS service — is recorded. When something goes wrong (a resource deleted, an IAM user created overnight), CloudTrail is how you answer: *who did what, from where, and when*.

This is the foundation of incident response and compliance in AWS.

### Event Types

| Event Type | Description | Default? | Cost |
|---|---|---|---|
| **Management Events** | Create, modify, delete resources (IAM, EC2, RDS, etc.) | Yes | Included |
| **Data Events** | S3 GetObject/PutObject, Lambda Invoke, DynamoDB operations | No | Extra cost |
| **Insights Events** | Unusual API activity detection (anomalous rate/pattern) | No | Extra cost |

**Why this matters at the Pro level**: In a security incident, you need to decide how much to log. Management events are sufficient for most auditing. Data events are required when you need to know "who read which S3 object" — critical for data exfiltration investigations. Insights events add ML-based anomaly detection on top.

### Multi-Region Trails

A **single-region trail** only captures events in one region. This is a security gap — an attacker operating in `eu-west-1` won't appear in a `us-east-1`-only trail.

A **multi-region trail** captures events from all current and future regions from a single trail configuration.

🎯 **EXAM TIP**: For compliance requirements (SOC 2, PCI DSS, HIPAA), you **must** use a multi-region trail. Single-region trails are considered insufficient for enterprise auditing. The exam often asks about "ensuring all regions are covered" — answer is multi-region trail.

```
Characteristics of multi-region trail:
- All regions → single S3 bucket
- Cannot be deleted without AWS Support if created by AWS Control Tower
- New regions added in future are automatically covered
- IsMultiRegionTrail: true
```

### Organization Trails

For organizations with multiple AWS accounts (common at SAP level), **organization trails** aggregate CloudTrail events from ALL member accounts into a single S3 bucket in the management account.

**Why this matters**: Without organization trails, each account manages its own CloudTrail independently, making cross-account forensics very difficult. With organization trails, the security team has a single place to search.

```
Architecture:
Management Account (delegated admin) → Creates trail
↓ Applies to all member accounts automatically
All API calls across organization → Central S3 bucket (management account)
```

🎯 **EXAM TIP**: Organization trails are managed from the management account (or delegated admin account). Member accounts cannot modify or delete the organization trail — this prevents tampering by account owners.

### Log File Integrity Validation

CloudTrail can generate a **digest file** every hour that contains cryptographic hashes of all log files delivered in that hour. This allows you to verify that log files have not been tampered with after delivery.

**Why this exists**: An attacker who compromises an S3 bucket might delete or modify CloudTrail logs to cover their tracks. Integrity validation detects this.

```bash
# Enable with integrity validation
aws cloudtrail create-trail \
  --name "SecureTrail" \
  --s3-bucket-name "my-cloudtrail-bucket" \
  --is-multi-region-trail \
  --enable-log-file-validation
```

🎯 **EXAM TIP**: "Ensure CloudTrail logs have not been tampered with" → enable log file integrity validation. Digest files are stored separately from log files in S3.

### Event Retention

- **CloudTrail Event History (Console)**: Last 90 days, free, searchable
- **S3**: Indefinite, you pay for storage, query with Athena
- **CloudWatch Logs**: Per retention policy you set

**Pro pattern**: Store in S3 (cheap, long retention) + send to CloudWatch Logs (real-time alerting on specific events).

---

## 2. CloudTrail — EventBridge Integration

### The Power of Triggering Automation from API Calls

On its own, CloudTrail just records events. The real power comes when you connect CloudTrail to EventBridge to trigger automated responses to API events in real time.

**Architecture**:
```
API Call happens → CloudTrail records it → EventBridge receives it
                                              ↓
                                         Rule matches
                                              ↓
                              Lambda / SNS / SQS / Step Functions
```

### Common Automation Patterns

**Pattern 1: Detect root account usage**
```
Condition: eventSource = "signin.amazonaws.com", userIdentity.type = "Root"
Action: SNS → Pager Duty alert
```
Root account should never be used for day-to-day operations. Any root login is a high-priority alert.

**Pattern 2: Detect unauthorized API calls**
```
Condition: errorCode = "AccessDenied" or "UnauthorizedAccess"
Action: SNS alert → Security team
```
A burst of AccessDenied errors often indicates credential probing or a misconfigured application.

**Pattern 3: Detect Security Group changes**
```
Condition: eventSource = "ec2.amazonaws.com", eventName = "AuthorizeSecurityGroupIngress"
Action: Lambda → verify change matches approved list, otherwise revert
```

**Pattern 4: Detect IAM changes**
```
Condition: eventSource = "iam.amazonaws.com", eventName = "CreateUser" OR "AttachUserPolicy"
Action: Lambda → notify SOC team, validate against provisioning system
```

🎯 **EXAM TIP**: The question "How do you automatically detect and respond when someone disables CloudTrail logging?" is a classic SAP scenario. Answer: CloudTrail delivers an event to EventBridge when `StopLogging` is called → Lambda re-enables logging and alerts security team.

### EventBridge Rule for CloudTrail Events

```json
{
  "source": ["aws.cloudtrail"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["iam.amazonaws.com"],
    "eventName": ["CreateUser", "DeleteUser", "AttachUserPolicy", "PutUserPolicy"]
  }
}
```

---

## 3. CloudTrail — SA Pro Patterns

### Cross-Account Logging to Central S3

**Problem**: In a multi-account organization, each account has its own CloudTrail. A security breach in one account leaves logs scattered across accounts. An attacker who compromises Account A and deletes its CloudTrail is invisible to the security team.

**Solution**: Central S3 bucket in a dedicated security/logging account. All accounts ship logs there.

```
Account A (production) ──────────────────┐
Account B (staging) ─────────────────────┤→ Security Account S3 bucket
Account C (development) ─────────────────┘
     ↑
  (member accounts have NO delete permission on central bucket)
```

**Implementation**:
1. Create dedicated S3 bucket in security account
2. Bucket policy grants `s3:PutObject` to all member account CloudTrail principals
3. Bucket policy denies `s3:DeleteObject` and `s3:DeleteBucket` to everyone except break-glass admin
4. Enable S3 Object Lock (WORM) on the bucket for tamper-proofing

**S3 Bucket Policy for Cross-Account CloudTrail (critical pattern)**:
```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "cloudtrail.amazonaws.com"
  },
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::central-logs-bucket/AWSLogs/*",
  "Condition": {
    "StringEquals": {
      "s3:x-amz-acl": "bucket-owner-full-control"
    }
  }
}
```

🎯 **EXAM TIP**: The bucket policy must allow CloudTrail's service principal (`cloudtrail.amazonaws.com`) to put objects. The `bucket-owner-full-control` ACL condition ensures the security account owns the objects even though CloudTrail from another account wrote them.

### Real-Time Threat Detection Architecture

```
CloudTrail → CloudWatch Logs Log Group
                    ↓
             Metric Filter (e.g., root logins, IAM changes)
                    ↓
             CloudWatch Alarm
                    ↓
             SNS → Slack / PagerDuty / Lambda response
```

**Why two paths (CloudWatch Logs AND S3)?**
- CloudWatch Logs: real-time alerting, Logs Insights queries
- S3: long-term storage, Athena analysis, compliance archive

### Athena for CloudTrail Analysis

For historical forensics, Athena is far more powerful than the CloudTrail console's 90-day limit.

```sql
-- Find all API calls from a suspicious IP in the last 30 days
SELECT eventTime, userIdentity, eventName, sourceIPAddress
FROM cloudtrail_logs
WHERE sourceIPAddress = '198.51.100.47'
  AND from_iso8601_timestamp(eventTime) > current_timestamp - interval '30' day
ORDER BY eventTime DESC;
```

🎯 **EXAM TIP**: "How do you investigate all API calls made over the past year?" → CloudTrail logs in S3 + Athena query. The 90-day console window is NOT sufficient for long-term forensics.

---

## 4. KMS

### Why KMS Exists

KMS is a delegated trust model: AWS manages the physical key infrastructure (HSMs, geo-redundant storage, access controls), while you control who can use the keys and for what purpose. You get cryptographic security without needing to run your own HSM infrastructure.

**Analogy**: KMS is like a bank vault for cryptographic keys. The bank (AWS) secures the vault; you control who has access to your specific safe-deposit box (your key).

### Key Types — The Three-Tier Model

| Key Type | Managed By | Cost | Rotation | Visibility | When to Use |
|---|---|---|---|---|---|
| **AWS Owned Keys** | AWS entirely | Free | AWS handles | Not visible in console | Default S3, SQS, DynamoDB |
| **AWS Managed Keys** | AWS (in your account) | Free | Auto every 365d | Can view, can't modify | When you need audit trail but not custom policy |
| **Customer Managed Keys (CMK)** | You | $1/month | You control | Full visibility + control | Cross-account, custom rotation, custom policy |

**Key policy rule (critical)**: Even if IAM allows a principal to use KMS, the **key policy must also explicitly allow it**. Both must say YES. This is different from most AWS resources where IAM alone is sufficient.

```
IAM Policy: "Role X can call kms:Encrypt"  +  Key Policy: silent on Role X
= DENIED (key policy blocks it)

IAM Policy: "Role X can call kms:Encrypt"  +  Key Policy: allows Role X
= ALLOWED (both must permit)
```

🎯 **EXAM TIP**: "An EC2 role has kms:Decrypt in its IAM policy but cannot decrypt data" — the key policy is missing the grant. Both must allow. This is the most common KMS troubleshooting scenario on the exam.

### Custom Key Store (CloudHSM-backed)

For compliance requirements demanding customer-controlled hardware (not shared AWS HSMs), you can create a KMS key backed by your own CloudHSM cluster. You get KMS API compatibility + CloudHSM's FIPS 140-2 Level 3 security.

**Use case**: Financial services regulators requiring "you must control the HSM hardware."

### Envelope Encryption — The Key Concept

**The problem**: KMS can only encrypt/decrypt up to 4 KB directly. What about 1 TB files?

**Envelope Encryption** solves this:

```
Step 1: Call GenerateDataKey
        KMS returns:
        ├── Plaintext Data Key (256-bit, use to encrypt your data locally)
        └── Encrypted Data Key (encrypted with your KMS CMK, store this)

Step 2: Encrypt your 1 TB file locally using the plaintext data key (AES-256-GCM)

Step 3: Discard the plaintext data key
        Store: Encrypted file + Encrypted Data Key (together)

To decrypt:
Step 4: Call Decrypt on the Encrypted Data Key → KMS returns plaintext data key
Step 5: Decrypt your file locally with the plaintext data key
Step 6: Discard the plaintext data key again
```

**Why this is better than encrypting directly with KMS**:
- Large files stay local (no 4 KB limit)
- Only a small key goes to KMS (fast, cheap API calls)
- Encryption/decryption of bulk data is local (fast AES hardware acceleration)
- KMS never sees your actual data

🎯 **EXAM TIP**: "Encrypt a 5 GB file using KMS" → envelope encryption with GenerateDataKey. "KMS throttles on S3 uploads" → enable S3 Bucket Key (caches the data key at S3 level, reduces KMS calls by ~99%).

### KMS Grants

Grants provide temporary, scoped delegation of key usage to a principal. Unlike key policies (permanent, modified via IAM), grants can be:
- Created programmatically (no key policy update needed)
- Retired when no longer needed
- Scoped to specific operations (`Decrypt` but not `Encrypt`)
- Constrained with encryption context

**Use case**: An AWS service (like AWS Nitro Enclaves, AWS services on your behalf) needs temporary access to a key. The service creates a grant, uses it, then retires it. No permanent key policy change required.

```bash
aws kms create-grant \
  --key-id alias/my-key \
  --grantee-principal "arn:aws:iam::111122223333:role/LambdaRole" \
  --operations Decrypt GenerateDataKey
```

🎯 **EXAM TIP**: If a question asks about "temporary delegation of KMS access without modifying key policy" → KMS Grants. Services like EBS, SSM, and RDS use grants internally when they encrypt data on your behalf.

### Multi-Region Keys

**Problem**: A global application encrypts PII in `us-east-1`, then needs to query that same data in `eu-west-1` for a GDPR compliance report. With regional KMS keys, you'd need to decrypt in us-east-1 and re-encrypt in eu-west-1 — a costly, latency-heavy roundtrip.

**Solution**: Multi-region keys have the same key ID and key material in multiple regions. Encrypt in one region, decrypt in another — no cross-region API call needed.

```
Primary key (us-east-1): mrk-1234abcd
Replica key  (eu-west-1): mrk-1234abcd (same key ID, same material)

Encrypt with mrk-1234abcd in us-east-1 → ciphertext
Decrypt with mrk-1234abcd in eu-west-1 → works! (same key material)
```

**Architecture considerations**:
- Primary key in one region; replicas in others
- Replicas become independent after replication (not continuously synced)
- Rotation must be performed on the primary key; replicas don't auto-rotate independently
- Cross-region replication of S3 objects encrypted with multi-region keys works seamlessly

🎯 **EXAM TIP**: Multi-region keys ≠ global keys. They are independent in each region after initial replication. "Encrypt in Region A, decrypt in Region B" → multi-region key. "Disaster recovery for encrypted DynamoDB global tables" → multi-region key.

### KMS Cross-Account Access

**Scenario**: Account A owns an encrypted EBS snapshot. Account B needs to copy and use it.

1. **Key Policy** (Account A): Must explicitly allow Account B's IAM role
2. **IAM Policy** (Account B): Must grant `kms:Decrypt`, `kms:ReEncrypt*`, `kms:GenerateDataKey*`
3. **Share the snapshot**: EC2 snapshot must also be shared with Account B

Both policies must permit the action. This "double allow" pattern is foundational to cross-account KMS.

---

## 5. Parameter Store

### Why Parameter Store Exists

Applications need configuration: database connection strings, feature flags, API endpoints, service URLs. Hardcoding these in your application code is a security and operational anti-pattern. Parameter Store provides a centralized, versioned, optionally-encrypted configuration repository.

**Analogy**: A hierarchical configuration database with built-in encryption and IAM-based access control.

### Tiers: Standard vs Advanced

| Feature | Standard (Free) | Advanced ($0.05/parameter/month) |
|---|---|---|
| Parameter count | Up to 10,000 | Up to 100,000 |
| Max value size | 4 KB | 8 KB |
| Parameter policies (TTL) | No | Yes |
| Cost | Free | $0.05/parameter/month |

**When to choose Advanced**: You need TTL (force periodic rotation), more than 10,000 parameters, or values larger than 4 KB.

### SecureString

SecureString parameters are encrypted with KMS. Unlike plain String types, accessing a SecureString requires:
1. IAM permission for `ssm:GetParameter` with `WithDecryption=true`
2. KMS permission for `kms:Decrypt` on the key used to encrypt it

This two-permission requirement enables fine-grained access control: a developer's IAM role might have SSM access but not KMS access, preventing them from reading production secrets.

```bash
# Store a database password securely
aws ssm put-parameter \
  --name "/myapp/prod/db-password" \
  --value "SuperSecretPassword" \
  --type SecureString \
  --key-id "alias/myapp-key"

# Retrieve and decrypt (requires both SSM + KMS permissions)
aws ssm get-parameter \
  --name "/myapp/prod/db-password" \
  --with-decryption
```

### Parameter Hierarchy

Parameters use path-based naming. This enables:
- IAM policies scoped to paths (`/myapp/prod/*` vs `/myapp/dev/*`)
- Batch retrieval by path (`GetParametersByPath`)
- Clean organization by environment, application, or team

```
/myapp/
  prod/
    db-password         (SecureString, KMS encrypted)
    db-endpoint         (String)
    feature-flags/
      dark-mode         (String: "true")
  dev/
    db-password         (SecureString)
    db-endpoint         (String)
```

```bash
# Get all production parameters at once
aws ssm get-parameters-by-path \
  --path "/myapp/prod" \
  --recursive \
  --with-decryption
```

### Parameter Policies and Rotation (Advanced Tier)

Advanced tier adds TTL policies that can:
- Force expiration (parameter auto-deleted after date)
- Trigger notifications before expiry (via EventBridge)
- Enable automatic rotation via Lambda triggered by expiry event

```
TTL Policy → Parameter expires → EventBridge triggers → Lambda rotates secret
                                                          ↓
                                               Updates DB + puts new value in SSM
```

🎯 **EXAM TIP**: SSM Parameter Store does NOT have native automatic rotation like Secrets Manager. Rotation in Parameter Store is achieved by combining Advanced tier TTL policies + EventBridge notifications + Lambda functions. If the question asks for "automatic, native rotation of RDS passwords" → Secrets Manager wins.

---

## 6. Secrets Manager

### Why Secrets Manager Exists (and When to Choose It Over Parameter Store)

Parameter Store is excellent for configuration + secrets. Secrets Manager is purpose-built for credentials that need automatic rotation. The key differentiator is **native integration with databases for zero-downtime password rotation**.

**Analogy**: Secrets Manager is like a password manager service that also automatically changes the password on a schedule and verifies the new password works before committing.

### Automatic Rotation — The Core Value

For RDS, Redshift, and DocumentDB, Secrets Manager handles the entire rotation process:

```
Rotation triggered (scheduled or manual)
    ↓
Lambda function (AWS-provided rotation function):
  1. Create new secret version (AWSPENDING stage)
  2. Update database password to match new secret
  3. Test connection with new credentials
  4. If test passes: promote AWSPENDING → AWSCURRENT
  5. Move old AWSCURRENT → AWSPREVIOUS (grace period for apps still using old)
  6. Delete AWSPREVIOUS after additional grace period
```

**Versioning labels**:
- `AWSCURRENT`: The active secret (applications use this)
- `AWSPREVIOUS`: Previous version (grace period — applications still using it will work)
- `AWSPENDING`: Being rotated (in flight)

This label-based rotation ensures zero downtime: applications that fetched the old password still work until AWSPREVIOUS is removed.

### Custom Rotation

For non-native services (MongoDB, Redis, custom APIs), you write a Lambda function that implements 4 steps:
1. `create_secret` — generate new credentials
2. `set_secret` — update the service with new credentials
3. `test_secret` — verify new credentials work
4. `finish_secret` — mark AWSPENDING as AWSCURRENT

### Cross-Account Access

A secret in Account A can be accessed by principals in Account B via the secret's **resource policy**.

```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT-B:role/AppRole"
  },
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "*"
}
```

Additionally, if the secret is encrypted with a CMK, Account B needs KMS access to that key.

🎯 **EXAM TIP**: Cross-account Secrets Manager access requires: (1) secret resource policy allows Account B's principal, AND (2) KMS key policy allows Account B's principal to decrypt. Both must be set.

### Secrets Manager vs Parameter Store — Decision Guide

| Requirement | Use |
|---|---|
| Auto-rotate RDS/Redshift/DocumentDB password | Secrets Manager (native integration) |
| Store 500 config values cheaply | SSM Parameter Store Standard (free) |
| Need TTL on parameters | SSM Parameter Store Advanced |
| Application secrets requiring audit trail | Either (both integrate with CloudTrail) |
| Cross-account credential sharing | Secrets Manager (resource policy + KMS) |
| Values larger than 4 KB (up to 65 KB) | Secrets Manager |
| Cost-sensitive: 100s of secrets | SSM Parameter Store ($0 vs $40/month) |

---

## 7. RDS Security

### Encryption at Rest

RDS encryption uses KMS. Key decisions:

- **Encrypt at creation time**: You cannot encrypt an existing unencrypted RDS instance in place
- **Migration path**: Create snapshot → copy snapshot with encryption → restore from encrypted snapshot
- **Read replicas**: Must be encrypted if the master is encrypted; can use a different key
- **Cross-region replicas**: Must have a KMS key in the destination region

🎯 **EXAM TIP**: "How do you encrypt an existing unencrypted RDS database?" Answer: Snapshot → copy with encryption enabled → restore. You cannot enable encryption on a running instance directly.

### Encryption in Transit (SSL/TLS)

RDS provides SSL/TLS certificates for encrypted connections. To enforce SSL:

- **MySQL/MariaDB**: `REQUIRE SSL` in user grant
- **PostgreSQL**: `ssl=1` parameter in parameter group
- **SQL Server/Oracle**: Force SSL via parameter group settings

For maximum security, combine in-transit encryption with the `rds.force_ssl` parameter set to `1` in the DB parameter group.

### IAM Database Authentication

Instead of database-native passwords, RDS supports IAM authentication for MySQL and PostgreSQL. The application's IAM role generates a temporary authentication token (valid 15 minutes), which replaces the database password.

**Benefits**:
- No long-lived passwords to rotate
- Centralized access control in IAM
- CloudTrail logs who authenticated when

**Architecture**:
```
EC2 Role → GenerateAuthToken (via RDS API) → Token (15 min TTL)
                                                    ↓
                                   Connect to RDS with token as password
```

🎯 **EXAM TIP**: IAM authentication for RDS is supported for MySQL and PostgreSQL only. It's ideal for EC2 applications where you want to eliminate static passwords. Tokens expire every 15 minutes, so the application must refresh them.

### TDE for Oracle and SQL Server

**Transparent Data Encryption (TDE)** encrypts database files at the storage level, transparent to the application. For Oracle and SQL Server on RDS, TDE is supported, and you can use **CloudHSM** as the key store instead of AWS KMS — useful when regulations require customer-managed HSM hardware.

---

## 8. SSL Encryption, SNI & MITM

### How SSL/TLS Handshake Works (Why It Matters for the Exam)

When a client connects to a server over HTTPS:
1. Client says: "I want to talk securely, here are the cipher suites I support"
2. Server responds: "Here's my certificate (with public key), here's my chosen cipher"
3. Client verifies: Certificate is signed by a trusted CA, hostname matches
4. Client generates session key, encrypts with server's public key
5. Server decrypts with private key → both have symmetric session key
6. Encrypted communication begins

All data in transit is protected. An eavesdropper sees only ciphertext.

### SNI — Server Name Indication

**Problem**: One load balancer (one IP address) needs to serve multiple HTTPS domains, each with its own SSL certificate. Before SNI, the server had to pick a certificate before knowing which hostname the client was connecting to.

**SNI Solution**: The client includes the **hostname it's connecting to** in the initial TLS handshake (ClientHello). The server sees the hostname, selects the correct certificate, and presents it.

```
Client → ALB: "I want to connect to api.company.com" (in ClientHello)
ALB: selects certificate for api.company.com
ALB → Client: "Here's the api.company.com certificate"
```

**AWS Implementation**:
- ALB and CloudFront support SNI natively
- You can attach multiple SSL certificates to a single ALB listener (one per domain)
- NLB supports SNI but only for pass-through TLS (no termination)

🎯 **EXAM TIP**: ALB supports SNI → multiple certificates on one listener → host-based routing. Classic Load Balancer does NOT support SNI. This is why migrating from CLB to ALB is often the answer for multi-domain HTTPS scenarios.

### Man-in-the-Middle (MITM) Prevention

**MITM**: Attacker intercepts communication and impersonates both client and server.

**Prevention mechanisms**:

1. **Certificate Validation**: Client checks server certificate is signed by trusted CA, hostname matches. MITM attacker cannot forge a certificate for `amazon.com` without the private key.

2. **Certificate Pinning**: Application only accepts specific certificates (not any CA-signed cert). Used for mobile apps connecting to specific APIs. Highly secure but inflexible (certificate update requires app update).

3. **HSTS (HTTP Strict Transport Security)**: Browser only connects via HTTPS to known domains. Prevents protocol downgrade attacks.

4. **HTTPS Everywhere**: Enforce TLS on all endpoints — even internal services. AWS Private CA enables TLS for internal services.

---

## 9. AWS Certificate Manager — ACM

### Why ACM Exists

Managing SSL certificates is operationally painful: purchasing from a CA, tracking expiry, manual renewal, distributing to servers. ACM automates all of this for AWS services.

**Key insight**: ACM certificates are free (for public certs). You never pay for the certificate — only for the compute serving HTTPS traffic.

### Public vs Private CA

| Type | Cost | Use Case |
|---|---|---|
| **Public Certificate** | Free | External-facing HTTPS (internet traffic) |
| **Private CA (ACM PCA)** | $400/month/CA + $0.75/cert | Internal services, mTLS, private PKI |

**Why Private CA?**: Internal microservices, VPNs, internal APIs need TLS but don't need a public CA. ACM Private CA lets you run your own CA hierarchy without managing HSMs.

### Automatic Renewal

ACM automatically renews public certificates 60 days before expiry — but only for certificates validated via DNS (not email). If you used DNS validation, renewal is fully automatic. If you used email validation, you must re-validate manually.

🎯 **EXAM TIP**: "Certificate expiry causing downtime" → migrate to ACM with DNS validation. ACM auto-renews. "Certificate not renewing automatically" → likely using email validation; switch to DNS validation.

### CloudFront + ACM Special Requirement

CloudFront is a global service but its TLS termination endpoints are distributed globally. However, AWS requires that the ACM certificate for CloudFront be in **`us-east-1` (N. Virginia)** — regardless of where your origin or users are.

🎯 **EXAM TIP**: "HTTPS on CloudFront" → certificate MUST be in `us-east-1`. This is one of the most commonly tested ACM facts. If the certificate is in `eu-west-1`, it cannot be attached to a CloudFront distribution.

### Integration Map

| Service | ACM Integration |
|---|---|
| ALB | Yes — certificate on listener |
| NLB | Yes — TLS listener |
| CloudFront | Yes — must be in us-east-1 |
| API Gateway | Yes — custom domain |
| Elastic Beanstalk | Yes — via ALB/NLB |
| EC2 directly | No — ACM private key cannot be exported |

🎯 **EXAM TIP**: You **cannot** export the private key of an ACM certificate. If you need to install a certificate directly on an EC2 instance, you must use a third-party certificate or use ACM Private CA with a self-managed certificate.

---

## 10. CloudHSM

### What Makes CloudHSM Different from KMS

KMS uses shared AWS HSMs (though isolated per key). CloudHSM gives you **dedicated hardware** that is physically allocated to your account.

```
KMS:      AWS HSM → multiple customers' keys (isolated logically)
CloudHSM: Your HSM → your keys only (isolated physically)
```

| Aspect | KMS | CloudHSM |
|---|---|---|
| Hardware | AWS shared (logical isolation) | Dedicated to you |
| FIPS Level | 140-2 Level 2 | 140-2 Level 3 |
| AWS access to keys | Possible (for AWS operations) | Zero — AWS cannot access keys |
| Cost | $1/month/key | $1.60/hour/HSM (~$1,200/month/HSM) |
| Complexity | Simple API | Complex (manage users, keys, TLS client) |
| Integration | Native AWS service integration | Limited — must use PKCS#11 or JCE |
| Key backup | AWS manages | You manage |

### Cluster Architecture for HA

For high availability, deploy CloudHSM in a **cluster** with one HSM per AZ:

```
VPC:
  AZ 1: HSM instance (subnet 1)
  AZ 2: HSM instance (subnet 2)
  AZ 3: HSM instance (subnet 3)

EC2 instances connect via CloudHSM client software to the cluster (active-active)
```

If one HSM fails, the cluster continues operating. Keys are replicated across all HSMs in the cluster automatically.

### Use Cases (When CloudHSM is the Right Answer)

1. **Regulatory mandate**: "Must use FIPS 140-2 Level 3 validated hardware"
2. **Government/defense**: Standards requiring dedicated hardware (not shared)
3. **Oracle TDE with customer key store**: Oracle databases requiring HSM-managed TDE keys
4. **Custom crypto operations**: PKCS#11 applications, custom key derivation
5. **PKI root CA**: Protect root CA private key in dedicated hardware

🎯 **EXAM TIP**: The exam tests you on FIPS 140-2 Level 3. KMS is Level 2; CloudHSM is Level 3. Any question mentioning "Level 3 compliance" or "dedicated HSM hardware" → CloudHSM. For all other cases, KMS is simpler and cheaper.

### CloudHSM + KMS (Custom Key Store)

You can configure KMS to use your CloudHSM cluster as the key store. This means:
- KMS API compatibility (all existing code works)
- Key material stored in your CloudHSM (never in AWS infrastructure)
- FIPS 140-2 Level 3 compliance
- Higher latency than native KMS

---

## 11. Solution Architecture — SSL on ELB

### End-to-End Encryption Patterns

There are three patterns for handling HTTPS at the load balancer layer:

**Pattern 1: TLS Termination at ALB (most common)**
```
Client → [HTTPS] → ALB (terminates TLS, ACM cert) → [HTTP] → EC2/ECS
```
- ACM certificate on ALB listener
- Traffic from ALB to backend is HTTP (unencrypted but within VPC)
- Simplest approach; backend handles no TLS overhead
- Security concern: backend traffic is unencrypted (acceptable within private VPC, not acceptable for compliance)

**Pattern 2: End-to-End Encryption (ALB + re-encryption)**
```
Client → [HTTPS] → ALB (terminates TLS, ACM cert) → [HTTPS] → EC2/ECS
```
- ALB terminates the client TLS, then re-encrypts to backend
- Backend EC2 needs its own certificate (can use ACM with self-signed or private CA cert)
- Two TLS sessions: client→ALB and ALB→backend
- Meets compliance requirements for "data must be encrypted in transit at all times"

🎯 **EXAM TIP**: "Data must be encrypted in transit even inside the VPC" → Pattern 2 (re-encryption). Use ACM Private CA certificates on the backend EC2 instances. ALB verifies the backend cert via a trust store.

**Pattern 3: TLS Pass-Through (NLB)**
```
Client → [HTTPS] → NLB (pass-through, no termination) → [HTTPS] → EC2/ECS
```
- NLB is Layer 4 — it passes TCP packets through unchanged
- The EC2 backend terminates TLS directly (has the private key)
- Client certificate-based auth (mTLS) is possible
- Use case: when the backend must see the original client TLS handshake (e.g., for certificate-based client authentication)

### ACM on ALB — Decision Points

```
Do you need backend certificate inspection?
  Yes → Use NLB with pass-through OR Pattern 2 with ALB trust store
  No  → Use Pattern 1 (ACM on ALB, HTTP to backend)

Does compliance require end-to-end encryption?
  Yes → Pattern 2 (ALB terminates + re-encrypts) or NLB pass-through
  No  → Pattern 1 is fine
```

---

## 12. S3 Security

### Layered Security Model

S3 security is a combination of multiple controls. Understanding how they interact is critical at the Pro level.

**Evaluation order**:
1. If Block Public Access is ON → deny all public access (overrides bucket policies)
2. If a Deny in bucket policy → denied
3. If an Allow in bucket policy → check if Block Public Access allows it
4. If no bucket policy → fall back to ACL (if enabled) or IAM (for same-account access)

### Bucket Policies

Resource-based JSON policies. Key patterns at SAP level:

**Enforce HTTPS**:
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::my-bucket/*"],
  "Condition": {"Bool": {"aws:SecureTransport": "false"}}
}
```

**Enforce SSE-KMS**:
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    }
  }
}
```

**Cross-account access**:
```json
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::ACCOUNT-B:role/AppRole"},
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```

🎯 **EXAM TIP**: Bucket policies can deny public access, require encryption, require HTTPS, or grant cross-account access. For cross-account object ownership, the PutObject must include `bucket-owner-full-control` ACL, otherwise the uploading account owns the objects.

### ACLs — When They're Still Relevant

ACLs are legacy. AWS recommends disabling them (Object Ownership = Bucket Owner Enforced). The only remaining reason to use ACLs: you need per-object permissions for objects uploaded by other AWS accounts (and even then, bucket policies are preferred).

### MFA Delete

MFA Delete requires the bucket owner (root account) to use MFA when permanently deleting a versioned object or suspending versioning. This protects against:
- Accidental deletion
- Compromised IAM credentials used to delete all versions

🎯 **EXAM TIP**: MFA Delete can ONLY be enabled by the root account. IAM users cannot enable or disable it. Versioning must be enabled first. This is a hard compliance control for regulatory requirements.

### S3 Object Lock — WORM Compliance

Object Lock prevents deletion or modification of objects for a defined period. Two retention modes:

| Mode | Who Can Override? | Use Case |
|---|---|---|
| **Compliance** | Nobody (including root) | Strongest: SEC, FINRA, healthcare records |
| **Governance** | Users with `s3:BypassGovernanceRetention` | Most orgs: protect from accidents, allow override for legitimate needs |

**Legal Hold**: Independent of retention period. Can be placed/removed by users with `s3:PutObjectLegalHold`. Protects objects indefinitely during litigation.

**Glacier Vault Lock**: Similar WORM concept for Glacier. Once you "lock" a vault lock policy, it cannot be changed — even by AWS. Used for 7-year SEC Rule 17a-4 compliance.

🎯 **EXAM TIP**: For regulatory archive requirements (SEC Rule 17a-4, CFTC, etc.) the answer is S3 Object Lock in Compliance mode or Glacier Vault Lock. These are the only AWS mechanisms where even root cannot delete data.

### CORS (Cross-Origin Resource Sharing)

CORS is a browser security mechanism. When a browser makes a request from origin A to origin B (different domain/port/scheme), the browser first sends a preflight `OPTIONS` request to B. B must respond with appropriate CORS headers allowing origin A.

**S3 CORS is needed when**: A JavaScript app hosted on `app.example.com` (or another S3 bucket) makes AJAX calls to your S3 bucket. Without CORS configured on the target bucket, the browser blocks the response.

🎯 **EXAM TIP**: CORS is browser-enforced, not a server-side security control. It doesn't prevent server-to-server calls. Only configure S3 CORS when JavaScript in browsers needs to access your bucket directly.

---

## 13. S3 Access Points

### Why Access Points Exist

Large organizations have many teams (Finance, HR, Analytics, Engineering) all accessing objects in one large S3 bucket. Managing bucket policies that serve all these teams becomes unmanageable — a single bucket policy has a 20 KB size limit.

**Access Points** give each team (or application) their own entry point with their own policy, their own DNS name, and their own VPC origin restriction if needed.

```
my-data-bucket
    ├── /finance/          ← finance-ap.s3-accesspoint.amazonaws.com
    ├── /hr/               ← hr-ap.s3-accesspoint.amazonaws.com
    └── /analytics/        ← analytics-ap.s3-accesspoint.amazonaws.com

finance-ap: finance team IAM roles can access /finance/* only
hr-ap: HR IAM roles can access /hr/* only (VPC-restricted, internet blocked)
analytics-ap: analytics roles get read-only access to /analytics/*
```

### VPC Origin Restriction

Access points can be restricted so they only accept requests from within a specific VPC. Internet access is blocked entirely.

```bash
aws s3control create-access-point \
  --account-id 123456789012 \
  --name finance-private-ap \
  --bucket my-data-bucket \
  --vpc-configuration VpcId=vpc-1234567890abcdef0
```

🎯 **EXAM TIP**: "Different IAM policies for different prefixes in one bucket" → S3 Access Points. "S3 bucket must only be accessible from inside a VPC" → Access Point with VPC origin restriction + bucket policy requiring `s3:DataAccessPointArn` condition.

### Bucket Policy + Access Point Interaction

The bucket policy must delegate authority to the access point:
```json
{
  "Effect": "Allow",
  "Principal": "*",
  "Action": "*",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringEquals": {
      "s3:DataAccessPointAccount": "123456789012"
    }
  }
}
```
This says "trust access points in this account to make the access control decisions."

---

## 14. S3 Multi-Region Access Points

### The Problem They Solve

A global application with users in multiple continents needs to read/write S3 data with low latency and high availability. Single-bucket S3 puts all data in one region — users in other regions experience higher latency. Multi-bucket solutions require application logic to route to the right region.

**Multi-Region Access Points (MRAP)** provide a single global endpoint that routes requests to the nearest S3 bucket replica, with automatic failover.

### Architecture

```
Global MRAP Endpoint (e.g., xyzabc.mrap.s3.global)
    ├── S3 Bucket (us-east-1)  ─────────────────┐ Active-Active
    ├── S3 Bucket (eu-west-1)  ─────────────────┤ (or Active-Passive for failover)
    └── S3 Bucket (ap-southeast-1) ─────────────┘

S3 Replication Rules: keep all buckets in sync (2-way replication)
```

**Failover Control**: You configure routing status per bucket (Active or Passive). In normal operation, all active buckets receive traffic. In disaster recovery, you can failover to switch a bucket from passive to active.

### Requirements

- All buckets must have S3 Replication configured (bidirectional for active-active)
- Replication time control (RTC) recommended: replicates 99.99% of objects within 15 minutes
- MRAP provides a unique ARN; requests use AWS Global Accelerator for routing

🎯 **EXAM TIP**: MRAP requires S3 Replication to be set up independently. The MRAP itself is just an intelligent routing layer — it doesn't replicate data. For active-active global S3 access with low latency, use MRAP + bidirectional replication.

---

## 15. S3 Object Lambda

### The Problem It Solves

You have one canonical version of data in S3, but different consumers need different views:
- Data scientists need full data
- External partners need PII redacted
- Mobile apps need images resized
- Analytics need format converted (JSON → CSV)

Without Object Lambda, you'd maintain multiple copies of the data. With Object Lambda, you transform on-the-fly during retrieval.

### Architecture

```
Consumer (GetObject request)
    ↓
S3 Object Lambda Access Point
    ↓
Lambda function triggered (receives original object)
    ↓
Lambda transforms (redact PII, resize, convert format)
    ↓
Transformed object returned to consumer

Original S3 bucket: unchanged (single source of truth)
```

### Use Cases

- **PII Redaction**: Marketing team gets customer data with SSNs/emails masked
- **Data Format Conversion**: Legacy app needs XML; source is JSON
- **Image Processing**: Resize product images based on requesting device type
- **Row-level Security**: Filter database export to show only rows the requester owns
- **Watermarking**: Add watermark to documents based on who's downloading

🎯 **EXAM TIP**: "Transform S3 data without storing multiple copies" → S3 Object Lambda. "Redact sensitive fields from S3 objects per requester" → S3 Object Lambda. The Lambda is invoked synchronously per GetObject call — latency consideration for high-throughput scenarios.

---

## 16. DDoS and AWS Shield

### Understanding the DDoS Threat

A DDoS attack floods your infrastructure with traffic to exhaust resources: bandwidth, CPU, connection tables, memory. Modern attacks combine multiple vectors:

- **Layer 3/4**: SYN floods, UDP amplification, ICMP floods (network/transport level)
- **Layer 7**: HTTP floods, slow HTTP attacks, Slowloris (application level)

### Shield Standard — Always On, Always Free

Shield Standard protects all AWS customers automatically with no configuration required:
- Absorbs SYN/UDP/reflection attacks at the AWS network edge
- Leverages AWS's massive global bandwidth (>100 Tbps) to absorb volumetric attacks
- Protects: EC2, ELB, CloudFront, Route53, Global Accelerator

What it doesn't cover: sophisticated, sustained attacks; real-time visibility; AWS expert support.

### Shield Advanced — The Enterprise Option ($3,000/month/organization)

**What you get beyond Standard**:

| Feature | Benefit |
|---|---|
| DDoS Response Team (DRT) | 24/7 experts who assist during active attacks |
| Advanced DDoS detection | Real-time attack visibility and metrics |
| Layer 7 DDoS protection | Application-layer attack mitigation (with WAF) |
| DDoS Cost Protection | AWS credits your bill if DDoS causes scaling charges |
| WAF included | No extra cost for WAF on protected resources |
| Protected resources | EC2, ALB, NLB, CloudFront, Global Accelerator, Route53 |

**DDoS Cost Protection** is unique: if an attacker causes your EC2 Auto Scaling group to launch 100x normal instances (costing you $200K), AWS credits the scaling costs caused by the DDoS attack.

🎯 **EXAM TIP**: "Company needs protection against DDoS with 24/7 expert support and cost reimbursement for scaling events caused by attacks" → Shield Advanced. "Company just needs basic DDoS protection" → Shield Standard (automatic, free, no action needed).

### DDoS-Resilient Architecture Pattern

```
Internet → Route53 (Shield Standard)
               ↓
         CloudFront (Shield Standard + WAF)
               ↓
           ALB (Shield Advanced)
               ↓
         EC2 Auto Scaling Group
```

CloudFront serves as the first line of defense: it absorbs Layer 7 attacks at edge locations worldwide, protecting your origin infrastructure from seeing most attack traffic.

---

## 17. AWS WAF

### Layer 7 Inspection — What Makes WAF Unique

WAF understands HTTP/HTTPS. It can inspect:
- URL path and query string
- HTTP headers (User-Agent, Referer, X-Forwarded-For)
- HTTP body (up to 8 KB inspected)
- IP address (source IP, or X-Forwarded-For headers)

This Layer 7 awareness means WAF can block SQL injection attempts, XSS payloads, and malformed HTTP requests that would bypass Layer 3/4 protections like NACLs and Security Groups.

### Where WAF Can Be Deployed

| Service | WAF Support | Why |
|---|---|---|
| ALB | Yes | Layer 7, HTTP-aware |
| CloudFront | Yes | Layer 7, global |
| API Gateway | Yes | REST/WebSocket APIs |
| AppSync | Yes | GraphQL |
| Cognito User Pool | Yes | Authentication layer |
| NLB | No | Layer 4, no HTTP awareness |
| EC2 directly | No | Must place ALB/CloudFront in front |

### Web ACL — Building Blocks

A Web ACL is a collection of rules evaluated in priority order. First match wins.

**Rule types**:
- **IP Set Match**: Allowlist/blocklist specific IPs or CIDRs
- **Geographic Match**: Block entire countries
- **String/Regex Match**: Inspect request components for patterns
- **Size Constraint**: Block requests with oversized bodies
- **Rate-Based**: Throttle IPs exceeding request rate (per 5-minute window)
- **Managed Rule Groups**: Pre-built rules from AWS or AWS Marketplace

### AWS Managed Rule Groups

**AWSManagedRulesCommonRuleSet**: Covers OWASP Top 10 (SQLi, XSS, LFI, RCE, SSRF)  
**AWSManagedRulesKnownBadInputsRuleSet**: Known malicious inputs  
**AWSManagedRulesBotControlRuleSet**: Bot detection and scoring  
**AWSManagedRulesAmazonIpReputationList**: Known AWS infrastructure abuse IPs

### Rate-Based Rules for Application DDoS

Rate-based rules automatically block IPs exceeding a threshold over a 5-minute rolling window:

```
Rule: Block source IP if > 2,000 requests in 5 minutes
Effect: Attacker's IP auto-blocked for 5 minutes, re-evaluated continuously
```

This is Layer 7 DDoS mitigation — protecting against HTTP floods that bypass Layer 3/4 protections.

🎯 **EXAM TIP**: "Block SQL injection attempts" → WAF with SQLi managed rules. "Block requests from France" → WAF geographic rule. "Throttle login endpoint to 100 req/min per IP" → WAF rate-based rule. "Block known bot traffic" → WAF Bot Control managed rule group.

### WAF Scope: Regional vs CloudFront

- **Regional WAF** (`--scope REGIONAL`): Protects ALB, API Gateway, AppSync in a specific region
- **CloudFront WAF** (`--scope CLOUDFRONT`): Must be created in `us-east-1` (global scope with CloudFront)

🎯 **EXAM TIP**: WAF for CloudFront must be configured in `us-east-1` — same constraint as ACM certificates for CloudFront.

---

## 18. AWS Firewall Manager

### The Problem at Scale

An organization with 200 AWS accounts needs consistent WAF rules across all ALBs, Shield Advanced protection on all CloudFront distributions, and compliant Security Groups everywhere. Doing this manually in each account doesn't scale — and accounts created in the future start unprotected.

**Firewall Manager** solves this: define security policies once at the organization level, automatically apply to all accounts (current and future).

### Prerequisites

- AWS Organizations must be enabled
- A designated Firewall Manager administrator account (typically the security account)
- Each member account's services automatically receive policies

### What Firewall Manager Can Manage

| Policy Type | What It Controls |
|---|---|
| WAF policy | Web ACLs applied to ALBs, CloudFront, API GW across all accounts |
| Shield Advanced policy | Enable protection on EC2, ELB, CloudFront, Route53 across all accounts |
| Security Group policy | Audit and enforce SG rules (e.g., no 0.0.0.0/0 on port 22) |
| Network Firewall policy | VPC-level firewall rules across accounts |
| Route53 DNS Firewall policy | DNS filtering rules |

### Policy Enforcement Modes

- **Auto-remediation**: Non-compliant resources are automatically fixed
- **Monitor mode**: Non-compliance is flagged but not fixed (for gradual rollout)

**Pro pattern**: Start with monitor mode to find current violations, review, then switch to auto-remediation.

🎯 **EXAM TIP**: "Enforce WAF rules across 50 AWS accounts automatically" → Firewall Manager. "New accounts automatically get security policies" → Firewall Manager (accounts joining OU get policies applied). "Centralized Security Group compliance" → Firewall Manager SG policy.

---

## 19. Blocking an IP Address

### The Decision Framework

Different threats require different blocking mechanisms. Understanding where in the stack to block is critical:

| Scenario | Recommended Mechanism | Why |
|---|---|---|
| Block a single IP from all traffic | NACL (Deny rule) | Stateless, applies to all traffic before it reaches any resource |
| Block a subnet/CIDR range | NACL | CIDR-based, fast, no per-connection state |
| Block Layer 7 (HTTP) patterns | WAF IP set rule | Can inspect HTTP headers, apply to ALB/CloudFront |
| Block IPs from a country | WAF geographic match rule | Country-level blocking at Layer 7 |
| Block an IP from CloudFront | CloudFront geo-restriction OR WAF IP set on CloudFront | Geographic at distribution level |
| Temporary throttle for DDoS | WAF rate-based rule | Auto-blocks IPs exceeding rate threshold |
| All of the above, centrally | Firewall Manager | Organization-wide enforcement |

### NACL vs Security Group vs WAF

| Property | NACL | Security Group | WAF |
|---|---|---|---|
| Layer | 3/4 (IP/Port) | 3/4 (IP/Port) | 7 (HTTP) |
| State | Stateless | Stateful | Stateful |
| Granularity | Subnet-level | Instance-level | Request-level |
| Deny rules | Yes | No (allow only) | Yes |
| Speed | Very fast (packet level) | Fast (connection level) | Slower (HTTP inspection) |

🎯 **EXAM TIP**: Security Groups cannot have explicit Deny rules — they only allow. To block a specific IP, you must use NACL (subnet level) or WAF (application level). "Block all traffic from 1.2.3.4" → NACL deny rule. "Block 1.2.3.4 only for HTTP requests to my web app" → WAF IP set.

### CloudFront IP Blocking Consideration

When using CloudFront, the EC2 origin sees CloudFront's IP, not the client's IP. A NACL blocking the client IP won't work at the EC2 level (it'll block legitimate CloudFront traffic). The correct approach:

```
Block client IP → WAF rule on CloudFront distribution (sees real client IP)
                OR
              CloudFront Geo-Restriction (block by country)
```

---

## 20. Amazon Inspector

### Why Inspector Exists

Software vulnerabilities (CVEs) are a primary attack vector. Inspector continuously scans your AWS resources against the National Vulnerability Database (NVD) and other threat intelligence feeds, identifying vulnerable software before attackers exploit it.

**Key distinction**: Inspector finds vulnerabilities in what you've deployed. GuardDuty detects active threats. Inspector is proactive; GuardDuty is reactive.

### Scanning Targets

| Target | What's Scanned | Requirement |
|---|---|---|
| **EC2 Instances** | OS packages, application packages, CVEs | SSM Agent (or agentless with Inspector v2) |
| **ECR Container Images** | Docker layer packages, CVEs | Push to ECR triggers scan |
| **Lambda Functions** | Code libraries, dependency packages | Enable Lambda scanning |

### Continuous vs On-Demand

Inspector v2 performs **continuous scanning** — not just scheduled. When a new CVE is published that affects a package you're running, Inspector re-evaluates your environment and creates a finding immediately, without waiting for the next scheduled scan.

### Risk Score Calculation

Inspector doesn't just report raw CVSS scores. It calculates a normalized risk score considering:
1. CVSS base score (severity of the vulnerability)
2. Network reachability (is the affected port exposed to the internet?)
3. Package exploitability (is there a known working exploit?)

This prioritization helps focus patching effort on the highest-risk vulnerabilities.

### Remediation Integration Pattern

```
Inspector Finding (high severity CVE in EC2)
    ↓
EventBridge event
    ↓
Lambda function
    ↓
Systems Manager Patch Manager → patch the instance automatically
OR
SNS → Notify operations team → manual patch
```

🎯 **EXAM TIP**: "Automated vulnerability scanning of EC2, ECR, and Lambda" → Inspector v2. "Scan container images for CVEs before deploying" → Inspector v2 with ECR integration. "Auto-patch EC2 instances when critical CVE detected" → Inspector + EventBridge + SSM Patch Manager.

---

## 21. AWS Config

### Why Config Exists (Beyond Monitoring)

CloudWatch tells you what's happening now (metrics). CloudTrail tells you who did what (audit). Config tells you **how your resource configurations have changed over time and whether those changes violated compliance policies**.

The key value: **configuration drift detection and automated remediation**.

### Config Rules — Managed vs Custom

**Managed Rules** (150+ available):
- `encrypted-volumes` — EBS volumes must be encrypted
- `s3-bucket-ssl-requests-only` — S3 buckets must deny HTTP
- `root-account-mfa-enabled` — Root account must have MFA
- `restricted-ssh` — No Security Groups with 0.0.0.0/0 on port 22

**Custom Rules** (Lambda functions):
- Check that EC2 AMIs come from an approved list
- Verify RDS instances have backup retention >= 7 days
- Validate that IAM roles follow your naming convention

### Conformance Packs

A **conformance pack** is a collection of Config rules and remediation actions packaged together for a compliance framework:
- PCI DSS conformance pack
- HIPAA conformance pack
- NIST 800-53 conformance pack
- CIS Benchmark conformance pack

Deploy a conformance pack → dozens of rules applied at once → compliance score calculated automatically.

🎯 **EXAM TIP**: "Implement CIS Benchmark across all accounts" → AWS Config conformance pack deployed via Firewall Manager or AWS Organizations. Individual rules take hours to set up; conformance packs do it in minutes.

### Automatic Remediation

Non-compliant resources can be automatically remediated using **SSM Automation Documents**:

```
Config Rule evaluation → Non-compliant resource found
    ↓
Remediation Action (SSM Automation Document)
    ↓
Examples:
- "EnableS3BucketEncryption" → enables encryption
- "RevokeUnrotatedKeys" → deletes IAM access keys older than 90 days
- "DisablePublicAccessForSecurityGroup" → removes 0.0.0.0/0 rules
```

Remediation can be:
- **Manual**: Operator reviews and triggers
- **Automatic**: Config triggers automatically on non-compliance

### Multi-Account Aggregation

Config **Aggregator** collects compliance data from multiple accounts and regions into a central account:

```
All member accounts (Config rules running locally)
    ↓ authorization
Central Security Account
    ↓ Aggregator
Compliance dashboard: all accounts, all regions, all rules
```

🎯 **EXAM TIP**: "View compliance posture across 200 AWS accounts" → Config Aggregator in central security account. Note: aggregation is read-only — you cannot apply rules from the aggregator to member accounts (use Firewall Manager for that).

---

## 22. AWS Managed Logs — Which Services Log Where

Understanding what gets logged where is critical for security investigations and compliance.

| AWS Service | Log Type | Destination | What's Captured |
|---|---|---|---|
| **CloudTrail** | API calls | S3 + CloudWatch Logs | Who called what API, when, from where |
| **VPC Flow Logs** | Network traffic | CloudWatch Logs, S3, or Kinesis Firehose | Source/dest IP, port, protocol, action (ACCEPT/REJECT) |
| **ELB Access Logs** | HTTP requests | S3 | Request details, client IP, latency, response code |
| **CloudFront Access Logs** | HTTP requests | S3 | Edge location, client IP, request, response |
| **Route53 Query Logs** | DNS queries | CloudWatch Logs | Queried domain, response, resolver IP |
| **S3 Server Access Logs** | S3 operations | S3 (different bucket) | Requester, bucket, operation, object, status |
| **WAF Logs** | Web requests | CloudWatch Logs, S3, Kinesis Firehose | Full HTTP request, rule match, action |
| **RDS Logs** | DB activity | CloudWatch Logs | Error, general, slow query, audit logs |
| **Lambda Logs** | Function output | CloudWatch Logs | stdout/stderr from function |
| **EKS Audit Logs** | K8s API calls | CloudWatch Logs | Kubernetes API activity |
| **GuardDuty Findings** | Threat detections | EventBridge, Security Hub | Finding details, affected resource |

### VPC Flow Logs — The Network Forensics Tool

VPC Flow Logs are particularly important at the SAP level. They capture:
```
<version> <account-id> <interface-id> <srcaddr> <dstaddr> <srcport> <dstport> <protocol> <packets> <bytes> <start> <end> <action> <log-status>
```

They do NOT capture:
- DNS traffic from instances (Route53 captures DNS query logs separately)
- DHCP traffic
- Metadata service traffic (169.254.169.254)
- License activation traffic
- Windows activation traffic

🎯 **EXAM TIP**: "Find out why a security group rule is blocking traffic" → VPC Flow Logs (look for REJECT actions). "Investigate network connection to suspicious IP" → VPC Flow Logs. "Capture DNS queries from all instances" → Route53 Resolver Query Logging.

---

## 23. Amazon GuardDuty

### What GuardDuty Is (and Isn't)

GuardDuty is a **threat detection service** — it identifies active threats and suspicious behavior in your AWS environment. It does NOT prevent attacks. Think of it as an intrusion detection system (IDS) rather than an intrusion prevention system (IPS).

**Key architectural fact**: GuardDuty works without agents. It analyzes existing AWS data sources that are already being generated.

### Data Sources Analyzed

| Data Source | What GuardDuty Detects |
|---|---|
| CloudTrail Management Events | Unusual IAM activity, console logins from suspicious locations |
| CloudTrail S3 Data Events | S3 data exfiltration, unusual access patterns |
| VPC Flow Logs | Communication with known malicious IPs, port scanning |
| DNS Logs | DNS lookups to command-and-control (C&C) domains |
| EKS Audit Logs | Kubernetes privilege escalation, unusual pod activity |
| RDS Login Events | Brute force attacks on databases |
| Lambda Network Activity | Lambda communicating with known malicious IPs |
| S3 Malware Protection | Uploaded objects scanned for malware signatures |

### Finding Types and Severity

**High Severity** (respond immediately):
- `CryptoCurrency:EC2/BitcoinTool.B` — Crypto mining (compromised EC2)
- `Trojan:EC2/DNSRequest.C` — DNS lookup to malware C&C domain
- `UnauthorizedAccess:IAM/AnomalousAPICall` — Unusual API pattern from IAM user

**Medium Severity** (investigate soon):
- `UnauthorizedAccess:EC2/SSHBruteforce` — SSH brute force detected
- `Recon:EC2/PortProbeUnprotectedPort` — Port scanning from external IP
- `Policy:S3/BucketPublicAccessGranted` — S3 bucket made public

### Multi-Account Setup

```
Organization Management Account
    ↓
Delegated GuardDuty Admin (security account)
    ↓ manages member accounts
Member Account A: GuardDuty findings → visible in admin account
Member Account B: GuardDuty findings → visible in admin account
Member Account C: GuardDuty findings → visible in admin account
```

In this setup, member account owners cannot disable GuardDuty (the admin account controls it).

### Automated Response Pattern

GuardDuty alone only detects. Automated response requires EventBridge:

```
GuardDuty Finding (e.g., CryptoCurrency:EC2)
    ↓
EventBridge Rule (match on finding type)
    ↓
Lambda Function:
  1. Stop the EC2 instance
  2. Snapshot the volume (forensics)
  3. Remove from ELB target group
  4. Isolate with restrictive security group
  5. Notify security team via SNS
  6. Create JIRA ticket
```

🎯 **EXAM TIP**: GuardDuty findings appear in EventBridge near real-time (configurable: 15 minutes, 1 hour, or 6 hours). For real-time automated response, configure 15-minute publishing frequency. GuardDuty does NOT automatically stop instances or block IPs — that requires EventBridge + Lambda.

### GuardDuty vs Other Services

| Service | Detects | When to Use |
|---|---|---|
| GuardDuty | Active threats, behavioral anomalies | "Is my account compromised?" |
| Inspector | Software vulnerabilities (CVEs) | "What vulnerabilities exist?" |
| Security Hub | Aggregates findings from both | "Single pane of glass" |
| Amazon Detective | Investigates why a finding occurred | "What happened and why?" |
| Macie | Sensitive data (PII) in S3 | "Where is our sensitive data?" |

---

## 24. IAM Advanced Policies

### Condition Keys — Scoping Permissions

Condition keys allow fine-grained control over when a permission applies:

**Common condition keys**:

| Condition Key | Example Use |
|---|---|
| `aws:SourceIp` | Only allow API calls from specific IP ranges |
| `aws:RequestedRegion` | Only allow actions in specific regions |
| `aws:MultiFactorAuthPresent` | Require MFA for sensitive actions |
| `aws:PrincipalTag/Department` | Match tag on the principal making the request |
| `s3:prefix` | Scope S3 access to specific key prefixes |
| `kms:CallerAccount` | KMS key policy — allow only from specific account |

**Example: Require MFA for sensitive IAM actions**:
```json
{
  "Effect": "Deny",
  "Action": ["iam:*", "sts:AssumeRole"],
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}
  }
}
```

### NotAction — Deny Everything Except

`NotAction` is the inverse of `Action`. Combined with Deny, it denies everything except the listed actions:

```json
{
  "Effect": "Deny",
  "NotAction": [
    "iam:CreateVirtualMFADevice",
    "iam:EnableMFADevice",
    "sts:GetSessionToken"
  ],
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}
  }
}
```

This policy says: "If MFA is not present, deny everything except setting up MFA." This is the SCP/IAM pattern for enforcing MFA enrollment.

🎯 **EXAM TIP**: `NotAction` with `Deny` = "deny everything except these actions." This appears in SCP patterns for forcing MFA, forcing specific regions, or blocking service shutdowns.

### PassRole — The Delegation Permission

`iam:PassRole` allows a principal to assign an IAM role to an AWS service. Without PassRole, a user could not create a Lambda function with a specific execution role — even if they have full Lambda permissions.

**Why this matters for security**: A user with EC2 full access but no `iam:PassRole` cannot launch an EC2 instance with an instance profile that has more permissions than the user. This prevents privilege escalation via service roles.

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::123456789012:role/LambdaExecutionRole",
  "Condition": {
    "StringEquals": {"iam:PassedToService": "lambda.amazonaws.com"}
  }
}
```

🎯 **EXAM TIP**: "User can create Lambda functions but cannot choose the execution role" → missing `iam:PassRole`. "Prevent privilege escalation via EC2 instance profiles" → restrict `iam:PassRole` to specific roles only.

### Permission Boundaries at Scale

Permission boundaries set the maximum permissions an IAM entity can have, regardless of what policies are attached. They're crucial for delegated administration:

**Use case**: You want developers to create IAM roles for their applications, but not create roles with admin access.

```
Developer's IAM Policy: "Allow iam:CreateRole, iam:PutRolePolicy"
Developer's Permission Boundary on their own role: Cannot escalate beyond DeveloperBoundary

When developer creates a new role:
  They must attach PermissionBoundary = "DeveloperBoundary" to the new role
  The new role can only have permissions within DeveloperBoundary
  Even if the new role policy says "Allow: *", boundary caps it
```

**AWS Organizations SCPs + Permission Boundaries**:
- SCPs apply to the entire account (set by org admin)
- Permission Boundaries apply to specific IAM entities (set by account admins)
- Effective permissions = IAM Policy ∩ Permission Boundary ∩ SCP

🎯 **EXAM TIP**: Permission boundaries prevent privilege escalation during delegated role creation. "Allow developers to create IAM roles but prevent them from creating admin roles" → permission boundaries. Effective permissions are the intersection — all three must allow for access to be granted.

---

## 25. EC2 Instance Connect

### Why EC2 Instance Connect Exists

Traditional SSH to EC2 requires:
1. Generate an SSH key pair
2. Store private key securely (loses, stolen keys = compromised access)
3. Open port 22 in security group to the network
4. Manage key rotation

EC2 Instance Connect eliminates key pair management for temporary SSH access:

### How It Works

```
1. User authenticates to AWS Console/CLI (with their IAM identity)
2. IAM verifies permission: ec2-instance-connect:SendSSHPublicKey
3. EC2 Instance Connect API pushes a one-time temporary public key to the instance
   (key is valid for 60 seconds — just long enough for SSH handshake)
4. SSH session established using that temporary key
5. Session recorded; access visible in CloudTrail
```

**Security advantages**:
- No long-lived SSH keys to manage or rotate
- Access controlled entirely by IAM (users/roles/groups)
- Every session logged in CloudTrail (who accessed which instance, when)
- Can restrict port 22 to the EC2 Instance Connect Service IP range only

**Security group requirement**:
- Only needs to allow inbound SSH from EC2 Instance Connect IP ranges for your region
- Can completely remove individual developer IP ranges from security groups

🎯 **EXAM TIP**: "Eliminate SSH key management for EC2 access" → EC2 Instance Connect. "Audit all SSH sessions to EC2 instances" → EC2 Instance Connect + CloudTrail (every `SendSSHPublicKey` call is logged). "Browser-based SSH without installing an SSH client" → EC2 Instance Connect console.

---

## 26. AWS Security Hub

### The Single Pane of Glass for Security

Security Hub aggregates security findings from multiple services and presents them in a normalized format (Amazon Security Finding Format — ASFF), with compliance scores and automated checks.

**Why this matters at SAP level**: A large organization running GuardDuty, Inspector, Macie, Config, and IAM Access Analyzer across 50 accounts generates thousands of findings. Security Hub is the central dashboard where the SOC team monitors everything.

### Data Sources

| Source | What It Provides |
|---|---|
| GuardDuty | Threat detection findings |
| Inspector | Vulnerability findings |
| Macie | Sensitive data findings |
| IAM Access Analyzer | External access findings |
| AWS Config | Compliance findings |
| Firewall Manager | Security policy compliance |
| Third-party tools | Splunk, Jira, CrowdStrike, etc. |

### Automated Security Standards Checks

Security Hub continuously evaluates your environment against:
- **CIS AWS Foundations Benchmark** (v1.2 and v1.4)
- **AWS Foundational Security Best Practices**
- **PCI DSS v3.2.1**
- **NIST SP 800-53 Rev 5**
- **HIPAA Final Omnibus Security Rule**

Each check generates a pass/fail with a security score (0-100%) per standard.

### Cross-Account and Cross-Region Aggregation

```
Security Hub Admin (security account)
    ↓ aggregates from
Member Account A (all regions)
Member Account B (all regions)
Member Account C (all regions)

Single security score for entire organization
```

🎯 **EXAM TIP**: "Single security dashboard across all accounts" → Security Hub. "Automated compliance checks against PCI DSS" → Security Hub with PCI DSS standard enabled. Security Hub itself doesn't do remediation — it aggregates and prioritizes; remediation is via EventBridge + Lambda or direct Config remediation.

---

## 27. Amazon Detective

### Why Detective Exists — Beyond Finding to Understanding

GuardDuty tells you *what* happened (crypto mining detected on EC2 instance). Detective tells you *why* and *how* — what the full attack chain was, what resources were involved, what preceded the compromise.

**Analogy**: GuardDuty is the smoke detector; Detective is the fire investigator who pieces together how the fire started.

### How Detective Works

Detective analyzes behavioral data over time (up to a year) and builds a graph model:
- IAM principals and their API activity patterns
- Network traffic patterns from VPC Flow Logs
- GuardDuty finding history
- CloudTrail events

When a GuardDuty finding occurs, Detective lets you visualize:
- What other resources did this IP communicate with?
- Is this API call rate unusual for this user?
- What did this user do before and after the suspicious call?
- Is this EC2 instance's network behavior abnormal for its peer group?

### Data Sources

Detective automatically ingests (no configuration needed):
- VPC Flow Logs
- CloudTrail
- GuardDuty findings
- EKS audit logs (if GuardDuty EKS protection enabled)

### Integration with GuardDuty

From a GuardDuty finding, you can click "Investigate in Detective" to launch the graph-based investigation. Detective shows the full context around the finding.

🎯 **EXAM TIP**: "Root cause analysis of a GuardDuty finding" → Amazon Detective. "Understand the full blast radius of a security incident" → Detective. "Machine learning-based investigation of security events" → Detective. Detective is NOT for blocking — only for understanding.

---

## 28. Decision Framework

### Which Service for Which Threat/Requirement

#### Threat Detection & Response

| Threat | Detect With | Respond With |
|---|---|---|
| Compromised EC2 (crypto mining) | GuardDuty | EventBridge → Lambda (stop instance) |
| SQL injection attack | WAF | WAF block action (automatic) |
| DDoS (Layer 3/4) | Shield Standard | Auto-mitigated by AWS |
| DDoS (Layer 7 HTTP flood) | WAF Rate-based rule | WAF block action |
| Unusual IAM API pattern | GuardDuty or CloudTrail Insights | EventBridge → SNS alert |
| Sensitive data in S3 | Macie | IAM policy + encryption |
| Software CVE in EC2 | Inspector | SSM Patch Manager |
| External access to S3 bucket | IAM Access Analyzer | Fix bucket policy |
| Malware in uploaded S3 object | GuardDuty (S3 protection) | Lambda quarantine |

#### Compliance & Audit

| Requirement | Service |
|---|---|
| "Who deleted the database?" | CloudTrail |
| "Show me all changes to security groups" | Config (resource history) |
| "Ensure all EC2 have encrypted volumes" | Config rule + remediation |
| "PCI DSS compliance score" | Security Hub |
| "FIPS 140-2 Level 3 key storage" | CloudHSM |
| "Auto-rotate RDS credentials" | Secrets Manager |
| "Audit all SSH access to EC2" | EC2 Instance Connect + CloudTrail |
| "Scan all S3 buckets for PII" | Macie |

#### Access Control

| Need | Service |
|---|---|
| Block a specific IP | NACL (any traffic) or WAF (HTTP) |
| Block a country | WAF geographic rule or CloudFront geo-restriction |
| Centralize WAF rules across 50 accounts | Firewall Manager |
| Temporary KMS key delegation | KMS Grants |
| Cross-account S3 access | Bucket policy + IAM |
| Cross-account KMS access | Key policy + IAM (both accounts) |
| Per-prefix S3 access control | S3 Access Points |

#### Encryption Decisions

| Scenario | Solution |
|---|---|
| Default S3 encryption | SSE-S3 (automatic since 2023) |
| S3 with audit trail + key control | SSE-KMS with CMK |
| Bring your own key to AWS | SSE-C or imported KMS key material |
| Encrypt data outside AWS | KMS Asymmetric key (public key export) |
| Encrypt 5 GB file with KMS | Envelope encryption (GenerateDataKey) |
| Global app: encrypt in US, decrypt in EU | KMS Multi-Region Key |
| Regulatory: dedicated HSM hardware | CloudHSM |

---

## Security Services Comparison Table

| Service | Category | What It Does | NOT Suitable For |
|---|---|---|---|
| **CloudTrail** | Audit | Records all API calls | Real-time monitoring |
| **Config** | Compliance | Tracks config changes, enforces rules | Real-time threat detection |
| **GuardDuty** | Threat Detection | Behavioral anomaly detection | Blocking attacks |
| **Inspector** | Vulnerability | CVE scanning of EC2/ECR/Lambda | Threat detection |
| **Macie** | Data Security | PII/sensitive data discovery in S3 | Non-S3 data |
| **Security Hub** | Aggregation | Central findings dashboard + compliance scoring | Individual service monitoring |
| **Detective** | Investigation | Root cause analysis of incidents | Prevention |
| **WAF** | Prevention | Block Layer 7 web attacks | Layer 3/4 attacks, NLB |
| **Shield Standard** | Prevention | Layer 3/4 DDoS absorption | L7 attacks, expert support |
| **Shield Advanced** | Prevention + Response | Advanced DDoS + DRT + cost protection | General security |
| **Firewall Manager** | Governance | Org-wide WAF/Shield/SG policy enforcement | Single account |
| **KMS** | Encryption | Key management, envelope encryption | FIPS L3, dedicated hardware |
| **CloudHSM** | Encryption | Dedicated HSM, FIPS L3 | Simple use cases (use KMS) |
| **ACM** | Certificates | TLS certificate provisioning and renewal | Private key export |
| **Secrets Manager** | Secrets | Auto-rotating credentials | Configuration values (use SSM) |
| **Parameter Store** | Config + Secrets | Hierarchical config, optional encryption | Auto-rotation (use Secrets Manager) |
| **IAM Access Analyzer** | Access | External resource access detection | Active threat detection |
| **EC2 Instance Connect** | Access | Keyless SSH access with IAM + audit | Persistent SSH key management |

---

*Section 4: Security — AWS Solutions Architect Professional*  
*Exam: SAP-C02 | Style: Expert-led — explain the WHY*
