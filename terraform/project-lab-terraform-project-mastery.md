# Terraform Project Mastery

> **👋 Hey Fresher — Read This First!**

> Terraform is the tool that every cloud company uses to build their infrastructure — instead of clicking in the AWS console, you write code that creates servers, databases, networks, and security rules automatically. You write it once, and Terraform creates the same infrastructure perfectly every time. This document uses **short code blocks** — each one does just one thing — followed by a plain-English explanation of exactly what it does and why a company uses it. No 50-line blocks to get lost in. One concept at a time.

> **Company in this project:** CloudPulse — a growing SaaS company in Chennai that is moving from manual AWS console setup to fully automated infrastructure. You just joined as a Junior DevOps/Cloud Engineer. Your lead is Bhavana. Let's build their AWS infrastructure from scratch using Terraform.

#### What You Will Build in This Project

You will write Terraform code that creates a real, production-grade AWS environment for CloudPulse — starting from nothing and building up piece by piece, each step explained clearly.

AWS VPC & Networking, EC2 Web Servers, S3 Storage, RDS Database, IAM Roles, Modules, Remote State, Workspaces

> **🏗️ Phase 1 — Foundations**

> Install Terraform, understand HCL syntax, providers, resources, variables, outputs, and the plan/apply workflow.

> Create a VPC with public and private subnets, internet gateway, route tables, and security groups — the network foundation of every AWS environment.

> Launch EC2 instances, create an S3 bucket for application assets, and attach IAM roles so EC2 can access S3 securely.

> Create an RDS PostgreSQL database in a private subnet, then refactor all resources into reusable modules that work across dev, staging, and production.

> Move Terraform state to S3 with DynamoDB locking, use workspaces for multiple environments, and set up a complete CI/CD pipeline for infrastructure.

**Scene 1 — CloudPulse Engineering Office, Chennai | Why Terraform?**

> **Bhavana** _Senior DevOps Engineer — CloudPulse_
> 
> Kiran, I just spent three hours in the AWS console recreating our staging environment because someone accidentally deleted it. No documentation, no record of what was there. Three hours of clicking. We are moving to Terraform this week. Every AWS resource we create will be code from now on. Infrastructure as Code means: the code IS the documentation, the code IS the audit trail, and the code IS how we rebuild anything in minutes instead of hours.

> **Kiran (You)** _Junior DevOps Engineer — Day 1 at CloudPulse_
> 
> I have heard of Terraform but I always thought cloud infrastructure meant using the AWS console. What exactly does Terraform code look like?

> **Bhavana** _Senior DevOps Engineer_
> 
> Terraform code reads almost like English. You write "I want an EC2 instance of type t3.micro in Mumbai region with this AMI" — and Terraform talks to AWS and creates it. You write "I want an S3 bucket named cloudpulse-assets" — Terraform creates it. The magic is: you can do the same for 50 servers, 10 databases, and 20 buckets — all in one command. And if you need a second, identical environment — run the same code again. Perfect copy every time.

### 1. Phase 1 — Terraform Foundations

Before writing any infrastructure code, understand the four core concepts that everything in Terraform is built on: HCL syntax, providers, resources, and the plan/apply workflow.

> **The Big Picture — How Terraform Works (Simple Explanation)**

> You write Terraform code in `.tf` files describing what infrastructure you want. You run `terraform plan` — it reads your code and tells you exactly what it will create, change, or delete (like a preview). Then you run `terraform apply` — it talks to AWS (or Azure, GCP, etc.) and actually creates the infrastructure. Terraform remembers what it created in a **state file**. If you change your code and run apply again, Terraform only changes what's different — not everything.

```
CloudPulse Terraform Project Structure
========================================

  cloudpulse-infra/
  ├── main.tf           ← Main resources (VPC, EC2, RDS etc)
  ├── variables.tf      ← Input variables (region, instance type...)
  ├── outputs.tf        ← Output values (IP addresses, URLs...)
  ├── providers.tf      ← Which cloud to connect to (AWS, Azure...)
  ├── terraform.tfvars  ← Actual values for variables
  ├── modules/
  │   ├── vpc/          ← Reusable VPC module
  │   ├── ec2/          ← Reusable EC2 module
  │   └── rds/          ← Reusable RDS module
  └── environments/
      ├── dev/          ← Dev environment config
      ├── staging/      ← Staging environment config
      └── prod/         ← Production environment config

  Key rule: ONE folder = ONE environment
  Each environment has its own terraform.tfvars with its own values
```

#### 1.1 Installation and First Commands

1. Install Terraform and verify
Download from terraform.io. On Linux, use package manager.

```bash
# Install on Ubuntu/Debian
sudo apt-get install terraform

# Verify installation
terraform version
```

> This installs Terraform globally. After install, `terraform version` should print something like **Terraform v1.7.0**. If you see this, Terraform is ready to use.

