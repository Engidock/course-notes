# ☸️ Advanced Kubernetes Project Mastery

### The CloudNative Platform — Continuation of Kubernetes Project

**Continuation:** In the basic Kubernetes Project, you learned core concepts: pods, deployments, services, persistent volumes. You deployed a monolithic application. Now you're scaling: 100+ microservices, 50+ teams, production grade security, automated deployments via Git.

**The Challenge:** Managing 100 YAML files manually is chaos. Developers need isolation (RBAC). Stateful applications (databases, caches) need special handling (StatefulSets). Changes must be audit-tracked and reversible. Every cluster must be identical.

**The Solution:** Helm for templating and packaging. RBAC for fine-grained access control. StatefulSets for stateful workloads. GitOps with Argo CD for declarative, Git-driven deployments. All changes in Git, no manual kubectl apply.

**📍 Scene: CloudNative's Kubernetes at Scale**

> **Vikram (Platform Lead)**
> 
> "We have 12 microservices, 50 developers, 3 environments (dev, staging, prod). Each service needs different config per environment. Manually editing YAML files is error-prone. Someone accidentally changed prod config for dev. Service crashed in production."

> **Priya (DevOps)**
> 
> "We implement Helm. Each service is a Helm chart. Chart has templates for deployment, service, ingress. We pass values.yaml per environment. helm install --values dev-values.yaml deploys to dev, same chart, different config. No copy-paste, no manual editing."

> **Amit (Security)**
> 
> "And security? Developers shouldn't access production. How do we enforce that?"

> **Priya**
> 
> "RBAC. We create ServiceAccounts for each team. Grant them ClusterRoles scoped to their namespace only. Team A can deploy to dev-a namespace, but not dev-b or production. Kubernetes enforces this at the API level."

> **Vikram**
> 
> "And deployments? How do we ensure consistency across all 3 environments?"

> **Priya**
> 
> "GitOps with Argo CD. Git repo has Helm values for all environments. Argo watches Git. When you push a commit, Argo automatically syncs the cluster to match Git. Deploy by pushing code. Rollback by reverting commit. All changes audited in Git history."

### 1. Helm: Package Management for Kubernetes

#### What is Helm?

#### Creating a Helm Chart

```
# Create chart structure
$ helm create myapp

# Chart structure:
myapp/
├── Chart.yaml              # Chart metadata (name, version)
├── values.yaml             # Default values
├── templates/
│   ├── deployment.yaml     # Deployment template
│   ├── service.yaml        # Service template
│   ├── ingress.yaml        # Ingress template
│   └── _helpers.tpl        # Helper functions
└── README.md

# Chart.yaml
apiVersion: v2
name: myapp
version: 1.0.0
appVersion: 2.0.0
description: My microservice
```

> **Chart.yaml:** metadata. name = chart name (helm install myapp), version = chart version (released as 1.0.0, 1.0.1, etc), appVersion = app version inside chart.
**values.yaml:** default values. Override with -f prod-values.yaml or --set key=value.
**templates/:** YAML files with {{template syntax}}. Helm processes them, substitutes values, generates manifests.

#### Templating Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: app
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        resources:
          limits:
            cpu: {{ .Values.resources.limits.cpu }}
            memory: {{ .Values.resources.limits.memory }}

# values.yaml
replicaCount: 3
image:
  repository: myapp
  tag: 1.0.0
resources:
  limits:
    cpu: 500m
memory: 512Mi
# Install for dev (2 replicas)
$ helm install myapp . --values dev-values.yaml

# dev-values.yaml overrides
replicaCount: 2
```

> **{{ .Release.Name }}:** interpolation. Helm substitutes actual release name (myapp-prod, myapp-dev, etc).
**{{ .Values.replicaCount }}:** reads from values.yaml. Can override: --set replicaCount=5.
**Conditional values:** dev-values.yaml has replicaCount=2, overrides default 3 from values.yaml.
**Same chart, different config:** helm install myapp . -f prod-values.yaml deploys to prod with prod-specific config.

#### Helm Chart Versioning & Releases

```
# Package chart as tarball
$ helm package myapp/
Successfully packaged chart and saved it to: myapp-1.0.0.tgz
# Install from local or remote
$ helm install myapp ./myapp-1.0.0.tgz

