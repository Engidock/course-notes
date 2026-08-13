# 🧩 Microservices & Kubernetes Project Mastery

> **👋 Hey Fresher — Read This First!**

> A monolith is one big application where every feature — orders, inventory, payments, shipping — lives in a single codebase and deploys as a single unit. That's fine at small scale, but once a company grows, one slow feature can crash the entire app, one team's bug blocks every other team's release, and you can't scale "just the busy part." Microservices split that one big application into small, independently deployable services — each owned by a small team, each with its own database, each scaled on its own. Kubernetes is what makes running dozens of these small services practical: it schedules them, restarts them, load-balances between them, and lets them find each other by name instead of by hardcoded IP address.

> **Company in this project:** Fabrilo — a mid-size e-commerce company in Bengaluru selling home furniture online. Fabrilo started five years ago as a single Django monolith called "fabrilo-core" that handles catalog browsing, cart, checkout, and order tracking all in one codebase. It has grown into a 40-engineer problem: a bug in the recommendation engine last quarter took down checkout for two hours. You just joined as a Junior Platform Engineer. Your mentor is Karthik, and the engineering lead driving the decomposition is Meera. Let's break fabrilo-core apart, one bounded context at a time, and run it on Kubernetes.

#### What You Will Learn and Build in This Project

You will take a single monolithic e-commerce codebase and decompose it into independently deployable microservices, containerize each one, and run them on Kubernetes with real inter-service communication, service discovery, resilience patterns, and independent scaling — learning the difference between a "microservice" as a buzzword and a microservice as an actual engineering decision with real trade-offs.

Bounded Contexts, Domain-Driven Design, Service Decomposition, Dockerfile per Service, Kubernetes Deployments and Services, REST and gRPC Communication, Service Discovery, Circuit Breakers and Retries, Independent Scaling, API Gateway

> **📦 Phase 1 — Identify the Seams**
>
> Use Domain-Driven Design thinking to find the natural bounded contexts inside fabrilo-core, and decide which one to extract first.

> **📦 Phase 2 — Extract and Containerize**
>
> Pull the Orders context out into its own service, give it its own Dockerfile and its own PostgreSQL schema, and stop sharing code with the monolith.

> **📦 Phase 3 — Talk Over the Network**
>
> Replace in-process Python function calls with HTTP calls between services, and understand what changes when a function call becomes a network call.

> **📦 Phase 4 — Deploy and Discover on Kubernetes**
>
> Write Deployments and Services for each microservice, and use Kubernetes DNS so services find each other by name, not by hardcoded IP.

> **📦 Phase 5 — Make It Resilient**
>
> Add retries, timeouts, and a circuit breaker so a slow Orders service doesn't cascade into a dead Catalog service.

> **📦 Phase 6 — Scale Independently and Gate Traffic**
>
> Put an API Gateway in front of everything, and prove that Orders can scale to 10 replicas during a flash sale while Catalog stays at 2.

**Scene 1 — Fabrilo Engineering Floor, Bengaluru | The Two-Hour Checkout Outage**

> **Meera** _Engineering Lead — Fabrilo_
>
> Karthik, walk the new hire through what happened last month. Our recommendation engine — "customers who bought this also bought that" — has a memory leak. It's part of fabrilo-core, the same Django process that handles checkout. When the recommendation worker ran out of memory, the whole pod got OOMKilled, including checkout. We lost two hours of sales during our biggest week of the quarter, because a *recommendation feature* took down *payments*.

> **Karthik** _Senior Platform Engineer — Fabrilo_
>
> Right. That's the core argument for microservices, and I want you to internalize it before you write a single line of code: it's not about "microservices are modern and monoliths are old." It's about blast radius. If Recommendations is its own service, it can crash, get OOMKilled, restart, whatever — and Checkout never even notices. Different process, different pod, different failure domain.

> **You** _Junior Platform Engineer — Day 1 on the Decomposition Project_
>
> So do we just... split every feature into its own service right away?

