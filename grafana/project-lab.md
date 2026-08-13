# 📊 Grafana Project Mastery

> **Hey Fresher — Read This First!**
>
> Grafana is the dashboard layer that sits on top of your metrics, logs, and traces — it doesn't collect data itself, it queries other systems (like Prometheus, Loki, or CloudWatch) and turns numbers into panels, graphs, and alerts that a human can actually act on. If Prometheus is the nervous system feeling every twitch in your infrastructure, Grafana is the eyes that let engineers, on-call responders, and even non-technical stakeholders see what's happening at a glance.
>
> You've just joined **WaveReel Streaming**, a video-on-demand platform serving originals and live sports to about 6 million subscribers across South Asia. Your first week, the on-call SRE got paged at 2 AM because "video playback felt slow" — except nobody could say *how* slow, *for whom*, or *since when*, because there were no dashboards, only raw Prometheus queries typed from memory. Your manager's exact words: "We ship metrics into Prometheus just fine. We're blind on the visualization side. Fix that."

#### What You Will Learn and Build in This Project

You will build a full Grafana observability layer for WaveReel's streaming platform — starting from connecting a data source, through building templated dashboards with variables, to writing alert rules that page the right team through the right channel. By the end, you'll have a provisioned, version-controlled dashboard-as-code setup that mirrors what real streaming and SaaS companies run in production.

Data sources, dashboard panels, template variables, transformations, heatmaps and histograms, Grafana Alerting, contact points and notification policies, dashboard provisioning as code, SLO dashboards, annotations

> **📦 Phase 1 — Connect the Data and Build Your First Dashboard**
>
> Wire Grafana to WaveReel's Prometheus instance and build the first real panel: video start failures.

> **📦 Phase 2 — Template Variables and Reusable Panels**
>
> Turn one hardcoded per-region dashboard into a single dashboard that works for every region and every CDN edge.

> **📦 Phase 3 — Advanced Visualizations**
>
> Move beyond line graphs — heatmaps for rebuffering latency, histograms for bitrate distribution, and transformations to reshape query results.

> **📦 Phase 4 — Alerting That Doesn't Cry Wolf**
>
> Build Grafana-managed alert rules, contact points, and notification policies so the right team is paged, on the right channel, without alert fatigue.

> **📦 Phase 5 — Dashboards as Code**
>
> Provision dashboards and alert rules from YAML/JSON checked into Git instead of clicking around the UI.

> **📦 Phase 6 — SLO Dashboards for the Business**
>
> Build an executive-facing dashboard around WaveReel's playback SLO so leadership stops asking engineers for screenshots.

**Scene 1 — WaveReel Streaming, Bengaluru | "We have the data. We just can't see it."**

> **Meera** _Senior SRE, Platform Reliability_
>
> We push every service's metrics into Prometheus — request rates, error rates, transcoding queue depth, CDN cache hit ratio. All of it. But when the Chennai region had a 12-minute playback outage last month, the fastest way anyone found out was a spike in support tickets, not our monitoring. That's backwards.

> **Rohit** _Platform Architect_
>
> Right, we have the exhaust data, we don't have the windshield. Your job this sprint is to give every on-call engineer a dashboard where the top row tells them, in five seconds, "is playback healthy right now, yes or no." Everything else is drill-down.

> **You**
>
> So this isn't really about pretty graphs — it's about cutting time-to-detect.

> **Meera**
>
> Exactly. A dashboard nobody understands in five seconds is a dashboard nobody looks at during an incident. Let's start with the data source connection, then build up.

### 1. Phase 1 — Connect the Data and Build Your First Dashboard

**Business Problem:** WaveReel's backend teams have been running `promql` queries by hand in the Prometheus UI whenever something feels wrong. There is no shared, persistent view of system health, so every incident starts with someone re-deriving the same five queries from memory. You need Grafana talking to Prometheus and a first dashboard live before your next on-call rotation.

#### 1.1 Adding the Prometheus Data Source

```yaml
# provisioning/datasources/prometheus.yaml
apiVersion: 1

datasources:
  - name: Prometheus-WaveReel
    type: prometheus
    access: proxy
    url: http://prometheus.monitoring.svc.cluster.local:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      timeInterval: 15s
      prometheusType: Prometheus
      prometheusVersion: 2.51.0
    editable: false
```

