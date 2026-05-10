# AWS SAA-C03 Study Notes - VPC Networking & Security

## Document Summary

**File:** `VPC-Networking-Security.md`  
**Size:** 2,039 lines  
**Format:** Markdown  
**Exam Focus:** AWS Solutions Architect Associate (SAA-C03)  
**Teaching Style:** Expert-led approach (Why over What, Deep Technical Detail, Real-World Analogies)

## Content Overview

This comprehensive study guide covers:

### Core VPC Topics (8 sections)
- **VPC Fundamentals** - Architecture, CIDR notation, 5 reserved IPs trap, default vs custom VPCs
- **Subnets & Availability** - Public vs private, AZ binding, auto-assign public IP
- **Internet Gateway (IGW)** - Creation, attachment, route configuration
- **NAT Gateway vs NAT Instance** - Managed vs self-managed, bandwidth, high availability patterns
- **Network ACLs (NACLs)** - Stateless rules, ephemeral ports, rule evaluation order
- **Security Groups** - Stateful firewalls, cross-SG references, instance-level control
- **VPC Peering** - Non-transitivity, cross-account/region, CIDR constraints
- **VPC Endpoints** - Gateway endpoints (free), interface endpoints (PrivateLink), S3 endpoint exam question

### Hybrid & Scale Topics (3 sections)
- **VPC Flow Logs** - Capture methodology, what's NOT captured, CloudWatch vs S3 analysis
- **Site-to-Site VPN & Direct Connect** - Speed/cost comparison, redundancy patterns, two-tunnel architecture
- **Transit Gateway** - Hub-and-spoke model, route tables, ECMP bandwidth scaling

### Security & Encryption (5 sections)
- **AWS KMS** - Key types, envelope encryption, key policies, asymmetric keys, multi-region keys
- **SSM Parameter Store & Secrets Manager** - Config vs secrets, automatic rotation, RDS integration
- **AWS WAF, Shield, Firewall Manager** - Layer 7 vs Layer 3/4 protection, DDoS coverage, centralized management
- **GuardDuty, Inspector, Macie** - Threat detection, vulnerability scanning, data discovery

### Reference Material (3 sections)
- **AWS CLI Quick Reference** - 50+ practical CLI commands covering all topics
- **Comparison Tables** - 
  - VPC Flow Logs vs CloudTrail vs GuardDuty vs Config
  - WAF vs Shield vs Firewall Manager vs GuardDuty vs Inspector
- **Exam Tips & Interview Patterns** - Common gotchas, interview questions, cost optimization

## Key Features

### Teaching Methodology
- **Analogies:** Apartment building, border control, hub airport, corporate proxy
- **Why explanations:** Why NACL is stateless, why NAT is one-way, why you need route tables
- **Exam tricks:** 5 reserved IPs, missing route table causes connectivity issues, non-transitive peering
- **Interview patterns:** 3-tier architecture design, VPC connection strategies, encryption patterns

### AWS CLI Commands (187 instances, 50+ unique commands)
Commands cover all major services:
- VPC operations (create, delete, describe)
- Route tables and routing
- NAT Gateway lifecycle
- Security groups with cross-SG rules
- NACLs with rule ordering
- VPC peering and acceptance
- VPC endpoints (gateway and interface)
- Flow logs configuration
- VPN and Direct Connect
- Transit Gateway attachments
- KMS key operations and rotation
- Secrets Manager rotation
- SSM Parameter Store with encryption
- GuardDuty detector management
- WAF web ACLs and IP sets

### Practical Examples
- Creating a public-private subnet pair
- NAT Gateway high availability setup
- Envelope encryption with GenerateDataKey
- Route propagation with Transit Gateway
- Security group cross-referencing patterns
- Flow logs analysis queries

