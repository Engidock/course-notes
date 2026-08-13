# Docker — Hands-On Project Lab

> Containerize a full app with a multi-stage Dockerfile and docker-compose.

## Objective

Take a real app with a database dependency and package it for reproducible local development.

## Prerequisites

- Docker Desktop or Docker Engine
- A sample app (any language)

## Steps

1. Write a multi-stage Dockerfile (build stage + slim runtime stage) for the app.
2. Confirm the final image is meaningfully smaller than a naive single-stage build.
3. Write a docker-compose.yml with the app service and a database service (Postgres/MySQL).
4. Use a named volume for database persistence and confirm data survives `docker compose down` (without `-v`).
5. Add a healthcheck to the app service and make the app wait on the DB healthcheck before starting.
6. Push the built image to Docker Hub or another registry with a version tag.

## Deliverable

The image size comparison (single-stage vs multi-stage) and a `docker compose up` that comes up healthy on the first try.

## Stretch goals

- Add a `.dockerignore` and re-measure build context size and build time.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
