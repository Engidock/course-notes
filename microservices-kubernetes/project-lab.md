# Microservices & Kubernetes — Hands-On Project Lab

> Split a monolith into microservices and deploy them on Kubernetes.

## Objective

Practice service decomposition and inter-service communication under real orchestration constraints.

## Prerequisites

- Kubernetes cluster (kind/minikube)
- A simple monolith app to split

## Steps

1. Identify two bounded contexts in a monolith (e.g. 'orders' and 'users') and split them into separate services.
2. Give each service its own Dockerfile and Deployment/Service in Kubernetes.
3. Replace in-process function calls with HTTP (or gRPC) calls between services.
4. Add a shared ConfigMap for service discovery URLs (or use K8s DNS directly).
5. Add basic retry/timeout logic in the calling service for resilience.
6. Load-test both services independently and confirm you can scale just the bottleneck service.

## Deliverable

Two independently deployable, independently scalable services with a documented API contract between them.

## Stretch goals

- Add a service mesh (Linkerd or Istio) for automatic retries, mTLS, and observability.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
