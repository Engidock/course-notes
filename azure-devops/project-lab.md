# 🔷 Azure DevOps Project Mastery

> **Hey Fresher — Read This First!**
>
> Azure DevOps is Microsoft's end-to-end DevOps platform — Boards for work tracking, Repos for Git hosting, and Pipelines for CI/CD, all wired together. The part you'll live in most as a release engineer is Azure Pipelines: YAML files that define exactly how code goes from a commit to a running service, including build steps, test gates, and staged releases across environments with human approvals in between.
>
> **MediSetu Health** is a telemedicine platform connecting patients in tier-2 and tier-3 Indian cities with doctors over video consultations and e-prescriptions. Their appointment-booking service runs on Azure App Service, and a bad deploy doesn't just break a demo — it can mean a patient waiting for a doctor who never connects. You're joining as a Release Engineer. Your first assignment: replace the team's habit of manually zipping a build and uploading it through the Azure Portal with a proper YAML pipeline that builds, tests, and promotes through dev, staging, and production with real approval gates.

#### What You Will Learn and Build in This Project

You will write a multi-stage `azure-pipelines.yml` from scratch, connect Azure DevOps to an Azure subscription with a service connection, build and publish artifacts, deploy to Azure App Service across dev, staging, and production environments, gate production behind manual approvals using Environments, centralize secrets with variable groups and Key Vault, and extract reusable pipeline templates for MediSetu's other microservices.

Azure Pipelines YAML, Service Connections, Build and Publish Artifacts, Multi-Stage Pipelines, Environments and Approvals, Variable Groups, Azure Key Vault Integration, Pipeline Templates, Deployment Jobs, Release Gates

> **📦 Phase 1 — Project Setup and Service Connections**
>
> Create the Azure DevOps project, connect it to the MediSetu Azure subscription, and scaffold the repo for pipeline-as-code.

> **📦 Phase 2 — The CI Build Stage**
>
> Write the YAML build stage that restores, builds, tests, and publishes the appointment-booking service as a deployable artifact.

> **📦 Phase 3 — Multi-Stage Release with Environments and Approvals**
>
> Extend the pipeline with dev, staging, and production stages, using Azure DevOps Environments to gate the production deploy behind manual approval.

> **📦 Phase 4 — Variable Groups and Key Vault**
>
> Move environment-specific configuration and secrets out of the YAML file and into variable groups backed by Azure Key Vault.

> **📦 Phase 5 — Reusable Pipeline Templates**
>
> Extract the build and deploy logic into shared YAML templates so MediSetu's five other microservices don't each duplicate the same pipeline.

> **📦 Phase 6 — Release Gates, Monitoring and Rollback**
>
> Add automated post-deployment gates and a rollback stage that redeploys the last known-good artifact on failure.

**Scene 1 — MediSetu Health, Pune | The Portal-Upload Problem**

> **Roshan** _Junior Release Engineer_
>
> I watched Karthik deploy last Friday. He built the app locally, zipped the publish folder, and dragged it into the Azure Portal's "Deploy Center" for the production App Service. If his laptop had crashed mid-zip, nobody else on the team even knows the exact steps he follows.

> **Priya** _Senior DevOps Engineer_
>
> That's the whole problem. No build log, no test gate, no record of what commit is actually running in production. We're going to write this as a YAML pipeline instead — it lives in the repo, it's code-reviewed like any other change, and it runs the same way every single time.

> **Karthik** _Engineering Manager_
>
> And appointment-booking is patient-facing. If staging looks fine but production breaks because someone forgot to set an environment variable, that's real patients unable to book a consultation. I want a manual approval step before anything touches production, and I want it impossible to skip.

> **Roshan**
>
> So: one pipeline file, automatic through dev and staging, a real human gate before prod, and secrets that aren't sitting in a YAML file in Git.

> **Priya**
>
> That's exactly the shape we're building.

### 1. Phase 1 — Project Setup and Service Connections

**Business Problem:** Azure DevOps needs permission to deploy into MediSetu's Azure subscription on the team's behalf, and it needs somewhere to authenticate to Azure without anyone pasting a password into a pipeline file.

#### 1.1 Creating the Service Connection

```bash
# Using the Azure DevOps CLI extension
az extension add --name azure-devops

az devops service-endpoint azurerm create \
  --azure-rm-service-principal-id <app-id> \
  --azure-rm-subscription-id 8f2a1c3e-9b7d-4e2a-af31-6d0e2c9b7a11 \
  --azure-rm-subscription-name "MediSetu-Production" \
  --azure-rm-tenant-id 3a1b2c3d-4e5f-6789-abcd-ef0123456789 \
  --name medisetu-azure-connection \
  --org https://dev.azure.com/medisetu \
  --project appointment-booking
```

