# 📈 Prometheus Project Mastery

> **Hey Fresher — Read This First!**
>
> Prometheus is a metrics collection and alerting system built around one simple idea: instead of your application pushing data somewhere, Prometheus *pulls* metrics from your services on a schedule, stores them as time series, and lets you query them with its own language, PromQL. It's the de facto standard for monitoring anything running in containers or Kubernetes, and it's usually the data source sitting behind every Grafana dashboard you'll ever look at.
>
> You've just joined **CargoLynx Logistics**, a same-day delivery and freight-matching platform operating across major Indian metros — think a marketplace that matches trucks to shipments in real time. Last month, a routing service silently started rejecting 8% of shipment-matching requests during peak evening hours, and nobody noticed until a warehouse partner called to ask why their trucks were idle. Your tech lead's assignment on day one: "We have zero instrumentation on the matching service. I don't want to find out about outages from phone calls anymore."

#### What You Will Learn and Build in This Project

You will instrument a real service end-to-end with Prometheus — exposing custom metrics from application code, configuring scrape targets and service discovery, writing PromQL queries that actually answer business questions, building recording rules for expensive queries, and writing alerting rules that fire through Alertmanager with proper routing and inhibition. By the end you'll understand not just the syntax but the reasoning behind every metric type and query pattern.

Metric types (counter, gauge, histogram, summary), client libraries and `/metrics` endpoints, `prometheus.yml` scrape configs, service discovery, PromQL selectors and functions, `rate()` and `increase()`, `histogram_quantile()`, recording rules, alerting rules, Alertmanager routing and inhibition, cardinality management

> **📦 Phase 1 — Instrument the Matching Service**
>
> Add a Prometheus client library to CargoLynx's shipment-matching service and expose a `/metrics` endpoint with real counters, gauges, and histograms.

> **📦 Phase 2 — Scrape Configuration and Service Discovery**
>
> Configure Prometheus to actually find and scrape the matching service across a fleet of autoscaling pods.

> **📦 Phase 3 — PromQL for Business Questions**
>
> Turn raw counters and histograms into answers: match success rate, p95 matching latency, and per-city breakdowns.

> **📦 Phase 4 — Recording Rules**
>
> Pre-compute expensive queries so dashboards stay fast as CargoLynx's metric volume grows.

> **📦 Phase 5 — Alerting Rules and Alertmanager**
>
> Write alerting rules with proper `for:` durations and route them through Alertmanager with grouping, silencing, and inhibition.

> **📦 Phase 6 — Cardinality and Scale**
>
> Learn what breaks Prometheus at scale, and how CargoLynx avoids a cardinality explosion as it onboards more cities.

**Scene 1 — CargoLynx Logistics, Pune | "We found out from a phone call"**

> **Aditya** _Senior Backend Engineer, Matching Platform_
>
> Our matching service decides which truck gets which shipment in under 200 milliseconds, at least it's supposed to. Last month it degraded to 2 seconds for almost an hour during evening peak and the first person to notice was a warehouse ops manager, not us.

> **Sneha** _Platform Architect_
>
> That's the core problem — we log everything, but logs don't answer "is the system healthy right now" at a glance, and nobody's tailing logs at 7 PM on a Tuesday. We need actual metrics: counters, rates, latency percentiles, all queryable and alertable.

> **You**
>
> So step one is just... getting numbers out of the service in the first place.

> **Aditya**
>
> Exactly. Right now the matching service has zero instrumentation. Let's fix that first, then worry about dashboards and alerts.

### 1. Phase 1 — Instrument the Matching Service

**Business Problem:** The matching service is a Python (FastAPI) application that takes shipment requests and returns a matched truck. It currently exposes no metrics at all — the only visibility is application logs, which nobody watches in real time. You need to add the `prometheus_client` library and expose meaningful counters, gauges, and a histogram.

#### 1.1 Adding the Client Library

