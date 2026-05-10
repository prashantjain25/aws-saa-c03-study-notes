# AWS SAA-C03 Security: KMS, WAF, Shield & GuardDuty

**Expert-led Study Style Notes**  
*Understand WHY, use analogies, contextual CLI commands, exam tips*

---

## Table of Contents

1. [AWS Encryption Overview](#1-aws-encryption-overview)
2. [AWS KMS (Key Management Service)](#2-aws-kms-key-management-service)
3. [CloudHSM](#3-cloudhsm)
4. [SSM Parameter Store](#4-ssm-parameter-store)
5. [AWS Secrets Manager](#5-aws-secrets-manager)
6. [AWS Certificate Manager (ACM)](#6-aws-certificate-manager-acm)
7. [AWS WAF (Web Application Firewall)](#7-aws-waf-web-application-firewall)
8. [AWS Shield](#8-aws-shield)
9. [AWS Firewall Manager](#9-aws-firewall-manager)
10. [Amazon GuardDuty](#10-amazon-guardduty)
11. [Amazon Inspector](#11-amazon-inspector)
12. [Amazon Macie](#12-amazon-macie)
13. [AWS Security Hub](#13-aws-security-hub)
14. [IAM Access Analyzer](#14-iam-access-analyzer)
15. [S3 Encryption Methods (Exam Favorite)](#15-s3-encryption-methods-exam-favorite)
16. [Comparison Tables](#16-comparison-tables)
17. [Points to Remember (Exam Focus)](#17-points-to-remember-exam-focus)
18. [AWS CLI Quick Reference](#18-aws-cli-quick-reference)

---

## 1. AWS Encryption Overview

### Encryption in Flight (TLS/SSL)
- **What it is**: Data is encrypted BEFORE sending, decrypted AFTER receiving
- **Analogy**: Like sending a locked box through the mail — no one can read the contents while in transit
- **Protects from**: MITM (Man-in-the-Middle) attacks
- **Example**: HTTPS = encryption in flight. Your browser encrypts data before sending to the web server
- **AWS Services**: ALB, API Gateway, CloudFront all support TLS by default

### Encryption at Rest (Server-Side)
- **What it is**: Data encrypted when stored, decrypted when read from storage
- **Analogy**: Like storing a document in a locked vault in a building — the vault protects it while sitting there
- **Key Challenge**: The key must be managed and stored somewhere secure
- **Who controls the key?**: Depends on the service:
  - AWS Owned: AWS has the key (limited visibility)
  - AWS Managed: AWS controls but you can view (better audit trail)
  - Customer Managed: YOU control the key (most secure)

### Client-Side Encryption
- **What it is**: Client encrypts data BEFORE sending to AWS; server never sees plaintext
- **Analogy**: You encrypt a document, put it in a sealed envelope, and mail it to AWS. AWS stores the sealed envelope without ever opening it
- **Key Management**: Client keeps the keys entirely — AWS never sees them
- **Use case**: Highly sensitive data where you don't trust AWS infrastructure with raw data
- **Downside**: AWS cannot help if you lose your key

### Shared Responsibility
- **AWS**: Provides encryption mechanisms (services, hardware, SSL/TLS infrastructure)
- **You**: Decide what to encrypt, manage keys (if Customer Managed), implement encryption

---

## 2. AWS KMS (Key Management Service)

### Core Concept: Why KMS Exists
Think of KMS as a secure key vault inside AWS. AWS manages the infrastructure; you control who can USE the keys. This is delegation of trust.

### Fundamental Properties
- **Regional Service**: A KMS key in `us-east-1` cannot encrypt data in `eu-west-1`
  - Analogy: Your safe deposit box is in a specific bank location. You can't use it in a different city branch without arrangements
  - Solution: Multi-region keys (discussed later) for global applications
  
- **Integrated Services**: EBS, S3, RDS, DynamoDB, Lambda, SSM, Secrets Manager, CloudWatch Logs, and more
  - These services check: "Is encryption enabled?" → "Use this KMS key"
  
- **Automatic vs Manual**: Most services auto-integrate with KMS once enabled

### Types of KMS Keys

#### 1. AWS Owned Keys (FREE)
- **Managed by**: AWS entirely
- **Control**: None — you cannot see or manage this key
- **Rotation**: AWS handles automatically
- **Examples**: S3 SSE-S3 (default), default SQS encryption, default DynamoDB encryption
- **Exam Context**: "What's the default S3 encryption?" → AWS Owned Key via SSE-S3
- **Cost**: Free
- **Visibility**: You don't see it in KMS console

#### 2. AWS Managed Keys (FREE)
- **Managed by**: AWS, created in YOUR account, on YOUR behalf
- **Control**: You can VIEW but NOT manage the key material
- **Rotation**: Automatic every 365 days
- **Examples**: `aws/s3`, `aws/rds`, `aws/lambda`, `aws/ebs`
- **Key Policy**: You can view but cannot customize
- **Exam Context**: "RDS is using an AWS-managed key" → automatic rotation, audit trail visible
- **Cost**: Free
- **Use case**: Services that automatically encrypt data and you don't need custom key control

#### 3. Customer Managed Keys (CMK) — $1/month/key
- **Managed by**: YOU
- **Control**: Complete — you create, manage rotation, control access
- **Rotation**: You decide: automatic (365 days) or manual
- **Key Policy**: You define and manage
- **Cross-account sharing**: You explicitly grant in key policy
- **Examples**: Custom encryption keys for sensitive applications
- **Cost**: $1/month per key
- **Exam Context**: "Control encryption key rotation and access" → CMK

### Key Material Sources

#### 1. KMS-Generated (Default)
- AWS generates the key material
- Stored within KMS
- Most secure — never exposed

#### 2. Custom Key Store (CloudHSM)
- Your own CloudHSM hardware
- Your key material never leaves your HSM
- FIPS 140-2 Level 3 certified
- Complex setup, high security bar

#### 3. Imported Key Material
- You generate key material externally (openssl, Hardware)
- Upload to KMS
- KMS never sees plaintext key
- Your responsibility: backup, rotation

### Key Policies (CRITICAL for the Exam)

**Rule**: Even if IAM allows access, the KEY POLICY must explicitly permit it.

**Analogy**: 
- IAM policy = rules about who can access the KMS service
- Key policy = rules about who can use THIS specific key
- Both must say "YES" for access to work

**No Key Policy = No One Can Use the Key** (even root account)

Example denial scenario:
```
IAM policy: "EC2 instance can call KMS Encrypt"
Key Policy: Silent on EC2 instance
Result: EC2 CANNOT encrypt (key policy blocks it)
```

### Symmetric vs Asymmetric Keys

#### Symmetric Keys (AES-256-GCM) — Default
- **Single key**: same key for encrypt AND decrypt
- **Who uses it**: AWS services, applications
- **Visibility**: You NEVER see the raw key material
- **Usage**: `GenerateDataKey` returns plaintext key (for data encryption) + encrypted key (for storage)
- **Performance**: Fast encryption/decryption
- **Cost**: $1/month for CMK
- **Exam**: Assume "KMS key" = symmetric unless stated otherwise

#### Asymmetric Keys (RSA, ECC)
- **Key pair**: public key (share) + private key (secret)
- **Public key**: You can download and use externally
- **Private key**: NEVER leaves KMS — KMS performs crypto operations
- **Use cases**: 
  - Digital signatures (sign outside AWS, verify with public key)
  - Encrypt by external party (they use public key, you use KMS to decrypt with private)
- **Performance**: Slower than symmetric
- **Exam Context**: "Encrypt data OUTSIDE AWS and decrypt inside" → asymmetric key with public key exported

### KMS API Limits (Throttling Alert)
- **Limit**: 5,500 requests/second per CMK
- **Throttling response**: `ThrottlingException`
- **Adjustable**: Contact AWS for limit increase (certain regions/key types only)
- **Common issue**: S3 uploading 1000 files with SSE-KMS → high KMS API calls → throttling
- **Solution**: S3 Bucket Key feature (reduces KMS calls by ~99%)

### KMS Envelope Encryption (CRITICAL — Expect on Exam)

**Problem**: KMS can only encrypt/decrypt up to 4 KB of data directly. What about 1 TB files?

**Solution**: Envelope Encryption

**Process**:
1. **Call `GenerateDataKey`**: KMS returns TWO items:
   - Plaintext data key (256-bit, random)
   - Encrypted data key (encrypted with your KMS key)
2. **Encrypt data locally**: Use plaintext data key to encrypt your 1 TB file locally (AES-256-GCM)
3. **Store together**: 
   - Encrypted file + Encrypted data key (side by side)
   - Plaintext key is discarded
4. **To decrypt**:
   - Call `Decrypt` on encrypted data key → get plaintext key
   - Use plaintext key to decrypt file locally
   - Discard plaintext key

**Why this works**:
- KMS only encrypts small key (not 1 TB file) — fast, no throttling
- Encryption/decryption of large file happens on client → no KMS bottleneck
- Encrypted key travels with encrypted data (safe)

**Exam tips**:
- "Encrypt 1 TB file with KMS" → envelope encryption mandatory
- "Why S3 SSE-KMS throttles?" → Direct KMS calls; solution = S3 Bucket Key

### KMS Key Rotation

#### Automatic Rotation (CMK only)
- **Frequency**: Every 365 days
- **What rotates**: The backing key material (logical key ID stays the same)
- **User impact**: ZERO — applications use same key ID
- **Audit trail**: CloudTrail shows which key version encrypted what data
- **Enable**: `aws kms enable-key-rotation --key-id <key-id>`

#### Manual Rotation
- **When**: For imported keys, or custom rotation policy
- **Process**: 
  1. Create new key
  2. Re-encrypt data with new key
  3. Update applications to use new key
- **Tracking**: Easier to track because key ID changes
- **Use case**: Compliance requirement to rotate every 90 days (faster than 365)

**Imported Key Rotation Challenge**: ONLY manual rotation possible. AWS doesn't have original key material to rotate.

### Multi-Region Keys (AWS Solution)

**Problem**: You encrypt data in `us-east-1`, need to decrypt in `eu-west-1`. Traditional KMS keys are regional.

**Solution**: Multi-Region Keys
- **Single Key ID**: Same key ID in multiple regions
- **Key Material**: Identical across regions
- **Architecture**: Primary key in one region, replicas in others
- **Replication**: Automatic from primary to replicas
- **Independence**: After initial replication, replicas are independent (not sync'd)
- **Use cases**:
  - Global applications (encrypt in US, decrypt in EU)
  - Disaster recovery (if primary region fails, replica is ready)
  - Cross-region replication (S3, RDS)

**Exam tips**:
- "Encrypt in us-east-1, decrypt in eu-west-1" → multi-region key
- "Disaster recovery with encryption" → multi-region key
- "NOT for real-time sync across regions" → keys are independent after replication

### KMS Cross-Account Access

**Scenario**: Account A has KMS key, Account B needs to encrypt with it.

**Solution**:
1. **Key Policy** (Account A): Allow Account B's principal (IAM role/user)
2. **IAM Policy** (Account B): Grant `kms:Encrypt`, `kms:Decrypt` for the key ARN

**Both must allow** — shared responsibility between accounts.

### Exam Scenarios & Solutions

| Scenario | Solution |
|----------|----------|
| Encrypt 1 TB file with KMS | Envelope encryption (GenerateDataKey) |
| KMS throttles during S3 upload | Enable S3 Bucket Key |
| Encrypt data outside AWS | Asymmetric key + download public key |
| Global app, encrypt/decrypt multi-region | Multi-region key |
| FIPS 140-2 Level 3 compliance | CloudHSM (not KMS) |
| Cross-account encryption access | Key policy + IAM policy both allow |

### KMS CLI Commands

```bash
# Create a new Customer Managed Key (CMK)
aws kms create-key \
  --description "My application encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --region us-east-1

# List all keys in your account
aws kms list-keys --region us-east-1

# Describe a specific key (see key metadata, rotation status, key policy)
aws kms describe-key --key-id arn:aws:kms:us-east-1:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab

# Enable automatic key rotation
aws kms enable-key-rotation --key-id arn:aws:kms:us-east-1:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab

# Create an alias for easier key reference
aws kms create-alias --alias-name alias/my-app-key \
  --target-key-id arn:aws:kms:us-east-1:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab

# Encrypt data (up to 4 KB)
aws kms encrypt \
  --key-id alias/my-app-key \
  --plaintext fileb://mysecret.txt \
  --region us-east-1

# Decrypt data
aws kms decrypt \
  --ciphertext-blob fileb://mysecret.encrypted \
  --region us-east-1

# Generate a data key for envelope encryption (plaintext + encrypted)
aws kms generate-data-key \
  --key-id alias/my-app-key \
  --key-spec AES_256 \
  --region us-east-1

# Generate a data key WITHOUT plaintext (for deferred decryption)
aws kms generate-data-key-without-plaintext \
  --key-id alias/my-app-key \
  --key-spec AES_256 \
  --region us-east-1

# Get key rotation status
aws kms get-key-rotation-status --key-id alias/my-app-key

# List key aliases
aws kms list-aliases --region us-east-1

# Update key policy
aws kms put-key-policy \
  --key-id alias/my-app-key \
  --policy-name default \
  --policy file://key-policy.json

# Get current key policy
aws kms get-key-policy --key-id alias/my-app-key --policy-name default

# Schedule key deletion (15-30 day waiting period)
aws kms schedule-key-deletion \
  --key-id alias/my-app-key \
  --pending-window-in-days 30

# Replicate key to another region (create multi-region key)
aws kms replicate-key \
  --key-id arn:aws:kms:us-east-1:111122223333:key/mrk-1234abcd-12ab-34cd-56ef-1234567890ab \
  --replica-region eu-west-1
```

---

## 3. CloudHSM

### What is CloudHSM?
A dedicated hardware security module (HSM) that you can deploy in AWS. Unlike KMS (managed), CloudHSM is YOUR hardware.

### Key Differences from KMS

| Aspect | KMS | CloudHSM |
|--------|-----|----------|
| **Hardware** | AWS managed | Your dedicated hardware |
| **Control** | AWS controls key storage | You control everything |
| **Compliance** | FIPS 140-2 Level 2 | FIPS 140-2 Level 3 (more stringent) |
| **Access by AWS** | AWS can access keys if needed | AWS cannot access keys |
| **Cost** | $1/month per key | $1.69/hour per HSM + data transfer |
| **Complexity** | Simple, managed | Complex, you manage users/keys |
| **Key Backup** | AWS manages backup | You manage backup |

### CloudHSM Architecture
- **Cluster**: Multiple HSMs across AZs for high availability
- **Multi-AZ**: Deploy CloudHSM in multiple AZs automatically
- **VPC-based**: CloudHSM lives in your VPC (not shared AWS infrastructure)
- **Connectivity**: CloudHSM Client software on EC2 instances connects to cluster

### CloudHSM Use Cases
1. **Regulatory requirement**: "Must use dedicated HSM" (banking, government, healthcare)
2. **On-premises HSM integration**: Extend your existing HSM infrastructure to AWS
3. **Custom HSM applications**: Build applications directly on HSM (certificates, key derivation)
4. **Oracle TDE, Redshift encryption**: Database encryption with HSM-managed keys

### CloudHSM Integration with KMS
- **KMS Custom Key Store**: KMS can use CloudHSM as backing storage for keys
- **Benefit**: Get KMS convenience + CloudHSM security
- **Trade-off**: Slightly higher latency, complex setup

### CloudHSM Considerations
- **Not managed by AWS** → you manage user accounts, key material, backups
- **Expensive** → per-hour billing + operational overhead
- **Complex** → requires HSM expertise on your team
- **Exam**: Only recommend CloudHSM when exam states regulatory/compliance requirement

### CloudHSM CLI Commands

```bash
# Create a CloudHSM cluster
aws cloudhsm create-cluster \
  --hsm-type hsm1.medium \
  --subnet-ids subnet-12345678 \
  --subnet-ids subnet-87654321

# Describe a cluster
aws cloudhsm describe-clusters --cluster-id cluster-abcdef12345678

# Create an HSM in the cluster (adds hardware to cluster)
aws cloudhsm create-hsm \
  --cluster-id cluster-abcdef12345678 \
  --availability-zone us-east-1a

# List HSMs in cluster
aws cloudhsm list-hsms --cluster-id cluster-abcdef12345678

# Delete a cluster (after deleting HSMs)
aws cloudhsm delete-cluster --cluster-id cluster-abcdef12345678
```

---

## 4. SSM Parameter Store

### What is SSM Parameter Store?
Secure storage for configuration data and secrets with hierarchical organization.

**Analogy**: Like a labeled filing cabinet inside a locked room. You organize documents by folder path; the room is encrypted.

### Hierarchical Storage
```
/myapp/dev/db-password
/myapp/dev/db-username
/myapp/prod/db-password
/myapp/prod/api-key
/terraform/state-lock-enabled
```

### Parameter Types

#### 1. String
- Plain text value
- No encryption
- Examples: URLs, configuration flags, region names
- Retrievable by anyone with GetParameter permission

#### 2. StringList
- Comma-separated values
- Plain text
- Example: `/myapp/allowed-regions: us-east-1,eu-west-1,ap-southeast-1`

#### 3. SecureString
- Encrypted with KMS key (default: aws/ssm managed key)
- Decryption requires `kms:Decrypt` permission
- Examples: Database passwords, API keys, secrets
- **Exam**: When you see "parameter encrypted" → SecureString type

### Tiers

#### Standard Tier (Free)
- **Cost**: Free
- **Parameter count**: Up to 10,000
- **Max size**: 4 KB per parameter
- **Expiration/TTL**: No built-in TTL
- **Use case**: Cheap, simple configuration storage

#### Advanced Tier ($0.05/parameter/month)
- **Cost**: $0.05 per parameter per month
- **Parameter count**: Up to 100,000
- **Max size**: 8 KB per parameter
- **TTL/Expiry**: Set auto-deletion policy, on-demand notification before expiry
- **Use case**: Secrets that need periodic rotation tracking, large config values

### Versioning
- Each `put-parameter` with the same name creates a new version
- History maintained; can retrieve by version number
- Useful for audit trail and rollback
- **Exam**: "Review parameter history" → versioning allows this

### Access Control
- **IAM policy** required for all operations
- **For SecureString**: Both IAM permission + KMS decrypt permission required
- **Parameter-level policies**: Not supported; use resource tags + IAM conditions

### Parameter Store vs Secrets Manager

| Feature | SSM Parameter Store | Secrets Manager |
|---------|-------------------|-----------------|
| **Auto-rotation** | No (manual Lambda) | Yes (built-in) |
| **Cost** | Free (Standard) / $0.05/mo (Advanced) | $0.40/secret/month + API calls |
| **RDS integration** | Manual setup | Native auto-rotation |
| **Max size** | 4 KB (Standard), 8 KB (Advanced) | 65 KB |
| **TTL feature** | Advanced tier only | Not available |
| **Primary use** | Configuration + secrets | Secrets only |
| **Exam context** | Cost-effective storage for many items | Auto-rotating RDS/DB passwords |

### Parameter Store Policies (Advanced Tier)
- **TTL policy**: Force regular rotation (expiry date set)
- **Notification**: EventBridge notified before expiry
- **Auto-update**: Can trigger Lambda to update parameter before expiry
- **Example**: Keep DB password rotated every 30 days without manual intervention

### SSM Parameter Store CLI Commands

```bash
# Put a simple string parameter
aws ssm put-parameter \
  --name "/myapp/dev/app-config" \
  --value "log-level=debug" \
  --type String

# Put a SecureString parameter (encrypted with default aws/ssm key)
aws ssm put-parameter \
  --name "/myapp/dev/db-password" \
  --value "SuperSecretPassword123!" \
  --type SecureString \
  --key-id alias/ssm-key

# Put with Advanced tier
aws ssm put-parameter \
  --name "/myapp/prod/api-key" \
  --value "sk-abc123xyz..." \
  --type SecureString \
  --tier Advanced \
  --key-id arn:aws:kms:us-east-1:111122223333:key/12345678

# Get a parameter (retrieves latest version)
aws ssm get-parameter --name "/myapp/dev/db-password" --with-decryption

# Get parameter value only (plain text)
aws ssm get-parameter --name "/myapp/dev/db-password" \
  --with-decryption \
  --query 'Parameter.Value' \
  --output text

# Get parameter with specific version
aws ssm get-parameter --name "/myapp/dev/db-password" \
  --version 2 \
  --with-decryption

# Get multiple parameters at once
aws ssm get-parameters \
  --names "/myapp/dev/db-password" "/myapp/dev/db-username" \
  --with-decryption

# Get parameters by path (all under a path)
aws ssm get-parameters-by-path \
  --path "/myapp/dev" \
  --recursive \
  --with-decryption

# Get parameter metadata (without value)
aws ssm describe-parameters --filters "Key=Name,Values=/myapp/dev/db-password"

# List all parameter versions
aws ssm list-tags-for-resource \
  --resource-type "Parameter" \
  --resource-id "/myapp/dev/db-password"

# Put parameter with expiration (Advanced tier)
aws ssm put-parameter \
  --name "/myapp/prod/rotating-key" \
  --value "key-value-xyz" \
  --type SecureString \
  --tier Advanced \
  --policies '[{"Type":"Expiration","Version":"1.0","Value":"2026-06-06T23:59:59.000Z"}]'

# Delete a parameter
aws ssm delete-parameter --name "/myapp/dev/old-config"

# Tag a parameter
aws ssm add-tags-to-resource \
  --resource-type "Parameter" \
  --resource-id "/myapp/dev/db-password" \
  --tags "Key=Environment,Value=development"
```

---

## 5. AWS Secrets Manager

### What is Secrets Manager?
Purpose-built for storing and automatically rotating secrets (passwords, API keys, credentials).

**Analogy**: Like a password manager service for AWS. It not only stores secrets but also rotates them automatically and logs access.

### Core Features

#### Automatic Rotation
- **Schedule**: Every X days (rotate-secret)
- **Native integration**: RDS (MySQL, PostgreSQL, Oracle, SQL Server), Redshift, DocumentDB
  - AWS automatically updates the database password using Lambda
- **Custom rotation**: For other services, you write a Lambda function
- **Process**:
  1. Lambda creates new secret
  2. Tests new secret works
  3. Finishes rotation
  4. AWSCURRENT label moves to new secret

#### Versioning
- **AWSCURRENT**: Active secret version (applications use this)
- **AWSPREVIOUS**: Previous version (grace period for applications to update)
- **AWSPENDING**: Pending rotation (during rotation process)
- **Why**: Allows rotation without breaking applications (brief overlap period)

#### Force Rotation
- Instantly rotate secret without waiting for schedule
- Useful for emergency key rotation if compromised
- CLI: `rotate-secret`

### Pricing
- **Per-secret cost**: $0.40/secret/month
- **API call cost**: $0.05 per 10,000 API calls
- **Example**: 10 secrets, 100,000 API calls/month = $4 + $0.50 = $4.50/month

### Integration with RDS
- **Automatic password generation**: Secrets Manager generates strong password
- **Automatic updates**: When rotation triggered, Secrets Manager:
  1. Changes DB master password
  2. Verifies connection works
  3. Stores new secret version
- **Zero downtime**: Applications with correct IAM role can rotate transparently
- **Exam**: "Auto-rotate RDS password" → Secrets Manager (not SSM Parameter Store)

### Cross-Account Access
- **Resource policy** on secret allows another AWS account to retrieve it
- **Use case**: Shared credential across accounts (avoid key duplication)

### Multi-Region Secrets
- Replicate secret to multiple regions with same SecretId
- **Use case**: Global applications, disaster recovery
- Primary region is source of truth; replicas are read-only copies

### Exam Scenarios

| Scenario | Tool |
|----------|------|
| Auto-rotate RDS password | Secrets Manager (native integration) |
| Store 50 config values cheaply | SSM Parameter Store (Standard tier, free) |
| "How many times was this secret accessed?" | Secrets Manager (cloudtrail logs access) |
| Rotate every 30 days automatically | Secrets Manager |

### Secrets Manager CLI Commands

```bash
# Create a secret with auto-rotation
aws secretsmanager create-secret \
  --name "prod/rds/master-password" \
  --description "RDS master database password" \
  --secret-string "MySecurePassword123!"

# Create secret with automatic rotation for RDS
aws secretsmanager create-secret \
  --name "prod/rds/db-credentials" \
  --secret-string '{"username":"admin","password":"InitialPassword123!"}' \
  --add-replica-regions RegionCode=eu-west-1

# Get current secret value
aws secretsmanager get-secret-value --secret-id "prod/rds/master-password"

# Get specific version of secret
aws secretsmanager get-secret-value \
  --secret-id "prod/rds/master-password" \
  --version-id "12345678-1234-1234-1234-123456789012"

# Get secret by version stage
aws secretsmanager get-secret-value \
  --secret-id "prod/rds/master-password" \
  --version-stage AWSCURRENT

# Enable rotation
aws secretsmanager rotate-secret \
  --secret-id "prod/rds/master-password" \
  --rotation-lambda-arn "arn:aws:lambda:us-east-1:111122223333:function:rotate-secret" \
  --rotation-rules "AutomaticallyAfterDays=30"

# Force immediate rotation
aws secretsmanager rotate-secret \
  --secret-id "prod/rds/master-password"

# Describe secret metadata
aws secretsmanager describe-secret --secret-id "prod/rds/master-password"

# List all secrets
aws secretsmanager list-secrets --filters "Key=name,Values=prod"

# Update secret value
aws secretsmanager update-secret \
  --secret-id "prod/rds/master-password" \
  --secret-string "NewPassword456!"

# Put secret resource policy (cross-account access)
aws secretsmanager put-resource-policy \
  --secret-id "prod/rds/master-password" \
  --resource-policy file://secret-policy.json

# Tag a secret
aws secretsmanager tag-resource \
  --secret-id "prod/rds/master-password" \
  --tags "Key=Environment,Value=production"

# Delete secret (with recovery window)
aws secretsmanager delete-secret \
  --secret-id "prod/rds/master-password" \
  --recovery-window-in-days 30
```

---

## 6. AWS Certificate Manager (ACM)

### What is ACM?
Provision, manage, and renew SSL/TLS certificates for HTTPS.

### Public Certificates (FREE)
- **Cost**: $0 — included in AWS
- **Validity**: Auto-renewed before expiration
- **Validation**: DNS or Email validation
- **Integration**: ALB, NLB, CloudFront, API Gateway, Elastic Beanstalk, AppSync
- **Key extraction**: Private key CANNOT be exported (AWS keeps it secure)

### Private Certificates ($0.75/month)
- ACM Private CA (Certificate Authority)
- Useful for internal applications, mTLS, internal PKI
- More expensive but private to your organization

### ACM Features

#### Auto-Renewal
- Public certificates auto-renew 60 days before expiration
- No manual intervention needed
- CloudWatch Logs notify of renewal status

#### Validation Methods
- **DNS validation** (preferred, automated):
  - ACM creates DNS record in Route53
  - Validates by checking DNS
  - Transparent, no email required
  - Exam: "Auto-renew certificate" → DNS validation
  
- **Email validation**:
  - ACM sends email to domain owner
  - Manual click-through in email
  - More hassle, not recommended

#### CloudFront Special Requirement
- **Certificate must be in `us-east-1`** regardless of CloudFront location
- Reason: CloudFront is a global service; AWS standardized on us-east-1 for certificates
- Exam: "HTTPS on CloudFront" → ensure certificate is in us-east-1

### ACM vs Third-Party Certificates

| Aspect | ACM Public | Third-Party |
|--------|-----------|-------------|
| **Cost** | Free | $20-200/year |
| **Auto-renewal** | Yes | You manage |
| **Export private key** | No | Usually yes |
| **Integration** | Deep AWS integration | Limited |
| **Exam context** | Use ACM for ALB, API Gateway, CloudFront | Use third-party if you need private key export |

### ACM Limitations
- **Cannot export private key** from ACM certificates
- **If you need to use cert on EC2 directly**: Use third-party certificate or import to ACM
- **Regional**: Must request certificate in same region as service (except CloudFront)

### ACM CLI Commands

```bash
# Request a public certificate
aws acm request-certificate \
  --domain-name example.com \
  --subject-alternative-names www.example.com \
  --validation-method DNS \
  --region us-east-1

# Request certificate for CloudFront (always us-east-1)
aws acm request-certificate \
  --domain-name example.com \
  --subject-alternative-names www.example.com \
  --validation-method DNS \
  --region us-east-1

# List all certificates in region
aws acm list-certificates --region us-east-1

# Describe certificate (see validation status, renewal status)
aws acm describe-certificate \
  --certificate-arn "arn:aws:acm:us-east-1:111122223333:certificate/12345678-1234-1234-1234-123456789012"

# Delete a certificate
aws acm delete-certificate \
  --certificate-arn "arn:aws:acm:us-east-1:111122223333:certificate/12345678-1234-1234-1234-123456789012"

# Import external certificate to ACM
aws acm import-certificate \
  --certificate fileb://Certificate.pem \
  --certificate-chain fileb://CertificateChain.pem \
  --private-key fileb://PrivateKey.pem \
  --region us-east-1

# Add tags to certificate
aws acm add-tags-to-certificate \
  --certificate-arn "arn:aws:acm:us-east-1:111122223333:certificate/12345678-1234-1234-1234-123456789012" \
  --tags "Key=Environment,Value=production"

# Export certificate (not possible for ACM-issued certs, only imported)
# Note: use describe-certificate to check domain validation
```

---

## 7. AWS WAF (Web Application Firewall)

### What is WAF?
Layer 7 (HTTP/HTTPS) protection against web-based attacks: SQL injection, XSS, bot traffic.

**Analogy**: Like a security guard at a web server's front door. It reads HTTP requests and decides "let through" or "block" based on rules.

### Key Characteristic: Layer 7 (Application Layer)
- **Understands HTTP/HTTPS**: Can inspect URLs, headers, query parameters, request body
- **NOT for Layer 4**: Cannot protect NLB directly (NLB is Layer 4 — transport layer)
- **Must be in front**: ALB, API Gateway, CloudFront, AppSync, Cognito User Pool

### Where to Deploy WAF

| Service | WAF Support |
|---------|-------------|
| ALB | Yes (ALB is Layer 7) |
| NLB | No (Layer 4) |
| CloudFront | Yes (L7 awareness) |
| API Gateway | Yes |
| AppSync | Yes |
| Cognito User Pool | Yes |
| EC2 directly | No (must use ALB in front) |

### Web ACL (Access Control List)

#### Structure
- **Rules**: Evaluated in order; first match wins
- **Actions**: ALLOW, BLOCK, COUNT (no action, just logging)
- **Managed Rule Groups**: AWS-provided rule sets (free and paid)
- **Custom rules**: You define matching conditions

#### Rule Types
1. **IP set match**: Block specific IP addresses or CIDR blocks
2. **String match**: Match patterns in headers, body, URI (e.g., `admin` in URI)
3. **Regex match**: Regular expression patterns
4. **Size constraint**: Block if request body > X bytes
5. **Rate-based**: Block IP if >X requests in 5 minutes (DDoS mitigation)
6. **Geo-match**: Block/allow by country

#### Managed Rule Groups (AWS-Provided)

**AWSManagedRulesCommonRuleSet** (most popular):
- SQL injection detection
- XSS (Cross-Site Scripting) detection
- Local file inclusion (LFI) detection
- Remote code execution (RCE) detection
- Core rules from OWASP

**AWSManagedRulesBotControlRuleSet**:
- Bot detection and filtering
- Blocks known malicious bots, crawlers
- Monitors for suspicious bot activity

**Custom managed rule groups**: AWS Marketplace or your own

### Rate-Based Rules (DDoS Mitigation)
- **Purpose**: Block IPs exceeding request threshold in 5-minute window
- **Example**: Block IPs with >2,000 requests/5 minutes
- **Useful for**: Mitigating application-level DDoS (Layer 7 DDoS)
- **Difference from Shield**: WAF is application-aware; Shield is network-aware

### WAF Logging
- **Destination**: CloudWatch Logs, S3, Kinesis Firehose
- **Content**: Full HTTP request details (headers, body, IP, action taken)
- **Use case**: Audit why requests were blocked, investigate false positives

### CAPTCHA and Challenge Actions
- **CAPTCHA**: Present visual puzzle; if failed, block request
- **Challenge**: Simulate token request; browsers handle automatically
- **Use case**: Differentiate humans from bots
- **Cost**: Charged per captcha solved

### WAF Security Automations (AWS Solutions)
- Pre-built solutions for:
  - IP reputation lists
  - Bot detection and filtering
  - HTTP flood protection
  - SQL injection/XSS patterns
- Available via AWS Solutions, AWS Marketplace

### Exam Scenarios

| Scenario | Solution |
|----------|----------|
| "Protect against SQL injection" | WAF with SQLi rule |
| "Block all traffic from Country X" | WAF geo-match rule |
| "WAF on EC2 directly" | NOT possible; use ALB in front |
| "Rate-limit attacker IPs" | WAF rate-based rule |
| "Log all blocked requests" | WAF logging to CloudWatch Logs |

### WAF CLI Commands

```bash
# Create IP set (list of IPs to block/allow)
aws wafv2 create-ip-set \
  --name "blocked-ips" \
  --scope REGIONAL \
  --ip-address-version IPV4 \
  --addresses "192.0.2.0/24" "198.51.100.0/24" \
  --region us-east-1

# Create a Web ACL
aws wafv2 create-web-acl \
  --name "my-web-acl" \
  --scope REGIONAL \
  --default-action Block={} \
  --rules file://waf-rules.json \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=myWebACL \
  --region us-east-1

# Get Web ACL details
aws wafv2 get-web-acl \
  --name "my-web-acl" \
  --scope REGIONAL \
  --id "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111" \
  --region us-east-1

# List all Web ACLs in region
aws wafv2 list-web-acls --scope REGIONAL --region us-east-1

# Associate Web ACL with ALB
aws wafv2 associate-web-acl \
  --web-acl-arn "arn:aws:wafv2:us-east-1:111122223333:regional/webacl/my-web-acl/a1b2c3d4-5678-90ab-cdef-EXAMPLE11111" \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:111122223333:loadbalancer/app/my-alb/1234567890123456" \
  --region us-east-1

# Disassociate Web ACL from ALB
aws wafv2 disassociate-web-acl \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:111122223333:loadbalancer/app/my-alb/1234567890123456" \
  --region us-east-1

# Create rate-based rule (block IPs exceeding threshold)
aws wafv2 create-rate-based-statement \
  --action Block={} \
  --rate-limit 2000 \
  --aggregate-key-type IP \
  --region us-east-1

# Update Web ACL (add new rule)
aws wafv2 update-web-acl \
  --name "my-web-acl" \
  --scope REGIONAL \
  --id "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111" \
  --default-action Allow={} \
  --rules file://updated-rules.json \
  --lock-token "abc123def456..." \
  --region us-east-1

# List resources associated with Web ACL
aws wafv2 list-resources-for-web-acl \
  --web-acl-arn "arn:aws:wafv2:us-east-1:111122223333:regional/webacl/my-web-acl/a1b2c3d4-5678-90ab-cdef-EXAMPLE11111" \
  --region us-east-1

# Delete Web ACL (must disassociate from resources first)
aws wafv2 delete-web-acl \
  --name "my-web-acl" \
  --scope REGIONAL \
  --id "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111" \
  --lock-token "abc123def456..." \
  --region us-east-1
```

---

## 8. AWS Shield

### What is Shield?
DDoS (Distributed Denial of Service) protection service.

**Analogy**: Like insurance against flood — Shield Standard is basic coverage you always have; Shield Advanced is comprehensive coverage with expert support.

### Shield Standard (FREE — Always Active)

**Included for all AWS customers automatically:**
- **Protection against**: Common L3/L4 DDoS attacks
  - SYN floods (Layer 4)
  - UDP floods (Layer 4)
  - DNS query floods (Layer 3/4)
  - Reflection attacks (spoofed requests)
- **How it works**: AWS infrastructure absorbs attacks, customers never see them
- **Cost**: Zero — always enabled
- **Setup required**: None
- **Limitations**: Not suitable for sophisticated, sustained attacks

**Example attack Shield Standard stops**:
- "Attacker sends 1 million SYN packets to EC2 instance"
- AWS infrastructure detects and filters before reaching customer

### Shield Advanced ($3,000/month/organization)

**Enhanced protection for critical applications:**
- **Protects**: EC2, ELB (ALB/NLB/CLB), CloudFront, Global Accelerator, Route53
- **DDoS Response Team (DRT)**: 24/7/365 expert support
  - Helps mitigate attacks in real-time
  - Provides guidance, not automatic
- **DDoS Cost Protection**: AWS provides credits for charges caused by DDoS attack scaling
  - **Example**: Attack causes auto-scaling surge → $50K bill → AWS refunds it
  - **Condition**: Must have Shield Advanced and attack meets criteria
- **Advanced metrics**: Real-time visibility into attack patterns
- **Global threat dashboard**: See attacks across all protected resources
- **WAF bundled**: No additional cost for WAF on Shield Advanced resources

**Example benefit**:
- Business running on ALB in Advanced Shield
- Large DDoS attack overwhelms capacity
- Auto-scaling triggers, 10x servers launch
- Normal bill would be $100K
- With Shield Advanced cost protection: AWS covers the $100K (minus Shield Advanced monthly fee)

### Shield Standard vs Advanced Comparison

| Feature | Standard | Advanced |
|---------|----------|----------|
| **Cost** | Free | $3,000/month |
| **Coverage** | L3/L4 (SYN/UDP floods) | L3/L4/L7 (incl. application DDoS) |
| **DRT Support** | No | 24/7 |
| **Cost Protection** | No | Yes (credits for scaling) |
| **WAF integration** | Extra cost | Included |
| **Best for** | Most AWS workloads | Mission-critical, publicly facing apps |

### Exam Scenarios

| Scenario | Solution |
|----------|----------|
| "Free DDoS protection" | Shield Standard (automatic) |
| "Business loses $100K due to DDoS scaling costs" | Shield Advanced (cost protection) |
| "WAF + DDoS = one service" | Shield Advanced (bundles WAF) |
| "EC2 under DDoS attack" | Shield Standard helps; Advanced for priority support |

### Shield CLI Commands

```bash
# Enable Shield Advanced on organization (can only be called from management account)
aws shield create-subscription --region us-east-1

# Create DDoS protection for specific resource
aws shield create-protection \
  --name "my-alb-protection" \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:111122223333:loadbalancer/app/my-alb/1234567890123456" \
  --region us-east-1

# List all protected resources
aws shield list-protections --region us-east-1

# Describe a specific protection
aws shield describe-protection \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:111122223333:loadbalancer/app/my-alb/1234567890123456" \
  --region us-east-1

# Describe attacks on protected resources
aws shield describe-attacks \
  --resource-arns "arn:aws:elasticloadbalancing:us-east-1:111122223333:loadbalancer/app/my-alb/1234567890123456" \
  --region us-east-1

# Delete a protection
aws shield delete-protection \
  --protection-id "12345678-1234-1234-1234-123456789012" \
  --region us-east-1
```

---

## 9. AWS Firewall Manager

### What is Firewall Manager?
Central management of security rules across an AWS Organization.

**Analogy**: Like a central IT security team that sets uniform firewall policies for all branch offices instead of each office managing their own.

### Prerequisite
- **AWS Organizations** must be enabled
- **Designated admin account** must be set in Firewall Manager
- **Member accounts**: Policies push down automatically

### Services Managed

1. **WAF rules**: Centrally manage Web ACLs across accounts
2. **Shield Advanced**: Enable/manage across organization
3. **VPC Security Groups**: Audit and enforce SG rules across accounts
4. **Network Firewall**: Manage firewall policies
5. **Route53 Resolver DNS Firewall**: DNS filtering policies

### Policy Workflow
1. Admin creates policy in Firewall Manager (e.g., "all ALBs must have WAF rule X")
2. Policy applies to all accounts in OU (organizational unit)
3. New accounts joining OU automatically get policy
4. Non-compliant resources: remediate automatically or manually

### Use Cases

**Scenario 1**: Multi-account organization (50 AWS accounts)
- **Problem**: Each account's teams manage WAF independently → inconsistent
- **Solution**: Firewall Manager policy enforces same WAF rules across all 50 accounts
- **Benefit**: Unified security posture, easier compliance audits

**Scenario 2**: SG compliance
- **Problem**: Teams create overly-permissive security groups (e.g., 0.0.0.0/0)
- **Solution**: Firewall Manager SG policy audits and remediates
- **Benefit**: Prevents misconfigurations

### Exam Context
- "Enforce WAF rules organization-wide" → Firewall Manager
- "Firewall Manager requires X" → AWS Organizations
- "Security Group policies across 50 accounts" → Firewall Manager

### Firewall Manager CLI Commands

```bash
# Enable Firewall Manager (admin account only)
aws fms enable-organization-admin-account \
  --admin-account "111122223333" \
  --region us-east-1

# Create WAF policy via Firewall Manager
aws fms put-policy \
  --policy file://firewall-policy.json \
  --region us-east-1

# List all Firewall Manager policies
aws fms list-policies --region us-east-1

# Get policy details
aws fms get-policy --policy-id "12345678-1234-1234-1234-123456789012" \
  --region us-east-1

# List non-compliant resources for policy
aws fms list-non-compliant-resources --policy-id "12345678-1234-1234-1234-123456789012" \
  --region us-east-1

# Delete policy
aws fms delete-policy \
  --policy-id "12345678-1234-1234-1234-123456789012" \
  --delete-all-policy-resources \
  --region us-east-1
```

---

## 10. Amazon GuardDuty

### What is GuardDuty?
Intelligent threat detection using machine learning, anomaly detection, and threat intelligence feeds.

**Analogy**: Like a security monitoring center that watches your AWS infrastructure 24/7, using AI to spot suspicious behavior and known threats.

### Key Features

#### No Setup Required
- **Enable once, forget about it**: GuardDuty automatically analyzes existing AWS data sources
- **No agent installation**: Unlike some security tools, GuardDuty doesn't require CloudWatch agent

#### Data Sources (GuardDuty analyzes these automatically)

1. **CloudTrail Management Events**: Unusual API calls
   - Example: "DeleteSecurityGroup called at 3 AM from unknown IP"

2. **CloudTrail S3 Data Events**: S3 object access patterns
   - Example: "EC2 instance downloading 1TB of S3 data (exfiltration)"

3. **VPC Flow Logs**: Network traffic analysis
   - Example: "EC2 talking to known malicious IP (C&C server)"

4. **DNS Logs**: DNS query analysis
   - Example: "Instance querying domain known for malware distribution"

5. **EKS Audit Logs**: Kubernetes API activity
   - Example: "Unusual role elevation in Kubernetes cluster"

6. **RDS Login Events**: Database authentication attempts
   - Example: "Brute-force attack on RDS master user"

7. **Lambda Network Activity**: Function to function communication
   - Example: "Lambda making requests to known malicious IP"

8. **S3 Malware Protection**: Scan uploaded objects
   - Example: "S3 file uploaded contains known malware signature"

### Finding Types (Examples)

**High Severity**:
- `CryptoCurrency:EC2/BitcoinTool.B` = Crypto mining detected (compromised EC2)
- `UnauthorizedAccess:IAM/AnomalousAPICall` = Unusual IAM API pattern
- `Trojan:EC2/DNSRequest.C` = DNS request to malicious domain

**Medium Severity**:
- `UnauthorizedAccess:EC2/SSHBruteforce` = SSH brute-force detected
- `DefenseEvasion:IAM/AnomalousAdminActivity` = Admin user behaving unusually

**Low Severity**:
- `Recon:IAM/TorIPCaller` = API call from Tor exit node

### Findings Workflow

1. **GuardDuty detects finding**: "CryptoCurrency:EC2/BitcoinTool.B on i-1234567890abcdef0"
2. **Finding sent to EventBridge**: Can trigger automated response
3. **Lambda function triggered**: Stops EC2, isolates security group, etc.
4. **Suppression rule**: Mark as false positive to ignore future similar findings

### Auto-Remediation with EventBridge + Lambda

**Example scenario**: EC2 running crypto miner detected
```
1. GuardDuty finding → EventBridge rule matches
2. Lambda function triggered
3. Lambda stops EC2 instance
4. Lambda removes it from load balancer
5. Lambda snapshots volume for forensics
```

### Multi-Account Setup
- **Master account**: Manages findings from all member accounts
- **Member accounts**: GuardDuty runs locally, findings visible to master
- **Centralized view**: Master account dashboard shows all findings
- **Member invite**: Non-disruptive, members can accept or decline

### Pricing
- **Free trial**: 30 days of full features
- **After trial**: Based on data volume analyzed (CloudTrail events, VPC Flow logs, DNS logs, etc.)
- **Exam**: "GuardDuty cost after 30 days" → per-GB analyzed

### GuardDuty Limitations (Exam)
- **Does NOT prevent attacks**: Only detects and reports
- **Automatic remediation**: Requires EventBridge + Lambda (GuardDuty alone = detection only)
- **Not a WAF/Shield replacement**: GuardDuty is behavioral detection; WAF is attack prevention

### Exam Scenarios

| Scenario | Solution |
|----------|----------|
| "Detect compromised EC2 mining cryptocurrency" | GuardDuty (CryptoCurrency finding) |
| "Automatically isolate malicious EC2" | GuardDuty → EventBridge → Lambda |
| "Prevent SQL injection attacks" | WAF (not GuardDuty) |
| "Multi-account threat detection" | GuardDuty with master account setup |

### GuardDuty CLI Commands

```bash
# Enable GuardDuty in a region
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --region us-east-1

# List detectors (show if GuardDuty is enabled)
aws guardduty list-detectors --region us-east-1

# Get detector details
aws guardduty get-detector \
  --detector-id "12345678901234567890123456789012" \
  --region us-east-1

# List findings (with filters)
aws guardduty list-findings \
  --detector-id "12345678901234567890123456789012" \
  --finding-criteria '{"Criterion":{"severity":{"Gte":4}}}' \
  --region us-east-1

# Get finding details
aws guardduty get-findings \
  --detector-id "12345678901234567890123456789012" \
  --finding-ids "finding-id-1" "finding-id-2" \
  --region us-east-1

# Create filter (suppress false positives)
aws guardduty create-filter \
  --detector-id "12345678901234567890123456789012" \
  --name "suppress-known-good-ips" \
  --finding-criteria file://filter-criteria.json \
  --action NOOP \
  --region us-east-1

# Update detector (change finding frequency)
aws guardduty update-detector \
  --detector-id "12345678901234567890123456789012" \
  --finding-publishing-frequency SIXTY_MINUTES \
  --region us-east-1

# Create threat intelligence set (known malicious IPs)
aws guardduty create-threat-intelligence-set \
  --detector-id "12345678901234567890123456789012" \
  --name "known-malicious-ips" \
  --format TXT \
  --location "s3://my-bucket/malicious-ips.txt" \
  --activate \
  --region us-east-1

# Disable GuardDuty
aws guardduty delete-detector \
  --detector-id "12345678901234567890123456789012" \
  --region us-east-1

# Create master account (aggregate findings from member accounts)
aws guardduty create-members \
  --detector-id "12345678901234567890123456789012" \
  --account-details '[{"AccountId":"111122223333","Email":"member@example.com"}]' \
  --region us-east-1

# Accept invitation (member account side)
aws guardduty accept-invitation \
  --detector-id "member-detector-id" \
  --invitation-id "invitation-id" \
  --master-id "master-account-id" \
  --region us-east-1
```

---

## 11. Amazon Inspector

### What is Inspector?
Automated vulnerability assessment for:
- **EC2 instances**: OS vulnerabilities, network reachability
- **Container images**: Software vulnerabilities in ECR
- **Lambda functions**: Code vulnerabilities and dependencies

**Analogy**: Like a security scanner that constantly checks your infrastructure for known vulnerabilities (CVEs).

### What Inspector Assesses

#### EC2 Instances
- **OS package vulnerabilities**: Detects known CVEs in installed packages
- **Requires**: SSM agent running on EC2 (AgentlessScanning now available without agent)
- **Example**: "OpenSSL 1.0.1 has CVE-2014-0160 (Heartbleed)"
- **Network reachability**: Checks if EC2 is exposed to unexpected ports

#### Container Images (ECR)
- **Layer scanning**: Each Docker layer scanned for CVEs
- **Continuous scanning**: Existing images re-scanned when new vulnerability data available
- **Example**: "Alpine Linux in your image has libc CVE"

#### Lambda Functions
- **Code scanning**: Detects vulnerabilities in application code
- **Dependency scanning**: Checks Python/Node.js packages for known CVEs
- **Example**: "requests library version 2.6.0 has CVE-2018-18074"

### Inspector Classic vs Inspector v2

| Aspect | Classic (Deprecated) | v2 (Current) |
|--------|-------------------|-------------|
| **EC2 Scope** | Instances with agent | All EC2 + Agentless scanning |
| **Container Scan** | Not available | Yes (ECR) |
| **Lambda Scan** | Not available | Yes |
| **Continuous** | Scheduled | Continuous |
| **Exam**: Use v2 (current standard) |

### Risk Score
- Calculated from:
  1. **CVSS score**: Industry-standard vulnerability severity (0-10)
  2. **Environmental context**: Is port open to internet? Is this critical app?
- **Result**: Normalized risk score helping prioritize patching

### Integration
- **AWS Security Hub**: Findings appear in central dashboard
- **EventBridge**: Trigger Lambda to auto-patch or notify
- **CloudWatch**: Metrics and dashboards

### Exam Scenarios

| Scenario | Tool |
|----------|------|
| "Scan EC2 for OS CVEs" | Inspector (requires SSM agent) |
| "Scan Docker images in ECR" | Inspector v2 |
| "Lambda code has vulnerable package" | Inspector v2 |
| "Automate EC2 patching on CVE detection" | Inspector → EventBridge → Systems Manager Patch Manager |

### Inspector CLI Commands

```bash
# Enable Inspector in region
aws inspector2 enable --resource-types EC2 ECR LAMBDA --region us-east-1

# List EC2 findings
aws inspector2 list-findings \
  --filter-criteria '{"resourceType":["EC2"]}' \
  --region us-east-1

# Get finding details
aws inspector2 get-findings \
  --finding-arns "arn:aws:inspector2:us-east-1:111122223333:finding/12345678-1234-1234-1234-123456789012" \
  --region us-east-1

# List findings for specific EC2 instance
aws inspector2 list-findings \
  --filter-criteria '{"resourceId":"i-1234567890abcdef0"}' \
  --region us-east-1

# Disable Inspector
aws inspector2 disable --resource-types EC2 ECR LAMBDA --region us-east-1
```

---

## 12. Amazon Macie

### What is Macie?
ML-powered S3 data security and privacy service that discovers and protects sensitive data.

**Analogy**: Like a data privacy auditor that scans all your S3 buckets and finds where PII, credit cards, API keys are stored.

### What Macie Detects

#### Sensitive Data Types
- **Personally Identifiable Information (PII)**:
  - Credit card numbers
  - Social Security numbers
  - Email addresses
  - Phone numbers
  
- **Protected Health Information (PHI)**:
  - Medical records
  - Patient IDs
  
- **Financial Information**:
  - Bank account numbers
  - Investment account numbers
  
- **Credentials**:
  - API keys
  - Passwords
  - Private encryption keys

### How Macie Works
1. **Discovery**: Analyzes S3 object content and metadata
2. **Classification**: Identifies data types using ML models
3. **Findings**: Reports what sensitive data exists where
4. **Ongoing**: Monitors new objects for sensitive data

### Findings Types

#### 1. Sensitive Data Finding
- **What**: "Found N instances of data-type in bucket/key"
- **Example**: "Found 45 credit card numbers in s3://logs/payment-data/"
- **Action**: Investigate access, consider encryption, restrict permissions

#### 2. Policy Finding
- **What**: "S3 bucket misconfiguration detected"
- **Example**: "Bucket is publicly accessible"
- **Action**: Remediate bucket policy

### Multi-Account Setup
- **Managed via AWS Organizations**
- **Admin account** aggregates findings from all member accounts
- **Centralized view**: Single dashboard for entire organization's S3 data sensitivity

### Pricing
- **Data classification**: Charged per GB of data analyzed
- **Object evaluation**: Per object scanned
- **Example**: Analyze 1 TB S3 bucket → ~$5 cost

### Exam Scenarios

| Scenario | Solution |
|----------|----------|
| "Find PII in all S3 buckets" | Macie |
| "Compliance audit: where is sensitive data?" | Macie (reports data locations) |
| "Prevent data exfiltration" | Macie (detection) + IAM policies (prevention) |
| "GDPR compliance: find personal data" | Macie |

### Macie CLI Commands

```bash
# Enable Macie in region
aws macie2 enable-macie --region us-east-1

# Get Macie status
aws macie2 get-macie-session --region us-east-1

# List findings
aws macie2 list-findings --region us-east-1

# Get finding details
aws macie2 get-findings \
  --finding-ids "finding-id-1" "finding-id-2" \
  --region us-east-1

# Create findings filter (custom search)
aws macie2 create-findings-filter \
  --name "public-buckets-with-pii" \
  --finding-criteria file://filter-criteria.json \
  --action ARCHIVE \
  --region us-east-1

# List sensitive data discoveries
aws macie2 list-sensitive-data-occurrences --region us-east-1

# Disable Macie
aws macie2 disable-macie --region us-east-1
```

---

## 13. AWS Security Hub

### What is Security Hub?
Central security dashboard aggregating findings from multiple AWS security services and third-party tools.

**Analogy**: Like a security operations center (SOC) where all security alerts appear on one screen, instead of checking 10 different dashboards.

### Data Sources (Aggregated)

**AWS Native**:
- GuardDuty (threat detection)
- Inspector (vulnerability assessment)
- Macie (data privacy)
- IAM Access Analyzer (access analysis)
- AWS Config (compliance)
- Firewall Manager (security policies)

**AWS Integrations**:
- Third-party tools: Splunk, Jira, ServiceNow, Slack

### Automated Security Checks

Security Hub continuously checks against:
1. **CIS AWS Foundations Benchmark**: 150+ best practices
2. **PCI DSS 3.2.1**: Payment card compliance
3. **AWS Foundational Security Best Practices**: AWS-specific security baselines
4. **HIPAA**: Health insurance compliance

**Examples**:
- "Ensure CloudTrail is enabled in all regions"
- "Ensure MFA is enabled for all IAM users"
- "Ensure S3 bucket has default encryption"

### Findings Normalization
- **ASFF** (Amazon Security Finding Format): Standardized format
- **Benefit**: Findings from GuardDuty, Inspector, Macie all use same format
- **Result**: Can create unified alerts and automation

### Cross-Account/Cross-Region Aggregation
- **Aggregation account**: Central account views findings from all linked accounts
- **Multi-region**: See findings from all regions
- **Use case**: Enterprise-wide security posture in one place

### EventBridge Integration
- Security Hub findings → EventBridge rules → Lambda/SNS/etc.
- **Example**: "Any finding with severity=HIGH → send Slack notification"

### Exam Context
- "Single pane of glass for all security findings" → Security Hub
- "Aggregate GuardDuty, Inspector, Macie findings" → Security Hub
- "Check AWS security best practices" → Security Hub automated checks

### Security Hub CLI Commands

```bash
# Enable Security Hub in region
aws securityhub enable-security-hub --region us-east-1

# Get hub status
aws securityhub describe-hub --region us-east-1

# Get findings (unfiltered)
aws securityhub get-findings --region us-east-1

# Get findings with filters
aws securityhub get-findings \
  --filters '{"Severity":{"Value":"HIGH","Comparison":"EQUALS"}}' \
  --region us-east-1

# List findings
aws securityhub list-findings \
  --finding-criteria '{"Criterion":{"severity":{"Value":"HIGH"}}}' \
  --region us-east-1

# Update finding state (mark as resolved)
aws securityhub update-findings \
  --finding-identifiers '[{"Id":"id-value","ProductArn":"arn:aws:securityhub:region:account:product/account/default"}]' \
  --workflow Status=RESOLVED \
  --region us-east-1

# Create insights (custom search saved)
aws securityhub create-insight \
  --name "high-severity-findings" \
  --filters '{"Severity":{"Value":"HIGH","Comparison":"EQUALS"}}' \
  --group-by RESOURCE_ID \
  --region us-east-1

# Disable Security Hub
aws securityhub disable-security-hub --region us-east-1
```

---

## 14. IAM Access Analyzer

### What is Access Analyzer?
Identifies resources (S3 buckets, KMS keys, IAM roles, etc.) that are shared outside your account or organization.

**Analogy**: Like an auditor that reviews your resource policies and warns "this S3 bucket is accessible to the public" or "this KMS key can be used by another company".

### Resources Analyzed

- **S3 buckets**: Bucket policies, ACLs
- **KMS keys**: Key policies
- **IAM roles**: Trust policies, resource-based policies
- **Lambda functions**: Resource-based policies
- **SQS queues**: Queue policies
- **Secrets Manager secrets**: Resource policies
- **SNS topics**: Topic policies
- **RDS databases**: IAM database authentication policies

### Finding Types

#### External Access Finding
- **What**: Resource is accessible to external principal
- **Examples**:
  - "S3 bucket publicly readable (Principal: '*')"
  - "KMS key can be used by AWS account 999888777666"
  - "IAM role can be assumed by EC2 service in another account"
- **Severity**: Varies (critical if public, medium if specific account)

### Access Preview
**Use case**: "If I apply this new policy, what access will it grant?"

**Process**:
1. Modify resource policy in Access Analyzer (don't apply yet)
2. "Preview" shows exactly what access it grants
3. Review; if safe, apply policy

**Benefit**: Catch misconfigurations before they go live

### Policy Validation
- Checks IAM policies for:
  - Syntax errors
  - Best practices (e.g., overly permissive actions)
  - Unused permissions
- **Integration**: Works with IAM Policy Simulator too

### Unused Access Analyzer
- Identifies:
  - Unused IAM roles (created 90+ days ago, never assumed)
  - Unused permissions (permissions in policy never used)
  - Unused access keys
- **Benefit**: Clean up unused, stale credentials

### Multi-Account Setup
- **Analyzer account**: Central account
- **Shared analyzer**: Analyze resources across all accounts in organization
- **Benefit**: Organization-wide visibility

### Exam Scenarios

| Scenario | Solution |
|----------|----------|
| "Find all publicly-accessible S3 buckets" | Access Analyzer |
| "Review new policy before applying" | Access Analyzer (Access Preview) |
| "Identify unused IAM roles" | Unused Access Analyzer |
| "Verify KMS key not shared outside account" | Access Analyzer |

### IAM Access Analyzer CLI Commands

```bash
# Enable Access Analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name "my-analyzer" \
  --type ACCOUNT \
  --region us-east-1

# List analyzers
aws accessanalyzer list-analyzers --region us-east-1

# List findings
aws accessanalyzer list-findings \
  --analyzer-arn "arn:aws:access-analyzer:us-east-1:111122223333:analyzer/ConsoleAnalyzer-123456" \
  --region us-east-1

# Get finding details
aws accessanalyzer get-finding \
  --analyzer-arn "arn:aws:access-analyzer:us-east-1:111122223333:analyzer/ConsoleAnalyzer-123456" \
  --id "finding-id" \
  --region us-east-1

# List analyzed resources
aws accessanalyzer list-analyzed-resources \
  --analyzer-arn "arn:aws:access-analyzer:us-east-1:111122223333:analyzer/ConsoleAnalyzer-123456" \
  --resource-type "AWS::S3::Bucket" \
  --region us-east-1

# Validate policy
aws accessanalyzer validate-policy \
  --policy-document file://policy.json \
  --policy-type IDENTITY_POLICY \
  --region us-east-1

# Delete analyzer
aws accessanalyzer delete-analyzer \
  --analyzer-arn "arn:aws:access-analyzer:us-east-1:111122223333:analyzer/ConsoleAnalyzer-123456" \
  --region us-east-1
```

---

## 15. S3 Encryption Methods (Exam Favorite)

### SSE-S3 (Server-Side Encryption with S3-Managed Keys)
- **Key management**: AWS manages encryption keys (AWS Owned Key)
- **Encryption**: AES-256-GCM
- **Header**: `x-amz-server-side-encryption: AES256`
- **Cost**: Free (included in S3)
- **Audit trail**: CloudTrail doesn't log individual S3 encryption calls
- **Default**: Since 2023, ALL S3 objects encrypted by default with SSE-S3
- **Use case**: Default, simple encryption; no audit trail needed
- **Exam context**: "S3 encrypted by default" → SSE-S3

### SSE-KMS (Server-Side Encryption with KMS-Managed Keys)
- **Key management**: You use KMS CMK (Customer Managed Key)
- **Encryption**: KMS encrypts data key; data encrypted locally with data key
- **Header**: `x-amz-server-side-encryption: aws:kms` + `x-amz-server-side-encryption-aws-kms-key-id: <key-arn>`
- **Cost**: $1/month per CMK + KMS API costs
- **Audit trail**: CloudTrail logs every S3 encryption/decryption (each S3 PutObject calls KMS Encrypt)
- **Throttling risk**: High S3 upload rate → KMS API throttling → 5,500 req/sec limit
- **Solution**: Enable **S3 Bucket Key** feature
  - Reduces KMS calls by ~99% (caches data key at S3 level for 5 minutes)
  - Recommended for high-volume S3 uploads
- **Use case**: Audit trail needed, control over encryption key, compliance requirement
- **Exam context**: "Control your own key and see CloudTrail audit trail" → SSE-KMS

### SSE-C (Server-Side Encryption with Customer-Provided Keys)
- **Key management**: You generate and provide the key in the request
- **Encryption**: AWS encrypts with your key, stores encrypted data
- **Key location**: Never stored by AWS; you manage it (e.g., KMS, on-premises HSM)
- **Header**: `x-amz-server-side-encryption-customer-algorithm: AES256` + `x-amz-server-side-encryption-customer-key: <base64-key>`
- **HTTPS required**: Must use HTTPS (key travels in header; TLS protects it)
- **Cost**: No additional cost
- **Audit trail**: No CloudTrail logging (AWS never sees plaintext key)
- **Use case**: You want complete key control; AWS cannot help if you lose key
- **Exam context**: "Bring your own key, AWS stores ciphertext only" → SSE-C

### CSE (Client-Side Encryption)
- **Where encryption happens**: On client (your application) BEFORE sending to S3
- **Key management**: You manage the key entirely (KMS, on-premises, etc.)
- **What AWS stores**: Ciphertext only
- **Audit trail**: AWS has no record of decryption (happens client-side)
- **Cost**: No KMS/S3 encryption cost (you manage keys)
- **Use case**: Most secure; AWS cannot help if you lose key
- **Exam context**: "Encrypt before even touching S3" → CSE

### Comparison Table

| Method | Key Location | Audit Trail | Cost | Setup Complexity |
|--------|---|---|---|---|
| SSE-S3 | AWS | No | Free | None (default) |
| SSE-KMS | KMS (your CMK) | Yes (CloudTrail) | $1/mo + API | Medium |
| SSE-C | You (external) | No | Free | High |
| CSE | You (external) | No | Free | Highest |

### S3 Bucket Key Feature (For SSE-KMS)
**Problem**: S3 uploads with SSE-KMS throttle because:
- Each S3 PutObject calls KMS GenerateDataKey
- High volume (1000s of uploads) → KMS API limit reached
- Request fails with ThrottlingException

**Solution**: Enable **S3 Bucket Key**
- Reduces KMS API calls by ~99%
- How: S3 generates data key once, caches it for 5 minutes, reuses for multiple uploads
- Benefit: Throttling eliminated, same encryption, minimal cost ($0.03/month per bucket)
- **Exam tip**: "KMS throttling with S3" → enable S3 Bucket Key

### S3 Encryption CLI Commands

```bash
# Upload with SSE-S3 (default, explicit)
aws s3 cp myfile.txt s3://my-bucket/ \
  --sse AES256

# Upload with SSE-KMS
aws s3 cp myfile.txt s3://my-bucket/ \
  --sse aws:kms \
  --sse-kms-key-id arn:aws:kms:us-east-1:111122223333:key/12345678

# Upload with SSE-C (customer-provided key)
# First, generate a key and convert to base64
aws s3 cp myfile.txt s3://my-bucket/ \
  --sse-c AES256 \
  --sse-c-key fileb://my-key.bin

# Enable S3 Bucket Key for SSE-KMS (reduces KMS calls)
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:111122223333:key/12345678"
      },
      "BucketKeyEnabled": true
    }]
  }'

# Get bucket encryption configuration
aws s3api get-bucket-encryption --bucket my-bucket

# Set default encryption (SSE-KMS with CMK)
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:111122223333:key/12345678"
      }
    }]
  }'

# Require SSE in bucket policy (deny unencrypted uploads)
aws s3api put-bucket-policy --bucket my-bucket \
  --policy file://bucket-policy.json

# Client-side encryption with AWS SDK (Python example comment)
# s3_client = boto3.client('s3')
# encrypted_data = encrypt_with_kms(plaintext_data)
# s3_client.put_object(Bucket='my-bucket', Key='file.txt', Body=encrypted_data)
```

---

## 16. Comparison Tables

### Security Services Quick Reference

| Service | Protects Against | Layer | Free? | Best For |
|---------|-----------------|-------|-------|----------|
| WAF | SQLi, XSS, L7 DDoS, bots | L7 (HTTP) | No (~$5/mo) | Web application attacks |
| Shield Standard | SYN/UDP floods, reflection | L3/L4 | Yes | Basic DDoS (all customers) |
| Shield Advanced | Sophisticated multi-layer DDoS | L3/L4/L7 | $3,000/mo | Critical apps, cost protection |
| GuardDuty | Compromised resources, malware | ML/behavioral | 30-day trial | Threat detection |
| Inspector | Software vulnerabilities (CVEs) | EC2/ECR/Lambda | Per usage | Vulnerability assessment |
| Macie | Sensitive data in S3 (PII) | S3 data | Per usage | Data privacy, compliance |
| Security Hub | Aggregates all findings | Central | Free (with sources) | Single pane of glass |
| Access Analyzer | External resource access | IAM/resource policies | Free | Misconfiguration audit |

### Secrets Storage Quick Reference

| Feature | SSM Parameter Store | Secrets Manager |
|---------|-------------------|-----------------|
| **Auto-rotation** | No (manual Lambda) | Yes (built-in for RDS, Redshift, etc.) |
| **Cost** | Free (Standard), $0.05/mo (Advanced) | $0.40/secret/mo + $0.05/10K API calls |
| **Native RDS integration** | No | Yes (auto-password rotation) |
| **Max size** | 4 KB (Standard), 8 KB (Advanced) | 65 KB |
| **TTL/Expiration** | Advanced tier only | No |
| **Versioning** | Yes (manual) | Yes (AWSCURRENT/AWSPREVIOUS) |
| **Primary use** | Configuration + secrets | Secrets only |
| **Cost-effectiveness** | Storing 50+ config items | Storing 10s of secrets |

### Encryption Methods for S3

| Method | Key Control | Audit Trail | Complexity | Cost | Exam Context |
|--------|---|---|---|---|---|
| SSE-S3 | AWS | CloudTrail (generic) | None | Free | Default since 2023 |
| SSE-KMS | You (CMK) | CloudTrail (detailed) | Medium | $1/mo + KMS API | Control + audit trail |
| SSE-C | You (external) | None | High | Free | "Bring your key" |
| CSE | You (external) | None | Highest | Free | "Encrypt before AWS" |

### Key Management Tools

| Tool | Key Storage | FIPS Level | Cost | Best For |
|------|---|---|---|---|
| KMS | AWS managed | Level 2 | $1/mo/key | Most use cases |
| CloudHSM | Your hardware | Level 3 | $1.69/hr | Regulatory, on-prem integration |
| ACM | AWS (cannot export) | Not applicable | Free (public) | HTTPS certificates |

---

## 17. Points to Remember (Exam Focus)

### KMS — Critical Exam Topics
- **Envelope Encryption**: GenerateDataKey for >4KB data; reduces KMS throttling
- **Multi-Region Keys**: Same key ID across regions; replicas independent after creation
- **Key Policies**: Even if IAM allows, key policy must permit access (both required)
- **Automatic Rotation**: Every 365 days for CMK; imported keys rotation = manual only
- **Cross-Account Access**: Key policy + IAM policy both must allow
- **Throttling Solution**: S3 Bucket Key for SSE-KMS (reduces KMS API calls by 99%)

### CloudHSM — When to Choose
- Regulatory: "Must use FIPS 140-2 Level 3" → CloudHSM
- On-premises integration required
- Complete key material control needed
- Don't choose for typical S3/RDS encryption (KMS is simpler)

### SSM Parameter Store vs Secrets Manager
- **Parameter Store**: Configuration values, cheap, Standard tier free
- **Secrets Manager**: Passwords/credentials, native auto-rotation, auto-RDS integration
- **Exam**: "Auto-rotate RDS password?" → Secrets Manager; "Store 100 config values?" → SSM PS Standard

### S3 Encryption — Exam Favorite
- **Default**: SSE-S3 (AWS-managed keys, free)
- **Audit trail + CMK control**: SSE-KMS
- **KMS throttling**: Enable S3 Bucket Key
- **Bring your own key**: SSE-C (you manage, AWS stores ciphertext)
- **Client encrypts before S3**: CSE (you manage, highest security)

### WAF — Layer 7 Only
- **Protects**: ALB, API Gateway, CloudFront, AppSync, Cognito User Pool
- **Does NOT protect**: NLB directly (Layer 4 service)
- **Rules**: SQL injection, XSS, geo-blocking, rate-limiting
- **Not for EC2 directly**: Must use ALB in front

### Shield — DDoS Protection
- **Standard** (free): Automatic L3/L4 protection
- **Advanced** ($3K/mo): DDoS Response Team, cost protection for scaling
- **DDoS Cost Protection**: AWS credits charges caused by DDoS attack scaling
- **Exam**: "Business loses $100K due to DDoS scaling" → Shield Advanced cost protection

### GuardDuty — Detection, Not Prevention
- **Input**: CloudTrail, VPC Flow Logs, DNS Logs, S3 events, EKS Audit Logs, RDS Logins, Lambda network activity
- **Findings**: CryptoCurrency, UnauthorizedAccess, Trojan, etc.
- **Auto-remediation**: GuardDuty finds threat → EventBridge → Lambda stops/isolates
- **Does NOT**: Prevent attacks by itself (only detects)

### Inspector — Vulnerability Scanner
- **EC2**: OS package CVEs (requires SSM agent)
- **ECR**: Container image layer scanning
- **Lambda**: Code and dependency vulnerabilities
- **v2**: Continuous scanning (not just scheduled)

### Macie — Data Privacy
- **Finds**: PII, PHI, financial data, credentials in S3
- **Reports**: Data locations and sensitivity
- **Multi-account**: Via AWS Organizations

### Security Hub — Central Pane of Glass
- **Aggregates**: GuardDuty, Inspector, Macie, IAM Access Analyzer, Firewall Manager, AWS Config
- **Automated checks**: CIS Benchmark, PCI DSS, AWS Foundational Best Practices
- **EventBridge**: Trigger automation on high-severity findings

### Firewall Manager — Organization-Wide Security
- **Prerequisite**: AWS Organizations enabled
- **Manages**: WAF policies, Shield Advanced, Security Groups, Network Firewall, Route53 DNS Firewall
- **Benefit**: Uniform security posture across 50+ accounts

### Access Analyzer — Misconfiguration Audit
- **Finds**: External access to S3, KMS, IAM roles, Lambda, Secrets Manager
- **Access Preview**: Test policy before applying
- **Unused Access**: Identify unused roles and permissions

### Encryption in Flight vs at Rest
- **In flight**: TLS/SSL, HTTPS, protects from MITM
- **At rest**: Data encrypted in storage (S3, EBS, RDS, DynamoDB)
- **Client-side**: Client encrypts before sending (AWS never sees plaintext)

---

## 18. AWS CLI Quick Reference

### KMS Commands
```bash
# Create key
aws kms create-key --description "My key" --region us-east-1

# List keys
aws kms list-keys --region us-east-1

# Describe key
aws kms describe-key --key-id alias/my-key

# Enable auto-rotation
aws kms enable-key-rotation --key-id alias/my-key

# Create alias
aws kms create-alias --alias-name alias/my-app-key --target-key-id <key-arn>

# Encrypt (max 4KB)
aws kms encrypt --key-id alias/my-key --plaintext fileb://secret.txt

# Decrypt
aws kms decrypt --ciphertext-blob fileb://encrypted.txt

# Generate data key (envelope encryption)
aws kms generate-data-key --key-id alias/my-key --key-spec AES_256

# Generate data key without plaintext
aws kms generate-data-key-without-plaintext --key-id alias/my-key --key-spec AES_256

# Get rotation status
aws kms get-key-rotation-status --key-id alias/my-key

# List aliases
aws kms list-aliases

# Get key policy
aws kms get-key-policy --key-id alias/my-key --policy-name default

# Schedule deletion (15-30 days)
aws kms schedule-key-deletion --key-id alias/my-key --pending-window-in-days 30

# Replicate key to another region
aws kms replicate-key --key-id <mrk-arn> --replica-region eu-west-1
```

### SSM Parameter Store Commands
```bash
# Put string parameter
aws ssm put-parameter --name "/app/config" --value "value" --type String

# Put secure string (encrypted)
aws ssm put-parameter --name "/app/secret" --value "password" --type SecureString --key-id alias/ssm-key

# Get parameter
aws ssm get-parameter --name "/app/secret" --with-decryption

# Get parameters by path
aws ssm get-parameters-by-path --path "/app" --recursive --with-decryption

# List all parameters
aws ssm describe-parameters

# Delete parameter
aws ssm delete-parameter --name "/app/old-config"

# Tag parameter
aws ssm add-tags-to-resource --resource-type "Parameter" --resource-id "/app/secret" --tags "Key=Env,Value=prod"
```

### Secrets Manager Commands
```bash
# Create secret
aws secretsmanager create-secret --name "prod/db-password" --secret-string "MyPassword123!"

# Get secret
aws secretsmanager get-secret-value --secret-id "prod/db-password"

# Update secret
aws secretsmanager update-secret --secret-id "prod/db-password" --secret-string "NewPassword456!"

# Describe secret
aws secretsmanager describe-secret --secret-id "prod/db-password"

# List secrets
aws secretsmanager list-secrets

# Rotate secret
aws secretsmanager rotate-secret --secret-id "prod/db-password"

# Delete secret (with recovery)
aws secretsmanager delete-secret --secret-id "prod/db-password" --recovery-window-in-days 30
```

### ACM Commands
```bash
# Request certificate
aws acm request-certificate --domain-name example.com --validation-method DNS --region us-east-1

# List certificates
aws acm list-certificates --region us-east-1

# Describe certificate
aws acm describe-certificate --certificate-arn <arn> --region us-east-1

# Import certificate
aws acm import-certificate --certificate fileb://cert.pem --private-key fileb://key.pem --region us-east-1

# Delete certificate
aws acm delete-certificate --certificate-arn <arn> --region us-east-1
```

### WAF Commands
```bash
# Create IP set
aws wafv2 create-ip-set --name "blocked-ips" --scope REGIONAL --ip-address-version IPV4 --addresses "192.0.2.0/24"

# Create Web ACL
aws wafv2 create-web-acl --name "my-acl" --scope REGIONAL --default-action Block={} --rules file://rules.json --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=myACL

# List Web ACLs
aws wafv2 list-web-acls --scope REGIONAL

# Associate Web ACL with ALB
aws wafv2 associate-web-acl --web-acl-arn <acl-arn> --resource-arn <alb-arn>

# Disassociate Web ACL
aws wafv2 disassociate-web-acl --resource-arn <alb-arn>

# Get Web ACL
aws wafv2 get-web-acl --name "my-acl" --scope REGIONAL --id <acl-id>

# Delete Web ACL
aws wafv2 delete-web-acl --name "my-acl" --scope REGIONAL --id <id> --lock-token <token>
```

### Shield Commands
```bash
# Enable Shield Advanced (admin account only)
aws shield create-subscription --region us-east-1

# Create protection for resource
aws shield create-protection --name "my-protection" --resource-arn <arn>

# List protections
aws shield list-protections

# Describe protection
aws shield describe-protection --resource-arn <arn>

# Describe attacks
aws shield describe-attacks --resource-arns <arn>

# Delete protection
aws shield delete-protection --protection-id <id>
```

### Firewall Manager Commands
```bash
# Enable Firewall Manager (admin account only)
aws fms enable-organization-admin-account --admin-account <account-id>

# Create policy
aws fms put-policy --policy file://policy.json

# List policies
aws fms list-policies

# Get policy
aws fms get-policy --policy-id <id>

# List non-compliant resources
aws fms list-non-compliant-resources --policy-id <id>

# Delete policy
aws fms delete-policy --policy-id <id> --delete-all-policy-resources
```

### GuardDuty Commands
```bash
# Enable GuardDuty
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES

# List detectors
aws guardduty list-detectors

# Get detector
aws guardduty get-detector --detector-id <id>

# List findings
aws guardduty list-findings --detector-id <id>

# Get findings
aws guardduty get-findings --detector-id <id> --finding-ids <finding-id>

# Create filter
aws guardduty create-filter --detector-id <id> --name "filter" --finding-criteria file://criteria.json --action NOOP

# Update detector
aws guardduty update-detector --detector-id <id> --finding-publishing-frequency SIXTY_MINUTES

# Disable GuardDuty
aws guardduty delete-detector --detector-id <id>
```

### Inspector Commands
```bash
# Enable Inspector v2
aws inspector2 enable --resource-types EC2 ECR LAMBDA

# List findings
aws inspector2 list-findings

# Get findings
aws inspector2 get-findings --finding-arns <arn>

# Disable Inspector
aws inspector2 disable --resource-types EC2 ECR LAMBDA
```

### Macie Commands
```bash
# Enable Macie
aws macie2 enable-macie

# Get Macie status
aws macie2 get-macie-session

# List findings
aws macie2 list-findings

# Get findings
aws macie2 get-findings --finding-ids <id>

# Create findings filter
aws macie2 create-findings-filter --name "filter" --finding-criteria file://criteria.json --action ARCHIVE

# Disable Macie
aws macie2 disable-macie
```

### Security Hub Commands
```bash
# Enable Security Hub
aws securityhub enable-security-hub

# Get findings
aws securityhub get-findings

# Get findings with filters
aws securityhub get-findings --filters file://filters.json

# List findings
aws securityhub list-findings

# Update findings
aws securityhub update-findings --finding-identifiers file://identifiers.json --workflow Status=RESOLVED

# Create insight
aws securityhub create-insight --name "insight" --filters file://filters.json --group-by RESOURCE_ID

# Disable Security Hub
aws securityhub disable-security-hub
```

### Access Analyzer Commands
```bash
# Create analyzer
aws accessanalyzer create-analyzer --analyzer-name "my-analyzer" --type ACCOUNT

# List analyzers
aws accessanalyzer list-analyzers

# List findings
aws accessanalyzer list-findings --analyzer-arn <arn>

# Get findings
aws accessanalyzer get-finding --analyzer-arn <arn> --id <finding-id>

# Validate policy
aws accessanalyzer validate-policy --policy-document file://policy.json --policy-type IDENTITY_POLICY

# Delete analyzer
aws accessanalyzer delete-analyzer --analyzer-arn <arn>
```

### S3 Encryption Commands
```bash
# Upload with SSE-S3
aws s3 cp file.txt s3://bucket/ --sse AES256

# Upload with SSE-KMS
aws s3 cp file.txt s3://bucket/ --sse aws:kms --sse-kms-key-id <key-arn>

# Upload with SSE-C
aws s3 cp file.txt s3://bucket/ --sse-c AES256 --sse-c-key fileb://key.bin

# Enable S3 Bucket Key (reduces KMS calls)
aws s3api put-bucket-encryption --bucket my-bucket --server-side-encryption-configuration file://config.json

# Get bucket encryption
aws s3api get-bucket-encryption --bucket my-bucket

# Set default encryption with CMK
aws s3api put-bucket-encryption --bucket my-bucket --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms","KMSMasterKeyID":"<key-arn>"},"BucketKeyEnabled":true}]}'
```

---

## Study Tips

1. **Understand the "WHY"**: Don't memorize commands; understand why each service exists.
   - KMS is for key management, not data encryption directly
   - WAF is Layer 7 (HTTP-aware), not for NLB (Layer 4)
   - GuardDuty detects, doesn't prevent

2. **Analogies work**: Use real-world comparisons to explain concepts
   - KMS = key vault (AWS manages building, you control keys)
   - Envelope encryption = buying a safe with a lock, mailing it (key locked inside)
   - Multi-region keys = having the same bank account number in multiple countries

3. **Exam scenarios**: Always map to the right tool
   - "Auto-rotate RDS password" → Secrets Manager (not SSM)
   - "Encrypt 1TB file" → Envelope encryption (not direct KMS)
   - "FIPS 140-2 Level 3" → CloudHSM (not KMS)

4. **Cost awareness**: Know pricing models for exam cost optimization questions
   - KMS: $1/month per CMK
   - Secrets Manager: $0.40/secret/month
   - Shield Advanced: $3,000/month
   - Inspector: Per usage

5. **Integration patterns**:
   - Service detects threat (GuardDuty, Inspector) → EventBridge → Lambda → Auto-remediate
   - Find security issue (Access Analyzer, Macie) → Security Hub → Audit/remediate

6. **Regional considerations**:
   - KMS keys are regional (multi-region keys for global)
   - ACM certificate for CloudFront must be in us-east-1
   - WAF can be CloudFront (us-east-1) or regional

---

**Good luck with your AWS SAA-C03 exam! Focus on understanding WHY each service exists, not just memorizing facts.**

