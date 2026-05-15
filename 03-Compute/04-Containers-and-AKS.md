# Containers and Azure Kubernetes Service (AKS)

## The Big Picture

Containers package an application with all its dependencies into a single, immutable image that runs the same on any host. Azure offers three main ways to run containers:

| Service | What it is | Best for |
|---------|-----------|---------|
| **Azure Container Instances (ACI)** | One-off serverless containers | Quick jobs, simple background tasks |
| **Azure Container Apps (ACA)** | Serverless containers with scale-to-zero, KEDA | Microservices without K8s ops |
| **Azure Kubernetes Service (AKS)** | Managed Kubernetes | Complex orchestration at scale |

For AZ-104, focus on **ACI** and **AKS** basics. ACA is increasingly relevant.

---

## Real-World Analogy: Shipping Containers vs. Custom Crates

A traditional VM is a **custom-built crate** for one item. Heavy, takes time to build, takes time to load.

A **container** is a **standard shipping container**: same dimensions worldwide, every port can handle it, you can stack thousands on one ship. You spend less effort on logistics and more on what's *inside* the container.

- **Container Registry** = the dockyard storing the containers
- **ACI** = one container ship doing one trip
- **AKS** = a full container terminal with cranes, schedulers, tracking, and routing

---

## Azure Container Registry (ACR)

ACR is Azure's **private Docker registry** for storing your container images.

### SKUs
| SKU | Storage | Geo-replication | Use case |
|-----|---------|-----------------|---------|
| **Basic** | 10 GB | No | Dev/test |
| **Standard** | 100 GB | No | Most production |
| **Premium** | 500 GB | Yes (multi-region) | Global apps, content trust |

### Key features
- **Tasks** — build images in Azure on push (no local Docker required)
- **Geo-replication** (Premium) — pull images from the closest region
- **Content trust** (Premium) — sign and verify images
- **Vulnerability scanning** via Microsoft Defender for Cloud

### Authenticate with ACR
- **Admin user** — quick, single account, treat as anti-pattern in prod
- **Service principal** — common for CI/CD
- **Managed identity** — best for Azure-resident workloads
- **Microsoft Entra ID tokens** — `az acr login` uses these

---

## Azure Container Instances (ACI)

ACI is the **simplest** way to run a container. You give Azure an image, CPU, memory, and ports — Azure runs it.

- Per-second billing
- No orchestration (no auto-restart on host failure across hosts)
- Supports both Linux and Windows containers
- Can mount Azure Files for persistent state
- Supports VNet integration

### When to use ACI
- One-off batch processing
- Build agents
- Simple sidecar containers
- Quick demos / scheduled jobs

### When NOT to use ACI
- You need orchestration, rolling updates, complex networking — use AKS or ACA.

### Container Groups
A group of containers scheduled together on the same host (think Kubernetes pod). They share lifecycle, IP, and storage.

```bash
az container create \
  --resource-group rg-aci-demo \
  --name myaciapp \
  --image mcr.microsoft.com/azuredocs/aci-helloworld \
  --cpu 1 --memory 1.5 \
  --ports 80 \
  --dns-name-label myaci-himanshu \
  --location centralindia
```

---

## Azure Kubernetes Service (AKS)

AKS is Microsoft's **managed Kubernetes**. You get a Kubernetes control plane managed by Azure (free!) and you pay for the worker node VMs.

### Architecture

```
AKS Cluster
├── Control plane (managed by Microsoft, free)
│   ├── API server
│   ├── etcd
│   ├── scheduler
│   └── controller manager
└── Node pools (you pay for these VMs)
    ├── System node pool (runs system pods)
    └── User node pools (your apps)
        ├── Linux pool
        └── Windows pool (if needed)
```

### Node pools
- **System pool** — runs critical components (CoreDNS, metrics-server). At least one required.
- **User pools** — for your application workloads.
- Can mix VM sizes, can add/remove pools dynamically.
- Each pool can scale independently.

### Networking modes (CNI)

| Mode | IP source for pods | Use case |
|------|--------------------|----------|
| **kubenet** | Translated NAT | Smaller clusters, private IP shortage |
| **Azure CNI** | Real subnet IPs | Direct connectivity, more flexibility |
| **Azure CNI Overlay** | Overlay network | Many pods, VNet IP conservation (recommended) |
| **BYOCNI / Cilium** | Custom | Advanced networking |

