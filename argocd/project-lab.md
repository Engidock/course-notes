# 🔄 ArgoCD Project Mastery

> **Hey Fresher — Read This First!**
>
> ArgoCD is a GitOps continuous delivery tool for Kubernetes — instead of engineers running `kubectl apply` by hand, you declare the desired state of your cluster as manifests in a Git repository, and ArgoCD continuously watches that repo, comparing it to what's actually running in the cluster. If they match, the Application shows "Synced." If they don't — because someone pushed a change to Git, or because someone manually edited the cluster — ArgoCD shows the drift and can automatically reconcile it, either direction.
>
> **ShelfKart** is a fast-growing e-commerce marketplace serving tier-2 Indian cities, and its checkout-service, inventory-service, and eleven other microservices are deployed to a shared Kubernetes cluster. Right now, deploys happen when an engineer runs `kubectl apply -f` from their laptop against whatever kubeconfig context they happen to have active — nobody can say with confidence what's actually running in production versus what's in Git. You're joining as a Platform Engineer during Diwali sale prep, the highest-traffic period of ShelfKart's year, and your task is to make Git the only way anything reaches the cluster, with automatic drift detection and one-command rollback, before the sale starts.

#### What You Will Learn and Build in This Project

You will install ArgoCD, connect it to a Git repository of Kubernetes manifests, create Applications that continuously sync cluster state to Git, detect and self-heal configuration drift, manage multiple environments with Kustomize overlays, roll back broken deployments through Git history, scale to many microservices with the App of Apps pattern, and lock down who can sync what with ArgoCD Projects and RBAC.

ArgoCD Installation, Applications and the Sync Loop, Git as Source of Truth, Drift Detection and Self-Healing, Sync Policies and Sync Waves, Kustomize Overlays for Multi-Environment, Rollback via Git Revert, App of Apps Pattern, ArgoCD Projects and RBAC, Notifications

> **📦 Phase 1 — Installing ArgoCD and Connecting Git**
>
> Install ArgoCD into the cluster and register the Git repository holding ShelfKart's Kubernetes manifests.

> **📦 Phase 2 — Applications and the Sync Loop**
>
> Create the checkout-service Application, understand Synced vs. Healthy, and perform the first Git-driven deploy.

> **📦 Phase 3 — Drift Detection and Self-Healing**
>
> Manually edit the cluster to simulate a rogue `kubectl apply`, watch ArgoCD detect it, and enable automated self-heal.

> **📦 Phase 4 — Multi-Environment with Kustomize Overlays**
>
> Manage staging and production checkout-service configuration from one base manifest set using Kustomize overlays.

> **📦 Phase 5 — App of Apps for ShelfKart's Microservices**
>
> Scale from one manually created Application to all thirteen ShelfKart services, managed declaratively from a single root Application.

> **📦 Phase 6 — Projects, RBAC and Notifications**
>
> Scope which teams can sync which services with ArgoCD Projects, and wire Slack notifications for sync failures during the Diwali sale window.

**Scene 1 — ShelfKart, Bengaluru | Three Weeks Before Diwali Sale**

> **Vikram** _Junior Platform Engineer_
>
> I asked five engineers what's actually running in production for checkout-service right now, and got five different answers. Two of them ran `kubectl apply` from stale local branches last week.

> **Priya** _Senior DevOps Engineer_
>
> That's the exact failure mode GitOps is built to eliminate. If Git is the only path to the cluster, then "what's running in production" and "what's the latest commit on main" become the same question, always.

> **Amit** _Platform Architect_
>
> During Diwali sale, checkout-service will handle 40x normal traffic. If someone manually patches a resource limit under pressure and forgets to commit it, and then a Git-driven deploy silently reverts that emergency fix an hour later because nobody knew about the drift — that's exactly the kind of incident we cannot afford that week.

> **Vikram**
>
> So ArgoCD needs to not just deploy from Git, it needs to actively tell us the moment cluster state and Git state disagree, in either direction.

> **Priya**
>
> Exactly — and by the time Diwali sale starts, every one of our thirteen services needs to be under GitOps, not just checkout-service.

