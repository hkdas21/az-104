# Load Balancing

## The Big Picture

If you have multiple servers behind a single front door, **load balancers** decide which server gets each incoming request. Azure offers four major load balancing services — each at a different layer of the OSI model and with different scopes.

| Service | Layer | Scope | Best for |
|---------|-------|-------|---------|
| **Azure Load Balancer** | L4 (TCP/UDP) | Regional | Generic high-throughput, fast |
| **Application Gateway** | L7 (HTTP/S) | Regional | Web apps with WAF, URL routing |
| **Front Door** | L7 (HTTP/S) | Global | Worldwide web traffic, CDN-like |
| **Traffic Manager** | DNS-level | Global | Multi-region routing by DNS |

---

## Real-World Analogy: Restaurant Hostesses at Different Levels

Imagine you own multiple restaurants:
- **Azure Load Balancer** = the hostess at the door of one restaurant who quickly directs you to a free table without caring what you'll order
- **Application Gateway** = the hostess who looks at your reservation and your order type ("vegetarian section") before seating you
- **Front Door** = a global concierge who routes you to the closest restaurant in your city
- **Traffic Manager** = a phone receptionist who tells you the address of the right restaurant — you still have to drive there yourself

---

## Azure Load Balancer

**Layer 4** — works on TCP/UDP packets without inspecting their content. Fast, cheap, scalable.

### SKUs
| Feature | Basic (legacy, retiring) | Standard |
|---------|--------------------------|----------|
| Backend pool | 1 VNet | Multiple VNets |
| Health probes | TCP, HTTP | TCP, HTTP, HTTPS |
| Availability zones | No | Yes |
| HA Ports | No | Yes |
| Outbound rules | No | Yes |
| Secure by default | No (open by default) | Yes (closed; needs NSG) |
| SLA | None | 99.99% |

> **Always use Standard.** Basic LB is being retired in 2025.

