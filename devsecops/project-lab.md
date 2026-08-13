# DevSecOps — Hands-On Project Lab

> Integrate security scanning into a CI pipeline.

## Objective

Make a pipeline fail the build automatically when it introduces a real vulnerability.

## Prerequisites

- A CI system (GitHub Actions/Jenkins)
- A sample app with a dependency file

## Steps

1. Add a SAST tool (e.g. Semgrep) as a pipeline stage that scans on every push.
2. Add a dependency/SCA scanner (e.g. `npm audit`, `pip-audit`, or Trivy) for known CVEs.
3. Add a secrets scanner (e.g. gitleaks) that scans the diff for committed credentials.
4. Set the pipeline to fail the build on any Critical or High finding.
5. Deliberately introduce one known-vulnerable dependency and confirm the pipeline blocks it.
6. Fix the vulnerability and confirm the pipeline goes green.

## Deliverable

Two pipeline runs: one red (blocked by a real finding) and one green (after the fix), both linked in your PR history.

## Stretch goals

- Add container image scanning (Trivy) as a stage before the image is allowed to push to a registry.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
