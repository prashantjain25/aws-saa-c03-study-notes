# Amazon S3 – Advanced & Security

## S3 Security Overview

### Bucket Policies (Resource-based)
- JSON-based policies attached to bucket
- Can grant cross-account access
- Can allow/deny public access
- Use case: grant public access, force HTTPS, cross-account access

### ACLs (Access Control Lists)
- Object-level or Bucket-level ACL
- Less common now; bucket policies preferred
- Can be disabled at bucket/account level

### Block Public Access Settings
- Account-level or bucket-level setting
- Created to prevent company data leaks
- If enabled, overrides bucket policies that grant public access
- Should be ON unless bucket needs to be public

## S3 Object Encryption

### Server-Side Encryption (SSE)
| Type | Key Managed By | Description |
|------|---------------|-------------|
| SSE-S3 | AWS | AWS-managed keys, AES-256, default encryption |
| SSE-KMS | AWS KMS | KMS Customer Managed Keys, audit with CloudTrail |
| SSE-C | Customer | Customer provides key in HTTP header, HTTPS only |
| DSSE-KMS | AWS KMS | Double-layer encryption (two separate KMS keys) |

- **SSE-S3**: default since Jan 2023 for new objects. Header: "x-amz-server-side-encryption": "AES256"
- **SSE-KMS**: more control + audit. Header: "x-amz-server-side-encryption": "aws:kms". Has KMS API quota limits.
- **SSE-C**: HTTPS required, key not stored by AWS, key must be in header every request
- **Client-Side Encryption**: client encrypts data before sending to S3

**Put object with SSE-S3:**
```bash
aws s3api put-object \
  --bucket my-bucket \
  --key sensitive-file.txt \
  --body sensitive-file.txt \
  --server-side-encryption AES256
```

**Put object with SSE-KMS:**
```bash
aws s3api put-object \
  --bucket my-bucket \
  --key sensitive-file.txt \
  --body sensitive-file.txt \
  --server-side-encryption aws:kms \
  --ssekms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123
```

**Put object with SSE-C (customer-managed key in request headers):**
```bash
aws s3api put-object \
  --bucket my-bucket \
  --key secure-file.txt \
  --body secure-file.txt \
  --sse-customer-algorithm AES256 \
  --sse-customer-key "$(echo -n 'my-32-byte-base64-encoded-key==' | base64)" \
  --sse-customer-key-md5 "$(echo -n 'my-32-byte-base64-encoded-key==' | md5sum | awk '{print $1}' | xxd -r -p | base64)"
```

**Set default bucket encryption (SSE-S3):**
```bash
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'
```

**Set default bucket encryption (SSE-KMS):**
```bash
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
    }}]
  }'
```

### Encryption in Transit (SSL/TLS)
- HTTP endpoint: not encrypted
- HTTPS endpoint: encrypted in transit
- Enforce HTTPS via bucket policy: deny HTTP requests (aws:SecureTransport condition)

**Force HTTPS-only bucket policy (deny HTTP):**
```bash
aws s3api put-bucket-policy \
  --bucket my-bucket \
  --policy '{
    "Statement": [{
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
      "Condition": {"Bool": {"aws:SecureTransport": "false"}}
    }]
  }'
```

## S3 CORS (Cross-Origin Resource Sharing)
- Cross-origin = different scheme, host, or port
- Browser makes preflight request with Origin header
- S3 bucket must have CORS configuration to allow requests from specific origins
- Use case: static website on S3 accessing another S3 bucket

**Set CORS configuration:**
```bash
aws s3api put-bucket-cors \
  --bucket my-bucket \
  --cors-configuration '{
    "CORSRules": [{
      "AllowedOrigins": ["https://www.example.com"],
      "AllowedMethods": ["GET", "HEAD"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 3000
    }]
  }'
```

**Get CORS configuration:**
```bash
aws s3api get-bucket-cors --bucket my-bucket
```

