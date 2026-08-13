# Advanced Azure Mastery

> **👋 Context — Who This Guide Is For**

> - **Prerequisite:** you have completed the EngiDock Azure Projects Mastery guide (12 beginner–advanced projects) and the AKS Mastery guide — this guide builds directly on those
> - **What changes at this level:** you are no longer deploying one service at a time — you are architecting interconnected systems, enforcing governance at scale, and making decisions that affect dozens of teams and hundreds of resources
> - **The career jump:** engineers who know AKS + Bicep + Landing Zones earn 40–60% more than those who know only App Service and VMs — these are the skills that separate Cloud Engineers from Cloud Architects
> - **Company:** CloudNest Solutions — same company from the Projects guide, now 3 years later; they've grown from one BFSI client to 14 enterprise clients; infrastructure managed manually is now impossible — everything must be repeatable, governed, and automated
> - **Lead:** Meera (now Cloud Architect). New character: **Rohan** (Platform Engineering Lead)

#### Six Advanced Modules — What You Will Build

- **Module 1 — AKS with Helm:** containerise all CloudNest workloads; deploy via Helm charts; autoscale across 14 client environments
- **Module 2 — Bicep IaC:** rewrite every Azure resource as Bicep code; one command recreates an entire client environment; no more manual portal clicks
- **Module 3 — Azure Landing Zones:** the enterprise blueprint — management groups, policy inheritance, hub-spoke networking, centralised logging for all 14 clients under one governance framework
- **Module 4 — Azure API Management:** expose all CloudNest microservices through one managed gateway; rate limiting, versioning, developer portal, OAuth2
- **Module 5 — Azure OpenAI Service:** add GPT-4 document intelligence to the BFSI client platform — invoice extraction, policy Q&A, fraud narrative generation
- **Module 6 — Azure Monitor + Grafana:** unified observability dashboard for all 14 clients; metrics, logs, traces, and SLA tracking on one screen

AKS + Helm, Bicep IaC, Landing Zones, APIM, Azure OpenAI, Azure Monitor, Grafana, Management Groups, Hub-Spoke VNet, Azure Policy at Scale

#### CloudNest Advanced Architecture — All 6 Modules Together

```
CloudNest Solutions — Advanced Azure Platform (14 Enterprise Clients)
══════════════════════════════════════════════════════════════════════════

  Azure Tenant: cloudnest.onmicrosoft.com

  Management Group Hierarchy (Module 3 — Landing Zones)
  ┌─────────────────────────────────────────────────────────┐
  │  Root MG: cloudnest-root                               │
  │  ├── Platform MG: platform                             │
  │  │   ├── Sub: connectivity (Hub VNet, Firewall, DNS)   │
  │  │   ├── Sub: identity (Azure AD DS, PIM)              │
  │  │   └── Sub: management (Monitor, Sentinel, Backup)   │
  │  └── Workloads MG: landing-zones                       │
  │      ├── Corp MG: corp                                 │
  │      │   ├── Sub: cloudnest-bfsi-prod  ← client 1     │
  │      │   ├── Sub: cloudnest-retail-prod ← client 2    │
  │      │   └── Sub: cloudnest-health-prod ← client 3    │
  │      └── Online MG: online                             │
  │          └── Sub: cloudnest-saas-prod                  │
  └─────────────────────────────────────────────────────────┘

  Hub-Spoke Network (per region)
  Hub VNet (connectivity sub) ←── peered ──→ Spoke VNets (each client sub)
  [Azure Firewall]  [DNS]  [ExpressRoute]     [AKS]  [SQL]  [Functions]

  AKS Cluster (Module 1) — runs across all spokes via AGIC + Helm
  ┌────────────────────────────────────────────────────────┐
  │  Helm Release: bfsi-app    │  Helm Release: retail-app  │
  │  Namespace: bfsi-prod      │  Namespace: retail-prod    │
  │  order-api + fraud-detect  │  storefront + inventory    │
  └────────────────────────────────────────────────────────┘

  APIM Gateway (Module 4) — single entry point for all clients
  Internet → APIM → [bfsi-api] [retail-api] [health-api] [saas-api]

  Azure OpenAI (Module 5)     Azure Monitor + Grafana (Module 6)
  GPT-4 invoice extraction    All 14 client dashboards, 1 pane
```

### Module 1 — AKS with Helm Charts (Full Containerisation)

**Business Problem:** CloudNest now runs the same application stack for 14 clients. Maintaining 14 separate sets of YAML files is unmanageable — one config change requires 14 manual edits. AKS + Helm: one chart, 14 values files, one deploy command per client.

**Scene 1 — CloudNest Architecture Review | "14 Clients, 14 Manual Deployments"**

> **Meera** _Cloud Architect — CloudNest Solutions_
> 
> Every time we update the order-api, Rohan's team manually edits 14 different Deployment YAMLs — one per client. Last month they missed client 7's deployment entirely. They discovered it 3 days later when the client called. With AKS and Helm, there's one chart. We pass a values file per client. The same chart version is proven once in staging, then deployed to all 14 clients with a loop in the CI pipeline. Miss one? Impossible — the loop either runs or it doesn't.

> **Rohan** _Platform Engineering Lead — CloudNest Solutions_
> 
> And rollback. Client 11 had a breaking change last quarter. We spent 40 minutes reverting 6 YAML files manually and re-applying them in the right order. With Helm: helm rollback bfsi-client11 3. One command. 90 seconds. Back to the last good revision. I've seen this save a production incident twice already in other companies I've worked at.

#### 1.1 Provision AKS Cluster for CloudNest

```
# Create AKS cluster — 3-AZ, Azure CNI, OIDC for workload identity
az aks create \
  --resource-group rg-cloudnest-platform \
  --name aks-cloudnest-prod \
  --location centralindia \
  --kubernetes-version 1.29.2 \
  --node-count 3 \
  --node-vm-size Standard_D8s_v3 \
  --zones 1 2 3 \
  --network-plugin azure \
  --enable-managed-identity \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --enable-addons monitoring,ingress-appgw \
  --appgw-name agw-cloudnest \
  --generate-ssh-keys
# Connect kubectl
az aks get-credentials \
  --resource-group rg-cloudnest-platform \
  --name aks-cloudnest-prod
```

**📖 Enterprise AKS Flags**

- **--enable-oidc-issuer** — enables OIDC; required for Workload Identity (pod-level Azure RBAC access without static credentials)
- **--enable-workload-identity** — next-generation IRSA equivalent on AKS; pods assume Azure AD identities tied to their service account
- **--enable-addons ingress-appgw** — AGIC add-on; Application Gateway created automatically; handles TLS termination and WAF for all 14 client namespaces
- **--zones 1 2 3** — nodes spread across 3 AZs; one AZ outage leaves 2/3 capacity running
- Add a dedicated system node pool (D4s_v3) to keep CoreDNS and AGIC off the application nodes

