# Pulumi — Hands-On Project Lab

> Provision the same style of infrastructure as Terraform/CloudFormation, using real code.

## Objective

Compare an imperative-language IaC tool against declarative templates for the same job.

## Prerequisites

- Pulumi CLI
- Python or TypeScript
- A cloud account

## Steps

1. Initialize a new Pulumi project in Python or TypeScript.
2. Write code provisioning a VPC/network, a compute instance, and a storage bucket.
3. Use a real language construct (a loop or function) to avoid repeating yourself across two environments.
4. Run `pulumi preview` and review the plan before applying.
5. Apply the stack, then change one input and re-apply, observing exactly what Pulumi updates in place vs replaces.
6. Destroy the stack and confirm all resources are removed.

## Deliverable

A short write-up comparing this experience to Terraform/CloudFormation — what real code made easier or harder.

## Stretch goals

- Add a unit test for your infrastructure code using Pulumi's testing framework.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