2. Configure AWS credentials
Terraform needs your AWS keys so it can talk to AWS on your behalf.

```
# Configure AWS CLI credentials
aws configure

# Enter when prompted:
AWS Access Key ID: AKIA...
AWS Secret Access Key: wJalr...
Default region name: ap-south-1
Default output format: json
```

> This stores your AWS credentials in `~/.aws/credentials`. Terraform automatically reads from there — you never paste credentials into Terraform code. **Never commit credentials to Git.** Use IAM roles in production instead.

#### 1.2 The Provider Block

The provider block tells Terraform which cloud platform to use and which region to create resources in.

```bash
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}
```

**📖 What This Does**

**terraform block:** Tells Terraform "download the AWS plugin from HashiCorp, version 5.x". This plugin is what lets Terraform talk to AWS APIs.  
  
**provider "aws":** Tells the AWS plugin which region (Mumbai = ap-south-1) to create resources in. Every resource you create will go into this region unless you say otherwise.

#### 1.3 Your First Resource Block

A resource block tells Terraform to create something — an EC2 instance, an S3 bucket, a VPC, anything. This is the most important block in Terraform.

```
resource "aws_instance" "web" {
  ami           = "ami-0f5ee92e2d63afc18"
  instance_type = "t3.micro"

  tags = {
    Name = "cloudpulse-web"
  }
}
```

**📖 What This Does**

**"aws_instance"** = the type of AWS resource (EC2 virtual machine).  
  
**"web"** = your local name for this resource — how you refer to it inside Terraform code.  
  
**ami** = which operating system image to use (this is Ubuntu 22.04 Mumbai).  
  
**instance_type** = how powerful the server is. t3.micro = 2 vCPU, 1GB RAM. Free tier eligible.

#### 1.4 The Plan / Apply / Destroy Workflow

These three commands are the entire Terraform workflow. You will use them hundreds of times.

```bash
# Step 1: Download the AWS provider plugin (run once per project)
terraform init
```

> **terraform init** downloads the AWS provider plugin into a `.terraform/` folder. Run this once when starting a project or after adding a new provider. Nothing gets created in AWS yet.

```bash
# Step 2: Preview what Terraform will create/change/delete
terraform plan
```

> **terraform plan** reads your .tf files and shows you exactly what it will do — how many resources will be added, changed, or destroyed. A `+` means create, `~` means update, `-` means delete. **Always read the plan before applying.** This is your safety net.

```bash
# Step 3: Actually create the resources in AWS
terraform apply
```

> **terraform apply** shows the plan again and asks you to type `yes` to confirm. After you confirm, it talks to AWS and creates everything. It takes 1–5 minutes for most resources. When done, it shows you the outputs.

```bash
# Destroy ALL resources (careful — deletes everything!)
terraform destroy
```

> **terraform destroy** deletes every resource that Terraform created. Use this to clean up a dev environment to save money. It will ask you to type `yes` to confirm. In production, you almost never run this — use targeted destroy instead.

