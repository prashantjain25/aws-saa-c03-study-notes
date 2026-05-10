# AWS IAM – Identity & Access Management
> 📚 Official Docs: https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html  
> 🎯 SAA-C03 Exam Weight: High — appears in almost every scenario question

---

## 🤔 Why Does IAM Exist?

Imagine you just created an AWS account. By default, you have ONE account — the **root user** — that has god-mode access to everything. If you share this with your team, or if it gets hacked, your entire infrastructure is compromised.

**IAM solves this** by letting you create individual identities (users, roles, groups) and precisely control what each identity can do — which services, which actions, which resources.

> 🔑 **Core principle of IAM**: Least Privilege — give only the minimum permissions needed, nothing more.

---

## 🧱 IAM Building Blocks

IAM has 4 main building blocks:

```
┌──────────────────────────────────────────────────────────┐
│                    AWS Account                           │
│                                                          │
│  👑 Root User (avoid using!)                             │
│                                                          │
│  👥 Groups ──────────── attach ──────────── 📋 Policies  │
│      │                                          │        │
│      └── 👤 Users ────── or attach directly ───┘         │
│                                                          │
│  🤖 Roles ──────────────────────────────── 📋 Policies   │
│     (for AWS services or cross-account)                  │
└──────────────────────────────────────────────────────────┘
```

### 👤 Users
- Represent **real people** (or applications) in your organization
- Can belong to **zero, one, or multiple groups**
- Can have permissions directly attached (but this is bad practice — use groups instead)

**Create a user:**

```bash
aws iam create-user --user-name prashant
```

**List all users:**

```bash
aws iam list-users
```

### 👥 Groups
- A collection of users
- **Groups can ONLY contain users — NOT other groups** (this is a common exam trap)
- Permissions applied to a group are inherited by all users in the group

**Create a group:**

```bash
aws iam create-group --group-name Developers
```

**Add a user to a group:**

```bash
aws iam add-user-to-group --user-name prashant --group-name Developers
```

### 📋 Policies
- JSON documents that define what is allowed or denied
- Can be attached to Users, Groups, or Roles
- **Managed Policies** (reusable, AWS-provided or customer-created) vs **Inline Policies** (embedded in one user/role — not recommended)

**Attach a managed policy to a group:**

```bash
# Attach AWS-managed policy to group
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

**Create and attach an inline policy:**

```bash
# Create inline policy from a JSON file
aws iam put-user-policy \
  --user-name prashant \
  --policy-name S3ReadPolicy \
  --policy-document file://s3-read-policy.json
```

**List policies attached to a group:**

```bash
aws iam list-attached-group-policies --group-name Developers
```

### 🤖 Roles
- IAM identity designed for **AWS services or external entities**, not humans
- Example: An EC2 instance needs to read from S3 → give the EC2 an IAM Role with S3 read permissions
- Roles use **temporary credentials** (more secure than embedding access keys in your code)

---

## 📋 IAM Policy Deep Dive

A policy is a **JSON document**. Let's break it down so you fully understand every field:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadWrite",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "eu-west-1"
        }
      }
    }
  ]
}
```

| Field | Required? | Meaning |
|-------|-----------|---------|
| `Version` | Yes | Always `"2012-10-17"` — the policy language version |
| `Statement` | Yes | Array of individual permission statements |
| `Sid` | No | Statement ID — a label for your own reference |
| `Effect` | Yes | `"Allow"` or `"Deny"` |
| `Action` | Yes | The AWS API action(s) to allow/deny (e.g., `s3:GetObject`) |
| `Resource` | Yes | The ARN of the resource(s) this applies to |
| `Principal` | For resource policies | Who this policy applies to (used in S3/KMS resource policies) |
| `Condition` | No | Extra conditions (e.g., only allow from specific IP, only via HTTPS) |

### ⚠️ Policy Evaluation Logic

This is critical and frequently tested:

