# SAP Section 3: Identity & Federation
> AWS Solutions Architect Professional — Expert Course Order  
> Depth Level: SAP (enterprise patterns, policy evaluation logic, cross-account design)

---

## Section Overview

Identity and Federation is the backbone of every enterprise AWS deployment. At the SAP level, you are no longer just asked "what is IAM?" — you are asked to design identity architectures for 500-employee companies migrating to AWS, explain exactly why an explicit Deny in an SCP overrides a role's Allow, or debug why a cross-account Lambda can't write to S3 despite having the right IAM role.

This section follows the expert course order exactly. Read every topic understanding not just the WHAT, but the WHY — why does AWS enforce SCP boundaries above the account root? Why does Cognito separate User Pools from Identity Pools? Why does STS need an ExternalId for third-party access?

The mental model for this section: **Identity is a series of gates, each gate has different rules, and the access decision is the product of all gates.**

---

## 1. IAM — Deep Dive for SAP

### The Foundational Problem IAM Solves

AWS starts with one account, one root user with infinite permissions. In any real organization, you need:
- Different people with different levels of access
- Services that need to call other services without embedding credentials
- External parties that need temporary, scoped access
- Auditable records of who did what

IAM is AWS's answer to all of these. But at the SAP level, the nuance is in **how permission decisions are made**, not just that permissions exist.

### IAM Entities: Users, Groups, Roles

**Users**: Long-term credentials for humans or applications. Users have permanent Access Key IDs and Secret Access Keys (unless you rotate them). In enterprise settings, you should have almost no IAM users — instead, federate from your corporate identity provider.

**Groups**: Organizational collections of users. Groups can ONLY contain users, never other groups. This is a deliberate simplicity constraint — AWS does not support nested group membership. If you need "all Developers and Managers can read S3," you create one group with both sets of users in it.

**Roles**: Temporary credential mechanism for services, cross-account access, and federation. A role is an identity with no static credentials — instead, STS issues short-lived tokens when someone "assumes" the role. This is the gold standard for granting access: no long-lived credentials to leak or rotate.

### Policy Types and When to Use Each

IAM policies come in multiple types, and SAP scenarios often test whether you choose the right type for the situation:

| Policy Type | Who/What Controls It | Scope | Use Case |
|---|---|---|---|
| **Identity Policy** | Attached to IAM user, group, or role | Defines what the principal CAN do | Grant S3 access to a role |
| **Resource Policy** | Attached to the resource (S3, KMS, SQS, etc.) | Defines who CAN access the resource | Allow cross-account S3 access without role assumption |
| **Permission Boundary** | Attached to a user or role | Sets the MAX permissions possible | Limit what a developer-created role can ever do |
| **SCP (Service Control Policy)** | Managed by AWS Organizations | Max permissions for the entire account | Prevent any account from using regions outside eu-west-1 |
| **Session Policy** | Passed at AssumeRole call time | Max permissions for this session only | Restrict a session to read-only even if role allows read/write |
| **Inline Policy** | Embedded inside user/role/group | 1:1 relationship, deleted with principal | One-off emergency access (avoid in general) |

### Permission Boundaries — The "Sandbox" Pattern

Permission boundaries are one of the most SAP-specific IAM concepts. The analogy: imagine your VP gives you a company credit card with a $5,000 limit and says "buy what you need for the project." Your purchasing decision still needs your manager's approval, but the VP has capped the maximum you could ever spend at $5,000 regardless of what your manager approves.

Permission boundaries work the same way: **the effective permissions of a principal are the intersection of their identity policies AND their permission boundary.** Even if the identity policy says "Allow S3:*", if the permission boundary only allows S3:GetObject, the principal can only GetObject.

**When to use**: Enterprise pattern for delegated administration. You want to let your platform team create IAM roles for developers, but you don't want platform team members to create all-powerful roles that exceed their own permissions. Attach a permission boundary to any role they create — this ensures they can't grant more than what you've bounded.

```
Effective Permission = Identity Policy ∩ Permission Boundary ∩ SCP
```

If any layer says "no," the answer is no.

### Policy Evaluation Logic — Complete Order

This is critical for SAP. AWS evaluates policies in a specific order, and knowing this order lets you debug complex multi-account, multi-policy scenarios:

```
1. Explicit DENY in any policy?         → DENIED (stop immediately)
2. Is there an applicable SCP?          → If SCP does not allow: DENIED
3. Is there a Resource-based policy?    → If it allows: ALLOWED (can skip role assumption for same-account)
4. Is there a Permission Boundary?      → If boundary does not allow: DENIED
5. Is there a Session Policy?           → If session policy does not allow: DENIED
6. Is there an Identity Policy Allow?   → ALLOWED
7. No match found?                      → Implicit DENY (default)
```

**The SAP trap**: Explicit Deny in any attached policy — even a managed policy you forgot about — always wins. A single `Deny` statement beats every `Allow` everywhere.

**Cross-account nuance**: In cross-account scenarios, BOTH the caller's identity policy AND the resource's resource policy must allow the action (if using resource policies), OR the caller must assume a role in the target account and the role's identity policy must allow the action.

### Condition Keys That Appear on SAP Exams

SAP scenarios frequently involve conditions. Know these:

| Condition Key | Example Use |
|---|---|
| `aws:RequestedRegion` | Restrict actions to specific regions |
| `aws:SourceIp` | Allow only from company IP range |
| `aws:SourceVpc` | Allow only from specific VPC |
| `aws:PrincipalOrgID` | Require caller to be in your AWS Organization |
| `aws:MultiFactorAuthPresent` | Require MFA for sensitive actions |
| `aws:PrincipalTag` | ABAC — allow based on principal's IAM tag |
| `aws:ResourceTag` | ABAC — allow based on resource's tag |
| `sts:ExternalId` | Prevent confused deputy in third-party access |
| `sts:TokenIssueTime` | Revoke old sessions |

