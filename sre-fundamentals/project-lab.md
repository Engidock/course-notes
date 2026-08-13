# 📈 SRE Fundamentals Project Mastery

> **👋 Hey Fresher — Read This First!**

> Site Reliability Engineering (SRE) replaces the vague question "is it up?" with a precise, measurable question: "are we meeting the reliability target we promised, and how much room do we have left before we break that promise?" Instead of chasing 100% uptime — which is both impossible and usually not worth the cost — SRE defines exactly how reliable a service needs to be (an SLO), measures reliability with real numbers (SLIs), and tracks the gap between "perfect" and "target" as a spendable budget (the error budget) that the whole team, not just on-call, uses to make decisions like "can we ship this risky feature this week, or should we focus on stability first?"

> **Company in this project:** StreamKarma — a music and podcast streaming platform based in Hyderabad, competing directly with global players in the Indian market. Their core "play a song" flow is the single most important user action in the entire product — if it's slow or fails, users churn to a competitor within minutes. You just joined as a Junior SRE. Your mentor is Arjun, and the engineering director who wants engineering decisions driven by data instead of gut feeling is Kavya. Let's build StreamKarma's first real SLO for the playback service.

#### What You Will Learn and Build in This Project

You will define a Service Level Indicator for StreamKarma's playback API, set a Service Level Objective backed by real business reasoning, instrument the service to expose that SLI as a Prometheus metric, calculate an error budget, write PromQL queries to track burn rate, and build multi-window multi-burn-rate alerts that catch both fast, severe outages and slow, creeping degradations — the same alerting pattern used at every major tech company running SRE.

SLI, SLO, SLA, Error Budgets, Prometheus, PromQL, Multi-Window Multi-Burn-Rate Alerting, Toil Reduction, Reliability Culture

> **📦 Phase 1 — SLI vs SLO vs SLA**
>
> Understand the three terms precisely, because mixing them up leads to alerting on the wrong thing entirely.

> **📦 Phase 2 — Define the SLI for Playback**
>
> Pick the one metric that actually represents whether a StreamKarma user had a good experience playing a song.

> **📦 Phase 3 — Instrument and Set the SLO**
>
> Expose the SLI as a Prometheus metric from the playback service, and set a target backed by real business and user research.

> **📦 Phase 4 — Calculate the Error Budget**
>
> Turn the SLO into a concrete, spendable number: how many failed requests are we allowed before we breach our promise?

> **📦 Phase 5 — Multi-Window, Multi-Burn-Rate Alerting**
>
> Build alerts that catch a severe five-minute outage fast, and a slow six-hour degradation before it quietly burns the whole month's budget.

> **📦 Phase 6 — The Error Budget Policy**
>
> Turn the error budget into an actual organizational decision-making tool, not just a dashboard number nobody looks at.

**Scene 1 — StreamKarma War Room, Hyderabad | "We Don't Know If We're Actually Reliable"**

> **Kavya** _Engineering Director — StreamKarma_
>
> Divya, here's the problem I want you to help fix. Right now our on-call gets paged whenever CPU crosses 80% or a pod restarts. Half those pages are meaningless — CPU spikes for ten seconds and recovers on its own, nobody's experience was affected. Meanwhile last month we had a slow database connection leak that degraded playback start times for six hours straight, and *nothing* paged, because no single metric crossed a hard threshold. We were "up" the whole time by every alert we had, and we were still failing our users.

> **Arjun** _Senior SRE — StreamKarma_
>
> That's the exact gap SRE closes. We stop asking "is the CPU high" and start asking "what fraction of the time did a real user get a good experience, and how does that compare to the reliability target we've committed to?" That's a Service Level Indicator measured against a Service Level Objective. Get this right, and alerts start meaning something — they page you when users are actually hurting, and they stay quiet when they're not.

> **Divya (You)** _Junior SRE — Day 1_
>
> So instead of "the server is unhealthy," we alert on "users are having a bad time" — measured precisely instead of guessed at?

> **Arjun** _Senior SRE_
>
> Exactly. And the "precisely" part matters more than people expect. We're going to build this for the playback API this week, end to end — definition, instrumentation, error budget, and alerting.

