# Virtual Networks (VNets)

## The Big Picture

A **Virtual Network (VNet)** is your private, isolated network space in Azure. Resources inside a VNet can talk to each other without going over the public internet. You define the address space (which IP ranges can be used), carve it into subnets, and decide who connects to what.

> Think of a VNet as **your own private floor in a giant office building**. It has its own corridors, rooms, and locked doors — and only your stuff is on it. The plumbing and ducting are managed by Azure; the floor plan is yours.

---

## Real-World Analogy: A Gated Community

Imagine a planned residential community:
- The **community itself** = your VNet (a defined plot of land)
- The **streets and blocks** inside = subnets
- The **houses** = your VMs, App Services, etc.
- The **community boundary wall** = isolation from outside
- The **gates** with security = gateways, public IPs, NAT Gateway
- A **bridge** to a sister community = VNet peering
- A **highway out of town** to other regions or on-prem = VPN / ExpressRoute

---

## Address Spaces

A VNet uses **private IP ranges** in CIDR notation. These come from RFC 1918:
- `10.0.0.0/8` (10.0.0.0 – 10.255.255.255)
- `172.16.0.0/12`
- `192.168.0.0/16`

You choose ranges that **don't overlap** with any other network you might peer to or VPN-connect to. (Overlapping address spaces = pain. Plan carefully.)

A VNet can have **multiple address spaces** if you outgrow the original. Subnets must always fall *inside* one of the VNet's address spaces.

### Picking a size

| CIDR | Total IPs | Usable IPs (per subnet) |
|------|-----------|--------------------------|
| /29 | 8 | 3 |
| /28 | 16 | 11 |
| /24 | 256 | 251 |
| /16 | 65,536 | 65,531 |

> **Azure reserves 5 IPs in every subnet:** the network address, the broadcast, and 3 others (`x.x.x.1`, `x.x.x.2`, `x.x.x.3`). So a /29 has 8 - 5 = 3 usable IPs.

---

## Subnets

Subnets are **logical slices** of a VNet's address space. You attach NICs, services, and gateway endpoints to subnets.

### Special subnet rules
- Subnet sizes: `/29` to `/2` (the larger the prefix number, the smaller the range)
- A NIC's IP must come from the subnet it's deployed to
- Some Azure services need a **dedicated subnet** they take over:
  - `GatewaySubnet` — for VPN / ExpressRoute Gateways (must be exactly named this)
  - `AzureFirewallSubnet`
  - `AzureBastionSubnet`
  - `RouteServerSubnet`
- **Subnet delegation** lets a managed PaaS service (e.g., App Service VNet Integration) inject into a subnet

### Reserved IPs (the 5 each subnet loses)
Example for `10.0.1.0/24`:
- `10.0.1.0` — network
- `10.0.1.1` — default gateway
- `10.0.1.2`, `10.0.1.3` — Azure DNS
- `10.0.1.255` — broadcast

---

## Public vs. Private IPs

### Private IPs
- Assigned to NICs from the subnet
- **Dynamic** (default) — changes if VM is deallocated
- **Static** — pinned to the resource lifecycle

### Public IPs
- A separate Azure resource you associate with NICs, LBs, gateways, etc.
- Two SKUs:
  - **Basic** — being retired in 2025; do not use for new
  - **Standard** — required for AZ resilience, Standard LB, etc.
- Two assignment types:
  - **Dynamic** — changes on deallocation (Basic only)
  - **Static** — pinned

### Public IP Prefixes
A reserved range of public IPs you own as a block. Useful for whitelisting in customer firewalls.

---

## VNet Peering

The cleanest way to connect two VNets. Traffic flows over the **Microsoft backbone**, not the public internet, with low latency.

### Two flavors
- **Regional peering** — VNets in the same region
- **Global peering** — VNets in different regions

### Properties you'll configure
- **Allow Forwarded Traffic** — allow traffic that didn't originate in the peered VNet (e.g., transitive via a hub)
- **Allow Gateway Transit** — let the peer use this VNet's gateway (hub-and-spoke pattern)
- **Use Remote Gateway** — the spoke uses the hub's gateway to reach on-prem
- **Allow virtual network access** — basic peering toggle

### Important rules
- Peering is **NOT transitive**. If A peers with B and B peers with C, A and C cannot talk unless explicitly peered.
- Use **Azure Virtual Network Manager** (or a hub-and-spoke topology with NVA / Azure Firewall) for transitive scenarios.
- VNet address spaces must **not overlap**.

---

## Hub-and-Spoke Topology

The classic Azure network design pattern.

```
                  ┌──────────────────┐
                  │   Hub VNet       │
                  │ (firewall, VPN,  │
                  │  shared services)│
                  └─────┬──────┬─────┘
              peering   │      │   peering
                ┌───────┘      └───────┐
                ▼                      ▼
        ┌──────────────┐       ┌──────────────┐
        │  Spoke VNet  │       │  Spoke VNet  │
        │  (workload)  │       │  (workload)  │
        └──────────────┘       └──────────────┘
```

