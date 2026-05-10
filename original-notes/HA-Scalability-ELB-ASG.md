# High Availability & Scalability — ELB & Auto Scaling
> 📚 Official Docs: https://docs.aws.amazon.com/elasticloadbalancing/ | https://docs.aws.amazon.com/autoscaling/
> 🎯 SAA-C03 Exam Weight: Very High — core architecture pattern

---

## 🧠 Scalability vs High Availability — What's the Difference?

These two terms are related but distinct, and the exam loves testing this:

**Scalability** = the system can handle increased load.
**High Availability** = the system keeps running even when something fails.

```
Scalability Types:
┌──────────────────────────────────────────────────────────────┐
│  VERTICAL SCALING (Scale Up)                                 │
│                                                              │
│  t2.micro ──▶ t2.large ──▶ m5.4xlarge ──▶ u-12tb1.metal      │
│  (1 vCPU)    (2 vCPU)     (16 vCPU)      (448 vCPU)          │
│                                                              │
│  + Simple, no code changes needed                            │
│  - Hardware limit exists (you can't scale forever)           │
│  - Single point of failure                                   │
│  - Requires downtime to resize                               │
│                                                              │
│  HORIZONTAL SCALING (Scale Out)                              │
│                                                              │
│  1 instance ──▶ 3 instances ──▶ 10 instances ──▶ 100...      │
│                                                              │
│  + No hardware limit                                         │
│  + No single point of failure                                │
│  + AWS makes this easy (ASG + ELB)                           │
│  - App must be designed to be stateless/distributed          │
└──────────────────────────────────────────────────────────────┘
```

**High Availability** means running your app in **at least 2 AZs** so if one data center has a fire, power outage, or network issue, your app keeps running in the other AZ.

```
HIGH AVAILABILITY ARCHITECTURE:
                     ┌──────────────────────────────┐
Users ──▶ [ELB] ─────┤ EC2 Instance (AZ-1a)         │
                     │                              │
                     │ EC2 Instance (AZ-1b) ←───────┘
                     └──────────────────────────────┘
If AZ-1a fails → ELB routes all traffic to AZ-1b → App still works!
```

---

## ⚖️ Elastic Load Balancer (ELB) — The Traffic Director

An ELB sits in front of your instances and distributes incoming traffic across them. Think of it as a smart traffic cop — it knows which instances are healthy and routes traffic accordingly.

### Why Use an ELB?

```
WITHOUT ELB:                       WITH ELB:
                                   
Users ──▶ EC2-1 IP                Users ──▶ [ELB DNS]  ──▶ EC2-1
          EC2-2 IP                                     ──▶ EC2-2
          EC2-3 IP                                     ──▶ EC2-3
          
Problems:                         Benefits:
- Which IP do users use?          - Single DNS entry point
- How do users know about EC2-3?  - Automatic health checks
- What if EC2-2 goes down?        - SSL termination
- No SSL termination              - Stickiness (sessions)
```

### ELB Health Checks
ELB constantly polls each instance on a port and path (e.g., `GET /health` on port 80). If an instance doesn't return HTTP 200, ELB marks it unhealthy and stops sending traffic to it — automatically!

---

## 🔀 Types of Load Balancers — Which to Use When?

AWS has 4 load balancers, each operating at a different level of the network stack:

```
OSI Model Layer → Load Balancer Type
────────────────────────────────────
Layer 7 (HTTP)  → Application Load Balancer (ALB)  — most common
Layer 4 (TCP)   → Network Load Balancer (NLB)
Layer 3 (IP)    → Gateway Load Balancer (GWLB)
Legacy          → Classic Load Balancer (CLB) — deprecated, avoid
```

### 🟠 Application Load Balancer (ALB) — Layer 7

ALB is the most commonly used. It can "see" the **HTTP** request and make routing decisions based on the content.

