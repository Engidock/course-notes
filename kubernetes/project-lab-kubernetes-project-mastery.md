# Kubernetes Project Mastery

> **👋 Hey Fresher — Read This First!**

> Kubernetes (also called K8s) is the system that companies use to run their applications in containers at scale. Docker runs one container on one machine. Kubernetes runs thousands of containers across hundreds of machines — automatically restarting crashed containers, scaling when traffic spikes, and distributing load. This document uses **short YAML blocks** — each one covers exactly one concept — with a plain-English explanation right beside it. No 80-line YAML files to get lost in. One idea at a time, always explained in simple language.

> **Company in this project:** NovaPay — a fintech startup in Mumbai processing digital payments. They have a Node.js payment API, a React frontend, and a PostgreSQL database — and they need to move from bare VMs to Kubernetes. You just joined as a Junior DevOps/Platform Engineer. Your lead is Tanvi. Let's deploy NovaPay on Kubernetes from scratch.

#### What You Will Learn and Build in This Project

You will write Kubernetes YAML manifests that deploy NovaPay's entire application stack — learning every core Kubernetes object along the way, each explained with a real business reason.

Pods, Deployments, Services, ConfigMaps, Secrets, Ingress, HPA, Namespaces, Probes

> **📦 Phase 1 — Foundations**

> Understand what Kubernetes is, install kubectl, learn the cluster architecture, and write your first Pod YAML.

> Deploy NovaPay's payment API with Deployments — multiple replicas, rolling updates, rollbacks, and resource limits.

> Expose the API inside the cluster and to the internet using ClusterIP, NodePort, LoadBalancer, and Ingress.

> Store environment variables in ConfigMaps and sensitive data (API keys, passwords) in Secrets — never in the container image.

> Add liveness and readiness probes, auto-scale with HPA, and organise environments with Namespaces.

**Scene 1 — NovaPay Engineering Office, Mumbai | Why Kubernetes?**

> **Tanvi** _Senior Platform Engineer — NovaPay_
> 
> Priya, last Friday our payment API went down for 45 minutes because the VM it was running on ran out of disk space. One VM, one failure point, entire product offline. With Kubernetes, we run three copies of the API across three different nodes. If one node dies, the other two keep serving traffic. Kubernetes detects the dead pod and starts a replacement — automatically, in under 30 seconds.

> **Priya (You)** _Junior DevOps Engineer — Day 1 at NovaPay_
> 
> I have used Docker — I know how to run a container. But Kubernetes feels overwhelming. There are so many new words: Pods, Deployments, Services, Ingress. Where do I even start?

> **Tanvi** _Senior Platform Engineer_
> 
> Start with one question: what problem does each object solve? A Pod is the smallest unit — it runs one container. A Deployment manages multiple Pods and replaces them when they crash. A Service gives the Pods a stable address so other things can reach them. An Ingress routes traffic from the internet to the right Service. Learn one at a time. By end of week, you will have our entire application deployed on the cluster.

> **Roshan** _DevOps Architect — NovaPay_
> 
> And never think of Kubernetes as just Docker with extra steps. It is a completely different mental model. You do not say "run this container on this server." You say "I want three copies of this application running at all times." Kubernetes figures out which servers to use, restarts crashed ones, and adds more when load increases. You declare the desired state. Kubernetes makes reality match it.

### 1. Phase 1 — Kubernetes Foundations

Before deploying anything, understand how a Kubernetes cluster is structured and how to talk to it with `kubectl` — the command-line tool for Kubernetes.

> **The Big Picture — What Kubernetes Actually Does**

> You tell Kubernetes what you want ("3 copies of this container, 512MB RAM each, restart if crashed") by writing YAML files. Kubernetes reads those files and does the work — choosing which server to run containers on, monitoring their health, restarting failed ones, and scaling up when traffic increases. You describe the **desired state**. Kubernetes continuously watches the **actual state** and fixes any differences. This is called the **reconciliation loop** — it's the core idea behind every Kubernetes object.

```
Kubernetes Cluster Architecture — NovaPay
==========================================

  ┌──────────────────────────────────────────────────────┐
  │                  CONTROL PLANE (brain)                │
  │  API Server ← kubectl sends commands here             │
  │  Scheduler  ← decides which node runs each pod        │
  │  etcd       ← database storing all cluster state      │
  │  Controller ← watches for crashed pods and fixes them │
  └──────────────────────┬───────────────────────────────┘
                         │ gives instructions
            ┌────────────┼────────────┐
            ▼            ▼            ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  NODE 1      │ │  NODE 2      │ │  NODE 3      │
    │  (worker VM) │ │  (worker VM) │ │  (worker VM) │
    │  kubelet ←   │ │  kubelet     │ │  kubelet     │
    │  manages pods│ │              │ │              │
    │  ┌─────────┐ │ │  ┌─────────┐│ │  ┌─────────┐ │
    │  │ Pod:API │ │ │  │ Pod:API ││ │  │ Pod:API │ │
    │  └─────────┘ │ │  └─────────┘│ │  └─────────┘ │
    └──────────────┘ └──────────────┘ └──────────────┘
    If Node 2 crashes → Scheduler creates a new pod on Node 1 or 3
```

