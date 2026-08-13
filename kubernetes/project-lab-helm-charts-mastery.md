# Helm Charts Mastery

> **👋 Hey Fresher — Read This First!**

> Helm is the **package manager for Kubernetes** — just like apt is to Ubuntu or pip is to Python. Without Helm, deploying an application to Kubernetes means writing and managing 8–15 separate YAML files: Deployment, Service, ConfigMap, Secret, Ingress, HPA, ServiceAccount, and more. With Helm, you bundle all of these into a single **chart**, define variables in a **values.yaml**, and deploy to any environment with **one command**. Change the environment — change the values. The code is the same.

> **Company in this project:** CloudBridge — a Hyderabad-based SaaS startup running a multi-tenant invoice management platform called **InvoiceX** on Kubernetes. They deploy to three environments: dev, staging, and production. You just joined as a Junior DevOps Engineer. Your lead is **Priya**. Without Helm, every deployment means editing 12 YAML files manually and re-applying them — per environment. Let's fix that.

#### What You Will Learn and Build in This Project

You will author a production-grade Helm chart for CloudBridge's InvoiceX application, covering every layer of Helm — from chart structure and templates to values overrides, helpers, hooks, secrets, and chart repositories.

Chart Structure, Templates, Values & Overrides, Template Functions, Hooks, Subcharts, Repositories, Secrets, Rollback, Testing

#### InvoiceX Architecture — What We Are Deploying

```
CloudBridge InvoiceX — Kubernetes Deployment via Helm
═══════════════════════════════════════════════════════════════════════════

  Developer Laptop
  ┌──────────────────────────────────────────────────────┐
  │  helm install invoicex ./invoicex-chart \            │
  │         -f values-prod.yaml              │           │
  └──────────────────────────────────┬───────────────────┘
                                     │
                                     ▼
  ┌──────────── Kubernetes Cluster (EKS / AKS) ────────────────┐
  │                                                             │
  │  Namespace: invoicex-prod                                   │
  │  ┌───────────────────────────────────────────────────────┐  │
  │  │                                                       │  │
  │  │   Helm Release: invoicex                              │  │
  │  │   ┌────────────┐   ┌──────────┐   ┌───────────────┐  │  │
  │  │   │ Deployment │   │ Service  │   │    Ingress    │  │  │
  │  │   │ (invoicex) │──▶│  :8080   │──▶│ invoicex.io   │  │  │
  │  │   │ replicas:3 │   └──────────┘   └───────────────┘  │  │
  │  │   └─────┬──────┘                                      │  │
  │  │         │                                             │  │
  │  │   ┌─────▼──────┐   ┌──────────┐   ┌───────────────┐  │  │
  │  │   │ ConfigMap  │   │  Secret  │   │      HPA      │  │  │
  │  │   │ (app conf) │   │ (db cred)│   │ min:2 max:10  │  │  │
  │  │   └────────────┘   └──────────┘   └───────────────┘  │  │
  │  │                                                       │  │
  │  │   Subchart: postgresql  │  Subchart: redis            │  │
  │  └───────────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────────┘

  helm list     → shows all releases
  helm history  → shows revision history (rollback target)
  helm rollback → reverts to any previous revision instantly
```

### 1. Phase 1 — Helm Installation and Core Concepts

**Business Problem:** CloudBridge deploys InvoiceX manually — the team copies YAML files, edits image tags, changes replica counts, and re-runs `kubectl apply`. A wrong value in production caused a 40-minute outage last month. Helm solves this with templated, versioned, repeatable deployments.

**Scene 1 — CloudBridge Engineering Standup | "We Need Helm Yesterday"**

> **Priya** _Lead DevOps Engineer — CloudBridge_
> 
> Ravi, the production deploy last Tuesday — you manually changed the DB_HOST value in the ConfigMap YAML, but forgot to change it in the Deployment env section. Two different configs pointing to two different databases. That is how we got inconsistent data for 3 clients. With Helm, there is ONE values file. One source of truth. Change DB_HOST in one place — every resource that references it updates automatically. We are migrating InvoiceX to Helm this sprint. Kiran, you're pairing with the new engineer today.

> **Kiran** _Senior DevOps Engineer — CloudBridge_
> 
> Helm is also how we get proper rollback. Right now if a deploy breaks prod, we are manually reverting YAML files from Git and hoping we get every file. With Helm, it is literally one command — helm rollback invoicex 3 — and we are back to revision 3 in under a minute.

#### 1.1 Install Helm

```bash
# Linux / macOS — install via script
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS — via Homebrew
brew install helm

# Verify installation
helm version
```

**📖 Helm 3 vs Helm 2**

Always install **Helm 3**. Helm 2 required a server-side component called Tiller — a major security risk. Helm 3 is client-only: it talks directly to the Kubernetes API server using your kubeconfig. No Tiller, no cluster-side installation needed. Just install the binary and it works with any cluster your kubectl is already connected to.

```
$ helm version
version.BuildInfo{Version:"v3.14.2", GitCommit:"c309b6f", GoVersion:"go1.21.7"}
```

#### 1.2 Core Helm Concepts — The Vocabulary

> **📦 Chart**

> - A Helm package — a directory of templates + metadata
> - Like a .deb or .rpm — but for Kubernetes
> - Can be shared via a chart repository
> - A running instance of a chart in a cluster
> - One chart can have multiple releases (dev, staging, prod)
> - Each release has its own name and history
> - Configuration variables for a chart
> - Defined in values.yaml (defaults)
> - Overridden per environment via -f or --set
> - A hosted collection of charts (like PyPI for Python)
> - Artifact Hub is the public chart registry
> - Companies host private repos on S3, Nexus, or Harbor

### 2. Phase 2 — Creating the InvoiceX Chart

**Business Problem:** No chart exists for InvoiceX yet. You need to scaffold one, understand every file, and wire up the application's Deployment, Service, ConfigMap, Secret, and Ingress.

#### 2.1 Scaffold a New Chart

```bash
# Create a new chart called invoicex
helm create invoicex

# Examine the generated structure
tree invoicex/
```

**📖 helm create**

**helm create** generates a complete working chart with sample templates for a web application. It is the fastest way to start. You then customise the generated files rather than writing from scratch. Every chart follows the same directory layout — this consistency is what lets anyone understand any chart immediately.

