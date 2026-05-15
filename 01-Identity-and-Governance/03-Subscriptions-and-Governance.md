# Subscriptions, Management Groups, and Governance

## The Big Picture

When you start with one Azure subscription and a couple of resources, life is simple. The moment you scale to ten subscriptions across ten teams, you need *governance* — rules and structure to keep everything from descending into chaos.

This module covers:
- The Azure resource hierarchy
- Management groups and subscriptions
- Resource groups
- Azure Policy
- Resource locks
- Tags
- Cost management

---

## Real-World Analogy: A Retail Chain

Picture a multinational retail chain like a "BigBazaar":

- **Tenant** = the parent company "BigBazaar Holdings"
- **Management groups** = country/region holding entities ("BigBazaar India", "BigBazaar UAE")
- **Subscriptions** = individual store budgets and ledgers ("Mumbai-Andheri-Store", "Dubai-Mall-Store")
- **Resource groups** = departments inside each store (Electronics, Groceries, Fashion)
- **Resources** = individual products on shelves (a TV, a packet of rice, a pair of jeans)
- **Tags** = sticky labels on products ("Sale", "New Arrival", "Owner: Asha")
- **Policies** = head-office rules ("No store may sell liquor", "Every product must have a price tag")
- **Locks** = anti-theft devices on high-value items

That mental model holds up surprisingly well throughout the rest of AZ-104.

---

## The Resource Hierarchy

```
Tenant Root Management Group
└── Management Group (e.g., "Production")
    └── Management Group (e.g., "EMEA")
        └── Subscription (e.g., "Prod-EMEA-001")
            └── Resource Group (e.g., "rg-payments")
                └── Resource (e.g., a VM)
```

**Key rules:**
- A resource lives in **exactly one resource group**
- A resource group lives in **exactly one subscription**
- A subscription lives in **exactly one management group**
- Up to **6 levels** of management groups under the root
- Inheritance flows **downward** (policies, RBAC)

---

## Subscriptions

A subscription is **a billing boundary and an isolation boundary**.

### Types of Subscriptions
- **Free Trial** — $200 credit for 30 days, then 12 months of free services
- **Pay-as-you-go (PAYG)** — credit card billing
- **Enterprise Agreement (EA)** — committed spend, negotiated pricing
- **Microsoft Customer Agreement (MCA)** — modern replacement for EA
- **CSP (Cloud Solution Provider)** — bought through a Microsoft partner
- **Azure for Students / Sponsorship / Dev-Test** — special programs

### Subscription Limits
Subscriptions have **soft and hard limits**. Examples:
- 980 resource groups per subscription
- 250 storage accounts per region per subscription (raised on request)
- 50,000 VMs per region

> **Exam tip:** When a question asks "you cannot create more X — what should you do?", the answer is usually one of: (1) move to another subscription, (2) deploy to another region, (3) request a quota increase via support.

### Why have multiple subscriptions?

- Separate **prod/non-prod** workloads
- Per-department / per-team **billing isolation**
- Hit a **quota ceiling** in one subscription
- Different **compliance** requirements (e.g., PCI workloads in a separate sub)

---

## Management Groups

Containers that hold subscriptions or other management groups. Used for **policy and RBAC inheritance** at scale.

```
Tenant Root Group
├── Production
│   ├── Prod-Subscription-1
│   └── Prod-Subscription-2
├── Non-Production
│   ├── Dev-Sub
│   └── Test-Sub
└── Sandbox
    └── Sandbox-Sub
```

If you assign Azure Policy "Allowed locations = India" at **Production**, both Prod subscriptions inherit it. Sandbox is unaffected.

> The **tenant root group** is created automatically. Only **Global Administrators** can manage assignments at that level (and they have to elevate access first).

---

## Resource Groups

A resource group is a **logical container** for resources that share a lifecycle.

