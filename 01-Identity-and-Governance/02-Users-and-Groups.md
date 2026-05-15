# Users and Groups

## The Big Picture

If Entra ID is the lobby of the building, **users are the people walking in** and **groups are the teams they belong to**. As an Azure admin, 80% of your identity work is creating, organizing, and disabling these two.

You'll learn to:
- Create users one at a time and in bulk
- Build security groups and Microsoft 365 groups
- Set up dynamic membership rules
- Manage external (guest) users via B2B
- Handle the difference between deletion, disabling, and the recycle bin

---

## Real-World Analogy: New Hire Day at a School

Picture a school on the first day of term:
- Each **student** = a user
- Each **class section** (like 9-A, 9-B) = a security group
- Each **club** (Chess, Robotics) = a Microsoft 365 group with shared mailbox + chat
- The **office staff who add new students** = admins with the User Administrator role
- The **automatic class assignment based on age** = dynamic group membership

Now multiply that school by ten thousand. That's why automation matters.

---

## User Types in Detail

### 1. Cloud-only (Internal) Users
Created directly in Entra ID. They have a UPN like `aisha@contoso.onmicrosoft.com` or a custom verified domain like `aisha@contoso.com`.

### 2. Directory-Synchronized Users
Created in your **on-prem AD**, then sync'd up by **Entra Connect**. The source of authority is on-prem — most attributes are read-only in the cloud portal. You'll see a little badge that says "Windows Server AD" next to them.

### 3. Guest Users (External)
Invited via **B2B collaboration**. Their UPN looks weird because it includes their original tenant: `aisha_externalcorp.com#EXT#@contoso.onmicrosoft.com`. They authenticate against *their* home tenant, not yours.

> **Exam tip:** When you invite a guest, an invitation email is sent. They click "Accept", complete MFA if required, and then they exist in your directory as a user object with `userType = Guest`.

---

## Creating Users — The Three Ways

### Way 1: Portal (one at a time)

`Microsoft Entra ID → Users → New user → Create new user`

You provide:
- User principal name (the login)
- Display name
- Password (auto-generated or set manually)
- Optional: groups, role assignments, job info

### Way 2: Bulk CSV Upload

`Microsoft Entra ID → Users → Bulk operations → Bulk create`

Download the CSV template. Fill it. Upload it. Done.

```csv
Name [displayName] Required,User name [userPrincipalName] Required,Initial password [passwordProfile] Required,...
Aisha Khan,aisha@contoso.com,TempP@ssw0rd!,...
Vikram Mehta,vikram@contoso.com,TempP@ssw0rd!,...
```

> **Exam trap:** Bulk operations log you a result file with success/fail per row. Always download it.

### Way 3: Azure CLI / PowerShell

```bash
# CLI — single user
az ad user create \
  --display-name "Vikram Mehta" \
  --user-principal-name vikram@contoso.com \
  --password 'StrongP@ss123!'

# PowerShell — single user
$pwProfile = @{ Password = "StrongP@ss123!" }
New-MgUser -DisplayName "Vikram Mehta" `
  -UserPrincipalName "vikram@contoso.com" `
  -MailNickname "vikram" `
  -PasswordProfile $pwProfile `
  -AccountEnabled
```

---

## Deleting vs. Disabling vs. Soft-Delete

| Action | What happens | Reversible? |
|--------|--------------|-------------|
| **Disable** (block sign-in) | Account exists but cannot log in | Yes, just re-enable |
| **Delete** | Moved to **Deleted users** (soft-delete) | Yes, within **30 days** |
| **Permanent delete** | Gone forever | No |

> When an employee leaves, the recommended pattern is: **block sign-in → wait some review period → delete**. The 30-day soft-delete window is your safety net.

---

## Groups in Detail

### Group Types

| Type | Use it for | Has a mailbox? |
|------|-----------|----------------|
| **Security** | Granting permissions to apps and resources | No |
| **Microsoft 365** | Collaboration (Outlook, Teams, SharePoint) | Yes |
| **Distribution List** | Email distribution only | Yes (legacy) |
| **Mail-enabled Security** | Permissions + email | Yes (legacy) |

### Membership Types

- **Assigned** — admin manually adds and removes members
- **Dynamic User** — Entra evaluates rules and updates membership automatically
- **Dynamic Device** — same, for devices

> **Important:** A group can be EITHER assigned OR dynamic. You can't switch back and forth without re-creating it. Plan ahead.

### Dynamic Group Rule Examples

```text
# All Finance department users
user.department -eq "Finance"

# All users in India whose title contains "Engineer"
user.country -eq "IN" -and user.jobTitle -contains "Engineer"

# All Windows 11 devices
device.deviceOSType -eq "Windows" -and device.deviceOSVersion -startsWith "10.0.22"
```

> Dynamic groups need **Entra ID P1**. The rule re-evaluates whenever attributes change — usually within minutes.

---

## Creating a Group — Quick Examples

### Portal
`Entra ID → Groups → New group → choose type, name, membership type → Create`