> **📖 What a service connection is**
> A service connection is Azure DevOps's stored, authenticated link to an external system — here, an Azure Resource Manager (ARM) subscription. `--azure-rm-service-principal-id` points to a Microsoft Entra ID app registration that has been granted a scoped role (Contributor, ideally scoped to just the App Service's resource group, not the whole subscription) on the MediSetu-Production subscription. Once created, pipeline YAML references the connection by `--name`, never by raw credentials — so a compromised pipeline file leaks no secrets, only a reference to a connection whose actual credentials are managed centrally in Project Settings.

#### 1.2 Repository Layout for Pipeline-as-Code

```
appointment-booking/
├── azure-pipelines.yml
├── src/
│   ├── AppointmentBooking.Api/
│   └── AppointmentBooking.Tests/
├── templates/
│   ├── build-stage.yml
│   └── deploy-stage.yml
└── AppointmentBooking.sln
```

> **azure-pipelines.yml at the repo root** — Azure DevOps auto-detects this file when you create a new pipeline pointed at the repo. Keeping build and deploy logic in `templates/` from day one (used starting in Phase 5) means the root file stays a short, readable orchestration script rather than growing into an unmaintainable wall of YAML.

> **Key takeaways**
> - Service connections store and scope Azure credentials centrally — pipeline YAML only ever references a connection name.
> - Scope the service principal's role to the specific resource group, not the whole subscription, following least privilege.
> - `azure-pipelines.yml` at the repo root is auto-detected by Azure DevOps when creating a pipeline from an existing repo.

### 2. Phase 2 — The CI Build Stage

**Business Problem:** Right now there is no automated build at all — "does it compile, do the tests pass" is discovered by Karthik running it locally. The first stage of the pipeline needs to restore dependencies, build the .NET solution, run unit tests, and publish a versioned, deployable artifact.

#### 2.1 The Build Stage

```yaml
trigger:
  branches:
    include:
      - main
      - release/*

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'

stages:
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: BuildJob
        displayName: 'Restore, Build, Test, Publish'
        steps:
          - task: UseDotNet@2
            inputs:
              version: $(dotnetVersion)
              packageType: 'sdk'

          - task: DotNetCoreCLI@2
            displayName: 'Restore'
            inputs:
              command: 'restore'
              projects: '**/*.sln'

          - task: DotNetCoreCLI@2
            displayName: 'Build'
            inputs:
              command: 'build'
              projects: '**/*.sln'
              arguments: '--configuration $(buildConfiguration) --no-restore'

          - task: DotNetCoreCLI@2
            displayName: 'Run Unit Tests'
            inputs:
              command: 'test'
              projects: '**/*Tests/*.csproj'
              arguments: '--configuration $(buildConfiguration) --no-build --collect "Code coverage"'

          - task: DotNetCoreCLI@2
            displayName: 'Publish'
            inputs:
              command: 'publish'
              projects: 'src/AppointmentBooking.Api/AppointmentBooking.Api.csproj'
              arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
              zipAfterPublish: true

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'appointment-booking-drop'
```

> **📖 Reading the build stage line by line**
> `trigger.branches.include` limits CI runs to `main` and any `release/*` branch — feature branches don't burn build minutes unless explicitly needed. `pool.vmImage: 'ubuntu-latest'` requests a Microsoft-hosted Linux build agent; no infrastructure to maintain. `UseDotNet@2` installs the exact .NET SDK version the team standardized on, so a developer's local SDK drift never causes "works on my machine" build failures. The three `DotNetCoreCLI@2` tasks — restore, build, test — run in sequence; `test` uses `--no-build` because the previous step already compiled everything, avoiding duplicate work. `publish --output $(Build.ArtifactStagingDirectory)` writes the deployable output to Azure Pipelines' predefined staging directory, and `zipAfterPublish: true` packages it as a single `.zip`, which is exactly the format the App Service deploy task expects. `PublishBuildArtifacts@1` uploads that directory as a named artifact (`appointment-booking-drop`) that later deploy stages can download by name — this is how build output crosses the stage boundary.

