# ⎈ Helm Project Mastery

> **👋 Hey Fresher — Read This First!**

> Helm is the package manager for Kubernetes — think of it as `apt` or `npm`, but for deploying applications onto a cluster. Without Helm, deploying an app to three environments (dev, staging, prod) means maintaining three nearly-identical sets of raw YAML files, hand-editing replica counts and image tags in each, and hoping nobody copy-pastes a change into the wrong file. Helm solves this with **charts** — a templated package of your Kubernetes manifests plus a `values.yaml` file holding the things that actually change per environment. One chart, multiple environments, one command to install, one command to upgrade, one command to roll back when something breaks.

> **Company in this project:** LearnOrbit — an edtech startup in Delhi running an online mock-test and exam-prep platform for students preparing for competitive exams. Their traffic is brutally spiky: calm most of the year, then 20x during the two weeks before a major exam registration deadline. Right now, every deploy means an engineer manually editing raw YAML files for whichever environment they're targeting — and last exam season, someone accidentally applied a dev-sized replica count to production during peak signup traffic. You just joined as a Junior Platform Engineer. Your mentor is Radhika, and the engineering manager pushing for a proper release process is Deepak. Let's turn LearnOrbit's raw manifests into a real Helm chart.

#### What You Will Learn and Build in This Project

You will convert LearnOrbit's raw Kubernetes manifests into a fully templated, versioned Helm chart, parameterize everything that changes between environments, install and upgrade the app through Helm's release lifecycle, deliberately break a release and roll it back, and finally add a real chart dependency so LearnOrbit's Redis-backed session cache ships bundled with the application chart itself.

Helm Charts, Chart.yaml, values.yaml, Go Templating, Helm Releases, helm upgrade, helm rollback, Chart Dependencies, Subcharts, Helm Repositories

> **📦 Phase 1 — From Raw YAML to a Chart**
>
> Scaffold a Helm chart with `helm create`, understand the anatomy of a chart, and move LearnOrbit's existing manifests into it.

> **📦 Phase 2 — Templating with values.yaml**
>
> Replace every hardcoded value — image tag, replica count, resource limits — with a template variable pulled from `values.yaml`.

> **📦 Phase 3 — Environment-Specific Overrides**
>
> Create `values-dev.yaml` and `values-prod.yaml` so the exact same chart deploys very differently to each environment, safely.

> **📦 Phase 4 — Install and Upgrade**
>
> Install the chart for real, understand what a Helm release actually is, and perform a live upgrade.

> **📦 Phase 5 — Break It on Purpose, Then Roll Back**
>
> Ship a broken image tag deliberately, watch the upgrade fail, and recover instantly with `helm rollback` — no manual YAML archaeology required.

> **📦 Phase 6 — Add a Chart Dependency**
>
> Bundle a Redis subchart into LearnOrbit's chart so the session cache installs and upgrades together with the app, as one release.

**Scene 1 — LearnOrbit Engineering Office, Delhi | The Dev-Sized Prod Deploy**

> **Deepak** _Engineering Manager — LearnOrbit_
>
> Ishaan, let me tell you why this project matters before we write a single template. Three weeks before the last major exam registration deadline, an engineer meant to deploy a hotfix to production. He had two YAML files open in adjacent tabs — `deployment-dev.yaml` with `replicas: 1`, and `deployment-prod.yaml` with `replicas: 12`. He ran `kubectl apply` against the wrong file, against production, during the exact week we get 20 times our normal traffic. Production dropped from 12 replicas to 1, right as 40,000 students were trying to register.

> **Radhika** _Senior Platform Engineer — LearnOrbit_
>
> The scary part isn't that he made a mistake — anyone can fat-finger a filename. The scary part is that our process made that mistake *easy* to make and *hard* to catch. Two YAML files that look almost identical, differing in one line, with no safety rail between "I meant to run this against dev" and "I actually ran it against prod." Helm fixes this at the structural level: one chart, one set of templates, and the environment-specific differences live in small, clearly-named values files that get passed explicitly on every command.

> **Ishaan (You)** _Junior Platform Engineer — Day 1_
>
> So Helm isn't just about saving typing — it's about making the dangerous mistake structurally harder to make?

> **Radhika** _Senior Platform Engineer_
>
> Exactly right. And as a bonus, you get versioned releases, one-command rollbacks, and reusable packages you can share across services. But the mistake-prevention part is why we're doing this the week after that incident, not six months from now.

