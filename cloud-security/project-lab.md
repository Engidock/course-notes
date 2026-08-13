# Cloud Security — Hands-On Project Lab

> Harden an AWS account against the most common misconfigurations.

## Objective

Take a deliberately loose AWS account and lock it down using least-privilege and detection tooling.

## Prerequisites

- AWS account
- IAM basics

## Steps

1. Enable MFA on the root account and all IAM users.
2. Replace any wildcard (`*:*`) IAM policies with least-privilege scoped policies.
3. Enable CloudTrail across all regions with log file validation.
4. Enable GuardDuty and review its first findings.
5. Find and remediate one publicly-readable S3 bucket (Block Public Access).
6. Set up an AWS Config rule that flags future public buckets automatically.

## Deliverable

A before/after IAM policy diff and a GuardDuty findings screenshot showing zero unresolved high-severity issues.

## Stretch goals

- Add Security Hub and aggregate findings across GuardDuty, Config, and IAM Access Analyzer.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