**ABAC (Attribute-Based Access Control)**: Instead of creating one policy per team, you tag resources (e.g., `Team=FinanceTeam`) and tag users (e.g., `Team=FinanceTeam`), then write one policy: "Allow access if `aws:PrincipalTag/Team` matches `aws:ResourceTag/Team`." This scales to thousands of users without managing individual policies.

🎯 **EXAM TIP**: Permission boundaries do NOT grant permissions — they only restrict them. If a permission boundary allows S3:* but the identity policy has no S3 statement, the result is DENY. Both must Allow.

🎯 **EXAM TIP**: Resource-based policies (S3 bucket policy, KMS key policy) can grant cross-account access WITHOUT role assumption — but only if both the resource policy and the caller's identity policy allow it. If only the resource policy allows it but the caller has no identity policy allowing S3, the access is still denied (for IAM principals — not for principals already in the resource's account, where resource policy alone suffices).

🎯 **EXAM TIP**: SCPs affect even the AWS account root user. This is unique — normally root cannot be restricted. But when an SCP in Organizations denies an action, root in that account is also denied.

---

## 2. IAM Access Analyzer

### Why It Exists

The problem: You've deployed 200 IAM roles, 50 S3 bucket policies, 30 KMS key policies. Which of these accidentally expose resources to the public internet? Which S3 bucket policies grant cross-account access to an external AWS account you don't recognize? Manual auditing is impossible at scale.

IAM Access Analyzer uses automated reasoning (formal mathematical proofs) to analyze resource policies and find **unintended external access**. It's not heuristic — it's provable. If it says a resource is not externally accessible, it's mathematically verified.

### How It Works

Access Analyzer creates an **Analyzer** scoped to your AWS Organization or account. It continuously monitors:
- S3 bucket policies and ACLs
- IAM roles (trust policies)
- KMS key policies
- SQS queue policies
- Lambda function policies
- Secrets Manager secrets
- SNS topics

When it finds a resource accessible from outside your defined **zone of trust** (your organization or account), it generates a **Finding**.

### Findings and Archive Rules

Each finding reports:
- Which resource is exposed
- Which external principal has access
- What actions they can perform
- Whether the access is public or cross-account

You can **archive** known-good findings (e.g., an S3 bucket that intentionally serves public content). You can create **archive rules** to automatically archive findings matching certain patterns.

### Policy Generation

A lesser-known but SAP-relevant feature: Access Analyzer can **generate an IAM policy** based on CloudTrail activity logs. The workflow:
1. Run your application with overly permissive permissions for 30 days
2. Access Analyzer analyzes CloudTrail to see what API calls were actually made
3. It generates a least-privilege policy covering exactly those calls

This is the automated path to least-privilege without manually cataloging every API call.

### IAM Access Analyzer vs. Trusted Advisor

| | IAM Access Analyzer | Trusted Advisor |
|---|---|---|
| Focus | External resource access | Cost, performance, security, fault tolerance |
| Mechanism | Formal reasoning on policies | Rule-based checks |
| Depth | Deep policy analysis | Broad surface-level checks |
| Findings | Specific resource/principal pairs | High-level recommendations |

🎯 **EXAM TIP**: If the scenario asks "how to find S3 buckets accidentally exposed to the public" or "audit cross-account access" — IAM Access Analyzer. If it asks about general security best practices across cost, performance, and security — Trusted Advisor.

---

## 3. STS — Security Token Service

### Why Temporary Credentials Matter

Permanent access keys are a liability: they can be leaked in code commits, stolen from configuration files, or captured in logs. Temporary credentials expire automatically — even if stolen, they're useless after the expiration time (minimum 15 minutes, maximum 12 hours for role assumption, up to 36 hours for some federation flows).

STS is the engine that issues these temporary credentials. Every IAM role, every federation flow, every Cognito identity ultimately calls STS under the hood.

### STS API Reference — Key Operations

#### AssumeRole

The most common STS operation. A principal (user, role, service) requests temporary credentials for another role.

**Requirements for success**:
1. The calling principal must have an identity policy allowing `sts:AssumeRole` on the target role's ARN
2. The target role's **trust policy** must allow the calling principal as a trusted entity

**Trust policy** is the key concept here. A trust policy is like a resource policy attached to the role itself — it declares "who is allowed to assume me." If the trust policy doesn't list you, you cannot assume the role even if your identity policy says you can.

**Cross-account AssumeRole**:
```
Account A (Caller) → Account B (Role)
  - Account A user/role needs sts:AssumeRole permission (in Account A's IAM)
  - Account B role trust policy must explicitly trust Account A's principal
  - Account B role's identity policies define what the session can do in Account B
```

**Role chaining**: You can assume Role A, then from that session assume Role B. However, the maximum session duration for chained assumptions is capped at 1 hour (even if the role normally allows longer). This is a deliberate security constraint.

#### AssumeRoleWithSAML

Used for enterprise federation via SAML 2.0. A user authenticates with your corporate IdP (Active Directory, Okta, etc.), receives a SAML assertion (signed XML), and exchanges that assertion for AWS temporary credentials.