#### 1.2 Create the CloudNest Helm Chart

```bash
# Scaffold the chart
helm create cloudnest-app

# Chart.yaml — chart identity
apiVersion: v2
name: cloudnest-app
description: CloudNest multi-tenant application chart
type: application
version: 2.1.0
appVersion: "3.4.1"
dependencies:
  - name: postgresql
version: "14.3.3"
repository: https://charts.bitnami.com/bitnami
condition: postgresql.enabled
```

**📖 One Chart, 14 Clients**

- Single `cloudnest-app` chart version deployed to all 14 clients — only the values file differs
- **version: 2.1.0** — chart version; increment when templates change
- **appVersion: 3.4.1** — Docker image tag; increment per application release
- **postgresql.enabled condition** — some clients (BFSI) run their own managed PostgreSQL; chart skips the subchart for those
- Store the chart in Azure Container Registry: `helm push cloudnest-app-2.1.0.tgz oci://cloudnestacr.azurecr.io/charts`

#### 1.3 Per-Client Values Files

```
# values-bfsi-prod.yaml — BFSI client overrides
replicaCount: 5
image:
  repository: cloudnestacr.azurecr.io/order-api
tag: "3.4.1"
ingress:
  enabled: true
host: bfsi.cloudnest.in
resources:
  requests: { cpu: 500m, memory: 512Mi }
  limits:   { cpu: 2000m, memory: 2Gi }
app:
  dbHost: bfsi-pg.postgres.database.azure.com
featureFlags:
    fraudDetection: true
auditLogging: true
postgresql:
  enabled: false # BFSI uses Azure Database for PostgreSQL
autoscaling:
  enabled: true
minReplicas: 5
maxReplicas: 20
```

**📖 Feature Flags per Client**

- BFSI client gets `fraudDetection: true` and `auditLogging: true` — these features are optional and only enabled for clients that need them
- Retail client has `fraudDetection: false`, lower resource limits, and a smaller replica count — different SLA, different cost
- The **same Docker image** runs for all clients — feature flags control behaviour at runtime via environment variables injected from the ConfigMap
- Values files are stored in Git alongside the chart — changing a client's config is a PR, reviewed, and audited

#### 1.4 Deploy to All 14 Clients — CI/CD Loop

```bash
# deploy-all-clients.sh — runs in Azure DevOps pipeline
CLIENTS=(bfsi retail health logistics saas \
         banking insurance fintech edtech \
         govtech agritech hrtech proptech mobility)

CHART_VERSION="2.1.0"
APP_TAG="3.4.1"

for CLIENT in "${CLIENTS[@]}"; do
  echo "Deploying to client: $CLIENT"
  helm upgrade --install cloudnest-${CLIENT} \
    oci://cloudnestacr.azurecr.io/charts/cloudnest-app \
    --version ${CHART_VERSION} \
    --namespace ${CLIENT}-prod \
    --create-namespace \
    -f values.yaml \
    -f values-${CLIENT}-prod.yaml \
    --set image.tag=${APP_TAG} \
    --atomic \
    --timeout 5m
  echo "✓ ${CLIENT} deployed"
done
```

**📖 --atomic — Self-Healing Deploys**

- **--atomic** — if the deploy fails (pod never becomes Ready within timeout), Helm automatically rolls back to the previous revision; the loop continues to the next client
- If client 7 fails: the loop logs the error, rolls client 7 back automatically, and deploys clients 8–14; you fix client 7 separately
- Helm release names are `cloudnest-bfsi`, `cloudnest-retail`, etc. — each release is independent with its own history
- `helm history cloudnest-bfsi -n bfsi-prod` — shows every deployment with timestamp, chart version, and status
- Emergency rollback: `helm rollback cloudnest-bfsi 4 -n bfsi-prod` — 90 seconds back to any revision

### Module 2 — Bicep Infrastructure as Code

**Business Problem:** CloudNest manually created Azure resources via the portal for 3 years. When onboarding a new client, it takes 2 engineers 3 days to click through the portal. Resources are inconsistently configured — one client's SQL has TDE enabled, another's doesn't. Bicep: define once, deploy anywhere, identical every time.

**Scene 2 — CloudNest New Client Onboarding | "3 Days to Provision, Zero Documentation"**

> **Rohan** _Platform Engineering Lead — CloudNest Solutions_
> 
> New client onboarding: 3 days of portal clicking, checking config manually, hoping nothing was missed. Last month's audit found that 4 client environments had SQL databases without Transparent Data Encryption enabled — a compliance violation. With Bicep, the SQL module always deploys with TDE. It's in the code. If the code is approved, TDE is enabled. No checkbox. No human memory required. And new client onboarding goes from 3 days to 45 minutes: az deployment sub create --template-file main.bicep --parameters client=newclient. Done.

#### 2.1 Bicep Basics — Your First Template

```
// storage.bicep — Bicep is cleaner than ARM JSON
param storageAccountName string
param location string = resourceGroup().location
param skuName string = 'Standard_LRS'
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: { name: skuName }
  kind: 'StorageV2'
properties: {
    minimumTlsVersion: 'TLS1_2'
supportsHttpsTrafficOnly: true
allowBlobPublicAccess: false
  }
}

output storageEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

**📖 Bicep vs ARM JSON vs Terraform**

- **ARM JSON** — Azure's native format; verbose, error-prone, hard to read; 80 lines for a storage account
- **Bicep** — Microsoft's DSL that compiles to ARM JSON; concise, readable, type-safe; same Bicep code above is 20 lines vs 80 ARM lines
- **Terraform** — cloud-agnostic; works on AWS, GCP, Azure; HCL syntax; requires Terraform state management
- **When to use Bicep** — Azure-only shops; native Azure integration; no state file to manage; first-class VS Code support
- `minimumTlsVersion: TLS1_2` and `allowBlobPublicAccess: false` — security hardening baked into every storage account, always, no exceptions

#### 2.2 Bicep Modules — Reusable Building Blocks

```
// modules/sql.bicep — reusable SQL module
param serverName string
param adminLogin string
@secure()
param adminPassword string
param skuTier string = 'GeneralPurpose'
resource sqlServer 'Microsoft.Sql/servers@2023-02-01-preview' = {
  name: serverName
  location: resourceGroup().location
  properties: {
    administratorLogin: adminLogin
    administratorLoginPassword: adminPassword
    minimalTlsVersion: '1.2'
  }
}

