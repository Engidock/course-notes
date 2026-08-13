# Shell Scripting — Hands-On Project Lab

> Automate a sysadmin task with a properly argument-parsed bash script.

## Objective

Write a bash script that handles bad input gracefully instead of doing something destructive.

## Prerequisites

- A Linux/macOS shell
- A task to automate, e.g. backing up a directory

## Steps

1. Write the core logic first (e.g. `tar` a directory into a timestamped archive).
2. Add flag parsing with `getopts` for options like `--source` and `--dest`.
3. Add `set -euo pipefail` at the top and fix whatever it now correctly complains about.
4. Validate inputs explicitly (does the source directory exist? is dest writable?) with clear error messages.
5. Add a log file that records every run with a timestamp and outcome.
6. Add a cron entry (documented, not necessarily installed) that would run this nightly.

## Deliverable

The script run against a real directory, producing a timestamped archive and a log entry, plus one run that correctly fails on bad input.

## Stretch goals

- Add a retention step that deletes backups older than 7 days.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