```
Request comes in
      │
      ▼
Is there an explicit DENY?  ──YES──▶  DENIED ❌
      │
      NO
      ▼
Is there an explicit ALLOW?  ──YES──▶  ALLOWED ✅
      │
      NO
      ▼
Implicit DENY (default) ──────────▶  DENIED ❌
```

> 💡 **Remember**: AWS defaults to DENY everything. An explicit DENY always wins over any ALLOW — even from another policy.

---

## 🔐 MFA – Multi-Factor Authentication

**What is MFA?**  
MFA adds a second layer of security beyond just a password. Even if someone steals your password, they can't log in without your physical MFA device.

Think of it like a bank: your card (password) + PIN (MFA device) = access.

### MFA Device Options:

```
┌─────────────────────────────────────────────────────────┐
│  Virtual MFA (app on your phone)                        │
│  ├── Google Authenticator (single device only)          │
│  └── Authy (multi-device, backup codes)                 │
│                                                         │
│  Hardware U2F Security Key (physical USB key)           │
│  └── YubiKey by Yubico (supports multiple IAM users     │
│      and root on a single device)                       │
│                                                         │
│  Hardware Key Fob                                       │
│  ├── Gemalto (3rd party)                                │
│  └── SurePassID (for AWS GovCloud)                      │
└─────────────────────────────────────────────────────────┘
```

**Create a virtual MFA device:**

```bash
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name prashant-mfa \
  --outfile /tmp/mfa-qr.png \
  --bootstrap-method QRCodePNG

# Scan the QR code, then enable MFA
aws iam enable-mfa-device \
  --user-name prashant \
  --serial-number arn:aws:iam::123456789012:mfa/prashant-mfa \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

> 🔗 Official MFA guide: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html

---

## 🚪 Three Ways to Access AWS

```
┌──────────────────────────────────────────────────────────────────┐
│                    HOW TO ACCESS AWS                             │
│                                                                  │
│  1️⃣  AWS Management Console (Browser)                            │
│      Authentication: Username + Password + MFA                   │
│      Best for: visual exploration, one-off tasks                 │
│                                                                  │
│  2️⃣  AWS CLI (Command Line Interface)                            │
│      Authentication: Access Key ID + Secret Access Key           │
│      Best for: automation scripts, quick commands                │
│      Install: https://aws.amazon.com/cli/                        │
│                                                                  │
│  3️⃣  AWS SDK (Software Development Kit)                          │
│      Authentication: Access Key ID + Secret Access Key           │
│      Best for: integrating AWS into your application code        │
│      Languages: Python (boto3), JavaScript, Java, Go, Ruby...    │
└──────────────────────────────────────────────────────────────────┘
```

**Access Keys** are like a username + password for programmatic access:
- `Access Key ID` ≈ username (can be shared)
- `Secret Access Key` ≈ password (**never share, never commit to git!**)

**Create access key for a user (for CLI/SDK access):**

```bash
aws iam create-access-key --user-name prashant
# Output contains AccessKeyId and SecretAccessKey — save it securely!
```

**Check who you are authenticated as:**

```bash
aws sts get-caller-identity
# Output: UserId, Account (Account ID), Arn
```

> ⚠️ **Real-world tip**: Accidentally pushed your AWS keys to GitHub? Rotate them IMMEDIATELY and set up git-secrets or AWS credential scanning to prevent this.

---

## 🤖 IAM Roles — For Services, Not Humans

**The problem they solve:**  
Imagine your EC2 instance needs to upload files to S3. You could hardcode AWS keys in the app... but that's terrible practice (keys in code = security nightmare). Instead, you assign an IAM Role to the EC2 instance. The instance automatically gets temporary credentials that rotate every few hours.

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   EC2 Instance                                              │
│   ┌───────────────────┐                                     │
│   │   Your App Code   │──── "I need S3 access" ────▶       │
│   └───────────────────┘                                     │
│           │                                                 │
│   IAM Role attached ──▶ AWS STS issues temp credentials     │
│           │                                                 │
│           ▼                                                 │
│   ✅ Can now call s3:PutObject (no hardcoded keys!)         │
└─────────────────────────────────────────────────────────────┘
```

