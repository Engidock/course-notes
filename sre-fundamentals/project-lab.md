# SRE Fundamentals — Hands-On Project Lab

> Define SLIs/SLOs for a sample service and alert on error budget burn.

## Objective

Move from 'is it up?' monitoring to SRE-style reliability targets that actually drive decisions.

## Prerequisites

- A sample service with metrics (or Prometheus + a demo app)
- Basic monitoring concepts

## Steps

1. Pick one service and define an SLI (e.g. successful request ratio).
2. Set an SLO (e.g. 99.5% success over 30 days) and calculate the resulting error budget.
3. Instrument the service to expose the SLI as a Prometheus metric.
4. Write a PromQL query that calculates current error budget burn rate.
5. Create a multi-window, multi-burn-rate alert (fast burn + slow burn) rather than a single threshold alert.
6. Simulate an outage and confirm the fast-burn alert fires well before the slow-burn one.

## Deliverable

The SLO document (SLI definition, target, error budget policy) plus screenshots of both alerts firing during your simulated outage.

## Stretch goals

- Build a Grafana error-budget burn-down dashboard for the service.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
