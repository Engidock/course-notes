# AWS Project Mastery

> **👋 Hey Fresher — Read This First!**

> - AWS (Amazon Web Services) is the world's largest cloud platform — used by Netflix, Zomato, BYJU'S, and thousands of Indian startups. **72% of cloud job postings mention AWS.**
> - Cloud means: instead of buying physical servers, you rent computing power, storage, and networking from Amazon — by the hour, scaling up or down as needed.
> - This document uses **short code blocks and console previews** — each one covers exactly one AWS concept — with bullet-point explanations so every line is clear.
> - **Company in this project:** ShopNest — an e-commerce startup in Bengaluru that outgrew its single shared server. You just joined as a Junior Cloud/DevOps Engineer. Your lead is Nandita. You will migrate ShopNest's application to AWS — from scratch — using the industry-standard setup every company uses.

#### What You Will Build in This Project

You will build **ShopNest's complete AWS infrastructure** — the same architecture used by production e-commerce companies — one service at a time, each explained clearly.

IAM & Security, VPC & Networking, EC2 Servers, S3 Storage, RDS Database, Load Balancer, Auto Scaling, CloudWatch

> **🔐 Phase 1 — IAM: Users, Roles & Policies**

> Create IAM users with least-privilege policies, set up roles for EC2, and configure MFA. Security is configured first — before anything else.

> Create a VPC with public and private subnets, Internet Gateway, Route Tables, and Security Groups — ShopNest's isolated network on AWS.

> Launch EC2 instances, connect via SSH, install the application, use User Data for auto-setup, and attach an Elastic IP.

> Create an S3 bucket for product images, configure an RDS MySQL database in the private subnet, and connect the app.

> Put an Application Load Balancer in front of multiple EC2 instances, configure Auto Scaling for traffic spikes, and set CloudWatch alarms.

**Scene 1 — ShopNest Engineering Office, Bengaluru | The Server Problem**

> **Nandita** _Senior Cloud Engineer — ShopNest_
> 
> Karthik, last Diwali our website went down for 4 hours. We were running everything on one shared hosting server — the database, the app, the images — all on the same machine. When traffic spiked to 50,000 users, the server ran out of memory and crashed. We lost over 18 lakh rupees in sales. AWS solves this: compute, storage, database, and networking are all separate services that scale independently. Your job is to rebuild our infrastructure on AWS properly.

> **Karthik (You)** _Junior Cloud Engineer — Day 1 at ShopNest_
> 
> I've heard of EC2 and S3, but I'm not sure where to start. There are hundreds of AWS services — do I need to know all of them?

> **Nandita** _Senior Cloud Engineer_
> 
> No. For most companies, 80% of their infrastructure uses just 8–10 services. Master those and you can get a job at any company. We'll start with IAM — Identity and Access Management. It controls WHO can do WHAT on AWS. Most freshers skip this and go straight to EC2. That's wrong. Security is always first. Never log into AWS as the root account after today.

> **Suresh** _DevOps Architect — ShopNest_
> 
> And remember the AWS shared responsibility model: AWS is responsible for the security OF the cloud — their physical data centres, the hardware, the hypervisor. You are responsible for security IN the cloud — your configurations, your IAM policies, your security group rules, your encryption settings. AWS gives you the tools. Using them correctly is your job.

### 1. Phase 1 — IAM: Identity and Access Management

IAM is the foundation of AWS security. It controls who can access your AWS account and what actions they are allowed to perform. Set up IAM correctly before creating any other resource.

> **The Big Picture — Why IAM Matters**

> - When you create an AWS account, you get a **root account**. It has unlimited access to everything. **Never use the root account for day-to-day work** — treat it like the master key to a building: only touch it to create the first IAM admin user, then lock it away.
> - IAM lets you create **users** (for people) and **roles** (for AWS services) with exactly the permissions they need — nothing more.
> - The principle of **Least Privilege**: give every user and service the minimum permissions they need to do their job. An EC2 server that serves web pages should not have permissions to delete databases.
> - IAM is **free** — no cost for creating users, roles, or policies.

```
ShopNest AWS Account — IAM Structure
========================================

  Root Account (locked away after setup)
  │
  └── IAM Users
      ├── karthik-dev   → Developer policy (EC2, S3 read/write, no IAM)
      ├── nandita-admin → AdminAccess policy
      └── ci-deploy     → Restricted deploy policy (specific S3 + EC2 only)

  IAM Roles (attached to AWS services, not people)
  ├── shopnest-ec2-role → allows EC2 to read from S3, write to CloudWatch
  └── shopnest-rds-role → allows RDS to publish metrics to CloudWatch

  IAM Groups
  ├── Developers  → dev-policy attached (all developers inherit this)
  └── Ops         → ops-policy attached
```

#### 1.1 Create an IAM User with AWS CLI

```
# Configure AWS CLI with your credentials first
aws configure
# AWS Access Key ID: AKIA...
# AWS Secret Access Key: ...
# Default region: ap-south-1
# Default output format: json
# Create a new IAM user
aws iam create-user --user-name karthik-dev
# Give the user console access (set password)
aws iam create-login-profile \
  --user-name karthik-dev \
  --password TempPass@2026! \
  --password-reset-required
```