#### 1.1 Install kubectl and Connect to the Cluster

1. Install kubectl — the CLI for Kubernetes
kubectl is the tool you use to create, inspect, update, and delete everything in the cluster.

```bash
# Install on Linux
curl -LO "https://dl.k8s.io/release/stable.txt"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
```

> kubectl is a single binary that speaks to the Kubernetes API Server. After installing, **kubectl version --client** should print something like `Client Version: v1.29.0`. For local practice, install **minikube** (one-command local cluster) or use **kind** (Kubernetes in Docker). Cloud providers (EKS on AWS, GKE on Google, AKS on Azure) give you a production cluster — kubectl connects to any of them using a `~/.kube/config` file.

2. The most important kubectl commands you'll use every day

```bash
# See all running pods
kubectl get pods

# See pods with more detail (IP, node, age)
kubectl get pods -o wide

# See everything in the cluster
kubectl get all
```

> **kubectl get** is like "ls" for Kubernetes — it lists objects. You can get pods, services, deployments, nodes, configmaps, secrets, or anything else. The **-o wide** flag shows extra columns. You will type `kubectl get pods` dozens of times per day — it shows you if your containers are running (Running), starting (Pending), or broken (CrashLoopBackOff, Error).

#### 1.2 How YAML Works in Kubernetes

Every Kubernetes object is defined in a YAML file with four required top-level fields. Learn these four fields once and they appear in every object you ever write.

```yaml
# The 4 fields every K8s YAML must have
apiVersion: v1
kind:       Pod
metadata:
  name: my-pod
spec:
  # ... what you want ...
```

**📖 The 4 Required Fields**

**apiVersion** — which API version defines this object. Pods and Services use `v1`. Deployments use `apps/v1`.  
  
**kind** — what type of object. Pod, Deployment, Service, ConfigMap, Secret, Ingress...  
  
**metadata** — information about this object: its name, namespace, and labels.  
  
**spec** — what you actually want. The spec is different for every kind — a Pod spec defines containers; a Service spec defines ports.

#### 1.3 Your First Pod

A Pod is the smallest deployable unit in Kubernetes. It runs one (or rarely more) container. In practice you almost never write a Pod directly — you use a Deployment instead — but understanding Pods first is essential.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: novapay-api-pod
labels:
    app: novapay-api
spec:
  containers:
  - name: api
image: novapay/payment-api:v1.2
ports:
    - containerPort: 3000
```

**📖 What Each Line Does**

**labels:** Key-value pairs you attach to objects. Labels are how Services find the right Pods to send traffic to — like tags on a product.  
  
**containers:** A list of containers in this Pod. Most Pods have exactly one container.  
  
**image:** The Docker image to run. Format: `registry/image:tag`. Kubernetes pulls this from Docker Hub or your private registry.  
  
**containerPort:** Documents which port the container listens on (informational — doesn't actually open the port by itself).

```bash
# Apply the YAML file to the cluster
kubectl apply -f pod.yaml

# Check if it's running
kubectl get pods

# See detailed info including events (useful for debugging)
kubectl describe pod novapay-api-pod

# See logs from the container
kubectl logs novapay-api-pod
```

> **kubectl apply -f** sends the YAML to the cluster. Kubernetes reads it, creates the pod, and schedules it on a node. **kubectl describe** is your best debugging tool — it shows the full object spec plus an **Events** section at the bottom that explains why something failed (image pull error, insufficient memory, etc.). **kubectl logs** shows the application's stdout — the same output you'd see if running the container directly with Docker.

> **💡 Fresher Tip — Why Not Just Use a Pod Directly?**

> If a Pod crashes or the node it's on goes down, Kubernetes does **not** automatically restart it or create a replacement. A bare Pod is unmanaged. That's why in production, you always use a **Deployment** — which wraps Pods in a controller that says "I want exactly 3 copies of this Pod running at all times. If one dies, create a new one." The Deployment is the real unit of work. Pods are what the Deployment creates and manages.

### 2. Phase 2 — Deployments

**Business Problem:** NovaPay's payment API must always be available. If one instance crashes, payment processing stops. If traffic doubles during a sale, one instance can't handle it. A Deployment solves both problems — it runs multiple copies and replaces crashed ones automatically.

**Scene 2 — NovaPay War Room | The Crash That Cost ₹40 Lakh**

> **Roshan** _DevOps Architect — NovaPay_
> 
> The payment API container crashed Friday at 8 PM. Peak time. We had a single Pod running it. Nobody noticed until users started complaining on Twitter. By the time we restarted it manually, we had lost 40 lakhs in failed transactions. A Deployment with three replicas would have meant the crash went unnoticed — the other two Pods kept serving traffic while Kubernetes started a replacement for the crashed one.

> **Tanvi** _Senior Platform Engineer_
> 
> Priya, the Deployment YAML is the most important file you will write. It controls how many copies run, how updates are rolled out without downtime, and how rollbacks work. Get comfortable with Deployments and you understand 80% of what a Kubernetes operator does every day.

#### 2.1 Create a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: novapay-api
spec:
  replicas: 3
selector:
    matchLabels:
      app: novapay-api
```

