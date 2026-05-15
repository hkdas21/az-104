# Azure Files

## The Big Picture

Azure Files is a fully managed **file share in the cloud** that supports the **SMB** and **NFS** protocols. Mount it from Windows, Linux, macOS, on-prem servers, or Azure VMs — exactly like you'd mount a network drive.

Best of all? It can replace your traditional file servers without you rewriting any apps. They just see a network drive.

---

## Real-World Analogy: A Shared Office Drive — But Owned by Microsoft

Imagine your office's `Z:\` drive that everyone maps to. The IT admin maintains the file server, deals with disk failures, schedules backups, panics about ransomware, and stays late on patch nights.

Azure Files is the **same Z:\ drive, but Microsoft is your IT admin**. You don't worry about hardware, replication, patching, or capacity ceilings.

---

## Use Cases

- **Replace on-prem file servers** — lift-and-shift the SMB share
- **Shared application configuration** — multiple VMs read the same config files
- **Dev/test tools** — shared dev environments
- **Container persistent storage** — mount Azure Files as a volume in AKS
- **Hybrid scenarios with Azure File Sync** — keep a "cache" on-prem and tier the rest to Azure

---

## Tiers

Azure Files has its own tier system distinct from blob.

| Tier | Backed by | Use case | Account type |
|------|-----------|---------|--------------|
| **Premium** | SSD | Latency-sensitive, transaction-heavy workloads | FileStorage account |
| **Transaction Optimized** (standard) | HDD | High transactions, latency not critical | GPv2 |
| **Hot** (standard) | HDD | Active general-purpose data | GPv2 |
| **Cool** (standard) | HDD | Archive/online with infrequent access | GPv2 |

> **Note:** Premium uses *provisioned* capacity (you pay for what you allocate, not what you use). Standard uses *consumption* (pay for what you store).

---

## Protocols

| Protocol | Versions | Auth options |
|----------|---------|--------------|
| **SMB** | 2.1, 3.0, 3.1.1 | Storage account key, AD DS, Entra Domain Services, Entra Kerberos (for hybrid users) |
| **NFS** | 4.1 | Network-only auth (no identity-based) — only on Premium FileStorage accounts |

> **Exam trap:** NFS shares cannot use storage account keys or AD-based auth — they rely entirely on **network restrictions** (private endpoints, service endpoints, or VNet rules).

---

## Mounting an Azure File Share

### Windows
```powershell
# Get the mount script from the portal, or:
$connectTestResult = Test-NetConnection -ComputerName contoso2026.file.core.windows.net -Port 445
if ($connectTestResult.TcpTestSucceeded) {
    cmd.exe /C "cmdkey /add:`"contoso2026.file.core.windows.net`" /user:`"localhost\contoso2026`" /pass:`"<storage-account-key>`""
    New-PSDrive -Name Z -PSProvider FileSystem `
        -Root "\\contoso2026.file.core.windows.net\engineering" -Persist
}
```

### Linux (SMB)
```bash
sudo mkdir /mnt/engineering
sudo mount -t cifs //contoso2026.file.core.windows.net/engineering /mnt/engineering \
  -o vers=3.1.1,username=contoso2026,password=<key>,dir_mode=0777,file_mode=0777
```

### Linux (NFS)
```bash
sudo mount -t nfs contoso2026.file.core.windows.net:/contoso2026/engineering \
  /mnt/engineering -o vers=4,minorversion=1,sec=sys,nconnect=4
