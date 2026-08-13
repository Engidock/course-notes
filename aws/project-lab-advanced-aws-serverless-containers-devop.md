# Advanced AWS: Serverless, Containers & DevOps Services

> **👋 Who This Document Is For**

> - You have completed the **AWS Foundations** module — you know IAM, VPC, EC2, S3, RDS, ALB, and Auto Scaling.
> - This module covers the **next layer** of AWS that companies use once their infrastructure is stable — serverless computing, containers, managed databases, and automated CI/CD pipelines.
> - Same company: **ShopNest** — Bengaluru e-commerce startup. Same engineers: Nandita, Suresh, Karthik. The team is now evolving from basic EC2 infrastructure to modern cloud-native architecture.
> - Every code block is **short and focused**. Every explanation is **bullet points**. One concept at a time.

#### What You Will Learn in This Module

> **⚡ Phase 1 — Lambda & API Gateway**

> Run code without servers. Trigger functions from HTTP requests, S3 uploads, DynamoDB streams. Build ShopNest's image resize function and order notification API.

> Run Docker containers on AWS without managing servers. Build and push a Docker image to ECR, then deploy it as a scalable ECS Fargate service.

> AWS's fully managed NoSQL database. Understand tables, partition keys, sort keys, indexes, and TTL. Store ShopNest's shopping cart data.

> Automated CI/CD on AWS. Push code to CodeCommit → automatically test → deploy to EC2 or ECS. Zero manual deployment steps.

> Store database passwords securely. Connect to EC2 without SSH keys. Deliver product images globally at low latency with CloudFront CDN.

Lambda, API Gateway, ECS Fargate, ECR, DynamoDB, CodePipeline, CloudFront, Secrets Manager

**Scene 1 — ShopNest Architecture Review | "We're Ready for the Next Level"**

> **Nandita** _Senior Cloud Engineer — ShopNest_
> 
> Karthik, the basic infrastructure is solid. EC2, RDS, S3, ALB, Auto Scaling — it all works. But we're doing too much manually. Every time we deploy, an engineer SSHs into the server and runs git pull. Every time a product image is uploaded, a developer manually resizes it. Every database password is stored in a config file on the EC2 instance. We're paying for EC2 instances to run background jobs that run once a day for 2 seconds. There's a better, cheaper, more automated way to do all of this.

> **Suresh** _DevOps Architect — ShopNest_
> 
> Lambda for the image resizing — it only runs when an image is uploaded, costs fractions of a rupee per invocation, and scales automatically. API Gateway in front of Lambda gives us a clean REST API. ECS Fargate for our microservices — containers with zero server management. CodePipeline for automated deployment. Secrets Manager for passwords. CloudFront for faster image delivery worldwide. These are the services that separate a startup AWS setup from an enterprise one.

### 1. Phase 1 — AWS Lambda and API Gateway

**Business Problem:** Every time ShopNest uploads a product image, it needs to be resized to three sizes: thumbnail (100×100), medium (400×400), and full (800×800). Running this on EC2 means paying for a server 24/7 even though resizing happens only when a new product is added — maybe 50 times a day for 2 seconds each. Lambda runs code only when triggered and charges only for the milliseconds it runs.

> **What is Lambda? — The Simplest Explanation**

> - Lambda lets you run a **function** (a piece of code) in response to an event — without creating or managing any server.
> - You upload your code. AWS handles the server, OS, runtime, scaling, and patching.
> - **You pay per invocation** — first 1 million calls per month are free. Then fractions of a rupee per call. Idle time costs ₹0.
> - Lambda **auto-scales instantly** — if 10,000 images are uploaded simultaneously, AWS runs 10,000 parallel function instances automatically.
> - Maximum execution time: **15 minutes** per invocation. For short tasks (image resize, send email, process order) — perfect.

```
ShopNest Image Processing — Lambda Trigger Flow
================================================

  User uploads product image to S3
        │
        ▼ S3 Event Notification (ObjectCreated)
  Lambda Function: resize-product-image
        │
        ├── Read original from s3://shopnest-images/original/shoe-001.jpg
        ├── Resize to 100x100  → save to s3://shopnest-images/thumb/shoe-001.jpg
        ├── Resize to 400x400  → save to s3://shopnest-images/medium/shoe-001.jpg
        └── Resize to 800x800  → save to s3://shopnest-images/full/shoe-001.jpg

  Cost: ~0.001 rupees per image processed
  Duration: ~800ms per image
  EC2 equivalent: ₹800/month for idle server vs ₹5/month Lambda
```

#### 1.1 Create a Lambda Function

```
# Create the Lambda function
aws lambda create-function \
  --function-name resize-product-image \
  --runtime python3.12 \
  --role arn:aws:iam::123456:role/lambda-s3-role \
  --handler lambda_function.handler \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 512
```

**📖 Lambda Creation Parameters**

- **--runtime** — the language/version. Options: python3.12, nodejs20.x, java21, go1.x, dotnet8. All managed by AWS.
- **--role** — IAM role the function assumes. Must grant permissions to read/write S3, write logs to CloudWatch. Lambda needs this to access other AWS services.
- **--handler** — filename.function_name. `lambda_function.handler` means: file `lambda_function.py`, function named `handler`.
- **--timeout 30** — kill the function after 30 seconds if it hasn't finished. Default is 3 seconds — increase for slower tasks.
- **--memory-size 512** — RAM in MB. More RAM = more CPU allocated too. Image processing needs at least 256MB.

