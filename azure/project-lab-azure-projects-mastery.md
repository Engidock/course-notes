# Azure Projects Mastery

> **👋 Hey Fresher — Read This First!**

> - **Why Azure projects matter:** Reading Azure documentation alone does not get you hired — hands-on project experience does; interviewers ask "tell me about a project you built on Azure"
> - **Azure is the #2 cloud provider globally** — only AWS has a larger market share; enterprises already embedded in the Microsoft ecosystem (Office 365, Teams, Active Directory) naturally choose Azure
> - **Projects = portfolio = career:** every project you build and document on GitHub is a permanent proof of skill that outlasts any certification on its own
> - **Free Azure resources:** create a free Azure account at azure.microsoft.com/free — ₹13,500 in credits for 30 days + 55+ services always free; use this for all beginner projects
> - **Company in this guide:** CloudNest Solutions — a Pune-based IT consulting firm that just migrated from on-premise to Azure. You joined as Cloud Engineer. Lead: **Meera**. Each project solves a real CloudNest business problem

#### What You Will Build Across 12 Projects

- **Beginner (Projects 1–3):** Web App Deployment, Azure Blob Storage, Virtual Machine Setup — get comfortable with core Azure services
- **Intermediate (Projects 4–6):** Virtual Network, Azure SQL Database, CI/CD Pipeline — combine multiple services and solve real problems
- **Advanced (Projects 7–12):** Serverless Architecture, Disaster Recovery, Security, Big Data, ML Pipeline, Data Governance — enterprise-grade solutions

App Service, Blob Storage, Virtual Machines, VNet + NSG, Azure SQL, Azure DevOps, Azure Functions, Site Recovery, Sentinel, Synapse Analytics, Azure ML, Data Governance

#### CloudNest Azure Architecture — All 12 Projects on One Platform

```
CloudNest Solutions — Azure Projects Architecture Overview
══════════════════════════════════════════════════════════════════════════

  Azure Subscription: cloudnest-prod  |  Region: Central India

  ┌─────────────────────────────────────────────────────────────────────┐
  │  Resource Group: rg-cloudnest-prod                                  │
  │                                                                     │
  │  COMPUTE                 STORAGE              NETWORKING            │
  │  ┌─────────────┐         ┌──────────────┐     ┌───────────────────┐ │
  │  │ App Service │         │ Blob Storage │     │  Virtual Network  │ │
  │  │ (Project 1) │         │ (Project 2)  │     │  (Project 4)      │ │
  │  │ Web App     │         │ Files/Images │     │  Subnets + NSGs   │ │
  │  └─────────────┘         └──────────────┘     └───────────────────┘ │
  │  ┌─────────────┐         ┌──────────────┐                           │
  │  │  Azure VM   │         │  Azure SQL   │     DEVOPS                │
  │  │ (Project 3) │         │ (Project 5)  │     ┌───────────────────┐ │
  │  │ Linux/Win   │         │  DB + Tables │     │  Azure DevOps     │ │
  │  └─────────────┘         └──────────────┘     │  CI/CD Pipeline   │ │
  │                                               │  (Project 6)      │ │
  │  SERVERLESS              SECURITY             └───────────────────┘ │
  │  ┌─────────────┐         ┌──────────────┐                           │
  │  │ Az Functions│         │  Sentinel    │     DATA & AI             │
  │  │ (Project 7) │         │  Key Vault   │     ┌───────────────────┐ │
  │  │ Event Grid  │         │  (Project 9) │     │ Synapse Analytics │ │
  │  └─────────────┘         └──────────────┘     │ Data Lake         │ │
  │  ┌─────────────┐         ┌──────────────┐     │ Azure ML          │ │
  │  │ Site Recovery│         │  Purview     │     │ (Projects 10-11) │ │
  │  │ (Project 8) │         │  (Project 12)│     └───────────────────┘ │
  │  │ DR Planning │         │  Governance  │                           │
  │  └─────────────┘         └──────────────┘                           │
  └─────────────────────────────────────────────────────────────────────┘
```

### 🟦 Beginner Projects (1–3) — Get Comfortable with Azure Basics

**Goal:** Build confidence with core Azure services. Each project is self-contained, takes 1–3 hours, and uses the Azure Portal with minimal CLI. Perfect for AZ-900 preparation.

**Scene — CloudNest Onboarding | "Your First Week on Azure"**

> **Meera** _Lead Cloud Engineer — CloudNest Solutions_
> 
> Welcome to CloudNest. Your first week: deploy a web app, set up file storage for our clients, and provision a Linux VM we can use as a jump box. These are the three most common asks from our clients every week. Master these three and you'll handle 60% of the day-to-day work. Don't just click through — understand what each service does and why we're choosing it over the alternatives.

**Business Problem:** CloudNest needs to host a client's company website — a Node.js app — on Azure without managing servers. App Service is fully managed: no OS patching, no IIS/Nginx config, no VM sizing.

Azure App Service App Service Plan Azure Resource Manager GitHub Deployment

#### 1.1 Create App Service Plan and Web App

```
# Create resource group
az group create \
  --name rg-cloudnest-webapp \
  --location centralindia

# Create App Service Plan (B1 = Basic tier)
az appservice plan create \
  --name plan-cloudnest \
  --resource-group rg-cloudnest-webapp \
  --sku B1 \
  --is-linux
# Create the Web App
az webapp create \
  --name cloudnest-client-site \
  --resource-group rg-cloudnest-webapp \
  --plan plan-cloudnest \
  --runtime "NODE:20-lts"
```

**📖 App Service vs VM**

- **App Service Plan** — the underlying compute; B1 = 1 vCore, 1.75 GB RAM; you pay for the plan, not per request
- **--is-linux** — Linux-based container runtime; cheaper than Windows App Service plans
- **Why App Service over VM:** Azure manages OS updates, load balancer, SSL, auto-restart on crash — you only manage your application code
- Free tier available: F1 (shared compute, 60 CPU minutes/day) — good for testing, not production
- Access your app: `https://cloudnest-client-site.azurewebsites.net` — DNS auto-configured

#### 1.2 Deploy Code from GitHub