### 1. Phase 1 — Installing ArgoCD and Connecting Git

**Business Problem:** ArgoCD isn't running anywhere yet, and ShelfKart's Kubernetes manifests are scattered across engineers' local clones with no canonical Git repository ArgoCD — or anyone else — can treat as the source of truth.

#### 1.1 Installing ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd get pods
```

```
NAME                                               READY   STATUS    RESTARTS
argocd-application-controller-0                    1/1     Running   0
argocd-repo-server-6f9d8b7c4-x2k9p                 1/1     Running   0
argocd-server-7c5d9f6b8d-m4n7q                      1/1     Running   0
argocd-dex-server-8b6c7d9f5-p8q2r                   1/1     Running   0
argocd-redis-5f7c8d9b6c-r3s5t                       1/1     Running   0
```

> **📖 What each component does**
> `argocd-application-controller` is the core reconciliation loop — it continuously compares live cluster state to the desired state in Git and drives sync operations. `argocd-repo-server` clones and caches Git repositories, and renders manifests (plain YAML, Kustomize, or Helm) into their final form. `argocd-server` is the API and UI backend. `argocd-dex-server` handles SSO integration (used in Phase 6 for RBAC). `argocd-redis` caches repository and manifest state to keep reconciliation fast across many Applications — important once ShelfKart is running thirteen of them.

#### 1.2 Accessing the UI and CLI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443

argocd admin initial-password -n argocd

argocd login localhost:8080 --username admin --insecure
```

> **`argocd` CLI vs. UI** — everything doable in the ArgoCD web UI is also scriptable through the `argocd` CLI, which is what ShelfKart's platform team standardizes on for anything that should be repeatable or automatable (like Application creation in Phase 2), reserving the UI mainly for visual sync-status review and troubleshooting during the Diwali sale window.

#### 1.3 Registering the Git Repository

```bash
argocd repo add https://github.com/shelfkart/k8s-manifests.git \
  --username shelfkart-argocd-bot \
  --password ${GIT_TOKEN}
```

```
Repository 'https://github.com/shelfkart/k8s-manifests.git' added
```

> Registering the repository with a scoped bot account's personal access token, rather than a personal credential, means the connection survives regardless of which human engineer set it up, and the bot account's permissions can be limited to read-only access on this specific repo.

> **Key takeaways**
> - `argocd-application-controller` is the actual reconciliation loop; `argocd-repo-server` renders manifests from Git; `argocd-server` serves the API and UI.
> - The `argocd` CLI mirrors UI functionality and is the preferred path for anything meant to be scripted or repeatable.
> - Repositories should be registered with scoped bot credentials, not personal developer accounts.

### 2. Phase 2 — Applications and the Sync Loop

**Business Problem:** ArgoCD is installed and can see the Git repository, but nothing is actually deployed yet. checkout-service needs its first real ArgoCD Application, and the team needs to understand exactly what "Synced" and "Healthy" mean before trusting this for production traffic.

#### 2.1 Creating the checkout-service Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: checkout-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shelfkart/k8s-manifests.git
    targetRevision: main
    path: apps/checkout-service/base
  destination:
    server: https://kubernetes.default.svc
    namespace: checkout
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f checkout-service-app.yaml

argocd app get checkout-service
```

> **📖 Reading the Application spec**
> `source.repoURL` and `source.path` tell ArgoCD exactly which Git repository and directory hold the manifests for this Application — `apps/checkout-service/base` in this case. `targetRevision: main` means ArgoCD tracks the `main` branch's HEAD; it could equally be a tag or a specific commit SHA for stricter control. `destination.server: https://kubernetes.default.svc` targets the same cluster ArgoCD runs in (ArgoCD can also manage remote clusters by registering them separately). `syncOptions: CreateNamespace=true` lets ArgoCD create the `checkout` namespace itself if it doesn't already exist, rather than failing the sync waiting for someone to create it manually.

#### 2.2 Understanding Synced vs. Healthy

```bash
argocd app get checkout-service
```