```python
# matching_service/metrics.py
from prometheus_client import Counter, Gauge, Histogram, make_asgi_app

match_requests_total = Counter(
    "cargolynx_match_requests_total",
    "Total shipment-matching requests received",
    ["city", "shipment_type"]
)

match_failures_total = Counter(
    "cargolynx_match_failures_total",
    "Matching requests that failed to find a truck",
    ["city", "shipment_type", "reason"]
)

active_trucks_available = Gauge(
    "cargolynx_active_trucks_available",
    "Trucks currently available for matching",
    ["city"]
)

match_duration_seconds = Histogram(
    "cargolynx_match_duration_seconds",
    "Time to find and confirm a truck match",
    ["city"],
    buckets=[0.05, 0.1, 0.2, 0.5, 1, 2, 5, 10]
)

metrics_app = make_asgi_app()
```

```python
# matching_service/main.py
from fastapi import FastAPI
from .metrics import (
    match_requests_total, match_failures_total,
    active_trucks_available, match_duration_seconds, metrics_app
)
import time

app = FastAPI()
app.mount("/metrics", metrics_app)

@app.post("/match")
async def match_shipment(request: MatchRequest):
    match_requests_total.labels(city=request.city, shipment_type=request.shipment_type).inc()
    start = time.perf_counter()
    try:
        result = await run_matching_algorithm(request)
        if result is None:
            match_failures_total.labels(
                city=request.city, shipment_type=request.shipment_type, reason="no_truck_available"
            ).inc()
        return result
    finally:
        match_duration_seconds.labels(city=request.city).observe(time.perf_counter() - start)
```

> **📖 Why these four metrics, and why these types**
>
> `match_requests_total` and `match_failures_total` are both **counters** — they only ever go up, which is correct because "how many requests have we ever received" naturally accumulates and never decreases. Splitting failures into their own counter with a `reason` label (instead of trying to derive failures from requests minus successes) means you can immediately tell *why* matches are failing — no trucks available versus a downstream timeout are very different incidents. `active_trucks_available` is a **gauge** because it goes up and down as trucks come online and get assigned — a counter would be wrong here since the value isn't cumulative. `match_duration_seconds` is a **histogram**, giving you bucketed latency counts so you can later compute percentiles with `histogram_quantile()` — a plain average latency metric would hide exactly the tail latency spikes that caused last month's incident.
>
> The `.labels(city=..., shipment_type=...)` calls attach dimensions to each metric so you can slice by city or shipment type in PromQL later, without needing separate metrics per city.

**Counter vs. Gauge vs. Histogram — the decision CargoLynx made**

- **Counter** — total requests, total failures, total trucks dispatched: anything that's a running cumulative total since process start.
- **Gauge** — trucks currently available, active matching-service replicas, queue depth: anything with a current value that can rise or fall.
- **Histogram** — match duration, payload size: anything you need percentiles or SLO compliance on, at the cost of higher storage (each histogram is really N+2 time series: one counter per bucket, plus `_sum` and `_count`).

#### 1.2 What `/metrics` Actually Returns

```text
# HELP cargolynx_match_requests_total Total shipment-matching requests received
# TYPE cargolynx_match_requests_total counter
cargolynx_match_requests_total{city="pune",shipment_type="same_day"} 48213

# HELP cargolynx_match_duration_seconds Time to find and confirm a truck match
# TYPE cargolynx_match_duration_seconds histogram
cargolynx_match_duration_seconds_bucket{city="pune",le="0.05"} 1022
cargolynx_match_duration_seconds_bucket{city="pune",le="0.1"} 8841
cargolynx_match_duration_seconds_bucket{city="pune",le="0.2"} 39102
cargolynx_match_duration_seconds_bucket{city="pune",le="+Inf"} 48213
cargolynx_match_duration_seconds_sum{city="pune"} 5892.31
cargolynx_match_duration_seconds_count{city="pune"} 48213
```

> **📖 Reading the exposition format**
>
> This is plain text, human-readable on purpose — Prometheus scrapes this endpoint over HTTP and parses it directly. Every histogram bucket is a **cumulative** counter: `le="0.2"` (less-than-or-equal 0.2 seconds) includes everything already counted in `le="0.1"` and `le="0.05"`. The `+Inf` bucket always equals the total count, acting as a sanity check. `_sum` and `_count` are exposed automatically alongside the buckets so you can also compute a simple average (`_sum / _count`) without needing `histogram_quantile()` at all, if that's all you need.

### 2. Phase 2 — Scrape Configuration and Service Discovery

