# Podman — Hands-On Project Lab

> Run a rootless multi-container pod with Podman.

## Objective

Replicate a docker-compose style multi-container app using Podman pods, without a root daemon.

## Prerequisites

- Linux or WSL
- Basic container concepts

## Steps

1. Install Podman and confirm it runs rootless (`podman info` shows rootless: true).
2. Build a custom image from a Dockerfile for a small app (e.g. Flask/Express).
3. Create a Podman pod and add the app container plus a Redis or Postgres container to it.
4. Confirm the containers can reach each other over localhost inside the shared pod network namespace.
5. Generate a systemd unit file with `podman generate systemd` so the pod restarts on boot.
6. Export the pod definition with `podman kube generate` to a Kubernetes YAML file.

## Deliverable

The Kubernetes YAML generated from your running pod, plus the systemd unit file.

## Stretch goals

- Deploy the generated Kubernetes YAML to a real cluster and confirm it behaves the same.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
