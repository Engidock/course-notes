# 🏗️ Infrastructure as Code Project Mastery

### The InfraScale Project — What We're Building

**InfraScale** is a global SaaS analytics platform. Customers in North America, Europe, and Asia. Each region needs: VPC, subnets, RDS database, ElastiCache, ALB, Auto Scaling EC2 instances, S3 buckets with replication, CloudWatch monitoring.

**The Problem:** Setting up infrastructure manually on AWS Console: create VPC, configure subnets, create security groups, RDS, ALB, Auto Scaling — 50+ clicks per region. Error-prone. Documentation falls out of date. Scaling to 10 regions = 500+ manual steps. If a change is needed (e.g., increase ALB idle timeout), manually update all 10 regions. Nightmare.

**The Solution:** Terraform. Write infrastructure in code (HCL). Version control it in Git. Run `terraform apply` to create/update infrastructure across all regions simultaneously. Change a value in code, apply, done. Destroy environment: `terraform destroy`. Reproducible, auditable, version-controlled infrastructure.

**📍 Scene: InfraScale's Multi-Region Problem**

> **Vikram (Infrastructure Lead)**
> 
> "We launched in US East. Took 2 weeks to manually set up the AWS infrastructure. Now our CEO says: 'We need to launch in Europe and Asia simultaneously.' That's 3 more regions. Doing it manually would take 6 weeks and someone will make mistakes on the third region."

> **Priya (DevOps Lead)**
> 
> "We use Terraform. One HCL file defines: VPC, subnets, RDS, ALB, EC2, S3, security groups, everything. We add a `for_each` loop for regions. Now it's: us-east-1, eu-west-1, ap-southeast-1. Run terraform apply. All three regions built in 10 minutes. Identical, zero manual steps."

> **Vikram**
> 
> "And if we discover a bug in the infrastructure? Like security group is too permissive?"

> **Priya**
> 
> "Change one line in the Terraform file. Run `terraform plan` to see what will change. Get approval. Run `terraform apply`. Fixed in all 3 regions simultaneously. No manual work. Git history shows who changed what and when."

### 1. Terraform Fundamentals — Infrastructure in Code

#### What is Terraform?

#### Terraform Installation & First Steps

```bash
$ terraform version
Terraform v1.7.0
$ mkdir infra-project
$ cd infra-project
$ cat > main.tf << 'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
EOF
$ terraform init
```

> **terraform version:** verify Terraform is installed. v1.7.0 or higher recommended.
**main.tf:** main configuration file (or split into multiple files). HCL syntax.
**required_providers:** specifies AWS provider, version constraint (~> means >=5.0, <6.0).
**provider "aws":** configures AWS region. Can be overridden with variables or environment variables.
**terraform init:** downloads provider plugins, initializes .terraform directory, creates terraform.lock file (lock dependencies).

### 2. Terraform Core Concepts — Resources, Variables, Outputs

#### Creating Your First Resource — VPC

```
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "infra-vpc"
Environment = "production"
ManagedBy   = "terraform"
  }
}

resource "aws_subnet" "public_1a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
availability_zone      = "us-east-1a"
map_public_ip_on_launch = true

  tags = {
    Name = "public-1a"
  }
}
```

> **resource "aws_vpc" "main":** creates AWS VPC. Type "aws_vpc", name "main". Referenced later as aws_vpc.main.
**cidr_block:** IP range for VPC. 10.0.0.0/16 = 65,536 IPs (10.0.0.0 - 10.0.255.255).
**enable_dns_hostnames/support:** required for ECS, RDS, and other services to work properly.
**aws_vpc.main.id:** interpolation. Subnet references the VPC ID created above. Automatic dependency.
**tags:** important for cost allocation, automation, and filtering. ManagedBy: terraform indicates Terraform-managed resource.

#### Variables — Parameterize Your Infrastructure

```
# variables.tf
variable "aws_region" {
  description = "AWS region for deployment"
type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment (dev, staging, prod)"
type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod"
  }
}

variable "instance_count" {
  description = "Number of EC2 instances"
type        = number
  default     = 2
}

# main.tf
provider "aws" {
  region = var.aws_region
}
```

