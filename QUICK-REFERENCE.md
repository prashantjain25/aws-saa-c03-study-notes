# AWS SAA-C03 — Classic Solutions Architecture — Quick Reference Card

## The Three Classic Architectures at a Glance

### Architecture 1: WhatIsTheTime.com (Stateless Web App)
```
What: Simple website, no user state needed
Why: Easiest to scale, no state management complexity
Pattern: EC2 ASG → ALB → Route 53 + Golden AMI
Scaling: Horizontal (add more instances)
Failover: Seconds (ALB health checks)
Cost: Elastic (scale down at night)
```

**Key Insight**: Every instance is identical and interchangeable.

---

### Architecture 2: MyClothes.com (Stateful, E-Commerce)
```
What: Shopping cart persists across requests
Why: User state matters (cart, preferences, session)
Pattern: EC2 ASG → ALB → ElastiCache (sessions) + RDS (data)
Session Storage: ElastiCache (Redis)
Persistence: RDS Multi-AZ
Cache Pattern: Lazy Loading (only cache what's needed)
Failover: ~5 seconds (cache), ~1-2 min (DB)
```

**Key Insight**: Sessions in ElastiCache (server-side), not cookies or sticky sessions.

---

### Architecture 3: MyWordPress.com (Stateful + Media)
```
What: Database content + uploaded images/files
Why: Content management with shared media across instances
Pattern: EC2 ASG → ALB → RDS/Aurora + EFS (media)
Database: Aurora MySQL (better than plain RDS)
Media Storage: EFS (NOT EBS!)
Caching: ElastiCache for frequently accessed data
Cost: Pay for RDS, per-GB for EFS
```

**Key Insight**: EFS for shared files (all instances see same files). EBS is single-instance only.

---

## Critical Decision Trees

### "Our shopping cart is disappearing!"
```
Symptom: Cart data lost when request routes to different instance

Option 1: ELB Stickiness ❌
  Problem: Single instance = lost cart on failure

Option 2: Client-side cookies ❌
  Problem: 4 KB limit, security risk

Option 3: ElastiCache + session_id ✅
  Solution: Session ID in cookie, full data in cache
  All instances read from same cache
```

---

### "Images aren't visible on some instances!"
```
Symptom: Upload on Instance A, can't see on Instance B

Option 1: EBS volume ❌
  Problem: Attached to one instance only

Option 2: EFS ✅
  Solution: Mount same EFS on all instances
  All instances read/write same shared storage
```

---

### "ASG scaling is too slow!"
```
Symptom: Traffic spike, but new instances take 10 minutes

Option 1: User Data bootstrap ❌
  Problem: 7-10 minutes to install everything

Option 2: Golden AMI ✅
  Solution: Pre-bake all dependencies into AMI
  New instances ready in 1-2 minutes
```

---

## AWS Service Cheat Sheet

### Compute
| Service | What | When |
|---------|------|------|
| **EC2** | Virtual machines | Always (foundation) |
| **ASG** | Auto scaling group | Automatic scaling |
| **Golden AMI** | Pre-configured snapshot | Fast instance launch |

### Networking
| Service | What | When |
|---------|------|------|
| **Route 53** | DNS | Domain name mapping |
| **ALB** | Application load balancer | HTTP/HTTPS traffic |
| **Health Checks** | Instance liveness | Failover automation |

### Caching & Sessions
| Service | What | When |
|---------|------|------|
| **ElastiCache (Redis)** | In-memory cache | Sessions, temporary data |
| **ElastiCache (Memcached)** | Simple cache | Session only (no persistence) |
| **Lazy Loading** | Cache pattern | Reduce DB load |

### Databases
| Service | What | When |
|---------|------|------|
| **RDS** | Managed relational DB | Traditional applications |
| **RDS Multi-AZ** | Standby in different AZ | High availability |
| **RDS Read Replicas** | Read-only copies | Scale read traffic |
| **Aurora** | AWS-optimized MySQL/PostgreSQL | Better than RDS |