### 1. Phase 1 — From Raw YAML to a Chart

**Business Problem:** LearnOrbit's mock-test platform currently has a `deployment.yaml`, `service.yaml`, and `ingress.yaml`, each duplicated by hand across `dev/` and `prod/` folders. There's no single source of truth for "what does the LearnOrbit app look like as a Kubernetes object" — just multiple, slowly diverging copies.

#### 1.1 Scaffolding the Chart

```bash
helm create learnorbit-app

# Generated structure:
learnorbit-app/
├── Chart.yaml              # Chart metadata: name, version, appVersion
├── values.yaml              # Default configuration values
├── charts/                  # Subchart dependencies live here (Phase 6)
├── templates/
│   ├── deployment.yaml       # Templated Deployment
│   ├── service.yaml          # Templated Service
│   ├── ingress.yaml           # Templated Ingress
│   ├── _helpers.tpl           # Reusable template snippets (naming, labels)
│   ├── NOTES.txt              # Printed to the user after install/upgrade
│   └── tests/
│       └── test-connection.yaml
└── .helmignore
```

> **📖 What helm create Actually Gives You**
>
> `helm create` scaffolds a working, opinionated starter chart — it's not empty boilerplate, it's a fully functional (if generic) NGINX deployment you're meant to replace piece by piece with your own app. `_helpers.tpl` deserves attention early: it defines reusable named templates (like `learnorbit-app.fullname` and `learnorbit-app.labels`) that every other template file calls, so your resource naming and labeling stays consistent across `deployment.yaml`, `service.yaml`, and `ingress.yaml` without repeating the same logic three times.

#### 1.2 Chart.yaml — The Chart's Identity

```yaml
# learnorbit-app/Chart.yaml
apiVersion: v2
name: learnorbit-app
description: LearnOrbit mock-test platform — API, worker, and web frontend
type: application
version: 1.0.0
appVersion: "2.4.1"
maintainers:
  - name: LearnOrbit Platform Team
    email: platform@learnorbit.in
```

> **📖 version vs appVersion — The Distinction Freshers Always Mix Up**
>
> `version: 1.0.0` is the **chart's** version — it goes up every time you change the templates or values structure (adding a new configurable field, restructuring `values.yaml`). `appVersion: "2.4.1"` is the version of the **application itself** that this chart deploys by default — LearnOrbit's actual Docker image tag. These two numbers are completely independent: you can release chart version `1.3.0` that still deploys app version `2.4.1` (a templating improvement with no app change), or keep chart version `1.0.0` and bump `appVersion` to `2.5.0` for every app release without changing a single template. Confusing the two is one of the most common Helm mistakes on a team.

### 2. Phase 2 — Templating with values.yaml

**Business Problem:** LearnOrbit's raw manifests have `replicas: 3`, `image: learnorbit/api:2.4.1`, and CPU/memory limits hardcoded directly into the YAML. Every environment-specific difference requires editing the manifest file itself — exactly the pattern that caused the incident in Scene 1.

**Scene 2 — Pairing Session | Turning Hardcoded Values into Variables**

> **Radhika** _Senior Platform Engineer_
>
> Here's the rule I want you to apply to every line of the raw Deployment: if this value would ever need to be different between dev and prod, it becomes a template variable pulled from `values.yaml`. If it would never change — like the container port the app listens on — it can stay hardcoded in the template itself.

#### 2.1 Templating the Deployment

```yaml
# learnorbit-app/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "learnorbit-app.fullname" . }}
  labels:
    {{- include "learnorbit-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "learnorbit-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "learnorbit-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

> **📖 Reading the Template Syntax Line by Line**
>
> `{{ include "learnorbit-app.fullname" . }}` calls the named template defined in `_helpers.tpl`, passing `.` (the entire template context) into it — this is what generates a consistent resource name like `learnorbit-app-prod` without repeating naming logic in every file. `{{ .Values.replicaCount }}` pulls straight from `values.yaml` — no hardcoded `3` anywhere in this file anymore. `{{ .Values.image.tag | default .Chart.AppVersion }}` is a Go template pipeline: if `image.tag` isn't set in `values.yaml`, it falls back to `Chart.AppVersion` from `Chart.yaml` — meaning a values file doesn't even need to specify an image tag unless it wants to override the chart's default app version. `{{- toYaml .Values.resources | nindent 12 }}` converts a nested YAML object from `values.yaml` directly into properly indented YAML in the template — this is how you template entire blocks, not just single strings.

#### 2.2 The Corresponding values.yaml

```yaml
# learnorbit-app/values.yaml
replicaCount: 3