### 1. Phase 1 — SLI vs SLO vs SLA: Getting the Vocabulary Right

**Business Problem:** These three terms get used interchangeably by people who haven't worked with them, and that confusion leads to real mistakes — like setting an internal engineering target identical to a legally binding customer contract, which either makes engineering overly conservative or makes the legal team nervous.

**SLI vs SLO vs SLA — Three Different Things**

- **SLI (Service Level Indicator)** — the actual, measured metric. A number you calculate from real data, e.g. "99.92% of playback requests in the last 5 minutes returned a song stream within 2 seconds." This is a measurement, not a target.
- **SLO (Service Level Objective)** — the internal target you set for that SLI, e.g. "99.9% of playback requests succeed within 2 seconds, measured over a rolling 30 days." This is what your team is actually held accountable to internally, and what drives engineering decisions.
- **SLA (Service Level Agreement)** — an external, often contractual promise to customers, usually with financial penalties attached, e.g. "99.5% uptime or you get a service credit." SLAs are typically set *looser* than your internal SLO, so you breach your own internal target and get a chance to fix things before you ever breach the customer-facing contractual promise.

> **📖 Why the Gap Between SLO and SLA Matters**
>
> StreamKarma's internal SLO for playback might be 99.9%, while any customer-facing SLA (for enterprise/B2B partners embedding StreamKarma's API) is set looser, at 99.5%. That 0.4% gap is deliberate — it's a buffer. If the team is burning error budget against the 99.9% internal target, they get to react and fix things *before* they ever risk breaching the contractual 99.5% SLA and owing a partner a financial penalty. Setting your SLO equal to your SLA removes this buffer entirely and turns every engineering incident into a potential legal/financial one.

### 2. Phase 2 — Define the SLI for Playback

**Business Problem:** StreamKarma's playback flow has many possible things to measure — HTTP status codes, latency, buffering events, song load time. Picking the wrong one means you could have a "healthy" SLI while users are still having a miserable experience.

**Scene 2 — Design Review | Picking the Right Signal**

> **Arjun** _Senior SRE_
>
> Don't just measure "HTTP 200 vs 500." A request that returns 200 but takes 11 seconds to start playing audio is a failure from the user's perspective, even though no error occurred. The SLI has to combine correctness *and* speed, because that's what "good experience" actually means for a streaming product.

#### 2.1 The SLI Definition

```
StreamKarma Playback SLI — Definition
========================================
SLI = (number of "good" playback requests) / (total playback requests)

A playback request counts as "good" if BOTH are true:
  1. The API returned HTTP 200 (song stream started successfully)
  2. Time-to-first-audio-byte was <= 2000ms

Measured over: POST /api/v1/playback/start
Excluded: requests where the client cancelled before the server responded
          (user closed the app mid-request — not a service failure)
```

> **📖 Why This SLI Is a Ratio, Not a Raw Count**
>
> A raw count of failures ("14 failed requests last hour") is meaningless without context — is that 14 out of 100 requests or 14 out of 10 million? Expressing the SLI as a ratio (good events / total valid events) makes it comparable across traffic levels: 99.9% means the same thing whether StreamKarma is serving 1,000 requests a minute during a quiet Tuesday morning or 500,000 requests a minute during a Friday evening peak. This ratio form is also exactly what PromQL is built to calculate efficiently.

### 3. Phase 3 — Instrument the Service and Set the SLO

**Business Problem:** The SLI definition above is useless until the playback service actually emits metrics that let you calculate it. And the SLO target itself needs to be a real number the team agrees to — not an arbitrary round number like "99.99%" picked because it sounds impressive.

#### 3.1 Instrumenting the Playback Service with prometheus_client

```python
# playback_service/metrics.py
from prometheus_client import Counter, Histogram

playback_requests_total = Counter(
    "playback_requests_total",
    "Total playback start requests",
    ["status_code"],
)

playback_time_to_first_byte_seconds = Histogram(
    "playback_time_to_first_byte_seconds",
    "Time from request received to first audio byte streamed",
    buckets=[0.1, 0.25, 0.5, 1.0, 2.0, 3.0, 5.0, 10.0],
)

# In the request handler
def start_playback(request):
    with playback_time_to_first_byte_seconds.time():
        response = stream_song(request.song_id)
    playback_requests_total.labels(status_code=response.status_code).inc()
    return response
```

