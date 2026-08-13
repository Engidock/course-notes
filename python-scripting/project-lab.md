# Python Scripting — Hands-On Project Lab

> Automate a repetitive task with a robust, error-handled Python script.

## Objective

Write a script good enough that you'd actually trust it to run unattended.

## Prerequisites

- Python 3
- A repetitive task to automate (e.g. renaming/organizing files, parsing logs)

## Steps

1. Write a script that solves the task for the happy-path case only.
2. Add argument parsing (`argparse`) instead of hardcoded paths/values.
3. Add explicit error handling for the failure cases that will actually happen (missing file, permission denied, malformed input).
4. Add logging (not just `print`) with at least INFO and ERROR levels.
5. Add a `--dry-run` flag that shows what the script would do without doing it.
6. Write 2-3 unit tests covering both a success case and a failure case.

## Deliverable

The script run once with `--dry-run` and once for real, plus passing unit tests.

## Stretch goals

- Package the script as an installable CLI tool with `pip install -e .` and a console-script entry point.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