**📖 Creating IAM Users**

- **aws configure** — stores your AWS credentials locally in `~/.aws/credentials`. Required before running any AWS CLI commands.
- **ap-south-1** — AWS region code for Mumbai (Asia Pacific South 1). Use this for lowest latency in India.
- **create-user** — creates the IAM user account. No permissions yet — just the identity.
- **create-login-profile** — gives the user a password to log into the AWS Console web UI.
- **--password-reset-required** — forces the user to change password on first login. Always set this for new users.

#### 1.2 Attach a Policy to the User

```
# Attach AWS managed policy — gives EC2 full access
aws iam attach-user-policy \
  --user-name karthik-dev \
  --policy-arn \
  arn:aws:iam::aws:policy/AmazonEC2FullAccess
# Attach read-only S3 access
aws iam attach-user-policy \
  --user-name karthik-dev \
  --policy-arn \
  arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

**📖 IAM Policies**

- A **policy** is a JSON document that lists what actions are allowed or denied on which AWS resources.
- **AWS Managed Policies** are pre-built by Amazon for common use cases. Use these as a starting point.
- **ARN** (Amazon Resource Name) — a unique identifier for every resource in AWS. Format: `arn:aws:service:region:account-id:resource`
- For managed policies, the account-id part is `aws` (Amazon's own account).
- Best practice: attach policies to **groups**, not individual users. Add users to groups. Easier to manage at scale.

#### 1.3 Create an IAM Role for EC2

```
# Create a trust policy file (who can use this role)
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect":    "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action":    "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name shopnest-ec2-role \
  --assume-role-policy-document file://trust-policy.json
```

**📖 IAM Roles vs Users**

- A **role** is like a user but for AWS services, not people. An EC2 instance assumes a role to get temporary credentials.
- **Trust policy** — answers "WHO can use this role?" Here: only EC2 instances.
- **ec2.amazonaws.com** — the EC2 service principal. This tells AWS: "EC2 instances are allowed to assume this role."
- Roles give **temporary credentials** that rotate automatically. This is more secure than putting long-lived access keys on EC2 instances.
- Never store AWS access keys inside EC2 instances. Always use IAM roles instead.

> **🔐 IAM Security Rules — Non-Negotiable**

> - Enable **MFA (Multi-Factor Authentication)** on the root account and all IAM users with admin access. Without MFA, a leaked password = full account compromise.
> - Never create or store access keys for the root account. If root access keys exist, delete them immediately.
> - **Never hardcode AWS credentials** in your application code, Dockerfiles, or Git repositories. Use IAM roles for EC2/Lambda and environment variables for local development.
> - Regularly audit IAM with **IAM Access Analyzer** and **AWS Trusted Advisor** to find over-privileged users and unused credentials.

### 2. Phase 2 — VPC: Your Private Network on AWS

**Business Problem:** ShopNest's database must never be directly reachable from the internet — only the application server should be able to connect to it. ShopNest's web servers need public IP addresses to serve customers. A VPC (Virtual Private Cloud) gives you a private, isolated network section in AWS where you control who can reach what.

**Scene 2 — Architecture Planning, ShopNest | "Design the Network First"**

> **Suresh** _DevOps Architect — ShopNest_
> 
> Karthik, think of a VPC like building a private office building. You decide how the floors are laid out, which doors open to the street, which rooms are internal-only. Public subnets are like the lobby — accessible from outside. Private subnets are like the server room — only accessible from inside. The Internet Gateway is the building's front entrance. Security Groups are the individual door locks on each room.

```
ShopNest VPC Architecture (ap-south-1 Mumbai)
================================================

  VPC: 10.0.0.0/16  (65,536 IP addresses)
  │
  ├── Public Subnet A  10.0.1.0/24  (AZ: ap-south-1a)
  │    └── EC2 Web Server 1 ← gets public IP, handles customer requests
  │    └── Application Load Balancer
  │
  ├── Public Subnet B  10.0.2.0/24  (AZ: ap-south-1b)
  │    └── EC2 Web Server 2 ← second AZ for high availability
  │
  ├── Private Subnet A  10.0.3.0/24  (AZ: ap-south-1a)
  │    └── RDS MySQL ← no public IP, only EC2 can reach it
  │
  └── Private Subnet B  10.0.4.0/24  (AZ: ap-south-1b)
       └── RDS Standby (Multi-AZ failover)

  Internet Gateway → allows public subnets to reach the internet
  NAT Gateway → allows private subnets to reach internet (for updates)
                without being reachable FROM the internet
```

#### 2.1 Create the VPC

```
# Create the VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications \
  'ResourceType=vpc,Tags=[{Key=Name,Value=shopnest-vpc}]'