```
# Enable GitHub Actions deployment
az webapp deployment source config \
  --name cloudnest-client-site \
  --resource-group rg-cloudnest-webapp \
  --repo-url https://github.com/cloudnest/client-site \
  --branch main \
  --manual-integration
# Or deploy a ZIP directly
az webapp deploy \
  --name cloudnest-client-site \
  --resource-group rg-cloudnest-webapp \
  --src-path ./dist.zip \
  --type zip
```

**📖 Deployment Options**

- **GitHub Actions** — push to main branch → Azure auto-deploys; best for team workflows
- **ZIP deploy** — fastest for quick manual deploys; good for demos and testing
- **Azure DevOps pipeline** — enterprise CI/CD with gating, approvals, rollback (covered in Project 6)
- View deployment logs: Azure Portal → App Service → Deployment Center → Logs
- Scale up: change App Service Plan SKU; scale out: increase instance count for load distribution

#### 1.3 Configure App Settings and Custom Domain

```
# Add environment variables (replaces .env files)
az webapp config appsettings set \
  --name cloudnest-client-site \
  --resource-group rg-cloudnest-webapp \
  --settings \
    NODE_ENV=production \
    API_ENDPOINT=https://api.cloudnest.in \
    PORT=8080

# Check running status
az webapp show \
  --name cloudnest-client-site \
  --query state
```

**📖 App Settings are Environment Variables**

- **appsettings** — injected as environment variables at runtime; app reads them via `process.env.NODE_ENV`
- Never store secrets here — use Azure Key Vault references instead: `@Microsoft.KeyVault(SecretUri=...)`
- App restarts automatically when settings change
- Custom domain: add a CNAME record in DNS, then `az webapp config hostname add --hostname www.clientsite.in`
- Free managed SSL: `az webapp config ssl bind --certificate-thumbprint ... --ssl-type SNI`

**Business Problem:** CloudNest's clients upload invoice PDFs, contract documents, and product images. Storing them on VMs is fragile and doesn't scale. Azure Blob Storage is infinitely scalable, highly available, and geo-redundant object storage.

Azure Blob Storage Storage Accounts Azure Storage Explorer SAS Tokens

#### 2.1 Create Storage Account and Containers

```
# Create storage account
az storage account create \
  --name cloudneststorage \
  --resource-group rg-cloudnest-webapp \
  --location centralindia \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot

# Create blob containers
az storage container create \
  --name invoices \
  --account-name cloudneststorage \
  --public-access off

az storage container create \
  --name product-images \
  --account-name cloudneststorage \
  --public-access blob
```

**📖 Storage Tiers and Redundancy**

- **Standard_LRS** — Locally Redundant Storage; 3 copies within one datacenter; cheapest; good for dev/test
- **Standard_GRS** — Geo-Redundant; 6 copies across 2 regions; use for production data
- **Hot tier** — frequently accessed data; lower read cost, higher storage cost
- **Cool tier** — infrequently accessed; cheaper storage, higher read cost; ideal for invoices older than 30 days
- **public-access: off** — invoices are private; only accessible via SAS token or Managed Identity
- **public-access: blob** — product images served publicly via URL without authentication

#### 2.2 Upload Files and Generate SAS Token

```
# Upload a file to blob storage
az storage blob upload \
  --account-name cloudneststorage \
  --container-name invoices \
  --name invoice-2026-001.pdf \
  --file ./invoice-2026-001.pdf

# Generate a time-limited SAS URL (1 hour)
az storage blob generate-sas \
  --account-name cloudneststorage \
  --container-name invoices \
  --name invoice-2026-001.pdf \
  --permissions r \
  --expiry 2026-04-01T02:00:00Z \
  --output tsv
```

**📖 SAS Tokens — Secure Sharing**

- **SAS (Shared Access Signature)** — a time-limited, permission-scoped URL for accessing a specific blob
- **--permissions r** — read-only; the recipient can download but not modify or delete
- Share the SAS URL with a client for 1 hour — after expiry, the URL returns 403 Forbidden automatically
- Never share the storage account key — it grants full access to everything in the account
- Lifecycle policy: auto-move blobs from Hot → Cool after 30 days → Archive after 180 days to save cost

**Business Problem:** CloudNest needs a secure entry point into the Azure environment — a Linux VM that acts as a jump box for SSH access to private resources. Admins connect to this VM first, then hop to internal VMs.

Azure Virtual Machines SSH Key Auth NSG Rules VM Configuration

#### 3.1 Create and Connect to a Linux VM

```
# Create Ubuntu 22.04 VM with SSH key auth
az vm create \
  --resource-group rg-cloudnest-webapp \
  --name vm-jumpbox \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard

# Open SSH port 22 only
az vm open-port \
  --port 22 \
  --resource-group rg-cloudnest-webapp \
  --name vm-jumpbox

# SSH into the VM
ssh azureuser@<public-ip>
```

**📖 VM Sizing and Cost**

- **Standard_B1s** — 1 vCPU, 1 GB RAM; burstable; ideal for a jump box that sits idle most of the time; ~₹600/month
- **--generate-ssh-keys** — creates an RSA key pair; private key saved to `~/.ssh/id_rsa` locally; public key installed on the VM
- **No password auth** — SSH keys only; much harder to brute-force than passwords
- Cost tip: **deallocate** the VM when not in use: `az vm deallocate --name vm-jumpbox` — stops billing for compute; you only pay for the OS disk
- For Windows VMs: use RDP (port 3389) instead of SSH

### 🟩 Intermediate Projects (4–6) — Combine Services, Solve Real Problems

**Goal:** Go beyond single services. These projects require connecting multiple Azure resources and understanding how they interact. Ideal for AZ-104 (Azure Administrator) preparation.

**Scene — CloudNest Month 2 | "The Client Wants Production-Grade Setup"**

> **Meera** _Lead Cloud Engineer — CloudNest Solutions_
> 
> The client just signed a 2-year contract. They want their dev and prod environments fully isolated — separate VNets, NSGs that block dev from touching prod databases. They want a proper SQL database setup with backups and read replicas. And their team deploys 3 times a day — we need a CI/CD pipeline so each push to main auto-deploys to staging, and a human approves before prod. These are not optional — this is what enterprise clients expect.

