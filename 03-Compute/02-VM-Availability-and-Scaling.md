# VM Availability and Scaling

## The Big Picture

A single VM can fail. Hardware dies, hosts get patched, datacenters lose power. **Availability** is about making sure your application keeps running anyway. **Scaling** is about adapting the number of VMs to match load.

The exam tests whether you can pick the right strategy:
- **Availability Set** — protect against rack-level failure inside one datacenter
- **Availability Zones** — protect against entire-datacenter failure within a region
- **VM Scale Sets (VMSS)** — auto-scale a fleet of identical VMs

---

## Real-World Analogy: A Restaurant Chain

Think of your app like a fast-food restaurant chain:
- **Single VM** = one cashier. If they go on lunch, no one gets served.
- **Availability Set** = multiple cashiers in the same shop, scheduled never to take breaks at the same time
- **Availability Zones** = multiple branches across the city; if one is flooded, the others still serve customers
- **VMSS** = automatically opening more branches when there's a queue, closing branches when it's quiet

---

## Service Level Agreements (SLAs)

| Configuration | Uptime SLA |
|---------------|------------|
| Single VM with Premium SSD only | 99.9% |
| Single VM with Premium SSD v2 / Ultra | 99.9% |
| Availability Set (2+ VMs) | 99.95% |
| Availability Zones (2+ VMs across 2+ zones) | 99.99% |

> **Exam trap:** A single VM has an SLA only if its OS disk is Premium SSD or better. Standard HDD = no SLA.

---

## Availability Sets

A logical grouping of VMs spread across **fault domains** (FDs) and **update domains** (UDs) inside a single datacenter.

### Fault Domains
- **Hardware silos** — different racks, network switches, power supplies
- Default: **2 FDs** in most regions, up to **3**
- Protects against hardware failure

### Update Domains
- **Reboot groups** — Azure patches VMs UD by UD
- Default: **5 UDs**, up to **20**
- Protects against host OS updates

### Important rules
- VMs must be **assigned to the availability set at creation**. You can't add later (well, there are workarounds, but exam answer = no).
- All VMs in the set should run the **same workload**, behind the **same load balancer**.
- Availability sets are **free** — you're just asking Azure to spread the VMs intelligently.

---

## Availability Zones (AZs)

**Physically separate datacenters** within an Azure region, each with independent power, cooling, and networking. A region with AZs has at least 3.

### How you use them
You create each VM with an explicit zone (1, 2, or 3). Azure ensures each lands in a different physical datacenter.

### Resources that support zones
- VMs and managed disks
- Public IPs (zone-redundant or zonal)
- Standard Load Balancer
- VPN Gateway, ExpressRoute Gateway
- App Gateway v2

### Zone-redundant vs. zonal
- **Zonal** — pinned to a specific zone (e.g., `Zone 1`). If that zone goes down, the resource is unavailable.
- **Zone-redundant** — automatically replicated across all zones. Typically used for control plane / shared services.

> **Exam trick:** A *VM* is zonal (it lives in one zone). A *Load Balancer frontend IP* can be zone-redundant.

### Comparing the two

| Feature | Availability Set | Availability Zones |
|---------|------------------|---------------------|
| Scope of protection | Rack/host within one DC | Entire datacenters |
| SLA | 99.95% | 99.99% |
| Cross-region | No | No (single region) |
| Latency between members | Low (same DC) | Low but slightly higher than AS |
| Cost | Free | Free, but cross-zone bandwidth is billed |

---

## Virtual Machine Scale Sets (VMSS)

A way to deploy and manage a **group of identical VMs** that auto-scale based on demand or schedule.

### Key concepts

- **Capacity** — current number of instances
- **Min / Max** — bounds for autoscale
- **Custom image** or marketplace image as the template
- **Upgrade policy** — Manual, Automatic, or Rolling
- **Scaling rules** — based on metrics (CPU, memory, custom)

### Orchestration modes

| Mode | Description | Use case |
|------|-------------|----------|
| **Uniform** (classic) | All instances identical, simpler API | Most common — pure scaleout fleets |
| **Flexible** | More VM-like, supports mixed sizes, AZ + FD spread, lifecycle hooks | Modern; recommended for new deployments |

> **Exam tip:** **Flexible** is the modern default. Uniform is still fine for stateless web tiers.

### Autoscale rules
You define rules like:

> **IF** CPU > 70% for 10 minutes **THEN** add 2 instances (cool down 5 min)  
> **IF** CPU < 20% for 10 minutes **THEN** remove 1 instance (cool down 5 min)

You also set absolute **min** and **max** instance counts. Without min, scale could collapse to zero.

### Distribution
- **Zonal VMSS** — pin to a single zone
- **Zone-redundant VMSS** — spread across multiple AZs (recommended)
- **Spread across FDs** within a region without AZs

