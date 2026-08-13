# Kubernetes Service Mesh Mastery

> **👋 Hey Fresher — Read This First!**

> - **Service Mesh = a dedicated infrastructure layer for service-to-service communication** — instead of each microservice writing its own retry logic, timeouts, TLS, and metrics, the mesh handles all of it transparently via sidecar proxies
> - Without a service mesh: 50 microservices each implement their own retry logic — inconsistently, incompletely, and in different languages. One service's bad retry causes a cascade. One service leaks plaintext internal traffic. You find out 3 days later from a security audit.
> - With a service mesh: every service gets automatic mTLS, retries, circuit breaking, distributed tracing, and traffic shaping — without changing a single line of application code
> - The two production-grade service mesh options are **Istio** (feature-rich, complex) and **Linkerd** (lightweight, simpler). This guide covers Istio end-to-end with Linkerd comparisons
> - **Company:** NexaPay — a Pune-based fintech startup processing ₹800 crore/month in UPI and BNPL transactions across 18 microservices. A payment-service timeout cascaded to auth-service and wallet-service, causing a 47-minute outage during month-end salary credits. Internal services communicate over plain HTTP — a PCI-DSS compliance blocker. You just joined as Junior DevOps Engineer. Lead: **Vikram**. Mission: deploy Istio service mesh, enforce mTLS across all 18 services, implement circuit breaking on payment-service, and add canary releases for zero-downtime deploys

#### What You Will Learn and Build

- Understand the sidecar proxy model — how Envoy sits beside every pod and intercepts all traffic transparently
- Install Istio on NexaPay's EKS cluster using istioctl with production-profile settings
- Enforce mTLS (mutual TLS) cluster-wide — all 18 NexaPay services communicate encrypted automatically
- Configure VirtualService and DestinationRule — traffic routing, retries, timeouts, and load balancing policies
- Implement circuit breaking on payment-service — stop cascade failures before they become 47-minute outages
- Deploy canary releases — route 5% of traffic to a new payment-service version, monitor, then promote
- Set up Kiali, Jaeger, and Prometheus for full mesh observability — see every service dependency in real time
- Write AuthorizationPolicy — zero-trust network: only auth-service may call payment-service, deny everything else
- Compare Istio vs Linkerd — when to use each, migration considerations, and performance overhead

Istio, mTLS, Envoy Proxy, VirtualService, DestinationRule, Circuit Breaking, Canary Release, Kiali, AuthorizationPolicy, Linkerd

### 0. Service Mesh Architecture — Big Picture

```
┌──────────────────────────────────────────────────────────────────────────┐
│              NEXAPAY — ISTIO SERVICE MESH ARCHITECTURE                   │
└──────────────────────────────────────────────────────────────────────────┘

  WITHOUT SERVICE MESH (NexaPay Before)          WITH SERVICE MESH (After)
  ══════════════════════════════════             ══════════════════════════
  ┌────────────┐  plain HTTP  ┌──────────┐       ┌───────────────────────┐
  │ auth-svc   │─────────────▶│ payment  │       │  ISTIO CONTROL PLANE  │
  └────────────┘              │ -svc     │       │  ┌─────────────────┐  │
  ┌────────────┐  plain HTTP  │          │       │  │ istiod (Pilot)  │  │
  │ wallet-svc │─────────────▶│ CRASHES  │       │  │ Citadel (certs) │  │
  └────────────┘              │ on       │       │  │ Galley (config) │  │
  No retries, no mTLS,        │ timeout  │       │  └─────────────────┘  │
  no circuit breaking,        └──────────┘       └──────────┬────────────┘
  no visibility                                             │ xDS API (config push)
                                                           │
  ┌──────────────────────────────────────────────────────────────────────┐
  │                    KUBERNETES DATA PLANE                             │
  │                                                                      │
  │  ┌─────────────────────────┐      ┌─────────────────────────┐       │
  │  │  auth-svc Pod           │      │  payment-svc Pod         │       │
  │  │ ┌──────────┐ ┌───────┐ │      │ ┌──────────┐ ┌───────┐  │       │
  │  │ │ auth-svc │ │Envoy  │ │─mTLS▶│ │payment   │ │Envoy  │  │       │
  │  │ │ container│ │sidecar│ │      │ │container │ │sidecar│  │       │
  │  │ └──────────┘ └───────┘ │      │ └──────────┘ └───────┘  │       │
  │  └─────────────────────────┘      └─────────────────────────┘       │
  │                                                                      │
  │  Envoy handles: mTLS · retries · circuit breaking · metrics · tracing│
  └──────────────────────────────────────────────────────────────────────┘

  ISTIO TRAFFIC FLOW:
  app-container → Envoy sidecar (iptables intercept) → mTLS → Envoy sidecar → app-container

  KEY ISTIO CRDS:
  ┌──────────────────────┬──────────────────────────────────────────────┐
  │ VirtualService       │ Routing rules: which version, retries,       │
  │                      │ timeouts, fault injection, traffic split      │
  ├──────────────────────┼──────────────────────────────────────────────┤
  │ DestinationRule      │ Traffic policy: mTLS mode, load balancing,   │
  │                      │ connection pool, circuit breaker settings     │
  ├──────────────────────┼──────────────────────────────────────────────┤
  │ Gateway              │ Ingress/egress — exposes services outside     │
  │                      │ the cluster via Istio's ingress gateway       │
  ├──────────────────────┼──────────────────────────────────────────────┤
  │ PeerAuthentication   │ Enforces mTLS mode: STRICT / PERMISSIVE /   │
  │                      │ DISABLE per namespace or workload            │
  ├──────────────────────┼──────────────────────────────────────────────┤
  │ AuthorizationPolicy  │ Zero-trust RBAC: allow/deny traffic by       │
  │                      │ source service, method, path, header         │
  └──────────────────────┴──────────────────────────────────────────────┘
```

### 1. Phase 1 — Istio Installation on EKS

**Business Problem:** NexaPay's 18 microservices run on plain HTTP with no encryption, no retry logic, and no visibility into service-to-service calls. A PCI-DSS audit flagged the plaintext internal traffic as a critical compliance violation. Istio needs to be installed without disrupting the ₹800 crore/month live transaction flow.

**Scene 1 — NexaPay Incident Room | "47 Minutes of Payment Downtime During Salary Day"**

