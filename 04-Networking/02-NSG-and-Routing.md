# NSGs, ASGs, and Routing

## The Big Picture

Once you have VNets and subnets, the next question is: **who can talk to whom, and how does the traffic get there?**

- **Network Security Groups (NSGs)** decide what is *allowed* — like firewall rules
- **Application Security Groups (ASGs)** group VMs by role so NSG rules can target the role, not the IP
- **Route Tables (User-Defined Routes / UDRs)** decide *where* traffic should go

Master this trio and you can secure and steer almost any Azure network.

---

## Real-World Analogy

Imagine a busy office building:
- **NSGs** = security guards at each elevator who check your badge against an allow/deny list
- **ASGs** = the role labels on the badge ("HR", "Engineering") that the guards can use instead of memorizing every name
- **Route Tables** = the building's signage telling you which elevator goes to which floor

---

## Network Security Groups (NSGs)

An NSG is an **ordered list of allow/deny rules** for inbound and outbound traffic. You can attach an NSG to a **subnet** and/or to a **NIC**.

### Rule properties
| Property | Example |
|----------|---------|
| **Priority** | 100–4096; lower numbers evaluated first |
| **Source / Destination** | IP, CIDR, service tag, ASG, "VirtualNetwork", "Any" |
| **Source / Destination port** | A port or range |
| **Protocol** | TCP, UDP, ICMP, Any |
| **Action** | Allow / Deny |
| **Direction** | Inbound / Outbound |

### Default rules (you can't delete, only override)

**Inbound defaults:**
1. `AllowVnetInBound` — allow traffic within the VNet
2. `AllowAzureLoadBalancerInBound` — allow Azure LB health probes
3. `DenyAllInBound` — block everything else

**Outbound defaults:**
1. `AllowVnetOutBound` — allow within VNet
2. `AllowInternetOutBound` — allow outbound to the internet
3. `DenyAllOutBound` — *only triggered by your custom rules; the default outbound is allow*

> **Exam trap:** By default, **outbound to the internet is allowed**. If you want to lock it down, add an explicit deny rule (lower priority number than `AllowInternetOutBound`).

### Service Tags
Predefined groups of IP prefixes Microsoft maintains:
- `Internet`
- `VirtualNetwork`
- `AzureLoadBalancer`
- `Storage` (or `Storage.CentralIndia` for region-specific)
- `Sql.<region>`
- `AzureCloud.<region>`

Use them in source/destination instead of trying to memorize Microsoft's CIDR blocks.

### NSG attachment

| Attached to | What it filters |
|-------------|-----------------|
| Subnet | All traffic to/from any resource in the subnet |
| NIC | Traffic to/from that specific VM/NIC |

If both are attached, **inbound** rules are evaluated subnet-first then NIC; **outbound** is NIC-first then subnet. Both must allow for traffic to flow.

### Order of evaluation
Rules are evaluated **in priority order**, lowest number first. **First match wins**, then traffic stops being evaluated.

### NSG flow logs
Capture metadata about traffic that hit the NSG (allowed and denied). Stored in a storage account. Critical for troubleshooting and security audit.

> **Newer:** **VNet flow logs** are replacing NSG flow logs going forward. They capture traffic at the VNet level.

---

## Application Security Groups (ASGs)

An ASG is just a **named group of NICs**. Once you assign NICs to an ASG, you can write NSG rules using the ASG name.

```
ASG: web-servers   → contains NIC1, NIC2, NIC3
ASG: db-servers    → contains NIC4, NIC5

NSG rule: Allow TCP 1433 from web-servers to db-servers
```

Why this is useful: when you add a 4th web server, you just add it to the ASG. No NSG rule change needed.

> **Exam trap:** ASGs are local to a region/VNet. You can't cross VNets with one ASG.

---

## Routing

Every subnet has a **default routing table** Azure provides automatically. It includes routes for:
- The VNet's address space (next hop = VirtualNetwork)
- `0.0.0.0/0` (next hop = Internet)
- BGP-learned routes (from VPN/ExpressRoute) if configured
- Routes for peered VNets

When you want to **override** these — for example, to send all internet traffic through a firewall — you create a **User-Defined Route (UDR)** in a custom route table and attach it to the subnet.

### UDR properties
- **Address prefix** — the destination CIDR
- **Next hop type:**
  - **Virtual network gateway** — VPN/ER traffic
  - **Virtual network** — within the VNet
  - **Internet** — public internet
  - **Virtual appliance** — a firewall NVA (specify private IP)
  - **None** — black-hole / drop the traffic

### Most common pattern: Forced Tunneling / Firewall

Send `0.0.0.0/0` to the firewall:
```
Address prefix : 0.0.0.0/0
Next hop type  : Virtual appliance
Next hop IP    : 10.10.0.4    (the firewall's NIC)
```

Now every internet-bound packet from VMs in this subnet goes through the firewall first.

### Route precedence (the order Azure uses)
1. **User-defined routes** (highest)
2. **BGP routes** (from VPN / ExpressRoute)
3. **System routes** (lowest)

> If two routes match the same prefix, the one higher in this list wins.

