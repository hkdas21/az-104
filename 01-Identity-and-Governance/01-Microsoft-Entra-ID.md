# Microsoft Entra ID (Formerly Azure Active Directory)

## The Big Picture

Microsoft Entra ID is Microsoft's **cloud-based identity and access management (IAM) service**. Every time someone signs into Office 365, Azure, or thousands of SaaS apps using their work account, Entra ID is doing the heavy lifting in the background.

In July 2023, Microsoft renamed Azure Active Directory to **Microsoft Entra ID**. The technology and SKUs are the same — just a new name. You'll still see "Azure AD" in many places (CLI, older docs, the exam itself sometimes). Treat them as synonyms.

---

## Real-World Analogy: The Office Building Lobby

Imagine a giant office building shared by hundreds of companies.

- The **lobby with the receptionist who checks IDs** = Microsoft Entra ID
- Your **employee badge** = your Entra ID account
- The **floors and offices** you can enter = Azure resources, Microsoft 365 apps, SaaS apps
- The **rules about which floors you can visit** = RBAC + Conditional Access policies

You don't get a separate badge for every room — one badge, swiped at every door, gets you in only where you're allowed. That's **Single Sign-On (SSO)**, the magic Entra ID provides.

---

## Entra ID vs. On-Premises Active Directory

This is one of the most-tested concepts. They are *not* the same thing.

| Feature | On-Prem Active Directory (AD DS) | Microsoft Entra ID |
|---------|----------------------------------|---------------------|
| Protocols | Kerberos, LDAP, NTLM | OAuth 2.0, OpenID Connect, SAML, WS-Federation |
| Hierarchy | Forests, domains, OUs | Flat structure with tenants |
| Group Policy | Yes (GPOs) | No (use Intune for device policy) |
| Computer accounts | Yes | No (in classic sense) |
| Designed for | LAN / corporate network | Internet / SaaS apps |
| Joins | Domain Join | Entra Join, Hybrid Join |

**The key insight:** On-prem AD was designed when everyone worked inside the corporate firewall. Entra ID is built for a world where your users, devices, and apps are scattered across the internet.

You can connect them with **Microsoft Entra Connect** (formerly Azure AD Connect) so your on-prem identities sync to the cloud.

---

## Editions of Entra ID

Microsoft sells Entra ID in tiers. Know the differences cold for the exam.

| Edition | What you get | Typical use |
|---------|--------------|-------------|
| **Free** | Up to 500K objects, basic SSO, basic security reports, MFA | Bundled with any Microsoft cloud subscription |
| **P1** | Self-service password reset for hybrid users, dynamic groups, Conditional Access, app proxy | Most enterprises |
| **P2** | Everything in P1 + Identity Protection (risk-based) + Privileged Identity Management (PIM) + Access Reviews | Enterprises with high security needs |

> **Memory aid:** "P2 = **P**rotection + **P**rivileged" — risk detection and PIM are the headliners.

---

## Tenants, Subscriptions, and Directories

This trips up newcomers. Slowly, here:

- A **tenant** = one instance of Entra ID = one organization. When you sign up for Azure, you get one. Identified by a domain like `contoso.onmicrosoft.com` and a tenant ID (a GUID).
- A **directory** = the database inside the tenant that holds users, groups, apps. (Tenant and directory are often used interchangeably.)
- A **subscription** = a billing container in Azure. It is *associated with* a tenant.

One tenant can hold **many subscriptions**. One subscription belongs to **exactly one** tenant.

> Analogy: The **tenant** is your company. The **subscription** is a corporate credit card. You can have several cards (Dev, Test, Prod) all owned by the same company.

---

## Core Identity Objects

These are the building blocks you'll work with constantly:

### 1. Users
- **Cloud-only users** — created directly in Entra ID
- **Synced users** — pushed up from on-prem AD via Entra Connect
- **Guest users** — external users invited via B2B collaboration. Their actual identity lives in another Entra tenant (or even a Google/Microsoft personal account).