**Quiz: Why does the "Run Unit Tests" task include `--no-build` in its arguments?**
- To skip running the tests entirely
- Because the solution was already compiled by the preceding Build task, so re-running dotnet build during test would waste time
- Because Azure Pipelines requires it for all test tasks
- To prevent code coverage collection

> **Answer/explanation:** The correct answer is the second option. `dotnet test` normally builds the project before running tests. Since the pipeline already ran `dotnet build` in the prior step, `--no-build` tells the test runner to reuse those existing binaries instead of rebuilding, saving pipeline time. It does not skip test execution — tests still run — and it has no special Azure Pipelines requirement or effect on code coverage collection, which is handled by the separate `--collect "Code coverage"` flag.

### 3. Phase 3 — Multi-Stage Release with Environments and Approvals

**Business Problem:** Karthik's requirement is explicit: dev and staging can deploy automatically, but nothing reaches production without a named human clicking approve — and that approval has to be enforced by Azure DevOps itself, not by team discipline.

**Scene 2 — Design Review**

> **Roshan** _Junior Release Engineer_
>
> If I just add a "prod" stage after "staging" in the YAML, doesn't it deploy automatically the moment staging succeeds?

> **Priya** _Senior DevOps Engineer_
>
> Yes, unless the stage targets an Environment that has an approval check configured. That's the actual gate — it's not a YAML condition, it's configuration on the Environment resource itself, so it can't be bypassed by editing the pipeline file.

#### 3.1 Creating Environments with Approval Checks

```bash
az pipelines environment create \
  --name production-app-service \
  --project appointment-booking \
  --org https://dev.azure.com/medisetu
```

> After creation, the approval check itself is configured in the Azure DevOps UI under **Environments → production-app-service → Approvals and checks**, adding Karthik and Priya as required approvers with a 24-hour timeout. This is deliberately not expressible purely in YAML — it protects the gate from being silently removed in a pull request.

#### 3.2 The Full Multi-Stage Pipeline

```yaml
stages:
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: BuildJob
        steps:
          - script: echo "Build steps from Phase 2 go here"

  - stage: DeployDev
    displayName: 'Deploy to Dev'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: DeployToDev
        environment: 'dev-app-service'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'medisetu-azure-connection'
                    appType: 'webAppLinux'
                    appName: 'medisetu-appointment-dev'
                    package: '$(Pipeline.Workspace)/appointment-booking-drop/**/*.zip'

  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: DeployDev
    condition: succeeded()
    jobs:
      - deployment: DeployToStaging
        environment: 'staging-app-service'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'medisetu-azure-connection'
                    appType: 'webAppLinux'
                    appName: 'medisetu-appointment-staging'
                    package: '$(Pipeline.Workspace)/appointment-booking-drop/**/*.zip'
                    deploymentMethod: 'auto'

  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: DeployToProd
        environment: 'production-app-service'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'medisetu-azure-connection'
                    appType: 'webAppLinux'
                    appName: 'medisetu-appointment-prod'
                    package: '$(Pipeline.Workspace)/appointment-booking-drop/**/*.zip'
                    deploymentMethod: 'auto'
                    slotName: 'staging-slot'
```

> **📖 What makes this a real gated release, not just three copy-pasted stages**
> Each `deployment` job (as opposed to a plain `job`) targets an `environment:` — `dev-app-service`, `staging-app-service`, `production-app-service` — and Azure Pipelines checks that environment's configured approvals *before* running any steps inside it. Because `production-app-service` has required approvers configured, the `DeployProduction` stage will start, immediately pause, notify Karthik and Priya, and wait — even though `dependsOn: DeployStaging` and `condition: succeeded()` are satisfied. `strategy: runOnce` is the simplest deployment strategy Azure Pipelines offers for a `deployment` job — it just runs the deploy steps once, as opposed to `rolling` or `canary` strategies used for VM-based or Kubernetes deployments. `slotName: 'staging-slot'` on the production deploy targets an App Service deployment slot rather than the live slot directly, which sets up the swap-based rollback used in Phase 6.

**Runonce vs Rolling vs Canary Deployment Strategy (Azure Pipelines `deployment` jobs)**

- **runOnce** — executes deploy steps a single time against all targets at once; simplest strategy, what MediSetu uses for App Service deploys since App Service itself doesn't expose partial-instance rollout the way a VM scale set or Kubernetes cluster does.
- **rolling** — updates a subset of targets (VMs, typically) at a time, useful when deploying to a VM scale set and you want to avoid taking every instance down simultaneously.
- **canary** — deploys to a small subset first, commonly paired with Kubernetes manifests, to validate before full rollout; not applicable to a single App Service instance the way it is to a replicated Kubernetes deployment.