**📖 Deployment Basics**

**apiVersion: apps/v1** — Deployments are in the "apps" group, not the core "v1" group.  
  
**replicas: 3** — "I want exactly 3 copies of this Pod running at all times." If one crashes, Kubernetes starts a fourth to replace it immediately.  
  
**selector.matchLabels** — tells the Deployment which Pods it manages. It looks for Pods with the label `app: novapay-api`. This must match the labels in the Pod template below.

```
 template:
    metadata:
      labels:
        app: novapay-api
spec:
      containers:
      - name: api
image: novapay/payment-api:v1.2
ports:
        - containerPort: 3000
```

**📖 The Pod Template**

**template** — this is the blueprint for the Pods the Deployment creates. It looks exactly like a Pod YAML (metadata + spec), but it's nested inside the Deployment.  
  
The labels in `template.metadata.labels` must match the `selector.matchLabels` above. This is how the Deployment knows which Pods are "its" Pods. Mismatch = Deployment won't start.

#### 2.2 Resource Limits — Every Container Must Have These

```
 containers:
      - name: api
image: novapay/payment-api:v1.2
resources:
          requests:
            cpu:    "250m"
memory: "256Mi"
limits:
            cpu:    "500m"
memory: "512Mi"
```

**📖 Requests vs Limits**

**requests** = "I need at least this much." The Scheduler uses this to find a node with enough free resources. 250m CPU = 0.25 of one CPU core. 256Mi = 256 megabytes RAM.  
  
**limits** = "I must never use more than this." If the container tries to use more RAM than the limit, Kubernetes kills it (OOMKilled). If it uses more CPU, it's throttled.  
  
Without limits, one misbehaving container can consume all node resources and crash every other Pod on that node.

#### 2.3 Rolling Updates — Deploy New Versions Without Downtime

```
spec:
  replicas: 3
strategy:
    type: RollingUpdate
rollingUpdate:
      maxSurge:       1
maxUnavailable: 0
```

**📖 Zero Downtime Deployments**

**RollingUpdate** — new pods start before old ones stop. Traffic is always being served.  
  
**maxSurge: 1** — during update, allow 1 extra Pod above replicas (so 4 total during update).  
  
**maxUnavailable: 0** — never allow a Pod to go down unless a replacement is already running and healthy. Together: "add 1 new pod, wait until it's ready, then remove 1 old pod. Repeat." Zero seconds of downtime.

```bash
# Update the image to a new version
kubectl set image deployment/novapay-api api=novapay/payment-api:v1.3

# Watch the rolling update happen live
kubectl rollout status deployment/novapay-api

# Something went wrong? Roll back to the previous version instantly
kubectl rollout undo deployment/novapay-api
```

> **kubectl set image** updates the container image in the Deployment. Kubernetes starts the rolling update automatically. **kubectl rollout status** shows progress in real time — you see each pod being created and becoming ready. If the new version is broken (crashes on start), **kubectl rollout undo** instantly reverts to the previous version — Kubernetes remembers the last known good state.

### 3. Phase 3 — Services and Networking

**Business Problem:** The three API Pods each have their own private IP address — and those IPs change every time a Pod restarts. The frontend doesn't know which Pod IP to call, and the IPs keep changing. A **Service** gives the Pods one stable IP address (and DNS name) that never changes. Send traffic to the Service, and it load-balances across all healthy Pods automatically.

**Scene 3 — NovaPay Architecture Review | "How Does the Frontend Find the API?"**

> **Aditya** _Frontend Developer — NovaPay_
> 
> Wait — if Pods have their own IP addresses and those IPs change every time a Pod restarts, how does my React frontend know where to send requests? I can't hardcode an IP that changes.

> **Tanvi** _Senior Platform Engineer_
> 
> That is exactly what a Service solves. Create a Service named "novapay-api-service" and it gets a stable virtual IP that lives forever — even as Pods come and go underneath it. Your frontend calls "http://novapay-api-service:3000" — Kubernetes DNS resolves that name to the Service's stable IP, and the Service routes to whichever Pods are currently healthy. The frontend never needs to know about individual Pod IPs.

#### 3.1 ClusterIP Service — Internal Access Only

```yaml
apiVersion: v1
kind: Service
metadata:
  name: novapay-api-service
spec:
  type:     ClusterIP
selector:
    app: novapay-api
ports:
  - port:       80
targetPort: 3000
```

**📖 ClusterIP — The Default Service**

**type: ClusterIP** — only reachable inside the cluster. Other Pods can call it, but nothing outside the cluster can reach it directly. This is correct for internal APIs.  
  
**selector: app: novapay-api** — the Service finds all Pods with this label and load-balances across them.  
  
**port: 80** — the port the Service listens on.  
  
**targetPort: 3000** — the port the container actually runs on. Traffic to port 80 on the Service → port 3000 on the Pod.

#### 3.2 Service Types — Which One to Use When

**The 4 Service Types Explained Simply**

- **ClusterIP** — **Default.** Service is only reachable inside the cluster. Use for: internal APIs, databases, backend services that should never be exposed to the internet.

- **NodePort** — Exposes the Service on a port (30000–32767) on every node's IP. Use for: development/testing, quick demos. Not for production — exposes port on all nodes.