```
Name:               checkout-service
Sync Status:        OutOfSync
Health Status:      Missing

KIND        NAME               STATUS       HEALTH
Deployment  checkout-service   OutOfSync    Missing
Service     checkout-service   OutOfSync    Missing
```

```bash
argocd app sync checkout-service
```

```
Name:               checkout-service
Sync Status:        Synced
Health Status:      Healthy

KIND        NAME               STATUS   HEALTH
Deployment  checkout-service   Synced   Healthy
Service     checkout-service   Synced   Healthy
```

> **📖 Two independent axes**
> **Sync Status** answers "does the live cluster state match Git?" — `OutOfSync` before the first sync simply means nothing exists yet that matches what Git describes. **Health Status** answers a completely different question: "is the resource actually working?" — a Deployment can be perfectly `Synced` (its spec matches Git exactly) while its Health is `Degraded`, because its pods are crash-looping. ArgoCD computes health per resource kind using built-in logic (a Deployment is Healthy when its `status.availableReplicas` matches the desired replica count; a Service with a `LoadBalancer` type is Healthy once it has an external IP assigned), which is why Amit trusts this signal for judging whether a Diwali-sale deploy is actually safe, not just technically applied.

**Quiz: An ArgoCD Application shows "Sync Status: Synced" and "Health Status: Degraded." What does this combination actually mean?**
- ArgoCD failed to apply the manifests from Git to the cluster
- The live cluster resources exactly match what's declared in Git, but the resulting workload isn't functioning correctly — for example, pods are crash-looping even though the Deployment spec is correctly applied
- ArgoCD detected a Git repository connectivity problem
- The Application needs to be manually synced again

> **Answer/explanation:** The correct answer is the second option. Synced and Healthy are independent signals. Synced means the live resource specs match Git — the apply genuinely succeeded. Degraded health means that despite the correct spec being applied, the workload itself isn't running correctly (crash-looping pods, failed readiness probes, etc.) — a problem with the application code or configuration values, not with ArgoCD's ability to apply manifests. It has nothing to do with repository connectivity, and re-syncing wouldn't fix a Degraded health caused by an actual application bug.

> **Key takeaways**
> - Sync Status compares live cluster state to Git; Health Status evaluates whether the resulting resource is actually functioning, using per-resource-kind logic.
> - A resource can be Synced and Degraded simultaneously — a correctly-applied spec doesn't guarantee a working workload.
> - `syncOptions: CreateNamespace=true` lets ArgoCD provision the target namespace itself as part of the sync.

### 3. Phase 3 — Drift Detection and Self-Healing

**Business Problem:** Amit's exact worry from the opening scene — an engineer manually patching the cluster under pressure and the change silently disappearing (or silently persisting when it shouldn't) — needs to be something the team can see and control deliberately, not discover by accident.

#### 3.1 Simulating a Rogue `kubectl edit`

```bash
kubectl -n checkout edit deployment checkout-service
# Manually change replicas: 3 to replicas: 8, save and exit
```

```bash
argocd app get checkout-service
```

```
Sync Status:        OutOfSync
Health Status:      Healthy

KIND        NAME               STATUS       HEALTH
Deployment  checkout-service   OutOfSync    Healthy
```

```bash
argocd app diff checkout-service
```

```diff
===== apps/v1/Deployment checkout/checkout-service ======
 spec:
-  replicas: 3
+  replicas: 8
```

> **📖 What just happened**
> The moment `kubectl edit` changed the live Deployment, ArgoCD's next reconciliation loop (or immediate detection if using resource watch, which the application controller does by default) noticed the live spec no longer matches Git's `replicas: 3`, and flipped Sync Status to `OutOfSync`. `argocd app diff` shows exactly what diverged — this is the tool Vikram now runs the moment he sees an unexpected OutOfSync status, instead of guessing what changed.

#### 3.2 Manual Sync Policy vs. Automated Sync Policy

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

