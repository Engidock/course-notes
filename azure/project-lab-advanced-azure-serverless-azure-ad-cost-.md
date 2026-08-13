# Advanced Azure: Serverless, Azure AD & Cost Optimisation

> **👋 Who This Document Is For**

> - You have completed the **Azure Foundations** module — you know Resource Groups, VNet, VM, App Service, Blob Storage, Azure SQL, Functions, AKS, Key Vault, Azure DevOps, and Azure Monitor.
> - This module covers the **enterprise layer** — identity, governance, cost control, and advanced integration services that large companies rely on every day.
> - Same company: **UrbanNest** — real estate platform in Pune. Same engineers: Priya, Sameer, Ravi. UrbanNest is now growing fast and needs enterprise-grade security, automated governance, and optimised costs.
> - Every code block is **short and focused**. Every explanation is **bullet points only**. No paragraphs in explanations.

#### What You Will Learn in This Module

> **🔄 Phase 1 — Logic Apps & Service Bus**

> Automate workflows without code using Logic Apps. Decouple microservices with Service Bus message queues and topics. Build UrbanNest's property alert and agent notification system.

> Azure's globally distributed NoSQL database. Understand partition keys, consistency levels, and change feed. Store UrbanNest's property search results and user preferences.

> Enterprise identity management. SSO for UrbanNest employees, B2C for customer login, Conditional Access policies, service principals for CI/CD pipelines.

> Enforce governance rules across all subscriptions. Write Bicep infrastructure-as-code to deploy entire environments reproducibly from a single command.

> Analyse and reduce UrbanNest's Azure bill. Reserved Instances, Spot VMs, Azure Hybrid Benefit, right-sizing, lifecycle policies, and budget alerts.

Logic Apps, Service Bus, Cosmos DB, Azure AD, Conditional Access, Azure Policy, Bicep, Cost Management

**Scene 1 — UrbanNest Leadership Review | "We're Growing Fast — Three New Problems"**

> **Sameer** _Platform Architect — UrbanNest_
> 
> Three things are breaking at scale. One: we have 40 employees all logging in with separate usernames and passwords for every tool — Jira, GitHub, Azure Portal, Salesforce. When someone leaves, we forget to disable half their accounts. Azure AD SSO and Conditional Access will fix this in one afternoon. Two: agents are supposed to get WhatsApp and email notifications when a new property matches their search — we're doing this with cron jobs in the app code. Logic Apps and Service Bus will automate this properly. Three: our Azure bill jumped from ₹52,000 to ₹1.1 lakh last month. Nobody knows where the increase came from. Cost Management and Reserved Instances will fix that.

> **Priya** _Senior Cloud Engineer — UrbanNest_
> 
> And the developers keep creating resources manually in the Portal — different regions, missing tags, no private endpoints. Two weeks ago someone created a VM in East US by mistake, left it running for 10 days, and we paid ₹8,000 for it before anyone noticed. Azure Policy stops this at the subscription level — it literally prevents resource creation that doesn't meet your rules. Combined with Bicep for IaC, no one is clicking in the Portal for production resources anymore.

### 1. Phase 1 — Logic Apps and Azure Service Bus

**Business Problem:** When a new property listing is created in UrbanNest's database, agents who have matching saved searches need to be notified via email. Currently this is hard-coded in the API controller — a mess to maintain. Logic Apps provides a visual, no-code workflow. Service Bus decouples the notification system so the API doesn't wait for emails to send.

> **Logic Apps vs Azure Functions — When to Use Which**

> - **Logic Apps** — visual, low-code workflow designer. 400+ pre-built connectors: Outlook, Teams, SAP, Salesforce, Dynamics 365, SQL, Blob, HTTP. No custom code needed for standard integrations. Best for: orchestration, approval workflows, data sync between services, notification pipelines.
> - **Azure Functions** — code-first, developer-driven. Write logic in Python, Node.js, C#. Best for: custom processing, complex business logic, transformations requiring code, short event-driven execution.
> - They work together: Logic Apps can call an Azure Function when it needs custom logic that has no pre-built connector.
> - Logic Apps is charged per action execution (~₹0.00025 per action). Functions is charged per invocation. For pure integration workflows, Logic Apps is often simpler and cheaper.

```
UrbanNest Property Alert Workflow — Logic Apps + Service Bus
==============================================================

  New property listed in Azure SQL
        │
        ▼ Trigger: SQL Connector (polls every 2 min)
  Logic App: property-alert-workflow
        │
        ├── Query: Find agents with matching saved searches
        │
        ├── Service Bus: Put message on "agent-alerts" topic
        │      │
        │      ├── Subscription: email-processor
        │      │    └── Logic App: send-email-alert (Outlook connector)
        │      │
        │      └── Subscription: teams-processor
        │           └── Logic App: post-teams-message (Teams connector)
        │
        └── Log to Application Insights

  Result: API creates listing. Service Bus decouples everything else.
  API response time: unchanged. Agents notified within 2 minutes.
```

#### 1.1 Create a Service Bus Namespace and Topic

```
# Create Service Bus namespace
az servicebus namespace create \
  --resource-group urbannest-prod-rg \
  --name urbannest-servicebus \
  --location centralindia \
  --sku Standard
# Create a topic (pub/sub messaging)
az servicebus topic create \
  --resource-group urbannest-prod-rg \
  --namespace-name urbannest-servicebus \
  --name agent-alerts
# Create a subscription (a consumer on the topic)
az servicebus topic subscription create \
  --resource-group urbannest-prod-rg \
  --namespace-name urbannest-servicebus \
  --topic-name agent-alerts \
  --name email-processor
```

**📖 Service Bus: Queue vs Topic**

- **Queue** — point-to-point messaging. One sender, one receiver. Each message is processed by exactly one consumer. Use for work distribution (e.g. image processing jobs).
- **Topic** — publish-subscribe messaging. One sender, multiple receivers. Each subscriber gets its own copy of the message. Use for event broadcasting (e.g. one property listing event → email service AND Teams service both receive it).
- **Subscription** — a named consumer on a topic. Each subscription maintains its own copy of the messages independently.
- **Standard SKU** — topics & subscriptions supported (Basic SKU only has queues). Use Standard for pub-sub patterns. Premium for larger messages & VNet integration.
- Messages persist up to **14 days** if not consumed. Dead-letter queue catches messages that fail processing after max retry count.