## S3 MFA Delete
- Requires MFA to: permanently delete a version, suspend versioning
- Does NOT require MFA to: enable versioning, list deleted versions
- Only bucket OWNER (root account) can enable/disable MFA Delete
- Versioning must be enabled first

**Enable MFA Delete (requires root credentials + MFA code):**
```bash
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device 123456"
```

## S3 Access Logs
- Log all requests to S3 bucket to another S3 bucket (in same region)
- Never set logging bucket = monitored bucket (creates infinite logging loop!)
- Used for audit purposes

**Enable server access logging:**
```bash
aws s3api put-bucket-logging \
  --bucket my-bucket \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "my-access-logs-bucket",
      "TargetPrefix": "my-bucket-logs/"
    }
  }'
```

## S3 Pre-Signed URLs
- Generate URL with time-limited access (using SDK or CLI)
- URL inherits permissions of the user who generated it
- Expiration: 3600s default (up to 604800s = 7 days for S3 console/CLI)
- Use case: allow premium users to download video, temporary file upload access

**Generate a pre-signed URL (GET — valid for 1 hour):**
```bash
aws s3 presign s3://my-bucket/private-video.mp4 --expires-in 3600
```

**Generate a pre-signed URL for PUT (allow upload for 5 minutes):**
```bash
aws s3 presign s3://my-bucket/uploads/document.pdf \
  --expires-in 300
```

## S3 Glacier Vault Lock & S3 Object Lock
- **Glacier Vault Lock**: adopt WORM model (Write Once Read Many), lock the policy for future edits, compliance/data retention
- **S3 Object Lock** (versioning must be enabled):
  - **Retention Mode – Compliance**: no user (including root) can overwrite/delete for retention period
  - **Retention Mode – Governance**: most users can't delete, some with special permissions can
  - **Legal Hold**: protect object indefinitely, independent of retention period

**Create bucket with Object Lock enabled (must be done at creation):**
```bash
aws s3api create-bucket \
  --bucket my-locked-bucket \
  --object-lock-enabled-for-bucket
```

**Put object with Compliance retention (30 days):**
```bash
aws s3api put-object-retention \
  --bucket my-locked-bucket \
  --key important-doc.pdf \
  --retention '{"Mode": "COMPLIANCE", "RetainUntilDate": "2026-12-31T00:00:00Z"}'
```

**Put legal hold on an object:**
```bash
aws s3api put-object-legal-hold \
  --bucket my-locked-bucket \
  --key important-doc.pdf \
  --legal-hold Status=ON
```

**Remove legal hold:**
```bash
aws s3api put-object-legal-hold \
  --bucket my-locked-bucket \
  --key important-doc.pdf \
  --legal-hold Status=OFF
```

## S3 Access Points
- Simplify security management for large S3 buckets
- Each access point has its own DNS name + access point policy
- Can be internet-facing or VPC-restricted
- Use case: separate access points for different teams (finance, HR, analytics)

**Create an S3 Access Point:**
```bash
aws s3control create-access-point \
  --account-id 123456789012 \
  --name finance-access-point \
  --bucket my-bucket \
  --vpc-configuration VpcId=vpc-1234567890abcdef0
```

**List access points:**
```bash
aws s3control list-access-points \
  --account-id 123456789012 \
  --bucket my-bucket
```

## S3 Object Lambda
- Use Lambda functions to modify objects before they're retrieved
- No need for multiple copies of data
- Use case: redact PII, convert data formats, resize images on-the-fly

## S3 Presigned URLs vs Access Points
- Presigned URLs = time-limited direct access to an object
- Access Points = simplified, scalable access management with separate policies

