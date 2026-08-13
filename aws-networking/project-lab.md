# 🌐 AWS Networking Project Mastery

> **👋 Hey Fresher — Read This First!**

> A VPC (Virtual Private Cloud) is your own private, isolated slice of AWS — a fenced-off network where you decide exactly which servers can talk to the internet, which stay hidden, and which can talk to each other. Get the networking wrong and either your database is exposed to the entire internet, or your application can't reach the payment gateway it needs. This document uses **short, focused code blocks** — each one covers exactly one networking concept — with a plain-English explanation right beside it. No 200-line templates to untangle. One idea at a time.

> **Company in this project:** Zyra Retail — a fast-growing e-commerce marketplace based in Bengaluru, selling everything from electronics to festival wear across India. Their entire platform — checkout API, product catalog, and order database — currently runs on a handful of EC2 instances thrown into AWS's **default VPC**, with no real separation between what's public and what's private. With the Diwali sale season six weeks away and last year's traffic spike having knocked their checkout database offline for 40 minutes, leadership has mandated a proper network redesign. You just joined as a Junior Cloud/Network Engineer. Your senior mentor is Arjun. Let's build Zyra a production-grade VPC from the ground up.

#### What You Will Learn and Build in This Project

You will design and build a multi-AZ VPC for Zyra Retail from scratch — subnets, routing, gateways, security layers, and connectivity to other networks — learning why each networking decision exists and what breaks in production without it.

VPC, CIDR blocks, Public and Private Subnets, Availability Zones, Internet Gateway, NAT Gateway, Route Tables, Security Groups, Network ACLs, VPC Peering, Transit Gateway, VPC Endpoints, Application Load Balancer

> **📦 Phase 1 — VPC Foundations**
>
> Design the CIDR block and carve it into public and private subnets spread across two Availability Zones.

> **📦 Phase 2 — Internet Gateway & Public Routing**
>
> Attach an Internet Gateway and configure route tables so only the intended subnets can reach the internet directly.

> **📦 Phase 3 — NAT Gateway & Private Egress**
>
> Give private subnets safe outbound-only internet access without exposing them to inbound traffic.

> **📦 Phase 4 — Security Groups & Network ACLs**
>
> Layer stateful and stateless firewalls so only the checkout API can reach the database, and nothing unwanted gets in.

> **📦 Phase 5 — Load Balancing Across AZs**
>
> Spread the checkout API across two Availability Zones behind an Application Load Balancer for real high availability.

> **📦 Phase 6 — VPC Peering, Transit Gateway & Endpoints**
>
> Connect Zyra's VPC to a partner's warehouse-management VPC, and reach S3/DynamoDB without touching the public internet.

**Scene 1 — Zyra Retail Engineering Office, Bengaluru | The Default VPC Problem**

> **Arjun** _Senior Cloud Architect — Zyra Retail_
>
> Kavya, right now everything — checkout API, product catalog, even our internal admin panel — sits in the same default VPC with public IPs on every instance. During last year's Diwali sale, someone's laptop got compromised, an attacker found our database's public IP through a simple port scan, and we spent the night rotating credentials. This year, nothing should be reachable from the internet unless it absolutely has to be.

> **Kavya (You)** _Junior Cloud/Network Engineer — Day 1 at Zyra Retail_
>
> I've used EC2 before, but always in the default VPC — I never had to think about subnets or routing, it just worked. Where do I even start designing a real one?

> **Arjun** _Senior Cloud Architect_
>
> Start by drawing a line: **public** means "the internet can reach it directly" — that's your load balancer and maybe a bastion host. **Private** means "only reachable from inside the VPC" — that's your checkout API and your database. Everything defaults to private unless there's a specific business reason to expose it. We'll build this in layers: the network shape first, then who can enter, then who can talk to whom.

> **Meera** _VP Engineering — Zyra Retail_
>
> And build it across two Availability Zones from day one, not one. An AZ is basically a separate physical data center. If we only deploy in one and that data center has a power issue during the Diwali sale, we're offline completely — this happened to a competitor last year and it was all over Twitter for the wrong reasons.