> **variable "aws_region":** defines input variable. Can be overridden via -var, environment variables, or terraform.tfvars.
**type = string/number:** enforces type validation. Terraform rejects invalid types.
**default:** value used if not provided. Optional.
**validation:** custom validation rules. Error message shown if validation fails.
**var.aws_region:** reference variable in code. Enables reusable, parameterized infrastructure.

#### Outputs — Export Important Values

```
# outputs.tf
output "vpc_id" {
  description = "VPC ID"
value       = aws_vpc.main.id
}

output "alb_dns_name" {
  description = "DNS name of Application Load Balancer"
value       = aws_lb.main.dns_name
  sensitive   = false
}

output "rds_endpoint" {
  description = "RDS database endpoint"
value       = aws_db_instance.main.endpoint
}

$ terraform apply
$ terraform output
vpc_id = "vpc-abc123def456"
alb_dns_name = "infra-alb-123456789.us-east-1.elb.amazonaws.com"
rds_endpoint = "infra-db.abc123def456.us-east-1.rds.amazonaws.com:3306"
```

> **output:** exports values for external use. Application config reads from these outputs.
**terraform output:** displays all outputs. Can also target specific: terraform output vpc_id.
**Outputs in CI/CD:** deployment scripts read ALB DNS, RDS endpoint, and configure applications to use them.
**sensitive = true:** marks output (e.g., passwords) as sensitive. Hidden in logs but still exported.

### 3. Terraform State — Track Infrastructure Reality

#### Understanding State Files

```
    Terraform State Management

        Desired State (HCL):         Current State (state file):
        VPC: 10.0.0.0/16             VPC: vpc-abc123 (10.0.0.0/16)
        EC2: 3 instances             EC2: 3 instances (i-aaa, i-bbb, i-ccc)
        RDS: MySQL 8.0               RDS: MySQL 8.0 (infra-db.rds.amazonaws.com)
        
        Terraform compares:
        Desired = Current? → No changes → terraform apply does nothing
        Desired ≠ Current? → Diff shown → terraform apply makes changes
        
        Example: Manual AWS Console change:
        Someone deletes 1 EC2 instance manually.
        Current State: 2 instances
        Desired State: 3 instances
        
        terraform plan shows:
        + Add 1 EC2 instance
        
        terraform apply recreates the deleted instance.
        Infrastructure is back in desired state.
        
```

#### Remote State — Team Collaboration

```bash
# backend.tf - Store state in AWS S3
terraform {
  backend "s3" {
    bucket         = "infra-terraform-state"
key            = "prod/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}

$ terraform init
Initializing the backend...
Initializing backend "s3"...
Successfully configured the backend "s3"!
# Multiple engineers can now apply changes safely
$ terraform plan
$ terraform apply  # Auto-locks state file during apply
```

> **backend "s3":** stores state in AWS S3 instead of local machine. Shared across team.
**encrypt = true:** S3 encryption at rest. State file contains sensitive info (DB passwords, API keys).
**dynamodb_table:** locking table prevents concurrent applies. Only 1 person can run terraform apply at a time (conflict prevention).
**key = "prod/terraform.tfstate":** S3 object key. Separate states for dev/staging/prod in same bucket.
**Auto-locking:** during terraform apply, DynamoDB lock is held. Other users see "waiting for lock" if they try to apply.

### 4. Multi-Region Deployment — Deploy Identical Infrastructure Globally

#### Provider Aliasing for Multiple Regions

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
  alias  = "us_east"
region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
region = "eu-west-1"
}

provider "aws" {
  alias  = "ap_southeast"
region = "ap-southeast-1"
}

# main.tf - Create VPC in each region
resource "aws_vpc" "us_east" {
  provider   = aws.us_east
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc" "eu_west" {
  provider   = aws.eu_west
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc" "ap_southeast" {
  provider   = aws.ap_southeast
  cidr_block = "10.0.0.0/16"
}
```

> **provider with alias:** multiple provider blocks with different regions. Aliased as us_east, eu_west, etc.
**provider = aws.us_east:** resource uses specific provider. Creates VPC in us-east-1.
**Same configuration, 3 regions:** one terraform apply creates infrastructure in all 3 regions simultaneously.
**CIDR 10.0.0.0/16 in all regions:** each VPC is isolated (different regions), no CIDR conflict.

#### Using Modules for DRY Multi-Region Code

```
# modules/region/main.tf
variable "aws_provider" {}
variable "region_name" {}