#### 1.2 The Lambda Function Code

```python
# lambda_function.py
import boto3, json
from PIL import Image
import io

s3 = boto3.client('s3')

def handler(event, context):
    # Get bucket and key from S3 event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key    = event['Records'][0]['s3']['object']['key']

    # Download original image
    obj  = s3.get_object(Bucket=bucket, Key=key)
    img  = Image.open(io.BytesIO(obj['Body'].read()))

    # Resize and upload thumbnail
    img.thumbnail((100, 100))
    buf = io.BytesIO()
    img.save(buf, 'JPEG')
    s3.put_object(Bucket=bucket,
                  Key=f"thumb/{key}",
                  Body=buf.getvalue())
    return {'statusCode': 200}
```

**📖 Lambda Function Structure**

- Every Lambda function needs a **handler function**. AWS calls this when the function is triggered.
- **event** — contains the trigger data. For S3 triggers, it has bucket name, object key, size, and event type.
- **context** — Lambda metadata: remaining time, request ID, function name. Useful for logging.
- **boto3** — the AWS SDK for Python. Use it to call any AWS service from inside Lambda.
- Return a dict with `statusCode` when Lambda is called via API Gateway. For S3 triggers, return value is ignored.
- Add **Pillow (PIL)** as a Lambda Layer — it's not in the default runtime. Layers let you share libraries across functions.

#### 1.3 Add S3 Trigger to Lambda

```
# Allow S3 to invoke this Lambda function
aws lambda add-permission \
  --function-name resize-product-image \
  --statement-id s3-trigger-permission \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::shopnest-product-images-2026
# Configure S3 to send events to Lambda
aws s3api put-bucket-notification-configuration \
  --bucket shopnest-product-images-2026 \
  --notification-configuration file://s3-notify.json
```

**📖 S3 → Lambda Trigger**

- **add-permission** — grants S3 the permission to call (invoke) your Lambda function. Without this, S3 can't trigger it even if configured to do so.
- **put-bucket-notification-configuration** — tells S3: "When a new object is created, send an event to this Lambda function."
- The notification config JSON specifies: which event type (s3:ObjectCreated:*), which Lambda ARN, and optionally a filter (only trigger for .jpg files in the /original/ prefix).
- **Every invocation is independent** — Lambda doesn't remember the previous call. If you need to share state, use S3, DynamoDB, or ElastiCache.

#### 1.4 API Gateway — HTTP Endpoint for Lambda

**Business Problem:** ShopNest's mobile app needs to check order status. Instead of calling EC2 directly, use API Gateway + Lambda — no server to manage, scales automatically, built-in HTTPS, and rate limiting.

```
# Create a REST API
aws apigateway create-rest-api \
  --name shopnest-api \
  --description "ShopNest mobile backend API"
# Test the Lambda function directly
aws lambda invoke \
  --function-name get-order-status \
  --payload '{"orderId":"ORD-12345"}' \
  --cli-binary-format raw-in-base64-out \
  response.json

cat response.json
```

**📖 API Gateway + Lambda**

- **API Gateway** — managed HTTP endpoint. Routes incoming requests to Lambda functions based on path and method. Handles HTTPS, rate limiting, authentication, and request validation.
- Flow: `Mobile App → HTTPS → API Gateway → Lambda → DynamoDB → Response`
- **aws lambda invoke** — test your function locally from CLI without needing API Gateway. Essential for debugging.
- **--payload** — the event JSON sent to the function. Simulates what API Gateway would send.
- For modern APIs, use **HTTP API** (cheaper, faster) instead of REST API unless you need advanced features like custom authorisers or response caching.

AWS

Lambda › Functions › resize-product-image

Runtime Python 3.12

Memory 512 MB

Timeout 30 seconds

Last invocation 2 minutes ago — Duration: 847ms, Billed: 900ms

Triggers S3: shopnest-product-images-2026 → ObjectCreated

Invocations (30d) 2,341 | Errors: 0 | Cost: ₹0.003

Test

View Logs

Configuration

### 2. Phase 2 — ECS Fargate and ECR: Containers Without Servers

**Business Problem:** ShopNest's recommendation engine is a Python microservice in a Docker container. Running it on EC2 means managing OS patches, Docker installation, and instance types. ECS Fargate runs containers without any EC2 instances to manage — you give AWS a Docker image, it runs it, scales it, and restarts it if it crashes.

**Scene 2 — ShopNest Engineering Standup | "Docker but No Servers"**

> **Nandita** _Senior Cloud Engineer_
> 
> Karthik, our recommendation engine team handed me a Dockerfile and said "make it run on AWS." I could launch an EC2 and manually set up Docker. But then we're managing another server — OS updates, Docker version, restart policies. ECS Fargate is better: give it the Docker image, tell it how much CPU and RAM you want, and AWS runs it. You never SSH into a server. AWS handles everything below the Docker layer.