```
User Request:
GET /api/users HTTP/1.1
Host: api.example.com
                │
                ▼
         ┌─────[*ALB]─────┐
         │    Routing     │
         │    Rules:      │
         └──┬──────────┬──┘
            │          │
   /api/* ──┘          └── /web/* 
            │                 │
   ┌────────▼───────┐  ┌──────▼──────────┐
   │  EC2 (API)     │  │  EC2 (Frontend) │
   │  Target Group  │  │  Target Group   │
   └────────────────┘  └─────────────────┘
```

**ALB Routing Rules:**
- Path-based: `/api/*` → backend service, `/web/*` → frontend service
- Hostname-based: `app.example.com` vs `admin.example.com`
- Query string: `?device=mobile` → mobile target group
- HTTP header values

**Target Group Types:**

- EC2 instances
- ECS tasks (containers)
- Lambda functions
- IP addresses (including on-premises servers)

![Application Load Balancer Diagram](images/alb.png)

> ⚠️ **Client IP note**: With ALB, your EC2 instances see the **ALB's IP** as the source, not the client's. The real client IP is in the `X-Forwarded-For` HTTP header.

#### Creating an ALB

```bash
# Create Application Load Balancer
aws elbv2 create-load-balancer \
  --name my-alb \
  --subnets subnet-0abc123def456 subnet-0def456abc123 \
  --security-groups sg-0abc123def456 \
  --scheme internet-facing \
  --type application

# Create a target group (where instances will be registered)
aws elbv2 create-target-group \
  --name my-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-1234567890abcdef0 \
  --target-type instance \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Register EC2 instances to target group
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/abc123 \
  --targets Id=i-1234567890abcdef0 Id=i-0987654321fedcba0

# Create HTTP listener (redirects to HTTPS)
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/abc123 \
  --protocol HTTP \
  --port 80 \
  --default-actions \
    Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'

# Create HTTPS listener with SSL certificate
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/abc123 \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/abc-123 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...

# Create path-based routing rule (/api/* → API backend)
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/my-alb/abc123/def456 \
  --conditions Field=path-pattern,Values='/api/*' \
  --priority 10 \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...api-targets...

# Describe load balancer
aws elbv2 describe-load-balancers --names my-alb
```

### 🔵 Network Load Balancer (NLB) — Layer 4

NLB operates at the TCP/UDP level — it doesn't look inside the packets, just routes based on port/protocol. This makes it incredibly fast.

```
Key NLB facts:
- Handles millions of requests per second
- Ultra-low latency (~100ms vs ~400ms for ALB)
- Preserves client IP (EC2 sees real client IP directly)
- ONE STATIC IP per AZ (supports Elastic IP)
- Not free tier eligible
```

**When to use NLB:**
- You need a **static IP** for your load balancer (e.g., firewall whitelist needs an IP, not DNS)
- Ultra-high performance, millions of rps
- Non-HTTP protocols (**TCP, UDP, TLS**)
- Gaming, financial trading systems

#### Creating an NLB

```bash
# Create Network Load Balancer (for ultra-high performance)
aws elbv2 create-load-balancer \
  --name my-nlb \
  --subnets subnet-0abc123def456 subnet-0def456abc123 \
  --type network \
  --scheme internet-facing

# NLB also uses target groups and listeners like ALB
aws elbv2 create-target-group \
  --name my-nlb-targets \
  --protocol TCP \
  --port 80 \
  --vpc-id vpc-1234567890abcdef0
```

### 🟣 Gateway Load Balancer (GWLB) — Layer 3

GWLB is special — it's designed for inserting **third-party security appliances** (firewalls, intrusion detection, deep packet inspection) transparently into your traffic flow.

```
Traffic flow with GWLB:
                                          ┌──────────────┐
User ──▶ [GWLB] ──▶ [Your Firewall/IDS] ──▶ [GWLB] ──▶   │ Your EC2 App │
                     (inspects packet)                   └──────────────┘
                     
Uses GENEVE protocol on port 6081
```

Use case: You're running a security-focused architecture and need to inspect all traffic through a Palo Alto or Fortinet firewall before it reaches your app.

