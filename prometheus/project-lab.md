# Prometheus — Hands-On Project Lab

> Instrument an app, scrape it with Prometheus, and write alerting rules.

## Objective

Go past 'Prometheus is already scraping something' to actually instrumenting your own code.

## Prerequisites

- A sample app you can add a metrics endpoint to
- Prometheus

## Steps

1. Add a client library to the app and expose a `/metrics` endpoint.
2. Instrument a counter (requests total) and a histogram (request duration).
3. Configure `prometheus.yml` to scrape the app's `/metrics` endpoint on an interval.
4. Write a PromQL query for request rate (`rate(...[5m])`) and one for p95 latency using `histogram_quantile`.
5. Write an alerting rule that fires when p95 latency exceeds a threshold for 5 minutes.
6. Load-test the app briefly to intentionally trigger the alert and confirm it fires in the Alerts UI.

## Deliverable

Your two PromQL queries with their results, plus a screenshot of the fired alert.

## Stretch goals

- Add a `recording rule` that pre-computes the p95 latency query so dashboards load faster.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
