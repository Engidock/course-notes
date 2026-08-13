# Azure Kubernetes Service (AKS) Mastery

> **👋 Hey Fresher — Read This First!**

> - **AKS = Azure's managed Kubernetes service** — Azure handles the control plane (API server, etcd, scheduler) for free; you manage the worker nodes
> - Without AKS: you install and maintain Kubernetes yourself on VMs — upgrading, patching, HA setup, load balancer wiring — weeks of work
> - With AKS: one command provisions a production-ready cluster in ~5 minutes — Azure handles control plane HA, node OS patching, and Kubernetes version upgrades
> - Every example uses **short CLI blocks** — one concept at a time — with a plain-English bullet explanation right beside it
> - **Company:** NovaPay — a Hyderabad fintech startup processing UPI/NEFT payments, running on Azure. You just joined as Junior DevOps Engineer. Lead: **Ananya**. Their current infrastructure is VMs — it's slow to scale, deployment takes 2 hours, and a node failure brings down the app. Your mission: migrate NovaPay to AKS

#### What You Will Learn and Build

- Provision an AKS cluster with node pools, networking, and RBAC using Azure CLI
- Connect kubectl to AKS and deploy NovaPay's payment API and dashboard apps
- Configure Application Gateway Ingress (AGIC), TLS, and Azure DNS
- Set up Cluster Autoscaler and Horizontal Pod Autoscaler for traffic spikes
- Integrate Azure Container Registry (ACR), Azure Key Vault, and Managed Identity
- Monitor with Azure Monitor, Container Insights, and Log Analytics
- Upgrade AKS cluster version and node pools with zero downtime
- Implement RBAC, Network Policies, and Pod Security Standards

AKS Cluster, Node Pools, ACR Integration, AGIC Ingress, Key Vault CSI, Autoscaling, Azure Monitor, RBAC, Upgrades, Network Policy

#### NovaPay AKS Architecture — Full Picture

```
NovaPay FinTech — AKS Production Architecture on Azure
══════════════════════════════════════════════════════════════════════════

  Azure Subscription: novapay-prod
  Resource Group: rg-novapay-prod  |  Region: Central India

  ┌─────────────────────────────────────────────────────────────────────┐
  │  Azure Virtual Network (vnet-novapay)  10.0.0.0/8                  │
  │                                                                     │
  │  ┌──────────────────────────────────────────────────────────────┐   │
  │  │  AKS Cluster: aks-novapay-prod                               │   │
  │  │                                                              │   │
  │  │  Control Plane (Azure Managed — FREE)                        │   │
  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │   │
  │  │  │API Server│  │Scheduler │  │  etcd    │  (HA, 3 replicas) │   │
  │  │  └──────────┘  └──────────┘  └──────────┘                   │   │
  │  │                                                              │   │
  │  │  System Node Pool (subnet: 10.1.0.0/16)                     │   │
  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │   │
  │  │  │Standard_ │  │Standard_ │  │Standard_ │ ← CoreDNS,        │   │
  │  │  │D4s_v3    │  │D4s_v3    │  │D4s_v3    │   metrics-server  │   │
  │  │  └──────────┘  └──────────┘  └──────────┘                   │   │
  │  │                                                              │   │
  │  │  App Node Pool (subnet: 10.2.0.0/16)  autoscale 3–10        │   │
  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │   │
  │  │  │Standard_ │  │Standard_ │  │Standard_ │ ← NovaPay Pods    │   │
  │  │  │D8s_v3    │  │D8s_v3    │  │D8s_v3    │   payment-api     │   │
  │  │  └──────────┘  └──────────┘  └──────────┘   dashboard       │   │
  │  └──────────────────────────────────────────────────────────────┘   │
  │                                                                     │
  │  ┌───────────────┐   ┌──────────────┐   ┌──────────────────────┐   │
  │  │  Azure ACR    │   │ Key Vault    │   │  Application Gateway │   │
  │  │ novapayacr    │   │ kv-novapay   │   │  (AGIC Ingress)      │   │
  │  │ (image store) │   │ (secrets)    │   │  pay.novapay.in      │   │
  │  └───────────────┘   └──────────────┘   └──────────────────────┘   │
  └─────────────────────────────────────────────────────────────────────┘
```

### 1. Phase 1 — AKS Prerequisites and Cluster Provisioning

**Business Problem:** NovaPay's VMs crash under payment spikes (month-end salary processing). Provision an AKS cluster that auto-scales and survives node failures without downtime.

**Scene 1 — NovaPay War Room | "Month-End Spike Took Us Down Again"**

> **Ananya** _Lead DevOps — NovaPay_
> 
> Third month in a row — salary day, payment volume triples, our VM-based payment API falls over. We restart manually, 22 minutes of downtime, clients are furious. AKS with autoscaler: when load spikes, new nodes spin up in under 2 minutes and new pods land on them. When load drops, nodes scale back down — we stop paying for idle VMs. We are moving to AKS this sprint. Vikram, pair with the new engineer on the provisioning today.

> **Vikram** _Senior DevOps — NovaPay_
> 
> And no more single points of failure. AKS spreads nodes across Availability Zones 1, 2, 3 in Central India. One zone goes down — the other two zones keep the app running. The control plane is Azure-managed and HA by default. We don't manage it. We don't patch it. We don't pay for it.

#### 1.1 Install Azure CLI and Login

```bash
# Install Azure CLI (Ubuntu/Debian)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login to Azure
az login

# Set working subscription
az account set --subscription "novapay-prod"
# Verify
az account show --query name
```

**📖 Azure CLI — AKS Gateway**

- **az login** — opens browser, authenticates your Azure account
- **az account set** — switches active subscription; critical if you have multiple subscriptions (dev vs prod)
- Azure CLI + kubectl are the two tools you will use for every AKS operation
- Register AKS provider once per subscription: `az provider register --namespace Microsoft.ContainerService`

#### 1.2 Create Resource Group and Virtual Network

```
# Resource group — logical container for all resources
az group create \
  --name rg-novapay-prod \
  --location centralindia

# Virtual network for AKS (Azure CNI needs pre-created VNet)
az network vnet create \
  --name vnet-novapay \
  --resource-group rg-novapay-prod \
  --address-prefix 10.0.0.0/8 \
  --subnet-name subnet-system \
  --subnet-prefix 10.1.0.0/16
```

**📖 Why Pre-create the VNet?**

