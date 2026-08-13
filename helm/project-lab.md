# Helm — Hands-On Project Lab

> Package a Kubernetes app as a configurable Helm chart.

## Objective

Turn a set of raw K8s manifests into a reusable, versioned, upgradeable chart.

## Prerequisites

- A Kubernetes cluster (kind/minikube)
- Existing app manifests (or use the Kubernetes lab's app)

## Steps

1. Run `helm create` to scaffold a chart and replace the boilerplate with your app's manifests.
2. Move hardcoded values (image tag, replica count, resource limits) into `values.yaml`.
3. Add a `values-prod.yaml` overriding replica count and resources for a production profile.
4. Install the chart with `helm install` and confirm the app comes up correctly.
5. Bump the image tag in `values.yaml` and run `helm upgrade`.
6. Deliberately break the upgrade (bad image tag) and use `helm rollback` to recover.

## Deliverable

A `helm history` output showing the install, the broken upgrade, and the successful rollback.

## Stretch goals

- Add a chart dependency (e.g. bundle a Redis subchart) via `Chart.yaml`.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
