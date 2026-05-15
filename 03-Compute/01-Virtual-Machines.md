# Azure Virtual Machines

## The Big Picture

A **virtual machine** in Azure is a software-defined computer with its own CPU, RAM, OS, disks, and network card — running on Microsoft's massive fleet of physical servers. You get full control, just like an on-prem server, except you don't have to buy hardware, plug it in, or replace dead disks.

VMs are **IaaS** (Infrastructure as a Service): Microsoft handles the hardware and hypervisor; you handle the OS, patches, software, and configuration.

---

## Real-World Analogy: Renting a Furnished Apartment

You rent an apartment in a managed building:
- **Building (datacenter)** — owned and maintained by the landlord (Microsoft)
- **Apartment (VM)** — your private space
- **Furniture (OS image)** — pre-installed; you can swap it
- **Rent (compute price)** — you pay only while you're in residence
- **Utilities** — water (network bandwidth), electricity (CPU/memory) metered separately
- **The lease (VM lifecycle)** — start, stop, deallocate, delete

---

## Anatomy of a VM

A VM is actually **several Azure resources** stitched together:

```
Virtual Machine
├── Network Interface (NIC)
│   └── associated with → VNet + Subnet, NSG (optional), Public IP (optional)
├── OS Disk (managed disk)
├── Data Disks (0 or more managed disks)
├── (Optional) Diagnostics Storage Account
└── (Optional) Boot Diagnostics
```

When you "create a VM" in the portal, Azure stitches all these together for you. Behind the scenes, each is its own resource you can manage independently.

---

## VM Sizes (SKUs)

VM sizes are organized by **family letter** with a use case:

| Family | Optimized for | Examples |
|--------|--------------|----------|
| **A** | Entry-level, dev/test | A0 — Av2 |
| **B** | Burstable workloads | B1s, B2ms |
| **D** | General purpose | D2s_v5, D8s_v5 |
| **E** | Memory-optimized | E2s_v5, E16s_v5 |
| **F** | Compute-optimized | F2s_v2 |
| **G / M** | Massive memory | M128s |
| **L** | Storage-optimized | L8s_v3 |
| **N** | GPU (AI, graphics) | NC6s_v3, ND |
| **H** | High-perf computing | H16r |

### Decoding a SKU like `Standard_D8s_v5`
- `D` — general purpose family
- `8` — 8 vCPUs
- `s` — supports premium SSDs
- `v5` — version 5

> **Exam tip:** You don't need to memorize every size. You DO need to know which family fits which workload (memory-heavy → E, compute-heavy → F, GPU → N).

### Burstable B-series
Different model. You earn **CPU credits** when CPU usage is below baseline (e.g., 20%) and burn them when bursting above it. Great for low-baseline workloads (small web servers, dev VMs).

### Spot VMs
- Up to **90% discount**
- Azure can **evict** them at any time with 30-second notice
- Use for **interruptible workloads** — batch jobs, dev/test, encoders

---

## Managed Disks

