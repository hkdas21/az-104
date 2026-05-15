# Network Connectivity: VPN, ExpressRoute, NAT, and DNS

## The Big Picture

Once your VNets and subnets are set, the next questions are:
- How does **on-premises** connect to Azure?
- How do VMs reach the **internet** without exposing public IPs?
- How does **DNS** resolve names inside and across VNets?

This file covers all three.

---

## Real-World Analogy: Roads to and from a Gated Community

- **VPN Gateway** = a small toll road over the public highways (encrypted, but uses the internet)
- **ExpressRoute** = a private dedicated highway, owned by you, that doesn't touch the public internet
- **NAT Gateway** = the community's exit gate where everyone leaving uses the same address
- **Azure DNS / Private DNS** = the postal directory that maps names to addresses inside your community

---

## Site-to-Site VPN

A **site-to-site (S2S) VPN** connects your on-prem network to an Azure VNet over the public internet using IPsec/IKE encryption.

### Components
- **Virtual Network Gateway** — Azure's VPN endpoint. Lives in the `GatewaySubnet`.
- **Local Network Gateway** — represents your on-prem VPN device's public IP and address spaces in Azure.
- **Connection** — links the two gateways.

### VPN Gateway SKUs
| SKU | Bandwidth | Tunnels | Use case |
|-----|-----------|---------|---------|
| **Basic** | 100 Mbps | 10 | Dev/test (legacy, retiring) |
| **VpnGw1 / 1AZ** | 650 Mbps | 30 | Most prod |
| **VpnGw2 / 2AZ** | 1 Gbps | 30 | Larger prod |
| **VpnGw3 / 3AZ** | 1.25 Gbps | 30 | Heavy traffic |
| **VpnGw4 / 4AZ** | 5 Gbps | 100 | Very heavy |
| **VpnGw5 / 5AZ** | 10 Gbps | 100 | Largest |

> **AZ variants** = zone-redundant gateway. Use them in regions with availability zones.

### VPN types
- **Route-based (preferred)** — uses any-to-any IPsec, supports BGP
- **Policy-based (legacy)** — pinned to specific traffic selectors, limited tunnels

### Active-Active vs. Active-Standby
- **Active-Standby** — default, two instances, only one handles traffic
- **Active-Active** — both gateways handle traffic in parallel; use **two on-prem devices** for redundancy

### Point-to-Site VPN (P2S)
For individual users connecting to Azure (e.g., remote workers):
- **OpenVPN, IKEv2, SSTP** protocols
- Authenticate with Azure certificates, RADIUS, or **Microsoft Entra ID**
- Doesn't require a separate on-prem device — just install client on the laptop

---

## ExpressRoute

ExpressRoute provides a **private, dedicated connection** between your on-prem and Azure that **does not traverse the public internet**.

### Key facts
- Up to **100 Gbps** circuit speeds
- **No internet** in the path; lower latency, higher reliability
- Comes via a **connectivity provider** (Equinix, ATT, Verizon, Tata, Airtel, etc.)
- Connects to **multiple Microsoft cloud services** (Azure, Microsoft 365 — though M365 over ER requires special approval and is usually discouraged)
- Resilience: deploy with **two circuits** in different peering locations

### Peering types
- **Private peering** — connect to your VNets (the most common AZ-104 scenario)
- **Microsoft peering** — connect to Microsoft 365, Dynamics, public Azure services with public IPs

### Pricing models
- **Metered** — pay for outbound bandwidth
- **Unlimited** — flat fee, unlimited outbound

### ExpressRoute Direct
A 10/100 Gbps connection straight into Microsoft's edge router (you bring your own fiber). For massive enterprises.

### ExpressRoute Global Reach
Lets two ExpressRoute circuits in different geographic locations exchange traffic over Microsoft's backbone — useful for connecting on-prem sites *via* Azure.

### VPN as backup for ExpressRoute
A common pattern. Configure routing so VPN is used only if ExpressRoute fails (uses BGP weights / AS-path prepending).

---

## Azure Virtual WAN

A unified network management service that brings together VPN, ExpressRoute, SD-WAN, and routing into one Microsoft-managed hub.

### Why use it
- Replaces hand-built hub-and-spoke for multi-region scale
- Native any-to-any branch connectivity
- Automated routing, easier management
- Supports **Secured Virtual Hub** with Azure Firewall integration

> AZ-104 typically asks about basic awareness — know that vWAN exists for large multi-region scenarios.

---

## NAT Gateway

A managed service that gives an entire subnet **outbound internet** through a static public IP, *without* exposing each VM with its own public IP.

### Why use it
- VMs don't need individual public IPs
- Predictable outbound source IP for whitelisting
- Avoids **SNAT port exhaustion** issues that plague Load Balancer outbound

### Properties
- Up to **16 public IPs** (or a /28 prefix) for SNAT scaling
- Idle timeout (4–120 minutes)
- Zone-pinned (deploy one per zone for resilience)
- Replaces "Outbound" feature of Standard LB for most scenarios

> **Best practice:** Always pair production subnets with a NAT Gateway for outbound. It's the cleanest, most reliable option.

---

## Outbound Connectivity Options Compared

| Option | Notes |
|--------|-------|
| **Default outbound IP** (legacy) | Microsoft assigns a temporary public IP. **Being deprecated** in late 2025 — do not depend on this. |
| **Public IP on the VM NIC** | Every VM has its own public IP. Fine for one-offs, doesn't scale. |
| **Standard LB outbound rules** | Uses LB frontend IPs for SNAT. Limited SNAT ports per backend. |
| **NAT Gateway** | Recommended. Fewer SNAT problems, dedicated public IP(s). |