# View installed releases
$ helm list
NAME     NAMESPACE   STATUS      CHART           APP VERSION
myapp    default     deployed    myapp-1.0.0     2.0.0
# Upgrade release (new chart version)
$ helm upgrade myapp ./myapp-1.1.0.tgz

# Rollback to previous release
$ helm rollback myapp 1
```

> **helm package:** creates tarball. Tarballs are immutable releases. Chart 1.0.0 never changes.
**helm list:** shows all releases. STATUS = deployed, pending, failed. Track what's running.
**helm upgrade:** updates running release to new chart version. Rolls out new pods.
**helm rollback:** reverts to previous release (revision 1). All pods roll back to old version.

### 2. RBAC: Secure Access Control

#### RBAC Components

#### Creating RBAC Policies

```yaml
# ServiceAccount for team-a
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-a-deployer
  namespace: team-a

# Role: permissions for team-a namespace only
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: team-a
rules:
- apiGroups: [""]
  resources: [pods, services, configmaps]
  verbs: [get, list, create, delete]

- apiGroups: [apps]
  resources: [deployments]
  verbs: [get, list, create, update, delete]

# Bind Role to ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: team-a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: deployer
subjects:
- kind: ServiceAccount
  name: team-a-deployer
  namespace: team-a
```

> **Role rules:** apiGroups (which API group), resources (pods, deployments, etc), verbs (get, list, create, delete, update).
**Namespace scoped:** this Role only applies to team-a namespace. team-a-deployer cannot access other namespaces.
**Verbs allowed:** get, list, create, delete, update. Does NOT include '*' (all verbs). Limited permissions.
**RoleBinding:** connects Role to ServiceAccount. Multiple ServiceAccounts can bind to same Role.

#### RBAC in Practice

```
# Test RBAC: Can team-a-deployer delete pods in team-a?
$ kubectl delete pod mypod --as=system:serviceaccount:team-a:team-a-deployer -n team-a
pod "mypod" deleted
# Can team-a-deployer access production namespace?
$ kubectl get pods -n production --as=system:serviceaccount:team-a:team-a-deployer
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:team-a:team-a-deployer" cannot list resource "pods" in API group "" in the namespace "production"
# RBAC enforced! team-a can only access team-a namespace.
```

> **--as flag:** kubectl runs command as different ServiceAccount. Useful for testing RBAC without changing users.
**Allowed:** delete pod in team-a (role grants delete verb).
**Forbidden:** list pods in production (user not bound to any role in production namespace).
**Kubernetes enforces at API level:** no kubectl tricks, no shell access can bypass RBAC.

### 3. StatefulSets: Stateful Workloads

#### Deployment vs StatefulSet

#### MySQL StatefulSet Example

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
  replicas: 3
selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 20Gi
# Headless Service (required for StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None  # Headless: no virtual IP
selector:
    app: mysql
  ports:
  - port: 3306
```

> **serviceName: mysql-headless:** references headless service. Required for DNS names.
**replicas: 3:** creates mysql-0, mysql-1, mysql-2 (ordered names, not random).
**volumeClaimTemplates:** each pod gets its own PVC. mysql-0 has data-0, mysql-1 has data-1, etc. Persistent across restarts.
**Headless service (clusterIP: None):** no load balancer. Each pod has stable DNS: mysql-0.mysql-headless.default.svc.cluster.local.
**Scale to 3 pods:** MySQL primary (mysql-0) + 2 replicas. Replication configured in init script.

#### Scaling and Rollout

```
# Scale from 3 to 5 replicas (orderly)
$ kubectl scale statefulset mysql --replicas=5
mysql-0: running (unchanged)
mysql-1: running (unchanged)
mysql-2: running (unchanged)
mysql-3: creating... (waits for mysql-2 ready)
mysql-4: creating... (waits for mysql-3 ready)
# Ordered: mysql-3 waits for mysql-2, mysql-4 waits for mysql-3
# If mysql-2 fails, mysql-3 blocks. Prevents corruption.
# Update image (rolling update)
$ kubectl set image statefulset mysql mysql=mysql:8.1
mysql-4: updating to mysql:8.1
mysql-3: updating to mysql:8.1
mysql-2: updating to mysql:8.1
mysql-1: updating to mysql:8.1
mysql-0: updating to mysql:8.1
# Reverse order: highest index first (mysql-4 → mysql-0)
# Avoids primary (mysql-0) being updated first
```