```
invoicex/
├── Chart.yaml          # Chart metadata — name, version, description
├── values.yaml         # Default configuration values
├── charts/             # Dependency subcharts go here
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl    # Reusable template helpers (NOT rendered)
│   ├── NOTES.txt       # Post-install instructions shown to user
│   └── tests/
│       └── test-connection.yaml
└── .helmignore         # Files to exclude from the chart package
```

#### 2.2 Chart.yaml — Chart Identity

```
# invoicex/Chart.yaml
apiVersion: v2
name: invoicex
description: CloudBridge InvoiceX SaaS Application
type: application
version: 1.4.0
appVersion: "2.9.1"
keywords:
  - invoicing
  - saas
maintainers:
  - name: Priya
email: priya@cloudbridge.io
```

**📖 Two Version Fields**

**version** is the chart version — increment this when you change the chart's templates or structure. Follows SemVer.  
  
**appVersion** is the application version — the Docker image tag of InvoiceX itself. Changing appVersion does NOT change the chart version. A chart at version 1.4.0 can deploy app versions 2.9.1, 2.9.2, etc.  
  
**type: application** — this chart deploys an app. A `library` chart only contains reusable helpers and cannot be installed directly.

### 3. Phase 3 — values.yaml and Template Variables

**Business Problem:** InvoiceX runs in dev, staging, and prod with different replica counts, resource limits, image tags, and database hosts. Without values, you'd maintain 3 separate sets of YAML files. With values, you have one chart and 3 small override files.

**Scene 3 — CloudBridge Slack | "The Staging Config Copied to Prod Again"**

> **Ravi** _Backend Engineer — CloudBridge_
> 
> I just found that our production DB_MAX_CONNECTIONS is set to 5. That is the staging value. Someone copy-pasted the staging ConfigMap YAML and forgot to update it. We are limiting the prod database to 5 connections. That is why the app was slow after noon yesterday — connection pool exhaustion. With Helm values, staging gets values-staging.yaml and prod gets values-prod.yaml. There is no copy-paste. They cannot cross-contaminate.

#### 3.1 Default values.yaml — Full InvoiceX Config

```
# invoicex/values.yaml — default values for all environments
replicaCount: 1
image:
  repository: cloudbridge/invoicex
pullPolicy: IfNotPresent
tag: "" # Defaults to .Chart.AppVersion if empty
service:
  type: ClusterIP
port: 8080
ingress:
  enabled: false
host: invoicex.local
resources:
  requests:
    cpu: 100m
memory: 128Mi
limits:
    cpu: 500m
memory: 512Mi
app:
  dbHost: localhost
dbPort: 5432
dbMaxConn: 5
logLevel: info
autoscaling:
  enabled: false
minReplicas: 1
maxReplicas: 5
targetCPUUtilizationPercentage: 70
```

> **values.yaml is the defaults file** — safe conservative values for local/dev. Every key here is accessible in templates as `{{ .Values.keyName }}`. Environment-specific files only need to override what differs — they inherit everything else from defaults. **image.tag: ""** — empty string means the Deployment template will fall back to `.Chart.AppVersion`, keeping app version and chart metadata in sync automatically.

#### 3.2 Environment Override Files

```
# values-prod.yaml — production overrides ONLY
replicaCount: 3
image:
  tag: "2.9.1"
ingress:
  enabled: true
host: app.invoicex.io
resources:
  requests:
    cpu: 500m
memory: 512Mi
limits:
    cpu: 2000m
memory: 2Gi
app:
  dbHost: prod-pg.cloudbridge.internal
dbMaxConn: 50
logLevel: warn
autoscaling:
  enabled: true
minReplicas: 3
maxReplicas: 12
```

**📖 Override Files — The Key Pattern**

**Only include what changes.** Helm deep-merges the values files — production only needs to list the keys that differ from defaults. Everything else inherits from values.yaml.  
  
Deploy command:  
`helm install invoicex ./invoicex -f values-prod.yaml`  
  
Helm merges: values.yaml + values-prod.yaml. Prod wins on conflicts. The result is the final values used to render all templates.  
  
This means dev has `dbMaxConn: 5` and prod has `dbMaxConn: 50` — with zero risk of mixing them.

### 4. Phase 4 — Writing Templates

**Business Problem:** The template files inside `templates/` are the heart of Helm — they are standard Kubernetes YAML with Go template syntax embedded. You need to write templates that read from values.yaml and produce correct, environment-aware Kubernetes manifests.

#### 4.1 Deployment Template

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "invoicex.fullname" . }}
labels:
    {{- include "invoicex.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
selector:
    matchLabels:
      {{- include "invoicex.selectorLabels" . | nindent 6 }}
template:
    metadata:
      labels:
        {{- include "invoicex.selectorLabels" . | nindent 8 }}
spec:
      containers:
        - name: {{ .Chart.Name }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
ports:
            - containerPort: {{ .Values.service.port }}
envFrom:
            - configMapRef:
                name: {{ include "invoicex.fullname" . }}-config
resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

> **{{ .Values.replicaCount }}** — reads the replicaCount from values.yaml (or the override file). **{{ include "invoicex.fullname" . }}** — calls the reusable helper defined in _helpers.tpl. **{{ .Values.image.tag | default .Chart.AppVersion }}** — if tag is empty in values, fall back to Chart.yaml's appVersion. **{{- toYaml .Values.resources | nindent 12 }}** — converts the resources map to YAML and indents it 12 spaces. The leading **-** trims the preceding whitespace/newline.

#### 4.2 Service Template

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "invoicex.fullname" . }}
spec:
  type: {{ .Values.service.type }}
ports:
    - port: {{ .Values.service.port }}
targetPort: {{ .Values.service.port }}
protocol: TCP
selector:
    {{- include "invoicex.selectorLabels" . | nindent 4 }}
```

**📖 Service Type from Values**

**.Values.service.type** is ClusterIP in dev (internal only) and can be overridden to LoadBalancer or NodePort in other envs — all without touching the template.  
  
The selector uses the same labels helper as the Deployment's matchLabels, ensuring they always match. If you change the label scheme, you update the helper once — not in 4 separate files.

#### 4.3 ConfigMap Template

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "invoicex.fullname" . }}-config
data:
  DB_HOST: {{ .Values.app.dbHost | quote }}
DB_PORT: {{ .Values.app.dbPort | quote }}
DB_MAX_CONN: {{ .Values.app.dbMaxConn | quote }}
LOG_LEVEL: {{ .Values.app.logLevel | quote }}
```

**📖 The | quote Function**

ConfigMap data values must be strings. `| quote` wraps the value in double quotes — critical when the value is a number like `5432`. Without `| quote`, YAML may interpret it as an integer and Kubernetes rejects it.  
  
This ConfigMap is referenced in the Deployment via `envFrom: configMapRef` — all keys are injected as environment variables into the container automatically.

#### 4.4 Ingress Template with Conditional

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "invoicex.fullname" . }}
spec:
  rules:
    - host: {{ .Values.ingress.host }}