**Business Problem:** The matching service runs as a Kubernetes Deployment that autoscales between 4 and 40 pods depending on time of day. A static list of scrape targets would be stale within minutes. You need Prometheus to discover pods dynamically as they scale up and down.

#### 2.1 Kubernetes Service Discovery in `prometheus.yml`

```yaml
scrape_configs:
  - job_name: "cargolynx-matching-service"
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ["matching"]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: matching-service
        action: keep
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        regex: "true"
        action: keep
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
    scrape_interval: 15s
```

> **📖 What's happening line by line**
>
> `kubernetes_sd_configs` with `role: pod` tells Prometheus to ask the Kubernetes API for every pod in the `matching` namespace on every discovery cycle — no hardcoded IPs, so newly scaled-up pods are picked up automatically and terminated pods drop out. The `relabel_configs` chain filters that firehose down to only what you want: the first rule keeps only pods labeled `app: matching-service`, the second requires an explicit `prometheus.io/scrape: "true"` pod annotation as a second safety check (so nobody accidentally starts scraping a random pod that happens to share the app label). The third rule rewrites `__address__` to use the port from a `prometheus.io/port` annotation instead of a guessed default, since not every pod exposes metrics on the same port. The last two rules copy Kubernetes metadata (`pod` name, `namespace`) into labels on every scraped metric, so a query can filter or group by which specific pod served a request — useful when one bad pod is skewing p95 latency for the whole fleet.

**Quiz: Why use `kubernetes_sd_configs` instead of a static list of pod IPs in `scrape_configs`?**
- Static configs are faster to scrape
- Pod IPs in Kubernetes change every time a pod restarts or the deployment scales, so a static list goes stale almost immediately; service discovery keeps the target list current automatically
- PromQL doesn't work with static targets

> **Answer/explanation:** The second option is correct. Kubernetes pods are ephemeral by design — a rolling deploy, a crash-restart, or an autoscaling event all assign new pod IPs, and CargoLynx's matching service scales between 4 and 40 replicas throughout the day. A static target list would need constant manual updates and would either miss new pods (blind spots) or keep scraping dead IPs (wasted cycles, false "down" alerts). Service discovery re-queries the Kubernetes API on each refresh cycle and keeps the active target list accurate automatically. Static configs aren't inherently faster to scrape (option 1 is false — scrape performance depends on the target, not how it was discovered), and PromQL functions identically regardless of how targets were found (option 3 is false).

### 3. Phase 3 — PromQL for Business Questions

**Business Problem:** Aditya's team now has raw metrics flowing but no queries yet. The immediate ask: "What's our match success rate by city, right now, and what's our p95 matching latency?" — these are the two numbers that would have caught last month's incident in minutes instead of an hour.

#### 3.1 Match Success Rate by City

```promql
1 - (
  sum by (city) (rate(cargolynx_match_failures_total[5m]))
  /
  sum by (city) (rate(cargolynx_match_requests_total[5m]))
)
```

> **📖 Breaking this down**
>
> `rate(cargolynx_match_failures_total[5m])` computes the per-second rate of failures over a trailing 5-minute window for every unique label combination, then `sum by (city)` collapses the `shipment_type` and `reason` dimensions so you get one number per city rather than a chart with dozens of tangled lines. The same happens for requests. Dividing failures-rate by requests-rate gives the failure ratio, and `1 - ...` flips it into a success rate, which is the number ops teams actually want to see: "Pune is at 97.2% match success" reads better than "Pune has a 2.8% failure ratio," even though they're the same fact.

#### 3.2 p95 Matching Latency

```promql
histogram_quantile(0.95,
  sum by (le, city) (rate(cargolynx_match_duration_seconds_bucket[5m]))
)
```

> **📖 Why `sum by (le, city)` and not just `sum by (city)`**
>
> `histogram_quantile()` needs the `le` (bucket boundary) label intact to interpolate a percentile — if you summed it away, you'd be left with one number with no bucket structure and the function would fail or return nonsense. Keeping `le` alongside `city` in the `by` clause aggregates all pod-level histograms into one histogram per city, per bucket, which is exactly the shape `histogram_quantile` expects: one full histogram per city, per time step.

#### 3.3 Trucks Available vs. Demand — a Gauge in Action

```promql
sum by (city) (cargolynx_active_trucks_available)
```