image:
  repository: learnorbit/api
  tag: ""            # falls back to Chart.appVersion if empty
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: false
  host: ""
```

> **📖 values.yaml Is the Contract, Not Just Defaults**
>
> This file is what every future engineer reads to understand *what's actually configurable* about the LearnOrbit chart — it's effectively the chart's public API. `image.tag: ""` deliberately left empty documents the fallback-to-appVersion behavior explicitly, rather than leaving a future reader to guess why no tag is set. Every top-level key here (`replicaCount`, `image`, `service`, `resources`, `ingress`) corresponds to a real decision an environment might need to make differently — which is exactly the set of values Phase 3's environment override files will target.

**Helm Templating vs Plain kubectl apply with Raw YAML**

- **Raw YAML + kubectl apply** — simplest possible starting point, no extra tooling, but every environment difference means either maintaining fully duplicated files (LearnOrbit's original problem) or reaching for external tools like `envsubst` or `sed` to patch values in — fragile, and with no built-in versioning or rollback.
- **Helm chart** — more upfront structure (templates + values), but one chart genuinely serves every environment, differences live in small override files reviewed easily in a pull request, and every install/upgrade is a tracked, versioned release you can list, inspect, and roll back with a single command.

### 3. Phase 3 — Environment-Specific Overrides

**Business Problem:** Dev needs to run cheap and small — 1 replica, minimal resources, no custom domain. Production needs to run at real scale — 12+ replicas during exam season, generous resource limits, and a real TLS-terminated domain. The same chart needs to produce both, safely and explicitly.

#### 3.1 values-dev.yaml — Minimal Footprint

```yaml
# environments/values-dev.yaml
replicaCount: 1

image:
  tag: "dev-latest"

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi

ingress:
  enabled: false
```

#### 3.2 values-prod.yaml — Exam Season Scale

```yaml
# environments/values-prod.yaml
replicaCount: 12

image:
  tag: "2.4.1"

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi

ingress:
  enabled: true
  host: "app.learnorbit.in"
  tls: true
```

> **📖 Override Files Only Contain What's Different**
>
> Neither `values-dev.yaml` nor `values-prod.yaml` repeats the *entire* structure of `values.yaml` — they only override the specific keys that need to differ. Anything not mentioned (like `service.targetPort`) silently falls back to the base `values.yaml` default. This is deliberate: a smaller override file is easier to review in a pull request, and it makes the actual environment-specific decisions visible at a glance instead of buried in a wall of duplicated config. `replicaCount: 12` in prod versus `replicaCount: 1` in dev is now a two-line diff between files, not a manual edit inside a much larger manifest — exactly the safety improvement Radhika described in Scene 1.

**Quiz: LearnOrbit's base `values.yaml` sets `resources.limits.memory: 512Mi`. `values-prod.yaml` only sets `resources.limits.memory: 1Gi` and does not mention `resources.limits.cpu` at all. When you run `helm install learnorbit-app . -f environments/values-prod.yaml`, what CPU limit does the production release actually get?**
- No CPU limit at all, since prod didn't specify one
- The base `values.yaml` default CPU limit (500m), because Helm merges override files on top of the base values — unspecified keys fall through to the base defaults
- The install fails because the override file is incomplete
- 1Gi, incorrectly applied to both memory and CPU

> **Answer/explanation:** The second option is correct. Helm merges values files hierarchically: the base `values.yaml` is the foundation, and any `-f` override file is deep-merged on top of it — only the specific keys present in the override file replace the base value; every key the override file doesn't mention is inherited unchanged from the base. Since `values-prod.yaml` never mentions `resources.limits.cpu`, that field falls through untouched from `values.yaml`'s `500m`. This deep-merge behavior is exactly why override files can stay small and only list what's actually different — Helm doesn't require (or even want) every value repeated in every override file, and an incomplete override file is the normal, correct pattern, not an error.

### 4. Phase 4 — Install and Upgrade

**Business Problem:** LearnOrbit needs a real, repeatable command to deploy to each environment — and needs to understand what a "release" actually is in Helm, because that concept is what makes rollback possible later.

#### 4.1 Installing the Chart

```bash
# Install into the dev namespace
helm install learnorbit-dev ./learnorbit-app \
  -f environments/values-dev.yaml \
  --namespace learnorbit-dev \
  --create-namespace

