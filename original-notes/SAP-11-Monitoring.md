# SAP Section 11: Monitoring & Observability

> **Scope**: CloudWatch, EventBridge, X-Ray, AWS Health, and AWS Config. CloudTrail has moved to SAP-04 (Security). This section is about operational visibility, event-driven automation, distributed tracing, and compliance governance.

---

## Table of Contents

1. [CloudWatch Overview](#1-cloudwatch-overview)
2. [CloudWatch Logs](#2-cloudwatch-logs)
3. [CloudWatch Metrics — Advanced](#3-cloudwatch-metrics--advanced)
4. [CloudWatch Alarms](#4-cloudwatch-alarms)
5. [CloudWatch Dashboards](#5-cloudwatch-dashboards)
6. [CloudWatch Evidently](#6-cloudwatch-evidently)
7. [CloudWatch RUM](#7-cloudwatch-rum-real-user-monitoring)
8. [CloudWatch Synthetics](#8-cloudwatch-synthetics)
9. [CloudWatch Insights Suite](#9-cloudwatch-insights-suite)
10. [Amazon EventBridge](#10-amazon-eventbridge)
11. [AWS X-Ray](#11-aws-x-ray)
12. [AWS Health Dashboard](#12-aws-health-dashboard)
13. [AWS Config](#13-aws-config)
14. [Multi-Account Observability Architecture](#14-multi-account-observability-architecture)
15. [Decision Framework](#15-decision-framework-which-service-for-which-need)

---

## 1. CloudWatch Overview

CloudWatch is the **central nervous system** of AWS observability. It is not just a metrics dashboard — it is a full-stack observability platform covering metrics, logs, alarms, dashboards, and anomaly detection.

The key mental model: **CloudWatch tells you what is happening right now and what has been happening over time.** It does not tell you who caused it (CloudTrail) or whether configuration drifted (Config).

### Core Concepts

| Concept | Definition | Example |
|---|---|---|
| **Namespace** | Logical grouping for metrics | `AWS/EC2`, `AWS/RDS`, `MyApp/Orders` |
| **Metric** | A time-series data point | `CPUUtilization` |
| **Dimension** | Key-value attribute that identifies a metric | `InstanceId=i-abc123` |
| **Resolution** | How frequently data points are stored | Standard (1 min) or High-Resolution (1 sec) |
| **Statistics** | Aggregations over a period | Average, Sum, Min, Max, p99 |
| **Period** | Time window for aggregation | 60s, 300s, 3600s |

### Metric Resolution — Standard vs High-Resolution

- **Standard Resolution**: 1-minute granularity. Data older than 15 days is rolled up to 5-minute intervals; older than 63 days is rolled up to 1-hour intervals.
- **High-Resolution**: 1-second granularity. Use `StorageResolution=1` when calling `PutMetricData`. More expensive.

> 🎯 **EXAM TIP**: High-resolution alarms can trigger every **10 seconds or 30 seconds**. Standard alarms fire at multiples of 60 seconds. If a question asks about "sub-minute" alerting, the answer always involves high-resolution metrics.

### Default EC2 Metrics vs CloudWatch Agent

**Default EC2 metrics** (sent automatically at 5-minute intervals, 1-minute with detailed monitoring enabled):
- CPUUtilization, NetworkIn, NetworkOut, DiskReadOps, DiskWriteOps, StatusCheckFailed

**What is NOT included by default** (requires CloudWatch Agent):
- RAM / memory utilization
- Disk space (used/available at OS level)
- Swap usage
- Per-process metrics
- Custom application metrics

> 🎯 **EXAM TIP**: If a question says "monitor memory utilization on EC2," the answer always requires the **CloudWatch Unified Agent**. This is one of the most common SAA/SAP gotchas.

### CloudWatch Agent Configuration

The Unified CloudWatch Agent supports:
- Sending logs AND metrics from the same agent
- Reading config from SSM Parameter Store (useful for fleet-wide rollout)
- Works on EC2, on-premises servers, and other cloud instances

---

## 2. CloudWatch Logs

### Hierarchy

```
Log Group  (/aws/lambda/my-function)
  └── Log Stream  (2024/01/15/[$LATEST]abc123)
        └── Log Event  (timestamp + message)
```

- **Log Group**: the primary unit of management (retention, access control, subscriptions)
- **Log Stream**: typically one per source instance or Lambda invocation
- **Log Event**: individual timestamped entry

### Retention and Encryption

- Default retention: **never expire** (costs money indefinitely)
- Settable values: 1 day up to 10 years
- Encryption: by default uses AWS-managed keys; can configure KMS CMK per log group

> 🎯 **EXAM TIP**: If a question mentions a compliance requirement like "logs must be kept for 7 years," you need a combination of CloudWatch Logs (for real-time access) + S3 export (for long-term archival). CloudWatch Logs alone is expensive for 7-year retention.

### Log Sources

| Source | How Logs Arrive |
|---|---|
| Lambda | Automatic — no agent needed |
| ECS / Fargate | `awslogs` log driver in task definition |
| EKS | Fluent Bit DaemonSet |
| EC2 | CloudWatch Unified Agent |
| API Gateway | Access logging enabled per stage |
| VPC Flow Logs | Delivered to log group |
| Route 53 | DNS query logging |
| On-premises | CloudWatch Agent or Kinesis Agent |

### CloudWatch Logs Insights

A purpose-built query language for log analysis. Key reasons to use it over exporting to Athena:
- No ETL needed — query in-place
- Interactive, fast for recent data
- Supports multiple log groups in a single query

**Example queries**:

```
# Count errors by 5-minute window
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() as errorCount by bin(5m)

# Find top 10 slowest requests
fields duration, requestId
| sort duration desc
| limit 10

# P50/P99 latency
fields responseTime
| stats avg(responseTime), pct(responseTime, 50), pct(responseTime, 99)
```

> 🎯 **EXAM TIP**: Logs Insights queries are **read-only** and work across multiple log groups. They do NOT support real-time alerting — you need Metric Filters + CloudWatch Alarms for that.

### Subscription Filters — Real-Time Log Delivery

A subscription filter streams matching log events to a downstream service in near real-time:

| Destination | Use Case |
|---|---|
| Lambda | Real-time processing, custom alerting |
| Kinesis Data Streams | Fan-out to multiple consumers |
| Kinesis Data Firehose | Buffered delivery to S3, OpenSearch, Splunk, Datadog |
| OpenSearch Service | Full-text search and dashboards |

**Important constraint**: each log group supports up to **2 subscription filters**.

### Cross-Account Log Aggregation via Subscriptions

This is a key SAP architecture pattern. Account A (workload) ships logs to a central logging account (Account B) via subscription filter pointing to a Kinesis Data Stream in Account B.

```
Account A (App)                    Account B (Central Logs)
  CloudWatch Logs                    Kinesis Data Stream
    Subscription Filter    ------>     Firehose --> S3
  (destination: Kinesis ARN           (long-term archive)
   in Account B)
```

Account B must grant cross-account permission via a resource policy on the Kinesis stream.

> 🎯 **EXAM TIP**: Subscription filters to cross-account destinations require a **destination** resource (Kinesis or Lambda) in the target account with a resource-based policy allowing the source account's CloudWatch Logs service principal (`logs.amazonaws.com`) to `kinesis:PutRecord`.

### Metric Filters — Logs to Metrics

Convert log patterns into CloudWatch metrics, then alarm on them:

```
Log Group (/app/production)
  Pattern: [... status=5* ...]
    --> Metric: MyApp/5xxCount
      --> Alarm: page on-call if > 10 in 5 minutes
```

Metric filters do **not** retroactively process old data — they only count matching events from the moment the filter is created.

### Export to S3

- `CreateExportTask` API — asynchronous, takes minutes to hours
- Not real-time (use subscriptions for real-time)
- Encrypted with SSE-S3 by default

---

## 3. CloudWatch Metrics — Advanced

### Custom Metrics

Published via `PutMetricData` API. Up to 30 dimensions per metric. Custom metrics are billed per metric per month.

```bash
aws cloudwatch put-metric-data \
  --namespace "MyApp/Orders" \
  --metric-name "CheckoutLatency" \
  --value 245 \
  --unit Milliseconds \
  --storage-resolution 1 \
  --dimensions Name=Region,Value=us-east-1 Name=Service,Value=checkout
```

### Metric Math

Perform calculations across multiple metrics to produce derived metrics without publishing custom data points. Examples:

- Error rate = `(errors / requests) * 100`
- Available capacity = `maxCapacity - currentUsage`
- Aggregate CPU across all instances in an ASG using `METRICS("CPUUtilization")` with `AVG()`

Metric math results can be used in alarms and dashboards. This is powerful because you can create ratio-based alarms without extra CloudWatch Logs infrastructure.

> 🎯 **EXAM TIP**: Metric math is evaluated at query time — it does not store new data points. You can alarm on a metric math expression, but the expression is re-evaluated each period.

### Anomaly Detection

CloudWatch uses ML to establish a baseline band for a metric and alerts when actual values fall outside that band.

- Automatically accounts for time-of-day and day-of-week patterns
- Can exclude known anomaly windows when training the model
- Alarms can use `ANOMALY_DETECTION_BAND` as the threshold instead of a static number

> 🎯 **EXAM TIP**: Anomaly detection is ideal when you do not know the right static threshold — e.g., traffic that naturally varies by 10x between day and night. Static thresholds would produce false positives at night and miss real anomalies during peak hours.

---

## 4. CloudWatch Alarms

### Three States

| State | Meaning |
|---|---|
| **OK** | Metric within threshold |
| **ALARM** | Metric breached threshold |
| **INSUFFICIENT_DATA** | Not enough data to evaluate yet |

An alarm stuck in `INSUFFICIENT_DATA` does not fire actions. This typically occurs right after creation or if the underlying metric stops reporting (e.g., an EC2 instance is stopped).

### Alarm Actions

- **EC2 actions**: Stop, Terminate, Reboot, Recover (restart on different host hardware)
- **Auto Scaling**: trigger scale-out or scale-in policies
- **SNS**: publish to topic, which can fan out to email, SMS, Lambda, SQS, HTTP endpoints
- **Systems Manager OpsItem or Automation**: trigger remediation runbooks

### Composite Alarms

Combine multiple alarms with `AND` / `OR` / `NOT` logic. Primary use case: reduce alarm noise.

```
ALARM("db-cpu-alarm") AND ALARM("db-connection-alarm")
  --> Only page if BOTH conditions are true simultaneously
```

> 🎯 **EXAM TIP**: Composite alarms are the answer when a question says "reduce alert fatigue" or "only trigger when multiple conditions are met simultaneously." You cannot do AND logic with a single metric alarm.

### EC2 Recovery Alarm

When `StatusCheckFailed_System` triggers, CloudWatch can automatically recover the instance (move it to healthy hardware). The instance retains its private IP, Elastic IP, instance ID, and data on EBS volumes. Instance store data is lost.

> 🎯 **EXAM TIP**: Recovery keeps the same private IP and EIP — this is different from a new instance launched by Auto Scaling. If preserving the IP address is a requirement, Recovery is better than ASG replacement.

### Alarm on Alarm

A composite alarm can reference other composite alarms, enabling multi-tier alert trees. You can also create alarms that watch whether another alarm is in ALARM state — this enables "alarm on alarm" chaining for complex escalation logic.

---

## 5. CloudWatch Dashboards

### Key Features

- **Cross-region**: a single dashboard can display metrics from any AWS region
- **Cross-account**: can display metrics from other AWS accounts (requires CloudWatch cross-account observability setup)
- **Sharing**: dashboards can be shared publicly (with or without authentication) for external stakeholders
- **Automatic refresh**: configurable from 10 seconds to 15 minutes

### Cross-Account and Cross-Region Dashboards

This requires **CloudWatch cross-account observability** (formerly called CloudWatch cross-account sharing):

1. In each **source account**: enable sharing and specify the monitoring account's ID
2. In the **monitoring account**: link to source accounts

Once linked, the monitoring account can view metrics, alarms, and log insights from source accounts as if they were local.

> 🎯 **EXAM TIP**: Cross-account CloudWatch observability is separate from the Config aggregator and separate from Organizations. You configure it in the CloudWatch console under "Settings > Monitoring account configuration." It does NOT require AWS Organizations, but can leverage it for bulk enrollment.

---

## 6. CloudWatch Evidently

A relatively new SAP-scope service for **feature flagging and A/B testing** within CloudWatch.

### What It Does

- **Feature flags**: roll out new code to a percentage of users without a new deployment
- **Experiments (A/B tests)**: compare two variants and measure the effect on metrics you define
- **Launches**: gradually increase the percentage of users seeing a new feature

### Why This Matters for the Exam

Evidently lets teams safely release features with controlled blast radius. Instead of a full deployment, you push a flag — if metrics degrade, flip it off instantly. The service integrates with CloudWatch metrics and alarms.

> 🎯 **EXAM TIP**: If a question asks how to test a new feature on 5% of users before full rollout while measuring impact on business metrics — the answer is **CloudWatch Evidently** (not a blue/green deployment, not weighted routing in Route 53). Route 53 weighted routing splits traffic but does not give you metric comparison between variants.

---

## 7. CloudWatch RUM (Real User Monitoring)

### What It Does

Collects performance data from **actual user browsers** — not from your servers. You add a JavaScript snippet to your web application, and RUM records:

- Page load times
- Web Vitals (Largest Contentful Paint, Cumulative Layout Shift, etc.)
- JavaScript errors
- HTTP request errors
- User sessions and user journeys

### RUM vs Synthetics

| | RUM | Synthetics |
|---|---|---|
| **Data source** | Real users | Scripted bots |
| **Always running?** | Only when users are present | Yes, runs on schedule |
| **Useful for** | Understanding real experience | Proactive SLA monitoring |
| **Off-hours coverage** | No (no users) | Yes |

> 🎯 **EXAM TIP**: RUM is the right answer when the question is "monitor end-user experience from their actual browsers." Synthetics is the right answer when the question is "detect outages before users notice" or "test from specific geographic locations on a schedule."

---

## 8. CloudWatch Synthetics

### Overview

Synthetics runs **canary scripts** — code that simulates user actions against your endpoints — on a configurable schedule (every minute to every hour). If the canary fails (wrong status code, element not found, timeout), it creates a CloudWatch metric and can trigger an alarm.

### Canary Types

| Blueprint | What It Tests |
|---|---|
| Heartbeat Monitor | HTTP GET, checks status code and response time |
| API Canary | REST API testing with request/response validation |
| Link Checker | Crawls a URL and verifies all links |
| Visual Monitor | Screenshots and pixel comparison (detects UI regressions) |
| Canary Recorder | Records browser interactions and replays them |

### SAP-Specific Depth: SLA Monitoring

For SLA commitments (e.g., 99.9% uptime, < 2s P99 latency), Synthetics is the primary tool because:
- It runs continuously on schedule, not dependent on real user traffic
- Results are stored as CloudWatch metrics — you can query historical availability
- Alarms on Synthetics failures can page on-call before users report issues
- It can test from multiple regions by deploying canaries in each region

> 🎯 **EXAM TIP**: Synthetics canaries run in **your account** in AWS-managed infrastructure. They incur costs per canary run. They use the **CloudWatch Synthetics** namespace for metrics, not `AWS/Lambda`.

---

## 9. CloudWatch Insights Suite

### Container Insights

Collects metrics from ECS, EKS, and self-managed Kubernetes:

- **ECS**: cluster, service, task-level metrics (CPU, memory, network)
- **EKS / Kubernetes**: node, pod, container metrics via **Fluent Bit DaemonSet**
- **Fargate**: limited metrics automatically; full Container Insights requires additional config

### Lambda Insights

An extension layer added to Lambda functions that captures:
- Memory used vs allocated (identify over-provisioning)
- Init duration (cold start impact)
- Duration, billed duration
- Network throughput

> 🎯 **EXAM TIP**: Lambda Insights is a **Lambda Layer** — you attach it to the function. It writes to a special log group `/aws/lambda-insights` and creates metrics in the `LambdaInsights` namespace. It does NOT use the `AWS/Lambda` namespace.

### Contributor Insights

Identifies top-N contributors to a log pattern. Useful for:
- Finding the top IPs making the most requests (DDoS triage)
- Finding the users generating the most errors
- Finding the most-called API paths

Contributor Insights creates **CloudWatch metrics** from log patterns so you can alarm on "top contributor exceeded threshold."

### Application Insights

Auto-discovers application components (.NET, Java, SQL Server) and creates monitoring dashboards using ML-based anomaly detection. Minimal configuration required — it reads from CloudWatch Logs and metrics to identify problems.

---

## 10. Amazon EventBridge

EventBridge is the successor to CloudWatch Events. It is a **serverless event router** that connects event producers to event consumers with filtering and transformation.

### Event Buses

| Type | Description |
|---|---|
| **Default** | Receives all AWS service events automatically |
| **Custom** | Receives events you publish via `PutEvents` API |
| **Partner** | Receives events from AWS Partner SaaS (Datadog, PagerDuty, Zendesk, etc.) |

### Rules: Two Types

**Schedule rules** — run on a cron or rate expression:
```
rate(5 minutes)
cron(0 8 * * ? *)   # 8 AM UTC daily
```

**Event pattern rules** — match specific event structures:
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["terminated"]
  }
}
```

> 🎯 **EXAM TIP**: EventBridge Scheduler is a **separate service** from EventBridge scheduled rules. The key difference: EventBridge Scheduler supports **one-time schedules** (fire at a specific datetime once), **flexible time windows** (fire within a window to distribute load), and **timezone-aware** scheduling. EventBridge cron rules do not support one-time firing.

### Targets

A single rule can have up to **5 targets**. Targets include Lambda, SQS, SNS, Kinesis, Step Functions, CodePipeline, CodeBuild, ECS tasks, EC2 API calls, and more.

Input transformation lets you reshape the event before it reaches the target — extract specific fields, inject static values, or restructure the JSON.

### EventBridge Pipes

**EventBridge Pipes** enables **point-to-point integrations** between a source and a target with optional filtering and enrichment in the middle.

```
Source                   Optional               Target
SQS Queue     -->     Enrichment Step    -->   Lambda
DynamoDB Stream        (Lambda or          Step Functions
Kinesis Stream          API Gateway call)  SNS
Kafka topic                                SQS
```

Key difference from Rules: Pipes are **one-to-one** (one source → one target, with enrichment), while Rules are **one-to-many** (one event pattern → up to 5 targets). Pipes also handle **batching** and **polling** sources like SQS and Kinesis automatically.

> 🎯 **EXAM TIP**: If a question asks "how to enrich an SQS message with data from DynamoDB before sending to Lambda," the answer is **EventBridge Pipes** with a Lambda enrichment step — not a Lambda trigger on SQS, because the latter requires you to write the enrichment logic inside the Lambda function itself.

### Schema Registry

EventBridge can automatically discover event schemas and store them in the Schema Registry. This enables:
- Code bindings generation (Python, Java, TypeScript) for type-safe event handling
- Discovery of what events are available from AWS services and partners

### Archive and Replay

- Archive events to replay them later (testing, disaster recovery, reprocessing after bug fix)
- Archived events can be filtered by event pattern
- Replay injects events back into an event bus as if they were arriving in real-time

### Cross-Account Event Routing

Source account puts events to a custom event bus in the central account using the bus ARN. The central bus needs a resource-based policy allowing the source account:

```json
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::SOURCE_ACCOUNT:root"},
  "Action": "events:PutEvents",
  "Resource": "arn:aws:events:us-east-1:CENTRAL_ACCOUNT:event-bus/central-bus"
}
```

> 🎯 **EXAM TIP**: Cross-account EventBridge events go from a source account's **default bus** to a **custom bus** in the central account — not to the central account's default bus. The central account's default bus only receives that account's own AWS service events.

---

## 11. AWS X-Ray

### What X-Ray Does

X-Ray provides **distributed tracing** — it follows a single request as it travels through multiple microservices, showing you exactly where latency is introduced and where errors occur.

Without X-Ray, you have logs in each service, but correlating them across 10 microservices for a single failed user request requires manual effort. X-Ray does this automatically via **trace IDs** propagated through HTTP headers and message attributes.

### Core Concepts

| Concept | Definition |
|---|---|
| **Trace** | End-to-end record of a single request (collection of segments) |
| **Segment** | Record of work done by one service (with start time, end time, errors) |
| **Subsegment** | Finer-grained unit within a segment (e.g., an outbound DynamoDB call) |
| **Annotation** | Indexed key-value pair — **can be used in filter expressions** |
| **Metadata** | Non-indexed key-value pair — stored in trace but not searchable |
| **Service Map** | Visual graph of all services and their interconnections |
| **Sampling** | Percentage of requests to trace (avoid tracing 100% for cost/performance) |

### Annotations vs Metadata

> 🎯 **EXAM TIP**: This distinction is frequently tested. **Annotations** are indexed and filterable (use for things you will search by: `userId`, `orderId`, `environment`). **Metadata** is stored but not indexed (use for rich debugging data you won't search by: full request payload, large objects). If a question asks "search traces by customer ID," you need annotations.

### Sampling

Default sampling: 1 request per second + 5% of additional requests. You can define custom sampling rules:
- Match on service name, URL path, HTTP method, host
- Set a fixed rate and a reservoir (requests per second)

This prevents high-volume APIs from generating excessive trace data while still capturing enough for analysis.

### Service Map

The Service Map shows:
- All services that processed requests
- Average latency and error rate per service-to-service connection
- Whether errors are in the source, the target, or the network

This is the fastest way to visually identify which microservice is causing degradation.

### X-Ray Groups

Groups let you filter traces by expression and assign sampling rules to them. Example: create a group for `annotation.environment = "production"` to see only production traces in a dedicated view.

### Integration with AWS Services

| Service | X-Ray Integration |
|---|---|
| **Lambda** | Enable Active Tracing in function config; X-Ray daemon runs automatically |
| **API Gateway** | Enable X-Ray tracing per stage; adds trace header to downstream calls |
| **ECS** | Run X-Ray daemon as a sidecar container in the task definition |
| **Elastic Beanstalk** | X-Ray daemon included in platform; enable via config |
| **EC2** | Install X-Ray daemon manually or via Systems Manager |
| **App Mesh** | Envoy sidecar sends traces automatically |

> 🎯 **EXAM TIP**: For Lambda, enabling X-Ray Active Tracing adds an IAM permission requirement (`xray:PutTraceSegments`, `xray:PutTelemetryRecords`) to the Lambda execution role. Forgetting this permission is a common exam scenario.

### X-Ray Daemon

The X-Ray SDK sends trace data to the **X-Ray daemon** (a local UDP listener on port 2000). The daemon buffers and batches segments before sending to the X-Ray API. This decouples your application from network latency of the X-Ray API.

---

## 12. AWS Health Dashboard

### Two Components

**AWS Service Health Dashboard** (public): shows the status of all AWS services in all regions. Visible to anyone at https://health.aws.amazon.com.

**AWS Personal Health Dashboard** (account-specific): shows events that specifically affect **your account and resources** — e.g., "your EC2 instances in us-east-1b are affected by degraded hardware."

### AWS Health API

Programmatic access to Personal Health Dashboard data. Two event types:

| Type | Description |
|---|---|
| **Account-specific events** | Directly affect your resources |
| **Public events** | General AWS service issues (visible to all customers) |

### EventBridge Integration

AWS Health events are automatically sent to the **default event bus** in EventBridge under source `aws.health`. This enables automated responses:

```
AWS Health Event (EC2 maintenance scheduled)
  --> EventBridge Rule
    --> Lambda (trigger instance migration before maintenance window)
    --> SNS (notify on-call team)
    --> SSM Automation (drain load balancer, snapshot EBS)
```

> 🎯 **EXAM TIP**: AWS Health via EventBridge is the key integration for **proactive automation** around AWS-initiated maintenance events. If a question says "automatically respond to AWS maintenance notifications," the answer is Health + EventBridge + Lambda/SSM Automation.

### AWS Organizations Health View

In an Organizations management account, you can enable **organizational view** in AWS Health to see health events across all member accounts from a single pane.

---

## 13. AWS Config

### What Config Does (and Why It Is Different)

Config is a **compliance and configuration tracking service**. It continuously records the configuration state of your AWS resources and evaluates them against rules.

The key mental model: Config answers **"how did this resource change over time, and does its current configuration comply with our policies?"**

| Question | Service |
|---|---|
| Is my application healthy right now? | CloudWatch |
| Who deleted that security group yesterday? | CloudTrail |
| Does every S3 bucket have encryption enabled? | Config |
| When was encryption disabled on this S3 bucket? | Config |

### Config Rules

**Managed rules** — AWS-provided rules for common compliance scenarios:

- `encrypted-volumes` — EBS volumes must be encrypted
- `s3-bucket-public-read-prohibited` — no public S3 buckets
- `vpc-sg-open-only-to-authorized-ports` — security groups cannot be wide open
- `iam-password-policy` — IAM password policy must meet requirements
- `required-tags` — resources must have specific tags

**Custom rules** — Lambda functions you write, triggered when resources change or on a schedule. Use for business-specific logic that AWS managed rules don't cover.

**Service Control Policy integration**: Config can detect non-compliant resources that SCPs may have missed (e.g., resources created before SCPs were applied).

### Conformance Packs

A **conformance pack** is a collection of Config rules and remediation actions packaged as a unit. AWS provides pre-built conformance packs for:
- CIS AWS Foundations Benchmark
- PCI DSS
- HIPAA
- NIST 800-53

You can deploy a conformance pack across all accounts in an organization from the management account using **Organizations integration**.

> 🎯 **EXAM TIP**: Conformance packs are the answer when a question asks "how to enforce CIS Benchmark compliance across all accounts in our organization" — not IAM policies, not SCPs (SCPs prevent actions, they don't check existing state).

### Remediation Actions

Config can automatically remediate non-compliant resources using **SSM Automation documents**:

- `AWS-EnableS3BucketEncryption` — auto-enable encryption
- `AWS-DisablePublicAccessForSecurityGroup` — revoke public ingress rules
- Custom SSM documents for business logic

Remediation can be:
- **Manual** — Config marks resource as non-compliant; a human triggers remediation
- **Automatic** — Config triggers SSM Automation immediately on non-compliance detection

> 🎯 **EXAM TIP**: Automatic remediation does NOT guarantee real-time enforcement. There is a delay between the configuration change and when Config evaluates it (typically seconds to minutes for change-triggered rules, up to 24 hours for periodic rules). For real-time enforcement, use **SCPs** (preventive) + Config (detective + corrective).

### Config Aggregator

A **Config aggregator** collects compliance data from multiple accounts and regions into a **central account**.

```
Account A (us-east-1)  ─┐
Account B (us-west-2)  ─┤──> Central Account Aggregator ──> Dashboard
Account C (eu-west-1)  ─┘                                    Athena Query
```

Setup:
1. In the central account: create aggregator specifying source accounts and regions (or use Organizations for automatic enrollment)
2. In each source account: authorize the central account (not required when using Organizations service-linked roles)

With Organizations integration, new accounts are automatically included when they join the organization.

> 🎯 **EXAM TIP**: The Config aggregator is **read-only** — it shows compliance state but does NOT deploy rules to member accounts. To deploy rules to member accounts, use **AWS Config Organization Rules** or **CloudFormation StackSets**. The aggregator only aggregates data that has already been reported by each account's own Config recorder.

### Config vs Organizations Config Rules

| | Config Aggregator | Organization Config Rules |
|---|---|---|
| **Purpose** | Centralize compliance visibility | Deploy rules to all member accounts |
| **Deploys rules?** | No | Yes |
| **Requires Organizations?** | No (can specify account IDs manually) | Yes |
| **Remediation?** | No | Yes, per-account |

---

## 14. Multi-Account Observability Architecture

This is a high-value SAP topic. Large organizations run hundreds of AWS accounts and need centralized observability without manually checking each account.

### Reference Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    MONITORING ACCOUNT                        │
│                                                             │
│  CloudWatch Cross-Account Observability                     │
│    - Linked metrics from all source accounts                │
│    - Unified dashboards                                     │
│    - Cross-account alarms                                   │
│                                                             │
│  Central S3 Log Archive                                     │
│    - CW Logs (via subscription → Firehose → S3)             │
│    - Athena for log analysis                                │
│                                                             │
│  Config Aggregator                                          │
│    - Compliance across all accounts                         │
│    - Organization-wide conformance packs                    │
│                                                             │
│  EventBridge Custom Bus (central-events)                   │
│    - Receives events from all source accounts               │
│    - Routes to SNS, ticketing system, SIEM                 │
└─────────────────────────────────────────────────────────────┘
           ▲              ▲               ▲
           │              │               │
    ┌──────┴───┐   ┌──────┴───┐   ┌──────┴───┐
    │Account A │   │Account B │   │Account C │
    │ CW Agent │   │ CW Agent │   │ CW Agent │
    │ X-Ray    │   │ X-Ray    │   │ Config   │
    │ Config   │   │ Config   │   │ EventBr. │
    └──────────┘   └──────────┘   └──────────┘
```

### Key Integration Points

**Metrics centralization** — CloudWatch cross-account observability (link source accounts to monitoring account). Monitoring account team can view any source account's metrics, alarms, and Log Insights.

**Log centralization** — Subscription filters in each account → Kinesis Data Firehose in monitoring account → S3 bucket. This gives you a tamper-resistant, centralized log archive.

**Event centralization** — EventBridge in each account sends critical events to a central event bus. Use Organizations integration to auto-enroll new accounts.

**Compliance centralization** — Config aggregator (Organizations-based) with conformance packs deployed via Organization Config Rules.

> 🎯 **EXAM TIP**: A question about "how to view CloudWatch metrics from 50 AWS accounts in a single dashboard" should answer with **CloudWatch cross-account observability** (not CloudTrail, not Config, not VPC Flow Logs). This is different from the AWS Config aggregator which only handles Config compliance data.

---

## 15. Decision Framework: Which Service for Which Need

### Quick Reference Table

| Scenario | Primary Service | Notes |
|---|---|---|
| EC2 CPU alert | CloudWatch Alarm | Standard metric, no agent needed |
| Memory/disk alert on EC2 | CloudWatch Alarm + Agent | Agent required for OS-level metrics |
| Alert when two conditions are both true | Composite Alarm | AND logic |
| Sub-minute alerting (10s, 30s) | High-Resolution Metric + Alarm | StorageResolution=1 |
| Query application logs for errors | CloudWatch Logs Insights | Interactive, no ETL |
| Real-time log processing | Subscription Filter → Lambda/Kinesis | Near real-time |
| Centralize logs from all accounts | Subscription → Kinesis Firehose → S3 | Cross-account resource policy needed |
| Monitor actual user experience | CloudWatch RUM | JS snippet in browser |
| Test endpoints proactively | CloudWatch Synthetics | Canary scripts, scheduled |
| SLA availability metrics | CloudWatch Synthetics | Continuous scheduled canaries |
| Feature flags / A/B testing | CloudWatch Evidently | Controlled rollout |
| Container metrics (ECS/EKS) | Container Insights | Fluent Bit for EKS |
| Lambda cold starts and memory | Lambda Insights | Extension layer |
| Trace a request across microservices | AWS X-Ray | Distributed tracing |
| Find which service causes latency | X-Ray Service Map | Visual trace analysis |
| Search traces by business attribute | X-Ray Annotations | Indexed, filterable |
| Respond to EC2 maintenance event | Health + EventBridge + Lambda | Automated pre-maintenance action |
| Ensure all S3 buckets are encrypted | AWS Config Rule | Detective control |
| Auto-fix non-compliant resources | Config + SSM Automation | Remediation action |
| Enforce CIS Benchmark across org | Config Conformance Pack | Organization-level deploy |
| Compliance report across all accounts | Config Aggregator | Central dashboard |
| Trigger action when resource created | EventBridge (event pattern rule) | Source: aws.ec2, aws.rds, etc. |
| Enrich SQS messages before Lambda | EventBridge Pipes | With enrichment step |
| One-time future-dated scheduled job | EventBridge Scheduler | Supports one-time schedules |
| View metrics from all accounts | CloudWatch Cross-Account Observability | Link source accounts |

### CloudWatch vs X-Ray vs Config: When to Use Each

```
Is the problem about metrics and real-time state?
  YES --> CloudWatch (alarms, dashboards, anomaly detection)

Is the problem about understanding a slow/failed request?
  YES --> X-Ray (distributed trace, service map, latency analysis)

Is the problem about configuration compliance or change history?
  YES --> Config (rules, conformance packs, configuration timeline)

Is the problem about triggering automation in response to events?
  YES --> EventBridge (rules, targets, pipes)

Is the problem about understanding what real users experience?
  YES --> CloudWatch RUM

Is the problem about proactively detecting outages?
  YES --> CloudWatch Synthetics
```

---

## Key SAP Exam Patterns

### Pattern 1: Alarm Noise Reduction

**Question**: "Multiple alarms fire simultaneously when a database fails. How to reduce noise?"

**Answer**: Create a **composite alarm** with AND logic. Only notify when both the database alarm AND application alarm are in ALARM state simultaneously. This filters out transient single-service blips.

### Pattern 2: Cross-Account Observability at Scale

**Question**: "Operations team must monitor 200 accounts from a single dashboard. They also need centralized log analysis."

**Answer**: 
1. CloudWatch cross-account observability (link all accounts to monitoring account for metrics)
2. CloudWatch Logs subscription filters → Kinesis Firehose → S3 (for logs)
3. Athena on S3 for log analysis

### Pattern 3: Compliance Drift Detection + Auto-Remediation

**Question**: "Developers sometimes disable S3 bucket encryption. How to auto-detect and fix?"

**Answer**: AWS Config rule (`s3-bucket-server-side-encryption-enabled`) + SSM Automation remediation action (`AWS-EnableS3BucketEncryption`) set to automatic. Config detects the change; SSM re-enables encryption.

### Pattern 4: Proactive vs Reactive Monitoring

**Question**: "Detect API endpoint failures before customers notice."

**Answer**: **CloudWatch Synthetics** canary running every minute, alarming on failure. Not CloudWatch metrics on the API (those are reactive — they only fire after a user request fails).

### Pattern 5: Distributed Tracing for Microservices

**Question**: "A checkout request takes 8 seconds. How to find which service is slow?"

**Answer**: **AWS X-Ray** — the Service Map shows latency per service connection; the trace shows which segment (service) consumed the most time. Enable X-Ray on Lambda, API Gateway, and ECS services that participate in the checkout flow.

---

*Section 11 — Last updated for SAP exam prep. CloudTrail covered in SAP-04 (Security). Next: SAP-12 (Disaster Recovery & Migration).*