> **📖 Why no `rate()` here**
>
> `active_trucks_available` is a gauge, not a counter — querying it directly gives you the current value at each scrape, which is exactly what you want for "how many trucks are online right now." Wrapping a gauge in `rate()` would be a mistake; `rate()` is only meaningful for counters that continuously accumulate.

**Quiz: A teammate wraps `active_trucks_available` in `rate(...[5m])` and gets a confusing, mostly-flat-near-zero graph. What went wrong?**
- The metric name is misspelled
- `rate()` is designed for monotonically increasing counters; applying it to a gauge that naturally rises and falls produces a meaningless result, since Prometheus treats any decrease as a possible counter reset
- Gauges can't be queried with PromQL at all

> **Answer/explanation:** The second option is correct. `rate()` assumes the input only increases over time (aside from resets back to zero, like a process restart) and calculates its slope accordingly. A gauge like `active_trucks_available` legitimately goes both up and down as trucks come online and get dispatched, so feeding it through `rate()` produces a number that doesn't represent anything real — Prometheus will interpret normal decreases as if the counter "reset," which distorts the output into something close to zero or noisy nonsense. Gauges are queried directly, or with functions like `deriv()` or `delta()` if you specifically want rate-of-change behavior for a gauge. Option 3 is false — gauges are fully queryable, just not with `rate()`.

> **Key takeaways**
> - `rate()` and `increase()` are for counters only; gauges are queried directly.
> - `histogram_quantile()` needs the `le` label preserved through any `sum by (...)` aggregation, or the percentile calculation breaks.
> - Dividing two rates (like failures/requests) gives you a traffic-independent ratio — the right shape of number for SLOs and alerts.

### 4. Phase 4 — Recording Rules

**Business Problem:** CargoLynx now has 40+ dashboard panels referencing the p95 latency query from Phase 3, and each one recomputes `histogram_quantile` over raw bucket data on every dashboard load. As more cities onboard, Prometheus's query load climbs and dashboards start feeling sluggish.

#### 4.1 Defining a Recording Rule

```yaml
# rules/cargolynx-recording-rules.yml
groups:
  - name: cargolynx_matching_slo
    interval: 30s
    rules:
      - record: city:cargolynx_match_duration_seconds:p95_5m
        expr: >
          histogram_quantile(0.95,
            sum by (le, city) (rate(cargolynx_match_duration_seconds_bucket[5m]))
          )
      - record: city:cargolynx_match_success_ratio:5m
        expr: >
          1 - (
            sum by (city) (rate(cargolynx_match_failures_total[5m]))
            /
            sum by (city) (rate(cargolynx_match_requests_total[5m]))
          )
```

```yaml
# prometheus.yml
rule_files:
  - "rules/cargolynx-recording-rules.yml"
```

> **📖 The naming convention isn't cosmetic**
>
> `city:cargolynx_match_duration_seconds:p95_5m` follows Prometheus's recommended recording rule naming pattern: `level:metric:operation`. The `city:` prefix tells anyone reading it that this is aggregated by city, `p95_5m` tells them exactly what operation and window produced it. This matters at scale — without a convention, recorded metrics become as confusing as the raw ones they were meant to simplify. `interval: 30s` means Prometheus pre-computes this result every 30 seconds and stores it as its own time series, so every dashboard panel referencing `city:cargolynx_match_duration_seconds:p95_5m` reads a cheap, already-computed number instead of re-running the full histogram aggregation on every page load.

**Raw query in every panel vs. recording rule**

- **Raw query in every panel** — fine while you have a handful of dashboards and low query volume; simplest to change since there's no separate rule file to keep in sync.
- **Recording rule** — the right call once the same expensive aggregation is used in more than a couple of places, or once query latency becomes visible to users; costs a small amount of extra storage for the precomputed series but pays for itself in query performance and consistency (every panel reads the exact same precomputed number).

### 5. Phase 5 — Alerting Rules and Alertmanager

**Business Problem:** Recording rules and dashboards are in place, but CargoLynx still relies on someone actively looking at a dashboard to catch a degradation. You need alerting rules wired to Alertmanager so the on-call engineer for the matching platform gets paged automatically — and only for things that matter.

**Scene 2 — Incident Retro, Week 3 | "Never again from a phone call"**