> **📖 What `automated.selfHeal` actually does**
> Without `selfHeal`, ArgoCD detects and reports drift (as seen above) but takes no action — a human has to run `argocd app sync` to reconcile it, in either direction. With `selfHeal: true`, ArgoCD automatically reverts the live cluster back to match Git the moment drift is detected — the manual `replicas: 8` edit above would be reverted back to `3` within seconds, without anyone running a sync command. `prune: true` is a related but distinct setting: it means if a resource is *removed* from Git, ArgoCD deletes it from the cluster too, rather than leaving an orphaned resource behind.

**Automated selfHeal vs. Manual Sync Policy**

- **Manual sync** — ArgoCD reports drift but never acts on its own; a human explicitly runs `argocd app sync`. ShelfKart uses this for any Application still being actively tuned, where an engineer might legitimately want to test a live change before committing it.
- **Automated with selfHeal** — drift is reverted automatically, in near real time, with zero human action required. ShelfKart enables this specifically for checkout-service and inventory-service ahead of the Diwali sale, precisely so an emergency `kubectl patch` under pressure gets flagged loudly (via the notification in Phase 6) rather than silently persisting as untracked cluster state — the fix that actually needs to happen is committing the emergency change to Git, not leaving it as a manual patch.

#### 3.3 Simulating a Broken Deploy and Recovering via Git Revert

```bash
# apps/checkout-service/base/deployment.yaml — a bad commit
# image: shelfkart/checkout-service:v2.9.0  ->  shelfkart/checkout-service:v2.9.0-typo
git commit -am "bump checkout-service to v2.9.0-typo"
git push origin main
```

```bash
argocd app get checkout-service
```

```
Sync Status:        Synced
Health Status:      Degraded

KIND        NAME               STATUS   HEALTH
Deployment  checkout-service   Synced   Degraded
Pod         checkout-service-7f9c-x2k9p  Synced   ImagePullBackOff
```

```bash
git revert HEAD
git push origin main
```

```
Sync Status:        Synced
Health Status:      Healthy
```

> **📖 The full incident-recovery loop**
> Notice the broken deploy shows `Synced` and `Degraded` at the same time — ArgoCD faithfully applied exactly what Git said (a nonexistent image tag), and the resulting pods correctly show `ImagePullBackOff` in their health. This is a live demonstration of the earlier quiz answer: Synced does not mean working. The recovery is deliberately simple — `git revert HEAD` undoes the bad commit, and once pushed, ArgoCD's next reconciliation (automatic, typically within its default 3-minute polling interval, or immediate if using a Git webhook) picks up the reverted manifest and restores the last known-good image tag. There is no `kubectl rollout undo` command involved — rollback in GitOps is a Git operation, and it leaves a clean, auditable trail of exactly what broke and what fixed it.

> **Key takeaways**
> - `argocd app diff` shows exactly what live cluster state diverges from Git, resource by resource.
> - `selfHeal: true` automatically reverts manual cluster drift back to match Git; `prune: true` automatically deletes resources removed from Git.
> - Rollback in a GitOps workflow is a `git revert`, not a `kubectl` command — the fix is committed and auditable, not an untracked manual action.

### 4. Phase 4 — Multi-Environment with Kustomize Overlays

**Business Problem:** checkout-service needs different configuration in staging (1 replica, debug logging, lower resource limits) versus production (8 replicas during Diwali sale prep, resource limits tuned for real traffic) — and duplicating the entire manifest set per environment means every future change has to be made twice, correctly, or staging and production quietly drift apart.

#### 4.1 Repository Structure with Kustomize

```
apps/checkout-service/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── production/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: replica-patch.yaml
images:
  - name: shelfkart/checkout-service
    newTag: v2.9.1
```

```yaml
# overlays/production/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-service
spec:
  replicas: 8
  template:
    spec:
      containers:
        - name: checkout-service
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
```

> **📖 Base and overlays, not copy-paste**
> `base/` contains the shared Deployment and Service definitions common to every environment. Each `overlays/<env>/kustomization.yaml` references the base via `resources: - ../../base` and layers environment-specific `patches` on top — production's overlay bumps `replicas` to 8 and sets real resource requests/limits, while staging's overlay (not shown, but structurally identical) keeps replicas at 1 with smaller limits. `images.newTag` in the production overlay pins the exact image tag for that environment, independent of whatever tag the base or staging overlay references. A fix to the Service definition in `base/service.yaml` automatically applies to every environment; only what's genuinely different between environments needs to be duplicated.