**Common Role use cases:**
- EC2 Instance Roles → access S3, DynamoDB, Parameter Store
- Lambda Execution Roles → write to CloudWatch Logs, read from DynamoDB
- CloudFormation Roles → create/update AWS resources
- Cross-Account Roles → Account A users assume role in Account B

**Create a role with a trust policy (for EC2 to assume):**

```bash
# First, create trust policy file (ec2-trust-policy.json):
# {
#   "Version": "2012-10-17",
#   "Statement": [
#     {
#       "Effect": "Allow",
#       "Principal": {
#         "Service": "ec2.amazonaws.com"
#       },
#       "Action": "sts:AssumeRole"
#     }
#   ]
# }

aws iam create-role \
  --role-name EC2-S3-Role \
  --assume-role-policy-document file://ec2-trust-policy.json

# Attach a managed policy to the role
aws iam attach-role-policy \
  --role-name EC2-S3-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

**Assume a role via STS (cross-account or temporary access):**

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/EC2-S3-Role \
  --role-session-name MySession
# Returns temporary AccessKeyId, SecretAccessKey, and SessionToken
```

**Create a customer-managed policy:**

```bash
aws iam create-policy \
  --policy-name MyCustomPolicy \
  --policy-document file://my-policy.json
```

**Simulate an IAM policy (check if action would be allowed):**

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/prashant \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/my-object
```

> 🔗 STS (Security Token Service) is the service that issues temporary credentials for roles: https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html

---

## 🔍 IAM Security Tools

### 1. IAM Credentials Report (Account-Level)
- Downloadable CSV file listing **all IAM users** and the status of their credentials:
  - When was the password last used?
  - Is MFA enabled?
  - Are access keys active?
- Use case: **Security audit** — find users with old/unused credentials

**Generate and download credential report:**

```bash
# Generate report (async operation)
aws iam generate-credential-report

# Get the report (base64 encoded CSV)
aws iam get-credential-report --output text --query Content | base64 -d > credentials-report.csv
```

### 2. IAM Access Advisor (User-Level)
- Shows **which services a user has permissions for** and **when they last accessed each service**
- Use case: **Right-sizing permissions** — if a user hasn't used EC2 in 90 days, maybe they don't need that permission

```
IAM Access Advisor Output Example:
Service          Last Accessed       Access Level
─────────────────────────────────────────────────
S3               2 days ago          Full Access  ← recently used ✅
EC2              Never               Full Access  ← remove? 🤔
DynamoDB         3 months ago        Read Only    ← maybe remove 🤔
```

> 🔗 https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_access-advisor.html

---

## ✅ IAM Best Practices (AWS Official Recommendations)

1. **Never use the root account** for daily operations — only for initial setup and billing
2. **One real person = one IAM user** — never share credentials
3. **Assign users to groups**, then assign policies to groups (not individual users)
4. **Create a strong password policy** with expiration and complexity requirements
5. **Enable MFA** for root and all privileged IAM users
6. **Use Roles** for AWS services — never embed access keys in EC2 or Lambda
7. **Use Access Keys only** for CLI/SDK access; rotate them regularly
8. **Audit with Credentials Report + Access Advisor** regularly
9. **Apply Least Privilege** — start with minimum permissions and add as needed
10. **Never share Access Keys** — each developer gets their own

> 🔗 Official best practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html

---

## 🗺️ IAM Full Architecture Overview

```
                    ┌─────────────────────────────────────┐
                    │           AWS Account               │
                    │                                     │
                    │  ┌─────────────────────────────┐    │
                    │  │         IAM (Global)        │    │
                    │  │                             │    │
                    │  │  Root User ──── MFA         │    │
                    │  │      │                      │    │
                    │  │   Groups                    │    │
                    │  │  ┌───┴──────────────┐       │    │
                    │  │  │ Dev Group        │       │    │
                    │  │  │ ├── User: Alice  │       │    │
                    │  │  │ └── User: Bob    │       │    │
                    │  │  └──────────────────┘       │    │
                    │  │         │                   │    │
                    │  │    Policies (JSON)          │    │
                    │  │  ┌──────────────────┐       │    │
                    │  │  │ AmazonS3ReadOnly │       │    │
                    │  │  │ EC2FullAccess    │       │    │
                    │  │  └──────────────────┘       │    │
                    │  │                             │    │
                    │  │  Roles (for services)       │    │
                    │  │  ┌──────────────────────┐   │    │
                    │  │  │ EC2-S3-Role          │   │    │
                    │  │  │ Lambda-DynamoDB-Role │   │    │
                    │  │  └──────────────────────┘   │    │
                    │  └─────────────────────────────┘    │
                    └─────────────────────────────────────┘
