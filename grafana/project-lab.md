# Grafana — Hands-On Project Lab

> Build a monitoring dashboard with alerts on a Prometheus data source.

## Objective

Go from raw metrics to a dashboard someone would actually leave open during an incident.

## Prerequisites

- Prometheus (or another data source) with some metrics flowing
- Grafana instance (Docker is fine)

## Steps

1. Add Prometheus as a data source in Grafana.
2. Build a dashboard with at least 3 panels: a time series, a gauge, and a table.
3. Use a dashboard variable (e.g. `$instance`) so one dashboard works across multiple hosts/services.
4. Add an alert rule on one panel (e.g. error rate > 5% for 5 minutes).
5. Configure a notification channel (email/Slack/webhook) for the alert.
6. Deliberately trigger the alert condition and confirm the notification fires.

## Deliverable

A screenshot of the dashboard plus the notification that fired when you triggered the alert condition.

## Stretch goals

- Export the dashboard as JSON and check it into the repo for version control.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