#### 1.2 Send a Message to Service Bus (Python)

```python
# In UrbanNest's API — after creating a listing
from azure.servicebus import ServiceBusClient, ServiceBusMessage
import json

conn_str = "Endpoint=sb://urbannest-servicebus..."
with ServiceBusClient.from_connection_string(conn_str) as client:
    with client.get_topic_sender(topic_name="agent-alerts") as sender:
        msg = ServiceBusMessage(json.dumps({
            "listingId": "PROP-789",
            "city":      "Pune",
            "price":     8500000,
            "bhk":       3
        }))
        sender.send_messages(msg)
```

**📖 Publishing to Service Bus**

- The API publishes one message to the `agent-alerts` topic and returns immediately. It does not wait for emails to be sent.
- **Decoupling** — the API's response time is unaffected by how long email sending takes. If the email service is down, messages queue up in Service Bus and are delivered when it recovers.
- Use **Managed Identity** instead of connection strings in production: `ServiceBusClient(fully_qualified_namespace, DefaultAzureCredential())` — no credentials in code.
- **Message properties** — set `subject`, `correlation_id`, and custom properties for routing and filtering within subscriptions.

#### 1.3 Logic App — Automated Workflow

```
# Create a Logic App resource
az logic workflow create \
  --resource-group urbannest-prod-rg \
  --location centralindia \
  --name send-email-alert \
  --definition @workflow-definition.json
# Test the Logic App manually
az logic workflow trigger run \
  --resource-group urbannest-prod-rg \
  --name send-email-alert \
  --trigger-name ServiceBusTrigger
```

**📖 Logic Apps Workflow**

- **workflow-definition.json** — the JSON definition of the Logic App workflow. In practice, you design it in the Logic Apps Designer (visual drag-and-drop) and export the JSON.
- The designer has: triggers (what starts the workflow), actions (what it does), conditions (if/else branching), loops (for each item in a list).
- The **Service Bus trigger** fires every time a message arrives on the `email-processor` subscription — no polling, near real-time.
- Common workflow pattern: Trigger → Parse JSON → For Each matched agent → Send Email via Outlook 365 connector → Log result.
- **Run history** in the Portal shows every execution: success/failure, inputs, outputs, and duration per action — invaluable for debugging.

Microsoft Azure

Logic Apps › send-email-alert › Run History

✓ ServiceBusTrigger 0.1s — Message: PROP-789, 3BHK Pune ₹85L

✓ Parse_JSON 0.0s

✓ Query_Matching_Agents (SQL) 0.4s — 7 agents found

✓ For_Each_Agent → Send_Email (Outlook 365) 1.8s — 7 emails sent

Total executions today 143 | Succeeded: 143 | Failed: 0 | Cost: ₹0.04

### 2. Phase 2 — Cosmos DB: Globally Distributed NoSQL

**Business Problem:** UrbanNest's property search results need to be cached for sub-100ms responses. Azure SQL is too slow for search queries across 500,000 listings with filters. Cosmos DB stores the search index and user saved searches with single-digit millisecond reads — regardless of how much data or how many users.

> **Cosmos DB in Plain English**

> - Cosmos DB is Azure's globally distributed NoSQL database. You can replicate data to any Azure region with one click — users in Mumbai and Delhi both read from their nearest region.
> - Supported APIs: **NoSQL (JSON documents)**, MongoDB, Cassandra, Gremlin (graph), Table. Choose the API based on your team's familiarity. NoSQL is the native Azure option.
> - Throughput is measured in **RU/s (Request Units per second)**. A read of a 1KB item costs 1 RU. A write costs 5 RU. You provision or auto-scale RU/s.
> - Unlike Azure SQL, Cosmos DB has **no schema** — each document can have different fields. Add a new property attribute tomorrow without any migration.
> - Comparable to AWS DynamoDB — same NoSQL key-value/document model, but Cosmos DB supports multiple APIs and more consistency level options.

#### 2.1 Create a Cosmos DB Account and Container

```
# Create Cosmos DB account (NoSQL API)
az cosmosdb create \
  --resource-group urbannest-prod-rg \
  --name urbannest-cosmos \
  --locations regionName=centralindia failoverPriority=0 isZoneRedundant=true \
  --default-consistency-level Session
# Create database and container
az cosmosdb sql database create \
  --resource-group urbannest-prod-rg \
  --account-name urbannest-cosmos \
  --name SearchDB
az cosmosdb sql container create \
  --resource-group urbannest-prod-rg \
  --account-name urbannest-cosmos \
  --database-name SearchDB \
  --name Listings \
  --partition-key-path /city \
  --throughput 400
```

**📖 Key Cosmos DB Concepts**

- **--partition-key-path /city** — Cosmos DB distributes data across partitions by this key. All Pune listings go to one partition, Delhi to another. Choose a high-cardinality key with uniform access.
- **--throughput 400** — minimum 400 RU/s. At ₹5.5 per 100 RU/s per hour this is ~₹475/month. Use **autoscale** instead (`--max-throughput 4000`) to automatically scale 400–4000 RU/s based on demand.
- **--default-consistency-level Session** — your read always sees your own writes (e.g. agent sees their own saved searches). 5 levels from Strong (slowest, fully consistent) to Eventual (fastest, may see stale data).
- **isZoneRedundant=true** — replicate across AZs in the region. Survives AZ failures with no data loss.

#### 2.2 Read and Write Documents (Python)

```python
from azure.cosmos import CosmosClient

client = CosmosClient.from_connection_string(conn_str)
container = (client
  .get_database_client("SearchDB")
  .get_container_client("Listings"))

# Write a document
container.upsert_item({
  "id":    "PROP-789",
  "city":  "Pune",
  "price": 8500000,
  "bhk":   3,
  "area":  "Baner",
  "ttl":   2592000 # auto-delete after 30 days
})

# Read all 3BHK listings in Pune
items = list(container.query_items(
  query="SELECT * FROM c WHERE c.city='Pune' AND c.bhk=3",
  partition_key="Pune"
))
```

**📖 Cosmos DB Operations**

