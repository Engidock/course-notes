# Ansible — Hands-On Project Lab

> Automate provisioning and configuration of multiple servers with a playbook.

## Objective

Configure a fleet of servers idempotently instead of by hand, one SSH session at a time.

## Prerequisites

- 2+ target servers or VMs
- SSH access configured

## Steps

1. Write an inventory file grouping servers by role (e.g. `web`, `db`).
2. Write a playbook that installs and configures nginx on the `web` group.
3. Use variables and templates (Jinja2) to generate the nginx config per environment.
4. Add a handler that reloads nginx only when the config file actually changes.
5. Run the playbook twice in a row and confirm the second run reports zero changes (idempotency).
6. Add a role structure (`roles/webserver/...`) instead of one flat playbook file.

## Deliverable

A playbook run log showing 0 changed on the second execution, plus the role directory structure.

## Stretch goals

- Add Ansible Vault to encrypt a secret variable (e.g. a DB password) used in the playbook.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