**Flow in detail**:
1. User accesses the AWS login portal or your custom app
2. Browser redirected to IdP (e.g., ADFS, Okta)
3. User authenticates with corporate credentials (username/password + MFA)
4. IdP generates SAML assertion (XML document signed with IdP's private key)
5. Browser POSTs assertion to `https://signin.aws.amazon.com/saml`
6. AWS validates assertion signature against registered IdP metadata
7. AWS calls STS `AssumeRoleWithSAML` with the assertion and desired role ARN
8. STS issues temporary credentials (up to 12 hours)
9. User gets console access or API credentials

**Key requirement**: You must register the IdP in AWS IAM as a SAML provider, providing the IdP's metadata XML (contains the public key to verify assertion signatures).

#### AssumeRoleWithWebIdentity

For web and mobile applications with social login (Google, Facebook, Amazon) or any OIDC-compliant IdP.

**Flow**:
1. User signs in with Google → receives Google JWT (ID token)
2. App calls STS `AssumeRoleWithWebIdentity` with the Google token and role ARN
3. STS validates token with Google's JWKS endpoint
4. STS issues temporary AWS credentials

**Modern recommendation**: Use Cognito Identity Pools instead of calling this API directly. Cognito adds user management, multiple IdP support, and unauthenticated identity support on top of this raw STS call.

#### GetSessionToken

Issues temporary credentials that inherit the calling user's permissions, but can enforce MFA for the session.

**Use case**: An IAM user wants to perform sensitive operations (delete objects, modify security groups) that require MFA validation. They call `GetSessionToken` with their MFA token code, receive temporary credentials that carry the `aws:MultiFactorAuthPresent=true` condition, and use those to perform the sensitive action.

```
User (IAM user, no MFA on session)
  → calls GetSessionToken with MFA token
  → STS verifies MFA code
  → returns temp credentials with MultiFactorAuthPresent=true
  → those credentials can now call s3:DeleteObject (if policy requires MFA)
```

#### GetFederationToken

Issues longer-duration credentials (up to 36 hours) for custom federation proxies. Less common in modern setups — used for custom identity brokers where you build the federation layer yourself.

### The Confused Deputy Problem and ExternalId

**Scenario**: You hire CloudMonitor Corp to monitor your AWS infrastructure. They need to assume a role in your account. You add CloudMonitor's AWS account ID to your role's trust policy. 

**The problem**: CloudMonitor serves 1,000 customers. If any of those customers are malicious, they could trick CloudMonitor's systems into assuming YOUR role by providing your account ID. The malicious customer is the "confused deputy" — CloudMonitor becomes an unwitting proxy doing the attacker's bidding.

**Solution**: Require an `ExternalId` in the trust policy — a shared secret between you and CloudMonitor that no other customer knows. CloudMonitor must pass this exact ExternalId when calling AssumeRole for your account.

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::CLOUDMONITOR_ACCOUNT:root" },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "Xp4dB9kQ2nLrM7jT"
    }
  }
}
```

The ExternalId should be unique per customer, generated by you (the customer) or CloudMonitor, and stored securely. It's not a secret in the cryptographic sense — it's a guard against accidental escalation.

### Revoking Active Sessions

If a role's credentials are compromised, how do you invalidate them before they expire? You can't "delete" STS tokens directly, but you can:

1. Add an inline policy to the role with a Deny on all actions where `aws:TokenIssueTime` is before the current time
2. This effectively invalidates all tokens issued before that moment

This is the incident response pattern for "we suspect someone is using stolen role credentials."

🎯 **EXAM TIP**: Role chaining (assuming a role from an assumed role session) caps the session to 1 hour maximum, regardless of what the role's `MaxSessionDuration` allows.

🎯 **EXAM TIP**: AssumeRoleWithWebIdentity is the underlying API, but Cognito Identity Pools is the recommended pattern for web/mobile apps. If the exam asks about mobile apps accessing DynamoDB with Google login, the answer is Cognito Identity Pools (not raw AssumeRoleWithWebIdentity).

🎯 **EXAM TIP**: GetSessionToken is for existing IAM users who want to enforce MFA on their session. It does NOT create a new identity — it creates temporary credentials with the same permissions as the user, plus MFA-present claim.

---

## 4. Identity Federation & Cognito

### The Core Concept: Federation

Federation solves the identity duplication problem. Without federation, every employee needs an IAM user in every AWS account they use. For a 1,000-person company with 20 AWS accounts, that's potentially 20,000 IAM users to manage. When someone leaves, you need to disable 20 accounts. When someone joins a new team, you add them to N accounts.

Federation says: "Let your existing identity system (corporate AD, Google, Okta) be the source of truth. AWS trusts that system and grants access based on that trust." No IAM users needed.

### SAML 2.0 Federation

SAML (Security Assertion Markup Language) is the enterprise standard for SSO. It's XML-based, verbose, and everywhere in large companies.

**Key parties**:
- **Identity Provider (IdP)**: Your corporate directory (ADFS, Okta, Azure AD, PingFederate)
- **Service Provider (SP)**: AWS (or any other service)
- **Assertion**: The signed XML document proving the user's identity and attributes

**IdP-initiated vs SP-initiated**:
- **SP-initiated**: User goes to AWS Console → gets redirected to IdP → authenticates → gets SAML assertion → returns to AWS with access
- **IdP-initiated**: User logs in to company portal → sees AWS tile → clicks it → IdP sends assertion directly to AWS

**Role mapping in SAML**: Your IdP assertion must contain an attribute that maps to an AWS IAM role ARN. The assertion says "this user should assume role arn:aws:iam::ACCOUNT_ID:role/Admins." AWS validates this and assumes that role on the user's behalf.

**Attribute mapping example**: A SAML assertion can include custom attributes. You can map these to IAM session tags for ABAC:
- SAML attribute `department=Finance` → IAM session tag `department=Finance`
- IAM policy condition: `StringEquals: aws:PrincipalTag/department: Finance`

### Web Identity Federation

For consumer applications (mobile apps, web apps) that use social login.

**Without Cognito** (raw AssumeRoleWithWebIdentity):
- App calls Google login
- Gets Google JWT
- Calls STS AssumeRoleWithWebIdentity with Google JWT
- STS validates with Google
- Returns temp AWS credentials

**With Cognito (recommended)**:
- App calls Cognito
- Cognito handles the Google login, token validation, user management
- Cognito issues its own token
- Cognito exchanges for AWS credentials (via Identity Pool → STS)
- App gets AWS credentials

Cognito is the abstraction layer that adds user management, profile storage, and consistent credential issuance regardless of which social provider was used.

### Custom Identity Broker

For legacy systems that don't speak SAML or OIDC. This is the "build your own" federation approach.

**Architecture**:
1. Custom broker app sits in your network
2. User authenticates to broker using legacy credentials (LDAP, proprietary system)
3. Broker validates credentials against the legacy system
4. Broker calls STS `AssumeRole` or `GetFederationToken` (broker has IAM credentials to do this)
5. Broker returns temporary credentials (or a pre-signed console URL) to the user
6. User accesses AWS with those credentials

**When to use**: When your existing auth system cannot be federated via SAML and you cannot change it. Adds complexity but unlocks access for systems that would otherwise be impossible to integrate.

### Cognito User Pools vs. Identity Pools

This distinction appears in nearly every SAP identity scenario involving web/mobile apps:

| | User Pools | Identity Pools |
|---|---|---|
| **Job** | Authentication — who are you? | Authorization — what AWS resources can you access? |
| **What it returns** | JWT tokens (ID token, access token, refresh token) | Temporary AWS credentials (via STS) |
| **Stores user data?** | Yes — user directory with profiles | No — just maps identities to IAM roles |
| **Social login** | Yes — Google, Facebook, Amazon, Apple | Yes — accepts tokens from User Pools or any OIDC/SAML provider |
| **Unauthenticated** | No | Yes — can issue credentials to guest/unauthenticated users with restricted IAM role |
| **Use case** | "Sign up, sign in to my app" | "My app needs to call S3 or DynamoDB directly" |

**Combined pattern** (the most common SAP scenario):
```
User → Cognito User Pool (authenticate) → JWT token
                                              ↓
                                    Cognito Identity Pool
                                    (exchange JWT for AWS creds)
                                              ↓
                                    AWS credentials (via STS)
                                              ↓
                                    App calls S3, DynamoDB, etc.