- **upsert_item** — insert if not exists, update if exists. Every document needs a mandatory `id` field (string, unique within partition).
- **query_items with partition_key** — scoped query: only searches within the "Pune" partition. Much faster and cheaper than a cross-partition query.
- **ttl (Time To Live)** — set a TTL in seconds on the document level. Cosmos DB auto-deletes expired documents for free. Great for caching search results.
- **Change Feed** — a stream of all inserts and updates to a container. Listen with an Azure Function to trigger downstream processing when new listings arrive (e.g. push to search index, notify agents).

### 3. Phase 3 — Azure AD / Microsoft Entra ID

**Business Problem:** UrbanNest has 40 employees using separate logins for GitHub, Azure Portal, Jira, and Salesforce. When an employee leaves, their accounts must be individually disabled across 8 tools — a compliance risk. Azure AD (now called Microsoft Entra ID) is the single identity provider for all UrbanNest's tools. One account, one disable, all access revoked.

> **Azure AD — Three Use Cases You Must Know**

> - **Employee SSO (Single Sign-On)** — employees log in once with their UrbanNest Microsoft account. Azure AD passes identity to all connected apps (GitHub, Azure Portal, Jira, Salesforce). Leave the company → one account disable → all access revoked simultaneously.
> - **Azure AD B2C (Business-to-Consumer)** — customer-facing identity for UrbanNest's buyer portal. Customers sign up and log in with Google, Facebook, or email. You control the login experience. Completely separate from employee accounts.
> - **Service Principal (for apps/pipelines)** — an identity for automated systems (Azure DevOps pipeline, Terraform, GitHub Actions) to authenticate to Azure. Like an IAM service account. Replaces humans in automated workflows.

#### 3.1 Create Users and Groups

```
# Create an Azure AD user
az ad user create \
  --display-name "Ravi Shankar" \
  --user-principal-name ravi@urbannest.in \
  --password TempPass@2026! \
  --force-change-password-next-sign-in
# Create a security group for dev team
az ad group create \
  --display-name "UrbanNest-DevTeam" \
  --mail-nickname urbannest-devteam
# Add user to the group
az ad group member add \
  --group UrbanNest-DevTeam \
  --member-id $(az ad user show --id ravi@urbannest.in --query id -o tsv)
```

**📖 Users, Groups and RBAC**

- Azure AD users are the identities your employees use to log in to everything — the Portal, Microsoft 365, GitHub (SSO), Jira, etc.
- **--force-change-password-next-sign-in** — user must set their own password at first login. Always set this for new users.
- **Groups** — assign RBAC roles to groups, not individual users. Add/remove a person from the group to grant/revoke access across all resources at once.
- Assign the **UrbanNest-DevTeam group** the Contributor role on the dev Resource Group. Every new developer added to the group automatically gets the right access — no manual per-user role assignments.

#### 3.2 Conditional Access — Zero Trust Security

```
# Create Conditional Access policy via CLI
# (requires Azure AD Premium P1 or P2 licence)
# Require MFA for all Azure Portal access
az rest --method POST \
  --uri "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --body '{
    "displayName": "Require MFA for Azure Portal",
    "state": "enabled",
    "conditions": {
      "applications": {"includeApplications": ["797f4846-ba00-4fd7-ba43-dac1f8f63013"]},
      "users": {"includeGroups": ["GROUP_OBJECT_ID"]}
    },
    "grantControls": {
      "operator": "OR",
      "builtInControls": ["mfa"]
    }
  }'
```

**📖 Conditional Access**

- **Conditional Access** — "If user does X under condition Y, then require Z." The most powerful security feature in Azure AD.
- Application ID `797f4846...` is the Microsoft Azure Management app — targeting this requires MFA for Azure Portal specifically.
- **Common policies:** Require MFA from outside India, block access from risky sign-in locations, require compliant device (enrolled in Intune), block legacy authentication protocols (SMTP, IMAP).
- Conditional Access requires **Azure AD Premium P1** licence (~₹500/user/month). Free tier has no Conditional Access.
- Always test with a **named exclusion account** (a break-glass admin account excluded from CA policies) so you don't accidentally lock yourself out.

#### 3.3 Service Principal — Identity for Automation

```
# Create service principal for Azure DevOps pipeline
az ad sp create-for-rbac \
  --name "urbannest-devops-sp" \
  --role Contributor \
  --scopes "/subscriptions/SUB_ID/resourceGroups/urbannest-prod-rg" \
  --json-auth
# Output: clientId, clientSecret, tenantId, subscriptionId
# Store this JSON in Azure DevOps as a Service Connection
# NEVER commit this JSON to Git
# Verify service principal's permissions
az role assignment list \
  --assignee urbannest-devops-sp \
  --output table
```

**📖 Service Principals**

- **Service Principal (SP)** — a non-human Azure AD identity for applications, automation scripts, and pipelines. Equivalent to AWS IAM service accounts.
- **--json-auth** — outputs JSON with all credentials needed. Save this as a DevOps Service Connection or Key Vault secret.
- The SP gets Contributor role only on the prod Resource Group — not the entire subscription. Least privilege: can only manage resources in that group.
- Client secrets expire. Set a calendar reminder to rotate secrets every 6–12 months. Or use **Federated Identity Credentials** (OIDC) with GitHub Actions — no client secret at all, much more secure.
- **Managed Identity** is always preferred over Service Principals when the app runs on Azure (VM, App Service, AKS) — no credentials to manage or rotate.

#### 3.4 Azure AD B2C — Customer Identity

```
# Create an Azure AD B2C tenant
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/SUB/resourceGroups/urbannest-prod-rg/providers/Microsoft.AzureActiveDirectory/b2cDirectories/urbannest" \
  --body '{
    "location": "unitedstates",
    "sku": { "name": "PremiumP1", "tier": "A0" },
    "properties": {
      "createTenantProperties": {
        "displayName": "UrbanNest Buyers",
        "countryCode":  "IN"
      }
    }
  }'
```

**📖 Azure AD B2C — Customer Identities**

- **B2C is completely separate** from your employee Azure AD tenant. Customers never share a tenant with your internal users.
- Supports **social identity providers**: Google, Facebook, Apple, Microsoft Account. Customers sign up with their existing social account — no new password to remember.
- Provides **User Flows** (pre-built policies): sign-up/sign-in, password reset, profile editing. Fully customisable UI with HTML/CSS templates to match UrbanNest's branding.
- Issues **JWT tokens** your API validates. Integrate with any OAuth 2.0/OIDC-compatible backend in any language.
- Free for first 50,000 MAU (Monthly Active Users). Scales to millions. Perfect for a growing real estate buyer portal.

