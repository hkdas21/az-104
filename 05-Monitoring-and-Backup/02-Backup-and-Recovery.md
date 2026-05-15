# Backup and Recovery

## The Big Picture

Two related but distinct services answer "what if something is destroyed?":

| Service | Purpose | Recovery Point | Recovery Time |
|---------|---------|----------------|---------------|
| **Azure Backup** | Take periodic copies of VMs, files, DBs | Hours to days back | Minutes to hours |
| **Azure Site Recovery (ASR)** | Continuously replicate workloads to another region | Seconds to minutes | Minutes (failover) |

Use **Backup** for accidental deletion / corruption / ransomware. Use **Site Recovery** for region-level disasters.

---

## Real-World Analogy: Photos and Insurance

- **Azure Backup** = scanning your photo album every night and storing copies in a safe — if you lose the original, you go back to last night's scan
- **Site Recovery** = a sister photo studio in another city continuously receiving copies of every new photo — if the original studio burns down, you walk into the sister studio and keep working

---

## Recovery Services Vault (RSV)

The **Recovery Services Vault** is the management container for both Backup and Site Recovery. Think of it as the "safe" where copies live.

### Properties
- **Region-bound** — vault lives in one Azure region
- **Replication option** at create time:
  - **GRS** (default) — geo-redundant, recommended for cross-region restore
  - **LRS** — cheaper, single-region only
  - **ZRS** — zone-redundant inside one region
- **Soft delete** — deleted backups recoverable for 14 days (extendable). On by default for VMs.
- **Immutability** — vault can be made tamper-proof against ransomware

> **Backup vault** is a newer vault type used for Azure Disks, Blobs, and PostgreSQL backups. Different resource type, similar concept.

---

## Azure Backup

A managed service that backs up many workload types into a vault — no infra to run.

### What can be backed up

| Workload | Method |
|----------|--------|
| **Azure VM (full VM)** | Snapshot of all disks, application-consistent (Windows VSS, Linux pre/post scripts) |
| **Files inside an Azure VM (MARS agent)** | File/folder level from the VM's guest OS |
| **On-prem servers** | MARS (file/folder), Azure Backup Server / DPM (system-state, app-aware) |
| **Azure Files (share)** | Snapshot-based, per-share policies |
| **SQL in Azure VM** | Stream-based, log/full/diff |
| **SAP HANA in Azure VM** | Stream-based |
| **Azure Disk** | Incremental snapshots stored in a Backup vault |
| **Azure Blob** | Operational + vaulted backup of containers |
| **Azure Database for PostgreSQL** | Long-term retention via Backup vault |

### Backup policy
A **policy** defines:
- **Backup frequency** — daily, weekly, hourly (for SQL log)
- **Time** — when the backup runs
- **Retention** — daily, weekly, monthly, yearly retention buckets (GFS — Grandfather-Father-Son)
- **Instant Restore snapshots** — kept locally for 1–5 days for fast restore

### Restore options for VM backup
- **Create new VM** from a recovery point
- **Restore disks** to a resource group, then reattach
- **File-level restore** — mount the recovery point as a temporary drive on any machine, copy out specific files

---

## Backup Center / Backup Vault

The **Backup Center** is a unified hub for managing backups across vaults, subscriptions, and tenants. Use it for:
- At-a-glance success/failure dashboards
- Cross-subscription policy management
- Compliance reports

---

## Azure Site Recovery (ASR)

ASR **replicates workloads** to another region (or even on-prem ↔ Azure) so you can fail over when the primary fails.

### Replication scenarios
- **Azure → Azure** — VMs replicated to a paired or chosen region
- **On-prem VMware → Azure**
- **On-prem Hyper-V → Azure**
- **On-prem physical → Azure**

### How it works (Azure-to-Azure)
1. Enable replication on a VM → ASR copies disks to **cache storage** in source region, then to **target region** disks
2. **Continuous data replication** — RPO of seconds (typically <30s)
3. **Recovery points** are generated periodically (crash-consistent ~5 min, app-consistent ~hourly)
4. On disaster: **failover** spins up a VM in the target region from the latest recovery point

### Recovery Plans
Group VMs and orchestrate failover order:
- **Pre/post scripts** (e.g., reconfigure DNS)
- **Manual actions** (e.g., notify ops)
- **Sequence groups** (DB tier first, then app, then web)

### Test failover
Failover into an isolated VNet **without disrupting production replication**. Best practice: run quarterly.

### Failback
After the primary region recovers, replicate **back** to it and switch the active site.

### Limits
- ASR adds **~5–15% disk write overhead** for replication
- Some VM families/SKUs are not supported (check docs)
- **Azure Backup and ASR can coexist** on the same VM — common pattern