```
ECS Fargate Architecture — ShopNest Recommendation Service
==============================================================

  ECR (Elastic Container Registry)
  └── shopnest-recommendations:latest  (Docker image stored here)
        │
        ▼ ECS pulls image and runs it
  ECS Cluster: shopnest-cluster
  └── ECS Service: recommendations-svc
      ├── Task: recommendations (2 running)
      │    ├── Container: rec-engine  ← Python Flask app
      │    │   CPU: 256  Memory: 512MB
      │    └── Port: 5000 exposed
      └── Application Load Balancer → Target Group → Tasks

  Fargate = No EC2 to manage. AWS handles:
  ✓ OS patching  ✓ Docker installation  ✓ Instance provisioning
  ✓ Task restart on crash  ✓ Auto scaling of tasks
```

#### 2.1 Push Docker Image to ECR

```bash
# Create a private ECR repository
aws ecr create-repository \
  --repository-name shopnest-recommendations \
  --region ap-south-1
# Authenticate Docker to ECR
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.ap-south-1.amazonaws.com
# Build, tag, and push the image
docker build -t shopnest-recommendations .
docker tag shopnest-recommendations:latest \
  123456789.dkr.ecr.ap-south-1.amazonaws.com/shopnest-recommendations:latest
docker push \
  123456789.dkr.ecr.ap-south-1.amazonaws.com/shopnest-recommendations:latest
```

**📖 ECR — Private Docker Registry**

- **ECR** (Elastic Container Registry) — AWS's private Docker registry. Like Docker Hub but inside your AWS account. No public internet access required.
- **get-login-password** — generates a temporary password valid for 12 hours that authenticates Docker to your ECR. Must run this before pushing.
- The ECR URL format is: `ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/REPO_NAME:TAG`
- **ECR Scanning** — enable image scanning on push to automatically detect CVE vulnerabilities in your containers. Free and takes seconds.
- In CI/CD pipelines, use OIDC with GitHub Actions to authenticate to ECR without storing long-lived credentials.

#### 2.2 Create an ECS Task Definition

```
# Register a Fargate task definition
aws ecs register-task-definition \
  --family recommendations-task \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 256 \
  --memory 512 \
  --execution-role-arn arn:aws:iam::123456:role/ecsTaskExecRole \
  --container-definitions file://container-def.json
```

**📖 Task Definition**

- A **task definition** is the blueprint for how to run your container — like a docker-compose.yml but for ECS.
- **--cpu 256 / --memory 512** — Fargate CPU is in vCPU units (256 = 0.25 vCPU). Memory in MB. You pay exactly for what you specify.
- **--network-mode awsvpc** — required for Fargate. Each task gets its own ENI (network interface) with a private IP in your VPC.
- **execution-role-arn** — IAM role that allows ECS to pull the image from ECR and write logs to CloudWatch. Separate from your app's IAM role.
- **container-definitions** — JSON that specifies the image URL, port mappings, environment variables, and log configuration.

#### 2.3 Run the ECS Service

```
# Create ECS cluster
aws ecs create-cluster \
  --cluster-name shopnest-cluster
# Create the service (keeps 2 tasks always running)
aws ecs create-service \
  --cluster shopnest-cluster \
  --service-name recommendations-svc \
  --task-definition recommendations-task:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration file://network-config.json
```

**📖 ECS Service**

- An **ECS Service** ensures a specified number of task copies (desired-count) are always running. If a task crashes, ECS replaces it automatically.
- **--desired-count 2** — run 2 container instances. Spread across AZs for high availability.
- **--launch-type FARGATE** — AWS manages the underlying EC2 instances. You never see or touch them.
- To deploy a new image version: update the task definition with the new image tag, then run `aws ecs update-service --force-new-deployment`. ECS rolls out the new version with zero downtime.
- ECS **Service Auto Scaling** works like EC2 Auto Scaling — scale task count based on CPU, memory, or custom metrics.

### 3. Phase 3 — DynamoDB: NoSQL at Any Scale

**Business Problem:** ShopNest's shopping cart needs to store items for millions of users simultaneously with single-digit millisecond read/write speed. Querying a relational database (RDS) for every cart update creates too much load. DynamoDB is purpose-built for this — high-speed key-value and document storage that scales to any traffic level without configuration changes.

> **DynamoDB vs RDS — When to Use Which**

> - **Use DynamoDB for:** shopping carts, user sessions, real-time gaming leaderboards, IoT sensor data, anything that needs sub-10ms reads/writes at any scale.
> - **Use RDS for:** orders, inventory, financial transactions — anything with complex relationships, joins, or ACID transaction requirements.
> - DynamoDB has **no fixed schema** — each item can have different attributes. Add a new field to one item without changing any other items.
> - DynamoDB **scales automatically** — no connections to manage, no read replicas to set up. It just works at any scale.

#### 3.1 Create a DynamoDB Table

```
aws dynamodb create-table \
  --table-name ShopNestCart \
  --attribute-definitions \
    AttributeName=userId,AttributeType=S \
    AttributeName=itemId,AttributeType=S \
  --key-schema \
    AttributeName=userId,KeyType=HASH \
    AttributeName=itemId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST
```

**📖 DynamoDB Keys**

- **Partition key (HASH)** — determines which physical server stores the item. Must be unique or used together with sort key. Here: userId groups all cart items for one user together.
- **Sort key (RANGE)** — sorts items within a partition. Enables range queries: "get all cart items for user-123" or "get items added after timestamp X."
- **Primary key = partition key + sort key** — the combination must be unique across all items.
- **PAY_PER_REQUEST** — pay only for actual reads and writes. No capacity planning needed. Perfect for variable workloads. Alternatively, set `PROVISIONED` mode for predictable, high-volume workloads at lower cost.

