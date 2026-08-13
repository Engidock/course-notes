# Azure Project Mastery

> **👋 Hey Fresher — Read This First!**

> - Microsoft Azure is the **world's second-largest cloud platform** — used by most Indian enterprises, banks, government projects, and MNCs including TCS, Infosys, Wipro, and HCL. If you work in IT services, you will encounter Azure.
> - Azure and AWS solve the same problems but with different naming. Once you understand one, the other makes sense quickly.
> - Every code block is **short and focused**. Every explanation is **bullet points**. Console previews show you exactly what the Azure Portal looks like after each action.
> - **Company in this project:** UrbanNest — a real-estate platform startup in Pune listing properties, connecting buyers with agents, and processing document uploads. Their monolith app on a single on-premises server keeps crashing under load. You just joined as a Junior Cloud Engineer. Your lead is Priya. You will migrate UrbanNest to Azure from scratch.

#### What You Will Build in This Project

You will build **UrbanNest's complete Azure infrastructure** — the same architecture used by real estate and enterprise companies across India.

Resource Groups, Virtual Network, Virtual Machines, App Service, Blob Storage, Azure SQL, Azure Functions, AKS, Key Vault

> **🏗️ Phase 1 — Resource Groups, IAM & VNet**

> Understand Azure's organisation model. Create a Resource Group, assign RBAC roles, and build a Virtual Network with subnets and Network Security Groups.

> Launch a Linux VM for backend APIs, connect via SSH, and deploy UrbanNest's frontend using App Service — no VM management needed for the web app.

> Store property images in Azure Blob Storage with SAS tokens. Set up Azure SQL Database in a private subnet for UrbanNest's transactional data.

> Run serverless document processing with Functions. Deploy containerised microservices on AKS. Put an Azure Load Balancer in front for traffic distribution.

> Store secrets in Key Vault. Build a CI/CD pipeline in Azure DevOps that deploys on every Git push. Set up Azure Monitor alerts for the ops team.

**Scene 1 — UrbanNest Engineering, Pune | The Migration Brief**

> **Priya** _Senior Cloud Engineer — UrbanNest_
> 
> Ravi, UrbanNest's server crashed three times last quarter during property listing peaks. We're running on a single on-premises Windows Server — the database, the web app, and file storage all on the same machine. Any spike takes it down. Our CTO wants us on Azure in 90 days. The good news: everything you know from any cloud applies here. Azure has different names for some things but the architecture is identical. Resource Group is the container. Virtual Network is your VPC. VM is your EC2. Blob Storage is your S3.

> **Ravi (You)** _Junior Cloud Engineer — Day 1 at UrbanNest_
> 
> Where do I start? Do I go straight to creating VMs or is there an order I should follow?

> **Priya** _Senior Cloud Engineer_
> 
> Always the same order: organisation first, then network, then compute, then storage and database, then automation. In Azure: create the Resource Group first — it's the folder for all your resources. Then the Virtual Network — your private network. Then VMs and App Service — your compute. Then Blob Storage and SQL. Then Functions and AKS for advanced workloads. Then automate with Azure DevOps. Never jump ahead to VMs without having the network ready first.

> **Sameer** _Platform Architect — UrbanNest_
> 
> One Azure-specific thing you must understand immediately: everything in Azure lives inside a Resource Group. When you delete a Resource Group, everything inside it is deleted — VMs, databases, storage accounts, everything. This is both powerful (clean teardown of dev environments) and dangerous (one wrong delete in production). Tag everything. Name everything clearly. And never put production and dev resources in the same Resource Group.

### 1. Phase 1 — Resource Groups, RBAC & Virtual Network

> **Azure's Organisation Hierarchy — How Resources Are Structured**

> - **Management Group** — top level. Groups multiple Azure subscriptions. Used by large enterprises.
> - **Subscription** — a billing container. All resources you create belong to a subscription. You get billed per subscription. Most teams use one subscription per environment (dev/prod) or per business unit.
> - **Resource Group** — a logical container within a subscription. Group all resources for one project or environment here. Every Azure resource must belong to exactly one Resource Group.
> - **Resource** — an individual service: a VM, a storage account, a database, a virtual network.
> - Think of it as: Management Group → Subscription → Resource Group → Resources. Like a folder structure.

#### 1.1 Create a Resource Group

```
# Login to Azure
az login
# Create the Resource Group
az group create \
  --name urbannest-prod-rg \
  --location centralindia
# List available regions in India
az account list-locations \
  --query "[?contains(name,'india')]" \
  --output table
```

**📖 Resource Groups**

- **az login** — opens a browser for Microsoft account authentication. After login, all CLI commands run under your account.
- **centralindia** — Azure region code for Pune (Central India). Other India regions: southindia (Chennai), westindia (Mumbai).
- All resources created later will specify `--resource-group urbannest-prod-rg` — that places them inside this group.
- Use a **naming convention**: projectname-environment-resourcetype. E.g. urbannest-prod-rg, urbannest-dev-rg. Consistency is critical at scale.
- Resource Groups are **free** — no cost to create them. Create separate ones for each environment and each project component.

#### 1.2 RBAC — Role-Based Access Control

```
# Get your subscription ID
az account show --query id --output tsv
# Assign "Contributor" role to a developer
az role assignment create \
  --assignee ravi@urbannest.in \
  --role Contributor \
  --scope "/subscriptions/SUB_ID/resourceGroups/urbannest-prod-rg"
# Assign "Reader" role at subscription level
az role assignment create \
  --assignee intern@urbannest.in \
  --role Reader \
  --scope "/subscriptions/SUB_ID"
```

**📖 Azure RBAC**

