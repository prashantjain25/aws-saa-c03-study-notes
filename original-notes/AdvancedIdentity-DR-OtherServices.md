# AWS SAA-C03: Advanced Identity, Disaster Recovery & Other Services

## Table of Contents
- [1. AWS Organizations](#1-aws-organizations)
- [2. AWS IAM Identity Center](#2-aws-iam-identity-center)
- [3. AWS Directory Service](#3-aws-directory-service)
- [4. AWS Control Tower](#4-aws-control-tower)
- [5. AWS Resource Access Manager (RAM)](#5-aws-resource-access-manager-ram)
- [6. AWS STS (Security Token Service)](#6-aws-sts-security-token-service)
- [7. Identity Federation & SAML 2.0](#7-identity-federation--saml-20)
- [8. AWS Cognito (Advanced)](#8-aws-cognito-advanced)
- [9. Disaster Recovery Strategies](#9-disaster-recovery-strategies)
- [10. AWS Backup](#10-aws-backup)
- [11. AWS DataSync](#11-aws-datasync)
- [12. AWS Elastic Disaster Recovery (DRS)](#12-aws-elastic-disaster-recovery-drs)
- [13. AWS Application Migration Service (MGN)](#13-aws-application-migration-service-mgn)
- [14. AWS Database Migration Service (DMS)](#14-aws-database-migration-service-dms)
- [15. AWS Snow Family](#15-aws-snow-family)
- [16. AWS Transfer Family](#16-aws-transfer-family)
- [17. AWS Batch & AppRunner](#17-aws-batch--apprunner)
- [18. AWS Storage Services](#18-aws-storage-services)
- [19. AWS Analytics Services](#19-aws-analytics-services)
- [20. AWS Machine Learning Services](#20-aws-machine-learning-services)
- [21. AWS Other Services](#21-aws-other-services)
- [22. Points to Remember (Exam Focus)](#22-points-to-remember-exam-focus)
- [23. AWS CLI Quick Reference](#23-aws-cli-quick-reference)

---

## 1. AWS Organizations

### What & Why
AWS Organizations lets you manage multiple AWS accounts from a single master (management) account. Think of it as a **corporate holding company structure for cloud accounts**. Why? Blast radius isolation — if one account gets compromised, your other production accounts stay safe. Also, volume discounts compound across all accounts.

### Key Concepts
- **Management Account**: the "parent" account that owns the organization
- **Organizational Units (OUs)**: hierarchical grouping of accounts (e.g., Dev OU → Dev-A, Dev-B; Prod OU → Prod-A, Prod-B)
- **Consolidated Billing**: single payment method, single invoice, but costs tracked per account. Volume discounts across org (e.g., 20 Reserved Instances split across accounts = 20-instance discount tier)
- **Root**: top level of OU hierarchy

### Service Control Policies (SCPs)
SCPs are **permission boundary policies** — they do NOT grant permissions, they restrict what's allowed. Even the AWS account root user is constrained by SCPs.

**Key difference**: IAM policies grant/deny for users/roles within an account; SCPs set max permissions for an entire account or OU.

**Inheritance**: SCPs cascade down the OU tree. If a Dev OU has an SCP denying DynamoDB, all accounts under Dev OU cannot use DynamoDB.

**Default policy**: `FullAWSAccess` — allows everything (must explicitly restrict).

**Common exam scenarios**:
- Prevent accounts from leaving org: SCP denies `organizations:LeaveOrganization`
- Restrict region usage: SCP denies services outside `us-east-1` and `eu-west-1`
- Enforce CloudTrail: SCP denies `cloudtrail:StopLogging`
- Block public S3: SCP denies `s3:PutBucketPublicAccessBlock` if Action = "Allow" and Principal = "*"

### Why AWS Accounts Isolation
A database admin in Account A makes a mistake (drops production table by accident). With separate accounts, Account B (Prod) is unaffected — different credentials, different blast radius. This is **the gold standard for large organizations**.

### Reserved Instances & Savings Plans
RI and Savings Plans can be shared across the organization at the management account level, even if purchased by individual accounts. This maximizes coverage of expensive compute commitments.

---

## 2. AWS IAM Identity Center (formerly AWS SSO)

### What It Is
A single sign-on service that grants employees access to **multiple AWS accounts** and **cloud applications** (Salesforce, Box, Office 365, GitHub, etc.) with one set of credentials.

**Real-world analogy**: You scan your badge at the office door once → you get access to your desk, the library, the cafeteria, and the parking lot. One badge, multiple resources.

### Architecture
1. **Identity Source**: where users come from
   - Built-in Identity Center directory (manage users in IAM Identity Center itself)
   - AWS Managed Microsoft AD (full Active Directory in AWS)
   - External IdP via SAML 2.0 (Okta, Azure AD, Active Directory)
   
2. **Permission Sets**: templates that define what access a user gets in each account
   - Maps to IAM roles in the target account
   - Can use AWS pre-defined policies or custom policies
   - Supports Attribute-Based Access Control (ABAC): grant access based on user attributes (department=Finance, role=Manager)

3. **Assignment**: map user/group + permission set → target account

### Why It's Powerful
Imagine you have 20 AWS accounts. Without IAM Identity Center, you'd need 20 sets of credentials. With it: one login, access all accounts via temporary credentials issued by STS under the hood.

### Exam Tip
- "SSO across multiple AWS accounts" → IAM Identity Center
- "Centralized access control, manage permissions from one place" → IAM Identity Center
- "Attribute-based access: users with department=HR only access HR account" → IAM Identity Center with ABAC

---

## 3. AWS Directory Service

### Three Options
AWS provides three flavors of managed directory services:

#### 3a. AWS Managed Microsoft AD
- **Full Active Directory** running in AWS
- Can establish **trust relationships** with on-premises AD (bi-directional or one-way)
- Supports MFA (multi-factor authentication)
- Deployed across multiple AZs for HA
- Use case: "We have on-prem AD and want it to extend into AWS and sync"

#### 3b. AD Connector
- **Gateway/proxy** that redirects AD requests to your on-premises AD
- No directory data stored in AWS — it's pass-through
- Users authenticate directly against on-premises AD servers
- Use case: "Existing on-prem AD, want AWS services to use it, don't want to sync/replicate"
- Limitation: no caching, so on-prem AD must be reachable

#### 3c. Simple AD
- **Samba-based**, low-scale, low-cost directory
- Suitable for <5000 users
- Good for: basic LDAP functionality, testing
- Limitation: does NOT support trust relationships, no MFA, NOT compatible with RDS SQL Server
- Use case: "Small company, basic directory needed, budget-conscious"

### Which One? Decision Tree
- Have on-prem AD and want full sync + trust? → **AWS Managed Microsoft AD**
- Have on-prem AD but want it to stay on-prem, just use from AWS? → **AD Connector**
- New directory, simple needs, <5000 users? → **Simple AD**

---

## 4. AWS Control Tower

### What It Does
Control Tower automates the setup of a multi-account AWS environment using AWS Organizations. It's the **"quick start" for enterprise governance**.

### Landing Zone
A **well-architected baseline** for your organization:
- Sets up AWS Organizations
- Creates foundational OUs (Security, Sandbox, Prod)
- Deploys account baselines (CloudTrail, Config, guardrails)
- AWS SSO integration for user access

### Account Factory (Account Vending Machine)
Self-service account provisioning: a developer submits a request → Control Tower auto-provisions a new account with baseline configs. Eliminates manual account setup.

### Guardrails
Rules enforced across all accounts:

**Preventive Guardrails** (use SCPs):
- Disallow public S3 buckets
- Disallow modification of CloudTrail
- Disallow deletion of CloudTrail logs

**Detective Guardrails** (use AWS Config rules):
- Detect unencrypted EBS volumes
- Detect security groups allowing port 22 to 0.0.0.0
- Detect untagged resources

### Dashboard
Real-time compliance dashboard across all accounts → see which accounts violate which guardrails.

### Exam Pattern
"Need to set up multi-account governance quickly with compliance monitoring" → **Control Tower**

---

## 5. AWS Resource Access Manager (RAM)

### What It Does
**Share AWS resources across AWS accounts** (within your organization or with external accounts) **without VPC peering, Route53 private zones, or VPN**.

### Shareable Resources
- VPC subnets (most important for SAA)
- Transit Gateway
- Route53 Resolver rules
- License Manager configurations
- Aurora DB clusters
- EC2 Dedicated Hosts
- Capacity Reservations
- Prefix Lists
- AWS Glue databases

### How It Works: Shared Subnets Example
1. Account A (owner) creates a VPC and subnet
2. Account A creates a Resource Share, includes the subnet, adds Account B as a participant
3. Account B accepts the share
4. Now both Account A and B can launch EC2 instances in the **same subnet**
5. Important: Account B cannot modify or delete the subnet (Account A owns it), but can launch resources

### Why This Matters
Without RAM, if you had two AWS accounts in the same organization and wanted them to share network infrastructure, you'd use VPC peering (complex routing) or a shared VPC design. RAM simplifies this.

### Exam Tip
- "Share a subnet between two accounts" → RAM
- "Participant account should NOT be able to modify shared resource" → That's fine with RAM (owner controls, participant just uses)
- "Share Transit Gateway routes across accounts" → RAM

---

## 6. AWS STS (Security Token Service)

### What It Does
Issues **temporary, limited-privilege credentials** (Access Key ID + Secret Access Key + Session Token + Expiration). Credentials are valid from minutes to hours (max 12 hours for AssumeRole).

**Why it matters**: Never store long-term credentials on EC2 or mobile apps. Instead, request temporary creds from STS.

### Key APIs

#### AssumeRole
Assume a role in the same or different AWS account.

**Requires**:
- Caller has `sts:AssumeRole` permission
- Target role's trust policy allows the caller
- Cross-account: trust policy must name the external account's principal

**Returns**: temporary credentials valid up to 12 hours

**Example flow**:
```
Lambda in Account A needs to read S3 in Account B
  → Lambda assumes role "CrossAccountS3Reader" in Account B
  → STS issues temp credentials
  → Lambda uses them to call S3 API in Account B
```

#### AssumeRoleWithSAML
Enterprise federation using SAML 2.0 assertion. Your on-premises AD authenticates a user → sends SAML assertion → STS exchanges it for AWS credentials.

#### AssumeRoleWithWebIdentity
Mobile/web app uses social login (Amazon, Google, Facebook) or Cognito token → exchange token for AWS credentials.

**Recommendation**: Use Cognito Identity Pools (wraps STS) rather than calling raw STS for web/mobile apps.

#### GetSessionToken
Request temporary credentials with MFA enforcement. Used to extend MFA protection to programmatic API calls.

**Flow**:
1. User passes Access Key + Secret Key + MFA device code to STS
2. STS verifies MFA, returns temp credentials
3. Those temp credentials require MFA for sensitive actions

#### GetFederationToken
Custom federation for non-SAML environments (less common in modern setups).

### STS Conditions & Security

**External ID**: Prevents "confused deputy" problem in cross-account roles.

**Scenario**: Acme Corp (third-party) wants to assume role in your account to manage your infrastructure. If you simply allow Acme's AWS account to assume the role, an attacker at Acme could potentially use the same role trust. 

**Solution**: Include `sts:ExternalId` in the role trust policy. Acme must provide the external ID when calling AssumeRole — only they know it.

**Example trust policy**:
```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::ACME_ACCOUNT_ID:root" },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "UNIQUE_EXTERNAL_ID_ONLY_ACME_KNOWS"
    }
  }
}
```

### Revoke Sessions
If a user's credentials are compromised, you can revoke all their active STS sessions by modifying the role's trust policy to deny tokens issued before a certain time (using `sts:TokenIssueTime` condition).

### Exam Patterns
- "Lambda needs temporary access to S3 in another account" → STS AssumeRole
- "Enforce MFA for API calls" → GetSessionToken
- "Mobile app with Google login → access S3" → Cognito Identity Pool (uses AssumeRoleWithWebIdentity internally)
- "Third-party vendor needs to access our infrastructure, prevent confused deputy" → AssumeRole with ExternalId

---

## 7. Identity Federation & SAML 2.0

### Concept
**Federation** = granting external identities access to AWS resources **without creating IAM users in that account**. One identity system (your corporate AD), multiple systems that trust it (AWS, Salesforce, GitHub).

### SAML 2.0 (Security Assertion Markup Language)
Enterprise standard for SSO.

**Parties**:
- **IdP (Identity Provider)**: your on-premises Active Directory, Okta, etc.
- **SP (Service Provider)**: AWS in this case
- **Assertion**: signed XML document from IdP proving user identity

**Flow**:
1. User opens AWS Console in browser
2. Browser redirected to corporate login portal (IdP)
3. User enters AD credentials
4. AD authenticates, generates SAML assertion (signed XML)
5. Assertion posted to AWS STS endpoint
6. STS validates assertion signature, issues temporary credentials
7. User gets AWS console access

**Key**: User never stores AWS credentials; they use corporate AD credentials and get temporary AWS access.

### Custom Identity Broker
For non-SAML compatible IdPs (legacy systems), build a custom broker:

1. Broker authenticates user against legacy system
2. Broker calls STS GetFederationToken or AssumeRole
3. Broker issues back a URL with temp credentials (AWS Console Federation)
4. User opens URL, gets AWS console access

### Web Identity Federation
For web/mobile apps with social logins:

1. User logs in via Amazon, Google, Facebook, or OIDC provider
2. Provider returns JWT token
3. App exchanges token for AWS credentials (via STS or Cognito)
4. App uses credentials to call AWS APIs (S3, DynamoDB, etc.)

**Best practice**: Use Cognito Identity Pools, not raw STS calls.

### IAM Identity Center (SSO)
Built on SAML 2.0 + OIDC; simplifies federation for AWS Organizations.

### Exam Scenario
- "Employees log in with corporate AD, need AWS Console access" → SAML 2.0 federation
- "Legacy system that doesn't support SAML" → Custom Identity Broker
- "Mobile app with social login needs S3 access" → Cognito Identity Pool
- "Org-wide SSO for 50 AWS accounts" → IAM Identity Center

---

## 8. AWS Cognito (Advanced)

### Two Components

#### User Pools
**User directory** — manage sign-up, sign-in, password resets, email verification.

- Returns **JWT tokens** (ID token for user info, access token for resource access)
- Supports social login: Google, Facebook, Amazon, SAML IdPs
- Built-in hosted UI: pre-built sign-in page (no coding)
- Can customize with custom auth flows
- Lambda triggers: execute code on pre-auth, post-auth, pre-token-generation, custom-message, etc.
- Advanced security: risk detection (unusual login patterns), compromised credentials check

**Example**: "Sign in with Google, get JWT tokens, call your API Gateway"

#### Identity Pools (Federated Identities)
**Exchanges tokens for AWS credentials**.

You get a JWT (from User Pool, social IdP, or SAML) → exchange it for temporary AWS credentials (via STS AssumeRole) → use those credentials to call AWS services (S3, DynamoDB, etc.).

**Key insight**: User Pool handles authentication (who you are); Identity Pool handles authorization (what AWS resources you can access).

### Combined Flow
```
User signs in to app (User Pool)
  ↓ (get JWT)
User requests S3 file from app
  ↓
App exchanges JWT for temp AWS creds (Identity Pool)
  ↓ (calls STS AssumeRole)
App uses temp creds to call S3 GetObject
  ↓
S3 returns file (restricted by IAM role's policy)
```

### Cognito Sync (Deprecated)
Synced user data across devices (e.g., game progress on phone syncs to tablet). Now deprecated in favor of AppSync.

### Exam Patterns
- "App needs to authenticate users and call API Gateway" → User Pool (JWT authorizer in API Gateway)
- "Users need direct access to S3/DynamoDB" → User Pool + Identity Pool
- "Social login → AWS service access" → Identity Pool
- "Lambda trigger to send custom welcome email on sign-up" → User Pool triggers

---

## 9. Disaster Recovery Strategies

### Why It Matters
**RPO (Recovery Point Objective)**: Maximum acceptable data loss (measured in time). If RPO = 1 hour, you lose at most 1 hour of data.

**RTO (Recovery Time Objective)**: Maximum acceptable downtime. If RTO = 30 minutes, service must be back up within 30 minutes.

Different strategies have different RPO/RTO/cost tradeoffs.

### 9a. Backup & Restore

**How**: Regular snapshots/backups to S3 or cross-region storage. When disaster strikes, restore infrastructure and data in DR region.

**RPO**: Hours
**RTO**: Hours
**Cost**: $ (cheapest — you only pay for storage, not compute/network)
**Suitable for**: Non-critical systems, dev/test environments

**Example**: 
- RDS automated backups every hour to S3 (cross-region)
- EC2 instances: create AMI weekly, store in S3
- On disaster, launch new EC2 from AMI, restore RDS from backup
- Service up in ~2-3 hours

### 9b. Pilot Light

**How**: A minimal version of the app always runs in the DR region (often just the database tier).

**Analogy**: A pilot light on a gas stove — a small flame burning, ready to ignite the full appliance.

**RPO**: Minutes
**RTO**: Tens of minutes
**Cost**: $$ (low — only critical minimal components always running)

**Example**:
- Primary: RDS MySQL + 10 m5.xlarge EC2 instances + ALB in us-east-1
- DR: RDS MySQL read replica + AMIs ready but **no instances running**
- On disaster: Launch EC2 instances from AMI, promote RDS replica to primary, update Route53
- Service up in ~10-20 minutes, data loss is minimal (last few minutes)

### 9c. Warm Standby

**How**: A reduced-scale version of the full app runs in DR region. On disaster, scale up to production capacity.

**RPO**: Seconds to minutes
**RTO**: Minutes
**Cost**: $$$ (moderate — reduced-scale resources always running)

**Example**:
- Primary: RDS MySQL + 10 m5.xlarge EC2 + ALB in us-east-1
- DR: RDS MySQL read replica + 2 t3.small EC2 + ALB + ASG configured to scale to 10 instances
- On disaster: promote RDS replica, scale ASG to 10, update Route53
- Service up in ~5-10 minutes at reduced capacity, then scales to full capacity in another 5 minutes

### 9d. Multi-Site Active-Active (Hot Standby)

**How**: Full production capacity in **both** regions, load balancer routes to both (or both serve traffic simultaneously).

**Analogy**: You have two identical factories running in parallel — both produce goods, both ship to customers.

**RPO**: Near zero (both regions always in sync)
**RTO**: Near zero (seconds — just failover DNS, both regions already active)
**Cost**: $$$$ (most expensive — running full production in two regions)

**Example**:
- Primary: RDS MySQL + 10 m5.xlarge EC2 in us-east-1
- DR: RDS MySQL (replicated from Primary) + 10 m5.xlarge EC2 in eu-west-1
- Route53 latency/weighted routing or active-active with both regions serving live traffic
- If us-east-1 fails, Route53 instantly routes to eu-west-1 (already serving traffic)

### Comparison Table
| Strategy | RPO | RTO | Cost | Use Case |
|----------|-----|-----|------|----------|
| **Backup & Restore** | Hours | Hours | $ | Non-critical, dev/test |
| **Pilot Light** | Minutes | 10s min | $$ | Important but tolerates brief downtime |
| **Warm Standby** | Seconds | Minutes | $$$ | Critical, requires fast recovery |
| **Multi-Site Active-Active** | ~0 | ~0 | $$$$ | Mission-critical, 24x7 uptime required |

### AWS Services for DR

**Route53**: Health checks + failover routing (failover records, latency records, weighted records, geolocation)

**S3 Cross-Region Replication (CRR)**: Automatic replication of every object/delete from source region to destination. Near-real-time.

**RDS**:
- Multi-AZ: synchronous replica in standby AZ (automatic failover within seconds, same region)
- Read Replicas in DR region: eventual consistency, manual promotion required

**DynamoDB Global Tables**: Multi-region replication, automatic failover, millisecond replication

**Aurora Global Database**: One primary region (read/write), up to 5 secondary regions (read-only). Failover in <1 minute, RPO = 1 second.

**CloudFormation StackSets**: Deploy infrastructure via CloudFormation to multiple regions/accounts automatically

**EC2 AMIs**: Backup and restore via custom AMIs (store in S3 or copy to DR region)

---

## 10. AWS Backup

### What It Does
**Centralized backup management** across 15+ AWS services. One policy, many services.

**Supported services**: EC2, EBS volumes, RDS, Aurora, DynamoDB, DocumentDB, Neptune, S3, EFS, FSx, Storage Gateway, VMware, SAP HANA, etc.

### Key Components

**Backup Plans**:
- Define frequency (hourly, daily, weekly, monthly)
- Define retention (keep for N days/months/years)
- Define lifecycle (move to cold storage after X days, then delete after Y days)
- Apply to multiple resources via tags

**Backup Vaults**:
- Container for backups
- Default: regional, encrypted
- Backup Vault Lock: WORM (Write Once Read Many) — once locked, even root user cannot delete backups (compliance requirement)

**Cross-Region Backup**:
- Automatically copy backups to another region for DR
- Helps with RPO/RTO

**Cross-Account Backup**:
- Copy backups to a separate AWS account (e.g., central backup account)
- Prevents compromised account from deleting backups

### Point-in-Time Restore
Supported services (RDS, Aurora, DynamoDB, EBS) allow restore to any moment in time (within retention window).

### Exam Tip
- "Backup 15 different AWS services with one policy" → AWS Backup
- "Prevent accidental/malicious backup deletion for compliance" → Backup Vault Lock
- "Need cross-account backups for security" → AWS Backup cross-account

---

## 11. AWS DataSync

### What It Does
**Online data transfer service** for large-scale data migration or ongoing replication. Transfers between on-premises and AWS, or between AWS storage services.

**Speed**: Up to 10 Gbps per task; faster than S3 Transfer Acceleration for bulk data.

### Key Components

**DataSync Agent**:
- Lightweight software agent deployed on-premises (on a server or VM)
- Connects to DataSync service endpoint
- Handles source authentication (NFS mount, SMB share, etc.)

**DataSync Task**:
- Defines source, destination, frequency, bandwidth limit, scheduling
- Incremental transfers: only changed data transferred

### Features
- **Encryption in transit**: TLS
- **Encryption at rest**: integrates with KMS
- **Bandwidth throttling**: limit network consumption
- **Scheduling**: run tasks at specific times/frequencies
- **Verification**: automatically verifies data integrity

### Supported Storage
- **Source**: NFS (network file system), SMB (Windows file share), AWS storage (S3, EFS, FSx)
- **Destination**: S3 (any storage class), EFS, FSx for Windows, FSx for Lustre

### Comparison
- **DataSync vs S3 Transfer Acceleration**: DataSync is managed service with automatic optimization; S3TA is purely accelerated upload; DataSync better for bulk migration
- **DataSync vs Storage Gateway**: DataSync = migration/sync (one-time or periodic); Storage Gateway = ongoing hybrid access (on-prem like local storage)
- **DataSync vs Snowball**: DataSync for <100 TB online transfer; Snowball for 100+ TB or offline transfer

### Exam Scenario
- "Migrate 50 TB NAS to AWS S3" → DataSync
- "Sync on-premises NFS to EFS daily" → DataSync
- "Migrate 500 TB offline" → Snowball (then DataSync for subsequent syncs)

---

## 12. AWS Elastic Disaster Recovery (DRS)

### What It Does (formerly CloudEndure)
**Continuous block-level replication** of physical/virtual servers to AWS. Real-time replication, minimal downtime recovery.

### How It Works
1. Install DRS agent on source server (physical, VMware, Hyper-V, etc.)
2. Agent replicates disk blocks to AWS staging area (cheap compute environment)
3. DRS maintains copy in sync continuously
4. On disaster, launch recovery instances from replicated data (minutes)
5. Post-recovery, can replicate back on-premises (failback)

### RPO/RTO
- **RPO**: Seconds (near-continuous replication)
- **RTO**: Minutes (launch full-spec instances from replicated data)

### Cost
- Minimal: only pay for staging area compute (t3.small instances), not full production spec

### Use Cases
- Migrate physical servers from on-premises → AWS
- Migrate from other clouds (Azure, Google Cloud, others) → AWS
- Disaster recovery for on-premises workloads

### Exam Tip
"Disaster recovery with continuous replication, minimal downtime, can failback" → DRS

---

## 13. AWS Application Migration Service (MGN)

### What It Does
**Simplified lift-and-shift migration** from physical, virtual, or cloud servers to AWS. Replaces CloudEndure Migration.

### Flow
1. Install MGN agent on source servers
2. Agent replicates disk data to AWS (staging area — low-cost EC2)
3. Test launch: spin up test instances from replicated data
4. Cutover: launch production instances
5. Source servers can remain on (optional)

### Key Difference from DRS
- **MGN**: optimized for migration (lift-and-shift); simpler, faster setup
- **DRS**: optimized for disaster recovery; emphasizes failback capability

### Exam Pattern
"Lift-and-shift migration with minimal downtime" → MGN

---

## 14. AWS Database Migration Service (DMS)

### What It Does
**Continuous, low-downtime database migration** from on-premises/other clouds to AWS. Source database remains available during migration.

### Scenarios

**Homogeneous Migration**:
- Source: MySQL 5.7, Target: RDS MySQL 8.0
- Simple, no schema conversion needed
- DMS replicates data directly

**Heterogeneous Migration**:
- Source: Oracle, Target: Aurora PostgreSQL
- Requires **AWS Schema Conversion Tool (SCT)**
- SCT converts Oracle schemas → PostgreSQL schemas
- DMS replicates converted data

### Key Components

**DMS Replication Instance**:
- EC2-based compute that runs the migration
- Reads from source, writes to target
- Multi-AZ option for HA

**Continuous Data Replication (CDC - Change Data Capture)**:
- Starts full load (copy all existing data)
- Then captures ongoing changes (inserts, updates, deletes) from source logs
- Keeps target in near real-time sync
- Allows minimal downtime cutover (switch app to target, CDC finishes sync, validate, swap)

### Supported Databases
- **Source**: Oracle, SQL Server, MySQL, PostgreSQL, MongoDB, SAP, DB2, Azure SQL, others
- **Target**: RDS (all engines), Aurora, Redshift, DynamoDB, S3, ElasticSearch, Kinesis, others

### Exam Scenarios
- "Migrate on-premises MySQL to RDS MySQL with minimal downtime" → DMS
- "Migrate Oracle to Aurora PostgreSQL, change schemas" → DMS + SCT
- "Ongoing replication from on-prem DB to Redshift for analytics" → DMS CDC to Redshift

---

## 15. AWS Snow Family

### Why It Exists
For large data transfers, network bandwidth is often the bottleneck. Example: 500 TB on 1 Gbps connection = ~50 days of continuous transfer. AWS Snow devices ship physical data.

### Snowcone
- **Smallest**: 8 TB HDD or 14 TB SSD usable storage
- **Rugged**: portable, connects via USB-C
- **Features**: supports DataSync, up to 2 vCPUs, good for edge computing
- **Shipping**: mail-in/return

### Snowball Edge Storage Optimized
- **Capacity**: 80 TB usable
- **Compute**: up to 40 vCPUs + 32 GB RAM
- **Use case**: large migrations, batch processing
- **Shipping**: courier delivery/return

### Snowball Edge Compute Optimized
- **Capacity**: 42 TB usable
- **Compute**: up to 52 vCPUs + GPU
- **Use case**: edge computing + migration
- **Example**: run ML inference on oil rig, ship data home

### Snowmobile
- **Capacity**: 100 PB per truck
- **Logistics**: AWS truck shows up, connects to your data center
- **Use case**: exabyte-scale migrations
- **Cost**: very high but amortized over 100 PB

### OpsHub
GUI management tool for Snow devices: configure, monitor, transfer data.

### Decision Logic
- **<10 TB**: regular S3 transfer is fine
- **10 TB - 1 PB**: Snowball or DataSync depending on transfer speed
- **>1 PB**: Snowmobile or split across multiple Snowballs
- **Disconnected location (oil rig)**: Snowball Edge Compute Optimized (run Lambda/EC2 AMIs, do work, ship back)

### Exam Calculation
"Transfer 50 TB over 1 Gbps connection. Should we use Snowball?"
- Time without Snowball: 50 TB / 1 Gbps = 400 Gbit / 1 Gbit = 400 seconds... wait, that's bandwidth not throughput. Actually: 50 * 8 Terabits / 1 Gbps = 400,000 seconds ≈ 5 days continuous. With network latency, retries, more like 1-2 weeks. Snowball shipping (2-3 days) + on-site transfer (hours) = faster overall.

---

## 16. AWS Transfer Family

### What It Does
**Fully managed SFTP, FTPS, FTP, AS2 servers** — move files to/from S3 and EFS without running your own SFTP infrastructure.

### Key Features
- **Protocols**: SFTP (secure), FTPS (FTP+TLS), FTP (unsecured), AS2 (business document exchange)
- **Endpoints**: public (internet-facing) or VPC (private, within VPC)
- **Integration**: IAM auth, CloudWatch logs, KMS encryption
- **No server management**: AWS manages scaling, patching, HA

### Exam Tip
"Legacy SFTP server/clients, want managed service in AWS" → AWS Transfer Family

---

## 17. AWS Batch & AppRunner

### AWS Batch

**What**: Fully managed **batch computing** at any scale.

**Use case**: Run 100,000s of jobs in parallel (genomic analysis, financial modeling, media transcoding, etc.)

**How it works**:
1. Write job as Docker container
2. Define Job Queue and Job Definition
3. Submit jobs to queue
4. Batch provisions EC2 On-Demand or Spot, runs jobs, scales down when done
5. Pay per job execution

**Key differences**:
- **vs Lambda**: Lambda max 15 min execution, 10 GB RAM; Batch no limit, can run for hours
- **vs ECS**: ECS for long-running services; Batch for batch workloads (start-process-finish-stop)
- **Cost optimization**: uses Spot instances for batch workloads (cheap, okay to interrupt)

**Example**:
```
Video transcoding: 1000 video files
  → submit 1000 Batch jobs (one per file, containerized ffmpeg)
  → Batch launches 100 EC2 spots, processes in parallel
  → completes in hours (vs weeks with single server)
  → total cost $20 (spot pricing)
```

### AWS AppRunner

**What**: Fully managed service to deploy **web apps and APIs** without infrastructure knowledge.

**How it works**:
1. Push source code to GitHub (or provide container image)
2. AppRunner builds (if needed), deploys to load-balanced, auto-scaled environment
3. Built-in HTTPS, health checks, auto-scaling

**Comparison to ECS/EKS**:
- **ECS**: manage container orchestration, tasks, services, load balancers (requires ops knowledge)
- **AppRunner**: "just deploy my code", no ops needed (simpler but less control)

**VPC support**: can connect AppRunner to private RDS/ElastiCache in VPC.

**Exam scenario**: "Deploy a web app without managing infrastructure or containers" → AppRunner (simpler than ECS); "Need orchestration control" → ECS

---

## 18. AWS Storage Services

### S3 Transfer Acceleration
- Upload files to CloudFront edge location (geographically closer) → fast transfer over AWS backbone to S3
- Enable per-bucket: `--accelerate` flag or special endpoint
- Cost: additional per-GB charge on top of S3
- Exam: "global users uploading large files to S3" → S3 Transfer Acceleration

### Storage Gateway (recap)
**File Gateway**: NFS/SMB interface → S3 backend. On-premises apps see S3 as file share.

**Volume Gateway**: iSCSI block device → EBS snapshots (stored or cached volumes). On-premises servers see EBS volume via iSCSI.

**Tape Gateway**: Virtual Tape Library interface → S3 and Glacier. Legacy backup software uses "tapes" (actually S3/Glacier).

---

## 19. AWS Analytics Services

### Amazon Athena
**What**: Serverless **SQL on S3 data**.

- Query S3 directly without loading into database
- Supports CSV, JSON, ORC, Avro, Parquet
- Pricing: $5 per TB scanned (columnar formats reduce this)
- No ETL needed

**Example**: 
```
SELECT count(*) FROM s3_logs WHERE status_code = 500
  → Athena scans S3, returns result in seconds, costs ~$0.50 (depending on data size)
```

**Use cases**: 
- Analyze CloudTrail logs (who did what, when)
- ALB access logs
- VPC flow logs

**Federated Queries**: Connect Athena to RDS, DynamoDB, on-premises databases via Lambda connectors.

### Amazon Redshift
**What**: **Petabyte-scale data warehouse** for OLAP (analytics) workloads.

- Columnar database (optimized for analytics, not OLTP)
- SQL-based, similar to PostgreSQL
- Cluster architecture: leader node + compute nodes
- Redshift Spectrum: query S3 data from Redshift without loading

**Comparison to Athena**:
- Athena: serverless, pay per query, good for ad-hoc queries, small-medium datasets
- Redshift: provisioned cluster, pay per node-hour, good for large/complex analytics, BI dashboards

**Exam**: "Data warehouse for BI reporting" → Redshift; "Ad-hoc SQL on S3 logs" → Athena

### Amazon OpenSearch (formerly Elasticsearch)
**What**: **Full-text search and analytics engine**.

- Complement to DynamoDB: DynamoDB for key-value access, OpenSearch for text search ("find all products with keyword 'blue'")
- Often combined: DynamoDB Streams → Lambda → OpenSearch (index new items)
- OpenSearch Dashboards for visualization

**Example**: E-commerce site, users search products → query OpenSearch → fast results

### Amazon EMR (Elastic MapReduce)
**What**: Managed Hadoop/Spark cluster for big data processing.

- Run Spark jobs on 1000s of EC2 nodes
- Cost: leverage Spot instances for savings
- Auto-scaling: add nodes as job needs

### Amazon QuickSight
**What**: Serverless **BI and visualization** service.

- Connect to RDS, Redshift, Athena, S3, Salesforce, etc.
- Create dashboards and visualizations
- SPICE: in-memory engine for fast queries
- Pricing: per-user or per-session (good for embedded dashboards)

### AWS Glue
**What**: Serverless **ETL (Extract, Transform, Load)** service.

**Components**:
- **Glue Data Catalog**: central repository of metadata (tables, columns, formats, locations)
- **Glue Crawler**: scans S3, RDS, Redshift → auto-discovers schema → populates Data Catalog
- **Glue ETL Jobs**: Spark-based scripts (Python/Scala) that transform data (read from source, transform, write to target)
- **Glue DataBrew**: visual data preparation (no coding)

**Exam**: "ETL from S3 to Redshift" → Glue; "Auto-discover S3 data schema" → Glue Crawler

---

## 20. AWS Machine Learning Services

### Amazon Rekognition
- **Image analysis**: detect objects, faces, text (OCR), inappropriate content
- **Video analysis**: detect persons, activities, text in video
- Example: "moderate user-uploaded photos for nudity" → Rekognition

### Amazon Transcribe
- **Speech → text**: convert audio/video to text transcripts
- Example: "transcribe customer support calls for QA" → Transcribe

### Amazon Polly
- **Text → speech**: natural-sounding audio from text
- Example: "audiobook from PDF" → Polly

### Amazon Translate
- **Language translation**: convert text between 100+ languages
- Example: "Translate support tickets from Spanish to English" → Translate

### Amazon Comprehend
- **NLP (Natural Language Processing)**: sentiment analysis, key phrase extraction, PII (personally identifiable information) detection
- Example: "Find PII in customer feedback, redact it" → Comprehend

### Amazon Lex
- **Chatbots**: build conversational interfaces
- Same tech as Alexa (ASR + NLU)
- Example: "Customer support chatbot that answers common questions" → Lex

### Amazon Connect
- **Cloud contact center**: phone/chat customer service platform
- Integrates with Lex for intelligent routing
- Example: "Build customer service center in AWS" → Connect

### Amazon SageMaker
- **Full ML platform**: build, train, deploy custom ML models
- High-level: provides notebooks (Jupyter), algorithms (regression, classification, clustering), hosting for inference
- Example: "Fraud detection: train model on transaction history, deploy for real-time scoring" → SageMaker

### Amazon Forecast
- **Time-series forecasting**: predict future values based on historical patterns
- Example: "Predict inventory demand for next quarter" → Forecast

### Amazon Kendra
- **Intelligent search**: ML-powered enterprise search (better than keyword search)
- Example: "Search internal wiki/documents with natural language" → Kendra

### Amazon Personalize
- **Recommendations engine**: real-time recommendations (like Amazon.com "customers who bought this also bought...")
- Example: "E-commerce site recommendations" → Personalize

### Amazon Textract
- **OCR + document extraction**: extract text and tables from documents (PDFs, images, scans)
- Example: "Extract data from insurance claim forms" → Textract

---

## 21. AWS Other Services

### AWS IoT Core
- **Connect IoT devices**: publish/subscribe messaging (MQTT)
- Rules Engine: route messages to Lambda, S3, DynamoDB, Kinesis, etc.
- Example: "Temperature sensors publish readings → IoT Rules → store in DynamoDB"

### Amazon WorkSpaces
- **Managed virtual desktops**: Windows/Linux desktops as a service
- Persistent (data saved), integrates with AD
- Example: "Remote workers need secure desktop access" → WorkSpaces

### Amazon AppStream 2.0
- **Stream applications to browser**: no full desktop, just individual apps (AutoCAD, Visual Studio, etc.)
- Per-user or per-hour pricing
- Example: "SaaS application accessible from any browser" → AppStream 2.0

### AWS Outposts
- **AWS infrastructure in your data center**: actual AWS racks deployed in your facility
- Same APIs, services, tools as cloud (EC2, RDS, ECS, etc.)
- Use case: latency-sensitive workloads, data residency requirements
- Example: "Manufacturing plant needs sub-10ms latency to analytics, on-premises data residency" → Outposts

### AWS Local Zones
- **AWS compute closer to major cities**: extend VPC to local zone
- Single-digit ms latency (compared to 10-50ms to regional data center)
- Example: "Real-time financial trading from NYC" → Local Zone in New York

### AWS Wavelength
- **AWS on 5G edge**: AWS infrastructure within telecom 5G providers (Verizon, others)
- Ultra-low latency (<10ms), great for mobile edge computing
- Example: "Real-time mobile gaming with <10ms latency" → Wavelength

---

## 22. Points to Remember (Exam Focus)

### Organizations & Governance
- **Organizations**: multi-account management, consolidated billing, volume discounts
- **SCPs**: permission boundaries, inherited down OU tree, do NOT grant permissions
- **Control Tower**: automated landing zone, guardrails (preventive + detective), account factory
- **IAM Identity Center**: one login across multiple accounts and cloud apps (replaces AWS SSO)
- **Resource Access Manager**: share resources (subnets, Transit Gateway, etc.) across accounts

### Identity & Access
- **Directory Service**: Managed AD (full + trust), AD Connector (proxy), Simple AD (Samba, basic)
- **STS**: temporary credentials, AssumeRole, MFA via GetSessionToken, prevent confused deputy with ExternalId
- **SAML 2.0**: enterprise federation via XML assertions, user → AD → SAML → STS → AWS
- **Cognito**: User Pools (directory) + Identity Pools (exchange token for AWS creds)

### Disaster Recovery
- **RPO/RTO tradeoff**: Backup (hours/hours, cheapest) → Pilot Light (min/10s min) → Warm Standby (sec/min) → Active-Active (near-zero/near-zero, expensive)
- **AWS Backup**: centralized backup, Vault Lock (WORM), cross-region/cross-account
- **Aurora Global**: 1s RPO, <1 min RTO
- **DynamoDB Global Tables**: multi-region, millisecond replication
- **Route53**: health checks + failover routing for DR

### Migration & Transfer
- **DataSync**: online migration/sync, NFS/SMB → S3/EFS/FSx
- **DRS**: continuous replication, failback capability (DR-focused)
- **MGN**: lift-and-shift (migration-focused)
- **DMS**: database migration with CDC, homogeneous or heterogeneous (+ SCT)
- **Snowball**: offline bulk transfer (10 TB - 1 PB)
- **Transfer Family**: managed SFTP/FTPS/FTP to S3/EFS

### Compute & Batch
- **Batch**: 100,000s jobs in containers, auto-scaling, no time limit (vs Lambda: 15 min max)
- **AppRunner**: deploy web app without ops, simpler than ECS

### Analytics
- **Athena**: serverless SQL on S3, $5/TB scanned
- **Redshift**: data warehouse, columnar, clusters
- **Glue**: ETL service, Data Catalog, Crawler (auto-discover schema)
- **OpenSearch**: full-text search, complement to DynamoDB
- **EMR**: big data, Spark/Hadoop clusters

### Machine Learning (Concepts)
- **Rekognition**: image/video analysis
- **Transcribe**: speech → text
- **Lex**: chatbots
- **SageMaker**: full ML platform
- **Comprehend**: NLP (sentiment, PII)
- **Textract**: OCR + document extraction
- **Personalize**: recommendations

### Infrastructure
- **Outposts**: AWS in your data center (on-prem data residency)
- **Local Zones**: AWS near major cities (single-digit ms latency)
- **Wavelength**: AWS on 5G edge (mobile edge computing)

---

## 23. AWS CLI Quick Reference

```bash
# ============ AWS ORGANIZATIONS ============

# Create organization (management account)
aws organizations create-organization --feature-set all

# List all accounts
aws organizations list-accounts

# Create new account in organization
aws organizations create-account \
  --email user@example.com \
  --account-name "Dev-Account"

# Get account details
aws organizations describe-account --account-id 123456789012

# Create organizational unit
aws organizations create-organizational-unit \
  --parent-id r-1234 \
  --name "Development"

# List OUs under parent
aws organizations list-organizational-units-for-parent --parent-id r-1234

# Attach SCP to account/OU
aws organizations attach-policy \
  --policy-id p-1234abcd \
  --target-id 123456789012

# List policies attached to target
aws organizations list-policies-for-target --target-id 123456789012 --filter SERVICE_CONTROL_POLICY

# Create SCP
aws organizations create-policy \
  --content file://policy.json \
  --description "Restrict to specific regions" \
  --type SERVICE_CONTROL_POLICY

# ============ AWS STS ============

# Assume role (basic)
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name my-session

# Assume role with external ID (prevent confused deputy)
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name my-session \
  --external-id UNIQUE_EXTERNAL_ID

# Assume role with duration (default 1 hour, max 12 hours)
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name my-session \
  --duration-seconds 3600

# Get temporary credentials as JSON (jq for parsing)
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name my-session | jq '.Credentials'

# Get session token (MFA enforcement)
aws sts get-session-token \
  --serial-number arn:aws:iam::123456789012:mfa/user \
  --token-code 123456 \
  --duration-seconds 3600

# ============ AWS BACKUP ============

# Create backup plan
aws backup create-backup-plan \
  --backup-plan file://backup-plan.json

# List backup plans
aws backup list-backup-plans

# Get backup plan details
aws backup get-backup-plan --backup-plan-id 01234567-89ab-cdef-0123-456789abcdef

# Create backup vault
aws backup create-backup-vault --backup-vault-name my-vault

# Enable backup vault lock (WORM)
aws backup put-backup-vault-lock-configuration \
  --backup-vault-name my-vault \
  --min-retention-days 7

# Start backup job
aws backup start-backup-job \
  --backup-vault-name my-vault \
  --recovery-point-arn arn:aws:ec2:us-east-1:123456789012:volume/vol-1234abcd \
  --iam-role-arn arn:aws:iam::123456789012:role/service-role/AWSBackupDefaultServiceRole

# List backup jobs
aws backup list-backup-jobs

# Get recovery points for resource
aws backup list-recovery-points-by-resource \
  --resource-arn arn:aws:ec2:us-east-1:123456789012:volume/vol-1234abcd

# ============ AWS DATASYNC ============

# List DataSync locations
aws datasync list-locations

# Create NFS location
aws datasync create-location-nfs \
  --subdirectory "/data" \
  --server-hostname 192.168.1.100 \
  --on-prem-config AgentArns=arn:aws:datasync:us-east-1:123456789012:agent/agent-0123456789abcdef0

# Create S3 location
aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::my-bucket \
  --subdirectory "/prefix" \
  --s3-config BucketAccessRoleArn=arn:aws:iam::123456789012:role/DataSyncS3Role

# Create task (source → destination)
aws datasync create-task \
  --source-location-arn arn:aws:datasync:us-east-1:123456789012:location/nfs/1234567890abcdef0 \
  --destination-location-arn arn:aws:datasync:us-east-1:123456789012:location/s3/1234567890abcdef0 \
  --options VerifyMode=POINT_IN_TIME_CONSISTENT

# Start task execution
aws datasync start-task-execution --task-arn arn:aws:datasync:us-east-1:123456789012:task/task-0123456789abcdef0

# List task executions
aws datasync list-task-executions --task-arn arn:aws:datasync:us-east-1:123456789012:task/task-0123456789abcdef0

# ============ AWS DMS (DATABASE MIGRATION SERVICE) ============

# Create replication subnet group
aws dms create-replication-subnet-group \
  --replication-subnet-group-identifier my-subnet-group \
  --replication-subnet-group-description "DMS subnets" \
  --subnet-ids subnet-12345678 subnet-87654321

# Create replication instance
aws dms create-replication-instance \
  --replication-instance-identifier my-instance \
  --replication-instance-class dms.t3.medium \
  --allocated-storage 100 \
  --multi-az \
  --publicly-accessible false \
  --replication-subnet-group-identifier my-subnet-group

# Create source endpoint (on-premises Oracle)
aws dms create-endpoint \
  --endpoint-type source \
  --engine-name oracle \
  --server-name oracle.example.com \
  --port 1521 \
  --database-name ORCL \
  --username admin \
  --password MyPassword \
  --endpoint-identifier oracle-source

# Create target endpoint (Aurora PostgreSQL)
aws dms create-endpoint \
  --endpoint-type target \
  --engine-name aurora-postgresql \
  --server-name aurora.123456789012.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --database-name postgres \
  --username postgres \
  --password MyPassword \
  --endpoint-identifier aurora-target

# Test endpoint connection
aws dms test-connection \
  --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:ABCDEFGHIJKLMNOP \
  --endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:oracle-source

# Create migration task
aws dms create-replication-task \
  --replication-task-identifier oracle-to-aurora \
  --source-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:oracle-source \
  --target-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:aurora-target \
  --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:ABCDEFGHIJKLMNOP \
  --migration-type cdc

# Start migration task
aws dms start-replication-task \
  --replication-task-arn arn:aws:dms:us-east-1:123456789012:task:ABCDEFGHIJKLMNOP \
  --start-replication-task-type start-replication

# List tasks
aws dms describe-replication-tasks

# Get task status
aws dms describe-replication-tasks --filters Name=replication-task-arn,Values=arn:aws:dms:us-east-1:123456789012:task:ABCDEFGHIJKLMNOP

# ============ AWS BATCH ============

# Create compute environment
aws batch create-compute-environment \
  --compute-environment-name my-compute-env \
  --type MANAGED \
  --state ENABLED \
  --compute-resources type=EC2,minvCpus=0,maxvCpus=256,desiredvCpus=0,instanceTypes=optimal,subnets=subnet-12345678,securityGroupIds=sg-12345678,instanceRole=arn:aws:iam::123456789012:instance-profile/ecsInstanceProfile

# Create job queue
aws batch create-job-queue \
  --job-queue-name my-queue \
  --state ENABLED \
  --priority 1 \
  --compute-environment-order order=1,computeEnvironment=arn:aws:batch:us-east-1:123456789012:compute-environment/my-compute-env

# Register job definition
aws batch register-job-definition \
  --job-definition-name my-job \
  --type container \
  --container-properties file://job-definition.json

# Submit job
aws batch submit-job \
  --job-name my-job-run \
  --job-queue my-queue \
  --job-definition my-job

# List jobs
aws batch list-jobs --job-queue my-queue

# Describe job
aws batch describe-jobs --jobs job-uuid-1234abcd

# ============ AMAZON ATHENA ============

# Create workgroup
aws athena create-work-group \
  --name my-workgroup \
  --description "Analytics workgroup" \
  --configuration ResultConfigurationUpdates={OutputLocation=s3://my-bucket/athena-results/}

# Run SQL query
aws athena start-query-execution \
  --query-string "SELECT COUNT(*) FROM s3_logs WHERE status_code = 500" \
  --query-execution-context Database=my_database \
  --work-group my-workgroup

# Get query results
aws athena get-query-results \
  --query-execution-id query-uuid-1234abcd

# Get query status
aws athena get-query-execution --query-execution-id query-uuid-1234abcd

# ============ AWS GLUE ============

# Create database in Data Catalog
aws glue create-database \
  --database-input Name=my_database,Description="Sales data"

# Create table
aws glue create-table \
  --database-name my_database \
  --table-input file://table-definition.json

# Create crawler
aws glue create-crawler \
  --name my-crawler \
  --database-name my_database \
  --role arn:aws:iam::123456789012:role/GlueServiceRole \
  --targets S3Targets=[{Path=s3://my-bucket/data/}]

# Start crawler
aws glue start-crawler --name my-crawler

# Get crawler status
aws glue get-crawler --name my-crawler

# List tables in database
aws glue get-tables --database-name my_database

# Create ETL job
aws glue create-job \
  --name my-etl-job \
  --role arn:aws:iam::123456789012:role/GlueServiceRole \
  --command Name=glueetl,ScriptLocation=s3://my-bucket/scripts/job.py

# Run job
aws glue start-job-run --job-name my-etl-job

# ============ AMAZON REDSHIFT ============

# Create Redshift cluster
aws redshift create-cluster \
  --cluster-identifier my-cluster \
  --node-type dc2.large \
  --number-of-nodes 3 \
  --master-username admin \
  --master-user-password MyPassword123

# List clusters
aws redshift describe-clusters

# Get cluster details
aws redshift describe-clusters --cluster-identifier my-cluster

# Resize cluster (add nodes)
aws redshift resize-cluster \
  --cluster-identifier my-cluster \
  --number-of-nodes 5

# Create snapshot
aws redshift create-cluster-snapshot \
  --snapshot-identifier my-snapshot \
  --cluster-identifier my-cluster

# Restore from snapshot
aws redshift restore-from-cluster-snapshot \
  --cluster-identifier restored-cluster \
  --snapshot-identifier my-snapshot

# Enable Enhanced VPC Routing (COPY/UNLOAD through VPC)
aws redshift modify-cluster \
  --cluster-identifier my-cluster \
  --enhanced-vpc-routing

# ============ AWS DIRECTORY SERVICE ============

# Create AWS Managed Microsoft AD
aws ds create-microsoft-ad \
  --name corp.example.com \
  --password MyPassword123 \
  --vpc-settings SubnetIds=subnet-12345678,subnet-87654321 \
  --edition Standard

# Create AD Connector (proxy to on-premises)
aws ds connect-directory \
  --name corp.example.com \
  --password MyPassword123 \
  --size Small \
  --connect-settings VpcSettings={SubnetIds=subnet-12345678,subnet-87654321},CustomerDnsIps=192.168.1.1,CustomerUserName=admin

# List directories
aws ds describe-directories

# Get directory details
aws ds describe-directories --directory-ids d-1234567890abcdef0

# ============ AWS COGNITO ============

# Create user pool
aws cognito-idp create-user-pool \
  --pool-name my-user-pool \
  --policies PasswordPolicy={MinimumLength=8,RequireUppercase=true,RequireLowercase=true,RequireNumbers=true}

# List user pools
aws cognito-idp list-user-pools --max-results 10

# Create user pool client (for app)
aws cognito-idp create-user-pool-client \
  --user-pool-id us-east-1_abcdefg12 \
  --client-name my-app

# Create identity pool
aws cognito-identity create-identity-pool \
  --identity-pool-name my-identity-pool \
  --allow-unauthenticated-identities false \
  --cognito-identity-providers ProviderName=cognito-idp.us-east-1.amazonaws.com/us-east-1_abcdefg12:client-id-123

# Get credentials for identity
aws cognito-identity get-credentials-for-identity \
  --identity-id us-east-1:12345678-1234-1234-1234-123456789012

# ============ AWS IAM IDENTITY CENTER ============

# List instances
aws sso-admin list-instances

# Get instance details
aws sso-admin describe-instance --instance-arn arn:aws:sso:::instance/ssoins-123456789abcdef01

# Create permission set
aws sso-admin create-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-123456789abcdef01 \
  --name ReadOnlyAccess \
  --session-duration PT8H

# List permission sets
aws sso-admin list-permission-sets --instance-arn arn:aws:sso:::instance/ssoins-123456789abcdef01

# Attach policy to permission set
aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-123456789abcdef01 \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-123456789abcdef01/ps-abcd1234 \
  --managed-policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# Create account assignment (user/group + permission set → account)
aws sso-admin create-account-assignment \
  --instance-arn arn:aws:sso:::instance/ssoins-123456789abcdef01 \
  --target-id 123456789012 \
  --target-type AWS_ACCOUNT \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-123456789abcdef01/ps-abcd1234 \
  --principal-type USER \
  --principal-id user-uuid-1234567890

# ============ AWS RESOURCE ACCESS MANAGER ============

# Create resource share
aws ram create-resource-share \
  --name my-share \
  --resource-arns arn:aws:ec2:us-east-1:123456789012:subnet/subnet-12345678 \
  --principals arn:aws:iam::999999999999:root

# List resource shares
aws ram list-resource-shares --resource-owner SELF

# Get resource share details
aws ram get-resource-shares --resource-share-arns arn:aws:ram:us-east-1:123456789012:resource-share/12345678-1234-1234-1234-123456789012

# List resources in share
aws ram list-resources --resource-share-arn arn:aws:ram:us-east-1:123456789012:resource-share/12345678-1234-1234-1234-123456789012

# Accept resource share invitation
aws ram accept-resource-share-invitation \
  --resource-share-invitation-arn arn:aws:ram:us-east-1:999999999999:resource-share-invitation/12345678-1234-1234-1234-123456789012
```

---

## Summary

This comprehensive guide covers the **advanced identity, disaster recovery, and other AWS services** critical for the AWS Solutions Architect Associate exam. Focus on understanding the **WHY** behind each service — why you'd choose one over another, what problems each solves, and the tradeoffs (cost vs. speed, simplicity vs. control).

**Key exam patterns**:
1. **Multi-account governance** → Organizations + Control Tower + SCPs
2. **Single sign-on** → IAM Identity Center or SAML 2.0 federation
3. **Disaster recovery** → Match RTO/RPO to strategy (Backup → Pilot Light → Warm → Active-Active)
4. **Data migration** → DataSync (online), DMS (database), Snowball (offline), MGN (lift-and-shift)
5. **Shared infrastructure** → RAM for subnets/Transit Gateway
6. **Temporary credentials** → STS AssumeRole, GetSessionToken, Cognito
7. **Analytics** → Athena (SQL on S3), Redshift (warehouse), Glue (ETL)
8. **Batch computing** → AWS Batch (100,000s jobs) vs Lambda (short-lived)

Good luck on the exam! Reference the CLI commands as needed for hands-on practice.