- AKS supports two CNI modes — **kubenet** (auto VNet, limited) and **Azure CNI** (bring your own VNet, full Azure integration)
- NovaPay uses **Azure CNI** — pods get real Azure IPs, accessible from other Azure services without NAT
- Pre-created VNet lets you control IP ranges, integrate with ExpressRoute, and peer with other VNets
- `centralindia` = Azure region with Availability Zones 1, 2, 3 for HA

#### 1.3 Create the AKS Cluster

```
# Get subnet ID for Azure CNI
SUBNET_ID=$(az network vnet subnet show \
  --resource-group rg-novapay-prod \
  --vnet-name vnet-novapay \
  --name subnet-system \
  --query id -o tsv)

# Create AKS cluster with system node pool
az aks create \
  --resource-group rg-novapay-prod \
  --name aks-novapay-prod \
  --location centralindia \
  --kubernetes-version 1.29.2 \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --zones 1 2 3 \
  --network-plugin azure \
  --vnet-subnet-id $SUBNET_ID \
  --enable-managed-identity \
  --enable-addons monitoring \
  --workspace-resource-id /subscriptions/.../workspaces/law-novapay \
  --generate-ssh-keys
```

> **--kubernetes-version 1.29.2** — pin to a specific version; never use "latest" in production — unexpected upgrades break things
**--zones 1 2 3** — spreads nodes across three Availability Zones; one zone failure leaves 2/3 of nodes running
**--network-plugin azure** — Azure CNI; pods get VNet IPs, directly routable to Azure SQL, Service Bus, Key Vault
**--enable-managed-identity** — AKS uses a system-assigned Managed Identity instead of a service principal; no credential rotation needed
**--enable-addons monitoring** — installs Container Insights agent; sends logs and metrics to Log Analytics Workspace automatically
Cluster creation takes ~5–7 minutes

#### 1.4 Connect kubectl to the Cluster

```bash
# Merge AKS credentials into ~/.kube/config
az aks get-credentials \
  --resource-group rg-novapay-prod \
  --name aks-novapay-prod

# Verify connection
kubectl get nodes
kubectl get nodes -o wide
```

**📖 get-credentials**

- Downloads the kubeconfig and merges into `~/.kube/config` — after this, all `kubectl` commands target this cluster
- Run this on every developer machine and CI/CD agent that needs cluster access
- Add `--admin` flag for break-glass admin access bypassing RBAC (use sparingly)
- `kubectl get nodes -o wide` — shows node IPs, OS image, and which zone each node is in

```
NAME                                STATUS   ROLES   AGE   VERSION   ZONE
aks-nodepool1-12345678-vmss000000   Ready    agent   5m    v1.29.2   centralindia-1
aks-nodepool1-12345678-vmss000001   Ready    agent   5m    v1.29.2   centralindia-2
aks-nodepool1-12345678-vmss000002   Ready    agent   5m    v1.29.2   centralindia-3
```

### 2. Phase 2 — Node Pools

**Business Problem:** NovaPay runs two workload types — system components (CoreDNS, metrics-server) and app workloads (payment-api, dashboard). Mixing them on the same nodes means a noisy app can starve system components. Separate node pools fix this.

#### 2.1 Add a Dedicated App Node Pool

```
# Create a separate app node pool for NovaPay workloads
az aks nodepool add \
  --resource-group rg-novapay-prod \
  --cluster-name aks-novapay-prod \
  --name apppool \
  --node-count 3 \
  --node-vm-size Standard_D8s_v3 \
  --zones 1 2 3 \
  --mode User \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10 \
  --node-taints workload=app:NoSchedule \
  --labels env=prod pool=app
```

**📖 Node Pool Design**