resource "aws_vpc" "main" {
  provider   = var.aws_provider
  cidr_block = "10.0.0.0/16"
tags       = { Region = var.region_name }
}

resource "aws_subnet" "public" {
  provider   = var.aws_provider
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

# root/main.tf - Use module 3 times
module "region_us_east" {
  source        = "./modules/region"
aws_provider  = aws.us_east
  region_name   = "us-east-1"
}

module "region_eu_west" {
  source        = "./modules/region"
aws_provider  = aws.eu_west
  region_name   = "eu-west-1"
}

module "region_ap_southeast" {
  source        = "./modules/region"
aws_provider  = aws.ap_southeast
  region_name   = "ap-southeast-1"
}
```

> **Modules:** reusable infrastructure templates. Define once, use many times with different parameters.
**source = "./modules/region":** Terraform module path. Can also be Git repos, Terraform Registry.
**Three module invocations:** same module used in all 3 regions with different providers. Much cleaner than copy-paste.
**terraform apply:** all 3 regions deployed in one command. No manual work per region.

```
    Multi-Region Architecture with Terraform

        terraform apply
              ↓
        ┌─────────────────────────────────────────────────────┐
        │            Terraform Main Configuration             │
        │                                                     │
        │  Module: region_us_east   (aws.us_east)           │
        │  Module: region_eu_west   (aws.eu_west)           │
        │  Module: region_ap_southeast (aws.ap_southeast)   │
        └──────────────────────────────────────────────────────┘
                    ↓          ↓                  ↓
        ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
        │ US EAST (1a) │ │ EU WEST (1b) │ │ AP SE (1)    │
        │ VPC          │ │ VPC          │ │ VPC          │
        │ EC2 (3)      │ │ EC2 (3)      │ │ EC2 (3)      │
        │ RDS MySQL    │ │ RDS MySQL    │ │ RDS MySQL    │
        │ ALB          │ │ ALB          │ │ ALB          │
        │ S3 (regional)│ │ S3 (regional)│ │ S3 (regional)│
        └──────────────┘ └──────────────┘ └──────────────┘
        
        One Terraform apply = All 3 regions deployed identically
        
```

### 5. Terraform Workflows — Plan, Apply, Destroy

#### Standard Terraform Workflow

```
# Step 1: Write infrastructure code
$ cat > main.tf << EOF
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
}
EOF

# Step 2: Initialize working directory
$ terraform init
Initializing the backend...
Initializing provider plugins...
# Step 3: Validate configuration
$ terraform validate
Success! The configuration is valid.
# Step 4: Format code (best practices)
$ terraform fmt -recursive

# Step 5: Plan changes (preview)
$ terraform plan -out=tfplan
Terraform will perform the following actions:
+ aws_instance.web will be created
Plan: 1 to add, 0 to change, 0 to destroy
# Step 6: Code review (human approves plan)
$ git add -A
$ git commit -m "Add EC2 instance"
$ git push origin feature/add-web-server
# Create PR, get 2 approvals
# Step 7: Apply changes
$ terraform apply tfplan
aws_instance.web: Creating...
aws_instance.web: Creation complete [id=i-0abc123def456]
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
# Step 8: Verify
$ terraform show
$ aws ec2 describe-instances
```

> **terraform init:** downloads providers, initializes backend. Run once per project.
**terraform validate:** syntax check. Catch errors before plan.
**terraform fmt:** auto-format code (indentation, spacing). Consistency.
**terraform plan:** shows what will change. Saved to tfplan file for later apply. Safe to run anytime.
**Code review:** plan is reviewed as part of PR. Changes are approved by humans before apply.
**terraform apply tfplan:** applies the specific plan (ensures no drift between plan and apply).

#### Destroying Infrastructure

```
# Preview what will be destroyed
$ terraform plan -destroy
Terraform will perform the following actions:
- aws_instance.web will be destroyed
- aws_vpc.main will be destroyed
Plan: 0 to add, 0 to change, 2 to destroy
# Destroy infrastructure
$ terraform destroy
Do you really want to destroy?
  Terraform will destroy all your managed infrastructure.
  There is no undo.
