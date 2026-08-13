# 🚀 Spinnaker Project Mastery

> **Hey Fresher — Read This First!**
>
> Spinnaker is an open-source continuous delivery platform built by Netflix and Google for releasing software changes with high velocity and confidence, across any cloud provider. Instead of writing a pipeline script per cloud, you describe applications, clusters, and deployment strategies once, and Spinnaker figures out how to talk to AWS, GCP, Azure, or Kubernetes underneath. It also ships with built-in deployment strategies — red/black, rolling, canary — so "how do I release safely" stops being a question every team answers differently.
>
> **FleetKargo Logistics** runs real-time fleet tracking and last-mile delivery routing for over 40,000 delivery riders across India and Southeast Asia. Their dispatch engine runs on AWS in Mumbai, but a new regulatory requirement means Southeast Asian rider data must also be served from GCP Singapore. You are joining FleetKargo as a Cloud Delivery Engineer. Your first week's task: stop the two-region, two-cloud deployment process from being "two engineers manually running two separate scripts and hoping they finish around the same time," and turn it into one Spinnaker pipeline that deploys safely to both.

#### What You Will Learn and Build in This Project

You will install and configure Spinnaker with Halyard, connect it to Kubernetes and AWS accounts, build multi-stage delivery pipelines with real triggers, implement red/black and canary deployment strategies, wire up Kayenta for automated canary analysis, deploy the same artifact simultaneously to AWS and GCP, and manage pipelines as version-controlled configuration with RBAC-gated production stages.

Halyard, Spinnaker Applications, Pipelines and Stages, Triggers, Deployment Strategies, Red/Black Deployment, Canary Analysis with Kayenta, Multi-Cloud Deployment, Manual Judgment Stages, Pipeline as Code, RBAC and Notifications

> **📦 Phase 1 — Installing and Configuring Spinnaker**
>
> Stand up Spinnaker with Halyard and register the Kubernetes and AWS cloud providers FleetKargo actually runs on.

> **📦 Phase 2 — Applications, Pipelines and Triggers**
>
> Model FleetKargo's dispatch-engine service as a Spinnaker Application and build a pipeline triggered automatically from a Docker registry push.

> **📦 Phase 3 — Deployment Strategies**
>
> Compare Highlander, Red/Black, and Rolling strategies and apply Red/Black to the dispatch-engine's production cluster.

> **📦 Phase 4 — Automated Canary Analysis with Kayenta**
>
> Add a canary stage that compares new-version metrics against baseline in real time and automatically fails the pipeline on regressions.

> **📦 Phase 5 — Multi-Cloud Deployment to AWS and GCP**
>
> Deploy the same artifact to an AWS EKS cluster in Mumbai and a GCP GKE cluster in Singapore from a single pipeline run.

> **📦 Phase 6 — Pipeline as Code, RBAC and Notifications**
>
> Move pipeline definitions into Git-managed JSON, lock production stages behind approval roles, and notify the on-call channel on every stage transition.

**Scene 1 — FleetKargo Logistics, Bengaluru | Two Clouds, One Broken Release Process**

> **Tanvi** _Junior Cloud Delivery Engineer_
>
> I looked at our release runbook for the dispatch engine. It says "SSH into the AWS deploy box, run deploy-mumbai.sh, wait for green health checks, then SSH into the GCP box and run deploy-singapore.sh separately." Last Tuesday someone ran only the AWS script and Singapore riders got stale ETAs for six hours.

> **Priya** _Senior DevOps Engineer_
>
> That is exactly the failure mode Spinnaker exists to remove. One pipeline, multiple deploy stages, each targeting a different cloud provider account, running with the same artifact version. If Mumbai deploys, Singapore deploys too — same run, same audit trail, no forgetting a step.

> **Karthik** _Principal Cloud Architect_
>
> And it is not just "deploy to two places." I want a canary stage in front of production. Our dispatch engine calculates rider ETAs — if a bad release quietly makes ETAs 20% less accurate, customers churn before anyone notices a crash. We need automated analysis, not just "did the pod start."

> **Tanvi**
>
> So we need: one pipeline, two cloud targets, a safety net that catches accuracy regressions before full rollout, and something that stops me from ever running a manual shell script again.

> **Priya**
>
> Exactly. Let's build it phase by phase.

### 1. Phase 1 — Installing and Configuring Spinnaker