### 4. Phase 4 — Azure Policy and Bicep Infrastructure as Code

**Business Problem:** Developers are creating VMs in random regions, without tags, without private endpoints. A VM was created in East US costing ₹8,000 before anyone noticed. Azure Policy prevents rule violations before they happen. Bicep replaces Portal clicking with version-controlled, reproducible infrastructure code.

**Scene 3 — UrbanNest Governance Crisis | "Nobody Is Following the Rules"**

> **Sameer** _Platform Architect — UrbanNest_
> 
> Ravi, last week I ran a report on our Azure resources. We have VMs in East US, West Europe, and Australia — none of which we approved. Storage accounts with no tags. SQL databases with public endpoints open. We've been saying "follow the architecture standards" in Confluence pages. Nobody reads them. Azure Policy is the technical enforcement layer. You write a policy that says "Deny any resource creation outside centralindia and southindia" — and Azure literally refuses to create the resource, even if the developer has Contributor access. Policy trumps RBAC for governance.

#### 4.1 Azure Policy — Enforce Governance Rules

```
# Assign built-in: allowed locations policy
az policy assignment create \
  --name allowed-locations \
  --policy e56962a6-4747-49cd-b67b-bf8b01975c4f \
  --scope "/subscriptions/SUB_ID" \
  --params '{"listOfAllowedLocations": {"value": ["centralindia","southindia"]}}'
# Assign built-in: require tags on resources
az policy assignment create \
  --name require-env-tag \
  --policy 96670d01-0a4d-4649-9c89-2d3abc0a5025 \
  --scope "/subscriptions/SUB_ID" \
  --params '{"tagName": {"value": "Environment"}}'
```

**📖 Azure Policy**

- **Policy assignment** — applies a policy rule to a scope (management group, subscription, or resource group). Built-in policies cover hundreds of common governance needs.
- **Effects:** Deny (block resource creation), Audit (allow but log to compliance report), Modify (auto-add tags), DeployIfNotExists (deploy a companion resource automatically).
- **Allowed locations policy (e56962a6...)** — blocks any resource creation outside the specified regions. Works even for users with Owner role — Policy overrides RBAC for Deny effects.
- **Require tag policy** — blocks resource creation if the "Environment" tag is missing. Forces developers to specify environment on every resource.
- Use **Policy Initiatives** (groups of policies) to apply 10–20 related policies at once (e.g. CIS Azure Benchmark initiative).

#### 4.2 Bicep — Infrastructure as Code

```
// main.bicep — Deploy the core UrbanNest resources
param location string = 'centralindia'
param env      string = 'prod'
resource rg 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name:     'urbannest-${env}-rg'
  location: location
  tags:     { Environment: env, Project: 'UrbanNest' }
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2022-09-01' = {
  name:       'urbannest${env}storage'
  location:   location
  sku:        { name: 'Standard_LRS' }
  kind:       'StorageV2'
  properties: { supportsHttpsTrafficOnly: true }
}
```

**📖 Bicep — Azure-Native IaC**

- **Bicep** — Microsoft's native Infrastructure as Code language. Simpler than ARM templates (ARM is JSON, Bicep is cleaner DSL). Compiles to ARM templates under the hood.
- **param** — declares a parameter with a type and optional default value. Pass different values at deployment time for dev vs prod.
- **resource** — declares an Azure resource. Type format: `'ResourceProvider/resourceType@apiVersion'`.
- **String interpolation**: `'urbannest-${env}-rg'` produces `urbannest-prod-rg` or `urbannest-dev-rg` based on the param.
- Deploy with: `az deployment sub create --template-file main.bicep --parameters env=prod location=centralindia`

```
// Bicep module — reusable App Service definition
// modules/appservice.bicep
param appName  string
param location string
param planId   string
resource webapp 'Microsoft.Web/sites@2022-09-01' = {
  name:     appName
  location: location
  properties: {
    serverFarmId: planId
    httpsOnly:    true
    siteConfig:   { minTlsVersion: '1.2' }
  }
  identity: { type: 'SystemAssigned' }
}

output principalId string = webapp.identity.principalId
```

**📖 Bicep Modules and Outputs**