> **Vikram** _Lead DevOps Engineer — NexaPay_
> 
> "Last month-end, payment-service started timing out on the credit-bureau API. No circuit breaker — so every call from auth-service to payment-service queued up, held threads, and within 4 minutes wallet-service was also down. 47 minutes of full payment outage. ₹11 crore in transactions delayed. And our PCI-DSS auditor found that auth-service to payment-service calls are plain HTTP. Cardholder data in plaintext on the internal network. That is a Level 1 violation. We are deploying Istio this sprint."

> **Priya (Junior DevOps)** _Junior DevOps Engineer — NexaPay_
> 
> "Do we need to change all 18 services to add TLS? And add retry logic to each one separately?"

> **Vikram** _Lead DevOps Engineer — NexaPay_
> 
> "Zero code changes. That's the entire point of a service mesh. Istio injects an Envoy sidecar proxy into every pod — the app talks to localhost, Envoy handles mTLS, retries, circuit breaking, and emits metrics automatically. The 18 services don't know the mesh exists. We configure it all from the outside via Kubernetes CRDs."

#### 1.1 Pre-Installation Checks

```bash
# Verify cluster requirements before Istio install
kubectl version --short
# Kubernetes 1.25+ required for Istio 1.20+
kubectl get nodes
# Minimum: 3 nodes, 4 vCPU + 8GB RAM each
# Istio control plane needs ~1 vCPU + 1.5GB RAM
# Check no existing service mesh is installed
kubectl get namespace istio-system
# Should return: Error from server (NotFound)
# Download istioctl — Istio's CLI tool
curl -L https://istio.io/downloadIstio | \
  ISTIO_VERSION=1.20.0 sh -
cd istio-1.20.0
export PATH=$PWD/bin:$PATH

# Verify istioctl is available
istioctl version
```

**📖 istioctl — Istio's Control Tool**

- **istioctl** is the Istio equivalent of kubectl — but specifically for installing, configuring, and debugging the mesh; use istioctl for mesh operations, kubectl for everything else
- **Minimum cluster resources** — Istio control plane (istiod) consumes ~1 vCPU and 1.5GB RAM; each Envoy sidecar adds ~50MB RAM and ~0.5% CPU per pod — budget this for 18 NexaPay services
- **Kubernetes version compatibility** — Istio releases track Kubernetes releases; always check the Istio support matrix before upgrading either component
- **istioctl x precheck** — run this command before install to detect configuration conflicts, API deprecations, and resource gaps; prevents 80% of failed installs

#### 1.2 Install Istio with Production Profile

```bash
# istioctl pre-installation check — always run this first
istioctl x precheck

# Install Istio using production profile
# Profiles: minimal · default · demo · preview · production
istioctl install \
  --set profile=production \
  --set values.pilot.resources.requests.cpu=500m \
  --set values.pilot.resources.requests.memory=2Gi \
  --set values.global.proxy.resources.requests.cpu=100m \
  --set values.global.proxy.resources.requests.memory=128Mi \
  --set values.gateways.istio-ingressgateway.serviceAnnotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb \
  -y

# Verify control plane is running
kubectl get pods -n istio-system
```

> **production profile vs demo profile** — demo profile enables all telemetry add-ons (Kiali, Jaeger, Prometheus) for easy testing; production profile is lean, HA, and production-tuned; at NexaPay we install add-ons separately with persistence and resource limits
**pilot = istiod** — Istio's control plane is a single binary called istiod (merged from Pilot + Citadel + Galley); it pushes Envoy configuration via xDS API and manages certificate issuance for mTLS
**proxy resources request** — 100m CPU and 128Mi RAM per sidecar; for 18 NexaPay services this adds ~1.8 vCPU and ~2.3GB RAM cluster-wide; size nodes accordingly
**NLB annotation** — on EKS, Istio's ingress gateway needs an AWS Network Load Balancer (not Classic LB) to handle TCP traffic for gRPC and long-lived connections correctly

```
NAME                                   READY   STATUS    RESTARTS   AGE
istiod-7d9d6b97c8-k2xpn               1/1     Running   0          2m
istio-ingressgateway-5c74f64d8-xvt9r   1/1     Running   0          2m
istio-ingressgateway-5c74f64d8-m3pqz   1/1     Running   0          2m
```

#### 1.3 Enable Sidecar Injection for NexaPay Namespaces

```bash
# Label namespaces for automatic sidecar injection
# Istio watches for this label and injects Envoy on pod create
kubectl label namespace nexapay-prod \
  istio-injection=enabled
kubectl label namespace nexapay-staging \
  istio-injection=enabled
# Restart existing deployments to inject sidecars
# (new pods get sidecar; old pods don't until restarted)
kubectl rollout restart deployment \
  -n nexapay-prod
# Verify sidecar was injected — should show 2/2 READY
# 2 containers: app + envoy sidecar
kubectl get pods -n nexapay-prod
```

**📖 How Sidecar Injection Works**

- **Mutating Admission Webhook** — Istio registers a webhook with Kubernetes API server; when a new Pod is created in a labelled namespace, the API server calls istiod's webhook before saving the pod spec; istiod injects the Envoy container and init containers automatically
- **Init container istio-init** — runs before the app container; sets iptables rules to redirect ALL inbound and outbound traffic through Envoy on port 15001/15006; the app does not know this is happening
- **2/2 READY pattern** — after injection, every NexaPay pod shows 2 containers: the app container + istio-proxy (Envoy); this is how you verify injection worked
- **Exclude specific pods** — add annotation `sidecar.istio.io/inject: "false"` to exclude a pod from injection; use for jobs, DaemonSets accessing host network, or pods that don't need mesh features
- Rolling restart is safe — Kubernetes does a rolling replace; at NexaPay we set maxUnavailable: 0 to ensure payment-service is never fully down during the restart

> **🔑 Phase 1 Key Takeaways**

> - Always run `istioctl x precheck` before installing — it catches compatibility and configuration problems before they become failed installs
> - Use production profile for real workloads — demo profile is for learning; it has no HA, no persistence, and too many enabled features
> - Sidecar injection is namespace-scoped via a label — label the namespace, restart pods; zero app code changes required
> - 2/2 READY in kubectl get pods confirms sidecar injection succeeded — any pod showing 1/1 in a labelled namespace has not been restarted yet

### 2. Phase 2 — mTLS (Mutual TLS) — Encrypting All Service Communication

