# AWS CLI Command Examples from 

```bash
# ─── IAM ────────────────────────────────────────────────
aws iam list-users
aws iam list-roles
aws iam get-user
aws iam create-user --user-name dev-user

# ─── S3 ─────────────────────────────────────────────────
aws s3 ls                                    # list all buckets
aws s3 mb s3://my-bucket-name               # make bucket
aws s3 cp file.txt s3://my-bucket/          # upload
aws s3 sync ./folder s3://my-bucket/folder  # sync directory
aws s3 rm s3://my-bucket/file.txt           # delete

# ─── EC2 ────────────────────────────────────────────────
aws ec2 describe-instances
aws ec2 describe-vpcs
aws ec2 describe-subnets
aws ec2 describe-security-groups
aws ec2 run-instances \
  --image-id ami-0f58b397bc5c1f2e8 \
  --instance-type t2.micro \
  --key-name my-key \
  --count 1

# ─── VPC ────────────────────────────────────────────────
aws ec2 create-vpc --cidr-block 10.0.0.0/16
aws ec2 create-subnet \
  --vpc-id vpc-xxx \
  --cidr-block 10.0.1.0/24

# ─── EKS ────────────────────────────────────────────────
aws eks list-clusters
aws eks describe-cluster --name my-cluster
aws eks update-kubeconfig --name my-cluster --region ap-south-1

# ─── RDS ────────────────────────────────────────────────
aws rds describe-db-instances
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password secret123 \
  --allocated-storage 20

# ─── Lambda ─────────────────────────────────────────────
aws lambda list-functions
aws lambda invoke \
  --function-name my-function \
  --payload '{"key":"value"}' \
  output.json

# ─── CloudWatch Logs ────────────────────────────────────
aws logs describe-log-groups
aws logs tail /aws/lambda/my-function --follow  # like tail -f

# ─── SSM (no SSH needed!) ───────────────────────────────
aws ssm start-session --target i-1234567890abcdef0
# Opens shell to EC2 WITHOUT ssh keys, purely via IAM
```



---

## File 1: IAM-Users-Groups-Policies-Roles.md

### Example 1: Users Section
Location: After "👤 Users" heading

```bash
aws iam create-user --user-name prashant
aws iam list-users
```

### Example 2: Groups Section
Location: After "👥 Groups" heading

```bash
aws iam create-group --group-name Developers
aws iam add-user-to-group --user-name prashant --group-name Developers
```

### Example 3: Policies Section
Location: After "📋 Policies" heading

```bash
# Attach AWS-managed policy to group
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create inline policy from a JSON file
aws iam put-user-policy \
  --user-name prashant \
  --policy-name S3ReadPolicy \
  --policy-document file://s3-read-policy.json

aws iam list-attached-group-policies --group-name Developers
```

### Example 4: IAM Roles Section
Location: After "🤖 IAM Roles" explanation

```bash
aws iam create-role \
  --role-name EC2-S3-Role \
  --assume-role-policy-document file://ec2-trust-policy.json

aws iam attach-role-policy \
  --role-name EC2-S3-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/EC2-S3-Role \
  --role-session-name MySession
```

---

## File 2: EC2-Basics-InstanceTypes-SecurityGroups.md

### Example 1: On-Demand Instances
Location: After "### On-Demand Instances" heading

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name my-key-pair \
  --security-group-ids sg-0abc123def456 \
  --subnet-id subnet-0abc123def456 \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyServer}]'

aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running"

aws ec2 stop-instances --instance-ids i-1234567890abcdef0
aws ec2 start-instances --instance-ids i-1234567890abcdef0
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0
```

### Example 2: Security Groups Creation
Location: After "### How Security Groups Work" diagram

```bash
aws ec2 create-security-group \
  --group-name my-app-sg \
  --description "Security group for web app" \
  --vpc-id vpc-1234567890abcdef0
```

### Example 3: Security Group Rules
Location: After specific SG rule explanations

```bash
# HTTP (port 80) from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123def456 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# HTTPS (port 443)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123def456 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# SSH from specific IP
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123def456 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.10/32

# SG-to-SG rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-backend123 \
  --protocol tcp \
  --port 3306 \
  --source-group sg-frontend456
```

### Example 4: Elastic IP
Location: After "## 💾 Elastic IP" section

```bash
aws ec2 allocate-address --domain vpc
aws ec2 associate-address \
  --instance-id i-1234567890abcdef0 \
  --allocation-id eipalloc-0abc123def456
```

---

## File 3: S3-Basics-BucketsObjectsStorageClasses.md

### Example 1: Bucket Creation
Location: After "### Bucket Naming Rules" heading

```bash
# Create bucket in default region (us-east-1)
aws s3 mb s3://my-unique-bucket-name

# Create bucket in specific region
aws s3 mb s3://my-unique-bucket-name --region us-east-1

aws s3 ls
aws s3 ls s3://my-unique-bucket-name/
aws s3 rb s3://my-unique-bucket-name
aws s3 rb s3://my-unique-bucket-name --force
```

### Example 2: Storage Classes
Location: After "## S3 Storage Classes" table

```bash
aws s3 cp large-file.zip s3://my-bucket/ \
  --storage-class STANDARD_IA