- **Modules** — reusable Bicep files. The main.bicep calls a module like a function: `module frontend 'modules/appservice.bicep' = { params: {...} }`
- **identity: SystemAssigned** — enables Managed Identity on the App Service directly in the Bicep code. No manual Portal step needed.
- **output** — exposes values from the module so the parent can use them (e.g. the principalId from the web app's identity can be passed to a Key Vault access policy module).
- **httpsOnly: true + minTlsVersion: '1.2'** — security best practices enforced in code, not configured manually after deployment.
- **Bicep vs Terraform:** Bicep is Azure-only but tightly integrated. Terraform is multi-cloud but requires separate provider setup. For Azure-only shops, Bicep is simpler. For multi-cloud, use Terraform.

### 5. Phase 5 — Cost Management and Optimisation

**Business Problem:** UrbanNest's Azure bill jumped from ₹52,000 to ₹1.1 lakh in one month. Nobody knows why. Azure Cost Management provides the analysis tools. Reserved Instances, Spot VMs, right-sizing, and lifecycle policies reduce the bill by 40–70% without removing any infrastructure.

**Scene 4 — UrbanNest Finance Review | "Find the ₹58,000 Jump"**

> **Priya** _Senior Cloud Engineer_
> 
> Ravi, I pulled the Cost Management breakdown. Three culprits: one, a developer left a VM running for 30 days during load testing — ₹22,000. Two, we're paying on-demand pricing for our production VMs — if we switch to 1-year Reserved Instances, that's 40% savings immediately. Three, our Blob Storage is all Hot tier — 80% of property images older than 6 months are never accessed. Moving them to Cool tier saves ₹8,000 per month. None of this requires any code changes. Pure configuration changes with immediate savings.

#### 5.1 Analyse Costs with Azure Cost Management

```
# Get cost breakdown by resource group (last 30 days)
az costmanagement query \
  --scope "/subscriptions/SUB_ID" \
  --type Usage \
  --timeframe MonthToDate \
  --dataset-granularity Daily \
  --dataset-grouping name=ResourceGroupName type=Dimension
# Create a budget with email alert at 80%
az consumption budget create \
  --budget-name monthly-budget \
  --amount 75000 \
  --time-grain Monthly \
  --time-period-start 2026-04-01 \
  --time-period-end 2027-03-31 \
  --category Cost
```

**📖 Cost Analysis Tools**

- **az costmanagement query** — programmatically pull cost data grouped by resource group, service, resource, or tag. Use this in scripts to detect runaway costs automatically.
- **Budget alerts** — set at 80% and 100% of monthly budget. Email or webhook action when threshold is crossed. Azure can also **automatically shut down VMs** when budget is exceeded (cost alert + action group + automation runbook).
- **Cost Analysis in Portal** — interactive charts showing cost by service, resource, location, and tags. Pivot table-like filtering. Use to find the top 5 most expensive resources.
- **Tags are essential for cost attribution** — without the Environment and Project tags, you can't tell which team caused the bill spike.

#### 5.2 Reserved Instances — Pay Upfront, Save 40–72%

```
# Check Azure Advisor recommendations first
az advisor recommendation list \
  --category Cost \
  --output table
# See reserved instance pricing (Portal is easier but CLI works)
az reservations reservation-order calculate \
  --sku Standard_B2s \
  --location centralindia \
  --reserved-resource-type VirtualMachines \
  --billing-scope /subscriptions/SUB_ID \
  --term P1Y \
  --billing-plan Monthly
```

**📖 Reserved Instances**

- **Reserved Instance (RI)** — pay for 1 or 3 years upfront in exchange for a massive discount. The VM keeps running on-demand but you pay the lower RI rate.
- **1-year RI**: ~40% savings vs on-demand. **3-year RI**: ~60–72% savings. Monthly payment option available (slightly less savings but no large upfront payment).
- RI applies to any VM of the matching size and region. **Flexible Instance Size** lets the RI apply across VM sizes in the same family (B2s savings can apply to B4ms if you resize).
- **Azure Advisor** automatically recommends which VMs to reserve based on 7-day usage patterns. Always check Advisor before buying RIs.
- RIs are purchased separately from VMs — they're a billing instrument. You can sell unused RIs on the Azure RI exchange marketplace.

#### 5.3 Spot VMs — Up to 90% Savings for Non-Critical Workloads

```
# Create a Spot VM (for batch jobs, CI/CD build agents)
az vm create \
  --resource-group urbannest-prod-rg \
  --name build-agent-spot \
  --image Ubuntu2204 \
  --size Standard_D4s_v5 \
  --priority Spot \
  --eviction-policy Deallocate \
  --max-price -1 \
  --admin-username azureuser \
  --generate-ssh-keys
```

**📖 Spot VMs**

- **Spot VMs** use Azure's spare compute capacity at up to 90% discount. The trade-off: Azure can evict (stop) the VM with 30 seconds notice when capacity is needed elsewhere.
- **--eviction-policy Deallocate** — when evicted, VM is stopped and deallocated (disk preserved). You can restart it when capacity returns. Alternative: Delete (VM and disk deleted on eviction).
- **--max-price -1** — accept the current spot price, whatever it is. Alternatively set a max price you're willing to pay — VM is evicted if spot price exceeds your max.
- **Best uses:** CI/CD build agents, batch data processing, dev/test environments, rendering farms. Never for production stateful services that can't handle sudden eviction.
- Azure DevOps scale sets use Spot VMs as build agents — auto-provision Spot VMs for builds, auto-deallocate when idle. UrbanNest saves ~₹15,000/month on CI/CD.

#### 5.4 Blob Storage Lifecycle Policy — Automatic Tier Migration

```
# Apply lifecycle policy to storage account
az storage account management-policy create \
  --account-name urbanneststorage2026 \
  --resource-group urbannest-prod-rg \
  --policy '{
    "rules": [{
      "name": "property-images-tiering",
      "type": "Lifecycle",
      "definition": {
        "filters": {"blobTypes":["blockBlob"], "prefixMatch":["property-images/"]},
        "actions": {
          "baseBlob": {
            "tierToCool":    {"daysAfterModificationGreaterThan": 30},
            "tierToArchive": {"daysAfterModificationGreaterThan": 180},
            "delete":        {"daysAfterModificationGreaterThan": 365}
          }
        }
      }
    }]
  }'
```

**📖 Storage Lifecycle Policy**

- This policy runs daily and automatically moves blobs through tiers based on age — zero code, zero effort after setup.
- **After 30 days → Cool tier**: storage cost drops from ₹1.5/GB/month to ₹0.8/GB/month. For UrbanNest's 500GB of property images, this saves ₹350/month on blobs older than 30 days.
- **After 180 days → Archive tier**: storage cost drops to ₹0.12/GB/month. These are images for sold properties rarely accessed again.
- **After 365 days → Delete**: removes very old listing images. Adjust this based on legal retention requirements.
- **prefixMatch** — apply rules only to blobs in a specific folder prefix. Different policies for different data types (aggressive archiving for logs, conservative for property images).

#### 5.5 Azure Hybrid Benefit — Use Existing Licences

```
# Enable Hybrid Benefit on existing VM (use your Windows licence)
# Applies if company has existing Windows Server Software Assurance
az vm update \
  --resource-group urbannest-prod-rg \
  --name urbannest-api-vm \
  --license-type Windows_Server
# Apply Hybrid Benefit to Azure SQL (use existing SQL Server licence)
az sql db update \
  --resource-group urbannest-prod-rg \
  --server urbannest-sqlserver \
  --name urbanNestDB \
  --license-type BasePrice
```

**📖 Azure Hybrid Benefit**

- **Azure Hybrid Benefit (AHB)** — if your company already owns Windows Server or SQL Server licences with Software Assurance (common in Indian enterprises), you can apply them to Azure VMs and Azure SQL instead of paying the licence cost again.
- For Windows VMs: saves up to 40% of the VM cost (the OS licence portion).
- For Azure SQL: switch from "License Included" to "Base Price" (bring your own SQL licence) — saves up to 55%.
- For Linux VMs with RHEL/SUSE subscriptions, Hybrid Benefit also applies.
- Check if your organisation has existing Microsoft EA (Enterprise Agreement) licences before buying cloud — AHB could be your single biggest Azure cost saving.

Microsoft Azure

Cost Management › Cost Analysis › urbannest-prod-rg

February 2026

₹1,10,420

Before optimisation

March 2026

₹61,800

After RI + Spot + Lifecycle

Saving

₹48,620

44% reduction

Top saving Reserved Instances on 3 VMs → -₹22,000/month

2nd saving Spot VMs for CI/CD build agents → -₹15,000/month

3rd saving Blob Storage lifecycle policy → -₹7,800/month

4th saving Deleted idle East US VM → -₹3,820/month

### 6. Advanced Azure Services — Quick Reference

Azure Service

Purpose

Key CLI / Concept

Logic Apps

Low-code workflow automation with 400+ connectors

az logic workflow create + designer JSON

Service Bus Queue

Point-to-point messaging, reliable delivery

az servicebus queue create + SDK

Service Bus Topic

Pub/sub fan-out — one message to multiple consumers

az servicebus topic + subscription create

Event Grid

Lightweight event routing for Azure resource events

az eventgrid event-subscription create

Cosmos DB

Globally distributed NoSQL — millisecond reads at any scale

az cosmosdb sql container create /partition-key /throughput

Azure Cache for Redis

In-memory cache for session data and hot queries

az redis create --sku Standard

Azure AD (Entra ID)

Employee identity, SSO, MFA, Conditional Access

az ad user create / group create

Azure AD B2C

Customer identity with social login (Google/Facebook)

B2C tenant + User Flows in Portal

Service Principal

Non-human identity for automation and pipelines

az ad sp create-for-rbac --json-auth

Managed Identity

Auto-managed identity for Azure resources (preferred)

identity: { type: SystemAssigned } in Bicep

Conditional Access

Enforce MFA, device compliance, location policies

Azure AD Portal → Security → Conditional Access

Azure Policy

Enforce governance rules (Deny/Audit/Modify)

az policy assignment create --policy ID --scope

Bicep

Azure-native IaC DSL — cleaner than ARM templates

az deployment sub create --template-file main.bicep

Reserved Instances

1–3 year commit for 40–72% VM/SQL savings

Azure Portal → Reservations → Purchase

Spot VMs

Up to 90% off for evictable non-critical workloads

az vm create --priority Spot --eviction-policy Deallocate

Azure Hybrid Benefit

Apply existing Windows/SQL licences to Azure

az vm update --license-type Windows_Server

Blob Lifecycle Policy

Auto-migrate blobs to cheaper tiers based on age

az storage account management-policy create

Cost Management Budgets

Alert teams when spend exceeds thresholds

az consumption budget create --amount

### 7. Interview Questions — Advanced Azure

##### Interview Q&A — Mid-Level (6 months–2 years Azure experience)

**Q: Q1. What is the difference between Azure Service Bus and Azure Event Grid?**

A: **Azure Service Bus** — enterprise message broker. Supports complex messaging patterns: queues (point-to-point), topics (pub-sub), dead-letter queues, message ordering, and deduplication. Messages persist until consumed (up to 14 days). Use for: reliable service-to-service communication, order processing, workflow orchestration.
**Azure Event Grid** — lightweight event routing. Purpose-built for reacting to Azure resource events (VM created, blob uploaded, SQL alert fired) and custom application events. Near real-time push delivery. No complex message features. Use for: reacting to infrastructure changes, webhook delivery, fan-out of simple events.
**Decision rule:** Does the consumer need guaranteed delivery, ordering, or dead-lettering? → Service Bus. Is it a simple "something happened, notify N subscribers"? → Event Grid.
**Cost:** Service Bus Standard ~₹7 per million operations. Event Grid ~₹4 per million events. Both scale to any volume.

**Q: Q2. What are Cosmos DB consistency levels and when do you use each?**

A: **Strong** — reads always return the latest committed data. Highest consistency, highest latency, lowest availability. Use for: financial ledgers, inventory counts. Only supported in single region.
**Bounded Staleness** — reads are at most N versions or T time behind. Predictable staleness. Use for: leader boards, job queues where slight staleness is acceptable.
**Session (default, most popular)** — within a single client session, reads always see your own writes. Different sessions may see slightly stale data. Use for: shopping carts, user profiles — your writes are always visible to you.
**Consistent Prefix** — reads never see out-of-order writes. If A then B were written, you either see neither, A, or A+B — never B alone. Use for: social media feeds, event sourcing.
**Eventual** — reads may return stale data but are the fastest and cheapest. Use for: heavily read-only data, recommendations, approximate counts.

**Q: Q3. What is the difference between a Service Principal and a Managed Identity in Azure?**

A: **Service Principal** — a manually created Azure AD application identity with client ID and client secret (or certificate). You are responsible for storing the secret securely, rotating it, and auditing its use. Use for: GitHub Actions (OIDC preferred), Terraform from local machines, third-party tools that run outside Azure.
**Managed Identity** — Azure creates and manages the credentials automatically. No client secret to store or rotate. Only resources running on Azure can use Managed Identities. Use for: App Service, Functions, VMs, AKS pods — anything running in Azure that needs to access other Azure services.
**Rule of thumb:** If your code runs on Azure, always use Managed Identity. Only use Service Principals for systems running outside Azure (GitHub, local scripts, on-premises tools).
Managed Identities also simplify auditing — Azure AD logs show exactly which resource (e.g. `urbannest-frontend` App Service) accessed Key Vault at what time.

**Q: Q4. What is Azure Policy and how does it differ from RBAC?**

A: **RBAC (Role-Based Access Control)** — controls what actions a user or service principal is allowed to perform. A Contributor can create any VM anywhere. RBAC is about permission.
**Azure Policy** — controls what resources are allowed to exist. A "Deny regions" policy blocks VM creation outside India even for a user with Owner role. Policy is about compliance.
**They work together:** RBAC grants the developer permission to create resources. Azure Policy enforces that those resources meet organisational standards (correct region, required tags, private endpoints).
**Policy effects:** Deny (hard block), Audit (allow but flag in compliance report), Modify (auto-add tags), DeployIfNotExists (auto-deploy a required companion resource like a diagnostic setting).
Azure Policy provides a **compliance report** — a dashboard showing what percentage of resources are compliant with each policy. Critical for security audits and regulatory compliance (GDPR, ISO 27001).

**Q: Q5. When should you use Azure Reserved Instances vs Spot VMs?**

A: **Reserved Instances (RIs)** — commit to 1 or 3 years for a 40–72% discount. The VM runs exactly like an on-demand VM — same performance, same availability. If Azure runs out of capacity, your RI is always served first. Use for: production VMs, databases, any workload that must run 24/7 continuously.
**Spot VMs** — use Azure's spare capacity at up to 90% discount. Azure can evict (stop) with 30 seconds notice when capacity is needed. Use for: CI/CD build agents, batch jobs, dev/test, rendering — anything that can handle sudden interruption.
**Combining both:** Use RIs for your core production VMs (App Service Plan, AKS nodes). Use Spot VMs for scale-set overflow nodes and CI/CD agents. Use on-demand as a fallback. This "blended" strategy minimises cost while maintaining reliability.
Never use Spot VMs for: databases, session-based apps, stateful services, or anything where 30-second eviction causes data loss or user impact.

**Q: Q6. What is Bicep and why would you use it over ARM templates?**

A: **ARM templates** — Azure's original IaC format. JSON-based. Very verbose and complex — a simple VM can require 200+ lines of JSON. Difficult to read, write, and debug. No native IDE support for complex features.
**Bicep** — compiles to ARM templates but uses a much cleaner syntax. The same VM in Bicep is 20 lines. Native VS Code extension with autocomplete, type checking, and linting. First-class support in Azure CLI and Azure DevOps.
Bicep **modules** allow creating reusable components (like Terraform modules) — define an App Service module once, use it for dev, staging, and prod by passing different parameters.
**Bicep vs Terraform:** Bicep is Azure-only, simpler for Azure-focused teams. Terraform is multi-cloud (Azure + AWS + GCP in one codebase) but requires more setup. For a company using only Azure, Bicep is the recommended choice by Microsoft. For multi-cloud or teams already using Terraform, stick with Terraform.

**Quiz: Quiz 1 — UrbanNest wants email and Teams notifications when a new property is listed. The API must not slow down. What is the correct architecture?**

- A) API directly calls the Outlook API and Teams API synchronously before returning the response
- B) API publishes one message to a Service Bus topic with two subscriptions. Two Logic Apps (one for email, one for Teams) consume from their subscriptions asynchronously.
- C) API calls an Azure Function that sends both notifications synchronously
- D) A cron job checks for new listings every 5 minutes and sends notifications