> **📖 What this file does**
>
> `type: prometheus` tells Grafana which query engine and UI to use for this source. `access: proxy` means Grafana's backend server makes the request to Prometheus (browser access mode would make the user's browser call Prometheus directly, which rarely works past a firewall). `httpMethod: POST` avoids URL-length limits on large PromQL queries with many label matchers. `timeInterval: 15s` tells Grafana the minimum step between data points — matching Prometheus's own `scrape_interval` avoids the dashboard rendering fake "flat" gaps between real scrapes. `editable: false` locks the data source from being changed in the UI, since it's managed by this provisioning file — anyone who wants to change it has to do it through a pull request, not a click.

#### 1.2 Your First Panel — Video Start Failures

```promql
sum(rate(wavereel_playback_start_errors_total[5m]))
/
sum(rate(wavereel_playback_start_attempts_total[5m]))
```

**Panel settings that matter:**

- **Visualization:** Time series
- **Unit:** Percent (0.0-1.0)
- **Thresholds:** Green below 0.01, Yellow 0.01–0.03, Red above 0.03

> **📖 Reading this query**
>
> `wavereel_playback_start_errors_total` and `wavereel_playback_start_attempts_total` are counters incremented by the playback service every time a user tries to start a video and every time that attempt fails. `rate(...[5m])` converts each counter into a per-second rate averaged over the last 5 minutes — counters only ever go up, so you can't graph them directly and expect anything meaningful; you graph their rate of increase. Dividing the two rates gives a ratio that's independent of traffic volume, so it reads the same whether it's 2 AM with 50k concurrent viewers or 8 PM primetime with 2 million. Setting the panel unit to "Percent (0.0-1.0)" tells Grafana the raw number is already a fraction and it should multiply by 100 for display, rather than treating `0.03` as literally 0.03%.

**Quiz: Why divide two `rate()` expressions instead of just graphing `wavereel_playback_start_errors_total` directly?**
- Because Grafana can't render counters at all
- Because a raw counter only tells you a cumulative total since the process started, not a comparable rate, and dividing by total attempts normalizes for changing traffic volume
- Because PromQL requires all panels to use division

> **Answer/explanation:** The second option is correct. A raw counter like `wavereel_playback_start_errors_total` just keeps climbing forever (until the process restarts and it resets to zero) — graphing it directly gives you a line that goes up and to the right no matter what, which tells you nothing about whether errors are getting worse or better relative to traffic. `rate()` converts the counter into "errors per second," and dividing errors-per-second by attempts-per-second gives a traffic-independent error ratio, which is the number that actually matters for an SLO. Grafana renders counters fine (option 1 is false), and PromQL doesn't require division anywhere (option 3 is false) — it's a modeling choice, not a syntax requirement.

### 2. Phase 2 — Template Variables and Reusable Panels

**Business Problem:** WaveReel operates in five regions (Mumbai, Chennai, Delhi, Bengaluru, Hyderabad edge PoPs) and a hardcoded "Mumbai playback health" dashboard means four more copy-pasted dashboards that all drift out of sync the moment someone tweaks one panel. Roshan from the CDN team wants a single dashboard any engineer can point at any region.

**Scene 2 — Standup, Week 2 | "Stop copy-pasting dashboards"**

> **Roshan** _CDN & Edge Engineer_
>
> I found four versions of the same "Playback Health" dashboard in our folder, all slightly different because someone always forgets to update one when they add a panel. Can we get this down to one dashboard that just... changes based on which region you pick?

> **You**
>
> That's exactly what template variables are for. Give me an hour.

#### 2.1 Defining the Region Variable

```json
{
  "name": "region",
  "type": "query",
  "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
  "query": "label_values(wavereel_playback_start_attempts_total, region)",
  "refresh": 2,
  "multi": true,
  "includeAll": true,
  "sort": 1
}
```