> **Key takeaways**
> - `deployment` jobs targeting an `environment:` are what actually enforce approval gates — the gate lives on the Environment resource, not in pipeline YAML.
> - `dependsOn` plus `condition: succeeded()` controls stage ordering; the environment approval controls whether the stage is allowed to run its steps at all.
> - `strategy: runOnce` is the standard choice for App Service deploys; rolling and canary apply to VM scale sets and Kubernetes respectively.

### 4. Phase 4 — Variable Groups and Key Vault

**Business Problem:** The pipeline above hardcodes App Service names, but the connection string for MediSetu's patient database, the SMS gateway API key for appointment reminders, and the JWT signing secret cannot live in plaintext YAML committed to Git.

#### 4.1 Creating a Variable Group Linked to Key Vault

```bash
az pipelines variable-group create \
  --name medisetu-prod-secrets \
  --project appointment-booking \
  --org https://dev.azure.com/medisetu \
  --authorize true \
  --variables placeholder=unused
```

> After creation, the variable group is switched to **Link secrets from an Azure key vault as variables** in the UI, pointing at the `medisetu-prod-kv` Key Vault and selecting the specific secrets to expose: `PatientDbConnectionString`, `SmsGatewayApiKey`, `JwtSigningSecret`. This means the variable group holds *references*, not copies — rotating a secret in Key Vault updates what the pipeline sees on its next run, with no pipeline edit required.

#### 4.2 Referencing the Variable Group and Key Vault Task in YAML

```yaml
  - stage: DeployProduction
    displayName: 'Deploy to Production'
    variables:
      - group: medisetu-prod-secrets
    jobs:
      - deployment: DeployToProd
        environment: 'production-app-service'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureKeyVault@2
                  inputs:
                    azureSubscription: 'medisetu-azure-connection'
                    KeyVaultName: 'medisetu-prod-kv'
                    SecretsFilter: 'PatientDbConnectionString,SmsGatewayApiKey,JwtSigningSecret'
                    RunAsPreJob: true

                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'medisetu-azure-connection'
                    appType: 'webAppLinux'
                    appName: 'medisetu-appointment-prod'
                    package: '$(Pipeline.Workspace)/appointment-booking-drop/**/*.zip'
                    appSettings: >
                      -ConnectionStrings__PatientDb $(PatientDbConnectionString)
                      -SmsGateway__ApiKey $(SmsGatewayApiKey)
                      -Jwt__SigningKey $(JwtSigningSecret)
```

> **📖 Two ways secrets reach this stage**
> `variables: - group: medisetu-prod-secrets` pulls in the variable group created above — those secrets become pipeline variables scoped to this stage only, never printed to logs (Azure Pipelines automatically masks values sourced from a variable group marked secret). The `AzureKeyVault@2` task is an alternative/complementary path: it fetches secrets directly from Key Vault at run time using the service connection's identity, useful when secrets need to be fetched fresh rather than cached in the variable group. `appSettings` passes the resolved values into the App Service's application settings at deploy time — MediSetu's ASP.NET Core app reads them from `IConfiguration` exactly as if they were set in `appsettings.json`, but they never touch source control.

**Variable Groups vs. Direct Key Vault Task**

- **Variable group linked to Key Vault** — secrets are resolved once when the pipeline queues, cached for that run, simplest to reference (`$(SecretName)` anywhere in the stage); MediSetu's default choice.
- **AzureKeyVault@2 task inline** — fetches secrets at the exact moment the task runs, useful for very short-lived or frequently rotated secrets, or when a stage needs secrets a variable group doesn't have authorization to expose.

> **Quiz: Why does MediSetu link the variable group to Key Vault instead of typing the database connection string directly into the variable group as a "secret" variable?**
> - Typed secret variables in a variable group don't actually get encrypted
> - Key Vault-linked variables update automatically when the underlying secret is rotated in Key Vault, with no pipeline edit needed, and centralize secret access control in one place
> - Azure Pipelines doesn't support typed secret variables at all
> - It makes the YAML file shorter

> **Answer/explanation:** The correct answer is the second option. Typed "secret" variables in a variable group are encrypted and masked in logs, so that isn't the distinguishing factor. The real advantage of linking to Key Vault is that MediSetu's security team manages secret rotation and access policies in one place (Key Vault), and every pipeline referencing that variable group automatically picks up a rotated secret's new value on the next run — nobody has to remember to update a value pasted into Azure DevOps. Azure Pipelines does support typed secret variables directly, and YAML length is not the deciding factor.