http:
        paths:
          - path: /
pathType: Prefix
backend:
              service:
                name: {{ include "invoicex.fullname" . }}
port:
                  number: {{ .Values.service.port }}
{{- end }}
```

**📖 Conditional Template Rendering**

`{{- if .Values.ingress.enabled -}}` — if this is false (dev), Helm renders an empty file and Kubernetes ignores it. If true (prod), the full Ingress object is created.  
  
This pattern controls which Kubernetes resources exist per environment. Dev has no Ingress (uses port-forward). Prod has one pointing to the real domain.  
  
The **-** on both sides of if/end trims surrounding whitespace so no blank lines appear in the rendered output.

#### 4.5 HPA Template with Conditional

```yaml
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "invoicex.fullname" . }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
kind: Deployment
name: {{ include "invoicex.fullname" . }}
minReplicas: {{ .Values.autoscaling.minReplicas }}
maxReplicas: {{ .Values.autoscaling.maxReplicas }}
metrics:
    - type: Resource
resource:
        name: cpu
target:
          type: Utilization
averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

**📖 HPA — Only in Prod**

Dev and staging have `autoscaling.enabled: false`, so no HPA object is ever created there — preventing accidental scale-up in non-production environments.  
  
In production, `autoscaling.enabled: true` and `minReplicas: 3 / maxReplicas: 12` give InvoiceX elastic scaling based on CPU. This is defined once in the template and controlled purely by the values file.

### 5. Phase 5 — _helpers.tpl and Named Templates

**Business Problem:** The name `invoicex` is referenced in every template file — Deployment, Service, ConfigMap, Ingress, HPA. If the naming convention changes, you'd edit 6 files. _helpers.tpl centralises all reusable logic in one place.

#### 5.1 The _helpers.tpl File — CloudBridge Edition

```bash
# templates/_helpers.tpl  (underscore = not rendered as a manifest)
{{/*  Full release name — truncated at 63 chars (DNS limit)  */}}
{{- define "invoicex.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{/*  Standard labels  */}}
{{- define "invoicex.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version }}
{{- include "invoicex.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
{{/*  Selector labels  */}}
{{- define "invoicex.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

> **Files starting with underscore (_) are never rendered** as Kubernetes manifests — Helm treats them as library files. **define "invoicex.fullname"** — declares a named template called anywhere with `{{ include "invoicex.fullname" . }}`. The **.** (dot) passes the current context (all values, chart info, release info). **trunc 63** — Kubernetes DNS names have a 63-character limit. **trimSuffix "-"** ensures no trailing dash if the truncation lands on one. **.Release.Name** is the name you give at install time (`helm install invoicex ...`).

### 6. Phase 6 — Install, Upgrade, Rollback

**Business Problem:** The team needs to deploy InvoiceX to dev, promote it to staging after QA, deploy to production during the release window, and be able to instantly roll back if a bug is found in production.

**Scene 6 — CloudBridge Release Night | "Something Is Wrong in Prod"**

> **Kiran** _Senior DevOps Engineer — CloudBridge_
> 
> We are 8 minutes into the production release. Error rate on the invoice generation endpoint just jumped from 0.1% to 14%. The new PDF rendering library in v2.9.2 is breaking something. I don't have time to debug this now. Rolling back to v2.9.1 — helm rollback invoicex 4. That is revision 4 — the last known good prod release. Done. Error rate dropping. We'll debug v2.9.2 in staging tomorrow.

#### 6.1 The Core Helm Commands

Command

What It Does

CloudBridge Use Case

helm install invoicex ./invoicex -f values-dev.yaml

First-time deploy of the chart

Initial dev environment setup

helm upgrade invoicex ./invoicex -f values-prod.yaml

Update existing release

Every new application release

helm upgrade --install invoicex ./invoicex

Install if not exist, upgrade if exists

CI/CD pipeline — idempotent deploy

helm list -n invoicex-prod

Show all releases in a namespace

See current deployed version

helm history invoicex -n invoicex-prod

Show all revisions of a release

Find revision to roll back to

helm rollback invoicex 4 -n invoicex-prod

Roll back to a specific revision

Production incident recovery

helm uninstall invoicex -n invoicex-prod

Delete the release and all resources

Environment teardown

helm template invoicex ./invoicex -f values-prod.yaml

Render templates to stdout (no deploy)

Review what will be applied before running

#### 6.2 Install with Namespace Creation

```bash
# Install InvoiceX to production namespace
helm upgrade --install invoicex ./invoicex \
  --namespace invoicex-prod \
  --create-namespace \
  -f values.yaml \
  -f values-prod.yaml \
  --set image.tag=2.9.1 \
  --atomic \
  --timeout 5m