# Output gives you the VPC ID, e.g. vpc-0abc1234
# Save this — you'll need it for every resource below
```

**📖 CIDR Block Explained**

- **10.0.0.0/16** — the IP address range for your entire VPC. The /16 means you have 65,536 private IP addresses (10.0.0.0 through 10.0.255.255).
- The **/24** used for subnets gives 256 addresses per subnet.
- These are **private IP addresses** — they only work inside your VPC, not on the public internet.
- **Tags** — always tag every AWS resource. Tags are key-value labels. Without them, a bill of 50 resources is unmanageable.

#### 2.2 Create Subnets

```
# Public subnet in AZ ap-south-1a
aws ec2 create-subnet \
  --vpc-id vpc-0abc1234 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications \
  'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]'
# Private subnet in AZ ap-south-1a
aws ec2 create-subnet \
  --vpc-id vpc-0abc1234 \
  --cidr-block 10.0.3.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications \
  'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]'
```

**📖 Subnets & Availability Zones**

- A **subnet** is a range of IP addresses within your VPC. Think of it as a floor in your building.
- **Public subnet** — connected to an Internet Gateway. Resources here can get public IP addresses and communicate with the internet.
- **Private subnet** — no direct internet access. Resources here can only be reached from inside the VPC.
- **Availability Zone (AZ)** — a physically separate data centre within a region. ap-south-1a and ap-south-1b are two different buildings in Mumbai. If one catches fire, the other keeps running.
- Always deploy in at least **2 AZs** for high availability.

#### 2.3 Security Group — Virtual Firewall

```
# Create security group for web servers
aws ec2 create-security-group \
  --group-name shopnest-web-sg \
  --description "Web server security group" \
  --vpc-id vpc-0abc1234
# Allow HTTP (port 80) from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc5678 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

**📖 Security Groups**

- A **security group** acts as a virtual firewall for EC2 instances. Controls incoming (ingress) and outgoing (egress) traffic.
- **0.0.0.0/0** — means "from any IP address in the world." Use for HTTP/HTTPS ports only.
- Security groups are **stateful** — if you allow inbound traffic on port 80, the response automatically goes back. No need to add an outbound rule for it.
- By default, security groups **allow all outbound traffic** and **deny all inbound traffic**.
- Separate security groups for web servers, database, and load balancer. Never put everything in one security group.

AWS

VPC › Security Groups › shopnest-web-sg

Security group ID sg-0abc5678def90123

VPC ID vpc-0abc1234

Inbound rules Port 80 (HTTP) → 0.0.0.0/0 | Port 443 (HTTPS) → 0.0.0.0/0 | Port 22 (SSH) → Your IP only

Outbound rules All traffic → 0.0.0.0/0 (default)

### 3. Phase 3 — EC2: Launch Your Web Server

**Business Problem:** ShopNest needs virtual machines in the cloud to run their Node.js application. EC2 (Elastic Compute Cloud) is AWS's service for virtual machines. You choose the OS, CPU, RAM, and storage. It launches in under 60 seconds. You pay only while it's running.

**Scene 3 — ShopNest Server Setup | "Launch the First Instance"**