**Business Problem:** NexaPay's PCI-DSS audit flagged plaintext HTTP between auth-service and payment-service as a critical violation. The fix must encrypt all 18 service communications without touching application code, without distributing certificates manually, and with zero downtime.

**Scene 2 — NexaPay Security Review | "The Auditor Found Plaintext Cardholder Data on the Wire"**

> **Vikram** _Lead DevOps Engineer — NexaPay_
> 
> "The PCI-DSS auditor ran Wireshark on the cluster network and captured auth-service sending a user's card token to payment-service over plain HTTP. That's a Level 1 finding. Without Istio, fixing this means adding TLS to all 18 services — generating certificates, distributing them, rotating them before expiry, handling failures when a cert expires. That's months of work and every service team needs to cooperate. With Istio's mTLS, we enforce it cluster-wide from one PeerAuthentication resource. istiod acts as the internal CA, issues certs automatically, rotates them every 24 hours. The services themselves don't see any of this."

#### 2.1 How Istio mTLS Works

```
┌──────────────────────────────────────────────────────────────────────┐
│                  ISTIO mTLS CERTIFICATE FLOW                         │
└──────────────────────────────────────────────────────────────────────┘

  istiod (Internal CA)
       │
       │ 1. Issues X.509 certificate to each pod's Envoy
       │    Identity = SPIFFE URI: spiffe://cluster.local/ns/nexapay-prod/sa/payment-svc
       │
       ▼
  ┌────────────────────────────────────────────────────────────────────┐
  │  auth-svc Pod              mTLS Handshake         payment-svc Pod │
  │  ┌──────────┐ ┌─────────┐ ◀══════════════════▶ ┌─────────┐ ┌───┐ │
  │  │ auth-svc │ │ Envoy   │                       │ Envoy   │ │pay│ │
  │  │ app      │ │ sidecar │                       │ sidecar │ │app│ │
  │  └──────────┘ └─────────┘                       └─────────┘ └───┘ │
  │                                                                    │
  │  Steps in mTLS handshake:                                          │
  │  1. auth-svc Envoy presents its cert → payment-svc Envoy verifies  │
  │  2. payment-svc Envoy presents its cert → auth-svc Envoy verifies  │
  │  3. Both sides verified → encrypted TLS 1.3 channel established    │
  │  4. auth-svc app sends HTTP (unencrypted) to localhost:8080        │
  │  5. Envoy intercepts → wraps in mTLS → sends to payment-svc Envoy  │
  │  6. payment-svc Envoy decrypts → passes plain HTTP to payment app  │
  └────────────────────────────────────────────────────────────────────┘

  CERTIFICATE ROTATION:
  istiod issues certs valid for 24 hours (configurable)
  Envoy requests renewal at 80% of cert lifetime → zero downtime rotation
  No manual cert management ever needed
```

#### 2.2 PeerAuthentication — Enforcing mTLS

```yaml
# peer-auth-strict.yaml
# Enforce STRICT mTLS across entire nexapay-prod namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: nexapay-mtls-strict
namespace: nexapay-prod
spec:
  mtls:
    mode: STRICT
---
# PERMISSIVE mode for gradual migration
# Accept both plain HTTP and mTLS — use during transition
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: nexapay-mtls-permissive
namespace: nexapay-staging
spec:
  mtls:
    mode: PERMISSIVE
```

**📖 mTLS Modes — STRICT vs PERMISSIVE**

- **STRICT** — rejects any plaintext HTTP connection; only encrypted mTLS traffic is accepted; use in production once all services have sidecars injected
- **PERMISSIVE** — accepts both mTLS and plain HTTP; use during migration when some services are still getting sidecars injected; prevents service disruption during rollout
- **DISABLE** — disables mTLS entirely; use only for debugging or specific legacy services that cannot accept TLS under any condition
- **NexaPay migration strategy** — set staging to PERMISSIVE, inject sidecars into all 18 services, verify everything works, then flip production to STRICT in one command
- Scope: a namespace-level PeerAuthentication applies to all pods in that namespace; a pod-level PeerAuthentication (with a selector) overrides the namespace-level for specific workloads

#### 2.3 DestinationRule — Client-Side TLS Policy

```yaml
# destinationrule-payment.yaml
# DestinationRule tells the CLIENT-side Envoy how to connect
# PeerAuthentication tells the SERVER-side Envoy what to accept
# Both are needed for full mTLS enforcement
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service-dr
namespace: nexapay-prod
spec:
  host: payment-service.nexapay-prod.svc.cluster.local
trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL # use Istio-managed certs (not custom)
loadBalancer:
      simple: LEAST_CONN # send to least-loaded pod
connectionPool:
      http:
        http1MaxPendingRequests: 100
http2MaxRequests: 1000
tcp:
        maxConnections: 100
connectTimeout: 3s
```