# Install into production
helm install learnorbit-prod ./learnorbit-app \
  -f environments/values-prod.yaml \
  --namespace learnorbit-prod \
  --create-namespace
```

> **📖 helm install Creates a Named Release, Not Just Resources**
>
> `learnorbit-dev` and `learnorbit-prod` are **release names** — Helm's unit of tracking. Every `helm install` or `helm upgrade` against a release name creates a new numbered **revision** of that release, and Helm stores the fully rendered manifest for every revision (by default, as a Secret in the target namespace). This history is what makes `helm rollback` possible later — Helm isn't guessing how to undo a change, it's reapplying the exact rendered manifest from a previous, known-good revision.

#### 4.2 Checking Release Status

```bash
helm list --namespace learnorbit-prod
# NAME             NAMESPACE         REVISION   STATUS     CHART                  APP VERSION
# learnorbit-prod  learnorbit-prod   1          deployed   learnorbit-app-1.0.0   2.4.1

helm status learnorbit-prod --namespace learnorbit-prod

helm get values learnorbit-prod --namespace learnorbit-prod
# Shows exactly which values are currently in effect for this release
```

> `helm get values` is genuinely useful during an incident — it shows you the *actual* effective configuration of a running release, not the base `values.yaml` defaults, which matters because it's easy to forget exactly which override file and which `--set` flags were used weeks ago when a release was first installed.

#### 4.3 Upgrading — A Real Image Bump

```bash
# LearnOrbit ships version 2.5.0 — bump the image tag for prod
helm upgrade learnorbit-prod ./learnorbit-app \
  -f environments/values-prod.yaml \
  --set image.tag=2.5.0 \
  --namespace learnorbit-prod

helm list --namespace learnorbit-prod
# REVISION now shows 2, APP VERSION reflects the new image
```

> **📖 --set vs Editing the Values File**
>
> `--set image.tag=2.5.0` overrides a single value directly on the command line, layered on top of everything from `-f environments/values-prod.yaml` — useful for a one-off change like a hotfix image bump you don't want to commit to the values file just yet. For anything meant to persist, the better practice is to actually edit `values-prod.yaml`, commit it, and run `helm upgrade` referencing the updated file — so the Git history of that file *is* the deploy history, and nobody has to reconstruct "what `--set` flags did we use three weeks ago" from memory or shell history.

> **Key takeaways**
> - A Helm release is a tracked, versioned entity — every install and upgrade creates a new revision with its own stored, fully-rendered manifest.
> - Values merge hierarchically: base `values.yaml` → `-f` override files (in the order given) → `--set` flags, each layer overriding only the keys it specifies.
> - `--set` is fine for quick one-off overrides; anything meant to be permanent belongs in a committed values file so Git history doubles as deploy history.
> - `helm get values` shows you the actual effective configuration of a running release — invaluable during an incident when nobody remembers exactly how it was deployed.

### 5. Phase 5 — Break It on Purpose, Then Roll Back

**Business Problem:** Deploys fail sometimes — a bad image tag, a typo in an environment variable, a config value that crashes the app on startup. LearnOrbit needs a fast, reliable recovery path that doesn't depend on someone remembering exactly what the previous working configuration looked like.

**Scene 3 — Simulated Incident | The Bad Tag**

> **Deepak** _Engineering Manager_
>
> Before we trust this in production, I want you to actually break it on purpose and recover, on stage, in front of the team. Confidence in a rollback procedure has to come from having actually run it, not from reading that it exists.

#### 5.1 Deliberately Shipping a Broken Image Tag

```bash
helm upgrade learnorbit-prod ./learnorbit-app \
  -f environments/values-prod.yaml \
  --set image.tag=2.6.0-does-not-exist \
  --namespace learnorbit-prod

kubectl get pods -n learnorbit-prod
# learnorbit-prod-api-7d9f8b6c9-x2k4p   0/1   ImagePullBackOff   0   45s
```

#### 5.2 Rolling Back Instantly

```bash
# Check revision history before deciding what to roll back to
helm history learnorbit-prod --namespace learnorbit-prod
# REVISION   STATUS       CHART                  APP VERSION   DESCRIPTION
# 1          superseded   learnorbit-app-1.0.0   2.4.1         Install complete
# 2          superseded   learnorbit-app-1.0.0   2.5.0         Upgrade complete
# 3          deployed     learnorbit-app-1.0.0   2.6.0         Upgrade "learnorbit-prod" failed

