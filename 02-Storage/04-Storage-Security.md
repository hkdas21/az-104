# Storage Security: Keys, SAS, Network, and Encryption

## The Big Picture

Storage security has **four layers** you'll be asked about repeatedly:

1. **Authentication** — who is asking? (keys vs. Entra ID vs. SAS)
2. **Authorization** — what can they do? (RBAC + ACLs)
3. **Network** — where can they ask from? (firewall, private endpoints)
4. **Encryption** — is the data scrambled at rest and in transit?

Layer them like onions. Don't rely on any single one.

---

## Real-World Analogy: A Bank Vault

Imagine a bank vault holding gold:
- **Identity check at the front door** — authentication (keys / SAS / Entra ID)
- **List of "this person can open which safe deposit box"** — authorization (RBAC, ACLs)
- **The vault is on the bank's private floor, no public entry** — network rules / private endpoints
- **The boxes themselves are inside locked metal containers** — encryption at rest
- **Money in transit is in armored cars** — encryption in transit (HTTPS)

Skip any one layer and you're vulnerable.

---

## Authentication Methods

### 1. Storage Account Keys
Each storage account has **two 512-bit keys** (key1, key2). Anyone with a key has **full control**. Treat them like the master key to your house.

- Rotate by **regenerating** keys (alternate between key1 and key2 to avoid downtime)
- Store keys in **Azure Key Vault**, never in source code
- Disable **shared key access** if you can — force everyone onto Entra ID

### 2. Shared Access Signatures (SAS)
A signed URL that grants **time-limited, scoped** access. Three types:

| Type | Signed by | Best for | Revocable? |
|------|-----------|---------|------------|
| **User delegation SAS** | Entra ID credentials | Modern, recommended | Yes (revoke the user) |
| **Service SAS** | Storage account key | Per-resource (one container/blob/queue) | Only by rotating the key |
| **Account SAS** | Storage account key | Account-wide operations | Only by rotating the key |

A SAS URL looks like:
```
https://contoso2026.blob.core.windows.net/invoices/jan.pdf?sv=2024-11-04&ss=b&srt=co&sp=r&se=2026-05-03T12:00:00Z&sig=...
```

Decoding:
- `sv` = service version
- `ss` = signed services (b/q/t/f)
- `srt` = resource types
- `sp` = permissions (r/w/d/l/a/c/u)
- `se` = expiry
- `sig` = the actual signature (HMAC of all the above)

### Stored Access Policy
A **server-side template** for SAS — defines start/expiry/permissions. The SAS just references the policy by name. To revoke a SAS, **delete or modify the policy** — no need to rotate keys.

> **Best practice:** Use **user delegation SAS** when possible. They're tied to an Entra identity, can be revoked by disabling the user, and don't require sharing storage keys.

### 3. Entra ID (OAuth)
Apps and users sign in with their Entra credentials, get an OAuth token, and present it to storage. Authorization is via **RBAC data-plane roles**:
- Storage Blob Data Reader
- Storage Blob Data Contributor
- Storage Blob Data Owner
- Storage File Data SMB Share Reader/Contributor/...
- Storage Queue Data Reader/Contributor/...
- Storage Table Data Reader/Contributor

> **Best practice:** Disable storage account key access entirely (`Allow storage account key access = No`) and force all clients onto Entra ID + RBAC.

### 4. Anonymous (Public) Access
Containers can be configured to allow public anonymous reads. **Disable at the account level** unless you have a specific public-website use case.

---

## Network Security

### Firewalls (Networking blade)
- **Public access enabled** (default) — anyone with credentials can reach the public endpoint
- **Selected networks** — only specified VNets/subnets and IP ranges
- **Disabled** — only private endpoints can connect

### Private Endpoints
Best practice for production. A private IP in your VNet that maps to the storage account.

- Each private endpoint is **per data service** (blob, file, queue, table separately)
- DNS automatically resolves to the private IP via **Private DNS zones**
- Public endpoint can be disabled entirely

### Service Endpoints
A simpler, older pattern. Adds a route from a subnet to Azure Storage so the traffic stays on Microsoft's backbone — but the storage still has a public IP. Used to be the go-to before private endpoints; now mostly considered legacy.

> **Exam trap:** Private endpoint = private IP in your VNet (preferred). Service endpoint = traffic over backbone but still public IP on storage.

---

## Encryption

### Encryption at rest
- **Storage Service Encryption (SSE)** — automatic, can't be disabled. AES-256.
- Two key options:
  - **Microsoft-managed keys (default)** — easy, free, Microsoft handles rotation
  - **Customer-managed keys (CMK)** — your key in Azure Key Vault. You rotate, you control. Required for some compliance scenarios.
  - **Customer-provided keys (CPK)** — only for blob, supplied per request