resource sqlDb 'Microsoft.Sql/servers/databases@2023-02-01-preview' = {
  parent: sqlServer
  name: 'clientdb'
location: resourceGroup().location
  sku: { name: 'GP_Gen5_4', tier: skuTier }
  properties: {
    transparentDataEncryption: { state: 'Enabled' }
  }
}
```

**📖 @secure() — Parameter Protection**

- **@secure()** decorator — marks the parameter as sensitive; Bicep/ARM never logs the value in deployment history; value passed at deploy time from Key Vault or pipeline secret
- **transparentDataEncryption: Enabled** — TDE always on; the compliance violation from the portal-click era cannot happen with this module
- **parent: sqlServer** — Bicep parent-child relationship; eliminates repetitive name concatenation and ordering problems
- Modules are called from main.bicep with `module sql 'modules/sql.bicep' = { name: ... params: { ... } }`
- Bicep registry: publish modules to ACR; teams consume them like Helm chart dependencies — versioned, shared, consistent

#### 2.3 main.bicep — Full Client Environment in One File

```
// main.bicep — one deployment creates a complete client environment
targetScope = 'subscription'
param clientName string // e.g. 'bfsi'
param environment string = 'prod'
param location string = 'centralindia'
@secure()
param sqlAdminPassword string
var rgName = 'rg-cloudnest-${clientName}-${environment}'
// Resource Group
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: rgName
  location: location
}

// Storage Account — using module
module storage 'modules/storage.bicep' = {
  name: 'storage-deploy'
scope: rg
  params: {
    storageAccountName: 'cloudnest${clientName}st'
skuName: 'Standard_GRS'
  }
}

// SQL Database — using module
module sql 'modules/sql.bicep' = {
  name: 'sql-deploy'
scope: rg
  params: {
    serverName: 'sql-cloudnest-${clientName}'
adminLogin: 'sqladmin'
adminPassword: sqlAdminPassword
  }
}

// Key Vault — using module
module kv 'modules/keyvault.bicep' = {
  name: 'kv-deploy'
scope: rg
  params: {
    keyVaultName: 'kv-cloudnest-${clientName}'
  }
}

