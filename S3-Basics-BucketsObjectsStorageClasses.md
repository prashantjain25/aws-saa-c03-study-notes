# Amazon S3 – Basics

## What is Amazon S3?
- Object storage service — "infinitely scaling" storage
- Use cases: backup/storage, disaster recovery, archive, hybrid cloud storage, hosting static websites, media hosting, data lakes, software delivery, static content

## S3 Buckets & Objects
- Buckets: globally unique name (across all AWS accounts, all regions), defined at region level
- Objects: files stored in S3, identified by Key (full path: prefix + object name)
- Max object size: 5 TB; for uploads > 5 GB must use multi-part upload
- Object metadata, tags, version ID also available

### Bucket Naming Rules
- No uppercase, no underscore
- 3-63 characters long
- Not an IP address
- Must start with lowercase letter or number
- Must NOT start with "xn--" or end with "-s3alias"

**Create a bucket:**

```bash
# Create bucket in default region (us-east-1)
aws s3 mb s3://my-unique-bucket-name

# Create bucket in specific region
aws s3 mb s3://my-unique-bucket-name --region us-east-1
```

**List all buckets:**

```bash
aws s3 ls
```

**List objects in a bucket:**

```bash
aws s3 ls s3://my-unique-bucket-name/
```

**Delete bucket (must be empty):**

```bash
aws s3 rb s3://my-unique-bucket-name
```

**Force delete bucket with all contents:**

```bash
aws s3 rb s3://my-unique-bucket-name --force
```

---

## S3 Storage Classes
| Class | Use Case | Availability | Min Duration |
|-------|----------|-------------|--------------|
| S3 Standard | General purpose, frequently accessed | 99.99% | None |
| S3 Standard-IA | Infrequent access, rapid retrieval | 99.9% | 30 days |
| S3 One Zone-IA | Non-critical, infrequent, single AZ | 99.5% | 30 days |
| S3 Glacier Instant Retrieval | Archive, millisecond retrieval | 99.9% | 90 days |
| S3 Glacier Flexible Retrieval | Archive, minutes-hours retrieval | 99.99% | 90 days |
| S3 Glacier Deep Archive | Long-term archive, hours retrieval | 99.99% | 180 days |
| S3 Intelligent-Tiering | Unknown/changing access patterns | 99.9% | None |

- Glacier Flexible Retrieval tiers: Expedited (1-5 min), Standard (3-5h), Bulk (5-12h)
- Glacier Deep Archive: Standard (12h), Bulk (48h)
- S3 Intelligent-Tiering: automatically moves objects between tiers, small monthly monitoring fee, no retrieval charges

**Upload a file with specific storage class:**

```bash
aws s3 cp large-file.zip s3://my-bucket/ \
  --storage-class STANDARD_IA
```

---

## S3 Object Operations

**Upload a single file:**

```bash
aws s3 cp local-file.txt s3://my-bucket/path/local-file.txt
```

**Upload entire directory (sync):**

```bash
aws s3 sync ./local-folder s3://my-bucket/prefix/
```

**Download a file from S3:**

```bash
aws s3 cp s3://my-bucket/file.txt ./local-file.txt
```

**Move (rename) a file:**

```bash
aws s3 mv s3://my-bucket/old-name.txt s3://my-bucket/new-name.txt
```

**Delete a specific object:**

```bash
aws s3 rm s3://my-bucket/file.txt
```

**Delete all objects in a prefix:**

```bash
aws s3 rm s3://my-bucket/folder/ --recursive
```

---

## S3 Lifecycle Rules
- Transition objects between storage classes based on age or last access time
- Expiration actions: delete objects after a certain time, incomplete multi-part uploads
- Rules can be applied to specific prefix or tags

**Set lifecycle policy from file:**

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json
```

---

## S3 Versioning
- Enabled at bucket level
- Protects against accidental deletes
- Deletion creates a "delete marker" (versioned delete is reversible)
- Suspending versioning does NOT delete previous versions
- "Null" version = object created before versioning was enabled

**Enable versioning:**

```bash
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled
```

**Check versioning status:**

```bash
aws s3api get-bucket-versioning --bucket my-bucket
```

**List object versions:**

```bash
aws s3api list-object-versions --bucket my-bucket
```

---

## S3 Replication
- Must enable versioning in source and destination buckets
- **CRR (Cross-Region Replication)**: compliance, lower latency, cross-account replication
- **SRR (Same-Region Replication)**: log aggregation, live replication between production and test
- Copying is asynchronous
- Must give proper IAM permissions to S3
- Only NEW objects are replicated after enabling (use S3 Batch Replication for existing)
- Delete markers can optionally be replicated; deletions with version ID are NOT replicated

**Set up S3 replication (after enabling versioning):**

```bash
aws s3api put-bucket-replication \
  --bucket source-bucket \
  --replication-configuration file://replication.json
```

---

## S3 Requester Pays
- Normally bucket owner pays for storage + data transfer
- With Requester Pays: requester pays for downloads (not storage)
- Requester must be authenticated AWS user
- Use case: share large datasets with other accounts

---

## S3 Event Notifications
- Trigger on: S3:ObjectCreated, S3:ObjectRemoved, S3:Replication, etc.
- Destinations: SNS, SQS, Lambda Function, EventBridge
- EventBridge: advanced filtering, multiple destinations, archive, replay events

---

## S3 Bucket Configuration

**Set bucket policy from file:**

```bash
aws s3api put-bucket-policy \
  --bucket my-bucket \
  --policy file://bucket-policy.json