> **📖 Counter and Histogram — Why Both**
>
> `playback_requests_total` is a **Counter** labeled by `status_code` — it only ever goes up, and Prometheus calculates rates from the increase over time. This gives you the "correctness" half of the SLI. `playback_time_to_first_byte_seconds` is a **Histogram** — it buckets every request's duration into predefined ranges (`0.1s`, `0.25s`, `0.5s`, ... up to `10.0s`). The bucket boundary at `2.0` is deliberate: it exactly matches the 2000ms threshold in the SLI definition, so PromQL can later ask "what fraction of requests fell in the `le=2.0` bucket or below" directly. Pick histogram buckets around your actual SLO threshold, not arbitrary round numbers — a bucket boundary that doesn't align with your SLO threshold makes the SLI impossible to calculate precisely.

#### 3.2 Scraping the Metrics

```yaml
# prometheus.yml
scrape_configs:
  - job_name: "playback-service"
    scrape_interval: 15s
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ["streamkarma-prod"]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: playback-service
        action: keep
```

> Prometheus discovers all pods labeled `app: playback-service` in the `streamkarma-prod` namespace automatically via Kubernetes service discovery, and scrapes their `/metrics` endpoint every 15 seconds — no manually maintained list of IPs to scrape, which would break every time a pod restarted.

#### 3.3 Setting the SLO Target

```
StreamKarma Playback SLO
===========================
SLO:    99.9% of playback requests are "good" (200 + <=2s TTFB)
Window: rolling 30 days
Error Budget: 0.1% of requests over 30 days may fail

Why 99.9% and not 99.99%:
  - User research showed churn risk increases sharply past
    ~0.1% of sessions having a bad playback experience — 99.9%
    already sits comfortably below that pain threshold.
  - 99.99% would require re-architecting playback with active-active
    multi-region failover — estimated 4 months of engineering time
    the business does not currently need to spend.
  - 99.9% still allows roughly 43 minutes of full-equivalent downtime
    per 30-day window, which is enough room for planned maintenance
    and the occasional real incident without breaching the SLO.
```

> **📖 SLOs Are a Business Decision, Not Just an Engineering One**
>
> Kavya insisted the 99.9% number come from actual user research (churn correlation) and actual cost trade-off (what 99.99% would require), not from picking a number that "feels enterprise-grade." A stricter SLO than necessary wastes engineering time chasing reliability nobody notices; a looser SLO than necessary lets real user pain go unaddressed. The right SLO sits right at the point where users start to notice and care — no tighter, no looser.

> **Key takeaways**
> - An SLI must reflect actual user experience — combine correctness and latency, not just HTTP status codes alone.
> - Histogram bucket boundaries should be chosen to align exactly with your SLO threshold, or you can't calculate the SLI precisely from the metric.
> - SLO targets come from business data (churn thresholds, cost of the next nine) — not from picking an impressive-sounding round number.
> - 99.9% "three nines" allows about 43 minutes of full-equivalent downtime per 30 days — know this number cold, because it's the concrete budget everything else in this project is built on.

### 4. Phase 4 — Calculate the Error Budget

**Business Problem:** "99.9% SLO" is abstract. The team needs it converted into a concrete number they can track daily: how many bad requests, or how many minutes of full-equivalent downtime, are still available before StreamKarma breaches its promise this month.

#### 4.1 The Error Budget Formula

```
Error Budget = (1 - SLO) × Total Valid Events (over the SLO window)

Example, StreamKarma playback, last 30 days:
  Total valid playback requests: 620,000,000
  SLO: 99.9%  →  allowed failure rate: 0.1%

  Error Budget = 0.001 × 620,000,000
               = 620,000 requests allowed to be "bad"
               over the 30-day window
```

#### 4.2 PromQL — Current Error Budget Remaining

```promql
# Total valid playback requests over the last 30 days
sum(increase(playback_requests_total[30d]))

# "Bad" requests: non-200 responses over the last 30 days
sum(increase(playback_requests_total{status_code!="200"}[30d]))

# Current SLI over the 30-day window
1 - (
  sum(increase(playback_requests_total{status_code!="200"}[30d]))
  /
  sum(increase(playback_requests_total[30d]))
)

# Error budget remaining, as a fraction of the total budget
1 - (
  (
    sum(increase(playback_requests_total{status_code!="200"}[30d]))
    /
    sum(increase(playback_requests_total[30d]))
  )
  /
  0.001
)
```