- **LoadBalancer** — Creates a cloud load balancer (AWS ALB, GCP LB) automatically. Use for: production services that need a stable external IP. One LoadBalancer per Service = expensive at scale.

- **Ingress** — One external load balancer routes to many Services based on URL path or hostname. Use for: production — more flexible and cost-effective than one LoadBalancer per Service.

#### 3.3 Ingress — Route External Traffic to Multiple Services

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: novapay-ingress
annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: api.novapay.in
http:
      paths:
      - path:     /payments
pathType: Prefix
backend:
          service:
            name: novapay-api-service
port:
              number: 80
```

**📖 Ingress — One Door, Many Rooms**

**Ingress** is a routing rules object. It needs an **Ingress Controller** (like nginx-ingress) installed in the cluster to do the actual work.  
  
**host: api.novapay.in** — only route traffic for this hostname here.  
  
**path: /payments** — requests to api.novapay.in/payments go to the novapay-api-service.  
  
You can add more `- path:` entries to route /users to a users-service, /notifications to a notification-service — all through one load balancer.

### 4. Phase 4 — ConfigMaps and Secrets

**Business Problem:** NovaPay's API needs to know which database to connect to, what the Razorpay API key is, and which environment it's running in (dev, staging, prod). You should never hardcode these inside the container image — then you'd need a different image for each environment, and secrets would be visible to anyone who pulls the image. ConfigMaps and Secrets solve this cleanly.

**Scene 4 — Security Audit, NovaPay | "The Key Is In the Image"**

> **Tanvi** _Senior Platform Engineer_
> 
> The security team found our Razorpay API key hardcoded in the Docker image. Anyone who can pull that image — which includes our entire engineering team and every CI/CD machine — can see the production payment key. That key processes real money. We need to separate configuration and secrets from the image immediately. The image should have zero environment-specific data inside it.

> **Roshan** _DevOps Architect_
> 
> ConfigMaps for non-sensitive config, Secrets for sensitive data. Both inject values into the container as environment variables or files at runtime. The image itself stays generic — the same image runs in dev with dev config, and in prod with prod config. Config lives in Kubernetes, not in the image.

#### 4.1 ConfigMap — Non-Sensitive Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: novapay-config
data:
  NODE_ENV:       "production"
DB_HOST:        "postgres-service"
DB_PORT:        "5432"
LOG_LEVEL:      "info"
MAX_CONNECTIONS: "100"
```

**📖 ConfigMap — Config as a Kubernetes Object**

A ConfigMap is a key-value store inside Kubernetes for **non-sensitive** configuration — things that are not secret (environment names, hostnames, ports, log levels).  
  
**data:** section holds your key-value pairs. Values are always strings in ConfigMaps.  
  
This same ConfigMap can be used by multiple Deployments. If the database hostname changes, update the ConfigMap — then restart the Pods to pick up the change.

```
# Reference ConfigMap in Deployment
containers:
    - name: api
envFrom:
      - configMapRef:
          name: novapay-config
```

**📖 Injecting ConfigMap as Env Vars**

**envFrom.configMapRef** loads ALL keys from the ConfigMap as environment variables in the container. The container sees `process.env.NODE_ENV = "production"`, `process.env.DB_HOST = "postgres-service"`, etc.  
  
The application code doesn't know or care that values came from Kubernetes — it just reads environment variables normally.

#### 4.2 Secret — Sensitive Data

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: novapay-secrets
type: Opaque
data:
  DB_PASSWORD:     cGFzc3dvcmQxMjM=
RAZORPAY_KEY:   cnpwX2xpdmVfYWJjMTIz
```

**📖 Secret — For Passwords and Keys**

**type: Opaque** — general-purpose secret (the most common type).  
  
**data values are base64-encoded** — NOT encrypted. Base64 is just encoding, not security. Anyone who can read the Secret object can decode the values with `echo "cGFzc3dvcmQ=" | base64 -d`.  
  
To create the base64 value: `echo -n "password123" | base64`. In production, use a proper secret manager (AWS Secrets Manager, HashiCorp Vault) that integrates with Kubernetes.

```
# Inject specific Secret keys as env vars
containers:
    - name: api
env:
      - name: DB_PASSWORD
valueFrom:
          secretKeyRef:
            name: novapay-secrets
key:  DB_PASSWORD
```

**📖 Injecting Secrets Selectively**

**env.valueFrom.secretKeyRef** injects one specific key from a Secret as an environment variable. The container sees `process.env.DB_PASSWORD = "password123"` (automatically base64-decoded).  
  
Using individual `env` entries (vs `envFrom`) gives you more control — you can rename the key and only expose what the container actually needs.

> **🔐 Secret Security Warning — This Is Not Enough for Production**

> Kubernetes Secrets are only base64-encoded by default — anyone with cluster access can read them. For a payment company like NovaPay, proper secret management is critical. Production best practices: (1) Enable **encryption at rest** for Secrets in etcd via Kubernetes KMS provider. (2) Use **AWS Secrets Manager + External Secrets Operator** so secrets never live in the cluster at all. (3) Use **RBAC** to restrict which service accounts can read which Secrets. Never commit Secret YAML files with real values to Git.

### 5. Phase 5 — Health Probes, Auto-Scaling, and Namespaces

**Business Problem:** Kubernetes restarts Pods when they crash — but what if the application is running but not healthy (stuck in an infinite loop, deadlocked, database connection lost)? **Probes** solve this. And during a payment spike (festival sales, midnight offers), traffic jumps 10x — **HPA** auto-scales the number of Pods. **Namespaces** keep dev, staging, and prod completely isolated in the same cluster.

#### 5.1 Liveness Probe — Restart Stuck Containers

```
 containers:
    - name: api
