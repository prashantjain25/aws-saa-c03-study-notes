# Section 14 — Migration

> **Purpose**: Cloud migration is not a one-time event — it is a continuous process of moving workloads, data, and dependencies to AWS. A senior architect designs migration strategies that minimize risk, maintain business continuity, and optimize for cloud-native patterns. This section covers the AWS migration portfolio: from mainframe replatforming to petabyte-scale database migrations.
>
> **Official Documentation**: [Migration Hub](https://docs.aws.amazon.com/migrationhub/) | [DMS](https://docs.aws.amazon.com/dms/) | [Application Migration Service](https://docs.aws.amazon.com/mgn/) | [AWS Backup](https://docs.aws.amazon.com/aws-backup/)

---

## 1. The 7 R's of Migration

| Strategy | Description | Effort | Risk | Example |
|----------|-------------|--------|------|---------|
| **Retire** | Decommission unused applications | Low | Low | Legacy reporting tool no longer needed |
| **Retain** | Keep on-premises for compliance or dependency | None | Low | Mainframe batch processing |
| **Rehost (Lift & Shift)** | Move as-is to AWS | Low | Medium | VMware to EC2 using MGN |
| **Relocate** | Move infrastructure without changing platform | Low | Medium | Hyper-V to VMware Cloud on AWS |
| **Replatform** | Move with minor optimizations | Medium | Medium | Oracle on EC2 → RDS Oracle |
| **Refactor/Re-architect** | Cloud-native redesign | High | High | Monolith → microservices on ECS |
| **Repurchase** | Move to SaaS | Medium | Medium | Exchange → Microsoft 365 |

> **Architectural Guidance**: Start with Rehost for quick wins and risk reduction. Then Replatform for operational efficiency. Refactor only for critical applications where the business case is clear.

---

## 2. AWS Application Migration Service (MGN)

MGN (formerly CloudEndure Migration) provides **agent-based replication** for lift-and-shift migrations:

**How it works**:
1. Install replication agent on source server (physical, virtual, or cloud)
2. Agent replicates entire disk volumes to AWS in real-time
3. AWS maintains a staging area with replicated volumes
4. When ready, launch EC2 instances from replicated volumes
5. Test, validate, then cut over traffic

**Key features**:
- **Continuous replication**: Changes sync in real-time; minimal cutover window
- **OS support**: Windows, Linux, common enterprise distributions
- **Networking**: Preserves source IP via VPC peering or Direct Connect
- **Post-launch actions**: Automatically install SSM agent, configure CloudWatch, run scripts

> **MGN vs SMS (Server Migration Service)**: SMS was the predecessor. MGN replaces SMS with better replication performance, broader OS support, and modern AWS integrations. Do not use SMS for new migrations.

---

## 3. AWS Database Migration Service (DMS)

DMS migrates databases **with minimal downtime**:

### 3.1 DMS Components

| Component | Purpose |
|-----------|---------|
| **Replication Instance** | EC2 instance that runs the migration engine |
| **Endpoints** | Source and target database connections |
| **Task** | Defines what to migrate (full load, CDC, or both) |

### 3.2 Migration Types

| Type | Behavior | Use Case |
|------|----------|----------|
| **Full Load** | One-time migration of existing data | Small databases, maintenance windows acceptable |
| **Full Load + CDC** | Migrate existing data, then replicate ongoing changes | Large databases, minimal downtime required |
| **CDC Only** | Replicate changes from a specific point | Already migrated data, catching up changes |

**CDC (Change Data Capture)**: DMS reads source database transaction logs to replicate INSERTs, UPDATEs, and DELETEs to the target in near real-time.

### 3.3 DMS Schema Conversion Tool (SCT)

SCT converts database schemas between engines:
- Oracle → PostgreSQL/Aurora
- SQL Server → MySQL/Aurora
- MySQL → Aurora MySQL
- Teradata/Netezza → Redshift

> **SCT Limitation**: SCT automates ~70-90% of schema conversion. Complex stored procedures, triggers, and proprietary features require manual rework. Always test application compatibility after conversion.

### 3.4 DMS Limitations

- No automatic schema creation for homogeneous migrations (you create target schema)
- LOB (Large Object) columns require special handling and slow replication
- Some DDL operations (ALTER TABLE) may not be captured during CDC
- Target must be accessible from the Replication Instance's VPC/security groups

---

## 4. AWS DataSync

DataSync is a **managed data transfer service** for moving data between on-premises and AWS:

**Supported locations**:
- NFS, SMB (on-premises file systems)
- S3, EFS, FSx (AWS storage)
- Other cloud providers (via agent)

**How it works**:
1. Deploy DataSync agent as a VM on-premises
2. Create task with source and destination locations
3. DataSync handles encryption, compression, verification, and bandwidth throttling
4. Incremental transfers after initial sync

**Use cases**:
- Initial data lake seeding (TB to PB)
- Recurring backups to S3
- Cloud migration file transfers
- Archiving to Glacier via S3

> **DataSync vs S3 Transfer Acceleration**: DataSync is for bulk file system migration. Transfer Acceleration is for individual S3 uploads over long distances. Use DataSync for migrating file servers; use Transfer Acceleration for application-level uploads.

---

## 5. AWS Backup

AWS Backup provides **centralized, policy-driven backup management**:

| Feature | Description |
|---------|-------------|
| **Cross-service backups** | EC2, EBS, RDS, DynamoDB, EFS, Aurora, Storage Gateway |
| **Backup plans** | Define frequency, retention, and lifecycle (transition to cold storage) |
| **Organization integration** | Central backup policies across all accounts |
| **Vault locking** | WORM (Write Once Read Many) protection for compliance |
| **Cross-region copy** | Automatic replication to DR regions |
| **Point-in-time restore** | Restore to specific backup time |

> **AWS Backup vs Service-Native Backup**: AWS Backup provides centralized management and cross-service consistency. Service-native backups (RDS snapshots, EBS snapshots) exist independently. Use AWS Backup for governance; service-native for operational simplicity.

---

## 6. AWS Elastic Disaster Recovery (DRS)

DRS (formerly CloudEndure DR) provides **continuous replication for disaster recovery**:

- Replicates on-premises or AWS servers to a staging area in a DR region
- **RPO**: Seconds (continuous replication)
- **RTO**: Minutes (launch replicated instances with a click or automation)
- No compute cost in DR region during normal operation (only storage)

**DRS vs MGN**: Both use the same underlying replication technology. DRS is optimized for DR (frequent drills, automated failover, DR testing). MGN is optimized for migration (one-time cutover, application transformation).

---

## 7. Migration Architecture Patterns

### Pattern: Database Migration with DMS

```
On-Prem Oracle ──► DMS Replication Instance ──► Aurora PostgreSQL
        │                    │
        └── CDC ─────────────┘ (ongoing sync)

Cutover steps:
1. Stop application writes to source
2. Wait for DMS CDC to catch up (typically seconds)
3. Verify data consistency
4. Point application to target
5. Monitor for issues
```

### Pattern: Mainframe Replatforming

```
Mainframe COBOL/DB2 ──► AWS Micro Focus ──► Refactor to Java/PostgreSQL
                               │
                               └── Interim state: preserves COBOL while
                                   modernizing incrementally
```

### Pattern: VMware to AWS

```
On-Prem VMware ──► VMware Cloud on AWS (VMC) ──► Gradual workload migration
                       │                              to native EC2/ECS/EKS
                       └── Same vSphere tools,
                           minimal retraining
```

---

## 8. Architectural Decision Challenges

* **Scenario:** You need to migrate an active on-premises Oracle database to Aurora PostgreSQL with near-zero downtime.
  * **Design:** Use the AWS Schema Conversion Tool (SCT) to translate the schema, then use AWS Database Migration Service (DMS) with Full Load + CDC (Change Data Capture). Because SCT automates the schema translation, and DMS CDC keeps the target database synchronized with the source until you are ready to cut over.

* **Scenario:** You are executing a lift-and-shift of 500 on-premises VMs to AWS.
  * **Design:** Use AWS Application Migration Service (MGN). Because it performs continuous block-level replication of servers to a staging area, allowing for rapid cutover with minimal downtime.

* **Scenario:** You need to securely transfer a 100 TB SMB file share to Amazon FSx for Windows File Server.
  * **Design:** Use AWS DataSync. Because it is a managed service optimized for migrating file systems, natively handling encryption in transit, compression, and data integrity verification.

* **Scenario:** Your company requires an RPO of seconds and an RTO of minutes for on-premises servers but wants to minimize idle compute costs in AWS.
  * **Design:** Use AWS Elastic Disaster Recovery (DRS). Because it replicates to low-cost storage in a staging area and only provisions fully powered compute instances during a failover or drill.

---

## 9. Points to Remember

- **The 7 R's provide a migration strategy framework** — not every application should be re-architected.
- **MGN is the standard for lift-and-shift server migration** — continuous replication minimizes cutover risk.
- **DMS with CDC enables near-zero-downtime database migration** — always test transaction log compatibility.
- **SCT converts ~70-90% of schema automatically** — budget for manual conversion of complex stored procedures.
- **DataSync is for file system migration** — agent-based, compresses and encrypts in transit.
- **AWS Backup centralizes policy-driven backups** — use for governance across services and accounts.
- **DRS provides cost-effective DR** — pay only for storage in DR region until failover.
- **Always validate data consistency after migration** — checksums, row counts, and application-level validation.

---

## 10. Further Reading

For deeper coverage, CLI commands, and exam-specific trivia, see the detailed reference:

- **DR strategies, AWS Backup, DMS, MGN, DataSync**: [`AdvancedIdentity-DR-OtherServices.md`](../../detailed-reference/AdvancedIdentity-DR-OtherServices.md)
- **Snowball, DataSync, Storage Gateway**: [`StorageExtras-FSx-Snowball-Gateway.md`](../../detailed-reference/StorageExtras-FSx-Snowball-Gateway.md)

---

*Section 14 — Migration | Last Validated: 2026-05-10*