- **RBAC** (Role-Based Access Control) — assign roles to users, groups, or service principals to control what they can do in Azure.
- **Owner** — full access including managing access. Reserve for subscription admins only.
- **Contributor** — can create and manage all resources but cannot grant access to others. For developers and DevOps engineers.
- **Reader** — can view all resources but cannot make changes. For auditors, interns, and read-only access.
- **--scope** — where the role applies. Can be subscription-wide, a specific Resource Group, or a single resource. Narrower scope = more secure.

#### 1.3 Virtual Network and Subnets

```
UrbanNest Azure Virtual Network — Central India Region
=======================================================

  VNet: urbannest-vnet  10.0.0.0/16
  │
  ├── Subnet: web-subnet       10.0.1.0/24   ← App Service + VMs (public-facing)
  ├── Subnet: app-subnet       10.0.2.0/24   ← API servers, AKS nodes
  ├── Subnet: db-subnet        10.0.3.0/24   ← Azure SQL (private, no internet)
  └── Subnet: mgmt-subnet      10.0.4.0/24   ← Bastion, management VMs

  Network Security Group (NSG) on each subnet:
  web-subnet NSG:  Allow 80, 443 from internet | Allow 22 from mgmt-subnet only
  db-subnet NSG:   Allow 1433 from app-subnet only | Deny all internet
  
  Private DNS Zone: urbannest.internal
  → sql.urbannest.internal → Azure SQL private endpoint IP
```

```
# Create the Virtual Network
az network vnet create \
  --resource-group urbannest-prod-rg \
  --name urbannest-vnet \
  --address-prefix 10.0.0.0/16 \
  --location centralindia
# Add a subnet for web servers
az network vnet subnet create \
  --resource-group urbannest-prod-rg \
  --vnet-name urbannest-vnet \
  --name web-subnet \
  --address-prefixes 10.0.1.0/24
```

**📖 Azure VNet vs AWS VPC**

- **Virtual Network (VNet)** — Azure's equivalent of AWS VPC. Your private, isolated network in the cloud.
- **Azure regions have multiple Availability Zones** (AZ1, AZ2, AZ3). Unlike AWS, Azure subnets span all AZs in the region by default — no subnet-per-AZ mapping needed.
- **No Internet Gateway resource** — Azure VMs with public IPs automatically have internet access. You don't create a separate gateway object (unlike AWS IGW).
- Use **Network Security Groups (NSG)** on subnets to control traffic — equivalent to AWS Security Groups and NACLs combined.

#### 1.4 Network Security Group — Firewall Rules

```
# Create NSG for the web subnet
az network nsg create \
  --resource-group urbannest-prod-rg \
  --name web-nsg
# Allow HTTPS from internet
az network nsg rule create \
  --resource-group urbannest-prod-rg \
  --nsg-name web-nsg \
  --name Allow-HTTPS \
  --priority 100 \
  --destination-port-range 443 \
  --protocol Tcp \
  --access Allow \
  --source-address-prefix Internet
```

**📖 NSG Rules**

- **NSG** — applies to a subnet or individual NIC. Controls inbound and outbound traffic with allow/deny rules.
- **--priority** — rules are evaluated lowest number first. Range 100–4096. Lower = higher priority. Rules stop evaluating once one matches.
- **Internet** — Azure service tag meaning "any public IP". Use instead of 0.0.0.0/0 for clarity.
- Other useful service tags: **VirtualNetwork** (all VNet IPs), **AzureLoadBalancer** (health probes), **AzureBastionSubnet**.
- Default rules at priority 65000–65500 **deny all** inbound from internet and allow all VNet-to-VNet. You override them with lower-priority rules.

### 2. Phase 2 — Virtual Machines and App Service

**Business Problem:** UrbanNest's backend API needs a VM with custom configuration. Their frontend React app needs a managed hosting environment that scales automatically without a DevOps engineer managing servers. Azure VM for the backend, Azure App Service for the frontend.

**Scene 2 — UrbanNest Architecture Planning | "Right Service for Each Workload"**

> **Priya** _Senior Cloud Engineer_
> 
> Ravi, one mistake beginners make is putting everything on VMs. A VM is raw compute — you manage the OS, patches, runtime, and everything above. App Service is a managed platform — you give it your code and Azure handles the infrastructure. Our React frontend: App Service. Our Node.js backend API: App Service or VM depending on complexity. Our PDF processing worker: Azure Functions. Our microservices: AKS. Match the workload to the right compute type — that is the most important architectural decision you'll make in Azure.

**Azure Compute Options — When to Use What**

- **🖥️ Virtual Machine** — 

- **🌐 App Service** — 

- **⚡ Azure Functions** — 

- **🐳 AKS (Kubernetes)** — 

#### 2.1 Create and Connect to a Virtual Machine

```
# Create Ubuntu 22.04 VM in the app subnet
az vm create \
  --resource-group urbannest-prod-rg \
  --name urbannest-api-vm \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name urbannest-vnet \
  --subnet app-subnet \
  --public-ip-sku Standard
```

**📖 VM Creation Parameters**

- **--image Ubuntu2204** — Ubuntu 22.04 LTS. Other popular images: Win2022Datacenter, RHEL92, Debian11. Full list: `az vm image list`.
- **Standard_B2s** — 2 vCPU, 4GB RAM. B-series = burstable, good for dev/test and variable workloads. D-series for steady production. Full list: `az vm list-sizes`.
- **--generate-ssh-keys** — creates a key pair automatically, saves private key to `~/.ssh/id_rsa`. No manual key creation needed.
- **--public-ip-sku Standard** — creates a static public IP (Standard SKU). Default is Basic SKU which is being deprecated.
- Default username for Azure Ubuntu VMs is **azureuser**. Use this when SSHing in.