### Storage
| Service | What | When |
|---------|------|------|
| **EBS** | Block storage | OS, single instance |
| **EFS** | Network file system | Shared storage (WordPress media) |
| **S3** | Object storage | Backups, static files (not covered in this guide) |

---

## "When to Use" Decision Matrix

### Session Storage Decision
```
If: Non-persistent, need fast access
→ ElastiCache (Redis/Memcached)

If: Serverless, auto-scaling important
→ DynamoDB

If: Simple, non-critical settings
→ Browser cookies

If: Very durable, don't care about speed
→ RDS (not recommended, but possible)
```

### Database Scaling Decision
```
If: Too many READS
→ Add RDS Read Replicas + ElastiCache Lazy Loading

If: Too many WRITES
→ Database sharding (advanced, not SAA-C03)

If: Need disaster recovery
→ Multi-AZ (automatic failover)

If: Need cross-region DR
→ Read Replicas in different regions
```

### Storage Decision
```
If: Single instance, high performance
→ EBS

If: Multiple instances need same data
→ EFS

If: Very large files, infrequent access
→ S3

If: Database files
→ EBS (or use managed RDS)
```

---

## The 30-Second Elevator Pitch

### For Stateless Apps
"Use EC2 ASG behind an ALB with Golden AMI. Route 53 points to ALB. All instances are identical, so any instance can handle any request. Scales automatically, fails over in seconds."

### For Stateful Apps
"Use ElastiCache for session storage. Session ID in cookie (tiny), full data in cache (safe, scalable). RDS Multi-AZ for persistent data. Read Replicas for scaling reads. Lazy Loading pattern to reduce DB load."

### For Apps with Media
"Use RDS/Aurora for database. EFS for uploaded files. All EC2 instances mount same EFS, so uploads visible everywhere. Multi-AZ throughout for resilience."

---

## Common Gotchas

| Gotcha | Correct Answer |
|--------|---|
| Use EBS for shared storage | Use EFS instead |
| Use sticky sessions for resilience | Use ElastiCache instead |
| Rely on User Data for fast scaling | Use Golden AMI instead |
| DNS multiple IPs for health checking | Use ALB with health checks instead |
| RDS for session storage | Use ElastiCache instead |
| Single AZ for production | Always deploy Multi-AZ |
| Add Read Replica for failover | Add Multi-AZ instead (Read Replicas are for scaling) |

---

## Exam Timing Strategy

**Quick-win questions (1-2 min)**:
- "What AWS service does X?" → Obvious answer (EFS for shared storage, ElastiCache for sessions)

**Think-it-through questions (3-5 min)**:
- "Design architecture for this scenario" → Use decision trees above
- "Why is X failing?" → Identify root cause (stickiness, EBS, slow startup)

**Time-consuming questions (skip, come back)**:
- Complex multi-step architecture with multiple services
- Questions about reserved instances, cost optimization
- Regulatory/compliance details

**Pro tip**: Flag questions, complete obvious ones first, come back to hard ones.

---

## Five-Minute Prep Before Exam

1. **Memorize the three architectures**
   - Stateless (WhatIsTheTime)
   - Stateful with sessions (MyClothes)
   - Stateful with media (MyWordPress)

2. **Know service trade-offs**
   - EBS vs EFS
   - Sticky sessions vs ElastiCache
   - User Data vs Golden AMI
   - Multi-AZ vs Read Replicas

3. **Remember the decision trees**
   - Cart disappearing → ElastiCache
   - Images missing → EFS
   - Scaling slow → Golden AMI
   - Reads slow → Read Replicas + Lazy Loading

4. **Know the health check principle**
   - ALB can detect failures in 5 seconds
   - Much better than DNS TTL (minutes to hours)

5. **Remember cost optimization**
   - ASG scales down at night (saves money)
   - EFS pay per GB used (not provisioned)
   - Golden AMI speeds up, reduces failed launches

---

## Good Luck!

**Key mindset**: Think like an architect.
- Why does this solution work?
- What could go wrong?
- How do we make it resilient?
- Can we scale it?
- How much does it cost?

You've got this!