## ⭐ Interview Tips & Key Points to Remember
- **SSE-S3 is now the DEFAULT encryption** for new objects (since Jan 2023)
- **SSE-KMS has rate limits** (KMS API calls) — can become a bottleneck at high request rates
- **SSE-C requires HTTPS** and key in every request header — AWS never stores the key
- **Block Public Access is account-level** — OVERRIDES any bucket policy granting public access
- **CORS is browser-side protection** — S3 CORS config tells browsers which origins are allowed
- **MFA Delete requires root account** — cannot be enabled by IAM users
- **S3 Object Lock Compliance mode**: even root cannot delete — strongest protection
- **Object Lock Legal Hold**: no expiry date, can be set/removed by users with s3:PutObjectLegalHold
- **Pre-Signed URL**: common pattern for giving temporary access to private objects
- **Never log to the same bucket you're monitoring** — causes infinite loop
- **S3 Access Points**: each team gets their own entry point with scoped policies
- **Glacier Vault Lock**: for regulatory compliance (WORM) — once locked, policy can't be changed
- **Bucket policy vs ACL**: bucket policies are preferred; ACLs can be disabled (object ownership settings)

---

## Quick Reference — AWS CLI Commands

### S3 Encryption
```bash
# Put object with SSE-S3
aws s3api put-object --bucket my-bucket --key sensitive-file.txt --body sensitive-file.txt --server-side-encryption AES256

# Put object with SSE-KMS
aws s3api put-object --bucket my-bucket --key sensitive-file.txt --body sensitive-file.txt --server-side-encryption aws:kms --ssekms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123

# Set default bucket encryption (SSE-S3)
aws s3api put-bucket-encryption --bucket my-bucket --server-side-encryption-configuration '{"Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]}'

# Set default bucket encryption (SSE-KMS)
aws s3api put-bucket-encryption --bucket my-bucket --server-side-encryption-configuration '{"Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms", "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"}}]}'

# Force HTTPS-only
aws s3api put-bucket-policy --bucket my-bucket --policy '{"Statement": [{"Effect": "Deny", "Principal": "*", "Action": "s3:*", "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"], "Condition": {"Bool": {"aws:SecureTransport": "false"}}}]}'
```

### CORS Configuration
```bash
# Set CORS configuration
aws s3api put-bucket-cors --bucket my-bucket --cors-configuration '{"CORSRules": [{"AllowedOrigins": ["https://www.example.com"], "AllowedMethods": ["GET", "HEAD"], "AllowedHeaders": ["*"], "MaxAgeSeconds": 3000}]}'

# Get CORS configuration
aws s3api get-bucket-cors --bucket my-bucket
```

### MFA Delete & Access Logs
```bash
# Enable MFA Delete
aws s3api put-bucket-versioning --bucket my-bucket --versioning-configuration Status=Enabled,MFADelete=Enabled --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device 123456"

# Enable server access logging
aws s3api put-bucket-logging --bucket my-bucket --bucket-logging-status '{"LoggingEnabled": {"TargetBucket": "my-access-logs-bucket", "TargetPrefix": "my-bucket-logs/"}}'
```

### Pre-Signed URLs
```bash
# Generate presigned URL (GET, 1 hour)
aws s3 presign s3://my-bucket/private-video.mp4 --expires-in 3600

# Generate presigned URL (PUT, 5 minutes)
aws s3 presign s3://my-bucket/uploads/document.pdf --expires-in 300
```

### S3 Object Lock
```bash
# Create bucket with Object Lock
aws s3api create-bucket --bucket my-locked-bucket --object-lock-enabled-for-bucket

# Put object with Compliance retention
aws s3api put-object-retention --bucket my-locked-bucket --key important-doc.pdf --retention '{"Mode": "COMPLIANCE", "RetainUntilDate": "2026-12-31T00:00:00Z"}'

# Put legal hold on object
aws s3api put-object-legal-hold --bucket my-locked-bucket --key important-doc.pdf --legal-hold Status=ON

# Remove legal hold
aws s3api put-object-legal-hold --bucket my-locked-bucket --key important-doc.pdf --legal-hold Status=OFF
```

### S3 Access Points
```bash
# Create access point
aws s3control create-access-point --account-id 123456789012 --name finance-access-point --bucket my-bucket --vpc-configuration VpcId=vpc-1234567890abcdef0

# List access points
aws s3control list-access-points --account-id 123456789012 --bucket my-bucket
```