// Output — used by CI/CD for next steps
output resourceGroupName string = rg.name
output storageEndpoint string = storage.outputs.storageEndpoint
```

> **targetScope = 'subscription'** — this template creates the resource group itself; subscription-level deployment; run with `az deployment sub create`
**var rgName = 'rg-cloudnest-${clientName}-${environment}'** — string interpolation; naming convention enforced in code, not in human memory
**scope: rg** — each module deploys into the resource group created above; dependency is implicit (Bicep resolves ordering automatically)
**output** — outputs from the deployment are available to calling scripts and subsequent pipeline steps via `az deployment sub show --query properties.outputs`
Deploy new client: `az deployment sub create --template-file main.bicep --parameters clientName=newclient sqlAdminPassword='...'` — 45 minutes vs 3 days

#### 2.4 Bicep in Azure DevOps Pipeline

```
# azure-pipelines-bicep.yml
trigger:
  paths:
    include: [infra/**]

steps:
  - task: AzureCLI@2
displayName: Validate Bicep
inputs:
      azureSubscription: CloudNest-SC
scriptType: bash
scriptLocation: inlineScript
inlineScript: |
        az bicep build --file infra/main.bicep
        az deployment sub validate \
          --location centralindia \
          --template-file infra/main.bicep \
          --parameters clientName=validate-test
  - task: AzureCLI@2
displayName: What-If (dry run)
inputs:
      inlineScript: |
        az deployment sub what-if \
          --template-file infra/main.bicep \
          --parameters @infra/params.${CLIENT}.json
```

**📖 What-If — Preview Before Apply**

- **az bicep build** — compiles Bicep to ARM JSON; catches syntax errors before deployment
- **deployment sub validate** — sends the template to Azure Resource Manager for validation; catches resource name conflicts, quota issues, invalid SKUs
- **deployment sub what-if** — shows exactly which resources will be created, modified, or deleted — like `terraform plan`; attach the what-if output to the PR for review
- Trigger on `infra/**` path — only runs when Bicep files change; not on every code push
- Add a manual approval gate before the actual deploy step — infrastructure changes require human sign-off

### Module 3 — Azure Landing Zones (Enterprise Governance)

**Business Problem:** CloudNest manages 14 client subscriptions. Each has its own policies, monitoring, networking, and security configuration — all applied manually. One subscription was exposed to the internet because no one remembered to assign the deny-public-IP policy. Landing Zones apply governance at the Management Group level — every new subscription inherits it automatically.

**Scene 3 — CloudNest Security Incident | "Client 9's VM Had a Public IP for 3 Weeks"**

> **Meera** _Cloud Architect — CloudNest Solutions_
> 
> Client 9's subscription had a VM with a public IP and no NSG for 3 weeks. A developer created it for testing and forgot to delete it. Our Sentinel alert was only configured for client 1 through 5 at the time. With Azure Landing Zones and Management Groups: the deny-public-ip policy is assigned at the Corp Management Group level. It inherits to all 14 client subscriptions automatically. A developer cannot create a public IP — the ARM API refuses it. Not a warning. A hard block. That VM would never have existed.

#### 3.1 Management Group Hierarchy

```
# Create Management Group hierarchy
az account management-group create \
  --name cloudnest-root \
  --display-name "CloudNest Root"

az account management-group create \
  --name platform \
  --display-name "Platform" \
  --parent cloudnest-root

az account management-group create \
  --name landing-zones \
  --display-name "Landing Zones" \
  --parent cloudnest-root

az account management-group create \
  --name corp \
  --display-name "Corp Clients" \
  --parent landing-zones

# Move a subscription into a management group
az account management-group subscription add \
  --name corp \
  --subscription <bfsi-sub-id>
```

**📖 Why Management Groups?**

- Azure Policy and RBAC assigned at a Management Group level **inherit to all child subscriptions automatically** — no per-subscription assignment
- **cloudnest-root** — top-level; global policies (require tags, deny public IPs) assigned here apply to every subscription CloudNest manages
- **platform** — subscriptions that host shared infrastructure (connectivity, identity, management); managed by CloudNest platform team only
- **corp** — enterprise client workloads; PCI DSS and ISO 27001 policies assigned here; all current and future corp clients inherit automatically
- New client subscription: add it to the correct MG — done; all policies, RBAC, and monitoring inherited instantly

#### 3.2 Hub-Spoke Networking

```
Azure Landing Zone — Hub-Spoke Network Topology
══════════════════════════════════════════════════════════════════════════

  Connectivity Subscription (platform MG)
  ┌───────────────────────────────────────────────────────────────────┐
  │  Hub VNet: 10.0.0.0/16 (Central India)                           │
  │  ┌──────────────┐  ┌─────────────┐  ┌────────────────────────┐   │
  │  │ Azure Firewall│  │  Azure DNS  │  │ ExpressRoute / VPN GW  │   │
  │  │ 10.0.0.0/26  │  │ 10.0.1.0/28 │  │ 10.0.2.0/27            │   │
  │  └──────────────┘  └─────────────┘  └────────────────────────┘   │
  └─────────┬─────────────────────┬──────────────────────────────────┘
            │ VNet Peering         │ VNet Peering
  ┌─────────▼───────┐    ┌────────▼───────────┐
  │  Spoke: BFSI    │    │  Spoke: Retail      │
  │  10.1.0.0/16    │    │  10.2.0.0/16        │
  │  Sub: bfsi-prod │    │  Sub: retail-prod   │
  │  [AKS] [SQL]    │    │  [AKS] [SQL]        │
  │  [Functions]    │    │  [App Service]      │
  └─────────────────┘    └─────────────────────┘

  All spoke traffic → Hub Firewall → Internet (centrally inspected)
  On-prem → ExpressRoute → Hub → Spokes (private connectivity)
  DNS: Azure Private DNS zones in Hub, shared across all spokes
```

#### 3.3 Hub-Spoke VNet Peering in Bicep

```
// spoke.bicep — creates spoke VNet and peers to hub
param hubVNetId string
param clientName string
param spokePrefix string // e.g. '10.1.0.0/16'
resource spokeVNet 'Microsoft.Network/virtualNetworks@2023-06-01' = {
  name: 'vnet-${clientName}-spoke'
location: resourceGroup().location
  properties: {
    addressSpace: { addressPrefixes: [spokePrefix] }
  }
}

// Peer spoke → hub
resource spokeToHub 'Microsoft.Network/virtualNetworks/virtualNetworkPeerings@2023-06-01' = {
  parent: spokeVNet
  name: 'peer-to-hub'
properties: {
    remoteVirtualNetwork: { id: hubVNetId }
    allowGatewayTransit:    false
useRemoteGateways:      true
allowForwardedTraffic:  true
  }
}
```

**📖 Hub-Spoke Benefits**

- **Centralised egress** — all internet traffic from all 14 client spokes exits through the single Azure Firewall in the hub; one place to apply firewall rules, one place to audit traffic
- **useRemoteGateways: true** — spokes use the ExpressRoute/VPN gateway in the hub to reach on-premises; each client doesn't need their own gateway (saves ~₹50,000/month per client)
- **allowForwardedTraffic: true** — traffic can flow from spoke → hub → spoke; needed for inter-client communication through the hub firewall
- Each spoke has a unique /16 CIDR — no overlap; allows peering between any combination of spokes if needed
- Private DNS zones (Azure Private DNS) hosted in hub and linked to all spokes — one DNS zone for `blob.core.windows.net` resolves correctly from all 14 client VNets

#### 3.4 Policy Assignments at Management Group Level

```
// policies.bicep — assigned at Corp MG scope
targetScope = 'managementGroup'
// Policy 1: Deny public IP creation
resource denyPublicIp 'Microsoft.Authorization/policyAssignments@2023-04-01' = {
  name: 'deny-public-ip'
properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/6c112d4e-5bc7-47ae-a041-ea2d9dccd749'
displayName: 'Deny public IP addresses'
enforcementMode: 'Default' // Deny effect
  }
}

// Policy 2: Require tag 'client' on all resources
resource requireClientTag 'Microsoft.Authorization/policyAssignments@2023-04-01' = {
  name: 'require-client-tag'
properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/96670d01-0a4d-4649-9c89-2d3abc0a5025'
parameters: {
      tagName: { value: 'client' }
    }
  }
}
```

**📖 Policy Inheritance Chain**

- Assigned at **Corp MG** → inherited by all child MGs → inherited by all subscriptions → applies to every resource in all 14 client subscriptions
- **Deny public IP** — hard block; the Azure Resource Manager API returns HTTP 403 when any user or automation tries to create a public IP in any corp subscription
- **Require client tag** — every resource must have a `client` tag; enables cost allocation reporting per client in Azure Cost Management
- Other CloudNest Landing Zone policies: require SQL TDE, require storage HTTPS-only, require Key Vault soft-delete, audit VMs without managed disks
- View compliance: Azure Portal → Policy → Compliance — shows percentage of compliant resources across all 14 subscriptions on one dashboard

### Module 4 — Azure API Management (APIM)

**Business Problem:** CloudNest's 14 clients each consume multiple APIs (order-api, payment-api, fraud-api, report-api). Every API team sets their own rate limits, auth methods, and versioning strategies — inconsistently. APIM: one gateway for all APIs; centrally managed rate limiting, OAuth2, versioning, and a developer portal for external partners.

#### 4.1 Create APIM Instance

```
# Create APIM (Developer tier for dev/test; Standard for prod)
az apim create \
  --name apim-cloudnest \
  --resource-group rg-cloudnest-platform \
  --location centralindia \
  --publisher-email api-team@cloudnest.in \
  --publisher-name "CloudNest Solutions" \
  --sku-name Standard \
  --sku-capacity 1

# APIM is accessible at:
# Gateway:  https://apim-cloudnest.azure-api.net
# Portal:   https://apim-cloudnest.developer.azure-api.net
# Mgmt API: https://apim-cloudnest.management.azure-api.net
```

**📖 APIM Tiers**

- **Developer** — single unit, no SLA; use for development and testing only; ₹0 cost (not for production)
- **Standard** — 99.9% SLA; VNet integration; up to 4 scale units; CloudNest production choice
- **Premium** — multi-region, 99.99% SLA, Availability Zones; for global BFSI APIs
- APIM provisioning takes ~30–45 minutes — plan ahead
- Custom domain: configure `api.cloudnest.in` instead of the default `.azure-api.net` URL

#### 4.2 Import API and Apply Policies

```
# Import order-api OpenAPI spec into APIM
az apim api import \
  --resource-group rg-cloudnest-platform \
  --service-name apim-cloudnest \
  --api-id order-api \
  --path orders \
  --specification-format OpenApi \
  --specification-url \
  https://cloudnest-client-site.azurewebsites.net/swagger.json \
  --protocols https
```

**📖 APIM as the Front Door**

- All client apps call `https://api.cloudnest.in/orders/...` — they never know the backend App Service URL
- Backend URL can change (App Service → AKS migration) without any client app change — APIM abstracts the backend
- Import from OpenAPI/Swagger, WSDL (SOAP), or define operations manually
- APIM validates requests against the OpenAPI schema — malformed requests rejected at the gateway, never reaching the backend

#### 4.3 APIM Policy — Rate Limiting and Caching

```
<!-- APIM inbound policy — applied to order-api -->
<policies>
  <inbound>
    <!-- Validate OAuth2 JWT token -->
    <validate-jwt header-name="Authorization"
      failed-validation-httpcode="401">
      <openid-config url="https://login.microsoftonline.com/...
.../v2.0/.well-known/openid-configuration"/>
    </validate-jwt>

    <!-- Rate limit: 1000 calls per minute per client -->
    <rate-limit-by-key
      calls="1000"
      renewal-period="60"
      counter-key="@(context.Subscription.Id)"/>

    <!-- Cache GET responses for 30 seconds -->
    <cache-lookup vary-by-developer="false"
      vary-by-developer-groups="false"/>
  </inbound>
  <backend><forward-request/></backend>
  <outbound>
    <cache-store duration="30"/>
  </outbound>
</policies>
```

**📖 APIM Policy Power**

- **validate-jwt** — verifies Azure AD OAuth2 tokens before the request ever reaches the backend; unauthenticated requests get 401 at the gateway
- **rate-limit-by-key by Subscription.Id** — each APIM subscription (one per client) gets its own 1000/min bucket; BFSI client's heavy usage cannot starve the Retail client
- **cache-lookup + cache-store** — GET responses cached for 30 seconds in APIM's Redis cache; reduces load on order-api backend by up to 60% for read-heavy workloads
- Other useful policies: retry (auto-retry failed backend calls), set-header (inject correlation IDs), rewrite-uri (path transformation), mock-response (return canned response without hitting backend)

### Module 5 — Azure OpenAI Service

**Business Problem:** CloudNest's BFSI client processes 800 invoice PDFs daily — each manually keyed into the system. Azure OpenAI GPT-4 Vision extracts structured data from invoice images automatically, reducing manual keying from 4 hours/day to 10 minutes of exception handling.

**Scene 5 — CloudNest BFSI Client Review | "4 Hours of Daily Data Entry, Automated"**

> **Meera** _Cloud Architect — CloudNest Solutions_
> 
> The BFSI client has 3 data entry operators spending 4 hours a day typing invoice details into the system. Same work, every day. Azure OpenAI GPT-4 Vision: upload the invoice PDF, ask it to extract vendor name, invoice number, line items, and total — get back structured JSON. The model handles handwritten invoices, scanned PDFs, and images. Accuracy: 97.3% on their invoice dataset in our pilot. The 3 operators now spend 10 minutes checking exceptions flagged by the model instead of typing 800 invoices.

#### 5.1 Create Azure OpenAI Resource

```
# Create Azure OpenAI resource (request access first at aka.ms/oai/access)
az cognitiveservices account create \
  --name aoai-cloudnest \
  --resource-group rg-cloudnest-ai \
  --kind OpenAI \
  --sku S0 \
  --location eastus  # GPT-4 available in East US
# Deploy GPT-4 model
az cognitiveservices account deployment create \
  --name aoai-cloudnest \
  --resource-group rg-cloudnest-ai \
  --deployment-name gpt4-prod \
  --model-name gpt-4 \
  --model-version "turbo-2024-04-09" \
  --model-format OpenAI \
  --sku-capacity 10 \
  --sku-name Standard
```

**📖 Azure OpenAI vs OpenAI API**

- **Azure OpenAI** — same GPT-4 models; hosted within Azure; data stays in your Azure tenant; no training on your data; compliant with GDPR, ISO 27001, PCI DSS — required for BFSI
- **openai.com API** — hosted by OpenAI; data may be used for training; not suitable for PII or financial data
- **sku-capacity: 10** — 10,000 tokens per minute quota; adjust based on invoice volume
- Private Endpoint: deploy AOAI behind a private endpoint so requests never leave the VNet — mandatory for BFSI client's PII data

#### 5.2 Invoice Extraction with GPT-4 Vision

```python
# invoice_extractor.py — called from Azure Function
from openai import AzureOpenAI
import base64, json

client = AzureOpenAI(
    azure_endpoint = "https://aoai-cloudnest.openai.azure.com/",
    api_version    = "2024-02-01"
    # auth via Managed Identity — no API key in code
)

def extract_invoice(image_bytes: bytes) -> dict:
    b64 = base64.b64encode(image_bytes).decode()
    response = client.chat.completions.create(
        model    = "gpt4-prod",
        messages = [{
            "role": "user",
            "content": [
                { "type": "image_url",
                  "image_url": { "url": f"data:image/jpeg;base64,{b64}" }},
                { "type": "text",
                  "text": "Extract: vendor_name, invoice_number, date, "
                          "line_items (array), total_amount. "
                          "Return ONLY valid JSON. No explanation." }
            ]
        }],
        response_format = { "type": "json_object" },
        max_tokens = 800
    )
    return json.loads(response.choices[0].message.content)
```

**📖 Production AI Patterns**

- **response_format: json_object** — forces GPT-4 to return valid JSON only; no prose, no markdown fences; safe to `json.loads()` directly
- **Managed Identity auth** — Azure Functions use their Managed Identity to call AOAI; no API key stored anywhere; rotate-proof
- **max_tokens: 800** — caps output length; prevents runaway token usage; invoices rarely exceed 500 tokens of structured output
- Confidence scoring: ask GPT-4 to return a `confidence` field (0–1) per extracted field; flag invoices with confidence below 0.85 for human review
- Cost: GPT-4 Turbo is ~$0.01 per 1K input tokens + $0.03 per 1K output tokens; 800 invoices/day ≈ ₹8,000/month

#### 5.3 RAG — Policy Q&A with Azure AI Search

```python
# RAG pipeline: Azure AI Search + Azure OpenAI
from azure.search.documents import SearchClient
from openai import AzureOpenAI

def answer_policy_question(question: str) -> str:
    # Step 1: retrieve relevant policy chunks
    search  = SearchClient(endpoint, index_name, credential)
    results = search.search(question, top=3, query_type="semantic",
                            semantic_configuration_name="policy-config")
    context = "\n".join([r["content"] for r in results])

    # Step 2: ask GPT-4 to answer using retrieved context
    aoai    = AzureOpenAI(azure_endpoint=endpoint, api_version="2024-02-01")
    resp    = aoai.chat.completions.create(
        model    = "gpt4-prod",
        messages = [
            { "role": "system",
              "content": "Answer using ONLY the provided policy text. "
                         "If the answer is not in the text, say 'Not found'." },
            { "role": "user",
              "content": f"Context:\n{context}\n\nQuestion: {question}" }
        ]
    )
    return resp.choices[0].message.content
```

**📖 RAG — Grounded AI Responses**

- **RAG (Retrieval Augmented Generation)** — retrieve relevant document chunks first, then ask GPT-4 to answer using only those chunks; prevents hallucination
- **Azure AI Search semantic ranking** — understands the meaning of the question, not just keyword matches; finds the right policy clause even if different words are used
- **"If not in the text, say Not found"** — critical system prompt guardrail; without it, GPT-4 invents plausible-sounding but wrong policy answers
- Use case at CloudNest: BFSI compliance team asks "what is the maximum transaction limit for a tier-2 customer under RBI circular 2024-03?" — RAG finds the exact clause, GPT-4 answers precisely

### Module 6 — Azure Monitor + Grafana (Unified Observability)

**Business Problem:** CloudNest manages 14 client environments — each with its own Application Insights, Log Analytics workspace, and custom dashboards. Checking client health requires opening 14 separate portals. Azure Monitor Workspace + Managed Grafana: one Grafana dashboard shows all 14 clients; one alert rule fires for any of them.

#### 6.1 Azure Monitor Workspace and Managed Grafana

```
# Create Azure Monitor Workspace (Prometheus-compatible)
az monitor account create \
  --name amw-cloudnest \
  --resource-group rg-cloudnest-monitoring \
  --location centralindia

# Create Managed Grafana instance
az grafana create \
  --name grafana-cloudnest \
  --resource-group rg-cloudnest-monitoring \
  --location centralindia \
  --sku-name Standard

# Link Grafana to the Monitor Workspace
az grafana update \
  --name grafana-cloudnest \
  --resource-group rg-cloudnest-monitoring \
  --grafana-major-version 10
```

**📖 Azure Managed Grafana vs Self-Hosted**

- **Azure Managed Grafana** — fully managed Grafana instance; automatic updates, HA, Azure AD SSO built-in; no VM to manage
- **Azure Monitor Workspace** — Prometheus-compatible remote write endpoint; AKS clusters send metrics here; globally queryable in Grafana
- All 14 client AKS clusters write to the same Azure Monitor Workspace; Grafana queries it with a `client` label filter — one dashboard, 14 clients selectable from a dropdown
- Access: `https://grafana-cloudnest.eus.grafana.azure.com` — staff log in with their Azure AD account; no separate Grafana user management

#### 6.2 Enable Prometheus Metrics on AKS

```
# Enable Azure Monitor Prometheus scraping on AKS cluster
az aks update \
  --name aks-cloudnest-prod \
  --resource-group rg-cloudnest-platform \
  --enable-azure-monitor-metrics \
--azure-monitor-workspace-resource-id \
  /subscriptions/.../resourceGroups/rg-cloudnest-monitoring/providers/Microsoft.Monitor/accounts/amw-cloudnest \
  --grafana-resource-id \
  /subscriptions/.../resourceGroups/rg-cloudnest-monitoring/providers/Microsoft.Dashboard/grafana/grafana-cloudnest
```

**📖 What Gets Collected**

- AKS node metrics — CPU, memory, disk, network per node
- Pod metrics — CPU/memory per container, restart count, OOMKill events
- Kubernetes API server metrics — request latency, error rate, etcd heartbeat
- Custom app metrics — any app that exposes a `/metrics` Prometheus endpoint is scraped automatically; order-api exposes `orders_processed_total`, `payment_duration_seconds`
- Azure Monitor workspace stores metrics for 18 months; Grafana visualises them with PromQL queries

#### 6.3 Multi-Client Grafana Dashboard

```
# PromQL queries for Grafana panels
# Panel 1: Pod count per client (variable: $client)
count(kube_pod_info{namespace=~"$client-prod"})
  by (namespace)

# Panel 2: CPU usage % per namespace
sum(rate(container_cpu_usage_seconds_total{
  namespace=~"$client-prod"}[5m])) by (namespace)
/ sum(kube_pod_container_resource_requests{
  resource="cpu", namespace=~"$client-prod"}) by (namespace)
* 100

# Panel 3: HTTP 5xx error rate
sum(rate(http_requests_total{
  namespace=~"$client-prod", status=~"5.."}[5m]))
/ sum(rate(http_requests_total{
  namespace=~"$client-prod"}[5m]))
* 100

# Panel 4: Order API p99 latency
histogram_quantile(0.99, sum(rate(
  http_request_duration_seconds_bucket{
  job="order-api", namespace=~"$client-prod"}[5m]))
  by (le))
```

**📖 Variable-Driven Multi-Tenant Dashboard**

- **$client Grafana variable** — dropdown in Grafana shows all 14 client names; selecting one filters all panels to that client's namespace
- **Panel 1** — at-a-glance pod health; if pod count drops, something crashed or HPA scaled down unexpectedly
- **Panel 3** — 5xx error rate; cross-panel link drills down to Log Analytics for the exact error log
- **Panel 4** — p99 latency; if the 99th percentile order-api response exceeds 2 seconds, an alert fires to the CloudNest on-call channel
- Import the AKS overview dashboard from Grafana.com (dashboard ID 15760) — pre-built, battle-tested, no query writing needed to get started

#### 6.4 Unified Alert Rule — All 14 Clients

```
# Create Prometheus alert rule for ALL client namespaces
az monitor metrics alert create \
  --name "AllClients-HighErrorRate" \
  --resource-group rg-cloudnest-monitoring \
  --scopes /subscriptions/.../resourceGroups/.../accounts/amw-cloudnest \
  --condition \
    "avg PromQL where query='sum(rate(http_requests_total{status=~\"5..\"}[5m]))/sum(rate(http_requests_total[5m]))*100' > 2" \
  --severity 2 \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action /subscriptions/.../actionGroups/cloudnest-oncall
```

**📖 One Alert Rule, 14 Clients**

- The PromQL query covers all namespaces matching `*-prod` — one alert rule monitors all 14 clients simultaneously
- Alert fires when 5xx rate exceeds 2% in any 5-minute window across any client
- Action Group sends Teams message with client namespace in the alert body — on-call knows which client is affected immediately
- Alert escalation: severity 2 (warning at 2%) → severity 1 (critical at 5%) — two thresholds, two separate action groups
- Before Grafana: CloudNest had 14 separate alert rules (one per client); changing the threshold required editing 14 rules; one alert rule now

### All 6 Modules — Summary Reference

Module

Azure Services

Business Value

Skills Developed

Cert

1. AKS + Helm

AKS, Helm, ACR, AGIC

- One chart deploys to all 14 clients
- 90-second rollback on any client

Helm templating, multi-tenant K8s, AGIC

AZ-104 / AZ-305

2. Bicep IaC

Bicep, ARM, Azure DevOps

- New client onboard: 45 min vs 3 days
- Compliance baked into code — no exceptions

IaC authoring, modules, what-if deployments

AZ-204 / AZ-305

3. Landing Zones

Management Groups, Azure Policy, Hub-Spoke VNet

- Policy auto-inherited by every new subscription
- Centralized firewall for all 14 clients

Enterprise governance, hub-spoke networking, policy at scale

AZ-305

4. APIM

API Management, Azure AD, Redis Cache

- One gateway for all APIs; one place to change auth/rate limits
- 60% backend load reduction via caching

API gateway design, OAuth2, policies, versioning

AZ-204

5. Azure OpenAI

Azure OpenAI, AI Search, Azure Functions

- 4 hours/day manual invoice entry → 10 min exception review
- Policy Q&A with grounded, accurate responses

LLM integration, RAG, GPT-4 Vision, prompt engineering

AI-102

6. Monitor + Grafana

Azure Monitor, Managed Grafana, Prometheus

- 14 client dashboards in one pane
- One alert rule monitors all clients

Prometheus/PromQL, Grafana dashboards, multi-tenant observability

AZ-104 / AZ-305

##### Advanced Azure Engineering Standards — CloudNest Platform Rules

- **Everything in code:** no resource is ever created via the Azure Portal in production; every resource has a Bicep module; Portal is for reading, not writing
- **Policy as the first line of defence:** security guardrails are Azure Policy deny effects — they prevent misconfiguration before it happens; Sentinel detects incidents after they happen; both are required
- **Helm atomic deploys always:** every `helm upgrade` in CI/CD uses `--atomic` — failed deploys auto-rollback; pipelines fail loudly rather than leaving clients in a partially upgraded state
- **One Management Group per client tier:** BFSI clients in one MG, Retail in another, Healthcare in another — different compliance requirements mean different policy sets; never put BFSI and startup clients in the same MG
- **APIM for every external API:** no backend service exposes a public endpoint directly; all external traffic goes through APIM; backend URLs are internal only
- **Azure OpenAI behind Private Endpoint:** any AOAI deployment processing PII or financial data must use a private endpoint — BFSI regulation requirement; no AOAI traffic over public internet
- **Bicep what-if before every infrastructure PR merge:** the what-if output is attached to every PR as a pipeline artifact; reviewers see exactly what resources will change before approving
- **Grafana variable-driven dashboards:** never build 14 separate dashboards for 14 clients; use Grafana variables with namespace selectors; adding client 15 requires zero dashboard changes

##### ⚠️ Advanced Azure Mistakes — CloudNest Hard-Learned Lessons

- **Bicep targetScope mismatch** — deployed a subscription-scoped template with `az deployment group create` instead of `az deployment sub create`; ARM rejected it with a cryptic error; always match the deploy command to the targetScope in the Bicep file
- **Management Group policy broke existing resources** — assigned a deny-public-IP policy to the root MG while 3 existing client VMs had public IPs; VMs stopped working for 40 minutes until policy was moved to DoNotEnforce mode, IPs removed, then re-enforced; always audit existing resources with `what-if` before assigning a deny policy at MG scope
- **Helm --atomic with too-short timeout** — set `--timeout 2m`; a legitimate database migration job (pre-install hook) took 3.5 minutes; Helm timed out, auto-rolled back, wiped the migration; set timeout to at least 2x the slowest hook runtime
- **APIM rate limit on subscription not product** — applied rate limit by `context.Subscription.Id` but different API products share the same subscription; one slow product consumed the entire quota; rate limit by `context.Product.Id` for product-level isolation
- **Azure OpenAI in wrong region** — GPT-4 Vision not available in all regions; deployed AOAI in Central India; GPT-4 Vision not there yet; had to redeploy in East US; always check model availability by region at aka.ms/oai/models before provisioning
- **Hub-Spoke peering missing allowForwardedTraffic** — set up peering but forgot `allowForwardedTraffic: true` on both sides; traffic from spoke A → hub firewall → spoke B was dropped silently; intermittent connectivity appeared as flapping, not as a clear configuration error

##### 🏋️ Hands-On Challenges — Extend the CloudNest Platform

1. **Bicep Module Registry:** publish all CloudNest Bicep modules (storage.bicep, sql.bicep, keyvault.bicep, vnet.bicep) to Azure Container Registry as a Bicep module registry. Reference them from main.bicep using the OCI path: `module sql 'br:cloudnestacr.azurecr.io/bicep/modules/sql:1.0.0' = { ... }`. Any team can consume the latest approved module version — no copying files across repos. Add versioning: patch for fixes, minor for new optional params, major for breaking changes.
2. **Helm Chart Testing with Helm Unittest:** install the helm-unittest plugin, write unit tests for the cloudnest-app chart that verify: (a) HPA is created only when autoscaling.enabled=true, (b) the Ingress host changes correctly between client values files, (c) the replica count is always at least 2 in prod values. Run tests in CI: `helm unittest cloudnest-app`. A chart with failing tests cannot be merged to main.
3. **Landing Zone Vending Machine:** build an Azure DevOps pipeline that takes a form input (client name, tier, CIDR range) and automatically runs the full Landing Zone deployment: creates the subscription, moves it to the right MG, deploys the spoke VNet via Bicep, peers to hub, deploys the AKS namespace, and sends a Teams notification to the new client's team. New client onboarding: fill a form, wait 45 minutes.
4. **APIM Monetization:** enable APIM's subscription tiers — Free (100 calls/day), Standard (10,000/day), Premium (unlimited). Configure usage-based billing via Azure API Management and Logic Apps: when a subscription exceeds 80% of its quota, trigger a Logic App that sends an upgrade email to the client. Track monthly API call counts per client in a Power BI report connected to APIM's built-in analytics.
5. **Azure OpenAI Fine-tuning:** collect 500 labelled examples of CloudNest's invoice extraction (input: invoice image, output: correct JSON). Fine-tune a GPT-3.5-turbo model on this dataset using Azure OpenAI fine-tuning (faster, cheaper than GPT-4 for structured extraction tasks on known invoice formats). Compare accuracy and cost: fine-tuned GPT-3.5 should achieve similar accuracy to base GPT-4 at 10x lower cost for CloudNest's specific invoice formats.

**Quiz: ❓ Interview Question: A new enterprise client joins CloudNest. Using Landing Zones, how does their Azure subscription automatically get the same security policies, networking, and monitoring as all existing clients — without any manual configuration?**

- A) Run the Bicep deployment for each policy and monitoring resource manually in the new subscription
- B) Copy the ARM templates from an existing client subscription and redeploy them
- C) Move the new subscription into the correct Management Group — policies, RBAC, and Azure Monitor integration defined at the MG level inherit automatically; then deploy the spoke VNet Bicep template with the client's CIDR range and it peers to hub automatically
- D) Azure subscriptions inherit policies from the directory, not management groups

> **Answer/explanation:** ✅ **Answer: C.** This is the core value proposition of Azure Landing Zones. Management Group policy assignments use inheritance — any subscription placed under a MG gets all the policies, RBAC roles, and diagnostic settings assigned at that MG and its parents automatically. The only manual step is running one Bicep deployment to create the spoke VNet with the client's IP range and peer it to the hub. Everything else — deny-public-IP, require-tags, Sentinel integration, Grafana scraping — is already in place before the first Azure resource is created in the new subscription. This is the answer that separates Cloud Architects from Cloud Engineers in an interview.

##### Advanced Q&A — CloudNest Architecture Decisions

**Q: Q: When should CloudNest use Bicep vs Terraform for IaC?**

A: **Use Bicep when:** 100% Azure workloads; team already uses Azure DevOps; want native Azure integrations (Bicep registry in ACR, what-if, parameter files); no need to manage Terraform state
**Use Terraform when:** multi-cloud (AWS + Azure + GCP); team already has Terraform expertise and state management processes; need Terraform's mature provider ecosystem
CloudNest chose Bicep — Azure-only shop, native what-if preview is better than `terraform plan` for Azure-specific resources, no state file to manage or back up
Not either/or for large enterprises: Bicep for Azure-native resources, Terraform for multi-cloud orchestration layer above it

**Q: Q: What is the difference between Azure Policy Deny, Audit, and DeployIfNotExists effects?**

A: **Deny** — ARM API returns 403; the resource cannot be created or modified to violate the policy; use for hard security requirements (no public IPs, require TLS 1.2)
**Audit** — resource is created but flagged as non-compliant in the Policy dashboard; use for reporting and gradual rollout before enforcing a Deny
**DeployIfNotExists** — if a resource is created without the required configuration (e.g., SQL without diagnostic settings), Azure Policy automatically deploys the missing config; use for remediation automation
**AuditIfNotExists** — same as Audit but only fires if a related resource is missing; e.g., audit VMs that don't have Log Analytics agent installed

**Q: Q: How does Helm manage state differently from Bicep/Terraform?**

A: **Helm state** — stored as Kubernetes Secrets in the cluster itself; every `helm install/upgrade/rollback` creates a new revision Secret; no external state file; state is intrinsic to the cluster
**Bicep/ARM state** — ARM is declarative and idempotent; no explicit state file; ARM tracks resource state in Azure Resource Manager natively; re-running the same template is safe
**Terraform state** — external state file (local or in Azure Blob); must be locked during operations; drift detection requires `terraform refresh`; requires state management strategy
Risk: if a Helm release's state Secret is accidentally deleted, Helm loses track of what it deployed; run `helm list` and back up the `sh.helm.release.v1.*` Secrets in production namespaces

### Quick Reference — Advanced Azure Commands

Command

What It Does

Module

helm upgrade --install NAME OCI_CHART --atomic --timeout 5m -f values.yaml

Idempotent Helm deploy with auto-rollback on failure

1

helm history NAME -n NAMESPACE

Show full revision history of a Helm release

1

helm rollback NAME REVISION -n NAMESPACE

Roll back to a specific revision in 90 seconds

1

helm diff upgrade NAME CHART -f values.yaml

Preview changes before applying (helm-diff plugin)

1

az bicep build --file main.bicep

Compile Bicep to ARM JSON (syntax validation)

2

az deployment sub validate --template-file main.bicep

Validate template against Azure RM without deploying

2

az deployment sub what-if --template-file main.bicep

Preview all resource changes (create/modify/delete)

2

az deployment sub create --template-file main.bicep --parameters @params.json

Deploy subscription-scoped Bicep template

2

az account management-group create --name ... --parent ...

Create a Management Group in the hierarchy

3

az account management-group subscription add --name MG --subscription SUB_ID

Move a subscription into a Management Group

3

az policy assignment create --policy POLICY_ID --scope /providers/Microsoft.Management/managementGroups/MG

Assign a policy at Management Group scope

3

az policy state summarize --management-group MG

View compliance summary across all MG subscriptions

3

az apim create --name ... --sku-name Standard

Create an APIM instance (takes ~30 min)

4

az apim api import --specification-format OpenApi --specification-url ...

Import an API from an OpenAPI/Swagger spec

4

az cognitiveservices account create --kind OpenAI --sku S0

Create an Azure OpenAI resource

5

az cognitiveservices account deployment create --model-name gpt-4

Deploy a GPT-4 model to the AOAI resource

5

az monitor account create --name amw-cloudnest

Create Azure Monitor Workspace (Prometheus endpoint)

6

az grafana create --name grafana-cloudnest --sku-name Standard

Create a Managed Grafana instance

6

az aks update --enable-azure-monitor-metrics --azure-monitor-workspace-resource-id ...

Enable Prometheus scraping from AKS to Monitor Workspace

6

az monitor metrics alert create --condition "avg PromQL where query='...' > threshold"

Create a Prometheus-based alert rule

6

### Advanced Azure Mastery Complete 🎉

You have built CloudNest's enterprise-grade Azure platform — AKS with Helm for multi-tenant Kubernetes deployments, Bicep IaC that reduces client onboarding from 3 days to 45 minutes, Azure Landing Zones with Management Groups and hub-spoke networking that governs all 14 clients automatically, API Management as the single gateway for all APIs, Azure OpenAI for invoice extraction and policy Q&A, and unified observability with Managed Grafana across all client environments. You are no longer deploying services — you are architecting platforms.

> **Meera**
> 
> "Three years ago we had one BFSI client and 2 engineers. Today we have 14 enterprise clients, 11 engineers, and the same 2 engineers from year one are now architects. What changed? Everything you built in these 6 modules. Bicep means a junior can onboard a client without senior review for every portal click. Landing Zones means the security team sleeps at night because no misconfiguration can survive a policy deny. AKS and Helm means we deploy to 14 clients in the time it used to take to deploy to one. That is what platform engineering looks like."

> **Rohan**
> 
> "The BFSI client's invoice extraction model processed 800 invoices yesterday. Zero human touches. 97.3% accuracy. The 3 data entry operators now run exception analysis and catch edge cases the model misses — more valuable work, same headcount. The Grafana dashboard showed a 5xx spike on the retail client at 2:14 AM, the alert fired, and it was auto-resolved by the HPA before anyone woke up. I got the Slack notification at 7 AM saying the incident auto-resolved. That is what we built. Infrastructure that heals itself."

> **What Comes After This — Azure Expert and Architect Track**

> - **Azure Service Mesh (Istio on AKS)** — mTLS between all CloudNest microservices; traffic splitting for canary releases (5% → 20% → 100%); distributed tracing to Application Insights; circuit breaking for payment API dependencies — zero code changes in the application
> - **Terraform + Azure — hybrid IaC** — use Bicep for Azure-native resources and Terraform for the orchestration layer; manage Terraform state in Azure Blob with state locking; integrate with Terratest for automated IaC testing in Go
> - **Azure DevBox and Dev Center** — standardised developer environments for all 14 client teams; developers spin up a pre-configured cloud dev machine in 10 minutes; no more "works on my machine" across 14 client projects
> - **Azure Arc** — extend Azure management to CloudNest's on-premises servers and Kubernetes clusters; apply the same Landing Zone policies, Azure Monitor, and Defender for Cloud to on-prem resources; hybrid cloud from one control plane
> - **FinOps with Azure Cost Management** — client-level cost allocation using the `client` tag policy from Landing Zones; monthly cost reports per client via Logic Apps; anomaly detection alerts when a client's spend exceeds ±20% of forecast
> - **AZ-305 Certification (Azure Solutions Architect Expert)** — the certification that validates everything in this guide: Landing Zones, hub-spoke networking, APIM, hybrid connectivity, disaster recovery, and security architecture — the highest Azure technical certification
