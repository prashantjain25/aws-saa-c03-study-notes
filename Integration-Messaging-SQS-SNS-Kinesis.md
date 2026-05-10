# AWS SAA-C03: Integration & Messaging Services
## SQS, SNS, Kinesis Data Streams, Amazon Data Firehose, Amazon MQ

**Last Updated:** 2026-03-05  
**Level:** Solutions Architect Associate  
**Exam Focus:** CRITICAL - Messaging and Application Integration

---

## Official AWS Documentation References

- **SQS**: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html
- **SNS**: https://docs.aws.amazon.com/sns/latest/dg/welcome.html
- **Kinesis Data Streams**: https://docs.aws.amazon.com/streams/latest/dev/introduction.html
- **Amazon Data Firehose**: https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html
- **Amazon MQ**: https://docs.aws.amazon.com/amazon-mq/latest/developer-guide/welcome.html

---

## Table of Contents

1. [Why Messaging & Integration?](#why-messaging--integration)
2. [Amazon SQS — Simple Queue Service](#amazon-sqs--simple-queue-service)
3. [Amazon SNS — Simple Notification Service](#amazon-sns--simple-notification-service)
4. [SNS + SQS Fan-Out Pattern](#sns--sqs-fan-out-pattern)
5. [Amazon Kinesis](#amazon-kinesis)
6. [Amazon Data Firehose](#amazon-data-firehose)
7. [Service Comparison](#service-comparison)
8. [Amazon MQ](#amazon-mq)
9. [Points to Remember](#points-to-remember)
10. [Interview Tips](#interview-tips)

---

## Why Messaging & Integration?

### The Integration Problem: Tight Coupling vs Loose Coupling

Imagine you're building a restaurant ordering system with two applications:

**App A** = Ordering Service (receives customer orders)  
**App B** = Video Encoding Service (encodes menu videos)

#### Pattern 1: Synchronous (Direct Call) - THE PROBLEM

```
    ┌─────────────────────────────────────────┐
    │                                         │
    │  Ordering Service                       │
    │  ┌──────────────────────────────────┐  │
    │  │ HTTP Request                     │  │
    │  │ POST /encode-video               │  │
    │  └────────────────────┬─────────────┘  │
    │                       │                 │
    │                       ▼                 │
    │  ┌──────────────────────────────────┐  │
    │  │ Encoding Service                 │  │
    │  │ ⚠ BUSY (5 minute queue!)        │  │
    │  └──────────────────────────────────┘  │
    │                       │                 │
    │                       ▼                 │
    │  Ordering Service BLOCKS...            │
    │  Customer can't place order! ✗         │
    │                                         │
    └─────────────────────────────────────────┘
```

**What went wrong?**
- If Encoding Service is slow or down → Ordering Service blocks
- Tight coupling: both apps must be running and responsive
- Cascading failures: one slow service slows down everything
- Can't scale independently: adding ordering capacity doesn't help if encoder is the bottleneck

#### Pattern 2: Asynchronous (Event-Based) - THE SOLUTION

```
    ┌──────────────────────────────────┐
    │                                  │
    │  Ordering Service                │
    │  ┌────────────────────────────┐  │
    │  │ SendMessage to SQS         │  │
    │  │ (Returns immediately) ✓    │  │
    │  └────────────┬───────────────┘  │
    │               │                   │
    │               ▼                   │
    │  ┌──────────────────────────────┐│
    │  │   SQS Queue                  ││
    │  │ ┌────────────────────────┐   ││
    │  │ │ Message 1: Encode Foo  │   ││
    │  │ │ Message 2: Encode Bar  │   ││
    │  │ │ Message 3: Encode Baz  │   ││
    │  │ │ ...                    │   ││
    │  │ │ (buffered, persistent) │   ││
    │  │ └────────────────────────┘   ││
    │  └──────────────────────────────┘│
    │                                  │
    └──────────────────────────────────┘
              │
              │
    ┌─────────▼──────────────────────────┐
    │                                    │
    │  Encoding Service                  │
    │  ┌────────────────────────────┐    │
    │  │ Poll SQS for work          │    │
    │  │ Process at own pace ✓      │    │
    │  │ Scale independently ✓      │    │
    │  │ (even if slow, ordering    │    │
    │  │  service unaffected) ✓     │    │
    │  └────────────────────────────┘    │
    │                                    │
    └────────────────────────────────────┘
```

**What's better?**
- Ordering Service returns immediately (fast response)
- Encoding Service processes when ready
- Queue acts as a buffer for traffic spikes
- If Encoding Service is down, messages persist in queue
- Services scale independently
- **Loose coupling**: services don't know about each other

### Why AWS-Scale Requires Asynchronous Messaging

**Scenario:** Friday Black Friday at 9 AM.

1000s of customers place orders simultaneously  
→ 1000 video encoding requests flood in  
→ Without a queue, encoding service gets hammered  
→ Encoding service goes down  
→ Ordering service can't deliver video links  
→ $$$ Lost revenue

**With SQS Queue:**
1000 encoding requests → SQS buffers them  
→ Encoding service scales up (Auto Scaling Group: watch queue depth)  
→ Over 1 hour, all videos encoded  
→ No dropped requests, happier customers

### AWS Messaging Services at a Glance

| Service | Model | Persistence | When to Use |
|---------|-------|-------------|------------|
| **SQS** | Queue (pull) | 4-14 days | Decouple producers from consumers |
| **SNS** | Pub/Sub (push) | No persistence | Fan-out to multiple services |
| **Kinesis Data Streams** | Stream (pull) | 1-365 days | Real-time data analytics with replay |
| **Amazon Data Firehose** | Delivery | No storage | Load data into S3/Redshift/OpenSearch |
| **Amazon MQ** | Message Broker | Depends | Migrate on-premises AMQP/MQTT apps |

---

## Amazon SQS — Simple Queue Service

### Mental Model: The Restaurant Order Window

Imagine a busy restaurant during lunch rush:

```
    ┌─────────────────────────────────┐
    │  Kitchen (Producer)             │
    │                                 │
    │  Puts order tickets in the      │
    │  order window (queue)           │
    └──────────────┬──────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  ORDER WINDOW (SQS Queue)        │
    │ ┌──────────────────────────────┐ │
    │ │ [Ticket 1] [Ticket 2] [T3]   │ │
    │ │ [T4] [T5] [T6]...            │ │
    │ │                              │ │
    │ │ Kitchen is slammed? No       │ │
    │ │ problem, tickets pile up     │ │
    │ │ (persistent!)                │ │
    │ └──────────────────────────────┘ │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  Waitstaff (Consumers)          │
    │                                  │
    │  Pick up tickets when ready ✓    │
    │  Multiple staff can work in      │
    │  parallel (horizontal scaling) ✓ │
    │                                  │
    │  If staff crashes before        │
    │  delivering order, ticket       │
    │  reappears (at-least-once) ✓    │
    └──────────────────────────────────┘
```

**Key insight:** Kitchen and waitstaff operate independently at their own pace. Kitchen can blast orders (surge), queue buffers them, waitstaff drain the queue at steady pace.

### SQS Standard Queue: The Workhorse

#### Core Characteristics

| Attribute | Value | Why It Matters |
|-----------|-------|----------------|
| **Throughput** | Unlimited | No ceiling on messages/sec |
| **Queue Depth** | Unlimited messages | Won't reject your messages |
| **Message Retention** | Default: 4 days, Max: 14 days | After expiry, msg is deleted |
| **Message Size** | Max 256 KB | For larger payloads, use S3 + reference in SQS |
| **Delivery Guarantee** | At-least-once | Could receive same msg twice |
| **Ordering** | Best-effort | Generally FIFO but NOT guaranteed |
| **Visibility Timeout** | Default: 30 sec, Max: 12 hours | Time consumer has to process |

**Why "at-least-once"?** See visibility timeout section below.

#### Producing Messages: SendMessage

```
Pseudocode:
sqs.SendMessage(
    QueueUrl='https://sqs.region.amazonaws.com/account/queue-name',
    MessageBody=json.dumps({
        'orderId': '12345',
        'customerId': 'CUST-789',
        'items': ['pizza', 'salad', 'coke'],
        'total': '$49.90'
    }),
    DelaySeconds=0  # Optional: delay delivery
)
```

**Key point:** Message persists in queue until explicitly deleted by consumer.

#### Consuming Messages: ReceiveMessage → DeleteMessage

```
Consumer Loop:
  WHILE True:
    messages = sqs.ReceiveMessage(
        QueueUrl='...',
        MaxNumberOfMessages=10  # Receive up to 10 at once
    )
    
    FOR each message IN messages:
        TRY
            # Process message (write to DB, call API, etc.)
            processOrder(message.Body)
            
            # CRITICAL: Delete after successful processing!
            sqs.DeleteMessage(
                QueueUrl='...',
                ReceiptHandle=message.ReceiptHandle
            )
        CATCH error
            # Don't delete! Message will reappear after visibility timeout
            # and another consumer can retry
            log_error(error)
```

**WHY explicit delete?** This enables "at-least-once delivery."

#### Visibility Timeout: The Safety Net

When a consumer receives a message, it becomes **invisible** to other consumers for a period:

```
Timeline:
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  t=0: Consumer polls, receives message                      │
│       Message becomes INVISIBLE (visibility timeout = 30s)  │
│                                                              │
│       ┌─────────────────────────────────────┐               │
│       │                                     │               │
│       │  Consumer processing...             │               │
│       │  (writing to DB, calling APIs)      │               │
│       │                                     │               │
│       └─────────────────────────────────────┘               │
│                                                              │
│  t=15s: Processing completes ✓                             │
│         Consumer calls DeleteMessage(ReceiptHandle)        │
│         Message GONE from queue (success!)                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
                                        
                                        ▼▼▼ DIFFERENT SCENARIO ▼▼▼

┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  t=0: Consumer polls, receives message                      │
│       Message becomes INVISIBLE                            │
│                                                              │
│  t=15s: Consumer CRASHES ✗                                 │
│         (before calling DeleteMessage)                      │
│         Processing INCOMPLETE                              │
│                                                              │
│  t=30s: Visibility timeout EXPIRES                         │
│         Message REAPPEARS in queue                         │
│         Another consumer picks it up → retries             │
│                                                              │
│  ✓ Ensures no message is lost!                             │
│  BUT: Consumer must be idempotent                          │
│       (must handle being called twice)                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Configuration:**
- **Default:** 30 seconds (reasonable for most workloads)
- **Too short** (5 sec): If processing takes 10 seconds, message reappears early → duplicate processing
- **Too long** (2 hours): If consumer crashes, message stuck invisible for 2 hours → slow recovery

**If you need more time:** Consumer calls `ChangeMessageVisibility(ReceiptHandle, NewVisibilityTimeout)` to extend.

**⚠ Critical for Exam:** Understand visibility timeout controls at-least-once delivery.

#### Long Polling: Stop Wasting Money

**Short Polling (default, WASTEFUL):**
```
Consumer: "Any messages for me?"
SQS:      "No."
Consumer: *waits 1 second*
Consumer: "Any messages?"
SQS:      "No."
Consumer: *waits 1 second*
Consumer: "Any messages?"
SQS:      "No."
...
[Message arrives in queue]
Consumer: *doesn't check for 1 more second*
Consumer: "Any messages?"
SQS:      "Yes, here you go."

Result: Many empty API calls (cost!) + latency waiting for poll interval
```

**Long Polling (EFFICIENT, PREFERRED):**
```
Consumer: "Any messages? Wait up to 20 seconds."
SQS:      "Waiting... waiting... waiting..."
          [Message arrives!]
SQS:      "Yes! Here you go." (returns immediately)

Consumer: "Any messages? Wait up to 20 seconds."
SQS:      "Waiting... waiting... [20 sec expires]"
SQS:      "No messages, returning empty." (returns after 20s)

Result: Fewer API calls (lower cost!) + minimal latency
```

**Settings:**
```python
# At queue level (default for all ReceiveMessage calls):
sqs.SetQueueAttributes(
    QueueUrl='...',
    Attributes={'ReceiveMessageWaitTimeSeconds': '20'}  # 1-20 seconds
)

# Or per ReceiveMessage call:
sqs.ReceiveMessage(
    QueueUrl='...',
    WaitTimeSeconds=20  # Overrides queue default
)
```

**Rule:** Always enable long polling. Reduces cost by 90%+.

#### Multiple Consumers (Horizontal Scaling)

Multiple consumers can read from the same queue:

```
    ┌──────────────────────────────────┐
    │  SQS Standard Queue              │
    │ ┌──────────────────────────────┐ │
    │ │ Msg1  Msg2  Msg3  Msg4       │ │
    │ │ Msg5  Msg6  Msg7  Msg8  ...  │ │
    │ └──────────────────────────────┘ │
    └────────┬────────┬────────┬───────┘
             │        │        │
    ┌────────▼──┐ ┌──▼────────┐ ┌──────▼────┐
    │ Consumer1 │ │ Consumer2 │ │ Consumer3 │
    │           │ │           │ │           │
    │ Msg1      │ │ Msg2      │ │ Msg3      │
    │ Msg4      │ │ Msg5      │ │ Msg6      │
    │ Msg7      │ │ Msg8      │ │ ...       │
    └───────────┘ └───────────┘ └───────────┘

Each message delivered to ONE consumer (not all)
Messages roughly distributed round-robin
```

**Key difference from SNS:** With SNS, ALL subscribers get the message. With SQS, each message goes to ONE consumer.

#### SQS + Auto Scaling Group: The Canonical Pattern

**Problem:** During peak hours, 1000 orders/minute flood in. You need to process them without losing any.

**Solution:** Scale EC2 workers based on queue depth.

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  Orders come in faster than workers can handle        │
│                                                        │
│    ┌──────────────────────────────────┐              │
│    │  SQS Standard Queue              │              │
│    │  ApproximateNumberOfMessages: 200│              │
│    └──────────────────────────────────┘              │
│                │                                      │
│                ▼                                      │
│    ┌──────────────────────────────────┐              │
│    │  CloudWatch Alarm                │              │
│    │  IF messages > 100                │              │
│    │    THEN TRIGGER SCALE-OUT        │              │
│    └──────────────────────────────────┘              │
│                │                                      │
│                ▼                                      │
│    ┌──────────────────────────────────┐              │
│    │  Auto Scaling Group               │              │
│    │  Desired Capacity: 2 → 10         │              │
│    │  (launch 8 new EC2 workers)      │              │
│    └──────────────────────────────────┘              │
│                │                                      │
│                ▼                                      │
│    ┌──────────────────────────────────┐              │
│    │  SQS Queue depth decreases        │              │
│    │  Messages: 200 → 100 → 50 → 0    │              │
│    └──────────────────────────────────┘              │
│                │                                      │
│                ▼                                      │
│    ┌──────────────────────────────────┐              │
│    │  CloudWatch Alarm                │              │
│    │  IF messages < 10                 │              │
│    │    THEN TRIGGER SCALE-IN         │              │
│    └──────────────────────────────────┘              │
│                │                                      │
│                ▼                                      │
│    ┌──────────────────────────────────┐              │
│    │  Auto Scaling Group               │              │
│    │  Desired Capacity: 10 → 2         │              │
│    │  (terminate excess workers)      │              │
│    └──────────────────────────────────┘              │
│                                                        │
└────────────────────────────────────────────────────────┘

This is THE most commonly tested pattern for SQS on the exam!
```

**CloudWatch Metric to watch:**
- `ApproximateNumberOfMessages` = messages waiting in queue
- `ApproximateNumberOfMessagesNotVisible` = messages being processed
- `MessageDelaySeconds` = delay before message is visible

**Real-world flow:**
```
Autoscaling Target:
  - Scale Out: IF avg(ApproximateNumberOfMessages) > 100
              THEN add 2 instances
  - Scale In:  IF avg(ApproximateNumberOfMessages) < 10
              THEN remove 1 instance (min 2)
```

#### SQS as a Decoupling Buffer (Multi-Tier Architecture)

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  TIER 1: Web Frontend (Load Balanced)                   │
│  ┌────────────────┬────────────────┬────────────────┐   │
│  │ Load Balancer  │ Load Balancer  │ Load Balancer  │   │
│  └────────────────┼────────────────┼────────────────┘   │
│                   │                │                    │
│                   └────────┬────────┘                    │
│                            │                            │
│                    POST /api/orders                      │
│                    (accept order in <100ms)             │
│                            │                            │
│                            ▼                            │
│  ┌─────────────────────────────────────────────────┐    │
│  │  SQS Standard Queue (order-processing-queue)    │    │
│  │  ┌───────────────────────────────────────────┐  │    │
│  │  │ Order1  Order2  Order3  ...  Order100  │  │    │
│  │  │                                         │  │    │
│  │  │ Frontend can accept MORE orders        │  │    │
│  │  │ even if backend is slow!               │  │    │
│  │  └───────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────┘    │
│                            │                            │
│                            ▼                            │
│  TIER 2: Order Processing Servers (Auto Scaled)        │
│  ┌────────────┬────────────────┬─────────────────┐    │
│  │ Worker 1   │ Worker 2       │ Worker 3        │    │
│  │ Poll SQS   │ Poll SQS       │ Poll SQS        │    │
│  │ (ReceiveMsg)│ (ReceiveMsg)   │ (ReceiveMsg)    │    │
│  │ Process    │ Process        │ Process         │    │
│  │ (save DB)  │ (save DB)      │ (save DB)       │    │
│  │ Delete     │ Delete         │ Delete          │    │
│  │            │                │                 │    │
│  │ Scale as   │ Scale as       │ Scale as        │    │
│  │ queue      │ queue          │ queue           │    │
│  │ grows      │ grows          │ grows           │    │
│  └────────────┴────────────────┴─────────────────┘    │
│                                                          │
└──────────────────────────────────────────────────────────┘

Benefits:
✓ Frontend fast (response time independent of backend speed)
✓ Backend scales independently
✓ If backend goes down, orders don't get lost (in queue)
✓ Surge pricing on frontend doesn't affect backend servers
```

**Real-world use case:** eCommerce. Checkout page returns in 100ms. Order processing can take minutes.

#### SQS Security: Encryption & Access Control

**Encryption in Transit:**
- All SQS endpoints use HTTPS
- If you need encrypted data at-rest, use Server-Side Encryption (SSE)

**Encryption at Rest (Server-Side):**
```
sqs.SetQueueAttributes(
    QueueUrl='...',
    Attributes={
        'KmsMasterKeyId': 'arn:aws:kms:region:account:key/key-id',
        'KmsDataKeyReusePeriod': '300'  # Reuse key for 5 minutes
    }
)
```
Uses AWS KMS to encrypt messages in the queue.

**Client-Side Encryption (Optional):**
```
Before sending:
plaintext = "secret order data"
ciphertext = encrypt(plaintext, client_kms_key)
sqs.SendMessage(MessageBody=ciphertext)

Consumer:
ciphertext = message.Body
plaintext = decrypt(ciphertext, client_kms_key)
```

**Access Control (IAM Policies):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/OrderService"
      },
      "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage"
      ],
      "Resource": "arn:aws:sqs:region:account:order-queue"
    }
  ]
}
```

**SQS Access Policies (Resource-Based):**
Allow other AWS accounts or services to send/receive:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:region:account:my-queue",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "123456789012"
        }
      }
    }
  ]
}
```

**Example:** Allow S3 to send event notifications to SQS queue (cross-service).

#### SQS FIFO Queue: Order Matters!

For applications where ordering and exactly-once processing are critical.

##### FIFO Characteristics

| Aspect | Standard | FIFO |
|--------|----------|------|
| **Ordering** | Best-effort | Guaranteed (exact order) |
| **Delivery** | At-least-once | Exactly-once (no duplicates!) |
| **Throughput** | Unlimited | 300 msg/s (3000 msg/s with batching) |
| **Queue Name** | Any | **MUST end with `.fifo`** |
| **Deduplication** | N/A | Content-based OR Message ID-based |
| **Group ID** | N/A | Messages in same group processed in order |

**Queue naming:**
```
Standard: "order-queue"
FIFO:     "order-queue.fifo"  ← Note the suffix!
```

##### Why Exactly-Once + Ordering?

**Financial Transactions Example:**
```
Account balance: $1000

Message 1: Debit $500  (transferred to account B)
Message 2: Credit $300 (received from account C)

With Standard SQS (at-least-once):
Scenario 1 (good):
  Debit $500 → Balance: $500
  Credit $300 → Balance: $800 ✓

Scenario 2 (bad, if debit processed twice):
  Debit $500 → Balance: $500
  Debit $500 → Balance: $0 ✗ (insufficient funds!)
  Credit $300 → Balance: $300

Scenario 3 (bad, if order reversed):
  Credit $300 → Balance: $1300
  Debit $500 → Balance: $800 ✗ (incorrect accounting)

With FIFO SQS (exactly-once + ordered):
  ONLY Scenario 1 (good)
  Guaranteed order + no duplicates ✓
```

##### Message Group ID

Messages with the same "Message Group ID" are processed in order by one consumer at a time:

```
FIFO Queue: "orders.fifo"

┌─────────────────────────────────────────────┐
│ Message Group ID = "customer-123"           │
│  ├─ Msg1: Update item 1 quantity → 5       │
│  ├─ Msg2: Update item 1 quantity → 10      │
│  └─ Msg3: Checkout                         │
│                                             │
│  Consumer A processes all three in order ✓  │
│                                             │
├─ Message Group ID = "customer-456"          │
│  ├─ Msg4: Add item 2 to cart               │
│  └─ Msg5: Checkout                         │
│                                             │
│  Consumer B processes in order ✓            │
│                                             │
│  NOTE: Message Group ID="customer-123"     │
│        and "customer-456" processed in     │
│        parallel (Consumer A & B work       │
│        simultaneously) ✓                   │
└─────────────────────────────────────────────┘
```

**Key insight:** Different groups can be processed in parallel. Same group processed sequentially by one consumer.

##### Deduplication

**Content-based (Recommended):**
```
Deduplication ID = SHA-256(MessageBody)

If same message sent twice:
  SendMessage(Body="Order #123", MessageDeduplicationId=auto)
  SendMessage(Body="Order #123", MessageDeduplicationId=auto)
  
  → Second message rejected as duplicate (5-minute window) ✓
```

**Token-based:**
```
SendMessage(
    Body="Order #123",
    MessageDeduplicationId="my-dedup-token-xyz"
)
```
You explicitly provide a unique token; if reused within 5 minutes, rejected as duplicate.

---

## Amazon SNS — Simple Notification Service

### Mental Model: The Newspaper Publisher

```
    ┌─────────────────────────────────┐
    │  Newspaper Publisher            │
    │  (e.g., New York Times)         │
    │                                 │
    │  Publishes ONE edition          │
    │  of the newspaper               │
    └──────────────┬──────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  Distribution Center             │
    │  (SNS Topic)                     │
    │                                  │
    │  Receives the newspaper and      │
    │  automatically sends copies to   │
    │  all subscribers                 │
    └──────────────┬───────────────────┘
      ┌────────────┼────────────┬───────┐
      │            │            │       │
      ▼            ▼            ▼       ▼
    ┌──────┐   ┌──────┐   ┌──────┐  ┌────┐
    │ Home │   │Shop  │   │Lib   │  │....│
    │ Subs │   │Subs  │   │Subs  │  │More│
    │      │   │      │   │      │  │Sub.│
    └──────┘   └──────┘   └──────┘  └────┘
      │          │          │
      │          │          │
    Gets copy   Gets copy   Gets copy
    (same msg) (same msg)  (same msg)

Difference from SQS:
  SQS: one message → ONE consumer
  SNS: one message → ALL subscribers (their own copy)
```

### SNS Core Characteristics

| Feature | Detail |
|---------|--------|
| **Max Subscriptions** | 12.5 million per topic |
| **Max Topics** | 100,000 per account |
| **Message Persistence** | NO — message sent once. If subscriber down, msg LOST |
| **Delivery Guarantee** | At-least-once (but no persistence means if subscriber is unavailable, message is lost unless using SQS as subscriber) |
| **Latency** | Immediate push to subscribers |
| **Use Case** | Event notifications, fan-out to multiple services |

**Critical difference from SQS:** SNS does NOT persist. If subscriber is offline, message is dropped. (Solution: use SQS as the subscriber!)

### SNS Subscriber Types

SNS can push to many different services:

```
SNS Topic
  │
  ├─→ SQS Queue (adds persistence!)
  │
  ├─→ Lambda Function (serverless processing)
  │
  ├─→ HTTP/HTTPS Endpoint (webhook)
  │
  ├─→ Email Address (human notification)
  │
  ├─→ SMS (text message)
  │
  ├─→ Mobile Push (Apple APNS, Google GCM)
  │
  ├─→ Kinesis Data Firehose (stream to S3)
  │
  └─→ SQS FIFO Queue (ordered, exactly-once)
```

### AWS Services That Can Publish to SNS

SNS can be a target for events from:

```
CloudWatch Alarms
    → "CPU > 80%" → Alarm fires → SNS sends notification

Auto Scaling Groups
    → Scale-out event → SNS notification

S3 Events
    → Object created → SNS notification

DynamoDB
    → Item updated → SNS notification

RDS Events
    → DB instance restarted → SNS notification

CloudFormation
    → Stack creation/deletion → SNS notification

AWS DMS (Database Migration Service)
    → Replication task completed → SNS notification

SageMaker
    → Model training complete → SNS notification
```

### Publishing to SNS

**Topic Publish (most common):**
```python
# Publish to topic → all subscribers get the message
response = sns.publish(
    TopicArn='arn:aws:sns:region:account:my-topic',
    Subject='Order Notification',
    Message='Your order #12345 has been shipped!'
)
```

**Direct Publish (mobile apps):**
```python
# For mobile push notifications
response = sns.publish(
    TargetArn='arn:aws:sns:region:account:app/GCM/MyApp/uuid',
    Message='Your notification'
)
```

### SNS Security

- **HTTPS encryption in transit** (default)
- **KMS encryption at rest** (optional)
- **IAM policies** (who can publish/subscribe)
- **SNS Access Policies** (resource-based, cross-account)

---

## SNS + SQS Fan-Out Pattern

### The Problem: S3 Events to Multiple Services

You want to send S3 upload notifications to:
1. Email service (send customer confirmation email)
2. Audit logging service (log all uploads for compliance)
3. Thumbnail generation service (create thumbnails)

**Challenge:** S3 can only send events to ONE destination per event+prefix+suffix combination.

```
S3 Bucket
  │
  └─→ Only ONE target for "ObjectCreated" event
      (can't go to multiple queues directly)
```

### The Solution: SNS Fan-Out

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  S3 Bucket                                               │
│  (object uploaded: /invoices/invoice-123.pdf)           │
│         │                                                │
│         └──→ S3 Event Notification                      │
│              └──→ SNS Topic: "s3-uploads"               │
│                                                          │
│                  ┌──────────────────────────────┐        │
│                  │  SNS Topic: s3-uploads      │        │
│                  │  (receives event ONCE)      │        │
│                  └──┬───────────────────────┬──┘        │
│                     │                       │           │
│          ┌──────────▼────┐        ┌────────▼─────────┐  │
│          │                │        │                 │  │
│          ▼                ▼        ▼                 ▼  │
│    ┌──────────────┐ ┌──────────┐ ┌──────────┐       │   │
│    │ SQS-Email    │ │ SQS-Audit│ │ SQS-Thumb│       │   │
│    │ (persists    │ │ (persists│ │ (persists)       │   │
│    │  event)      │ │  event)  │ │ event)          │   │
│    └───────┬──────┘ └────┬─────┘ └────┬─────┘       │   │
│            │             │            │             │   │
│            ▼             ▼            ▼             │   │
│    ┌──────────────┐ ┌──────────┐ ┌──────────┐       │   │
│    │Email Service │ │Audit Log │ │Thumbnail │      │   │
│    │(consumer 1)  │ │(consumer2)│ │(consumer3)      │   │
│    └──────────────┘ └──────────┘ └──────────┘       │   │
│                                                          │
└──────────────────────────────────────────────────────────┘

Each subscriber gets its own persistent copy (via SQS)
Services can fail independently
New services added by just subscribing (no S3 config change!)
```

**Flow:**
1. S3 sends event to SNS topic (push)
2. SNS immediately delivers to ALL subscribers (SQS queues)
3. Each queue persists the event (if consumer down, event not lost)
4. Email service polls SQS-Email
5. Audit service polls SQS-Audit
6. Thumbnail service polls SQS-Thumb
7. Each processes independently, at their own pace

**Advantages over direct S3 → SQS:**
- Add new consumers without changing S3 configuration
- Multiple queues for different purposes (decoupling)
- Each service is independent

### SNS FIFO Topic

Similar to SQS FIFO, but for topics:

```
┌─────────────────────────────────────────────┐
│  SNS FIFO Topic: "orders.fifo"              │
│                                             │
│  Message 1: Order placed (group="order123")│
│  Message 2: Payment processed (g="order123")│
│  Message 3: Order shipped (group="order123")│
│  Message 4: Order placed (group="order456")│
│  ...                                        │
│                                             │
│  Guarantees:                                │
│  ✓ Ordered delivery (within group)         │
│  ✓ Exactly-once (no duplicates)            │
│  ✓ Only SQS FIFO can subscribe             │
│    (not HTTP, email, Lambda, etc.)        │
│                                             │
│         ├─→ SQS FIFO Queue: "shipping.fifo"│
│         ├─→ SQS FIFO Queue: "inventory.fifo"│
│         └─→ SQS FIFO Queue: "billing.fifo" │
│                                             │
│  Max throughput: 300 msg/s (same as SQS)  │
└─────────────────────────────────────────────┘
```

### SNS Message Filtering

By default, all subscribers get all messages. With filters, you can be selective:

```
┌────────────────────────────────────────────────┐
│  SNS Topic: "order-events"                     │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │ Message 1: {state: "PLACED", total: 50} │ │
│  └──────────────────────────────────────────┘ │
│                │                              │
│                ├─→ Subscription 1: SQS       │
│                │   Filter: {state: ["PLACED"]}
│                │   → MATCHES, delivers msg  │
│                │                            │
│                ├─→ Subscription 2: SQS      │
│                │   Filter: {state: ["FULFILLED"]}
│                │   → NO MATCH, drops msg    │
│                │                            │
│                └─→ Subscription 3: SQS      │
│                    Filter: NONE             │
│                    → Gets ALL messages      │
│                                                │
└────────────────────────────────────────────────┘
```

**Filter Policy Example:**
```json
{
  "state": ["PLACED", "CONFIRMED"],
  "source": ["mobile-app", "web"],
  "totalAmount": [
    {
      "numeric": [">", 100]
    }
  ]
}
```

Subscriber receives message only if ALL filter conditions match.

---

## Amazon Kinesis

### Why Kinesis? (What SQS Can't Do)

SQS is designed for **task queues** (discrete jobs):
- Job: "Process order #123"
- Job: "Generate invoice"
- One worker processes one job, then moves to next

**Kinesis is for data streams** (continuous flow of events):
- 1000 IoT sensors sending readings every second (1M events/sec)
- Website clickstream (millions of events/minute)
- Application logs (high-volume continuous data)
- Financial transaction stream (millions/second)

**Key requirements for streaming data:**
1. **Multiple independent consumers** reading the SAME data simultaneously
2. **Replay capability** (replay last hour of data for recovery/debugging)
3. **Ordering** (events in order, per logical partition)
4. **Real-time** (low latency, ~200ms)

**SQS limitations for streaming:**
- Can't replay (messages deleted after consumption)
- Only one consumer per message (not multiple independent readers)
- Not optimized for millions of tiny events/second

### Kinesis Data Streams Core Concepts

#### Shards: Partitioning for Parallelism

```
Kinesis Stream: "clickstream"
  │
  ├─→ Shard 1 ──→ Ordered sequence of records
  │   (1 MB/s input, 2 MB/s output)
  │
  ├─→ Shard 2 ──→ Ordered sequence of records
  │   (1 MB/s input, 2 MB/s output)
  │
  ├─→ Shard 3 ──→ Ordered sequence of records
  │   (1 MB/s input, 2 MB/s output)
  │
  └─→ Shard N ──→ Ordered sequence of records

Each shard is independent
→ Horizontal scalability
→ Multiple consumers can read different shards in parallel
```

**Partition Key:**
```
Producer sends event with partition key:

PutRecord(
    StreamName='clickstream',
    Data=json.dumps({
        'user_id': 'user-123',
        'action': 'click',
        'timestamp': '2025-03-05T10:15:00Z'
    }),
    PartitionKey='user-123'  ← This determines shard!
)

SHA-256(PartitionKey) → maps to a specific shard
→ All events for 'user-123' go to same shard
→ All events for 'user-456' go to same (or different) shard
→ Order is GUARANTEED within a shard
```

**Why partition keys?**
- **Ordering:** Events with same key → same shard → ordered
- **Load balancing:** Different keys → potentially different shards → parallelism

### Capacity Modes

#### Provisioned Mode (Manual, Cost-Effective)

You choose number of shards. You manage scaling.

```
Configuration:
  Stream: "clickstream"
  Shards: 10
  
Per shard capacity:
  Input:  1 MB/s or 1000 records/s (whichever lower)
  Output: 2 MB/s
  
Total capacity:
  Input:  10 MB/s or 10,000 records/s
  Output: 20 MB/s
  
Cost:
  $0.25/shard/hour (approximately)
  10 shards × $0.25 × 24 hours × 30 days ≈ $1800/month
  
Scaling:
  Manual resharding (takes minutes)
  Monitor metrics, increase shards during peaks
```

**When to use:** Predictable traffic, cost-sensitive, acceptable manual scaling.

#### On-Demand Mode (Auto-Scaling)

AWS automatically scales. You pay for throughput.

```
Configuration:
  Stream: "clickstream"
  Mode: On-Demand
  
Baseline capacity:
  4 MB/s input or 4000 records/s
  Auto-scales based on 30-day peak
  
Scaling:
  Automatic (managed by AWS)
  No manual resharding needed
  
Cost:
  $0.47/million records ingested
  + $0.12/GB scanned (for consumers)
  + $0.014/stream/hour
  
Example:
  100 million records/day
  = (100M × $0.47) / 1M = $47/day ≈ $1410/month
```

**When to use:** Unpredictable traffic, prefer simplicity, willing to pay more for auto-scaling.

### Kinesis Producers: Getting Data In

**AWS SDK (PutRecord):**
```python
# Single record
response = kinesis.put_record(
    StreamName='clickstream',
    Data=json.dumps({
        'user_id': 'user-123',
        'action': 'click',
        'timestamp': '...'
    }),
    PartitionKey='user-123'
)
```

**Kinesis Producer Library (KPL):**
```python
# High-throughput batching & compression
from aws_kinesis_agg import aggregated_record_pb2

producer = KinesisProducer(
    stream_name='clickstream',
    batch_size=100,  # Batch before sending
    batch_timeout_msec=1000  # Or timeout after 1 sec
)

# Library handles batching, compression, retry
producer.add_user_record(
    partition_key='user-123',
    data=json.dumps({...})
)
```

**Kinesis Agent:**
```
# Monitor log files, auto-send to stream
# Linux/Windows agent that tails logs
# Config file points to log paths
# → Automatic delivery to Kinesis
```

### Kinesis Consumers: Reading Data

**Kinesis Client Library (KCL):**
```python
# Handles shard management, checkpointing
# Ensures records processed in order per shard

from amazon_kinesis_client_library import RecordProcessorFactory

class MyRecordProcessor:
    def process_records(self, records, checkpointer):
        for record in records:
            # Process record
            print(f"Data: {record['data']}")
        
        # Checkpoint (remember position for recovery)
        checkpointer.checkpoint()
```

**Lambda Function:**
```python
def lambda_handler(event, context):
    # Kinesis invokes Lambda with batch of records
    for record in event['Records']:
        data = json.loads(record['kinesis']['data'])
        # Process data
    
    return {'statusCode': 200}
```

**Kinesis Data Firehose:**
```
Kinesis Stream → Firehose → S3 / Redshift / OpenSearch
(transforms, batches, delivers)
```

### Retention & Replay

**Data Retention:**
- Kinesis stores records for 1-365 days (default 24 hours)
- Consumers can replay data from any point within retention window

**Use case:**
```
t=0:  Application processes clickstream
      Consumer A reads events 1-1000, processes

t=3600: Bug discovered in processing logic
        Consumer B reads events 1-1000 AGAIN (from retention)
        Using corrected logic
        Recomputes metrics
        
No data lost! Can replay and reprocess.

(SQS can't do this — messages deleted after consumption)
```

---

## Amazon Data Firehose

### Mental Model: The Managed Delivery Service

```
Kinesis Data Streams:
  Producer → [Consumer manages buffering, batching, delivery]
  Complex, but flexible

Amazon Data Firehose:
  Producer → [AWS manages buffering, batching, delivery]
                      ↓
                  Destination (S3, Redshift, etc.)
  Simple, AWS handles complexity
```

**Key insight:** Firehose is NOT a data store. It's a delivery SERVICE. Data flows through it and lands in destinations.

### Core Characteristics

| Feature | Detail |
|---------|--------|
| **Latency** | ≥60 seconds minimum (buffering before delivery) |
| **Destinations** | S3, Amazon Redshift, Amazon OpenSearch, Splunk, MongoDB, Datadog, New Relic, custom HTTP |
| **Transformations** | Optional Lambda function to transform records |
| **Scaling** | Automatic (fully managed) |
| **Data Retention** | NO — data not stored in Firehose, delivered immediately to destination |
| **Failed Records** | Sent to separate S3 bucket for inspection |

### Typical Flow

```
Producers (multiple sources):
  ├─ Kinesis Data Stream
  ├─ CloudWatch Logs
  ├─ CloudWatch Events
  ├─ AWS IoT
  ├─ AWS SDK (direct PutRecord)
  └─ ...
       │
       ▼
  ┌─────────────────────────────────────────┐
  │  Amazon Data Firehose Delivery Stream   │
  │                                         │
  │  ┌──────────────────────────────────┐  │
  │  │ Buffer (size or time based)      │  │
  │  │ Default: 128 MB or 60 seconds    │  │
  │  │ (whichever comes first)          │  │
  │  └──────────────────────────────────┘  │
  │           │                             │
  │           ▼                             │
  │  ┌──────────────────────────────────┐  │
  │  │ Optional: Lambda Transform       │  │
  │  │ (modify records before delivery) │  │
  │  └──────────────────────────────────┘  │
  │           │                             │
  │           ▼                             │
  │  ┌──────────────────────────────────┐  │
  │  │ Deliver to Destination           │  │
  │  └──────────────────────────────────┘  │
  └─────────────────────────────────────────┘
       │
       ├─→ S3 (most common)
       │   Partitions by date/time/custom keys
       │   Format: Parquet, JSON, CSV
       │   Compression: gzip, zip, SNAPPY
       │
       ├─→ Redshift (via S3 intermediate)
       │   Firehose → S3 → Redshift COPY command
       │
       ├─→ Amazon OpenSearch
       │   Real-time indexing of logs
       │
       ├─→ 3rd Party Services
       │   Splunk, Datadog, New Relic, etc.
       │
       └─→ Custom HTTP Endpoint
           Your own API
```

### S3 Partitioning Example

```
Firehose can partition S3 delivery by:
  - Date (YYYY/MM/DD)
  - Time (HH:MM/SS)
  - Logical keys (region, service, etc.)

Result in S3:
s3://my-bucket/logs/
  ├─ 2026/03/05/
  │   ├─ 10/00/data-abc123.json.gz
  │   ├─ 10/01/data-def456.json.gz
  │   └─ 10/02/data-ghi789.json.gz
  ├─ 2026/03/06/
  │   ├─ 10/00/data-jkl012.json.gz
  │   └─ ...
```

This makes it easy to query with Athena or load into Redshift.

### Lambda Transformation Example

```python
def lambda_handler(event, context):
    output = []
    
    for record in event['records']:
        # Firehose sends base64-encoded data
        payload = json.loads(
            base64.b64decode(record['data'])
        )
        
        # Transform: add timestamp, enrich, filter
        payload['processed_at'] = datetime.now().isoformat()
        payload['enriched_region'] = lookup_region(payload['ip'])
        
        # Only deliver records from USA
        if payload['enriched_region'] == 'USA':
            output.append({
                'recordId': record['recordId'],
                'result': 'Ok',
                'data': base64.b64encode(
                    json.dumps(payload).encode()
                ).decode()
            })
        else:
            # Drop non-USA records
            output.append({
                'recordId': record['recordId'],
                'result': 'Dropped'
            })
    
    return {'records': output}
```

### Redshift Loading via Firehose

```
┌───────────────────────────────────────────┐
│  CloudWatch Logs (application logs)       │
│  → Firehose reads continuously            │
└─────────────────┬─────────────────────────┘
                  │
          ┌───────▼───────────────────┐
          │ Firehose                  │
          │ ├─ Batches records        │
          │ ├─ Compresses (GZIP)      │
          │ └─ Delivers to S3         │
          └───────┬───────────────────┘
                  │
          ┌───────▼────────────────┐
          │ S3 Intermediate Bucket │
          │ (firehose-data/)       │
          └───────┬────────────────┘
                  │
          ┌───────▼───────────────────┐
          │ Firehose triggers         │
          │ Redshift COPY command     │
          │ COPY table FROM 's3://...'│
          │ CREDENTIALS '...'         │
          └───────┬───────────────────┘
                  │
                  ▼
          ┌────────────────────┐
          │ Redshift Table     │
          │ Data loaded!       │
          │ Ready for analysis │
          └────────────────────┘
```

Firehose automates the Redshift COPY, no manual work needed.

### Failed Records Handling

```
Firehose Delivery Stream
  │
  ├─→ Successfully delivered → S3 destination
  │   (data.parquet)
  │
  └─→ Failed delivery → S3 error bucket
      (processing_failed/)
      → you can inspect, fix, replay
```

---

## Service Comparison

### SQS vs SNS vs Kinesis Data Streams

| Feature | SQS | SNS | Kinesis Data Streams |
|---------|-----|-----|---------------------|
| **Model** | Queue (pull) | Pub/Sub Topic (push) | Stream (pull) |
| **Multiple Consumers** | Each message → 1 consumer | All subscribers | Multiple independent consumers |
| **Persistence** | 4-14 days | NO (unless subscriber is SQS) | 1-365 days |
| **Ordering** | Best-effort (FIFO: guaranteed) | NO (FIFO topic: guaranteed) | Per-shard guaranteed |
| **Delivery Guarantee** | At-least-once (FIFO: exactly-once) | At-least-once | At-least-once |
| **Replay Capability** | NO | NO | YES (key differentiator!) |
| **Real-time Latency** | Seconds | Immediate | ~200ms |
| **Consumer Scalability** | Horizontal (add more workers) | Push (AWS manages) | Shards (manual or on-demand) |
| **Message Size** | Max 256 KB (256 KB SQS + S3) | Varies by subscriber type | 1 MB max |
| **Typical Latency** | 100s ms - seconds | <100ms | ~200ms |
| **Best Use Case** | Decouple microservices, task queues | Fan-out notifications, event broadcasting | Real-time analytics, stream processing |

### Detailed Comparison Matrix

#### When to Use Each Service

**Use SQS when:**
- Decoupling producer & consumer
- Task queue (jobs to process)
- Consumer polls for work
- Standard: high throughput, best-effort
- FIFO: guaranteed order & exactly-once

Example: Order service → SQS → Email/SMS/Analytics services

**Use SNS when:**
- Fan-out to multiple subscribers
- Push notifications
- Broadcast events to many services
- Immediate delivery

Example: S3 event → SNS → multiple SQS queues (fan-out)

**Use Kinesis Data Streams when:**
- High-volume, real-time data
- Multiple independent consumers
- Replay/reprocessing required
- Data ordering important

Example: IoT sensors → Kinesis → real-time dashboard + S3 archival

**Use Amazon Data Firehose when:**
- Load data into data stores
- No need for consumer management
- Transformation before delivery
- Near-real-time OK (≥60s buffer)

Example: CloudWatch Logs → Firehose → S3 → Redshift queries

---

## Amazon MQ

### The Problem: Existing On-Premises Message Brokers

You have existing on-premises systems using message brokers:

```
On-Premises Data Center:
  ├─ RabbitMQ (AMQP protocol)
  │   Python/Java apps using AMQP client libraries
  │
  ├─ Apache ActiveMQ (OpenWire protocol)
  │   Java apps using javax.jms API
  │
  └─ Custom messaging apps
      Using MQTT, STOMP, WSS protocols
```

**Challenge:** Migrate to AWS without rewriting application code.

**Naive solution:** Use SQS/SNS → Requires rewriting apps to use AWS SDK.

**Better solution:** Amazon MQ → Run ActiveMQ or RabbitMQ managed → Apps unchanged!

### What is Amazon MQ?

AWS-managed service for Apache ActiveMQ and RabbitMQ.

```
┌──────────────────────────────────────────────────┐
│  Your EC2 Instances / On-Premises Hybrid Cloud   │
│                                                  │
│  ┌─────────────┐         ┌────────────────────┐ │
│  │ Java App    │         │ Python App         │ │
│  │ (using JMS) │         │ (using AMQP lib)   │ │
│  └──────┬──────┘         └────────┬───────────┘ │
│         │                          │              │
│         └──────────────┬───────────┘              │
│                        │                          │
│                ┌───────▼─────────┐               │
│                │ AMQP / OpenWire │               │
│                │ Protocol        │               │
│                └───────┬─────────┘               │
│                        │                          │
└────────────────────────┼──────────────────────────┘
                         │
                   AWS Region
                         │
            ┌────────────▼────────────┐
            │ Amazon MQ Service       │
            │                         │
            │ ┌─────────────────────┐ │
            │ │ Active Broker (AZ1) │ │
            │ │ ActiveMQ/RabbitMQ   │ │
            │ └─────────────────────┘ │
            │          ↕              │
            │ ┌─────────────────────┐ │
            │ │ Standby Broker (AZ2)│ │
            │ │ (reads same EFS)    │ │
            │ └─────────────────────┘ │
            │          ↕              │
            │ ┌─────────────────────┐ │
            │ │ EFS Storage         │ │
            │ │ (shared, replicated)│ │
            │ └─────────────────────┘ │
            │                         │
            └─────────────────────────┘

Applications work with ZERO changes!
```

### Key Features

| Feature | Detail |
|---------|--------|
| **Protocols Supported** | AMQP (RabbitMQ), OpenWire (ActiveMQ), MQTT, STOMP, WSS |
| **High Availability** | Active-Standby with EFS shared storage, auto-failover |
| **Durability** | Messages persisted to EFS |
| **Integrations** | Not serverless like SQS (runs on EC2 behind the scenes) |
| **Scaling** | Manual (add more brokers), doesn't auto-scale like SQS |
| **Cost** | Per-broker + storage (more expensive than SQS) |

### High Availability Architecture

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Scenario 1: Normal Operation                           │
│                                                          │
│  Clients ──→ Active Broker (AZ1)                        │
│              ↓ (writes messages)                         │
│              EFS Storage (replicated across AZs)         │
│                                                          │
│  Standby Broker (AZ2)                                    │
│    ↑ (reads from same EFS, ready to take over)         │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Scenario 2: Active Broker Fails                        │
│                                                          │
│  Active Broker (AZ1) ✗ CRASHES                          │
│                                                          │
│  AWS detects failure (health check)                     │
│  ↓                                                       │
│  Standby takes over (AZ2)                              │
│  ↓                                                       │
│  Clients now connect to new Active (AZ2)               │
│  ↓                                                       │
│  Messages read from EFS (not lost!)                    │
│  ↓                                                       │
│  New Standby broker launched in AZ1                    │
│                                                          │
│  No message loss ✓                                      │
│  Brief interruption during failover (~30 seconds)      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### When to Use Amazon MQ

**Use Amazon MQ when:**
- Migrating on-premises ActiveMQ/RabbitMQ
- Apps use AMQP, MQTT, STOMP, OpenWire protocols
- Need both queues AND topics (ActiveMQ/RabbitMQ have both)
- Willing to manage a cluster (not fully serverless)

**Don't use Amazon MQ when:**
- Building new serverless app → use SQS/SNS
- Need unlimited scale → use SQS/SNS
- Don't have on-premises system to migrate

### Amazon MQ vs SQS/SNS

| Feature | Amazon MQ | SQS/SNS |
|---------|-----------|---------|
| **Setup Effort** | Medium (cluster management) | Minimal (serverless) |
| **Scaling** | Manual | Automatic |
| **Protocols** | AMQP, MQTT, STOMP, OpenWire | AWS SDK only |
| **Cost** | Higher (per broker) | Lower (pay per request) |
| **Use Case** | Legacy migration | New apps |
| **HA/DR** | Broker failover | Built-in redundancy |

---

## Points to Remember

### Critical Exam Concepts

1. **SQS Standard Queue**
   - At-least-once delivery (duplicates possible)
   - Best-effort ordering (not guaranteed)
   - Unlimited throughput
   - Visibility timeout controls retry behavior
   - Decouples producer & consumer

2. **SQS FIFO Queue**
   - Exactly-once delivery (no duplicates!)
   - Guaranteed FIFO order
   - Message Group ID ensures ordering within group
   - 300 msg/s throughput (3000 with batching)
   - Queue name MUST end in `.fifo`
   - Financial transactions, order processing

3. **Visibility Timeout**
   - Default: 30 seconds
   - If consumer doesn't delete within timeout, message reappears
   - Consumer can extend with `ChangeMessageVisibility`
   - Too short → duplicates. Too long → slow recovery.

4. **Long Polling**
   - Prefer over short polling
   - Reduces API calls (cost)
   - Fewer empty responses
   - Set `WaitTimeSeconds` = 1-20 seconds

5. **SQS + Auto Scaling Group**
   - Most common scaling pattern for SQS
   - CloudWatch metric: `ApproximateNumberOfMessages`
   - Scale out: if queue depth > threshold
   - Scale in: if queue depth < threshold
   - Workers drain queue at own pace

6. **SNS: Push, No Persistence**
   - Immediately pushes to all subscribers
   - If subscriber down, message LOST (unless subscriber is SQS!)
   - Multiple subscriber types (SQS, Lambda, email, SMS, etc.)
   - No persistence without persistent subscriber

7. **SNS + SQS Fan-Out**
   - S3 can only send to one destination → use SNS
   - SNS → multiple SQS queues
   - Each queue persists the event
   - Services scale independently
   - Add new consumers without changing publisher

8. **Kinesis Data Streams: Replay & Real-Time**
   - Key advantage: REPLAY (1-365 day retention)
   - Multiple independent consumers
   - Per-shard ordering (partition key)
   - Real-time: ~200ms latency
   - Shards: provisioned or on-demand
   - Use: IoT, clickstreams, log processing

9. **Amazon Data Firehose: Delivery Service**
   - Fully managed delivery (no consumer code)
   - ≥60 second buffering (NOT real-time)
   - Destinations: S3, Redshift, OpenSearch, Splunk, etc.
   - NO replay (doesn't store data)
   - Optional Lambda transformation

10. **Amazon MQ: Legacy Migration**
    - Managed ActiveMQ/RabbitMQ
    - Supports AMQP, MQTT, STOMP, OpenWire
    - Active-Standby HA with EFS shared storage
    - For migrating on-premises systems
    - NOT serverless, manual scaling

11. **SNS Message Filtering**
    - Per-subscription JSON filter policy
    - Only matching messages delivered to subscriber
    - Reduces unnecessary processing
    - Common with fan-out patterns

12. **SQS Delayed Delivery**
    - `DelaySeconds` parameter (0-900 seconds)
    - Consumer won't see message until delay expires
    - Useful for scheduled processing

---

## Interview Tips

### Pattern Recognition Questions

**"We need to decouple two microservices..."**
→ **Answer: SQS Standard Queue**
- Service A sends message to queue
- Service B polls queue at own pace
- They operate independently
- Scales horizontally (add more Service B instances)

**"We need to send the same notification to email, SMS, and Slack..."**
→ **Answer: SNS Topic + Multiple Subscriptions**
- Publish once to topic
- Each subscriber gets a copy
- Decouples publisher from subscribers
- Add new subscribers without changing publisher

**"We need guaranteed message order for financial transactions..."**
→ **Answer: SQS FIFO or SNS FIFO Topic**
- FIFO = exactly-once + ordered
- Standard won't work (duplicates + out-of-order possible)
- Use Message Group ID for logical ordering
- Lower throughput but correctness guaranteed

**"S3 sends events to MULTIPLE systems..."**
→ **Answer: SNS Fan-Out**
- S3 → SNS Topic
- SNS → SQS queue (email service)
- SNS → SQS queue (audit service)
- SNS → Lambda (image processing)
- One event, multiple destinations

**"We have real-time clickstream data (millions/second)..."**
→ **Answer: Kinesis Data Streams**
- SQS would bottleneck
- Need ordering per user (partition key)
- Multiple analytics consumers reading same stream
- Replay capability for recovery

**"We need to load logs into S3 & Redshift..."**
→ **Answer: Amazon Data Firehose**
- Firehose → S3 (with partitioning)
- Firehose → Redshift (auto COPY)
- No consumer management
- Handles buffering & batching

**"We're migrating our on-premises RabbitMQ app..."**
→ **Answer: Amazon MQ**
- Run ActiveMQ/RabbitMQ on AWS
- Apps use existing AMQP client libs
- Zero code changes
- HA with EFS shared storage

### Throughput Questions

**"We need unlimited throughput..."**
→ **SQS Standard** (unlimited msg/sec)

**"We need guaranteed order..."**
→ **SQS FIFO** (but limited to 300/sec) or **Kinesis** (per-shard)

**"We need real-time data analysis..."**
→ **Kinesis Data Streams** (~200ms latency)

**"We need near-real-time with minimal ops..."**
→ **Amazon Data Firehose** (≥60s buffering, fully managed)

### Cost Optimization

**SQS:**
- Pay per million requests
- Enable long polling (90% cost reduction)
- Batch reads (request 10 messages at once)
- Delete messages immediately after processing

**SNS:**
- Pay per million requests
- Filtering reduces downstream processing (SQS cost)

**Kinesis Streams:**
- Provisioned: $0.25/shard/hour → know your baseline
- On-Demand: $0.47/million records → for spiky traffic

**Firehose:**
- $0.03-0.04 per GB ingested (cheap!)
- Plus destination costs (S3, Redshift)

### Tricky Scenarios

**Scenario:** "Our app polls SQS but sometimes gets empty responses."
→ **Fix: Enable long polling**
- Reduces empty API calls
- Lower cost
- Lower latency

**Scenario:** "We're losing messages after sending to SNS."
→ **Problem: SNS has no persistence**
→ **Fix: Subscribe SQS queue to SNS topic**
- SNS pushes to SQS
- SQS persists
- Consumer polls SQS

**Scenario:** "We need to replay data from 1 hour ago."
→ **SQS:** Can't do it (messages deleted after consumption)
→ **Kinesis:** Easy! (retention 1-365 days)

**Scenario:** "We have 1 million IoT sensors sending readings/second."
→ **SQS:** Would bottleneck
→ **Kinesis:** Built for this (sharding)

**Scenario:** "Our Lambda is invoked but gets empty messages."
→ **Check:** Is SNS subscribed Lambda synchronous?
→ **Check:** Is visibility timeout too short on SQS?
→ **Check:** Did you enable long polling?

---

## Key Takeaways Summary

### Service Selection Flowchart

```
START: "I need messaging..."

├─ "...to decouple two services?"
│  └─→ SQS Standard (simple decoupling)
│      or SQS FIFO (guaranteed order)
│
├─ "...to notify multiple services?"
│  └─→ SNS Topic
│      + SQS queues (fan-out)
│
├─ "...for real-time analytics?"
│  └─→ Kinesis Data Streams
│      (replay + ordering)
│
├─ "...to load into data stores?"
│  └─→ Amazon Data Firehose
│      (S3, Redshift, OpenSearch)
│
├─ "...migrating on-prem broker?"
│  └─→ Amazon MQ
│      (ActiveMQ/RabbitMQ)
│
└─ "...not sure?"
   ├─ High volume + replay needed → Kinesis
   ├─ High volume + no replay → SQS
   ├─ Multiple subscribers → SNS
   └─ Managed delivery → Firehose
```

### The Most Important Concepts

1. **Asynchronous > Synchronous** for distributed systems
2. **Visibility timeout** enables at-least-once delivery
3. **SNS + SQS** = fan-out pattern (critical!)
4. **Kinesis** = only service with replay
5. **Firehose** = fully managed delivery (less code)
6. **SQS + ASG** = canonical scaling pattern
7. **Long polling** = cost optimization (do it!)

---

## Quick Reference — AWS CLI Commands

### SQS Commands

```bash
# Create a Standard SQS queue
aws sqs create-queue \
  --queue-name my-app-queue \
  --attributes '{
    "MessageRetentionPeriod": "345600",
    "VisibilityTimeout": "30",
    "ReceiveMessageWaitTimeSeconds": "20"
  }'

# Create a FIFO queue (name must end in .fifo)
aws sqs create-queue \
  --queue-name orders.fifo \
  --attributes '{
    "FifoQueue": "true",
    "ContentBasedDeduplication": "true"
  }'

# Send a message
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-app-queue \
  --message-body "Order #1234 created" \
  --delay-seconds 0

# Send a message to FIFO queue
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/orders.fifo \
  --message-body '{"orderId":"1234","amount":99.99}' \
  --message-group-id "customer-456" \
  --message-deduplication-id "order-1234"

# Receive messages (long polling — 20 seconds)
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-app-queue \
  --max-number-of-messages 10 \
  --wait-time-seconds 20 \
  --visibility-timeout 30

# Delete a message after processing
aws sqs delete-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-app-queue \
  --receipt-handle "AQEBwJnKyrHigUMZj6reyNuryz..."

# Change message visibility (extend processing time)
aws sqs change-message-visibility \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-app-queue \
  --receipt-handle "AQEBwJnKyrHigUMZj6reyNuryz..." \
  --visibility-timeout 60

# Get queue attributes (includes ApproximateNumberOfMessages)
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-app-queue \
  --attribute-names All

# Set queue policy (allow S3 to send events)
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-app-queue \
  --attributes file://queue-policy.json

# List queues
aws sqs list-queues --queue-name-prefix my-app

# Purge queue (delete all messages)
aws sqs purge-queue \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-app-queue

# Delete queue
aws sqs delete-queue \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-app-queue
```

### SNS Commands

```bash
# Create an SNS topic
aws sns create-topic --name my-notifications

# Create a FIFO topic
aws sns create-topic \
  --name orders.fifo \
  --attributes FifoTopic=true,ContentBasedDeduplication=true

# Subscribe an SQS queue to topic
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-notifications \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789012:my-app-queue

# Subscribe email to topic
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-notifications \
  --protocol email \
  --notification-endpoint user@example.com

# Subscribe Lambda to topic
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-notifications \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:us-east-1:123456789012:function:my-function

# Publish a message to SNS topic
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-notifications \
  --subject "Alert" \
  --message "High CPU usage detected on i-1234567890abcdef0"

# Publish with message attributes (for filtering)
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-notifications \
  --message "New order placed" \
  --message-attributes '{
    "state": {"DataType": "String", "StringValue": "PLACED"}
  }'

# Set subscription filter policy
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456789012:my-notifications:abc123 \
  --attribute-name FilterPolicy \
  --attribute-value '{"state": ["PLACED", "CONFIRMED"]}'

# List topics
aws sns list-topics

# List subscriptions for a topic
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-notifications
```

### Kinesis Data Streams Commands

```bash
# Create a Kinesis stream (provisioned mode, 2 shards)
aws kinesis create-stream \
  --stream-name my-clickstream \
  --shard-count 2

# Create on-demand stream
aws kinesis create-stream \
  --stream-name my-clickstream-ondemand \
  --stream-mode-details StreamMode=ON_DEMAND

# Put a record
aws kinesis put-record \
  --stream-name my-clickstream \
  --partition-key "user-123" \
  --data "$(echo '{"userId":"123","action":"click","page":"/home"}' | base64)"

# Put multiple records (batch)
aws kinesis put-records \
  --stream-name my-clickstream \
  --records '[
    {"Data": "cmVjb3JkMQ==", "PartitionKey": "pk1"},
    {"Data": "cmVjb3JkMg==", "PartitionKey": "pk2"}
  ]'

# Get shard iterator (to start reading)
aws kinesis get-shard-iterator \
  --stream-name my-clickstream \
  --shard-id shardId-000000000000 \
  --shard-iterator-type TRIM_HORIZON

# Read records using shard iterator
aws kinesis get-records \
  --shard-iterator "AAAAAAAAAAHSywljv0zEgPX4NyKdZ5wryMzP9yALs8NeKbUjp1IxtZs1Sp+KEd+xzzN29W9IjHa..."

# Describe stream
aws kinesis describe-stream-summary --stream-name my-clickstream

# List shards
aws kinesis list-shards --stream-name my-clickstream

# Add shards (reshard)
aws kinesis update-shard-count \
  --stream-name my-clickstream \
  --target-shard-count 4 \
  --scaling-type UNIFORM_SCALING

# Delete stream
aws kinesis delete-stream --stream-name my-clickstream
```

---

## AWS Documentation Links

- **SQS**: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html
- **SNS**: https://docs.aws.amazon.com/sns/latest/dg/welcome.html
- **Kinesis Data Streams**: https://docs.aws.amazon.com/streams/latest/dev/introduction.html
- **Amazon Data Firehose**: https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html
- **Amazon MQ**: https://docs.aws.amazon.com/amazon-mq/latest/developer-guide/welcome.html

---

**Document Created:** 2026-03-05  
**Version:** 1.0  
**Prepared for:** AWS Solutions Architect Associate (SAA-C03) Exam

This document emphasizes understanding the "why" behind each service. During the exam, focus on patterns: When would you use this? What problem does it solve? How does it handle failures?

Good luck on your SAA exam!