```

**Get bucket policy:**

```bash
aws s3api get-bucket-policy --bucket my-bucket
```

**Block all public access:**

```bash
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

---

## S3 Encryption

**Enable server-side encryption (SSE-S3 by default):**

```bash
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

**Enable SSE-KMS encryption:**

```bash
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
      }
    }]
  }'
```

**Upload with SSE-KMS encryption:**

```bash
aws s3 cp secret-file.txt s3://my-bucket/ \
  --sse aws:kms \
  --sse-kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123
```

---

## S3 Presigned URLs

**Create pre-signed URL (valid for 1 hour = 3600 seconds):**

```bash
aws s3 presign s3://my-bucket/private-video.mp4 \
  --expires-in 3600
```

---

## S3 CORS Configuration

**Enable CORS on bucket:**

```bash
aws s3api put-bucket-cors \
  --bucket my-bucket \
  --cors-configuration file://cors.json
```

---

## S3 Performance
- 3,500 PUT/COPY/POST/DELETE requests per second per prefix
- 5,500 GET/HEAD requests per second per prefix
- Multi-Part Upload: recommended for files > 100 MB, required for > 5 GB
- S3 Transfer Acceleration: upload to Edge Location → AWS private network → S3 bucket
- Byte-Range Fetches: parallelize downloads, resilient to failures

---

## S3 Select & Glacier Select
- Retrieve a subset of data from S3 using SQL (server-side filtering)
- Up to 400% faster, 80% cheaper

---

## ⭐ Interview Tips & Key Points to Remember
- **S3 is global but buckets are region-specific** — bucket names must be globally unique
- **Max object size: 5 TB**; uploads > 5 GB require multi-part upload
- **Versioning delete = delete marker** (not permanent) — can be reversed by deleting the marker
- **Replication requires versioning** on both source and destination
- **Only NEW objects replicated** after replication is enabled — use Batch Replication for existing
- **S3 Standard-IA vs One Zone-IA**: One Zone-IA is cheaper but data is lost if AZ fails
- **Glacier Instant Retrieval**: millisecond access (same as Standard-IA but cheaper for infrequent) — min 90 days
- **S3 Intelligent-Tiering**: no retrieval fees, auto-tiering based on access patterns
- **S3 Transfer Acceleration** ≠ CloudFront — TA is for UPLOADS via Edge Locations
- **Lifecycle rules** automate moving data to cheaper storage classes over time
- **S3 performance tip**: use multi-prefix strategy for high request rates

---

## Quick Reference — AWS CLI Commands

### Bucket Operations
```bash
# Create a bucket (mb = make bucket)
aws s3 mb s3://my-unique-bucket-name --region us-east-1

# List all buckets
aws s3 ls

# List objects in bucket
aws s3 ls s3://my-unique-bucket-name/

# Delete bucket (must be empty)
aws s3 rb s3://my-unique-bucket-name

# Force delete bucket (including all objects)
aws s3 rb s3://my-unique-bucket-name --force
```

### Object Operations
```bash
# Upload a single file
aws s3 cp local-file.txt s3://my-bucket/path/local-file.txt

# Upload entire directory (sync)
aws s3 sync ./local-folder s3://my-bucket/prefix/

# Download a file
aws s3 cp s3://my-bucket/file.txt ./local-file.txt

# Move (rename) a file
aws s3 mv s3://my-bucket/old-name.txt s3://my-bucket/new-name.txt

# Delete a specific object
aws s3 rm s3://my-bucket/file.txt

# Delete all objects in a prefix
aws s3 rm s3://my-bucket/folder/ --recursive
```

### Versioning
```bash
# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

# Check versioning status
aws s3api get-bucket-versioning --bucket my-bucket

# List object versions
aws s3api list-object-versions --bucket my-bucket
```

### Bucket Configuration & Policies
```bash
# Set bucket policy from file
aws s3api put-bucket-policy \
  --bucket my-bucket \
  --policy file://bucket-policy.json

# Get bucket policy
aws s3api get-bucket-policy --bucket my-bucket

# Block all public access
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### Encryption
```bash
# Enable server-side encryption (SSE-S3 by default)
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# Enable SSE-KMS encryption
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
      }
    }]
  }'

# Upload with specific storage class
aws s3 cp large-file.zip s3://my-bucket/ \
  --storage-class STANDARD_IA

# Upload with SSE-KMS
aws s3 cp secret-file.txt s3://my-bucket/ \
  --sse aws:kms \
  --sse-kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123
```

### Presigned URLs & CORS
```bash
# Create pre-signed URL (valid for 1 hour = 3600 seconds)
aws s3 presign s3://my-bucket/private-video.mp4 \
  --expires-in 3600

# Enable CORS on bucket
aws s3api put-bucket-cors \
  --bucket my-bucket \
  --cors-configuration file://cors.json
```

### Lifecycle & Replication
```bash
# Set lifecycle policy from file
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json

# Set up S3 replication (after enabling versioning)
aws s3api put-bucket-replication \
  --bucket source-bucket \
  --replication-configuration file://replication.json
```

