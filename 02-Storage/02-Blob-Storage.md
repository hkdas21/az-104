# Azure Blob Storage

## The Big Picture

Blob storage is **object storage** — a place for unstructured data of any kind: photos, videos, log files, backups, virtual machine disks, ML datasets. It's massively scalable, dirt-cheap when used well, and ridiculously durable (eleven 9's of durability).

If a workload needs to store a *lot* of data and access it via HTTP/S, it's almost always blob storage.

---

## Real-World Analogy: A Postal Storage Locker

Imagine a row of post-office lockers:
- The **post office** = your storage account
- Each **locker bank** (e.g., A, B, C) = a *container*
- Each **package** inside a locker = a *blob*
- The **locker number + package label** = the blob's URL
- Some packages are accessed daily (Hot), some monthly (Cool), some yearly (Cold), some "just keep it forever, I'll claim it eventually" (Archive)

---

## The Three Blob Types

| Blob type | Best for | Notes |
|-----------|---------|-------|
| **Block blob** | Files, documents, images, videos | The most common type. Made of blocks up to 4000 MiB |
| **Append blob** | Logs (write-only-append) | Optimized for appending, no random writes |
| **Page blob** | Random read/write disks (VHDs) | Backs unmanaged VM disks |

> Day-to-day, you'll work with **block blobs**. Append blobs show up for log scenarios. Page blobs are mostly hidden behind managed disks.

---

## Containers and Blobs

```
Storage Account: contoso2026
└── Container: invoices
    ├── 2026/01/inv-1001.pdf
    ├── 2026/01/inv-1002.pdf
    └── 2026/02/inv-1003.pdf
```

Container names:
- **3 to 63 characters**
- lowercase letters, digits, hyphens
- must start with a letter or digit
- cannot have consecutive hyphens

Blob names can be up to 1024 characters and can contain `/`, which clients render as folders even though they're just part of the name (in flat namespace).

---

## Access Tiers

This is **the** big cost lever in blob storage. Choose poorly and your bill explodes.

| Tier | Best for | Storage cost | Access cost | Min retention |
|------|----------|--------------|-------------|---------------|
| **Hot** | Frequently accessed | High | Low | None |
| **Cool** | Infrequently accessed | Medium | Medium | 30 days |
| **Cold** | Rarely accessed | Low | High | 90 days |
| **Archive** | Long-term retention, can wait hours to access | Lowest | Very high | 180 days |

> **Memory aid:** Cheaper to **store**, more expensive to **read**. The colder the tier, the colder your wallet feels when you read.

### Archive specifics
- Data is **offline** — you can't read it directly
- To access, you must **rehydrate** to Hot or Cool first (can take **several hours**)
- Two priorities: **Standard** (up to 15 hrs) or **High** (up to 1 hr, costs more)

### Setting tier
- **Account default tier** (Hot or Cool) — the tier new blobs inherit unless overridden
- **Per-blob tier** — override at the blob level
- **Lifecycle management policy** — automatically move blobs based on age / last access

---

## Lifecycle Management

A JSON-based policy that runs once a day and moves blobs between tiers (or deletes them) based on rules.

### Example policy

```json
{
  "rules": [
    {
      "name": "MoveToCoolThenArchive",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": [ "blockBlob" ],
          "prefixMatch": [ "logs/" ]
        },
        "actions": {
          "baseBlob": {
            "tierToCool":    { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete":        { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```

> **Exam trap:** Lifecycle rules apply to **block blobs** only by default. Page blobs and append blobs have limited support.

---

## Soft Delete and Versioning

These two features save you when an admin or app accidentally deletes data.

### Blob soft delete
- Deleted blobs are **kept in a soft-deleted state** for 1 to 365 days (you choose)
- During retention, you can undelete them
- Costs the same as a normal blob during retention

### Container soft delete
- Same idea but for entire containers (1 to 365 days)
- Required if you ever delete-and-recreate containers as part of automation

### Blob versioning
- Every change to a blob produces a **new version**
- The current version is the latest; previous versions are kept until deleted
- Combined with soft delete, gives you point-in-time recovery

> Best practice: enable **soft delete (7 days minimum)** and **versioning** on all production storage accounts.

---

## Immutable Blob Storage (WORM)

Required for compliance scenarios — SEC, FINRA, HIPAA. **Write Once, Read Many.** Once you set an immutability policy, even a Storage Account owner cannot delete or modify the blob until the policy expires.

### Two policy types
- **Time-based retention** — protected for N days from now
- **Legal hold** — protected indefinitely until the hold is removed

> **Exam trap:** A legal hold can be added/removed without affecting time-based retention. Both can co-exist; both must be cleared for the blob to be deletable.

---