**Business Problem:** CloudNest's dev and prod VMs must not be able to reach each other. NSGs act as firewalls at the subnet level — controlling which traffic is allowed in and out of each tier.

Azure Virtual Network Subnets Network Security Groups VNet Peering

#### 4.1 Create VNet with Subnets

```
# Create the main VNet
az network vnet create \
  --name vnet-cloudnest \
  --resource-group rg-cloudnest-webapp \
  --address-prefix 10.0.0.0/16

# Create subnets for each tier
az network vnet subnet create \
  --name subnet-web \
  --vnet-name vnet-cloudnest \
  --resource-group rg-cloudnest-webapp \
  --address-prefix 10.0.1.0/24

az network vnet subnet create \
  --name subnet-db \
  --vnet-name vnet-cloudnest \
  --resource-group rg-cloudnest-webapp \
  --address-prefix 10.0.2.0/24
```

**📖 VNet Design Principles**

- **10.0.0.0/16** — 65,534 usable IPs; plan generously; impossible to expand a VNet CIDR after creation without disruption
- **subnet-web** — web servers and app tier; accessible from the internet via Load Balancer
- **subnet-db** — database tier; no direct internet access; only reachable from subnet-web
- Resources in the same VNet can communicate freely by default — NSGs add restrictions on top
- VNet Peering connects two VNets (even across regions) — traffic stays on Azure backbone, not public internet

#### 4.2 Create NSG Rules — Block Direct DB Access

```
# Create NSG for the database subnet
az network nsg create \
  --name nsg-db \
  --resource-group rg-cloudnest-webapp

# Allow ONLY web subnet to reach SQL port 1433
az network nsg rule create \
  --nsg-name nsg-db \
  --resource-group rg-cloudnest-webapp \
  --name allow-web-to-db \
  --priority 100 \
  --source-address-prefixes 10.0.1.0/24 \
  --destination-port-ranges 1433 \
  --protocol Tcp \
  --access Allow

# Deny everything else inbound
az network nsg rule create \
  --nsg-name nsg-db \
  --name deny-all-inbound \
  --priority 4000 \
  --source-address-prefixes "*" \
  --destination-port-ranges "*" \
  --access Deny \
  --resource-group rg-cloudnest-webapp
```

**📖 NSG Rule Priority**

- **Priority 100** — lower number = processed first; range is 100–4096
- **allow-web-to-db** — only the web subnet (10.0.1.0/24) can reach port 1433; any other source is blocked
- **deny-all-inbound at 4000** — catches everything not already allowed; explicit deny is clearer than relying on the implicit default deny
- NSG can be attached to a subnet (applies to all VMs in it) or to a specific NIC (applies to one VM only)
- Effective security rules: Azure Portal → NIC → Effective Security Rules — shows exactly what is allowed/denied and why

**Business Problem:** CloudNest's client app needs a relational database for orders, users, and inventory. Azure SQL is a fully managed PaaS — no SQL Server instance to manage, patch, or back up manually.

Azure SQL Database SQL Server Firewall Rules Elastic Pools

#### 5.1 Create SQL Server and Database

```
# Create the logical SQL Server
az sql server create \
  --name sql-cloudnest \
  --resource-group rg-cloudnest-webapp \
  --location centralindia \
  --admin-user sqladmin \
  --admin-password "Str0ng!Pass2026"

# Create database (General Purpose, 4 vCores)
az sql db create \
  --name db-clientapp \
  --server sql-cloudnest \
  --resource-group rg-cloudnest-webapp \
  --service-objective GP_Gen5_4 \
  --backup-storage-redundancy Geo
```

**📖 Azure SQL Tiers**

- **Basic/Standard** — DTU-based; simple; good for dev/test; predictable pricing
- **General Purpose (GP_Gen5_4)** — vCore-based; 4 vCores; production workloads; separates compute and storage
- **Business Critical** — built-in read replica, in-memory OLTP; highest performance
- **Serverless** — auto-pauses after idle; billed per second; perfect for dev databases
- **--backup-storage-redundancy Geo** — backups stored in a paired Azure region; survives regional disaster
- Point-in-time restore: revert to any point in the last 35 days — built-in, no extra config

#### 5.2 Configure Firewall and Run Queries

```
# Allow App Service to access SQL
az sql server firewall-rule create \
  --server sql-cloudnest \
  --resource-group rg-cloudnest-webapp \
  --name allow-azure-services \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Connect and run a query (using sqlcmd)
sqlcmd -S sql-cloudnest.database.windows.net \
  -d db-clientapp \
  -U sqladmin \
  -Q "CREATE TABLE orders (id INT, amount DECIMAL, created_at DATETIME)"
```

**📖 Firewall Best Practices**

- **0.0.0.0 → 0.0.0.0** — special Azure rule that allows all Azure services (including App Service); does NOT open to public internet
- For developer access: add your public IP: `--start-ip-address 103.x.x.x --end-ip-address 103.x.x.x`
- Best practice: use **Private Endpoint** instead of firewall rules — SQL accessible only via VNet, never over public internet
- Connection string: stored in App Service App Settings referencing Key Vault — never hardcoded
- Elastic Pool: share DTUs/vCores across multiple databases — cost-efficient when databases have varying load patterns

**Business Problem:** CloudNest's client deploys 3 times a day. Manual deployments take 45 minutes each and sometimes break prod. Azure DevOps automates the entire pipeline: code push → build → test → deploy to staging → manual gate → deploy to prod.

Azure DevOps Azure Pipelines YAML Pipelines Release Gates

#### 6.1 Azure Pipelines YAML — Build and Deploy