```

**📖 Production Install Flags**

**--create-namespace** — creates the namespace if it doesn't exist. Safe to re-run.  
  
**-f values.yaml -f values-prod.yaml** — layered: defaults first, then prod overrides.  
  
**--set image.tag=2.9.1** — CLI override, highest priority. Perfect for CI/CD passing the build tag.  
  
**--atomic** — if the deploy fails (pod doesn't become Ready within timeout), Helm automatically rolls back to the previous revision. Zero manual intervention needed.

#### 6.3 Helm History and Rollback

```bash
# View revision history of invoicex in production
helm history invoicex -n invoicex-prod
```

```
REVISION  UPDATED                   STATUS      CHART            APP VERSION   DESCRIPTION
1         Mon Jan 06 09:12:11 2025  superseded  invoicex-1.2.0   2.8.0         Install complete
2         Fri Jan 17 14:33:42 2025  superseded  invoicex-1.3.0   2.8.5         Upgrade complete
3         Tue Feb 04 11:05:19 2025  superseded  invoicex-1.4.0   2.9.0         Upgrade complete
4         Mon Feb 10 18:22:07 2025  superseded  invoicex-1.4.0   2.9.1         Upgrade complete
5         Tue Feb 11 20:14:55 2025  failed      invoicex-1.4.0   2.9.2         Upgrade failed
6         Tue Feb 11 20:16:02 2025  deployed    invoicex-1.4.0   2.9.1         Rollback to 4
```

```bash
# Roll back to revision 4 (the last good production deploy)
helm rollback invoicex 4 -n invoicex-prod
```

> Helm stores the complete manifest state of every revision as a Kubernetes Secret. When you rollback to revision 4, Helm re-applies the exact manifests from that revision — including image tag, ConfigMap values, resource limits, and replica count. It creates a new revision (6 here) so the rollback itself is in the history and auditable.

### 7. Phase 7 — Template Functions and Pipelines

**Business Problem:** InvoiceX templates need logic — conditionals, loops, default values, string formatting, and type conversions. Helm's template functions (from Go's Sprig library) handle all of this without writing code.

#### 7.1 Essential Template Functions

```
# default — use fallback if value is empty
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
# quote — wrap value in double quotes
DB_PORT: {{ .Values.app.dbPort | quote }}
# upper / lower — string case
LOG_LEVEL: {{ .Values.app.logLevel | upper }}
# printf — format strings
{{ printf "%s-%s" .Release.Name .Chart.Name }}
# trunc — limit string length
{{ .Release.Name | trunc 63 | trimSuffix "-" }}
```

**📖 Sprig Functions**

Helm templates use Go's text/template engine plus the **Sprig** library — over 70 utility functions for strings, math, dates, dicts, and lists.  
  
**Pipelines** chain functions with `|` — left-to-right: `.Values.app.logLevel | upper` reads the value then converts to uppercase.  
  
Essential functions for every chart: `default`, `quote`, `toYaml`, `nindent`, `trunc`, `trimSuffix`, `printf`, `include`.

#### 7.2 toYaml and nindent — Embedding Maps

```
# In values.yaml
resources:
  requests:
    cpu: 500m
memory: 512Mi
limits:
    cpu: 2000m
memory: 2Gi
# In deployment.yaml template
resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**📖 toYaml + nindent Pattern**

**toYaml** converts the Go map structure to a YAML string.  
  
**nindent 12** adds a leading newline plus 12 spaces of indentation to every line — matching the indentation level in the Deployment spec.  
  
Without `nindent`, the YAML would be at column 0 and Kubernetes would reject it as invalid YAML indentation. This is the most common pattern for embedding nested value objects in templates.

#### 7.3 range — Looping over Lists

```
# values.yaml — list of environment variables
extraEnv:
  - name: FEATURE_PDF_WATERMARK
value: "true"
  - name: MAX_INVOICE_SIZE_MB
value: "25"
# deployment.yaml template — render the list
env:
            {{- range .Values.extraEnv }}
            - name: {{ .name }}
value: {{ .value | quote }}
            {{- end }}
```

**📖 range — Iterate Lists**

`range .Values.extraEnv` loops over each item in the list. Inside the range block, **.** (dot) refers to the current list item — so `.name` and `.value` access that item's fields.  
  
This lets ops teams add arbitrary environment variables to InvoiceX per environment without modifying the template — just add items to the extraEnv list in the values file.

#### 7.4 with — Scoping Context

```
# with — enter a sub-scope if the value exists
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 2 }}
{{- end }}
{{- with .Values.tolerations }}
tolerations:
  {{- toYaml . | nindent 2 }}
{{- end }}
```

**📖 with — Safe Conditional Scope**

`with .Values.nodeSelector` — if nodeSelector is empty (nil/null/{}), the block is skipped entirely. If it has values, the block executes with `.` set to `.Values.nodeSelector` — so you write `toYaml .` instead of `toYaml .Values.nodeSelector`.  
  
This prevents the rendered YAML from having `nodeSelector: {}` or `tolerations: null` which Kubernetes ignores but clutters the manifest.

### 8. Phase 8 — Secrets Management

**Business Problem:** InvoiceX needs a PostgreSQL password, a Razorpay API key, and a JWT signing secret. These cannot go into values.yaml in plaintext — that file is committed to Git. Helm provides two approaches: external secrets and sealed secrets.

**Scene 8 — CloudBridge Security Review | "We Found a Password in Git"**

> **Priya** _Lead DevOps Engineer — CloudBridge_
> 
> The security audit found values-prod.yaml committed in our GitLab repo with the PostgreSQL password in plaintext. The repo is private but that is not the point — any developer with repo access can see the production database credentials. And if the repo is ever accidentally made public, we are exposed. No more secrets in values files. We are moving to Kubernetes Secrets created outside of Helm and referenced by name. The chart references the secret by name — the secret value lives only in the cluster.

#### 8.1 External Secret Pattern — Reference by Name

```
# values.yaml — reference, not the value
secrets:
  dbSecretName: invoicex-db-credentials
appSecretName: invoicex-app-secrets
# templates/deployment.yaml — mount secret as env
envFrom:
            - secretRef:
                name: {{ .Values.secrets.dbSecretName }}
            - secretRef:
                name: {{ .Values.secrets.appSecretName }}
```

**📖 Secrets Outside the Chart**

The Helm chart only stores the **name** of the Secret — never the value. The actual Secret is created separately (by Terraform, by an operator like External Secrets Operator, or manually in a one-time setup) and exists in the cluster.  
  
This means values-prod.yaml can be committed to Git safely — it contains no sensitive data. The Secret values live only in Kubernetes (etcd, ideally encrypted at rest).

#### 8.2 Create the Secret Manually (One-Time)

```bash
# Create the DB credentials secret (one-time, not via Helm)
kubectl create secret generic invoicex-db-credentials \
  --from-literal=DB_PASSWORD=SuperSecretProdPass \
  --from-literal=DB_USER=invoicex_prod \
  -n invoicex-prod

# Create app secrets (JWT key, Razorpay key)
kubectl create secret generic invoicex-app-secrets \
  --from-literal=JWT_SECRET=jwt-signing-key-here \
  --from-literal=RAZORPAY_KEY=rzp_live_key_here \
  -n invoicex-prod
```