- **--mode User** — app workloads go here; system pool runs only Kubernetes system pods
- **Standard_D8s_v3** — 8 vCPU, 32 GB RAM; larger than system pool (D4s_v3) because app pods need more memory
- **--enable-cluster-autoscaler** — AKS automatically adds nodes when pods are Pending (can't be scheduled) and removes nodes when they are underutilised
- **--node-taints workload=app:NoSchedule** — only pods with the matching toleration can land here; keeps system pods off this pool
- **--labels** — used by nodeSelector in pod specs to target this pool

#### 2.2 Taint + Toleration — Direct Pods to the Right Pool

```
# NovaPay deployment — tolerate the app pool taint
spec:
  tolerations:
    - key: "workload"
operator: "Equal"
value: "app"
effect: "NoSchedule"
nodeSelector:
    pool: "app"
```

**📖 Taint + Toleration Pattern**

- **Taint** on node — "don't schedule here unless you tolerate me"
- **Toleration** on pod — "I can handle this taint, schedule me here"
- **nodeSelector** — additionally pins the pod to nodes labelled `pool=app`
- Result: payment-api and dashboard pods land ONLY on the apppool nodes — system pods cannot compete for those resources

#### 2.3 Node Pool Operations

Command

What It Does

az aks nodepool list --cluster-name aks-novapay-prod -g rg-novapay-prod

List all node pools and their status

az aks nodepool scale --name apppool --node-count 5

Manually scale to 5 nodes (overrides autoscaler)

az aks nodepool upgrade --name apppool --kubernetes-version 1.29.3

Upgrade node pool OS/K8s version independently

az aks nodepool delete --name apppool

Remove a node pool (drains pods first)

az aks nodepool show --name apppool --query provisioningState

Check if node pool is Succeeded/Updating/Failed

### 3. Phase 3 — Azure Container Registry (ACR) Integration

**Business Problem:** NovaPay's Docker images need a private registry on Azure — not Docker Hub. ACR is Azure's managed private registry and integrates natively with AKS via Managed Identity (no credentials needed in CI/CD).

**Scene 3 — NovaPay Slack | "Docker Hub Rate Limit Hit Production"**

> **Ananya** _Lead DevOps — NovaPay_
> 
> Docker Hub rate-limited our pulls during the prod deploy. 6 hour deploy window blocked because image pulls started failing. Azure Container Registry has no pull rate limits, sits inside our Azure network — pulls are faster, private, and integrated with our AKS Managed Identity. No imagePullSecrets to manage. No credentials to rotate. AKS can pull from our ACR automatically because we grant the Managed Identity access at the role level.

#### 3.1 Create ACR and Attach to AKS

```
# Create a Premium ACR (geo-replication capable)
az acr create \
  --name novapayacr \
  --resource-group rg-novapay-prod \
  --sku Premium \
  --location centralindia

# Attach ACR to AKS — grants AcrPull role to AKS Managed Identity
az aks update \
  --name aks-novapay-prod \
  --resource-group rg-novapay-prod \
  --attach-acr novapayacr
```

**📖 attach-acr — Zero-Credential Integration**

- **--attach-acr** — grants `AcrPull` role to the AKS Managed Identity; AKS nodes can now pull images from this ACR without any imagePullSecrets
- **Premium SKU** — enables geo-replication, content trust, private endpoints, and 500 GB storage
- Image path: `novapayacr.azurecr.io/payment-api:v1.2.3`
- CI/CD pushes images here; AKS pulls them — zero credentials in cluster

#### 3.2 Build and Push Images to ACR

```bash
# Build directly in ACR (no local Docker needed)
az acr build \
  --registry novapayacr \
  --image payment-api:v1.2.3 \
  --file Dockerfile.payment .

# Or push a locally built image
az acr login --name novapayacr
docker tag payment-api:latest novapayacr.azurecr.io/payment-api:v1.2.3
docker push novapayacr.azurecr.io/payment-api:v1.2.3
```

**📖 az acr build**

- **az acr build** — uploads source context to Azure, builds the image in the cloud — no Docker daemon needed on CI agent
- Faster for large images because the build happens close to the registry in Azure infrastructure
- Supports multi-arch builds with `--platform linux/amd64,linux/arm64`
- List images: `az acr repository list --name novapayacr`

#### 3.3 Reference ACR Image in Deployment

```
# payment-api Deployment — no imagePullSecrets needed
spec:
  containers:
    - name: payment-api
image: novapayacr.azurecr.io/payment-api:v1.2.3
ports:
        - containerPort: 8080
```

**📖 No imagePullSecrets**

- Because AKS Managed Identity has `AcrPull` on novapayacr, the kubelet pulls the image automatically
- No `imagePullSecrets` field needed — this is the advantage of Managed Identity over Docker registry credentials
- If the ACR attachment is removed, pods will fail with `ImagePullBackOff` — always run `az aks check-acr` to verify the link

### 4. Phase 4 — Deploying NovaPay Applications

**Business Problem:** Deploy NovaPay's payment-api (3 replicas) and dashboard (2 replicas) to AKS with proper resource limits, health checks, and pod disruption budgets for zero-downtime deploys.

#### 4.1 Namespace Setup

```bash
# Create namespaces — isolate by concern
kubectl create namespace novapay-prod
kubectl create namespace novapay-monitoring

# Label namespace for Network Policy targeting
kubectl label namespace novapay-prod \
  env=prod team=payments
```

**📖 Namespace Isolation**

- All NovaPay app pods run in `novapay-prod` namespace — separate from system pods in `kube-system`
- RBAC roles, Network Policies, and Resource Quotas are all namespaced — easier to govern per team
- Labels on namespaces are used by Network Policies for cross-namespace traffic rules

#### 4.2 payment-api Deployment

```yaml
# payment-api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
namespace: novapay-prod
spec:
  replicas: 3
selector:
    matchLabels:
      app: payment-api
strategy:
    type: RollingUpdate
rollingUpdate:
      maxSurge: 1
maxUnavailable: 0
template:
    metadata:
      labels:
        app: payment-api
spec:
      tolerations:
        - key: workload
operator: Equal
value: app
effect: NoSchedule
containers:
        - name: payment-api
image: novapayacr.azurecr.io/payment-api:v1.2.3
ports:
            - containerPort: 8080
resources:
            requests: { cpu: "500m", memory: "512Mi" }
            limits:  { cpu: "2000m", memory: "2Gi" }
          readinessProbe:
            httpGet: { path: /health, port: 8080 }
            initialDelaySeconds: 10
periodSeconds: 10
livenessProbe:
            httpGet: { path: /health, port: 8080 }
            initialDelaySeconds: 30
failureThreshold: 3
```

> **maxSurge: 1, maxUnavailable: 0** — during rolling update, add 1 new pod before removing any old pod; at no point does the available pod count drop below 3 — zero-downtime deploy
**resources.requests** — what the scheduler uses to find a node with enough free capacity; always set this or the scheduler has no basis to place pods
**resources.limits** — hard ceiling; pod is OOM-killed if it exceeds memory limit; throttled if it exceeds CPU limit
**readinessProbe** — pod receives traffic only after /health returns 200; during startup, pod is Pending traffic even if the container is running
**livenessProbe** — if /health fails 3 times, Kubernetes restarts the container — catches hangs and deadlocks automatically

#### 4.3 PodDisruptionBudget — Protect During Upgrades

```yaml
# pdb-payment-api.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-payment-api
namespace: novapay-prod
spec:
  minAvailable: 2
selector:
    matchLabels:
      app: payment-api
```

**📖 PDB — Mandatory for Production**

- **minAvailable: 2** — during any voluntary disruption (node drain, upgrade), Kubernetes ensures at least 2 payment-api pods are always running
- Without a PDB, a node drain during AKS upgrade could evict all 3 pods simultaneously — full outage
- With PDB: drain waits, evicts pods one at a time, respects the minimum — zero downtime during upgrades
- Voluntary = node drain, upgrade. Involuntary = node failure (PDB does not apply)

### 5. Phase 5 — Ingress with Application Gateway (AGIC)

**Business Problem:** NovaPay needs HTTPS on pay.novapay.in and dashboard.novapay.in, path-based routing, WAF protection, and SSL termination — all managed by Azure Application Gateway.

#### 5.1 Enable AGIC Add-on

```
# Enable Application Gateway Ingress Controller add-on
az aks enable-addons \
  --name aks-novapay-prod \
  --resource-group rg-novapay-prod \
  --addons ingress-appgw \
  --appgw-name agw-novapay \
  --appgw-subnet-cidr 10.3.0.0/16
```

**📖 AGIC vs nginx Ingress**

- **AGIC** — Azure Application Gateway as Ingress; managed by Azure, includes WAF, auto-scaling, SSL offload, multi-site hosting
- **nginx Ingress** — community controller running as a pod; you manage it
- NovaPay uses AGIC because WAF (Web Application Firewall) is required for PCI DSS compliance for payment processing
- AGIC watches Kubernetes Ingress resources and automatically configures the Application Gateway — no manual AGW config

#### 5.2 Ingress Resource with TLS

```yaml
# novapay-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: novapay-ingress
namespace: novapay-prod
annotations:
    kubernetes.io/ingress.class: azure/application-gateway
appgw.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts: [pay.novapay.in]
      secretName: novapay-tls-secret
rules:
    - host: pay.novapay.in
http:
        paths:
          - path: /api
pathType: Prefix
backend:
              service:
                name: payment-api-svc
port: { number: 8080 }
          - path: /
pathType: Prefix
backend:
              service:
                name: dashboard-svc
port: { number: 3000 }
```

**📖 Path-Based Routing**

- **ssl-redirect: true** — AGW redirects HTTP → HTTPS automatically; no code change needed in the app
- **/api paths** → payment-api service; **/ paths** → dashboard service — one domain, two backends
- **secretName: novapay-tls-secret** — TLS certificate stored as Kubernetes Secret; AGW reads it for SSL termination
- Create cert secret: `kubectl create secret tls novapay-tls-secret --cert=cert.pem --key=key.pem -n novapay-prod`
- Or use cert-manager with Azure DNS for automatic Let's Encrypt certificate management

### 6. Phase 6 — Azure Key Vault Integration (CSI Driver)

**Business Problem:** NovaPay's payment-api needs database passwords, payment gateway API keys, and JWT secrets. Storing them in Kubernetes Secrets is base64 — not encrypted. Azure Key Vault + CSI Driver mounts secrets as files or env vars, encrypted at rest in Azure.

**Scene 6 — NovaPay Security Audit | "Secrets Were in ConfigMaps"**

> **Ananya** _Lead DevOps — NovaPay_
> 
> The PCI DSS auditor found our payment gateway API key in a ConfigMap in the prod namespace. ConfigMaps are plaintext. Anyone with cluster read access could see it. Key Vault CSI Driver: secrets live in Azure Key Vault, encrypted with Azure-managed keys, access controlled by Managed Identity. The pod mounts them as files at runtime — the values never appear in YAML, never in Git, never in etcd unencrypted.

#### 6.1 Enable Key Vault CSI Driver

```
# Enable the Secrets Store CSI Driver add-on
az aks enable-addons \
  --name aks-novapay-prod \
  --resource-group rg-novapay-prod \
  --addons azure-keyvault-secrets-provider

# Enable Managed Identity on node pools (for KV access)
az aks update \
  --name aks-novapay-prod \
  --resource-group rg-novapay-prod \
  --enable-managed-identity
```

**📖 CSI Driver Architecture**

- Secrets Store CSI Driver runs as a DaemonSet — one pod per node
- When a pod mounts a `SecretProviderClass` volume, the CSI driver fetches the secret from Key Vault via Managed Identity at pod startup
- No credentials in YAML — access is purely identity-based
- Secrets can be synced to Kubernetes Secrets for use as environment variables

#### 6.2 Create SecretProviderClass

```yaml
# secretproviderclass.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: novapay-keyvault
namespace: novapay-prod
spec:
  provider: azure
parameters:
    usePodIdentity: "false"
useVMManagedIdentity: "true"
keyvaultName: kv-novapay
tenantId: "<tenant-id>"
objects: |
      array:
        - |
          objectName: pg-password
          objectType: secret
        - |
          objectName: razorpay-key
          objectType: secret
        - |
          objectName: jwt-secret
          objectType: secret
```

**📖 SecretProviderClass**

- Declares which Key Vault secrets to fetch and how to authenticate
- **useVMManagedIdentity: true** — uses the node's VM Managed Identity; no client ID or secret needed
- **objectName** — must match the exact secret name in Key Vault (kv-novapay)
- Grant Key Vault access: `az keyvault set-policy --name kv-novapay --object-id <MI-object-id> --secret-permissions get list`

#### 6.3 Mount Secrets in Payment-API Pod

```
# Mount Key Vault secrets as files in the pod
spec:
  volumes:
    - name: kv-secrets
csi:
        driver: secrets-store.csi.k8s.io
readOnly: true
volumeAttributes:
          secretProviderClass: novapay-keyvault
containers:
    - name: payment-api
volumeMounts:
        - name: kv-secrets
mountPath: /mnt/secrets
readOnly: true
```

**📖 Secrets as Files**

- At pod start, CSI driver mounts `/mnt/secrets/pg-password`, `/mnt/secrets/razorpay-key`, etc. as files
- App reads secrets from files — never from environment variables (env vars can appear in crash logs)
- Secrets are refreshed from Key Vault periodically — rotation is automatic without pod restart
- If Key Vault is unreachable at pod start, the pod fails to start — a security feature, not a bug

### 7. Phase 7 — Autoscaling (HPA + Cluster Autoscaler)

**Business Problem:** On month-end salary day, NovaPay payment volume hits 10x normal. HPA scales pods. When pods exceed node capacity, Cluster Autoscaler adds nodes. When load drops, both scale back down — costs drop with load.

#### 7.1 Horizontal Pod Autoscaler

```yaml
# hpa-payment-api.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-payment-api
namespace: novapay-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
kind: Deployment
name: payment-api
minReplicas: 3
maxReplicas: 20
metrics:
    - type: Resource
resource:
        name: cpu
target:
          type: Utilization
averageUtilization: 60
```

**📖 HPA — Pod-Level Scaling**

- **minReplicas: 3** — always at least 3 pods for HA (one per AZ)
- **maxReplicas: 20** — upper bound preventing runaway scaling
- **averageUtilization: 60** — when average CPU across pods exceeds 60% of request (500m), HPA adds pods
- HPA reads metrics from metrics-server every 15 seconds; scales up fast, scales down slowly (5 min cooldown) to avoid flapping
- Must set `resources.requests.cpu` in the Deployment — HPA can't work without it

#### 7.2 Cluster Autoscaler (Already Enabled on Node Pool)

```bash
# Verify autoscaler is working
kubectl get events -n novapay-prod \
  --field-selector reason=TriggeredScaleUp

# Check autoscaler status
kubectl get configmap cluster-autoscaler-status \
  -n kube-system -o yaml
```

**📖 Cluster Autoscaler — Node-Level Scaling**

- When HPA creates new pods and no node has enough CPU/memory to schedule them, pods stay **Pending**
- Cluster Autoscaler detects Pending pods → requests Azure to add a new node to the VMSS → node joins → pods schedule on it
- Node provisioning takes ~2 minutes on AKS (pre-pulling images helps)
- Scale-down: if a node is underutilised for 10 minutes, autoscaler drains it and removes it — cost savings
- Scale-down is blocked if a pod has no PDB or cannot be safely evicted — always set PDBs

#### 7.3 Vertical Pod Autoscaler (VPA) — Recommender Mode

```yaml
# vpa-payment-api.yaml — recommender only (safe for prod)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-payment-api
spec:
  targetRef:
    apiVersion: apps/v1
kind: Deployment
name: payment-api
updatePolicy:
    updateMode: "Off"
```

**📖 VPA — Right-size Resources**

- **updateMode: Off** — VPA only recommends; does NOT change running pods. Safe for production
- Check recommendations: `kubectl describe vpa vpa-payment-api`
- If payment-api is using 200m CPU but requests 500m, VPA will recommend lowering requests — saves node capacity
- **Never use updateMode: Auto with HPA on the same Deployment** — they conflict and cause pod churn

### 8. Phase 8 — Azure Monitor and Container Insights

**Business Problem:** NovaPay needs visibility into cluster health, pod failures, memory pressure, and payment API latency — all in one place, with alerts.

#### 8.1 Container Insights — Already Enabled at Cluster Creation

```bash
# Verify Container Insights is running
kubectl get pods -n kube-system \
  -l component=oms-agent

# View live logs from payment-api
kubectl logs -n novapay-prod \
  -l app=payment-api \
  --since=1h --tail=100
```

**📖 What Container Insights Collects**

- Node CPU, memory, disk, and network metrics every 60 seconds
- Pod CPU and memory usage per container
- Container stdout/stderr logs shipped to Log Analytics Workspace
- Kubernetes events (OOMKilled, Failed scheduling, CrashLoopBackOff)
- All queryable in Azure Monitor with KQL (Kusto Query Language)

#### 8.2 Log Analytics KQL Queries

```
-- Find OOMKilled containers in last 24h
KubeEvents
| where Reason == "OOMKilling"
| where TimeGenerated > ago(24h)
| project TimeGenerated, Name, Message

-- Payment API CPU usage over time
KubePodInventory
| where Namespace == "novapay-prod"
| where Name has "payment-api"
| join Perf on $left.Name == $right.ObjectName
| where CounterName == "cpuUsageNanoCores"
| summarize avg(CounterValue) by bin(TimeGenerated, 5m)
```

**📖 KQL in Azure Monitor**

- All AKS metrics and logs go to **Log Analytics Workspace** — queryable with KQL
- KQL tables: `KubeEvents`, `KubePodInventory`, `ContainerLog`, `Perf`, `KubeNodeInventory`
- Save queries as Azure Monitor Alerts — get notified when OOMKills exceed threshold
- Create dashboards in Azure Portal from saved queries — share with the team

#### 8.3 Create an Alert — OOMKill Alert for payment-api

```
# Create alert rule via Azure CLI
az monitor scheduled-query create \
  --name "PaymentAPI-OOMKill" \
  --resource-group rg-novapay-prod \
  --scopes /subscriptions/.../workspaces/law-novapay \
  --condition-query \
    "KubeEvents | where Reason=='OOMKilling' and Namespace=='novapay-prod'" \
  --condition-threshold 1 \
  --condition-time-aggregation Count \
  --evaluation-frequency 5m \
  --window-size 10m \
  --severity 1 \
  --action-groups /subscriptions/.../actionGroups/novapay-oncall
```

**📖 Alert Setup**

- **--severity 1** — Critical severity; triggers PagerDuty/Teams/email via Action Group
- **--evaluation-frequency 5m** — query runs every 5 minutes
- Action Groups send alerts via email, SMS, webhook, Azure Functions, or Logic Apps
- Common NovaPay alerts: OOMKill, pod CrashLoopBackOff > 5, node NotReady, payment-api 5xx rate > 1%

### 9. Phase 9 — RBAC and Azure AD Integration

**Business Problem:** NovaPay has 3 teams — DevOps (full access), Developers (deploy but not delete), and Finance (read-only to view pod status). Kubernetes RBAC + Azure AD groups controls this without sharing kubeconfig files.

#### 9.1 Enable Azure AD Integration

```
# Enable Azure AD integration on existing cluster
az aks update \
  --name aks-novapay-prod \
  --resource-group rg-novapay-prod \
  --enable-aad \
--aad-admin-group-object-ids <devops-group-id>
# Devs get credentials tied to their AAD identity
az aks get-credentials \
  --name aks-novapay-prod \
  --resource-group rg-novapay-prod
```

**📖 Azure AD RBAC for AKS**

- After enabling, `kubectl` commands prompt for Azure AD login — the user's AD identity is used for RBAC decisions
- **--aad-admin-group-object-ids** — members of this AD group get cluster-admin automatically
- Developers authenticate with their own Azure AD credentials — no shared service account tokens
- Audit trail: every kubectl action is tied to a named user in Azure AD logs

#### 9.2 Namespace-Level RBAC for Developers

```yaml
# developer-role.yaml — deploy but not delete
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: novapay-developer
namespace: novapay-prod
rules:
  - apiGroups: ["", "apps"]
    resources: [pods, deployments, services]
    verbs: [get, list, watch, create, update, patch]
  - apiGroups: [""]
    resources: [pods/log]
    verbs: [get]
```

**📖 Role vs ClusterRole**

- **Role** — permissions scoped to one namespace (novapay-prod only)
- **ClusterRole** — permissions across all namespaces or for cluster-wide resources (nodes, PVs)
- Developers can get/list/patch pods and deployments — they can deploy — but cannot `delete`
- Bind to Azure AD group via RoleBinding + `kind: Group` and the AD group's Object ID

#### 9.3 Bind Role to Azure AD Group

```yaml
# rolebinding-developers.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-developer-role
namespace: novapay-prod
subjects:
  - kind: Group
name: <AAD-dev-group-object-id>
apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
name: novapay-developer
apiGroup: rbac.authorization.k8s.io
```

**📖 RoleBinding to AD Group**

- **kind: Group** + AD group Object ID — every member of that AD group inherits the Role
- Add a new developer to the AD group in Azure Portal → they get Kubernetes access instantly — no cluster change needed
- Remove from AD group → access revoked instantly
- Finance read-only: create a Role with only `get`, `list`, `watch` verbs and bind to Finance AD group

### 10. Phase 10 — Network Policy

**Business Problem:** In NovaPay's cluster, any pod can talk to any other pod by default — including the payment-api talking to the dashboard database. Network Policies restrict this to only the required paths.

#### 10.1 Default-Deny All Ingress in Namespace

```yaml
# default-deny.yaml — block all traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
namespace: novapay-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**📖 Default-Deny First**

- **podSelector: {}** — matches ALL pods in novapay-prod namespace
- **no ingress/egress rules** — denies all traffic in and out by default
- After this, you add specific allow rules — only permitted paths work
- Network Policy requires a CNI that supports it — Azure CNI with Azure Network Policy or Calico on AKS
- Enable: `az aks create --network-policy azure` or `--network-policy calico`

#### 10.2 Allow payment-api to Access PostgreSQL Only

```yaml
# allow-payment-to-db.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-payment-to-pg
namespace: novapay-prod
spec:
  podSelector:
    matchLabels:
      app: postgresql
ingress:
    - from:
        - podSelector:
            matchLabels:
              app: payment-api
ports:
        - protocol: TCP
port: 5432
```

**📖 Allowlist Pattern**

- PostgreSQL pod only accepts connections from pods labelled `app: payment-api` on port 5432
- Dashboard pods, monitoring pods, and other pods cannot reach the database — even from the same namespace
- This limits blast radius: if dashboard is compromised, it cannot pivot to the payment database
- Always add a policy to allow DNS: port 53 egress to kube-system DNS pods, or CoreDNS stops working

### 11. Phase 11 — AKS Upgrades (Zero Downtime)

**Business Problem:** Azure regularly releases new Kubernetes versions with security patches. NovaPay must upgrade without taking the payment-api offline during business hours.

**Scene 11 — NovaPay Change Meeting | "Upgrade Without Downtime"**

> **Vikram** _Senior DevOps — NovaPay_
> 
> The 1.28 control plane is end-of-support in 3 months. We upgrade the control plane first — zero impact, it's Azure-managed and they do it live. Then we upgrade node pools one node at a time — cordon, drain, upgrade, re-join. Our PDB ensures payment-api always has at least 2 pods running. The upgrade process respects PDBs — if draining a node would violate the PDB, it waits. We test the full upgrade in staging first — same sequence, same commands.

#### 11.1 Check Available Upgrade Versions

```
# See what Kubernetes versions are available
az aks get-upgrades \
  --name aks-novapay-prod \
  --resource-group rg-novapay-prod \
  -o table
```

**📖 AKS Upgrade Order**

- **Step 1:** Upgrade control plane — Azure handles it, takes ~10 minutes, no downtime
- **Step 2:** Upgrade system node pool
- **Step 3:** Upgrade app node pools — pods are drained and rescheduled
- Can only upgrade one minor version at a time — 1.28 → 1.29 → 1.30 (not 1.28 → 1.30)
- Upgrade in staging first: same commands, catch issues before prod

#### 11.2 Upgrade Cluster and Node Pools

```
# Step 1 — upgrade control plane only
az aks upgrade \
  --name aks-novapay-prod \
  --resource-group rg-novapay-prod \
  --kubernetes-version 1.29.3 \
  --control-plane-only
# Step 2 — upgrade app node pool
az aks nodepool upgrade \
  --cluster-name aks-novapay-prod \
  --resource-group rg-novapay-prod \
  --name apppool \
  --kubernetes-version 1.29.3 \
  --max-surge 1
```

**📖 --max-surge for Speed**

- **--max-surge 1** — Azure adds 1 extra node (surge node), drains one old node onto it, removes the old node. Never reduces capacity below the configured count
- Without max-surge: upgrade drains existing nodes in place — slower and riskier
- PDB is respected: if draining a node would violate `minAvailable: 2`, the drain waits
- **--control-plane-only** — upgrades only the API server/etcd/scheduler; node pools stay on old version until you explicitly upgrade them

### 12. Phase 12 — Storage with Azure Disks and Files

**Business Problem:** NovaPay's PostgreSQL needs persistent storage that survives pod restarts and node replacement. Azure Disk PVs provide block storage tied to a pod; Azure Files provides shared storage accessible by multiple pods.

#### 12.1 Azure Disk — Dynamic PVC for PostgreSQL

```yaml
# pvc-postgres.yaml — dynamic provisioning
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
namespace: novapay-prod
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: managed-premium
resources:
    requests:
      storage: 100Gi
```

**📖 Azure Disk vs Files**

- **managed-premium** — SSD-backed Azure Managed Disk; 100 GB SSD for PostgreSQL data files
- **ReadWriteOnce** — one pod at a time; disks cannot be shared between nodes (block storage limitation)
- For shared storage (reports, uploads): use `storageClassName: azurefile-premium` with `ReadWriteMany`
- AKS creates the Azure Disk automatically via the built-in storage class — no manual disk provisioning
- Disk persists even if the pod or PVC is deleted — manual cleanup required

### 13. Phase 13 — Resource Quotas and LimitRanges

**Business Problem:** Multiple teams deploy to the same cluster. Without limits, a badly configured deployment could consume all node CPU/memory and starve other services.

#### 13.1 ResourceQuota — Cap Namespace Usage

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: novapay-quota
namespace: novapay-prod
spec:
  hard:
    requests.cpu: "20"
requests.memory: 40Gi
limits.cpu: "40"
limits.memory: 80Gi
pods: "50"
```

**📖 ResourceQuota Enforcement**

- Total CPU requests across all pods in novapay-prod cannot exceed 20 vCPU
- If a new Deployment would exceed the quota, Kubernetes rejects the pod creation
- **pods: 50** — maximum pod count; prevents runaway HPA
- Check current usage: `kubectl describe quota novapay-quota -n novapay-prod`
- Pair with LimitRange to set default requests/limits so pods without them don't get rejected by quota

### 14. Phase 14 — AKS CI/CD Integration with GitHub Actions

**Business Problem:** NovaPay developers push code, the pipeline must build, push to ACR, and deploy to AKS automatically — with staging promotion and production manual gate.

#### 14.1 GitHub Actions Pipeline

```bash
# .github/workflows/deploy.yml
name: NovaPay Deploy
on:
  push:
    branches: [main]

env:
  ACR: novapayacr.azurecr.io
IMAGE: payment-api
TAG: ${{ github.sha }}
jobs:
  build-push:
    runs-on: ubuntu-latest
steps:
      - uses: actions/checkout@v4
      - name: Login to ACR
uses: azure/docker-login@v1
with:
          login-server: ${{ env.ACR }}
username: ${{ secrets.ACR_USERNAME }}
password: ${{ secrets.ACR_PASSWORD }}
      - name: Build and Push
run: |
          docker build -t $ACR/$IMAGE:$TAG .
          docker push $ACR/$IMAGE:$TAG
      - name: Deploy to AKS Staging
uses: azure/aks-set-context@v3
with:
          cluster-name: aks-novapay-staging
resource-group: rg-novapay-staging
      - run: |
          kubectl set image deployment/payment-api \
            payment-api=$ACR/$IMAGE:$TAG \
            -n novapay-staging
```

> **github.sha** — commit SHA as image tag; every deploy is traceable to its exact commit in Git
**azure/aks-set-context** — official GitHub Action that runs `az aks get-credentials` and configures kubectl
**kubectl set image** — updates the image tag on the Deployment; triggers a rolling update without editing YAML files
Add a second job with `environment: production` and a manual approval gate for prod deploy
Use Workload Identity Federation instead of ACR_USERNAME/PASSWORD — no credentials to store in GitHub Secrets

### 15. Phase 15 — AKS Production Best Practices Summary

**Scene 15 — NovaPay Quarterly Review | "What AKS Changed"**

> **Ananya** _Lead DevOps — NovaPay_
> 
> Month-end salary day — 8x normal payment volume. Cluster autoscaler spun up 4 new nodes in 90 seconds. HPA scaled payment-api from 3 to 14 pods. Zero downtime. Zero manual intervention. Last quarter on VMs: 22 minutes of downtime, 3 engineers on a call, manually restarting processes. This quarter: I was asleep. The alert fired, auto-scaling handled it, and I woke up to a green dashboard. That is what AKS gives you — the infrastructure handles scale; you focus on the application.

> **AKS Production Checklist — NovaPay Engineering Standards**

> - Always use **Availability Zones** (--zones 1 2 3) for node pools — single zone failure must not reduce availability below 2/3 capacity
> - Always create a **PodDisruptionBudget** for every production Deployment — without it, node drains during upgrades can cause full outages
> - Always set **resources.requests and limits** on every container — scheduler cannot make correct placement decisions without requests; unlimited containers will OOM-kill neighbours
> - Always use **readinessProbe** — pods without probes receive traffic the instant they start, before the app is actually ready, causing client errors
> - Never use the **system node pool** for app workloads — use taints to enforce separation; a misbehaving app must not starve CoreDNS
> - Always use **Azure CNI** over kubenet for production — pod IPs are real VNet IPs, compatible with all Azure networking features (Private Endpoints, Service Endpoints, NSGs)
> - Never store **secrets in ConfigMaps or plaintext values files** — use Key Vault CSI Driver; secrets encrypted at rest in Azure
> - Always test upgrades in a **staging cluster first** — same Kubernetes version, same node pool size, same app workloads

##### AKS Cost Optimisation — NovaPay FinOps Rules

- Enable **Cluster Autoscaler with aggressive scale-down** (--scale-down-delay-after-add 5m) — idle nodes cost money whether pods are on them or not
- Use **Spot Node Pools** for batch/non-critical workloads — 70–80% discount over on-demand; perfect for report generation, ML inference, background jobs
- Use **Start/Stop cluster feature** for dev/test clusters — `az aks stop` deallocates nodes (stops billing); `az aks start` brings it back; control plane stays, etcd state is preserved
- Check **Azure Advisor recommendations** for AKS — surfaces underutilised nodes, unattached PVCs, and oversized VMs automatically
- Use **VPA in recommender mode** for 2 weeks before adjusting resource requests — right-sized requests lead to better node utilisation and fewer wasted cores
- Enable **node image auto-upgrade** on non-production clusters — reduces patching overhead; in prod, upgrade nodes on a controlled maintenance schedule

##### ⚠️ Common AKS Mistakes — NovaPay Incident Log

- **No PDB set** — node drain during upgrade evicted all replicas simultaneously → 4-minute payment outage (fixed: minAvailable: 2)
- **No resource limits** — a memory leak in dashboard OOM-killed neighbouring payment-api pods on the same node → cascading failure (fixed: set limits on all containers)
- **Upgrading node pools before control plane** — kubelet version ahead of API server version is unsupported → upgrade refused, cluster locked (fix: always upgrade control plane first)
- **Using :latest image tag** — two nodes pulled different versions of the same "latest" image → inconsistent behaviour impossible to debug (fix: always pin to commit SHA tag)
- **No readinessProbe** — pods got traffic during 30-second startup → 500 errors from clients during every deploy (fix: readinessProbe with initialDelaySeconds: 15)

### Quick Reference — All AKS Commands at NovaPay

Command

What It Does

az aks create --name ... --node-count 3 --zones 1 2 3

Provision AKS cluster with AZ-spread nodes

az aks get-credentials --name ... --resource-group ...

Configure kubectl to connect to the cluster

az aks nodepool add --name apppool --enable-cluster-autoscaler

Add a new node pool with autoscaler

az aks nodepool scale --name apppool --node-count 5

Manually scale a node pool

az aks update --attach-acr novapayacr

Grant AKS Managed Identity AcrPull on ACR

az aks enable-addons --addons ingress-appgw

Enable Application Gateway Ingress Controller

az aks enable-addons --addons azure-keyvault-secrets-provider

Enable Key Vault CSI secrets driver

az aks enable-addons --addons monitoring

Enable Container Insights → Log Analytics

az aks get-upgrades --name ... -o table

List available Kubernetes upgrade versions

az aks upgrade --kubernetes-version 1.29.3 --control-plane-only

Upgrade only the managed control plane

az aks nodepool upgrade --name apppool --kubernetes-version 1.29.3

Upgrade node pool Kubernetes version

az aks stop --name ... --resource-group ...

Stop cluster (deallocate nodes, save cost)

az aks start --name ... --resource-group ...

Start a stopped cluster

az aks check-acr --acr novapayacr --name aks-novapay-prod

Verify AKS can pull from ACR

az aks show --name ... --query kubernetesVersion

Show current cluster Kubernetes version

kubectl top nodes

Show CPU and memory usage per node (requires metrics-server)

kubectl top pods -n novapay-prod

Show CPU and memory usage per pod

kubectl drain NODE --ignore-daemonsets --delete-emptydir-data

Evict all pods from a node before maintenance

kubectl cordon NODE

Mark node unschedulable (no new pods)

kubectl uncordon NODE

Mark node schedulable again

kubectl rollout status deployment/payment-api -n novapay-prod

Monitor rolling update progress

kubectl rollout undo deployment/payment-api -n novapay-prod

Roll back to previous Deployment revision

##### 🏋️ Hands-On Exercises — Extend NovaPay's AKS Setup

1. **Add a Spot Node Pool for batch reports:** Create a node pool named `batchpool` using `--priority Spot --eviction-policy Delete --spot-max-price -1` and `Standard_D4s_v3` SKU. Add a taint `kubernetes.azure.com/scalesetpriority=spot:NoSchedule` and deploy NovaPay's monthly report generation Job to this pool using the matching toleration. Spot pools cost 70% less — ideal for fault-tolerant batch work.
2. **Configure Workload Identity for Key Vault:** Enable OIDC issuer (`az aks update --enable-oidc-issuer --enable-workload-identity`), create a user-assigned Managed Identity, federate it with the cluster's OIDC issuer, assign Key Vault Secrets Officer role to it, and annotate the service account used by payment-api with the identity's client ID. This is more secure than VM-level Managed Identity — only the payment-api pod, not all pods on the node, can access Key Vault.
3. **Set up Keda for event-driven scaling:** Install KEDA on AKS (`helm install keda kedacore/keda -n keda`), create a ScaledObject that targets the payment-api Deployment and scales based on Azure Service Bus queue depth (payments pending processing). When the queue has 0 messages: 0 pods. When queue > 100 messages: scale up to 10 pods. This saves cost outside business hours when no payments are being processed.
4. **Implement GitOps with Flux:** Enable the Flux add-on on the cluster (`az aks enable-addons --addons flux`), bootstrap it to NovaPay's GitLab repo, and configure a Kustomization that watches the `k8s/prod/` directory. Any YAML committed to that directory is automatically applied to the cluster — no kubectl in the CI/CD pipeline, no cluster credentials in GitHub Secrets, full audit trail in Git.
5. **Configure automated node image upgrades with maintenance windows:** Create a maintenance configuration (`az aks maintenanceconfiguration add`) that allows node OS patching only on Sunday 02:00–04:00 IST. Enable auto-upgrade channel (`az aks update --auto-upgrade-channel patch`) so AKS automatically applies patch-level Kubernetes updates (1.29.2 → 1.29.3) within the maintenance window — no manual upgrade command needed for patch versions.

**Quiz: ❓ Interview Question: NovaPay has 3 replicas of payment-api. During an AKS node pool upgrade, all 3 pods were evicted simultaneously causing a 5-minute outage. What was missing and how do you fix it?**

- A) The HPA was not configured with the right min replicas
- B) A PodDisruptionBudget was missing — nothing prevented all 3 pods from being evicted at once
- C) The Deployment should have used Recreate strategy instead of RollingUpdate
- D) The node pool should have been in a single Availability Zone