Destroy complete! Resources: 2 destroyed.
# Or target specific resources
$ terraform destroy -target=aws_instance.web
# Destroys only the EC2 instance, keeps VPC
```

> **terraform plan -destroy:** shows what will be deleted. Safer than immediate destroy.
**terraform destroy:** deletes all infrastructure in state file. Confirmation required (type "yes").
**-target:** destroy only specific resource. Use when you need to remove just one component.
**No undo:** destroy is destructive. RDS databases deleted, S3 buckets emptied. Be careful.

### 6. Advanced Patterns — Workspaces, Environments, for_each

#### Workspaces for Environment Separation

```
$ terraform workspace list
* default
$ terraform workspace new dev
$ terraform workspace new staging
$ terraform workspace new prod

$ terraform workspace select prod
$ terraform apply  # Applies to prod workspace
# State files are separate per workspace
# terraform.tfstate.d/dev/terraform.tfstate
# terraform.tfstate.d/staging/terraform.tfstate
# terraform.tfstate.d/prod/terraform.tfstate
# Use workspace name in configuration
variable "instance_count" {
  type = number
  default = terraform.workspace == "prod" ? 5 : 2
}
# Dev/Staging: 2 instances, Prod: 5 instances
```

> **Workspaces:** separate state files for different environments. One codebase, multiple deployments.
**terraform.workspace:** variable containing current workspace name. Use in conditionals.
**Dev uses 2 instances, Prod uses 5:** ternary operator selects based on environment. Cost-efficient.
**Each workspace has own state:** switching workspaces switches state file. Prevents accidental prod changes.

#### for_each — Dynamic Resource Creation

```
# variables.tf
variable "instances" {
  type = map(object({
    instance_type = string
    subnet_id     = string
  }))
  default = {
    web_1 = {
      instance_type = "t3.medium"
subnet_id     = "subnet-1a"
    }
    web_2 = {
      instance_type = "t3.medium"
subnet_id     = "subnet-1b"
    }
    api_1 = {
      instance_type = "t3.large"
subnet_id     = "subnet-1a"
    }
  }
}

# main.tf - Create multiple instances with for_each
resource "aws_instance" "app" {
  for_each      = var.instances
  ami           = "ami-0c55b159cbfafe1f0"
instance_type = each.value.instance_type
  subnet_id     = each.value.subnet_id

  tags = {
    Name = each.key
  }
}

# Reference specific instance
$ terraform output
instances = {
  "api_1" : i-api1234,
  "web_1" : i-web5678,
  "web_2" : i-web9012
}
```

> **for_each:** creates multiple resources from a map. Much better than copy-paste 10 times.
**each.key:** map key (web_1, web_2, api_1). Used for naming.
**each.value:** map value (the object with instance_type, subnet_id).
**Dynamic scaling:** add one line to the map, Terraform creates new instance. Remove line, instance destroyed.

### 7. GitOps for Infrastructure — Automated Deployments via Git

#### GitOps Workflow

```
    GitOps Infrastructure Deployment

        Developer workflow:
        1. Clone repo: git clone infra-repo
        2. Create branch: git checkout -b feature/add-rds
        3. Edit main.tf: add RDS database configuration
        4. Validate: terraform plan
        5. Commit: git commit -m "Add RDS MySQL"
        6. Push: git push origin feature/add-rds
        7. Create PR on GitHub
        
        CI/CD (GitHub Actions):
        1. PR opened → terraform plan runs automatically
        2. Plan output shown in PR comment
        3. Reviewer reads plan, approves
        4. PR merged to main
        
        CD (Automated):
        1. Main branch received commit
        2. CI/CD triggers automatically
        3. terraform apply executed
        4. RDS created in AWS
        5. Slack notification: "RDS deployed"
        
        Rollback:
        1. Revert commit: git revert abc1234
        2. Push: git push origin main
        3. CI/CD automatically applies previous state
        4. RDS destroyed (reverted)
        
        Benefits:
        - All changes in Git (audit trail)
        - Code review before infrastructure changes
        - Reproducible deployments
        - Easy rollbacks (just revert commit)
        
```

#### .github/workflows/terraform.yml

```bash
name: Terraform CI/CD
on:
  pull_request:
    paths:
      - 'terraform/**'
push:
    branches: [main]
    paths:
      - 'terraform/**'
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: '1.7.0'
      
      - name: Configure AWS
run: |
          aws configure set aws_access_key_id ${{ secrets.AWS_ACCESS_KEY }}
          aws configure set aws_secret_access_key ${{ secrets.AWS_SECRET_KEY }}
      
      - name: Terraform Plan