```
# azure-pipelines.yml — CloudNest client app pipeline
trigger:
  branches:
    include: [main, develop]

variables:
  appServiceName: cloudnest-client-site
azureSubscription: CloudNest-Azure-SC
buildConfiguration: Release
stages:
  - stage: Build
jobs:
      - job: BuildJob
pool: { vmImage: ubuntu-latest }
        steps:
          - task: NodeTool@0
inputs: { versionSpec: "20.x" }
          - script: npm install && npm run build && npm test
          - task: PublishBuildArtifacts@1
inputs: { PathtoPublish: dist, ArtifactName: drop }

  - stage: DeployStaging
dependsOn: Build
jobs:
      - deployment: DeployToStaging
environment: staging
strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
inputs:
                    azureSubscription: $(azureSubscription)
appName: cloudnest-client-staging
package: $(Pipeline.Workspace)/drop

  - stage: DeployProd
dependsOn: DeployStaging
condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
jobs:
      - deployment: DeployToProd
environment: production # manual approval gate configured here
strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
inputs:
                    azureSubscription: $(azureSubscription)
appName: $(appServiceName)
package: $(Pipeline.Workspace)/drop
```

> **trigger: branches: main, develop** — pipeline fires on every push to main or develop branches; feature branches are not deployed automatically
**stages: Build → DeployStaging → DeployProd** — three-stage pipeline; each stage depends on the previous one succeeding
**environment: production** — set up approval requirements in Azure DevOps Environments; named approvers must click Approve before prod deploy proceeds
**condition: eq(SourceBranch, refs/heads/main)** — prod deploys only from main branch; develop branch changes stop at staging
**AzureWebApp@1 task** — handles ZIP deploy to App Service; no manual az cli commands needed in the pipeline

### 🟣 Advanced Projects (7–12) — Enterprise-Grade Azure Solutions

**Goal:** Deep Azure expertise. These projects require integrating multiple Azure services, understanding enterprise patterns, and solving complex business problems. Ideal for AZ-204, AZ-305 (Solutions Architect), and DP-203 certification preparation.

**Scene — CloudNest Q3 Review | "Enterprise Client Onboarding"**

> **Meera** _Lead Cloud Engineer — CloudNest Solutions_
> 
> We just signed a BFSI client — banking, financial services, insurance. Their requirements: serverless architecture for event processing, disaster recovery with sub-2-minute RTO, WAF and SIEM for security compliance, big data analytics on transaction logs, ML-based fraud detection, and full data governance with lineage tracking. These are the six advanced projects we need to deliver in the next quarter. Each one is a skill that commands 40–60% salary premium in the market.

**Business Problem:** CloudNest's BFSI client receives 50,000 payment events per hour at peak. Running a VM-based processor is wasteful during off-peak hours. Azure Functions process events on demand — pay per execution, scale to zero when idle.

Azure Functions Event Grid Service Bus Cosmos DB Logic Apps

#### 7.1 Create Function App and HTTP-Triggered Function

```
# Create a Function App
az functionapp create \
  --name func-cloudnest-payments \
  --resource-group rg-cloudnest-webapp \
  --storage-account cloudneststorage \
  --consumption-plan-location centralindia \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4
```

**📖 Consumption Plan vs Premium Plan**

- **Consumption Plan** — serverless; scales from 0 to thousands of instances automatically; billed per execution (first 1 million free/month)
- **Premium Plan** — pre-warmed instances; no cold start; VNet integration; ideal for production financial APIs
- **Dedicated (App Service)** — always-on; predictable cost; use when function runs more than 20 hours/day
- Cold start: first invocation after idle may take 1–3 seconds on Consumption Plan; Premium eliminates this

#### 7.2 Service Bus Triggered Function — Payment Processor

```python
# function_app.py — Service Bus trigger
import azure.functions as func
import logging
import json

app = func.FunctionApp()

@app.service_bus_queue_trigger(
    arg_name="msg",
    queue_name="payment-events",
    connection="ServiceBusConnection"
)
def process_payment(msg: func.ServiceBusMessage):
    payload = json.loads(msg.get_body().decode('utf-8'))
    amount  = payload.get('amount')
    txn_id  = payload.get('transaction_id')
    logging.info(f"Processing: txn={txn_id}, amount={amount}")
    # Validate → Persist → Notify
    if amount > 100000:
        flag_for_review(txn_id)
```

**📖 Function Triggers**

- **Service Bus trigger** — fires when a message arrives in the queue; guaranteed delivery; messages retry on failure
- **HTTP trigger** — REST API endpoint; function runs on every HTTP request
- **Timer trigger** — cron schedule; runs at fixed intervals; ideal for nightly batch jobs
- **Blob trigger** — fires when a file is uploaded to Blob Storage; ideal for image processing, document parsing
- **Event Grid trigger** — reacts to Azure resource events (VM created, blob deleted) across any Azure service

**Business Problem:** CloudNest's BFSI client has a 2-hour RTO (Recovery Time Objective) SLA. If the primary Azure region goes down, the application must be running in the secondary region within 2 hours. Azure Site Recovery automates the failover.

Azure Site Recovery Multi-Region Architecture Recovery Plans Traffic Manager

#### 8.1 Enable VM Replication to Secondary Region

```
# Create Recovery Services Vault
az backup vault create \
  --name rsv-cloudnest-dr \
  --resource-group rg-cloudnest-dr \
  --location southindia  # secondary region
# Enable replication for the VM (via Portal or REST API)
# Site Recovery replicates: OS disk, data disks, VM config
# RPO (Recovery Point Objective) = 5 minutes
# RTO target = under 2 hours including manual approval
# Test failover — does NOT affect production
az site-recovery protected-item failover-cancel \
  --fabric-name centralindia \
  --vault-name rsv-cloudnest-dr \
  --resource-group rg-cloudnest-dr
```

**📖 RTO vs RPO**

- **RPO (Recovery Point Objective)** — how much data you can afford to lose; ASR achieves RPO of 5 minutes (replication is near-continuous)
- **RTO (Recovery Time Objective)** — how quickly you must be back online; target is under 2 hours
- **Test Failover** — spins up a copy of your VMs in an isolated network; verify the app works before a real disaster
- **Traffic Manager** — DNS-level failover; automatically routes users to the secondary region when health probe fails on primary
- Run test failover drills quarterly — a DR plan you've never tested is not a DR plan

**Business Problem:** CloudNest's BFSI client must comply with RBI and ISO 27001 security requirements. Azure Sentinel provides SIEM (Security Information and Event Management). Key Vault manages all credentials centrally.

Microsoft Sentinel Defender for Cloud Azure Key Vault Log Analytics