```

**Identity Pool role mapping**: Identity Pools can map authenticated vs. unauthenticated users to different IAM roles. You can also create rules — "users with ID token claim `custom:tier=premium` get the PremiumRole; all others get the BasicRole."

🎯 **EXAM TIP**: Cognito User Pools integrate with API Gateway as an authorizer. User Pool JWT tokens are directly validated by API Gateway without the Identity Pool step — because API Gateway is not an AWS SDK service, you don't need AWS credentials to call it.

🎯 **EXAM TIP**: If the scenario says "mobile app users need to read their own S3 folder" — the answer combines User Pool (authentication) + Identity Pool (AWS credentials) + IAM role policy that uses the Cognito identity ID in the resource ARN condition.

🎯 **EXAM TIP**: Unauthenticated (guest) access via Cognito Identity Pools is a legitimate pattern for apps that let users browse content before logging in. Guests get restricted IAM credentials (read-only, limited resources).

---

## 5. AWS Directory Services

### The Problem: Active Directory Everywhere

Most enterprises run Microsoft Active Directory on-premises. It's the source of truth for who employees are, what groups they belong to, and what systems they can access. When moving to AWS, you need AWS services to work with this existing identity system.

AWS offers three options depending on how much you want to "lift" AD into the cloud vs. keep it on-premises.

### AWS Managed Microsoft AD

**What it is**: A fully managed Microsoft Active Directory running in AWS, deployed in two AZs within your VPC.

**Key capabilities**:
- Real Microsoft AD — supports all AD protocols, Group Policy, Kerberos, LDAP, NTLM
- Can establish **trust relationships** with on-premises AD (one-way or bi-directional)
- With trust: users in on-prem AD can access AWS resources without having two AD accounts
- Supports MFA (via RADIUS integration)
- Supports RDS SQL Server, WorkSpaces, QuickSight AD auth, IAM Identity Center

**With trust relationship**:
```
On-premises AD (corp.example.com) ←—trust—→ AWS Managed AD (aws.example.com)

On-prem user john@corp.example.com
  → authenticates to AWS console
  → AWS Managed AD trusts assertion from corp.example.com
  → john gets access based on his on-prem group memberships