> **Answer/explanation:** ✅ Answer: **B — Service Bus pub/sub with Logic Apps**
The API publishes one lightweight message to Service Bus and returns immediately. Response time unchanged.
Service Bus topics fan out the message to both subscriptions independently. Email and Teams processing happen asynchronously and don't affect each other.
Option A is synchronous coupling — if Outlook API is slow (2 seconds), every property listing API call takes 2+ seconds.
Option C is still coupled through the Function — if the Function fails, you lose the notification.
Option D has a 5-minute delay and is not real-time. Service Bus delivers within seconds of publishing.

**Quiz: Quiz 2 — A new developer creates a VM in Australia region, costing ₹3,000/month unexpectedly. What is the BEST way to prevent this in future?**

- A) Remove the developer's Contributor role so they can't create VMs
- B) Create an Azure Policy with "Deny" effect for regions outside centralindia and southindia, assigned at subscription scope
- C) Send an email to all developers asking them to create VMs only in India
- D) Add a budget alert that fires when spend exceeds ₹75,000

> **Answer/explanation:** ✅ Answer: **B — Azure Policy Deny effect on allowed regions**
Azure Policy with Deny effect is the **only technical enforcement mechanism**. It prevents the resource creation before it happens — not after.
Option A removes too much access — developers need to create VMs for legitimate work.
Option C is a process control that relies on humans following instructions — it clearly didn't work.
Option D catches the cost after the fact — the VM is already running. Budget alerts are complementary but don't prevent the problem.
Azure Policy cannot be bypassed by RBAC — even a subscription Owner cannot create resources that violate a "Deny" policy.