```
# Get the VM's public IP
az vm show \
  --resource-group urbannest-prod-rg \
  --name urbannest-api-vm \
  --show-details \
  --query publicIps \
  --output tsv
# SSH into the VM
ssh azureuser@20.204.100.55
# Run script remotely via Azure CLI (no SSH needed)
az vm run-command invoke \
  --resource-group urbannest-prod-rg \
  --name urbannest-api-vm \
  --command-id RunShellScript \
  --scripts "apt-get update && apt-get install -y nodejs"
```

**📖 Connecting to VMs**

- **az vm show --show-details** — retrieves full VM details including public IP, private IP, OS disk, and power state.
- **az vm run-command invoke** — runs a shell command on the VM directly via Azure without needing SSH. Equivalent to AWS SSM Run Command. Useful for automation.
- **Azure Bastion** — connect to VMs via browser with no public IP required. Provides browser-based SSH/RDP in the Azure Portal. More secure than exposing port 22.
- To open port 22 only for your IP: `az vm open-port --port 22 --priority 900 --source-address-prefix YOUR_IP/32`

#### 2.2 Azure App Service — Managed Web Hosting

```
# Create an App Service Plan (the pricing tier)
az appservice plan create \
  --resource-group urbannest-prod-rg \
  --name urbannest-plan \
  --sku P1V3 \
  --is-linux
# Create the Web App on this plan
az webapp create \
  --resource-group urbannest-prod-rg \
  --plan urbannest-plan \
  --name urbannest-frontend \
  --runtime "NODE:20-lts"
```

**📖 App Service Plan vs App**

- **App Service Plan** — defines the region, OS, and pricing tier. One plan can host multiple apps. You pay for the plan, not per app.
- **SKU tiers:** F1 (Free, 60 min/day), B1 (Basic, dev), S1 (Standard, auto-scale), P1V3 (Premium, production). P1V3 = 2 vCPU, 8GB RAM.
- **--is-linux** — Linux-based App Service. Required for Node.js, Python, Ruby runtimes. Windows plans support .NET, classic ASP.
- App Service gives you a free domain: `urbannest-frontend.azurewebsites.net`. Add a custom domain with HTTPS cert for free via App Service managed certificates.
- **Deployment slots** — create a staging slot, deploy there, test, then swap to production with zero downtime. Included in S and P tiers.

```
# Deploy code via local Git
az webapp deployment source config-local-git \
  --resource-group urbannest-prod-rg \
  --name urbannest-frontend
# Set an environment variable (app setting)
az webapp config appsettings set \
  --resource-group urbannest-prod-rg \
  --name urbannest-frontend \
  --settings NODE_ENV=production API_URL=https://api.urbannest.in
# Enable auto-scale: 1 to 5 instances at 70% CPU
az monitor autoscale create \
  --resource-group urbannest-prod-rg \
  --name frontend-autoscale \
  --resource urbannest-plan \
  --min-count 1 --max-count 5 --count 1
```

**📖 Deploying to App Service**

- **Local Git deployment** — Azure generates a Git remote URL. `git push azure main` triggers a build and deployment automatically.
- **App settings** — environment variables for your app. Visible as OS environment variables to the process. More secure than hardcoding in code. Changes restart the app.
- **az monitor autoscale** — App Service auto-scale. Adds instances when CPU or memory threshold is crossed. Scales back in when load drops.
- Other deployment methods: GitHub Actions integration (push to GitHub → auto deploy), Docker container, Azure DevOps pipelines, zip deploy via REST API.

Microsoft Azure

Home › App Services › urbannest-frontend

URL https://urbannest-frontend.azurewebsites.net

App Service Plan urbannest-plan (P1V3)

Runtime stack Node 20 LTS on Linux

Region Central India

Instances 2 running (auto-scaled)

Custom domain urbannest.in (HTTPS ✓)

Browse

Deployment Center

Scale up (Plan)

### 3. Phase 3 — Blob Storage and Azure SQL

**Business Problem:** UrbanNest stores 200,000 property photos. Keeping them on VM disk is wrong — disk is expensive per GB, doesn't scale, and disappears with the VM. Azure Blob Storage is the right home: durable, cheap, globally accessible, with SAS tokens for secure time-limited sharing. The property database needs managed SQL Server that handles backups, HA, and patching automatically.

#### 3.1 Create a Storage Account and Blob Container

```
# Create a storage account (globally unique name)
az storage account create \
  --resource-group urbannest-prod-rg \
  --name urbanneststorage2026 \
  --location centralindia \
  --sku Standard_LRS \
  --kind StorageV2
# Create a container for property images
az storage container create \
  --account-name urbanneststorage2026 \
  --name property-images \
  --auth-mode login
```

**📖 Storage Account vs Blob Container**

- **Storage Account** — the namespace and billing unit. One account holds multiple containers. Name must be globally unique (3–24 lowercase letters/numbers).
- **Standard_LRS** — Locally Redundant Storage. 3 copies within one data centre. Cheapest. Other options: ZRS (3 zones), GRS (2 regions, 6 copies).
- **StorageV2** — current generation. Supports all features: Blob, Files, Queues, Tables. Always use StorageV2.
- **Blob Container** — like an S3 bucket prefix. Groups related files. Access can be private, blob-public, or container-public.
- **Never make a production container public** — use SAS tokens for controlled access instead.

#### 3.2 Upload Files and Generate SAS Token

```
# Upload a file to blob storage
az storage blob upload \
  --account-name urbanneststorage2026 \
  --container-name property-images \
  --name flat-001-main.jpg \
  --file ./images/flat-001.jpg \
  --auth-mode login
# Generate SAS token valid for 2 hours
az storage blob generate-sas \
  --account-name urbanneststorage2026 \
  --container-name property-images \
  --name flat-001-main.jpg \
  --permissions r \
  --expiry 2026-12-01T12:00:00Z \
  --output tsv
```

**📖 SAS Tokens**