#### 9.1 Azure Key Vault — Centralise All Secrets

```
# Create Key Vault
az keyvault create \
  --name kv-cloudnest \
  --resource-group rg-cloudnest-webapp \
  --location centralindia \
  --enable-rbac-authorization true \
  --enable-soft-delete true \
  --retention-days 90

# Store the SQL connection string
az keyvault secret set \
  --vault-name kv-cloudnest \
  --name SqlConnectionString \
  --value "Server=sql-cloudnest.database.windows.net;..."

# App Service reads secret via Managed Identity reference
# App Setting value: @Microsoft.KeyVault(SecretUri=https://kv-cloudnest.vault.azure.net/secrets/SqlConnectionString/)
```

**📖 Key Vault — Zero Credential Sprawl**

- **--enable-rbac-authorization** — use Azure RBAC instead of legacy access policies; grant Key Vault Secrets User role to specific identities only
- **--enable-soft-delete** — deleted secrets recoverable for 90 days; prevents accidental permanent deletion
- App Service reads secrets at runtime via Managed Identity — no credential in code, config, or environment variables
- Key rotation: update the secret in Key Vault → App Service picks up the new version at next restart or via periodic refresh
- Audit trail: every get/set/delete operation logged to Log Analytics — required for compliance

#### 9.2 Microsoft Sentinel — SIEM Setup

```
# Enable Sentinel on Log Analytics Workspace
az sentinel workspace create \
  --workspace-name law-cloudnest \
  --resource-group rg-cloudnest-security \
  --location centralindia

# Connect Azure AD sign-in logs as a data source
# (Done via Portal: Sentinel → Data Connectors → Azure Active Directory)

# Example KQL analytics rule — detect brute force login
SigninLogs
| where ResultType != "0"                    // failed logins
| summarize FailCount=count() by UserPrincipalName, bin(TimeGenerated, 10m)
| where FailCount > 10                       // 10 failures in 10 min
| project TimeGenerated, UserPrincipalName, FailCount
```

**📖 Sentinel vs Defender for Cloud**

- **Defender for Cloud** — posture management; tells you what is misconfigured (open ports, unpatched VMs, missing encryption)
- **Microsoft Sentinel** — SIEM + SOAR; collects logs from all sources, detects threats with analytics rules, automates response with playbooks
- Data connectors: Azure AD, Azure Activity, Office 365, Defender, Palo Alto, Cisco — all feed logs into Sentinel
- Analytics rules fire incidents; Logic Apps playbooks auto-respond (block IP, disable user, send alert to Teams)
- BFSI compliance: Sentinel's built-in workbooks provide RBI, PCI DSS, and ISO 27001 compliance dashboards

**Business Problem:** CloudNest's BFSI client generates 500 GB of transaction logs per day. Analysts need to query months of data in seconds and build dashboards. Azure Synapse Analytics combines a data lake, SQL warehouse, and Spark all in one.

Azure Synapse Analytics Azure Data Lake Gen2 Synapse Pipelines Power BI Integration

#### 10.1 Create Synapse Workspace and Data Lake

```
# Create Data Lake Storage Gen2
az storage account create \
  --name datalakecloudnest \
  --resource-group rg-cloudnest-data \
  --kind StorageV2 \
  --hns true \  # hierarchical namespace = ADLS Gen2
--sku Standard_LRS

# Create Synapse Workspace
az synapse workspace create \
  --name synapse-cloudnest \
  --resource-group rg-cloudnest-data \
  --location centralindia \
  --storage-account datalakecloudnest \
  --file-system synapse-fs \
  --sql-admin-login-user synapseadmin \
  --sql-admin-login-password "Str0ng!Pass2026"
```

**📖 Synapse Architecture**

- **Data Lake Gen2 (--hns true)** — hierarchical namespace enables folder-level permissions and POSIX ACLs; required for Synapse to work optimally
- **Serverless SQL Pool** — query files (CSV, Parquet, JSON) directly in the data lake using T-SQL; pay per TB queried; no cluster needed
- **Dedicated SQL Pool** — traditional data warehouse; provision compute separately; for high-concurrency BI workloads
- **Apache Spark Pool** — distributed processing; ideal for data transformation and ML feature engineering on large datasets
- Synapse Link: query Cosmos DB or SQL DB transactional data analytically without ETL — near real-time analytics

#### 10.2 Ingest and Query Transaction Data

```dockerfile
-- Serverless SQL — query Parquet in Data Lake directly
SELECT
    transaction_date,
    COUNT(*)       AS total_txns,
    SUM(amount)    AS total_volume,
    AVG(amount)    AS avg_amount
FROM
    OPENROWSET(
        BULK 'https://datalakecloudnest.dfs.core.windows.net/raw/transactions/2026/**',
        FORMAT = 'PARQUET'
    ) AS txn
WHERE
    status = 'COMPLETED'
GROUP BY
    transaction_date
ORDER BY
    transaction_date DESC;
```

**📖 OPENROWSET — Zero-ETL Querying**

- **OPENROWSET** — reads files directly from Data Lake without loading into a database; query 6 months of Parquet files in seconds
- Cost: Serverless SQL billed at $5 per TB queried; use Parquet format (compressed) to minimize query cost
- External tables: wrap OPENROWSET in a CREATE EXTERNAL TABLE — analysts query it like a regular table
- Power BI Connect: Synapse serverless SQL endpoint → Power BI Desktop → publish dashboards for business users
- Synapse Pipeline: schedule daily ingestion from Azure SQL → Data Lake → transform with Spark → surface in SQL Pool

**Business Problem:** CloudNest's BFSI client wants an automated fraud detection model — trained weekly on new transaction data, deployed as a REST API endpoint that the payment app calls in real-time. Azure ML automates the entire lifecycle.

Azure Machine Learning ML Pipelines Model Registry Managed Endpoints MLflow

#### 11.1 Create ML Workspace and Compute

```
# Create Azure ML Workspace
az ml workspace create \
  --name mlw-cloudnest \
  --resource-group rg-cloudnest-ml \
  --location centralindia

# Create compute cluster for training
az ml compute create \
  --name cpu-cluster \
  --workspace-name mlw-cloudnest \
  --resource-group rg-cloudnest-ml \
  --type AmlCompute \
  --min-instances 0 \
  --max-instances 4 \
  --size Standard_DS3_v2
```