> **📖 What each field controls**
>
> `query: label_values(...)` asks Prometheus for every distinct value of the `region` label seen on that metric — so the dropdown is always in sync with what regions actually exist, no manual list to maintain. `refresh: 2` means "reload the variable's options every time the dashboard time range changes," not just on page load, which matters when a new region gets added mid-quarter. `multi: true` plus `includeAll: true` lets an engineer pick one region, several, or "All" from the same dropdown. `sort: 1` sorts values alphabetically so the list is predictable instead of ordered by whatever Prometheus happened to return.

#### 2.2 Using the Variable in a Panel Query

```promql
sum by (region) (
  rate(wavereel_playback_start_errors_total{region=~"$region"}[5m])
)
```

**Hardcoded panel vs. variable-driven panel**

- **Hardcoded panel** (`region="mumbai"` baked into the query) — fine for a one-off investigation dashboard you'll delete in a day, but it rots the moment you need the same view for another region.
- **Variable-driven panel** (`region=~"$region"`) — the right default for anything that lives in a shared dashboards folder; one panel definition serves every region and every future region added to the fleet.

> **📖 Note the regex matcher**
>
> `region=~"$region"` uses `=~` (regex match) instead of `=` (exact match) because when `multi: true` is enabled and a user picks several regions, Grafana substitutes `$region` as a pipe-separated list like `mumbai|chennai`, which only works correctly against a regex operator, not an exact-match one.

> **Key takeaways**
> - Template variables turn N copy-pasted dashboards into 1 dashboard with a dropdown, which is the difference between dashboards someone maintains and dashboards that quietly go stale.
> - `label_values()` variables self-update as new label values appear in Prometheus — no manual list editing.
> - Multi-select variables need `=~` regex matching in the PromQL, not `=` exact matching.

### 3. Phase 3 — Advanced Visualizations

**Business Problem:** A single "average rebuffer time" line hides the real problem — most viewers are fine, but a meaningful tail is buffering for 8+ seconds on weak mobile networks, and averages smear that tail into invisibility. The Video QoE team needs to see the *distribution*, not just the mean.

#### 3.1 Heatmap for Rebuffering Duration

```promql
sum(increase(wavereel_rebuffer_duration_seconds_bucket[5m])) by (le)
```

**Panel settings:**

- **Visualization:** Heatmap
- **Format:** Time series buckets
- **Y-axis:** Data format = "Prometheus histogram," unit = seconds

> **📖 Why a heatmap here**
>
> `wavereel_rebuffer_duration_seconds_bucket` is a Prometheus histogram metric — under the hood it's actually several counters, one per bucket boundary (`le` label, meaning "less than or equal to"). `increase(...[5m])` gives you how many rebuffer events landed in each bucket over the last 5 minutes, and grouping `by (le)` keeps every bucket boundary separate instead of summing them into one number. Grafana's heatmap panel then renders each bucket as a colored cell — darker where more events landed — so you can see at a glance that most rebuffers are sub-second (healthy) while a thin but persistent band sits at 6-10 seconds (the tail worth chasing), something a single average line would completely hide.

#### 3.2 Histogram Quantiles for the Big Number Panel

```promql
histogram_quantile(0.95,
  sum(rate(wavereel_rebuffer_duration_seconds_bucket[5m])) by (le)
)
```

> **📖 Reading `histogram_quantile`**
>
> This computes the 95th percentile rebuffer duration — the value below which 95% of rebuffering events fall. `rate(..._bucket[5m])` gets the per-second rate of events falling into each bucket, `sum by (le)` merges the rates across all instances while keeping bucket boundaries intact, and `histogram_quantile(0.95, ...)` interpolates within those buckets to estimate the p95 value. This is the number WaveReel actually cares about for its SLO — "buffer time," not "average buffer time" — because a P95 that creeps from 2s to 5s is a real user-facing regression that an average can absorb without visibly moving.

**Counter vs. Gauge vs. Histogram — what to expose from your service**

- **Counter** — use for anything that only accumulates, like total playback attempts or total errors; always paired with `rate()` or `increase()` before graphing.
- **Gauge** — use for a value that goes up and down, like current concurrent streams or transcoding queue depth; graph it directly, no `rate()` needed.
- **Histogram** — use when you need percentiles or distribution shape, like rebuffer duration or startup latency; costs more cardinality than a counter or gauge, so reserve it for metrics where the shape of the data actually matters to a decision.