```

**When to use**: You want a full AD in AWS, you need to migrate AD-dependent applications to AWS, or you want to extend your on-prem AD with a trust (not replicate it, but federate through trust).

### AD Connector

**What it is**: A gateway that proxies all AD requests from AWS to your on-premises AD. It stores nothing itself.

**Analogy**: AD Connector is like a call forwarding service for your phone — when AWS calls for directory info, AD Connector forwards the call to your on-prem AD and relays the answer back.

**Key characteristics**:
- No users stored in AWS
- On-premises AD must always be reachable (via Direct Connect or VPN) — if the connection goes down, authentication fails
- Lower cost than AWS Managed AD (no compute for directory)
- No trust relationship needed — AWS just talks through the connector to your real AD

**When to use**: Your company policy requires all user data to stay on-premises. You don't want to replicate or sync AD to AWS. You just want AWS services to authenticate against your existing on-prem AD.

**Limitation**: No caching. Every auth request goes on-premises in real time. High-latency connections will cause slow logins.

### Simple AD

**What it is**: A Samba-based LDAP directory managed by AWS. It's AD-compatible for basic use cases but is not actual Microsoft AD.

**Capabilities and limits**:
- Supports basic LDAP and Kerberos
- Does NOT support trust relationships with Microsoft AD
- Does NOT support MFA
- Does NOT support RDS SQL Server AD authentication
- Maximum 5,000 users (small tier: 500 users)
- Cheapest option

**When to use**: Greenfield deployment with no existing on-prem AD. Small team (<5,000 users). You need basic LDAP/Kerberos for Linux instances or basic AWS service integration. Budget-constrained.

### Decision Framework: Which Directory Service?

| Scenario | Answer |
|---|---|
| Full AD in AWS, migrate AD-dependent apps | AWS Managed Microsoft AD |
| Extend on-prem AD to AWS with trust | AWS Managed Microsoft AD |
| On-prem AD stays on-prem, just proxy requests | AD Connector |
| New directory, <5,000 users, simple needs | Simple AD |
| Need RDS SQL Server AD auth | AWS Managed Microsoft AD (NOT Simple AD) |
| On-prem AD must remain authoritative, no replication | AD Connector |
| Disconnected from internet / Direct Connect outage tolerance needed | AWS Managed Microsoft AD (with trust, failover to local) |

🎯 **EXAM TIP**: Simple AD does NOT support trust relationships with on-premises Microsoft AD. If the scenario mentions an existing corporate AD, Simple AD is wrong.

🎯 **EXAM TIP**: AD Connector is pass-through only — if the on-premises AD is unreachable, authentication fails completely. There's no local caching or fallback.

---

## 6. AWS Organizations

### Why Multi-Account?

The single-account approach is fine for individuals and small projects. But in enterprises, everything in one account means:
- A developer's mistake in staging can affect production (same account = same blast radius)
- One billing invoice for everything — no departmental cost visibility
- One set of IAM policies to govern everyone — becomes unmanageable
- Compliance requirements (e.g., PCI scope cannot include your dev tools account)

AWS Organizations solves this by providing a **hierarchy of accounts** with centralized governance.

### Organization Structure

```
Root (Management Account)
├── Security OU
│   ├── Security-Tooling Account (GuardDuty master, Config aggregator)
│   └── Log-Archive Account (centralized CloudTrail, Config logs)
├── Infrastructure OU
│   ├── Network Account (Transit Gateway, Route53, shared VPCs)
│   └── Shared-Services Account (Docker registry, AMI pipeline)
├── Sandbox OU
│   └── Individual developer accounts (auto-provisioned)
├── Workloads OU
│   ├── Prod OU
│   │   ├── Prod-App-A Account
│   │   └── Prod-App-B Account
│   └── Dev OU
│       ├── Dev-App-A Account
│       └── Dev-App-B Account
└── Management Account (billing, Organizations management only)
```

**Management Account**: The account that owns the Organization. It cannot be restricted by SCPs (this is a gotcha — the management account is exempt from SCPs it creates). Best practice: use the management account ONLY for Organizations management and billing — deploy nothing in it.

**Organizational Units (OUs)**: Hierarchical groupings. An account can only be in one OU at a time (not multiple OUs). Accounts inherit all policies from every OU above them in the tree.

### Consolidated Billing

All accounts in the Organization roll up their costs to the management account's payment method. Benefits:
- Single invoice across all accounts
- **Volume discounts aggregate**: if you buy 1,000 EC2 hours across 10 accounts, you get the 1,000-hour discount tier (not 10 × 100-hour tier)
- **Reserved Instances and Savings Plans share**: an RI purchased in Account A automatically covers matching usage in Account B if Account A doesn't need it
- Cost allocation tags propagate for breakdown by account/project/team

🎯 **EXAM TIP**: The management account is NOT subject to SCPs. If a question asks "how do you prevent the management account from doing X," the answer is "you can't via SCPs — it's a best practice to do nothing in the management account."

🎯 **EXAM TIP**: Reserved Instances and Savings Plans share across the Organization via consolidated billing. If an account with an RI is producing less usage than covered, other accounts automatically get the discount rate for the uncovered capacity.

---

## 7. AWS Organizations Policies — SCPs

### What SCPs Are (and Are Not)

Service Control Policies (SCPs) are **guardrails**, not grants. This is the most important distinction:
- SCPs do NOT grant permissions
- SCPs define the maximum possible permissions that any principal in the account (including root) can have
- A permission must be allowed by BOTH the SCP and the identity policy to take effect

**Analogy**: The SCP is the walls of a room. Your IAM policies are where you choose to stand inside the room. If the SCP wall cuts off a corner of the room, your IAM policy can't put you there — even if it says you can go anywhere.

### SCP Inheritance Rules

SCPs cascade down the OU hierarchy. An account inherits SCPs from:
1. The root (applies to all accounts in the org)
2. Every OU in the path from root to the account's OU
3. The account itself (SCPs can be attached directly to accounts)

**All SCPs must ALLOW the action** for it to be permitted. If any SCP in the chain denies it (or doesn't allow it, in allowlist mode), the action is blocked.

**Example inheritance**:
```
Root SCP: Deny services outside us-east-1 and eu-west-1
├── Prod OU SCP: Deny cloudtrail:StopLogging
│   └── Prod-App Account:
│       EFFECTIVE: Cannot use services outside 2 regions AND cannot stop CloudTrail
└── Dev OU SCP: Allow all (no restrictions beyond root)
    └── Dev-App Account:
        EFFECTIVE: Cannot use services outside 2 regions (inherits root SCP)
