# Section 13 — Cost Control

> **Purpose**: Cloud cost optimization is not about minimizing spend — it is about maximizing value per dollar spent. A senior architect understands AWS pricing models, identifies waste, and designs systems that scale cost-efficiently. Cost is a first-class architectural constraint.
>
> **Official Documentation**: [Billing](https://docs.aws.amazon.com/account-billing/) | [Cost Explorer](https://docs.aws.amazon.com/cost-management/) | [Savings Plans](https://docs.aws.amazon.com/savingsplans/) | [Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/)

---

## 1. AWS Pricing Models

### 1.1 Fundamental Pricing Dimensions

| Dimension | Services | Optimization Strategy |
|-----------|----------|---------------------|
| **Compute time** | EC2, Lambda, Fargate | Right-size instances, use Spot, leverage Savings Plans |
| **Storage capacity** | S3, EBS, EFS | Lifecycle policies, storage class selection, compression |
| **Data transfer** | All services | Keep traffic within AZ (free), use VPC endpoints, cache at edge |
| **API requests** | S3, DynamoDB, Lambda | Batch operations, reduce request frequency, use caching |
| **Provisioned capacity** | RDS, DynamoDB, NAT Gateway | Auto-scaling, right-sizing, use serverless where appropriate |

### 1.2 Data Transfer Cost Architecture

Data transfer is often the largest unexpected cost:

```
EC2 ──► EC2 (same AZ, same VPC):     FREE
EC2 ──► EC2 (different AZ, same VPC): $0.01/GB
EC2 ──► Internet:                     $0.09/GB (first 10 TB)
EC2 ──► S3 (same region):             FREE (via Gateway Endpoint)
EC2 ──► S3 (same region, no endpoint): $0.01/GB (via NAT Gateway + $0.045/GB processing)
CloudFront ──► Internet:              $0.085/GB (regional) to $0.020/GB (cheapest edge)
```

**Cost optimization strategies**:
- Use **VPC Gateway Endpoints** for S3/DynamoDB (free, avoids NAT Gateway charges)
- Use **CloudFront** for content delivery (cheaper than direct internet egress for cached content)
- Keep microservices in the **same AZ** when possible (free data transfer)
- Use **PrivateLink** instead of public internet for cross-account service access

---

## 2. Cost Management Tools

### 2.1 AWS Cost Explorer and Budgets

**Cost Explorer**: Analyze spend by service, region, account, tag, and time period.

**Budgets**: Set thresholds and receive alerts:
- Cost budgets ("alert if monthly spend > $10K")
- Usage budgets ("alert if EC2 hours > 1,000")
- RI utilization budgets ("alert if RI coverage < 80%")

### 2.2 AWS Compute Optimizer

ML-powered recommendations for right-sizing:
- Analyzes CloudWatch metrics over 14 days
- Recommends optimal instance types, sizes, and configurations
- Shows estimated savings and risk (under-provisioning vs over-provisioning)

> **Limitation**: Compute Optimizer analyzes historical metrics. It cannot predict future workload changes. Use it as a starting point, not a definitive answer.

### 2.3 AWS Cost Anomaly Detection

ML-powered detection of unusual spending patterns:
- Monitors services for unexpected cost increases
- Alerts via SNS when anomalies detected
- Distinguishes between expected scaling and actual anomalies

---

## 3. Savings Plans and Reserved Capacity

### 3.1 Savings Plans vs Reserved Instances

| Feature | Savings Plans | Reserved Instances |
|---------|--------------|-------------------|
| **Commitment** | $/hour compute spend | Per instance type, region, AZ |
| **Flexibility** | High (applies across instance families, regions for Compute SP) | Low (specific to purchased instance type) |
| **Payment options** | All Upfront, Partial Upfront, No Upfront | Same |
| **Term** | 1 or 3 years | 1 or 3 years |
| **Discount** | Up to 66% (Compute), 72% (EC2 Instance) | Up to 72% |
| **Modifiable** | Cannot modify commitment | Standard RIs can be sold on marketplace; Convertible RIs can exchange |

**Recommendations**:
- Start with **Compute Savings Plans** for maximum flexibility
- Add **EC2 Instance Savings Plans** once workload patterns stabilize
- Purchase through AWS Cost Explorer recommendations

### 3.2 Spot Instances Strategy

**Spot Instance savings**: Up to 90% off On-Demand pricing.

**Production-safe Spot patterns**:
- **Mixed Instances Policy**: Base capacity On-Demand, burst with Spot
- **Spot Fleets**: Diversify across instance types and AZs to reduce interruption probability
- **Checkpointing**: Save progress frequently for long-running jobs
- **Capacity Rebalancing**: ASG proactively replaces Spot instances when AWS signals interruption risk

> **Spot interruption behavior**: AWS gives a 2-minute warning. Applications must handle graceful shutdown: save state, drain connections, and exit cleanly.

---

## 4. Tagging and Cost Allocation

**Mandatory tags for cost visibility**:
- `Environment` (dev/staging/prod)
- `Project` or `Application`
- `Team` or `CostCenter`
- `Owner`

**AWS Cost Allocation Tags**: Activate user-defined tags in the Billing console to enable Cost Explorer filtering.

**SCP enforcement**: Use Service Control Policies to require tags on resource creation:
```json
{
  "Condition": {
    "Null": {
      "aws:RequestTag/Environment": "true"
    }
  },
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*"
}
```

---

## 5. Points to Remember

- **Data transfer is often the hidden cost** — VPC endpoints, CloudFront, and AZ-aware placement save significant money.
- **NAT Gateway charges $0.045/GB processing** — use Gateway Endpoints for S3/DynamoDB to eliminate this.
- **Savings Plans provide flexibility that RIs lack** — start with Compute Savings Plans.
- **Spot Instances are safe for fault-tolerant workloads** — use Mixed Instances Policies with On-Demand base capacity.
- **Tagging is mandatory for cost accountability** — enforce via SCPs.
- **Compute Optimizer provides data-driven right-sizing** — review recommendations monthly.
- **Cost Anomaly Detection catches unexpected spend** — configure for all production accounts.
- **S3 Intelligent-Tiering eliminates guesswork** — small monitoring fee, automatic cost optimization.

---

*Section 13 — Cost Control | Last Validated: 2026-05-10*
