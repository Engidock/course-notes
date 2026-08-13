# GIT — Hands-On Project Lab

> Practice a real branching workflow: feature branches, rebase, and conflict resolution.

## Objective

Build muscle memory for the Git operations that actually come up on a team, not just `add/commit/push`.

## Prerequisites

- Git installed
- A GitHub/GitLab account

## Steps

1. Create a repo and two feature branches off `main` that both touch the same file.
2. Merge the first branch normally.
3. Rebase the second branch onto the updated `main` and resolve the resulting conflict by hand.
4. Use `git bisect` to find which of five deliberately-seeded commits introduced a 'bug' (a failing script).
5. Create an annotated tag for a release and push tags to the remote.
6. Rewrite the last 3 commits on a scratch branch with interactive rebase to squash them into one clean commit.

## Deliverable

A clean git log showing the resolved rebase, the tag, and the squashed commit history — plus the commit hash `git bisect` identified as the culprit.

## Stretch goals

- Set up a pre-push hook that blocks pushes containing the word 'TODO'.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