> **Ordered scale-up:** mysql-3 doesn't start until mysql-2 is Ready. Prevents data loss.
**Ordered scale-down:** if scaling to 3, mysql-4 is deleted first, then mysql-3. Reverse order.
**Rolling updates:** reverse order (mysql-4 → mysql-0). Primary is updated last, stays stable longest.
**Predictable pod names:** deployments have random names (pod-abc123), statefulsets have deterministic (mysql-0, mysql-1).

### 4. GitOps with Argo CD: Declarative Deployments

#### What is GitOps?

```
    GitOps Workflow

        Developer: push code to Git
                        ↓
        GitHub webhook → Argo CD detects change
                        ↓
        Argo reads Helm values from Git repo
                        ↓
        Argo applies to Kubernetes cluster
                        ↓
        Cluster state matches Git
        
        Rollback: revert commit on GitHub
                        ↓
        Argo detects Git revert
                        ↓
        Argo applies previous manifest
                        ↓
        Cluster rolls back automatically
        
```

#### Installing Argo CD

```
# Install Argo CD
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access Argo UI
$ kubectl port-forward svc/argocd-server -n argocd 8080:443

# Default password
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Login to http://localhost:8080
```

> **argocd namespace:** isolates Argo CD components from applications.
**argocd-server:** web UI. Access via port-forward or expose via LoadBalancer.
**Initial password:** stored in secret, decoded with base64 -d.

#### Creating an Argo CD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
spec:
  project: default
  
  # Git source
source:
    repoURL: https://github.com/company/apps
    path: myapp/helm
    plugin:
      name: helm
      env:
      - name: HELM_VALUES_FILES
        value: prod-values.yaml
  
  # Target cluster
destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  
  # Sync policy (automatic)
syncPolicy:
    automated:
      prune: true      # Delete resources not in Git
selfHeal: true   # Fix drift from cluster
syncOptions:
    - CreateNamespace=true
```

> **source.repoURL:** Git repository URL. Argo pulls from here.
**source.path:** directory in repo containing Helm chart. myapp/helm/values.yaml.
**HELM_VALUES_FILES: prod-values.yaml:** Argo passes this to helm. Override default values.yaml with prod config.
**destination.server:** https://kubernetes.default.svc = local cluster. Can also deploy to remote clusters.
**automated.prune: true:** if a resource is deleted from Git, Argo deletes it from cluster too.
**automated.selfHeal: true:** if someone manually deletes a pod, Argo recreates it (restores Git state).

#### GitOps Workflow: Code to Cluster

```bash
# Step 1: Developer commits change to Git
$ git checkout -b feature/increase-replicas
# Edit myapp/helm/prod-values.yaml: replicaCount: 3 → 5
$ git add myapp/helm/prod-values.yaml
$ git commit -m "Scale to 5 replicas"
$ git push origin feature/increase-replicas
# Create PR on GitHub
# Step 2: PR review + approval
# Team lead approves: looks good, scales production safely
# Step 3: Merge to main
$ git merge feature/increase-replicas --squash
$ git push origin main
# GitHub webhook sends notification to Argo CD
# Step 4: Argo CD detects change (automatic)
Argo CD Application Status:
- Git revision: abc1234 (latest)
- Cluster state: 3 replicas (out of sync)
- Sync status: OutOfSync
- Action: Auto-syncing...
# Step 5: Argo applies Helm chart with new values
Syncing myapp-prod...
helm install myapp ./helm -f prod-values.yaml
- Deployment: myapp-prod scaled 3 → 5 replicas
Status: Synced ✓
# Step 6: Production cluster now has 5 replicas
$ kubectl get deployment myapp-prod -n myapp
NAME             DESIRED   CURRENT   READY
myapp-prod       5         5         5
```

> **PR-based workflow:** all changes go through Git. Code review before production deployment.
**No kubectl apply:** no manual cluster changes. All changes from Git.
**Automatic sync:** Argo watches Git. Any change triggers auto-sync to cluster.
**Audit trail:** Git history shows who made what change when. Can git blame any resource.
**Easy rollback:** git revert commit, Argo auto-syncs, cluster reverts.

### 5. Advanced Patterns: Multi-Environment, Kustomize, Secrets

#### Multi-Environment with Helm

```yaml
# Git repo structure:
apps/
├── myapp/
│   ├── helm/
│   │   ├── Chart.yaml
│   │   ├── values.yaml           # Default values
│   │   └── templates/
│   │
│   └── environments/
│       ├── dev-values.yaml        # dev: 1 replica, small resources
│       ├── staging-values.yaml    # staging: 2 replicas, medium
│       └── prod-values.yaml       # prod: 5 replicas, large
# Argo Application: dev
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
spec:
  source:
    repoURL: https://github.com/company/apps
    path: myapp/helm
    plugin:
      env:
      - name: HELM_VALUES_FILES
        value: ../environments/dev-values.yaml
  destination:
    server: https://dev-cluster:6443      # dev cluster