---

## Azure DNS

Three flavors of DNS in Azure:

### 1. Azure-provided DNS (default)
- Every VNet has a built-in resolver at `168.63.129.16`
- Resolves names of VMs in the same VNet (`<vmname>.internal.cloudapp.net` style)
- No config needed

### 2. Azure DNS (Public Zones)
- Hosted authoritative DNS for **public domains** you own
- Azure manages the name servers
- Use Alias records for Azure resources (App Service, public IP, Front Door, Traffic Manager)

### 3. Azure Private DNS Zones
- DNS for private names that resolve **only inside your VNets**
- Link the zone to one or more VNets ("VNet links")
- Optional **registration** of VMs created in linked VNets
- Required for **Private Endpoints** to resolve to private IPs (each Azure service has a specific private DNS zone, e.g., `privatelink.blob.core.windows.net`)

### Custom DNS Servers
You can override the default Azure DNS at the VNet level and point to your own DNS servers (e.g., on-prem AD DCs). VMs will inherit those DNS settings.

---

## DNS for Private Endpoints — The Tricky Bit

When you create a private endpoint for, say, a storage account:

1. The storage account's public DNS still works (e.g., `contoso2026.blob.core.windows.net`)
2. You need a **Private DNS zone** named `privatelink.blob.core.windows.net`
3. An A record `contoso2026` in that zone points to the private IP
4. The CNAME chain magically resolves inside your VNet to the private IP

If you don't link the Private DNS zone to your VNets, name resolution falls back to the public IP and your private endpoint is useless. **This trips up almost everyone.**

---

## Hands-On: Create a NAT Gateway

```bash
# Public IP for the NAT Gateway
az network public-ip create \
  --resource-group rg-net-demo --name pip-nat \
  --sku Standard --allocation-method Static

# NAT Gateway
az network nat gateway create \
  --resource-group rg-net-demo --name natgw-prod \
  --public-ip-addresses pip-nat \
  --idle-timeout 30

# Attach to a subnet
az network vnet subnet update \
  --resource-group rg-net-demo --vnet-name vnet-prod \
  --name web --nat-gateway natgw-prod
```

---

## Common Pitfalls

1. **Default outbound IP** is going away. Migrate to NAT Gateway or explicit public IPs before late 2025.
2. **Mixing Basic and Standard SKUs** in VPN gateways. Match SKUs across components.
3. **Private DNS zone not linked to the VNet.** Private endpoints will resolve to the public IP and fail.
4. **VPN tunnels with overlapping address spaces.** Won't establish.
5. **ExpressRoute without redundancy.** A single circuit means one provider outage takes you offline.

---

## Quiz: Network Connectivity

**1.** Which Azure service provides a **private, non-internet** connection from on-prem to Azure?  
A. Site-to-Site VPN  B. ExpressRoute  C. NAT Gateway  D. Front Door

**2.** True or False: A VPN Gateway lives in any subnet of your choosing.

**3.** Which VPN type is preferred for production today?  
A. Policy-based  B. Route-based  C. Static  D. None — use ExpressRoute only

**4.** Which is the recommended outbound connectivity option for most subnets in 2025?  
A. Default outbound IP  B. Per-VM public IPs  C. Standard LB outbound rules  D. NAT Gateway

**5.** True or False: Azure Private DNS Zones are required for Private Endpoint name resolution to work inside a VNet.

**6.** Which protocol(s) are supported by Point-to-Site VPN?  
A. SSTP only  B. OpenVPN, IKEv2, SSTP  C. SSH  D. RDP

**7.** What does ExpressRoute Global Reach do?  
A. Provides global anycast like Front Door  
B. Lets two ExpressRoute circuits exchange traffic over Microsoft backbone  
C. Provides global VPN  
D. Routes M365 traffic

**8.** Which SKU of Azure Load Balancer no longer suffers from SNAT exhaustion when paired with NAT Gateway?  
A. Basic  B. Standard with NAT Gateway attached to subnet  C. Premium  D. None

**9.** A point-to-site VPN client cannot connect. The cause is most often:  
A. Address overlap  
B. Missing client certificate, wrong root certificate, or inactive Entra auth  
C. NSG on GatewaySubnet  
D. Wrong VPN SKU

**10.** What address does Azure-provided DNS use as the default resolver inside a VNet?  
A. 8.8.8.8  B. 168.63.129.16  C. 1.1.1.1  D. 169.254.169.254

---

### Answers

1. **B** — ExpressRoute is private.  
2. **False** — Must be in `GatewaySubnet`.  
3. **B** — Route-based.  
4. **D** — NAT Gateway is the modern recommendation.  
5. **True** — Without the linked Private DNS zone, resolution falls back to public IP.  
6. **B** — OpenVPN, IKEv2, SSTP.  
7. **B** — Connects two ER circuits via the backbone.  
8. **B** — Standard LB + NAT Gateway eliminates SNAT issues.  
9. **B** — Most P2S issues are auth (cert mismatch / Entra config).  
10. **B** — `168.63.129.16` is the magic DNS IP.

---

**You finished Module 4!** 🎉

Last module ahead: [Module 5 — Monitoring & Backup](../05-Monitoring-and-Backup/README.md). The home stretch.