### Why hub-and-spoke?
- Centralized **firewall**, **VPN**, **DNS**, **Bastion**
- Spokes can be isolated from each other
- Easier to onboard new spokes without re-architecting
- Cost-efficient (shared services in the hub)

---

## Service Endpoints vs. Private Endpoints

### Service Endpoints
Extend the subnet's identity to a specific Azure service. The service still has a public IP, but you can restrict it to "only allow traffic from this VNet/subnet."

- Free
- Per-service flag
- Slightly older pattern

### Private Endpoints
Give the Azure service a **private IP in your VNet**. The service is reachable only via that IP.

- Costs per endpoint per hour
- Per-resource (one storage account, one SQL DB)
- Modern, recommended pattern
- Requires Private DNS zones for name resolution

> **Exam tip:** **Service endpoint** = "I trust this VNet, public IP stays." **Private endpoint** = "I bring you into my VNet with a private IP."

---

## Hands-On: Create a VNet with Two Subnets

```bash
# Create a VNet with one subnet, then add a second
az network vnet create \
  --resource-group rg-net-demo \
  --name vnet-prod \
  --address-prefix 10.10.0.0/16 \
  --subnet-name web \
  --subnet-prefix 10.10.1.0/24 \
  --location centralindia

az network vnet subnet create \
  --resource-group rg-net-demo \
  --vnet-name vnet-prod \
  --name app \
  --address-prefix 10.10.2.0/24

# Peer two VNets
az network vnet peering create \
  --name vnet-prod-to-vnet-shared \
  --resource-group rg-net-demo \
  --vnet-name vnet-prod \
  --remote-vnet "/subscriptions/<sub-id>/resourceGroups/rg-shared/providers/Microsoft.Network/virtualNetworks/vnet-shared" \
  --allow-vnet-access

az network vnet peering create \
  --name vnet-shared-to-vnet-prod \
  --resource-group rg-shared \
  --vnet-name vnet-shared \
  --remote-vnet "/subscriptions/<sub-id>/resourceGroups/rg-net-demo/providers/Microsoft.Network/virtualNetworks/vnet-prod" \
  --allow-vnet-access
```

---

## Common Pitfalls

1. **Overlapping address spaces** between VNets you intend to peer or VPN-connect.
2. **Underestimating subnet size.** Once a subnet has resources, expanding it is hard. Plan for growth.
3. **Forgetting Azure reserves 5 IPs per subnet.** A /29 only gives you 3 usable.
4. **Peering not bidirectional.** Both directions must be created and "Allow virtual network access" enabled.
5. **Peering is not transitive.** Hub-and-spoke needs a firewall or route appliance for spoke-to-spoke.

---

## Quiz: Virtual Networks

**1.** How many IP addresses does Azure reserve in every subnet?  
A. 1  B. 3  C. 5  D. 10

**2.** True or False: VNet peering is transitive — if A peers with B and B peers with C, A can reach C.

**3.** What is the smallest subnet size you can use in Azure?  
A. /24  B. /28  C. /29  D. /30

**4.** Which subnet must be named exactly `GatewaySubnet`?  
A. Where Azure Bastion lives  
B. Where a VPN/ExpressRoute Gateway lives  
C. Where a Firewall lives  
D. Any subnet hosting public IPs

**5.** Which IP SKU is required to use Availability Zones?  
A. Basic  B. Standard  C. Premium  D. Either

**6.** Two VNets must communicate, both in the same region. What's the simplest, fastest option?  
A. Site-to-Site VPN  B. ExpressRoute  C. VNet peering  D. Internet routing

**7.** True or False: Subnet delegation gives a managed PaaS service permissions to inject network resources into a subnet.

**8.** Which is the primary difference between a Service Endpoint and a Private Endpoint?  
A. Cost only  
B. Service endpoint = public IP retained; Private endpoint = private IP in your VNet  
C. They are the same  
D. Private endpoints work only for storage

**9.** Your VNet uses `10.0.0.0/16`. You need to peer with a partner who uses `10.0.0.0/24`. What's the issue?  
A. None — the partner range fits inside yours  
B. Address spaces overlap; peering fails  
C. Need to use IPv6  
D. Use ExpressRoute

**10.** You have a hub VNet with a VPN Gateway. Spokes need to use the hub's VPN to reach on-prem. What two peering settings do you need?  
A. Allow Gateway Transit on the hub side; Use Remote Gateway on the spoke side  
B. Use Remote Gateway on both  
C. Allow Forwarded Traffic on the hub only  
D. Service endpoint to Storage

---

### Answers

1. **C** — 5 reserved IPs.  
2. **False** — Peering is NOT transitive.  
3. **C** — /29.  
4. **B** — `GatewaySubnet` for the gateway.  
5. **B** — Standard SKU for AZ.  
6. **C** — VNet peering.  
7. **True** — Subnet delegation.  
8. **B** — Public IP retention is the key.  
9. **B** — Overlapping ranges block peering.  
10. **A** — Allow Gateway Transit on hub; Use Remote Gateway on spoke.

---

**Next up:** [02-NSG-and-Routing.md](./02-NSG-and-Routing.md)