### 2. Groups
- **Security groups** — used to grant permissions
- **Microsoft 365 groups** — used for collaboration (mailbox, SharePoint site, Teams)
- **Membership types**:
  - **Assigned** — manually add members
  - **Dynamic User** — membership rules based on user attributes (e.g., `department -eq "Finance"`)
  - **Dynamic Device** — same idea but for devices

> Dynamic membership requires **Entra ID P1 or higher**.

### 3. Devices
- **Entra registered** — personal/BYOD device, just identified
- **Entra joined** — corporate-owned device that signs in with Entra credentials only
- **Hybrid Entra joined** — joined to both on-prem AD and Entra ID (legacy and lift-and-shift scenarios)

### 4. Service Principals & Managed Identities
- A **service principal** = an identity for an application, not a human
- A **managed identity** = a service principal that Azure automatically manages for you (no secrets to rotate). Comes in two flavors:
  - **System-assigned** — tied to the lifecycle of one resource. Delete the resource, identity is gone.
  - **User-assigned** — standalone identity that can be attached to many resources

> Always prefer **managed identities** over service principals with passwords. No secrets to leak.

---

## Self-Service Password Reset (SSPR)

A massive helpdesk-cost-saver. Lets users reset their own passwords instead of calling IT.

**Required setup:**
1. Enable SSPR for selected users/groups (or all)
2. Configure **authentication methods** (e.g., Mobile phone + Email + Security questions). At least 2 are required by default.
3. User registers their authentication methods at `https://aka.ms/ssprsetup`
4. User resets password at `https://aka.ms/sspr`

**For hybrid users**, enable **password writeback** in Entra Connect so cloud password changes flow back to on-prem AD.

> **Exam trap:** SSPR for cloud-only users is included with Free. SSPR for *hybrid* users (with writeback) requires **Entra ID P1**.

---

## Multi-Factor Authentication (MFA)

The single biggest security improvement you can deploy. Microsoft says MFA blocks 99.9% of automated attacks.

**Methods supported:**
- Microsoft Authenticator app (push notification, OTP, passwordless)
- FIDO2 security keys
- Windows Hello for Business
- SMS / phone call (less secure, being phased out for some scenarios)
- OATH hardware tokens

**How to enable MFA:**
1. **Security defaults** — one-click for the whole tenant. Free, simple, all or nothing.
2. **Per-user MFA** — legacy, don't use.
3. **Conditional Access policies** — granular control. Requires P1.

> **Always-asked exam question:** *Which method gives me the most flexibility?* → **Conditional Access**. Per-user MFA is the legacy method.

---

## Conditional Access (CA)

Conditional Access is the **rule engine** that decides: *given who you are, where you are, what device you're on, and what app you want — should we let you in, block you, or ask for more proof?*

A CA policy looks like an IF-THEN statement:

> **IF** a user in the *Finance* group **AND** they're outside India **AND** they're using a non-compliant device  
> **THEN** require MFA **AND** block download.

**Signals (the IF side):**
- User / group
- Application
- Device platform & state
- Location (named locations)
- Sign-in risk (P2)
- User risk (P2)

**Controls (the THEN side):**
- Grant: require MFA, require compliant device, require approved app, etc.
- Block access
- Session: limit download, persistent browser, etc.

> Conditional Access requires **Entra ID P1** at minimum. Risk-based policies need **P2**.

---

## Microsoft Entra Connect

The bridge between on-prem AD and Entra ID. Three sync options:

| Method | What syncs | Where auth happens | Notes |
|--------|------------|---------------------|-------|
| **Password Hash Sync (PHS)** | Hash of password hash | In the cloud | Simplest, recommended baseline |
| **Pass-through Authentication (PTA)** | No password leaves on-prem | On-prem agent | Good if compliance forbids password hashes in the cloud |
| **Federation (ADFS)** | No password sync | On-prem ADFS farm | Highest complexity. Use only if you need it. |