### 5. Phase 5 — Reusable Pipeline Templates

**Business Problem:** MediSetu has five other microservices — doctor-directory, e-prescription, billing, notification-service, patient-records — and each team is about to copy-paste this entire YAML file and slowly drift out of sync with it.

#### 5.1 Extracting a Build Template

```yaml
# templates/build-stage.yml
parameters:
  - name: projectPath
    type: string
  - name: testProjectPattern
    type: string
    default: '**/*Tests/*.csproj'

steps:
  - task: UseDotNet@2
    inputs:
      version: '8.0.x'
      packageType: 'sdk'

  - task: DotNetCoreCLI@2
    displayName: 'Restore'
    inputs:
      command: 'restore'
      projects: '**/*.sln'

  - task: DotNetCoreCLI@2
    displayName: 'Build'
    inputs:
      command: 'build'
      projects: '**/*.sln'
      arguments: '--configuration Release --no-restore'

  - task: DotNetCoreCLI@2
    displayName: 'Test'
    inputs:
      command: 'test'
      projects: '${{ parameters.testProjectPattern }}'
      arguments: '--configuration Release --no-build'

  - task: DotNetCoreCLI@2
    displayName: 'Publish'
    inputs:
      command: 'publish'
      projects: '${{ parameters.projectPath }}'
      arguments: '--configuration Release --output $(Build.ArtifactStagingDirectory)'
      zipAfterPublish: true
```

> **📖 Templates as parameterized functions for pipelines**
> `parameters:` declares inputs the template accepts, with `testProjectPattern` given a sensible `default` so most callers don't need to specify it. `${{ parameters.projectPath }}` is compile-time template expansion — Azure Pipelines substitutes the actual value before the YAML is even parsed as a pipeline, which is why template parameters can be used in places (like `projects:`) where runtime variables sometimes can't. This single file is now the one place build logic for any MediSetu service lives; fixing a bug in it fixes it for every service that references the template.

#### 5.2 Consuming the Template from Each Microservice's Pipeline

```yaml
# doctor-directory/azure-pipelines.yml
resources:
  repositories:
    - repository: templates
      type: git
      name: appointment-booking

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - template: templates/build-stage.yml@templates
            parameters:
              projectPath: 'src/DoctorDirectory.Api/DoctorDirectory.Api.csproj'
              testProjectPattern: '**/DoctorDirectory.Tests/*.csproj'
```

> **Cross-repository templates** — `resources.repositories` registers the `appointment-booking` repo (where the shared templates live) under the alias `templates`. `template: templates/build-stage.yml@templates` then pulls the shared build logic into doctor-directory's own pipeline, passing only the two things that differ between services: the project path and test pattern. When Priya fixes a bug in the shared template — say, pinning a different .NET SDK patch version — every microservice's next pipeline run picks it up automatically, no coordinated multi-repo change required.

> **Key takeaways**
> - `${{ parameters.x }}` is template-expansion syntax, resolved before the pipeline runs, distinct from runtime `$(variable)` syntax.
> - Templates can live in a separate repository and be referenced with `resources.repositories` + `@templates` alias — enabling one shared pipeline library across many services.
> - Centralizing build/deploy logic in templates means a fix or policy change propagates to every consuming pipeline automatically.

### 6. Phase 6 — Release Gates, Monitoring and Rollback

**Business Problem:** Approval gates catch human-judgment issues, but Karthik also wants an automated check — if the newly deployed production version fails its health check, the pipeline should not just report red, it should actively roll back to the previous working deployment slot.

#### 6.1 Deployment Slots and Slot Swap

```yaml
                - task: AzureAppServiceManage@0
                  displayName: 'Swap staging slot into production'
                  inputs:
                    azureSubscription: 'medisetu-azure-connection'
                    action: 'Swap Slots'
                    webAppName: 'medisetu-appointment-prod'
                    resourceGroupName: 'medisetu-prod-rg'
                    sourceSlot: 'staging-slot'

                - task: PowerShell@2
                  displayName: 'Health check after swap'
                  inputs:
                    targetType: 'inline'
                    script: |
                      $response = Invoke-WebRequest -Uri "https://medisetu-appointment-prod.azurewebsites.net/health" -UseBasicParsing
                      if ($response.StatusCode -ne 200) {
                        Write-Error "Health check failed with status $($response.StatusCode)"
                        exit 1
                      }
```