---

## Backup vs. Site Recovery — When to Use What

| Need | Service |
|------|---------|
| Recover from "I deleted that file last week" | Backup |
| Ransomware encrypted everything yesterday | Backup (with soft delete + immutability) |
| Whole region went offline | Site Recovery |
| Datacenter migration (cutover) | Site Recovery |
| Compliance: 7-year retention | Backup |
| RPO < 1 minute | Site Recovery |

> Use **both** for production. Backup gives you point-in-time recovery; ASR gives you geo-resilience.

---

## Hands-On: Enable VM Backup

```bash
# Create a Recovery Services Vault
az backup vault create \
  --resource-group rg-backup \
  --name rsv-prod \
  --location centralindia

# Set vault redundancy (default is GRS; this confirms it)
az backup vault backup-properties set \
  --resource-group rg-backup --name rsv-prod \
  --backup-storage-redundancy GeoRedundant

# Enable backup for a VM with the default daily policy
az backup protection enable-for-vm \
  --resource-group rg-backup \
  --vault-name rsv-prod \
  --vm $(az vm show -g rg-compute -n vm-web01 --query id -o tsv) \
  --policy-name DefaultPolicy

# Trigger an on-demand backup
az backup protection backup-now \
  --resource-group rg-backup \
  --vault-name rsv-prod \
  --container-name vm-web01 \
  --item-name vm-web01 \
  --retain-until 31-12-2026 \
  --backup-management-type AzureIaasVM
```

---

## Common Pitfalls

1. **Vault region mismatch.** A Recovery Services Vault must be **in the same region** as the VM you back up. (You restore *to* anywhere with GRS; the vault itself is regional.)
2. **No soft delete + ransomware** = backups can be deleted by attacker. Keep soft delete on.
3. **Forgetting to test failover.** "We have ASR" doesn't help if no one knows the runbook.
4. **Policy retention misalignment** with compliance (e.g., regulator requires 7 years; your policy keeps 1).
5. **Backup adjacent costs** — extended Instant Restore snapshots, GRS vault, Cross Region Restore — small choices, big bills.
6. **App-consistent backup needs VSS / pre-post scripts**, otherwise you only get crash-consistent.

---

## Quiz: Backup and Recovery

**1.** Which service is best for recovering from a **regional** outage?  
A. Azure Backup  B. Azure Site Recovery  C. Storage versioning  D. Soft delete

**2.** True or False: A Recovery Services Vault can back up a VM in any Azure region.

**3.** What is the **default** soft-delete retention for deleted backup data?  
A. 7 days  B. 14 days  C. 30 days  D. None

**4.** Which redundancy option is recommended for a vault that may need cross-region restore?  
A. LRS  B. ZRS  C. GRS  D. None

**5.** Application-consistent VM backup on **Windows** uses:  
A. Pre/post scripts  B. VSS  C. KQL  D. Snapshots only

**6.** Which is a **valid** restore option from a VM recovery point?  
A. Create a new VM  
B. Restore disks  
C. File-level recovery via mounted drive  
D. All of the above

**7.** ASR's typical RPO for Azure-to-Azure is:  
A. 24 hours  B. 1 hour  C. Seconds (~30s typical)  D. 1 week

**8.** True or False: Azure Backup and Azure Site Recovery cannot be used on the same VM.

**9.** The MARS agent backs up:  
A. Files and folders from a Windows machine  
B. Entire Hyper-V hosts  
C. SQL databases only  
D. Azure VMs only

**10.** A **Recovery Plan** in ASR is used to:  
A. Configure RBAC  
B. Group VMs and orchestrate failover order with scripts  
C. Define backup schedule  
D. Replace Action Groups

---

### Answers

1. **B** — Site Recovery for region failover.  
2. **False** — Vault is regional; back up VMs in the same region as the vault.  
3. **B** — 14 days.  
4. **C** — GRS for cross-region restore capability.  
5. **B** — VSS on Windows.  
6. **D** — All three are supported.  
7. **C** — Seconds-level RPO is typical for ASR.  
8. **False** — They can coexist and are often used together.  
9. **A** — MARS = files/folders agent.  
10. **B** — Recovery Plan orchestrates failover order with scripts.

---

**Congratulations — you finished the course!** 🎓

Next steps:
- Run through every quiz again (close the file, score yourself)
- Do **at least one full practice exam** under timed conditions
- Spend a weekend in a free Azure subscription building a hub-and-spoke with VMs, NSGs, a Storage account, and a backed-up VM. Hands-on beats reading every time.

Good luck on the exam!

---

← [Back to course home](../README.md)