**📖 Pre-existing Secrets**

These are created once. Helm upgrade/rollback does not touch them — they persist independently of the chart lifecycle. When Helm deploys, Kubernetes injects the secret's keys as environment variables into every pod (via `envFrom: secretRef`).  
  
For a fully automated approach, use the **External Secrets Operator** which syncs secrets from AWS Secrets Manager or Azure Key Vault into Kubernetes Secrets automatically — no manual kubectl commands.

#### 8.3 Secret Template (For Non-Sensitive Generated Values)

```yaml
# templates/secret.yaml — for non-sensitive generated values
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "invoicex.fullname" . }}-generated
type: Opaque
data:
  INTERNAL_TOKEN: {{ randAlphaNum 32 | b64enc | quote }}
```

**📖 randAlphaNum + b64enc**

**randAlphaNum 32** — generates a random 32-character alphanumeric string. **b64enc** — base64 encodes it (Kubernetes Secret data values must be base64).  
  
⚠️ Warning: This generates a NEW random token on every helm upgrade — the token changes on each deploy. For stable tokens, use the external secret pattern instead. Only use randAlphaNum for ephemeral or rotation-tolerant values.

### 9. Phase 9 — Helm Hooks

**Business Problem:** Before InvoiceX v2.9.1 goes live, the database schema must be migrated. After a successful deploy, a Slack notification must fire. These are pre-install and post-install tasks — Helm hooks handle this lifecycle.

#### 9.1 Pre-Install Hook — Database Migration

```yaml
# templates/pre-install-migration.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "invoicex.fullname" . }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
"helm.sh/hook-weight": "-5"
"helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      restartPolicy: Never
containers:
        - name: migrate
image: "cloudbridge/invoicex:{{ .Values.image.tag | default .Chart.AppVersion }}"
command: ["python", "manage.py", "migrate"]
          envFrom:
            - secretRef:
                name: {{ .Values.secrets.dbSecretName }}
```

> **helm.sh/hook: pre-upgrade,pre-install** — runs this Job BEFORE Helm installs or upgrades any other resource. The migration runs first. If the migration Job fails, Helm aborts the install/upgrade and nothing else is deployed — the database and app stay in sync. **hook-weight: -5** — lower weight runs first (can sequence multiple hooks). **hook-delete-policy: before-hook-creation** — deletes the previous migration Job before creating a new one (prevents name conflicts on re-deploy).

#### 9.2 Post-Install Hook — Slack Notification

```yaml
# templates/post-install-notify.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "invoicex.fullname" . }}-notify
  annotations:
    "helm.sh/hook": post-upgrade,post-install
"helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
containers:
        - name: notify
image: curlimages/curl:latest
command:
            - sh
            - -c
            - "curl -X POST $SLACK_WEBHOOK -d '{\"text\":\"InvoiceX deployed\"}'"
```

**📖 Hook Lifecycle Values**

**pre-install** — before first install  
**post-install** — after first install  
**pre-upgrade** — before every upgrade  
**post-upgrade** — after every upgrade  
**pre-rollback** — before rollback  
**post-rollback** — after rollback  
**pre-delete** — before helm uninstall  
  
**hook-delete-policy: hook-succeeded** — cleans up the Job pod only on success. Failed hook pods persist for debugging.

### 10. Phase 10 — Chart Dependencies (Subcharts)

**Business Problem:** InvoiceX needs PostgreSQL and Redis. Instead of managing those YAML files manually, use the official Bitnami Helm charts as dependencies — they are pre-built, battle-tested, and configurable via the parent chart's values.

#### 10.1 Declare Dependencies in Chart.yaml

```
# invoicex/Chart.yaml — add dependencies section
dependencies:
  - name: postgresql
version: "14.3.3"
repository: https://charts.bitnami.com/bitnami
condition: postgresql.enabled
  - name: redis
version: "19.1.0"
repository: https://charts.bitnami.com/bitnami
condition: redis.enabled
```

**📖 Subchart Dependencies**

**condition: postgresql.enabled** — the subchart is only deployed if this value is true in values.yaml. Dev can set `postgresql.enabled: false` and point to an external DB. Prod can enable it to run PostgreSQL inside the same cluster.  
  
Run `helm dependency update ./invoicex` after adding dependencies — it downloads the subchart tarballs into the `charts/` directory.

#### 10.2 Configure Subcharts via Parent values.yaml

```
# values.yaml — subchart config under the chart name key
postgresql:
  enabled: true
auth:
    database: invoicex
username: invoicex_user
existingSecret: invoicex-db-credentials
primary:
    persistence:
      size: 20Gi
redis:
  enabled: true
architecture: standalone
auth:
    enabled: false
```

**📖 Subchart Values Namespacing**

Subchart configuration keys must be nested under the subchart's name (`postgresql:`, `redis:`). Helm passes these down to the subchart's own templates.  
  
The parent chart's `postgresql.auth.existingSecret` tells the Bitnami PostgreSQL chart to read the DB password from an existing Kubernetes Secret — not from a plaintext value in values.yaml. Best practice for production.

#### 10.3 Update and Verify Dependencies

```bash
# Download all declared dependencies
helm dependency update ./invoicex

# Verify what was downloaded
helm dependency list ./invoicex
```

**📖 charts/ Directory**

After running `helm dependency update`, the `charts/` directory contains `postgresql-14.3.3.tgz` and `redis-19.1.0.tgz`. These are downloaded from the Bitnami repository and packaged into your chart.  
  
Commit `Chart.lock` (not the .tgz files) to Git so CI can reproduce the exact same dependency versions. Add `charts/*.tgz` to `.helmignore`.

### 11. Phase 11 — helm template and helm lint

**Business Problem:** Before applying to a live cluster, the team needs to review the rendered manifests and catch template errors. helm template and helm lint are the safety net.

#### 11.1 helm template — Dry Run Rendering

```bash
# Render all templates without deploying
helm template invoicex ./invoicex \
  -f values.yaml \
  -f values-prod.yaml \
  --set image.tag=2.9.1 \
  > rendered-prod.yaml

# Review what Kubernetes will receive
cat rendered-prod.yaml | head -80
```