```

---

## ⭐ Interview Tips & Key Points to Remember

- **IAM is a GLOBAL service** — no region selection, applies to entire account
- **Root account**: only for account creation and billing — NEVER for day-to-day work
- **Groups cannot contain other groups** — only users (classic trap question)
- **Explicit DENY always wins** — even if another policy allows it
- **Default behavior = implicit DENY** — everything is denied until explicitly allowed
- **IAM Roles use temporary credentials** (via STS) — more secure than long-term access keys
- **Roles vs Users**: Users = long-term credentials for humans; Roles = temporary credentials for services/cross-account
- **Least Privilege** is the foundational security principle — always start with minimum permissions
- **Access Advisor** = find and remove unused permissions (right-size your IAM)
- **Credentials Report** = audit all users and credential status in one CSV download
- **MFA types**: Virtual (Google Authenticator, Authy), Hardware U2F Key (YubiKey), Hardware Fob (Gemalto)
- **Policy Evaluation**: Deny > Allow > Implicit Deny
- **STS = Security Token Service** — the backend service that issues temporary credentials for roles
- **Inline vs Managed Policy**: Managed = reusable, attach to multiple entities; Inline = 1:1 relationship, not recommended for scale
- If exam asks "how to grant EC2 access to S3 securely?" → **IAM Role attached to EC2** (not access keys!)

---

## Quick Reference — AWS CLI Commands

### User Management
```bash
# Create a user
aws iam create-user --user-name prashant

# List all users
aws iam list-users

# Delete a user
aws iam delete-user --user-name prashant
```

### Group Management
```bash
# Create a group
aws iam create-group --group-name Developers

# Add user to group
aws iam add-user-to-group --user-name prashant --group-name Developers

# Remove user from group
aws iam remove-user-from-group --user-name prashant --group-name Developers

# List users in group
aws iam get-group --group-name Developers
```

### Policy Management
```bash
# Attach managed policy to group
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create inline policy for user
aws iam put-user-policy \
  --user-name prashant \
  --policy-name S3ReadPolicy \
  --policy-document file://s3-read-policy.json

# List attached policies on a group
aws iam list-attached-group-policies --group-name Developers

# Create a customer-managed policy
aws iam create-policy \
  --policy-name MyCustomPolicy \
  --policy-document file://my-policy.json
```

### Role Management
```bash
# Create a role (with trust policy file)
aws iam create-role \
  --role-name EC2-S3-Role \
  --assume-role-policy-document file://ec2-trust-policy.json

# Attach policy to role
aws iam attach-role-policy \
  --role-name EC2-S3-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Assume a role (STS)
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/EC2-S3-Role \
  --role-session-name MySession
```

### Access Keys & MFA
```bash
# Create access key for user (programmatic access)
aws iam create-access-key --user-name prashant

# Create virtual MFA device
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name prashant-mfa \
  --outfile /tmp/mfa-qr.png \
  --bootstrap-method QRCodePNG

# Enable MFA for user
aws iam enable-mfa-device \
  --user-name prashant \
  --serial-number arn:aws:iam::123456789012:mfa/prashant-mfa \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

### Security & Auditing
```bash
# Get current caller identity (who am I?)
aws sts get-caller-identity

# Generate credential report
aws iam generate-credential-report
aws iam get-credential-report --output text --query Content | base64 -d

# Simulate IAM policy (check if action is allowed)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/prashant \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/my-object
```