run: |
          terraform init
          terraform plan -out=tfplan
      
      - name: Comment PR with Plan
if: github.event_name == 'pull_request'
run: |
          terraform show tfplan > /tmp/plan.txt
          curl -X POST -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -d @/tmp/plan.txt \
            https://api.github.com/repos/${{ github.repository }}/issues/${{ github.event.number }}/comments

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v2
      
      - name: Terraform Apply
run: |
          terraform init
          terraform apply -auto-approve
      
      - name: Notify Slack
run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -d '{"text": "✅ Infrastructure deployed successfully"}'
```

> **on: pull_request + on: push:** triggers on both PR and main branch pushes.
**plan job:** runs terraform plan on every PR. Output shown in PR comment. Reviewers see exactly what will change.
**apply job:** runs only on main branch merges. terraform apply -auto-approve (no human confirmation needed, approved via PR review).
**AWS credentials:** stored as GitHub Secrets. Never exposed in logs.
**Slack notification:** team is informed when infrastructure changes are applied.

### 8. Testing & Validation — Ensure Infrastructure Quality

#### Pre-Apply Validation

```
# Validate syntax
$ terraform validate
Success! The configuration is valid.
# Format check
$ terraform fmt -check -recursive
main.tf (needs formatting)
# Security linting with tfsec
$ tfsec .
Rule: aws-ec2-no-public-ip-on-launch (HIGH)
Description: EC2 instances should not have public IPs
Location: main.tf:15
# Policy as code with OPA (Open Policy Agent)
$ conftest test tfplan
PASS - All resources have required tags
FAIL - RDS not in private subnet (security policy)
```

> **terraform validate:** syntax check. Required before apply.
**terraform fmt -check:** checks if code follows formatting standards. Consistency in team repos.
**tfsec:** security scanner. Catches: public IPs, unencrypted databases, too-permissive security groups.
**OPA/Conftest:** policy as code. Define company policies (e.g., "all RDS must be private", "all resources must have tags").

**📍 Scene: InfraScale's Global Expansion with Terraform**

> **Vikram**
> 
> "Six months ago, we wrote Terraform code for US East. VPC, subnets, RDS, EC2, ALB, security groups. 2000 lines of HCL. Now the CEO says: launch in 5 more regions tomorrow. Impossible manually."

> **Priya**
> 
> "We add a for_each loop for the 5 new regions. Change 5 lines. terraform plan shows 2000 resources being created. All 8 regions deployed in one apply. 15 minutes later, infrastructure is live globally. Identical in every region."

> **Neeraj**
> 
> "And when a security audit found that RDS wasn't encrypted? We updated one line in the Terraform code: encryption_enabled = true. terraform plan showed the change. PR approved. Applied. All 8 RDS instances encrypted, no manual work."

> **Vikram**
> 
> "That's infrastructure as code. When everything is in Git, changes are code reviews, not manual console clicks. Rollbacks are git revert. Audits are git log. This is how enterprises manage infrastructure at scale."

> **Infrastructure as Code — Core Takeaways for Freshers**

> - **Terraform is declarative.** You declare the desired state. Terraform figures out how to reach it. Change values, run terraform apply, infrastructure adjusts.
> - **State file is source of truth.** Terraform compares desired state (HCL) with current state (state file). Drift detection is automatic.
> - **Plan before apply.** terraform plan shows exactly what will change. Humans review and approve. No surprises in production.
> - **Multi-region is trivial.** One codebase, provider aliasing for regions, for_each loops. Deploy to 10 regions in one command.
> - **GitOps = infrastructure changes via Git.** All changes are code, reviewed, audited. Rollback is git revert. Much safer than manual console access.
> - **Modules are reusable components.** Write once, use many times. Reduces copy-paste, ensures consistency.
> - **Workspaces separate environments.** dev, staging, prod in one repo. Different instance counts, configurations per environment.
> - **Remote state enables collaboration.** S3 backend + DynamoDB locking. Multiple engineers can safely run terraform apply without conflicts.

##### Infrastructure as Code Standards — InfraScale Production Rules

- Always use version control for infrastructure. Every change should be git commit → PR → review → merge. No manual console changes.
- Use remote state (S3 + DynamoDB lock). Never use local state in production. Local state makes collaboration impossible.
- Plan before apply. terraform plan is not optional. It's your safety net. Review output, get approval, then apply.
- Tag everything. Every resource should have Name, Environment, ManagedBy, Owner tags. Enables cost allocation, automation, filtering.
- Use workspaces or separate state files for dev/staging/prod. Never mix environments in one state. Mistakes have smaller blast radius.
- Encrypt sensitive state data. S3 encryption, state file encryption. RDS passwords, API keys are stored in state.
- Use modules for reusability. Don't copy-paste infrastructure. DRY principle applies to infrastructure too.
- Run validation and security checks in CI/CD. terraform validate, tfsec, OPA policies. Catch issues before apply.

##### 🏋️ Hands-On Exercises — Master Infrastructure as Code

1. **Deploy single-region infrastructure:** Write Terraform for a simple VPC (10.0.0.0/16), public subnet (10.0.1.0/24), private subnet (10.0.2.0/24), Internet Gateway, NAT Gateway. Create EC2 instance in public subnet, RDS MySQL in private subnet. Run terraform plan to preview. Apply. Use terraform output to display RDS endpoint and EC2 public IP. Destroy when done.
2. **Multi-region deployment:** Refactor single-region code to use provider aliasing. Set up 3 providers: us-east-1, eu-west-1, ap-southeast-1. Create VPC, EC2, RDS in all 3 regions with one terraform apply. Verify resources exist in all regions via AWS console or CLI.
3. **Use modules for DRY:** Extract VPC+EC2+RDS into a module (modules/region/). Root module calls the region module 3 times with different providers. terraform apply should create infrastructure in all 3 regions. Change instance type in module, apply, verify all 3 regions updated.
4. **Workspace-based environments:** Create workspaces: dev, staging, prod. In variables.tf, set instance_count to 1 for dev, 2 for staging, 3 for prod. Switch between workspaces, apply to each. Verify dev has 1 instance, prod has 3. Destroy all workspaces.
5. **Set up GitOps workflow:** Create GitHub repo with Terraform code. Add .github/workflows/terraform.yml that: (1) runs terraform plan on PR, comments plan output, (2) on main merge, runs terraform apply automatically. Push code, create PR, verify plan comment. Merge, verify apply triggered. Create another PR changing instance type, merge, verify apply in 2 regions.

### Infrastructure as Code Project Complete 🎉

You have mastered Infrastructure as Code with Terraform. You can define cloud infrastructure in code, version control it, deploy to multiple regions simultaneously, and manage infrastructure changes via Git with code review and GitOps workflows. You understand state management, remote backends, locking, modules for reusability, and testing/validation. This is how Google, Netflix, and other tech giants manage infrastructure at scale: everything is code, everything is versioned, everything is reviewed, everything is automated.

> **Priya**
> 
> "Infrastructure as Code changed everything. Before: infrastructure was manual, undocumented, inconsistent. Security group permissions didn't match documentation. Someone had root access and made changes without telling anyone. After: every change is git commit → review → approve → deploy. Rollback is as simple as git revert. Audits are git log. Multi-region deployments are one command. Infrastructure becomes reliable, predictable, safe."

> **Vikram**
> 
> "And scaling becomes effortless. We went from 1 region to 8 regions. With manual infrastructure, that would take 3 months and 10+ people. With Terraform, it took 1 engineer 2 weeks to parameterize the code. Now, scaling to 20 regions is just changing a variable. The same 2000-line Terraform file manages 20 regions globally. That's the power of infrastructure as code."

> **Next: Advanced DevOps — Kubernetes, Service Mesh, Observability at Scale**

> - Kubernetes with Terraform — Deploy entire K8s clusters, applications, networking with IaC. Helm for package management.
> - Service Mesh (Istio) — Advanced traffic management, circuit breaking, distributed tracing, mTLS between services.
> - Observability at Scale — Prometheus (metrics), Grafana (dashboards), ELK/Splunk (logs), Jaeger (distributed tracing). Know exactly what's happening.
> - Policy as Code (OPA) — Define company policies (security, compliance, cost). Enforce automatically at deploy time.
> - Infrastructure Disaster Recovery — Multi-region failover, automated backups, recovery time objectives (RTO), recovery point objectives (RPO).
> - Cost Optimization — Spot instances, reserved instances, rightsizing, auto-scaling policies. Reduce cloud bills by 40-60%.