- **SAS (Shared Access Signature)** — a time-limited, permission-limited URL token. Append it to a blob URL to grant temporary access.
- **--permissions r** — read only. Options: r (read), w (write), d (delete), l (list), c (create). Combine: `rwdl`.
- SAS token URL: `https://account.blob.core.windows.net/container/blob?sv=2023...&sig=...`
- When the expiry time passes, the link stops working automatically — no revocation needed.
- For app-to-blob access, use **Managed Identity** with RBAC (Storage Blob Data Contributor role) instead of SAS — more secure, no token management.

#### 3.3 Azure SQL Database

```
# Create the SQL Server logical server
az sql server create \
  --resource-group urbannest-prod-rg \
  --name urbannest-sqlserver \
  --location centralindia \
  --admin-user sqladmin \
  --admin-password 'UrbanNest@2026!'
# Create the database on that server
az sql db create \
  --resource-group urbannest-prod-rg \
  --server urbannest-sqlserver \
  --name urbanNestDB \
  --service-objective GP_Gen5_2 \
  --backup-storage-redundancy Zone
```

**📖 Azure SQL Structure**

- **SQL Server (logical server)** — the namespace and admin credentials. Not a VM — just a logical container. One server can host many databases.
- **SQL Database** — the actual database. Billed separately from the server.
- **GP_Gen5_2** — General Purpose tier, Gen5 hardware, 2 vCores. Service objective is Azure's term for instance size. Also: Basic, Standard, Business Critical.
- **--backup-storage-redundancy Zone** — backups replicated across availability zones. More durable than LRS (default).
- Azure SQL provides **automatic backups**: full weekly, differential daily, transaction log every 5–12 minutes. Retained up to 35 days by default.

```
# Disable public network access — private only
az sql server update \
  --resource-group urbannest-prod-rg \
  --name urbannest-sqlserver \
  --restrict-outbound-network-access true
# Create private endpoint (replaces public endpoint)
az network private-endpoint create \
  --resource-group urbannest-prod-rg \
  --name sql-private-endpoint \
  --vnet-name urbannest-vnet \
  --subnet db-subnet \
  --private-connection-resource-id \
                   /subscriptions/SUB_ID/.../urbannest-sqlserver \
  --group-id sqlServer \
  --connection-name sql-connection
```

**📖 Private Endpoint — No Public IP**

- **Private Endpoint** — gives the Azure SQL database a private IP address inside your VNet. Traffic to the database never leaves Microsoft's network.
- After creating the private endpoint, the database is only reachable from inside the VNet via its private IP (e.g. 10.0.3.5) — not from the public internet.
- This is the Azure equivalent of AWS's RDS with `--no-publicly-accessible` + VPC Endpoint.
- Configure a **Private DNS Zone** (`privatelink.database.windows.net`) so that `urbannest-sqlserver.database.windows.net` resolves to the private IP inside the VNet.

### 4. Phase 4 — Azure Functions, AKS & Load Balancer

**Business Problem:** UrbanNest receives PDF floor plans and brochures that need text extraction and resizing on upload. Running this on a VM 24/7 for a task that runs 200 times per day is wasteful. Azure Functions handles it serverlessly. The property recommendation microservice needs Kubernetes for orchestration and scaling.

#### 4.1 Azure Functions — Serverless Processing

```
# Create a Function App
az functionapp create \
  --resource-group urbannest-prod-rg \
  --name urbannest-functions \
  --storage-account urbanneststorage2026 \
  --consumption-plan-location centralindia \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --os-type Linux
```

**📖 Function App vs AWS Lambda**

- **Function App** — the container for one or more Azure Functions. Like a Lambda deployment package that can hold many functions.
- **--consumption-plan-location** — use the Consumption (serverless) plan. Pay only per execution. First 1 million executions/month are free.
- **Alternatives to Consumption plan:** Premium plan (pre-warmed instances, no cold starts, VNet integration), Dedicated plan (App Service plan, always-on).
- Supported runtimes: Python, Node.js, C#, Java, PowerShell. Max execution time: 5 min (Consumption), unlimited (Premium/Dedicated).

```python
# function_app.py — Blob-triggered function
import azure.functions as func
import logging

app = func.FunctionApp()

@app.blob_trigger(
  arg_name="blob",
  path="property-images/{name}",
  connection="AzureWebJobsStorage"
)
def process_property_image(blob: func.InputStream):
    logging.info(f"Processing: {blob.name} ({blob.length} bytes)")
    # Extract text, resize, generate thumbnails...
```

**📖 Blob Trigger Function**

- **@app.blob_trigger** — fires automatically when a new blob is created in `property-images/` container.
- **path with {name}** — the `{name}` binding captures the blob's filename and passes it to the function.
- **arg_name="blob"** — the function parameter name that receives the blob's content stream.
- Other common triggers: HTTP, Timer (cron schedule), Queue, Event Hub, Service Bus, Cosmos DB change feed.
- Azure Functions v4 with Python uses the **decorator-based programming model** — same pattern as Flask routes but for serverless triggers.

#### 4.2 AKS — Azure Kubernetes Service

```
# Create an AKS cluster
az aks create \
  --resource-group urbannest-prod-rg \
  --name urbannest-aks \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --enable-managed-identity \
  --vnet-subnet-id /subscriptions/.../app-subnet \
  --network-plugin azure \
  --generate-ssh-keys
# Get credentials to run kubectl
az aks get-credentials \
  --resource-group urbannest-prod-rg \
  --name urbannest-aks
```

**📖 AKS — Managed Kubernetes**