> **Karthik** _Senior Platform Engineer_
>
> No — and that's the mistake most teams make. Splitting badly is worse than not splitting at all. You end up with a "distributed monolith": ten services that are still so tightly coupled they have to deploy together, except now every function call is also a slow, unreliable network call. We're going to use bounded contexts from Domain-Driven Design to find the *real* seams — the places where the coupling is naturally low — and extract there first.

### 1. Phase 1 — Identify the Seams: Bounded Contexts

**Business Problem:** fabrilo-core is a single Django app with a single PostgreSQL database. Catalog, Cart, Orders, and Recommendations all share the same models.py, the same database connections, and the same deploy pipeline. A bug or a slow query in any one of them can take down all of them, and every deploy requires re-testing the entire app even if you only touched one feature.

**Scene 2 — Whiteboard Session | Drawing the Boundaries**

> **Meera** _Engineering Lead_
>
> A bounded context is a part of the business with its own vocabulary and its own rules. In Catalog, a "Product" has a price, description, and stock count. In Orders, a "Product" is really just a product_id and a price-at-time-of-purchase — Orders doesn't care about descriptions or images at all. That's a signal: Catalog and Orders can live apart, as long as Orders keeps a small copy of what it needs.

#### 1.1 Mapping fabrilo-core's Bounded Contexts

```
fabrilo-core monolith — current state
=======================================
Django app (single deployable, single DB: fabrilo_db)

  ┌─────────────────────────────────────────────┐
  │  models.py (all contexts, one file)          │
  │  ├── Product, Category, Inventory   (Catalog)│
  │  ├── Cart, CartItem                 (Cart)   │
  │  ├── Order, OrderItem, Payment      (Orders) │
  │  └── RecommendationScore            (Recos)  │
  └─────────────────────────────────────────────┘

Proposed bounded contexts:
  1. Catalog       — product data, search, categories
  2. Cart          — session-scoped, in-progress carts
  3. Orders        — checkout, payment status, order history
  4. Recommendations — "customers also bought" (read-heavy, ML-driven)
```