**Business Problem:** FleetKargo's platform team has no Spinnaker installation yet. Before any pipeline can exist, Spinnaker itself must be deployed and told which cloud accounts it is allowed to manage — the AWS account running EKS in Mumbai and the GCP project running GKE in Singapore.

#### 1.1 Installing Halyard

```bash
# Halyard is the CLI used to configure, install, and update Spinnaker
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh

# Verify Halyard is installed and healthy
hal -v
hal config
```

**Halyard** is not Spinnaker itself — it is the installer and configuration manager for Spinnaker's many microservices (Deck, Gate, Orca, Clouddriver, Front50, Igor, Echo, Rosco). `InstallHalyard.sh` provisions Halyard on a dedicated VM or your workstation. `hal config` prints the current configuration tree, which starts empty until you add providers.

#### 1.2 Registering the Kubernetes (EKS) Provider

```bash
# Point hal at the kubeconfig context for the Mumbai EKS cluster
hal config provider kubernetes enable

hal config provider kubernetes account add fleetkargo-mumbai-eks \
  --context arn:aws:eks:ap-south-1:483920175640:cluster/fleetkargo-prod \
  --docker-registries fleetkargo-ecr

hal config features edit --artifacts true
```

> **📖 What each line does**
> `hal config provider kubernetes enable` turns on the Kubernetes provider globally for this Spinnaker installation. `account add fleetkargo-mumbai-eks` registers a named Kubernetes account — Spinnaker pipelines reference this name, never raw kubeconfig paths, so the underlying cluster can change without breaking pipelines. `--context` points to the kubeconfig context that has credentials for the EKS cluster. `--docker-registries` links a previously configured Docker registry account so Spinnaker can resolve image tags when baking manifests. Enabling `artifacts` lets pipeline stages reference Git files, S3 objects, or Helm charts as first-class artifacts instead of hardcoded strings.

#### 1.3 Registering the AWS Provider

```bash
hal config provider aws enable

hal config provider aws account add fleetkargo-aws-prod \
  --account-id 483920175640 \
  --assume-role role/spinnakerManaged \
  --regions ap-south-1
```

> **AWS account registration** — `--assume-role role/spinnakerManaged` is the IAM role Spinnaker's Clouddriver microservice assumes to describe and mutate resources (ASGs, ALBs, security groups) in that account. Using assume-role instead of static access keys means credentials rotate automatically and nothing sensitive lives in Halyard's config files. `--regions ap-south-1` scopes discovery to Mumbai only, keeping API calls (and AWS bills) down.

#### 1.4 Deploying and Applying Configuration

```bash
hal config deploy edit --type distributed --account-name fleetkargo-mumbai-eks

hal deploy apply
hal deploy connect
```

> **📖 Deploying Spinnaker itself**
> `deploy edit --type distributed` tells Halyard to install Spinnaker as a set of Kubernetes microservices inside the fleetkargo-mumbai-eks cluster rather than as a single monolithic VM — this is how production Spinnaker installs scale each component (Orca for orchestration, Clouddriver for cloud calls) independently. `hal deploy apply` reads the entire accumulated config and reconciles the live installation to match it — this is the command you re-run after every `hal config` change. `hal deploy connect` port-forwards Deck (the UI) and Gate (the API) to your local machine for the first login.

**Local vs Distributed Halyard Install**

- **Localdebian** — installs all Spinnaker services on a single VM; fine for a lab or proof-of-concept, but a single point of failure and hard to scale.
- **Distributed (Kubernetes)** — installs each Spinnaker microservice as its own deployment inside a Kubernetes cluster; this is what FleetKargo uses in Mumbai because it can scale Orca (pipeline execution) independently from Clouddriver (cloud API calls) as pipeline volume grows.

> **Key takeaways**
> - Halyard is a config-management CLI, not Spinnaker itself — `hal deploy apply` is what actually reconciles the live install.
> - Cloud provider accounts are named abstractions (`fleetkargo-mumbai-eks`) — pipelines reference the name, never raw credentials or contexts.
> - IAM assume-role (AWS) or workload identity (GCP) is preferred over static keys for Clouddriver's cloud access.
> - A distributed install lets each Spinnaker microservice scale independently as pipeline and account count grows.

### 2. Phase 2 — Applications, Pipelines and Triggers

**Business Problem:** FleetKargo's dispatch-engine team pushes a new Docker image to ECR on every merge to `main`. Right now nothing happens automatically — someone has to notice the new tag and kick off a deploy by hand, which is slow and inconsistent.

