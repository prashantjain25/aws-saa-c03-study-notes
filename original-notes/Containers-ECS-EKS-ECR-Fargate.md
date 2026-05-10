# AWS SAA-C03: Containers on AWS — ECS, EKS, ECR & Fargate

**Study Guide by [Expert-led Style]**  
*Last Updated: 2026-03-05*

---

## Quick Navigation

- [Official AWS Documentation](#official-aws-documentation)
- [Section 1: Docker Containers Overview](#section-1-docker-containers-on-aws-overview)
- [Section 2: Amazon ECS](#section-2-amazon-ecs)
- [Section 3: ECS — IAM Roles](#section-3-ecs--iam-roles)
- [Section 4: ECS — Load Balancer Integration](#section-4-ecs--load-balancer-integration)
- [Section 5: ECS — Data Volumes (EFS)](#section-5-ecs--data-volumes-efs)
- [Section 6: ECS Service Auto Scaling](#section-6-ecs-service-auto-scaling)
- [Section 7: ECS Integration Patterns](#section-7-ecs-integration-patterns)
- [Section 8: Amazon ECR](#section-8-amazon-ecr--elastic-container-registry)
- [Section 9: Amazon EKS](#section-9-amazon-eks--elastic-kubernetes-service)
- [Section 10: AWS App Runner](#section-10-aws-app-runner)
- [Section 11: AWS App2Container](#section-11-aws-app2container-a2c)
- [Container Decision Tree](#container-decision-tree)
- [Points to Remember](#points-to-remember-exam-focus)
- [Interview Tips & Scenarios](#interview-tips--scenarios)

---

## Official AWS Documentation

Before diving in, bookmark these official resources:

- **ECS Guide**: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html
- **EKS Guide**: https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html
- **ECR Guide**: https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html
- **Fargate Guide**: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html
- **App Runner Guide**: https://docs.aws.amazon.com/apprunner/latest/dg/what-is-apprunner.html

---

# SECTION 1: Docker Containers on AWS — Overview

## The Problem: "It Works on My Machine"

Imagine you're a developer. Your app runs perfectly on your laptop. You ship it to the production server, and it explodes because the production server has a different Python version, missing libraries, different environment variables, etc.

**Docker solves this problem**: A Docker container is like a shipping container for software. It packages your application + ALL its dependencies (libraries, runtime, OS layer) into a single unit. "Run this container" = "run this app with guaranteed dependencies everywhere."

---

## The 4 Container Services on AWS

When you move Docker to AWS, you need to answer:
1. **WHERE** do I run my containers? → **ECS** or **EKS**
2. **WHO** manages the servers? → **EC2 Launch Type** (you) or **Fargate** (AWS)
3. **HOW** do I store my container images? → **ECR**

### Quick Comparison

| Service | What it does | Language | Best for |
|---------|-------------|----------|----------|
| **Amazon ECS** | Container orchestration (AWS native) | AWS API | AWS-first, simpler deployments |
| **Amazon EKS** | Managed Kubernetes | Kubernetes/kubectl | Existing Kubernetes shops, cloud portability |
| **AWS Fargate** | Serverless container runtime | Works with both ECS & EKS | No infrastructure management |
| **Amazon ECR** | Docker image registry | Docker | Storing & versioning container images |

### Mental Model: Shipping Analogy

```
┌─────────────────────────────────────────────────────────┐
│  BEFORE DOCKER: Chaos                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │App on Laptop │  │App on Server │  │App on Cloud  │  │
│  │Python 3.8    │  │Python 3.6    │  │Python 3.10   │  │
│  │Deps: OK      │  │Deps: MISSING │  │Deps: WRONG   │  │
│  │"Works here!" │  │ERROR!        │  │ERROR!        │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  WITH DOCKER: Consistency                               │
│  ┌──────────────────┐  ┌──────────────────┐             │
│  │  Docker Image    │  │  Docker Image    │             │
│  │  ┌────────────┐  │  │  ┌────────────┐  │             │
│  │  │App         │  │  │  │App         │  │             │
│  │  │Python 3.8  │  │  │  │Python 3.8  │  │             │
│  │  │All Deps    │  │  │  │All Deps    │  │             │
│  │  └────────────┘  │  │  └────────────┘  │             │
│  │                  │  │                  │             │
│  │Runs everywhere   │  │Runs everywhere   │             │
│  │ same way         │  │ same way         │             │
│  └──────────────────┘  └──────────────────┘             │
└─────────────────────────────────────────────────────────┘
```

---

# SECTION 2: Amazon ECS

## What is ECS?

**Amazon Elastic Container Service (ECS)** is AWS's proprietary container orchestration platform. Think of it as AWS's answer to Kubernetes, but simpler and AWS-native.

**Core concept**: "Launch Docker containers on AWS" = "Launch ECS Tasks on ECS Clusters"

- **Cluster** = Logical group of compute resources (EC2 instances or Fargate)
- **Task** = Running instance of a Docker container (equivalent to a container)
- **Task Definition** = Blueprint for a task (specifies Docker image, CPU, RAM, environment variables, IAM role, etc.)
- **Service** = Manages multiple tasks (ensures a desired number of tasks always run)

---

## EC2 Launch Type

### The Responsibility Model

```
┌──────────────────────────────────────────────────────────┐
│                    YOUR RESPONSIBILITY                   │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Provision & Manage EC2 Instances                   │  │
│  │ Choose instance types                              │  │
│  │ Update security groups, IAM roles                  │  │
│  │ Patch/update the instances                         │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                 AWS'S RESPONSIBILITY                     │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Schedule & run containers (tasks)                  │  │
│  │ Health checks, task restarts                       │  │
│  │ Container networking                              │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Architecture: EC2 Launch Type

```
ECS Cluster (Amazon ECS)
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────┐  │
│  │  EC2 Instance 1  │  │  EC2 Instance 2  │  │EC2 Inst 3│  │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │ ┌──────┐ │  │
│  │  │Task A      │  │  │  │Task B      │  │  │ │Task C│ │  │
│  │  │[nginx]     │  │  │  │[nodejs]    │  │  │ │[api] │ │  │
│  │  └────────────┘  │  │  └────────────┘  │  │ └──────┘ │  │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │          │  │
│  │  │Task D      │  │  │  │Task E      │  │  │ (scale)  │  │
│  │  │[redis]     │  │  │  │[api]       │  │  │          │  │
│  │  └────────────┘  │  │  └────────────┘  │  │          │  │
│  │                  │  │                  │  │          │  │
│  │  Docker daemon   │  │  Docker daemon   │  │ Docker   │  │
│  │  + ECS Agent     │  │  + ECS Agent     │  │ + Agent  │  │
│  └──────────────────┘  └──────────────────┘  └──────────┘  │
│                                                             │
│  (Each instance registers with ECS cluster)               │
└─────────────────────────────────────────────────────────────┘
```

### How EC2 Launch Type Works

1. **Setup**: You launch EC2 instances with an ECS-optimized AMI (has Docker + ECS Agent pre-installed)
2. **Registration**: ECS Agent on each instance registers it with the ECS cluster
3. **Task Scheduling**: When you create a service, ECS decides which EC2 instance gets which task
4. **Scaling**: Add more tasks → ECS schedules on available instances. If capacity full → add more EC2 instances (separate Auto Scaling Group)

### When to Use EC2 Launch Type

✅ **Use EC2 Launch Type when:**
- You want full control over instance types (need GPU instances? Specific CPU architectures?)
- You're using Reserved Instances or Savings Plans for cost optimization
- You have long-running services (EC2 instances running 24/7 pays for itself)
- You need custom AMIs with pre-installed software
- Your workload is predictable (can reserve instances)

❌ **Don't use when:**
- Your workload is unpredictable and bursty
- You want zero infrastructure management (use Fargate instead)

### Key Insight: WHY Separate Instance Profile?

Each EC2 instance has an **IAM Instance Profile** (attached to the EC2 instance). This role is used by the **ECS Agent** running on that instance.

**Example Task Flow**:
```
1. ECS says "Run Task A on EC2-instance-1"
2. ECS Agent (on instance-1) receives the command
3. ECS Agent uses Instance Profile credentials to:
   → Call ECR API to pull Docker image
   → Call CloudWatch Logs API to send logs
   → Call Secrets Manager to fetch DB password
4. Once image is pulled, Docker daemon runs the container
5. Task A (the container) runs using its own Task Role credentials
```

This separation is CRITICAL for security (more in Section 3).

---

## Fargate Launch Type

### The Serverless Promise

With Fargate, AWS manages ALL the infrastructure. You only care about your tasks.

```
┌──────────────────────────────────────────────────────────┐
│                 AWS'S RESPONSIBILITY                     │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Provision compute infrastructure                   │  │
│  │ Manage EC2 instances (you never see them)          │  │
│  │ Patch/update infrastructure                        │  │
│  │ Handle capacity management                         │  │
│  │ Billing is granular (per task)                     │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                    YOUR RESPONSIBILITY                   │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Define task definitions (CPU, RAM)                 │  │
│  │ Define services & desired task count               │  │
│  │ Set up networking (VPC, security groups)           │  │
│  │ Configure auto-scaling policies                    │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Architecture: Fargate Launch Type

```
Your Code
  │
  ▼
Docker Image (in ECR)
  │
  ▼
Task Definition
  ├─ Docker image
  ├─ CPU: 256 (.25 vCPU)
  ├─ Memory: 512 MB
  └─ Task Role: [S3 access, DynamoDB access, etc.]
  │
  ▼
ECS Service (Fargate)
  │ "Run 3 tasks of this definition"
  │
  ▼
AWS Fargate (no servers you manage)
  ├─ Task 1 ─┐
  ├─ Task 2  ├─ All running on AWS's infrastructure
  └─ Task 3 ─┘
```

### Fargate vs EC2 Launch Type: Deep Comparison

| Dimension | EC2 Launch Type | Fargate |
|-----------|-----------------|---------|
| **Infrastructure** | You provision EC2 instances | AWS manages serverless infrastructure |
| **Instance Management** | You manage OS, patches, updates | Zero infrastructure maintenance |
| **Scaling** | Scale EC2 instances + tasks (2 things!) | Scale only tasks (1 thing!) |
| **Cost Model** | Pay for EC2 24/7 (even if idle) | Pay only for vCPU + RAM * task runtime |
| **Best for predictable** | Yes (reserve instances) | No (better for variable) |
| **Best for bursty** | No (have idle capacity) | Yes (scale up/down instantly) |
| **Learning curve** | Steeper (manage instances) | Easier (just define tasks) |
| **Use existing AMIs** | Yes | No (use container images) |
| **Control** | Full (instance type, monitoring, software) | Limited (task-level only) |

### Fargate Cost Example

**Scenario**: Web API serving traffic
- **EC2 t3.medium**: $0.0416/hour → $300/month (running 24/7, even with 10% CPU usage)
- **Fargate 1vCPU, 2GB RAM**: $0.04048/hour × hours used only → $200-300/month if serving 10hrs/day

**Insight**: Fargate wins when workload is unpredictable or doesn't run 24/7.

---

## Fargate: Requirements & Constraints

### Networking Requirement
Fargate tasks MUST run in a **VPC** (not EC2-Classic, which is outdated anyway).

### Supported Task CPU/Memory Combinations (Not all combinations work!)

```
CPU: 256 (.25 vCPU)  → Memory: 512MB to 2GB
CPU: 512 (.5 vCPU)   → Memory: 1GB to 4GB
CPU: 1024 (1 vCPU)   → Memory: 2GB to 8GB
CPU: 2048 (2 vCPU)   → Memory: 4GB to 16GB
CPU: 4096 (4 vCPU)   → Memory: 8GB to 30GB

(Only these exact vCPU/RAM combinations are valid)
```

**Exam tip**: If a question says "0.5 vCPU with 8GB RAM on Fargate" → INVALID combination!

---

# SECTION 3: ECS — IAM Roles

This is where many engineers get confused. There are TWO different IAM roles, and they do different things.

## The Two-Role Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  EC2 Instance (EC2 Launch Type)                                   │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                                                              │ │
│  │  Role 1: EC2 INSTANCE PROFILE                              │ │
│  │  (Used by ECS Agent)                                        │ │
│  │  ┌──────────────────────────────────────────────────────┐   │ │
│  │  │ Permissions:                                         │   │ │
│  │  │ • ecr:GetAuthorizationToken                         │   │ │
│  │  │ • ecr:BatchGetImage                                 │   │ │
│  │  │ • ecr:GetDownloadUrlForLayer                        │   │ │
│  │  │ • logs:CreateLogGroup                               │   │ │
│  │  │ • logs:CreateLogStream                              │   │ │
│  │  │ • logs:PutLogEvents                                 │   │ │
│  │  │ • ecs:DiscoverPollEndpoint                          │   │ │
│  │  │ • ecs:Poll                                          │   │ │
│  │  │ • ecs:UpdateContainerInstanceStatus                 │   │ │
│  │  │ (all ECS Agent functionality)                       │   │ │
│  │  └──────────────────────────────────────────────────────┘   │ │
│  │                                                              │ │
│  │  ┌─────────────────┐  ┌─────────────────┐                  │ │
│  │  │  Task A         │  │  Task B         │                  │ │
│  │  │ (Container)     │  │ (Container)     │                  │ │
│  │  │                 │  │                 │                  │ │
│  │  │  Role 2:        │  │  Role 2:        │                  │ │
│  │  │  TASK ROLE A    │  │  TASK ROLE B    │                  │ │
│  │  │  ┌────────────┐ │  │  ┌────────────┐ │                  │ │
│  │  │  │ s3:Get*    │ │  │  │ dynamodb:* │ │                  │ │
│  │  │  │ s3:List*   │ │  │  │ sns:Publish│ │                  │ │
│  │  │  └────────────┘ │  │  └────────────┘ │                  │ │
│  │  │                 │  │                 │                  │ │
│  │  │ → Access S3     │  │ → Access DDB    │                  │ │
│  │  └─────────────────┘  └─────────────────┘                  │ │
│  │                                                              │ │
│  └──────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

---

## Role 1: EC2 Instance Profile (EC2 Launch Type ONLY)

**Used by**: ECS Agent (the AWS software managing containers on the EC2 instance)

**When applied**: Attached directly to the EC2 instance

**What it needs**:
- Pull Docker images from ECR
- Send container logs to CloudWatch Logs
- Call ECS API (register instance, report task status)
- Reference secrets from AWS Secrets Manager or Systems Manager Parameter Store
- (Optionally) Pull data from S3

**Example Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:CreateCluster",
        "ecs:DeregisterContainerInstance",
        "ecs:DiscoverPollEndpoint",
        "ecs:Poll",
        "ecs:RegisterContainerInstance",
        "ecs:UpdateContainerInstanceStatus"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Role 2: ECS Task Role

**Used by**: The task (container application) itself

**When applied**: Defined in the task definition, not on the EC2 instance

**What it needs**: Depends on what your application does!
- Task A needs S3 access → give Task Role A S3 permissions
- Task B needs DynamoDB access → give Task Role B DynamoDB permissions
- Task C needs SNS access → give Task Role C SNS permissions

**Example Task Definition with Task Role**:
```json
{
  "family": "my-app",
  "taskRoleArn": "arn:aws:iam::123456789:role/ecsTaskRoleS3",
  "container": {
    "name": "app",
    "image": "my-ecr-repo/my-app:latest",
    "memory": 512
  }
}
```

When this task runs:
1. The application inside the container can call AWS SDK (boto3, AWS SDK for Java, etc.)
2. The SDK automatically assumes the Task Role credentials
3. The app can now call S3 APIs (because Task Role has S3 permissions)

---

## The Security Principle: Least Privilege

**Why separate roles?**

Imagine both the ECS Agent AND the task share the same role with all permissions:
- ECS Agent only needs ECR, CloudWatch, ECS APIs
- But if the role has DynamoDB permissions (for the task)...
- And an attacker compromises the ECS Agent...
- They get DynamoDB access too!

**With separate roles**:
- ECS Agent has ONLY what it needs (ECR, CloudWatch, ECS)
- Task has ONLY what it needs (in this case, S3)
- Breach of ECS Agent ≠ access to DynamoDB
- This is **principle of least privilege**

---

## Troubleshooting: Common IAM Errors

| Error | Cause | Solution |
|-------|-------|----------|
| "Task failed to pull image from ECR" | Missing ECR permissions in Instance Profile | Add `ecr:*` to Instance Profile |
| "Container can't write logs to CloudWatch" | Missing CloudWatch Logs permissions in Instance Profile | Add `logs:*` to Instance Profile |
| "Application in task gets 'Access Denied' to S3" | Missing S3 permissions in Task Role | Add S3 permissions to Task Role |
| "Task can't access Secrets Manager" | Missing SecretsManager permissions in Instance Profile (or Task Role if Fargate) | Add to appropriate role |

---

# SECTION 4: ECS — Load Balancer Integration

When you run multiple ECS tasks, you need a load balancer to distribute traffic across them.

## Supported Load Balancers

| Load Balancer | Recommended? | Why | Fargate Support |
|---------------|--------------|-----|-----------------|
| **ALB** (Application) | ✅ YES | Path-based routing, hostname-based routing, dynamic port mapping | ✅ Yes |
| **NLB** (Network) | ⚠️ ONLY for special cases | Extreme throughput, PrivateLink | ✅ Yes |
| **CLB** (Classic) | ❌ NO | Legacy, limited features | ❌ No |

### Application Load Balancer (ALB) — Recommended

**Why ALB is perfect for ECS**:

1. **Dynamic Port Mapping**: Multiple tasks on same EC2 instance → different port numbers
2. **Path-based routing**: `/api/*` → task-A, `/static/*` → task-B
3. **Hostname-based routing**: `api.example.com` → service-A, `web.example.com` → service-B

### Architecture: ECS + ALB

```
Clients (port 80/443)
    │
    │ HTTP/HTTPS
    ▼
Application Load Balancer (in public subnet)
    │
    │ Target Groups
    │ ├─ Port 32768 → Task 1 (nginx)
    │ ├─ Port 32769 → Task 2 (nginx)
    │ ├─ Port 32770 → Task 3 (nodejs API)
    │ └─ Port 32771 → Task 4 (nodejs API)
    │
    ▼
ECS Cluster (private subnet)
┌──────────────────────────────────┐
│ EC2 Instance 1                   │
│  ┌─────────────────────────────┐ │
│  │ Task 1 (nginx) :32768       │ │
│  │ Task 2 (nginx) :32769       │ │
│  └─────────────────────────────┘ │
│                                  │
│ EC2 Instance 2                   │
│  ┌─────────────────────────────┐ │
│  │ Task 3 (nodejs) :32770      │ │
│  │ Task 4 (nodejs) :32771      │ │
│  └─────────────────────────────┘ │
└──────────────────────────────────┘
```

### Dynamic Port Mapping Deep Dive

**The Problem** (without dynamic port mapping):
```
EC2 Instance has Tasks A and B:
  ┌──────────────┐
  │ Task A: port 8080
  │ Task B: port 8080  ← CONFLICT! Can't have 2 services on port 8080
  └──────────────┘
```

**The Solution** (with dynamic port mapping):
```
EC2 Instance has Tasks A and B:
  ┌──────────────┐
  │ Task A: port 32768 (automatic)
  │ Task B: port 32769 (automatic)
  └──────────────┘
  
ALB knows:
  Client request → ALB:80 → Task A:32768
  Client request → ALB:80 → Task B:32769
```

**How it works**:
1. Define service with ALB target group
2. When task starts, ECS chooses random available host port (ephemeral range: 32768-65535)
3. ECS registers task with ALB target group (ALB:port → Task:host-port mapping)
4. ALB automatically discovers and routes to correct port
5. When task is stopped, ALB automatically deregisters it

---

## Example: Service Definition with ALB

```json
{
  "serviceName": "my-service",
  "launchType": "EC2",
  "desiredCount": 4,
  "loadBalancers": [
    {
      "targetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/my-tg/...",
      "containerName": "my-app",
      "containerPort": 8080
    }
  ]
}
```

What happens:
- ECS creates 4 tasks
- Each task runs container on port 8080 (internal to container)
- ECS picks random host ports (32768, 32769, 32770, 32771)
- ALB routes external traffic to these dynamic ports
- Result: No port conflicts, efficient packing of tasks per EC2 instance

---

# SECTION 5: ECS — Data Volumes (EFS)

## The Storage Problem

Containers are ephemeral (temporary). When a task stops, its data is lost.

```
┌──────────────────────┐
│  ECS Task            │
│  ┌────────────────┐  │
│  │ Container      │  │
│  │ (writes data)  │  │
│  └────────────────┘  │
│                      │
│  Task stops → data lost!
└──────────────────────┘
```

**Solution**: Mount persistent storage into tasks.

---

## EFS (Elastic File System)

### What is EFS?

**EFS** is AWS's managed NFS (Network File System):
- Multiple EC2 instances (or Fargate tasks) can mount the same EFS
- Data persists even after tasks stop
- Multi-AZ by default (if configured across multiple AZs)
- Pay for storage used (not pre-provisioned like EBS)

### EFS vs EBS

| Feature | EBS | EFS |
|---------|-----|-----|
| **Type** | Block storage | Network file system |
| **Access** | Single instance only | Multiple instances/tasks |
| **Persistence** | Yes | Yes |
| **Performance** | Higher IOPS | Lower latency, lower throughput |
| **Price** | Volume-based | Usage-based |
| **Multi-AZ** | No (single AZ) | Yes (across AZs) |
| **Docker mount** | No | Yes |

---

### Architecture: ECS with EFS

```
┌──────────────────────────────────────────────────────────────────────┐
│ AWS Region (us-east-1)                                              │
│                                                                      │
│ ┌────────────────────────────────────────────────────────────────┐  │
│ │ Availability Zone 1                                             │  │
│ │                                                                │  │
│ │ ┌──────────────┐  ┌──────────────┐                            │  │
│ │ │ EC2 Instance │  │ EC2 Instance │                            │  │
│ │ │  Task A      │  │  Task B      │                            │  │
│ │ │  Task C      │  │              │                            │  │
│ │ │ mount /data  │  │ mount /data  │                            │  │
│ │ └──────┬───────┘  └──────┬───────┘                            │  │
│ │        │                 │                                    │  │
│ │        └─────────┬───────┘                                    │  │
│ │                  │                                            │  │
│ └──────────────────┼────────────────────────────────────────────┘  │
│                    │                                                │
│ ┌────────────────────────────────────────────────────────────────┐  │
│ │ Availability Zone 2                                             │  │
│ │                                                                │  │
│ │ ┌──────────────┐                                              │  │
│ │ │  Fargate     │                                              │  │
│ │ │  Task D      │                                              │  │
│ │ │  mount /data │                                              │  │
│ │ └──────┬───────┘                                              │  │
│ │        │                                                      │  │
│ │        └─────────┬────────────────────────────────────────┐   │  │
│ │                  │                                        │   │  │
│ └──────────────────┼────────────────────────────────────────┘   │  │
│                    │                                             │  │
│              ┌─────▼─────┐                                       │  │
│              │ Amazon EFS │  ← Same file system                  │  │
│              │ /data      │     for all tasks!                   │  │
│              │ [shared]   │                                       │  │
│              └────────────┘                                       │  │
└──────────────────────────────────────────────────────────────────────┘
```

### ECS + Fargate + EFS = Fully Serverless Storage

```
Fargate Task → EFS Mount → Persistent Data
(no servers)                (managed by AWS)

= Zero infrastructure management!
```

---

### Practical Example: Database Backup Service

```
Scenario: Every night at 2 AM, back up PostgreSQL database

ECS Service Definition:
  ├─ Launch Type: Fargate
  ├─ Task Definition:
  │   ├─ Container: postgres-backup script
  │   ├─ Task Role: S3 access (for upload)
  │   └─ Volumes: EFS mount at /backup
  │
  └─ Scheduled Rule (CloudWatch Events)
      └─ Every night 2 AM: Start 1 task

Task execution:
  1. Task starts (Fargate)
  2. Mounts EFS at /backup
  3. Backup script dumps PostgreSQL to /backup/backup-2026-03-05.sql
  4. Uploads to S3
  5. Task exits
  6. Data persists in EFS (even though task is gone)
  7. Next month, task runs again, can access previous backups on EFS
```

---

### Important: S3 Cannot Be Mounted

**Common Misconception**:
> "Can I mount S3 as a file system in ECS?"

**Answer**: NO. S3 is object storage (like a warehouse of files), not a file system.

**What you can mount**:
- ✅ EFS (NFS file system)
- ✅ EBS (block storage, but single instance only)
- ✅ FSx for Lustre (high-performance)
- ✅ FSx for NetApp ONTAP (enterprise NAS)

**If you need S3 access from a task**: Use AWS SDK (boto3/Java SDK) to upload/download files. Not a mount.

---

# SECTION 6: ECS Service Auto Scaling

## The Scaling Problem

You have a web service with 3 tasks running. Traffic suddenly spikes (Black Friday). Now you need 30 tasks. Should you:

A) Manually scale (slow, doesn't react to traffic changes)
B) Auto scale (intelligent, reactive)

**Obviously B.**

---

## ECS Service Auto Scaling vs EC2 Auto Scaling

### Two Different Scaling Dimensions

```
┌──────────────────────────────────────────┐
│ ECS Cluster (EC2 Launch Type)            │
│                                          │
│ ┌────────────────────────────────────┐   │
│ │  Scaling Dimension 1: TASKS        │   │
│ │  (ECS Service Auto Scaling)        │   │
│ │                                    │   │
│ │  Current: 3 tasks                  │   │
│ │  Target: Maintain avg CPU at 70%   │   │
│ │  Action: Add task 4, 5, 6...       │   │
│ │                                    │   │
│ └────────────────────────────────────┘   │
│                                          │
│ ┌────────────────────────────────────┐   │
│ │  Scaling Dimension 2: EC2 INSTANCES│   │
│ │  (EC2 Auto Scaling Group)          │   │
│ │                                    │   │
│ │  Current: 1 EC2 instance           │   │
│ │  Problem: No capacity for new task!│   │
│ │  Action: Add EC2 instance 2, 3...  │   │
│ │                                    │   │
│ └────────────────────────────────────┘   │
└──────────────────────────────────────────┘
```

---

## ECS Service Auto Scaling (Task Scaling)

### What It Does

Automatically increases/decreases the number of tasks in an ECS service based on metrics.

### Three Scaling Policies

#### 1. Target Tracking

**Simplest to set up**. AWS maintains a target value for a metric.

**Example**:
```
Policy: Target Tracking CPU Utilization
Target Metric: ECS Service Average CPU = 70%

Current State:
  ├─ 3 tasks running
  ├─ Average CPU: 85%
  └─ ABOVE target → Scale up!

Action:
  ├─ Add tasks
  ├─ Now: 5 tasks
  ├─ Average CPU: 70% (target reached!)
  └─ Stop scaling

Later, traffic drops:
  ├─ Average CPU: 30%
  ├─ BELOW target → Scale down
  └─ Remove tasks to reach 70% again
```

#### 2. Step Scaling

**More granular control**. Based on CloudWatch Alarms with thresholds.

```
CloudWatch Metric: ECS Service CPU Utilization

┌─ 0-30%: Do nothing
├─ 30-60%: Do nothing
├─ 60-80%: Add 2 tasks
├─ 80-100%: Add 4 tasks
└─ >100%: Add 8 tasks (emergency!)

Alarm triggers when CPU > 80%
→ Add 4 tasks
```

#### 3. Scheduled Scaling

**For predictable traffic patterns**. Scale based on time/date.

```
Rule 1: Monday-Friday 9 AM
  └─ Scale up to 20 tasks (office hours)

Rule 2: Monday-Friday 6 PM
  └─ Scale down to 5 tasks (office closes)

Rule 3: Black Friday
  └─ Scale up to 100 tasks

Rule 4: Christmas
  └─ Scale to 150 tasks
```

---

## Metrics for Scaling

### Available Metrics

| Metric | What it measures | Use case |
|--------|------------------|----------|
| **ECS Service Average CPU Utilization** | CPU usage across all tasks | CPU-bound workloads |
| **ECS Service Average Memory Utilization** | RAM usage across all tasks | Memory-constrained workloads |
| **ALB Request Count Per Target** | HTTP requests per task | API requests volume |
| **DynamoDB Consumed Read/Write Capacity** | Database utilization | Database-backed services |

---

## The Capacity Provider: Smart EC2 Scaling

### Problem with Manual EC2 ASG Scaling

```
Scenario: EC2 Launch Type with fixed ASG

Setup:
  ├─ ASG: min=1, max=5 EC2 instances
  ├─ ECS Service: desired=3 tasks
  └─ Simple CPU scaling rule on ASG

Traffic spikes:
  1. ECS Service Auto Scaling adds tasks (3→5)
  2. But EC2 instances still at 1
  3. No capacity! Tasks can't schedule
  4. ECS throws "insufficient capacity" error
  5. Now manually add EC2 instances (slow!)
```

### Solution: Capacity Providers

**Capacity Provider** = Automatically scales EC2 instances when ECS needs capacity.

```
ECS Service Auto Scaling → needs more capacity
        │
        ▼
ECS Capacity Provider checks:
  "Do we have room for new tasks?"
  
  If NO:
    └─ Auto Scaling Group launches new EC2 instance
       └─ Instance registers with ECS cluster
       └─ Now capacity available!
       └─ New tasks schedule immediately

  If YES:
    └─ Just schedule the task (no new EC2 needed)
```

### Architecture: With Capacity Provider

```
User Traffic Increases
        │
        ▼
CloudWatch detects high CPU
        │
        ▼
ECS Service Auto Scaling triggered
        │ "Need 6 tasks, currently have 3"
        │
        ▼
ECS Capacity Provider triggered
        │ "Do we have EC2 capacity?"
        │
        ├─ NO → Launch new EC2 instance
        │       (Auto Scaling Group)
        │       ↓
        │       Instance registers → ready for tasks
        │
        └─ YES → Skip EC2 launch, just add tasks
                 ↓
                 3 new tasks schedule on available EC2
```

---

## Scaling for Different Launch Types

### EC2 Launch Type: Need Both!

```
Scale scenario: Traffic increases, need 10 tasks (from 3)

Step 1: ECS Service Auto Scaling
  └─ Increases desired task count 3 → 10

Step 2: Check EC2 capacity
  └─ Only 1 EC2 instance → not enough capacity!

Step 3: Capacity Provider OR Manual EC2 Scaling
  └─ Launch more EC2 instances
  └─ Now 5 EC2 instances available

Step 4: Tasks schedule on new EC2 instances
```

**With Capacity Provider**: Step 3 is automatic!

### Fargate Launch Type: Only Task Scaling!

```
Scale scenario: Traffic increases, need 10 tasks (from 3)

Step 1: ECS Service Auto Scaling
  └─ Increases desired task count 3 → 10

Step 2: Tasks launch on Fargate
  └─ AWS automatically provisions compute
  └─ No EC2 instances to worry about!

That's it! ✓
```

**Fargate is simpler** because AWS handles all the infrastructure.

---

# SECTION 7: ECS Integration Patterns

## Pattern 1: EventBridge → ECS Task (Event-Driven)

### Use Case: Process Event When Something Happens

```
Example: S3 object uploaded → process it immediately

Client uploads image.jpg
  │ to S3 Bucket
  ▼
S3 sends event:
{
  "source": "aws.s3",
  "detail-type": "Object Created",
  "detail": {
    "bucket": "my-images",
    "key": "image.jpg"
  }
}
  │
  ▼
EventBridge Rule matches:
  "If source == aws.s3, run ECS task"
  │
  ▼
ECS Fargate Task launches
  ├─ Image: image-processor:latest
  ├─ Task Role: S3 read + DynamoDB write
  │
  ▼
Container execution:
  1. Download image from S3
  2. Process (resize, compress)
  3. Save metadata to DynamoDB
  4. Task exits
  5. CloudWatch billing: ~0.0003 USD (seconds * vCPU-hour rate)
```

**Why this pattern?**
- Fully event-driven (no polling)
- Zero cost when no events (Fargate + EventBridge serverless)
- Automatic scaling (each event = new task)
- Decoupled (S3 doesn't know about processor)

---

## Pattern 2: EventBridge Schedule → ECS Task (Cron Job)

### Use Case: Run Task on a Schedule

```
Scenario: Batch process all uploaded images every hour

CloudWatch Event Rule:
  Rule: "cron(0 * * * ? *)"  [Every hour]
  Target: "ECS Fargate Task"
  │
  ▼
Every hour at :00:
  1. EventBridge triggers
  2. ECS Fargate task launches
  3. Task queries DynamoDB for unprocessed images
  4. Processes all of them in batch
  5. Task completes and exits

Next hour:
  └─ Repeat (only processes new images since last run)
```

**Configuration**:
```json
{
  "ScheduleExpression": "cron(0 * * * ? *)",
  "Targets": [
    {
      "RoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
      "EcsParameters": {
        "LaunchType": "FARGATE",
        "TaskDefinitionArn": "arn:aws:ecs:...:task-definition/batch-processor:1",
        "Subnets": ["subnet-12345"],
        "SecurityGroups": ["sg-12345"],
        "DesiredCount": 1
      }
    }
  ]
}
```

---

## Pattern 3: ECS + SQS (Polling from Queue)

### Use Case: Process Messages from Queue

```
Publishers send messages to SQS
  │
  ▼
SQS Queue (buffer of messages)
  │
  ▼
ECS Tasks (workers) poll the queue
  ├─ Task 1: GET message, process, DELETE
  ├─ Task 2: GET message, process, DELETE
  └─ Task 3: GET message, process, DELETE

  (All tasks polling same queue)

  If queue grows:
    └─ ECS Auto Scaling adds more tasks
  
  If queue empty:
    └─ ECS Auto Scaling removes tasks
```

**Auto Scaling based on queue depth**:

```
CloudWatch Custom Metric: SQS ApproximateNumberOfMessagesVisible

ECS Service Auto Scaling Policy:
  "Scale based on SQS queue depth"
  
  Queue has 100 messages:
    └─ Each task can process 10 msgs/min
    └─ Need 100/10 = 10 tasks minimum
    └─ Scale up to 10 tasks
  
  Queue has 0 messages:
    └─ Scale down to 1 task (minimum)
```

---

## Pattern 4: ECS Task Failure → Alert

### Use Case: Get Notified When Task Dies Unexpectedly

```
ECS Task running
  ├─ Essential container crashes (e.g., out of memory)
  ├─ Task status changes to STOPPED
  │
  ▼
EventBridge event generated:
{
  "source": "aws.ecs",
  "detail-type": "ECS Task State Change",
  "detail": {
    "lastStatus": "STOPPED",
    "stoppedReason": "Essential container in task exited",
    "taskArn": "arn:aws:ecs:...:task/my-service/abc123",
    "createdAt": "2026-03-05T10:00:00Z",
    "stoppedAt": "2026-03-05T10:05:00Z"
  }
}
  │
  ▼
EventBridge Rule matches:
  "If detail.lastStatus == STOPPED, publish to SNS"
  │
  ▼
SNS Topic publishes message
  │
  ▼
SNS Subscribers:
  ├─ Email: ops@company.com
  ├─ Slack webhook
  └─ PagerDuty (on-call alert)

Result: Engineer is notified immediately!
```

---

# SECTION 8: Amazon ECR — Elastic Container Registry

## What is ECR?

**Amazon Elastic Container Registry (ECR)** is AWS's managed Docker image registry.

Think of it as:
- **Private DockerHub**: Your company's Docker images (access controlled)
- **Or public**: Amazon ECR Public Gallery (https://gallery.ecr.aws)
- **S3-backed**: Images stored in S3 behind the scenes
- **IAM-integrated**: Access controlled by IAM roles/policies

---

## Architecture: ECR

```
Your Development Environment
  │
  ├─ Develop code
  ├─ Run tests
  ├─ Build Docker image
  │ $ docker build -t my-app:v1.0 .
  │
  ▼
ECR Repository (Private)
┌─────────────────────────────────┐
│ my-company/my-app               │
│ ├─ v1.0    [pushed]             │
│ ├─ v1.1    [pushed]             │
│ ├─ v1.2    [pushed, latest]     │
│ ├─ v2.0    [pushed]             │
│ └─ build-123  [untagged]        │
│                                 │
│ (Backed by S3)                  │
│ (IAM-protected)                 │
│ (Vulnerability scanning)        │
│ (Lifecycle policies)            │
└─────────────────────────────────┘
  │
  │ pull (needs ECR permission)
  │
  ▼
ECS Cluster
┌─────────────────────────────────┐
│ EC2 Instance (ECS Agent)        │
│ ┌─────────────────────────────┐ │
│ │ Task A                      │ │
│ │ runs my-app:v1.2            │ │
│ └─────────────────────────────┘ │
└─────────────────────────────────┘
```

---

## ECR Features

### 1. Private Repositories

Access controlled by **IAM policies**. Only users/roles with `ecr:*` permissions can pull.

**Example ECR URI**:
```
123456789.dkr.ecr.us-east-1.amazonaws.com/my-company/my-app:v1.0
│         │
│         └─ AWS Account ID
└─ AWS ECR domain
```

---

### 2. Public Repositories

Images publicly accessible at https://gallery.ecr.aws

Example public images:
- `public.ecr.aws/amazonlinux/amazonlinux:latest`
- `public.ecr.aws/amazon-ecs-sample/amazon-ecs-sample:latest`

---

### 3. Vulnerability Scanning

ECR can automatically scan images for known vulnerabilities.

```
Push image to ECR
  │
  ▼
Scan triggered:
  ├─ Image analysis
  ├─ Check for known CVEs
  ├─ List vulnerable packages
  │
  ▼
Scan results:
  ├─ CRITICAL: 2 vulnerabilities (patch immediately!)
  ├─ HIGH: 5 vulnerabilities
  ├─ MEDIUM: 12 vulnerabilities
  ├─ LOW: 8 vulnerabilities
  └─ INFORMATIONAL: 3
```

---

### 4. Lifecycle Policies

Automatically delete old images to save S3 costs.

**Example Policy**:
```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Delete untagged images older than 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Keep only last 10 tagged versions",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

Result: Old versions automatically deleted, storage costs reduced.

---

## ECR in Action: Pull Error Troubleshooting

### Scenario: "Unauthorized: authentication required"

```
ECS Task tries to pull image:
  $ docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
  
Error: Unauthorized: authentication required

Root cause check:
  1. EC2 Instance Profile has ecr:GetAuthorizationToken? → NO
  2. Add to Instance Profile:
     {
       "Effect": "Allow",
       "Action": [
         "ecr:GetAuthorizationToken",
         "ecr:BatchGetImage",
         "ecr:GetDownloadUrlForLayer"
       ],
       "Resource": "*"
     }
  3. Retry pull → SUCCESS!
```

---

# SECTION 9: Amazon EKS — Elastic Kubernetes Service

## What is Kubernetes?

**Kubernetes** is the open-source industry standard for container orchestration.

- Created by Google
- Cloud-agnostic (runs on AWS, Azure, GCP, on-premises)
- Powerful but complex
- Community-driven, huge ecosystem

**Mental model**: 
- ECS = AWS proprietary, simpler, AWS-specific
- Kubernetes = industry standard, more powerful, portable

---

## Why EKS vs ECS?

| Scenario | Use ECS | Use EKS |
|----------|---------|---------|
| Starting with containers on AWS | ✅ Yes | ❌ No |
| Already using Kubernetes on-prem | ❌ No | ✅ Yes |
| Running K8s in multiple clouds | ❌ No | ✅ Yes |
| Want maximum simplicity | ✅ Yes | ❌ No |
| Need advanced scheduling (operators) | ❌ No | ✅ Yes |
| Team knows kubectl | ❌ No | ✅ Yes |

---

## Kubernetes on AWS: EKS Components

### Control Plane (Managed by AWS)

```
AWS Managed Kubernetes Control Plane
┌──────────────────────────────────────────┐
│ API Server (kubectl communicates)        │
├──────────────────────────────────────────┤
│ Scheduler (decides pod placement)        │
├──────────────────────────────────────────┤
│ Controller Manager (health, scaling)     │
├──────────────────────────────────────────┤
│ etcd (state database)                    │
├──────────────────────────────────────────┤
│ (AWS manages all of the above)           │
│ (Multi-AZ, HA by default)                │
└──────────────────────────────────────────┘
```

**You never see or manage the control plane.**

### Data Plane (Nodes Running Your Pods)

Your EC2 instances (or Fargate) running your containers.

---

## EKS Node Types

### Type 1: Managed Node Groups (RECOMMENDED)

**AWS manages the EC2 instances for you.**

```
EKS Cluster
┌──────────────────────────────────────────┐
│ Managed Node Group                       │
│ └─ Auto Scaling Group (AWS manages)      │
│    ├─ EC2 Instance 1 (node-1)           │
│    ├─ EC2 Instance 2 (node-2)           │
│    ├─ EC2 Instance 3 (node-3)           │
│    └─ [Scales automatically]            │
│                                          │
│ Pods schedule on these nodes            │
└──────────────────────────────────────────┘
```

**Advantages**:
- AWS manages node lifecycle (patching, OS updates)
- Automatic scaling via Auto Scaling Group
- Supports on-demand and Spot instances
- Simplest approach

**Launch types**:
- On-Demand EC2
- Spot Instances (cheap, can be interrupted)

---

### Type 2: Self-Managed Nodes

**You create and manage EC2 instances yourself.**

```
EKS Cluster
┌──────────────────────────────────────────┐
│ Custom Node Setup (You manage)           │
│                                          │
│ ├─ EC2 Instance 1                       │
│ │  └─ kubelet (join cluster)            │
│ │  └─ EBS storage                       │
│ │  └─ (You patch/update)                │
│ │                                        │
│ ├─ EC2 Instance 2                       │
│ │  └─ (Same)                            │
│ │                                        │
│ └─ Your ASG (you manage scaling)        │
│                                          │
│ Pods schedule on these nodes            │
└──────────────────────────────────────────┘
```

**Use when**:
- You need custom AMI (pre-installed software)
- You want maximum control
- Managed Node Groups don't support your workload

---

### Type 3: AWS Fargate (Serverless Nodes)

**AWS manages everything. No nodes visible to you.**

```
EKS Cluster + Fargate
┌──────────────────────────────────────────┐
│ Fargate (serverless, AWS-managed)        │
│                                          │
│ You just define Pods:                    │
│ ├─ Pod A (0.5 vCPU, 1GB RAM)            │
│ ├─ Pod B (1 vCPU, 2GB RAM)              │
│ └─ Pod C (2 vCPU, 4GB RAM)              │
│                                          │
│ AWS provisions compute automatically    │
│ (nodes are invisible to you)            │
│                                          │
│ Billing: vCPU-hours + GB-hours used     │
└──────────────────────────────────────────┘
```

**Best for**:
- Simplicity (no node management)
- Bursty workloads (scale to zero)
- Serverless mindset

**Limitations**:
- Higher cost than Spot instances
- Longer pod startup time
- Can't use host-based storage (EBS volumes)

---

## EKS Architecture Diagram

```
AWS Cloud (VPC: 10.0.0.0/16)
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Availability Zone 1         AZ 2               AZ 3               │
│  ┌─────────────────────┐    ┌──────────────┐   ┌──────────────┐   │
│  │ Public Subnet 1     │    │ Public SN 2  │   │ Public SN 3  │   │
│  │ ┌─────────────────┐ │    │ ┌──────────┐ │   │ ┌──────────┐ │   │
│  │ │ NAT Gateway     │ │    │ │ NAT Gtwy │ │   │ │ NAT Gtwy │ │   │
│  │ └────────────┬────┘ │    │ └────────┬─┘ │   │ └────────┬─┘ │   │
│  │              │      │    │          │   │   │          │   │   │
│  │ ┌─────────────────┐ │    │ ┌──────────┐ │   │ ┌──────────┐ │   │
│  │ │ ELB (Ingress)   │ │    │ │ ELB      │ │   │ │ ELB      │ │   │
│  │ └────────────┬────┘ │    │ └────────┬─┘ │   │ └────────┬─┘ │   │
│  │              │      │    │          │   │   │          │   │   │
│  └──────────────┼──────┘    └─────────┼───┘   └──────────┼───┘   │
│                 │                     │                  │         │
│  ┌──────────────┼────────────────────┼──────────────────┼──────┐  │
│  │              │                    │                  │      │  │
│  │  ┌──────────▼─────────┐  ┌────────▼─────┐  ┌────────▼────┐ │  │
│  │  │ Private Subnet 1   │  │ Private SN 2 │  │ Private SN 3│ │  │
│  │  │                    │  │              │  │             │ │  │
│  │  │ ┌────────────────┐ │  │ ┌──────────┐ │  │ ┌─────────┐ │ │  │
│  │  │ │ EKS Node 1     │ │  │ │ Node 2   │ │  │ │ Node 3  │ │ │  │
│  │  │ │ ┌────────────┐ │ │  │ │ ┌──────┐ │ │  │ │ ┌─────┐ │ │ │  │
│  │  │ │ │Pod (nginx) │ │ │  │ │ │Pod   │ │ │  │ │ │Pod  │ │ │ │  │
│  │  │ │ └────────────┘ │ │  │ │ │(api) │ │ │  │ │ │(db) │ │ │ │  │
│  │  │ │ ┌────────────┐ │ │  │ │ │      │ │ │  │ │       │ │ │  │
│  │  │ │ │Pod (api)   │ │ │  │ │ │      │ │ │  │ │       │ │ │  │
│  │  │ │ └────────────┘ │ │  │ │ └──────┘ │ │  │ │       │ │ │  │
│  │  │ └────────────────┘ │  │ └──────────┘ │  │ │       │ │ │  │
│  │  │ [kubelet]          │  │ [kubelet]    │  │ │[kube] │ │ │  │
│  │  └────────────────────┘  │              │  │ │       │ │ │  │
│  │  (Managed ASG)           └──────────────┘  │ └─────┘ │ │  │
│  │                          (Managed ASG)    │ (ASG)    │ │  │
│  └──────────────────────────────────────────┼─────────┘  │  │
│                                             │            │  │
│                                      Kubernetes etcd    │  │
│                                      (cluster state)    │  │
└─────────────────────────────────────────────────────────────┘

Outside VPC:
┌──────────────────────────────┐
│ AWS Managed Control Plane    │
│ (API Server, Scheduler, etc) │
│ (You don't see this)         │
└──────────────────────────────┘
```

---

## EKS Storage Options

### EKS uses StorageClass + CSI Drivers

When you need persistent storage for Pods, EKS uses **Container Storage Interface (CSI)** drivers.

**Available options**:

| Storage | Type | Use Case | Multi-AZ |
|---------|------|----------|----------|
| **Amazon EBS** | Block (single pod) | Database volumes, OS disks | No (single AZ) |
| **Amazon EFS** | File system (shared) | Shared data, read-only mounts | Yes |
| **FSx for Lustre** | File system (HPC) | Machine learning, scientific computing | No |
| **FSx for NetApp ONTAP** | NAS (enterprise) | Enterprise data management | Yes |

**Example: EKS Pod with EBS Volume**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    awsElasticBlockStore:
      volumeId: vol-12345
      fsType: ext4
```

**Example: EKS Pod with EFS Volume** (shared across pods)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: efs-storage
      mountPath: /data
  volumes:
  - name: efs-storage
    nfs:
      server: fs-12345.efs.us-east-1.amazonaws.com
      path: "/"
```

---

# SECTION 10: AWS App Runner

## What is App Runner?

**AWS App Runner** is the ultimate managed platform for deploying web applications and APIs.

**Concept**: "Give us your code or Docker image. We handle everything else."

---

## App Runner: Responsibility Model

```
┌──────────────────────────────────────────────────────────────┐
│                 APP RUNNER (Fully Managed)                   │
│                                                              │
│  ✓ Source code → Build → Containerize → Deploy              │
│  ✓ Auto-scaling                                             │
│  ✓ Load balancing                                           │
│  ✓ Health checks                                            │
│  ✓ HTTPS/TLS                                                │
│  ✓ VPC connectivity (databases, caches)                    │
│  ✓ Environment variables, secrets management                │
│  ✓ Monitoring & logging                                     │
│                                                              │
│  = ZERO infrastructure management                           │
└──────────────────────────────────────────────────────────────┘

Your responsibility: Write code. That's it!
```

---

## App Runner Deployment Options

### Option 1: Source Code

```
GitHub/GitLab Repository
    │
    ├─ Contains: Source code + Dockerfile (or buildpack auto-detect)
    │
    ▼
App Runner
    ├─ Pulls code from repo
    ├─ Builds image (using Dockerfile)
    ├─ Pushes to ECR
    ├─ Deploys & runs
    │
    ▼
Your app accessible at:
https://xxxxxxxxxx.awsapprunner.com
```

### Option 2: Container Image

```
ECR Repository (or Docker Hub)
    │
    ├─ Already built image
    │
    ▼
App Runner
    ├─ Pulls image from ECR
    ├─ Deploys & runs
    │
    ▼
Your app accessible at:
https://xxxxxxxxxx.awsapprunner.com
```

---

## App Runner Features

### Auto Scaling

```
Requests come in
    │
    ▼
App Runner monitors CPU/memory
    │
    ├─ LOW usage → Scale down (pay less)
    ├─ HIGH usage → Scale up (more containers)
    │
    ▼
Automatically maintains performance
```

### High Availability

```
App Runner service deployed across AZs:

AZ 1: Container running
AZ 2: Container running
AZ 3: Container running

If AZ 1 fails → traffic auto-routes to AZ 2,3
Automatic failover (no downtime)
```

### Load Balancing

Built-in Application Load Balancer. No need to create one separately!

### VPC Connectivity

Connect to private databases, caches in VPC:

```
App Runner (public)
    │ VPC Connector
    ▼
VPC (private)
    ├─ RDS Database
    ├─ ElastiCache
    └─ Other private resources

App can access these!
```

---

## App Runner vs Fargate vs ECS

| Feature | App Runner | Fargate | ECS EC2 |
|---------|-----------|--------|---------|
| **Complexity** | Minimal (best) | Low | Medium |
| **Setup time** | 5 minutes | 15 minutes | 30 minutes |
| **Build pipeline** | Built-in (auto-build) | Manual | Manual |
| **Load balancer** | Included | Create separate | Create separate |
| **Auto-scaling** | Built-in | Manual policy | Manual policy |
| **Cost for 10 req/sec** | $$ | $$$ | $$$$ |
| **For startups** | Perfect! | Good | Less ideal |
| **For enterprises** | Simple apps | Better control | Most control |

---

## App Runner Use Cases

```
✓ Microservices
✓ Web APIs
✓ Backend services
✓ Rapid prototypes
✓ Startups (no DevOps team)
✓ Internal tools
✓ CI/CD pipelines

✗ GPU workloads (not supported)
✗ Custom scheduling (need EKS)
✗ Extreme high-throughput (need NLB on EC2)
```

---

# SECTION 11: AWS App2Container (A2C)

## What is App2Container?

**App2Container (A2C)** is a CLI tool that **automatically converts** Java and .NET applications into Docker containers.

**Purpose**: Lift-and-shift containerization without code changes.

---

## A2C Workflow

```
Existing Java or .NET App (on bare metal or VM)
    │
    ├─ Can be on-premises
    ├─ Can be on EC2
    ├─ Can be self-hosted anywhere
    │
    ▼
Run A2C analysis:
    $ app2container analyze [process-id]
    
    A2C discovers:
    ├─ Runtime (Java 11, .NET 6, etc.)
    ├─ Dependencies
    ├─ Environment variables
    ├─ Config files
    └─ Data volumes
    │
    ▼
Generate artifacts:
    ├─ Dockerfile (optimized for your app)
    ├─ CloudFormation template (AWS infrastructure)
    ├─ Docker compose file (local testing)
    └─ ECR push commands
    │
    ▼
Build & push image:
    $ app2container containerize [process-id]
    
    → Builds image locally
    → Pushes to ECR
    │
    ▼
Deploy to AWS:
    Choose one:
    ├─ Amazon ECS (Fargate or EC2)
    ├─ Amazon EKS
    └─ AWS App Runner
    │
    ▼
Your app now runs containerized on AWS!
```

---

## A2C Example: Java Spring Boot App

```
Current state:
  Java Spring Boot app running on premise
  ├─ Java 11
  ├─ MySQL database dependency
  ├─ 512MB heap memory
  ├─ Port 8080
  └─ Logging to files

A2C analysis:
  $ app2container analyze 12345
  
  Generates:
  ├─ Dockerfile
  │   FROM openjdk:11
  │   RUN apt-get install mysql-client
  │   COPY springboot.jar /opt/
  │   ENV JAVA_OPTS="-Xmx512m"
  │   EXPOSE 8080
  │
  ├─ CloudFormation template
  │   └─ ECS service + ALB + security groups + IAM roles
  │
  └─ Environment mapping
      └─ Existing env variables → Docker environment variables

Deploy:
  $ app2container containerize 12345
  $ aws cloudformation create-stack --template-body file://template.yaml
  
  Result: App running on ECS Fargate within minutes!
```

---

## A2C Benefits

```
✓ Zero code changes (existing app runs as-is)
✓ Fully automated analysis & container generation
✓ Pre-built CloudFormation templates (infrastructure-as-code)
✓ Supports Java & .NET
✓ Can deploy to ECS, EKS, or App Runner
✓ Reduces migration time from weeks to days
✓ No application expertise required (template handles it)
```

---

## A2C Limitations

```
✗ Only Java and .NET (not Python, Node, etc.)
✗ Works best with single-process apps
✗ May need manual adjustment for complex architectures
✗ Requires the app to run on a VM/EC2 first
```

---

# CONTAINER DECISION TREE

When deciding which AWS container service to use:

```
┌─ Do you have existing code to containerize?
│  │
│  ├─ YES (Docker image exists)
│  │  │
│  │  ├─ Where to store it?
│  │  │  └─ Amazon ECR ✓
│  │  │
│  │  ├─ Where to run it?
│  │  │  │
│  │  │  ├─ Want serverless? → AWS Fargate
│  │  │  ├─ Want Kubernetes? → Amazon EKS
│  │  │  ├─ Want AWS-native? → Amazon ECS
│  │  │  └─ Want simplicity? → AWS App Runner
│  │
│  └─ NO (have Java/. NET source code)
│     │
│     └─ Migrate with AWS App2Container (A2C) ✓
│
├─ Are you using Kubernetes?
│  │
│  ├─ YES → Amazon EKS (Managed Kubernetes)
│  │  │
│  │  └─ Node options:
│  │     ├─ Managed Node Groups (AWS handles EC2)
│  │     ├─ Self-Managed (you handle EC2)
│  │     └─ AWS Fargate (serverless)
│  │
│  └─ NO → Skip EKS
│
├─ Do you want full infrastructure control?
│  │
│  ├─ YES → ECS EC2 Launch Type
│  │  └─ You manage EC2 instances
│  │
│  └─ NO → ECS Fargate or App Runner
│
├─ Do you want extreme simplicity (code → URL in 5 min)?
│  │
│  ├─ YES → AWS App Runner
│  │  └─ Give code/image, we handle everything
│  │
│  └─ NO → ECS or EKS
│
├─ Workload is unpredictable/bursty?
│  │
│  ├─ YES → Fargate or App Runner (pay for usage)
│  │
│  └─ NO → ECS EC2 (use Reserved Instances)
│
└─ Final Decision:
   ├─ Startup/simple app? → App Runner
   ├─ Kubernetes expertise? → EKS
   ├─ AWS-first, varied needs? → ECS Fargate
   └─ Complex, cost-optimized? → ECS EC2 + ASG
```

---

# POINTS TO REMEMBER (EXAM FOCUS)

## Critical Concepts

1. **ECS EC2 Launch Type**
   - YOU provision and maintain EC2 instances
   - Each instance must run ECS Agent
   - Instance Profile needed for ECS Agent to work
   - Task Role for task to access AWS services
   - Exam question: "Our tasks can't access S3" → check Task Role (not Instance Profile!)

2. **ECS Fargate Launch Type**
   - Serverless! No EC2 instances to manage
   - AWS handles all infrastructure
   - Define task CPU/RAM (must be valid combination)
   - Scale by adding tasks, not EC2 instances
   - Higher cost per task but no idle capacity waste

3. **EC2 Instance Profile vs Task Role**
   - **Instance Profile**: Used by ECS Agent on EC2, for ECR/CloudWatch/ECS APIs
   - **Task Role**: Used by application in task, for S3/DynamoDB/etc.
   - **Different tasks can have different Task Roles** (principle of least privilege)
   - **Exam tip**: "Container can't access DynamoDB" → check Task Role, not Instance Profile

4. **ALB with ECS**
   - Recommended for ECS (supports dynamic port mapping)
   - Multiple tasks per EC2 instance possible
   - ECS automatically manages port registration
   - No manual port configuration needed

5. **EFS with ECS**
   - Multi-AZ shared file system
   - Works with both EC2 and Fargate
   - Tasks can share data across AZs
   - S3 cannot be mounted (it's object storage, not file system)

6. **ECS Service Auto Scaling**
   - Scales **TASKS** (not EC2 instances!)
   - Three policies: Target Tracking, Step Scaling, Scheduled
   - Metrics: CPU, Memory, ALB request count
   - EC2 Launch Type needs BOTH task scaling + EC2 scaling (use Capacity Provider!)

7. **EventBridge → ECS Patterns**
   - Event-driven: S3 event → trigger ECS task
   - Scheduled: cron rule → trigger ECS task on schedule
   - Cost-effective with Fargate (pay per execution)

8. **ECR (Elastic Container Registry)**
   - Private Docker registry (IAM-protected)
   - Images stored in S3 (you only see registry interface)
   - Vulnerability scanning available
   - Lifecycle policies for cost optimization

9. **EKS (Elastic Kubernetes Service)**
   - Managed Kubernetes for AWS
   - Use when already familiar with K8s or need cloud portability
   - Node types: Managed Groups, Self-Managed, or Fargate
   - API Server/Scheduler managed by AWS (control plane)

10. **App Runner**
    - Simplest service for deploying apps
    - Takes source code or container image
    - Handles build, deploy, scaling, load balancing
    - Built-in HTTPS, no infrastructure management needed

11. **App2Container (A2C)**
    - Migrates Java/.NET apps to Docker without code changes
    - Generates Dockerfile + CloudFormation templates
    - Lift-and-shift containerization

12. **Capacity Provider (EC2 Launch Type)**
    - Automatically scales EC2 instances when ECS needs capacity
    - Better than manual ASG scaling
    - Works with ECS Service Auto Scaling for complete solution

---

# INTERVIEW TIPS & SCENARIOS

## Scenario 1: "We're migrating a containerized microservices app to AWS"

**Your thought process**:
```
Questions to ask:
1. "Are you using Kubernetes?" 
   → YES: Recommend EKS
   → NO: Recommend ECS

2. "What's your comfort level with infrastructure?"
   → Experts: ECS EC2 (more control, cost optimization)
   → Beginners: ECS Fargate or App Runner (simpler)

3. "Is the workload predictable?"
   → Predictable 24/7: ECS EC2 + Reserved Instances (cost savings)
   → Bursty/variable: Fargate (no idle costs)

4. "Do you have existing DevOps/K8s expertise?"
   → YES: EKS
   → NO: ECS Fargate or App Runner
```

**Suggested answer**:
> "I'd recommend ECS Fargate for your microservices. It's fully managed, scales automatically, and you pay only for the resources you use. If you later need more control, you can migrate to ECS EC2. And if you want extreme simplicity for individual services, App Runner is excellent."

---

## Scenario 2: "Our container can't access S3 from ECS tasks"

**Root cause analysis**:
```
Check this order:
1. Is the Task Role attached to the task definition? → NO? Add it!
2. Does the Task Role have S3 permissions? → NO? Add S3 policy!
3. Is the container using correct AWS SDK call? → YES (usually)
4. Are credentials being passed correctly? → Task assumes role automatically

WRONG: Add S3 permissions to EC2 Instance Profile
CORRECT: Add S3 permissions to ECS Task Role
```

**Answer**:
> "The issue is that the S3 permissions are likely on the EC2 Instance Profile (used by ECS Agent), but the task itself needs permissions via its Task Role. Define a Task Role in the task definition with S3 permissions (s3:GetObject, s3:PutObject, etc.), and the container will automatically assume that role's credentials."

---

## Scenario 3: "We need to scale web services based on traffic"

**Solution**:
```
Setup:
1. Define ECS Service with desired task count (e.g., 3)
2. Set up Application Load Balancer (ALB)
3. Create CloudWatch metric for ALB Request Count Per Target
4. Create ECS Service Auto Scaling policy:
   → Target: 100 requests per target
   → Min tasks: 1
   → Max tasks: 10

When traffic increases:
→ ALB detects request spike
→ Metric exceeds target
→ ECS Auto Scaling adds tasks (3 → 5 → 7)
→ Traffic distributed across more containers

When traffic decreases:
→ ECS Auto Scaling removes tasks (back to baseline)
```

---

## Scenario 4: "We want to run batch jobs on a schedule (like cron)"

**Solution**:
```
Setup:
1. Create Docker image for job (runs once, exits)
2. Push to ECR
3. Create ECS Task Definition
4. Create CloudWatch Event Rule:
   Cron: "0 2 * * ? *" (2 AM every day)
   Action: Run ECS Fargate Task
   Count: 1
   Task Definition: batch-job:1

Execution:
→ 2 AM: CloudWatch Event triggers
→ Fargate Task starts
→ Job runs to completion
→ Task exits
→ Next day: Repeat

Cost: Very low (task runs once per day for a few seconds/minutes)
```

---

## Scenario 5: "Process files from S3 in real-time"

**Solution**:
```
Setup:
1. S3 bucket uploads trigger → EventBridge event
2. EventBridge rule:
   Source: aws.s3
   Detail-type: "Object Created"
   Action: Run ECS Fargate Task

3. ECS Task Definition:
   ├─ Image: file-processor:latest
   ├─ Task Role: S3 read + DynamoDB write permissions
   └─ Environment: Pass S3 bucket name

Execution:
→ User uploads file.txt to S3
→ S3 emits event: {bucket: my-bucket, key: file.txt}
→ EventBridge matches rule
→ ECS Fargate task starts
→ Container downloads file from S3
→ Processes it (resize, convert, validate, etc.)
→ Writes results to DynamoDB
→ Task exits
→ Billing: seconds × (vCPU-hour rate)

This is truly event-driven and cost-effective!
```

---

## Scenario 6: "Container exits unexpectedly. How do we monitor this?"

**Solution**:
```
Setup:
1. Enable ECS Task State Change events:
   EventBridge Rule:
   Source: aws.ecs
   Detail-type: "ECS Task State Change"
   
2. Filter for failures:
   detail.lastStatus = "STOPPED"
   detail.stoppedReason contains "exited"
   
3. Actions:
   ├─ SNS → Email ops team (alert)
   ├─ CloudWatch Logs → Log for analysis
   ├─ Lambda → Auto-restart if recoverable
   └─ PagerDuty → Page on-call engineer

Monitoring dashboard:
   ├─ Tasks running count
   ├─ Task failures per hour
   ├─ Average task duration
   └─ CPU/Memory per task
```

---

## Scenario 7: "Migrate existing Java app to containers"

**Solution**:
```
Approach: Use AWS App2Container (A2C)

Step 1: Install A2C CLI
Step 2: Run analysis on existing Java process
  $ app2container analyze [process-id]
  
Step 3: Review generated artifacts
  ├─ Dockerfile (verified correct)
  ├─ CloudFormation (infrastructure template)
  └─ docker-compose.yml (local testing)

Step 4: Build and test locally
  $ docker-compose up

Step 5: Containerize and push to ECR
  $ app2container containerize [process-id]
  
Step 6: Deploy to AWS
  $ aws cloudformation create-stack \
    --template-body file://template.yaml
    
Step 7: App running on ECS Fargate (or EKS/App Runner)
  No code changes required!

Benefits:
  ├─ Fast migration (days, not weeks)
  ├─ Zero code modifications
  ├─ Infrastructure-as-code from day 1
  └─ Fully managed on AWS
```

---

## Scenario 8: "We have multiple services with different deployment frequencies"

**Solution**:
```
Architecture:
├─ Service A: High-traffic API
│  └─ ECS Fargate (auto-scales based on requests)
│
├─ Service B: Scheduled batch job
│  └─ EventBridge + ECS Task (runs hourly)
│
├─ Service C: Complex microservices (50+ services)
│  └─ Amazon EKS (full Kubernetes management)
│
├─ Service D: Simple REST API
│  └─ AWS App Runner (source code → URL)
│
└─ Service E: Legacy Java app
   └─ App2Container + ECS Fargate

Result: Each service deployed with right tool for job
        No over-engineering or under-engineering
        Cost-optimized across all services
```

---

## Scenario 9: "Cost is a big concern"

**Optimization strategies**:
```
1. Right-size tasks:
   Don't use 4 vCPU if app needs 512MB CPU
   Use Fargate for variable workloads (pay for usage)

2. Use Spot instances for non-critical services:
   EKS Managed Node Group with Spot
   70% cost savings (but can be interrupted)

3. Reserved Instances for predictable workloads:
   ECS EC2 Launch Type + Reserve instances
   40% cost savings vs on-demand

4. ECR Lifecycle Policies:
   Delete old untagged images
   Saves S3 storage costs

5. Appropriate launch type:
   ├─ 24/7 services → ECS EC2 + Reserved
   ├─ Bursty/event-driven → Fargate (no idle costs)
   └─ Simple apps → App Runner (built-in optimization)

6. Cross-service efficiency:
   ├─ Don't run underutilized EC2 instances
   ├─ Pack multiple tasks per instance (ALB dynamic port mapping)
   └─ Scale down during off-peak hours
```

---

## Scenario 10: "We need compliance and security for regulated workloads"

**Approach**:
```
Security layers:

1. Network:
   ├─ ECS tasks in private subnets (no public IPs)
   ├─ Security groups with minimal access
   ├─ NACLs for network segmentation
   └─ VPC Flow Logs for monitoring

2. IAM (Least Privilege):
   ├─ EC2 Instance Profile (only ECR/CloudWatch)
   ├─ Task Role (only what the app needs)
   ├─ No wildcard (*) permissions
   └─ Regular audits with IAM Access Analyzer

3. Secrets Management:
   ├─ Never hardcode passwords!
   ├─ Use AWS Secrets Manager or Systems Manager Parameter Store
   ├─ Task assumes role, automatically gets credentials
   └─ Rotate secrets regularly

4. Monitoring & Logging:
   ├─ CloudWatch Logs (all container output)
   ├─ CloudTrail (AWS API calls)
   ├─ VPC Flow Logs (network traffic)
   ├─ GuardDuty (threat detection)
   └─ CloudWatch Alarms (anomalies)

5. Container Security:
   ├─ ECR vulnerability scanning (before deploy)
   ├─ Image signing (verify authenticity)
   ├─ Run as non-root user
   └─ Read-only root filesystem

6. Compliance Tracking:
   ├─ AWS Config (compliance status)
   ├─ AWS Systems Manager (patch management)
   ├─ Audit logs (who did what, when?)
   └─ Regular compliance audits

Result: Production-ready, auditable, secure deployment
```

---

## Exam Tips Summary

```
MUST KNOW:
□ ECS = AWS container orchestration (simpler)
□ EKS = Kubernetes on AWS (industry standard)
□ Fargate = Serverless containers (no EC2 management)
□ EC2 Instance Profile = ECS Agent permissions
□ Task Role = Container application permissions
□ ALB = Load balancer for ECS (recommended)
□ EFS = Shared file system (multi-AZ, mountable)
□ ECR = Docker image registry (IAM-protected)
□ Auto Scaling scales TASKS, not EC2 (for task scaling)
□ Capacity Provider auto-scales EC2 when needed
□ EventBridge → ECS = Event-driven containers
□ App Runner = Simplest (code → URL)
□ App2Container = Migrate Java/.NET to Docker

COMMON MISTAKES:
✗ Putting S3 permissions on Instance Profile (goes on Task Role!)
✗ Thinking Fargate needs EC2 instances (it doesn't!)
✗ Using wrong ALB type (ECS needs ALB, not CLB)
✗ Forgetting valid Fargate CPU/RAM combinations
✗ Scaling EC2 when you should scale tasks
✗ Mounting S3 (not possible; use SDK instead)
✗ Using EKS for simple apps (overkill; use Fargate/App Runner)

EXAM QUESTIONS LIKELY TO ASK:
1. IAM roles: Instance Profile vs Task Role
2. Cost optimization: Which launch type?
3. Scaling: Task scaling vs EC2 scaling
4. Access denied: Which role needs which permission?
5. Architecture: Draw diagram with ALB + ECS + RDS
6. Troubleshooting: "Container can't pull image/access S3/write logs"
7. Migration: Java app → containers
8. Event-driven: S3 + EventBridge + ECS
9. High availability: Multi-AZ setup
10. Security: Least privilege IAM design
```

---

## Final Quick Reference

### Service Comparison Matrix

```
┌────────────┬──────────────┬──────────┬────────────┬────────────┐
│ Feature    │ ECS EC2      │ ECS Far  │ EKS        │ App Runner │
├────────────┼──────────────┼──────────┼────────────┼────────────┤
│ Simplicity │ Medium       │ High     │ Low        │ Highest    │
│ Cost       │ $$$ (24/7)   │ $$ (pay) │ $$ (pay)   │ $$ (pay)   │
│ Control    │ High         │ Medium   │ Highest    │ Low        │
│ Community  │ Good         │ Growing  │ Huge       │ Growing    │
│ Learning   │ 1-2 weeks    │ 1 week   │ 3-4 weeks  │ 1 day      │
│ Multi-cloud│ No           │ No       │ Yes        │ No         │
│ Deploy time│ 30 min       │ 15 min   │ 30 min     │ 5 min      │
└────────────┴──────────────┴──────────┴────────────┴────────────┘
```

### Decision in 30 Seconds

```
Don't know Kubernetes? → ECS or App Runner
Know Kubernetes? → EKS
Want simplest? → App Runner
Want most control? → EKS or ECS EC2
Want cheapest 24/7 service? → ECS EC2 + Reserved Instances
Want pay-per-execution? → Fargate or App Runner
Running Java app? → App2Container → Fargate
```

---

## Additional Resources

- **AWS Container Services Blog**: https://aws.amazon.com/blogs/containers/
- **ECS Best Practices**: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-best-practices.html
- **EKS Best Practices**: https://aws.github.io/aws-eks-best-practices/
- **ECR Tutorials**: https://docs.aws.amazon.com/AmazonECR/latest/userguide/ecr-getting-started.html
- **AWS Containers Community Day**: https://www.youtube.com/watch?v=j_UCVmIBXUs
- **AWS Solutions Architect Associate Course**: Ultimate AWS Certified Solutions Architect Associate

---

**Study Strategy**:
1. Read sections 1-5 first (core concepts)
2. Do practice questions on ECS IAM roles (most confusing)
3. Review architecture diagrams (visualize the flow)
4. Study integration patterns (EventBridge + ECS)
5. Practice troubleshooting scenarios
6. Do practice exams with container questions

Good luck! You've got this! 🎓


## Quick Reference — AWS CLI Commands

### ECR (Elastic Container Registry) Commands

```bash
# Create an ECR private repository
aws ecr create-repository \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

# Authenticate Docker to ECR (login)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS \
  --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Tag a local Docker image for ECR
docker tag my-app:latest \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# Push image to ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# Pull image from ECR
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# List repositories
aws ecr describe-repositories

# List images in repository
aws ecr list-images --repository-name my-app

# Delete an image
aws ecr batch-delete-image \
  --repository-name my-app \
  --image-ids imageTag=old-version

# Set lifecycle policy (keep only last 10 images)
aws ecr put-lifecycle-policy \
  --repository-name my-app \
  --lifecycle-policy-text '{
    "rules": [{
      "rulePriority": 1,
      "selection": {
        "tagStatus": "untagged",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {"type": "expire"}
    }]
  }'
```

### ECS (Elastic Container Service) Commands

```bash
# Create an ECS cluster
aws ecs create-cluster \
  --cluster-name my-app-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1

# Register a task definition (Fargate)
aws ecs register-task-definition \
  --family my-app-task \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu "256" \
  --memory "512" \
  --execution-role-arn arn:aws:iam::123456789012:role/ecsTaskExecutionRole \
  --container-definitions '[{
    "name": "my-app",
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
    "portMappings": [{"containerPort": 80, "hostPort": 80}],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/my-app-task",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }]'

# Create an ECS Service (Fargate)
aws ecs create-service \
  --cluster my-app-cluster \
  --service-name my-app-service \
  --task-definition my-app-task:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-0abc123def456", "subnet-0def456abc123"],
      "securityGroups": ["sg-0abc123def456"],
      "assignPublicIp": "ENABLED"
    }
  }' \
  --load-balancers '[{
    "targetGroupArn": "arn:aws:elasticloadbalancing:...",
    "containerName": "my-app",
    "containerPort": 80
  }]'

# Run a one-off ECS task
aws ecs run-task \
  --cluster my-app-cluster \
  --task-definition my-app-task:1 \
  --launch-type FARGATE \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-0abc123def456"],
      "securityGroups": ["sg-0abc123def456"],
      "assignPublicIp": "ENABLED"
    }
  }'

# Describe running tasks
aws ecs list-tasks --cluster my-app-cluster
aws ecs describe-tasks \
  --cluster my-app-cluster \
  --tasks arn:aws:ecs:us-east-1:123456789012:task/my-app-cluster/abc123

# Update service (rolling deploy new image)
aws ecs update-service \
  --cluster my-app-cluster \
  --service my-app-service \
  --task-definition my-app-task:2 \
  --desired-count 3

# Force new deployment (redeploy same task definition)
aws ecs update-service \
  --cluster my-app-cluster \
  --service my-app-service \
  --force-new-deployment

# Scale service
aws ecs update-service \
  --cluster my-app-cluster \
  --service my-app-service \
  --desired-count 5

# Delete service (must scale down first)
aws ecs update-service \
  --cluster my-app-cluster \
  --service my-app-service \
  --desired-count 0
aws ecs delete-service \
  --cluster my-app-cluster \
  --service my-app-service
```

### EKS (Elastic Kubernetes Service) Commands

```bash
# Create an EKS cluster
aws eks create-cluster \
  --name my-k8s-cluster \
  --kubernetes-version "1.29" \
  --role-arn arn:aws:iam::123456789012:role/eks-cluster-role \
  --resources-vpc-config \
    subnetIds=subnet-0abc123def456,subnet-0def456abc123,\
securityGroupIds=sg-0abc123def456

# Update kubeconfig (allow kubectl access)
aws eks update-kubeconfig \
  --name my-k8s-cluster \
  --region us-east-1

# Create a managed node group
aws eks create-nodegroup \
  --cluster-name my-k8s-cluster \
  --nodegroup-name my-node-group \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --subnets subnet-0abc123def456 subnet-0def456abc123 \
  --instance-types t3.medium \
  --scaling-config minSize=2,maxSize=5,desiredSize=2

# List clusters
aws eks list-clusters

# Describe cluster
aws eks describe-cluster --name my-k8s-cluster

# kubectl commands (after aws eks update-kubeconfig)
kubectl get nodes
kubectl get pods --all-namespaces
kubectl apply -f deployment.yaml
kubectl get services
```

---

