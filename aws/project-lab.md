# AWS — Hands-On Project Lab

> Deploy a 3-tier web application on AWS.

## Objective

Stand up a load-balanced, auto-scaling web tier backed by a managed database.

## Prerequisites

- AWS free-tier account
- Basic Linux

## Steps

1. Create a VPC with public and private subnets.
2. Launch an RDS instance (MySQL/Postgres) in a private subnet.
3. Create a launch template and an Auto Scaling Group for the app tier.
4. Put an Application Load Balancer in front of the ASG.
5. Configure health checks and scaling policies (target CPU 60%).
6. Point a Route 53 record at the ALB.

## Deliverable

A working URL that survives one EC2 instance being manually terminated (ASG replaces it automatically).

## Stretch goals

- Add a CloudFront distribution in front of the ALB with HTTPS via ACM.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