**Scene 2 — Standup, Wednesday Morning**

> **Tanvi** _Junior Cloud Delivery Engineer_
>
> Karthik, if I register the dispatch-engine repo as a Spinnaker Application, does that mean pipelines just appear?

> **Karthik** _Principal Cloud Architect_
>
> No — the Application is just the namespace that groups clusters, load balancers, and pipelines under one name in the UI and in permissions. You still define the pipeline: what triggers it, what stages it runs, in what order.

#### 2.1 Creating the Application

```bash
hal config application add dispatch-engine \
  --owner-email platform-team@fleetkargo.io \
  --cloud-providers kubernetes,aws
```

> **What an Application is** — In Spinnaker, an Application is a logical container: it groups every cluster, server group, load balancer, and pipeline that belongs to the dispatch-engine service, across both the Kubernetes and AWS providers. Permissions (who can trigger production pipelines) are typically scoped at the Application level, which is why `--owner-email` matters — it is the contact of record shown in the UI.

#### 2.2 Defining the Pipeline with a Docker Trigger

```json
{
  "name": "dispatch-engine-release",
  "application": "dispatch-engine",
  "triggers": [
    {
      "type": "docker",
      "account": "fleetkargo-ecr",
      "organization": "fleetkargo",
      "repository": "dispatch-engine",
      "tag": "^v[0-9]+\\.[0-9]+\\.[0-9]+$",
      "enabled": true
    }
  ],
  "stages": [
    {
      "name": "Deploy to Staging",
      "type": "deployManifest",
      "account": "fleetkargo-mumbai-eks",
      "cloudProvider": "kubernetes",
      "manifests": ["${trigger.artifacts[0]}"],
      "namespace": "staging"
    },
    {
      "name": "Run Smoke Tests",
      "type": "wait",
      "waitTime": 30,
      "requisiteStageRefIds": ["1"]
    }
  ]
}
```

> **📖 Reading the trigger and stages**
> `"type": "docker"` means this pipeline fires automatically whenever a new image tag lands in the `fleetkargo-ecr` registry for the `dispatch-engine` repository. The `tag` field is a regex — here it only fires on semantic-version tags like `v2.4.1`, ignoring `latest` or dev-branch tags, so accidental pushes cannot trigger a production pipeline chain. `deployManifest` is the Kubernetes-provider stage type that applies a manifest (Deployment, Service) to a named account/namespace; `${trigger.artifacts[0]}` interpolates the image reference that fired the trigger directly into the manifest, so the pipeline always deploys exactly the image that was pushed, never a stale tag. `requisiteStageRefIds` wires stage ordering — the smoke-test wait stage only starts after the staging deploy stage completes.

**Quiz: Why does the Docker trigger use a regex like `^v[0-9]+\.[0-9]+\.[0-9]+$` instead of matching every pushed tag?**
- To make the pipeline run faster
- To ensure only intentional, semantically versioned release images trigger a deployment, not every dev or debug tag pushed to the registry
- Because Spinnaker requires all Docker tags to be regex-validated
- To reduce the Docker image size

> **Answer/explanation:** The correct answer is the second option. Without a tag filter, every push to the registry — including throwaway debug builds or feature-branch tags — would trigger a full deployment pipeline, which is both wasteful and dangerous for a production service like dispatch-engine. The regex ensures only tags matching the team's release convention (`v2.4.1`) start a pipeline run. It has nothing to do with pipeline speed or image size, and Spinnaker does not mandate regex tag validation globally — it is a per-trigger configuration choice the team makes deliberately.

### 3. Phase 3 — Deployment Strategies

**Business Problem:** Staging deploys are low-risk, but pushing a new dispatch-engine version straight to production, replacing every pod at once, means a bad release affects 100% of riders instantly with no fast rollback path.

#### 3.1 Comparing Spinnaker's Built-In Strategies

**Deployment Strategy Comparison**

- **Highlander** — deploys a new server group and immediately destroys all previous ones; fastest and cheapest, but zero rollback safety net — never used for dispatch-engine production.
- **Red/Black (Blue/Green)** — deploys a new server group ("black") alongside the old one ("red"), shifts traffic over, and keeps the old one disabled (not deleted) for instant rollback; FleetKargo uses this for production because a bad ETA-calculation bug needs a one-click revert, not a redeploy.
- **Rolling** — replaces instances of the existing server group incrementally, in place; good for stateless services with no need to keep the old version standing by, but rollback means rolling forward again with the previous image.
- **Canary** — routes a small percentage of production traffic to the new version, compares metrics against baseline, and only proceeds if metrics are healthy; FleetKargo layers this in front of Red/Black in Phase 4.

