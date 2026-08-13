# Cloud Formation — Hands-On Project Lab

> Provision a VPC, EC2 instance, and S3 bucket with a CloudFormation template.

## Objective

Learn AWS's native IaC tool and how it tracks and updates a stack over time.

## Prerequisites

- AWS account
- Basic YAML/JSON

## Steps

1. Write a template provisioning a VPC, one public subnet, and a security group.
2. Add an EC2 instance resource that uses the VPC and security group via `!Ref`.
3. Add an S3 bucket resource with versioning enabled.
4. Deploy the stack and confirm all resources exist and are correctly linked.
5. Modify one parameter (e.g. instance type) and update the stack, observing the changeset before applying.
6. Delete the stack and confirm CloudFormation cleans up every resource it created.

## Deliverable

A changeset screenshot showing exactly what would change before you applied the update.

## Stretch goals

- Convert the template to use Nested Stacks for the networking piece.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