A **managed disk** is a virtual hard disk where Azure handles the storage account, replication, and capacity behind the scenes. (You don't manage the storage account; Azure does.)

### Disk types

| Type | Tech | Use case |
|------|------|----------|
| **Standard HDD** | HDD | Backup, archive, low-IOPS |
| **Standard SSD** | SSD (lower throughput) | Web servers, light prod workloads |
| **Premium SSD** | SSD | Most prod databases, heavy IO |
| **Premium SSD v2** | SSD (newer) | Best price/perf for high IOPS |
| **Ultra Disk** | NVMe-class | Mission-critical, sub-ms latency |

### Disk sizing
Disks come in fixed sizes (e.g., P10, P20, P30, ...). Each tier has IOPS and throughput limits. Larger = more IOPS, but you pay for the whole disk.

### Bursting
- **Disk-level bursting** — small disks (P20 and below) can burst occasionally
- **VM-level bursting** — some VM SKUs allow temporary burst across all attached disks

### Encryption
- **SSE (Storage Service Encryption)** — server-side, default, free
- **Azure Disk Encryption (ADE)** — BitLocker (Windows) / dm-crypt (Linux), key in Key Vault
- **Encryption at host** — encrypts cache + OS + data + temp on the host machine
- **Confidential VMs** — full memory encryption (specialized SKUs)

---

## Disk Snapshots and Images

### Snapshots
Point-in-time copy of a managed disk. Used for:
- Backup before major changes
- Cloning a VM
- Cross-region copy (snapshot then copy then create disk)

### Images
A reusable template combining one or more disks. Two types:
- **Managed Image** — older, single-region
- **Azure Compute Gallery (formerly Shared Image Gallery)** — modern, supports versioning, replication to many regions, RBAC sharing

> **Best practice:** Use Azure Compute Gallery for production image management.

---

## VM Lifecycle States

| State | Billed? | Notes |
|-------|---------|-------|
| **Running** | ✅ Compute + storage | Working as expected |
| **Stopped (from inside the OS)** | ✅ Compute + storage | OS is off, but the VM still has reserved hardware |
| **Stopped (deallocated)** | Storage only | Hardware released — public IP may change too |
| **Starting / Stopping** | Transitional | Brief |

> **Exam trap:** Shutting down inside Windows ≠ deallocated. To stop billing for compute, **deallocate from Azure** (`az vm deallocate`). The portal's "Stop" button does deallocate; the OS shutdown does not.

---

## Connecting to a VM

### Windows
- **RDP** on port 3389 (default)
- **Azure Bastion** — RDP/SSH through the portal, no public IP, no open ports
- **Just-in-Time (JIT) access** — open RDP/SSH only when needed via Defender for Cloud

### Linux
- **SSH** on port 22
- Same Bastion / JIT options
- SSH key authentication is recommended over passwords

### Azure Bastion
A managed jump-host service. Costs per hour but eliminates the need for public IPs on every VM. Two SKUs:
- **Basic** — VM in the same VNet only
- **Standard** — peered VNets, native client support, scale units, IP-based connection

---

## VM Extensions

Plugins you can attach to VMs to run scripts or install software at deploy time or later.

Common extensions:
- **Custom Script Extension** — run any Bash/PowerShell script
- **VM Agent** (preinstalled) — required for most other extensions
- **Antimalware** (Microsoft Antimalware for Windows)
- **Diagnostic** — log/metric collection
- **Azure Monitor Agent (AMA)** — modern monitoring
- **DSC (Desired State Configuration)** — config management

---

## Hands-On: Create a VM with CLI

```bash
# Resource group
az group create --name rg-vm-demo --location centralindia

# Create the VM (creates VNet, NSG, NIC, public IP automatically)
az vm create \
  --resource-group rg-vm-demo \
  --name web01 \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --nsg-rule SSH

# Open port 80 for HTTP
az vm open-port --resource-group rg-vm-demo --name web01 --port 80 --priority 1010

# Install nginx via custom script
az vm extension set \
  --resource-group rg-vm-demo \
  --vm-name web01 \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --settings '{"commandToExecute":"sudo apt-get update && sudo apt-get install -y nginx"}'

# Stop (deallocate) to save money
az vm deallocate --resource-group rg-vm-demo --name web01

# Start again
az vm start --resource-group rg-vm-demo --name web01
```

---

## Common Pitfalls

1. **Forgetting to deallocate.** "Stopped" inside the OS still bills compute.
2. **Public IP changes after deallocation** unless it's a **Standard SKU Static** IP.
3. **Selecting Basic SKU for new IPs / load balancers.** Basic is being retired in 2025; choose Standard.
4. **Wrong region for premium disks.** A few regions don't support every SKU.
5. **Not enabling boot diagnostics.** When the VM fails to boot, you'll have no console screenshot to debug.

---

## Quiz: Virtual Machines

**1.** Which VM family is best for a memory-heavy database workload?  
A. F-series  B. D-series  C. E-series  D. B-series

**2.** A VM is "Stopped" but Azure is still billing for compute. What state is it in?  
A. Running  B. Deallocated  C. Stopped (from OS)  D. Failed

**3.** Which managed disk type provides the **lowest latency** for mission-critical databases?  
A. Standard HDD  B. Premium SSD  C. Ultra Disk  D. Standard SSD

**4.** Spot VMs are best suited for:  
A. Production databases  
B. Domain controllers  
C. Batch jobs that can tolerate interruption  
D. Online transactional websites

**5.** What is the maximum discount you can get with a Spot VM compared to pay-as-you-go?  
A. 30%  B. 50%  C. 72%  D. 90%

**6.** Which Azure service lets you SSH/RDP into a VM **without a public IP** and through the portal?  
A. JIT  B. Azure Bastion  C. Site-to-Site VPN  D. NAT Gateway

**7.** Which is true about VM extensions?  
A. They run only at VM creation time  
B. They require the VM Agent  
C. They are free  
D. They cannot be added after VM creation

**8.** Which encryption option encrypts both the data and the cache on the **host**?  
A. SSE  B. Azure Disk Encryption  C. Encryption at host  D. Customer-provided keys

**9.** An image you create with **Azure Compute Gallery** can be replicated:  
A. To one other region only  
B. Within a subscription only  
C. To many regions, with versioning, and shared via RBAC  
D. Only to the same VNet

**10.** True or False: You can resize a VM to a different SKU (e.g., D2s_v5 → D8s_v5) by stopping and changing size; data on managed disks persists.

---

### Answers

1. **C** — E-series is memory-optimized.  
2. **C** — Stopped from OS = still billed for compute. Deallocate to stop billing.  
3. **C** — Ultra Disk for lowest latency / highest IOPS.  
4. **C** — Spot VMs can be evicted; only good for interruptible work.  
5. **D** — Up to 90% discount.  
6. **B** — Azure Bastion.  
7. **B** — Extensions need the VM Agent (preinstalled on most images).  
8. **C** — Encryption at host covers data, cache, OS, and temp disk.  
9. **C** — Multi-region replication, versioning, RBAC sharing.  
10. **True** — Resize keeps disks; just a brief downtime.

---

**Next up:** [02-VM-Availability-and-Scaling.md](./02-VM-Availability-and-Scaling.md)