**GWLB Architecture — from the course slides:**

![Gateway Load Balancer Diagram](images/image-20260307210707603.png)

> 💡 **Tutor note**: Notice the symmetric path — traffic flows OUT through GWLB to your firewall fleet (Target Group of 3rd-party appliances), gets inspected, then returns back through GWLB to reach the application. The Route Table is what forces traffic through GWLB. If you didn't have the route table entry, traffic would bypass the firewall entirely.

---

## 🔒 SSL/TLS on Load Balancers

**What is SSL Termination?**
**Your users connect to the ELB over HTTPS (encrypted). The ELB decrypts the traffic ("terminates" SSL) and forwards it to your EC2 instances over plain HTTP internally.** This saves CPU on your instances.

```
User ──HTTPS──▶ [ELB] ──HTTP──▶ EC2 (internal traffic, in VPC = safe)
               ↑
        SSL certificate managed here (via ACM)
```

**SNI (Server Name Indication)** — This is a TLS extension that solves a specific problem: how do you host multiple HTTPS websites (with different SSL certs) on one IP address?

```
Without SNI: One IP can only have ONE SSL certificate
With SNI:    Client tells the server which hostname it wants in the TLS handshake
             Server picks the right certificate
```

SNI is supported on **ALB and NLB** — NOT on the legacy CLB.

**SNI in action — from the course slides:**

![alb-ssl-cert-host-routing](data:image/jpeg;base64,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)

> 💡 **Tutor note**: The client says "I want www.mycorp.com" in the TLS handshake. The ALB looks up which target group that hostname maps to, selects the right SSL certificate (`www.mycorp.com` cert), and routes the request — all before any HTTP even happens. This is why ALB can serve two completely different HTTPS domains from a single listener on port 443.

---

## 🔄 Auto Scaling Group (ASG) — Automatic Elasticity

An ASG automatically launches and terminates EC2 instances based on demand. It's how you achieve both scalability and HA automatically.

```
                   CloudWatch Alarm: CPU > 70%
                              │
                              ▼
               ┌─────── Auto Scaling Group  ──────┐
               │  Min: 2  │  Desired: 4  │ Max: 8 │
               │                                  │
               │  [EC2-1] [EC2-2] [EC2-3] [EC2-4] │──▶ Launch [EC2-5]
               │    AZ-a    AZ-b    AZ-a    AZ-b  │
               └──────────────────────────────────┘
                              │
               Registers new instance with ELB automatically!
```

**ASG + CloudWatch scaling trigger — from the course slides:**

![auto-scaling-group-cloudwatch-trigger](data:image/jpeg;base64,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)

> 💡 **Tutor note**: The key insight in this diagram is the feedback loop. CloudWatch watches the average CPU across ALL instances in the ASG (not just one). When the average exceeds your threshold, it fires the alarm, which tells ASG to add more instances. The newly launched instances bring the average CPU down, and the alarm resets. This is closed-loop automatic scaling.

### ASG Launch Template
The "blueprint" for new instances — defines everything about the EC2 instance:
- AMI (which OS/software image)
- Instance type (t3.medium, etc.)
- Security Groups
- IAM Role
- User Data (bootstrap script)
- EBS volume configuration

#### Creating a Launch Template

```bash
# Create a launch template (blueprint for ASG)
aws ec2 create-launch-template \
  --launch-template-name my-lt \
  --version-description "v1" \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "t3.micro",
    "SecurityGroupIds": ["sg-0abc123def456"],
    "UserData": "IyEvYmluL2Jhc2gKeXVtIHVwZGF0ZSAteQpodHRwZCAtdA=="
  }'

# The UserData is base64-encoded. Here's plain:
# #!/bin/bash
# yum update -y
# httpd -t
```

### Creating an Auto Scaling Group