### 1. Phase 1 — VPC Foundations

**Business Problem:** Zyra's current default VPC gives every instance a public IP by default and provides no real segmentation between the checkout API, the database, and the internet. Before writing a single routing rule, the network needs a deliberate shape — one CIDR block, split into subnets that are each pinned to a specific Availability Zone and a specific purpose (public or private).

#### 1.1 Choose the VPC CIDR Block

```bash
aws ec2 create-vpc \
  --cidr-block 10.20.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=zyra-prod-vpc}]'
```

> **📖 Why 10.20.0.0/16?**
>
> A `/16` CIDR block gives Zyra 65,536 private IP addresses (10.20.0.0 through 10.20.255.255) — far more than needed today, but cheap to reserve and impossible to expand later without a painful migration. AWS reserves 5 IPs per subnet (network address, VPC router, DNS, future use, and broadcast), so plan subnets generously. Using the `10.x.x.x` private range (rather than `172.16.x.x` or `192.168.x.x`) is a convention that avoids collisions if Zyra later peers with a partner VPC or connects to an on-prem office network — always check the other side's range before picking your own.

#### 1.2 Carve Out Public and Private Subnets Across Two AZs

```bash
# Public subnets — one per AZ, for the load balancer and NAT Gateway
aws ec2 create-subnet --vpc-id vpc-0abc123 --cidr-block 10.20.0.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=zyra-public-1a}]'

aws ec2 create-subnet --vpc-id vpc-0abc123 --cidr-block 10.20.1.0/24 \
  --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=zyra-public-1b}]'

# Private subnets — one per AZ, for the checkout API and database
aws ec2 create-subnet --vpc-id vpc-0abc123 --cidr-block 10.20.10.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=zyra-private-1a}]'

aws ec2 create-subnet --vpc-id vpc-0abc123 --cidr-block 10.20.11.0/24 \
  --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=zyra-private-1b}]'
```

> **📖 One Subnet, One AZ, One Purpose**
>
> A subnet always lives entirely inside one Availability Zone — it cannot span two. That's why Zyra needs **four** subnets for a two-AZ design: public-1a, public-1b, private-1a, private-1b. Each `/24` gives 251 usable IPs. Splitting `10.20.0.0/24` and `10.20.1.0/24` for public, and `10.20.10.0/24` / `10.20.11.0/24` for private, leaves clean, memorable gaps for future subnet tiers (e.g. a `10.20.20.0/24` for a database-only tier later). The numbering gap between public (0-9) and private (10-19) is a habit that pays off during incident response at 2 AM — you can tell a subnet's purpose from its CIDR alone.

**Public vs Private Subnet — The Real Difference**

- **Public subnet** — its route table has a route to an **Internet Gateway**. Anything launched here with a public IP is directly reachable from the internet. Use for: load balancers, bastion hosts, NAT Gateways.
- **Private subnet** — its route table has **no** route to an Internet Gateway. Nothing here is reachable from the internet no matter what IP it has. Use for: application servers, databases, internal caches — Zyra's checkout API and its PostgreSQL database both belong here.

### 2. Phase 2 — Internet Gateway & Public Routing

**Business Problem:** Zyra's public subnets exist, but right now nothing in them can reach the internet, and the internet can't reach them either — a subnet is just an IP range until routing says otherwise. The Internet Gateway is the door; the route table is the sign telling traffic which door to use.

**Scene 2 — Zyra War Room | "Why Can't I Even Ping Out?"**

> **Kavya (You)** _Junior Cloud/Network Engineer_
>
> I launched a test EC2 instance in the public subnet with a public IP assigned, but it can't reach the internet at all — not even `apt update` works. What's missing?

> **Arjun** _Senior Cloud Architect_
>
> A subnet doesn't automatically know how to reach the internet just because you called it "public" in the tag. You need three things: an Internet Gateway attached to the VPC, a route in that subnet's route table pointing `0.0.0.0/0` at the Internet Gateway, and the instance itself must have a public IP. Miss any one of the three and you get exactly what you're seeing — silence.