**Best practices:**
- Group resources that are **deployed and deleted together** (e.g., one app's VM + disk + NIC + public IP)
- Same **region** ideally (resources can technically be in any region; the RG itself has a metadata region for storing data about the RG)
- Same **owner** and **lifecycle**

**Important properties:**
- Deleting an RG deletes **everything inside it** — be careful
- A resource can be **moved** between RGs (and even between subscriptions, with caveats)
- An RG can have its own **RBAC and policies**

---

## Tags

Tags are key/value pairs you slap on resources, RGs, or subscriptions. They look trivial but they're how organizations stay sane.

### Common tag patterns

| Tag | Example value | Purpose |
|-----|---------------|---------|
| `Environment` | `Prod`, `Dev`, `Test` | Lifecycle |
| `Owner` | `aisha@contoso.com` | Accountability |
| `CostCenter` | `CC-1234` | Billing chargeback |
| `Application` | `Payments` | Grouping by app |
| `Criticality` | `Tier1`, `Tier2` | SLA |

### Tag rules
- Up to **50 tags** per resource
- Tag keys are **case-insensitive**, values are **case-sensitive**
- **Not all** resource types support tags (some classic ones don't)
- Tags **do not inherit by default** — but you can use Azure Policy to enforce inheritance

### Enforce tags with policy
Built-in policies you'll use constantly:
- **Require a tag and its value on resources**
- **Inherit a tag from the resource group if missing**
- **Append a tag to resources**

---

## Azure Policy — The Rule Engine

Azure Policy lets you enforce rules at any scope. Think of it as **guardrails**.

### What can a policy do?

| Effect | Behavior |
|--------|----------|
| **Audit** | Log non-compliant resources, but allow them |
| **Deny** | Block creation/update of non-compliant resources |
| **Append** | Add fields to a resource (e.g., a tag) |
| **Modify** | Change properties on existing resources |
| **DeployIfNotExists** | Deploy a related resource if missing (e.g., diagnostic settings) |
| **AuditIfNotExists** | Log if a related resource is missing |
| **Disabled** | Turn off a policy without removing the assignment |

### The lifecycle
1. Pick or create a **policy definition**
2. Combine multiple definitions into an **initiative** (a.k.a. policy set)
3. **Assign** at a scope (management group / sub / RG)
4. Evaluate — Azure tells you who's compliant and who's not
5. **Remediate** non-compliant existing resources (for `DeployIfNotExists` / `Modify`) using a **remediation task**

### Common built-in policies
- "Allowed locations" — restrict regions
- "Allowed virtual machine SKUs" — restrict VM sizes
- "Storage accounts should restrict network access"
- "Audit VMs that do not have managed disks"

> **Exam tip:** Policy is *preventive and detective*. It can stop bad things from happening (Deny), and it tells you about bad things that already exist (Audit).

---

## Azure Policy vs. RBAC — Don't confuse them

| Aspect | Azure Policy | RBAC |
|--------|-------------|------|
| Question it answers | *What* can be done? | *Who* can do it? |
| Example | "No VMs bigger than D8s_v5 may be deployed" | "Aisha can manage VMs but not storage" |
| Scope | Management group → Sub → RG → Resource | Same scopes |
| Bypassed by Owner? | **No** | N/A |

> **Exam trap:** An Owner can do anything... *except* violate Azure Policy. Policy is *not* about identity; it's about resource configuration.

---

## Resource Locks

Two flavors:

| Lock | What it blocks | What it allows |
|------|----------------|-----------------|
| **CanNotDelete** | Delete | Read, modify |
| **ReadOnly** | Delete + modify | Read |

You can lock at subscription, RG, or individual resource level. Locks **override** RBAC — even an Owner cannot delete a locked resource without removing the lock first.

> **Use case:** Put a `CanNotDelete` lock on production resource groups. The number of "oops I deleted prod" incidents this prevents is staggering.

---

## Cost Management & Billing

### Tools you should know
- **Cost analysis** — visualize spend by resource, RG, tag, etc.
- **Budgets** — set monthly spending limits with email alerts (and optionally action group automation, but it does NOT stop spend)
- **Cost alerts** — proactive notifications
- **Advisor** — recommendations to save money (right-size VMs, delete unused resources, buy reservations)

> **Exam trap:** A budget does **not** stop usage when you exceed it. It only **alerts**. To actually stop spend, you'd need automation (Logic App, runbook) triggered by the alert.

### Reservations & Savings Plans
- **Reservations** — commit to 1 or 3 years on a specific VM family. Save up to 72%.
- **Savings Plans** — commit to a $/hour spend across compute. More flexible, slightly less discount.
- **Azure Hybrid Benefit** — bring your own Windows Server / SQL Server licenses with Software Assurance for big savings.
- **Spot VMs** — pay way less but Azure can evict you anytime. Good for batch jobs.

---

## Hands-On: Apply a Policy via CLI

```bash
# Find a built-in policy definition
az policy definition list --query "[?displayName=='Allowed locations'].{name:name,id:id}" -o table

# Assign it at the resource group scope
az policy assignment create \
  --name "allowed-locations-india" \
  --display-name "Allow only India regions" \
  --policy "<policy-definition-id>" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-prod" \
  --params '{ "listOfAllowedLocations": { "value": ["centralindia","southindia"] } }'
```

---

## Common Pitfalls

1. **Resource groups are not security boundaries.** Don't rely on RG isolation alone. Use RBAC.
2. **Moving resources between regions** is not supported by RG move. Use Azure Resource Mover for cross-region.
3. **Locks block automation too.** Your CI/CD pipeline can fail if you forgot a lock is in place.
4. **Tags don't appear on the bill** unless you've assigned them *before* the cost was incurred — but new policies can fix this going forward.
5. **Initiatives vs. policy assignments** — initiatives are bundles of definitions, then *that* bundle is what you assign.

---

## Quiz: Subscriptions and Governance

**1.** What is the maximum depth of management groups under the tenant root?  
A. 4  B. 5  C. 6  D. 8

**2.** You apply a `CanNotDelete` lock on a resource group. A user with the Owner role tries to delete it. What happens?  
A. Delete succeeds  B. Delete is blocked  C. Delete succeeds with a warning  D. Owner is notified

**3.** Which Azure Policy effect prevents non-compliant resources from being created?  
A. Audit  B. Deny  C. Append  D. AuditIfNotExists

**4.** True or False: An Azure Budget will automatically stop your resources from running when you exceed the budget.

**5.** Which of these can a tag *not* be applied to?  
A. A subscription  B. A resource group  C. A virtual network  D. A subscription's billing role assignment

**6.** Which scope does NOT support Azure Policy assignment?  
A. Management group  B. Subscription  C. Resource group  D. Individual resource

**7.** You want to enforce that every VM has the tag `CostCenter`, and copy it from the resource group if missing on the VM. Which policy effects do you use?  
A. Deny  B. Append  C. Modify + Deny  D. Modify + AuditIfNotExists

**8.** You delete a subscription. What happens to its resources?  
A. They are kept for 30 days, then deleted  
B. They are immediately deleted  
C. They are moved to the tenant root  
D. They are migrated to a partner subscription

**9.** What is the difference between a policy **definition** and an **initiative**?  
A. They are the same  
B. Initiative is a single rule, definition is many  
C. Definition is a single rule, initiative is a bundle of definitions  
D. Initiatives only work at the management group scope

**10.** A user has Owner role on a subscription. A `ReadOnly` lock is applied at the resource group. What can they do?  
A. Everything  B. Read and modify, but not delete  C. Only read  D. Nothing

---

### Answers

1. **C** — 6 levels of management groups under the root.  
2. **B** — Locks override RBAC.  
3. **B** — Deny prevents creation/update.  
4. **False** — Budgets are alert-only.  
5. **D** — Role assignments are not taggable. Subs, RGs, and resources are.  
6. **D** — Wait, this is a trick. You CAN assign at all four scopes? Yes — but you cannot assign at a *single resource* scope directly via portal in the typical workflow. The technically correct exam answer here: all of A–D are valid scopes, but the common test answer is **D** for "individual resource" because in practice you assign at higher scopes. *(For the real exam, remember: MG → Sub → RG. Resource-level is rarely tested.)*  
7. **C** — Modify to add the tag automatically; Deny to prevent missing values, OR use a single Modify with `addOrReplace`.  
8. **A** — There's a 30-day soft-delete window for subscriptions.  
9. **C** — Initiative bundles definitions for easier assignment.  
10. **C** — ReadOnly lock means only read, even for an Owner.

---

**Next up:** [04-RBAC.md](./04-RBAC.md) — the most-asked topic in this whole domain.