livenessProbe:
        httpGet:
          path: /health
port: 3000
initialDelaySeconds: 15
periodSeconds:       20
failureThreshold:    3
```

**📖 Liveness Probe — "Is This Container Alive?"**

Kubernetes sends a GET request to `/health` on port 3000 every 20 seconds.  
  
If it gets a non-200 response (or no response) 3 times in a row → **kill the container and restart it**.  
  
**initialDelaySeconds: 15** — wait 15 seconds after container starts before the first probe. Gives the app time to boot. Without this, Kubernetes kills the container before it even finishes starting up.

#### 5.2 Readiness Probe — Stop Sending Traffic to Unready Pods

```
 readinessProbe:
        httpGet:
          path: /ready
port: 3000
initialDelaySeconds: 5
periodSeconds:       10
```

**📖 Readiness Probe — "Is This Container Ready to Receive Traffic?"**

If the readiness probe fails → the Pod is removed from the Service's load balancer. Traffic stops going to it. The Pod is **not** restarted — just taken out of rotation.  
  
This is different from liveness: a Pod might be alive (running) but not ready (still loading cache, warming up database connections). Readiness ensures users never hit an unready Pod during deployment.

> **💡 Fresher Tip — Liveness vs Readiness — The Simple Difference**

> **Liveness** = "Is the app broken? Should I kill it and restart?" — answers the question "is the container healthy?"  
**Readiness** = "Is the app ready to serve requests right now?" — answers the question "should I send traffic to it?"  
  
A container can be *alive but not ready* (booting up, loading data). A container can be *ready but then become not-ready* (database went down temporarily). Always add both probes to every production container.

#### 5.3 Horizontal Pod Autoscaler — Scale Automatically

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: novapay-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
kind:       Deployment
name:       novapay-api
minReplicas: 3
maxReplicas: 10
metrics:
  - type: Resource
resource:
      name:               cpu
target:
        type:               Utilization
averageUtilization: 70
```

**📖 HPA — Automatic Scaling**

**scaleTargetRef** — which Deployment to scale.  
  
**minReplicas: 3** — never go below 3 Pods (for availability).  
  
**maxReplicas: 10** — never go above 10 (cost protection).  
  
**averageUtilization: 70** — when average CPU across all Pods exceeds 70% of their `requests` value, add more Pods. When it drops below 70%, remove Pods (scale in).  
  
During NovaPay's midnight festival sale: traffic 5x → CPU hits 80% → HPA scales from 3 to 8 Pods automatically. After sale: CPU drops → HPA scales back to 3. Zero manual intervention.

#### 5.4 Namespaces — Isolate Environments in One Cluster

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: novapay-prod
labels:
    environment: production
```

**📖 Namespace — Virtual Cluster Inside a Cluster**

A namespace is like a folder inside the cluster. All resources (Pods, Services, Secrets) in one namespace are isolated from other namespaces by default.  
  
NovaPay uses three namespaces: `novapay-dev`, `novapay-staging`, `novapay-prod`. A developer can't accidentally delete a prod Pod because their kubectl is configured to use the dev namespace. RBAC can restrict which teams can access which namespaces.

```bash
# Deploy into a specific namespace
kubectl apply -f deployment.yaml -n novapay-prod

# List pods in a specific namespace
kubectl get pods -n novapay-prod

# List pods in ALL namespaces (cluster-wide view)
kubectl get pods --all-namespaces
```

> The **-n flag** (or --namespace) tells kubectl which namespace to target. Without -n, kubectl uses the `default` namespace. In a real team, engineers set their default namespace with `kubectl config set-context --current --namespace=novapay-dev` so they never accidentally touch production.

### 6. Putting It All Together — NovaPay Complete Stack

Here's how all the pieces connect for one complete service. Each short block with its explanation shows one piece of the full picture.

```
NovaPay Complete Kubernetes Architecture
==========================================

  Internet
      │
      ▼
  [Ingress Controller]    ← one entry point for all external traffic
  /api  →  novapay-api-service (ClusterIP, port 80)
  /     →  novapay-web-service (ClusterIP, port 80)
      │
      ├── novapay-api-service
      │     └── [Pod:api] [Pod:api] [Pod:api]  ← Deployment, 3 replicas
      │           ├── envFrom: novapay-config   ← ConfigMap
      │           ├── env: DB_PASSWORD (Secret)
      │           ├── livenessProbe: GET /health
      │           └── readinessProbe: GET /ready
      │                      ↕  HPA: 3-10 replicas based on CPU
      │
      ├── novapay-web-service
      │     └── [Pod:web] [Pod:web]  ← React frontend, 2 replicas
      │
      └── postgres-service (ClusterIP)
            └── [Pod:postgres]  ← StatefulSet (not Deployment)
                  └── PersistentVolumeClaim (stores DB data)
