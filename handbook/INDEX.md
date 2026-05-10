# AWS Solutions Architect Handbook — Index

> **Purpose**: This is a production-quality architect handbook, not study notes. Each section is designed to survive scrutiny from senior cloud architects, principal engineers, and AWS interviewers. Depth and correctness take precedence over brevity.
>
> **Structure**: 15 sections organized by architectural domain. Each section covers purpose, core architecture, scaling, HA, security, performance, cost, operational realities, failure scenarios, misconceptions, interview challenges, tradeoffs, and cross-service integrations.

---

## Section Index

| # | Section | Topics | Depth |
|---|---------|--------|-------|
| 03 | [Identity & Federation](sections/03-Identity-and-Federation.md) | IAM, Organizations, STS, IAM Identity Center, Cognito, Directory Service, ABAC, permission boundaries, SCPs, cross-account patterns, confused deputy | Enterprise identity architecture, multi-account governance, interview-level policy evaluation |
| 04 | [Security](sections/04-Security.md) | KMS, CloudHSM, Secrets Manager, Parameter Store, ACM, WAF, Shield, GuardDuty, Inspector, Macie, Security Hub, Access Analyzer, CloudTrail, Config, encryption architecture, threat detection | Encryption at scale, centralized security account, compliance frameworks, defense in depth |
| 05 | [Compute & Load Balancing](sections/05-Compute-and-Load-Balancing.md) | EC2, Lambda, ECS, EKS, ALB, NLB, GWLB, Auto Scaling, Spot, Graviton, placement groups, ENI, instance types, purchasing options | Production compute design, container orchestration decisions, load balancer selection framework |
| 06 | [Storage](sections/06-Storage.md) | S3 (all tiers), EBS (all types), EFS, FSx (all variants), Storage Gateway, S3 replication, lifecycle, encryption, performance optimization | Data lake architecture, storage class economics, hybrid cloud patterns |
| 07 | [Caching](sections/07-Caching.md) | CloudFront, ElastiCache (Redis, Memcached), Global Accelerator, Lambda@Edge, cache design patterns, invalidation strategies, thundering herd mitigation | Edge architecture, cache consistency tradeoffs, CDN design |
| 08 | [Databases](sections/08-Databases.md) | RDS, Aurora (provisioned + Serverless v2), DynamoDB, DAX, Global Tables, Streams, single-table design, Redshift, Neptune (mentioned) | Database selection framework, partition key design, HA/DR patterns |
| 09 | [Service Communication](sections/09-Service-Communication.md) | SQS, SNS, EventBridge, Step Functions, API Gateway (REST/HTTP/WebSocket), AppSync, fan-out, saga pattern, event-driven architecture | Async vs sync decisions, orchestration vs choreography, API design patterns |
| 10 | [Data Engineering](sections/10-Data-Engineering.md) | Kinesis (Streams, Firehose, Analytics), Glue, EMR, Athena, Redshift, Redshift Spectrum, data lake architecture, lake house pattern | Streaming analytics, ETL design, data lake zones, cost-optimized querying |
| 11 | [Monitoring & Observability](sections/11-Monitoring-and-Observability.md) | CloudWatch (metrics, logs, alarms, dashboards, Contributor Insights), X-Ray, CloudTrail, Config, anomaly detection, distributed tracing | Production observability, SLO-based alerting, cross-service tracing |
| 12 | [Deployment & Instance Management](sections/12-Deployment-and-Instance-Management.md) | Systems Manager (Session Manager, Patch Manager, Run Command, Automation), CloudFormation, CodeDeploy, AppConfig | Infrastructure as code, zero-downtime deployments, operational automation |
| 13 | [Cost Control](sections/13-Cost-Control.md) | Cost Explorer, Budgets, Compute Optimizer, Savings Plans, Spot strategy, data transfer optimization, tagging, anomaly detection | FinOps architecture, cost-aware design decisions, waste elimination |
| 14 | [Migration](sections/14-Migration.md) | MGN, DMS, SCT, DataSync, AWS Backup, DRS, 7 R's strategy, database migration patterns, VMware migration | Risk-minimized migration, CDC replication, DR strategies |
| 15 | [VPC & Networking](sections/15-VPC-and-Networking.md) | VPC design, subnetting, routing, Security Groups, NACLs, NAT Gateway, VPC Peering, Endpoints, Transit Gateway, VPN, Direct Connect, PrivateLink, Route53 | Network architecture, hybrid cloud, multi-account connectivity, DNS design |
| 16 | [Machine Learning](sections/16-Machine-Learning.md) | SageMaker (training, hosting, pipelines, Feature Store), Bedrock, Rekognition, Comprehend, Trainium, Inferentia, managed AI services | ML infrastructure, model deployment options, AI service selection |
| 17 | [Other Services](sections/17-Other-Services.md) | WorkSpaces, AppStream, IoT Core, Amplify, AppSync, Connect, Ground Station, Local Zones, Wavelength | Specialized service recognition, edge computing, end-user computing |

---

## Cross-Service Architecture Patterns

| Pattern | Services | Section Reference |
|---------|----------|------------------|
| **Three-Tier Web App** | Route53 → CloudFront → ALB → ECS/EC2 → RDS/ElastiCache | 05, 07, 08 |
| **Event-Driven Microservices** | EventBridge → SQS → Lambda → DynamoDB | 09, 08 |
| **Serverless API** | API Gateway → Lambda → DynamoDB | 05, 09, 08 |
| **Secure CI/CD** | GitHub OIDC → IAM Role → ECR → ECS Deploy | 03, 12 |
| **Centralized Security** | Organizations Trail → S3 → Athena + GuardDuty → Security Hub | 03, 04, 11 |
| **Data Lake** | Kinesis/Glue → S3 (raw/curated) → Athena/Redshift/SageMaker | 10, 06 |
| **Global Application** | Route53 Latency Routing → CloudFront → ALB (multi-region) → DynamoDB Global Tables | 07, 08, 15 |
| **Hybrid Connectivity** | Direct Connect → Transit Gateway → VPC (multi-account) | 15 |
| **Disaster Recovery** | Aurora Global Database + Route53 Failover + S3 Cross-Region Replication | 08, 15 |

---

## How to Use This Handbook

### For Exam Preparation (SAA-C03)
1. Read each section sequentially, focusing on "Why" and architectural tradeoffs
2. Review interview challenges at the end of each section — these reflect exam scenario questions
3. Study cross-service integration patterns — the exam heavily tests integrated architectures
4. Memorize the "Points to Remember" at section ends as concise review material

### For Interview Preparation
1. Focus on Sections 03 (Identity), 04 (Security), 05 (Compute), 08 (Databases), 15 (Networking)
2. Practice explaining tradeoffs: ALB vs NLB, ECS vs EKS, DynamoDB vs Aurora, SQS vs EventBridge
3. Be ready to whiteboard the cross-service patterns listed above
4. Understand failure scenarios and operational realities, not just happy-path features

### For Architecture Reviews
1. Use the "Points to Remember" as a review checklist for your designs
2. Reference the interview challenges to anticipate reviewer questions
3. Validate your designs against the tradeoff analysis in each section
4. Ensure cross-service integrations are explicitly documented

---

*Index for AWS Solutions Architect Handbook | Refactored: 2026-05-10*