> **Sneha** _Platform Architect_
>
> I want two alerts minimum before we call this done: match success rate below 95% for 5 minutes, and p95 latency above 1 second for 5 minutes. Both scoped per city, because a Pune outage shouldn't page the whole company if Mumbai is fine.

#### 5.1 The Alerting Rules

```yaml
# rules/cargolynx-alerts.yml
groups:
  - name: cargolynx_matching_alerts
    rules:
      - alert: MatchSuccessRateLow
        expr: city:cargolynx_match_success_ratio:5m < 0.95
        for: 5m
        labels:
          severity: critical
          team: matching-platform
        annotations:
          summary: "Match success rate in {{ $labels.city }} is {{ $value | humanizePercentage }}"
          runbook_url: "https://wiki.cargolynx.internal/runbooks/match-success-low"

      - alert: MatchLatencyHigh
        expr: city:cargolynx_match_duration_seconds:p95_5m > 1
        for: 5m
        labels:
          severity: warning
          team: matching-platform
        annotations:
          summary: "p95 match latency in {{ $labels.city }} is {{ $value }}s"
          runbook_url: "https://wiki.cargolynx.internal/runbooks/match-latency-high"
```

> **📖 Alerting off recording rules, not raw expressions**
>
> Both alert rules reference the precomputed `city:...` metrics from Phase 4 instead of re-deriving the histogram_quantile or ratio math inline — this keeps the alerting logic and dashboard logic guaranteed to agree, and it's cheaper for Prometheus to evaluate on every scrape interval. `{{ $labels.city }}` and `{{ $value }}` are Go template functions that pull the specific city label and computed value into the human-readable summary, so the Slack notification says "Pune is at 91%" instead of a generic "an alert fired somewhere."

#### 5.2 Alertmanager Routing and Inhibition

```yaml
# alertmanager.yml
route:
  receiver: matching-platform-default
  group_by: ["alertname", "city"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 3h
  routes:
    - matchers:
        - team = "matching-platform"
          severity = "critical"
      receiver: matching-platform-pagerduty
    - matchers:
        - team = "matching-platform"
          severity = "warning"
      receiver: matching-platform-slack

inhibit_rules:
  - source_matchers:
      - alertname = "MatchSuccessRateLow"
    target_matchers:
      - alertname = "MatchLatencyHigh"
    equal: ["city"]

receivers:
  - name: matching-platform-pagerduty
    pagerduty_configs:
      - service_key: "$PAGERDUTY_MATCHING_KEY"
  - name: matching-platform-slack
    slack_configs:
      - api_url: "$SLACK_WEBHOOK_MATCHING"
        channel: "#matching-platform-alerts"
```

> **📖 Why the inhibit rule exists**
>
> When match success rate collapses in a city, latency almost always spikes too — the matching algorithm is retrying and timing out. Without the `inhibit_rules` block, both `MatchSuccessRateLow` and `MatchLatencyHigh` would fire for the same city at the same time, paging on-call twice for what's really one incident. The `equal: ["city"]` clause scopes the inhibition correctly: a critical success-rate alert in Pune silences the latency warning *for Pune only*, while Mumbai's latency warning (if unrelated) still fires normally. Routing critical alerts to PagerDuty and warnings to Slack means on-call only gets woken up for things that actually need a human awake at 2 AM.

> **Key takeaways**
> - Alert on recording rules, not raw ad hoc expressions, so alerting and dashboards never disagree.
> - `for:` durations absorb transient blips; tune them against your metric's natural noise, not just gut feel.
> - Inhibition rules stop one root cause from paging the same person twice through two different symptoms.

### 6. Phase 6 — Cardinality and Scale

**Business Problem:** CargoLynx is about to onboard 15 new cities. Someone on the team suggests adding a `driver_id` label to `match_requests_total` "for better debugging." Sneha shuts it down immediately — and you need to understand why before you make the same mistake.

#### 6.1 The Cardinality Trap

```promql
# DON'T do this:
match_requests_total{city="pune", shipment_type="same_day", driver_id="D-4471"} 1
```