```

### SCP Strategy: Denylist vs. Allowlist

**Denylist approach** (default, most common):
- Start with AWS-managed `FullAWSAccess` SCP on all OUs (allows everything)
- Add explicit Deny statements for what you want to block
- Pro: easy to get started, explicit about what's blocked
- Con: you need to know what to block; new AWS services are accessible by default

**Allowlist approach**:
- Remove `FullAWSAccess`
- Add Allow statements only for services you explicitly permit
- Pro: zero trust — new services are blocked until explicitly approved
- Con: operational overhead, easy to accidentally block something needed

**SAP recommendation**: Allowlist for the most sensitive OUs (Security, Management), denylist for general Workloads OU.

### Common SCP Patterns

| Goal | SCP Approach |
|---|---|
| Restrict to specific AWS regions | `Deny` with `Condition: StringNotEquals aws:RequestedRegion` |
| Prevent leaving the organization | `Deny organizations:LeaveOrganization` |
| Enforce CloudTrail | `Deny cloudtrail:StopLogging, cloudtrail:DeleteTrail` |
| Block public S3 buckets | `Deny s3:PutBucketPublicAccessBlock` with condition allowing only blocking |
| Prevent root account usage | `Deny *` with `Condition: StringLike aws:PrincipalArn *root*` |
| Enforce encryption on EBS | `Deny ec2:RunInstances` with condition unless `ec2:Encrypted=true` |
| Prevent IAM role creation without boundary | `Deny iam:CreateRole` unless `iam:PermissionsBoundary` is specified |

### SCP vs. Resource Policy vs. Identity Policy

This comparison appears on SAP exams regularly:

| | SCP | Resource Policy | Identity Policy |
|---|---|---|---|
| **Attached to** | Organization/OU/Account | Resource (S3, KMS, SQS) | IAM principal |
| **Purpose** | Cap maximum permissions in account | Control who accesses this resource | Grant permissions to this principal |
| **Grants permissions?** | NO | YES | YES |
| **Cross-account** | Controls all accounts under org | Can grant cross-account access | Controls what caller can do |
| **Affects root?** | YES (unique to SCPs) | No | No |
| **Region scope** | Global (when org-level) | Per-resource | Global (IAM is global) |

🎯 **EXAM TIP**: SCPs do not affect the management account. They also do NOT affect service-linked roles. If a service-linked role needs to perform an action, SCPs won't block it.

🎯 **EXAM TIP**: When calculating effective permissions in a member account: Effective = (Identity Policies) ∩ (All SCPs in the inheritance chain). If any SCP doesn't allow the action, it's blocked — even if the identity policy is AdministratorAccess.

---

## 8. AWS IAM Identity Center (formerly AWS SSO)

### The Problem It Solves

Without IAM Identity Center, a developer with access to 10 AWS accounts needs to:
- Maintain 10 sets of IAM credentials (or role switch 10 times)
- Log in to each account separately
- Have an admin manually create and manage their access in each account

IAM Identity Center provides **one login, one place to manage access, centralized across all accounts and cloud apps.**

### Architecture Components

**Identity Source** (where users come from — pick one):
- Built-in Identity Center directory (manage users directly in IAM Identity Center)
- AWS Managed Microsoft AD (connect to your AD in AWS)
- External IdP via SAML 2.0 or OIDC (Okta, Azure AD, on-prem ADFS)

**Permission Sets** (what users can do — the access template):
- A permission set is essentially a role definition
- Contains IAM policies (AWS managed or customer managed)
- When assigned to a user for an account, IAM Identity Center creates an actual IAM role in that account matching the permission set
- Support for IAM policy conditions, permission boundaries

**Account Assignments** (connecting users/groups to accounts via permission sets):
- The triple: User/Group + Permission Set + Target Account
- Can be assigned at OU level (assign once, applies to all accounts in OU, including future accounts)

### How Access Works at Runtime

1. User navigates to the IAM Identity Center portal (or company SSO portal)
2. Authenticates (with IdP credentials + MFA if configured)
3. Sees a list of accounts and roles they have access to
4. Clicks an account/role combination
5. Identity Center calls STS to assume the corresponding role in that account
6. User gets temporary credentials (CLI via `aws sso login`) or console access

**SCIM Provisioning**: When using an external IdP, SCIM (System for Cross-domain Identity Management) automatically syncs users and groups from the IdP to IAM Identity Center. When a user is deprovisioned in Okta, they're automatically removed from Identity Center — no manual cleanup.

### IAM Identity Center vs. SAML 2.0 Federation Direct

| | IAM Identity Center | SAML 2.0 Direct |
|---|---|---|
| **Setup complexity** | Low — wizard-based | Higher — IdP configuration, metadata exchange |
| **Multi-account** | Built-in | Manual per account |
| **Permission management** | Centralized permission sets | Distributed per account |
| **SCIM sync** | Supported | Not applicable |
| **Attribute-based access** | ABAC with Identity Center tags | Via SAML attribute mappings |
| **Audit** | Centralized CloudTrail | Per-account CloudTrail |
| **Best for** | New AWS Organizations setups | Existing SAML infrastructure that must remain |

🎯 **EXAM TIP**: IAM Identity Center is the recommended approach for AWS Organizations multi-account SSO. For SAP, if a question describes "org-wide SSO, manage access centrally, sync from Okta" — IAM Identity Center with SCIM provisioning.

🎯 **EXAM TIP**: Permission sets are templates that become IAM roles in each target account. IAM Identity Center creates these roles automatically — you don't manage them in each account's IAM console.

🎯 **EXAM TIP**: ABAC in IAM Identity Center: tags on users/groups in Identity Center can be passed as session tags when assuming roles. The IAM role's policy can use `aws:PrincipalTag` conditions — so `Department=Finance` users automatically get scoped access to Finance resources without custom permission sets per department.

---

## 9. AWS Control Tower

### Why It Exists

Setting up a multi-account AWS environment correctly is hard. You need to:
- Set up AWS Organizations
- Create the right OU structure
- Deploy CloudTrail in every account
- Enable AWS Config everywhere
- Set up centralized logging
- Apply baseline SCPs
- Configure SSO

Do this manually for the first account: 2-3 days of work. Now provision 50 accounts this way: months. And it won't be consistent.

Control Tower automates this entire landing zone setup and provides ongoing governance.

### Landing Zone

The **Landing Zone** is the well-architected baseline environment Control Tower creates:

**Foundational OUs** (created automatically):
- **Security OU**: Contains Log Archive account and Security Tooling (Audit) account
- **Sandbox OU**: For developers to experiment freely (with guardrails)

**Mandatory accounts**:
- **Management Account**: Owns Control Tower, Organizations management
- **Log Archive Account**: Centralized immutable logging (CloudTrail, Config)
- **Security Tooling (Audit) Account**: Security team account with read-only access across org

**Baseline services deployed in every account**:
- AWS CloudTrail (organization trail)
- AWS Config (recording all resource changes)
- CloudWatch Logs centralization
- SNS notifications for guardrail violations

### Account Factory

The **Account Vending Machine** for enterprise AWS:
1. Developer submits a self-service request (via Service Catalog product)
2. Specifies account name, email, OU placement, optional parameters
3. Control Tower automatically provisions the account with full baseline configuration
4. Account is ready to use in minutes, fully compliant from day one

Account Factory can be configured with Account Factory for Terraform (AFT) for IaC-driven account provisioning.

### Guardrails

Guardrails are governance rules that apply across your entire Control Tower environment:

**Mandatory Guardrails** (always on, cannot disable):
- Disallow changes to CloudTrail configuration
- Disallow deletion of Log Archive S3 buckets
- Disallow public write access to Log Archive S3

**Strongly Recommended Guardrails** (AWS recommends enabling):
- Enable encryption at rest for EBS volumes
- Disallow public S3 buckets

**Elective Guardrails** (optional, use case specific):
- Restrict regions to specific subset
- Disallow root account access

**Preventive Guardrails** (implemented via SCPs):
- Block the action before it happens
- "You cannot make this API call"

**Detective Guardrails** (implemented via AWS Config rules):
- Find violations that already exist
- "This resource is non-compliant"
- Trigger remediation (manually or via Lambda auto-remediation)

### Control Tower Dashboard

Provides organization-wide compliance visibility:
- Which accounts violate which guardrails
- Aggregate Config rule compliance
- Account enrollment status
- Drift detection (if someone modifies a Control Tower-managed resource manually)

### Control Tower vs. Organizations + SCPs (DIY)

| | Control Tower | DIY Organizations + SCPs |
|---|---|---|
| **Setup** | Automated, opinionated, fast | Manual, flexible, time-consuming |
| **Baseline config** | CloudTrail, Config, SSO auto-deployed | Manual deployment in each account |
| **Account vending** | Built-in (Account Factory) | Custom automation required |
| **Guardrails** | Pre-built library | Write your own SCPs + Config rules |
| **Compliance dashboard** | Built-in | Custom CloudWatch/Security Hub dashboards |
| **Best for** | Most enterprise use cases, greenfield | When Control Tower's opinions don't fit your structure |

🎯 **EXAM TIP**: Preventive guardrails use SCPs (they PREVENT the action). Detective guardrails use AWS Config rules (they DETECT violations after the fact). Know the difference — an exam scenario about "preventing" a misconfiguration → preventive/SCP; "alerting on" or "finding" a misconfiguration → detective/Config.

🎯 **EXAM TIP**: Control Tower uses Account Factory (backed by Service Catalog) for self-service account provisioning. If a scenario describes "automated, compliant account creation with baseline security controls" — Control Tower with Account Factory.

---

## 10. AWS Resource Access Manager (RAM)

### The Problem Without RAM

You have a hub-and-spoke VPC design: one shared Network account owns the Transit Gateway and shared subnets. Application teams in separate accounts need to use these shared network resources.

Without RAM, options are:
1. Create separate VPCs in each account and use VPC peering (complex routing, manual setup)
2. Deploy network infrastructure in every account (expensive, management nightmare)

With RAM: the Network account creates the resources once and shares them directly with other accounts.

### How RAM Works

1. **Resource Owner** (e.g., Network account) creates a Resource Share
2. Owner specifies which resources to share (VPC subnets, Transit Gateway, etc.)
3. Owner specifies which principals can access the share:
   - Specific AWS account IDs
   - Specific IAM roles or users
   - An entire AWS Organizations OU
   - The entire Organization
4. If sharing with an external account (outside org): the recipient must accept the share invitation
5. If sharing within an Organization with `EnableSharingWithAwsOrganization` turned on: shares are automatically accepted — no invitation needed

### Shared Resource Behavior

The key access model: **the owner manages the resource, participants just use it.**

For shared VPC subnets:
- Owner account creates, modifies, and deletes the subnet
- Participant accounts can launch EC2 instances, RDS instances, Lambda functions into the subnet
- Participant accounts cannot modify or delete the subnet itself
- Resources from different accounts appear in the same subnet and can communicate privately (no NAT, no peering needed)
- Security Groups from the owner account work across all accounts; participant accounts can create their own SGs within the shared subnet

### Commonly Shared Resources

| Resource | Key Use Case |
|---|---|
| VPC Subnets | Shared network architecture across accounts |
| Transit Gateway | Centrally managed inter-account routing |
| Route 53 Resolver Rules | Centralized DNS resolution rules |
| EC2 Capacity Reservations | Reserve capacity in one account, share across org |
| EC2 Dedicated Hosts | Licensing compliance with dedicated hardware |
| AWS License Manager Configurations | Centralized license enforcement |
| Aurora DB Clusters | Share Aurora cluster across accounts |
| Glue Data Catalog Databases | Share data catalog entries |

### RAM vs. VPC Peering vs. VPC Sharing

| | RAM (Shared Subnets) | VPC Peering | Transit Gateway |
|---|---|---|---|
| **Complexity** | Low — one resource share | Medium — per-pair connection | High — centralized routing |
| **Transitive routing** | N/A (same subnet) | No | Yes |
| **Resource ownership** | Owner account | Each account owns VPC | Central TGW account |
| **Use case** | Shared subnet architecture | Simple A↔B connectivity | Hub-and-spoke, many VPCs |
| **Cross-account discovery** | Same subnet, private IP | Route tables, VPC endpoints | TGW route tables |

🎯 **EXAM TIP**: When RAM shares within an Organization (and org sharing is enabled), invitations are automatically accepted. External account sharing requires manual acceptance. SAP scenarios about automating cross-account access within an org → RAM with org sharing enabled.

🎯 **EXAM TIP**: Participant accounts using a shared subnet cannot see resources in that subnet that belong to other participant accounts — they can only see their own. The owner can see all. Communication between accounts still requires routing (Security Groups referencing each other's resources, etc.).

---

## 11. Decision Framework — Identity & Federation

### Primary Question: Where Are Your Users?

```
Where are your users/identities?
├── Corporate AD/LDAP on-premises or in the cloud
│   ├── Need to extend to AWS with full AD features → AWS Managed Microsoft AD
│   ├── Want on-prem AD to stay authoritative, proxy requests → AD Connector
│   └── Want organization-wide SSO, multi-account access → IAM Identity Center (connect to your AD)
├── External IdP (Okta, Azure AD, Google Workspace)
│   ├── Enterprise employees needing AWS Console/CLI access → IAM Identity Center (SAML/OIDC integration)
│   └── App users needing AWS API access → Cognito Identity Pools (with SAML/OIDC IdP)
├── Consumer apps (Google, Facebook social login)
│   └── Mobile/web app needing AWS services → Cognito User Pools + Identity Pools
├── AWS IAM users in another account
│   └── Cross-account access → STS AssumeRole (with trust policy)
└── Completely new directory, small team
    └── Simple AD (< 5,000 users, no on-prem AD)