### Combine with a Load Balancer
A VMSS without a load balancer is a fleet of orphans. Pair it with:
- **Standard Load Balancer** for L4 traffic
- **Application Gateway** for L7 (web)
- **Azure Front Door** for global anycast

---

## Update / Patch Management

### Auto-Guest OS Patching
Azure automatically applies critical/security patches to Windows/Linux VMs that opt-in. Restarts during off-hours.

### Update Manager
Centralized patching for Azure and Arc-enabled servers. Replaces the older Automation Update Management (which is being retired).

Features:
- Schedule deployments
- Pre/post scripts
- Compliance reporting
- Reboot control

### Maintenance Configurations
Schedule when **host updates** can hit your VM (every 1 to 35 days). Useful for stable production windows.

---

## Hands-On: Create a VMSS

```bash
# Create a Flexible orchestration mode VMSS spread across zones
az vmss create \
  --resource-group rg-scale-demo \
  --name vmss-web \
  --orchestration-mode Flexible \
  --image Ubuntu2204 \
  --instance-count 2 \
  --vm-sku Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --zones 1 2 3 \
  --upgrade-policy-mode Rolling \
  --load-balancer "" \
  --public-ip-per-vm

# Set autoscale
az monitor autoscale create \
  --resource-group rg-scale-demo \
  --resource vmss-web \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name vmss-autoscale \
  --min-count 2 --max-count 10 --count 2

# Add scale-out rule on CPU > 70
az monitor autoscale rule create \
  --resource-group rg-scale-demo \
  --autoscale-name vmss-autoscale \
  --condition "Percentage CPU > 70 avg 10m" \
  --scale out 2

# Add scale-in rule on CPU < 30
az monitor autoscale rule create \
  --resource-group rg-scale-demo \
  --autoscale-name vmss-autoscale \
  --condition "Percentage CPU < 30 avg 10m" \
  --scale in 1
```

---

## Common Pitfalls

1. **Mixing Availability Set + Zones.** A VM is in *either* an AS *or* an AZ — not both.
2. **Adding a VM to an AS post-creation.** Officially not supported; you must redeploy.
3. **Forgetting cool-down on autoscale rules.** Without it, VMs flap up and down rapidly.
4. **Manual upgrade policy on VMSS** means you have to push updates yourself — common gotcha.
5. **Running stateful workloads on a scale set** — bad idea unless you've designed for it (external state in DB / queue / blob).

---

## Quiz: Availability and Scaling

**1.** What SLA do you get with two VMs in an Availability Set?  
A. 99.9%  B. 99.95%  C. 99.99%  D. 100%

**2.** True or False: Availability Sets and Availability Zones can be used together for a single VM.

**3.** Which VMSS orchestration mode supports mixed VM sizes and AZ spread with FD count > 1?  
A. Uniform  B. Flexible  C. Mixed  D. Hybrid

**4.** Which is **NOT** typically zone-redundant?  
A. A specific VM  B. A Standard Public IP  C. A Standard Load Balancer frontend  D. A managed disk in zone-redundant SKU

**5.** Your CPU autoscale rule fires every minute. VMs flap up and down. What's the fix?  
A. Lower the threshold  B. Increase the cool-down  C. Switch to Spot  D. Disable autoscale

**6.** A VMSS is configured with **manual** upgrade policy. You update the model. What happens?  
A. All VMs reboot immediately  B. Existing VMs are upgraded automatically  C. Nothing — you must trigger upgrade per instance  D. The VMSS recreates

**7.** Which protects against an entire datacenter losing power?  
A. Availability Set  B. Availability Zone  C. Multiple regions  D. Both B and C

**8.** Which VM SKU does NOT carry an SLA on a single VM?  
A. Premium SSD-backed  B. Ultra Disk-backed  C. Standard SSD-backed  D. Premium SSD v2-backed

**9.** A VMSS without a load balancer is:  
A. Best practice for cost  B. Effectively useless for inbound traffic  C. Required for ZRS storage  D. Not allowed by Azure

**10.** Update Manager replaces which legacy service?  
A. Azure Backup  B. Automation Update Management  C. Recovery Services Vault  D. Site Recovery

---

### Answers

1. **B** — 99.95%.  
2. **False** — A VM is either zonal or in an AS, not both.  
3. **B** — Flexible mode.  
4. **A** — A VM is zonal (lives in one zone). The other resources can be zone-redundant.  
5. **B** — Increase cool-down so the trigger doesn't fire repeatedly.  
6. **C** — Manual mode requires you to push the upgrade.  
7. **D** — AZs protect within a region, multi-region protects against region failure.  
8. **C** — Standard SSD has no single-VM SLA. (Premium SSD/Ultra/v2 do.)  
9. **B** — A VMSS without a LB has no traffic distribution — not useful.  
10. **B** — Update Manager replaces Automation Update Management.

---

**Next up:** [03-App-Services.md](./03-App-Services.md)