> **Answer/explanation:** ✅ **Answer: B.** Without a PodDisruptionBudget, the AKS upgrade process can drain all nodes simultaneously, evicting all replicas at once. Fix: `kubectl apply -f pdb-payment-api.yaml` with `minAvailable: 2`. Now the upgrade respects the PDB — it can only evict pods one at a time, waiting until a new pod is Running before evicting the next one. The upgrade takes slightly longer but payment-api stays available throughout. This is not just theory — it was NovaPay's actual incident in Scene 11.

##### Common Fresher Questions — AKS at NovaPay

**Q: Q: What is the difference between the System node pool and a User node pool?**

A: **System pool** — reserved for Kubernetes system pods (CoreDNS, metrics-server, kube-proxy); every AKS cluster must have exactly one system pool; cannot be deleted
**User pool** — for application workloads; optional, can have multiple, can be deleted; taint system pool with `CriticalAddonsOnly=true:NoSchedule` to prevent app pods landing there
Separating them ensures CoreDNS is never starved of resources by misbehaving app pods — DNS failure would break every pod in the cluster

**Q: Q: What happens when a Cluster Autoscaler tries to scale down but PDB blocks it?**

A: Autoscaler marks the node for scale-down, attempts to evict pods one by one
If evicting a pod would violate the PDB (go below minAvailable), the eviction is refused
Autoscaler backs off and retries after a configurable delay (default 10 minutes)
This is correct and expected behaviour — the PDB protects the workload even from autoscaler-driven changes
If a node is permanently blocked from scale-down by a PDB, increase minReplicas on the Deployment so pods spread across more nodes

