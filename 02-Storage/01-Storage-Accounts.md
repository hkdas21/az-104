# Storage Accounts

## The Big Picture

A **Storage Account** is the top-level container for all Azure Storage data. Every blob, file share, queue, and table you create lives inside one. It also defines the **redundancy**, **performance tier**, and **security boundary** for everything inside.

You can think of a Storage Account as a **bank account number for data**. Once you have it, you can deposit different kinds of valuables (photos, files, queues, key-value records) into different "vaults" inside, but they all live under that one account.

---

## Real-World Analogy: A Self-Storage Facility

Imagine renting space at a self-storage facility:

- **Storage Account** = your rental contract with the facility (one address, one access code)
- **Containers / File Shares / Queues / Tables** = different rooms inside your unit
- **Blobs / Files / Messages / Entities** = the actual stuff inside the rooms
- **Region** = the city the facility is located in
- **Redundancy (LRS / ZRS / GRS)** = whether the facility has just one warehouse, multiple buildings on the same campus, or sister facilities in another city
- **Access tier (Hot / Cool / Cold / Archive)** = how quickly you can pull stuff out — front shelf vs. back of the unit vs. deep cold storage

---

## Storage Account Types

This is one of those topics where the exam *will* ask "Which type do I pick for X workload?" Memorize the matrix.

| Type | What it supports | Common use |
|------|------------------|------------|
| **Standard general-purpose v2 (GPv2)** | Blob, Files, Queues, Tables, Disks | The default for most workloads |
| **Premium block blobs** | Block blobs only | High transactions / low-latency object workloads |
| **Premium page blobs** | Page blobs only | Unmanaged disks (rare in 2025) |
| **Premium file shares** | Azure Files only | High-perf file workloads, especially SMB |
| **BlockBlobStorage / FileStorage** | Specialized | Niche / legacy variants |
| **Standard general-purpose v1 (GPv1)** | Legacy | Don't choose for new workloads |

> **Rule of thumb:** Default to **Standard GPv2** unless you have a specific reason. It supports everything and is fine for most workloads.

---

## Performance Tiers

- **Standard** — backed by HDDs. Cheap, plenty of throughput for general workloads.
- **Premium** — backed by SSDs. Lower latency, higher cost. Use for transactional / chatty workloads.

The premium tier is **per-data-service**: you pick Premium for blobs, OR for files, OR for page blobs — not all in one account.

---

## Replication / Redundancy Options

Azure offers six replication modes. **Memorize the comparison.**

| Replication | Copies | Across Zones? | Across Regions? | Read access from secondary? |
|-------------|--------|---------------|------------------|------------------------------|
| **LRS** (Locally redundant) | 3 | ❌ (same DC) | ❌ | N/A |
| **ZRS** (Zone-redundant) | 3 | ✅ across 3 AZs | ❌ | N/A |
| **GRS** (Geo-redundant) | 6 | ❌ in primary, ❌ in secondary | ✅ | ❌ |
| **GZRS** (Geo-zone-redundant) | 6 | ✅ in primary, ❌ in secondary | ✅ | ❌ |
| **RA-GRS** (Read-access GRS) | 6 | ❌ | ✅ | ✅ |
| **RA-GZRS** (Read-access GZRS) | 6 | ✅ in primary, ❌ in secondary | ✅ | ✅ |

### Decoding the names
- **L** = Local (single datacenter)
- **Z** = Zone (across AZs in same region)
- **G** = Geo (across regions)
- **RA** = Read-Access to the secondary region during normal operation

> **Exam trap:** With GRS/GZRS (no RA), you can only read from the secondary **after Microsoft initiates a failover**. With RA-GRS/RA-GZRS, you can read from the secondary endpoint anytime via a different URL (`<account>-secondary.blob.core.windows.net`).

### Picking redundancy
- **LRS** — cheapest. Good for non-critical, easily replaceable data.
- **ZRS** — protect against datacenter outages within a region.
- **GRS** — protect against entire region outages.
- **GZRS** — most resilient. Region + AZ protection.
- **RA-** — when you need to actively read from the secondary location for analytics/DR testing.

---

## Naming and Endpoints

Storage account names must be:
- **3 to 24 characters**
- Lowercase letters and numbers only
- **Globally unique** across all of Azure (because they map to a public DNS name)