```bash
# Create Auto Scaling Group attached to ALB
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --launch-template LaunchTemplateName=my-lt,Version='$Latest' \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 2 \
  --vpc-zone-identifier "subnet-0abc123def456,subnet-0def456abc123" \
  --target-group-arns arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/abc123

# Check ASG status
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-asg

# Manually set desired capacity (useful for testing scaling down)
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name my-asg \
  --desired-capacity 4 \
  --honor-cooldown
```

### ASG Scaling Policies — 4 Types

**1. Target Tracking Scaling** (Simplest, recommended)
```
"Keep average CPU at 50%"
ASG figures out when to add/remove instances automatically.
```

```bash
# Create Target Tracking scaling policy (CPU at 70%)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 70.0,
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

**2. Step Scaling**
```
CloudWatch Alarm: CPU > 70% → Add 2 instances
CloudWatch Alarm: CPU < 30% → Remove 1 instance
```

```bash
# Create a CloudWatch alarm for high CPU
aws cloudwatch put-metric-alarm \
  --alarm-name my-asg-cpu-high \
  --alarm-description "Alarm when CPU > 70" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=AutoScalingGroupName,Value=my-asg \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:autoscaling:us-east-1:123456789012:autoScalingGroup:abc123:auto-scaling-group/my-asg

# Create step scaling policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name add-capacity \
  --policy-type StepScaling \
  --adjustment-type ChangeInCapacity \
  --step-adjustments MetricIntervalLowerBound=0,ScalingAdjustment=2
```

**3. Scheduled Scaling**
```
"Every Friday at 5 PM, set desired capacity to 10"
For predictable traffic patterns (e.g., end-of-week batch, market open)
```

```bash
# Scale up for business hours (Mon-Fri 9 AM)
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name my-asg \
  --scheduled-action-name scale-up-weekdays \
  --recurrence "0 9 * * MON-FRI" \
  --desired-capacity 5

# Scale down for off-hours (6 PM)
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name my-asg \
  --scheduled-action-name scale-down-evening \
  --recurrence "0 18 * * MON-FRI" \
  --desired-capacity 2
