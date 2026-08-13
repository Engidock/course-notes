# SonarQube — Hands-On Project Lab

> Set up SonarQube, scan a codebase, and get it to pass the quality gate.

## Objective

Turn static analysis into an actual gate your code has to clear, not just a report nobody reads.

## Prerequisites

- Docker (to run SonarQube)
- A codebase with some existing issues

## Steps

1. Run SonarQube via Docker and create a new project.
2. Run a scan with the SonarScanner CLI against your codebase.
3. Review the reported issues: bugs, vulnerabilities, code smells, and duplication.
4. Fix every issue tagged as a 'Blocker' or 'Critical'.
5. Re-scan and confirm the project passes the default Quality Gate.
6. Wire the scan into your CI pipeline so future PRs are scanned automatically.

## Deliverable

A before/after Quality Gate screenshot (failing → passing) plus the CI job that runs the scan automatically.

## Stretch goals

- Add a custom Quality Gate condition (e.g. 0 new Critical issues on every PR).

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