Example endpoints (assuming account name `contoso2026`):
- Blob: `https://contoso2026.blob.core.windows.net`
- File: `https://contoso2026.file.core.windows.net`
- Queue: `https://contoso2026.queue.core.windows.net`
- Table: `https://contoso2026.table.core.windows.net`
- DFS (Data Lake Gen2): `https://contoso2026.dfs.core.windows.net`

---

## Hierarchical Namespace (HNS) — Azure Data Lake Storage Gen2

A flag you set when creating the account: **"Enable hierarchical namespace."**

Without HNS, blob storage is *flat* — folders are simulated by prefixes in the blob name.

With HNS, blob storage gains real folders, atomic rename, POSIX-like permissions (ACLs). Required if you want to use this storage account as a **data lake** with Synapse, HDInsight, Databricks, etc.

> **Exam trap:** HNS can be enabled only at *creation time*. You cannot toggle it on for existing accounts (well — there's a migration tool, but treat it as a one-way decision in the exam).

---

## Hands-On: Create a Storage Account via CLI

```bash
# Create the resource group first
az group create --name rg-storage-demo --location centralindia

# Create the storage account
az storage account create \
  --name contoso2026demo \
  --resource-group rg-storage-demo \
  --location centralindia \
  --sku Standard_ZRS \
  --kind StorageV2 \
  --access-tier Hot \
  --allow-blob-public-access false \
  --min-tls-version TLS1_2

# List storage accounts
az storage account list --output table

# Get the access keys
az storage account keys list --account-name contoso2026demo --resource-group rg-storage-demo
```

---

## Common Pitfalls

1. **Choosing GPv1 by accident.** GPv2 has nearly all features at the same price.
2. **Mixing premium types.** "Premium block blob" doesn't support page blobs. Different SKU per workload.
3. **Underestimating egress cost on GRS reads.** Reading the secondary in another region = cross-region traffic = $$.
4. **Using account name with special characters.** Lowercase letters and digits only.
5. **Forgetting that GZRS is GA only in select regions.** Always verify region availability.

---

## Quiz: Storage Accounts

**1.** What is the maximum length of a storage account name?  
A. 15  B. 20  C. 24  D. 32

**2.** Which redundancy gives you copies in 3 availability zones in the same region AND a copy in a paired region?  
A. LRS  B. ZRS  C. GRS  D. GZRS

**3.** True or False: You can enable hierarchical namespace on an existing storage account through the portal toggle.

**4.** Which storage account kind supports all data services?  
A. BlobStorage  B. FileStorage  C. StorageV2 (GPv2)  D. Storage (GPv1)

**5.** Which redundancy option is the cheapest?  
A. LRS  B. ZRS  C. GRS  D. RA-GZRS

**6.** Your application needs sub-millisecond latency on a queue workload. Which choice fits best?  
A. Standard GPv2 + Hot tier  
B. Premium Block Blob  
C. Premium Files  
D. Standard with Premium SSD disk attached

**7.** You enable RA-GRS. Where can your app read from while everything is healthy?  
A. Only the primary  
B. Only the secondary  
C. Both primary and secondary  
D. None — you must initiate failover first

**8.** True or False: Blob storage paths use forward slashes that are *real* folders without hierarchical namespace.

**9.** Which characters are allowed in a storage account name?  
A. lowercase letters, digits, hyphens  
B. lowercase letters and digits only  
C. uppercase letters, lowercase letters, digits  
D. any character

**10.** You want disaster recovery from a regional outage but don't need read access to the secondary unless there's a failover. What's the cheapest option that fits?  
A. LRS  B. ZRS  C. GRS  D. RA-GRS

---

### Answers

1. **C** — 24 chars max.  
2. **D** — GZRS combines zones (primary) + geo replication.  
3. **False** — HNS is set at creation. Migration is possible but not a portal toggle.  
4. **C** — GPv2 supports everything.  
5. **A** — LRS is cheapest.  
6. **A** — Queue workloads work fine on Standard GPv2; queues don't have a Premium variant. (Premium Block Blob is for blobs, Premium Files for SMB shares.)  
7. **C** — RA = read access from both, anytime.  
8. **False** — Without HNS, "folders" are just name prefixes.  
9. **B** — Lowercase letters and digits only.  
10. **C** — GRS gives you region failover at the cheapest cross-region price; you only need RA-GRS if you read the secondary during normal ops.

---

**Next up:** [02-Blob-Storage.md](./02-Blob-Storage.md)