> **Nandita** _Senior Cloud Engineer_
> 
> Karthik, for EC2 you need to choose four things: the AMI (which operating system), the instance type (how much CPU and RAM), the subnet (which part of our network), and the key pair (how you'll SSH in). Our app runs on Ubuntu 22.04. For the instance type, start with t3.micro for testing — it's free tier eligible. We'll upgrade to t3.medium for production. And always use User Data to auto-install software on launch — never SSH in and install things manually.

#### 3.1 Create a Key Pair for SSH Access

```
# Create an SSH key pair
aws ec2 create-key-pair \
  --key-name shopnest-key \
  --query 'KeyMaterial' \
  --output text \
  > shopnest-key.pem

# Secure the file — SSH refuses to use it if others can read it
chmod 400 shopnest-key.pem
```

**📖 Key Pairs Explained**

- A **key pair** is an SSH certificate. AWS stores the public key on the EC2 instance. You keep the private key (.pem file) on your laptop.
- When you SSH, your laptop proves identity with the private key — no password needed.
- **chmod 400** — makes the file readable only by you. SSH refuses to use a .pem file that anyone else can read (security protection).
- **Never share or commit .pem files** to Git. Add *.pem to .gitignore immediately.
- If you lose the .pem file, you lose SSH access to that instance forever.

#### 3.2 Launch an EC2 Instance

```
aws ec2 run-instances \
  --image-id ami-0f5ee92e2d63afc18 \
  --instance-type t3.micro \
  --key-name shopnest-key \
  --subnet-id subnet-public-1a \
  --security-group-ids sg-0abc5678 \
  --iam-instance-profile Name=shopnest-ec2-role \
  --associate-public-ip-address \
  --count 1
```

**📖 EC2 Launch Parameters**

- **--image-id** — AMI (Amazon Machine Image). This is the operating system template. ami-0f5ee92e2d63afc18 = Ubuntu 22.04 LTS in Mumbai region. AMI IDs are different per region.
- **--instance-type** — CPU + RAM size. t3.micro = 2 vCPU + 1GB RAM. Free tier eligible. t3.small = 2 vCPU + 2GB. t3.medium = 2 vCPU + 4GB.
- **--iam-instance-profile** — attaches the IAM role. The instance gets temporary credentials to access S3, CloudWatch, etc. without storing any access keys.
- **--associate-public-ip-address** — gives the instance a public IP so it's reachable from the internet.

#### 3.3 User Data — Auto-Install Software on Launch

```bash
# user-data.sh — runs automatically on first boot
#!/bin/bash
apt-get update -y
apt-get install -y nodejs npm nginx

cd /home/ubuntu
git clone https://github.com/shopnest/app.git
cd app && npm install

systemctl start nginx
systemctl enable nginx
```

**📖 User Data — Bootstrap Script**

- **User Data** is a shell script that runs automatically as root when the EC2 instance boots for the first time.
- Use it to install packages, clone your application code, and start services — no manual SSH needed after launch.
- Pass it in the launch command with: `--user-data file://user-data.sh`
- Logs are at `/var/log/cloud-init-output.log` — check this if startup fails.
- User Data only runs **once** on first launch. If you need to re-run it, use SSM Run Command or re-launch the instance.

#### 3.4 Connect to the Instance via SSH

```
# Find the public IP of your instance
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].PublicIpAddress' \
  --output text
# SSH into the instance
ssh -i shopnest-key.pem ubuntu@13.235.100.45
# Check if User Data script ran successfully
cat /var/log/cloud-init-output.log
```

> **describe-instances** with `--query` — filters AWS API output using JMESPath syntax to get only the public IP. Without --query, you get hundreds of lines of JSON.
**ubuntu@IP** — the default username for Ubuntu AMIs is `ubuntu`. For Amazon Linux it's `ec2-user`. For RHEL it's `ec2-user`.
**cloud-init-output.log** — every line of your User Data script output is logged here. If your app didn't install correctly, this file tells you why.

AWS

EC2 › Instances › shopnest-web-1

Instance ID i-0a1b2c3d4e5f67890

Instance type t3.micro (2 vCPU / 1 GB RAM)

Public IPv4 13.235.100.45

Private IPv4 10.0.1.47

AMI Ubuntu 22.04 LTS (ami-0f5ee92e2d63afc18)

Availability zone ap-south-1a

IAM role shopnest-ec2-role

Connect

Stop

Actions ▾

### 4. Phase 4 — S3 and RDS: Storage and Database

**Business Problem:** ShopNest has 50,000 product images. Storing them on the EC2 disk is wrong — the disk disappears if the instance is replaced. S3 is permanent, globally accessible object storage. The MySQL database must be managed — automatic backups, patches, high availability — without the team maintaining a database server. RDS (Relational Database Service) provides this.

#### 4.1 Create an S3 Bucket

```
# Create bucket (must be globally unique)
aws s3api create-bucket \
  --bucket shopnest-product-images-2026 \
  --region ap-south-1 \
  --create-bucket-configuration \
  LocationConstraint=ap-south-1
# Enable versioning (recover accidentally deleted files)
aws s3api put-bucket-versioning \
  --bucket shopnest-product-images-2026 \
  --versioning-configuration \
  Status=Enabled
```

**📖 S3 Buckets**

- **S3** (Simple Storage Service) — stores any file (objects) up to 5TB. Highly durable: 99.999999999% (11 nines). Your data is replicated across multiple data centres automatically.
- **Bucket names are globally unique** across all AWS accounts worldwide. If "shopnest-images" is taken by any other account, your create fails.
- **Versioning** — every time a file is overwritten, S3 keeps the old version. You can restore any previous version. Critical for production data.
- S3 is **not a file system** — it's an object store. No folders (just key prefixes that look like folders). No "open file" operations.

#### 4.2 Upload and Access Files in S3

```
# Upload a single file
aws s3 cp product-123.jpg \
  s3://shopnest-product-images-2026/products/
# Upload an entire folder recursively
aws s3 sync ./images/ \
  s3://shopnest-product-images-2026/images/
# List objects in a bucket
aws s3 ls s3://shopnest-product-images-2026/
# Generate a pre-signed URL (valid 1 hour)
aws s3 presign \
  s3://shopnest-product-images-2026/products/product-123.jpg \
  --expires-in 3600
```

**📖 Key S3 Operations**

- **aws s3 cp** — copies a single file to/from S3. Works like Linux `cp` but for cloud storage.
- **aws s3 sync** — synchronises a local folder with S3. Only uploads changed files. Used in CI/CD to deploy frontend builds.
- **Pre-signed URL** — a temporary, time-limited URL that grants access to a private S3 object. Used to share files with customers without making the bucket public.
- S3 buckets are **private by default**. Never make a bucket public unless it's specifically for serving public web assets (CSS, JS, images).

#### 4.3 Create an RDS MySQL Database

```
# Create DB subnet group (required before RDS)
aws rds create-db-subnet-group \
  --db-subnet-group-name shopnest-db-subnets \
  --db-subnet-group-description "Private subnets for RDS" \
  --subnet-ids subnet-private-1a subnet-private-1b
# Launch RDS instance in private subnet
aws rds create-db-instance \
  --db-instance-identifier shopnest-mysql \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0.35 \
  --master-username admin \
  --master-user-password ShopNest@2026! \
  --allocated-storage 20 \
  --db-subnet-group-name shopnest-db-subnets \
  --no-publicly-accessible \
  --multi-az
```

**📖 RDS Key Options**

- **db.t3.micro** — smallest RDS instance. 2 vCPU, 1GB RAM. Good for dev/test. Use db.t3.small or db.t3.medium for production.
- **--no-publicly-accessible** — critical security setting. RDS will only have a private IP inside the VPC. EC2 instances can connect; the internet cannot.
- **--multi-az** — creates an automatic standby in a second AZ. If the primary fails, AWS automatically fails over to the standby in 1–2 minutes with no data loss.
- **--allocated-storage** — storage in GB. RDS has automatic storage scaling — it grows automatically when it gets close to full.
- RDS gives you automatic daily backups, automatic minor version patches, and CloudWatch metrics for free.

### 5. Phase 5 — Load Balancer, Auto Scaling & CloudWatch

**Business Problem:** During ShopNest's Diwali sale, traffic jumped from 1,000 to 50,000 users in 30 minutes. One EC2 instance cannot handle that. Auto Scaling automatically launches more EC2 instances when load increases and terminates them when it drops — paying only for what's needed. The Application Load Balancer distributes traffic across all running instances.

**Scene 4 — ShopNest Post-Mortem | "We Need to Auto-Scale"**

> **Suresh** _DevOps Architect — ShopNest_
> 
> Karthik, after the Diwali incident I analysed the traffic. At 8 PM, we had 1,200 users. At 8:15 PM — 48,000. That 40x spike happened in 15 minutes. No human can react that fast. Auto Scaling with a Load Balancer is the answer. At 60% CPU, automatically launch another EC2. If a health check fails, automatically route around the bad instance. When the sale ends, automatically terminate the extra instances. Zero manual intervention, zero downtime.

#### 5.1 Create an Application Load Balancer

```
# Create the ALB in public subnets
aws elbv2 create-load-balancer \
  --name shopnest-alb \
  --type application \
  --scheme internet-facing \
  --subnets subnet-public-1a subnet-public-1b \
  --security-groups sg-alb-id
# Create a target group (the EC2 instances to route to)
aws elbv2 create-target-group \
  --name shopnest-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-0abc1234 \
  --health-check-path /health
```

**📖 ALB Concepts**

- **ALB (Application Load Balancer)** — distributes incoming HTTP/HTTPS traffic across multiple EC2 instances. If one instance fails, ALB stops sending it traffic automatically.
- **internet-facing** — the ALB gets a public DNS name. Users hit the ALB, not individual EC2 instances.
- **Target Group** — the pool of EC2 instances the ALB routes to. Auto Scaling registers and deregisters instances here automatically.
- **Health check path (/health)** — ALB pings this URL on each instance every 30 seconds. Instance is only kept in rotation if it returns HTTP 200.

#### 5.2 Auto Scaling Group

```
# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name shopnest-asg \
  --launch-template LaunchTemplateId=lt-0abc,Version=1 \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 2 \
  --target-group-arns arn:aws:...:shopnest-targets \
  --vpc-zone-identifier subnet-public-1a,subnet-public-1b
```

**📖 Auto Scaling Group**

- **min-size: 2** — always keep at least 2 instances running. One per AZ for redundancy. If one AZ goes down, traffic routes to the other.
- **max-size: 10** — never scale beyond 10 (cost protection). Set this carefully.
- **desired-capacity: 2** — the current target. ASG works to maintain this count. Scaling policies adjust this number up or down.
- ASG **automatically registers new instances with the ALB target group** and deregisters terminated ones.
- If an instance fails its health check, ASG terminates it and launches a replacement automatically.

#### 5.3 Auto Scaling Policy — Scale on CPU

```
# Target Tracking Policy — maintain 60% avg CPU
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name shopnest-asg \
  --policy-name cpu-tracking-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration \
  '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 60.0
  }'
```

**📖 Target Tracking Policy**

- **TargetTrackingScaling** — AWS automatically adds or removes instances to keep average CPU near 60%.
- If CPU goes above 60%, ASG launches more instances (scale out).
- If CPU drops well below 60%, ASG terminates instances (scale in). Has a 5-minute cooldown after scale-in to prevent thrashing.
- This is the simplest and most recommended scaling policy for most applications.
- Other metrics you can track: RequestCountPerTarget (requests per second), NetworkIn, custom metrics.

#### 5.4 CloudWatch Alarms and Monitoring

```
# Create alarm: alert when CPU stays above 80%
aws cloudwatch put-metric-alarm \
  --alarm-name shopnest-high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:...:shopnest-alerts
```

**📖 CloudWatch Alarms**

- **period: 300** — evaluate the metric every 300 seconds (5 minutes).
- **evaluation-periods: 2** — alarm fires only if threshold is breached in 2 consecutive periods (10 minutes). Prevents false alarms from brief spikes.
- **threshold: 80** — fire the alarm when CPU > 80%.
- **alarm-actions** — the SNS topic ARN. When alarm fires, SNS sends an email/SMS notification to the ops team.
- CloudWatch automatically collects CPU, network, disk metrics from every EC2 instance. No setup needed for basic metrics. Enable "Detailed Monitoring" (1-minute granularity, extra cost) for production.

AWS

CloudWatch › Alarms › shopnest-high-cpu

Metric EC2 → CPUUtilization → Average

Condition CPU > 80% for 2 of 2 periods (5 min each)

Current value 42.3%

State OK — below threshold

When alarm fires → SNS topic: shopnest-alerts → Email to ops@shopnest.in

### 6. Essential AWS Services & CLI Reference

Service

What It Does

Common CLI Command

IAM

Users, roles, permissions

aws iam create-user / attach-user-policy

EC2

Virtual machines in the cloud

aws ec2 run-instances / describe-instances

S3

Object/file storage (unlimited)

aws s3 cp / sync / ls / presign

RDS

Managed relational database

aws rds create-db-instance / describe-db-instances

VPC

Private network on AWS

aws ec2 create-vpc / create-subnet

ELB/ALB

Load balancer across multiple EC2

aws elbv2 create-load-balancer / describe-target-health

Auto Scaling

Auto add/remove EC2 based on load

aws autoscaling create-auto-scaling-group

CloudWatch

Metrics, logs, alarms, dashboards

aws cloudwatch put-metric-alarm

SNS

Notifications: email, SMS, push

aws sns create-topic / subscribe / publish

Route 53

DNS — map domain name to ALB/EC2

aws route53 create-hosted-zone

ACM

Free SSL/TLS certificates

aws acm request-certificate

CloudFront

CDN — cache content closer to users

aws cloudfront create-distribution

Lambda

Run code without managing servers

aws lambda create-function / invoke

ECS/EKS

Run containers at scale

aws ecs create-cluster / run-task

Elastic IP

Static public IP that stays yours

aws ec2 allocate-address / associate-address

### 7. Interview Questions — AWS

##### Interview Q&A — Fresher Level (0–1 Year AWS Experience)

**Q: Q1. What is the difference between a Region, Availability Zone, and Edge Location in AWS?**

A: **Region** — a geographic area with multiple data centres. Example: ap-south-1 = Mumbai. You choose a region when creating most AWS resources. Data stays in your chosen region by default.
**Availability Zone (AZ)** — a physically separate data centre within a region, connected by high-speed private fibre. ap-south-1 has 3 AZs: ap-south-1a, 1b, 1c. If one AZ has a power failure, the others keep running. Always deploy in at least 2 AZs for high availability.
**Edge Location** — a CloudFront CDN cache point. There are 400+ worldwide. When a user in Chennai requests content, CloudFront serves it from the nearest edge location, not all the way from your region. Reduces latency from 200ms to 5ms for cached content.

**Q: Q2. What is the difference between Security Groups and Network ACLs?**

A: **Security Groups** — act at the EC2 instance level. Rules are stateful: if you allow inbound port 80, the response traffic is automatically allowed. You only define allow rules (no explicit deny).
**Network ACLs (NACLs)** — act at the subnet level, before traffic reaches any EC2 instance. Rules are stateless: you must explicitly allow both inbound AND outbound traffic. Support both allow and deny rules, evaluated in order by rule number.
Security Groups are your primary defense line — configure these correctly for every instance.
NACLs are the second layer — useful for blocking entire IP ranges at the subnet level (e.g., block a specific country).
Most companies only use Security Groups. NACLs are for advanced network-level defence.

**Q: Q3. What is the difference between S3 storage classes?**

A: **S3 Standard** — high availability, low latency. For frequently accessed data. Product images, application files. Costs most per GB stored.
**S3 Standard-IA** (Infrequent Access) — cheaper per GB, but retrieval costs money. Use for backups and logs accessed a few times a month.
**S3 Glacier Instant** — millisecond retrieval but much cheaper. For archival data accessed occasionally (compliance records).
**S3 Glacier Deep Archive** — cheapest (₹0.001/GB/month). Retrieval takes 12 hours. For data you almost never access but must keep for legal/compliance reasons.
Use **S3 Lifecycle Policies** to automatically move objects between classes based on age (e.g., move logs to Glacier after 90 days).

**Q: Q4. What is the difference between vertical scaling and horizontal scaling? Which does AWS favour?**

A: **Vertical scaling (Scale Up)** — make the existing server bigger. t3.micro → t3.large → m5.xlarge. Has limits (you can't make one server infinitely large). Requires downtime to restart with new instance type.
**Horizontal scaling (Scale Out)** — add more servers. 1 EC2 → 5 EC2 instances behind a load balancer. No limit. No downtime. This is what Auto Scaling does.
AWS **strongly favours horizontal scaling**. Almost all AWS managed services (ALB, ECS, Lambda, DynamoDB) scale horizontally by design.
Design applications to be **stateless** (no session data stored on individual servers) so any instance can handle any request — enabling horizontal scaling.

**Q: Q5. What is the AWS Shared Responsibility Model?**

A: AWS is responsible for security **OF the cloud**: physical data centres, hardware, hypervisors, the global network infrastructure, and the underlying software for managed services.
You (the customer) are responsible for security **IN the cloud**: your IAM users and roles, security group rules, encryption of your data at rest and in transit, OS patching on your EC2 instances, application security, and access controls.
For managed services: AWS takes more responsibility. For RDS, AWS patches the database engine. For EC2, you patch the OS.
Common exam and interview question: "Who patches the OS on an EC2 instance?" → You do. "Who patches the MySQL version on RDS?" → AWS does.

**Q: Q6. What is an Elastic IP and when do you need one?**

A: When you start an EC2 instance, it gets a public IP automatically. When you **stop and restart** the instance, this IP changes — a new one is assigned.
An **Elastic IP (EIP)** is a static public IP address that belongs to your AWS account. You associate it with an EC2 instance. It stays the same even when you stop and restart the instance.
Use Elastic IP when you need a fixed IP: for DNS A records, for whitelisting in external firewalls, or for a bastion/jump server.
EIPs are **free while associated with a running instance**. You are charged per hour when an EIP is allocated but NOT associated with a running instance — to prevent IP hoarding.
In modern architectures, you typically don't need EIPs because you use an ALB with a DNS name instead of pointing to individual EC2 IPs.

**Quiz: Quiz 1 — ShopNest's RDS database is configured with --no-publicly-accessible. What does this mean?**

- A) The database is turned off and cannot be accessed
- B) Only the AWS account owner can access it
- C) The database has no public IP address and can only be accessed from within the VPC — not from the internet
- D) The database requires a special password to access