```

### Service Selection Table

| Scenario | Primary Service | Key Supporting Services |
|---|---|---|
| Org-wide SSO for employees | IAM Identity Center | AWS Organizations, IdP (Okta/ADFS/Azure AD) |
| Enterprise federation (employees → AWS Console) | SAML 2.0 via IAM Identity Center OR direct SAML | STS AssumeRoleWithSAML |
| Mobile app with social login → DynamoDB | Cognito User Pools + Identity Pools | STS (via Identity Pools) |
| App authenticates users, calls your API | Cognito User Pools | API Gateway (JWT authorizer) |
| On-prem AD, extend to AWS | AWS Managed Microsoft AD | AD Trust, WorkSpaces, RDS |
| On-prem AD, proxy only | AD Connector | WorkSpaces, RDS |
| Cross-account role access | IAM Roles + STS AssumeRole | Trust policies |
| Third-party access without confusion | STS AssumeRole + ExternalId | Role trust policy |
| MFA enforcement on programmatic calls | GetSessionToken | IAM policy condition |
| Multi-account governance + compliance | AWS Control Tower | Organizations, SCPs, Config |
| Centralized account provisioning | Control Tower Account Factory | Service Catalog, AFT |
| Share subnets across accounts | RAM | VPC, Organizations |
| Share Transit Gateway across accounts | RAM | TGW, Organizations |
| Audit external resource access | IAM Access Analyzer | CloudTrail |
| Generate least-privilege policies | IAM Access Analyzer (policy generation) | CloudTrail |

### Policy Type Decision Tree

```
What are you trying to control?
├── Max permissions for an entire AWS account in your org → SCP
├── Max permissions for a specific IAM user or role → Permission Boundary
├── What this IAM principal can do → Identity Policy (user/role policy)
├── Who can access this specific resource → Resource Policy (S3, KMS, SQS, Lambda)
├── Restrict a specific STS session → Session Policy (passed at AssumeRole call)
└── One-off permissions embedded in one principal → Inline Policy (avoid)
```

### Cross-Account Access Pattern Selector

| Goal | Pattern |
|---|---|
| Account A user accesses Account B resource, needs role assumption | Account B IAM role with trust policy for Account A |
| Account A user accesses Account B S3 bucket without role assumption | S3 bucket policy in Account B allowing Account A principal |
| All accounts in org can access Account B S3 bucket | S3 bucket policy with `aws:PrincipalOrgID` condition |
| Third-party SaaS accesses your account | Cross-account role with ExternalId in trust policy |
| Centralized governance without per-account role setup | RAM (for shareable resource types) |

### STS API Quick Reference

| API | When | Max Duration |
|---|---|---|
| `AssumeRole` | Cross-account or same-account role assumption | 12 hours (1 hour if role chained) |
| `AssumeRoleWithSAML` | SAML enterprise federation | 12 hours |
| `AssumeRoleWithWebIdentity` | OIDC/social login token exchange | 12 hours |
| `GetSessionToken` | Add MFA to IAM user's programmatic session | 36 hours |
| `GetFederationToken` | Custom identity broker (legacy) | 36 hours |

### Quick Distinguishers for Exam Day

- **SCP vs Permission Boundary**: SCP = account/OU level (Organizations); Permission Boundary = individual user/role level
- **SCP vs IAM Policy**: SCP restricts max possible; IAM policy grants actual permissions; both must allow
- **AD Connector vs Managed AD**: Connector = proxy (on-prem stays authoritative); Managed = real AD in AWS
- **User Pool vs Identity Pool**: User Pool = authentication (JWT tokens); Identity Pool = authorization (AWS credentials)
- **IAM Identity Center vs SAML Direct**: Identity Center = managed, multi-account, SCIM sync; SAML direct = custom, single IdP, manual per-account
- **RAM vs VPC Peering**: RAM = share existing resources (subnet, TGW); VPC Peering = connect two VPCs with routing
- **Control Tower vs Organizations**: Organizations = the tool; Control Tower = the opinionated setup wizard + ongoing governance built on Organizations
- **IAM Access Analyzer vs Trusted Advisor**: Access Analyzer = formal proof of external access; Trusted Advisor = broad best-practice checks

---

*End of SAP Section 3: Identity & Federation*  
*Next Section: SAP-04 — Security (KMS, CloudHSM, Shield, WAF, Macie, Inspector, GuardDuty)*