### Exam-Focused Content
- **SAA-C03 Common Questions:**
  - "Instances can't reach internet" → Check route table (not just SG/NACL)
  - "Connect hundreds of VPCs" → Transit Gateway (not peering)
  - "Private EC2 needs S3" → Gateway endpoint (free, not NAT)
  - "Encrypt database password" → Secrets Manager (automatic rotation)
  - "5 IPs per subnet" → 251 usable IPs in /24, not 256

## How to Use This Guide

### For Exam Preparation
1. Read each section with focus on "Why" explanations
2. Review "Points to Remember" at end of each section
3. Practice AWS CLI commands before exam day
4. Study comparison tables for scenario questions
5. Review exam tips section 1 day before test

### For Interview Preparation
1. Study interview questions in final section
2. Practice explaining architecture patterns (3-tier, hybrid)
3. Know the trade-offs (NAT Gateway vs Instance, VPN vs Direct Connect)
4. Be ready to discuss security in depth

### As Reference Material
- Use AWS CLI Quick Reference while building in console
- Refer to comparison tables when unsure between similar services
- Check exam tips for common gotchas

## Coverage Checklist

### VPC Fundamentals
✓ VPC definition and analogies
✓ CIDR notation and math
✓ 5 reserved IPs per subnet (exam trap)
✓ Default vs custom VPC
✓ VPC spans AZs, subnets are AZ-specific

### Networking Components
✓ Internet Gateway (attachment + routing)
✓ NAT Gateway (managed, per-AZ HA)
✓ NAT Instance (self-managed, Source/Destination check)
✓ Public vs private subnets
✓ Routing and route tables

### Security & Filtering
✓ Network ACLs (stateless, rules, ephemeral ports)
✓ Security Groups (stateful, cross-SG references)
✓ NACL vs SG comparison
✓ Endpoint policies for VPC endpoints

### VPC Connectivity
✓ VPC peering (1:1, non-transitive, cross-region/account)
✓ VPC endpoints (gateway for S3/DynamoDB, interface for others)
✓ Flow logs (CloudWatch/S3, what's NOT captured)

### Hybrid Connectivity
✓ Site-to-Site VPN (quick, internet-based, 1.25 Gbps)
✓ Direct Connect (dedicated, slow setup, 1-100 Gbps)
✓ VPN + DX redundancy patterns
✓ Transit Gateway (hub-and-spoke, solves peering limitation)

### Encryption & Secrets
✓ KMS key types (AWS owned/managed/customer managed)
✓ Encryption operations (Encrypt, Decrypt, GenerateDataKey)
✓ Envelope encryption for large files
✓ Key policies and permissions
✓ Automatic key rotation
✓ Asymmetric keys for external use

### Secrets Management
✓ Parameter Store (free config/secrets)
✓ Secrets Manager (automatic rotation)
✓ RDS integration
✓ Difference and when to use each

### Threat Detection & Compliance
✓ GuardDuty (threat detection via ML)
✓ Inspector (vulnerability scanning)
✓ Macie (data discovery and classification)
✓ WAF (Layer 7 protection)
✓ Shield (DDoS protection, free vs advanced)
✓ Firewall Manager (centralized management)

## Statistics

- **Total lines:** 2,039
- **Major sections:** 18
- **Subsections:** 80+
- **AWS CLI commands:** 187 instances (50+ unique)
- **Comparison tables:** 2 comprehensive
- **Exam tips:** 10 key gotchas + 5 interview questions
- **Code blocks:** 100+
- **Analogies:** 15+

## Notes

- Written specifically for SAA-C03 exam level (not Developer Associate)
- Expert-led teaching style: Why > What, Real-world analogies, Deep technical detail
- All CLI commands use region flag or are region-agnostic
- Pricing information current as of March 2026
- Focus on concepts that appear frequently on exams

## Related Study Materials

To complement this guide:
- Official AWS documentation (for latest CLI syntax)
- AWS Solutions Architect Associate certification course
- AWS Well-Architected Framework whitepaper
- AWS pricing calculator (for cost scenarios)

---

**Created:** 2026-03-06  
**Format:** Markdown (can be read in any text editor or on GitHub)  
**License:** Educational use