- **AKS** — Azure manages the Kubernetes control plane (API server, etcd, scheduler) for free. You only pay for the agent nodes (VMs).
- **--node-count 2** — start with 2 nodes (VMs) in the default node pool. AKS auto-scales nodes when pods can't be scheduled.
- **--enable-managed-identity** — the cluster uses a Managed Identity to pull images from ACR and manage Azure resources. No service principal credentials needed.
- **--network-plugin azure** — uses Azure CNI. Each pod gets a real VNet IP address (routable). Alternative: kubenet (pods use NAT, simpler).
- **az aks get-credentials** — merges AKS cluster credentials into your local kubeconfig. After this, `kubectl` commands work against your cluster.

```yaml
# Deploy a containerised service to AKS
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
name: recommendations
spec:
replicas: 2
selector:
matchLabels:
app: recommendations
template:
spec:
containers:
      - name: rec-engine
image: urbannestack.azurecr.io/rec:latest
ports:
        - containerPort: 5000
EOF
```

**📖 Kubernetes Deployment on AKS**

- **Deployment** — declares the desired state: 2 replicas of the container running at all times. If one crashes, Kubernetes replaces it automatically.
- **ACR (Azure Container Registry)** — Azure's private Docker registry. Format: `registryname.azurecr.io/image:tag`. AKS can pull from ACR with no extra credentials when Managed Identity is enabled.
- **replicas: 2** — two container instances, spread across nodes. If a node fails, pods are rescheduled on remaining nodes.
- To expose this deployment via a LoadBalancer Service: `kubectl expose deployment recommendations --type=LoadBalancer --port=80 --target-port=5000`

### 5. Phase 5 — Key Vault, Azure DevOps & Azure Monitor

**Business Problem:** UrbanNest's database password is hardcoded in config files. Deployment is manual — an engineer runs scripts. When something breaks at 2 AM, no one knows until a customer complains. Key Vault solves secrets management. Azure DevOps automates deployments. Azure Monitor sends alerts before customers notice problems.

#### 5.1 Azure Key Vault — Secrets Management

```
# Create the Key Vault
az keyvault create \
  --resource-group urbannest-prod-rg \
  --name urbannest-kv \
  --location centralindia \
  --sku standard
# Store a secret (database password)
az keyvault secret set \
  --vault-name urbannest-kv \
  --name sql-password \
  --value 'UrbanNest@2026!'
# Retrieve a secret
az keyvault secret show \
  --vault-name urbannest-kv \
  --name sql-password \
  --query value \
  --output tsv
```

**📖 Azure Key Vault**

- **Key Vault** stores secrets, keys, and certificates. Access is controlled by RBAC and vault access policies. Encrypted at rest using HSMs (Hardware Security Modules).
- **Vault name must be globally unique** — 3–24 chars, alphanumeric and hyphens only.
- To let an App Service read from Key Vault: enable the App Service's **Managed Identity**, then assign it the **Key Vault Secrets User** RBAC role on the vault.
- Reference secrets in App Service settings without storing the value: set app setting value to `@Microsoft.KeyVault(VaultName=urbannest-kv;SecretName=sql-password)` — Azure resolves it automatically at runtime.
- **Soft delete** is enabled by default — deleted secrets are recoverable for 90 days. Prevents accidental permanent deletion.

#### 5.2 Azure DevOps Pipeline — Automated CI/CD

```
# azure-pipelines.yml — in your repo root
trigger:
branches:
include: [main]

pool:
vmImage: ubuntu-latest
steps:
- task: NodeTool@0
inputs:
versionSpec: '20.x'

- script: npm install && npm test
displayName: 'Install and Test'

- task: AzureWebApp@1
inputs:
azureSubscription: 'UrbanNest-ServiceConnection'
appName: 'urbannest-frontend'
package: '$(Build.ArtifactStagingDirectory)/**/*.zip'
```

**📖 Azure Pipelines YAML**

- **trigger: branches: include: [main]** — pipeline runs automatically every time code is pushed to the main branch.
- **vmImage: ubuntu-latest** — Microsoft-hosted build agent. Ubuntu VM provisioned fresh for each run. Free for 1 parallel job (1,800 min/month).
- **tasks** — pre-built steps from the Azure DevOps marketplace. NodeTool installs Node.js, AzureWebApp deploys to App Service. Hundreds of tasks available.
- **azureSubscription** — a Service Connection configured in the Azure DevOps project settings. It holds the credentials to deploy to your Azure subscription. Create once, reuse across pipelines.
- **AzureWebApp@1 task** — equivalent of AWS CodeDeploy for App Service. Handles deployment, swap, and health check verification.

Azure DevOps

UrbanNest › Pipelines › urbannest-frontend-pipeline

INSTALL & TEST

✓ 47s

All 23 tests passed

→

BUILD

✓ 1m 12s

npm build → zip

→

DEPLOY

✓ 38s

App Service → ✓

Triggered by ravi@urbannest.in pushed "feat: add property filters" to main

Total duration 2 min 37 sec | Deployed to: urbannest-frontend.azurewebsites.net

#### 5.3 Azure Monitor — Alerts and Logs

```
# Create an action group (who to notify)
az monitor action-group create \
  --resource-group urbannest-prod-rg \
  --name ops-team-alerts \
  --short-name UrbanOps \
  --email-receiver name="Priya" email-address=priya@urbannest.in
# Alert when VM CPU > 85% for 5 minutes
az monitor metrics alert create \
  --resource-group urbannest-prod-rg \
  --name high-cpu-alert \
  --scopes /subscriptions/.../urbannest-api-vm \
  --condition "avg Percentage CPU > 85" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action ops-team-alerts
```

**📖 Azure Monitor + Action Groups**

