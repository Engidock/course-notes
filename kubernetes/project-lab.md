# Kubernetes — Hands-On Project Lab

> Deploy a multi-tier application on Kubernetes with Ingress.

## Objective

Take a frontend, backend, and database and run them as a properly networked, configurable K8s app.

## Prerequisites

- kind or minikube
- kubectl basics

## Steps

1. Create Deployments for a frontend, backend API, and database (or use a managed DB stub).
2. Expose each with a ClusterIP Service; only the frontend gets external exposure.
3. Move all config (API URLs, feature flags) into a ConfigMap and secrets into a Secret.
4. Add resource requests/limits to every container.
5. Install an Ingress controller and add an Ingress resource routing `/` to frontend and `/api` to backend.
6. Add a liveness and readiness probe to each Deployment and kill a pod to confirm it self-heals.

## Deliverable

A single Ingress URL that serves the frontend and correctly proxies `/api` calls to the backend, surviving a manual pod deletion.

## Stretch goals

- Convert the manifests into a Helm chart with environment-specific values files.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
