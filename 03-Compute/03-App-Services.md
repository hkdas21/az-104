# Azure App Service

## The Big Picture

App Service is Azure's **fully-managed web hosting platform** (PaaS). You hand it your code or container, and it runs, scales, and keeps the OS patched for you. No VMs to manage, no IIS to configure, no Linux daemons to babysit.

Use it for:
- Web apps (any language: .NET, Java, Node.js, PHP, Python, Ruby, Go)
- REST APIs
- Mobile back-ends
- Scheduled jobs (WebJobs)
- Static and dynamic sites

---

## Real-World Analogy: A Cloud Kitchen

Imagine you want to start a restaurant but don't want to deal with leases, plumbing, or chefs:
- A **cloud kitchen** rents you a slot, the equipment, the staff
- You bring the **recipe** (your app code) and they cook it
- They handle the **dishwashing** (OS patching) and **inventory** (scaling)
- You pay per **slot you reserved** (App Service Plan), not per meal made

That's App Service.

---

## Anatomy: App Service Plan + App Service

You **always** have these two layers:

```
App Service Plan (the compute)
├── App Service (Web App #1)
├── App Service (Web App #2)
└── App Service (Function App)
```

- The **App Service Plan** is the actual VM(s) running underneath. You pay for it.
- The **App Service** is your individual app definition. Multiple apps can share one plan.

---

## Pricing Tiers

| Tier | Best for | Key features |
|------|---------|--------------|
| **Free / Shared** | Demos, dev | No SLA, shared infra, no custom domain (Free) |
| **Basic** | Dev/test | Dedicated VMs, manual scale, no slots |
| **Standard** | Production | Auto-scale, 5 slots, custom domain + SSL |
| **Premium v3** | Production at scale | Faster CPUs, 20 slots, VNet integration, zone redundancy |
| **Isolated v2 (App Service Environment)** | Compliance, network isolation | Dedicated VNet, single-tenant |

> **Exam tip:** Deployment slots, VNet integration, and zone redundancy first appear at **Standard** or **Premium**. Free/Shared have severe limits.

---

## Operating Systems

App Service runs apps on **Windows** or **Linux**. Choose at plan creation — cannot be changed afterward.

- **Windows** — supports .NET Framework apps, classic ASP, Java, Node, PHP
- **Linux** — supports modern stacks plus **container deployments**

> **Exam trap:** A single App Service Plan is either all Windows or all Linux. Apps in it inherit that OS.

---

## Deployment Options

### From source control
- **GitHub Actions** (most common)
- **Azure DevOps Pipelines**
- **Bitbucket** (legacy)

### Direct upload
- ZIP deploy (`az webapp deploy`)
- WAR deploy (Java)
- Run from package (mount the ZIP read-only — fastest)
- FTP/FTPS

### Containers
Deploy a Docker image from:
- Azure Container Registry (ACR)
- Docker Hub
- Any private registry

### CI/CD with deployment slots
1. Slots are full-blown copies of your app at separate URLs
2. Deploy to **staging slot**, test it
3. **Swap** staging → production with zero downtime
4. If broken, **swap back**

---

## Deployment Slots

This is one of the most-tested App Service features.

### What's swapped
On swap, App Service:
- Starts the new app in the target slot
- **Warms it up** (calls `applicationInitialization` URLs)
- Verifies it's healthy
- Atomically routes traffic to it

### Slot-specific settings
By default, **all** app settings and connection strings move with the swap. You can mark specific keys as **slot-sticky** (e.g., a different DB connection string for staging vs. prod) so they stay with the slot.

### Auto-swap
Configure a slot to automatically swap into production once a deployment succeeds. Be careful — no manual gate.

---

## Scaling

### Scale Up (vertical)
Move to a bigger plan tier (e.g., S1 → P1v3). Restarts the app.

### Scale Out (horizontal)
Add more instances behind the load balancer. **Autoscale rules** based on:
- CPU
- Memory
- Queue length
- Custom metrics
- Schedule

### Per App scaling
A single plan can host many apps; you can pin specific apps to a subset of instances (Premium tiers).

---

## Networking

### Inbound — restricting access
- **Access restrictions** — allow/deny by IP range or VNet
- **Private Endpoint** — give the app a private IP in your VNet (best for internal-only apps)
- **App Service Environment (ASE)** — full VNet isolation; entire ASE in your VNet

### Outbound — letting your app reach private resources
- **VNet Integration** — outbound to a delegated subnet so the app can reach private DBs / VMs
- **Hybrid Connections** — TCP tunnel back to on-prem servers
- **Service endpoints / Private endpoints** on the dependent resource