- **Action Group** — who gets notified when an alert fires. Can include email, SMS, push notification, Logic App webhook, or Azure Function call.
- **--window-size 5m** — evaluate the last 5 minutes of metric data. Alert fires only if the condition is true for the entire window (not just a spike).
- **--evaluation-frequency 1m** — check the condition every 1 minute. Combined with 5m window = alerts within 5 minutes of a sustained issue.
- **Azure Monitor Logs (Log Analytics)** — collects and queries logs from all Azure resources using KQL (Kusto Query Language). Query VM logs, App Service logs, AKS logs in one place.
- **Application Insights** — APM (Application Performance Monitoring). Add the SDK to your app to track request rates, failures, dependencies, and traces automatically.

### 6. Azure Services — Quick Reference

Azure Service

AWS Equivalent

Key CLI Command

Resource Group

No direct equivalent (closest: Tags)

az group create / delete

Virtual Network (VNet)

VPC

az network vnet create

Network Security Group

Security Group + NACL

az network nsg rule create

Virtual Machine

EC2

az vm create / start / stop / run-command

App Service

Elastic Beanstalk

az webapp create / deploy

Azure Functions

Lambda

az functionapp create / deploy

AKS

EKS

az aks create / get-credentials

ACR (Container Registry)

ECR

az acr create + docker push

Blob Storage

S3

az storage blob upload / generate-sas

Azure SQL Database

RDS

az sql db create

Cosmos DB

DynamoDB

az cosmosdb create

Azure Cache for Redis

ElastiCache

az redis create

Key Vault

Secrets Manager

az keyvault secret set / show

Azure DevOps Pipelines

CodePipeline

az pipelines create / run

Azure Monitor / Alerts

CloudWatch

az monitor metrics alert create

Application Insights

CloudWatch Application Signals

az monitor app-insights component create

Azure Load Balancer

Network Load Balancer

az network lb create

Azure Front Door / CDN

CloudFront

az afd profile create

Azure DNS

Route 53

az network dns zone create

Managed Identity

IAM Role (for EC2/Lambda)

az identity create / assign

### 7. Interview Questions — Azure

##### Interview Q&A — Fresher Level (0–1 Year Azure Experience)

**Q: Q1. What is a Resource Group in Azure and why is it important?**

A: A **Resource Group** is a logical container that holds related Azure resources (VMs, databases, storage accounts, VNets) for a single application or environment.
Every Azure resource must belong to exactly one Resource Group. You cannot move some resources between groups after creation.
**Lifecycle management** — deleting a Resource Group deletes all resources inside it. Great for cleaning up dev/test environments. Dangerous in production.
**Access control** — RBAC roles assigned at the Resource Group level apply to all resources inside. Assign a developer Contributor on the dev Resource Group — they can't touch prod.
**Cost tracking** — Azure bills show costs per Resource Group. Tag Resource Groups by project and environment for clear cost attribution.
Best practice: separate Resource Groups for each environment (dev, staging, prod) and never share a Resource Group between environments.

**Q: Q2. What is the difference between Azure App Service and Azure Functions?**

A: **App Service** — managed PaaS for web applications and REST APIs that run continuously. You choose a pricing tier (B1, S1, P1V3) and pay hourly for the plan regardless of traffic. Supports auto-scaling. Best for always-on services like your frontend, API, or CMS.
**Azure Functions** — serverless compute for event-driven, short-duration tasks. You pay only per execution. Scales to zero when idle — no traffic, no cost. Best for image processing on upload, queue message processing, scheduled jobs, webhooks, and email triggers.
Key decision: "Does this code run all the time, or only when something happens?" Always-on → App Service. Event-driven → Functions.
Azure Functions have execution time limits: 5 minutes (Consumption plan), up to 60 minutes (Premium). Not suitable for long-running batch jobs — use App Service or Azure Batch instead.

**Q: Q3. What is Managed Identity in Azure and why should you use it?**

A: A **Managed Identity** is an automatically managed Azure AD identity for a resource (VM, App Service, Function App, AKS). Azure creates and rotates the credentials automatically.
Instead of storing a storage account key or Key Vault password in your application config, you enable Managed Identity on the resource, then assign it an RBAC role (e.g. "Storage Blob Data Reader" on the storage account).
Your application calls the Azure SDK — it automatically uses the Managed Identity to authenticate. **No credentials in code, config files, or environment variables.**
**System-assigned** — tied to one resource. Deleted when the resource is deleted. Simplest for single-resource scenarios.
**User-assigned** — a standalone identity that can be assigned to multiple resources. Useful when multiple VMs all need the same access (e.g. all VMs in a scale set access the same Key Vault).

**Q: Q4. What is the difference between Azure Blob Storage access tiers?**

A: **Hot tier** — highest storage cost, lowest access cost. For data accessed frequently (product images, active documents, current backups).
**Cool tier** — lower storage cost, higher access cost. For data accessed occasionally (monthly reports, older backups). Minimum storage duration of 30 days.
**Archive tier** — lowest storage cost (~₹0.002/GB/month), very high access cost, and retrieval takes hours. For compliance data, old logs, rarely-accessed archives. Minimum 180-day retention.
**Lifecycle Management Policy** — automatically moves blobs between tiers based on age. Example rule: "move to Cool after 30 days, Archive after 90 days, delete after 365 days." Runs daily, costs nothing to configure.
Unlike S3, Azure blob tier changes are immediate for Hot↔Cool but Archive requires explicit rehydration (hours to days) before reading.

**Q: Q5. How does Azure DevOps differ from GitHub Actions for CI/CD?**

A: **Azure DevOps** — Microsoft's end-to-end DevOps platform: Repos (Git), Pipelines (CI/CD), Boards (project management), Artifacts (package management), Test Plans. Used heavily in enterprise environments, especially with existing Microsoft infrastructure.
**GitHub Actions** — CI/CD built into GitHub. YAML-based workflows. Massive community marketplace. Better for open-source projects and teams already on GitHub.
**Key Azure DevOps advantages:** tight Azure integration (Service Connections, native Azure tasks), advanced release management with approvals and gates, YAML and Classic (GUI) pipeline modes, dedicated self-hosted agent pools.
**Key GitHub Actions advantages:** simpler syntax, GitHub Marketplace has 20,000+ pre-built actions, free for public repos, excellent for open-source.
Both can deploy to Azure resources. Most Indian enterprises using Azure choose Azure DevOps. Startups and open-source projects lean toward GitHub Actions.