> **📖 Why `driver_id` as a label is dangerous**
>
> Every unique combination of label values creates a brand-new time series in Prometheus, stored in memory. `city` has maybe 20 values, `shipment_type` has maybe 4 — that's a manageable ~80 series for this metric. Adding `driver_id`, with tens of thousands of distinct drivers, multiplies that by tens of thousands, turning one metric into hundreds of thousands of time series. This is the classic **cardinality explosion**: Prometheus's memory usage and query performance are directly driven by the number of active time series, not the number of samples per series, so a single badly chosen label can degrade the entire Prometheus instance for every team, not just the matching service.

**Comparison: what belongs in a label vs. what belongs in a log line**

- **Label** — a low-cardinality dimension you'll actually want to filter or group by in a query: `city`, `shipment_type`, `severity`, `status_code`. Bounded, predictable set of values.
- **Log line / trace attribute** — a high-cardinality identifier you need for debugging one specific request after the fact: `driver_id`, `shipment_id`, `request_id`, raw error messages. Use structured logging or tracing (like Loki or Tempo) for these instead, and link out from Grafana rather than cramming them into a metric label.

##### 🏋️ Hands-On Exercises — Extend the Project

1. Add a new counter `cargolynx_truck_reassignments_total` that fires whenever a matched truck is reassigned mid-route due to a breakdown, and write a PromQL alert for when reassignment rate exceeds 2% of active matches over 10 minutes.
2. Write a recording rule that computes the ratio of `no_truck_available` failures to total failures per city, to distinguish "we're out of capacity" from "something is broken."
3. Add a second Alertmanager route that pages the on-call warehouse-integration team (a different team than matching-platform) when a specific `reason` label value spikes, using a matcher on the `reason` label from the alert's underlying query result.
4. Simulate a cardinality problem locally: add a high-cardinality label to a test counter, generate 5,000 unique label combinations, and observe `prometheus_tsdb_symbol_table_size_bytes` and query latency before and after.
5. Build a `federation` config that lets a regional Prometheus in each city forward only its recording-rule outputs (not raw metrics) up to a global Prometheus, and explain in a short note why federating raw metrics instead would be a mistake at CargoLynx's scale.

**Quiz: CargoLynx's Prometheus memory usage doubled overnight with no code changes. What's the most likely Prometheus-specific explanation?**
- Prometheus has a memory leak that requires a restart
- A new deploy likely introduced a new label (or a label with unexpectedly many distinct values, like a UUID) on an existing metric, causing a cardinality spike
- The scrape interval was changed from 15s to 30s

> **Answer/explanation:** The second option is correct — cardinality changes are by far the most common cause of sudden, large memory jumps in Prometheus, and they're almost always traceable to a code or config change that added a new label or introduced unbounded values into an existing one (a UUID, a raw email address, a timestamp used as a label value, etc.). Checking `prometheus_tsdb_head_series` before and after the change, and cross-referencing with recent deploys, is the standard first diagnostic step. A memory leak (option 1) is possible in theory but far rarer than a cardinality issue and shouldn't be assumed first. Reducing scrape frequency (option 3) would *decrease* memory pressure over time, not increase it, since fewer samples are ingested per series.

### Prometheus Project Complete 🎉

You took CargoLynx's matching service from zero instrumentation to a fully monitored, alerting-aware system: custom counters, gauges, and histograms exposed from application code, Kubernetes service discovery that keeps scrape targets current as the fleet autoscales, PromQL queries that answer real business questions like match success rate and p95 latency, recording rules that keep dashboards fast at scale, and Alertmanager routing with inhibition so on-call gets paged once, for the right reason, on the right channel.

> **Aditya** _Senior Backend Engineer, Matching Platform_
>
> Yesterday Chennai's success rate dipped to 93% for six minutes. We got paged, fixed a bad deploy, and the warehouse partners never even noticed. That's the whole point.

> **Sneha** _Platform Architect_
>
> And we did it without a single new label that's going to blow up our cardinality in six months. That part matters just as much as catching the incident.

> **You**
>
> Honestly the instrumentation felt like the easy part — the real skill was deciding what *not* to turn into a label.

> **Next: Grafana Project Mastery**
>
> - Take everything scraped and computed here and turn it into dashboards a non-engineer can read
> - Learn how Grafana-managed alerting differs from Prometheus/Alertmanager alerting, and when to use each
> - Build template-variable-driven dashboards so one view works across every city CargoLynx operates in