**Quiz: Quiz 3 — UrbanNest's App Service needs to read from Azure Cosmos DB. What is the most secure authentication method?**

- A) Store the Cosmos DB primary key in an app setting environment variable
- B) Store the Cosmos DB connection string in Azure Key Vault, fetch it at startup using a Service Principal client secret
- C) Enable System-assigned Managed Identity on the App Service, grant it "Cosmos DB Built-in Data Reader" RBAC role on the Cosmos account
- D) Hardcode the connection string in config.json and store it in the Git repository

> **Answer/explanation:** ✅ Answer: **C — Managed Identity + RBAC**
Managed Identity means **zero credentials stored anywhere**. Azure manages the identity and rotation automatically.
Option A stores the key in a setting — visible to anyone with Contributor access to the App Service.
Option B is an improvement but still requires a client secret that must be stored and rotated. Two credential problems instead of one.
Option D is the worst option — credentials in Git are permanent and can never be truly revoked from commit history.
Managed Identity is the recommended approach for any Azure-to-Azure communication. It satisfies all three security principles: no credentials to leak, automatic rotation, fine-grained RBAC permissions.

> **Advanced Azure — Core Takeaways**

> - **Service Bus decouples services** — the property listing API should not wait for email notifications. Publish to Service Bus and return immediately. Downstream consumers process asynchronously at their own pace. This pattern is the difference between a 200ms API and a 3-second API.
> - **Cosmos DB partition key is the most important design decision** — a bad partition key (low cardinality or hot partition) causes performance cliffs at scale that are expensive to fix. Design access patterns first, then design the schema.
> - **Azure AD is your security foundation** — SSO, Conditional Access, and Managed Identity together eliminate the three biggest enterprise identity risks: password reuse, unrevoked offboarded-employee accounts, and hardcoded credentials in code.
> - **Azure Policy is governance-as-code** — documentation, wikis, and Confluence pages that describe the architecture standards are ignored. Policy rules written in JSON and assigned at subscription scope are enforced by the platform. There is no way around them.
> - **Bicep makes infrastructure reproducible** — clicking in the Portal cannot be code-reviewed, version-controlled, or reliably reproduced. A Bicep file can be deployed to recreate an entire environment in 15 minutes. That's the difference between a 4-hour incident recovery and a 30-minute one.
> - **Cost optimisation is not a one-time task** — Reserved Instances for predictable workloads, Spot for interruptible, lifecycle policies for storage, right-sizing based on Azure Advisor. Review Cost Management weekly and act on Advisor recommendations. A mature team saves 40–60% vs a team that ignores cost.
> - **The Managed Identity pattern beats all credential alternatives** — for any code running on Azure, enable Managed Identity and use RBAC instead of connection strings or client secrets. This pattern is now the Azure Well-Architected Framework recommendation for all Azure-to-Azure authentication.