> **📖 Why Extract Orders First**
>
> Karthik picked Orders as the first extraction target for a specific reason: Orders has the clearest boundary (it only needs a *snapshot* of product data, not live access to Catalog's full schema), it has independent scaling needs (checkout traffic spikes during sales while catalog browsing stays flat), and it's the highest business risk — the recent outage makes it the easiest one to get budget and buy-in for. You don't extract the *easiest* service first — you extract the one with the clearest boundary and the strongest business case.

#### 1.2 The Strangler Fig Pattern — Extract Without a Big-Bang Rewrite

```
Strangler Fig Migration — Orders Extraction
=============================================
Week 1-2:  New orders-service exists, but fabrilo-core
           still owns writes. orders-service only reads
           (shadow mode) — compare its answers to the monolith's.

Week 3-4:  Route 10% of real checkout traffic to
           orders-service via feature flag. Monitor error
           rate and latency against the monolith baseline.

Week 5-6:  Ramp to 100% of checkout traffic on
           orders-service. fabrilo-core's Order model
           becomes read-only, then gets deleted.
```

> **📖 Strangler Fig — Why Not Just Cut Over Overnight**
>
> Named after a fig vine that slowly grows around a host tree until the tree is no longer needed. You never do a risky "big bang" cutover where you flip a switch and hope. Instead, the new service grows alongside the old one, taking over traffic gradually, with the ability to roll back to the monolith at any point during the migration. For Fabrilo — a live e-commerce site processing real payments — a big-bang rewrite of Orders would be an unacceptable risk.

**Decomposition Approach**

- **Extract by bounded context (Domain-Driven Design)** — split along business capability boundaries where coupling is naturally low. This is what Fabrilo is doing — it produces services that map to how the business actually works and rarely need to change together.
- **Extract by technical layer (e.g. "auth service", "logging service")** — split by infrastructure concern. Useful for genuinely shared platform capabilities, but splitting business logic this way tends to produce services that must deploy together anyway — a distributed monolith.

**Quiz: Fabrilo's Recommendations service needs product names and prices to display "customers also bought." What's the correct approach in a microservices architecture?**
- Give the Recommendations service direct read access to Catalog's PostgreSQL database
- Recommendations calls Catalog's API to fetch product data it needs, or subscribes to a Catalog event stream to keep a local read cache
- Merge Recommendations into the Catalog service so they share one database
- Recommendations should not need any product data at all

> **Answer/explanation:** The second option is correct. A core microservices rule is that **each service owns its own data, and no other service touches that database directly** — not even for reads. If Recommendations reads Catalog's database directly, you've created a hidden coupling: Catalog can never change its schema without breaking Recommendations, even though there's no visible dependency in the code. The correct pattern is either synchronous API calls (Recommendations calls `GET /products/{id}` on Catalog) or asynchronous event-driven sync (Catalog publishes a `ProductUpdated` event, Recommendations keeps its own local cache updated). Merging the services defeats the purpose of decomposition, and pretending no data is needed ignores the actual business requirement.

### 2. Phase 2 — Extract and Containerize the Orders Service

**Business Problem:** Orders needs to become a standalone, independently deployable unit — its own codebase, its own Dockerfile, its own database schema — with zero shared Python imports from fabrilo-core. Anything shared today (models, utility functions) has to either be duplicated intentionally or exposed through an API.

#### 2.1 The Orders Service Dockerfile

```dockerfile
# orders-service/Dockerfile
FROM python:3.12-slim AS base

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd --create-home --uid 1001 orderssvc
USER orderssvc

EXPOSE 8001
CMD ["gunicorn", "orders_service.wsgi:application", \
     "--bind", "0.0.0.0:8001", "--workers", "3"]
```

> **📖 One Dockerfile, One Service, One Responsibility**
>
> `FROM python:3.12-slim` — a slim base image keeps the container small and reduces attack surface compared to a full `python:3.12` image with build tools baked in. `USER orderssvc` — the container runs as a non-root user; if an attacker exploits a vulnerability in the app, they don't get root inside the container. `EXPOSE 8001` — Orders runs on its own port, separate from Catalog (8000) and Recommendations (8002), so they can all run side-by-side during local development with `docker compose`. Every microservice gets exactly one Dockerfile like this — no shared base image with the monolith's dependencies baked in.

#### 2.2 Orders Gets Its Own Database Schema

```yaml
# orders-service/k8s/postgres-orders.yaml (StatefulSet excerpt)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-orders
spec:
  serviceName: postgres-orders
  replicas: 1
  selector:
    matchLabels:
      app: postgres-orders
  template:
    metadata:
      labels:
        app: postgres-orders
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_DB
              value: fabrilo_orders
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: orders-db-secret
                  key: username
          volumeMounts:
            - name: orders-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: orders-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

> **📖 Database-per-Service**
>
> This is a separate PostgreSQL instance (`fabrilo_orders`), completely independent from Catalog's database. No foreign keys reach across service boundaries — Orders stores a `product_id` and a `price_at_purchase` as plain columns, not a foreign key into Catalog's `products` table, because Catalog's database isn't even reachable from Orders. This is the single most important rule in microservices: **shared databases recreate the tight coupling you were trying to escape.** If two services share a database, you don't have two services — you have one service with two deploy pipelines, which is strictly worse.

> **Key takeaways**
> - Extraction starts with finding real bounded contexts, not arbitrary technical splits — use Domain-Driven Design vocabulary boundaries as your guide.
> - Use the Strangler Fig pattern for live production systems — shadow traffic, then gradual rollout, never a big-bang rewrite.
> - Database-per-service is non-negotiable. A shared database between "microservices" is a distributed monolith wearing a costume.
> - Each service gets its own Dockerfile, its own port, its own deploy pipeline, and its own on-call rotation ideally.

### 3. Phase 3 — Talk Over the Network: REST and gRPC

**Business Problem:** Inside the monolith, checkout called `Product.objects.get(id=...)` — a Python function call that takes microseconds and never fails unless the database is down. Once Orders and Catalog are separate services, that becomes an HTTP call across the network — which can be slow, can time out, and can fail in ways a function call never could. Every engineer on the team needs to understand what actually changes.

**Scene 3 — Code Review | "Why Does This Need a Timeout?"**

> **You** _Junior Platform Engineer_
>
> I wrote the call from Orders to Catalog to fetch product details during checkout — it's just a `requests.get()` call. Karthik flagged it in review and asked where the timeout is. I didn't think it needed one; it's just an HTTP GET.

> **Karthik** _Senior Platform Engineer_
>
> That's exactly the mental shift I want you to make this week. A Python function call either returns or raises immediately. A network call can hang for 30 seconds, or 5 minutes, waiting for a TCP connection that will never complete, because Catalog's pod is overloaded and not accepting new connections. Without a timeout, one slow dependency can pile up threads in Orders until Orders itself falls over. Every network call gets a timeout. No exceptions.

#### 3.1 REST Call from Orders to Catalog

```python
# orders_service/clients/catalog_client.py
import requests
from requests.exceptions import Timeout, ConnectionError

CATALOG_BASE_URL = "http://catalog-service.fabrilo.svc.cluster.local:8000"

def get_product(product_id: str) -> dict:
    try:
        response = requests.get(
            f"{CATALOG_BASE_URL}/api/products/{product_id}",
            timeout=(2, 3),  # (connect timeout, read timeout) in seconds
        )
        response.raise_for_status()
        return response.json()
    except Timeout:
        raise CatalogUnavailableError(
            f"Catalog service timed out fetching product {product_id}"
        )
    except ConnectionError:
        raise CatalogUnavailableError("Catalog service unreachable")
```

> **📖 The Timeout Tuple and the Kubernetes DNS Name**
>
> `CATALOG_BASE_URL` uses `catalog-service.fabrilo.svc.cluster.local` — Kubernetes' internal DNS name for a Service named `catalog-service` in the `fabrilo` namespace. Orders never hardcodes an IP address; Kubernetes resolves this name to whichever Catalog pods are currently healthy. `timeout=(2, 3)` sets a 2-second connect timeout and a 3-second read timeout — if Catalog doesn't even accept a connection within 2 seconds, or accepts one but doesn't respond within 3 more seconds, the call fails fast instead of hanging. Failing fast with a clear error is always better than hanging silently.

#### 3.2 When to Reach for gRPC Instead of REST

**REST vs gRPC for Fabrilo's Internal Service Calls**

- **REST (JSON over HTTP)** — use for the Orders → Catalog call above. It's simple to debug with `curl`, browser-friendly, easy for any engineer to reason about, and the request volume (checkout traffic) doesn't need the extra performance gRPC offers. Also the right choice for anything a browser or third-party partner calls directly.
- **gRPC (protobuf over HTTP/2)** — Fabrilo uses this for the Recommendations service's internal calls to a high-throughput scoring engine that gets called on every single product page view (much higher volume than checkout). gRPC's binary protobuf encoding is smaller and faster to parse than JSON, and HTTP/2 multiplexing lets many requests share one connection efficiently. The cost is that it's less human-readable and requires generating client/server code from `.proto` files — worth it only when you're calling a service at very high frequency, internally, between services you control.

#### 3.3 Retries with Exponential Backoff

```python
# orders_service/clients/catalog_client.py (continued)
import time
import random

def get_product_with_retry(product_id: str, max_attempts: int = 3) -> dict:
    for attempt in range(1, max_attempts + 1):
        try:
            return get_product(product_id)
        except CatalogUnavailableError:
            if attempt == max_attempts:
                raise
            backoff = (2 ** attempt) + random.uniform(0, 1)  # jitter
            time.sleep(backoff)
    raise CatalogUnavailableError("Exhausted retries")
```

> **📖 Exponential Backoff with Jitter**
>
> If Catalog is briefly overloaded, retrying immediately just adds more load at the worst possible moment. `2 ** attempt` waits 2s, then 4s, then 8s between retries — giving Catalog time to recover. The `random.uniform(0, 1)` jitter prevents a "thundering herd": if 50 Orders pods all failed at the same instant and all retried with the exact same delay, they'd all hammer Catalog again at the exact same moment. Jitter spreads those retries out over time.

**Quiz: Orders calls Catalog, which calls Inventory, which calls a third-party warehouse API that's currently down and hanging for 30 seconds per request. Without any resilience patterns, what happens to Orders under checkout load?**
- Nothing — Orders and Inventory are unrelated services so Orders is unaffected
- Orders' threads/connections pile up waiting on Catalog, which is waiting on Inventory, which is waiting on the warehouse API — Orders eventually runs out of capacity and checkout fails for everyone, even orders that don't need inventory
- Kubernetes automatically detects the slow warehouse API and reroutes traffic
- The request fails instantly with a clear error

> **Answer/explanation:** The second option describes a real, common failure mode called cascading failure. Without timeouts and circuit breakers, a slowdown at the bottom of a call chain (the third-party warehouse API) propagates upward: Inventory's threads block waiting on the warehouse, Catalog's threads block waiting on Inventory, and Orders' threads block waiting on Catalog. Eventually Orders exhausts its worker threads or connection pool and can't serve *any* checkout request — including ones that never even needed inventory data. This is exactly why Phase 5 of this project adds circuit breakers: after enough failures, the circuit "opens" and Orders stops even trying to call the failing dependency, failing fast instead of piling up blocked threads. Kubernetes has no idea a third-party API is slow — that's an application-level concern, not an orchestration-level one.

### 4. Phase 4 — Deploy and Discover on Kubernetes

**Business Problem:** Each service (Catalog, Orders, Recommendations) needs its own Deployment, its own Service for stable networking, and a way to be reached by name from any other service in the cluster — without anyone hardcoding pod IPs that change every time a pod restarts.

#### 4.1 The Orders Deployment and Service

```yaml
# orders-service/k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
  namespace: fabrilo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders-service
  template:
    metadata:
      labels:
        app: orders-service
    spec:
      containers:
        - name: orders
          image: fabrilo/orders-service:1.4.0
          ports:
            - containerPort: 8001
          envFrom:
            - configMapRef:
                name: orders-config
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: orders-service
  namespace: fabrilo
spec:
  type: ClusterIP
  selector:
    app: orders-service
  ports:
    - port: 8001
      targetPort: 8001
```

> **📖 Why the Service Name Matters More Than Any IP**
>
> Other services never talk to Orders' pod IPs directly — those change constantly as pods restart, scale, and get rescheduled. They talk to `orders-service.fabrilo.svc.cluster.local:8001`, a name that Kubernetes' internal DNS resolves to whichever Orders pods currently pass their readiness probe. This is what "service discovery" means in Kubernetes: discovery happens via DNS, backed by a live, constantly updated list of healthy pod IPs — not a static config file someone has to update by hand.

#### 4.2 One Namespace per Environment, Shared Across All Services

```bash
# Every Fabrilo microservice deploys into the same namespace per environment
kubectl create namespace fabrilo-staging
kubectl create namespace fabrilo-prod

kubectl apply -f catalog-service/k8s/ -n fabrilo-staging
kubectl apply -f orders-service/k8s/ -n fabrilo-staging
kubectl apply -f recommendations-service/k8s/ -n fabrilo-staging
```

> All of Catalog, Orders, and Recommendations run inside the same `fabrilo-staging` namespace. This lets them find each other by short DNS names (`orders-service` instead of the full FQDN) since same-namespace lookups don't need the namespace suffix, while `fabrilo-staging` and `fabrilo-prod` stay fully isolated from each other — a bug deployed to staging cannot reach production's database or services.

> **Key takeaways**
> - Kubernetes Services are the mechanism for microservice-to-microservice discovery — DNS names backed by live health checks, not static IP lists.
> - Group related microservices in the same namespace per environment so they can use short DNS names and share RBAC/NetworkPolicy boundaries.
> - `resources.requests` and `resources.limits` matter even more in microservices than in a monolith — you now have many small containers competing for node resources instead of one big one.

### 5. Phase 5 — Make It Resilient: Circuit Breakers

**Business Problem:** During a flash sale, if the Inventory service starts timing out, Orders should stop hammering it with retries and instead fail fast with a clear "try again shortly" message — rather than letting the slowdown cascade and take down checkout entirely, as it nearly did last quarter.

**Scene 4 — Incident Retro | Circuit Breaker Design**

> **Meera** _Engineering Lead_
>
> A circuit breaker works exactly like the one in your house's fuse box. Normal state is "closed" — current flows, requests go through. If failures cross a threshold, the breaker "opens" — it stops sending requests to the failing dependency entirely, for a cooldown period, and returns an error immediately instead of waiting on a call you already know will fail. After the cooldown, it goes "half-open" and lets one test request through to see if the dependency has recovered.

#### 5.1 A Simple Circuit Breaker Around the Inventory Client

```python
# orders_service/clients/circuit_breaker.py
import time

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.state = "closed"  # closed, open, half_open
        self.opened_at = None

    def call(self, func, *args, **kwargs):
        if self.state == "open":
            if time.time() - self.opened_at > self.recovery_timeout:
                self.state = "half_open"
            else:
                raise CircuitOpenError("Inventory circuit is open — failing fast")

        try:
            result = func(*args, **kwargs)
        except Exception:
            self.failure_count += 1
            if self.failure_count >= self.failure_threshold:
                self.state = "open"
                self.opened_at = time.time()
            raise
        else:
            self.failure_count = 0
            self.state = "closed"
            return result
```

> **📖 The Three States of a Circuit Breaker**
>
> **closed** — normal operation, calls pass through, failures are counted. **open** — after `failure_threshold` (5) consecutive failures, the breaker trips; for the next `recovery_timeout` (30) seconds, every call fails instantly with `CircuitOpenError` without even attempting the network call — this is what protects Orders from piling up blocked threads waiting on a dead Inventory service. **half_open** — after the cooldown, the next call is allowed through as a test; if it succeeds, the breaker closes and normal traffic resumes; if it fails, the breaker reopens for another cooldown period.

**Circuit Breaker vs Simple Retry**

- **Retry with backoff** — good for transient, short-lived failures (a single dropped packet, a momentary blip). Keeps trying the same call a few times.
- **Circuit breaker** — good for sustained outages (a dependency is genuinely down for minutes). Stops trying entirely for a cooldown window, protecting your own service's resources instead of wasting them on calls that will keep failing. In production, you use both together: retry a few times with backoff, and if the circuit breaker trips because failures kept happening across many calls, stop retrying altogether until the cooldown passes.

### 6. Phase 6 — Scale Independently and Gate Traffic

**Business Problem:** During Fabrilo's Diwali flash sale, checkout traffic (Orders) spikes 8x while catalog browsing (Catalog) only doubles. In the monolith, this meant over-provisioning the *entire* app for Orders' peak — wasting money the other 360 days a year. With microservices, each one scales independently based on its own load.

#### 6.1 Independent HPA per Service

```yaml
# orders-service/k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orders-service-hpa
  namespace: fabrilo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orders-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
```

> **📖 Independent Scaling Is the Real Payoff**
>
> Orders scales from 3 to 20 replicas during the flash sale, while Catalog's own HPA (a separate object, same pattern, different `maxReplicas`) might only go from 2 to 5. Before decomposition, scaling meant adding more copies of the *entire monolith* — every feature, whether it needed the extra capacity or not. Now you pay for exactly the capacity each business capability needs, and Recommendations, which is cheap to run and rarely spikes, never scales past 2 replicas at all.

#### 6.2 API Gateway — One Front Door for Many Services

```yaml
# fabrilo-gateway/k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fabrilo-gateway
  namespace: fabrilo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: api.fabrilo.in
      http:
        paths:
          - path: /catalog(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: catalog-service
                port:
                  number: 8000
          - path: /orders(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: orders-service
                port:
                  number: 8001
          - path: /recommendations(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: recommendations-service
                port:
                  number: 8002
```

> **📖 The Gateway Hides the Decomposition from the Outside World**
>
> Fabrilo's mobile app and website only ever talk to `api.fabrilo.in` — they have no idea that `/orders/*` is a completely different service, running in a different pod, scaled differently, deployed on a different schedule, than `/catalog/*`. The Ingress routes each path prefix to the right internal Service. This also means the team can split, merge, or rewrite internal services over time without ever changing a public URL — the gateway is the stable contract with the outside world.

**Quiz: Fabrilo wants to rewrite the Recommendations service from Python to Go for performance. Because it was properly decomposed as a microservice, what does this rewrite require from Catalog and Orders?**
- Catalog and Orders must be rewritten too, since they're all part of the same deployable
- Nothing — as long as the new Go service implements the same API contract (same endpoints, same request/response shapes), Catalog and Orders never notice the change happened
- The entire cluster needs to be redeployed
- Catalog and Orders need new database migrations

> **Answer/explanation:** The second option is the entire point of microservices done correctly. Because Recommendations is its own independently deployable service with its own Dockerfile and its own database, and other services only interact with it through its published API (not by importing its code or reading its database directly), the *implementation* behind that API can change completely — different language, different framework, different internal data model — with zero impact on any caller, as long as the API contract stays the same. This is the payoff for the coupling discipline enforced throughout this whole project: no shared databases, no shared code imports, only network calls through defined APIs. A poorly decomposed "distributed monolith" would not have this property — a language rewrite would ripple through every service that had informally come to depend on Recommendations' internals.

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Extract the Cart service:** Following the same pattern as Orders, pull Cart out of fabrilo-core into its own service with its own Dockerfile, its own Redis-backed session store (carts are ephemeral, so Redis fits better than PostgreSQL here), and its own Deployment/Service pair.
2. **Add distributed tracing:** Instrument Orders, Catalog, and Recommendations with OpenTelemetry so a single checkout request can be traced end-to-end across all three services, showing exactly where time is spent when checkout is slow.
3. **Write a contract test:** Using a tool like Pact, write a consumer-driven contract test between Orders (consumer) and Catalog (provider) so that if Catalog's team changes the `/api/products/{id}` response shape in a breaking way, their CI pipeline fails before it ever reaches production.
4. **Add a NetworkPolicy:** Write a Kubernetes NetworkPolicy that only allows Orders to receive traffic from the API Gateway and from Recommendations (not from Catalog directly), demonstrating that service-to-service access can be locked down at the network layer, not just by convention.
5. **Simulate the cascading failure:** Deploy a deliberately slow mock warehouse API (sleep 30 seconds per request), point Inventory at it, and observe Orders' behavior with the circuit breaker disabled versus enabled. Capture the difference in checkout success rate under load using a simple load-testing script.

### Microservices & Kubernetes Project Complete 🎉

You took Fabrilo's monolithic e-commerce backend and decomposed it into independently deployable, independently scalable microservices — Catalog, Orders, and Recommendations — each with its own database, its own Dockerfile, and its own Kubernetes Deployment. You replaced in-process function calls with resilient network calls using timeouts, retries with backoff, and circuit breakers, and you fronted the whole system with an API Gateway so the outside world never sees the internal decomposition.

> **Meera**
>
> "The real test came during this year's Diwali sale. Orders scaled to 18 replicas automatically while Catalog barely moved. And when the third-party warehouse API had an outage for eleven minutes, the circuit breaker in Orders kicked in — checkout degraded gracefully with a 'processing may be delayed' message instead of crashing entirely. That's the difference between an outage and an incident report that just says 'handled automatically.'"

> **Karthik**
>
> "You extracted Orders the right way — bounded context first, strangler fig migration, database-per-service from day one. Most teams skip straight to writing microservices code and end up with a distributed monolith that's harder to operate than the original. You did the boring architecture homework first, and it paid off."

> **You**
>
> "The biggest shift for me was realizing a network call is never 'just like' a function call. Every single one needs a timeout, needs to handle the dependency being slow or down, and needs a plan for what happens when it fails. That mindset now shows up in every service I write."

> **Next: Service Mesh & Advanced Kubernetes Operations**

> - Service mesh (Istio or Linkerd) — get mTLS, retries, and circuit breaking automatically injected as a sidecar, without writing resilience code in every service
> - Distributed tracing with Jaeger or Tempo — visualize a request's full path across every microservice it touches
> - GitOps with Argo CD — deploy each microservice through Git commits instead of manual `kubectl apply`
> - Event-driven communication with Kafka — decouple services further by publishing events instead of making synchronous calls
> - Multi-cluster and canary deployments — roll out a new version of one microservice to 5% of traffic before going to 100%