> **Answer/explanation:** ✅ Answer: **C — Only accessible from within the VPC**
RDS with no public accessibility has only a private IP in the VPC (e.g. 10.0.3.45).
EC2 instances in the same VPC can connect because they share the same private network.
An attacker on the internet cannot reach the database at all — there is no route.
This is the correct production configuration — **databases should never have public IPs.**
You would access it for administration from a bastion host (a small EC2 in the public subnet that you SSH into first).

**Quiz: Quiz 2 — ShopNest's Auto Scaling Group has min=2, max=10, desired=2. Traffic spikes and CPU hits 85% (above the 60% target). What happens?**

- A) The existing 2 instances are made larger (vertical scaling)
- B) The Auto Scaling Group launches new instances (up to max=10) and registers them with the ALB
- C) An alarm fires but nothing changes until you manually approve
- D) The oldest instance is terminated and a new one launched

> **Answer/explanation:** ✅ Answer: **B — New instances are launched automatically**
The Target Tracking Policy detects that average CPU (85%) exceeds the target (60%).
ASG calculates how many instances are needed: roughly 85/60 × 2 ≈ 3 instances needed.
ASG launches 1 additional instance (desired goes from 2 to 3).
The new instance runs User Data to install the app, passes the health check, and ALB starts sending it traffic — all automatically.
This entire process takes 2–3 minutes from CPU spike to new instance serving traffic.