#### 3.2 Configuring Red/Black in the Pipeline

```json
{
  "name": "Deploy to Production (Red/Black)",
  "type": "deployManifest",
  "account": "fleetkargo-mumbai-eks",
  "cloudProvider": "kubernetes",
  "namespace": "production",
  "manifests": ["${trigger.artifacts[0]}"],
  "trafficManagement": {
    "enabled": true,
    "options": {
      "strategy": "redblack",
      "maxServerGroups": 2,
      "enableTraffic": true
    }
  },
  "requisiteStageRefIds": ["2"]
}
```

> **📖 What the traffic management block does**
> `"strategy": "redblack"` tells Spinnaker's Kubernetes provider to deploy the new manifest as a fresh, separate server group rather than mutating the existing one in place. `"maxServerGroups": 2` caps how many old versions stay around disabled but not deleted — Spinnaker automatically disables (not deletes) the previous server group once the new one is healthy, so a rollback is "re-enable the old one," typically under 30 seconds. `"enableTraffic": true` means the new server group starts receiving traffic as soon as its pods pass readiness checks; pairing this with a manual judgment stage (below) lets a human hold that switch until the canary stage says it is safe.

#### 3.3 Adding a Manual Judgment Gate Before Traffic Cutover

```json
{
  "name": "Approve Production Cutover",
  "type": "manualJudgment",
  "instructions": "Canary analysis passed. Approve traffic cutover to the new dispatch-engine version in ap-south-1 production.",
  "notifications": [
    {
      "type": "slack",
      "address": "#dispatch-engine-releases",
      "level": "manualJudgment"
    }
  ],
  "requisiteStageRefIds": ["3"]
}
```

> **manualJudgment stage** pauses pipeline execution and waits for a human with sufficient permissions to click Continue or Stop in the Spinnaker UI (or respond via Slack integration). It is placed after canary analysis and before the traffic cutover so that even an automated "canary passed" verdict gets a final human sign-off for FleetKargo's production dispatch engine, given how directly it affects live riders.

> **Key takeaways**
> - Highlander is fast but has no rollback; Red/Black keeps the previous version standing by disabled for instant rollback; Rolling replaces in place; Canary adds metric-driven gating in front of any of the above.
> - `trafficManagement.strategy: redblack` plus `maxServerGroups` controls how many old versions Spinnaker retains for rollback.
> - `manualJudgment` stages insert a required human approval anywhere in the pipeline — commonly right before a production traffic cutover.

### 4. Phase 4 — Automated Canary Analysis with Kayenta

**Business Problem:** A bad dispatch-engine release does not always crash — sometimes it just quietly makes rider ETAs less accurate or API latency creep up. Karthik wants the pipeline itself to detect that kind of regression before 100% of production traffic hits the new version, without waiting for a human to notice a dashboard.

#### 4.1 Configuring Kayenta with a Metrics Source

```bash
hal config canary enable
hal config canary edit --default-metrics-account fleetkargo-prometheus
hal config canary prometheus enable
hal config canary prometheus account add fleetkargo-prometheus \
  --base-url http://prometheus.monitoring.svc.cluster.local:9090
```

> **📖 Kayenta setup**
> Kayenta is Spinnaker's automated canary analysis engine. `hal config canary enable` turns the feature on for the installation. `canary prometheus account add` registers FleetKargo's in-cluster Prometheus as the source Kayenta queries for both the baseline (old version) and canary (new version) time series — CPU, latency, error rate, and a custom `eta_accuracy_percent` metric the dispatch-engine team exposes.

#### 4.2 Defining the Canary Configuration

```json
{
  "name": "dispatch-engine-canary-config",
  "metrics": [
    {
      "name": "eta-accuracy",
      "query": "avg(eta_accuracy_percent{service=\"dispatch-engine\"})",
      "groups": ["accuracy"],
      "analysisConfigurations": {
        "canary": { "direction": "increase" }
      }
    },
    {
      "name": "p99-latency-ms",
      "query": "histogram_quantile(0.99, rate(http_request_duration_ms_bucket{service=\"dispatch-engine\"}[2m]))",
      "groups": ["latency"],
      "analysisConfigurations": {
        "canary": { "direction": "decrease" }
      }
    }
  ],
  "classifier": {
    "groupWeights": { "accuracy": 60, "latency": 40 }
  }
}
```