```

**4. Predictive Scaling** (ML-powered)

```
AWS analyzes your historical traffic patterns and provisions capacity
BEFORE demand arrives (not reactive — proactive!)
Best for regular, cyclical patterns.
```

![predictive-scaling-forecast](data:image/jpeg;base64,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8t+WGWch7I4oY9uaWSR4Yxg36DdzgN/DvVZ0BgsrjMxLYvVOxidXcwO7RruvM07dCfUVZdUYaLPYWbGyzSVy9zJI5o/fRSMeHseN/U5oO3ipNWuF1u+EOWyVSlJZ1DjYquz2tjZjnzXnO3372thDgRt4Aj2r0l1fp6GtWnkvnsbDBIyRsEjmtaXcvM8hvoAO6Eu22PfsuK/gtSZLHR17+oqTpIrDJAYcfJFHK0NcCyVon3cCXNd0c0btHQhQw0NkG1W6ejyUYoS0Ja9yd1Qkyskne9zGen6DtnEcx5vXtumzLdcPPuOGaZinyWI7UjiyPtKsrI5HBpcWtkLQxxDQTsCT0PqXJLqjHP7M0JmStdcirGWSOVkTi9/KRHIGFr3b9Ngdt+8hRrdIW3aur5qfM9vDXvPtxRPheZAHQvjEXMZOUNbzkjZgPr33JSjpK/DiaGHfl4JKGNtwT0wKRbKGRScwY93aEO6Dl5g0evYpsbu3C6xxORklh/fVZ7b7qLDPUlY18g5ugcWgbnld033HQHqRvNY+9VvMlkqTCVkUz4HuAIAexxa4de/Ygjp4gqoaiwGQpYLMwUXuvSXbvlmPjirbSV7JeJGue8v2LA9oO/KNhuOu6tOncazD4SnjWPMnYRBr5D3yP73PPtc4kn2lJoSSIiio/Tf+b2N/RIv+gKQHco/Tf+b2N/RIv+gKQHciRwIiIoiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICg8NkIa+GpV5oLzZIq8bHt8hmOxDQCOjVOIjFH+d6vwN/wCgT/cTzvV+Bv8A0Cf7i7+idEXdwedqnwN/6BP9xPO1T4G/9An+4u/onRDdwed6vwN/6BP9xPO9X4G/9An+4u/onRDdwedqnwN/6BP9xPO1T4G/9An+4u/onRDdwedqnwN/6BP9xPO1T4G/9An+4u/onRDdwed6nwN/6BP9xPO9X4K/9An+4u/onRDdwed6vwV/6BP9xPO9T4G/9An+4u/onRDdwedqnwN/6BP9xPO1T4G/9An+4u/onRDdwedqnwN/6BP9xPO1T4G/9An+4u/onRDdwed6vwN/6BP9xPO9X4G/9An+4u/onRDdwedqnwN/6BP9xPO9T4G/9An+4u/onRDdw4Bj48HQjkY5j2VY2ua4bEENG4IXeiIoiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIg//Z)

> 💡 **Tutor note**: The three charts tell the story perfectly. Step 1: AWS looks at your last weeks of traffic (historical load line). Step 2: Its ML model generates a forecast (dashed line) predicting next week's demand. Step 3: It creates scheduled scaling actions AHEAD of the predicted spikes — so capacity is already there when the load arrives. This is the opposite of reactive scaling, which always lags behind demand by a cooldown period.

### Scaling Cooldown Period
After a scaling event, ASG waits 300 seconds (default) before another scaling action. This prevents ASG from rapidly adding/removing instances based on short spikes.

> 💡 **Pro tip**: Use pre-built AMIs with your software already installed to reduce launch time and shorten the cooldown period you need.

### ASG Termination Policy
By default, ASG terminates instances in the AZ with the most instances first, then the oldest launch template. You can customize this.

---

## 🍪 Sticky Sessions (Session Affinity)

By default, ELB distributes requests evenly. But some applications store session data locally on the EC2 instance (e.g., a shopping cart in server memory). If user goes to a different instance, their cart is lost!

**Sticky Sessions** solve this by ensuring the same client always goes to the same instance.

```
With sticky sessions:
User's browser ──▶ [ALB] ──▶ EC2-1 (always! thanks to session cookie)
```

**Sticky Sessions routing — from the course slides:**

![alb-ssl-cert-host-routing](data:image/jpeg;base64,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)

> 💡 **Tutor note**: Look at the color-coded lines. Client 1 (orange) always goes to EC2 Instance 1. Client 2 (blue) always goes to EC2 Instance 1. Client 3 (green) always goes to EC2 Instance 2. The ALB uses a cookie (`AWSALB`) to remember this mapping — every request that cookie comes back in gets routed to the same target, regardless of which AZ it's in. This is fine for a few users, but imagine 90% of users hitting the same instance — that instance becomes a hotspot while the other idles.

```bash
# Enable sticky sessions on target group
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/abc123 \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=86400
```

Cookie types:
- **Application-based**: App generates the cookie, contains custom info, you choose the name
- **Duration-based**: ALB generates the cookie (`AWSALB`), purely time-based



> ⚠️ **Trade-off**: Sticky sessions can cause unequal load distribution. Better approach: use a shared cache (ElastiCache) or DynamoDB to store session state — then any instance can handle any request.

---

## 🌐 Cross-Zone Load Balancing

```
WITHOUT Cross-Zone:
AZ-1 (2 instances) ← receives 50% of traffic → each instance gets 25%
AZ-2 (8 instances) ← receives 50% of traffic → each instance gets 6.25%
Result: Uneven load per instance!

WITH Cross-Zone:
Total 10 instances across both AZs → each gets exactly 10% of traffic
```

```bash
# Enable cross-zone load balancing on ALB (usually default)
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/abc123 \
  --attributes \
    Key=load_balancing.cross_zone.enabled,Value=true

# Note: NLB has cross-zone disabled by default (costs extra)
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/net/my-nlb/abc123 \
  --attributes \
    Key=load_balancing.cross_zone.enabled,Value=true  # Incurs inter-AZ charges