### Effective routes
The combined view from a NIC's perspective. Use `az network nic show-effective-route-table` to see what a VM is *actually* using. Lifesaver when troubleshooting.

---

## Azure Firewall (the big brother of NSGs)

Azure Firewall is a managed, stateful, L3–L7 firewall as a service. Unlike NSGs which are basically L4 packet filters, Azure Firewall does:
- FQDN-based filtering ("allow only `*.contoso.com`")
- Threat intelligence integration
- DNAT (publish a backend service)
- Application rules with TLS inspection (Premium)
- Rule collection groups, hierarchical policies

You typically deploy Azure Firewall in the **hub** of a hub-and-spoke. Pair with UDRs that send `0.0.0.0/0` and "anything spoke-to-spoke" through the firewall.

### Tiers
- **Basic** — small offices, dev/test
- **Standard** — most workloads
- **Premium** — IDPS, TLS inspection, URL filtering

---

## Hands-On: NSG with ASG

```bash
# Create ASGs
az network asg create -g rg-net-demo -n asg-web -l centralindia
az network asg create -g rg-net-demo -n asg-db  -l centralindia

# Create NSG
az network nsg create -g rg-net-demo -n nsg-web -l centralindia

# Allow web in (HTTPS)
az network nsg rule create -g rg-net-demo --nsg-name nsg-web -n allow-https \
  --priority 100 --direction Inbound --access Allow \
  --source-address-prefixes Internet --destination-port-ranges 443 \
  --protocol Tcp \
  --destination-asgs asg-web

# Allow web → DB
az network nsg rule create -g rg-net-demo --nsg-name nsg-web -n allow-web-to-db \
  --priority 200 --direction Outbound --access Allow \
  --source-asgs asg-web --destination-asgs asg-db \
  --destination-port-ranges 1433 --protocol Tcp

# Attach NSG to subnet
az network vnet subnet update \
  -g rg-net-demo --vnet-name vnet-prod --name web \
  --network-security-group nsg-web
```

---

## Common Pitfalls

1. **Forgetting the default `AllowAzureLoadBalancerInBound`.** If you replace it with a deny, your LB health probes fail and the LB takes the VM out of rotation.
2. **Order matters.** A deny at priority 100 will block traffic an allow at 200 wanted to permit.
3. **NSG attached to both subnet and NIC.** Both must allow; surprising when only one denies.
4. **UDR with next hop "None"** silently black-holes traffic. Great for blocking, easy to forget you set it.
5. **Putting an NSG on the GatewaySubnet.** Generally **not supported**; you'll cause connectivity issues.

---

## Quiz: NSGs and Routing

**1.** Which is true about NSG rule priorities?  
A. Higher numbers are evaluated first  
B. Lower numbers are evaluated first  
C. Order doesn't matter  
D. Equal priorities are random

**2.** True or False: By default, outbound internet traffic from an Azure VM is blocked.

**3.** Which is the lowest valid priority number you can use in an NSG rule?  
A. 1  B. 100  C. 4096  D. 0

**4.** Which next hop type effectively drops traffic?  
A. Internet  B. Virtual appliance  C. None  D. Virtual network

**5.** You add a UDR with destination `0.0.0.0/0` and next hop `Virtual appliance` (firewall IP). What happens to internet-bound traffic?  
A. Goes directly to the internet (system route still wins)  
B. Goes through the firewall  
C. Is blocked  
D. Loops in the VNet

**6.** Which is correct about ASGs?  
A. They are global across regions  
B. They group NICs to simplify NSG rules  
C. They replace NSGs  
D. They define routing

**7.** Both a subnet NSG and a NIC NSG apply to a VM. For inbound traffic, which is evaluated **first**?  
A. NIC-level NSG  B. Subnet-level NSG  C. Both at once  D. Random

**8.** Which service tag represents traffic from the Azure Load Balancer for health probes?  
A. AzureCloud  B. AzureLoadBalancer  C. Internet  D. VirtualNetwork

**9.** Where does Azure Firewall typically live in a hub-and-spoke?  
A. In every spoke  B. In the hub  C. In the GatewaySubnet  D. As an extension

**10.** Effective routes for a VM differ from the route table you assigned. What's likely?  
A. BGP routes from a VPN gateway are merging in  
B. The route table is misapplied  
C. NSG rules are routing  
D. Azure has a bug

---

### Answers

1. **B** — Lower priority number = evaluated first.  
2. **False** — Outbound to internet is allowed by default.  
3. **B** — 100 is the lowest. (Range 100–4096; 0–99 reserved.)  
4. **C** — Next hop "None" black-holes the packet.  
5. **B** — UDRs win over system routes; firewall receives the traffic.  
6. **B** — ASGs simplify NSG rule writing.  
7. **B** — Inbound: subnet-first, then NIC.  
8. **B** — AzureLoadBalancer service tag.  
9. **B** — In the hub.  
10. **A** — BGP routes from VPN/ER often merge in and you may not have noticed.

---

**Next up:** [03-Load-Balancing.md](./03-Load-Balancing.md)