> **📖 What the canary config measures**
> Each metric entry pairs a human-readable `name` with the actual query run against Prometheus, once against the canary pods and once against baseline pods. `direction: "increase"` on eta-accuracy tells Kayenta that a *drop* in this metric on the canary side is bad — the opposite of latency, where `direction: "decrease"` flags an *increase* in p99 latency as bad. `groupWeights` lets FleetKargo say accuracy matters more than raw latency for this service — a 60/40 split means an accuracy regression drags the overall canary score down harder than a latency blip.

#### 4.3 Wiring the Canary Stage into the Pipeline

```json
{
  "name": "Kayenta Canary Analysis",
  "type": "kayentaCanary",
  "canaryConfig": {
    "canaryConfigId": "dispatch-engine-canary-config",
    "scopes": [
      {
        "controlScope": "dispatch-engine-baseline",
        "experimentScope": "dispatch-engine-canary",
        "scopeName": "default"
      }
    ],
    "scoreThresholds": { "pass": 75, "marginal": 50 },
    "lifetimeDuration": "PT30M"
  },
  "requisiteStageRefIds": ["2"]
}
```

> **scoreThresholds and lifetimeDuration** — Kayenta runs the analysis for `PT30M` (30 minutes, ISO-8601 duration format), repeatedly comparing canary vs. baseline metrics. If the aggregate score is 75 or above, the stage passes and the pipeline proceeds. Between 50 and 75 is `marginal`, which FleetKargo configured to still require manual judgment. Below 50, the stage fails automatically and the pipeline halts — no human even needs to be paged before the bad version is prevented from reaching 100% of riders.

**Quiz: In the FleetKargo canary config, `eta-accuracy` uses `"direction": "increase"` while `p99-latency-ms` uses `"direction": "decrease"`. What does this configuration actually control?**
- Whether the metric value itself goes up or down over time
- Which direction of change in the canary, relative to baseline, Kayenta should treat as a regression
- The direction traffic is routed during the canary phase
- Whether Prometheus scrapes the metric more or less frequently

> **Answer/explanation:** The correct answer is the second option. The `direction` field tells Kayenta which direction of change counts as bad for that specific metric. For eta-accuracy, a value that goes down (decreases) on the canary compared to baseline is the regression Kayenta should flag — so it's configured as "increase" being the good/expected direction, and any decrease is penalized. For latency, an increase in p99 latency is the regression, so "decrease" is the good direction. It has no bearing on traffic routing direction or Prometheus scrape intervals — those are unrelated concerns configured elsewhere.

### 5. Phase 5 — Multi-Cloud Deployment to AWS and GCP

**Business Problem:** The regulatory requirement is explicit: Southeast Asian rider data has to be served from infrastructure inside the region, on GCP Singapore, while the existing AWS Mumbai deployment continues serving the Indian market. Both need the same dispatch-engine version deployed from the same pipeline run, not two separately triggered processes that can drift out of sync.

#### 5.1 Registering the GCP Provider

```bash
hal config provider kubernetes account add fleetkargo-singapore-gke \
  --context gke_fleetkargo-sea_asia-southeast1_fleetkargo-prod \
  --docker-registries fleetkargo-gcr \
  --provider-version v2
```

> **Second Kubernetes account, second region** — Spinnaker does not care that this cluster lives on a different cloud than `fleetkargo-mumbai-eks`; from a pipeline's perspective, both are just named Kubernetes accounts. `--provider-version v2` uses Spinnaker's newer manifest-based Kubernetes integration (as opposed to the older v1 provider), which is what supports artifact substitution and traffic management strategies used earlier in this project.

#### 5.2 Fan-Out Deployment Stages

```json
[
  {
    "refId": "4a",
    "name": "Deploy to Mumbai (AWS EKS)",
    "type": "deployManifest",
    "account": "fleetkargo-mumbai-eks",
    "cloudProvider": "kubernetes",
    "namespace": "production",
    "manifests": ["${trigger.artifacts[0]}"],
    "requisiteStageRefIds": ["3"]
  },
  {
    "refId": "4b",
    "name": "Deploy to Singapore (GCP GKE)",
    "type": "deployManifest",
    "account": "fleetkargo-singapore-gke",
    "cloudProvider": "kubernetes",
    "namespace": "production",
    "manifests": ["${trigger.artifacts[0]}"],
    "requisiteStageRefIds": ["3"]
  }
]
```