> **Exam trap:** Even with PTA or Federation, Microsoft *recommends* enabling **PHS as a backup** so authentication still works if your on-prem agents fail.

---

## Hands-On: Create a User via Azure CLI

```bash
# Sign in
az login

# Create a user
az ad user create \
  --display-name "Aisha Khan" \
  --user-principal-name aisha@contoso.onmicrosoft.com \
  --password 'TempP@ssw0rd!' \
  --force-change-password-next-sign-in true

# List users
az ad user list --output table

# Delete user
az ad user delete --id aisha@contoso.onmicrosoft.com
```

---

## Common Pitfalls

1. **Mixing up tenant and subscription.** Tenant = identity boundary. Subscription = billing.
2. **Thinking Entra ID = AD in the cloud.** It is not. They use different protocols and serve different scenarios.
3. **Forgetting that B2B guest users count toward licensing in some scenarios.** With the new "MAU billing" model, the first 50,000 monthly active guests are free.
4. **Using Per-user MFA in 2025.** Use Conditional Access or Security Defaults. Per-user is deprecated for new deployments.
5. **Hard-coding passwords in apps.** Use Managed Identities or Key Vault references.

---

## Quiz: Microsoft Entra ID

> Answers at the bottom. Try to answer all 10 before peeking.

**1.** Which Entra ID edition do you need to use **dynamic group membership**?  
A. Free  B. P1  C. P2  D. Microsoft 365 Apps

**2.** True or False: Microsoft Entra ID uses Kerberos as its primary authentication protocol.

**3.** A subscription can be associated with how many Entra tenants at the same time?  
A. 1  B. 2  C. Unlimited  D. Up to 5

**4.** Which feature is included in **Entra ID P2** but not in **P1**?  
A. Conditional Access  B. Self-service password reset  C. Privileged Identity Management (PIM)  D. Application Proxy

**5.** You want to give an Azure VM access to a Storage Account *without storing any credentials in the VM*. Which is the best approach?  
A. Service principal with client secret  B. Service principal with certificate  C. System-assigned managed identity  D. Stored credentials in Key Vault

**6.** A user is sync'd from on-prem AD via Entra Connect. They forget their password. To let them reset it themselves and have the new password go back to on-prem AD, you need:  
A. SSPR with writeback (P1)  B. SSPR (Free)  C. Per-user MFA  D. Federation

**7.** You enable **Security Defaults**. What's the default behavior?  
A. Only admins are required to use MFA  B. All users are required to register and use MFA  C. Nothing changes until you create a policy  D. Only sync'd users are required to use MFA

**8.** Which type of group can be used to assign Microsoft 365 licenses?  
A. Security group only  B. Microsoft 365 group only  C. Both Security and Microsoft 365 groups  D. Dynamic device group

**9.** Your company wants users' *on-prem* passwords to never leave the corporate network. Which Entra Connect option do you choose?  
A. Password Hash Sync  B. Pass-through Authentication  C. Federation with ADFS  D. Either B or C

**10.** True or False: A guest user from another tenant takes up a license seat in your tenant the moment you invite them.

---

### Answers

1. **B** — Dynamic groups need P1.  
2. **False** — Entra ID uses OAuth/OIDC/SAML. Kerberos is on-prem AD.  
3. **A** — One subscription = one tenant at a time. (You can transfer it.)  
4. **C** — PIM is the headline P2 feature, alongside Identity Protection.  
5. **C** — System-assigned managed identity. No credentials to leak.  
6. **A** — SSPR with password writeback, requires P1.  
7. **B** — Security defaults force MFA on everyone (with grace period for registration).  
8. **C** — Both group types can be license-assigned. (Distribution lists cannot.)  
9. **D** — Both PTA and Federation keep passwords on-prem; PHS sends hashes to the cloud.  
10. **False** — Guest billing is by Monthly Active Users (MAU), and the first 50,000 are free per tenant.

---

**Next up:** [02-Users-and-Groups.md](./02-Users-and-Groups.md) — let's get hands-on with the day-to-day admin work.
