# Terraform — Hands-On Project Lab

> Provision cloud infrastructure with reusable Terraform modules.

## Objective

Move from a flat single-file Terraform config to a proper, reusable module structure with remote state.

## Prerequisites

- Terraform CLI
- A cloud account (AWS/Azure/GCP)

## Steps

1. Write a root module that provisions a VPC, a compute instance, and a storage bucket.
2. Refactor the VPC and compute pieces into their own reusable child modules with input variables.
3. Move state from local to a remote backend (S3+DynamoDB, or Terraform Cloud).
4. Add a `terraform.tfvars` per environment (dev/prod) using the same modules with different inputs.
5. Run `terraform plan` and review it carefully before every apply.
6. Destroy the dev environment and confirm the module structure lets you recreate it identically.

## Deliverable

Two environments (dev/prod) built from the same modules, plus a plan output showing zero drift on a re-run.

## Stretch goals

- Add a pre-commit hook running `terraform fmt` and `tflint` automatically.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
