# Linux — Hands-On Project Lab

> Practice user management, permissions, systemd, and log troubleshooting.

## Objective

Get comfortable doing real sysadmin tasks on a Linux box without looking up every command.

## Prerequisites

- A Linux VM or container you can break safely
- Root/sudo access

## Steps

1. Create a new user and group, and add the user to the group.
2. Set up directory permissions so only that group can read/write a shared folder (chmod/chown, not 777).
3. Write a simple script and create a systemd service that runs it and restarts on failure.
4. Start the service, deliberately kill the process, and confirm systemd restarts it.
5. Introduce a fault that fills a log (e.g. a noisy loop) and use `journalctl` to find the exact time it started.
6. Set up a cron job as the new (non-root) user and confirm it runs with correct permissions.

## Deliverable

The systemd service surviving a manual kill, plus the `journalctl` command and output that pinpointed the log issue.

## Stretch goals

- Set resource limits on the systemd service (`MemoryMax`, `CPUQuota`) and prove they're enforced.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