#### 3.2 Read and Write to DynamoDB

```
# Write an item to the cart
aws dynamodb put-item \
  --table-name ShopNestCart \
  --item '{
    "userId": {"S": "user-123"},
    "itemId": {"S": "shoe-456"},
    "name":   {"S": "Nike Air Max"},
    "price":  {"N": "4999"},
    "qty":    {"N": "2"},
    "ttl":    {"N": "1767225600"}
  }'
```

**📖 DynamoDB Data Types**

- **{"S": "value"}** — String type. Used for IDs, names, status codes.
- **{"N": "42"}** — Number type. Numbers are stored as strings internally but handled as numbers for comparisons and math.
- **{"BOOL": true}** — Boolean. **{"L": [...]}** — List. **{"M": {...}}** — Map (nested object).
- **ttl** — Time to Live. Set a Unix timestamp epoch. DynamoDB automatically deletes the item after that time. Perfect for expiring sessions and temporary data. Free feature.

```
# Get all cart items for a user
aws dynamodb query \
  --table-name ShopNestCart \
  --key-condition-expression \
    "userId = :uid" \
  --expression-attribute-values \
    '{":uid": {"S": "user-123"}}'
```

**📖 DynamoDB Query vs Scan**

- **Query** — retrieves items by partition key (and optionally sort key). Fast — reads only items for that user. Use this in production.
- **Scan** — reads every item in the table and filters. Very slow and expensive at scale. Only use for small tables or admin scripts.
- **key-condition-expression** — filter using key attributes. Use `:uid` as a placeholder; actual value goes in `--expression-attribute-values`.
- For querying on non-key attributes (e.g. "find all items with price > 1000"), create a **Global Secondary Index (GSI)** on that attribute.

### 4. Phase 4 — CodePipeline and CodeDeploy: Automated CI/CD

**Business Problem:** Every ShopNest deployment requires an engineer to SSH into EC2, run git pull, and restart the app. If done at 11 PM, they make mistakes. CodePipeline automates the entire flow: code pushed to GitHub → automatically tested → automatically deployed to EC2 or ECS — with zero human steps and a full audit trail.

```
ShopNest CI/CD Pipeline — CodePipeline Flow
==============================================

  Developer pushes code to GitHub
        │
        ▼ Stage 1: SOURCE
  CodePipeline detects change → downloads code
        │
        ▼ Stage 2: BUILD (CodeBuild)
  Install dependencies → Run tests → Build Docker image
  → Push image to ECR
        │
        ▼ Stage 3: DEPLOY (CodeDeploy)
  Blue/Green deployment to ECS:
  ├── Launch new tasks with new image (Green)
  ├── Wait for health checks to pass
  ├── Shift 10% traffic → wait 5min → check error rate
  ├── Shift 100% traffic to new version
  └── Terminate old tasks (Blue)

  If health check fails at any step → automatic rollback to Blue
```

#### 4.1 Create a CodeBuild Project

```
# buildspec.yml — inside your GitHub repo
version: 0.2

phases:
  install:
    commands:
      - npm install

  pre_build:
    commands:
      - npm test
      - aws ecr get-login-password | docker login ...

  build:
    commands:
      - docker build -t $IMAGE_URI .
      - docker push $IMAGE_URI

artifacts:
  files:
    - imagedefinitions.json
```

**📖 buildspec.yml**

- **buildspec.yml** — the instructions file that CodeBuild reads. Lives in the root of your repository.
- **phases** — install (setup tools), pre_build (auth, lint, test), build (compile, package), post_build (tag, notify).
- **$IMAGE_URI** — CodeBuild environment variable containing the ECR image URL. Set in the CodeBuild project configuration.
- **pre_build runs npm test** — if tests fail, CodeBuild exits with non-zero and the pipeline stops. Broken code never reaches production.
- **artifacts** — files passed to the next pipeline stage. imagedefinitions.json tells CodeDeploy which new image to deploy to ECS.

#### 4.2 Create the Pipeline

```
# Create the full pipeline
aws codepipeline create-pipeline \
  --pipeline file://pipeline-config.json
# Manually trigger a run
aws codepipeline start-pipeline-execution \
  --name shopnest-pipeline
# Check pipeline status
aws codepipeline get-pipeline-state \
  --name shopnest-pipeline \
  --query 'stageStates[*].{Stage:stageName,Status:latestExecution.status}' \
  --output table
```

**📖 CodePipeline**

- **Pipeline config JSON** — defines stages, their order, and the provider for each. Source (GitHub/CodeCommit), Build (CodeBuild), Deploy (CodeDeploy/ECS).
- After creation, the pipeline automatically runs every time you push to the configured branch.
- **--query with --output table** — displays the stage names and statuses in a readable table in your terminal. Very useful for monitoring deployments.
- Add an **approval stage** between Build and Deploy to require a human to click "Approve" before production deployments. Configured in the pipeline JSON — takes 2 minutes to add.

AWS

CodePipeline › Pipelines › shopnest-pipeline

SOURCE

✓ Succeeded

GitHub: main

→

BUILD

⟳ Running…

CodeBuild: 1m 23s

→

