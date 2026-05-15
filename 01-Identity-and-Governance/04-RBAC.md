# Role-Based Access Control (RBAC)

## The Big Picture

RBAC answers the question: **"Who can do what, where?"**

- **Who** = a user, group, service principal, or managed identity
- **What** = a role (a set of allowed actions)
- **Where** = a scope (management group, subscription, resource group, or specific resource)

If you understand this triangle, you understand 90% of RBAC. Master it. The exam loves this topic.

---

## Real-World Analogy: Hotel Keycards

You check into a hotel. Reception hands you a keycard. The card opens:
- The main entrance (the building)
- Your floor (your wing)
- Your specific room
- Maybe the gym, but not the kitchen

That keycard is your **role assignment**. The "kitchen vs. gym" matrix is the **role definition**. The "your room only" part is the **scope**. Same model in Azure.

---

## The Three Components of an RBAC Assignment

### 1. Security Principal (the "Who")
- **User** — a single human
- **Group** — preferred over individual users; easier to manage
- **Service Principal** — an application identity
- **Managed Identity** — a service principal Azure manages for you

### 2. Role Definition (the "What")
A list of permitted (and denied) actions. Think of it as a permission template.

Roles have:
- **`Actions`** — what's allowed (e.g., `Microsoft.Storage/*/read`)
- **`NotActions`** — exceptions to subtract from Actions
- **`DataActions`** — actions on the *data plane* (e.g., reading blob contents)
- **`NotDataActions`** — exceptions to subtract from DataActions

### 3. Scope (the "Where")
- Management group
- Subscription
- Resource group
- Resource

> **Inheritance flows DOWN.** Assign a role at the subscription, and it applies to every RG and resource inside.

---

## Built-in Roles You Must Know Cold

### The Big Four
| Role | Permissions | Common use |
|------|------------|------------|
| **Owner** | Full access INCLUDING managing access | Don't hand out lightly |
| **Contributor** | Full access EXCEPT managing access | Most developers |
| **Reader** | Read-only | Auditors, monitoring tools |
| **User Access Administrator** | Manage access ONLY (no resource ops) | Identity admins |

> **Exam trap:** *Contributor* cannot grant access to others. Only Owner and User Access Administrator can.

### Common Resource-Specific Roles
- **Virtual Machine Contributor** — manage VMs but not the network/storage they sit on
- **Storage Blob Data Reader/Contributor/Owner** — *data plane* access to blobs
- **Storage Account Contributor** — *control plane* — can manage the storage account but not necessarily read the data
- **Network Contributor** — manage networking resources
- **Backup Operator** — manage backups but cannot delete vaults

### Microsoft Entra ID Roles vs Azure (Resource) Roles

This is **the most confused topic in the whole exam**. Pay attention.

| Aspect | Entra ID roles | Azure RBAC roles |
|--------|----------------|------------------|
| Scope | The Entra tenant (and its objects: users, groups, apps) | Azure resources (VMs, storage, etc.) |
| Examples | Global Administrator, User Administrator, Helpdesk Admin | Owner, Contributor, Reader, VM Contributor |
| Where you assign them | `Entra ID → Roles and administrators` | `Resource → Access control (IAM)` |
| Inherit through MGs? | No (uses AUs instead) | Yes |

> If a question says "Aisha needs to reset users' passwords", that's an **Entra ID role** (Helpdesk Administrator). If it says "Aisha needs to start/stop a VM", that's an **Azure RBAC role** (VM Contributor or Operator).

---

## How Permissions Are Calculated

The effective permission = **all Actions** at and above the scope, **minus all NotActions**, **minus all Deny assignments**.

### Deny Assignments
Newer feature. Most users can't create them — they're added by Azure-managed services (like Azure Blueprints, Managed Apps). They **always win** against Allow.

> Order: **Deny > Allow**. There is no "explicit allow overrides explicit deny" in Azure RBAC. Deny is final.

---

## Custom Roles

When built-in roles don't fit, build your own. Custom roles are JSON.

```json
{
  "Name": "VM Operator",
  "Description": "Can start, stop, and restart VMs but cannot create or delete them.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/instanceView/read"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/<sub-id>"
  ]
}
```

### Rules for custom roles
- You can have up to **5,000 custom roles** per tenant (much higher in Azure China)
- Use the `--assignable-scopes` field to control where they can be applied
- You need the **`Microsoft.Authorization/roleDefinitions/write`** permission to create them — Owner or User Access Administrator have this

### Create via CLI
```bash
az role definition create --role-definition role.json
```

---

## Privileged Identity Management (PIM)

PIM is a **P2 feature** that turns "always-on" admin roles into "just-in-time" admin roles.

### Workflow
1. A user is **eligible** for an admin role (not active by default)
2. When they need it, they **activate** the role for a limited time (e.g., 4 hours)
3. Activation can require **MFA**, **justification**, **approval**, or all three
4. After the time expires, they go back to being just a regular user

### Why PIM matters
- Reduces *standing* admin privileges
- Creates an audit trail of activations
- Forces approvals for sensitive operations

