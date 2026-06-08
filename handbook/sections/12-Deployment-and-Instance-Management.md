# Section 12 — Deployment & Instance Management

> **Purpose**: How code gets from a developer's machine to production, and how instances are managed throughout their lifecycle, are foundational operational concerns. This section covers deployment automation, systems management, and the AWS services that enable safe, repeatable infrastructure changes.
>
> **Official Documentation**: [CloudFormation](https://docs.aws.amazon.com/cloudformation/) | [Systems Manager](https://docs.aws.amazon.com/systems-manager/) | [CodeDeploy](https://docs.aws.amazon.com/codedeploy/)

---

## 1. AWS Systems Manager (SSM)

### 1.1 SSM Core Capabilities

Systems Manager is AWS's unified operations platform:

| Capability | Purpose |
|-----------|---------|
| **Session Manager** | Secure shell access to EC2 without SSH keys, bastion hosts, or open port 22. Logs all sessions to S3/CloudWatch. |
| **Run Command** | Execute commands on managed instances at scale (patching, configuration updates, diagnostics). |
| **Patch Manager** | Automate OS patching with defined patch baselines and maintenance windows. |
| **Automation** | Pre-defined runbooks (e.g., create AMI, stop/start instances, copy snapshots). |
| **Parameter Store** | Secure hierarchical storage for configuration and secrets. |
| **Inventory** | Collect metadata about managed instances (OS, applications, drivers). |
| **OpsCenter** | Aggregates operational issues and provides remediation runbooks. |

### 1.2 Session Manager vs SSH

| Dimension | SSH | Session Manager |
|-----------|-----|-----------------|
| **Port requirements** | TCP 22 inbound | HTTPS outbound (443) via SSM agent |
| **Key management** | SSH keys (security risk) | IAM-based authentication |
| **Audit trail** | Custom logging required | Built-in — logs to S3/CloudWatch |
| **Bastion host** | Often required | Not required |
| **Cross-platform** | Linux/macOS primarily | Windows (PowerShell), Linux, macOS |

> **Security Best Practice**: Disable SSH (port 22) entirely and use Session Manager. This eliminates SSH key management, bastion hosts, and brute-force attack surface.

### 1.3 Patch Manager

**Patch baselines** define which patches are approved:
- AWS-provided baselines (e.g., `AWS-AmazonLinux2DefaultPatchBaseline`)
- Custom baselines with specific approval rules

**Maintenance windows**: Scheduled times when patching is allowed. Associate targets (instances) and tasks (patch, reboot).

> **Operational Reality**: Patching can fail (dependency conflicts, insufficient disk space). Always configure maintenance windows with sufficient rollback time and monitor patch compliance via Systems Manager Compliance Dashboard.

---

## 2. AWS CloudFormation

### 2.1 Infrastructure as Code

CloudFormation treats infrastructure as code: declarative templates that AWS converges to the desired state.

**Template structure**:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Example stack'
Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'my-bucket-${Environment}'
Outputs:
  BucketArn:
    Value: !GetAtt MyBucket.Arn
```

### 2.2 Stack Operations

| Operation | Behavior | Risk |
|-----------|----------|------|
| **Create** | New resources from template | Low |
| **Update** | Changes detected, resources modified | Medium — review change sets |
| **Delete** | All resources removed | High — data loss if no backups |
| **Drift detection** | Compare actual state to template | Identifies manual changes |

**Change Sets**: Preview changes before applying. Essential for production updates.

### 2.3 StackSets

Deploy the same CloudFormation template across multiple accounts and regions:
- **Administrator account**: Creates and manages StackSets
- **Target accounts**: Receive stacks via trusted access
- **Automatic deployment**: New accounts added to Organization automatically receive stacks

> **CloudFormation vs Terraform**: CloudFormation is AWS-native, free, and has deep service integration. Terraform is multi-cloud, has a larger provider ecosystem, and manages state independently. Many organizations use Terraform for multi-cloud/orchestration and CloudFormation for AWS-specific resources.

---

## 3. AWS CodeDeploy

CodeDeploy automates application deployments to EC2, Lambda, and ECS.

**Deployment strategies**:

| Strategy | Behavior | Downtime | Use Case |
|----------|----------|----------|----------|
| **In-Place** | Replace code on existing instances | Yes (brief) | Quick deployments, stateful instances |
| **Blue/Green** | Deploy to new instances, swap traffic | No | Zero-downtime deployments, easy rollback |
| **Canary** | Route small % of traffic to new version, gradually increase | No | Risk mitigation, A/B testing |
| **Linear** | Increase traffic by fixed percentage at fixed intervals | No | Controlled rollout |

**Deployment hooks**: Scripts run at lifecycle events (ApplicationStop, BeforeInstall, AfterInstall, ApplicationStart, ValidateService).

> **ECS/EKS Deployment**: Use CodeDeploy for ECS blue/green deployments. For EKS, use native Kubernetes deployment strategies (rolling update, canary with Argo Rollouts or Flagger).

---

## 4. AWS OpsWorks

OpsWorks provides managed Chef and Puppet automation. It is a legacy service — most new workloads use CloudFormation, Terraform, or container orchestrators instead.

> **Architectural Guidance**: Avoid OpsWorks for new projects. Use Systems Manager + CloudFormation + CI/CD pipelines for modern deployment automation.

---

## 5. AWS AppConfig

AppConfig manages application configuration separate from code:
- **Feature flags**: Enable/disable features without deployment
- **Configuration profiles**: JSON/YAML/text configurations
- **Deployment strategies**: Canary, linear, all-at-once
- **Rollback**: Automatic rollback on CloudWatch alarm

> **AppConfig vs Parameter Store**: AppConfig is for application configuration that changes frequently and needs safe deployment strategies. Parameter Store is for static/semi-static values and secrets. Use both: Parameter Store for secrets and stable config, AppConfig for feature flags and dynamic config.

---

## 6. Architectural Decision Challenges

* **Scenario:** Design a zero-downtime deployment for an EC2-based web application.
  * **Design:** Implement a Blue/Green deployment with CodeDeploy. Because this allows you to maintain the current environment (Blue) running v1 behind an ALB while deploying v2 to a new ASG (Green) with the same ALB but a separate target group. After running smoke tests against Green, you swap ALB traffic from the Blue to Green target group and monitor CloudWatch metrics and error rates. If issues are detected, you can swap back to Blue (rollback in seconds). If stable, terminate the Blue instances after a validation period. Alternatively, use ALB weighted target groups for a canary deployment (gradual traffic shift).

---

## 7. Points to Remember

- **Session Manager replaces SSH and bastion hosts** — more secure, fully auditable.
- **Patch Manager requires maintenance windows** — define approved baselines and monitor compliance.
- **CloudFormation change sets are essential** — always preview changes before production updates.
- **StackSets enable multi-account, multi-region IaC** — use for baseline security configurations.
- **CodeDeploy blue/green provides instant rollback** — swap target groups to revert in seconds.
- **AppConfig enables safe feature flag deployment** — canary rollout with automatic rollback on alarm.
- **Drift detection identifies manual changes** — run regularly to catch configuration drift.

---

## 8. Further Reading

For deeper coverage, CLI commands, and exam-specific trivia, see the detailed reference:

- **EC2 placement groups, ENI, hibernate, instance management**: [`EC2-Associate-PlacementGroups-ENI-Hibernate.md`](../../detailed-reference/EC2-Associate-PlacementGroups-ENI-Hibernate.md)
- **ECS, EKS, container deployment patterns**: [`Containers-ECS-EKS-ECR-Fargate.md`](../../detailed-reference/Containers-ECS-EKS-ECR-Fargate.md)

---

*Section 12 — Deployment & Instance Management | Last Validated: 2026-05-10*
