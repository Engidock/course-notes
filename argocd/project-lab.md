# ArgoCD — Hands-On Project Lab

> Deploy an app to Kubernetes via ArgoCD, synced from Git, with a rollback.

## Objective

Practice the GitOps loop: Git is the source of truth, not `kubectl apply` run by hand.

## Prerequisites

- A Kubernetes cluster
- ArgoCD installed
- A Git repo with K8s manifests

## Steps

1. Create an ArgoCD Application pointing at your Git repo and a manifests path.
2. Sync it and confirm the app is deployed and shows 'Healthy' and 'Synced' in the ArgoCD UI.
3. Change a manifest directly in the cluster with `kubectl edit` and watch ArgoCD detect and flag the drift.
4. Push a real change to Git (e.g. bump replica count) and confirm ArgoCD picks it up automatically.
5. Push a deliberately broken manifest (bad image tag) and watch the app go 'Degraded'.
6. Revert the bad commit in Git and confirm ArgoCD self-heals the app back to healthy.

## Deliverable

The ArgoCD Application history showing the drift detection, the broken deploy, and the git-revert recovery.

## Stretch goals

- Set up an App of Apps pattern to manage multiple ArgoCD Applications from one root app.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
