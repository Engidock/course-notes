# AWS Networking — Hands-On Project Lab

> Design a custom VPC with public/private subnets and secure routing.

## Objective

Build a production-style VPC from scratch and prove traffic flows exactly where it should.

## Prerequisites

- AWS free-tier account
- Basic understanding of IP/CIDR

## Steps

1. Create a VPC with a /16 CIDR block.
2. Add two public and two private subnets across two availability zones.
3. Attach an Internet Gateway and route public subnet traffic through it.
4. Deploy a NAT Gateway in a public subnet; route private subnet egress through it.
5. Launch an EC2 instance in a private subnet and confirm it can reach the internet but isn't reachable from it.
6. Add a security group and NACL that only allow SSH from a bastion host in the public subnet.

## Deliverable

Architecture diagram + a terminal session proving the private instance has outbound internet but no inbound access.

## Stretch goals

- Add VPC Peering or a Transit Gateway to connect a second VPC.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