> **📖 Reading the Error Budget Remaining Query**
>
> The innermost fraction calculates the *actual* current failure rate over 30 days. Dividing that by `0.001` (the allowed failure rate at 99.9% SLO) tells you what fraction of the *entire budget* has been consumed — if the actual failure rate is exactly 0.001, you've spent 100% of the budget and have zero left. Subtracting from 1 flips that into "remaining budget": a result of `0.4` means 40% of this month's error budget is still available; a result at or below `0` means the SLO has already been breached for this rolling window.

**Quiz: StreamKarma's error budget remaining query returns 0.15 (15% left) on day 22 of a 30-day rolling window. What does this tell the team?**
- Everything is fine — 15% is a passing grade
- 15% of the allowed failures for the month are still unspent, but 8 days remain in the window — the team needs to assess whether the current burn rate would exhaust the remaining 15% before the window resets, and slow down risky deploys if so
- The service has been down for 15% of the month
- The SLO should immediately be lowered to make the number look better

> **Answer/explanation:** The second option reflects how error budgets are actually used operationally. 15% remaining is not automatically "fine" or "bad" in isolation — it depends entirely on the *rate* at which that remaining 15% is being consumed relative to the days left in the window. If the team is burning budget steadily and 15% remaining with 8 days left roughly matches the expected pace, no action is needed. But if recent burn rate suggests that 15% will be exhausted in 2 days rather than 8, the team should treat this as an early warning — freeze risky deploys, prioritize the current degradation, and protect the remaining budget. Treating 15% as simply "passing" ignores the trend entirely, which is exactly the blind spot multi-window burn-rate alerting (Phase 5) is designed to catch.

### 5. Phase 5 — Multi-Window, Multi-Burn-Rate Alerting

**Business Problem:** A single alert like "fire if error rate > 0.1% for 5 minutes" is too noisy for brief blips and too slow for a severe, sudden outage — and a single "fire if error budget is >50% consumed this month" alert would only catch slow, chronic problems days after the fact, long after a fast severe outage would have already burned the whole month's budget. StreamKarma needs both a fast detector and a slow detector, simultaneously.

**Scene 3 — Postmortem for the Database Connection Leak**

> **Kavya** _Engineering Director_
>
> Walk me through why nothing paged during that six-hour degradation.

> **Arjun** _Senior SRE_
>
> Our only alert was "error rate over 5% for 5 minutes." The connection leak degraded us to about 2% error rate — bad, and sustained for six hours, but never crossed that 5% threshold in any single 5-minute window. A slow, chronic problem completely evaded a fast-only alert. What we need is a burn-rate alert that also looks at longer windows, so a smaller but sustained error rate still trips it.

#### 5.1 The Multi-Burn-Rate Alerting Rules

```yaml
# prometheus-rules/playback-slo-alerts.yaml
groups:
  - name: playback-slo-burn-rate
    rules:
      # Fast burn: catches severe, short outages (e.g. bad deploy)
      # 14.4x burn rate sustained for 5m AND 1h -> would exhaust
      # the entire 30-day budget in about 2 hours if it continued
      - alert: PlaybackSLOFastBurn
        expr: |
          (
            sum(rate(playback_requests_total{status_code!="200"}[5m]))
            /
            sum(rate(playback_requests_total[5m]))
          ) > (14.4 * 0.001)
          and
          (
            sum(rate(playback_requests_total{status_code!="200"}[1h]))
            /
            sum(rate(playback_requests_total[1h]))
          ) > (14.4 * 0.001)
        for: 2m
        labels:
          severity: page
        annotations:
          summary: "Playback SLO fast burn — page on-call immediately"

      # Slow burn: catches chronic, low-grade degradations
      # 3x burn rate sustained for 30m AND 6h -> would exhaust the
      # entire 30-day budget in about 10 days if it continued
      - alert: PlaybackSLOSlowBurn
        expr: |
          (
            sum(rate(playback_requests_total{status_code!="200"}[30m]))
            /
            sum(rate(playback_requests_total[30m]))
          ) > (3 * 0.001)
          and
          (
            sum(rate(playback_requests_total{status_code!="200"}[6h]))
            /
            sum(rate(playback_requests_total[6h]))
          ) > (3 * 0.001)
        for: 15m
        labels:
          severity: ticket
        annotations:
          summary: "Playback SLO slow burn — file a ticket, review within 1 business day"
```