```

> **Exam trap:** SMB requires **port 445** outbound. Many ISPs and corporate firewalls block 445. Test with `Test-NetConnection`.

---

## Identity-Based Auth for SMB

Three options for "no shared key" authentication:

| Option | Best for | Requires |
|--------|---------|----------|
| **On-premises AD DS** | Hybrid orgs with existing AD | AD DS sync'd to Entra ID via Entra Connect |
| **Entra Domain Services** | Cloud-only orgs that still need Kerberos | Entra Domain Services managed domain |
| **Entra Kerberos for hybrid identities** | FSLogix profile containers, AVD | Hybrid users |

Once configured, you assign these RBAC roles for SMB share access:
- **Storage File Data SMB Share Reader** — read-only
- **Storage File Data SMB Share Contributor** — read/write
- **Storage File Data SMB Share Elevated Contributor** — read/write/manage NTFS permissions

---

## Snapshots

Azure Files supports **share-level snapshots** — point-in-time read-only copies.

- Up to **200 snapshots per share**
- Incremental — only changes after the last snapshot consume space
- Used by **Azure Backup** under the hood
- Can be triggered manually or scheduled

> Snapshots are share-level. There is no individual file snapshot.

---

## Azure File Sync

The killer feature that turns Azure Files into a **distributed file system**.

### How it works
1. You install the **Azure File Sync agent** on Windows Server(s)
2. The server's local folder becomes a **server endpoint**
3. The Azure file share is the **cloud endpoint**
4. They sync bidirectionally
5. **Cloud Tiering** keeps only "hot" files locally; cold files exist as **placeholders** that fetch on access

### Why this is powerful
- Branch offices keep low-latency local file access
- Central office has a single source of truth in the cloud
- Disk space at the branch is bounded — File Sync evicts cold files automatically
- If the branch server dies, you reinstall, point to the cloud endpoint, and the data flows back

### Components
- **Storage Sync Service** — the top-level resource in Azure
- **Sync Group** — a logical container linking one cloud endpoint and many server endpoints
- **Cloud Endpoint** — the Azure file share
- **Server Endpoint** — a folder on a Windows Server
- **Registered Server** — the server itself

---

## Backup with Azure Backup

Recovery Services Vault → "File Share" workload → schedule daily backups → keep snapshots for up to 10 years.

Restore options:
- **Original location** — overwrite or skip
- **Alternate location** — restore to another file share
- **Item-level restore** — pick specific files, restore only those

---

## Common Pitfalls

1. **Port 445 blocked.** Test it. Most ISPs block it; use VPN/ExpressRoute or Azure VMs.
2. **Picking Standard for a chatty workload.** Premium IOPS scaling will save you.
3. **Not enabling soft delete on file shares.** Default 7 days is a lifesaver.
4. **Mixing identity providers.** Pick AD DS, Entra Domain Services, OR Entra Kerberos — not multiple.
5. **NFS on Standard accounts.** Doesn't exist. NFS = Premium FileStorage only.

---

## Quiz: Azure Files

**1.** Which protocol is supported on **Standard** Azure file shares?  
A. SMB only  B. NFS only  C. SMB and NFS  D. Neither

**2.** Which port does SMB require to be open outbound from the client?  
A. 22  B. 443  C. 445  D. 1433

**3.** True or False: NFS file shares can be authenticated using the storage account key.

**4.** What is the maximum number of snapshots per file share?  
A. 50  B. 100  C. 200  D. Unlimited

**5.** You want branch office users to have low-latency access to files but a central authoritative copy in the cloud. Which feature?  
A. Object Replication  B. Azure File Sync  C. RA-GZRS  D. AzCopy

**6.** Which RBAC role grants read/write access to an Azure file share over SMB with identity-based auth?  
A. Storage File Data SMB Share Reader  
B. Storage File Data SMB Share Contributor  
C. Storage Account Contributor  
D. Reader

**7.** Premium file shares are billed based on:  
A. Actual data stored  
B. Provisioned capacity  
C. Number of operations only  
D. Egress only

**8.** Which storage account kind supports NFS 4.1 file shares?  
A. StorageV2 (GPv2)  B. BlobStorage  C. FileStorage (Premium)  D. Storage (GPv1)

**9.** True or False: Azure File Sync can tier old files to the cloud and leave only placeholders on the local server.

**10.** A user can mount an SMB share by URL but the connection times out. Most likely cause?  
A. Wrong storage account name  
B. Port 445 blocked outbound  
C. NFS not enabled  
D. The user lacks Owner role

---

### Answers

1. **A** — Standard supports SMB only. NFS is Premium-only.  
2. **C** — 445.  
3. **False** — NFS uses network-only auth.  
4. **C** — 200.  
5. **B** — Azure File Sync.  
6. **B** — SMB Share Contributor.  
7. **B** — Provisioned capacity model.  
8. **C** — Premium FileStorage account.  
9. **True** — Cloud tiering with placeholders.  
10. **B** — Port 445 is the classic culprit.

---

**Next up:** [04-Storage-Security.md](./04-Storage-Security.md)
