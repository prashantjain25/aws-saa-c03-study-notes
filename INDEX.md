## AWS SAA-C03 Study Guide — Complete Index

## Overview

Comprehensive AWS Solutions Architect Associate (SAA-C03) study notes in expert tutor style: explains the WHY not just the WHAT, real-world analogies, AWS CLI commands, architecture diagrams, and exam tips throughout.

**Total Files**: 21 notes + 2 reference files | **Total Content**: ~23,000 lines / ~800 KB | **Estimated Study Time**: 60–80 hours

---

## All Notes Files

| # | File | Lines | Topics |
|---|------|-------|--------|
| 1 | AWS-GettingStarted-GlobalInfrastructure.md | 176 | Regions, AZs, Edge Locations, Global vs Regional services |
| 2 | IAM-Users-Groups-Policies-Roles.md | 550 | Users, Groups, Policies, Roles, STS, MFA, Access Analyzer |
| 3 | EC2-Basics-InstanceTypes-SecurityGroups.md | 539 | Instance types, pricing models, security groups, key pairs |
| 4 | EC2-Associate-PlacementGroups-ENI-Hibernate.md | 393 | Placement groups, ENI, Hibernate |
| 5 | EC2-InstanceStorage-EBS-EFS-InstanceStore.md | 554 | EBS types, snapshots, EFS, Instance Store |
| 6 | HA-Scalability-ELB-ASG.md | 650 | ALB, NLB, GLB, CLB, ASG, scaling policies |
| 7 | RDS-Aurora-ElastiCache.md | 664 | RDS Multi-AZ, Read Replicas, Aurora, ElastiCache Redis/Memcached |
| 8 | Route53-DNS-RoutingPolicies.md | 949 | DNS, routing policies, health checks, Alias records |
| 9 | S3-Basics-BucketsObjectsStorageClasses.md | 450 | Buckets, versioning, storage classes, lifecycle, replication |
| 10 | S3-Advanced-Security-Encryption.md | 336 | Bucket policies, encryption, access points, S3 Object Lambda |
| 11 | CloudFront-GlobalAccelerator.md | 511 | CloudFront CDN, distributions, OAC, Global Accelerator |
| 12 | Classic-SolutionsArchitecture.md | 933 | 3 real-world architectures (stateless, stateful, media) ⭐ |
| 13 | StorageExtras-FSx-Snowball-Gateway.md | 688 | FSx, Snow family, Storage Gateway, DataSync |
| 14 | Integration-Messaging-SQS-SNS-Kinesis.md | 2044 | SQS, SNS, Kinesis Data Streams/Firehose/Analytics, Amazon MQ |
| 15 | Containers-ECS-EKS-ECR-Fargate.md | 2582 | ECS, EKS, ECR, Fargate, App Runner, Copilot |
| 16 | Serverless-Lambda-EdgeFunctions.md | 966 | Lambda pricing/limits/concurrency, Lambda@Edge, CloudFront Functions, RDS Proxy |
| 17 | DynamoDB-APIGateway-Serverless.md | 1939 | DynamoDB, DAX, API Gateway, Step Functions, Cognito |
| 18 | Monitoring-CloudWatch-CloudTrail-Config.md | 1482 | CloudWatch, EventBridge, CloudTrail, AWS Config |
| 19 | VPC-Networking-DeepDive.md | 1507 | VPC, subnets, NAT, NACLs, peering, endpoints, VPN, DX, Transit GW |
| 20 | Security-KMS-WAF-Shield-GuardDuty.md | 2279 | KMS, CloudHSM, SSM, Secrets Manager, WAF, Shield, GuardDuty, Inspector, Macie |
| 21 | AdvancedIdentity-DR-OtherServices.md | 1431 | Organizations, STS, Identity Center, DR strategies, Backup, DMS, Analytics, ML |

**Reference Files**: `QUICK-REFERENCE.md` (exam cheat sheet) | `DIAGRAM-APPROACH-COMPARISON.md`

---

## Recommended Study Paths

### 4-Week Exam Prep Plan