> **📖 Why Two Time Windows Per Alert, and Why These Burn-Rate Multipliers**
>
> Each alert checks **two windows at once** (5m and 1h for fast burn; 30m and 6h for slow burn) joined with `and`. The short window makes the alert responsive — it can fire within minutes of a real problem starting. The long window prevents false positives from a brief, self-resolving blip — a 2-minute spike that recovers won't hold true across the full 1-hour window too, so it won't page anyone. `14.4 * 0.001` and `3 * 0.001` are burn-rate multipliers relative to the SLO's allowed failure rate: a 14.4x burn rate would exhaust an entire 30-day error budget in about 2 hours if sustained — clearly page-worthy. A 3x burn rate would exhaust the same budget in about 10 days — not an emergency, but worth a ticket and a review well before the month is over. This exact multi-window multi-burn-rate approach is the same pattern documented in Google's SRE workbook, and it's what would have caught the six-hour connection leak: 2% sustained error rate is a 20x burn rate, well above the slow-burn threshold, so `PlaybackSLOSlowBurn` would have fired within 30-45 minutes instead of staying silent for six hours.

**Single-Threshold Alerting vs Multi-Window Multi-Burn-Rate Alerting**

- **Single-threshold alert** (e.g. "error rate > 5% for 5 minutes") — simple to write, but structurally blind to slow chronic degradations that never cross the threshold in any single short window, and can be noisy for brief self-resolving blips if the window is too short.
- **Multi-window multi-burn-rate alert** — requires more PromQL, but gives you two properly-tuned detectors: a fast one for severe outages that pages immediately, and a slow one for chronic degradations that would otherwise silently burn the whole month's budget before anyone notices. This is the standard SRE approach precisely because it closes both blind spots at once.

### 6. Phase 6 — The Error Budget Policy

**Business Problem:** An error budget dashboard that nobody actually uses to make decisions is just decoration. StreamKarma needs a written policy for what actually happens when the budget runs low — otherwise "we're at 90% budget consumed" stays an interesting number with zero organizational teeth.

#### 6.1 StreamKarma's Error Budget Policy

```
StreamKarma Playback Error Budget Policy
===========================================
Budget remaining > 50%:
  Normal operations. Feature launches and risky deploys proceed
  as planned.

Budget remaining 10-50%:
  Engineering lead reviews upcoming risky changes (major schema
  migrations, new caching layers) with the on-call SRE before
  merging. Non-critical feature launches may proceed.

Budget remaining < 10%:
  Feature freeze on the playback service. Only reliability work,
  bug fixes, and rollbacks are permitted until budget recovers
  above 25% OR the 30-day window rolls forward enough to
  naturally replenish budget.

Budget fully exhausted (SLO breached):
  Mandatory postmortem. All playback-service engineering capacity
  redirects to reliability work until the team has a concrete plan
  reviewed by Kavya. This is treated with the same seriousness as
  a customer-facing incident, because for our biggest B2B partners,
  it is one.
```

> **📖 The Error Budget as a Shared Currency Between Product and Engineering**
>
> This policy is what actually makes SLOs valuable beyond a dashboard: it gives product managers and engineers a shared, objective currency to negotiate with, instead of an endless subjective argument about whether it's "safe" to ship a risky feature this week. If the budget is healthy, product ships fast, and that's the correct default — reliability work has a real opportunity cost too, and StreamKarma doesn't want engineers gold-plating reliability nobody needs. If the budget is nearly gone, the freeze isn't a punishment — it's the pre-agreed consequence of the numbers, decided calmly in advance rather than argued about in the middle of an incident.