#### 3.3 Transformations — Reshaping Query Results Without a New Query

**Table panel showing top 5 slowest CDN edges**, built with a transformation instead of a more complex query:

1. Query: `topk(5, avg by (edge_pop) (wavereel_edge_response_seconds))`
2. Transformation: **Sort by** → Value, descending
3. Transformation: **Rename by regex** → `edge_pop` → `CDN Edge`

> **📖 Why use transformations instead of a fancier query**
>
> `topk(5, ...)` already does the heavy lifting inside PromQL — it returns only the 5 series with the highest values. The transformations after that are purely presentational: reordering rows and relabeling a column header so the table reads cleanly for a non-engineer glancing at it during an incident review. Keeping the presentation logic in Grafana transformations (rather than trying to force PromQL to also format output) keeps the query simple and debuggable on its own in the Prometheus UI.

### 4. Phase 4 — Alerting That Doesn't Cry Wolf

**Business Problem:** WaveReel's on-call rotation is drowning. Every panel got an alert bolted onto it during a rushed rollout last quarter, and half of them fire on normal traffic dips (like 4 AM, when almost nobody streams). Meera's ask: cut noisy alerts, keep the ones that matter, and route them to the right team instead of one shared pager.

#### 4.1 An Alert Rule That Reflects the Actual SLO

```yaml
# provisioning/alerting/playback-errors.yaml
apiVersion: 1

groups:
  - orgId: 1
    name: playback-slo
    folder: WaveReel Alerts
    interval: 1m
    rules:
      - uid: playback-error-rate-high
        title: Playback start error rate above SLO
        condition: C
        data:
          - refId: A
            relativeTimeRange: { from: 300, to: 0 }
            datasourceUid: prometheus-wavereel
            model:
              expr: >
                sum(rate(wavereel_playback_start_errors_total[5m]))
                /
                sum(rate(wavereel_playback_start_attempts_total[5m]))
              refId: A
          - refId: C
            datasourceUid: __expr__
            model:
              type: threshold
              expression: A
              conditions:
                - evaluator: { type: gt, params: [0.03] }
        for: 5m
        labels:
          severity: critical
          team: playback
        annotations:
          summary: "Playback start error rate is {{ $values.A }} (SLO threshold 3%)"
          runbook_url: "https://wiki.wavereel.internal/runbooks/playback-errors"
```

> **📖 Why `for: 5m` matters as much as the query**
>
> The `condition` block reuses the exact same error-rate expression from Phase 1 — that consistency matters, because an alert should page on the same number the dashboard shows, or on-call ends up staring at a green dashboard while getting paged. The `for: 5m` field is the noise filter: the condition has to stay true for five straight minutes before the alert actually fires, which absorbs the kind of brief 30-second blip that self-resolves and isn't worth waking anyone up for. `labels: { team: playback }` is what lets the notification policy in the next step route this specific alert to the playback team's Slack channel instead of a catch-all pager that every team ignores because 90% of the noise isn't theirs.

#### 4.2 Contact Points and Notification Policy

```yaml
# provisioning/alerting/contact-points.yaml
apiVersion: 1

contactPoints:
  - orgId: 1
    name: playback-team-slack
    receivers:
      - uid: playback-slack-receiver
        type: slack
        settings:
          url: "$SLACK_WEBHOOK_PLAYBACK"
          title: "{{ .CommonAnnotations.summary }}"
          text: "{{ .CommonAnnotations.runbook_url }}"

policies:
  - orgId: 1
    receiver: playback-team-slack
    group_by: ["alertname", "team"]
    matchers:
      - team = playback
    group_wait: 30s
    group_interval: 5m
    repeat_interval: 4h
```

> **📖 Routing logic explained**
>
> `matchers: team = playback` means this policy only handles alerts carrying the `team: playback` label — other teams' alerts route through their own policies to their own Slack channels. `group_by: ["alertname", "team"]` bundles multiple firing alerts that share those labels into a single Slack message instead of spamming one message per alert, which matters when a cascading failure trips five related rules at once. `group_wait: 30s` gives Grafana a short window to bundle near-simultaneous alerts before sending the first notification. `repeat_interval: 4h` stops the same still-firing alert from re-notifying every evaluation cycle — once every 4 hours is enough to remind on-call it's still broken without burying them.