namespace: myapp

# Argo Application: prod (same app, different env)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
spec:
  source:
    path: myapp/helm
    plugin:
      env:
      - name: HELM_VALUES_FILES
        value: ../environments/prod-values.yaml
  destination:
    server: https://prod-cluster:6443    # prod cluster
namespace: myapp
```

> **One helm chart, multiple values files:** dev-values.yaml, staging-values.yaml, prod-values.yaml.
**Different target clusters:** dev-cluster, prod-cluster. Same Application CRD, different destination.
**Different resource limits per env:** dev: 100m CPU, prod: 2 CPU. Defined in values files.
**Git controls all environments:** single source of truth. Change prod-values.yaml, commit, Argo syncs prod cluster.

#### Sealed Secrets: Encrypting Secrets in Git

```yaml
# Create secret locally (don't commit!)
$ kubectl create secret generic db-password \
  --from-literal=password=mysecretpass \
  --dry-run=client -o yaml | \
  kubeseal -f - > sealed-secret.yaml

# sealed-secret.yaml (safe to commit)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-password
spec:
  encryptedData:
    password: AgBvx1234...encrypted...
  template:
    type: Opaque

# Argo detects SealedSecret in Git, applies to cluster
# SealedSecret controller decrypts it using sealing key
# Kubernetes Secret created automatically
$ kubectl get secrets -n myapp
NAME                    TYPE
db-password             Opaque
```

> **kubeseal:** encrypts secrets using cluster's sealing key. Encrypted text only decryptable on that cluster.
**SealedSecret:** Kubernetes custom resource. Looks like secret but encrypted.
**Safe in Git:** can commit sealed-secret.yaml to GitHub. Even if repo is compromised, encrypted secrets are useless without sealing key (on cluster only).
**SealedSecret controller:** watches for SealedSecrets, decrypts, creates Secrets. Automatic on Argo sync.

**📍 Scene: CloudNative's GitOps Success**

> **Vikram**
> 
> "We implemented Helm + RBAC + Argo CD. Now deploying is simple. Developer commits Helm values change to Git, creates PR, team reviews, merges. Argo automatically syncs. No manual kubectl commands."

> **Priya**
> 
> "And RBAC is working. Each team has its own ServiceAccount with permissions only to their namespace. Team A cannot touch Team B's pods, cannot access production. Kubernetes enforces it at the API level."

> **Amit**
> 
> "StatefulSets handle our databases. MySQL runs as StatefulSet with persistent volumes. Each pod has stable name and storage. Scale up/down is ordered. Replication works perfectly."

> **Vikram**
> 
> "Rollback is now trivial. If a release breaks production, we just revert the commit on GitHub. Argo detects it, rolls back the cluster. Takes 30 seconds. Before GitOps, rollback meant manual editing and 30 minutes of downtime."

> **Advanced Kubernetes — Core Takeaways for Freshers**

> - **Helm templates reduce YAML duplication.** One chart, multiple values files (dev/staging/prod). Same deployment logic, different config.
> - **RBAC is your security boundary.** ServiceAccounts, Roles, RoleBindings enforce permissions at API level. Developers cannot access production, even with kubectl tricks.
> - **StatefulSets are for stateful workloads.** Stable pod names, ordered rollout, persistent volumes per pod. Databases and caches need StatefulSets, not Deployments.
> - **GitOps = Git is source of truth.** All changes in Git, reviewed via PR, automatically applied by Argo. Audit trail. Easy rollback (revert commit).
> - **Sealed Secrets encrypt secrets in Git.** Safe to commit encrypted secrets. Sealing key on cluster only. Automatic decryption by Argo.
> - **Multi-environment from single chart.** One Helm chart, different values per environment (dev/staging/prod). Consistency guaranteed.
> - **Argo CD detects drift automatically.** If someone manually deletes a pod, Argo recreates it (selfHeal=true). Cluster always matches Git.
> - **No more manual kubectl apply.** All deployments via Git. Fewer operator errors, more consistency, better auditability.

##### Advanced Kubernetes Standards — CloudNative Production Rules

Always use Helm for templating. Never commit raw YAML. Charts are reusable, tested, versioned.

Implement RBAC from day one. Use namespaces + roles to isolate teams. Prevent prod-breaking mistakes.

Use StatefulSets for stateful apps. Don't force stateless pod model on databases. StatefulSets handle state correctly.

GitOps for all changes. No manual kubectl apply. All changes via Git PR → review → merge → Argo syncs. Audit trail for everything.

Never commit plain secrets. Use SealedSecrets or ExternalSecrets. Encrypt at rest, encrypt in transit.

Test Helm charts before deploying. helm template to preview, helm lint to validate, helm diff to see what changes.

Argo ApplicationSets for multi-environment. Generator-based approach. One AppSet definition = deploy to 10 environments automatically.

Monitor Argo sync status. Argo reports out-of-sync if Git diverges from cluster. Set alerts on sync failures.

##### 🏋️ Hands-On Exercises — Advanced Kubernetes

1. **Create a Helm chart for multi-environment:** Create helm chart with deployment, service, ingress templates. Write values.yaml (default), dev-values.yaml (1 replica, small resources), prod-values.yaml (5 replicas, large resources). helm template myapp . -f dev-values.yaml and helm template myapp . -f prod-values.yaml to show different outputs. helm install both versions to clusters.
2. **Set up RBAC with team isolation:** Create namespaces: team-a, team-b, production. Create ServiceAccounts and Roles for each team. team-a can only access team-a namespace, team-b can only access team-b, neither can touch production. Test with --as flag to verify RBAC works.
3. **Deploy StatefulSet with persistent data:** Create StatefulSet for PostgreSQL with 3 replicas, volumeClaimTemplates for storage, headless service. Scale up to 5 replicas, observe ordered creation (waits for previous pod). Update image, observe reverse-order rolling update (highest index first). Verify PVCs persist across updates.
4. **Implement GitOps with Argo CD:** Create Argo Application referencing Helm chart + values.yaml in Git. Push commit changing replicas: 3 → 5. Argo detects change, auto-syncs. Verify cluster scales to 5 replicas automatically. Revert commit, Argo rolls back. No manual kubectl commands.
5. **Encrypt secrets with SealedSecrets:** Install sealed-secrets controller. Create secret: kubectl create secret generic db-pass --from-literal=pass=mysecret. Seal it: kubeseal. Commit sealed-secret.yaml to Git. Argo syncs it, SealedSecret decrypts, Secret created in cluster. Verify kubectl get secret shows decrypted secret.

### Advanced Kubernetes Project Complete 🎉

You have mastered advanced Kubernetes: Helm templating for multi-environment deployments, RBAC for team isolation and security, StatefulSets for stateful workloads, and GitOps with Argo CD for declarative, Git-driven infrastructure. You can package applications with Helm, enforce security with RBAC, manage stateful databases with StatefulSets, and deploy everything via Git using Argo CD. This is enterprise-grade Kubernetes.

> **Priya**
> 
> "Advanced Kubernetes transformed how we operate. Before: manual deployments, no security isolation, database failures. After: Helm packages everything, RBAC isolates teams, StatefulSets handle state, Argo CD ensures consistency. Every deployment is auditable in Git. This is production-grade infrastructure."

> **Next: Kubernetes Observability & Service Mesh**

> - **Monitoring:** Prometheus for metrics, Grafana for dashboards. Know everything in production.
> - **Logging:** ELK stack (Elasticsearch, Logstash, Kibana) or Loki. Search logs across all pods.
> - **Tracing:** Jaeger or Datadog. Follow requests across services, see where latency is added.
> - **Service Mesh:** Istio or Linkerd. Advanced traffic management, mTLS between services, circuit breaking, retry policies.
> - **Policy Engines:** OPA/Conftest or Kyverno. Enforce policies: only specific registries, resource limits required, labels mandatory.