**📖 Why helm template**

`helm template` renders the Go templates to plain YAML without connecting to a cluster. Pipe to a file and review before every production deploy.  
  
This is also how GitOps tools like ArgoCD work — they render the Helm chart and apply the resulting YAML. You can commit the rendered output and diff it against previous releases to see exactly what changes in a PR.

#### 11.2 helm lint — Catch Errors Before Deploy

```bash
# Lint the chart for errors and warnings
helm lint ./invoicex -f values-prod.yaml

# Strict mode — treat warnings as errors
helm lint ./invoicex --strict
```

**📖 What helm lint Catches**

Helm lint validates: chart structure (missing required files), template syntax errors, YAML validity in rendered output, missing required values, deprecated API versions, and chart metadata issues.  
  
Run `helm lint` in CI/CD on every PR that touches the chart. It prevents broken charts from ever reaching the cluster. Use `--strict` in pipelines to fail on warnings, not just errors.

```
$ helm lint ./invoicex -f values-prod.yaml
==> Linting invoicex/
[INFO] Chart.yaml: icon is recommended
[WARNING] templates/: directory not found

1 chart(s) linted, 0 chart(s) failed
```

#### 11.3 helm diff — See What Will Change

```bash
# Install the helm-diff plugin first
helm plugin install https://github.com/databus23/helm-diff

# Compare current deployed state vs new chart
helm diff upgrade invoicex ./invoicex \
  -f values-prod.yaml \
  --set image.tag=2.9.2
```

**📖 helm diff Plugin**

Shows a diff between what is currently deployed and what would be deployed — like `git diff` but for Kubernetes resources. Reveals exactly which fields change before you commit to an upgrade.  
  
CloudBridge runs `helm diff` in staging before every prod deploy — the diff output goes into the change management ticket for approval. This is how production surprises are eliminated.

### 12. Phase 12 — Helm Test

**Business Problem:** After deploying InvoiceX, how does the team automatically verify it is working — not just that pods started, but that the application is actually responding correctly?

#### 12.1 Write a Test Pod

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "invoicex.fullname" . }}-test
  annotations:
    "helm.sh/hook": test
"helm.sh/hook-delete-policy": before-hook-creation
spec:
  restartPolicy: Never
containers:
    - name: test-health
image: curlimages/curl:latest
command:
        - curl
        - -f
        - "http://{{ include "invoicex.fullname" . }}:{{ .Values.service.port }}/healthz"
```

**📖 Helm Tests**

A test is a Pod with annotation `helm.sh/hook: test`. It is NOT deployed during `helm install` — only when you explicitly run `helm test invoicex`.  
  
The test passes if the pod exits 0. The test fails if the pod exits non-zero or times out.  
  
CloudBridge runs `helm test` in CI immediately after deploying to staging — if the health check fails, the pipeline blocks and alerts the team before anything reaches production.

```bash
# Run the tests for a deployed release
helm test invoicex -n invoicex-staging

# Show logs from the test pod
helm test invoicex -n invoicex-staging --logs
```

**📖 Test Workflow**

1. Deploy with `helm upgrade --install`  
2. Run `helm test invoicex`  
3. If tests pass → promote to next environment  
4. If tests fail → investigate logs, do not promote  
  
This gives you automated post-deploy verification that is versioned with the chart — not a separate test script that someone forgets to update.

### 13. Phase 13 — Chart Repositories

**Business Problem:** CloudBridge now has 6 microservices — InvoiceX, AuthService, NotificationService, BillingService, ReportService, and ApiGateway. Each has a Helm chart. They need to be stored, versioned, and shared across teams via a private chart repository.

**Scene 13 — CloudBridge All-Hands | "We Need a Chart Registry"**

> **Priya** _Lead DevOps Engineer — CloudBridge_
> 
> Right now every team has their chart as a directory in their own Git repo. When Team B wants to deploy AuthService from Team A's chart, they clone Team A's repo and hope it works. We need a central chart registry — like npm registry but for Helm charts. When Team A releases AuthService chart v2.1.0, they push it to the registry. Team B adds the registry as a repo and does helm install authservice cloudbridge/authservice —version 2.1.0. Clean dependency management. Versioned. Artifact Hub for public charts. Harbor or ChartMuseum for our private ones.

#### 13.1 Add and Use Public Repositories

```bash
# Add the Bitnami repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Add the Ingress-Nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Update all repos (like apt update)
helm repo update

# Search for a chart
helm search repo bitnami/postgresql --versions | head -5

# Install directly from repository
helm install my-pg bitnami/postgresql --version 14.3.3 \
  -f pg-values.yaml -n invoicex-prod
```

> **helm repo add** registers a repository URL locally. **helm repo update** fetches the latest chart index from all registered repos — always run this before searching or installing. **helm search repo** searches all registered repositories. The `--versions` flag shows all available chart versions, not just the latest. You pin to a specific version with `--version` to ensure reproducibility.

#### 13.2 Package and Push to a Private Repository

```bash
# Package the chart into a .tgz archive
helm package ./invoicex
# Output: invoicex-1.4.0.tgz
# Push to Harbor (OCI-based registry)
helm push invoicex-1.4.0.tgz \
  oci://harbor.cloudbridge.io/charts

# Install from OCI registry
helm install invoicex \
  oci://harbor.cloudbridge.io/charts/invoicex \
  --version 1.4.0 \
  -f values-prod.yaml
```

**📖 OCI-Based Chart Registries**

Helm 3 supports OCI (Open Container Initiative) registries — the same type used for Docker images. Harbor, AWS ECR, Azure Container Registry, and GitHub Container Registry all support OCI Helm charts.  
  
**harbor.cloudbridge.io** stores CloudBridge's private charts. Teams install charts by version using the OCI URL — no local chart directory needed. CI/CD runs `helm package + helm push` on every main branch merge.

### 14. Phase 14 — Multi-Environment CI/CD Pipeline with Helm

**Business Problem:** CloudBridge needs to automate the full promote-to-prod flow: build Docker image → deploy to dev → run tests → promote to staging → run tests → deploy to prod — all via Helm, triggered by GitLab CI.

#### 14.1 GitLab CI Pipeline — Helm Deploy

```
# .gitlab-ci.yml — CloudBridge InvoiceX pipeline
stages:
  - build
  - deploy-dev
  - test-dev
  - deploy-staging
  - deploy-prod
