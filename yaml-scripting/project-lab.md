# YAML Scripting — Hands-On Project Lab

> Author a complex, valid multi-document YAML file with anchors and references.

## Objective

Get past copy-paste YAML and use the features that keep large configs DRY and correct.

## Prerequisites

- Any text editor
- A YAML linter/validator

## Steps

1. Write a Kubernetes-style multi-document YAML file (`---` separated) with 2 Deployments.
2. Use a YAML anchor (`&base`) to define shared labels/resources, and reference it (`<<: *base`) in both Deployments.
3. Add a nested structure with lists of maps (e.g. multiple env vars per container).
4. Introduce a deliberate indentation bug and use a linter (`yamllint`) to catch it.
5. Fix the bug and validate the file parses correctly with a YAML parser (e.g. `python -c 'import yaml...'`).
6. Convert the same structure to JSON and confirm the parsed data is identical.

## Deliverable

The valid YAML file, the yamllint output before and after the fix, and the equivalent JSON.

## Stretch goals

- Parameterize the file with Jinja2 templating (as Ansible would) and render it for two environments.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
