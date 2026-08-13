# Github Actions — Hands-On Project Lab

> Build a full CI/CD workflow: lint, test, build, and deploy.

## Objective

Automate the path from commit to deployment with proper gating at each stage.

## Prerequisites

- A GitHub repo
- A deployable sample app

## Steps

1. Add a workflow that runs on every push and pull request.
2. Add a lint job and a test job that run in parallel.
3. Add a build job that only runs after lint and test succeed, producing a build artifact.
4. Add a deploy job that only runs on pushes to `main` and only after build succeeds.
5. Use repository secrets for any deployment credentials — never hardcode them.
6. Add a required status check on the PR branch protection rule so merges are blocked on a red pipeline.

## Deliverable

A merged PR whose checks show lint → test → build all green, plus one PR you deliberately let fail to prove branch protection works.

## Stretch goals

- Add a matrix build testing against two runtime versions in parallel.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
