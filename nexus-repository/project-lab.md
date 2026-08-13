# Nexus Repository — Hands-On Project Lab

> Stand up Nexus as a private artifact repository.

## Objective

Host and consume your own build artifacts instead of relying only on public registries.

## Prerequisites

- Docker (to run Nexus)
- A build tool (Maven or npm)

## Steps

1. Run Nexus 3 via Docker and complete initial admin setup.
2. Create a hosted Maven (or npm) repository.
3. Configure your build tool's settings file to publish to that repository.
4. Publish a versioned artifact from a sample project.
5. Create a proxy repository for the public registry (Maven Central / npmjs) and a group repository combining both.
6. Point your build tool at the group repository and confirm it resolves both your private artifact and a public one.

## Deliverable

A screenshot of your published artifact in the Nexus UI and a build log resolving dependencies through the group repo.

## Stretch goals

- Add a cleanup policy that deletes snapshot artifacts older than 30 days.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
