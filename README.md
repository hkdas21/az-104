# AZ-104: Microsoft Azure Administrator — Complete Course

Welcome! This repository is your one-stop study companion for the **AZ-104 Microsoft Azure Administrator** certification. Everything here is in plain Markdown so you can read it anywhere — VS Code, GitHub, your phone, or print it out and scribble on it.

> *"The cloud is just someone else's computer."* — But it turns out **managing** that someone else's computer is a serious skill, and that's exactly what AZ-104 certifies.

---

## What is AZ-104?

The AZ-104 exam validates that you can:

- Manage **Azure identities and governance**
- Implement and manage **storage**
- Deploy and manage **Azure compute resources**
- Implement and manage **virtual networking**
- **Monitor and maintain** Azure resources

You should think of an Azure Administrator like the **building manager of a giant skyscraper**. You don't design the building, you don't run the businesses inside, but you do make sure:
- Only the right people have keys (Identity)
- The plumbing, electricity, and storage rooms work (Storage)
- Tenants have their offices and equipment (Compute)
- The hallways, elevators, and external roads connect everything (Networking)
- The fire alarms, smoke detectors, and backup generators are working (Monitoring & Backup)

---

## Exam Domain Weighting (Official Microsoft Learn — 2025)

| # | Domain | Weight |
|---|--------|--------|
| 1 | Manage Azure identities and governance | 20–25% |
| 2 | Implement and manage storage | 15–20% |
| 3 | Deploy and manage Azure compute resources | 20–25% |
| 4 | Implement and manage virtual networking | 15–20% |
| 5 | Monitor and maintain Azure resources | 10–15% |

> Weighting changes occasionally. Always cross-check with the [official Microsoft Learn AZ-104 page](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-104/) before exam day.

---

## Repository Structure

```
AZ-104-Course/
├── README.md                          ← You are here
├── 01-Identity-and-Governance/
│   ├── README.md
│   ├── 01-Microsoft-Entra-ID.md
│   ├── 02-Users-and-Groups.md
│   ├── 03-Subscriptions-and-Governance.md
│   └── 04-RBAC.md
├── 02-Storage/
│   ├── README.md
│   ├── 01-Storage-Accounts.md
│   ├── 02-Blob-Storage.md
│   ├── 03-Azure-Files.md
│   └── 04-Storage-Security.md
├── 03-Compute/
│   ├── README.md
│   ├── 01-Virtual-Machines.md
│   ├── 02-VM-Availability-and-Scaling.md
│   ├── 03-App-Services.md
│   └── 04-Containers-and-AKS.md
├── 04-Networking/
│   ├── README.md
│   ├── 01-Virtual-Networks.md
│   ├── 02-NSG-and-Routing.md
│   ├── 03-Load-Balancing.md
│   └── 04-Network-Connectivity.md
├── 05-Monitoring-and-Backup/
│   ├── README.md
│   ├── 01-Azure-Monitor.md
│   └── 02-Backup-and-Recovery.md
└── practice-quizzes/
    └── (Each module file contains its own quiz at the end)
```

---

## How to Use This Course

### If you have **4 weeks** until the exam
- **Week 1:** Identity & Governance + Storage
- **Week 2:** Compute (VMs are heavy, give them love)
- **Week 3:** Networking (this trips up most people — go slow)
- **Week 4:** Monitoring, Backup, and full revision + practice tests

### If you have **2 weeks**
- **Week 1:** Identity, Storage, Compute
- **Week 2:** Networking, Monitoring, full revision

### Each topic file follows this format

1. **The Big Picture** — What is this thing and why do you care?
2. **Real-World Analogy** — A relatable mental model
3. **Deep Dive** — The actual technical detail
4. **Hands-On / Examples** — Sample CLI / Portal steps
5. **Common Pitfalls** — Things people get wrong on the exam
6. **Quiz** — 8–10 questions to test understanding

---

## Prerequisites

You should already be comfortable with:
- Basic IT concepts: servers, networks, storage
- Command line basics (PowerShell or Bash)
- General cloud concepts (AZ-900 level knowledge helps but isn't required)

You don't need to be a developer or coder. AZ-104 is administrator-focused.

---

## Tools You'll Need

- An **Azure subscription** (free tier works for almost everything in this course)
- **Azure CLI** installed locally, OR use the [Azure Cloud Shell](https://shell.azure.com)
- **Azure PowerShell** module (optional but useful)
- A code editor — **VS Code** recommended, with the Azure extensions

---

## Tips for Exam Success

1. **Build it, don't just read about it.** Spin up a VM. Break it. Fix it. Delete it.
2. **Master the portal AND the CLI.** The exam tests both.
3. **Networking is where people lose marks.** Spend extra time on NSGs, route tables, and peering.
4. **Read the question twice.** Microsoft loves to phrase things in tricky ways like *"least privilege"* or *"minimize cost"*.
5. **Don't memorize — understand.** If you understand *why* something works, you can answer questions you've never seen before.
6. **All the best forthe exam!!!!

---

## A Quick Note on Pace

Plan for around **60–80 hours of study** if you're new to Azure. Less if you have prior cloud experience. Rushing this exam is the #1 reason candidates fail — it's broad, and broad means you can't bluff your way through.

---

## Let's go!

Start with [01-Identity-and-Governance](./01-Identity-and-Governance/README.md). 🚀

*Maintainer's note: All content reflects Microsoft Learn modules current as of 2025. Microsoft updates Azure constantly — when you hit something that looks different in the portal, trust the portal and update this content. That's the beauty of Markdown.*