### Custom domains and SSL
- Free **App Service Managed Certificate** for custom domains
- Or import a certificate / use Key Vault references
- Bind the cert to the custom domain

---

## Authentication / Authorization (Easy Auth)

App Service has built-in auth: a few clicks and your app requires sign-in via:
- Microsoft Entra ID
- Microsoft Account
- Google
- Facebook
- Twitter
- Apple
- OpenID Connect (any provider)

Without writing a line of auth code. Enable it on the **Authentication** blade.

---

## Backups

Standard tier and above can take **manual or scheduled backups** of your app + linked database. Stored in a storage account you specify. Useful but not a replacement for source-control + repeatable deploys.

---

## Hands-On: Deploy a Web App via CLI

```bash
# Create plan and app
az appservice plan create \
  --name plan-demo \
  --resource-group rg-app-demo \
  --sku P1V3 \
  --is-linux

az webapp create \
  --resource-group rg-app-demo \
  --plan plan-demo \
  --name myapp-himanshu \
  --runtime "NODE:20-lts"

# Deploy a ZIP
az webapp deploy \
  --resource-group rg-app-demo \
  --name myapp-himanshu \
  --src-path ./app.zip \
  --type zip

# Add a staging slot
az webapp deployment slot create \
  --resource-group rg-app-demo \
  --name myapp-himanshu \
  --slot staging

# Swap slots
az webapp deployment slot swap \
  --resource-group rg-app-demo \
  --name myapp-himanshu \
  --slot staging \
  --target-slot production
```

---

## Common Pitfalls

1. **Mixing Windows and Linux apps in one plan.** Not allowed.
2. **Running too many apps on one plan.** Each app shares the plan's CPU/memory.
3. **Forgetting slot-sticky settings.** Production DB connection accidentally swapped to staging is a classic incident.
4. **Free/Shared tiers have a daily compute quota.** Apps will go offline once exceeded.
5. **VNet integration ≠ private endpoint.** VNet integration is **outbound** for the app. Private endpoint is **inbound** to the app.

---

## Quiz: App Service

**1.** Which App Service tier is the lowest where deployment slots are available?  
A. Free  B. Basic  C. Standard  D. Premium

**2.** True or False: An App Service Plan can host both Windows and Linux apps simultaneously.

**3.** When you swap a staging slot to production, what does Azure do *before* routing traffic?  
A. Restarts production  B. Warms up the new app and verifies health  C. Reboots the VM  D. Backs up the DB

**4.** You want your web app to reach a private SQL Database via a private endpoint. Which feature do you enable on the App Service?  
A. Private endpoint on the App Service  B. VNet integration  C. Hybrid Connection  D. Service endpoint policy

**5.** Which auth provider is **not** natively built into Easy Auth?  
A. Entra ID  B. Google  C. Apple  D. Okta (custom only via OIDC)

**6.** You configure auto-scale on CPU > 70%. What is good practice?  
A. Use a 1-minute window  B. Set a cool-down of several minutes to avoid flapping  C. Skip min count  D. Disable monitoring

**7.** Where is a **slot-sticky** app setting useful?  
A. To keep a setting tied to a slot during swap (e.g., staging DB string)  
B. To prevent a setting from being deleted  
C. To encrypt the value  
D. To replicate it across regions

**8.** Which tier offers full network isolation in your own VNet?  
A. Standard  B. Premium v3  C. Isolated v2 (ASE)  D. Basic

**9.** You deploy from GitHub Actions to App Service. After a faulty release, you want to revert with zero downtime. Best approach:  
A. Restore from backup  B. Re-deploy old commit  C. Swap slots back  D. Recreate app

**10.** A web app has a Daily Quota and is on Free tier. After reaching it:  
A. Azure auto-upgrades the plan  
B. The app stops responding for the rest of the day  
C. The app slows down by 50%  
D. Microsoft sends an email but it keeps running

---

### Answers

1. **C** — Slots from Standard.  
2. **False** — One OS per plan.  
3. **B** — Warmup + health check before swap.  
4. **B** — VNet integration is the outbound path.  
5. **D** — Okta is supported via generic OIDC, not natively pre-listed.  
6. **B** — Cool-down avoids flapping.  
7. **A** — Slot-sticky keeps settings tied to a slot during swap.  
8. **C** — App Service Environment (Isolated v2).  
9. **C** — Slot swap back is the fastest path.  
10. **B** — Free tier stops on quota exhaustion.

---

**Next up:** [04-Containers-and-AKS.md](./04-Containers-and-AKS.md)