# Roll back to the last known-good revision
helm rollback learnorbit-prod 2 --namespace learnorbit-prod

helm history learnorbit-prod --namespace learnorbit-prod
# REVISION   STATUS       ... DESCRIPTION
# 4          deployed     ... Rollback to 2
```

> **📖 Why helm rollback Is Safer Than Manually Reapplying Old YAML**
>
> `helm rollback learnorbit-prod 2` doesn't ask anyone to remember or reconstruct what revision 2 looked like — Helm already has the exact, fully-rendered manifest for revision 2 stored from when it was originally deployed, and it reapplies that precisely. Note that the rollback itself creates a **new** revision (4) rather than deleting revision 3 — Helm's history is append-only, so even a broken deploy and its subsequent rollback remain visible in `helm history` as a permanent, honest record of what happened. This matters for postmortems: you can see exactly when the bad deploy went out and exactly when it was reverted, without anyone having to reconstruct the timeline from memory or Slack messages.

**Quiz: After the rollback in section 5.2, an engineer asks "should we delete revision 3, since it was broken and we don't want it cluttering the history?" What's the correct answer?**
- Yes, delete it immediately to keep the history clean
- No — Helm's revision history is meant to be a permanent, honest record; a failed revision followed by a rollback is valuable postmortem evidence, and `helm history` is append-only by design
- Only delete it if the broken deploy caused a customer-facing incident
- Revisions are automatically deleted after 24 hours regardless

> **Answer/explanation:** The second option is correct. Helm intentionally does not provide an easy way to delete an individual revision from history, because that history is meant to function as an audit trail — exactly the kind of record a postmortem needs to establish precisely when a bad deploy went out, what changed, and when it was corrected. Deleting the "embarrassing" failed revision would erase evidence that's actually useful for understanding what went wrong and preventing it next time. (Helm does let you cap how many revisions are *retained* going forward using `--history-max` on install/upgrade, which trims very old history for storage reasons — but that's different from selectively deleting one specific revision because it looked bad.)

### 6. Phase 6 — Add a Chart Dependency: Bundling Redis

**Business Problem:** LearnOrbit's API uses Redis to cache exam question sets and store user session state during a mock test. Right now, Redis is deployed completely separately, with its own manual `kubectl apply`, disconnected from the app's release lifecycle — meaning a `helm rollback` of the app doesn't roll back Redis, and there's no single command that installs "LearnOrbit, fully working" from nothing.

#### 6.1 Declaring the Dependency in Chart.yaml

```yaml
# learnorbit-app/Chart.yaml (updated)
apiVersion: v2
name: learnorbit-app
version: 1.1.0
appVersion: "2.5.0"
dependencies:
  - name: redis
    version: "19.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```bash
# Add the Bitnami repo and pull the dependency down locally
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm dependency build learnorbit-app/
# Downloads redis-19.x.x.tgz into learnorbit-app/charts/
```

> **📖 Subcharts — A Chart Inside Your Chart**
>
> `dependencies` in `Chart.yaml` declares that this chart depends on the public Bitnami Redis chart at a specific version range. `condition: redis.enabled` means the entire Redis subchart only actually deploys if `redis.enabled: true` is set in `values.yaml` — useful because a staging environment might point at a shared external Redis instance instead of running its own. `helm dependency build` resolves and downloads the actual chart archive into `charts/redis-19.x.x.tgz`, which now gets installed and upgraded as part of the exact same `helm install`/`helm upgrade` command as the LearnOrbit app itself — one release, two components, one lifecycle.

#### 6.2 Configuring the Redis Subchart from the Parent's values.yaml

```yaml
# learnorbit-app/values.yaml (additions)
redis:
  enabled: true
  auth:
    enabled: true
    existingSecret: learnorbit-redis-auth
  master:
    persistence:
      enabled: true
      size: 4Gi

# The app itself now points at the subchart's generated Service name
env:
  REDIS_HOST: "{{ .Release.Name }}-redis-master"
  REDIS_PORT: "6379"
