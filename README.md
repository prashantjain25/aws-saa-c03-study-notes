# AWS Solutions Architect Handbook

> **Purpose**: A production-quality architect reference, not certification notes. This handbook covers AWS services with the depth, correctness, and operational realism expected in senior cloud architecture roles. Each section is designed to survive scrutiny from principal engineers, infrastructure reviewers, and AWS interviewers.
>
> **Scope**: 15 sections covering Identity, Security, Compute, Storage, Caching, Databases, Service Communication, Data Engineering, Monitoring, Deployment, Cost Control, Migration, Networking, Machine Learning, and Other Services.

## What Makes This Different

Unlike certification cram materials, this handbook:

- **Explains WHY before WHAT** — architectural reasoning, not feature lists
- **Validates against modern AWS behavior** — no deprecated terminology or outdated assumptions
- **Includes operational realities** — failure modes, cost implications, scaling limits
- **Provides interview hardening** — common traps, hidden limitations, tradeoff analysis
- **Emphasizes cross-service integration** — architects think in systems, not silos
- **Uses correct networking semantics** — packet flow, ingress/egress, routing correctness

## Section Structure

Each major section follows a consistent template:

1. **Purpose** — Why the service category matters
2. **Core Architecture** — How it works internally
3. **Scaling Behavior** — How it handles growth
4. **HA Behavior** — Failure modes and recovery
5. **Security Implications** — Threat model and controls
6. **Performance Characteristics** — Latency, throughput, bottlenecks
7. **Cost Implications** — Pricing model and optimization
8. **Operational Realities** — What breaks in production
9. **Failure Scenarios** — How to debug and recover
10. **Common Misconceptions** — What senior engineers get wrong
11. **Interview Challenges** — Defensible architecture decisions
12. **Tradeoffs** — When NOT to use this service
13. **Alternative Comparisons** — When to choose a competitor

## Sections

| # | Section | Description |
|---|---------|-------------|
| 03 | [Identity & Federation](03-Identity-and-Federation.md) | IAM, Organizations, STS, Identity Center, Cognito, ABAC, SCPs |
| 04 | [Security](04-Security.md) | KMS, WAF, Shield, GuardDuty, CloudTrail, Config, encryption |
| 05 | [Compute & Load Balancing](05-Compute-and-Load-Balancing.md) | EC2, Lambda, ECS, EKS, ALB, NLB, Auto Scaling |
| 06 | [Storage](06-Storage.md) | S3, EBS, EFS, FSx, Storage Gateway, lifecycle, tiers |
| 07 | [Caching](07-Caching.md) | CloudFront, ElastiCache, Global Accelerator, edge compute |
| 08 | [Databases](08-Databases.md) | RDS, Aurora, DynamoDB, DAX, Redshift |
| 09 | [Service Communication](09-Service-Communication.md) | SQS, SNS, EventBridge, Step Functions, API Gateway |
| 10 | [Data Engineering](10-Data-Engineering.md) | Kinesis, Glue, EMR, Athena, Redshift, data lakes |
| 11 | [Monitoring & Observability](11-Monitoring-and-Observability.md) | CloudWatch, X-Ray, CloudTrail, Config |
| 12 | [Deployment & Instance Management](12-Deployment-and-Instance-Management.md) | Systems Manager, CloudFormation, CodeDeploy |
| 13 | [Cost Control](13-Cost-Control.md) | Cost Explorer, Savings Plans, Spot, tagging, optimization |
| 14 | [Migration](14-Migration.md) | MGN, DMS, DataSync, Backup, DRS, 7 R's strategy |
| 15 | [VPC & Networking](15-VPC-and-Networking.md) | VPC, subnets, routing, Security Groups, NACLs, TGW, DX |
| 16 | [Machine Learning](16-Machine-Learning.md) | SageMaker, Bedrock, Rekognition, Comprehend |
| 17 | [Other Services](17-Other-Services.md) | WorkSpaces, IoT, Amplify, AppSync, Connect |

## Cross-Service Patterns

| Pattern | Services Involved |
|---------|-------------------|
| Three-Tier Web App | Route53 → CloudFront → ALB → ECS/EC2 → RDS/ElastiCache |
| Event-Driven Microservices | EventBridge → SQS → Lambda → DynamoDB |
| Serverless API | API Gateway → Lambda → DynamoDB |
| Secure CI/CD | GitHub OIDC → IAM Role → ECR → ECS |
| Centralized Security | Org Trail → S3 → Athena + GuardDuty → Security Hub |
| Data Lake | Kinesis/Glue → S3 → Athena/Redshift/SageMaker |
| Global Application | Route53 → CloudFront → ALB (multi-region) → DynamoDB Global Tables |
| Hybrid Cloud | Direct Connect → Transit Gateway → VPC (multi-account) |
| Disaster Recovery | Aurora Global Database + Route53 Failover + S3 CRR |

## Usage

### For SAA-C03 Certification
- Sections align with exam domains but go deeper than required
- Focus on "Points to Remember" and "Interview Challenges" for exam scenario questions
- Cross-service patterns reflect the integrated architecture questions on the exam

### For Architecture Interviews
- Every section contains questions a principal engineer might ask
- Tradeoff analysis demonstrates senior-level thinking
- Operational realities show production experience

### For Production Architecture
- Use "Points to Remember" as design review checklists
- Reference sections when selecting between similar services
- Validate networking and security assumptions

## Validation

All technical claims have been validated against:
- AWS Official Documentation
- AWS Architecture Center
- AWS Well-Architected Framework
- AWS re:Invent guidance
- AWS FAQs and service pages

Outdated terminology has been corrected:
- IAM Identity Center (not AWS SSO)
- Amazon OpenSearch Service (not Elasticsearch)
- EC2 Auto Scaling (consistent naming)

---

**Version:** 3.0 Architect Handbook  
**Refactored:** 2026-05-10  
**Format:** Markdown  
**Validation:** AWS documentation aligned, modern behavior verified