```

### Example 3: Object Operations
Location: After "## S3 Object Operations" heading

```bash
aws s3 cp local-file.txt s3://my-bucket/path/local-file.txt
aws s3 sync ./local-folder s3://my-bucket/prefix/
aws s3 cp s3://my-bucket/file.txt ./local-file.txt
aws s3 mv s3://my-bucket/old-name.txt s3://my-bucket/new-name.txt
aws s3 rm s3://my-bucket/file.txt
aws s3 rm s3://my-bucket/folder/ --recursive
```

### Example 4: Versioning
Location: After "## S3 Versioning" section

```bash
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

aws s3api get-bucket-versioning --bucket my-bucket
aws s3api list-object-versions --bucket my-bucket
```

### Example 5: Encryption
Location: After "## S3 Encryption" section

```bash
# SSE-S3 (default)
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# SSE-KMS
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

# Upload with KMS encryption
aws s3 cp secret-file.txt s3://my-bucket/ \
  --sse aws:kms \
  --sse-kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123
```

### Example 6: Presigned URLs
Location: After "## S3 Presigned URLs" section

```bash
aws s3 presign s3://my-bucket/private-video.mp4 \
  --expires-in 3600
```

---

## Integration Pattern

All three files follow the same integration pattern:

1. **Concept Explanation** (existing content preserved)
2. **Contextual CLI Commands** (placed immediately after)
3. **Quick Reference Section** (at end of document)

Example flow:
```
┌─────────────────────────────────┐
│  Concept Explanation            │  Original content
│  (What is X? Why use it?)       │  (expanded)
└─────────────────────────────────┘
            ↓
┌─────────────────────────────────┐
│  Contextual CLI Commands        │  NEW
│  (How to do it in practice)     │  Commands shown
└─────────────────────────────────┘
            ↓
    [... more concepts ...]
            ↓
┌─────────────────────────────────┐
│  Quick Reference Section        │  NEW
│  (All commands organized)       │  Cheat sheet
└─────────────────────────────────┘
```

---

## Benefits

- **Reading → Learning → Doing**: Students see explanation, then immediately see the CLI command
- **Exam Prep**: Understand exactly what commands do when answering scenario questions
- **Real-World Use**: Commands are production-ready and can be copy-pasted
- **Quick Lookup**: Reference section enables fast command discovery
- **Progressive Learning**: Commands range from simple to complex within each section

~~~bash
# Modern enterprise login — no static keys
aws configure sso
# SSO Start URL:    https://mycompany.awsapps.com/start
# SSO Region:       ap-south-1
# Account ID:       123456789012
# Role name:        DevOpsAccess

# Login via browser
aws sso login --profile dev

# Token auto-refreshes, no long-lived credentials stored
```

---

## Part 2 — AWS CLI vs Terraform in Interviews

### Honest Reality Check
```
Year    | What interviews tested
--------|--------------------------------
2015-18 | AWS Console clicks
2018-21 | AWS CLI commands
2021-23 | Terraform + CLI together
2023+   | Terraform primary + CLI for debugging
```

### Where CLI Still Wins Over Terraform
```
Task                          | CLI  | Terraform | Why
------------------------------|------|-----------|---------------------------
Debugging live infra          | ✅   | ❌        | Query real state instantly
One-off investigation         | ✅   | ❌        | No state file needed
CI/CD pipeline checks         | ✅   | ⚠️        | aws sts, health checks
Bootstrapping Terraform       | ✅   | ❌        | Create S3 backend bucket
Reading logs live             | ✅   | ❌        | aws logs tail
EKS kubeconfig setup          | ✅   | ⚠️        | aws eks update-kubeconfig
Emergency hotfix              | ✅   | ❌        | Modify SG rule in 10 sec
Cost exploration              | ✅   | ❌        | aws ce get-cost-and-usage
Scripting + automation        | ✅   | ⚠️        | Bash + CLI pipelines
IAM debugging                 | ✅   | ❌        | aws iam simulate-principal-policy
```

---

### Interview Reality — What They Actually Ask
```
Junior DevOps / Cloud Engineer:
  ✅ aws configure, S3, EC2 basics
  ✅ IAM user creation
  ✅ Basic Terraform (EC2 + VPC)

Mid-level DevOps / SRE:
  ✅ CLI for debugging (logs, describe, get)
  ✅ Terraform modules, state management
  ✅ Both together in pipelines

Senior / Lead:
  ✅ CLI for incident response
  ✅ Terraform at scale (workspaces, remote state)
  ✅ When NOT to use Terraform (why CLI for X)
```

---

### The Combined Mental Model (What Interviewers Want to See)
```
Provision infra    → Terraform  (repeatable, versioned, reviewed)
Debug infra        → AWS CLI    (fast, live, queryable)
Automate ops tasks → AWS CLI    (scripts, pipelines, health checks)
Modify infra       → Terraform  (change tracked in state + git)
Emergency fix      → AWS CLI    (then backport to Terraform!)
~~~