```
Plan: 1 to add, 0 to change, 0 to destroy.

aws_instance.web: Creating...
aws_instance.web: Still creating... [10s elapsed]
aws_instance.web: Creation complete after 32s [id=i-0abc123def456]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

#### 1.5 Variables — Don't Hardcode Values

Variables let you write flexible code that works for different environments. Instead of hardcoding `"t3.micro"` everywhere, define a variable — then change it in one place for production.

```
# variables.tf — define the variable
variable "instance_type" {
  description = "EC2 instance size"
  type        = string
  default     = "t3.micro"
}
```

**📖 What This Does**

**variable block:** Creates a named variable called `instance_type`.  
  
**description:** Human-readable explanation — shown in --help output.  
  
**type:** Must be string, number, bool, list, or map.  
  
**default:** Used if no value is provided. Without a default, Terraform asks you to type the value when you run apply.

```
# Use the variable with var. prefix
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
}
```

**📖 What This Does**

**var.instance_type** = reads the value of the variable named `instance_type`.  
  
Now if you change the variable, it changes everywhere it's used — not just here. This is the same idea as variables in any programming language.

```
# terraform.tfvars — set actual values
instance_type = "t3.small"
ami_id        = "ami-0f5ee92e2d63afc18"
region        = "ap-south-1"
```

**📖 What This Does**

**terraform.tfvars** is where you set real values for your variables. Terraform automatically reads this file. For production vs staging, you'd have different `.tfvars` files with different values — same code, different environments. **Never commit secrets (passwords, keys) in .tfvars to Git.**

#### 1.6 Outputs — See What Terraform Created

```
# outputs.tf
output "server_ip" {
  description = "Public IP of web server"
  value = aws_instance.web.public_ip
}
```

**📖 What This Does**

After `terraform apply`, outputs are printed to the terminal. Here, `aws_instance.web.public_ip` reads the public IP address that AWS assigned to the EC2 instance we created. Use outputs to see important values like IP addresses, database endpoints, and S3 URLs after Terraform finishes creating resources.

> **💡 Fresher Tip — terraform.tfstate — Never Edit This File!**

> When Terraform creates resources, it saves a record of everything in a file called `terraform.tfstate`. This is Terraform's memory — it knows what exists in AWS so it can figure out what needs to change next time. **Never edit, delete, or commit this file to Git** — if it gets corrupted or out of sync with real AWS, Terraform gets confused. In Phase 5, we move this file to S3 (called "remote state") so the whole team shares one source of truth.

### 2. Phase 2 — VPC and Networking

**Business Problem:** CloudPulse's EC2 servers need to be on a private network that they control — not on a default shared AWS network. The web servers need to be reachable from the internet (public subnet), but the database must not be directly reachable from the internet (private subnet). Terraform builds this network in minutes.

**Scene 2 — Architecture Review, CloudPulse | Building the Network**

> **Bhavana** _Senior DevOps Engineer_
> 
> Kiran, think of a VPC like a private apartment building. The building has its own walls, its own internal corridors. Public subnets are like the lobby — accessible from outside. Private subnets are like the apartments — only accessible from inside the building. Our web servers go in the lobby. Our database goes in an apartment. The internet gateway is the building's front door.

> **Prabhath** _Cloud Architect — CloudPulse_
> 
> And every piece — the VPC, subnets, internet gateway, route tables — is a separate Terraform resource block. They reference each other by their Terraform names. Terraform figures out the order to create them based on those references. You don't have to worry about "create the VPC before the subnet" — Terraform handles the dependency order automatically.

```
CloudPulse AWS Network Architecture
======================================

  VPC: 10.0.0.0/16  (our private network space)
  │
  ├── Public Subnet: 10.0.1.0/24  (AZ: ap-south-1a)
  │    └── EC2 Web Servers ← accessible from internet
  │    └── Internet Gateway attached → traffic can flow in/out
  │
  ├── Private Subnet: 10.0.2.0/24  (AZ: ap-south-1b)
  │    └── RDS Database ← NOT accessible from internet
  │    └── Only EC2 in public subnet can talk to it
  │
  └── Security Groups (virtual firewalls)
       ├── web-sg: allow port 80, 443 from internet
       └── db-sg: allow port 5432 only from web-sg
```

#### 2.1 Create the VPC

```
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "cloudpulse-vpc"
  }
}
```

**📖 What This Does**

Creates a VPC — a private, isolated network section in AWS.  
  
**cidr_block = "10.0.0.0/16"** defines the IP address range. The `/16` means we have 65,536 private IP addresses available (10.0.0.0 through 10.0.255.255) to assign to subnets, EC2s, and databases inside this VPC.

#### 2.2 Create Public and Private Subnets

```
resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-south-1a"

  map_public_ip_on_launch = true
}
```

**📖 What This Does**

**vpc_id = aws_vpc.main.id** — this is how Terraform connects resources. It reads the ID of the VPC we just created and puts this subnet inside it. Terraform automatically creates the VPC first because this line creates a dependency.  
  
**map_public_ip_on_launch = true** means any EC2 launched in this subnet automatically gets a public IP address.

```
resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "ap-south-1b"
}
```

**📖 What This Does**

Creates a private subnet in a different availability zone (1b instead of 1a). No `map_public_ip_on_launch` here — resources placed here have no public IP and cannot be reached directly from the internet. This is where the database will live.

#### 2.3 Internet Gateway and Route Table

```
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}
```

**📖 What This Does**

Creates an Internet Gateway and attaches it to our VPC. Without this, nothing in the VPC can communicate with the public internet. Think of it as the front door of the building. Just creating it isn't enough — we also need to create a route that tells the subnet to use it.

```
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}
```

**📖 What This Does**

Creates a routing rule: **"any traffic going to 0.0.0.0/0 (= any internet address) should go through the Internet Gateway."**  
  
This is like a signpost in the VPC corridor saying: "Internet traffic → use the front door (IGW)."

```
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

**📖 What This Does**

Links the route table to the public subnet. Without this association, the route exists but the subnet doesn't use it. Think of it as telling the public subnet: "use the route table that has the internet door." The private subnet gets no such association — it stays isolated.

#### 2.4 Security Groups — Firewall Rules