**Q: Q6. What is Azure Private Endpoint and when do you need it?**

A: A **Private Endpoint** gives an Azure PaaS service (Azure SQL, Storage, Key Vault, etc.) a private IP address inside your VNet. Traffic between your VMs and the service never leaves Microsoft's network.
Without Private Endpoint, your VM communicates with Azure SQL over its public internet hostname — even though both are in Azure, traffic may route over public internet depending on DNS resolution.
With Private Endpoint: Azure SQL gets a private IP (e.g. 10.0.3.5 in your db-subnet). Your VM connects to 10.0.3.5 — entirely within the private network. NSG rules can control access.
Required for compliance-sensitive data (banking, healthcare, government) where data must never traverse the public internet.
Paired with a **Private DNS Zone** — so `yourserver.database.windows.net` resolves to the private IP inside the VNet, and to the public IP outside. Transparent to applications — they use the same connection string.

**Quiz: Quiz 1 — A developer runs "az group delete --name urbannest-prod-rg --yes". What happens?**

- A) Only the Resource Group metadata is deleted — resources stay
- B) The Resource Group and every resource inside it (VMs, databases, storage, VNet) are permanently deleted
- C) Azure moves the resources to the default Resource Group before deleting
- D) The command fails because Resource Groups with resources cannot be deleted

> **Answer/explanation:** ✅ Answer: **B — Everything inside is permanently deleted**
**az group delete** cascades — it deletes the Resource Group AND every resource within it: all VMs, databases, storage accounts, VNets, NSGs.
This is why separating dev and prod into different Resource Groups is critical. A developer with Contributor access on the dev RG can delete it without affecting prod.
The **--yes** flag skips the confirmation prompt. Without it, Azure asks "Are you sure?" — always let that prompt run in production.
Soft-deleted resources (Key Vault secrets, Storage with soft delete) may be recoverable. But VMs, databases, and most resources are gone permanently.

**Quiz: Quiz 2 — UrbanNest wants the App Service to connect to Key Vault without storing any credentials. What is the correct approach?**

- A) Store the Key Vault access key in an App Service environment variable
- B) Enable System-assigned Managed Identity on the App Service, then grant it the "Key Vault Secrets User" RBAC role on the vault
- C) Create an Azure AD service principal and store its client secret in the App Service
- D) Make the Key Vault publicly accessible so the App Service can read from it

> **Answer/explanation:** ✅ Answer: **B — Managed Identity + RBAC role**
Managed Identity means Azure manages the credentials automatically. The App Service can prove its identity to Key Vault without storing any keys.
Options A and C both require storing credentials somewhere — defeating the purpose of Key Vault.
Option D (public Key Vault) removes all security — anyone on the internet could access your secrets.
The reference syntax in App Settings: `@Microsoft.KeyVault(VaultName=urbannest-kv;SecretName=sql-password)` — Azure resolves this at runtime using the Managed Identity automatically.

**Quiz: Quiz 3 — UrbanNest's App Service is currently running on B1 SKU (1 vCPU, 1.75GB). During property listing peaks, CPU hits 100% and the app becomes slow. What is the BEST fix?**

- A) Create a second App Service and balance traffic manually
- B) Upgrade to S1 or P1V3 SKU and configure auto-scale rules to add instances when CPU > 70%
- C) Move the app to a VM and add more RAM manually
- D) Restart the App Service every hour to clear memory

> **Answer/explanation:** ✅ Answer: **B — Upgrade SKU and enable auto-scale**
B1 (Basic) tier does not support auto-scaling. You must upgrade to Standard (S1) or Premium (P1V3) to unlock auto-scale.
With auto-scale: at 70% CPU, Azure adds another instance. At 30% CPU (after load drops), Azure removes it. You pay only for the extra instances during peak periods.
This is horizontal scaling — more instances of the same size. Much more resilient than vertical scaling (bigger VM) because one instance failure doesn't take down the service.
P1V3 is preferred for production: dedicated resources (not shared with other tenants like B-series), VNet integration, faster performance, more memory.

> **Azure Project — Core Takeaways for Freshers**

> - **Resource Groups are your foundation** — create them first, name them consistently, separate dev from prod. One wrong delete wipes everything inside. This is Azure's most important organisational concept.
> - **Never hardcode credentials** — always use Managed Identity + RBAC for Azure-to-Azure authentication. Use Key Vault references in App Settings. Never put connection strings in source code or environment variables as plain text.
> - **Choose compute based on workload type** — App Service for always-on web apps, Functions for event-driven short tasks, AKS for containerised microservices, VMs when you need full OS control. Mismatching compute to workload wastes money and creates operational burden.
> - **Private Endpoints for all data services** — Azure SQL, Storage, Key Vault, Cosmos DB should use Private Endpoints in production. Public endpoints are off for data that matters. NSG rules on db-subnet should allow only the app-subnet.
> - **Automate deployments with Azure DevOps** — manual deployments lead to human errors, especially late at night. Pipelines that run on git push are consistent, auditable, and faster than any manual process.
> - **Azure Monitor + Action Groups before going live** — set CPU, memory, and error rate alerts before your first production deployment. Know about problems from an automated email before your customers do.
> - **Tags on everything** — at minimum: Environment (dev/prod), Project, Owner. Without tags, Azure Cost Management shows a ₹50,000 bill with no way to tell which team or project generated it.
> - **Azure regions in India** — use centralindia (Pune) or southindia (Chennai) as primary. Add the other as disaster recovery. Both regions have multiple AZs for high availability within the region.