#### 4.2 Two ArgoCD Applications, One Base

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: checkout-service-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shelfkart/k8s-manifests.git
    targetRevision: main
    path: apps/checkout-service/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: checkout-production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

> **`path: apps/checkout-service/overlays/production`** — the Application points directly at the overlay directory, not the base. `argocd-repo-server` detects the `kustomization.yaml` present there and automatically runs the Kustomize build (equivalent to `kustomize build`) to render the final manifests before applying them — no separate build step or CI job is needed to "compile" Kustomize output. A parallel `checkout-service-staging` Application points at `overlays/staging` and targets the `checkout-staging` namespace, with `selfHeal` deliberately left off for staging so engineers can still poke at it live without ArgoCD immediately reverting exploratory changes.

**Kustomize Overlays vs. Separate Helm Values Files**

- **Kustomize overlays** — patch-based, no templating language; you edit real YAML and Kustomize merges patches onto a base. ShelfKart prefers this for straightforward per-environment differences (replica count, resource limits, image tags) because there's no templating syntax to learn and the rendered output is easy to reason about.
- **Helm values files** — templating-based, more powerful for complex conditional logic and reusable charts distributed to others, but requires learning Go template syntax and can make the rendered output harder to predict just from reading the chart. ArgoCD natively supports both; ShelfKart's platform team uses Helm only for third-party charts (like their Redis and cert-manager installs) and Kustomize for their own services.

> **Quiz: Why does ArgoCD not need a separate CI build step to process the Kustomize overlay before deploying checkout-service-production?**
> - Kustomize overlays don't require any processing, they're just plain YAML applied directly
> - `argocd-repo-server` runs the Kustomize build itself when it detects a `kustomization.yaml` in the Application's source path, rendering final manifests at sync time
> - ArgoCD requires all Kustomize processing to happen in GitHub Actions before ArgoCD ever sees the repository
> - Kustomize overlays are only supported through a separate plugin that must be installed manually

> **Answer/explanation:** The correct answer is the second option. `argocd-repo-server` has built-in support for Kustomize (along with plain YAML, Helm, and Jsonnet) — when it clones the repo and finds a `kustomization.yaml` at the Application's configured path, it runs the equivalent of `kustomize build` internally to render the final manifests before they're compared to live cluster state and applied. Overlays absolutely do require processing (patches merged onto a base) — they aren't plain YAML by themselves. No external CI step is required, and Kustomize support is native to ArgoCD, not a separately installed plugin.

### 5. Phase 5 — App of Apps for ShelfKart's Microservices

**Business Problem:** checkout-service is fully under GitOps, but manually running `kubectl apply -f` for a new Application YAML file, one at a time, for the remaining twelve ShelfKart services (inventory-service, cart-service, payments-service, and so on) is exactly the kind of manual, error-prone process this whole project is meant to eliminate.

#### 5.1 The Root Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shelfkart-root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shelfkart/k8s-manifests.git
    targetRevision: main
    path: applications
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```
k8s-manifests/applications/
├── checkout-service-production.yaml
├── checkout-service-staging.yaml
├── inventory-service-production.yaml
├── cart-service-production.yaml
├── payments-service-production.yaml
└── ... (nine more)
```

> **📖 Applications managing Applications**
> The `shelfkart-root` Application's source path, `applications/`, contains ArgoCD Application manifests themselves — not workload YAML. Because ArgoCD Applications are just Kubernetes custom resources, ArgoCD can manage them the exact same way it manages a Deployment: `shelfkart-root` syncs the `applications/` directory, which creates (or updates, or with `prune: true`, deletes) every individual service's Application resource in the cluster. Adding a fourteenth microservice becomes "commit one new Application YAML file to the `applications/` directory" — `shelfkart-root` picks it up on its next sync and creates it automatically, with zero manual `kubectl` or `argocd` commands.

#### 5.2 Verifying the Full Tree