**Q: Q: Azure CNI or kubenet — which to choose for production?**

A: **Azure CNI** — pods get real VNet IP addresses; works with Private Endpoints, Network Security Groups, Azure Firewall, and Service Endpoints; required for AGIC; required for Windows node pools
**kubenet** — pods get an overlay network IP; not routable from outside the cluster without NAT; simpler setup, fewer IP addresses consumed from the VNet
**NovaPay uses Azure CNI** — payment-api pods need to talk directly to Azure SQL Managed Instance via Private Endpoint, which requires real VNet IPs
Rule of thumb: use Azure CNI in enterprise/production; kubenet only for dev/test or IP-constrained environments

### AKS Mastery Complete 🎉

You have built NovaPay's complete production AKS platform — cluster provisioning with Availability Zones, system and app node pools with Cluster Autoscaler, ACR integration, Application Gateway Ingress with TLS, Key Vault CSI secrets, HPA + VPA autoscaling, Azure Monitor with KQL alerts, Azure AD RBAC, Network Policies, PodDisruptionBudgets, zero-downtime upgrades, and CI/CD with GitHub Actions.

> **Ananya**
> 
> "Salary day — 8x load. Zero downtime. Zero calls. Zero manual steps. The cluster scaled itself, the app scaled itself, Key Vault served secrets securely, AGIC terminated TLS, Azure AD controlled who could see what. Everything we built this sprint — node pools, PDBs, HPA, autoscaler — all of it worked in exactly the scenario that used to take us down every single month. That is what production-grade AKS looks like."