##### Advanced Azure Engineering Standards — UrbanNest Rules

- Enable **Microsoft Defender for Cloud** on all subscriptions — it continuously evaluates your resources against security baselines, flags misconfigurations, and provides a Secure Score with specific remediation steps. Free tier covers core recommendations
- Use **Bicep parameter files** (main.bicepparam) for environment-specific values — never hardcode environment names, resource sizes, or region names in the Bicep template itself. One template, multiple parameter files for dev/staging/prod
- Store Bicep modules in a **private module registry (Azure Container Registry)** — teams can reference shared, versioned modules instead of copying Bicep files across repos. Fixes a bug in the module once and all projects get the update on next deploy
- Set up **Azure AD Emergency Access accounts** (break-glass accounts) — two accounts excluded from all Conditional Access policies with very strong passwords stored in a physical safe. For the scenario where a misconfigured CA policy locks everyone out of Azure AD
- Review **Azure Advisor weekly and act on Cost recommendations** — Advisor automatically identifies idle VMs, underutilised disks, unused public IPs, and RI purchase opportunities. Responding weekly prevents the surprise bill spikes that UrbanNest experienced
- Use **Logic Apps Standard tier** for production workflows requiring VNet integration — Consumption tier Logic Apps run on shared infrastructure with no private network access. Standard tier supports Private Endpoints so your Logic App can reach private SQL and Storage without public exposure

##### 🏋️ Hands-On Exercises — Extend UrbanNest's Advanced Services

1. **Build a complete Service Bus fan-out notification system:** Create a Service Bus namespace with a topic called `property-events`. Add three subscriptions: email-alerts, teams-alerts, and audit-log. Write a Python script that publishes a property listing message. Build two Logic Apps (email via Outlook 365 and a Teams post) consuming from their subscriptions. Verify both receive the message independently. Add a SQL filter on the email subscription: only receive messages where city = "Pune" (use message properties for filtering).
2. **Design a Cosmos DB multi-access-pattern schema:** UrbanNest needs to query listings by: (1) city + BHK type, (2) agentId, (3) price range. Design a Cosmos DB schema with the optimal partition key. Create the container and a Global Secondary Index equivalent (using Cosmos DB's composite indexes). Write Python code for each of the three query patterns and measure the RU consumption for each. Identify which query is cross-partition and explain how to fix it.
3. **Implement a full Conditional Access rollout:** Create four Conditional Access policies in sequence: (1) Require MFA for Azure Portal access from outside India, (2) Block access from known risky locations (use named locations), (3) Require a compliant device for access to production App Services, (4) Block all legacy authentication protocols (disable SMTP auth, IMAP auth). Test each policy in Report-Only mode first. Document how to roll back each policy safely if it breaks legitimate access.
4. **Write a Bicep template for all 5 phases:** Convert all UrbanNest's Azure resources from the foundations and advanced modules into a single Bicep project with modules: networking.bicep (VNet, subnets, NSGs), compute.bicep (VM, App Service), storage.bicep (Storage Account, Cosmos DB), security.bicep (Key Vault, Managed Identity role assignments), and monitoring.bicep (Application Insights, action groups). Deploy with a single command: `az deployment sub create --template-file main.bicep --parameters @prod.bicepparam`. Verify the entire stack deploys from scratch in under 15 minutes.
5. **Run a cost optimisation sprint:** Use Azure Cost Management to identify the 5 most expensive resources in your subscription for the last 30 days. For each: (1) determine if it's over-provisioned (check utilisation metrics via Azure Monitor), (2) check if an RI would apply, (3) check Azure Advisor for specific recommendations. Implement at least 3 changes. Create a budget at ₹80,000/month with email alerts at 70% and 90%. Set up a monthly Cost Management scheduled report emailed to the ops team every 1st of the month.

### Advanced Azure Module Complete 🎉

You have mastered the advanced Azure services used by Indian enterprises — Logic Apps and Service Bus for decoupled integration, Cosmos DB for globally distributed NoSQL, Azure AD / Entra ID for enterprise identity and B2C customer login, Azure Policy and Bicep for governance-as-code, and Cost Management with Reserved Instances, Spot VMs, and Hybrid Benefit for 40–70% cost reduction.

> **Priya**
> 
> "Ravi, the cost report for March: we went from ₹1.1 lakh to ₹61,800. Reserved Instances on the core VMs, Spot VMs for CI/CD build agents, Blob lifecycle policy moving 18 months of old property images to Archive. No performance change. No downtime. Pure billing optimisation. Azure Advisor is now showing yellow for two resources — we'll handle those next week. That's what mature cloud management looks like: weekly reviews, continuous improvement, never surprised by the bill."

> **Sameer**
> 
> "The security audit passed with a 96/100 Secure Score in Defender for Cloud. The auditor specifically mentioned: Managed Identity everywhere, Conditional Access requiring MFA for all admin access, Azure Policy blocking non-compliant resource creation, and no client secrets in any application configuration. Six months ago we had a shared PEM file and a database password in a Confluence page. Today we have zero credentials in code, zero manually managed secrets, and automated revocation when someone leaves the company. That is enterprise-grade Azure."

> **Next: Azure Well-Architected Framework & Production Readiness**

> - Reliability — multi-region deployment with Azure Traffic Manager, RTO/RPO planning, Azure Site Recovery for disaster recovery
> - Security — Microsoft Defender for Cloud, Sentinel (SIEM), Privileged Identity Management (PIM) for just-in-time admin access
> - Cost Optimisation — FinOps practices, tag-based chargebacks, Savings Plans vs Reserved Instances, right-sizing automation
> - Operational Excellence — Azure Monitor Workbooks, Log Analytics KQL queries, automated runbooks with Azure Automation
> - Performance Efficiency — Azure Front Door for global load balancing, Azure Cache for Redis, read replicas for Azure SQL
> - Sustainability — Azure Carbon Optimisation dashboard, Graviton-equivalent ARM64 VMs (Ampere Altra), region selection for renewable energy percentage