**📖 AML Workspace Components**

- **Compute Cluster** — scales from 0 to N nodes; scales to 0 when idle (no cost); scales up when a training job starts
- **Compute Instance** — personal Jupyter notebook VM; for interactive development; stop it when not working to save cost
- **Datastores** — registered connections to Blob Storage, Data Lake, SQL — referenced in training scripts by name, not hard-coded URLs
- **Environments** — Docker images with pinned library versions; reproducible training across time and team members
- **MLflow integration** — automatic experiment tracking; log metrics, params, and model artifacts without extra code

#### 11.2 Train and Deploy Fraud Detection Model

```python
# train_fraud_model.py — submitted as an AML job
from azure.ai.ml import MLClient, command
from azure.ai.ml.entities import Environment

ml_client = MLClient.from_config()

job = command(
    code="./src",
    command="python train.py --data ${{inputs.txn_data}}",
    inputs={"txn_data": Input(path="azureml:transactions-dataset:1")},
    environment="azureml:fraud-env:3",
    compute="cpu-cluster",
    experiment_name="fraud-detection-v2",
    display_name="weekly-fraud-retrain"
)
returned_job = ml_client.jobs.create_or_update(job)
print(f"Job submitted: {returned_job.name}")
```

**📖 AML Job → Model → Endpoint**

- **command job** — submits training code to the compute cluster; tracks metrics, logs, and outputs automatically
- **Model Registry** — trained model registered with version number; each weekly retrain creates a new version; rollback to any version
- **Managed Online Endpoint** — deploy registered model as a REST API; auto-scales, handles SSL, built-in monitoring
- Blue/green deployment: route 10% traffic to new model version, 90% to old; promote after validation
- Batch endpoint: score millions of records offline; results written to Blob Storage or Data Lake

**Business Problem:** CloudNest's BFSI client stores customer data across Azure SQL, Blob Storage, Synapse, and Cosmos DB. The compliance team needs to know: where is every piece of PII data, who can access it, and how did it flow from source to report?

Microsoft Purview Data Catalog Data Lineage Information Protection Azure Policy

#### 12.1 Create Purview Account and Scan Data Sources

```
# Create Microsoft Purview account
az purview account create \
  --account-name purview-cloudnest \
  --resource-group rg-cloudnest-governance \
  --location centralindia \
  --managed-resource-group-name rg-purview-managed

# Register Azure SQL as a data source (via Portal)
# Then create a scan:
# - scan ruleset: detects PII (Aadhaar, PAN, phone, email)
# - schedule: weekly
# - results: catalog all tables, columns, data types
# - sensitivity labels applied automatically to PII columns
```

**📖 What Purview Provides**

- **Data Catalog** — searchable inventory of every dataset across all registered sources; business users find data without asking IT
- **Automated classification** — scans detect PII (Aadhaar number, PAN card, phone, email) and apply sensitivity labels automatically
- **Data Lineage** — visual graph showing how data flows: SQL DB → Synapse Pipeline → Data Lake → Power BI report; trace where any value came from
- **Data Map** — live map of all data assets, their owners, classifications, and access controls
- Compliance: Purview's data estate insights show percentage of PII assets with proper classification — required for RBI and GDPR audits

#### 12.2 Azure Policy — Enforce Governance at Scale

```
# Assign built-in policy: require HTTPS on App Service
az policy assignment create \
  --name require-https-appservice \
  --policy a4af4a39-4135-47fb-b175-47fbdf85311d \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-cloudnest-webapp

# Assign policy: require SQL TDE (transparent data encryption)
az policy assignment create \
  --name require-sql-tde \
  --policy 17k78e20-9358-41c9-923c-fb736d382a12 \
  --scope /subscriptions/<sub-id>
```

**📖 Azure Policy — Guardrails at Scale**

- **Policy assignment** — attaches a rule to a scope (subscription, resource group, or management group); all resources in that scope must comply
- **Audit effect** — flags non-compliant resources without blocking; reports compliance percentage in the dashboard
- **Deny effect** — blocks creation of non-compliant resources; enforcement mode
- **DeployIfNotExists** — automatically remediates; deploys missing configuration (e.g., enables diagnostic settings on all Storage Accounts)
- Initiative = group of related policies; assign one initiative instead of 20 separate policies for PCI DSS compliance

### All 12 Projects — Summary Reference

Project

Level

Azure Services

Skills Developed

Cert Aligned

1. Web App Deployment

Beginner

App Service, App Service Plan

- Deploy apps without managing servers
- Configure app settings and custom domains

AZ-900

2. Blob Storage

Beginner

Storage Account, Blob Containers

- Object storage and file management
- SAS tokens and access tiers

AZ-900

3. Virtual Machine

Beginner

Azure VM, NSG, Public IP

- VM sizing and SSH key auth
- Remote connectivity (SSH/RDP)

AZ-900

4. Virtual Network

Intermediate

VNet, Subnets, NSG, VNet Peering

- Network segmentation and security
- Traffic rule priority and NSG design

AZ-104

5. Azure SQL Database

Intermediate

Azure SQL, SQL Server, Elastic Pool

- Managed database setup and querying
- Firewall rules and backup config

AZ-104

6. CI/CD Pipeline

Intermediate

Azure DevOps, Pipelines, Environments

- YAML pipeline authoring
- Multi-stage deploy with approval gates

AZ-400

7. Serverless Architecture

Advanced

Functions, Service Bus, Event Grid, Cosmos DB

- Event-driven architecture design
- Trigger types and consumption billing

AZ-204

8. Disaster Recovery

Advanced

Site Recovery, Traffic Manager, Recovery Plans

- RPO/RTO planning and failover testing
- Multi-region HA architecture

AZ-305

9. Advanced Security

Advanced

Sentinel, Defender, Key Vault, Log Analytics

- SIEM setup and threat detection rules
- Centralised secrets management

SC-200

10. Big Data with Synapse

Advanced

Synapse Analytics, Data Lake Gen2, Power BI

