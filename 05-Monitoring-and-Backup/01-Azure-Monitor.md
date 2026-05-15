# Azure Monitor

## The Big Picture

**Azure Monitor** is the umbrella service that collects, analyzes, and alerts on telemetry from everything in Azure (and even outside Azure).

It answers three operational questions:
1. **What is happening right now?** — metrics & dashboards
2. **What happened before, and why?** — logs & queries
3. **Tell me when something goes wrong** — alerts & action groups

---

## Real-World Analogy: A Smart Building's Control Room

- **Metrics** = real-time gauges (temperature, occupancy) updating every second
- **Logs** = the security camera archive — searchable history of everything
- **Alerts** = the fire alarm — when a sensor crosses a threshold, the right people get paged
- **Workbooks & dashboards** = the wall of monitors in the control room
- **Application Insights** = the doctor monitoring vital signs of the building's apps

---

## The Two Pillars: Metrics and Logs

### Metrics
- **Numerical values** sampled at regular intervals (e.g., CPU %, request count)
- Stored in a **time-series database**, optimized for fast retrieval
- Retained for **93 days** by default
- Near-real-time (1-minute granularity standard)

### Logs
- **Structured records** (events, traces, performance data) stored in **Log Analytics workspaces**
- Queried with **Kusto Query Language (KQL)**
- Pay-as-you-go (per GB ingested + per GB retained beyond default 31 days)
- Retain up to **12 years** with archive tier

> **Rule of thumb:** Use **metrics** for dashboards/alerts on numerical data. Use **logs** for diagnostics, audits, and complex queries.

---

## Data Sources

Azure Monitor pulls from many places:

| Source | What it produces |
|--------|------------------|
| **Subscription / Resource activity logs** | Control-plane events (who did what) |
| **Resource diagnostic settings** | Data-plane logs/metrics from each resource |
| **VM agents** (Azure Monitor Agent / AMA) | Guest OS metrics, perf counters, syslog/event logs |
| **Application Insights SDK** | App-level traces, requests, dependencies |
| **Custom logs/metrics** | Anything you push via API |

### Diagnostic Settings (the most important AZ-104 control)
For each resource, configure where to send its logs/metrics:
- **Log Analytics workspace** (most common)
- **Storage account** (cheap archive)
- **Event Hub** (forward to a SIEM)

> Without diagnostic settings, the resource's logs go nowhere. Activity logs are still captured at the subscription level, but resource-level logs are not.

---

## Log Analytics Workspaces

A workspace is the **container** for logs. Choose:
- **Region** — workspace lives in a region; data residency
- **Pricing tier** — Pay-as-you-go (default), Commitment Tiers (100 GB/day+), Free (legacy)
- **Retention** — 30–730 days interactive; up to 7 years archive

### Workspace design
- One workspace per **environment per region** is a common pattern (centralized but not over-consolidated)
- **Cross-workspace queries** are supported — you can query multiple in one KQL
- Sentinel sits on top of a workspace if you need SIEM

---

## KQL — Kusto Query Language

KQL is read-only, pipeline-style, similar to SQL but with `|` chaining.

### Basics
```kql
// Last 1h of error events from machine 'web01'
Event
| where TimeGenerated > ago(1h)
| where Computer == "web01"
| where EventLevelName == "Error"
| project TimeGenerated, EventID, RenderedDescription
| order by TimeGenerated desc
```

### Common operators
| Operator | Purpose |
|----------|---------|
| `where` | Filter rows |
| `project` | Pick columns |
| `summarize` | Aggregate (count, avg, percentile) |
| `extend` | Add a calculated column |
| `join` | Combine two tables |
| `render` | Chart the result (in the portal) |
| `top` | First N rows by an order |
| `ago(1h)` | "1 hour ago" function |