variables:
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA
deploy-to-dev:
  stage: deploy-dev
image: alpine/helm:latest
script:
    - helm upgrade --install invoicex ./invoicex
        --namespace invoicex-dev
        --create-namespace
        -f values.yaml
        -f values-dev.yaml
        --set image.tag=$IMAGE_TAG
        --atomic --timeout 5m
test-dev:
  stage: test-dev
image: alpine/helm:latest
script:
    - helm test invoicex -n invoicex-dev --logs
```

> **$CI_COMMIT_SHORT_SHA** — GitLab CI built-in variable: the first 8 characters of the commit SHA. Every commit produces a unique image tag — no more `:latest` in production. The same tag is passed to Helm via `--set image.tag=$IMAGE_TAG`, making every deployment fully traceable to its source commit. **--atomic** ensures that if the dev deploy fails (pods never become Ready), the pipeline fails and staging is never reached.

#### 14.2 Production Deploy with Manual Approval Gate

```
# .gitlab-ci.yml — production deploy (manual gate)
deploy-to-prod:
  stage: deploy-prod
image: alpine/helm:latest
when: manual
only:
    - main
script:
    - helm upgrade --install invoicex ./invoicex
        --namespace invoicex-prod
        --create-namespace
        -f values.yaml
        -f values-prod.yaml
        --set image.tag=$IMAGE_TAG
        --atomic --timeout 10m
```

**📖 Manual Gate for Production**

`when: manual` — the pipeline pauses here. A human must click "Deploy" in GitLab's UI to proceed. This is the standard CloudBridge release process.  
  
`only: main` — production deploys only from the main branch. Feature branches and MR pipelines run dev/staging, never prod.  
  
The **same IMAGE_TAG** that passed dev tests and staging tests is now deployed to prod — bit-for-bit identical, just with different values.

### 15. Phase 15 — Helm Best Practices for Production

**Scene 15 — CloudBridge Quarterly Review | "What Helm Saved Us"**

> **Priya** _Lead DevOps Engineer — CloudBridge_
> 
> Q1 summary for the CTO: We migrated all 6 InvoiceX services to Helm. Dev-to-prod deploy time dropped from 45 minutes to 8 minutes. We had one production incident — we rolled back in 43 seconds using helm rollback. Before Helm, that rollback would have taken 20 minutes of manual YAML editing and kubectl apply, hoping we reverted everything. We have not had a config mismatch between environments since the migration. Zero copy-paste errors. The staging DB is staging. The prod DB is prod. One chart. Three value files. Four environments. That is the ROI.

> **Helm Architecture Rules — CloudBridge Engineering Standards**

> - One chart per microservice. Never cram multiple unrelated applications into one chart — it makes rollback dangerous and testing impossible
> - Never put secrets in values files. Use external secrets or existingSecret references. Your values files go to Git. Your secret values must not
> - Always use --atomic in CI/CD deploy commands. If the new version fails to become Ready, Helm automatically rolls back — no manual cleanup, no half-deployed state
> - Pin subchart dependency versions exactly. `version: "~14.3"` is risky — the patch version can change on next helm dependency update and break your chart unexpectedly
> - Use helm diff before every production upgrade. Add it to your runbook and change management process. The diff is your changelog
> - Write at least one helm test per service — a basic health check against the service endpoint. If pods start but the app crashes immediately, kubectl shows Running but your test will fail
> - Use chart versions independently from app versions. The chart version tracks template changes. The app version tracks the Docker image. They evolve at different rates
> - Store rendered manifests (helm template output) as CI artifacts. Attach to release notes. This is your audit trail of exactly what was applied to production

##### Helm Code Standards — CloudBridge Chart Review Checklist

- All string values in ConfigMap data must use `| quote` — prevents YAML type coercion errors when values are numbers or booleans
- All embedded YAML maps (resources, nodeSelector, tolerations, affinity) use `toYaml | nindent N` — never hardcode resource values directly in templates
- All resource names use `{{ include "chart.fullname" . }}` — never hardcode the release name. Helm releases must be rename-safe for multi-instance deployments
- Run `helm lint --strict` in CI on every PR touching chart files. Zero warnings in a production-grade chart
- Set `nameOverride` and `fullnameOverride` in values.yaml as empty strings — let users override names without touching templates
- Every template file except _helpers.tpl should render exactly one Kubernetes resource. One file = one resource kind = easy to find and reason about

##### 🏋️ Hands-On Exercises — Extend the InvoiceX Chart

1. **Add a PodDisruptionBudget template:** Create `templates/pdb.yaml` with a PodDisruptionBudget that sets `minAvailable: 2`, wrapped in a conditional (`if .Values.pdb.enabled`). Add the `pdb.enabled: false` default in values.yaml and set it to `true` in values-prod.yaml. Verify with `helm template` that PDB only appears in the prod rendered output.
2. **Add a Liveness and Readiness probe:** Add `livenessProbe` and `readinessProbe` to the Deployment template, configurable via values. Define defaults in values.yaml for `httpGet.path: /healthz`, `initialDelaySeconds: 10`, `periodSeconds: 15`. Override `initialDelaySeconds: 30` in values-prod.yaml to account for longer startup time in production.
3. **Create a values-staging.yaml:** Define staging values with `replicaCount: 2`, production-like resource limits (75% of prod), autoscaling disabled, and the staging DB host. Deploy to a staging namespace with `helm upgrade --install invoicex ./invoicex -f values.yaml -f values-staging.yaml` and verify with `helm get values invoicex -n invoicex-staging`.
4. **Write a pre-delete hook for backup:** Create a Job with `helm.sh/hook: pre-delete` that runs a `pg_dump` before `helm uninstall` removes the release. The Job should write the dump to a PersistentVolumeClaim or S3. This ensures a database snapshot exists before any uninstall destroys data.
5. **Package and push to a local ChartMuseum:** Run ChartMuseum locally with Docker (`docker run -p 8080:8080 ghcr.io/helm/chartmuseum:latest`), add it as a Helm repo, package your invoicex chart with `helm package`, push with `curl --data-binary @invoicex-1.4.0.tgz http://localhost:8080/api/charts`, then install from it with `helm install invoicex local-charts/invoicex`. This simulates a real private chart registry workflow.