**Week 1 — Compute & Networking Foundations**
| Day | File | Time |
|-----|------|------|
| 1 | AWS-GettingStarted-GlobalInfrastructure.md | 30 min |
| 1 | IAM-Users-Groups-Policies-Roles.md | 90 min |
| 2 | EC2-Basics-InstanceTypes-SecurityGroups.md | 90 min |
| 2 | EC2-Associate-PlacementGroups-ENI-Hibernate.md | 60 min |
| 3 | EC2-InstanceStorage-EBS-EFS-InstanceStore.md | 90 min |
| 4 | HA-Scalability-ELB-ASG.md | 2 hrs |
| 5 | Classic-SolutionsArchitecture.md ⭐ | 3 hrs |
| 6 | RDS-Aurora-ElastiCache.md | 2 hrs |
| 7 | Review + practice questions | 3 hrs |

**Week 2 — Storage, DNS, Networking & Security**
| Day | File | Time |
|-----|------|------|
| 1 | Route53-DNS-RoutingPolicies.md | 2 hrs |
| 2 | S3-Basics-BucketsObjectsStorageClasses.md + S3-Advanced-Security-Encryption.md | 2 hrs |
| 3 | CloudFront-GlobalAccelerator.md | 90 min |
| 4 | VPC-Networking-DeepDive.md ⭐ | 3 hrs |
| 5 | Security-KMS-WAF-Shield-GuardDuty.md ⭐ | 3 hrs |
| 6 | StorageExtras-FSx-Snowball-Gateway.md | 2 hrs |
| 7 | Review + practice questions | 3 hrs |

**Week 3 — Messaging, Containers & Serverless**
| Day | File | Time |
|-----|------|------|
| 1 | Integration-Messaging-SQS-SNS-Kinesis.md | 3 hrs |
| 2 | Containers-ECS-EKS-ECR-Fargate.md | 3 hrs |
| 3 | Serverless-Lambda-EdgeFunctions.md | 2 hrs |
| 4 | DynamoDB-APIGateway-Serverless.md | 3 hrs |
| 5 | Monitoring-CloudWatch-CloudTrail-Config.md | 2 hrs |
| 6 | AdvancedIdentity-DR-OtherServices.md | 3 hrs |
| 7 | Review + practice questions | 3 hrs |

**Week 4 — Review & Mock Exams**
| Day | Activity | Time |
|-----|----------|------|
| 1–2 | QUICK-REFERENCE.md daily + revisit weak areas | 4 hrs/day |
| 3–4 | Full-length mock exams (65 questions, 130 min) | 4 hrs/day |
| 5–6 | Review wrong answers, reread relevant sections | 4 hrs/day |
| 7 | Light review, rest | 2 hrs |

---

### 2-Week Intensive Plan

**Week 1: Foundations (50 hrs)**
- Days 1–2: Files 1–5 (IAM, EC2, Storage basics)
- Days 3–4: Files 6–8 (ELB/ASG, RDS, Route53) + Classic-SolutionsArchitecture ⭐
- Days 5–6: Files 9–11 + VPC-Networking-DeepDive ⭐
- Day 7: Security-KMS-WAF-Shield-GuardDuty ⭐

**Week 2: Advanced + Review (50 hrs)**
- Days 1–2: Messaging, Containers, Serverless, DynamoDB/API GW
- Days 3–4: Monitoring, Identity, DR, Other Services
- Days 5–7: Mock exams, review, QUICK-REFERENCE daily

---

## File Summaries by Topic Area

### 🔐 Security & Identity
| File | Key Concepts |
|------|-------------|
| IAM-Users-Groups-Policies-Roles.md | IAM foundation, least privilege, roles, STS |
| Security-KMS-WAF-Shield-GuardDuty.md | Encryption, WAF, DDoS protection, threat detection |
| AdvancedIdentity-DR-OtherServices.md | Organizations, SSO, STS, federation, Cognito |
| VPC-Networking-DeepDive.md | Security Groups, NACLs, VPC endpoints (private access) |

### 🌐 Networking
| File | Key Concepts |
|------|-------------|
| VPC-Networking-DeepDive.md | VPC, subnets, NAT, IGW, peering, Transit GW, DX, VPN |
| Route53-DNS-RoutingPolicies.md | DNS, health checks, all routing policies |
| CloudFront-GlobalAccelerator.md | CDN, edge caching, Global Accelerator |
| HA-Scalability-ELB-ASG.md | ALB/NLB/GLB, target groups, ASG |