```

| Load Balancer | Cross-Zone Default | Inter-AZ Cost |
|---------------|-------------------|---------------|
| ALB | ✅ Enabled | Free |
| NLB | ❌ Disabled | $ charged if enabled |
| GWLB | ❌ Disabled | $ charged if enabled |

---

## ⭐ Interview Tips & Key Points to Remember

- **ALB = Layer 7 (HTTP)** — content-based routing; **NLB = Layer 4 (TCP/UDP)** — ultra-low latency, static IP
- **ALB hides client IP** — read `X-Forwarded-For` header; **NLB preserves client IP**
- **NLB has 1 static IP per AZ** — use when clients need to whitelist a fixed IP
- **GWLB = 3rd party security appliances** (firewalls, IDS) — uses GENEVE protocol port 6081
- **SNI = multiple SSL certs on one ALB/NLB** — CLB does NOT support SNI
- **ASG Cooldown = 300s default** — prevents rapid scaling thrash
- **Predictive Scaling** = ML-based proactive scaling — for known cyclical patterns
- **Sticky sessions trade-off**: convenience vs uneven load; better to use shared session store (ElastiCache)
- **Cross-zone load balancing**: ALB always on (free); NLB/GWLB off by default (costs extra)
- **ASG always runs in min capacity** — even at zero demand, min instances are running
- Scenario: "Need static IP for load balancer" → **NLB with Elastic IP**
- Scenario: "Route /api to microservice A, /web to microservice B" → **ALB path-based routing**
- Scenario: "Insert firewall appliance before traffic reaches app" → **GWLB**

---

## Quick Reference — AWS CLI Commands

### Load Balancer Creation

```bash
# Create Application Load Balancer
aws elbv2 create-load-balancer \
  --name my-alb \
  --subnets subnet-0abc123def456 subnet-0def456abc123 \
  --security-groups sg-0abc123def456 \
  --type application \
  --scheme internet-facing

# Create Network Load Balancer
aws elbv2 create-load-balancer \
  --name my-nlb \
  --subnets subnet-0abc123def456 subnet-0def456abc123 \
  --type network \
  --scheme internet-facing

# Describe load balancers
aws elbv2 describe-load-balancers --names my-alb
```

### Target Groups & Routing

```bash
# Create target group
aws elbv2 create-target-group \
  --name my-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-1234567890abcdef0 \
  --health-check-path /health \
  --health-check-interval-seconds 30

# Register targets (EC2 instances)
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/abc123 \
  --targets Id=i-1234567890abcdef0 Id=i-0987654321fedcba0

# Create listener (HTTP → HTTPS redirect)
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/abc123 \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'

# Create path-based routing rule
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/... \
  --conditions Field=path-pattern,Values='/api/*' \
  --priority 10 \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...
```

### Launch Templates

```bash
# Create launch template
aws ec2 create-launch-template \
  --launch-template-name my-lt \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "t3.micro",
    "SecurityGroupIds": ["sg-0abc123def456"],
    "UserData": "IyEvYmluL2Jhc2gKeXVtIHVwZGF0ZSAteQ=="
  }'

# Describe launch templates
aws ec2 describe-launch-templates --launch-template-names my-lt
```

### Auto Scaling Groups

```bash
# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --launch-template LaunchTemplateName=my-lt,Version='$Latest' \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 2 \
  --vpc-zone-identifier "subnet-0abc123def456,subnet-0def456abc123" \
  --target-group-arns arn:aws:elasticloadbalancing:...

# Describe ASG
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-asg

# Set desired capacity
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name my-asg \
  --desired-capacity 4
```

### Scaling Policies

```bash
# Create Target Tracking scaling policy (CPU-based)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 70.0
  }'

# Create scheduled scaling action (scale up for business hours)
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name my-asg \
  --scheduled-action-name scale-up-weekdays \
  --recurrence "0 9 * * MON-FRI" \
  --desired-capacity 5
```