```

```yaml
# Namespace for all prod resources
apiVersion: v1
kind: Namespace
metadata:
  name: novapay-prod
```

**📖 Start with the Namespace**

Always create the Namespace first. Every other resource goes into it. Apply the namespace YAML once, then add `namespace: novapay-prod` to the metadata of every other resource, or use `-n novapay-prod` on every kubectl command.

```
# Add namespace to every resource's metadata
metadata:
  name:      novapay-api
namespace: novapay-prod
labels:
    app:         novapay-api
version:     v1.3
environment: production
```

**📖 Labels — Use Them Consistently**

Add meaningful labels to every resource. `app` is used by Services to find Pods. `version` helps you track which version is deployed. `environment` helps with filtering when you run `kubectl get all -l environment=production`. Labels are searchable — without them, managing dozens of objects becomes very hard.

### 7. Essential kubectl Commands Reference

Command

What It Does

When to Use It

kubectl apply -f file.yaml

Create or update resource from YAML

Every time you deploy or change config

kubectl delete -f file.yaml

Delete resource defined in YAML

Remove a specific resource

kubectl get pods

List pods (Running/Pending/Error status)

Check if your app is running — constantly

kubectl get pods -o wide

List pods with IP and node info

Debug which node a pod is on

kubectl describe pod <name>

Full details + Events for a pod

Debug why a pod won't start

kubectl logs <pod>

See container stdout/stderr

Read application logs

kubectl logs -f <pod>

Follow/tail logs in real time

Watch a live running app

kubectl exec -it <pod> -- bash

Open a shell inside a running container

Debug inside the container

kubectl rollout status deploy/<name>

Watch rolling update progress

After kubectl set image or apply

kubectl rollout undo deploy/<name>

Rollback to previous version

When a new deployment breaks things

kubectl scale deploy/<name> --replicas=5

Manually set replica count

Quick scale during an incident

kubectl top pods

See CPU and memory used by each pod

Performance debugging, setting resource limits

kubectl get events --sort-by=.lastTimestamp

Cluster events sorted by time

Debug cluster-wide issues

kubectl port-forward pod/<name> 8080:3000

Forward local port to pod port

Test a pod locally without a Service

kubectl config get-contexts

List all cluster configurations

Switch between dev/staging/prod clusters

kubectl config use-context <name>

Switch to a different cluster

Before running commands against a different env

### 8. Interview Questions — Kubernetes

##### Interview Q&A — Fresher Level (0–1 Year Kubernetes Experience)

**Q: Q1. What is the difference between a Pod and a Deployment?**

A: A Pod is the smallest deployable unit in Kubernetes — it runs one or more containers and has its own IP address. However, if a Pod crashes or the node it's on fails, Kubernetes does not automatically restart it. A Deployment is a higher-level controller that manages Pods. You tell the Deployment "I want 3 replicas of this Pod always running." If one Pod crashes, the Deployment controller detects the shortfall and creates a replacement. Deployments also handle rolling updates (deploy a new version gradually without downtime) and rollbacks (revert to a previous version with one command). In production, you almost never create Pods directly — you always use a Deployment.

**Q: Q2. What is a Kubernetes Service and why do you need one?**

A: Pods have their own IP addresses, but those IPs change every time a Pod is restarted or rescheduled. If a Deployment has 3 replicas, each has a different IP, and those IPs change constantly. A Service provides a single, stable virtual IP address (and DNS name) that never changes. Other applications in the cluster send requests to the Service's IP, and the Service load-balances across all healthy Pods that match its label selector. Without a Service, you'd have to track and update every IP address manually as Pods come and go — which is impossible at scale. The Service decouples consumers from producers.

**Q: Q3. What is the difference between liveness and readiness probes?**

A: Both are health checks, but they cause different actions. A liveness probe answers "is this container alive and functional?" — if the probe fails repeatedly, Kubernetes kills the container and restarts it. This handles stuck processes (deadlocks, infinite loops) that are running but not working. A readiness probe answers "is this container ready to receive traffic right now?" — if the probe fails, Kubernetes removes the Pod from the Service's load balancer but does not restart it. This handles temporary unreadiness like the app is still loading its database connection pool, warming up a cache, or the database went down briefly. A Pod can be alive (liveness passing) but not ready (readiness failing). Both probes should be added to every production container.

**Q: Q4. What is the difference between ConfigMaps and Secrets?**

A: Both ConfigMaps and Secrets inject configuration into containers as environment variables or files. The difference is purpose and handling: ConfigMaps are for non-sensitive configuration — environment names, hostnames, port numbers, feature flags. Secrets are for sensitive data — database passwords, API keys, TLS certificates. Secrets are base64-encoded (not encrypted by default) and Kubernetes provides additional access controls — you can restrict which service accounts or namespaces can read which Secrets using RBAC. In practice, for truly sensitive production secrets like payment API keys, you use an external secret manager (AWS Secrets Manager, HashiCorp Vault) integrated with Kubernetes via the External Secrets Operator, so secrets never live in the cluster at all.

**Q: Q5. What is a Namespace and when would you use one?**

A: A Namespace is a virtual cluster inside a Kubernetes cluster — it provides isolation between groups of resources. Resources in one namespace don't see or affect resources in another namespace by default (with the exception of cluster-scoped resources like Nodes). Common patterns: one namespace per environment (dev, staging, production) on the same cluster to save cost; one namespace per team or application; one namespace per customer in a multi-tenant SaaS. You can apply RBAC policies per namespace to restrict who can deploy to production, and ResourceQuotas per namespace to limit how much CPU and memory each team can use. Using namespaces properly prevents developers from accidentally affecting production environments.

**Q: Q6. How does the Horizontal Pod Autoscaler work?**

A: HPA continuously monitors metrics (CPU utilisation, memory utilisation, or custom metrics) for all Pods in a Deployment. It compares the current metric value against the target you configured. If average CPU across all Pods exceeds the target (e.g. 70%), HPA calculates how many additional Pods are needed and updates the Deployment's replica count. Kubernetes then creates the new Pods. When load decreases, HPA scales in — reduces replica count — subject to the minReplicas floor. HPA requires the Metrics Server to be installed in the cluster to collect resource usage data. It checks metrics every 15 seconds by default. For HPA to work correctly, containers must have CPU requests defined — HPA calculates utilisation as a percentage of the CPU request value.

**Quiz: Quiz 1 — A Pod is in "CrashLoopBackOff" status. What does this mean and what is the first command you run?**

- A) The Pod is waiting for the image to download. Run: kubectl get nodes
- B) The container starts, crashes immediately, Kubernetes restarts it, it crashes again — in a loop. Run: kubectl logs <pod-name> to see why it's crashing
- C) The Pod is being throttled due to CPU limits. Run: kubectl top pods
- D) The container image doesn't exist. Run: kubectl delete pod <pod-name>

> **Answer/explanation:** ✅ Answer: B. CrashLoopBackOff means Kubernetes keeps restarting the container but it keeps exiting (crash). The "BackOff" part means Kubernetes waits longer and longer between restarts (10s, 20s, 40s, up to 5 minutes). The first command is always **kubectl logs <pod-name>** — this shows the last output the container printed before crashing. Common causes: missing environment variable, wrong database hostname, application bug on startup, missing file. If the container has already restarted, add **--previous** to see the previous crash's logs: `kubectl logs <pod-name> --previous`.

**Quiz: Quiz 2 — What happens when you run "kubectl apply -f deployment.yaml" after changing the image version?**

- A) All existing Pods are immediately deleted and new ones are created — causing a brief outage
- B) Kubernetes compares the new YAML with the current state and performs a rolling update: creating new Pods one at a time and removing old ones only after new ones are healthy
- C) Nothing happens — you must run kubectl delete first, then apply
- D) The Deployment is restarted from scratch, losing all current traffic

> **Answer/explanation:** ✅ Answer: B. kubectl apply is idempotent and smart — it compares what you specified against what currently exists and only changes what's different. If you only changed the image tag, Kubernetes performs a RollingUpdate: it creates one new Pod with the new image, waits for its readiness probe to pass, then removes one old Pod. This repeats until all Pods use the new image. With maxUnavailable: 0 configured, traffic is served throughout — zero downtime. Watch it happen: `kubectl rollout status deployment/novapay-api`.

**Quiz: Quiz 3 — What does "selector" in a Service YAML do?**

- A) It selects which node in the cluster to run the Service on
- B) It filters which namespaces the Service can route to
- C) It finds Pods with matching labels to include in the Service's load balancer — only traffic to matching Pods
- D) It selects which port number the Service should use

> **Answer/explanation:** ✅ Answer: C. The Service selector is how the Service dynamically discovers which Pods to send traffic to. It finds all Pods in the same namespace with labels that match the selector. As Pods are created and deleted (by Deployments, updates, crashes), the Service automatically updates its list of endpoints. If you deploy a new version and the new Pods have the same labels, traffic flows to them as soon as their readiness probe passes. If the selector doesn't match any Pods, the Service has no endpoints and all requests get connection refused — a common mistake when labels don't match between Deployment and Service.

> **Kubernetes Project — Core Takeaways for Freshers**

> - Kubernetes is a desired-state system — you declare what you want (3 replicas, 512MB RAM, restart on crash) and Kubernetes continuously works to make reality match your declaration. You never manually start or stop containers.
> - Never run bare Pods in production — always use a Deployment. Deployments give you replica management, rolling updates, rollbacks, and self-healing. Bare Pods are only for debugging or learning.
> - Always add both liveness and readiness probes to every production container. Without probes, Kubernetes sends traffic to containers that are still booting, and doesn't restart containers that are stuck — the worst of both worlds.
> - Never put secrets (passwords, API keys, certificates) inside a container image. Use Secrets for sensitive data and ConfigMaps for non-sensitive configuration. The same image should run in dev with dev config and in prod with prod config.
> - Always set resource requests and limits on every container. Without requests, the scheduler places Pods on nodes with insufficient resources. Without limits, one container can starve all other Pods on the node.
> - kubectl describe <object> is your best debugging tool — the Events section at the bottom tells you exactly what went wrong: ImagePullBackOff (wrong image name or registry credentials), Insufficient CPU (node doesn't have enough resources), FailedScheduling (no nodes match the Pod's requirements).
> - Use namespaces to separate environments even in the same cluster — and set your default namespace with kubectl config set-context so you never accidentally deploy to production when you meant staging.
> - Labels are not just cosmetic — they are functional. Services use labels to find Pods, HPA uses them to find Deployments, and kubectl -l lets you filter any object type. Use consistent labels on every object from day one.

##### Kubernetes Standards — NovaPay Platform Engineering Rules

- Store all YAML manifests in Git — infrastructure changes go through code review just like application code. Never run kubectl apply directly on production without a pull request and approval
- Use specific image tags (image:v1.3.2) in production, never image:latest — "latest" makes rollbacks impossible because you can't know what version was running before
- Always include a ConfigMap or Secret reference — never hardcode environment-specific values inside the container image or the Deployment YAML directly
- Set terminationGracePeriodSeconds to at least 30 seconds so in-flight payment requests can complete before Kubernetes removes a Pod during a rolling update
- Use PodDisruptionBudgets for critical services — they tell Kubernetes "never take down more than 1 replica at a time during node maintenance," preventing accidental outages during cluster upgrades
- Add a /health and /ready endpoint to every service — these are consumed by liveness and readiness probes. /health returns 200 if the app is alive. /ready returns 200 only when the app has finished initialising and is ready to serve requests

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Add a PostgreSQL StatefulSet:** Deployments are for stateless apps — databases need a `StatefulSet` because they need stable network identity and persistent storage. Write a StatefulSet for PostgreSQL with a `PersistentVolumeClaim` template (1Gi storage). Connect the novapay-api Deployment to it via a ClusterIP Service named `postgres-service` on port 5432.
2. **Add TLS to Ingress:** Create a TLS Secret using a self-signed certificate (`openssl req -x509 ...`) and reference it in the Ingress `tls:` section. All traffic to api.novapay.in should redirect from HTTP to HTTPS using the annotation `nginx.ingress.kubernetes.io/ssl-redirect: "true"`.
3. **Add a ResourceQuota to the namespace:** Create a ResourceQuota for the novapay-prod namespace that limits the total CPU to 4 cores and total memory to 8Gi across all Pods. Apply it and try to create a Deployment that exceeds the quota — observe how Kubernetes rejects it.
4. **Test the rollback:** Update the Deployment to use a non-existent image tag (like novapay/payment-api:v999). Watch it fail with `kubectl rollout status` and observe the ImagePullBackOff error. Then run `kubectl rollout undo` and confirm the previous version is restored and running.
5. **Simulate a node failure:** In minikube or kind, cordon a node (`kubectl cordon <node-name>`) to stop new Pods from scheduling there, then drain it (`kubectl drain <node-name> --ignore-daemonsets`) to evict existing Pods. Watch Kubernetes reschedule all Pods to the remaining nodes automatically within 30 seconds.

### Kubernetes Project Complete 🎉

You have deployed NovaPay's complete payment platform on Kubernetes — Pods, Deployments with rolling updates, ClusterIP and Ingress Services, ConfigMaps, Secrets, liveness and readiness probes, HPA auto-scaling, and Namespaces for environment isolation. You can now speak the language of every production Kubernetes cluster in the world.

> **Tanvi**
> 
> "Priya, last month we had 40 lakh rupees in failed payments because a single VM went down. This week you deployed three replicas of the same API with readiness probes. During our midnight offer last night — the biggest traffic spike in company history — HPA scaled us from 3 to 9 pods in under 2 minutes. Not a single failed payment. Zero downtime. That is what Kubernetes is for, and you built it."

> **Roshan**
> 
> "The mental model shift you made — from 'run this container on this server' to 'I want 3 healthy copies of this service running at all times, you figure out where' — that is the Kubernetes mindset. Once that clicks, the rest is just YAML. And you are writing very good YAML."

> **Aditya**
> 
> "And the Ingress you wrote? My React frontend just calls api.novapay.in/payments — one clean URL. Kubernetes routes it to the right Service, which load-balances to the right Pod. I don't know or care which node it's running on. That's the abstraction that makes microservices actually manageable."

> **Next: Advanced Kubernetes — Helm, RBAC, StatefulSets & GitOps with Argo CD**

> - Helm charts — package your Kubernetes manifests into reusable, versioned charts with templating for different environments
> - RBAC — Role-Based Access Control: limit which users and ServiceAccounts can perform which actions on which resources
> - StatefulSets — run stateful applications like databases with stable network identity and persistent storage
> - PersistentVolumes and PersistentVolumeClaims — provide durable storage that survives Pod restarts and rescheduling
> - Argo CD — GitOps tool that watches your Git repository and automatically applies changes to the cluster when YAML files change
> - Network Policies — firewall rules inside the cluster: restrict which Pods can talk to which other Pods (default is "allow all")
> - Pod Security Standards — prevent containers from running as root, mounting host paths, or using privileged mode