### Encryption in transit
- **Secure transfer required** — flag on the storage account that forces HTTPS. Enabled by default.
- **Minimum TLS version** — set to TLS 1.2 (or 1.3 where supported)

### Infrastructure encryption
A double-encryption mode. Encrypt twice with two different keys at the service and infrastructure levels. Costs slightly more, used for high-compliance scenarios.

---

## Key Management Best Practices

1. **Disable shared key access** — force Entra ID
2. **Rotate keys regularly** if you must use them
3. **Store keys in Key Vault** with rotation policies
4. **Use Managed Identities** in your apps — no secrets to store
5. **Use SAS with stored access policies** when sharing externally so you can revoke

---

## Hands-On: Tighten Down a Storage Account

```bash
# Disable shared key access
az storage account update \
  --name contoso2026 \
  --resource-group rg-storage-demo \
  --allow-shared-key-access false

# Disable public network access
az storage account update \
  --name contoso2026 \
  --resource-group rg-storage-demo \
  --public-network-access Disabled

# Set minimum TLS
az storage account update \
  --name contoso2026 \
  --resource-group rg-storage-demo \
  --min-tls-version TLS1_2

# Disable anonymous blob access
az storage account update \
  --name contoso2026 \
  --resource-group rg-storage-demo \
  --allow-blob-public-access false
```

---

## Common Pitfalls

1. **SAS tokens leaked in logs.** They go in URLs — easy to log accidentally. Use HEADER-based auth where possible.
2. **Long-expiry SAS for "convenience."** A 10-year SAS = a 10-year liability. Keep them short and use stored access policies.
3. **Forgetting to deny public network access** when private endpoints are configured. Both must be addressed.
4. **Mixed RBAC + key access.** Disable the keys to enforce Entra-only.
5. **Customer-managed key without proper Key Vault permissions.** If the storage account loses access to the KV key, all data becomes inaccessible. Configure soft delete and purge protection on the vault.

---

## Quiz: Storage Security

**1.** Which SAS type is signed using **Entra ID credentials** rather than the storage account key?  
A. Account SAS  B. Service SAS  C. User delegation SAS  D. None

**2.** True or False: A Service SAS can be revoked instantly without rotating the storage account keys.

**3.** Which RBAC role lets a user read blob *contents* (not just metadata)?  
A. Reader  B. Storage Account Contributor  C. Storage Blob Data Reader  D. Owner

**4.** What's the minimum TLS version recommended on a storage account in 2025?  
A. TLS 1.0  B. TLS 1.1  C. TLS 1.2  D. SSL 3.0

**5.** Which is the **most secure** way to give an Azure App Service access to a storage account?  
A. Storage account key in app settings  
B. Long-lived SAS token in app settings  
C. System-assigned managed identity + RBAC data role  
D. Anonymous container access

**6.** Which network feature gives the storage account a **private IP inside your VNet**?  
A. Service endpoint  B. Private endpoint  C. Application Gateway  D. NAT Gateway

**7.** Which is true about **stored access policies**?  
A. They generate a SAS automatically on first request  
B. They are server-side templates SAS references; modifying them can revoke the SAS  
C. They replace storage keys  
D. They work only with NFS shares

**8.** True or False: Storage Service Encryption can be disabled if you don't want it.

**9.** A SAS URL contains `sp=rwl`. What does this mean?  
A. Read, write, list  B. Read, write, lock  C. Replicate, write, lifecycle  D. Random, weighted, low

**10.** You use customer-managed keys (CMK) and accidentally delete the Key Vault key. What happens?  
A. Storage continues to work for 24 hours  
B. Storage data becomes inaccessible immediately  
C. Microsoft auto-restores the key  
D. The storage account is deleted

---

### Answers

1. **C** — User delegation SAS is signed by Entra ID.  
2. **False** — Service SAS revocation requires rotating the key (or using a stored access policy).  
3. **C** — Data-plane role required.  
4. **C** — TLS 1.2 minimum.  
5. **C** — Managed identity + RBAC = no secrets to leak.  
6. **B** — Private endpoint.  
7. **B** — Server-side template; modifying or deleting them can revoke active SAS.  
8. **False** — SSE is always on, can't be disabled.  
9. **A** — Read, Write, List.  
10. **B** — Without the key, data is inaccessible. (This is why Key Vault soft delete + purge protection are mandatory.)

---

**You've completed Module 2!** 🎉

Continue to [Module 3 — Compute](../03-Compute/README.md). Time to start building VMs.