**Quiz: Quiz 3 — A developer accidentally committed AWS access keys to a public GitHub repo. What are the first two things to do?**

- A) Delete the GitHub repo and hope nobody saw it
- B) Immediately deactivate/delete the IAM access keys in AWS Console, then check CloudTrail for any unauthorized API calls made with those keys
- C) Change the AWS account root password
- D) Create a new IAM user to replace the old one

> **Answer/explanation:** ✅ Answer: **B — Deactivate keys immediately, then audit with CloudTrail**
Bots scan GitHub continuously for leaked credentials. Within **minutes** of a key being committed, attackers find it and start using it (typically to mine cryptocurrency on your bill).
**Step 1:** Go to IAM → Users → Security credentials → Deactivate and then delete the access key. This immediately stops any further misuse.
**Step 2:** Check AWS CloudTrail for any API calls made with those keys in the last 24 hours. Look for CreateInstance, LaunchSpot, CreateUser calls from unusual regions.
**Step 3:** Check your AWS bill for unexpected charges.
Deleting from GitHub history is important but it comes after securing the account — GitHub's history is public and bots cache it immediately.

> **AWS Project — Core Takeaways for Freshers**

> - **IAM first, always.** Secure your AWS account before creating any other resource. Enable MFA on root. Create IAM users. Use IAM roles for services. Never put access keys on EC2 instances.
> - **Deploy in multiple AZs** — always put your application in at least 2 Availability Zones. If one AZ goes down (which has happened to AWS before), your application stays up in the other.
> - **Databases go in private subnets** — never give your database a public IP. Only EC2 instances in the same VPC should be able to reach it, via Security Group rules.
> - **Never store files on EC2 disk** for anything important — when an instance is replaced, the local disk is gone. Use S3 for files, EFS for shared filesystems, RDS for database state.
> - **Use tags on everything** — every EC2, S3 bucket, RDS, security group, subnet. Without tags, you cannot tell which resources belong to which application, and your bill becomes incomprehensible.
> - **Security Groups are your first line of defence** — open only the ports that are needed. Port 22 (SSH) should only be open to your IP, not 0.0.0.0/0. Databases should only be open to your application's security group.
> - **Design for failure** — assume any individual component will fail. Multiple AZs, multiple EC2 instances behind ALB, Multi-AZ RDS, S3 for persistent storage. This is AWS's Well-Architected Framework principle.
> - **Monitor and set alarms** — a CloudWatch alarm that emails you when CPU hits 80% or disk is 90% full costs nothing. Not knowing about a problem until a customer calls you costs thousands.