> **ISTIO_MUTUAL mode** — tells the caller's Envoy to use Istio-managed certificates for the TLS handshake; never use SIMPLE mode (one-way TLS) in production — it doesn't verify the server's identity
**LEAST_CONN load balancing** — sends each new request to the pod with the fewest active connections; better than ROUND_ROBIN for payment-service because some transactions take longer and ROUND_ROBIN would pile requests on slow pods
**http1MaxPendingRequests: 100** — part of circuit breaking; if more than 100 requests are waiting for payment-service, new requests immediately return 503 instead of queuing forever (which caused NexaPay's 47-minute outage)
**connectTimeout: 3s** — if payment-service doesn't accept the TCP connection within 3 seconds, Envoy marks this pod as unhealthy; this prevents the hung-connection scenario from the incident

```
✓ Verifying mTLS is active across nexapay-prod

$ istioctl authn tls-check payment-service.nexapay-prod.svc.cluster.local

HOST:PORT                                          STATUS      SERVER        CLIENT
payment-service.nexapay-prod.svc.cluster.local:8080  OK         STRICT        ISTIO_MUTUAL
auth-service.nexapay-prod.svc.cluster.local:8080     OK         STRICT        ISTIO_MUTUAL
wallet-service.nexapay-prod.svc.cluster.local:8080   OK         STRICT        ISTIO_MUTUAL
```

### 3. Phase 3 — Traffic Management: VirtualService & DestinationRule

**Business Problem:** NexaPay's payment-service v1.2 has a critical bug — it incorrectly rounds down UPI amounts below ₹10. A new v1.3 is ready but cannot be deployed to 100% of traffic immediately — a bad deploy affecting ₹800 crore/month in transactions is unacceptable. The team needs canary releases and fine-grained traffic control.

**Scene 3 — NexaPay Engineering Slack | "We Need to Roll Back payment-service in 30 Seconds"**

> **Vikram** _Lead DevOps Engineer — NexaPay_
> 
> "Before Istio, every deploy was binary — 0% or 100% of traffic. Last quarter a bad payment-service deploy hit 100% of users before anyone noticed the rounding bug. Rollback took 8 minutes — kubectl set image, wait for rollout, pods terminating, new pods starting. With Istio VirtualService, we can send 5% of traffic to v1.3, watch the error rate in Kiali, and either shift to 100% or shift back to 0% by editing a single YAML field. The new version never gets more than 5% until we're confident."

#### 3.1 VirtualService — Routing Rules

```yaml
# virtualservice-payment.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service-vs
namespace: nexapay-prod
spec:
  hosts:
    - payment-service
http:
    - retries:
        attempts: 3
perTryTimeout: 2s
retryOn: gateway-error,connect-failure,retriable-4xx
timeout: 8s
route:
        - destination:
            host: payment-service
subset: v1
weight: 95
        - destination:
            host: payment-service
subset: v2
weight: 5
```

**📖 VirtualService — What Each Field Does**

- **retries.attempts: 3** — if payment-service returns 503 or times out, Envoy automatically retries up to 3 times before returning the error to the caller; the calling app sees 0 retries — it just eventually gets a response
- **perTryTimeout: 2s** — each individual attempt must complete within 2 seconds; the overall timeout is 8s, so 3 attempts × 2s + inter-retry gaps = fits within 8s budget
- **retryOn: gateway-error,connect-failure,retriable-4xx** — only retry on infrastructure errors and specific 4xx codes; never retry on 4xx like 400 Bad Request (those are caller bugs, not transient failures)
- **weight: 95 / 5** — 95% of requests go to v1 (stable), 5% go to v2 (canary); weights must sum to 100; change to 50/50 or 100/0 by editing and re-applying
- VirtualService does not know about pods — it routes to subsets defined in the DestinationRule

#### 3.2 DestinationRule — Defining Subsets for Canary

```yaml
# destinationrule-payment-subsets.yaml
# Subsets map to Kubernetes pod labels
# v1 = pods with label version: v1, v2 = pods with label version: v2
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service-subsets
namespace: nexapay-prod
spec:
  host: payment-service
trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
subsets:
    - name: v1
labels:
        version: v1 # matches pod label version: v1
trafficPolicy:
        connectionPool:
          http:
            http1MaxPendingRequests: 200
    - name: v2
labels:
        version: v2 # matches pod label version: v2
trafficPolicy:
        connectionPool:
          http:
            http1MaxPendingRequests: 20 # limit canary load
```

> **Subsets map to pod labels** — the label `version: v1` on NexaPay's payment-service pods is what Istio uses to identify which pods belong to which subset; without this label on the Deployment, subsets don't work
**Separate connection pool per subset** — v1 gets 200 pending requests (stable, handles full load); v2 gets 20 (canary, only gets 5% weight = much lower volume; limit prevents canary from accidentally absorbing too much load if weights are misconfigured)
**Progressive traffic shift** — after 1 hour at 5% with no errors, update VirtualService weight to 20/80, wait, then 50/50, then 0/100; the entire promotion is YAML edits with zero pod restarts
Rollback = edit VirtualService weight back to 100/0 for v1; takes effect in under 1 second (Envoy hot-reload config without dropping connections)

#### 3.3 Header-Based Routing — QA Testing in Production

```yaml
# Route only QA team to v2 using request header
# All other traffic stays on v1
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-header-routing
namespace: nexapay-prod
spec:
  hosts:
    - payment-service
http:
    - match:
        - headers:
            x-nexapay-canary:
              exact: "true"
route:
        - destination:
            host: payment-service
subset: v2
    - route:        # default — no match
        - destination:
            host: payment-service
subset: v1
```

**📖 Header-Based Routing Use Cases**

- **QA testing in production** — NexaPay's QA team sends header `x-nexapay-canary: true` in Postman; only their requests hit v2; all real user traffic stays on v1
- **Beta user routing** — route users with a specific account flag or session cookie to the new version; validated by header value matching
- **Dark launching** — mirror 100% of traffic to v2 simultaneously (VirtualService mirror field) without affecting responses; v2 processes requests but results are discarded; validates v2 under real load without risk
- **Fault injection testing** — add a fault block to inject 5% 500 errors or 2s delays into a service; test that upstream circuit breakers and retries handle it correctly — without writing any testing code

### 4. Phase 4 — Circuit Breaking

**Business Problem:** NexaPay's 47-minute outage was caused by a cascade failure — payment-service became slow, callers kept retrying, thread pools exhausted, and the failure spread to auth-service and wallet-service. A circuit breaker would have isolated payment-service in under 10 seconds and kept the rest of the platform running.

**Scene 4 — NexaPay Post-Mortem | "One Slow Service Took Down Four Healthy Ones"**

> **Vikram** _Lead DevOps Engineer — NexaPay_
> 
> "In the incident timeline: 14:22 — credit-bureau API goes slow. 14:23 — payment-service starts queueing requests. 14:25 — auth-service thread pool exhausted waiting for payment-service responses. 14:26 — wallet-service times out waiting for auth-service. 14:28 — full platform down. All because there was no circuit breaker. A circuit breaker on payment-service would have detected the slowness by 14:23, started returning immediate 503s by 14:24, and auth-service and wallet-service would have gotten fast failures they could handle — not slow hangs that exhausted their threads."

#### 4.1 Circuit Breaker Configuration in DestinationRule

```yaml
# circuit-breaker-payment.yaml
# Istio circuit breaking is configured in DestinationRule
# No app code changes — Envoy enforces all limits
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-circuit-breaker
namespace: nexapay-prod
spec:
  host: payment-service
trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100 # max concurrent TCP connections
http:
        http1MaxPendingRequests: 50 # queue limit before 503
http2MaxRequests: 200 # concurrent request cap
maxRequestsPerConnection: 10 # force connection reuse
outlierDetection:
      consecutive5xxErrors: 5 # 5 errors → eject pod
interval: 10s # check every 10 seconds
baseEjectionTime: 30s # first ejection lasts 30s
maxEjectionPercent: 50 # never eject more than 50% of pods
minHealthyPercent: 50 # keep at least 50% always available
```

> **http1MaxPendingRequests: 50** — the critical setting that would have prevented NexaPay's outage; once 50 requests are queued waiting for payment-service, all new requests get immediate 503 instead of queuing; this releases auth-service threads instantly
**outlierDetection** — Istio's passive health checking; Envoy tracks error rates per pod; if a specific pod returns 5 consecutive 5xx errors within a 10-second window, Envoy stops routing traffic to that pod for 30 seconds
**baseEjectionTime: 30s** — after first ejection, the pod gets a 30-second timeout; if it's ejected again, the timeout doubles (30s → 60s → 120s); this prevents a flapping bad pod from repeatedly being added back to the pool
**maxEjectionPercent: 50** — never eject more than 50% of payment-service pods; prevents a scenario where all pods have a transient error and Istio ejects all of them, causing a worse outage than the original problem
Circuit breaker state is per-Envoy-instance — each calling pod's Envoy tracks its own view of payment-service health; there is no central circuit breaker state, which makes it highly scalable

#### 4.2 Circuit Breaker States and Behavior

```
┌──────────────────────────────────────────────────────────────────────┐
│          ISTIO CIRCUIT BREAKER STATES — NEXAPAY PAYMENT-SERVICE      │
└──────────────────────────────────────────────────────────────────────┘

                    ┌──────────────────────────────────────┐
                    │           CLOSED (Normal)            │
                    │   All requests flow to payment-svc   │
                    │   Envoy tracking: error count = 0    │
                    └──────────────────┬───────────────────┘
                                       │
                         5 consecutive 5xx errors
                         OR >50 pending requests
                                       │
                                       ▼
                    ┌──────────────────────────────────────┐
                    │      POD EJECTED (Per-Pod CB)        │
                    │   Failing pod removed from pool      │
                    │   Traffic rerouted to healthy pods   │
                    │   Ejection lasts: baseEjectionTime   │
                    └──────────────────┬───────────────────┘
                                       │
                         baseEjectionTime expires (30s)
                                       │
                                       ▼
                    ┌──────────────────────────────────────┐
                    │           HALF-OPEN (Probe)          │
                    │   Pod re-added to pool               │
                    │   If next request succeeds → CLOSED  │
                    │   If fails again → eject 60s (2×)   │
                    └──────────────────────────────────────┘

  CONNECTION POOL CIRCUIT BREAK (separate from outlier detection):
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Queue > 50 requests → new request returns 503 immediately
  Connections > 100   → new connection refused immediately
  Result: auth-svc gets fast failure, releases threads, stays healthy
```

##### 🛠️ Exercise 4 — Simulate and Observe Circuit Breaking at NexaPay

1. Deploy the circuit breaker DestinationRule for payment-service (yaml above)
2. Install fortio load testing tool: `kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/httpbin/sample-client/fortio-deploy.yaml`
3. Run 20 concurrent connections to payment-service: `kubectl exec fortio-deploy -- fortio load -c 20 -qps 0 -n 200 http://payment-service:8080/pay`
4. Observe: requests exceeding the connectionPool limit return 503 (Overflow) immediately
5. Check Kiali dashboard — the circuit breaker icon (lightning bolt) appears on the payment-service node
6. Kill one payment-service pod mid-test — watch outlierDetection eject it and traffic automatically shift to remaining pods
7. Read the stats: `kubectl exec fortio-deploy -- fortio report` and identify the 503 percentage vs 200 OK

### 5. Phase 5 — Zero-Trust Security with AuthorizationPolicy

**Business Problem:** After enforcing mTLS (all traffic encrypted), NexaPay's security team realises encryption alone is not enough. Any service that is compromised can still call payment-service — because there is no rule preventing wallet-service from directly making payment calls, bypassing fraud-detection-service. AuthorizationPolicy implements zero-trust: only explicitly allowed service identities may call specific endpoints.

**Scene 5 — NexaPay Security Audit | "Encrypted Traffic is Not the Same as Authorised Traffic"**

> **Vikram** _Lead DevOps Engineer — NexaPay_
> 
> "mTLS tells us that wallet-service IS wallet-service — the identity is verified. But it doesn't tell us that wallet-service is ALLOWED to call payment-service's /initiate-transfer endpoint. Right now, any compromised service in the cluster can call any other service freely — we just know who it is. AuthorizationPolicy adds the second layer: even with a verified identity, you can only call what you're explicitly permitted to. payment-service should only receive calls from auth-service and fraud-detection-service. Nothing else. Not wallet-service directly, not notification-service, nothing."

#### 5.1 AuthorizationPolicy — Deny-All Then Allow-Specific

```yaml
# Step 1: Deny ALL traffic to payment-service by default
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-deny-all
namespace: nexapay-prod
spec:
  selector:
    matchLabels:
      app: payment-service
# Empty spec with selector = DENY ALL
---
# Step 2: Allow ONLY auth-service and fraud-detection-service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-allow-trusted
namespace: nexapay-prod
spec:
  selector:
    matchLabels:
      app: payment-service
action: ALLOW
rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/nexapay-prod/sa/auth-service"
              - "cluster.local/ns/nexapay-prod/sa/fraud-detection-svc"
to:
        - operation:
            methods: ["POST"]
            paths: ["/api/v1/payment/*"]
```

**📖 AuthorizationPolicy — Zero Trust Explained**

- **SPIFFE principals** — the source identity is a SPIFFE URI tied to a Kubernetes ServiceAccount; `cluster.local/ns/nexapay-prod/sa/auth-service` means "any pod running with the Kubernetes ServiceAccount named auth-service in namespace nexapay-prod"
- **Deny-all first, then allow** — never start with allow-all and then deny specific services; that approach has gaps; always start with deny-all and explicitly allow each permitted caller
- **Method + path restriction** — only allow POST to /api/v1/payment/* paths; even if auth-service is trusted, it cannot make GET calls to internal health endpoints or DELETE calls accidentally
- **No app code change** — the 403 Forbidden is returned by Envoy before the request ever reaches the payment-service container; the application never sees unauthorised requests
- Test with: `istioctl x authz check <payment-pod-name>` — lists all AuthorizationPolicies applied to a pod and their effect

#### 5.2 NexaPay Complete Zero-Trust Policy Matrix

Caller Service

Target Service

Allowed Methods

Allowed Paths

Policy

auth-service

payment-service

POST

/api/v1/payment/*

ALLOW

fraud-detection-svc

payment-service

POST, GET

/api/v1/payment/*, /api/v1/check/*

ALLOW

wallet-service

payment-service

—

—

DENY (wallet must go through auth-service)

notification-svc

payment-service

—

—

DENY

auth-service

user-service

GET, POST

/api/v1/user/*

ALLOW

payment-service

fraud-detection-svc

POST

/api/v1/check

ALLOW

Any service

Any service

—

/healthz, /metrics

ALLOW (monitoring)

### 6. Phase 6 — Observability with Kiali, Jaeger, and Prometheus

**Business Problem:** NexaPay engineers have no visibility into service-to-service calls. During the outage, it took 12 minutes to identify that payment-service was the root cause because there was no distributed tracing. Post-Istio, every call generates traces automatically — pinpointing failures to the exact service, method, and millisecond.

**Scene 6 — NexaPay NOC | "We Spent 12 Minutes Running kubectl logs on the Wrong Service"**

> **Vikram** _Lead DevOps Engineer — NexaPay_
> 
> "During the incident, the first alert fired on wallet-service — that's what the on-call saw. He spent 12 minutes checking wallet-service logs, then auth-service logs, before finding the actual problem in payment-service. With Kiali, you open the service graph, see the red error rate on payment-service in real time, click on it, and you're looking at distributed traces showing exactly which calls failed and why. Root cause in under 60 seconds. That's the difference between a 47-minute outage and a 5-minute one."

#### 6.1 Installing Observability Add-ons

```bash
# Install Istio observability add-ons
# These are the official Istio sample addons — production needs Helm charts with persistence
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml

# Verify all add-ons are running
kubectl get pods -n istio-system

# Access Kiali dashboard (service graph + traffic flow)
istioctl dashboard kiali

# Access Jaeger distributed tracing dashboard
istioctl dashboard jaeger

# Access Grafana (Istio metrics dashboards)
istioctl dashboard grafana

# Key Prometheus metrics Istio exposes automatically
# istio_requests_total — request count by source, dest, code
# istio_request_duration_milliseconds — latency histogram
# istio_tcp_connections_opened_total — connection tracking
```

> **Kiali** — the service mesh topology visualiser; shows a live graph of all NexaPay services, traffic flow, error rates, and request volumes; the fastest way to find a failing service during an incident
**Jaeger** — distributed tracing; every request across all 18 NexaPay services gets a unique trace ID; you can follow a single UPI payment from the API gateway through auth-service → fraud-detection → payment-service → notification-service as one connected trace
**Prometheus** — Envoy emits 70+ metrics automatically per service; no instrumentation code needed; Prometheus scrapes these and Grafana visualises them on pre-built Istio dashboards
**Production setup differs** — sample addons have no persistence; in production use Kiali Helm chart with an external Prometheus, Jaeger with Elasticsearch backend for trace storage, and Grafana with persistent PVC

#### 6.2 Key Istio Metrics for NexaPay SLOs

Metric

PromQL Query

NexaPay SLO

Request success rate

sum(rate(istio_requests_total{reporter="destination",response_code!~"5.*"}[5m])) / sum(rate(istio_requests_total{reporter="destination"}[5m]))

> 99.9%

P99 latency (payment)

histogram_quantile(0.99, sum(rate(istio_request_duration_milliseconds_bucket{destination_service="payment-service"}[5m])) by (le))

< 500ms

Circuit breaker trips

sum(rate(envoy_cluster_upstream_cx_overflow_total{cluster_name="payment-service"}[5m]))

Alert > 10/min

mTLS failure rate

sum(rate(istio_requests_total{connection_security_policy!="mutual_tls"}[5m]))

Must be 0

### 7. Phase 7 — Istio Gateway (Ingress & Egress)

**Business Problem:** NexaPay's external API (used by partner banks and merchant apps) needs TLS termination, rate limiting, and header-based routing at the cluster entry point. The existing NGINX Ingress doesn't integrate with the mesh's mTLS model. Istio Gateway replaces it with a mesh-native ingress that participates in the same VirtualService routing rules used inside the cluster.

#### 7.1 Istio Gateway — Exposing NexaPay API Externally

```yaml
# nexapay-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: nexapay-gateway
namespace: nexapay-prod
spec:
  selector:
    istio: ingressgateway
servers:
    - port:
        number: 443
name: https
protocol: HTTPS
tls:
        mode: SIMPLE
credentialName: nexapay-tls-cert
hosts:
        - api.nexapay.in
    - port:
        number: 80
name: http
protocol: HTTP
tls:
        httpsRedirect: true # redirect HTTP → HTTPS
hosts:
        - api.nexapay.in
```

**📖 Istio Gateway vs Kubernetes Ingress**

- **Istio Gateway is not a Kubernetes Ingress** — it's a separate CRD that configures the Istio Ingress Gateway pod (an Envoy proxy); routing rules are in VirtualService (not Ingress annotations)
- **credentialName** — points to a Kubernetes TLS Secret containing NexaPay's certificate and private key; Istio reads it and configures TLS on the gateway automatically
- **httpsRedirect: true** — any HTTP request to api.nexapay.in returns a 301 redirect to HTTPS; PCI-DSS requirement for all public-facing payment APIs
- **Gateway + VirtualService pair** — Gateway defines what ports and hosts to listen on; VirtualService defines where to route the matched traffic; they reference each other via the gateways field in VirtualService
- EgressGateway: control all outbound traffic to external services (credit bureau, UPI, BNPL APIs) through a central proxy; enables egress mTLS, rate limiting, and full audit logging of external calls

### 8. Phase 8 — Istio vs Linkerd — Choosing Your Service Mesh

**Business Problem:** NexaPay's platform team is evaluating whether Istio is the right long-term choice. A new microservices team joining from another company has used Linkerd and argues it's simpler, faster, and has lower overhead. The decision affects 18 services, a critical financial platform, and 3 years of operational muscle memory.

**Scene 8 — NexaPay Architecture Review | "The New Team Says Linkerd is Better. Are They Right?"**

> **Vikram** _Lead DevOps Engineer — NexaPay_
> 
> "Both are production-grade. Linkerd2 uses a Rust-based micro-proxy (linkerd-proxy) instead of Envoy — it's smaller, faster, and uses about 10MB of memory per sidecar vs Envoy's 50MB+. For NexaPay's 18 services, that's a real difference. But Istio's feature set is deeper: multi-cluster, VM integration, WebAssembly extensions, JWT authentication, advanced traffic management. If you need all of that, Istio. If you need mTLS + basic traffic management + observability with lower overhead, Linkerd is excellent."

#### 8.1 Istio vs Linkerd — Detailed Comparison

Feature

Istio 1.20

Linkerd 2.14

NexaPay Verdict

Sidecar proxy

Envoy (C++, ~50MB RAM)

linkerd-proxy (Rust, ~10MB RAM)

Linkerd wins on resource efficiency

mTLS

Full mTLS, SPIFFE/X.509, cert rotation

Full mTLS, SPIFFE/X.509, cert rotation

Equal — both production-grade

Traffic management

VirtualService, DestinationRule, Gateway, fault injection, mirroring

HTTPRoute (Gateway API), traffic split via SMI

Istio wins — deeper control

Circuit breaking

Full outlierDetection + connectionPool in DestinationRule

Basic via ServiceProfile; less granular

Istio wins — NexaPay needs granular CB

Observability

Prometheus, Jaeger, Kiali, Grafana via add-ons

Built-in tap, Prometheus, Linkerd Viz dashboard

Linkerd wins — observability built-in, simpler

Installation complexity

High — istioctl, profiles, many CRDs, many knobs

Low — linkerd CLI, 2 commands, minimal config

Linkerd wins — faster to production

Multi-cluster

Full multi-cluster: flat network + gateway modes

Multi-cluster via linkerd multicluster extension

Istio wins — more mature multi-cluster

gRPC support

Full gRPC, HTTP/2, WebSocket

Full gRPC, HTTP/2

Equal for NexaPay's use cases

CNCF status

Graduated (2023)

Graduated (2021)

Both enterprise-ready

#### 8.2 Linkerd Installation — Quick Reference

```bash
# Linkerd installation — far simpler than Istio
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL \
  https://run.linkerd.io/install | sh
export PATH=$HOME/.linkerd2/bin:$PATH

# Pre-installation check
linkerd check --pre

# Install control plane (2 commands total)
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# Verify installation
linkerd check

# Inject sidecars into a namespace
kubectl annotate namespace nexapay-prod \
  linkerd.io/inject=enabled
# Install observability (Linkerd Viz)
linkerd viz install | kubectl apply -f -
linkerd viz dashboard  # opens browser dashboard
```

**📖 Linkerd Key Differences in Practice**

- **2-command install** — Linkerd's install is dramatically simpler; no profiles, no IstioOperator, no dozens of helm values; the defaults are production-appropriate for most clusters
- **Linkerd Viz** — the built-in dashboard shows live traffic, success rates, latency, and dependency graphs; it's comparable to Kiali but requires no separate configuration
- **linkerd tap** — real-time request inspection from the CLI: `linkerd viz tap deploy/payment-service` shows each live request with response code, latency, and route; extremely useful for debugging
- **ServiceProfile for traffic policy** — Linkerd's equivalent of VirtualService; defines retries and timeouts per-route; less feature-rich than Istio's VirtualService but covers 80% of use cases
- NexaPay decision: Istio for now — the granular circuit breaking and multi-cluster roadmap justify the complexity; revisit Linkerd if operational overhead becomes a team bottleneck

##### ✅ Service Mesh Best Practices — NexaPay Production Standards

- Start with PERMISSIVE mTLS and migrate namespaces to STRICT one at a time — never flip the whole cluster to STRICT without verifying every service has a sidecar injected
- Always set resource requests and limits on Envoy sidecars — use 100m CPU / 128Mi RAM as baseline; tune based on actual profiling per service
- Use outlierDetection on every DestinationRule for production services — passive health checking is free and prevents cascade failures
- Version your Deployments with a version label — without it, VirtualService subsets cannot target specific pod versions for canary releases
- Canary: always start at 1–5% weight, monitor for 30 minutes minimum, then increment; never jump from 5% to 100%
- AuthorizationPolicy: apply deny-all to all sensitive services (payment, auth, database proxies) immediately after mTLS enforcement, before any other work
- Upgrade Istio minor versions within 6 months — Istio's n-2 Kubernetes version support means falling behind creates upgrade complexity
- Exclude batch jobs and DaemonSets from sidecar injection — sidecars on short-lived pods waste resources and can cause job completion issues with Istio's TCP connection management

**Quiz: ❓ Interview Question: NexaPay's payment-service is experiencing a cascade failure — every service calling it is hanging and exhausting thread pools. You have Istio installed with mTLS but no DestinationRule configured. What is the fastest Istio configuration change to stop the cascade, and what does it do internally?**

- A) Apply a PeerAuthentication with STRICT mode — this will reject all plaintext connections and force callers to fail fast
- B) Apply a DestinationRule with outlierDetection and connectionPool limits — outlierDetection ejects consistently failing payment-service pods from the load balancer pool, and connectionPool's http1MaxPendingRequests cap makes new requests return 503 immediately instead of queuing, releasing caller thread pools within seconds
- C) Delete the payment-service pods — Kubernetes will restart them and the problem resolves automatically
- D) Increase the replica count of payment-service — more pods handle the load and prevent timeouts

> **Answer/explanation:** ✅ **Answer: B.** PeerAuthentication (A) controls encryption, not request queuing — it would add TLS overhead without stopping the cascade. Deleting pods (C) causes a brief outage and doesn't address the upstream slow dependency. Scaling up (D) doesn't help if all new pods also hang waiting for the same slow external dependency. The correct fix is a DestinationRule with two mechanisms working together: **connectionPool.http1MaxPendingRequests** immediately rejects new requests once the queue limit is hit — callers get a fast 503, release their threads, and stay healthy. **outlierDetection** additionally identifies and ejects individual payment-service pods that are returning 5xx errors, routing traffic only to healthy pods. In NexaPay's incident, this would have stopped the cascade within 10–30 seconds of the credit bureau API slowing down, instead of the 47-minute full-platform outage.

##### Common Fresher Questions — Service Mesh at NexaPay

**Q: Q: What is the difference between a VirtualService and a Kubernetes Service?**

A: **Kubernetes Service** — a stable network endpoint (ClusterIP) that load balances traffic to matching pods using kube-proxy and iptables; it knows nothing about HTTP, retries, versions, or traffic policies
**Istio VirtualService** — an Istio CRD that configures the Envoy sidecar's routing behaviour; it intercepts traffic destined for a Kubernetes Service and applies HTTP-level rules: retries, timeouts, fault injection, traffic splitting by weight or header
They work together — Kubernetes Service provides the DNS name and pod selection; VirtualService adds intelligent routing on top of that DNS resolution
Without VirtualService: traffic hits the Kubernetes Service and goes to any matching pod with no retry or weight logic. With VirtualService: traffic goes through Envoy which applies all configured rules before forwarding

**Q: Q: Why does Istio need both PeerAuthentication AND DestinationRule for mTLS?**

A: **PeerAuthentication** configures the *server-side* Envoy — what type of traffic it will accept (STRICT = mTLS only, PERMISSIVE = both)
**DestinationRule** with tls.mode: ISTIO_MUTUAL configures the *client-side* Envoy — how it initiates connections to the target service
If only PeerAuthentication is set to STRICT: the server rejects plaintext, but the client-side Envoy might still try plaintext — connection refused
If only DestinationRule is set: the client sends mTLS but the server is in PERMISSIVE mode — works, but the server will also accept plaintext from any other caller
Both together: client initiates mTLS, server requires mTLS — fully locked down in both directions

**Q: Q: What happens to a request when the Istio circuit breaker trips (connectionPool limit exceeded)?**

A: The caller's Envoy sidecar sees that the pending request count to the target service has exceeded `http1MaxPendingRequests`
Envoy immediately returns a **503 Service Unavailable** response to the calling application — without ever forwarding the request to the target service
This is a fast failure — the calling app's thread is released immediately instead of hanging; this is what prevents cascade failures
The Envoy stat `upstream_rq_pending_overflow` increments — this is what the PromQL query monitors for circuit breaker trips in NexaPay's Grafana dashboard
The 503 response should be handled by the calling service's error handling — return a cached result, a user-friendly error, or trigger a fallback; do not retry immediately (that defeats the circuit breaker)

**Q: Q: What is the difference between Istio's circuit breaking and a library like Resilience4j or Hystrix?**

A: **Library-based (Resilience4j)** — the circuit breaker logic is inside the application code; each service team must implement it; different services implement it differently; Java-only or language-specific
**Istio circuit breaking** — implemented in the Envoy sidecar proxy; language-agnostic; configured via Kubernetes YAML; applied to all services uniformly without any app code; teams don't need to know it exists
Key difference: Istio circuit breaking is at the infrastructure level — it applies even to services you don't control (third-party services, legacy services, services in other teams' repos)
Limitation of Istio CB: it's simpler than Resilience4j — no half-open state probing with custom logic, no fallback method, no event listeners; for complex circuit breaking logic, combine Istio CB (infrastructure level) with Resilience4j (application level)

### Kubernetes Service Mesh Mastery Complete 🎉

You have built NexaPay's complete service mesh platform — Istio production-profile installation on EKS, namespace-scoped sidecar injection across 18 services with zero code changes, cluster-wide STRICT mTLS resolving the PCI-DSS Level 1 compliance violation, VirtualService and DestinationRule for retries and timeouts, 95/5 canary release for payment-service v1.3, circuit breaking with outlierDetection that would have reduced the 47-minute outage to under 60 seconds, zero-trust AuthorizationPolicy allowing only auth-service and fraud-detection-service to reach payment APIs, full observability with Kiali service graph and Jaeger distributed tracing, and an Istio Gateway with TLS for the external partner bank API.

> **Vikram**
> 
> "Month-end salary credits last Friday — ₹380 crore processed in 4 hours, zero downtime. The credit-bureau API went slow again at 23:14, exactly like last time. This time: outlierDetection ejected 2 payment-service pods within 10 seconds. Circuit breaker returned 503s to callers immediately — they hit the retry logic in VirtualService, waited 500ms, and the next attempt hit a healthy pod. Kiali showed a yellow warning on payment-service for 40 seconds. 40 seconds vs 47 minutes. PCI auditor signed off on mTLS last week — the compliance finding is closed. And the canary release system means our last 3 deploys have been zero-downtime weight shifts. Zero on-call pages for deploys."

> **Priya**
> 
> "And the AuthorizationPolicy denied 14 unauthorized cross-service calls last week that nobody knew were happening — some leftover test code in notification-service was directly calling payment-service endpoints. Without the policy, those calls would have silently succeeded. With it, they returned 403 and showed up immediately in Kiali. We fixed them in 20 minutes. That's zero-trust working exactly as designed."

> **Next: Advanced Service Mesh — Multi-Cluster Istio, WebAssembly Extensions, eBPF with Cilium & Service Mesh Interface (SMI)**

> - **Multi-cluster Istio** — connect NexaPay's Mumbai production cluster and Chennai DR cluster into a single mesh; services in Mumbai can call services in Chennai transparently with automatic failover; two models: flat network (direct pod-to-pod) and gateway model (traffic crosses via Istio gateways)
> - **WebAssembly (WASM) Envoy Extensions** — write custom Envoy filters in any WASM-compatible language (Go, Rust, C++); NexaPay use case: a custom rate limiter that reads UPI transaction limits from Redis and enforces them at the Envoy layer before any request hits payment-service
> - **Ambient Mesh (Istio 1.22+)** — sidecars are optional in the new ambient mode; a per-node ztunnel proxy handles mTLS and L4 policies; a per-namespace waypoint proxy handles L7 policies; reduces sidecar overhead from 50MB/pod to near-zero for basic mTLS
> - **Cilium + eBPF as kube-proxy replacement** — replace Istio's iptables-based traffic interception with Cilium's eBPF-based interception; 40% lower latency, kernel-level network policy enforcement, Hubble for real-time network flow visibility without any sidecars
> - **Service Mesh Interface (SMI)** — a standard API specification for service meshes; write traffic policies once in SMI spec, run on Istio, Linkerd, or Consul Connect without rewriting; useful if NexaPay ever needs to migrate mesh implementations
> - **Istio + Argo Rollouts** — automated canary analysis using Prometheus metrics; instead of manually shifting VirtualService weights, Argo Rollouts automatically promotes from 5% → 20% → 50% → 100% based on error rate and latency thresholds — fully automated progressive delivery