DEPLOY

Pending

ECS Fargate

Last commit "feat: add product recommendations endpoint" — karthik@shopnest.in

### 5. Phase 5 — Secrets Manager, SSM Parameter Store & CloudFront

**Scene 3 — ShopNest Security Audit | Three Problems to Fix Today**

> **Suresh** _DevOps Architect — ShopNest_
> 
> Security audit flagged three things. One: the RDS password is in a config file on EC2 — anyone who gets into the instance can read it. Fix: Secrets Manager, secrets injected at runtime. Two: our team SSHs into EC2 using a shared PEM file that three engineers have — no audit trail of who did what. Fix: SSM Session Manager, no PEM files needed. Three: product images load in 1.2 seconds for a customer in Delhi — they're fetching from S3 in Mumbai. Fix: CloudFront CDN, serve images from Delhi edge location in 60ms.

#### 5.1 Secrets Manager — Store Passwords Securely

```
# Store RDS password in Secrets Manager
aws secretsmanager create-secret \
  --name shopnest/rds/credentials \
  --description "ShopNest RDS MySQL credentials" \
  --secret-string '{
    "username": "admin",
    "password": "ShopNest@2026!",
    "host":     "shopnest-mysql.xyz.ap-south-1.rds.amazonaws.com",
    "port":     3306,
    "dbname":   "shopnest"
  }'
```

**📖 Secrets Manager**

- **Secrets Manager** — stores, encrypts, and rotates secrets. The secret value is never stored in plaintext anywhere — encrypted with KMS.
- Your application **calls the Secrets Manager API at startup** to fetch credentials. No plaintext in config files or environment variables.
- **Automatic rotation** — configure Secrets Manager to rotate the RDS password every 30 days automatically. It updates both the secret and the RDS instance password atomically.
- **Cost**: ~₹32/secret/month + ₹0.04 per 10,000 API calls. Very cheap for the security it provides.

```python
# Retrieve the secret in your application (Python)
import boto3, json

client = boto3.client('secretsmanager',
                      region_name='ap-south-1')

response = client.get_secret_value(
  SecretId='shopnest/rds/credentials'
)
creds = json.loads(response['SecretString'])
# creds['host'], creds['password'] are now available
```

**📖 Fetching Secrets in Code**

- Call `get_secret_value` at application startup — not on every database query (too slow).
- Cache the credentials in memory. Refresh the cache periodically or when a connection fails (in case the secret rotated).
- The EC2/Lambda must have an IAM role with `secretsmanager:GetSecretValue` permission for this specific secret ARN — not all secrets.
- SDK SDKs for all major languages: Python (boto3), Node.js, Java, Go. Same API, same result.

#### 5.2 SSM Session Manager — Connect Without SSH Keys

```
# Connect to EC2 without SSH — just AWS CLI
aws ssm start-session \
  --target i-0a1b2c3d4e5f67890
# Run a command on all ShopNest web servers
aws ssm send-command \
  --targets 'Key=tag:Project,Values=shopnest' \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["systemctl restart nginx"]'
```

**📖 SSM Session Manager**

- **No PEM files needed** — authentication is via IAM. Only engineers with the correct IAM permission can connect.
- **No port 22 open** — security groups don't need SSH inbound. Removes a major attack surface.
- **Full audit trail** — every session and every command is logged to CloudWatch and S3. You know exactly who ran what and when on which server.
- **send-command** — run a shell command on multiple instances simultaneously, targeted by tags. Essential for fleet management (patching, config updates).
- Requires the **SSM Agent** to be installed on EC2 (pre-installed on Amazon Linux 2, Ubuntu 20.04+) and an IAM role with SSM permissions.

#### 5.3 CloudFront — Global Content Delivery

```
# Create CloudFront distribution for S3 bucket
aws cloudfront create-distribution \
  --distribution-config '{
    "Origins": {
      "Items": [{
        "Id":            "shopnest-s3",
        "DomainName":    "shopnest-product-images-2026.s3.ap-south-1.amazonaws.com",
        "S3OriginConfig": {"OriginAccessIdentity": ""}
      }]
    },
    "DefaultCacheBehavior": {
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId":        "658327ea-f89d-4fab-a63d-7e88639e58f6"
    },
    "Enabled": true,
    "PriceClass": "PriceClass_200"
  }'
```

**📖 CloudFront CDN**

- **CloudFront** — AWS's global Content Delivery Network. Has 450+ edge locations worldwide including Delhi, Mumbai, Chennai, Hyderabad, Kolkata.
- The first request fetches from S3 (origin). CloudFront caches it at the edge for subsequent requests.
- **redirect-to-https** — all HTTP requests are automatically redirected to HTTPS. Free SSL included with CloudFront.
- **PriceClass_200** — use North America, Europe, and Asia edge locations. PriceClass_All includes all locations but costs more.
- After creation, you get a URL like `d1abc123.cloudfront.net`. Configure a CNAME in Route 53 to map `images.shopnest.in` to this URL.

AWS

CloudFront › Distributions › E1ABCDEF2GHIJK

Domain name d1abc123.cloudfront.net

CNAME images.shopnest.in

Origin shopnest-product-images-2026.s3.ap-south-1.amazonaws.com

SSL Certificate ACM: *.shopnest.in (valid)