```
resource "aws_security_group" "web" {
  name   = "cloudpulse-web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**📖 What This Does**

**ingress** = incoming traffic rules.  
Allow port 80 (HTTP) from anywhere (0.0.0.0/0 = all IPs).  
  
**egress** = outgoing traffic rules.  
protocol = "-1" means all protocols. Allow all outbound traffic — EC2 can reach the internet to download packages, call APIs, etc.  
  
In production you'd also add port 443 for HTTPS and port 22 for SSH (restricted to your office IP only).

### 3. Phase 3 — EC2, S3 and IAM

**Business Problem:** CloudPulse needs a web server running in the public subnet, an S3 bucket to store uploaded files and static assets, and an IAM role that gives the EC2 server permission to access S3 — without hardcoding any AWS keys inside the server.

**Scene 3 — CloudPulse Standup | "The EC2 Can't Access S3"**

> **Snehal** _Backend Developer — CloudPulse_
> 
> The application on our EC2 instance needs to upload files to S3. The developer hardcoded AWS access keys inside the application. That's a security disaster — if those keys leak, someone can access our entire AWS account. Is there a better way?

> **Bhavana** _Senior DevOps Engineer_
> 
> Yes — IAM Roles. Instead of giving the EC2 a username and password (access key), we attach a Role to it. The role says "this EC2 is allowed to read and write to S3." AWS automatically provides temporary credentials to the EC2 that rotate every few hours. The application never sees or stores any keys. Kiran, add the IAM role to our Terraform code.

#### 3.1 Create the EC2 Instance

```
resource "aws_instance" "web" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2_profile.name

  tags = { Name = "cloudpulse-web" }
}
```

**📖 What This Does**

Creates an EC2 instance with all the pieces connected:  
  
• **subnet_id** = put it in the public subnet we created  
• **vpc_security_group_ids** = apply the web firewall rules  
• **iam_instance_profile** = attach the IAM role so EC2 can access S3 without keys  
  
Notice how we reference other resources by name — Terraform wires them together automatically.

#### 3.2 User Data — Run a Script on First Boot

```
resource "aws_instance" "web" {
  # ... other fields ...
  user_data = <<-EOF
    #!/bin/bash
    apt-get update -y
    apt-get install -y nginx
    systemctl start nginx
  EOF
}
```

**📖 What This Does**

**user_data** is a startup script that runs once when the EC2 boots for the first time.  
  
**<<-EOF ... EOF** is a multi-line string in Terraform (called a heredoc).  
  
This script installs nginx and starts it — so the web server is ready immediately after Terraform creates the EC2. No manual SSH needed.

#### 3.3 S3 Bucket for Application Assets

```
resource "aws_s3_bucket" "assets" {
  bucket = "cloudpulse-assets-2026"

  tags = { Name = "CloudPulse Assets" }
}
```

**📖 What This Does**

Creates an S3 bucket. S3 bucket names must be **globally unique** across all AWS accounts — that's why we add "2026" or a random suffix. If the name is taken, the apply fails with a bucket name conflict error. In production, use a variable or random suffix generated by Terraform's `random` provider.

```
resource "aws_s3_bucket_versioning" "assets" {
  bucket = aws_s3_bucket.assets.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

**📖 What This Does**

Enables versioning on the bucket — every time a file is overwritten, S3 keeps the old version. If someone accidentally deletes or overwrites an important file, you can restore the previous version. This is a separate resource block — Terraform applies it after creating the bucket (because it references the bucket ID).

#### 3.4 IAM Role — Secure EC2 Access to S3

IAM roles have two parts: a **trust policy** (who can use this role) and a **permission policy** (what this role can do). We create both as separate blocks.

```
resource "aws_iam_role" "ec2_role" {
  name = "cloudpulse-ec2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}
```

**📖 What This Does**

Creates an IAM Role with a **trust policy** that says: *"EC2 instances (ec2.amazonaws.com) are allowed to assume/use this role."*  
  
**jsonencode()** converts the HCL map into the JSON format that AWS requires for policies. This is the "who can use this role" part.

```
resource "aws_iam_role_policy" "s3_access" {
  name = "s3-read-write"
  role = aws_iam_role.ec2_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "${aws_s3_bucket.assets.arn}/*"
    }]
  })
}
```

**📖 What This Does**

Attaches a permissions policy to the role saying: *"Allow reading (GetObject) and writing (PutObject) to any object in the cloudpulse-assets bucket."*  
  
**${aws_s3_bucket.assets.arn}** — this uses string interpolation to insert the bucket's ARN (Amazon Resource Name, its unique identifier) dynamically. No hardcoding needed.

```
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "cloudpulse-ec2-profile"
  role = aws_iam_role.ec2_role.name
}
```

**📖 What This Does**

An instance profile is a container that holds an IAM Role so it can be attached to an EC2 instance. You can't attach a role directly to EC2 — it must be wrapped in an instance profile first. This profile is what we referenced in `iam_instance_profile` in the EC2 resource block above.

### 4. Phase 4 — RDS Database and Reusable Modules

**Business Problem:** CloudPulse's application needs a PostgreSQL database in the private subnet. And currently all the Terraform code is in one big main.tf file — as the infrastructure grows, this becomes unmaintainable. The solution: modules — reusable packages of Terraform code, like functions in programming.

**Scene 4 — Code Review, CloudPulse | The Tangled main.tf**

> **Prabhath** _Cloud Architect — CloudPulse_
> 
> Kiran, our main.tf is already 200 lines and we only have one environment. When we add staging and production, the team will copy-paste all of this code three times. Any bug fix will need to be applied in three places. The fix is modules. A module is a folder of .tf files that does one job — like "create a VPC" or "create an EC2". You call it like a function and pass in variables. One fix in the module fixes all environments.

#### 4.1 RDS PostgreSQL Instance

```
resource "aws_db_subnet_group" "main" {
  name       = "cloudpulse-db-subnets"
  subnet_ids = [aws_subnet.private.id]
}
```

**📖 What This Does**

RDS (Relational Database Service) requires a subnet group — a list of subnets where it can place the database. We point it to the private subnet so the database has no public IP and cannot be reached from the internet. For production, you'd include subnets in at least two availability zones for high availability.

```
resource "aws_db_instance" "postgres" {
  identifier        = "cloudpulse-db"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.micro"
  allocated_storage = 20

  db_name  = "cloudpulse"
  username = "dbadmin"
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]
  skip_final_snapshot    = true
}
```

**📖 What This Does**

Creates a managed PostgreSQL 15.4 database.  
  
• **instance_class = "db.t3.micro"** — smallest available, good for dev  
• **allocated_storage = 20** — 20 GB disk  
• **password = var.db_password** — read from a variable, never hardcoded  
• **skip_final_snapshot = true** — for dev only; in production set to false so a backup is taken before any deletion

#### 4.2 Building a Reusable Module

A module is just a folder with .tf files. You call it with a `module` block and pass in values. This is how professionals write Terraform — not one giant file.

```
# modules/vpc/main.tf — the module definition
variable "vpc_cidr" { type = string }
variable "environment" { type = string }

resource "aws_vpc" "this" {
  cidr_block = var.vpc_cidr
  tags = { Name = "${var.environment}-vpc" }
}
```

**📖 What This Does**

This is the module file inside `modules/vpc/`. It accepts variables as inputs (`vpc_cidr`, `environment`) and creates a VPC with a name that includes the environment — like "dev-vpc" or "prod-vpc".  
  
Notice we use `"this"` as the resource name inside modules — a convention meaning "the main resource of this module".

```
# modules/vpc/outputs.tf — expose values to callers
output "vpc_id" {
  value = aws_vpc.this.id
}
```

**📖 What This Does**

Modules expose values through outputs — just like a function returns a value. Here the module exports the VPC's ID so the caller (main.tf) can reference it when creating subnets. Without this output, the caller can't access anything created inside the module.

```
# main.tf — calling the module
module "vpc" {
  source      = "./modules/vpc"
  vpc_cidr    = "10.0.0.0/16"
  environment = "dev"
}

# Use the module's output
resource "aws_subnet" "public" {
  vpc_id = module.vpc.vpc_id
}
```

**📖 What This Does**

**module "vpc"** calls our vpc module, passing in values for its variables.  
  
**source = "./modules/vpc"** tells Terraform where to find the module folder.  
  
**module.vpc.vpc_id** reads the output from the module — the VPC ID that was created. This is how resources outside the module connect to resources inside it.

### 5. Phase 5 — Remote State, Workspaces and Teams

**Business Problem:** The team has three engineers all running `terraform apply` on their laptops. Each has their own local state file. When two people apply at the same time, the state files conflict and infrastructure gets broken. Solution: store state in S3 (shared) and lock it with DynamoDB (so only one person can apply at a time).

**Scene 5 — Incident Review, CloudPulse | The State Conflict**

> **Bhavana** _Senior DevOps Engineer_
> 
> Kiran, do you know what happened yesterday? You and Prabhath both ran terraform apply at the same time. Your local state files were out of sync. Terraform tried to create resources that already existed, got confused, and corrupted the state. We spent two hours recovering it. From today: remote state in S3, state locking in DynamoDB. No exceptions.

#### 5.1 S3 Bucket and DynamoDB for State

First, create the S3 bucket and DynamoDB table that will hold the state. This is a bootstrapping step — done once, manually.

```
resource "aws_s3_bucket" "tf_state" {
  bucket = "cloudpulse-tfstate-2026"
}

resource "aws_s3_bucket_versioning" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  versioning_configuration { status = "Enabled" }
}
```

**📖 What This Does**

Creates an S3 bucket to store the Terraform state file. Versioning is enabled — if the state file gets corrupted, you can restore a previous version. This bucket must exist before you configure the backend, so it's created separately first (often via the AWS console or a tiny bootstrap Terraform config).

```
resource "aws_dynamodb_table" "tf_lock" {
  name         = "cloudpulse-tf-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**📖 What This Does**

Creates a DynamoDB table for state locking. When someone runs `terraform apply`, Terraform writes a lock record to this table. If another person tries to apply at the same time, Terraform sees the lock and refuses — "Error: Error locking state: ConditionalCheckFailedException." This prevents two people from corrupting the state simultaneously.

#### 5.2 Configure Remote Backend

```bash
# providers.tf — add backend configuration
terraform {
  backend "s3" {
    bucket         = "cloudpulse-tfstate-2026"
    key            = "dev/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "cloudpulse-tf-locks"
    encrypt        = true
  }
}
```

**📖 What This Does**

Tells Terraform: *"Store my state file in S3 at key 'dev/terraform.tfstate', and use DynamoDB for locking."*  
  
**key** is the path inside the S3 bucket — different environments use different keys (dev/ staging/ prod/) so their state files don't overwrite each other.  
  
**encrypt = true** encrypts the state file at rest — important because state files can contain database passwords and other secrets.

```bash
# After adding the backend block, re-initialise to migrate local state to S3
terraform init -migrate-state
```

> **terraform init -migrate-state** reads the local `terraform.tfstate` file and uploads it to S3. After this, all apply runs use the S3 state. All team members run `terraform init` (without -migrate-state) to connect to the shared backend.

#### 5.3 Workspaces — Multiple Environments, One Codebase

```bash
# List all workspaces
terraform workspace list

# Create dev and prod workspaces
terraform workspace new dev
terraform workspace new prod

# Switch to a workspace
terraform workspace select prod
```

**📖 What This Does**

Workspaces let you use the same Terraform code for different environments. Each workspace has its own state file in S3 (automatically at `env:/dev/terraform.tfstate` and `env:/prod/terraform.tfstate`).  
  
When you switch to prod and run apply, it reads prod's state — so it manages prod's resources, not dev's.

```
# Use workspace name in resources
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"

  tags = {
    Name        = "${terraform.workspace}-web"
    Environment = terraform.workspace
  }
}
```

**📖 What This Does**

**terraform.workspace** is a built-in variable that holds the current workspace name.  
  
The `? :` is a conditional expression (ternary operator) — if workspace is "prod" use t3.large, otherwise use t3.micro. This lets the same code create small instances for dev and big instances for production automatically.

### 6. Essential Terraform Commands Reference

Command

What It Does

When to Use

terraform init

Download providers, set up backend

First time, after adding providers/modules

terraform plan

Preview changes — nothing gets created

Before every apply

terraform apply

Create/update infrastructure in AWS

After reviewing the plan

terraform apply -auto-approve

Apply without typing "yes" (for CI/CD)

Automated pipelines only

terraform destroy

Delete all resources in state

Cleanup dev/test environments

terraform destroy -target=aws_instance.web

Delete only one specific resource

When you want to recreate just one thing

terraform fmt

Auto-format all .tf files to standard style

Before every commit

terraform validate

Check syntax — catch typos before plan

After writing new code

terraform state list

Show all resources Terraform is tracking

Check what's in state

terraform state show aws_instance.web

Show all details of one resource in state

Debugging, inspecting resource attributes

terraform import aws_instance.web i-0abc123

Import an existing AWS resource into state

When a resource was created manually and needs to be managed by Terraform

terraform output

Print all output values

See IPs, URLs after apply

terraform workspace list/new/select

Manage environment workspaces

Switching between dev, staging, prod

terraform graph | dot -Tpng

Generate a visual dependency graph

Understanding resource relationships

### 7. Interview Questions — Terraform

##### Interview Q&A — Fresher Level (0–1 Year Terraform Experience)

**Q: Q1. What is Terraform and how is it different from using the AWS console?**

A: Terraform is an Infrastructure as Code (IaC) tool that lets you define cloud infrastructure in configuration files using HCL (HashiCorp Configuration Language). With the AWS console, you click to create resources — but that's not repeatable, not versioned, and not auditable. With Terraform, you write code that describes your infrastructure, commit it to Git, review it like any other code, and apply it to create identical infrastructure every time. If someone accidentally deletes a resource, you re-run Terraform and it's recreated in minutes. The code is also the documentation — it shows exactly what infrastructure exists and how it's configured.

**Q: Q2. What are the three main Terraform commands and what does each one do?**

A: `terraform init` initialises the working directory — it downloads the required provider plugins (like the AWS plugin) and configures the backend where state is stored. You run this once when starting a project. `terraform plan` reads your .tf files, compares them against the current state, and shows you exactly what changes it will make — resources to add (+), modify (~), or delete (-). Nothing actually changes in AWS during plan. `terraform apply` executes the plan — it calls the AWS APIs to create, update, or delete resources. It shows the plan again and asks you to type "yes" before making any changes.

**Q: Q3. What is Terraform state and why is it important?**

A: Terraform state is a JSON file (terraform.tfstate) that records every resource Terraform has created — its ID, all its attributes, and the relationship between Terraform resource names and real AWS resource IDs. Terraform uses state to calculate what needs to change when you update your code — without state, it wouldn't know whether aws_instance.web already exists or needs to be created. In a team, state must be stored in a shared remote location (like S3) so everyone uses the same view of infrastructure. State files can contain sensitive data like database passwords, so they must be encrypted and never committed to Git.

**Q: Q4. What is the difference between a variable and a local in Terraform?**

A: A `variable` is an input parameter — its value comes from outside (terraform.tfvars, -var flag, environment variables, or user input at runtime). It's how callers customise the behaviour of your code. A `local` is a computed value defined inside your configuration — like a local variable in a function. It can reference variables and resource attributes to compute a derived value. For example: `local { environment_tag = "${var.project}-${var.env}" }`. Locals reduce repetition — define once, use everywhere. The key difference: variables are inputs from outside; locals are internal computations.

**Q: Q5. What is a Terraform module and when would you use one?**

A: A module is a folder of Terraform .tf files that encapsulates a set of related resources — like a reusable function. You use modules when the same infrastructure pattern needs to exist in multiple places or environments. Instead of copy-pasting VPC code three times (dev, staging, prod), you write it once as a module and call it three times with different input variables. Modules also enforce consistency — if the security policy for VPCs changes, you update the module and all environments get the fix. You call a module using the `module` block with a `source` parameter pointing to the module folder. Public modules are also available in the Terraform Registry (registry.terraform.io).

**Q: Q6. What happens if two engineers run terraform apply at the same time?**

A: Without state locking, both apply runs read the same state, make different changes, and write conflicting state files back — causing state corruption. Resources may be created twice, or one person's changes may overwrite the other's. The solution is state locking with DynamoDB: when a terraform apply starts, it writes a lock record to DynamoDB with the workspace ID. If another apply starts, it checks DynamoDB, sees the lock, and immediately fails with an error message telling you who holds the lock and when they started. Once the first apply completes, the lock is released and the second apply can proceed. Always use S3 + DynamoDB backend in team environments.

**Quiz: Quiz 1 — What does this line mean in a resource block? vpc_id = aws_vpc.main.id**

- A) It sets the vpc_id variable to a string "aws_vpc.main.id"
- B) It reads the id attribute of the resource named "main" of type aws_vpc, creating a dependency
- C) It imports an existing VPC from AWS
- D) It creates a new VPC named "main"

> **Answer/explanation:** ✅ Answer: B. `aws_vpc.main.id` is a resource reference — it reads the `id` attribute from the resource block `resource "aws_vpc" "main"`. Terraform automatically knows it must create the VPC before this resource (since this resource needs the VPC's ID). This is how Terraform builds its dependency graph — by following these references. It will never try to create the subnet before the VPC because the subnet's vpc_id references the VPC's output.

**Quiz: Quiz 2 — Why should you never commit terraform.tfstate to Git?**

- A) Because it is too large for Git to handle
- B) Because it contains sensitive data like passwords and keys in plain text, and becomes out of sync in team environments causing corruption
- C) Because Terraform deletes it after apply
- D) Because the .gitignore automatically excludes it

> **Answer/explanation:** ✅ Answer: B. State files contain database passwords, IAM key IDs, and other sensitive resource attributes in plain text JSON — committing them to Git exposes secrets to everyone with repo access. Additionally, in a team, if two engineers commit different state files, they become out of sync — Terraform thinks resources exist that don't, or vice versa, causing failed applies and infrastructure drift. The correct approach: store state in S3 with encryption and use DynamoDB locking. Add `*.tfstate` and `*.tfstate.backup` to your `.gitignore`.

**Quiz: Quiz 3 — What does terraform plan show and why should you always read it?**

- A) A list of all resources that currently exist in AWS
- B) The cost of running the infrastructure per month
- C) Exactly what will be created (+), modified (~), or deleted (-) if you run apply — before anything changes in AWS
- D) The syntax errors in your .tf files

> **Answer/explanation:** ✅ Answer: C. terraform plan is your safety net — it shows the precise diff between what your code says should exist and what currently exists (according to state). A + means a new resource will be created. A ~ means an existing resource will be modified in-place. A - means a resource will be deleted. A -/+ means a resource will be destroyed and recreated (common with name changes). **Always check for unexpected deletions** in the plan — a single misconfigured variable can cause Terraform to propose destroying a production database. Read every plan before typing yes.

> **Terraform Project — Core Takeaways for Freshers**

> - Infrastructure as Code means your infrastructure is version-controlled, reviewable, and reproducible — the same code creates identical environments every time. This is the single most valuable thing Terraform gives a company.
> - Always run terraform plan before terraform apply — read every + and - carefully. A plan that shows unexpected deletions is a sign something is wrong with your code or variables.
> - Resource references (aws_vpc.main.id) are how Terraform builds its dependency graph — you never need to manually specify order. Terraform figures out what to create first by following the references.
> - Remote state in S3 + DynamoDB locking is not optional in a team — local state files cause corruption when multiple people run apply. Set this up on day one of any team project.
> - Never hardcode passwords or access keys in .tf files or .tfvars files that go into Git. Use variables with no defaults for secrets, and pass them via environment variables (TF_VAR_db_password) or a secrets manager integration.
> - Modules are the key to maintainable Terraform at scale — one module definition, used by dev/staging/prod with different variable values. When the module changes, all environments get the fix automatically.
> - terraform fmt and terraform validate should run before every commit — format keeps the codebase consistent, validate catches syntax errors before they reach your team.
> - Use tags on every resource with at minimum Environment, Project, and Owner tags — without tags, a cloud bill of 50 resources is impossible to understand or attribute to the right team or feature.

##### Terraform Code Standards — CloudPulse Engineering Rules

- One resource type per file is a common convention — ec2.tf, vpc.tf, rds.tf, iam.tf. It makes it easier for teammates to find and modify specific infrastructure
- Always add a description to every variable — the description appears in --help output and makes it clear what value is expected, especially for newcomers to the codebase
- Use terraform.tfvars for non-sensitive defaults, and pass sensitive values via TF_VAR_ environment variables — never let passwords appear in files that could be committed to Git
- Tag every AWS resource with Environment, Project, Team, and ManagedBy=terraform tags — cost allocation and resource ownership becomes much simpler with consistent tagging
- Keep modules small and focused — a module that does one thing (create a VPC, create an RDS instance) is reusable. A module that creates everything is just a copy of main.tf and can't be reused
- Use data sources (data "aws_ami" "ubuntu") to look up dynamic values like the latest AMI ID — never hardcode AMI IDs which change per region and become stale
- Store the backend configuration key as environment/component — "prod/vpc/terraform.tfstate" — so state files are clearly organised by environment and component

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Add HTTPS to the web server:** Create an `aws_lb` (Application Load Balancer) in the public subnet, an `aws_lb_target_group` that contains the EC2 instance, and an `aws_lb_listener` on port 443. Use `aws_acm_certificate` for the SSL certificate. This is how every production web application on AWS handles HTTPS.
2. **Add auto-scaling:** Replace the single `aws_instance` with an `aws_launch_template` and an `aws_autoscaling_group` that maintains 2–5 EC2 instances. Configure a scaling policy that adds instances when CPU exceeds 70%. This is what "auto-scaling" means on AWS.
3. **Use a data source for AMI:** Replace the hardcoded `ami_id` variable with a `data "aws_ami" "ubuntu"` data source that automatically looks up the latest Ubuntu 22.04 AMI in your region. Your code will always use the latest version without manual updates.
4. **Create a complete staging environment:** Create an `environments/staging/` folder with its own `main.tf` that calls the same modules with different variable values — smaller instance types, shorter RDS backup retention, a different S3 bucket name. Run apply for both dev and staging and verify two separate environments exist in AWS.
5. **Add CloudWatch alarms:** Write an `aws_cloudwatch_metric_alarm` resource that triggers an `aws_sns_topic` notification when the EC2 CPU exceeds 90% for 5 minutes. Add your email as a subscriber to the SNS topic. You will receive an email alert whenever the server is under high load — infrastructure monitoring as code.

### Terraform Project Complete 🎉

You have built CloudPulse's complete AWS infrastructure from scratch — VPC, subnets, internet gateway, security groups, EC2, S3, IAM roles, RDS database, reusable modules, remote state with locking, and environment workspaces. Every resource is code, versioned, repeatable, and team-ready.

> **Bhavana**
> 
> "Kiran, when you joined, recreating our staging environment took three hours of clicking. Today you ran terraform apply and the entire network, servers, database, IAM roles, and S3 bucket were created in four minutes. That same code will create an identical production environment in another four minutes. That is the power of Infrastructure as Code — and it only exists because you wrote it."

> **Prabhath**
> 
> "And the modules you wrote are already being used for our client's environment. They called our vpc module with their CIDR block and their environment name, ran apply, and had a complete network in six minutes. You wrote infrastructure code that other teams are using. That is the professional standard."

> **Snehal**
> 
> "The IAM role you created means our application never stores AWS credentials anywhere. The EC2 gets temporary credentials automatically. Security reviewed the setup and called it textbook correct. That is exactly how AWS recommends it be done — and you implemented it in eight lines of Terraform."

> **Next: Advanced Terraform — Terraform Cloud, Sentinel Policies & Enterprise Patterns**

> - Terraform Cloud — managed remote state, run history, team access control, and VCS integration
> - Sentinel policies — write policies that block applies which violate security rules (e.g. "no public S3 buckets in production")
> - Dynamic blocks — generate repeated nested blocks (like multiple ingress rules) from a list variable instead of copy-pasting
> - for_each and count — create multiple similar resources from a map or number without duplicating resource blocks
> - Data sources — look up existing AWS resources (like the latest Ubuntu AMI or your organisation's existing VPC) without hardcoding IDs
> - Terraform with CI/CD — integrate plan and apply into GitHub Actions so every pull request shows the infrastructure diff as a comment