**Quiz: An alert fires every night around 3-4 AM even though nothing is actually wrong. What's the most likely fix?**
- Delete the alert rule entirely
- The threshold is probably tuned for peak traffic and doesn't account for normal low-traffic variance; use a ratio-based query (like error rate, not raw error count) or a traffic-aware threshold
- Switch the panel visualization from time series to a table

> **Answer/explanation:** The second option is correct. Low-traffic periods often have naturally noisier ratios — if only 40 people are streaming at 3 AM and 3 of them hit an error, that's a 7.5% error rate on a statistically tiny sample, which can trip a static threshold that was reasonable during 2-million-concurrent primetime. The fix is either querying a ratio (which Phase 1 already does) with a `for:` duration long enough to smooth out small-sample noise, or building traffic-aware thresholds. Deleting the rule (option 1) throws away real signal for genuine incidents; changing the visualization type (option 3) is purely cosmetic and has zero effect on when the alert fires.

### 5. Phase 5 — Dashboards as Code

**Business Problem:** Every dashboard so far has been built by clicking in the Grafana UI. That's fine for prototyping, but WaveReel just failed an internal audit because nobody could say who changed the "Playback Health" dashboard's threshold from 3% to 8% last month, or why. Rohit wants dashboards in Git with a review trail, same as application code.

#### 5.1 Provisioning a Dashboard from JSON

```yaml
# provisioning/dashboards/dashboards.yaml
apiVersion: 1

providers:
  - name: wavereel-dashboards
    orgId: 1
    folder: WaveReel
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: false
    options:
      path: /var/lib/grafana/dashboards/wavereel
```

```bash
# Exporting a dashboard you built in the UI, to commit into Git
curl -s -H "Authorization: Bearer $GRAFANA_API_TOKEN" \
  "http://grafana.wavereel.internal/api/dashboards/uid/playback-health" \
  | jq '.dashboard' > dashboards/wavereel/playback-health.json

git add dashboards/wavereel/playback-health.json
git commit -m "dashboards: bump error-rate threshold to 3% per SLO revision"
```

> **📖 Why `allowUiUpdates: false` is the important line**
>
> Setting this to `false` means Grafana still *renders* the dashboard from the JSON file, but any edit made through the UI won't persist — it forces every real change through a pull request against the JSON in Git. That's what fixes the audit problem: `git log` on that file now shows exactly who changed the threshold and why, with a commit message and a reviewer, instead of a silent UI edit nobody can trace. `updateIntervalSeconds: 30` means Grafana polls the mounted folder every 30 seconds and picks up new commits automatically once your CI pipeline deploys the updated JSON, no manual re-import needed.

> **Key takeaways**
> - Dashboard JSON and alert rule YAML belong in version control, same as application code — it gives you diffs, review, and rollback.
> - `allowUiUpdates: false` is what actually enforces "changes go through Git," not just a suggestion.
> - Exporting via the HTTP API (`/api/dashboards/uid/...`) is how you migrate a UI-built dashboard into your provisioning pipeline the first time.

### 6. Phase 6 — SLO Dashboards for the Business

**Business Problem:** WaveReel's VP of Engineering keeps asking Meera for a "playback health number" in exec reviews, and Meera keeps sending screenshots of raw Prometheus graphs that nobody outside SRE can parse. You're building a single top-level dashboard row that answers "are we meeting our promise to customers" without any PromQL literacy required.

#### 6.1 SLO Budget Panel

```promql
1 - (
  sum(increase(wavereel_playback_start_errors_total[30d]))
  /
  sum(increase(wavereel_playback_start_attempts_total[30d]))
)
```

**Panel type:** Stat panel, with a gauge-style threshold bar underneath showing 99.5% as the SLO target line.