- Data lake design and Parquet querying
- Serverless SQL and BI integration

DP-203

11. ML Pipeline

Advanced

Azure ML, MLflow, Managed Endpoints

- End-to-end ML pipeline automation
- Model versioning and deployment

DP-100

12. Data Governance

Advanced

Purview, Data Catalog, Azure Policy

- PII classification and data lineage
- Policy-driven compliance enforcement

DP-900 / SC-400

##### Tips for All Azure Projects — CloudNest Engineering Rules

- **Start small, build up:** complete Project 1 before jumping to Project 7; each project builds on concepts from previous ones; skipping ahead creates knowledge gaps that cause failures later
- **Use resource groups as project containers:** create one resource group per project; deleting the resource group removes everything cleanly when you're done — no orphaned resources accumulating charges
- **Always use the free tier for learning:** App Service F1, Azure SQL Serverless, Functions Consumption Plan — most Azure services have a free tier sufficient for learning projects
- **Document in GitHub:** create a README for every project with architecture diagram, services used, what you learned, and cost incurred — this becomes your portfolio; future employers look at GitHub before your resume
- **Use Azure Cost Alerts:** set a budget alert at ₹1,000/month; you'll get an email before you accidentally run up a large bill from a forgotten VM or SQL database
- **Use Managed Identity everywhere:** never store connection strings or passwords in code or environment variables; use Managed Identity + Key Vault references from Projects 3 onwards
- **Join the Azure community:** Microsoft Tech Community (techcommunity.microsoft.com) and the Azure Discord server — ask questions, share your GitHub projects, get feedback from practitioners
- **Pair projects with certifications:** Projects 1–3 align with AZ-900; Projects 4–6 with AZ-104; Projects 7–12 with AZ-204, AZ-305, DP-203, DP-100; every project makes the corresponding exam easier

##### ⚠️ Common Mistakes — CloudNest Project Lessons Learned

- **Leaving VMs running overnight** — Standard_D4s_v3 costs ~₹10,000/month; deallocate after every practice session; set auto-shutdown in VM settings
- **Using App Service F1 tier for performance testing** — F1 is shared compute with CPU quotas; performance results are unreliable; use B1 for any benchmarking
- **No soft-delete on Key Vault** — accidental key deletion with no soft-delete enabled is permanent; enable soft-delete and purge protection before storing any secrets
- **Public endpoint on Azure SQL in production** — always use Private Endpoint for production databases; public endpoint with firewall rules is acceptable only for learning projects
- **Wrong redundancy on Storage Account** — LRS loses data if the datacenter has a fire; use GRS for any data you cannot afford to lose
- **Deploying ML model without monitoring** — model performance degrades as real-world data drifts from training data; configure Azure ML data drift monitoring from day one

##### 🏋️ Portfolio Extension Challenges — Go Beyond the 12 Projects

1. **Project 1 Extension — Add staging slots:** Azure App Service deployment slots let you deploy to a staging slot and then swap it to production with zero downtime. Deploy your app to a staging slot, verify it works, and swap. If something breaks, swap back in 30 seconds. Document the slot URL, the swap process, and the auto-swap configuration in your GitHub README.
2. **Project 4 Extension — Set up Azure Bastion:** Instead of exposing a VM with a public IP, deploy Azure Bastion — a managed jump host service that provides SSH/RDP via the Azure Portal browser, no public IP on the VM needed. Delete the public IP from your VM-jumpbox and access it only via Bastion. This is the production-standard way to manage VMs securely.
3. **Project 7 Extension — Add Durable Functions:** Durable Functions enable stateful orchestration — coordinate a sequence of Function calls with state maintained between steps. Build a payment processing workflow: validate → reserve stock → charge card → send confirmation email. Each step is a separate Function; Durable Functions orchestrate them with retry logic and compensation actions on failure.
4. **Projects 10+11 Combined — Real-time fraud detection pipeline:** Connect Synapse Analytics (Project 10) with Azure ML (Project 11). Build a Synapse Pipeline that: (a) reads new transactions from the Data Lake every hour, (b) calls the Azure ML batch endpoint for fraud scoring, (c) writes flagged transactions back to Azure SQL for the compliance team. This end-to-end pipeline is a portfolio centrepiece for data engineering and ML engineering roles.
5. **All Projects — Add Bicep IaC:** Rewrite all 12 project deployments as Azure Bicep templates (Infrastructure as Code). Commit to a GitHub repo. Add a GitHub Actions workflow that runs `az deployment group create --template-file main.bicep` on push. Now every project is reproducible in any Azure subscription with one command. This demonstrates DevOps maturity and is specifically tested in AZ-400 and AZ-305 exams.

**Quiz: ❓ Interview Question: A client's web app on Azure App Service cannot connect to Azure SQL Database — connection times out. The SQL Server firewall shows "Allow Azure Services" is enabled. What else could be wrong?**

- A) App Service needs a managed identity to connect to SQL regardless of firewall settings
- B) "Allow Azure Services" only permits Azure datacentre IPs in general; if the App Service and SQL are in different regions or the VNet integration is misconfigured, the connection could still be blocked by NSG rules on the SQL subnet
- C) The App Service plan tier is too low to make outbound connections
- D) Azure SQL only allows connections from VMs, not App Service

> **Answer/explanation:** ✅ **Answer: B.** "Allow Azure Services" is often misunderstood — it permits any Azure-hosted resource's IP, not just resources in your own subscription or VNet. The actual troubleshooting path: (1) check App Service Outbound IPs in the portal and add them to the SQL firewall explicitly, (2) if VNet integration is enabled, check NSG rules on the SQL subnet, (3) use App Service Diagnose and Solve Problems → Network → check outbound connectivity to the SQL server's FQDN and port 1433. In production, always use Private Endpoint instead of firewall rules — it eliminates this class of problem entirely.

##### Common Questions — Azure Projects at CloudNest

**Q: Q: In what order should I do these 12 projects if I'm completely new to Azure?**

