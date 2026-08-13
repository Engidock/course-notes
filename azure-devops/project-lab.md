# Azure DevOps — Hands-On Project Lab

> Build a YAML pipeline for build and release to Azure App Service.

## Objective

Get a real app from commit to a live Azure URL using Azure Pipelines.

## Prerequisites

- Azure DevOps org
- Azure subscription

## Steps

1. Create a YAML pipeline (`azure-pipelines.yml`) with a build stage.
2. Add a service connection to your Azure subscription.
3. Add a release stage that deploys the build artifact to an Azure App Service.
4. Split the pipeline into 'dev' and 'prod' stages with an approval gate before prod.
5. Add variable groups for environment-specific configuration.
6. Trigger a deployment and verify the live app reflects the latest commit.

## Deliverable

A pipeline run showing both stages, with a screenshot of the approval gate and the deployed app URL.

## Stretch goals

- Add a rollback stage that redeploys the previous successful build artifact on failure.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