```bash
argocd app list
```

```
NAME                          SYNC STATUS   HEALTH STATUS
shelfkart-root                Synced        Healthy
checkout-service-production   Synced        Healthy
checkout-service-staging      Synced        Healthy
inventory-service-production  Synced        Healthy
cart-service-production       Synced        Healthy
payments-service-production   Synced        Healthy
```

> One `argocd app list` now shows the sync and health status of every ShelfKart service at a glance — exactly the visibility Vikram was missing in the opening scene, where five engineers gave five different answers about what was actually running.

> **Key takeaways**
> - The App of Apps pattern uses one root Application whose source is a directory of other Application manifests, letting ArgoCD manage Applications declaratively, the same way it manages workloads.
> - Adding a new microservice under GitOps becomes a Git commit of one Application YAML file, not a manual `kubectl apply`.
> - `argocd app list` gives one consolidated view of sync and health status across every managed Application.

### 6. Phase 6 — Projects, RBAC and Notifications

**Business Problem:** Every ArgoCD Application currently lives in the `default` Project with no restrictions — meaning any engineer with ArgoCD access could, in principle, sync a change to payments-service even if they only work on the recommendations team. During Diwali sale week, Amit wants sync failures on critical services to page the on-call channel immediately, not be discovered by someone refreshing the UI.

#### 6.1 Creating an ArgoCD Project

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: checkout-team
  namespace: argocd
spec:
  description: Applications owned by the checkout and payments team
  sourceRepos:
    - https://github.com/shelfkart/k8s-manifests.git
  destinations:
    - namespace: 'checkout-*'
      server: https://kubernetes.default.svc
    - namespace: 'payments-*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: Deployment
    - group: ''
      kind: Service
    - group: ''
      kind: ConfigMap
