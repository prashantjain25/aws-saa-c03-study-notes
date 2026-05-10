# AWS SAA-C03: Monitoring, CloudWatch, CloudTrail & Config

## Table of Contents
1. [Amazon CloudWatch — Metrics](#amazon-cloudwatch--metrics)
2. [CloudWatch Alarms](#cloudwatch-alarms)
3. [CloudWatch Logs](#cloudwatch-logs)
4. [CloudWatch Container Insights](#cloudwatch-container-insights)
5. [CloudWatch Lambda Insights](#cloudwatch-lambda-insights)
6. [CloudWatch Contributor Insights](#cloudwatch-contributor-insights)
7. [CloudWatch Application Insights](#cloudwatch-application-insights)
8. [Amazon EventBridge](#amazon-eventbridge)
9. [AWS CloudTrail](#aws-cloudtrail)
10. [AWS Config](#aws-config)
11. [Critical Comparison: CloudWatch vs CloudTrail vs Config](#critical-comparison-cloudwatch-vs-cloudtrail-vs-config)
12. [Interview Tips and Exam Patterns](#interview-tips-and-exam-patterns)
13. [Points to Remember (Exam Focus)](#points-to-remember-exam-focus)
14. [AWS CLI Quick Reference](#aws-cli-quick-reference)

---

## Amazon CloudWatch — Metrics

### What is CloudWatch?

Think of **CloudWatch** as the "nervous system" of your AWS infrastructure. It's a **monitoring + observability service** that continuously watches your AWS resources, collects metrics, stores logs, and lets you set alarms when things go wrong. Without CloudWatch, you'd be flying blind—you wouldn't know if your EC2 instances are overloaded, if your Lambda functions are failing, or if your application performance is degrading.

> **Tutor Tip**: CloudWatch is not just about metrics. It's a complete observability platform: metrics (quantitative data), logs (textual data), and alarms (actions). Think of metrics as your "vital signs" and logs as your "medical history."

### EC2 Default Metrics

Every EC2 instance automatically sends metrics to CloudWatch **every 5 minutes** (or every 1 minute with detailed monitoring enabled). These are provided **at no extra charge**:

- **CPUUtilization**: percentage of CPU used
- **NetworkIn/NetworkOut**: bytes received/sent
- **DiskReadOps/DiskWriteOps**: disk I/O operations
- **StatusCheckFailed**: instance status checks

**Critical Exam Point**: RAM (memory usage) is NOT in the default metrics. If you need to monitor RAM, you MUST install the **CloudWatch Agent** on your EC2 instance.

```bash
# Enable detailed monitoring (1-minute intervals)
aws ec2 monitor-instances --instance-ids i-1234567890abcdef0 --region us-east-1
```

### Custom Metrics

You can publish **custom metrics** to CloudWatch using the `PutMetricData` API. Custom metrics allow you to track application-specific data like "Orders Processed Per Minute" or "Cache Hit Rate."

**Two Resolution Options**:
1. **Standard Resolution** (default): 1-minute intervals, free tier includes 10 metrics
2. **High-Resolution** (`StorageResolution=1`): 1-second intervals, higher cost but for real-time monitoring

```bash
# Put a custom metric (standard resolution, 1-minute)
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "OrdersProcessed" \
  --value 150 \
  --unit Count \
  --region us-east-1

# Put a high-resolution metric (1-second intervals)
aws cloudwatch put-metric-data \
  --namespace "MyApp/HighResolution" \
  --metric-name "APILatency" \
  --value 45 \
  --unit Milliseconds \
  --storage-resolution 1 \
  --region us-east-1
```

### Metric Namespaces and Dimensions

A **namespace** is a logical grouping: `AWS/EC2`, `AWS/RDS`, `MyApp`, etc. **Dimensions** are key-value pairs that identify a specific metric:

```bash
# Create metric with dimensions
aws cloudwatch put-metric-data \
  --namespace "WebApp" \
  --metric-name "ResponseTime" \
  --value 120 \
  --dimensions Name=Environment,Value=Production Name=Service,Value=UserAPI \
  --region us-east-1
```

> **Why Dimensions Matter**: Without dimensions, all instances look the same. With dimensions, you can track "Response Time for Production UserAPI" vs "Response Time for Staging UserAPI" separately.

### CloudWatch Agent: Unlocking Full Monitoring

The **CloudWatch Agent** is software you install on your EC2 instance (or on-premises server) that enables:

- **RAM usage** (memory monitoring)
- **Disk space utilization**
- **Log file monitoring**
- **Custom application metrics**
- **More detailed OS-level metrics**

There are two versions:

1. **Unified CloudWatch Agent** (newer, recommended): Handles both metrics AND logs, supports Systems Manager Parameter Store for configuration
2. **CloudWatch Logs Agent** (legacy): Logs only, being phased out

**Installation Flow**:

<p align="center"><img src="diagrams/icons/named/ec2.png" width="52" title="EC2"/>&nbsp;&nbsp;<b>→ Install Agent →</b>&nbsp;&nbsp;<img src="diagrams/icons/named/lambda.jpg" width="52" title="IAM Role"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/cloudfront.png" width="52" title="CloudWatch"/></p>

```bash
# Install Unified CloudWatch Agent (Amazon Linux 2)
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm

# Start the agent (config in /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

### Points to Remember
- **Default metrics**: CPU, network, disk I/O (NOT RAM)
- **CloudWatch Agent**: enables RAM, disk space, custom metrics
- **Custom metrics**: PutMetricData API with dimensions
- **Standard vs High-Resolution**: 1-minute vs 1-second intervals
- **Namespace + Dimensions**: organize and identify metrics

---

## CloudWatch Alarms

### The Three States

Every CloudWatch alarm has exactly **three possible states**:

1. **OK**: The metric is within the threshold
2. **ALARM**: The metric exceeded the threshold
3. **INSUFFICIENT_DATA**: Not enough data points to evaluate (happens right after alarm creation)

> **Why This Matters**: An alarm stuck in INSUFFICIENT_DATA means it can't evaluate yet. Don't panic—it will transition once enough data is collected (usually after the evaluation period).

### What Alarms Can Trigger

When an alarm enters **ALARM** state, it can trigger:

- **EC2 Actions**: Stop, terminate, or reboot an instance
- **Auto Scaling Actions**: Scale up/down capacity
- **SNS Notifications**: Send email/SMS to administrators
- **Lambda Functions**: Custom automation (via SNS target)
- **Systems Manager Automation**: Trigger remediation documents

```bash
# Create an alarm to stop an EC2 instance if CPU > 80% for 5 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name "High-CPU-Stop-Instance" \
  --alarm-description "Stop instance if CPU sustained above 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:autoscaling:us-east-1:123456789012:lifecycleHookTarget:asgName:hookName \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --region us-east-1
```

### Period and Evaluation

- **Period**: The duration (in seconds) to evaluate the metric (e.g., 60s, 300s, 3600s)
- **Evaluation Periods**: How many consecutive periods must breach the threshold before ALARM state
- **Datapoint to Alarm**: Minimum number of data points that must breach (e.g., 1 out of 3 periods)

Example: Period=300s, EvaluationPeriods=2 means "If average CPU > 80% for TWO consecutive 5-minute periods, trigger alarm."

### Composite Alarms: AND/OR Logic

**Composite Alarms** combine multiple alarms with logical operators:

```bash
# Create composite alarm: Fire if BOTH database AND application alarms are in ALARM state
aws cloudwatch put-composite-alarm \
  --alarm-name "Critical-System-Failure" \
  --alarm-description "Alert if both DB and App fail" \
  --alarm-rule "ALARM(db-alarm) AND ALARM(app-alarm)" \
  --actions-enabled \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:AlertTopic" \
  --region us-east-1
```

> **When to Use**: Composite alarms prevent "alarm fatigue." Instead of getting 10 alerts, you get 1 alert when the COMBINATION of problems occurs.

### EC2 Instance Recovery

CloudWatch can **automatically recover** a failed EC2 instance (different from reboot):

- **Recover** = restart instance on different hardware (if original hardware failed)
- **Reboot** = soft restart on same hardware
- **Stop** = powered off
- **Terminate** = delete forever

```bash
# Create alarm to recover instance if status check fails
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-StatusCheck-Recovery" \
  --metric-name StatusCheckFailed_System \
  --namespace AWS/EC2 \
  --statistic Minimum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions "arn:aws:automate:us-east-1:ec2:recover" \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --region us-east-1
```

### Testing Alarms

You can test alarm logic without waiting for real data:

```bash
# Manually set alarm to ALARM state (for testing)
aws cloudwatch set-alarm-state \
  --alarm-name "High-CPU-Alarm" \
  --state-value ALARM \
  --state-reason "Testing alarm for CPU monitoring" \
  --region us-east-1

# Set alarm back to OK
aws cloudwatch set-alarm-state \
  --alarm-name "High-CPU-Alarm" \
  --state-value OK \
  --state-reason "Test complete" \
  --region us-east-1
```

### Points to Remember
- **Three states**: OK, ALARM, INSUFFICIENT_DATA
- **Period**: evaluation window (60s-3600s typical)
- **Evaluation Periods**: consecutive breaches needed
- **Actions**: EC2, ASG, SNS, Lambda, SSM
- **Composite Alarms**: combine multiple alarms with AND/OR
- **Recovery**: restart on different hardware vs reboot

---

## CloudWatch Logs

### Structure: Log Groups → Log Streams → Log Events

Think of CloudWatch Logs like a **hierarchical filing cabinet**:

- **Log Group**: Top-level container (e.g., `/aws/lambda/my-function`)
- **Log Stream**: Sub-container within a group (e.g., stream name = request ID)
- **Log Event**: Individual log messages with timestamps

```bash
# Create a log group
aws logs create-log-group \
  --log-group-name "/my-app/production" \
  --region us-east-1

# Put log events into a stream
aws logs put-log-events \
  --log-group-name "/my-app/production" \
  --log-stream-name "instance-1" \
  --log-events timestamp=$(date +%s000),message="Application started" \
  --region us-east-1
```

### Log Retention Policies

By default, logs are stored **indefinitely** (never expire). You can set **retention** to:
- 1 day, 3 days, 5 days, 1 week, 2 weeks, 1 month, 2 months, 3 months, 6 months, 1 year, 3 years, 5 years, 10 years, or Never Expire

> **Cost Implication**: Longer retention = higher cost. Set appropriate retention to balance compliance needs with budget.

```bash
# Set log retention to 7 days
aws logs put-retention-policy \
  --log-group-name "/my-app/production" \
  --retention-in-days 7 \
  --region us-east-1
```

### Log Sources: Where Logs Come From

CloudWatch Logs receives logs from:
- **SDK**: Direct calls from your application
- **Lambda**: Automatic logging to `/aws/lambda/function-name`
- **ECS/Fargate**: Container logs via awslogs driver
- **EKS**: Via DaemonSet or Fargate logging
- **Elastic Beanstalk**: Application logs
- **CloudTrail**: API call logs
- **Route53**: Query logs
- **Custom Agents**: CloudWatch Agent, Fluent Bit, Logstash

```bash
# Get log events from a stream
aws logs get-log-events \
  --log-group-name "/my-app/production" \
  --log-stream-name "instance-1" \
  --start-time 1609459200000 \
  --region us-east-1
```

### CloudWatch Logs Insights: Query Language

**CloudWatch Logs Insights** is a query language for analyzing logs in real-time (without loading them into expensive data warehouses).

```bash
# Start a query (returns queryId)
aws logs start-query \
  --log-group-name "/my-app/production" \
  --start-time 1609459200 \
  --end-time 1609545600 \
  --query-string 'fields @timestamp, @message | stats count() by bin(5m)' \
  --region us-east-1

# Get query results (may need to poll)
aws logs get-query-results --query-id "query-id-from-above" --region us-east-1
```

**Common Query Patterns**:
```
# Count errors
fields @timestamp, @message | filter @message like /ERROR/ | stats count()

# Top IPs
fields srcIP | stats count() as requests by srcIP | sort requests desc

# Response time percentiles
fields responseTime | stats pct(responseTime, 50), pct(responseTime, 99)
```

### Log Subscriptions: Real-Time Log Delivery

**Subscriptions** deliver log events to another service **in real-time** as they arrive:

- **Lambda**: Process logs immediately (e.g., send alerts)
- **Kinesis Data Firehose**: Buffer and load to S3, Splunk, DataDog
- **Kinesis Data Streams**: For custom real-time processing

<p align="center"><img src="diagrams/icons/named/cloudfront.png" width="52" title="CloudWatch Logs"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/lambda.jpg" width="52" title="Lambda"/>&nbsp;&nbsp;<b>or</b>&nbsp;&nbsp;<img src="diagrams/icons/named/kinesis.jpg" width="52" title="Kinesis"/></p>

```bash
# Create subscription to Lambda
aws logs put-subscription-filter \
  --log-group-name "/my-app/production" \
  --filter-name "SendErrorsToLambda" \
  --filter-pattern "[... ERROR ...]" \
  --destination-arn "arn:aws:lambda:us-east-1:123456789012:function:ProcessLogs" \
  --region us-east-1
```

> **Key Insight**: Subscriptions are **asynchronous**. Logs are batched and sent in near real-time, not immediately per event.

### Cross-Account Log Aggregation

You can subscribe to logs from Account A and deliver them to a Kinesis stream in Account B:

1. Create subscription filter in Account A pointing to Account B's resource
2. Add resource-based policy in Account B to allow Account A's logs

```bash
# In Account A: Subscribe to Account B's Kinesis
aws logs put-subscription-filter \
  --log-group-name "/my-app/production" \
  --filter-name "AggregateToAccountB" \
  --filter-pattern "" \
  --destination-arn "arn:aws:kinesis:us-east-1:ACCOUNT-B:stream/AggregatedLogs" \
  --destination-account-id ACCOUNT-B \
  --region us-east-1
```

### Metric Filters: Logs → Metrics

Convert log patterns to CloudWatch metrics:

```bash
# Create metric filter for 4xx errors
aws logs put-metric-filter \
  --log-group-name "/my-app/production" \
  --filter-name "4xxErrors" \
  --filter-pattern "[ip, id, user, timestamp, request, status_code=4*, size]" \
  --metric-transformations \
    metricName=4xxCount,metricNamespace=MyApp,metricValue=1 \
  --region us-east-1
```

Then create an alarm on this metric:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "High-4xx-Errors" \
  --metric-name 4xxCount \
  --namespace MyApp \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:AlertTopic" \
  --region us-east-1
```

### Points to Remember
- **Hierarchy**: Log Group → Stream → Event
- **Retention**: 1 day to 10 years, or never expire
- **Sources**: Lambda, ECS, CloudTrail, custom agents
- **Logs Insights**: SQL-like query language
- **Subscriptions**: Real-time delivery to Lambda/Kinesis
- **Metric Filters**: Convert log patterns to metrics

---

## CloudWatch Container Insights

**Container Insights** provides deep visibility into containerized applications:

- **ECS**: Cluster, service, task metrics
- **EKS**: Cluster, node, pod metrics
- **Kubernetes**: On-premises or EC2 clusters
- **Fargate**: Automatic metrics (no agent installation needed for some metrics)

For **EKS**, you deploy **FluentD** or **Fluent Bit** as a DaemonSet to collect metrics from each node:

```bash
# Deploy Fluent Bit to EKS cluster
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name aws-for-fluent-bit \
  --region us-east-1
```

> **Why Container Insights**: Traditional CloudWatch metrics don't include "How many tasks are running in this ECS service?" Container Insights answers those questions.

### Points to Remember
- **ECS/EKS/Kubernetes** supported
- **Fargate**: Requires Container Insights
- **EKS**: Deploy Fluent Bit or FluentD
- **Automatic dashboards** provided

---

## CloudWatch Lambda Insights

**Lambda Insights** is an **extension layer** you add to your Lambda function for detailed metrics:

- **Duration**
- **Memory used**
- **Init duration** (cold starts)
- **Log parsing** for errors and warnings
- **Cost estimation**

```bash
# Add Lambda Insights extension layer to function
aws lambda update-function-configuration \
  --function-name my-function \
  --layers "arn:aws:lambda:us-east-1:580254703988:layer:LambdaInsightsExtension:21" \
  --region us-east-1
```

> **Layer Architecture**: The layer contains a small agent that sends metrics to CloudWatch Logs Insights, then transforms them to CloudWatch Metrics.

### Points to Remember
- **Extension layer**: Separate monitoring layer
- **Metrics**: Duration, memory, cold start duration
- **Automatic dashboards** for Lambda performance

---

## CloudWatch Contributor Insights

Identifies **top-N contributors** to your logs. Examples:

- Top IPs making requests
- Top users with errors
- Top API endpoints by latency

```bash
# Create contributor insights rule
aws logs put-insight-rule \
  --insight-rule-name "TopErrorIPs" \
  --insight-rule-state ENABLED \
  --log-group-pattern-rules logGroupPattern=/my-app/production \
  --region us-east-1
```

### Points to Remember
- **Top-N analysis** of log patterns
- **Auto-aggregation** of contributors
- **Cost**: Per rule evaluation

---

## CloudWatch Application Insights

**Application Insights** automatically creates dashboards for your applications using **SageMaker-powered anomaly detection**:

- **.NET applications**: monitors CLR, Windows Server, SQL Server
- **Java applications**: monitors JVM
- **SQL Server**: monitors databases

No code changes needed—it discovers your application automatically.

> **When to Use**: You have a business-critical application that you want monitored automatically without manual dashboard creation.

### Points to Remember
- **Automatic dashboards** via machine learning
- **.NET, Java, SQL Server** supported
- **Anomaly detection** included

---

## Amazon EventBridge

Previously called **CloudWatch Events**, EventBridge is a serverless **event-driven architecture** service.

### Event Buses

EventBridge has three types of event buses:

1. **Default Event Bus**: AWS events (EC2, RDS, etc.)
2. **Partner Event Bus**: Events from SaaS partners
3. **Custom Event Bus**: Your own application events

```bash
# Create custom event bus
aws events create-event-bus \
  --name "my-custom-events" \
  --region us-east-1
```

### Rules: Schedule vs Event Pattern

**Rules** define what events to match. Two types:

1. **Schedule Rule**: Cron expression or rate
```bash
aws events put-rule \
  --name "DailyBackup" \
  --schedule-expression "cron(0 2 * * ? *)" \
  --region us-east-1
```

2. **Event Pattern Rule**: Match event structure
```bash
aws events put-rule \
  --name "EC2StateChange" \
  --event-pattern '{"source":["aws.ec2"],"detail-type":["EC2 Instance State-change Notification"],"detail":{"state":["running"]}}' \
  --region us-east-1
```

### Targets: Where Events Go

A rule can have multiple **targets**:

<p align="center"><img src="diagrams/icons/named/ec2.png" width="52" title="EC2"/>&nbsp;&nbsp;<b>Event</b>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/eventbridge.png" width="52" title="EventBridge"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/lambda.jpg" width="52" title="Lambda"/>&nbsp;&nbsp;<b>, SNS, SQS, Step Functions</b></p>

```bash
# Add Lambda as target
aws events put-targets \
  --rule "EC2StateChange" \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:ProcessEC2Event" \
  --region us-east-1

# Add multiple targets (SNS + Lambda)
aws events put-targets \
  --rule "EC2StateChange" \
  --targets \
    "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:ProcessEC2Event" \
    "Id"="2","Arn"="arn:aws:sns:us-east-1:123456789012:AlertTopic" \
  --region us-east-1
```

**Target Types**:
- Lambda, SNS, SQS, Kinesis, Firehose
- Step Functions, CodePipeline, CodeBuild
- Systems Manager Automation, EC2 Actions
- ECS Task, Batch Job

### Event Bus Permissions: Cross-Account

Allow another AWS account to send events to your event bus:

```bash
# In Account B: Add resource-based policy allowing Account A
aws events put-permission \
  --action events:PutEvents \
  --principal ACCOUNT-A-ID \
  --statement-id AllowAccountAPutEvents \
  --region us-east-1
```

Then from Account A, put events to Account B's bus:
```bash
# In Account A: Send event to Account B's event bus
aws events put-events \
  --entries '[
    {
      "Source": "my-app",
      "DetailType": "CustomEvent",
      "Detail": "{\"key\": \"value\"}",
      "EventBusName": "arn:aws:events:us-east-1:ACCOUNT-B:event-bus/my-custom-bus"
    }
  ]' \
  --region us-east-1
```

### Schema Registry: Event Discovery

**Schema Registry** stores event schemas so you can auto-generate code:

```bash
# Discover available event schemas
aws schemas list-registries --region us-east-1

# Get schema details
aws schemas describe-schema \
  --registry-name aws.events \
  --schema-name aws.events.ec2.ec2-instance-state-change-notification \
  --region us-east-1
```

### Replay: Archive and Replay Events

EventBridge can **archive events** and **replay** them later:

```bash
# Create archive for events
aws events create-event-bus-archive \
  --archive-name "MyArchive" \
  --event-bus-name "my-custom-bus" \
  --event-pattern '{}' \
  --retention-days 30 \
  --region us-east-1

# Replay events from archive
aws events replay-archived-events \
  --archive-name "MyArchive" \
  --event-bus-name "my-custom-bus" \
  --start-time 2025-01-01T00:00:00Z \
  --end-time 2025-01-02T00:00:00Z \
  --region us-east-1
```

### Dead Letter Queue (DLQ)

If event delivery fails, send to **DLQ** (SQS or SNS):

```bash
# Add target with DLQ
aws events put-targets \
  --rule "ProcessOrder" \
  --targets \
    "Id"="1" \
    "Arn"="arn:aws:lambda:us-east-1:123456789012:function:ProcessOrder" \
    "DeadLetterConfig"="{"Arn":"arn:aws:sqs:us-east-1:123456789012:OrderDLQ"}" \
  --region us-east-1
```

### Points to Remember
- **Event buses**: Default, Partner, Custom
- **Rules**: Schedule (Cron/Rate) vs Event Pattern
- **Targets**: Lambda, SNS, SQS, Step Functions, etc.
- **Cross-account**: Via resource-based policies
- **Schema Registry**: Discover event structures
- **Replay**: Archive and replay events
- **DLQ**: Handle failed deliveries

---

## AWS CloudTrail

### What It Does: API Audit Logging

**CloudTrail** records **every API call** made to your AWS account. Think of it as a **security camera for your infrastructure**—every action (who, what, when) is logged.

> **Critical Distinction**: CloudTrail logs PAST events (audit), CloudWatch monitors CURRENT state (real-time).

### Event Types

CloudTrail logs three types of events:

1. **Management Events** (default, enabled): Create/delete resources (EC2, RDS, IAM, etc.)
2. **Data Events** (optional): S3 GetObject/PutObject, Lambda Invoke, DynamoDB operations
3. **Insights Events** (optional): Unusual API activity (unusual rate, unusual pattern)

```bash
# Create a CloudTrail to log all events
aws cloudtrail create-trail \
  --name "MyTrail" \
  --s3-bucket-name "my-cloudtrail-bucket" \
  --region us-east-1

# Enable management + data events
aws cloudtrail put-event-selectors \
  --trail-name "MyTrail" \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [{
      "Type": "AWS::S3::Object",
      "Values": ["arn:aws:s3:::my-bucket/*"]
    }]
  }]' \
  --region us-east-1
```

### Trail Configuration

A **Trail** logs to:
- **S3**: Long-term storage, analyze with Athena
- **CloudWatch Logs**: Real-time monitoring + alarms
- **Both**: Belt-and-suspenders approach

```bash
# Create trail with both S3 and CloudWatch Logs
aws cloudtrail create-trail \
  --name "MyTrail" \
  --s3-bucket-name "my-cloudtrail-bucket" \
  --cloud-watch-logs-log-group-arn "arn:aws:logs:us-east-1:123456789012:log-group:/aws/cloudtrail/MyTrail" \
  --cloud-watch-logs-role-arn "arn:aws:iam::123456789012:role/CloudTrailLogsRole" \
  --region us-east-1

# Start logging
aws cloudtrail start-logging --trail-name "MyTrail" --region us-east-1

# Check trail status
aws cloudtrail get-trail-status --trail-name "MyTrail" --region us-east-1
```

### Multi-Region Trail

A **multi-region trail** logs events from all regions. Single-region trails are cheaper but less comprehensive:

```bash
# Create multi-region trail (logs from all regions)
aws cloudtrail create-trail \
  --name "GlobalTrail" \
  --s3-bucket-name "my-cloudtrail-bucket" \
  --is-multi-region-trail \
  --region us-east-1
```

### CloudTrail Insights

**Insights** automatically detects unusual API activity:

```bash
# Enable Insights for unusual API calls
aws cloudtrail put-insight-selectors \
  --trail-name "MyTrail" \
  --insight-selectors '[{"InsightType": "ApiCallRateInsight"}]' \
  --region us-east-1
```

When unusual activity is detected, CloudTrail logs an **Insights event** that you can query.

### Event Retention

- **CloudTrail Console**: Last 90 days (free)
- **S3 Bucket**: As long as you keep objects (set lifecycle policies for cost)
- **CloudWatch Logs**: Per log retention policy

> **Compliance Strategy**: Store in S3 for 7 years (typical compliance requirement), query with Athena.

```bash
# Query CloudTrail logs in S3 with Athena
aws athena start-query-execution \
  --query-string "SELECT eventTime, userIdentity, eventSource, eventName FROM cloudtrail_logs WHERE year='2025' AND month='03' LIMIT 100" \
  --query-execution-context "Database=cloudtrail_logs" \
  --result-configuration "OutputLocation=s3://my-athena-results/" \
  --region us-east-1
```

### Looking Up Events

Query recent events (last 90 days) directly:

```bash
# Lookup EC2 RunInstances events
aws cloudtrail lookup-events \
  --lookup-attributes "AttributeKey=EventName,AttributeValue=RunInstances" \
  --region us-east-1

# Lookup events by username
aws cloudtrail lookup-events \
  --lookup-attributes "AttributeKey=Username,AttributeValue=john.doe@example.com" \
  --start-time 2025-01-01T00:00:00Z \
  --end-time 2025-01-31T23:59:59Z \
  --region us-east-1
```

### Points to Remember
- **Purpose**: Audit all API calls (who, what, when)
- **Event types**: Management, Data, Insights
- **Storage**: S3 + CloudWatch Logs for comprehensive coverage
- **Retention**: 90 days in console, longer in S3
- **Multi-region**: Single trail logs from all regions
- **CloudTrail Insights**: Detect unusual activity

---

## AWS Config

### What It Does: Resource Configuration Audit

**AWS Config** is a **resource compliance service** that:

1. Records resource configuration changes **over time**
2. Evaluates resources against rules (compliant or non-compliant)
3. Provides dashboards showing compliance state
4. Can **automatically remediate** non-compliant resources

Think of Config as **"the before and after photos"** of your infrastructure. CloudWatch tells you what's happening NOW; Config tells you HOW IT CHANGED and IF IT VIOLATED POLICY.

<p align="center"><img src="diagrams/icons/named/ec2.png" width="52" title="EC2"/>&nbsp;&nbsp;<b>→ Configuration Changes →</b>&nbsp;&nbsp;<img src="diagrams/icons/named/eventbridge.png" width="52" title="Config"/>&nbsp;&nbsp;<b>→ S3</b></p>

### Config Rules: Compliance Checks

A **Config Rule** is a compliance check. Two types:

1. **Managed Rules**: AWS-provided (e.g., "ec2-ebs-encryption-enabled")
2. **Custom Rules**: Lambda functions you write

```bash
# Enable AWS Config
aws configservice start-configuration-recorder \
  --configuration-recorder-name default \
  --region us-east-1

# Create managed rule: All EC2 instances must be encrypted
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "ec2-ebs-encryption-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "ENCRYPTED_VOLUMES"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::EC2::Volume"]
    }
  }' \
  --region us-east-1

# Create custom rule with Lambda
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "custom-ami-compliance",
    "Source": {
      "Owner": "CUSTOM_LAMBDA",
      "SourceIdentifier": "arn:aws:lambda:us-east-1:123456789012:function:CheckAMI",
      "SourceDetails": [{
        "EventSource": "aws.config",
        "MessageType": "ConfigurationItemChangeNotification"
      }]
    }
  }' \
  --region us-east-1
```

### Compliance Dashboard

Config provides a dashboard showing:

- **Compliant**: Resources pass the rule
- **Non-Compliant**: Resources fail the rule
- **Not Applicable**: Resource type doesn't apply to rule

```bash
# Get compliance summary
aws configservice describe-compliance-by-config-rule --region us-east-1

# Get non-compliant resources for a rule
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name "ec2-ebs-encryption-enabled" \
  --region us-east-1
```

### Automatic Remediation

Config can **automatically fix** non-compliant resources using **SSM Automation documents**:

```bash
# Create config rule with auto-remediation
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "required-tags",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "REQUIRED_TAGS"
    }
  }' \
  --region us-east-1

# Add remediation action
aws configservice put-remediation-configs \
  --remediation-configs '[{
    "ConfigRuleName": "required-tags",
    "TargetType": "SSM_DOCUMENT",
    "TargetIdentifier": "AWS-AddTagsToResource",
    "TargetVersion": "1",
    "Parameters": {
      "AutomationAssumeRole": {"StaticValue": {"Values": ["arn:aws:iam::123456789012:role/AutoRemediationRole"]}},
      "TagKey": {"StaticValue": {"Values": ["Environment"]}},
      "TagValue": {"StaticValue": {"Values": ["Production"]}}
    },
    "Automatic": true,
    "MaximumAutomaticAttempts": 5,
    "RetryAttemptSeconds": 60
  }]' \
  --region us-east-1
```

> **Remediation Strategy**: Start with **manual review** (Automatic=false), then enable automatic once you trust the rule.

### Config Aggregator: Multi-Account, Multi-Region

A **Aggregator** collects compliance data from multiple accounts + regions into a **central account**:

```bash
# In central account: Create aggregator
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name "CentralAggregator" \
  --account-aggregation-sources '[{
    "AccountIds": ["111111111111", "222222222222"],
    "AwsRegions": ["us-east-1", "us-west-2"]
  }]' \
  --region us-east-1

# In member accounts: Authorize central account
aws configservice put-aggregation-authorization \
  --authorized-account-id "central-account-id" \
  --authorized-aws-region "us-east-1" \
  --region us-east-1
```

Then query compliance across all accounts:
```bash
# Get compliance data across aggregator
aws configservice get-aggregate-compliance-details-by-config-rule \
  --configuration-aggregator-name "CentralAggregator" \
  --config-rule-name "ec2-ebs-encryption-enabled" \
  --account-id "111111111111" \
  --aws-region "us-east-1" \
  --region us-east-1
```

### Config Recorder and Snapshots

The **Config Recorder** captures resource configurations:

```bash
# Start config recorder
aws configservice start-configuration-recorder \
  --configuration-recorder-name "default" \
  --region us-east-1

# Create S3 bucket for config snapshots
aws s3api create-bucket \
  --bucket "my-config-bucket" \
  --region us-east-1

# Configure delivery channel (where configs are stored)
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "my-config-bucket",
    "configSnapshotDeliveryProperties": {
      "deliveryFrequency": "TwentyFour_Hours"
    }
  }' \
  --region us-east-1
```

### Pricing Model

AWS Config charges per:
- **Configuration Item** (each resource's configuration recorded)
- **Rule Evaluation** (each rule evaluation per resource)

Typical cost: ~$2-5 per rule per month + per-config-item charge.

### Points to Remember
- **Purpose**: Track resource configuration compliance over time
- **Rules**: Managed (AWS-provided) vs Custom (Lambda)
- **Remediation**: Auto-fix non-compliant resources
- **Aggregator**: Multi-account, multi-region view
- **Storage**: S3 bucket for configurations
- **Pricing**: Per config item + rule evaluation

---

## Critical Comparison: CloudWatch vs CloudTrail vs Config

This is a **critical exam pattern**. Many questions test your ability to distinguish:

| Aspect | **CloudWatch** | **CloudTrail** | **Config** |
|--------|---|---|---|
| **Purpose** | Monitor CURRENT state | Audit WHO DID WHAT | Track resource CONFIGURATIONS |
| **Data Type** | Metrics + Logs | API call logs | Resource configurations |
| **Time Scope** | Real-time/recent | Last 90 days (console) | Configuration history |
| **Use Case** | "Is my app healthy?" | "Who deleted the database?" | "Did we violate compliance?" |
| **Actions** | Alarms, dashboards | Forensics, compliance | Remediation, compliance |
| **Example** | CPU at 95%, trigger alert | EC2 TerminateInstances call at 2pm | EC2 not encrypted (non-compliant) |
| **Storage** | CloudWatch (short term) | S3 (long term) | S3 (long term) |
| **Cost** | Per metric | Per 100K events | Per config item + rule |

### Exam Question Pattern

**"You need to find out which IAM user deleted an RDS database yesterday."**
→ Answer: CloudTrail (audit log)

**"You need to ensure all S3 buckets have encryption enabled."**
→ Answer: Config with Managed Rule

**"Your application's response time is degrading. You need to know why."**
→ Answer: CloudWatch Logs Insights + Metrics

**"Compliance officer asks: 'Show me the history of changes to our security group rules.'"**
→ Answer: Config (configuration timeline)

---

## Interview Tips and Exam Patterns

### Pattern 1: "Monitoring → Action" Chain

Many exam questions follow this pattern:

```
Metric (CloudWatch) → Alarm (threshold) → Action (EC2/ASG/SNS)
```

Example: "When CPU > 80% for 5 minutes, send SNS alert and scale up ASG."

**Key CLI concept**: `put-metric-alarm` with `--alarm-actions` target.

### Pattern 2: "Log Analysis" Scenarios

Questions often ask about log aggregation:

```
App Logs (EC2/Lambda) → CloudWatch Logs Insights (query) → Detect errors
```

Or:

```
Logs → Subscription Filter → Lambda (process) → SNS (alert)
```

**Key Point**: Subscriptions are real-time but ASYNC (batched).

### Pattern 3: "API Audit" Questions

If the question mentions "compliance," "audit," "who," "when," or "forensics," think **CloudTrail**.

### Pattern 4: "Configuration Compliance"

If the question mentions "ensure all resources," "compliance," or "automatic remediation," think **Config**.

### Pattern 5: "Real-Time Events"

If the question mentions "trigger action when resource changes," think **EventBridge** (formerly CloudWatch Events).

### High-Value Interview Talking Points

1. **"Why not just use CloudWatch for everything?"** Because CloudWatch doesn't audit API calls (CloudTrail does) and doesn't track configuration compliance (Config does).

2. **"Why CloudWatch Agent?"** Default metrics don't include RAM. Many monitoring gaps require agents.

3. **"Why EventBridge over CloudWatch Events?"** EventBridge is the evolution—more targets, better integration, schema registry.

4. **"Why Config Rules?"** Automated compliance checking is cheaper than manual reviews.

5. **"Why cross-account log aggregation?"** Central security team monitors all accounts from one place.

---

## Points to Remember (Exam Focus)

### CloudWatch Metrics
- **Default EC2 metrics**: CPU, network, disk I/O (NOT RAM)
- **Custom metrics**: PutMetricData API with dimensions
- **High-resolution**: 1-second intervals vs standard 1-minute
- **CloudWatch Agent**: Unlocks RAM, disk space, custom metrics

### CloudWatch Alarms
- **Three states**: OK, ALARM, INSUFFICIENT_DATA
- **Period**: Evaluation window (60s, 300s, etc.)
- **Evaluation Periods**: Consecutive breaches needed
- **Targets**: EC2, ASG, SNS, Lambda, SSM
- **Composite Alarms**: Combine multiple with AND/OR

### CloudWatch Logs
- **Structure**: Log Group → Stream → Event
- **Retention**: 1 day to 10 years (or never)
- **Logs Insights**: SQL-like query language
- **Subscriptions**: Real-time delivery to Lambda/Kinesis
- **Metric Filters**: Convert logs to metrics

### CloudWatch Extensions
- **Container Insights**: ECS/EKS/Kubernetes metrics
- **Lambda Insights**: Cold start, memory, duration
- **Application Insights**: Auto dashboards with ML
- **Contributor Insights**: Top-N analysis

### EventBridge
- **Event Buses**: Default, Partner, Custom
- **Rules**: Schedule (Cron) vs Pattern
- **Targets**: Lambda, SNS, SQS, Step Functions, CodePipeline
- **Replay**: Archive and replay events
- **DLQ**: Handle failed deliveries

### CloudTrail
- **Purpose**: Audit all API calls
- **Event Types**: Management, Data, Insights
- **Storage**: S3 + CloudWatch Logs
- **Retention**: 90 days console, longer S3
- **Multi-region**: Single trail covers all regions

### AWS Config
- **Purpose**: Track resource compliance
- **Rules**: Managed (AWS) vs Custom (Lambda)
- **Remediation**: Auto-fix via SSM Automation
- **Aggregator**: Multi-account, multi-region
- **Pricing**: Per config item + rule evaluation

### The Critical Distinction
- **CloudWatch**: "What's happening NOW?" (metrics, logs, alarms)
- **CloudTrail**: "Who did WHAT?" (API audit)
- **Config**: "How did THINGS CHANGE?" (compliance history)

---

## AWS CLI Quick Reference

### CloudWatch Metrics
```bash
# Put custom metric
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "OrdersProcessed" \
  --value 150 \
  --unit Count \
  --region us-east-1

# Get metric statistics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time 2025-01-01T00:00:00Z \
  --end-time 2025-01-02T00:00:00Z \
  --period 300 \
  --statistics Average,Maximum \
  --region us-east-1

# List metrics
aws cloudwatch list-metrics \
  --namespace AWS/EC2 \
  --region us-east-1

# Describe metric alarms
aws cloudwatch describe-alarms \
  --alarm-names "High-CPU-Alarm" \
  --region us-east-1
```

### CloudWatch Alarms
```bash
# Create metric alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "High-CPU-Alert" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:AlertTopic" \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --region us-east-1

# Create composite alarm
aws cloudwatch put-composite-alarm \
  --alarm-name "System-Critical" \
  --alarm-rule "ALARM(db-alarm) AND ALARM(app-alarm)" \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:AlertTopic" \
  --region us-east-1

# Set alarm state (for testing)
aws cloudwatch set-alarm-state \
  --alarm-name "High-CPU-Alert" \
  --state-value ALARM \
  --state-reason "Testing" \
  --region us-east-1

# Delete alarm
aws cloudwatch delete-alarms \
  --alarm-names "High-CPU-Alert" \
  --region us-east-1

# Disable alarm actions
aws cloudwatch disable-alarm-actions \
  --alarm-names "High-CPU-Alert" \
  --region us-east-1

# Enable alarm actions
aws cloudwatch enable-alarm-actions \
  --alarm-names "High-CPU-Alert" \
  --region us-east-1
```

### CloudWatch Logs
```bash
# Create log group
aws logs create-log-group \
  --log-group-name "/my-app/production" \
  --region us-east-1

# Put log events
aws logs put-log-events \
  --log-group-name "/my-app/production" \
  --log-stream-name "instance-1" \
  --log-events timestamp=$(date +%s000),message="App started" \
  --region us-east-1

# Get log events
aws logs get-log-events \
  --log-group-name "/my-app/production" \
  --log-stream-name "instance-1" \
  --region us-east-1

# Describe log groups
aws logs describe-log-groups \
  --region us-east-1

# Describe log streams
aws logs describe-log-streams \
  --log-group-name "/my-app/production" \
  --region us-east-1

# Filter log events
aws logs filter-log-events \
  --log-group-name "/my-app/production" \
  --filter-pattern "ERROR" \
  --region us-east-1

# Put retention policy
aws logs put-retention-policy \
  --log-group-name "/my-app/production" \
  --retention-in-days 7 \
  --region us-east-1

# Put metric filter
aws logs put-metric-filter \
  --log-group-name "/my-app/production" \
  --filter-name "ErrorCount" \
  --filter-pattern "[... ERROR ...]" \
  --metric-transformations metricName=ErrorCount,metricNamespace=MyApp,metricValue=1 \
  --region us-east-1

# Put subscription filter
aws logs put-subscription-filter \
  --log-group-name "/my-app/production" \
  --filter-name "SendToLambda" \
  --filter-pattern "" \
  --destination-arn "arn:aws:lambda:us-east-1:123456789012:function:ProcessLogs" \
  --region us-east-1

# Start query
aws logs start-query \
  --log-group-name "/my-app/production" \
  --start-time 1609459200 \
  --end-time 1609545600 \
  --query-string 'fields @timestamp, @message | stats count() by bin(5m)' \
  --region us-east-1

# Get query results
aws logs get-query-results \
  --query-id "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
  --region us-east-1

# Delete log group
aws logs delete-log-group \
  --log-group-name "/my-app/production" \
  --region us-east-1
```

### EventBridge (CloudWatch Events)
```bash
# Create event bus
aws events create-event-bus \
  --name "my-custom-events" \
  --region us-east-1

# Create schedule rule
aws events put-rule \
  --name "DailyBackup" \
  --schedule-expression "cron(0 2 * * ? *)" \
  --state ENABLED \
  --region us-east-1

# Create event pattern rule
aws events put-rule \
  --name "EC2StateChange" \
  --event-pattern '{"source":["aws.ec2"],"detail-type":["EC2 Instance State-change Notification"]}' \
  --state ENABLED \
  --region us-east-1

# Add target to rule
aws events put-targets \
  --rule "EC2StateChange" \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:ProcessEvent" \
  --region us-east-1

# Describe rule
aws events describe-rule \
  --name "EC2StateChange" \
  --region us-east-1

# List rules
aws events list-rules \
  --region us-east-1

# Put events
aws events put-events \
  --entries '[{
    "Source": "my-app",
    "DetailType": "CustomEvent",
    "Detail": "{\"key\": \"value\"}"
  }]' \
  --region us-east-1

# List targets
aws events list-targets-by-rule \
  --rule "EC2StateChange" \
  --region us-east-1

# Delete rule (must remove targets first)
aws events remove-targets \
  --rule "EC2StateChange" \
  --ids "1" \
  --region us-east-1

aws events delete-rule \
  --name "EC2StateChange" \
  --region us-east-1
```

### CloudTrail
```bash
# Create trail
aws cloudtrail create-trail \
  --name "MyTrail" \
  --s3-bucket-name "my-cloudtrail-bucket" \
  --is-multi-region-trail \
  --region us-east-1

# Start logging
aws cloudtrail start-logging \
  --trail-name "MyTrail" \
  --region us-east-1

# Get trail status
aws cloudtrail get-trail-status \
  --trail-name "MyTrail" \
  --region us-east-1

# Describe trails
aws cloudtrail describe-trails \
  --region us-east-1

# Lookup events
aws cloudtrail lookup-events \
  --lookup-attributes "AttributeKey=EventName,AttributeValue=RunInstances" \
  --start-time 2025-01-01T00:00:00Z \
  --end-time 2025-01-31T23:59:59Z \
  --region us-east-1

# Put event selectors (management + data events)
aws cloudtrail put-event-selectors \
  --trail-name "MyTrail" \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [{
      "Type": "AWS::S3::Object",
      "Values": ["arn:aws:s3:::my-bucket/*"]
    }]
  }]' \
  --region us-east-1

# Stop logging
aws cloudtrail stop-logging \
  --trail-name "MyTrail" \
  --region us-east-1

# Delete trail
aws cloudtrail delete-trail \
  --trail-name "MyTrail" \
  --region us-east-1
```

### AWS Config
```bash
# Start configuration recorder
aws configservice start-configuration-recorder \
  --configuration-recorder-name "default" \
  --region us-east-1

# Put delivery channel
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "my-config-bucket",
    "configSnapshotDeliveryProperties": {
      "deliveryFrequency": "TwentyFour_Hours"
    }
  }' \
  --region us-east-1

# Put managed config rule
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "ec2-ebs-encryption-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "ENCRYPTED_VOLUMES"
    }
  }' \
  --region us-east-1

# Describe config rules
aws configservice describe-config-rules \
  --region us-east-1

# Get compliance details
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name "ec2-ebs-encryption-enabled" \
  --region us-east-1

# Describe compliance by config rule
aws configservice describe-compliance-by-config-rule \
  --region us-east-1

# Put remediation configs
aws configservice put-remediation-configs \
  --remediation-configs '[{
    "ConfigRuleName": "required-tags",
    "TargetType": "SSM_DOCUMENT",
    "TargetIdentifier": "AWS-AddTagsToResource",
    "Automatic": true
  }]' \
  --region us-east-1

# Get resource config history
aws configservice get-resource-config-history \
  --resource-type "AWS::EC2::Instance" \
  --resource-id "i-1234567890abcdef0" \
  --region us-east-1

# Create config aggregator
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name "CentralAggregator" \
  --account-aggregation-sources '[{
    "AccountIds": ["111111111111"],
    "AwsRegions": ["us-east-1", "us-west-2"]
  }]' \
  --region us-east-1
```

---

**Last Updated**: March 2025  
**Exam Focus**: SAA-C03 (Associate-level monitoring and compliance knowledge)  
**Key Learning**: Understand WHEN to use CloudWatch vs CloudTrail vs Config—this distinction appears in 10-15% of SAA exam questions.
