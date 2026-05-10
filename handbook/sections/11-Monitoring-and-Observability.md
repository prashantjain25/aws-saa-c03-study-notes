# Section 11 — Monitoring & Observability

> **Purpose**: You cannot operate what you cannot observe. Monitoring in AWS is not merely about dashboards and alerts — it is about understanding system behavior through metrics, logs, and traces. A senior architect designs observable systems: metrics for health, logs for debugging, traces for latency analysis, and alarms for automated response.
>
> **Official Documentation**: [CloudWatch](https://docs.aws.amazon.com/cloudwatch/) | [X-Ray](https://docs.aws.amazon.com/xray/) | [CloudTrail](https://docs.aws.amazon.com/awscloudtrail/) | [AWS Config](https://docs.aws.amazon.com/config/)

---

## 1. Amazon CloudWatch

### 1.1 CloudWatch Metrics

CloudWatch collects metrics from AWS services and custom applications:

| Metric Source | Granularity | Notes |
|-------------|------------|-------|
| **AWS service metrics** | 1-minute (detailed) or 5-minute (basic) | Automatic for most services |
| **Custom metrics** | 1-minute (standard) or 1-second (high-resolution) | `PutMetricData` API |
| **Container Insights** | 1-minute | Auto-collects from ECS/EKS |
| **Lambda Insights** | 1-minute | Enhanced Lambda monitoring |

**Metric structure**: `Namespace/MetricName [Dimensions]`
- **Namespace**: Logical grouping (e.g., `AWS/EC2`, `MyApplication/Backend`)
- **Dimensions**: Key-value pairs that identify a specific metric stream (e.g., `InstanceId=i-123`, `Environment=Production`)

> **Critical gap**: CloudWatch does NOT monitor EC2 memory (RAM) by default. Install the CloudWatch Agent to publish memory metrics.

### 1.2 CloudWatch Alarms

Alarms evaluate metrics and trigger actions:

| Alarm Type | Behavior | Use Case |
|------------|----------|----------|
| **Static threshold** | Metric > X for N periods | CPU > 80% for 5 minutes |
| **Anomaly detection** | ML-based band; alarm when metric deviates | Dynamic baselines (traffic that varies by time of day) |
| **Metric math** | Alarm on computed expressions | "Error rate > 1%" = `Errors / (Errors + Successes)` |
| **Composite alarm** | Combine multiple alarms with AND/OR | "Alarm if CPU high AND memory high" |

**Alarm states**: `OK` → `INSUFFICIENT_DATA` → `ALARM` (and back)

**Actions**: SNS notification, Auto Scaling action, EC2 action (stop/terminate/recover), Lambda, SSM Automation.

### 1.3 CloudWatch Logs

**Log Groups** and **Log Streams**:
- Log group: Named container (e.g., `/aws/lambda/my-function`)
- Log stream: Sequence of log events from a single source (e.g., one Lambda execution environment)

**CloudWatch Logs Insights**: Query language for log analysis:
```
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as error_count by bin(5m)
| sort error_count desc
```

**Log retention**: Configurable (1 day to indefinite). Costs increase with retention. Use S3 export for long-term archival.

### 1.4 CloudWatch Dashboards and Contributor Insights

- **Dashboards**: Composite views of metrics, logs, and alarms. Can be cross-account.
- **Contributor Insights**: Identify top contributors to metrics (e.g., which IP addresses generate the most 5xx errors, which DynamoDB keys are most accessed).

---

## 2. AWS X-Ray: Distributed Tracing

X-Ray traces requests across AWS services and custom applications:

```
Client ──► API Gateway ──► Lambda ──► DynamoDB
              │              │           │
              └──────────────┴───────────┘
                        X-Ray Trace
```

**Trace structure**:
- **Trace**: End-to-end request path (unique ID)
- **Segment**: Work done by a single service
- **Subsegment**: Detailed breakdown within a service (e.g., DynamoDB query within Lambda)
- **Annotations**: Key-value pairs for filtering and grouping
- **Metadata**: Detailed data not indexed for search

**Service Map**: Visual representation of service dependencies and latency/error rates.

> **X-Ray requires instrumentation**: SDK integration in application code (or Lambda active tracing). Not automatic for all services.

---

## 3. Service Comparison: CloudWatch vs CloudTrail vs Config

| Question | CloudWatch | CloudTrail | Config |
|----------|-----------|------------|--------|
| **What happened?** | Performance metrics, logs | Who called which API when | What did the resource look like at time T? |
| **Query type** | Time-series metrics, log search | Event lookup, Athena SQL | Resource configuration history |
| **Latency** | Real-time (metrics), minutes (logs) | 5-15 minutes | Minutes to hours |
| **Retention** | 15 months (metrics), configurable (logs) | 90 days (event history), indefinite (S3) | Indefinite (with S3) |
| **Primary use** | Health monitoring, alerting, troubleshooting | Security audit, compliance, incident investigation | Configuration compliance, drift detection |

**All three are complementary**. A complete observability stack uses:
- **CloudWatch**: "Is the system healthy?"
- **CloudTrail**: "Who did what?"
- **Config**: "Is the configuration compliant?"

---

## 4. AWS Config: Configuration Compliance

Config records resource configurations and evaluates rules:

**Config Rules**:
- **Managed rules**: AWS-provided (e.g., `s3-bucket-public-read-prohibited`, `ec2-volume-inuse-check`)
- **Custom rules**: Lambda functions evaluating any logic

**Remediation**: SSM Automation actions triggered by non-compliant rules.

**Conformance Packs**: Collections of rules for standards (PCI DSS, HIPAA, NIST).

**Aggregator**: Centralize Config data from multiple accounts and regions.

> **Cost Warning**: Config charges per configuration item recorded. In busy accounts, this adds up. Be selective about which resource types to record.

---

## 5. Operational Best Practices

1. **Enable CloudWatch detailed monitoring** for production EC2/ELB (1-minute granularity)
2. **Set up log aggregation** from all accounts to a central S3 bucket
3. **Create dashboards** for each service with key health metrics
4. **Configure alarms** with SNS → PagerDuty/Slack/Opsgenie for critical metrics
5. **Use anomaly detection** for metrics with variable baselines (traffic, queue depth)
6. **Instrument applications** with X-Ray for distributed tracing
7. **Enable AWS Config** for configuration compliance and drift detection
8. **Set up Organizations CloudTrail** for centralized audit logging
9. **Define SLOs and SLIs** — alarm on SLO breaches, not just infrastructure metrics
10. **Automate responses** — Lambda + SSM for auto-remediation of common issues

---

## 6. Interview Challenges

### Q1: "An EC2 instance appears healthy but users report timeouts. How do you debug?"

**Answer**:
1. **CloudWatch**: Check CPU, network, disk I/O. High CPU? Check `CPUUtilization`. Network saturated? Check `NetworkIn/Out`.
2. **CloudWatch Logs**: Check application logs for errors, slow queries, or exceptions.
3. **X-Ray**: If instrumented, trace the request path to identify which segment adds latency.
4. **ALB Target Group**: Check target health. Instance might be failing health checks (returning 5xx on `/health`) even if EC2 status checks pass.
5. **RDS/Database**: Check database connection count and slow query log. Connection pool exhaustion causes timeouts with healthy CPU.
6. **Security Groups/NACLs**: Recent change might be blocking egress traffic (e.g., database port).

---

## 7. Points to Remember

- **CloudWatch does not monitor memory by default** — install CloudWatch Agent.
- **CloudTrail is eventually consistent** — expect 5-15 minute delays.
- **Config records configuration state; CloudTrail records API calls** — they answer different questions.
- **X-Ray requires instrumentation** — enable active tracing in Lambda, use SDK in applications.
- **Anomaly detection alarms adapt to baselines** — better than static thresholds for variable workloads.
- **Log retention costs increase with time** — export to S3 and use lifecycle policies for cost control.
- **CloudWatch Contributor Insights identifies top contributors** — use to find which IPs, users, or resources cause errors.

---

*Section 11 — Monitoring & Observability | Last Validated: 2026-05-10*