##### AWS Architecture Standards — ShopNest Engineering Rules

- Every AWS resource must have at minimum these 4 tags: Name, Environment (dev/staging/prod), Project (shopnest), Owner (team or person)
- Use different AWS accounts for prod and non-prod environments — AWS Organizations makes this free and easy. A mistake in dev can never affect prod if they're in separate accounts
- Enable CloudTrail in all regions — it logs every API call to your account. Free for management events. Required for security audits and incident response
- Enable S3 versioning and object lock for any bucket containing backups, compliance data, or production assets — prevents accidental deletion
- Use AWS Secrets Manager or Parameter Store for database passwords and API keys — never pass them as EC2 User Data or environment variables in plain text
- Set up AWS Budgets with email alerts at 80% and 100% of monthly budget — cloud costs can spiral unexpectedly, especially with Auto Scaling. Know before the bill arrives

##### 🏋️ Hands-On Exercises — Extend ShopNest on AWS

1. **Add HTTPS with ACM and ALB:** Request a free SSL certificate from AWS Certificate Manager (ACM) for your domain. Add an HTTPS listener (port 443) to the ALB and attach the certificate. Add a redirect rule: all HTTP (port 80) requests redirect to HTTPS. Test that accessing http://shopnest.in redirects to https://shopnest.in. This is mandatory for any production website.
2. **Set up CloudFront for S3 images:** Create a CloudFront distribution that uses your S3 bucket as the origin. Configure a cache policy to cache product images for 24 hours. Update the application to serve images from the CloudFront URL (d1abc123.cloudfront.net) instead of directly from S3. Measure the latency difference: a user in Delhi loading images from the Mumbai CloudFront edge vs from S3.
3. **Implement S3 lifecycle policy:** Write a lifecycle policy JSON that: moves objects in the /logs/ prefix to S3 Standard-IA after 30 days, moves them to Glacier after 90 days, and expires (deletes) them after 365 days. Apply it to the shopnest bucket and verify in the console. This directly reduces storage costs without any application change.
4. **Set up a Bastion Host:** Create a small t3.micro EC2 instance in a public subnet with SSH access restricted to your IP only (port 22 from your IP/32). Configure the RDS security group to allow port 3306 only from the bastion host's security group. Connect to the database via: SSH to bastion → MySQL from bastion. This is the standard way to securely administer a private RDS instance.
5. **Build a cost monitoring dashboard:** Enable AWS Cost Explorer. Create a CloudWatch billing alarm that fires when estimated charges exceed ₹500. Create an AWS Budget for ₹2,000/month with email alerts at 80% and 100%. Enable Cost Allocation Tags and map all your resources. Take a screenshot of the Cost Explorer view showing cost breakdown by service.

