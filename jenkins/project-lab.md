# Jenkins — Hands-On Project Lab

> Build a declarative Jenkins pipeline with a webhook trigger.

## Objective

Automate build/test/deploy for a sample app entirely from a Jenkinsfile checked into the repo.

## Prerequisites

- A Jenkins instance (Docker is fine)
- A sample app repo

## Steps

1. Write a declarative `Jenkinsfile` with `build`, `test`, and `deploy` stages.
2. Configure a GitHub webhook so a push triggers the pipeline automatically.
3. Use Jenkins credentials (not plaintext) for any deployment secrets.
4. Add a `post` block that sends a notification (email/Slack) on failure.
5. Parameterize the pipeline so a manual run can target `dev` or `prod`.
6. Introduce a failing test on purpose and confirm the deploy stage is correctly skipped.

## Deliverable

A pipeline run history showing an automatic trigger from a push, plus one run that correctly stopped at a failed test.

## Stretch goals

- Move the shared notification logic into a Jenkins Shared Library.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