### Public vs. Internal
- **Public Load Balancer** — public IP frontend; receives internet traffic
- **Internal Load Balancer (ILB)** — private IP frontend; balances inside the VNet (e.g., a 3-tier app's middle tier)

### Components

```
Frontend IP        →  Backend Pool      →  Health Probes
(public/private)      (VMs / VMSS)         (TCP/HTTP/S checks)
       ↓
Load Balancing Rules / Inbound NAT Rules
```

- **Frontend IP** — where clients hit
- **Backend pool** — VMs or VMSS instances that receive traffic
- **Health probes** — periodic checks; unhealthy instances are removed from rotation
- **LB rules** — map frontend → backend (e.g., 80→80)
- **Inbound NAT rules** — port forward to a single backend (e.g., 50001→3389 for RDP)

### Distribution modes
- **5-tuple hash** (default) — based on src IP, src port, dst IP, dst port, protocol
- **Source IP affinity (3-tuple)** — same client → same backend
- **Source IP + Protocol (2-tuple)** — variant

### HA Ports
A special LB rule that listens on **all** ports (1–65535). Useful for NVAs (firewalls) where you can't enumerate ports.

---

## Application Gateway

**Layer 7** — understands HTTP/S. Can do SSL termination, URL-based routing, header rewriting, and **Web Application Firewall (WAF)**.

### Tiers
- **Standard v2** — modern, autoscale, AZ support
- **WAF v2** — Standard v2 + Web Application Firewall

> Skip Standard v1 / WAF v1 — legacy.

### Capabilities

| Feature | What it does |
|---------|--------------|
| **URL-based routing** | `/images/*` → image servers, `/api/*` → API servers |
| **Multi-site hosting** | Different domains routed to different backend pools |
| **SSL/TLS termination** | Decrypt at the gateway, optionally re-encrypt to backend |
| **End-to-end TLS** | Re-encrypt on the way to backend |
| **WAF** | Block OWASP top 10, custom rules, bot mitigation |
| **Autoscale** | Scale capacity units automatically |
| **Session affinity (cookie-based)** | Sticky sessions |
| **Header rewrite** | Modify headers in flight |
| **Connection draining** | Graceful removal of unhealthy backends |

### Components
- **Listener** — receives traffic on a frontend IP/port (HTTP or HTTPS)
- **Routing rules** — how to route requests
- **Backend pool** — VMs, VMSS, App Services, IPs/FQDNs
- **HTTP settings** — port, protocol, timeouts, probe link
- **Health probe** — like LB probes but HTTP-aware

### Subnet requirement
App Gateway needs **its own dedicated subnet** in the VNet, ideally `/24` for room to grow. The subnet can host **only Application Gateways** (different versions can coexist).

---

## Azure Front Door

**Layer 7, GLOBAL.** Powered by Microsoft's anycast network — clients hit the closest Microsoft edge POP, then traffic is routed to your backend.

### Tiers
- **Standard / Premium**
  - **Premium** adds private link to backends, advanced WAF, bot management
- **Classic** — legacy

### Use Front Door when
- You have backends in multiple regions
- You want global load balancing
- You need a CDN with smart routing
- You want a global WAF

### Routing methods
- **Latency** (default) — closest backend by latency
- **Priority** — primary/secondary failover
- **Weighted** — A/B testing
- **Session affinity** — sticky based on cookie

### Difference vs. App Gateway
- App Gateway is **regional**; Front Door is **global**
- App Gateway is in your VNet; Front Door is at Microsoft's edge
- Often used **together** — Front Door at the edge, App Gateway per region

---

## Traffic Manager

**DNS-based** global load balancer. Doesn't see your traffic — it just hands clients an IP/hostname when they do a DNS lookup.

### Routing methods
- **Priority** — failover (primary/secondary)
- **Weighted** — % split
- **Performance** — closest endpoint by latency
- **Geographic** — by client country
- **MultiValue** — multi-record response
- **Subnet** — specific client IP ranges to specific endpoints

### Caveat
Because it's DNS-based, **TTL caching** can delay failover. Clients with cached DNS will keep going to the down endpoint.

---

## Picking the Right One

| Need | Service |
|------|---------|
| Balance VMs in one region, fast TCP/UDP | Azure Load Balancer |
| Balance HTTP with WAF and URL routing | Application Gateway |
| Global HTTP traffic with edge caching | Front Door |
| DNS-based multi-region failover with cheapest cost | Traffic Manager |

> **Exam trap:** "User in India should reach the India backend, user in Germany should reach the EU backend, with **WAF protection**" — that's **Front Door Premium**, or App Gateway with Front Door. Traffic Manager alone has no WAF.

---

## Hands-On: Standard Load Balancer

```bash
# Create LB
az network lb create \
  --resource-group rg-net-demo \
  --name lb-prod \
  --sku Standard \
  --public-ip-address pip-lb \
  --frontend-ip-name fe1 \
  --backend-pool-name be1

# Health probe
az network lb probe create \
  --resource-group rg-net-demo \
  --lb-name lb-prod \
  --name probe-http \
  --protocol Http --port 80 --path /

# LB rule
az network lb rule create \
  --resource-group rg-net-demo \
  --lb-name lb-prod \
  --name rule-http \
  --protocol Tcp --frontend-port 80 --backend-port 80 \
  --frontend-ip-name fe1 --backend-pool-name be1 \
  --probe-name probe-http
```

---

## Common Pitfalls

1. **Mixing SKUs.** A Basic LB cannot send traffic to a VM with a Standard public IP. Standard with Standard, end of story.
2. **Forgetting health probes** — without them, traffic goes everywhere including dead VMs.
3. **App Gateway subnet too small.** /27 minimum, /24 strongly recommended.
4. **Using Traffic Manager when you want WAF** — Traffic Manager is DNS only, no security inspection.
5. **TCP probe for HTTP services.** TCP only checks if port is open. An HTTP probe verifies the app actually works.

---

## Quiz: Load Balancing

**1.** Which Azure load balancer operates at **Layer 7**?  
A. Azure Load Balancer (Standard)  B. Traffic Manager  C. Application Gateway  D. NAT Gateway

**2.** True or False: Azure Front Door is a regional service.

**3.** Which load balancer is **DNS-based** and does not see actual traffic?  
A. Standard LB  B. App Gateway  C. Front Door  D. Traffic Manager

**4.** You need a Web Application Firewall in front of a regional web app. Best fit?  
A. Standard LB  B. Application Gateway WAF v2  C. Front Door Standard  D. Traffic Manager

**5.** Which session distribution mode in Azure Load Balancer makes the same client always reach the same backend?  
A. 5-tuple hash  B. Source IP affinity  C. HA Ports  D. Random

**6.** Health probes for a Standard Load Balancer can be:  
A. TCP only  B. HTTP only  C. TCP, HTTP, HTTPS  D. ICMP

**7.** Which service is best for **global** HTTP traffic with edge caching?  
A. Standard LB  B. App Gateway  C. Front Door  D. Traffic Manager

**8.** A user reports slow DNS failover with Traffic Manager after a regional outage. Why?  
A. Traffic Manager is broken  B. DNS TTL caching at the client  C. Anycast issue  D. App Gateway in the way

**9.** Which subnet rule applies to Application Gateway v2?  
A. Must be `/29` or larger  B. Must be empty (no other AppGw versions allowed)  C. Dedicated subnet, /24 recommended, can coexist with other AppGw versions  D. Cannot have NSG

**10.** You need to balance internal traffic between 3 VMs in a VNet, no public exposure. Best?  
A. Public Standard LB  B. Internal Standard LB  C. App Gateway public  D. Front Door

---

### Answers

1. **C** — App Gateway is L7.  
2. **False** — Front Door is global.  
3. **D** — Traffic Manager is DNS-only.  
4. **B** — App Gateway WAF v2.  
5. **B** — Source IP affinity (3-tuple).  
6. **C** — TCP, HTTP, HTTPS.  
7. **C** — Front Door for global HTTP.  
8. **B** — DNS TTL means clients keep using cached records.  
9. **C** — Dedicated subnet; AppGw v2 subnets can host App Gateway resources.  
10. **B** — Internal Standard LB.

---

**Next up:** [04-Network-Connectivity.md](./04-Network-Connectivity.md)