> **Vikram**
> 
> "And the PCI DSS auditor — secrets in Key Vault, Network Policies denying cross-service lateral movement, Azure AD RBAC with full audit logs, Pod Security Standards enforced. We passed the audit. Last quarter we failed because of a password in a ConfigMap. This quarter: encrypted secrets, identity-based access, least privilege across the board."

> **Next: Advanced AKS — GitOps with Flux, KEDA, Workload Identity, Private Clusters & Azure Service Mesh**

> - **Private AKS cluster** — API server has no public IP; all kubectl traffic goes through Azure Private Link; mandatory for banking/fintech compliance
> - **Flux GitOps** — cluster state defined entirely in Git; no kubectl in pipelines; Flux syncs cluster to Git continuously — drift detection included
> - **KEDA** — Kubernetes Event-Driven Autoscaling; scale pods to zero based on Azure Service Bus, Event Hub, or custom metrics — pay nothing when idle
> - **Workload Identity** — pod-level Managed Identity federation with OIDC; only specific pods can access specific Key Vault secrets — not all pods on a node
> - **Azure Service Mesh (Istio)** — mTLS between all NovaPay services, traffic shifting for canary deployments, distributed tracing to Azure App Insights
> - **Defender for Containers** — runtime threat detection inside pods, vulnerability scanning of ACR images, Kubernetes audit log analysis for suspicious API calls