### Azure CLI
```bash
# Create a security group
az ad group create --display-name "Finance Team" --mail-nickname "financeteam"

# Add a user to a group
az ad group member add --group "Finance Team" --member-id <user-object-id>

# List members
az ad group member list --group "Finance Team" --output table
```

### PowerShell (Microsoft Graph)
```powershell
New-MgGroup -DisplayName "Finance Team" `
            -MailNickname "financeteam" `
            -SecurityEnabled `
            -MailEnabled:$false `
            -GroupTypes @()
```

---

## External Identities (B2B Collaboration)

Inviting an external partner is a 4-step flow:

1. **You** invite their email address from `Entra ID → Users → New user → Invite external user`
2. **They** receive an email and click the redemption link
3. **They** complete MFA (if required) using their **home tenant's** credentials
4. **They** appear as a user object in your directory with `userType = Guest`

### Cross-tenant Access Settings

This is the high-level "should we even allow this collab?" toggle. You can:
- **Allow** B2B with all external orgs (default)
- **Block** specific orgs
- **Customize per-org** — e.g., allow Contoso to use their MFA claim, but require Fabrikam to re-MFA

### B2B vs B2C — don't confuse them

- **B2B** = inviting business partners. Uses guest accounts in your tenant.
- **B2C** = customer-facing identity for your apps (e.g., a retail app where customers sign up with Google/Facebook). Lives in a *separate* B2C tenant.

> **Exam trap:** If the question is about **employees from another company**, that's B2B. If it's about **end customers signing up for your e-commerce app**, that's B2C.

---

## Administrative Units (AUs)

Big tenants need delegation. AUs let you slice the tenant into subsets so a sub-admin can manage *only* their slice.

> **Analogy:** Think of a multinational where the "India HR Admin" can only reset passwords for India employees, not Singapore ones. That's an Administrative Unit.

You can scope built-in roles (User Administrator, Helpdesk Administrator, etc.) to a specific AU.

---

## Common Pitfalls

1. **Soft-deleted users still count for some things.** They don't take licenses but they may show up in audit logs.
2. **Adding a user to a Microsoft 365 group adds them to Teams/SharePoint too.** This often surprises admins.
3. **Dynamic group rules don't apply retroactively to onPrem-sync'd attributes that aren't sync'd.** Make sure the attribute is being sync'd up.
4. **Guest users count toward your directory object limit.** Especially relevant in Free tier (500K objects).
5. **Password writeback is enabled in Entra Connect, not in Entra ID.** This is a configuration on the sync server itself.

---

## Quiz: Users and Groups

**1.** A user account was deleted yesterday. Within how many days can you restore it?  
A. 7  B. 14  C. 30  D. 90

**2.** You want to grant access to an Azure subscription. Which group type should you use?  
A. Distribution list  B. Mail-enabled security  C. Security  D. Microsoft 365

**3.** True or False: You can change a group's membership type from Assigned to Dynamic without recreating it.

**4.** What is the UPN format of a B2B guest user from `partner.com` invited into `contoso.onmicrosoft.com`?  
A. `user@partner.com`  
B. `user_partner.com#EXT#@contoso.onmicrosoft.com`  
C. `user@contoso.onmicrosoft.com`  
D. `user.guest@contoso.com`

**5.** Which feature lets you delegate admin rights to **only the users in the Mumbai office**?  
A. Conditional Access  B. Administrative Unit  C. Custom Role  D. Subscription scope

**6.** Which method is best to create 500 users at once?  
A. Portal one-by-one  B. Bulk CSV upload  C. PowerShell loop  D. Either B or C

**7.** A user reports they cannot reset their password even though SSPR is enabled. What is the most likely cause?  
A. They haven't registered authentication methods  
B. They are a guest user  
C. The tenant is on Free  
D. They are a hybrid user

**8.** True or False: Distribution lists in Entra ID can be used to grant Azure RBAC roles.

**9.** Which is the correct rule syntax for a dynamic group containing all users in the Finance department?  
A. `user.department = "Finance"`  
B. `user.department -eq "Finance"`  
C. `user.dept eq "Finance"`  
D. `dept = Finance`

**10.** Your company is building a customer-facing mobile app where users sign up with Google or Facebook. Which Entra service do you use?  
A. Entra ID with B2B  B. Entra External ID / B2C  C. Conditional Access  D. PIM

---

### Answers

1. **C** — 30 days soft-delete window.  
2. **C** — Security groups for permissions. M365 groups bring a mailbox/Teams along, often unwanted.  
3. **False** — Membership type is fixed at creation.  
4. **B** — That weird `#EXT#` format.  
5. **B** — Administrative Units scope admin roles to a subset of users.  
6. **D** — CSV bulk import or scripting both work great.  
7. **A** — User must register methods first at `aka.ms/ssprsetup`.  
8. **False** — Distribution lists are for email only; cannot be used for RBAC.  
9. **B** — KQL-like syntax with `-eq`.  
10. **B** — B2C (now part of Entra External ID) is for consumer-facing scenarios.

---

**Next up:** [03-Subscriptions-and-Governance.md](./03-Subscriptions-and-Governance.md)