**Quiz: ❓ Interview Question: What is the difference between helm install and helm upgrade --install, and when would you use each?**

- A) They are identical — helm upgrade --install is just a longer version of install
- B) helm install fails if the release already exists; helm upgrade --install installs if not present, upgrades if already deployed
- C) helm upgrade --install always deletes and recreates the release
- D) helm install is for development; helm upgrade --install is for production only

> **Answer/explanation:** ✅ **Answer: B.** `helm install` fails with "release already exists" if you run it twice. In CI/CD pipelines, you don't know if this is the first deploy or the tenth — so `helm upgrade --install` is the correct idempotent command for pipelines. It installs on first run, upgrades on subsequent runs. CloudBridge uses `helm upgrade --install` exclusively in CI/CD so the pipeline works regardless of whether the environment already has the release deployed.

##### Common Fresher Questions — Helm at CloudBridge

**Q: Q: When I run helm upgrade, which values win — values.yaml or values-prod.yaml?**

A: The last -f file wins on conflicts. When you run `helm upgrade ... -f values.yaml -f values-prod.yaml`, Helm deep-merges them left-to-right. values-prod.yaml is applied last, so its keys override values.yaml. `--set` flags have the highest priority — they override both files. Think of it as: defaults → environment file → CLI flags.

**Q: Q: Helm rollback sounds dangerous — does it also rollback the database migration?**

A: No — and this is critical. Helm rollback only reverts Kubernetes resources (Deployment, ConfigMap, Service, etc.) to a previous revision. It does NOT rollback database schema changes. Database migrations are forward-only operations. This is why pre-upgrade migration hooks must be written as backward-compatible migrations — the rolled-back application code must still work with the already-migrated schema. Always write non-destructive, additive migrations.

**Q: Q: Can I have two different releases of the same chart in the same cluster?**

A: Yes — this is a core Helm feature. Run `helm install invoicex-v1 ./invoicex -f values-v1.yaml -n client-a` and `helm install invoicex-v2 ./invoicex -f values-v2.yaml -n client-b`. Different release names, different namespaces, different values — same chart. CloudBridge does this for multi-tenant deployments where each enterprise client runs their own InvoiceX instance with different configs.

**Q: Q: What is the difference between helm.sh/hook-weight and hook execution order?**

A: When multiple hooks fire at the same lifecycle event (e.g., two pre-upgrade Jobs), Helm sorts them by weight (ascending — lower weight runs first). A DB migration hook at weight -10 runs before a data-seed hook at weight 0 at the same pre-upgrade event. All hooks must complete successfully before Helm proceeds with the main release resources. Hooks with the same weight run in alphabetical order by resource name.

### Quick Reference — All Helm Commands at CloudBridge

Command

What It Does

helm create CHART

Scaffold a new chart with default templates

helm install NAME ./CHART -f values.yaml

First-time deploy of a chart

helm upgrade NAME ./CHART -f values.yaml

Upgrade an existing release

helm upgrade --install NAME ./CHART

Idempotent install-or-upgrade (use in CI/CD)

helm list -n NAMESPACE

List all releases in a namespace

helm status NAME

Show status and NOTES.txt for a release

helm get values NAME

Show the values used for a deployed release

helm get manifest NAME

Show the rendered Kubernetes manifests of a release

helm history NAME

Show full revision history of a release

helm rollback NAME REVISION

Roll back a release to a specific revision

helm uninstall NAME

Delete a release and all its resources

helm template NAME ./CHART

Render templates to stdout without deploying

helm lint ./CHART

Validate chart structure and templates

helm test NAME

Run the test pods for a deployed release

helm package ./CHART

Package chart into a .tgz archive

helm repo add NAME URL

Register a chart repository

helm repo update

Refresh chart index from all repositories

helm search repo KEYWORD

Search registered repositories for charts

helm dependency update ./CHART

Download declared subchart dependencies

helm dependency list ./CHART

List subchart dependencies and their status

helm show values CHART

Display the default values.yaml of a chart

helm plugin install URL

Install a Helm plugin (e.g., helm-diff)

helm diff upgrade NAME ./CHART

Show diff between current and new state (plugin)

### Helm Charts Mastery Complete 🎉

You have built CloudBridge's complete Helm-based deployment framework for InvoiceX — chart structure, values layering, templates with Go functions and conditionals, helpers, secrets management, lifecycle hooks for database migrations, subchart dependencies for PostgreSQL and Redis, chart repositories, CI/CD integration, and helm test for automated post-deploy verification.

> **Priya**
> 
> "Six months ago this team spent 45 minutes on every production deploy — manually editing YAML, praying we didn't miss a file, and crossing fingers nothing broke. Last Tuesday's release: 8 minutes, zero manual steps, one rollback executed in 43 seconds when we caught a bug, and zero config mismatches between staging and prod. That's Helm. One chart. Three value files. Six environments. Every deploy is identical to the last — predictable, reversible, and auditable."

> **Kiran**
> 
> "And the pre-install migration hook — that was the game-changer. Database schema and application code are always in sync. The hook fails, the deploy fails, old code keeps running. No more 'app is up but can't find the new column because the migration didn't run' incidents. Production is not a place for surprises. Helm gives us predictability. The rest is just YAML."

> **Next: Advanced Helm — Library Charts, OCI Registries, ArgoCD Integration & Helmfile**

> - Library charts — create a shared chart with common helpers (fullname, labels, resource templates) used by all CloudBridge microservices, so naming conventions and label standards are enforced at the library level
> - Helmfile — declarative configuration for multiple Helm releases across multiple clusters and namespaces — manage all 6 CloudBridge microservices in one helmfile.yaml
> - ArgoCD + Helm — GitOps-style deployment where ArgoCD watches Git for chart changes and auto-syncs to Kubernetes — no CI/CD helm commands needed
> - OCI registries — store Helm charts in AWS ECR or Azure Container Registry alongside Docker images, using `helm push` and `helm install oci://`
> - Schema validation — add values.schema.json to your chart so Helm validates that required values are present and correctly typed before every install
> - Helm Secrets plugin — transparent encryption/decryption of secrets in values files using Mozilla SOPS with AWS KMS or Azure Key Vault — secrets in Git, encrypted at rest