**Quiz: StreamKarma's playback error budget is at 8% remaining with 12 days left in the 30-day window. A product manager wants to ship a new "crossfade between songs" feature that touches the playback pipeline. What does the error budget policy say should happen?**
- Ship it — new features are always more important than reliability metrics
- The playback service is in feature freeze (budget below 10%) — only reliability work, bug fixes, and rollbacks are permitted until the budget recovers above 25% or the window rolls forward
- Lower the SLO target so the budget looks healthier
- Ship it, but only during off-peak hours

> **Answer/explanation:** The second option is exactly what the written policy in section 6.1 specifies, and it's the entire point of having a written policy agreed in advance: at 8% remaining, the playback service is below the 10% threshold, triggering a feature freeze until budget recovers above 25% or the rolling window naturally replenishes it. This isn't an arbitrary engineering veto — it's a pre-negotiated, objective rule that both product and engineering agreed to before this specific pressure situation arose, which is precisely what prevents the "ship it anyway, we'll deal with the fallout" dynamic that error budgets are designed to eliminate. Lowering the SLO to make the number look better defeats the entire purpose of having an SLO tied to real user churn data in the first place.

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Define a second SLI:** Playback isn't the only critical flow — define an SLI and SLO for StreamKarma's search endpoint (e.g. "search results returned within 500ms with a non-empty result set for a valid query"), and justify the SLO target with a one-paragraph business rationale like the one in section 3.3.
2. **Build the Grafana burn-down dashboard:** Create a Grafana panel showing error budget remaining as a burn-down line over the current 30-day window, with a red threshold line at 10% and a yellow line at 50%, matching the policy tiers in Phase 6.
3. **Simulate the fast-burn and slow-burn scenarios separately:** Using a load-testing tool, inject a short severe failure spike (should trip `PlaybackSLOFastBurn` but not `PlaybackSLOSlowBurn`) and separately a long low-grade failure rate (should trip the slow-burn alert only). Confirm both alerts behave exactly as designed.
4. **Write the alert routing in Alertmanager:** Route `severity: page` alerts to PagerDuty with immediate notification, and `severity: ticket` alerts to a Jira integration that opens a ticket without paging anyone, matching the two different urgency levels defined in Phase 5.
5. **Run a real error budget review meeting:** Using 30 days of real (or simulated) burn-rate data, write a one-page "error budget review" doc in the style product/engineering leadership would actually read, recommending whether the team should focus the next sprint on feature work or reliability work, backed by the numbers.

### SRE Fundamentals Project Complete 🎉

You defined StreamKarma's first real SLI for the playback service, backed it with an SLO grounded in actual churn data rather than a guess, instrumented the service to expose the metric in Prometheus, calculated a concrete error budget, and built multi-window multi-burn-rate alerts that catch both sudden severe outages and slow chronic degradations — the exact blind spot that let a six-hour incident go completely unnoticed before this project.

> **Kavya**
>
> "The error budget policy has already saved us from two arguments this quarter — both times, instead of a heated debate about whether shipping was 'safe,' someone just pulled up the dashboard and said 'we're at 60%, we're fine' or 'we're at 6%, this waits.' That's the real win here — not the dashboard itself, but that it ended arguments instead of starting them."

> **Arjun**
>
> "Remember the six-hour connection leak that started this whole project? I backtested our new slow-burn alert against that incident's historical data. It would have fired at the 40-minute mark. Not eliminated — SRE doesn't promise zero incidents — but caught five hours and twenty minutes earlier than it actually was."

> **Divya (You)**
>
> "The thing that clicked for me was that an SLI isn't just 'a metric' — it has to be defined precisely enough that two different engineers calculate the exact same number from the exact same data. Vague metrics produce vague alerts. Precise SLIs produce alerts people actually trust."

> **Next: Incident Response, On-Call, and Postmortem Culture**

> - Structured incident response — severity levels, incident commander role, and clear communication cadence during an active outage
> - Blameless postmortems — writing a postmortem that focuses on systemic fixes, not individual blame, so people report problems honestly
> - On-call rotation design — balancing page load, escalation policies, and reducing alert fatigue across a growing SRE team
> - Toil reduction — identifying repetitive manual operational work and systematically automating it away
> - Capacity planning and load testing — using historical SLI data to forecast infrastructure needs before the next big traffic spike