```

> **📖 Values Namespacing for Subcharts**
>
> Any key nested under `redis:` in the parent chart's `values.yaml` gets passed straight into the Redis subchart as if it were that subchart's own top-level `values.yaml` — this is how the Bitnami Redis chart's own documented options (`auth.enabled`, `master.persistence.size`, and so on) become configurable from LearnOrbit's single values file, without needing to fork or modify the Bitnami chart at all. `existingSecret: learnorbit-redis-auth` tells the subchart to use a Secret LearnOrbit's own team manages, rather than letting the subchart auto-generate a random password — important because LearnOrbit's Redis password needs to be a tracked, rotatable credential, not an opaque auto-generated value nobody has a record of.

**Bundled Subchart vs Fully Separate Redis Deployment**

- **Bundled as a chart dependency** — one `helm install`/`helm upgrade` deploys and versions the app and Redis together as a single release; a `helm rollback` reverts both consistently. Right choice when Redis is dedicated to this one app and its lifecycle should genuinely move in lockstep with it — exactly LearnOrbit's session-cache use case.
- **Fully separate Redis deployment/chart** — Redis has its own independent release, its own upgrade schedule, and can be shared across multiple apps. The right choice for a shared platform-level datastore multiple teams depend on, where you don't want one app's rollback to accidentally roll back a Redis instance three other services also rely on.

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Add a values schema:** Write a `values.schema.json` for the LearnOrbit chart that enforces `replicaCount` must be a positive integer and `image.tag` must be a non-empty string, and confirm `helm install` rejects an invalid override file before it ever reaches the cluster.
2. **Package and publish the chart:** Run `helm package learnorbit-app/` to produce a versioned `.tgz`, host it in a simple chart repository (even a GitHub Pages-backed `helm repo index`), and install it with `helm install learnorbit-prod learnorbit-repo/learnorbit-app` instead of a local path.
3. **Add a Helm hook:** Add a `pre-upgrade` hook (a Kubernetes Job annotated with `helm.sh/hook: pre-upgrade`) that runs the app's database migration before the new Deployment revision rolls out, and confirm it blocks the upgrade if the migration fails.
4. **Write a `helm test`:** Add a test pod under `templates/tests/` that curls the app's `/health` endpoint after install, and run `helm test learnorbit-prod` to confirm the release is actually healthy, not just "deployed" according to Kubernetes.
5. **Simulate exam-season scaling via values:** Create a `values-exam-week.yaml` that bumps `replicaCount` to 30 and raises resource requests, and practice switching production between normal and exam-week profiles with a single `helm upgrade -f` command, then switching back afterward.

### Helm Project Complete 🎉

You converted LearnOrbit's raw, hand-duplicated Kubernetes manifests into a single, properly templated Helm chart — parameterized with `values.yaml`, deployed differently to dev and production through small override files, installed and upgraded as tracked releases, and recovered from a deliberately broken deploy with a one-command rollback. You also bundled Redis in as a real chart dependency, so the app and its session cache now install, upgrade, and roll back together as one release.

> **Deepak**
>
> "The incident that started this project was a dev-sized replica count hitting production during our highest-traffic week. I just watched you run `helm upgrade learnorbit-prod -f environments/values-prod.yaml` — there's no longer a file called `deployment-dev.yaml` sitting one tab away that anyone could accidentally apply. The chart made the dangerous version of this mistake structurally impossible, not just less likely."

> **Radhika**
>
> "And the rollback drill — going from a broken `ImagePullBackOff` in prod back to a known-good, fully working state in under thirty seconds, with one command, no YAML archaeology — that's the payoff for all the templating discipline we put in earlier. `helm history` remembered what we couldn't."

> **Ishaan (You)**
>
> "The version-vs-appVersion distinction tripped me up at first, and the values-merging order took a minute to really trust. But once it clicked, I stopped thinking of Kubernetes YAML as files I edit and started thinking of it as a template plus configuration — which is exactly the mental model that makes multi-environment deploys stop being scary."

> **Next: GitOps with Argo CD — Deploying Helm Charts from Git**

> - Argo CD Application resources that point at this exact Helm chart and values files, syncing the cluster to match Git automatically
> - Automated sync policies — deploy on every merge to main, with self-healing if someone manually `kubectl edit`s a resource out of drift
> - Progressive delivery — canary and blue-green rollouts layered on top of the Helm release you just built
> - Multi-cluster Helm deployments — the same chart, deployed consistently across LearnOrbit's dev, staging, and production clusters via GitOps
> - Helm chart testing in CI — running `helm lint` and `helm template` validation on every pull request that touches the chart