#### 2.1 Create and Attach the Internet Gateway

```bash
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=zyra-igw}]'

aws ec2 attach-internet-gateway \
  --vpc-id vpc-0abc123 \
  --internet-gateway-id igw-0def456
```

> **📖 What an Internet Gateway Actually Is**
>
> An Internet Gateway (IGW) is a horizontally scaled, highly available AWS-managed component — it costs nothing to run and you never patch or resize it. It performs one job: **1:1 NAT** between a resource's private IP and its assigned public IP for internet-bound traffic. A VPC can have exactly one IGW attached. Creating it does nothing on its own — it must be attached to the VPC, and referenced from a route table, before any traffic flows through it.

#### 2.2 Create the Public Route Table

```bash
aws ec2 create-route-table --vpc-id vpc-0abc123 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=zyra-public-rt}]'

# Route all internet-bound traffic (0.0.0.0/0) to the Internet Gateway
aws ec2 create-route \
  --route-table-id rtb-0pub111 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0def456

# Associate the route table with both public subnets
aws ec2 associate-route-table --route-table-id rtb-0pub111 --subnet-id subnet-pub1a
aws ec2 associate-route-table --route-table-id rtb-0pub111 --subnet-id subnet-pub1b
```

> **📖 Route Tables — The Actual Traffic Rules**
>
> Every subnet is associated with exactly one route table (though one route table can serve many subnets). `0.0.0.0/0 → igw-0def456` reads as: "for any destination not already covered by a more specific route, send it to the Internet Gateway." Every VPC has an implicit **local** route (traffic within `10.20.0.0/16` stays inside the VPC) that you never have to add — AWS creates it automatically and it can't be removed. This is what makes a subnet "public": the presence of that `0.0.0.0/0 → igw` route, nothing else.

> **Quiz: An instance in a public subnet has a public IP but still can't be reached from the internet. What's the most likely missing piece?**
> - The instance needs an Elastic IP instead of a public IP
> - The subnet's route table has no route to the Internet Gateway, or the security group doesn't allow the inbound port
> - The VPC CIDR block is too small
>
> > **Answer/explanation:** The second option. A public IP alone does nothing without a route — the subnet's route table must send `0.0.0.0/0` traffic to the Internet Gateway, and separately, the instance's security group must explicitly allow the inbound port (e.g. 443 or 22). Both conditions are required simultaneously; missing either one results in total silence, which is exactly why this is one of the most common "why can't I connect" tickets in real AWS environments. Elastic IPs matter for keeping the *same* public IP across stop/start cycles, not for reachability itself. VPC CIDR size is unrelated to internet reachability.

### 3. Phase 3 — NAT Gateway & Private Egress

**Business Problem:** Zyra's checkout API lives in a private subnet — correctly, since it should never be directly reachable from the internet. But it still needs *outbound* access: to call the Razorpay payment API, download OS security patches, and fetch a third-party SMS library. Private subnets have no route to an Internet Gateway, so outbound calls fail entirely without a NAT Gateway.

#### 3.1 Deploy the NAT Gateway in a Public Subnet

```bash
# Allocate an Elastic IP for the NAT Gateway
aws ec2 allocate-address --domain vpc

# Create the NAT Gateway inside the PUBLIC subnet (this is the key detail)
aws ec2 create-nat-gateway \
  --subnet-id subnet-pub1a \
  --allocation-id eipalloc-0aaa111 \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=zyra-nat-1a}]'
```

> **📖 Why the NAT Gateway Lives in the Public Subnet**
>
> This trips up almost every fresher: the NAT Gateway itself sits in a **public** subnet (so it can reach the Internet Gateway), while the traffic it serves originates from the **private** subnet. Think of it as a one-way translator standing at the door — private instances hand it their request, it steps into the public subnet, goes out through the IGW using its own Elastic IP, and brings the response back. The private instance's real IP is never exposed to the internet — the outside world only ever sees the NAT Gateway's Elastic IP.