## Object Replication

Asynchronous copy of blobs from a **source** account to a **destination** account. Used for disaster recovery scenarios beyond what GRS gives you (e.g., copying selectively, across subscriptions, or across regions you choose).

Requires:
- Both accounts must be **GPv2 or premium block blob**
- Both must have **versioning enabled**
- Both must have **change feed enabled** on the source

---

## AzCopy and Storage Explorer

Two tools you'll use a lot:

- **AzCopy** — command-line tool for high-throughput copy. Use for migrations, backups, large transfers.
- **Storage Explorer** — desktop GUI to browse and manage storage. Great for ad-hoc work.

```bash
# AzCopy basic upload
azcopy copy "C:\source\folder\*" \
  "https://contoso2026.blob.core.windows.net/uploads?<SAS>" \
  --recursive
```

---

## Hands-On: Working with Blobs (CLI)

```bash
# Create a container
az storage container create \
  --account-name contoso2026 \
  --name invoices \
  --auth-mode login

# Upload a blob
az storage blob upload \
  --account-name contoso2026 \
  --container-name invoices \
  --name jan-invoices.pdf \
  --file ./jan-invoices.pdf \
  --auth-mode login

# Set the tier
az storage blob set-tier \
  --account-name contoso2026 \
  --container-name invoices \
  --name jan-invoices.pdf \
  --tier Cool \
  --auth-mode login

# Generate a download URL with SAS (1 hour)
az storage blob generate-sas \
  --account-name contoso2026 \
  --container-name invoices \
  --name jan-invoices.pdf \
  --permissions r \
  --expiry $(date -u -d "1 hour" '+%Y-%m-%dT%H:%MZ') \
  --auth-mode login \
  --as-user
```

---

## Common Pitfalls

1. **Setting the tier without considering the minimum retention.** Move to Cool, change your mind in 5 days, get charged a *deletion* fee.
2. **Using Archive for things you'll need this week.** Rehydration is hours.
3. **Lifecycle policies based on `lastAccessTime` require Last Access Tracking to be enabled** — it's off by default.
4. **Forgetting that soft delete is independent for blobs and containers.** Enable both for full coverage.
5. **Object Replication only copies new writes.** Existing data needs a one-time AzCopy.

---

## Quiz: Blob Storage

**1.** Which tier has the **lowest** storage cost but **highest** access cost?  
A. Hot  B. Cool  C. Cold  D. Archive

**2.** True or False: You can read a blob in the Archive tier directly with a GET request.

**3.** What is the minimum retention period for a blob in the **Cool** tier before moving it without an early-deletion charge?  
A. 7 days  B. 30 days  C. 90 days  D. 180 days

**4.** Which blob type is best for a log file that only ever has new lines added to it?  
A. Block  B. Append  C. Page  D. Snapshot

**5.** Which two features should you enable to get point-in-time recovery for blobs?  
A. RA-GRS + change feed  
B. Soft delete + versioning  
C. Immutability + legal hold  
D. Lifecycle + tags

**6.** A user accidentally deleted a container. Soft delete for **containers** is enabled with 14 days retention. What can you do?  
A. Nothing — only blob soft delete protects this  
B. Restore the container from the deleted state  
C. Restore from GRS secondary  
D. Restore from a snapshot

**7.** What is the maximum size of a single block blob (in 2025)?  
A. 200 GB  B. 4.75 TB  C. 190.7 TiB (~200 TB)  D. Unlimited

**8.** You apply a 5-year time-based retention policy on a blob. The Storage Account owner tries to delete it next week. What happens?  
A. Delete succeeds  B. Delete fails  C. Delete succeeds with audit log  D. Delete is queued for the future

**9.** Which is required to use lifecycle rules based on **last access time**?  
A. Premium tier  B. Hierarchical namespace  C. Last access time tracking enabled  D. RA-GRS

**10.** Object replication requires both accounts to have which features enabled on the source?  
A. Versioning + change feed  B. Soft delete + lifecycle  C. RA-GRS  D. Hierarchical namespace

---

### Answers

1. **D** — Archive: cheapest to store, most expensive to read.  
2. **False** — Must rehydrate first.  
3. **B** — 30 days for Cool. (Cold is 90, Archive is 180.)  
4. **B** — Append blob.  
5. **B** — Soft delete + versioning.  
6. **B** — Container soft delete protects you.  
7. **C** — A block blob can be ~190.7 TiB with 4000 MiB blocks × max blocks.  
8. **B** — Immutability blocks even owners.  
9. **C** — Last access tracking must be enabled.  
10. **A** — Versioning + change feed on source; versioning on destination.

---

**Next up:** [03-Azure-Files.md](./03-Azure-Files.md)