### Sample: Top 10 noisy VMs by CPU last hour
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize avg(CounterValue) by Computer
| top 10 by avg_CounterValue desc
```

---

## Alerts

An **alert rule** runs on a schedule, evaluates a condition, and triggers an **action group** when the condition is met.

### Alert types
| Type | Source | Latency | Cost |
|------|--------|---------|------|
| **Metric alert** | Metrics | Near-real-time | Cheap (per metric per month) |
| **Log alert** | Log Analytics | 5–15 min cycle | Per-query cost |
| **Activity log alert** | Activity log | Real-time-ish | Free |
| **Smart detection** | App Insights ML | Variable | Free |

### Severities
0 (Critical) → 4 (Verbose). Use them consistently across teams.

### Action groups
A reusable list of who/what to notify:
- **Email**, **SMS**, **push** (Azure mobile app)
- **Voice call**
- **Webhook** (to PagerDuty, Slack, ServiceNow…)
- **Logic App** (workflow)
- **Function** (custom)
- **Automation Runbook** (auto-remediate, e.g., restart VM)
- **ITSM connector** (ServiceNow, BMC)

### Alert processing rules
Suppress, route, or modify alerts (e.g., suppress non-critical alerts during a maintenance window).

---

## Application Insights

A specialization of Azure Monitor for **applications** (web apps, APIs, mobile backends).

### Tracks
- **Requests** — HTTP requests with status code, duration
- **Dependencies** — outbound calls (SQL, HTTP, queues)
- **Exceptions** — unhandled errors with stack
- **Page views / user sessions** (web)
- **Custom events** — anything you log via SDK
- **Live Metrics** — real-time stream

### Workspace-based vs. Classic
**Workspace-based** App Insights (the modern default) writes into a Log Analytics workspace. Classic mode is being retired — migrate.

### Smart Detection
ML-driven anomaly detection. Detects sudden response time degradations or failure rate spikes without rule-writing.

---

## VM Insights and Container Insights

Pre-built monitoring solutions, light-weight to enable, deep on visualization:
- **VM Insights** — performance, processes, dependency map
- **Container Insights** — AKS cluster, node, and pod metrics + logs
- **App Service Insights** — App Service-tailored

They install required agents and create the standard queries/workbooks for you.

---

## Network Watcher

A regional network-troubleshooting toolkit. Free for most tools; some have small charges.

### Most-used tools (AZ-104 favorites)
| Tool | Purpose |
|------|---------|
| **IP flow verify** | Will this 5-tuple be allowed? Tests NSG rules without sending traffic |
| **Next hop** | Where would a packet go from VM X to destination Y? |
| **Connection troubleshoot** | End-to-end test (latency, hops, drops) |
| **Effective security rules** | Combined NSG rules applied to a NIC |
| **Effective routes** | Combined route table for a NIC |
| **Connection Monitor** | Continuous synthetic checks |
| **NSG flow logs / VNet flow logs** | Capture flow records to storage |
| **Packet capture** | Run a tcpdump on a VM |
| **Topology** | Visual map of resources in a VNet |

> **Exam tip:** "User can't reach the VM" → Network Watcher's **IP flow verify** or **Connection troubleshoot** is usually the answer.

---

## Hands-On: Diagnostic Setting + Metric Alert

```bash
# Send a Storage Account's logs/metrics to a Log Analytics workspace
az monitor diagnostic-settings create \
  --name diag-stg-prod \
  --resource $(az storage account show -g rg-storage -n stgprod001 --query id -o tsv) \
  --workspace $(az monitor log-analytics workspace show -g rg-monitor -n law-prod --query id -o tsv) \
  --logs   '[{"category":"StorageRead","enabled":true},{"category":"StorageWrite","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Metric alert: VM CPU > 80% for 5 minutes
az monitor metrics alert create \
  --name alert-vm-cpu \
  --resource-group rg-compute \
  --scopes $(az vm show -g rg-compute -n vm-web01 --query id -o tsv) \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action $(az monitor action-group show -g rg-monitor -n ag-oncall --query id -o tsv)
```

---

## Common Pitfalls

1. **No diagnostic setting → no logs.** People assume "Azure logs everything." It doesn't, unless you tell it to.
2. **Wrong workspace region.** Cross-region log queries cost more and are slower.
3. **Excessive log retention.** Logs cost per GB-month; review what's being ingested.
4. **Alerts without action groups.** Trigger fires, no one is notified.
5. **Metric alert thresholds set too sensitively.** Alert fatigue → real alerts ignored.

---

## Quiz: Azure Monitor

**1.** Which language is used to query Log Analytics?  
A. SQL  B. KQL (Kusto)  C. PromQL  D. JSON

**2.** True or False: Without configuring a diagnostic setting, a resource's resource-level logs are still automatically saved.

**3.** Which is the best store for logs you must keep for 5 years cheaply?  
A. Log Analytics interactive tier  B. Storage account / archive tier  C. Event Hub  D. Application Insights

**4.** Which alert type is **near-real-time**?  
A. Log alert  B. Metric alert  C. Activity log alert  D. Smart detection

**5.** Network Watcher tool to test "would this packet be allowed?":  
A. Topology  B. IP flow verify  C. Packet capture  D. Effective routes

**6.** Application Insights tracks all of the following EXCEPT:  
A. Requests  B. Dependencies  C. Database table sizes  D. Exceptions

**7.** The default metric retention in Azure Monitor is:  
A. 7 days  B. 30 days  C. 93 days  D. 365 days

**8.** Which is the best way to forward Azure logs to a third-party SIEM?  
A. Storage account  B. Event Hub  C. Log Analytics  D. Action group webhook

**9.** True or False: An Action Group can trigger an Azure Automation Runbook to auto-remediate.

**10.** Smart Detection in Application Insights uses:  
A. Static thresholds  B. KQL queries  C. Machine learning anomaly detection  D. User-defined functions

---

### Answers

1. **B** — KQL.  
2. **False** — You must configure a diagnostic setting; only activity logs are auto-captured at the subscription level.  
3. **B** — Storage account archive is cheapest for long retention.  
4. **B** — Metric alerts are near-real-time.  
5. **B** — IP flow verify.  
6. **C** — App Insights doesn't track DB table sizes (that's a DB metric).  
7. **C** — 93 days for metrics.  
8. **B** — Event Hub is the canonical SIEM forwarder.  
9. **True** — Runbook actions are supported.  
10. **C** — ML-driven anomaly detection.

---

**Next up:** [02-Backup-and-Recovery.md](./02-Backup-and-Recovery.md)