> **📖 Fan-out, not sequential, deployment**
> Both stages list `"requisiteStageRefIds": ["3"]` — the same upstream stage — instead of pointing at each other. That means once the canary/approval stage (refId 3) completes, Orca (Spinnaker's orchestration engine) runs both deploy stages in parallel, not one after another. This is the structural fix for FleetKargo's original bug: it is now architecturally impossible to deploy to Mumbai without also deploying the same artifact version to Singapore in the same pipeline execution, because they are two branches of one graph, not two separate scripts.

#### 5.3 Verifying Both Regions Received the Same Version

```bash
kubectl --context fleetkargo-mumbai-eks -n production \
  get deploy dispatch-engine -o jsonpath='{.spec.template.spec.containers[0].image}'

kubectl --context fleetkargo-singapore-gke -n production \
  get deploy dispatch-engine -o jsonpath='{.spec.template.spec.containers[0].image}'

# Both should print:
# 483920175640.dkr.ecr.ap-south-1.amazonaws.com/fleetkargo/dispatch-engine:v2.4.1
```

**Fan-Out Parallel Stages vs. Sequential Region Chaining**

- **Parallel (fan-out) deploys** — both regional deploy stages share the same `requisiteStageRefIds`; they start together, and a failure in one does not block the other from completing (though the pipeline overall reports failed). This is what FleetKargo uses, since Mumbai and Singapore are independent regions with no ordering dependency.
- **Sequential (chained) deploys** — Singapore's stage lists Mumbai's stage as its prerequisite, so it only starts after Mumbai fully succeeds; useful when you want a canary region to prove out a release before a second region receives it, but adds latency to full rollout.

### 6. Phase 6 — Pipeline as Code, RBAC and Notifications

**Business Problem:** The pipeline currently lives only inside the Spinnaker UI's database. Nobody can code-review a pipeline change, and any engineer with UI access can currently edit the production dispatch-engine pipeline — including interns still ramping up.

#### 6.1 Exporting the Pipeline as JSON (Pipeline as Code)

```bash
# Export the existing UI-built pipeline to a JSON file tracked in Git
spin pipeline get --application dispatch-engine --name dispatch-engine-release \
  > pipelines/dispatch-engine-release.json

# After editing pipelines/dispatch-engine-release.json in a PR and merging:
spin pipeline save --file pipelines/dispatch-engine-release.json
```

> **`spin` CLI** — the `spin` command-line tool talks to Spinnaker's Gate API the same way the UI does. `pipeline get` dumps the current pipeline definition as JSON so it can be committed to the `fleetkargo/spinnaker-pipelines` Git repo. `pipeline save` pushes an edited JSON file back to Spinnaker, applying the change. This turns pipeline changes into a normal pull-request workflow — reviewed, diffable, revertable via `git revert` — instead of untracked UI clicks.

#### 6.2 Locking Production Stages with RBAC

```yaml
# application permissions, applied via hal or the Application config UI
permissions:
  READ:
    - fleetkargo-platform-team
    - fleetkargo-oncall
  WRITE:
    - fleetkargo-platform-leads
  EXECUTE:
    - fleetkargo-platform-team
```

> **📖 Application-level permission model**
> `READ` lets anyone on the platform team or on-call rotation view the pipeline and its execution history. `WRITE` — the ability to edit pipeline definitions — is restricted to `fleetkargo-platform-leads` only, so junior engineers like Tanvi cannot accidentally (or unilaterally) change what production deploys do. `EXECUTE` lets the broader platform team trigger runs or approve `manualJudgment` stages without needing WRITE access, separating "who can run this" from "who can change what it does."

#### 6.3 Slack Notifications on Stage Events

```json
{
  "notifications": [
    {
      "type": "slack",
      "address": "#dispatch-engine-releases",
      "level": "pipeline",
      "when": ["pipeline.starting", "pipeline.complete", "pipeline.failed"]
    },
    {
      "type": "slack",
      "address": "#dispatch-engine-oncall",
      "level": "stage",
      "when": ["stage.failed"]
    }
  ]
}
```

> **Two notification scopes** — the `pipeline`-level block posts to a general releases channel whenever the whole pipeline starts, completes, or fails, giving the team a running feed of releases. The `stage`-level block only fires on `stage.failed` and posts to the on-call channel specifically — so a failed canary or a failed Singapore deploy pages the right people immediately, rather than getting lost in the general chatter of every pipeline start.

> **Key takeaways**
> - `spin pipeline get` / `spin pipeline save` turn pipeline definitions into Git-reviewable JSON — "pipeline as code."
> - RBAC separates READ (view), WRITE (edit pipeline definition), and EXECUTE (trigger/approve runs) at the Application level.
> - Notification blocks can be scoped at the pipeline level (general awareness) or stage level (targeted paging on failure).

**Quiz: Why does FleetKargo give `fleetkargo-platform-team` EXECUTE permission but not WRITE permission on the dispatch-engine pipeline?**
- EXECUTE and WRITE are the same permission under a different name
- So the team can trigger runs and approve manual judgment stages without being able to change what the pipeline actually does
- Because Spinnaker requires EXECUTE before WRITE can be granted
- To reduce the number of Slack notifications sent

> **Answer/explanation:** The correct answer is the second option. Separating EXECUTE from WRITE lets FleetKargo give the whole platform team the ability to run pipelines and approve manual judgment gates — needed for day-to-day on-call work — while keeping the ability to actually modify pipeline logic (stages, strategies, canary thresholds) restricted to leads who review changes via pull request. They are distinct permissions, not aliases, and there is no such prerequisite ordering, nor any relationship to notification volume.

##### 🏋️ Hands-On Exercises — Extend the Project

1. Add a third `deployManifest` stage targeting a new `fleetkargo-jakarta-aks` Azure AKS account, so the dispatch-engine pipeline fans out to three clouds instead of two, and confirm all three regional deploys share the same `requisiteStageRefIds`.
2. Write a second Kayenta metric for `error_rate_percent` with `direction: "decrease"`, add it to a new `reliability` group, and rebalance `groupWeights` across accuracy, latency, and reliability.
3. Convert the Red/Black production deploy stage to a Canary (traffic-percentage) strategy instead, routing 10% of production traffic to the new version for the first 15 minutes before Kayenta analysis runs.
4. Export the current dispatch-engine-release pipeline with `spin pipeline get`, modify the `lifetimeDuration` of the canary stage from `PT30M` to `PT15M` in the JSON, and reapply it with `spin pipeline save`.
5. Add a `manualJudgment` stage before the Singapore GKE deploy only (not Mumbai), simulating a requirement that Southeast Asia releases need separate compliance sign-off, and verify Mumbai's deploy stage is unaffected by the added gate.

### Spinnaker Project Complete 🎉

You have built a real multi-cloud continuous delivery pipeline for FleetKargo's dispatch engine: a Halyard-managed Spinnaker install spanning AWS EKS and GCP GKE, a Docker-registry-triggered pipeline, Red/Black production deployment with instant rollback, automated Kayenta canary analysis gating rider-ETA accuracy and latency, parallel fan-out deployment to two regions from one execution, and Git-reviewable pipeline definitions locked down with RBAC.

> **Tanvi** _Junior Cloud Delivery Engineer_
>
> The old runbook is gone. Now a merge to main pushes an image, the pipeline bakes through staging, canary analysis runs against real Prometheus metrics, and if it passes, Mumbai and Singapore both get the new version in the same execution. I don't have to remember two steps anymore — the graph remembers for me.

> **Priya** _Senior DevOps Engineer_
>
> And when the canary caught that ETA-accuracy regression last month before it hit 100% of riders — that's the pipeline doing its job. No page, no incident, just a failed stage and a Slack message.

> **Karthik** _Principal Cloud Architect_
>
> Next quarter we add Azure for our Jakarta expansion. Because we built this as named accounts and parallel stages instead of hardcoded scripts, that's a new `deployManifest` block, not a rewrite.

> **Next: Deep observability for multi-cloud fleets**
>
> - Prometheus federation across AWS, GCP, and Azure clusters to power a single Kayenta canary config for all three.
> - Distributed tracing (Jaeger) to see a rider's ETA request cross regional boundaries during Singapore's canary phase.
> - Spinnaker Managed Delivery (deliverable configs) to formalize environment promotion from staging → canary → production as declarative YAML.