### Identity and security
- **Microsoft Entra-integrated authentication** for cluster admins
- **Azure RBAC for Kubernetes Authorization** — assign Azure roles directly to Kubernetes namespaces
- **Workload Identity** — pods authenticate to Azure as managed identities (replaces deprecated Pod Identity)

### Scaling
- **Horizontal Pod Autoscaler (HPA)** — pod-level autoscale on CPU/memory
- **Cluster Autoscaler** — adds/removes nodes when pods can't be scheduled
- **KEDA** — event-driven autoscale (e.g., on queue depth)

### Upgrades
- Upgrade the **control plane** version first
- Then upgrade **node pools** (rolling)
- Plan upgrades in maintenance windows
- AKS supports the latest 3 minor Kubernetes versions

### Storage in AKS
- **Azure Disks** (ReadWriteOnce) — single-pod attached
- **Azure Files** (ReadWriteMany) — shared across pods
- **Azure NetApp Files** — high-perf
- **CSI drivers** are now the standard — old in-tree drivers are deprecated

---

## Deploying to AKS — High Level

```bash
# Create cluster
az aks create \
  --resource-group rg-aks-demo \
  --name aks-demo \
  --node-count 2 \
  --enable-managed-identity \
  --network-plugin azure \
  --enable-azure-rbac \
  --enable-aad \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group rg-aks-demo --name aks-demo

# Verify
kubectl get nodes

# Deploy something
kubectl create deployment hello --image=mcr.microsoft.com/azuredocs/aci-helloworld
kubectl expose deployment hello --port=80 --type=LoadBalancer
kubectl get service hello -w
```

---

## Common Pitfalls

1. **Confusing ACR Basic with Premium.** Basic doesn't replicate; using it across regions = slow pulls.
2. **Using admin user in ACR for production.** Always use managed identity or service principal.
3. **Forgetting that the AKS control plane runs in Azure subscription managed by MS, but worker nodes are in your subscription** — they show up as a resource group named `MC_<rg>_<cluster>_<region>`. Don't delete it!
4. **Choosing kubenet, then needing direct IP connectivity.** Switching CNI later is painful.
5. **Skipping a system pool design.** Mixing system + user workloads on the same nodes is risky.

---

## Quiz: Containers and AKS

**1.** Which ACR SKU supports geo-replication?  
A. Basic  B. Standard  C. Premium  D. All of them

**2.** True or False: The AKS control plane is billed separately as a VM.

**3.** What kind of Azure resource is created automatically by AKS to hold the worker nodes?  
A. A separate subscription  B. A managed resource group named `MC_<rg>_<cluster>_<region>`  C. A management group  D. A virtual machine scale set without a parent RG

**4.** Which networking mode in AKS gives pods real VNet IPs?  
A. kubenet  B. Azure CNI / Azure CNI Overlay  C. None  D. Service endpoint

**5.** Which is a key reason to use ACI over a VM?  
A. Easier orchestration  B. Per-second billing for short tasks  C. Free  D. Auto-pilot for AKS

**6.** Which scaling component **adds nodes** to an AKS cluster when pods are unschedulable?  
A. HPA  B. Cluster Autoscaler  C. KEDA  D. VMSS auto-scale

**7.** Workload Identity in AKS lets pods:  
A. Use a shared client secret  
B. Authenticate to Azure as managed identities  
C. Bypass RBAC  
D. Mount Azure Files

**8.** True or False: ACI supports both Linux and Windows containers.

**9.** Which feature is exclusive to ACR Premium?  
A. Image push  B. Tasks  C. Content trust + geo-replication  D. Authentication

**10.** A pod needs read/write access to a shared volume from multiple replicas at once. Which Azure storage option fits?  
A. Azure Disk (RWO)  B. Azure Files (RWX)  C. Local SSD  D. ACR

---

### Answers

1. **C** — Premium for geo-replication.  
2. **False** — AKS control plane is free; you pay only for nodes.  
3. **B** — `MC_<rg>_<cluster>_<region>`.  
4. **B** — Azure CNI assigns real subnet IPs.  
5. **B** — Per-second billing for short jobs.  
6. **B** — Cluster Autoscaler adds/removes nodes.  
7. **B** — Workload Identity = pod authenticates as a managed identity.  
8. **True** — ACI supports both.  
9. **C** — Geo-replication and content trust are Premium-only.  
10. **B** — Azure Files for ReadWriteMany.

---

**You finished Module 3!** 🎉

Next up: [Module 4 — Networking](../04-Networking/README.md). The biggest section by sweat-per-question — give it the time it deserves.
