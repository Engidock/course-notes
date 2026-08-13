# Spinnaker — Hands-On Project Lab

> Build a multi-stage deployment pipeline in Spinnaker.

## Objective

Promote an application through dev, staging, and prod with manual judgment gates.

## Prerequisites

- A running Spinnaker instance (Halyard or minispinnaker)
- A containerized sample app

## Steps

1. Register your image registry and a target deployment provider (Kubernetes) in Spinnaker.
2. Create an application and a pipeline with a 'Deploy' stage targeting the dev environment.
3. Add a 'Manual Judgment' stage gating promotion to staging.
4. Add an automated smoke-test stage that must pass before promoting to prod.
5. Trigger the pipeline from a new image tag being pushed to your registry.
6. Deliberately fail the smoke test once and confirm the pipeline halts before prod.

## Deliverable

A pipeline execution graph showing dev → staging → prod, plus one recorded run that correctly halted on a failed smoke test.

## Stretch goals

- Add an automated rollback stage triggered by a failed canary analysis.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