Edge locations 450+ worldwide incl. Delhi, Mumbai, Chennai, Hyderabad

Cache hit rate (7d) 94.3% | Avg latency Delhi: 58ms (was 1,240ms from S3)

### 6. Advanced AWS Services — Quick Reference

Service

One-Line Purpose

Key CLI Command

Lambda

Run functions without servers, pay per execution

aws lambda invoke / create-function

API Gateway

HTTPS endpoint that triggers Lambda or routes to EC2

aws apigateway create-rest-api

ECR

Private Docker image registry inside AWS

aws ecr create-repository + docker push

ECS Fargate

Run containers without managing EC2 instances

aws ecs create-service / update-service

DynamoDB

NoSQL key-value DB with millisecond latency at any scale

aws dynamodb put-item / query

ElastiCache

In-memory Redis/Memcached — cache DB queries, sessions

aws elasticache create-replication-group

CodeCommit

AWS-managed private Git repository

aws codecommit create-repository

CodeBuild

Managed CI server — compile, test, build Docker images

aws codebuild start-build

CodeDeploy

Automate code deployment to EC2 or ECS with rollbacks

aws deploy create-deployment

CodePipeline

Orchestrate Source→Build→Deploy as an automated pipeline

aws codepipeline start-pipeline-execution

Secrets Manager

Store and auto-rotate secrets: DB passwords, API keys

aws secretsmanager get-secret-value

SSM Parameter Store

Store config values and non-sensitive secrets for free

aws ssm get-parameter

SSM Session Manager

Connect to EC2 without SSH keys or open port 22

aws ssm start-session --target i-xxx

CloudFront

CDN — cache and serve content from edge locations worldwide

aws cloudfront create-distribution

SQS

Message queue — decouple services, buffer bursts of work

aws sqs send-message / receive-message

SNS

Pub/Sub notifications — send email, SMS, trigger Lambda

aws sns publish / subscribe

EventBridge

Event bus — route events between AWS services and apps

aws events put-rule / put-targets

Kinesis

Real-time streaming data ingestion (logs, clickstreams)

aws kinesis put-record / get-records

### 7. Interview Questions — Advanced AWS

##### Interview Q&A — Mid-Level (6 months–2 years AWS experience)

**Q: Q1. When would you choose Lambda over EC2 for running code?**

A: **Choose Lambda when:** the task runs for less than 15 minutes, is triggered by events (S3 upload, API call, schedule, DynamoDB stream), and does not run continuously 24/7.
**Examples:** image resizing on upload, sending order confirmation emails, processing queue messages, scheduled reports that run once a day, webhook handlers.
**Choose EC2 when:** the workload runs continuously (web server, long-running process), requires more than 10GB RAM or 6 vCPUs, needs specific OS configuration, or runs longer than 15 minutes.
**Cost comparison:** Lambda is much cheaper for infrequent workloads. For a function that runs 1 million times for 100ms each, Lambda costs ~₹1.50/month. Running the equivalent EC2 t3.micro = ₹640/month.

**Q: Q2. What is the difference between ECS with EC2 launch type and ECS with Fargate?**

A: **ECS EC2 launch type** — you manage a cluster of EC2 instances. ECS places containers on them. You control instance type, OS, Docker version, and pay for the EC2 whether or not containers are running. More control, more responsibility.
**ECS Fargate** — AWS manages the underlying instances completely. You specify CPU and memory per task. You only pay per second of task execution. No EC2 instances to patch, no container agents to manage.
**Choose Fargate when:** you don't want to manage infrastructure, workloads are variable, and cost-per-task model works for you.
**Choose EC2 launch type when:** you need GPU instances, Windows containers, very high task density (packing many small tasks on large instances for cost efficiency), or specific networking configurations.

**Q: Q3. What is DynamoDB's partition key and why is choosing it carefully so important?**

A: The **partition key** determines which physical partition (shard) of DynamoDB stores an item. DynamoDB distributes items across multiple partitions for scale.
**Problem — hot partition:** If many items share the same partition key (e.g., all orders in a day use `date` as partition key), one partition gets all the traffic while others sit idle. Performance degrades badly.
**Good partition key**: has *high cardinality* (many distinct values) and *uniform access* (all values accessed equally). UserId, orderId, productId — all good choices.
**Bad partition key**: status (only a few values like "pending", "completed"), date, country (hot partitions for popular countries).
If your natural key is bad, add a random suffix to distribute writes: `productId + random(1-10)` — then query all 10 shards and merge results.

**Q: Q4. How does CodeDeploy Blue/Green deployment work and what is its advantage?**

A: **Blue** = current running version serving production traffic. **Green** = new version being deployed.
CodeDeploy provisions a new set of instances/tasks (Green) with the new code and runs health checks. Meanwhile Blue keeps serving all traffic.
Once Green passes health checks, CodeDeploy shifts a percentage of traffic to Green (e.g. 10%). Monitors error rate for a configurable period. If healthy, shifts 100% to Green.
**Key advantage**: if Green has a bug, CodeDeploy automatically rolls back to Blue in seconds — old instances are still running, just not receiving traffic. Zero downtime rollback.
Contrast with in-place deployment: old version is overwritten directly. If the new version breaks, rollback means re-deploying the old version — takes minutes, not seconds.

**Q: Q5. What is the difference between Secrets Manager and SSM Parameter Store?**