#### 3.2 Route Private Subnet Traffic Through the NAT Gateway

```bash
aws ec2 create-route-table --vpc-id vpc-0abc123 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=zyra-private-rt-1a}]'

aws ec2 create-route \
  --route-table-id rtb-0priv111 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-0bbb222

aws ec2 associate-route-table --route-table-id rtb-0priv111 --subnet-id subnet-priv1a
```

> **📖 Private Route Table — Egress Only**
>
> The private route table also sends `0.0.0.0/0` traffic somewhere — but to the **NAT Gateway ID**, not an Internet Gateway ID. That distinction is what makes this subnet "private with outbound access" rather than "public." Traffic can leave, but nothing can initiate a connection *into* this subnet from the internet — the NAT Gateway only ever forwards responses to requests that originated inside the VPC, it never accepts new inbound connections.

**NAT Gateway vs NAT Instance**

- **NAT Gateway** — AWS-managed, scales automatically to 45 Gbps, no patching, highly available within its AZ, costs an hourly rate plus per-GB data processing. Use for: production workloads — this is what Zyra uses.
- **NAT Instance** — a regular EC2 instance running NAT software, which you patch, size, and monitor yourself. Cheaper for very low, predictable traffic, and can act as a bastion host too, but is a single point of failure and a maintenance burden. Use only for: dev/test environments or extreme cost-sensitivity at very low traffic.

> **💡 Fresher Tip — One NAT Gateway per AZ for High Availability**
>
> A NAT Gateway is scoped to a single AZ. If Zyra only creates one NAT Gateway in `ap-south-1a` and routes *both* private subnets through it, then an outage in `ap-south-1a` takes down outbound internet for the private subnet in `ap-south-1b` too — even though that subnet's own AZ is healthy. Production design: one NAT Gateway per AZ, each private subnet routes through the NAT Gateway in its *own* AZ. Costs more (each NAT Gateway has its own hourly charge), but removes a cross-AZ single point of failure.

### 4. Phase 4 — Security Groups & Network ACLs

**Business Problem:** Even with correct routing, Zyra's checkout API and database are both technically reachable from anything else inside the VPC — including a compromised test instance a developer forgot to terminate. Security Groups and NACLs are the two firewall layers that restrict exactly who can talk to whom, at the instance level and the subnet level.

**Scene 3 — Zyra Security Review | "Anything Can Talk to Anything"**

> **Meera** _VP Engineering_
>
> Our pen-test report flagged this: any EC2 instance inside our VPC can currently open a connection to the database on port 5432. That includes a dev/test box, a jump box, anything. If one of those gets compromised, the database is one hop away. I want the database reachable from exactly one thing — the checkout API — and nothing else.

#### 4.1 Security Group for the Checkout API

```bash
aws ec2 create-security-group \
  --group-name zyra-checkout-api-sg \
  --description "Zyra checkout API - allows HTTPS from ALB only" \
  --vpc-id vpc-0abc123

# Only allow inbound HTTPS traffic from the Application Load Balancer's security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-0api111 \
  --protocol tcp --port 443 \
  --source-group sg-0alb999
```

> **📖 Security Groups Are Stateful and Deny-by-Default**
>
> A Security Group only needs **allow** rules — there's no explicit "deny" rule type because everything not explicitly allowed is denied by default. `--source-group sg-0alb999` (instead of a CIDR block) means "only traffic originating from instances in the ALB's security group" — this is far more precise and self-documenting than an IP range, and it automatically stays correct even if the ALB's IP changes. Security Groups are **stateful**: if inbound traffic on port 443 is allowed in, the response traffic is automatically allowed back out — you never write a matching outbound rule for replies.

#### 4.2 Security Group for the Database — Only the API Can Connect

```bash
aws ec2 create-security-group \
  --group-name zyra-db-sg \
  --description "Zyra PostgreSQL - only checkout API can connect" \
  --vpc-id vpc-0abc123

aws ec2 authorize-security-group-ingress \
  --group-id sg-0db222 \
  --protocol tcp --port 5432 \
  --source-group sg-0api111
```