### AWS Project Complete 🎉

You have built ShopNest's complete production AWS infrastructure — IAM with least-privilege policies, a VPC with public and private subnets, EC2 instances with User Data, S3 for product images, RDS MySQL in a private subnet, an Application Load Balancer, Auto Scaling from 2 to 10 instances, and CloudWatch alarms. This is the architecture used by real e-commerce companies every day.

> **Nandita**
> 
> "Karthik, this year's Diwali sale: 68,000 peak concurrent users. Auto Scaling launched us from 2 to 9 EC2 instances in under 4 minutes. The ALB distributed the load perfectly. Zero downtime. Zero manual intervention. Last year we had a 4-hour outage and lost 18 lakh rupees. This year, the infrastructure handled it without a single page on Slack from the ops team. That is what you built."

> **Suresh**
> 
> "The IAM structure you designed means our developers can do their work without touching production. The RDS is in a private subnet — completely unreachable from the internet. S3 versioning has already saved us twice when a developer accidentally overwrote product images. These aren't just technical configurations — they are what separates a company that survives a security incident from one that doesn't."

> **Next: Advanced AWS — Serverless, Containers & DevOps Services**

> - Lambda — run code triggered by events (S3 uploads, API calls, schedule) without managing servers. Pay per execution, not per hour
> - API Gateway — create REST and WebSocket APIs backed by Lambda or EC2 with built-in rate limiting, auth, and monitoring
> - ECS/EKS — run Docker containers at scale on AWS. ECS is simpler; EKS gives full Kubernetes
> - DynamoDB — AWS's fully managed NoSQL database. Scales to millions of requests per second with single-digit millisecond latency
> - CodePipeline + CodeDeploy — AWS-native CI/CD. Push to Git → automatically test → deploy to EC2 or ECS
> - Systems Manager (SSM) — connect to EC2 instances without SSH keys, patch fleets of instances, run commands at scale
> - AWS Well-Architected Framework — the five pillars: operational excellence, security, reliability, performance efficiency, cost optimisation