A: Week 1: Projects 1, 2, 3 (Beginner) — get the Azure Portal and CLI familiar; understand resource groups, regions, and basic billing
Week 2–3: Projects 4, 5, 6 (Intermediate) — VNet before SQL (SQL goes inside the VNet); CI/CD last (it deploys the web app from Project 1)
Week 4–8: Projects 7–12 (Advanced) — each builds on all previous; don't skip straight to Synapse without understanding SQL first
Between each: take the corresponding AZ certification practice test — projects make the exam concepts concrete

**Q: Q: How much will completing all 12 projects cost on Azure?**

A: **Beginner (Projects 1–3):** ₹0–₹200 if you use free tiers (App Service F1, Storage LRS, Standard_B1s VM stopped when not in use)
**Intermediate (Projects 4–6):** ₹500–₹1,500 total; SQL Database Serverless is cheapest; Azure DevOps is free for 5 users
**Advanced (Projects 7–12):** ₹2,000–₹5,000 total if you clean up resources after each project; Synapse dedicated pool is the most expensive — use serverless mode for learning
Total budget: ₹3,000–₹7,000 for all 12 projects over 8 weeks — less than one month of a typical Azure certification course
Set an Azure budget alert at ₹1,000 — you'll know immediately if something is running unexpectedly

**Q: Q: Can I complete all Azure projects without coding experience?**

A: **Projects 1–6:** mostly possible via Azure Portal with minimal CLI; no programming knowledge required
**Project 7 (Azure Functions):** requires basic Python or JavaScript; even simple function code teaches core concepts
**Projects 10, 11:** SQL querying (Project 10) is learnable in 2 days; Python for ML (Project 11) requires basic Python — invest a week in Python fundamentals first
**Project 12 (Governance):** no coding required; Azure Policy uses JSON but Azure Portal provides a GUI policy designer
Recommendation: learn Python basics alongside cloud projects — cloud engineers who can write simple automation scripts earn significantly more than those who cannot

### Quick Reference — Essential Azure CLI Commands Across All Projects

Command

What It Does

Project

az login

Authenticate to Azure interactively

All

az group create --name ... --location centralindia

Create a resource group

All

az group delete --name ... --yes --no-wait

Delete all resources in a group (clean up)

All

az webapp create --name ... --plan ... --runtime NODE:20-lts

Create an App Service web app

1

az webapp config appsettings set --settings KEY=VALUE

Set environment variables on App Service

1

az storage account create --name ... --sku Standard_LRS

Create a storage account

2

az storage blob upload --file ... --container-name ...

Upload a file to Blob Storage

2

az storage blob generate-sas --permissions r --expiry ...

Generate a read-only SAS URL

2

az vm create --image Ubuntu2204 --generate-ssh-keys

Create a Linux VM with SSH keys

3

az vm deallocate --name ... --resource-group ...

Stop billing for VM compute (keep disk)

3

az network vnet create --address-prefix 10.0.0.0/16

Create a Virtual Network

4

az network nsg rule create --priority 100 --access Allow

Add an NSG allow rule

4

az sql server create --admin-user ... --admin-password ...

Create Azure SQL logical server

5

az sql db create --service-objective GP_Gen5_4

Create a General Purpose SQL database

5

az functionapp create --consumption-plan-location ... --runtime python

Create a serverless Function App

7

az keyvault create --name ... --enable-rbac-authorization true

Create a Key Vault with RBAC

9

az keyvault secret set --vault-name ... --name ... --value ...

Store a secret in Key Vault

9

az synapse workspace create --storage-account ... --file-system ...

Create Synapse Analytics workspace

10

az ml workspace create --name ... --resource-group ...

Create Azure ML workspace

11

az policy assignment create --policy ... --scope ...

Enforce a governance policy on a scope

12

az purview account create --account-name ...

Create a Microsoft Purview account

12

az resource list --resource-group ... --output table

List all resources in a resource group

All

az consumption usage list --query "[].{cost:pretaxCost}"

Check current month's spending

All

### Azure Projects Mastery Complete 🎉

You have designed and deployed all 12 CloudNest Azure projects — from a basic web app on App Service to an enterprise data governance platform with Microsoft Purview. Each project solves a real business problem and builds directly on what came before. You now have a portfolio spanning compute, storage, networking, DevOps, serverless, security, big data, machine learning, and governance — covering the breadth of the Azure certification stack.

> **Meera**
> 
> "Eight weeks ago you couldn't deploy a web app. Today you delivered 12 production-ready Azure solutions for our BFSI client — from the website to the fraud detection ML pipeline to full data governance with Purview. The client extended the contract for 2 years. That outcome came from building the skills project by project, not from reading documentation. Every project in your GitHub is a problem you solved. That is what cloud engineering is."

> **Vikram (Senior Engineer)**
> 
> "The fraud detection model caught ₹12 lakh of fraudulent transactions in its first week. The Sentinel SIEM flagged a brute-force attack on the client's admin account within 4 minutes and auto-blocked the IP via a Logic App playbook. The disaster recovery test completed in 87 minutes against a 2-hour RTO SLA. Everything you built is working in production, at scale, for a regulated financial institution. That goes on your resume. And your GitHub. Both matter."

> **Next: Advance to AKS, Bicep IaC, and Azure Landing Zones**

> - **Azure Kubernetes Service (AKS)** — containerise all 12 CloudNest workloads; deploy using Helm charts; autoscale under load; see the EngiDock AKS Mastery guide for the complete end-to-end project
> - **Azure Bicep / Terraform IaC** — rewrite all 12 projects as declarative infrastructure code; reproducible in any Azure subscription; version-controlled; the foundation of enterprise-grade cloud engineering
> - **Azure Landing Zones** — the enterprise Azure architecture framework; management groups, policy inheritance, hub-spoke networking, centralised logging — everything that governs 100+ subscriptions
> - **Azure API Management (APIM)** — expose all CloudNest microservices through a single managed gateway; rate limiting, authentication, versioning, developer portal — one layer that governs all APIs
> - **Azure OpenAI Service** — add a GPT-4-powered document intelligence layer to CloudNest; extract structured data from invoice PDFs, answer questions over the BFSI client's policy documents
> - **Azure Monitor + Grafana** — build a unified observability dashboard covering all 12 projects; metrics, logs, traces, and SLA tracking in one screen for the CloudNest operations team