> **📖 Chaining Security Groups Instead of CIDRs**
>
> By referencing `sg-0api111` (the checkout API's security group) as the source instead of `10.20.10.0/24`, this rule says "only instances that are members of the checkout-api security group may connect on 5432" — regardless of which private subnet they happen to land in. A random EC2 instance in the same private subnet, but not in the API's security group, still gets connection refused. This is the exact fix for Meera's pen-test finding: the database is no longer reachable from "anything in the VPC," only from instances explicitly tagged as the checkout API.

#### 4.3 Network ACLs — The Second, Subnet-Wide Layer

```bash
aws ec2 create-network-acl --vpc-id vpc-0abc123 \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=zyra-private-nacl}]'

# Rule 100: allow inbound from the VPC CIDR only, on the Postgres port
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0nnn333 --rule-number 100 --protocol tcp \
  --port-range From=5432,To=5432 --cidr-block 10.20.0.0/16 \
  --rule-action allow --ingress

# Rule 200: explicitly deny everything else inbound
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0nnn333 --rule-number 200 --protocol -1 \
  --cidr-block 0.0.0.0/0 --rule-action deny --ingress
```

> **📖 NACLs — Stateless, Numbered, and Subnet-Wide**
>
> Unlike Security Groups, NACLs are **stateless** — you must write explicit rules for both inbound and outbound (return) traffic, or responses get silently dropped. Rules are evaluated in order by rule number (lowest first); the first match wins, and everything not explicitly allowed is denied at the end by an implicit final deny rule. NACLs apply to the entire subnet, not one instance, making them a coarse second layer — a backstop in case a Security Group is ever misconfigured. Zyra's rule: Security Groups do the precise, per-service filtering; NACLs enforce a broad subnet-level boundary that stays correct even if someone fat-fingers a Security Group rule.

> **Security Groups vs NACLs — When Each Layer Matters**
>
> - **Security Groups** — stateful, attached to instances/ENIs, allow-only, evaluates all rules together. Use for: precise, per-service access control — this is where 95% of Zyra's rules live.
> - **Network ACLs** — stateless, attached to subnets, allow and explicit deny, evaluated in rule-number order. Use for: subnet-wide guardrails, blocking a known-bad IP range at the subnet boundary, or defense-in-depth.

> **Key takeaways**
> - Public subnets are defined by a route to an Internet Gateway; private subnets are defined by its absence — tagging alone means nothing to AWS.
> - A NAT Gateway lives in a public subnet but serves private subnets — one per AZ in production, to avoid a cross-AZ single point of failure.
> - Security Groups are stateful and allow-only; reference other security groups as sources instead of CIDR blocks whenever the source is another AWS resource, not a fixed IP range.
> - NACLs are stateless and subnet-wide — they need explicit inbound *and* outbound rules and are best used as a coarse second layer behind precise Security Group rules.
> - Never expose a database's security group to `0.0.0.0/0` or to an entire subnet CIDR — scope it to the exact security group of the one service that should reach it.

### 5. Phase 5 — Load Balancing Across Availability Zones

**Business Problem:** Zyra's checkout API currently runs as a single EC2 instance in one private subnet. If that instance — or its entire Availability Zone — goes down during the Diwali sale, checkout stops completely. An Application Load Balancer spread across both AZs, targeting instances in both private subnets, removes both single points of failure.

#### 5.1 Create the Application Load Balancer in the Public Subnets

```bash
aws elbv2 create-load-balancer \
  --name zyra-checkout-alb \
  --subnets subnet-pub1a subnet-pub1b \
  --security-groups sg-0alb999 \
  --scheme internet-facing \
  --type application
```

> **📖 Why the ALB Sits in Public Subnets, Targeting Private Ones**
>
> The ALB itself needs a public subnet in each AZ because it's the internet-facing entry point — customers' browsers connect directly to it. But the ALB's **targets** (the actual checkout API instances) live in the private subnets, where they're never directly reachable. AWS automatically provisions the ALB's own elastic network interfaces across the subnets you list, one per AZ, giving true multi-AZ resilience for the load balancer itself, not just the backend.

#### 5.2 Target Group Spanning Both Private Subnets

```bash
aws elbv2 create-target-group \
  --name zyra-checkout-tg \
  --protocol HTTPS --port 443 \
  --vpc-id vpc-0abc123 \
  --health-check-path /health \
  --health-check-interval-seconds 15

aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:ap-south-1:111111111111:targetgroup/zyra-checkout-tg/abc \
  --targets Id=i-0api1a Id=i-0api1b
```

> **📖 Health Checks Drive Routing**
>
> `Id=i-0api1a` and `Id=i-0api1b` are checkout API instances in AZ 1a and 1b respectively. The ALB polls `/health` on each target every 15 seconds; only instances passing health checks receive traffic. If the instance in `ap-south-1a` fails its health check — or the entire AZ has an issue — the ALB stops routing to it within a couple of failed checks and sends 100% of traffic to `ap-south-1b` automatically, with no manual failover step. This is the actual mechanism that would have prevented last year's 40-minute Diwali outage.

### 6. Phase 6 — VPC Peering, Transit Gateway & VPC Endpoints

**Business Problem:** Zyra's warehouse-management partner runs their own AWS account with its own VPC, and Zyra's checkout API needs to query real-time stock levels from it. Separately, Zyra's Lambda functions need to read and write to S3 and DynamoDB — but routing that traffic over the public internet (even encrypted) adds latency and an unnecessary exposure path.

**Scene 4 — Zyra Architecture Review | "Two VPCs, One Business"**

> **Arjun** _Senior Cloud Architect_
>
> The warehouse partner's VPC uses `10.30.0.0/16` — good, no overlap with our `10.20.0.0/16`, so peering is possible. For just two VPCs needing to talk, VPC Peering is simpler than a Transit Gateway. If we ever need five or six VPCs all talking to each other, we'll revisit — a full mesh of peering connections gets unmanageable past three or four VPCs.

#### 6.1 VPC Peering Connection

```bash
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-0abc123 \
  --peer-vpc-id vpc-0warehouse999 \
  --peer-owner-id 222222222222

# The warehouse account must accept it
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-0peer555

# Add routes on BOTH sides pointing at each other's CIDR
aws ec2 create-route --route-table-id rtb-0priv111 \
  --destination-cidr-block 10.30.0.0/16 \
  --vpc-peering-connection-id pcx-0peer555
```

> **📖 Peering Is Not Transitive**
>
> VPC Peering is a direct, non-transitive 1:1 connection — traffic flows only between the two peered VPCs, never through one to reach a third. Both sides need their own route added; forgetting the warehouse account's side means traffic reaches the peering connection and silently drops. There's no bandwidth bottleneck (traffic uses AWS's private backbone, not the public internet) and no data transfer charges within the same region for peered traffic in some pricing tiers — always confirm current pricing.

**VPC Peering vs Transit Gateway**

- **VPC Peering** — free-ish, simple, but non-transitive and becomes an unmanageable full mesh past 3-4 VPCs. Use for: Zyra's exact case — exactly two VPCs need to talk.
- **Transit Gateway** — a central hub that many VPCs (and VPNs, and Direct Connect links) attach to; add a fifth VPC by attaching it to the hub once, not by creating four new peering connections. Costs more (hourly + per-GB), but scales cleanly. Use for: an organization with many VPCs, or VPCs that need transitive routing.

#### 6.2 VPC Endpoint — Reach S3 Without the Public Internet

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0abc123 \
  --service-name com.amazonaws.ap-south-1.s3 \
  --route-table-ids rtb-0priv111 \
  --vpc-endpoint-type Gateway
```

> **📖 Gateway Endpoints vs Interface Endpoints**
>
> A **Gateway Endpoint** (used for S3 and DynamoDB only) adds a route directly in your route table — traffic to S3 stays entirely on AWS's internal network and never touches a NAT Gateway or the public internet, and it's free. An **Interface Endpoint** (used for most other services, like Lambda, SNS, or Secrets Manager) creates an Elastic Network Interface with a private IP inside your subnet, backed by AWS PrivateLink, and has an hourly + per-GB cost. For Zyra, adding the S3 Gateway Endpoint means checkout API calls to fetch product images from S3 no longer route through — and pay for — the NAT Gateway at all.

> **Quiz: Zyra's checkout API (in a private subnet) needs to write logs to an S3 bucket. Which is the most cost-effective, most secure way to do this?**
> - Give the checkout API instance a public IP and let it reach S3 over the internet
> - Route the traffic through the NAT Gateway to reach S3's public endpoint
> - Add an S3 Gateway VPC Endpoint and route table entry, so traffic never leaves the AWS network
>
> > **Answer/explanation:** The third option. A Gateway VPC Endpoint for S3 is free, keeps traffic entirely on AWS's private backbone (never touching the public internet), and avoids per-GB NAT Gateway data processing charges entirely for S3 traffic. Giving the instance a public IP directly contradicts the entire point of putting it in a private subnet. Routing through the NAT Gateway would technically work but is both slower and needlessly expensive compared to a Gateway Endpoint, which exists specifically for this S3/DynamoDB use case.

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Build a bastion host with Session Manager instead of SSH:** Launch a small EC2 instance in the public subnet with no inbound Security Group rules at all, and connect using AWS Systems Manager Session Manager instead of opening port 22. Confirm you can reach a private-subnet instance from it without ever exposing SSH to the internet.
2. **Add a third AZ:** Extend Zyra's design to `ap-south-1c` — add a public and private subnet pair, a third NAT Gateway, and register a third checkout API instance with the ALB target group. Verify traffic distributes across all three AZs.
3. **Enable VPC Flow Logs:** Turn on Flow Logs for the VPC, send them to CloudWatch Logs, and write a query that finds all REJECTED connections to the database security group in the last 24 hours — useful for confirming your Security Group rules are actually being enforced.
4. **Replace the NAT Gateway cost with NAT instance for a dev environment:** In a separate, low-traffic dev VPC, launch a small NAT instance instead of a NAT Gateway, disable source/destination checking on it, and compare the monthly cost difference against the production NAT Gateway.
5. **Add a Transit Gateway and simulate a third VPC:** Create a third VPC (e.g. for a new "returns processing" service), attach all three VPCs to a Transit Gateway instead of point-to-point peering, and confirm transitive routing works — the returns VPC can reach the warehouse VPC without a direct peering connection between them.

### AWS Networking Project Complete 🎉

You have designed and built Zyra Retail's production VPC from a bare CIDR block to a fully routed, multi-AZ, security-layered network — public and private subnets, an Internet Gateway, per-AZ NAT Gateways, chained Security Groups, subnet-wide NACLs, a multi-AZ Application Load Balancer, and both VPC Peering and a Gateway Endpoint for external and AWS-service connectivity. This is the same shape of network running behind real production e-commerce platforms today.

> **Arjun**
>
> "Kavya, look at what changed: the database used to be reachable from anything in the VPC. Now it's reachable from exactly one security group. The checkout API used to be a single instance in one AZ. Now it's two instances behind a load balancer, in two AZs, with automatic failover. If Diwali traffic takes down one AZ this year, customers won't even notice."

> **Meera**
>
> "The pen-test re-run came back clean on network exposure — for the first time. Every rule you wrote has a reason a business person can understand: 'the database only talks to the checkout API' isn't a networking abstraction anymore, it's a sentence I can say in a board meeting."

> **Next: AWS Cloud Security — IAM, GuardDuty, CloudTrail & Compliance**

> - Least-privilege IAM policies so that even a compromised checkout API instance can't do more than its job requires
> - GuardDuty and CloudTrail to detect and log exactly who did what across this VPC, down to the API call
> - AWS Config rules that continuously check the network you just built stays compliant as it evolves
> - Security Hub to aggregate all of these signals into a single dashboard your security team actually watches