A: **SSM Parameter Store (free tier)** — stores configuration values (strings, string lists) and optionally encrypts them with KMS. No automatic rotation. Costs nothing for standard parameters.
**SSM Parameter Store (Advanced)** — parameters larger than 4KB, higher throughput. Small cost per parameter per month.
**Secrets Manager** — purpose-built for secrets. Automatic rotation (can rotate RDS, Redshift, DocumentDB passwords automatically). Full secret version history. Native integration with RDS for rotation. Costs ~₹32/secret/month.
**Decision rule:** Use Parameter Store (Standard) for config values (environment names, feature flags, non-sensitive settings). Use Secrets Manager for passwords, API keys, and anything that needs automatic rotation.
Both integrate with Lambda, EC2, ECS via the SDK. Application code to retrieve from both is nearly identical.

**Q: Q6. What are SQS and SNS, and how do they work together?**

A: **SQS** (Simple Queue Service) — a message queue. Producer sends a message, it sits in the queue until a consumer reads and processes it. Messages are not pushed — consumers poll for them. Scales to millions of messages.
**SNS** (Simple Notification Service) — a pub/sub notification service. A publisher sends one message to a topic. All subscribers (email, SMS, Lambda, SQS) receive it simultaneously.
**Fan-out pattern** — SNS + SQS together. An order event is published to an SNS topic. Multiple SQS queues subscribe: one for the inventory service, one for the email service, one for the analytics service. Each service processes at its own pace.
**SQS benefits decoupling:** if the inventory service is slow, the queue absorbs the backlog — the order API doesn't need to wait. Services scale independently.

**Quiz: Quiz 1 — ShopNest uploads 500 product images simultaneously. Lambda is triggered for each. What happens?**

- A) Lambda queues the requests and processes them one at a time
- B) Lambda runs up to 500 parallel instances simultaneously — each image is processed independently
- C) Lambda throws an error — it can only process 10 concurrent requests
- D) Lambda processes 10 at a time and waits for each batch to finish

> **Answer/explanation:** ✅ Answer: **B — Up to 500 parallel Lambda instances run simultaneously**
Lambda scales horizontally with no configuration. Each invocation is completely independent.
Default **concurrency limit** is 1,000 per account per region. All 500 images process in parallel instantly.
Each invocation is billed separately — 500 × 800ms × 512MB would cost approximately ₹0.50 total.
This is the fundamental advantage over EC2: no warm-up, no queue, no configuration — instant scale to demand.

**Quiz: Quiz 2 — A DynamoDB table has userId as partition key and orderId as sort key. Which query is efficient?**

- A) Get all orders where status = "pending" (across all users)
- B) Get all orders for userId = "user-123"
- C) Get the 10 highest-value orders across all users
- D) Count total orders placed today across all users

> **Answer/explanation:** ✅ Answer: **B — Get all orders for a specific userId**
DynamoDB Query uses the partition key to go directly to the right partition — reads only items for user-123. Fast and cheap.
Options A, C, and D require **Scan** — reads every item in the table. Very slow and expensive for large tables.
To efficiently query by status: create a **Global Secondary Index (GSI)** with status as partition key.
Design rule: **know your access patterns before designing your DynamoDB schema.** Design the keys around how you query, not how data is logically structured.

**Quiz: Quiz 3 — ShopNest stores the RDS password in an EC2 environment variable. What is the security problem and the correct fix?**

- A) No problem — environment variables are encrypted automatically by EC2
- B) The password is visible in plaintext to anyone who can SSH into the instance, check process environment, or read EC2 user data. Fix: store in Secrets Manager and fetch at runtime via IAM role.
- C) Environment variables are too slow. Fix: hardcode the password in the application file.
- D) The only fix is to not use passwords — use only SSL certificates.

> **Answer/explanation:** ✅ Answer: **B — Plaintext password in environment variable, fix with Secrets Manager**
Environment variables on EC2 are readable by: any process on the instance, anyone who SSHs in, anyone who reads the User Data, and CloudFormation/CDK logs if set there.
**Secrets Manager**: the password is stored encrypted (AES-256 via KMS). The application calls the API at startup. Even if an attacker gets into the EC2, they'd need the IAM role and Secrets Manager API access — a separate layer.
Additionally, Secrets Manager can rotate the password automatically every 30 days. An old leaked password stops working automatically.
This exact scenario (leaked credentials in environment variables or config files) is in the OWASP Top 10 and is tested in AWS Security Specialty exam.

> **Advanced AWS — Core Takeaways**

> - **Lambda is not a replacement for EC2** — it's for event-driven, short-duration, intermittent workloads. Long-running, always-on services still belong on EC2 or ECS. Match the service to the workload.
> - **ECS Fargate removes operational overhead** — you write Dockerfiles and deploy task definitions. AWS manages all the underlying infrastructure. Great for teams that want containers without platform engineering burden.
> - **DynamoDB access patterns must be designed upfront** — unlike relational databases where you can add indexes later without much penalty, bad DynamoDB table design is expensive to fix at scale. Model your data around your queries.
> - **CodePipeline makes deployment boring** — that's the goal. Automated pipelines mean deployments are the same every time: no missed steps, no 11 PM mistakes, full audit trail. Boring deployments are good deployments.
> - **Secrets Manager is not optional for production** — any database password, API key, or certificate that lives in a config file, environment variable, or EC2 user data is a security liability. Move it to Secrets Manager.
> - **SSM Session Manager eliminates PEM file sharing** — shared PEM files with no audit trail are an operational risk. Session Manager gives per-engineer access controlled by IAM with every command logged.
> - **CloudFront dramatically improves user experience** — a 94% cache hit rate means 94% of your users get images in milliseconds instead of seconds. It also reduces S3 data transfer costs significantly.
> - **SQS decouples services** — never have one service call another synchronously in production for non-critical paths. Use SQS in between. If the downstream service is slow or down, the queue buffers the load without affecting the upstream service.