```

> **📖 What a Project scopes**
> An `AppProject` restricts what any Application assigned to it is allowed to do, independent of who's syncing it. `destinations` limits this project's Applications to only deploy into namespaces matching `checkout-*` or `payments-*` — an Application in this Project cannot be pointed at, say, the `inventory-production` namespace, even by a user with sync permissions, because the Project itself forbids it. `clusterResourceWhitelist: []` blocks any cluster-scoped resources (like ClusterRoles or CustomResourceDefinitions) from being deployed by Applications in this Project at all — only the specific namespaced resource kinds listed in `namespaceResourceWhitelist` are permitted, so this team cannot accidentally (or deliberately) deploy a cluster-wide RBAC change through their service pipeline.

#### 6.2 RBAC: Mapping SSO Groups to ArgoCD Roles

```csv
# argocd-rbac-cm ConfigMap, policy.csv
p, role:checkout-team-member, applications, sync, checkout-team/*, allow
p, role:checkout-team-member, applications, get, checkout-team/*, allow
g, shelfkart-checkout-team, role:checkout-team-member
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    p, role:checkout-team-member, applications, sync, checkout-team/*, allow
    p, role:checkout-team-member, applications, get, checkout-team/*, allow
  policy.default: role:readonly
```

> **📖 Reading the policy syntax**
> Each `p` line is a permission: role `checkout-team-member` may `sync` and `get` any Application in the `checkout-team` Project (`checkout-team/*`). The `g` line maps an external identity — here, the `shelfkart-checkout-team` group from ShelfKart's SSO provider, wired through `argocd-dex-server` — to that ArgoCD role, so group membership managed centrally in the company's identity provider determines ArgoCD access, rather than ArgoCD maintaining its own separate user list. `policy.default: role:readonly` means anyone authenticated but not explicitly granted a role can view Applications but cannot sync or modify anything — a safe default for the rest of the engineering org who should be able to see deployment status without being able to change it.

#### 6.3 Notifications on Sync Failure

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [sync-failed-slack]
  template.sync-failed-slack: |
    message: |
      :rotating_light: {{.app.metadata.name}} failed to sync.
      Revision: {{.app.status.sync.revision}}
      Error: {{.app.status.operationState.message}}
  subscriptions: |
    - recipients:
      - slack:shelfkart-oncall
      triggers:
      - on-sync-failed
```

> **Argo CD Notifications controller** — evaluates `trigger` conditions against every Application's live state on each reconciliation. `on-sync-failed` fires when `operationState.phase` is `Error` or `Failed`, rendering the `sync-failed-slack` template (which interpolates the actual failing revision and error message from the Application's status) and sending it to the `shelfkart-oncall` Slack channel via the subscription. During Diwali sale week, this is what turns "a sync silently failed and nobody knew for two hours" into "the on-call channel gets a message within one reconciliation cycle, with the exact error message already included."

> **Key takeaways**
> - `AppProject` restricts source repos, destination namespaces, and allowed resource kinds for every Application assigned to it — a hard boundary independent of individual user permissions.
> - ArgoCD RBAC policies map roles to Project-scoped permissions and can bind external SSO groups to those roles via Dex, avoiding a separate ArgoCD-only user list.
> - The Notifications controller watches Application status and can push targeted alerts (Slack, email, webhook) on specific conditions like sync failure, without any custom scripting.

##### 🏋️ Hands-On Exercises — Extend the Project

1. Create a second `AppProject` named `platform-team`, scoped to `namespaceResourceWhitelist` including `Ingress` and `NetworkPolicy` resources that `checkout-team` is explicitly forbidden from deploying, and confirm a sync attempt of an Ingress from within the checkout-team project is rejected.
2. Add a `staging` overlay for a second service (`inventory-service`) following the same Kustomize base/overlay structure from Phase 4, and register it as an Application under `shelfkart-root`'s App of Apps tree.
3. Change `checkout-service-production`'s `targetRevision` from `main` to a specific Git tag (e.g. `v2.10.0`), sync, and confirm that new commits to `main` no longer trigger a diff or sync — only a new tag reference would.
4. Add a second notification trigger, `on-deployed`, that posts a (non-urgent) message to a general `#shelfkart-releases` channel whenever an Application's health transitions to `Healthy` after a sync, giving the team a positive-confirmation feed alongside the failure alerts.
5. Deliberately misconfigure `checkout-service-staging`'s `AppProject` destination to exclude the `checkout-staging` namespace, attempt a sync, and document the exact error ArgoCD returns — then fix it and confirm the sync succeeds.

### ArgoCD Project Complete 🎉

You have brought ShelfKart's entire microservice fleet under GitOps ahead of the Diwali sale: ArgoCD installed and connected to a canonical Git repository, Applications with clear Sync and Health status separating "did the apply succeed" from "is it actually working," automated drift detection and self-healing protecting against untracked manual changes, Kustomize overlays managing staging and production from one shared base, the App of Apps pattern scaling to thirteen services without manual per-service setup, and Projects, RBAC, and Notifications enforcing who can touch what and alerting the team the moment something breaks.

> **Vikram** _Junior Platform Engineer_
>
> Nobody asks "what's actually running in production" anymore. `argocd app list` answers it in one glance, and if it says Synced and Healthy, that's genuinely true, because ArgoCD is the only thing that's allowed to touch the cluster now.

> **Priya** _Senior DevOps Engineer_
>
> The drift detection paid off during sale week itself — someone hotfixed a resource limit directly on the cluster during a traffic spike, ArgoCD flagged it OutOfSync within a minute, and instead of that fix silently getting reverted or silently staying untracked forever, we committed it to Git immediately and it became the real, permanent fix.

> **Amit** _Platform Architect_
>
> And when a bad checkout-service tag did go out mid-week, recovery was a `git revert` and a two-minute wait. No incident bridge call, no scrambling to remember what the last good config was — Git already had the answer.

> **Next: Progressive delivery on top of GitOps**
>
> - Argo Rollouts for canary and blue-green deployments, integrated directly with ArgoCD Applications for automated, metric-gated progressive rollout.
> - ArgoCD ApplicationSets to generate Applications dynamically from a cluster or Git-directory generator, replacing hand-written entries in the `applications/` directory.
> - Sealed Secrets or External Secrets Operator to bring secret management under the same GitOps model used for everything else in this project.