### 💾 Storage
| File | Key Concepts |
|------|-------------|
| EC2-InstanceStorage-EBS-EFS-InstanceStore.md | EBS types, EFS, Instance Store |
| S3-Basics-BucketsObjectsStorageClasses.md | S3 fundamentals, storage classes, lifecycle |
| S3-Advanced-Security-Encryption.md | S3 encryption, policies, access points |
| StorageExtras-FSx-Snowball-Gateway.md | FSx, Snow family, Storage Gateway, DataSync |

### 🗄️ Databases
| File | Key Concepts |
|------|-------------|
| RDS-Aurora-ElastiCache.md | RDS, Aurora, ElastiCache, Multi-AZ, Read Replicas |
| DynamoDB-APIGateway-Serverless.md | DynamoDB, DAX, Global Tables, Streams |

### ⚡ Serverless & Application Integration
| File | Key Concepts |
|------|-------------|
| Serverless-Lambda-EdgeFunctions.md | Lambda, concurrency, RDS Proxy, Lambda@Edge |
| DynamoDB-APIGateway-Serverless.md | API Gateway, Step Functions, Cognito |
| Integration-Messaging-SQS-SNS-Kinesis.md | SQS, SNS, Kinesis, Amazon MQ |
| Containers-ECS-EKS-ECR-Fargate.md | ECS, EKS, Fargate, ECR, App Runner |

### 📊 Monitoring, Analytics & ML
| File | Key Concepts |
|------|-------------|
| Monitoring-CloudWatch-CloudTrail-Config.md | CloudWatch, EventBridge, CloudTrail, Config |
| AdvancedIdentity-DR-OtherServices.md | Athena, Redshift, Glue, EMR, QuickSight, SageMaker, Rekognition |

### 🔄 Disaster Recovery & Migration
| File | Key Concepts |
|------|-------------|
| AdvancedIdentity-DR-OtherServices.md | DR strategies (4 types), AWS Backup, DMS, MGN, DataSync |

---

## Critical Exam Topics (Highest Frequency)

### Must Know Cold
- **Multi-AZ vs Read Replicas** — RDS-Aurora-ElastiCache.md
- **ALB vs NLB vs GLB** — HA-Scalability-ELB-ASG.md
- **SQS Standard vs FIFO, SNS fanout** — Integration-Messaging-SQS-SNS-Kinesis.md
- **S3 storage classes + lifecycle** — S3-Basics-BucketsObjectsStorageClasses.md
- **IAM roles vs users vs policies** — IAM-Users-Groups-Policies-Roles.md
- **VPC fundamentals: SGs vs NACLs, NAT Gateway** — VPC-Networking-DeepDive.md
- **CloudWatch vs CloudTrail vs Config** — Monitoring-CloudWatch-CloudTrail-Config.md
- **Lambda concurrency, cold starts, limits** — Serverless-Lambda-EdgeFunctions.md
- **DR strategies: RPO/RTO tradeoffs** — AdvancedIdentity-DR-OtherServices.md
- **KMS envelope encryption, SSE-KMS vs SSE-S3** — Security-KMS-WAF-Shield-GuardDuty.md

### Architecture Patterns (Scenario Questions)
- **Stateless web app** → EC2 + ALB + ASG → Classic-SolutionsArchitecture.md
- **Session management** → ElastiCache → Classic-SolutionsArchitecture.md
- **Shared file storage** → EFS → EC2-InstanceStorage-EBS-EFS-InstanceStore.md
- **Event-driven serverless** → S3 Event → Lambda → DynamoDB → Serverless-Lambda-EdgeFunctions.md
- **Message fan-out** → SNS → SQS (multiple) → Integration-Messaging-SQS-SNS-Kinesis.md
- **Real-time analytics** → Kinesis Data Streams → Integration-Messaging-SQS-SNS-Kinesis.md
- **Container orchestration** → ECS Fargate → Containers-ECS-EKS-ECR-Fargate.md
- **Multi-region active-active** → DynamoDB Global Tables + Route53 → AdvancedIdentity-DR-OtherServices.md
- **Hybrid connectivity** → VPN (quick) or Direct Connect (dedicated) → VPC-Networking-DeepDive.md
- **Private S3 access from VPC** → VPC Gateway Endpoint → VPC-Networking-DeepDive.md