> **📖 Turning an engineering metric into a business number**
>
> This is the same error-rate math from Phase 1, just widened to a 30-day window and inverted (`1 - error_ratio`) to express it as a success rate — "99.7% of playback attempts succeeded this month" reads naturally to a VP in a way "error ratio 0.003" does not. Pairing it with a visible 99.5% threshold line turns the panel into an instant traffic-light read: green means WaveReel is within its promised reliability budget, red means it's time for an incident review, no PromQL required to interpret it.

#### 6.2 Annotating the Dashboard with Deploys

```bash
curl -s -X POST http://grafana.wavereel.internal/api/annotations \
  -H "Authorization: Bearer $GRAFANA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "dashboardUID": "playback-slo-exec",
    "time": '"$(date +%s000)"',
    "tags": ["deploy", "playback-service"],
    "text": "Deployed playback-service v2.14.0"
  }'
```

> **📖 Why annotations close the loop**
>
> Firing this from your CD pipeline right after every deploy draws a vertical marker directly on the dashboard timeline. When the error-rate line spikes ten minutes after a deploy marker, the correlation is visible without anyone cross-referencing a separate deploy log — this is often the single fastest way an SRE narrows an incident down to "it's the last release" versus "something upstream changed."

##### 🏋️ Hands-On Exercises — Extend the Project

1. Add a second data source for Loki (log aggregation) and build a panel that shows the last 20 error log lines filtered by the `$region` variable, linked from the error-rate panel via a data link.
2. Build a recording-rule-backed panel: define a Prometheus recording rule for the p95 rebuffer duration query from Phase 3, then repoint the panel at the pre-computed metric and compare dashboard load time before and after.
3. Create a second notification policy that pages the CDN team (not playback) when `wavereel_edge_response_seconds` p99 exceeds 800ms for 10 minutes, using its own contact point and Slack channel.
4. Add a "silence" runbook step: use the Grafana Alerting API to programmatically create a silence during a planned maintenance window on the Chennai edge, and verify no notification fires during that window.
5. Build a "Region Comparison" dashboard using the `repeat` panel feature so one row template automatically clones itself once per value of the `$region` variable.

**Quiz: You provisioned a dashboard with `allowUiUpdates: false`, but a teammate insists they can still edit panels in the browser. What's actually happening?**
- Grafana ignored the provisioning file
- They can still drag/resize panels and explore ad-hoc, but any attempt to click "Save" on the dashboard will be blocked or silently discarded, since the source of truth is the file, not the database
- `allowUiUpdates: false` only applies to alert rules, not dashboards

> **Answer/explanation:** The second option is correct. `allowUiUpdates: false` doesn't lock the UI into a read-only, non-interactive state — users can still explore, temporarily rearrange panels, and query ad hoc within a session. What it prevents is those changes persisting: saving is blocked because Grafana treats the provisioning file as the sole source of truth and will overwrite any unsaved database state on the next `updateIntervalSeconds` sync. Option 1 is wrong because the setting is very much being honored (that's why saves are blocked); option 3 is wrong because this particular setting is a dashboard-provisioning field, unrelated to alert rule provisioning.

### Grafana Project Complete 🎉

You took WaveReel Streaming from "we have metrics but nobody can see them" to a full observability layer: a Prometheus-backed dashboard with regional template variables, heatmaps and histograms that expose the tail latency averages were hiding, alert rules tuned against the real SLO with proper routing and noise suppression, dashboards and alerts checked into Git with an audit trail, and an executive-facing SLO dashboard annotated with every deploy.

> **Meera** _Senior SRE, Platform Reliability_
>
> Last week's Hyderabad edge degradation — we caught it from the dashboard eleven minutes before the first support ticket came in. Eleven minutes we didn't have before.

> **Rohit** _Platform Architect_
>
> And the exec review this month took four minutes instead of forty, because the VP could read the SLO panel herself instead of waiting for Meera to translate a PromQL graph.

> **You**
>
> Turns out "we already collect the data" and "we can see what's happening" are two completely different problems. Glad we finally closed that gap.

> **Next: Prometheus Project Mastery**
>
> - Go deeper into the PromQL and instrumentation choices that feed every panel you built here
> - Learn how recording rules and alerting rules live at the Prometheus layer, before Grafana ever sees them
> - Understand cardinality, retention, and federation — the constraints that decide what you're even allowed to graph