> **📖 Zero-downtime deploy with an automated safety check**
> The service deployed earlier in Phase 3 to `slotName: 'staging-slot'` is a warm, fully running copy of the app on the same App Service plan — not yet receiving production traffic. `AzureAppServiceManage@0` with `action: 'Swap Slots'` performs an atomic swap: the staging slot becomes production and the old production becomes the staging slot, with essentially no downtime because both were already warmed up. The PowerShell health-check step immediately after hits the `/health` endpoint; a non-200 response fails the task with `exit 1`, which fails the pipeline stage.

#### 6.2 Automatic Rollback on Failed Health Check

```yaml
                - task: AzureAppServiceManage@0
                  displayName: 'Rollback: swap back on health check failure'
                  condition: failed()
                  inputs:
                    azureSubscription: 'medisetu-azure-connection'
                    action: 'Swap Slots'
                    webAppName: 'medisetu-appointment-prod'
                    resourceGroupName: 'medisetu-prod-rg'
                    sourceSlot: 'staging-slot'
```

> **`condition: failed()`** — this task only runs if a previous step in the same job failed, which here means the health check. Because a slot swap is its own inverse (swapping the same two slots again reverses the previous swap), running the exact same swap task rolls production back to the previous known-good version in seconds, without needing a separate redeploy of old artifacts. Roshan no longer needs to manually find the last good build and re-upload it — the pipeline does it automatically the moment the health check fails.

> **Key takeaways**
> - Deployment slots let you warm up a new version before it takes production traffic, and a slot swap is near-instant and reversible.
> - `condition: failed()` on a pipeline step lets you attach automated recovery logic — like a rollback swap — directly to the pipeline that made the change.
> - Automated health-check-triggered rollback removes the dependency on a human noticing a broken production deploy at 2 a.m.

##### 🏋️ Hands-On Exercises — Extend the Project

1. Add a fourth stage, `DeploySmokeTest`, that runs after `DeployStaging` and before `DeployProduction`, hitting three critical endpoints (`/health`, `/api/appointments/availability`, `/api/doctors`) and failing the pipeline if any returns a non-200 status.
2. Create a second variable group, `medisetu-staging-secrets`, and update the `DeployStaging` stage to use staging-specific Key Vault secrets instead of reusing production values.
3. Extract the `AzureWebApp@1` deploy steps into a second shared template, `templates/deploy-stage.yml`, parameterized by `appName` and `environmentName`, and update all three stages (dev, staging, production) to consume it.
4. Add a scheduled trigger that runs the full pipeline nightly against `main` even without new commits, to catch environment drift (expired certificates, changed Key Vault permissions) before a real deploy needs to happen.
5. Configure a second required approver group on the `production-app-service` Environment so that any two of three senior engineers (not just Karthik or Priya alone) can approve a production release, and document why single-approver gates are a risk for a patient-facing system.

### Azure DevOps Project Complete 🎉

You have built a complete, gated release pipeline for MediSetu's appointment-booking service: a YAML-defined CI build with automated tests, three-stage promotion through dev, staging, and production using Azure DevOps Environments with enforced manual approvals, secrets sourced from Key Vault instead of hardcoded in YAML, shared templates so five other microservices reuse the same battle-tested build and deploy logic, and an automated slot-swap rollback that reacts to failed health checks without waiting for a human.

> **Roshan** _Junior Release Engineer_
>
> The zip-and-drag-into-the-portal deploy is gone. Every release is a pipeline run with a full log, and production literally will not deploy without Karthik or Priya clicking approve in Azure DevOps. That's not a policy anymore, it's enforced.

> **Priya** _Senior DevOps Engineer_
>
> And the rollback saved us for real two weeks ago — a config typo made the new version fail its health check right after the slot swap, and production was back on the old version in under a minute, automatically, before a single patient noticed.

> **Karthik** _Engineering Manager_
>
> The templates are what make this scale. Doctor-directory and billing didn't need a new pipeline written from scratch — they pulled in the same build template we wrote here, in an afternoon each.

> **Next: Infrastructure as Code for MediSetu's Azure footprint**
>
> - Bicep or Terraform to provision the App Service plans, Key Vaults, and deployment slots themselves as versioned code, not manual portal clicks.
> - Azure Monitor and Application Insights integration so the health-check gate in Phase 6 becomes a richer, metric-driven canary check.
> - Multi-region App Service deployment for disaster recovery, using the same Environments-and-approvals pattern across two Azure regions.
