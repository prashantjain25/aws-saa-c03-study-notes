# AWS SAA-C03: Serverless Computing — Lambda & Edge Functions

*Study notes in expert tutor style — understanding the WHY behind the WHAT*

---

## Table of Contents
1. [AWS Lambda Overview](#aws-lambda-overview)
2. [Lambda Pricing & Free Tier](#lambda-pricing--free-tier)
3. [Lambda Execution Limits (EXAM CRITICAL)](#lambda-execution-limits-exam-critical)
4. [Lambda Concurrency & Throttling](#lambda-concurrency--throttling)
5. [Cold Starts & Provisioned Concurrency](#cold-starts--provisioned-concurrency)
6. [Lambda SnapStart](#lambda-snapstart)
7. [Lambda Invocation Patterns](#lambda-invocation-patterns)
8. [Lambda Outside VPC (Default)](#lambda-outside-vpc-default)
9. [Lambda Inside VPC](#lambda-inside-vpc)
10. [Lambda with RDS Proxy](#lambda-with-rds-proxy)
11. [Invoking Lambda from RDS/Aurora](#invoking-lambda-from-rdsaurora)
12. [Edge Computing Fundamentals](#edge-computing-fundamentals)
13. [CloudFront Functions vs Lambda@Edge](#cloudfront-functions-vs-lambdaedge-exam-comparison)
14. [Interview Tips & Exam Patterns](#interview-tips--exam-patterns)
15. [Quick Reference — AWS CLI Commands](#quick-reference--aws-cli-commands)

---

## AWS Lambda Overview

### The Core Concept: Serverless Compute

Imagine you run a pizza restaurant. Normally, you'd hire a full-time staff member to wait for phone orders — they sit there all day, whether customers call or not. **That's a traditional server model.**

With Lambda, you only pay when orders come in. Someone takes the order, you process it, and they're gone. You don't pay for idle time. **That's serverless.**

**AWS Lambda** is a fully managed compute service that:
- **Runs your code on-demand** without provisioning or managing servers
- **Scales automatically** — from zero to thousands of concurrent invocations
- **Charges only for what you use** — requests and compute time

### Supported Runtimes

Lambda natively supports:
- **Node.js** (18.x, 20.x, 22.x)
- **Python** (3.11, 3.12, 3.13)
- **Java** (17, 21, 23)
- **C#** (.NET 8, .NET Framework 4.8)
- **Ruby** (3.3, 3.4)
- **Go** (1.x)
- **Custom Runtime** — use any language via Lambda Layers

You can also provide a **Docker image** up to 10GB for maximum flexibility.

---

## Lambda Pricing & Free Tier

### How You Pay

Lambda pricing has two components:

**1. Per Invocation (Request-based)**
- First **1,000,000 requests per month**: FREE
- After that: **$0.20 per million requests** = $0.0000002 per request

**2. Per Duration (Compute-based)**
- You're charged in **1-millisecond increments**
- Base allocation: **400,000 GB-seconds per month** (FREE)

**What's a GB-second?**
- 1 GB of RAM running for 1 second = 1 GB-second
- If your Lambda uses 256MB RAM and runs for 100ms, that's 0.025 GB-seconds
- 128MB for 1 second = 0.125 GB-seconds
- 512MB for 1 second = 0.5 GB-seconds

**Free tier breakdown (400,000 GB-seconds/month):**
- At **1 GB RAM**: 400,000 seconds
- At **128 MB RAM**: 3,200,000 seconds
- At **512 MB RAM**: 800,000 seconds

**After free tier:**
- **$1.00 per 600,000 GB-seconds** = $0.0000016667 per GB-second

### Why This Matters for the Exam

Very cheap + automatic scaling = **Lambda is everywhere in modern AWS architectures**. It's the default choice for event-driven processing: S3 uploads, API requests, DynamoDB streams, SNS notifications, scheduled tasks (EventBridge/CloudWatch Events).

---

## Lambda Execution Limits (EXAM CRITICAL)

These limits are **per region** and absolutely crucial for SAA-C03:

### Execution Limits

| Limit | Value | Notes |
|-------|-------|-------|
| **Memory** | 128 MB – 10 GB | 1 MB increments; more memory = more CPU |
| **Maximum execution time** | **900 seconds (15 minutes)** | EXAM FACT: Anything longer → use EC2, Fargate, or Step Functions |
| **Environment variables** | 4 KB | Includes names and values |
| **Temporary disk (/tmp)** | 512 MB – 10 GB | Can store temp files, layer dependencies, pre-loaded data |
| **Layers** | 5 maximum | Pre-packaged dependencies |

### Deployment Limits

| Limit | Value | Notes |
|-------|-------|-------|
| **Compressed .zip** | 50 MB | Direct upload limit |
| **Uncompressed code + deps** | 250 MB | Total size after decompression |
| **Request body** (synchronous) | 6 MB | Event payload size limit |
| **Response body** | 6 MB | Return value size limit |

### Why Memory Matters

Lambda allocates CPU proportionally to RAM. If you set 128 MB → you get ~0.08 vCPUs. If you set 1 GB → you get ~0.6 vCPUs. **So more memory = faster execution = better throughput.**

> **Interview Tip:** "Should I use 128 MB or 1024 MB?" → Depends on your code. A simple webhook handler might be fine at 256 MB. A data transformation with heavy JSON parsing benefits from 1 GB. Use CloudWatch Metrics → Max Memory Used to right-size.

---

## Lambda Concurrency & Throttling

### The Default Limit: 1000 Concurrent Executions per Region

**Concurrency** = the number of Lambda instances simultaneously executing code.

AWS sets a **default account-level limit of 1000 concurrent executions** per region. This is a **hard soft-limit** — you can request an increase via AWS Support.

### What Happens at Capacity?

When you hit 1000 concurrent executions and a new request arrives:

**Synchronous Invocation (API Gateway, ALB, direct SDK call):**
- Returns **HTTP 429 (Too Many Requests)** immediately
- Caller sees: `ThrottleError`

**Asynchronous Invocation (S3, SNS, SQS, DynamoDB Streams):**
- Request goes back into the queue
- Lambda retries with **exponential backoff** (1s → 5min → 6 hours max)
- If all retries fail → sent to Dead Letter Queue (DLQ)

### Reserved Concurrency — The Cap

Set a **maximum** concurrency per function:

```
aws lambda put-function-concurrency \
  --function-name my-function \
  --reserved-concurrent-executions 100
```

**Effect:**
- That function can **never exceed 100 concurrent** invocations
- Protects other functions from being starved
- If the function hits 100 concurrent, **new requests are throttled** (429)

### The Starving Problem (Real-World Scenario)

Imagine:
- 1000 total concurrency limit
- Function A (ALB-backed): processes slowly, holds instances for 30s each
- Function B (API Gateway): lightweight, needs quick responses
- Function C (SDK/CLI): batch jobs

If Function A gets hammered, it could consume **all 1000 concurrent slots**, leaving B and C with nothing. They get throttled.

**Solution:** Set `reserved-concurrent-executions = 200` on Function A. Now it can max out at 200, leaving 800 for others.

---

## Cold Starts & Provisioned Concurrency

### What Is a Cold Start?

When Lambda invokes a new instance:

1. **Infrastructure initialization** — AWS allocates a container, network setup (network time protocol)
2. **Code download and extraction** — .zip is unpacked
3. **Runtime initialization** — language runtime starts (JVM for Java, Python interpreter)
4. **Init code runs** — code outside the handler function executes (e.g., database connections, SDK initialization)
5. **Handler invoked** — your business logic runs

**This entire process takes 20–500 ms depending on:**

- Runtime (JVM slower than Node.js)
- Dependencies (heavy libraries add overhead)
- Code complexity

**Subsequent invocations** on the same instance are warm: ~1 ms latency.

> **Real-world analogy:** Cold start is like firing up a factory at 6 AM (initialization overhead). Once it's running, each widget (request) processes fast. If you shut down the factory (no traffic for 5–15 min), the next request triggers another cold start.

### Provisioned Concurrency — Pre-Warming

**Provisioned Concurrency** pre-allocates and initializes instances **before** invocations arrive:

```
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier prod \
  --provisioned-concurrent-executions 10
```

**Effect:**
- AWS initializes 10 instances immediately
- All invocations consume these pre-warmed instances
- **Zero cold start** — consistent, predictable latency
- Charges apply even if instances aren't invoked

**Cost:** Provisioned concurrency is **more expensive** than on-demand. Use for:
- **Critical endpoints** (user-facing APIs)
- **Predictable traffic** (set a static number)
- **Cost-sensitive** environments where cold starts hurt UX

### Reserved vs Provisioned Concurrency (EXAM TABLE)

| Aspect | Reserved Concurrency | Provisioned Concurrency |
|--------|----------------------|-------------------------|
| **Purpose** | Limit (cap) | Warmth (floor) |
| **Cold starts** | Not prevented | Eliminated |
| **Cost** | Free | Paid (~0.015 USD/hour per execution) |
| **Use case** | Protect other functions from starvation | Critical user-facing APIs |
| **AWS CLI** | `put-function-concurrency` | `put-provisioned-concurrency-config` |

---

## Lambda SnapStart

### The Problem SnapStart Solves

Java and .NET applications have **heavy initialization overhead** because:
- JVM must start and compile bytecode (JIT compilation)
- Class loaders must load all dependencies
- Frameworks (Spring, Entity Framework) perform setup
- Database connection pools initialize

Result: **first invocation = 1–5 seconds latency**. Unacceptable for user-facing APIs.

### The Solution: Snapshot and Replay

**AWS Lambda SnapStart** (launched 2023) snapshots your Lambda's memory and disk state **once**, then replays it for every invocation:

**Without SnapStart:**
```
[Cold: Init 1s → Invoke 50ms → Shutdown] → [Warm: Invoke 50ms] → [Warm: Invoke 50ms]
```

**With SnapStart:**
```
[Cold: Init 1s → Snapshot] → [Invoke 50ms (from snapshot)] → [Invoke 50ms (from snapshot)]
```

### How SnapStart Works

When you **publish a new function version** or **update the code**:

1. Lambda initializes the function (runs init code)
2. Captures the memory and disk state as a **snapshot**
3. Caches the snapshot for low-latency access

Every invocation loads the snapshot → skips the Init phase → runs handler → Shutdown.

**Result:** Up to **10x faster cold starts** for Java/Python/.NET — **no extra cost**.

> **Exam note:** SnapStart is available for Java, Python, and .NET runtimes. Not for Node.js or Go yet.

---

## Lambda Invocation Patterns

### Architecture: Typical Serverless Application

<p align="center"><img src="diagrams/icons/named/user-bw.png" width="52" title="User / Client"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/api-gateway.jpg" width="52" title="API Gateway"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/lambda.jpg" width="52" title="Lambda Function"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/dynamodb.jpg" width="52" title="DynamoDB"/>&nbsp;/&nbsp;<img src="diagrams/icons/named/s3.jpg" width="52" title="S3 Bucket"/></p>
<p align="center"><sub>User → API Gateway (sync) → Lambda → DynamoDB (reads/writes) / S3 (files)</sub></p>

### Synchronous Invocations
- **Caller waits for response**
- **Blocks until function completes**
- Sources: API Gateway, ALB, SDK direct invoke, CloudFormation custom resources
- If throttled → **immediate 429 error**

### Asynchronous Invocations
- **Caller does NOT wait**
- **Function queued, invoked in background**
- Sources: S3, SNS, SQS, DynamoDB Streams, EventBridge, CloudWatch Events
- If throttled → **queued and retried** (exponential backoff, up to 6 hours)
- If all retries fail → sent to **Dead Letter Queue (DLQ)**

---

## Lambda Outside VPC (Default)

### The Default: Lambda in AWS-Managed VPC

By default, **Lambda runs OUTSIDE your VPC** in an **AWS-managed, isolated environment**.

**You CAN access:**
- S3 buckets
- DynamoDB tables
- Public internet (APIs, webhooks)
- SNS, SQS, EventBridge
- CloudWatch Logs
- Secrets Manager, Parameter Store

**You CANNOT access:**
- RDS databases (private endpoint)
- ElastiCache clusters (private)
- Internal Application Load Balancer (private)
- Any resource in your VPC's private subnets

> **Why?** Because Lambda isn't connected to your VPC. No ENI, no security group. It's in a separate network.

---

## Lambda Inside VPC

### When You MUST Use Lambda in VPC

When your Lambda needs to access **private RDS, ElastiCache, or internal services**, you **must configure VPC**.

### How VPC Works with Lambda

```
aws lambda update-function-configuration \
  --function-name my-function \
  --vpc-config SubnetIds=subnet-abc123,subnet-def456,SecurityGroupIds=sg-xyz789
```

**What happens:**
1. Lambda creates an **Elastic Network Interface (ENI)** in your specified subnet
2. The ENI gets a private IP address
3. Lambda instance uses this ENI to communicate with private resources

### Security Group Rules

**Lambda's security group** must have:
- **Outbound:** Allow HTTPS (443) and your DB port (3306 for MySQL, 5432 for PostgreSQL)
- **Target resource's security group:** Allow inbound from Lambda's security group

**Example:**
```
Lambda SG: Outbound → 5432 to RDS SG ✓
RDS SG:   Inbound  → port 5432 from Lambda SG ✓
```

### The Cold Start Penalty in VPC

**BEFORE Nov 2019:** Lambda in VPC had **massive cold start delays** (30–45 seconds) because ENI attachment was slow.

**AFTER Nov 2019:** AWS massively optimized ENI attachment. Cold starts in VPC are now nearly identical to outside VPC.

> **Exam detail:** If a question mentions "Lambda in VPC has poor cold start performance," check the date. If it's recent, the answer is likely "No longer true after the 2019 optimization."

---

## Lambda with RDS Proxy

### The Problem: Connection Pool Exhaustion

Imagine:
- You have 100 concurrent Lambda functions
- Each opens a direct connection to RDS
- RDS max connections = 100
- **RDS is at max capacity**
- 101st Lambda → connection refused

Each cold start creates a new connection. Each warm instance holds it. Result: **connection pool exhaustion**.

### The Solution: RDS Proxy

**RDS Proxy** is a managed database connection pool between Lambda and RDS:

```
Lambda 1 ─┐
Lambda 2 ─┤
Lambda 3 ─┼─→ RDS Proxy (pooling) → RDS (1 real connection)
...       │
Lambda N ─┘
```

**Benefits:**
- **Connection pooling:** 1000 Lambda instances → 20 actual DB connections (configurable)
- **Improved availability:** Failover time reduced by 66%
- **Security:** Enforces IAM database authentication, stores credentials in Secrets Manager
- **Connection preservation:** Connections persist during RDS failover

### Architecture: Lambda + RDS Proxy

<p align="center"><img src="diagrams/icons/named/eventbridge.png" width="52" title="Event Trigger"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/lambda.jpg" width="52" title="Lambda fn1/fn2/fn3 (in VPC)"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/rds.png" width="52" title="RDS Proxy (connection pooling)"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/rds-green.jpg" width="52" title="RDS Instance"/></p>
<p align="center"><sub>Events → Lambda functions (concurrent) → RDS Proxy (pools connections) → RDS</sub></p>
<p align='center'><sub>💡 Without RDS Proxy: each Lambda invocation = new DB connection → exhaustion. Proxy multiplexes them.</sub></p>

### CRITICAL EXAM DETAIL

**RDS Proxy is NEVER publicly accessible.** It's deployed in a private subnet of your VPC.

**Therefore:** Your Lambda **MUST also be in the VPC** to communicate with RDS Proxy.

If Lambda is outside VPC → cannot reach RDS Proxy → cannot use database.

---

## Invoking Lambda from RDS/Aurora

### Reverse Trigger: Lambda Invoked by Database

Instead of Lambda calling the database, your **database calls Lambda**. Useful for:
- User registration (INSERT) → invoke Lambda to send welcome email
- Data changes → trigger business logic without application code

### Supported Engines

- **RDS PostgreSQL** (10+)
- **Aurora MySQL** (5.7+, 8.0+)

**NOT supported:** RDS MySQL, RDS MariaDB, RDS Oracle, RDS SQL Server

### Requirements

1. **Database connectivity:** RDS must reach Lambda
   - Public Lambda (outside VPC) → via internet gateway
   - Private Lambda (in VPC) → same VPC + security groups

2. **Database permissions:**
   - Lambda **Resource-based Policy** allows RDS principal to invoke
   - IAM policy on the RDS service role allows `lambda:InvokeFunction`

### Example Use Case

```sql
-- PostgreSQL stored procedure
CREATE FUNCTION send_welcome_email()
RETURNS void AS $$
BEGIN
  SELECT aws_lambda.invoke(
    'arn:aws:lambda:us-east-1:123456789012:function:SendEmail',
    json_build_object('user_id', NEW.user_id, 'email', NEW.email)
  );
END;
$$ LANGUAGE plpgsql;

-- Trigger on user_created table
CREATE TRIGGER on_user_created
  AFTER INSERT ON users
  FOR EACH ROW EXECUTE FUNCTION send_welcome_email();
```

Insert user → RDS invokes Lambda → Lambda sends email via SES.

---

## Edge Computing Fundamentals

### Why Edge Matters

Modern web applications must serve users **globally** with minimal latency. A request from Tokyo to us-east-1 AWS region = 100+ ms round-trip.

**Solution:** Execute code **at the edge** — in data centers geographically close to users.

**AWS CloudFront** has 500+ edge locations worldwide. You can execute code **right at the edge location** before sending requests to your origin.

### Two Types of Edge Functions

AWS offers **two serverless options** for edge computing:

1. **CloudFront Functions** — lightweight, fast, limited scope
2. **Lambda@Edge** — powerful, flexible, can access origin and execute longer logic

---

## CloudFront Functions vs Lambda@Edge (EXAM COMPARISON)

This is **guaranteed to appear** on the SAA-C03 exam. Memorize this table:

### Comparison Table

| Feature | CloudFront Functions | Lambda@Edge |
|---------|----------------------|-------------|
| **Language** | JavaScript only | Node.js or Python |
| **Scale** | Millions of requests/second | Thousands of requests/second |
| **Triggers** | 2: Viewer Request, Viewer Response | 4: Viewer Request/Response, Origin Request/Response |
| **Max execution time** | < 1 ms | 5–10 seconds |
| **Max memory** | 2 MB | 128 MB – 10 GB |
| **Package size** | 10 KB | 1–50 MB |
| **Network access** | NO | YES |
| **Filesystem access** | NO | YES |
| **Request body access** | NO | YES |
| **Pricing** | Free tier + 1/6th cost of Lambda@Edge | No free tier; per-request + duration |
| **Managed where** | Native CloudFront (built-in) | AWS Lambda (us-east-1 only) |

### CloudFront Functions — Use When

- Cache key normalization (remove query params, sort headers)
- Header manipulation (add/remove/modify headers)
- URL rewriting or redirects
- Request authentication (JWT token validation)
- **Sub-1ms execution needed**

**Example:** Normalize URLs for cache efficiency
```javascript
function handler(event) {
    var request = event.request;
    
    // Remove query string for cache efficiency
    request.querystring = {};
    
    // Standardize headers
    request.headers['x-request-id'] = { value: generateId() };
    
    return request;
}
```

### Lambda@Edge — Use When

- Longer processing (>1ms, <10s)
- Access to third-party APIs or AWS services
- Complex authentication (OAuth, 2FA)
- Request body inspection/modification
- Image transformation, resizing
- Database lookups
- Accessing filesystem (temp files, pre-loaded data)

**Example:** Add AWS X-Ray tracing, call DynamoDB
```python
import json
import boto3
import aws_xray_sdk

dynamodb = aws_xray_sdk.client('dynamodb')

def lambda_handler(event, context):
    request = event['Records'][0]['cf']['request']
    
    # Lookup user from DynamoDB
    user_id = request['headers'].get('x-user-id', [{}])[0].get('value')
    
    response = dynamodb.get_item(
        TableName='Users',
        Key={'user_id': {'S': user_id}}
    )
    
    # Add user data to request
    request['headers']['x-user-tier'] = [{'value': response['Item']['tier']['S']}]
    
    return request
```

### Trigger Points Visualization

<p align="center"><img src="diagrams/icons/named/user-bw.png" width="52" title="Client Request"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/cloudfront.png" width="52" title="CloudFront Edge"/>&nbsp;&nbsp;<b>→</b>&nbsp;&nbsp;<img src="diagrams/icons/named/s3-bw.png" width="52" title="Origin (S3 / ALB)"/></p>
<p align="center"><sub>Client → CloudFront (4 Lambda@Edge triggers: Viewer Req, Origin Req, Origin Res, Viewer Res) → Origin</sub></p>

**CloudFront Functions triggers:**
1. Viewer Request (before CloudFront cache lookup)
2. Viewer Response (before returning to viewer)

**Lambda@Edge triggers (4 total):**
1. Viewer Request (before cache lookup)
2. Origin Request (if cache miss, before forwarding to origin)
3. Origin Response (from origin, before caching)
4. Viewer Response (before returning to viewer)

---

## Interview Tips & Exam Patterns

### Question 1: "Lambda timeout is 15 minutes. My job takes 2 hours. What do I do?"

**Answer:** Don't use Lambda. Options:
- **AWS Batch** for long-running jobs
- **Amazon EC2** for sustained workloads
- **AWS Fargate** for containerized long-running tasks
- **Step Functions** to orchestrate multiple Lambda invocations (each <15 min)
- **SQS + polling** for asynchronous job processing

---

### Question 2: "Lambda can't connect to RDS. Why?"

**Checklist:**
1. **Is Lambda in a VPC?** Default = outside VPC. Must explicitly add to VPC.
2. **Are the subnets correct?** Should be in same VPC as RDS (or have route to RDS).
3. **Are security groups configured?**
   - Lambda SG: outbound to RDS port (3306 MySQL, 5432 Postgres)
   - RDS SG: inbound from Lambda SG
4. **Is RDS publicly accessible?** If Lambda is in VPC, RDS doesn't need public access.
5. **Using RDS Proxy?** Lambda MUST be in VPC to reach Proxy.

---

### Question 3: "How do I prevent Lambda function A from throttling function B?"

**Answer:** Set **reserved concurrency** on function A:

```
aws lambda put-function-concurrency \
  --function-name functionA \
  --reserved-concurrent-executions 200
```

Now A maxes at 200 concurrent, leaving 800 for others.

---

### Question 4: "My API has unpredictable traffic spikes. How do I eliminate cold starts?"

**Answer:** Use **Provisioned Concurrency** with **Application Auto Scaling**:

```
aws lambda put-provisioned-concurrency-config \
  --function-name my-api \
  --qualifier prod \
  --provisioned-concurrent-executions 10
```

Or scale provisioned concurrency based on CloudWatch metrics (CPU, memory):

```
aws autoscaling register-scalable-target \
  --service-namespace lambda \
  --resource-id function:my-api:provisioned \
  --scalable-dimension lambda:function:ProvisionedConcurrentExecutions \
  --min-capacity 5 \
  --max-capacity 100
```

---

### Question 5: "CloudFront Functions or Lambda@Edge?"

**Decision tree:**
- Need <1ms execution? → **CloudFront Functions**
- Only JS? Not accessing origin? → **CloudFront Functions**
- Need to access origin, use Python/Node? → **Lambda@Edge**
- Need database access? → **Lambda@Edge**
- Need filesystem? → **Lambda@Edge**
- Price-conscious? → **CloudFront Functions** (10x cheaper)

---

### Question 6: "What's the difference between reserved and provisioned concurrency?"

**Reserved Concurrency = Protection (Cap)**
- Sets a **maximum** for that function
- Prevents throttling of other functions
- Free
- Does NOT eliminate cold starts

**Provisioned Concurrency = Performance (Floor)**
- Pre-warms instances **before** requests arrive
- Eliminates cold starts
- Paid (charges even when idle)
- Improves latency consistency

Use reserved for protecting background jobs. Use provisioned for user-facing APIs.

---

### Question 7: "Lambda@Edge not available in my region. What do I do?"

**Answer:** Lambda@Edge functions must be created/published in **us-east-1** only. Then CloudFront replicates them globally to all edge locations.

---

## Quick Reference — AWS CLI Commands

### Basic Lambda Operations

```bash
# Create a Lambda function
aws lambda create-function \
  --function-name my-function \
  --runtime python3.12 \
  --role arn:aws:iam::123456789012:role/lambda-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 256 \
  --environment Variables={DB_HOST=mydb.us-east-1.rds.amazonaws.com,ENV=prod}

# Update function code
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip

# Update function configuration (memory, timeout, environment)
aws lambda update-function-configuration \
  --function-name my-function \
  --timeout 900 \
  --memory-size 1024 \
  --environment Variables={TIMEOUT=900}

# Get function details
aws lambda get-function-configuration \
  --function-name my-function

# List all functions
aws lambda list-functions --max-items 50
```

### Concurrency Management

```bash
# Set reserved concurrency (cap function to 100 concurrent invocations)
aws lambda put-function-concurrency \
  --function-name my-function \
  --reserved-concurrent-executions 100

# Remove reserved concurrency
aws lambda delete-function-concurrency \
  --function-name my-function

# Set provisioned concurrency (pre-warm 10 instances)
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier prod \
  --provisioned-concurrent-executions 10

# Get concurrency config
aws lambda get-function-concurrency \
  --function-name my-function
```

### VPC Configuration

```bash
# Add Lambda to VPC
aws lambda update-function-configuration \
  --function-name my-function \
  --vpc-config SubnetIds=subnet-abc123,subnet-def456 SecurityGroupIds=sg-xyz789

# Remove Lambda from VPC
aws lambda update-function-configuration \
  --function-name my-function \
  --vpc-config SubnetIds= SecurityGroupIds=
```

### Layers (Dependencies)

```bash
# Create a layer (for libraries, custom runtimes)
aws lambda publish-layer-version \
  --layer-name my-dependencies \
  --zip-file fileb://layer.zip \
  --compatible-runtimes python3.12 python3.11

# Add layer to function
aws lambda update-function-configuration \
  --function-name my-function \
  --layers arn:aws:lambda:us-east-1:123456789012:layer:my-dependencies:1
```

### Dead Letter Queue (DLQ)

```bash
# Set SQS DLQ for failed asynchronous invocations
aws lambda update-function-configuration \
  --function-name my-function \
  --dead-letter-config TargetArn=arn:aws:sqs:us-east-1:123456789012:my-dlq

# Set SNS DLQ alternative
aws lambda update-function-configuration \
  --function-name my-function \
  --dead-letter-config TargetArn=arn:aws:sns:us-east-1:123456789012:my-topic
```

### Invoking Lambda

```bash
# Synchronous invocation (blocking, waits for response)
aws lambda invoke \
  --function-name my-function \
  --payload '{"key": "value"}' \
  --cli-binary-format raw-in-base64-out \
  output.json

# Asynchronous invocation (fire-and-forget)
aws lambda invoke \
  --function-name my-function \
  --invocation-type Event \
  --payload '{"key": "value"}' \
  --cli-binary-format raw-in-base64-out \
  output.json

# View response
cat output.json
```

### Aliases & Versioning

```bash
# Publish new version (snapshot of current code)
aws lambda publish-version \
  --function-name my-function \
  --description "Version with bug fix"

# Create alias pointing to version
aws lambda create-alias \
  --function-name my-function \
  --name prod \
  --function-version 3

# Update alias to point to new version
aws lambda update-alias \
  --function-name my-function \
  --name prod \
  --function-version 4

# Invoke specific alias
aws lambda invoke \
  --function-name my-function:prod \
  --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  output.json
```

### SnapStart (Java/Python/.NET)

```bash
# Enable SnapStart for published versions
aws lambda update-function-configuration \
  --function-name my-java-function \
  --snap-start ApplyOn=PublishedVersions

# Disable SnapStart
aws lambda update-function-configuration \
  --function-name my-java-function \
  --snap-start ApplyOn=None
```

### Lambda@Edge Deployment

```bash
# Publish Lambda function in us-east-1 (required for Lambda@Edge)
aws lambda publish-version \
  --function-name my-edge-function \
  --region us-east-1

# Once published, use the version ARN in CloudFront distribution config
# (via AWS CloudFront update-distribution)
aws cloudfront update-distribution \
  --id ABCDEFG1234567 \
  --distribution-config file://distribution-config.json
```

### Event Source Mapping (S3, DynamoDB, SQS)

```bash
# Create event source mapping (S3 → Lambda via SNS)
aws s3api put-bucket-notification-configuration \
  --bucket my-bucket \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [
      {
        "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:my-function",
        "Events": ["s3:ObjectCreated:*"],
        "Filter": {"Key": {"FilterRules": [{"Name": "prefix", "Value": "uploads/"}]}}
      }
    ]
  }'

# SQS event source mapping
aws lambda create-event-source-mapping \
  --event-source-arn arn:aws:sqs:us-east-1:123456789012:my-queue \
  --function-name my-function \
  --batch-size 10

# DynamoDB Streams event source mapping
aws lambda create-event-source-mapping \
  --event-source-arn arn:aws:dynamodb:us-east-1:123456789012:table/my-table/stream/2024-01-01T12:00:00.000 \
  --function-name my-function \
  --batch-size 100 \
  --starting-position LATEST
```

### CloudWatch Monitoring

```bash
# Get Lambda duration metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=my-function \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 300 \
  --statistics Average,Maximum

# Get Lambda error count
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=my-function \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 300 \
  --statistics Sum

# Create CloudWatch alarm for high error rate
aws cloudwatch put-metric-alarm \
  --alarm-name lambda-function-errors \
  --alarm-description "Alert when Lambda errors exceed 5" \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --statistic Sum \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=FunctionName,Value=my-function
```

---

## Points to Remember — Exam Cheat Sheet

### Lambda Execution
- **Max timeout:** 900 seconds (15 minutes) — anything longer = not Lambda
- **Default VPC state:** Outside your VPC (can reach S3, DynamoDB, public internet)
- **To access RDS:** Must be in your VPC + correct security groups
- **Memory range:** 128 MB – 10 GB (CPU scales with memory)

### Concurrency
- **Account default:** 1000 concurrent/region (soft limit, increase via support)
- **Reserved concurrency:** Caps function at N concurrent (prevents throttling others)
- **Provisioned concurrency:** Pre-warms N instances (eliminates cold starts, costs money)

### Cold Starts
- **Typical latency:** 20–500ms depending on runtime and dependencies
- **SnapStart:** Eliminates cold start for Java/Python/.NET (10x faster, no extra cost)
- **VPC cold starts:** No longer a penalty (optimized since Nov 2019)

### Edge Computing
- **CloudFront Functions:** JS only, <1ms, viewer triggers only, millions/sec, cheap
- **Lambda@Edge:** Node/Python, 5–10s, all 4 triggers, thousands/sec, expensive
- **Lambda@Edge location:** Must be created in us-east-1 (replicated globally)

### VPC & Networking
- **Lambda default:** Outside VPC
- **Lambda in VPC:** Creates ENI in your subnet, must have correct security groups
- **RDS Proxy:** Never public, requires Lambda in VPC, pools connections
- **RDS → Lambda:** Only PostgreSQL and Aurora MySQL, not RDS MySQL/Oracle

### Async Invocation & Retries
- **Throttle handling:** Auto-retry with exponential backoff (1s → 5min → 6 hours)
- **Failed after retries:** Sent to Dead Letter Queue (DLQ)
- **DLQ options:** SQS or SNS

### Pricing
- **Requests:** 1M free/month, then $0.20/million
- **Compute:** 400,000 GB-seconds free/month, then $1/600,000 GB-seconds
- **Provisioned concurrency:** Additional per-hour charge (~$0.015/hour per execution)

---

## Official AWS Documentation

- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/)
- [Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [Lambda Quotas](https://docs.aws.amazon.com/lambda/latest/dg/limits.html)
- [Lambda VPC Configuration](https://docs.aws.amazon.com/lambda/latest/dg/configuring-vpc.html)
- [CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html)
- [Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-the-edge.html)
- [RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)

---

*Last updated: March 6, 2026*
*Study efficiently. Master the concepts. Pass the SAA-C03.*