---

## Common Exam Question Triggers

| If the question mentions... | Check this file |
|---------------------------|-----------------|
| "Temporary credentials", "cross-account", "assume role" | IAM + AdvancedIdentity |
| "Connection pooling", "too many DB connections" | Serverless-Lambda (RDS Proxy) |
| "Real-time streaming", "Kinesis", "data pipeline" | Integration-Messaging |
| "SQL on S3", "serverless query" | AdvancedIdentity (Athena) |
| "Compliance", "resource config history", "drift detection" | Monitoring (Config) |
| "Patch management", "run command on EC2" | Security-KMS (SSM) |
| "DDoS protection", "scrub traffic" | Security-KMS (Shield/WAF) |
| "Centralize logs from 50 accounts" | Monitoring (CloudWatch cross-account) |
| "Rotate database password automatically" | Security-KMS (Secrets Manager) |
| "5G edge computing", "latency <10ms" | AdvancedIdentity (Wavelength/Local Zones) |
| "Lift and shift migration" | AdvancedIdentity (MGN/DMS) |
| "Enforce policies across all accounts" | AdvancedIdentity (Organizations/SCPs) |

---

## AWS CLI Coverage Summary

Every notes file contains embedded CLI commands and a CLI Quick Reference section.
Total CLI commands across all files: **500+**

Key commands by file:
- IAM: `aws iam create-user`, `attach-role-policy`, `assume-role`
- EC2: `aws ec2 run-instances`, `describe-instances`, `create-security-group`
- S3: `aws s3 cp`, `s3api put-bucket-policy`, `s3api put-object` with `--server-side-encryption`
- RDS: `aws rds create-db-instance`, `create-db-cluster`, `describe-db-instances`
- VPC: `aws ec2 create-vpc`, `create-subnet`, `create-nat-gateway`, `create-vpc-peering-connection`
- Lambda: `aws lambda create-function`, `invoke`, `put-concurrency`
- CloudWatch: `aws cloudwatch put-metric-alarm`, `put-metric-data`, `logs start-query`
- KMS: `aws kms create-key`, `encrypt`, `decrypt`, `generate-data-key`
- GuardDuty: `aws guardduty create-detector`, `get-findings`

---

## Study Aids

### QUICK-REFERENCE.md
Fast-lookup cheat sheet. Review daily during final 2 weeks. Contains:
- Service selection decision trees
- Comparison tables (RDS vs DynamoDB, SQS vs Kinesis, etc.)
- Common gotchas and exam traps
- Last-minute review checklist

### DIAGRAM-APPROACH-COMPARISON.md
Technical comparison of diagram approaches used in these notes (SVG, Mermaid, HTML icons).

---

## Exam Day Checklist

**Night before:**
- [ ] Re-read QUICK-REFERENCE.md
- [ ] Review IAM-Users-Groups-Policies-Roles.md Points to Remember
- [ ] Review VPC-Networking-DeepDive.md Points to Remember
- [ ] Sleep 8 hours

**Morning of:**
- [ ] 15-min skim of QUICK-REFERENCE.md
- [ ] Mental walkthrough of 3 Classic Architecture patterns

**During exam (65 questions, 130 minutes):**
- ~2 min per question; flag uncertain ones; come back
- For architecture questions: identify the constraint (cost? performance? availability? security?)
- "Most cost-effective" → Spot instances, S3 IA, Reserved capacity, Fargate vs EC2
- "Highly available" → Multi-AZ, ASG, Route53 failover
- "Serverless" → Lambda, DynamoDB, API Gateway, Fargate, S3, SQS

**Score target**: 720/1000 to pass (roughly 72% correct)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | March 2026 | Initial 15-file package (Slot 1) |
| 2.0 | March 2026 | Added 6 files (Slot 2): Monitoring, VPC, Security, Serverless, Containers, Messaging, DynamoDB, AdvancedIdentity/DR |

---

*These notes are a supplementary study resource. Always cross-reference with official AWS documentation and the SAA-C03 exam guide for the most current information.*