### Access Reviews
A periodic prompt that asks "Aisha, you have role X — do you still need it?" Reviewers approve or remove. Required for SOX/ISO compliance in many orgs.

> PIM works for **both** Entra ID roles and Azure RBAC roles. P2 license required.

---

## Hands-On: RBAC via CLI

```bash
# List all role definitions
az role definition list --query "[].{role:roleName,id:name}" --output table

# List role assignments at a scope
az role assignment list --scope "/subscriptions/<sub-id>" --output table

# Assign Reader to a user at a resource group
az role assignment create \
  --assignee aisha@contoso.com \
  --role "Reader" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-prod"

# Remove the assignment
az role assignment delete \
  --assignee aisha@contoso.com \
  --role "Reader" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-prod"
```

---

## Real-World Scenarios

### Scenario 1: Junior Developer
*"Aisha can deploy and modify resources in the dev environment but should never touch production or grant access to others."*

✅ **Solution:** Contributor on the `rg-dev` resource group. Done.

### Scenario 2: Auditor
*"The internal auditor needs to see everything but change nothing."*

✅ **Solution:** Reader at the management group level. Inherits to all subs.

### Scenario 3: VM Power-Cycler
*"The Ops team should restart VMs at night but not create or delete them."*

✅ **Solution:** Custom role with start/stop/restart actions, assigned at the VM resource group.

### Scenario 4: Application reading a Storage Account
*"My web app needs to read blobs from a specific container."*

✅ **Solution:** Managed Identity for the web app + **Storage Blob Data Reader** role on the storage account. No keys.

---

## Common Pitfalls

1. **Storage Account Contributor ≠ access to data.** Control plane ≠ data plane. To read a blob, you need a **Data** role.
2. **Removing an assignment doesn't kick people out instantly.** Tokens cached up to ~1 hour will still work. Don't assume immediate effect.
3. **A user who is Owner of a resource group can grant Owner to anyone else.** This is a privilege escalation risk. Use User Access Administrator + Contributor as a safer split.
4. **Custom roles are not inherited from a parent management group automatically.** They're scoped to the `assignableScopes` you defined.
5. **You cannot assign roles to distribution lists.** Only security groups and M365 groups (and individual principals).

---

## Quiz: RBAC

**1.** Which built-in role allows a user to manage all resources in a subscription but **not** grant access to others?  
A. Owner  B. Contributor  C. Reader  D. User Access Administrator

**2.** You assign **Storage Account Contributor** to a developer. They cannot read blob contents. What's wrong?  
A. They need to be a global admin  
B. Storage Account Contributor is control-plane only; they need a data-plane role  
C. The storage account is offline  
D. RBAC takes 24 hours to apply

**3.** True or False: Azure RBAC roles automatically apply to Microsoft Entra ID administrative tasks like resetting passwords.

**4.** What does PIM let you do that ordinary RBAC does not?  
A. Assign roles  
B. Create custom roles  
C. Make roles eligible (just-in-time) instead of permanently active  
D. Delete roles

**5.** Where in the JSON do you specify the scopes a custom role can be assigned to?  
A. `Actions`  B. `NotActions`  C. `AssignableScopes`  D. `Id`

**6.** A Deny assignment and an Allow assignment apply to the same user for the same operation. Which wins?  
A. Allow  B. Deny  C. The most recent one  D. Neither

**7.** Which is NOT a valid scope for an RBAC assignment?  
A. Management group  B. Subscription  C. Resource group  D. A specific blob inside a container

**8.** Which role can grant access to other users?  
A. Contributor  B. Reader  C. Owner and User Access Administrator only  D. VM Administrator

**9.** You delete a user's role assignment. They report still being able to access the resource minutes later. Why?  
A. RBAC is broken  B. Cached access token is still valid  C. Assignments take 24 hours  D. They have a hidden assignment

**10.** A user activates a PIM role. What happens?  
A. They become Owner permanently  
B. They have the role for the duration they specified, then it expires  
C. The role is granted only after admin approval (always)  
D. Nothing — PIM is for tracking only

---

### Answers

1. **B** — Contributor: do everything except manage access.  
2. **B** — Need Storage Blob Data Reader/Contributor/Owner for data plane.  
3. **False** — Azure RBAC and Entra ID roles are separate systems.  
4. **C** — Just-in-time elevation is PIM's main value.  
5. **C** — `AssignableScopes`.  
6. **B** — Deny always wins.  
7. **D** — You can't scope an RBAC assignment to a specific blob — assignments stop at resource level (the storage account or container, depending on the role).  
8. **C** — Owner and User Access Administrator. Contributor cannot manage access.  
9. **B** — Tokens are cached up to ~1 hour. Often less, but real.  
10. **B** — Time-bound activation. Approval is *optional* depending on policy.

---

**You finished Module 1!** 🎉

Take a break, then head to [Module 2 — Storage](../02-Storage/README.md). The next chunk is heavier on services and lighter on theory.