##### Azure Engineering Standards — UrbanNest Rules

- Always use **Azure Bicep or Terraform** for infrastructure as code — never create production resources by clicking in the Portal. Clicking cannot be version-controlled, reviewed, or reproduced reliably
- Enable **Azure Defender for Cloud** (formerly Security Center) on all subscriptions — it continuously scans your configuration for security misconfigurations and gives a Secure Score with specific remediation steps
- Set **budget alerts** in Azure Cost Management at 80% and 100% of monthly budget — cloud costs compound quickly when auto-scale is enabled; know before the invoice arrives
- Use **deployment slots** in App Service for all production deployments — deploy to staging slot, run smoke tests, then swap. If post-swap issues appear, swap back in 30 seconds
- Enable **diagnostic settings** on every resource and route logs to a Log Analytics Workspace — centralised logging is required for security audits, incident response, and AKS troubleshooting
- Review **Azure Advisor recommendations** weekly — it automatically identifies over-provisioned VMs, unused resources, security gaps, and reliability issues specific to your subscription

##### 🏋️ Hands-On Exercises — Extend the UrbanNest Azure Infrastructure

1. **Implement Bicep Infrastructure as Code:** Convert all five phases of UrbanNest's infrastructure into a single Bicep file. Bicep is Azure's native IaC language — cleaner than ARM templates, similar to Terraform. Define the Resource Group, VNet, subnets, NSG, VM, App Service, Storage Account, and SQL Database as Bicep resources. Deploy with `az deployment sub create --template-file main.bicep --parameters location=centralindia`. Verify the entire stack deploys from scratch in one command.
2. **Configure Azure Front Door for global load balancing:** UrbanNest is expanding to Delhi, Mumbai, and Pune users. Create an Azure Front Door profile with UrbanNest App Service as the origin. Enable caching for static assets (.js, .css, images) at the edge. Add a custom domain and a free Azure-managed TLS certificate. Measure response time before and after from different Indian cities using `curl -o /dev/null -w "%{time_total}" https://urbannest.in/`.
3. **Set up AKS with Azure Container Registry integration:** Create an Azure Container Registry (ACR). Build the UrbanNest recommendations service Docker image and push it to ACR. Attach the ACR to the AKS cluster with `az aks update --attach-acr`. Deploy the container to AKS referencing the ACR image. Configure an HPA (Horizontal Pod Autoscaler) to scale from 2 to 10 pods when CPU exceeds 60%.
4. **Build a complete Key Vault + Managed Identity setup:** Enable System-assigned Managed Identity on the UrbanNest App Service. Assign it the "Key Vault Secrets User" role on urbannest-kv. Move all app settings (SQL connection string, storage account key, Blob SAS key) to Key Vault secrets. Replace each app setting value with a Key Vault reference. Verify the app starts and connects to the database correctly — with zero credentials stored in App Service settings.
5. **Implement Azure Monitor full observability:** Enable Application Insights on the App Service and add the Node.js SDK. Create a Log Analytics Workspace and connect it to the VM, App Service, and AKS cluster diagnostic settings. Write two KQL queries: (1) the top 5 slowest API endpoints in the last 24 hours, (2) all failed requests with status code 5xx in the last hour. Create metric alerts for App Service response time > 2 seconds and VM disk usage > 80%. Configure weekly digest emails via Azure Monitor scheduled reports.

### Azure Project Complete 🎉

You have built UrbanNest's complete Azure infrastructure — Resource Groups and RBAC, Virtual Network with private subnets, VMs and App Service, Blob Storage and Azure SQL with Private Endpoints, Azure Functions, AKS, Azure DevOps CI/CD pipeline, Key Vault for secrets, and Azure Monitor for observability. This is the architecture running at thousands of Indian enterprises right now.

> **Priya**
> 
> "Ravi, UrbanNest launched on Azure last month. The Dussehra season brought 12,000 concurrent users — our worst-case scenario. App Service auto-scaled from 2 to 8 instances in 4 minutes. The Functions processed 3,400 property document uploads that day. AKS kept the recommendations service at 99.9% uptime. Azure Monitor paged me once — a VM disk at 78%, which we resolved before it became a problem. The on-premises server that crashed every quarter is decommissioned. We're paying ₹52,000 a month for the exact same resources our previous server cost ₹1.8 lakh per year to maintain — plus downtime."

> **Sameer**
> 
> "The security audit passed. No credentials in code. No public endpoints on the database. Every developer access is logged. Key Vault references replaced every hardcoded connection string. The auditor said our Azure setup scores higher than clients ten times our size. That's because you understood the principles — not just the commands. Managed Identity, Private Endpoints, NSG rules, RBAC — you built it right the first time."

> **Next: Advanced Azure — Serverless Architecture, Azure AD, and Cost Optimisation**

> - Azure Logic Apps — low-code workflow automation; integrate Salesforce, SAP, Office 365, and 400+ connectors without writing code
> - Azure AD / Entra ID — enterprise identity, SSO, Conditional Access, B2C for customer login, service principals for CI/CD
> - Azure Service Bus — enterprise message broker with topics, subscriptions, and dead-letter queues for reliable microservice communication
> - Azure Cosmos DB — globally distributed NoSQL with 5 consistency levels; <10ms reads globally; multi-region writes
> - Azure Policy — enforce governance at scale; require tags on all resources, restrict VM SKUs, mandate private endpoints
> - Cost Optimisation — Reserved Instances (up to 72% savings), Spot VMs, Azure Hybrid Benefit, Savings Plans, right-sizing recommendations
> - Azure Bicep & Terraform — infrastructure as code at enterprise scale; modules, parameter files, remote state management