##### Advanced AWS Engineering Standards — ShopNest Rules

- Always set Lambda **reserved concurrency** for critical functions — prevents a traffic spike on one function from consuming all account-level concurrency and throttling other functions
- Enable **ECR image scanning on push** for all repositories — automatically detects known CVE vulnerabilities in Docker images before they reach production
- Set **DynamoDB TTL** on session data, temporary tokens, and cart items — removes expired data automatically with no cost, keeping the table lean and fast
- Use **CodePipeline approval stages** before production deployments — even a 2-click approval from a senior engineer prevents accidental production pushes from unreviewed feature branches
- Store **non-sensitive config in SSM Parameter Store** (free) and **secrets in Secrets Manager** (paid) — don't pay for Secrets Manager to store values like ENVIRONMENT=production or LOG_LEVEL=info
- Enable **CloudFront access logging** to S3 — you get full logs of every edge request, invaluable for debugging, cache analysis, and security monitoring

##### 🏋️ Hands-On Exercises — Extend ShopNest Advanced Services

1. **Build an order notification system with SNS + SQS + Lambda:** Create an SNS topic. Create two SQS queues: one for the email service, one for the inventory service. Subscribe both to the SNS topic. Write two Lambda functions — one reads from each queue. Publish a test order event to SNS and verify both Lambda functions receive it independently. This is the fan-out pattern used by every large e-commerce platform.
2. **Add a DynamoDB GSI for order queries:** ShopNest needs to query all orders by status ("pending", "shipped", "delivered") regardless of userId. Add a Global Secondary Index with `status` as partition key and `createdAt` as sort key. Test querying all "pending" orders sorted by creation date. Observe that the query uses the GSI, not a Scan.
3. **Create a Lambda Layer for Python dependencies:** The image resize function uses Pillow. Package Pillow into a Lambda Layer (a zip file with a python/ folder). Publish the layer. Attach it to the Lambda function. Remove Pillow from the function's deployment package. Verify the function still works. Layers can be shared across multiple Lambda functions — update once, all functions get the update.
4. **Implement ECS Service Auto Scaling:** Configure Application Auto Scaling on the ECS Fargate recommendations service. Set minimum 1 task, maximum 10 tasks. Add a target tracking policy: scale when ALBRequestCountPerTarget exceeds 100 requests per minute per task. Use the AWS Load Balancer testing tool to simulate traffic and watch the service scale from 1 to multiple tasks, then scale back in after load drops.
5. **Build a CloudFront invalidation into CodePipeline:** After CodePipeline deploys new frontend assets to S3, add a final pipeline stage that runs a Lambda function to create a CloudFront invalidation for `/*`. Without this, users get the cached old version from edge locations even after a new deployment. Automating invalidation in the pipeline means users always get the latest version within seconds of deployment.

### Advanced AWS Module Complete 🎉

You have mastered the advanced AWS services that companies use to build modern cloud-native applications — Lambda and API Gateway for serverless, ECS Fargate and ECR for containers, DynamoDB for NoSQL at scale, CodePipeline for automated CI/CD, Secrets Manager for security, SSM for fleet management, and CloudFront for global content delivery.

> **Nandita**
> 
> "Karthik, six months ago we were manually deploying code by SSHing into EC2 with a shared PEM file, storing database passwords in config files, and paying for EC2 instances that sat idle 23 hours a day running background jobs. Today: every deployment is automated through CodePipeline, passwords are in Secrets Manager with automatic rotation, Lambda costs ₹5 a month for 2 million image resize jobs, and our product images load in 58 milliseconds for customers across India. This is what cloud engineering looks like when done right."

> **Suresh**
> 
> "The security audit passed for the first time in company history. No plaintext secrets anywhere. No SSH keys shared between engineers. Every EC2 connection goes through SSM with full audit logging. The auditor said our AWS setup is more secure than companies ten times our size. That's not because we have a bigger team — it's because we used the right tools."

> **Next: AWS Well-Architected Framework & Cost Optimisation**

> - Operational Excellence — infrastructure as code, CI/CD pipelines, runbooks, and post-incident reviews
> - Security — IAM least privilege, encryption at rest and in transit, GuardDuty threat detection, Security Hub
> - Reliability — multi-AZ, backup strategies, disaster recovery with RTO/RPO targets, AWS Resilience Hub
> - Performance Efficiency — choosing the right instance family, ElastiCache for query caching, read replicas
> - Cost Optimisation — Savings Plans, Spot Instances, rightsizing EC2, S3 Intelligent-Tiering, Cost Anomaly Detection
> - Sustainability — ARM-based Graviton instances (40% cheaper, 60% more energy-efficient than x86)
