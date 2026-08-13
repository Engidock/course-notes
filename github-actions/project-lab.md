# GitHub Actions Project Mastery

> **👋 Hey Fresher — Read This First!**

> GitHub Actions is the automation system built inside GitHub. Every time you push code, it can automatically run tests, build a Docker image, and deploy to your server — without you doing anything manually. This document uses **short YAML blocks** — each one covers exactly one concept — with a plain-English explanation right beside it. No 80-line workflow files to get lost in. One idea at a time, always explained in simple language.

> **Company in this project:** NovaPay — the same fintech startup from the Kubernetes module. They have a Node.js payment API and they need every code push to automatically test, build, and deploy to the Kubernetes cluster you set up earlier. You're still the Junior DevOps Engineer. Your lead is still Tanvi. Let's build their CI/CD pipeline.

#### What You Will Learn and Build in This Project

You will write GitHub Actions workflow files that automate NovaPay's entire software delivery — from code push to production deployment.

Workflows, Jobs, Steps, Secrets, Environments, Matrix Builds, Docker Build, K8s Deploy, Caching

> **🔧 Phase 1 — Foundations**

> Understand what GitHub Actions is, how workflows work, and write your first automated workflow that runs on every push.

> Run tests, lint code, and build the Docker image automatically every time someone opens a pull request.

> Build a Docker image and push it to Docker Hub (or GitHub Container Registry) as part of the automated pipeline.

> Deploy the new image to the NovaPay Kubernetes cluster automatically after every merge to main — with environment protection.

**Scene 1 — NovaPay Engineering Office | "Why Is Deploying So Slow?"**

> **Tanvi** _Senior Platform Engineer — NovaPay_
> 
> Priya, right now deploying NovaPay takes 45 minutes and four people. A developer pushes code. Someone manually runs tests. Someone else builds the Docker image. A third person SSHes into the cluster and runs kubectl apply. If anyone makes a mistake, we roll back manually. We do this six times a day. It's killing us.

> **Priya (You)** _Junior DevOps Engineer_
> 
> Can we automate all of that?

> **Tanvi** _Senior Platform Engineer_
> 
> That is exactly what GitHub Actions does. You write a YAML file that says "when code is pushed to main, do these steps in this order." From that moment on, every deployment is automatic, consistent, and auditable. No humans in the loop. No manual mistakes. Developer pushes code → tests run → image builds → cluster updates → done. In under 5 minutes.

> **Roshan** _DevOps Architect — NovaPay_
> 
> Think of it this way: GitHub Actions is a robot that watches your repository. You give it a set of instructions in a YAML file. Every time something happens — a push, a pull request, a release — it follows those instructions exactly. Never forgets a step. Never has a bad day. Always works the same way.

### 1. Phase 1 — GitHub Actions Foundations

Before writing any pipeline, understand how GitHub Actions is structured and how every workflow file is organised.

> **The Big Picture — How GitHub Actions Works**

> You create a file in your repository at `.github/workflows/pipeline.yml`. GitHub reads it automatically — no setup required. When the event you specified happens (a push, a PR, a schedule), GitHub starts a **Runner** — a temporary virtual machine — and executes every step in your workflow. The runner disappears when the workflow finishes. Each run is completely fresh. You pay nothing for public repos; private repos get 2,000 free minutes/month on the free plan.

```
GitHub Actions Architecture — NovaPay Pipeline
===============================================

  Developer pushes code
         │
         ▼
  GitHub detects push event
         │
         ▼
  ┌─────────────────────────────────────┐
  │         WORKFLOW FILE               │
  │  .github/workflows/deploy.yml       │
  │                                     │
  │  ┌────────┐  ┌────────┐  ┌───────┐ │
  │  │ Job 1  │→ │ Job 2  │→ │ Job 3 │ │
  │  │  test  │  │ build  │  │deploy │ │
  │  └────────┘  └────────┘  └───────┘ │
  │   Step 1       Step 1      Step 1  │
  │   Step 2       Step 2      Step 2  │
  └─────────────────────────────────────┘
         │
         ▼
  GitHub-hosted Runner (Ubuntu VM)
  Executes each step → passes result → next job
         │
         ▼
  Kubernetes Cluster updated ✓
```

#### 1.1 Workflow File Location and Structure

1. Where to put the file
Every workflow YAML lives inside `.github/workflows/` in your repository root. GitHub discovers and runs all files in that folder automatically.

```
# Your repository structure
novapay-api/
├── .github/
│   └── workflows/
│       ├── ci.yml       # runs on every PR
│       └── deploy.yml   # runs on merge to main
├── src/
└── package.json
```

> GitHub scans the **.github/workflows/** folder on every event. You can have multiple workflow files — one for CI (testing PRs) and one for CD (deploying main). Keep them separate so a failed deployment doesn't block your test run, and vice versa.

#### 1.2 The 5 Parts of Every Workflow

```
name: NovaPay CI
on:
  push:
    branches: [main]
```

**📖 name + on (the trigger)**

**name** — what you call this workflow. Shows up in the GitHub Actions tab.  
  
**on** — the trigger. This says "run this workflow when code is pushed to the main branch." Without `on`, the workflow never runs. It's like a switch that turns the automation on.

```
jobs:
  test:
    runs-on: ubuntu-latest
```

**📖 jobs + runs-on (the machine)**

**jobs** — a workflow has one or more jobs. Each job runs independently on its own machine.  
  
**runs-on: ubuntu-latest** — tells GitHub which operating system to use. GitHub provides free Ubuntu, Windows, and macOS runners. Ubuntu is the standard for DevOps pipelines.

```
 steps:
      - name: Checkout code
uses: actions/checkout@v4

      - name: Run tests
run:  npm test
```

**📖 steps (the actual commands)**

**steps** — the list of things to do inside a job. Each step is either:  
  
**uses:** — a pre-built Action (someone else's reusable automation, like checking out code)  
  
**run:** — a shell command you type yourself, like `npm test` or `docker build`.  
  
Steps run in order. If one fails, all following steps are skipped.

> **💡 Fresher Tip — What is actions/checkout@v4?**

> The runner (GitHub's VM) starts completely empty — it doesn't have your code. `actions/checkout@v4` is a pre-built step that clones your repository onto the runner. **This is always the first step in every job.** Without it, the runner has no code to test, build, or deploy. Think of it as "download my code so we can work with it."

#### 1.3 Common Triggers — When Should the Workflow Run?

```
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

**📖 The Two Most Common Triggers**

**push: branches: [main]** — runs when someone merges code into main. Used for deployment workflows.  
  
**pull_request: branches: [main]** — runs when someone opens or updates a PR targeting main. Used for CI (test before merging). This is how you prevent broken code from reaching main.

```
on:
  schedule:
    - cron: "0 2 * * *"
workflow_dispatch: # manual trigger
```

**📖 Other Useful Triggers**

**schedule.cron** — run on a timer. `"0 2 * * *"` means "every day at 2 AM UTC." NovaPay uses this to run a nightly security scan on their dependencies.  
  
**workflow_dispatch** — adds a "Run workflow" button in GitHub's UI. Useful for manual deployments, emergency rollbacks, or running a job on demand without pushing code.

### 2. Phase 2 — CI Pipeline (Test on Every Pull Request)

**Business Problem:** A NovaPay developer once merged a PR that broke the payment validation logic. It passed code review but nobody ran the tests. The bug reached production and caused 12 failed transactions before someone noticed. Automated CI ensures tests run on every PR — broken code simply cannot be merged.

**Scene 2 — NovaPay Code Review | "Who Ran the Tests?"**

> **Tanvi** _Senior Platform Engineer_
> 
> Vikram, the PR you merged last Thursday broke payment validation. Tests were passing locally on your machine, but you didn't run the full suite. Twelve payments failed. I need a system where this is physically impossible — where the PR cannot be merged unless all tests pass. Automatically. Without anyone having to remember to run them.

> **Priya (You)** _Junior DevOps Engineer_
> 
> I can write a GitHub Actions workflow that runs npm test automatically on every PR. And we can set it as a required status check — GitHub won't let you merge unless it passes.

#### 2.1 Your First CI Workflow

```
name: CI — Test & Lint
on:
  pull_request:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
```

**📖 The CI Trigger**

This workflow runs every time a PR is opened or updated that targets the `main` branch.  
  
The job is called `test` (you choose the name) and runs on Ubuntu. GitHub shows this job's pass/fail status directly on the PR page — with a green checkmark or red ❌.  
  
In GitHub settings, you can make this a **required status check** — merge button stays greyed out until it passes.

```
 steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
uses: actions/setup-node@v4
with:
          node-version: "20"
```

**📖 Setting Up Node.js**

**actions/setup-node@v4** installs Node.js on the runner. The `with:` block passes configuration to the action.  
  
**node-version: "20"** — pins the exact Node version. This guarantees tests run on the same version used in production. Without this, the runner uses whatever Node version GitHub happened to install — which might be different.

```
      - name: Install dependencies
run:  npm ci

      - name: Run tests
run:  npm test

      - name: Lint code
run:  npm run lint
```

**📖 The Test Steps**

**npm ci** (not npm install) — installs exact versions from package-lock.json. Faster and more reliable in CI because it never upgrades packages.  
  
**npm test** — runs your test suite. If any test fails, the step exits with an error code and GitHub marks the job as failed.  
  
**npm run lint** — checks code style. Catches unused variables, wrong formatting, common mistakes before they hit review.

#### 2.2 Dependency Caching — Make CI Faster

```
      - name: Cache node_modules
uses: actions/cache@v4
with:
          path: ~/.npm
key:  ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
```

**📖 Caching — Don't Re-download Packages Every Time**

Without caching, `npm ci` downloads all packages from the internet on every run — even when nothing changed. That's 30–60 seconds wasted every time.  
  
**actions/cache@v4** saves `~/.npm` between runs. The **key** includes a hash of package-lock.json — if dependencies didn't change, the cache hits and packages are restored instantly. NovaPay went from 4-minute CI runs to 90 seconds just by adding this.

#### 2.3 Job Dependencies — Run Jobs in the Right Order

```
jobs:
  test:
    runs-on: ubuntu-latest
steps: [...]

  build:
    runs-on:  ubuntu-latest
needs:    test
steps: [...]
```

**📖 needs — Job Order Control**

**needs: test** means "only start the `build` job after the `test` job passes successfully."  
  
Without `needs`, all jobs run in parallel at the same time. With `needs`, you create a sequence: test → build → deploy.  
  
If `test` fails, `build` never starts. This saves money (no point building a broken image) and prevents broken code from being deployed.

### 3. Phase 3 — Build & Push Docker Image

**Business Problem:** After tests pass, NovaPay needs a Docker image built and pushed to a registry so Kubernetes can deploy it. Previously a developer did this manually on their laptop. Different machines → slightly different images → hard-to-reproduce bugs. GitHub Actions builds the image in a clean, identical environment every single time.

**Scene 3 — NovaPay Slack | "It Works on My Machine"**

> **Vikram** _Backend Developer — NovaPay_
> 
> The Docker image I built locally works fine. But the image that went to production keeps crashing on startup. I don't understand — it's the same code.

> **Tanvi** _Senior Platform Engineer_
> 
> Your laptop is macOS ARM. Production is Linux x86. The node_modules you installed locally include some native binaries compiled for your Mac. When you build the image on your Mac, those Mac binaries go in. The Linux container can't run them. If we build the image in GitHub Actions — which runs Linux — the binaries are compiled for Linux. Problem solved. And it's the same image every time, built from the same clean environment.

#### 3.1 Log In to Docker Hub

```
      - name: Log in to Docker Hub
uses: docker/login-action@v3
with:
          username: ${{ secrets.DOCKER_USERNAME }}
password: ${{ secrets.DOCKER_PASSWORD }}
```

**📖 Logging In Securely with Secrets**

**secrets.DOCKER_USERNAME** — this is NOT a hardcoded value. It's a reference to a Secret stored in GitHub (Settings → Secrets → Actions). The actual value is never visible in the YAML or logs.  
  
If you put your Docker password directly in the YAML file and commit it, anyone who views your repo (or its history) can steal it. Secrets keep credentials out of code entirely.

#### 3.2 Build and Push the Image

```
      - name: Build and push image
uses: docker/build-push-action@v5
with:
          push:  true
tags:  novapay/payment-api:${{ github.sha }}
```

**📖 Build + Push in One Step**

**push: true** — build the image AND push it to Docker Hub automatically.  
  
**github.sha** — a unique ID for this exact commit (e.g. `a3f8c1d`). Using it as the image tag means every commit gets its own image. You can always roll back to a previous image by its SHA.  
  
Never tag production images with `:latest` — it makes rollbacks impossible because you can't tell what version "latest" is.

```
      - name: Set up Docker Buildx
uses: docker/setup-buildx-action@v3

      - name: Cache Docker layers
uses: actions/cache@v4
with:
          path: /tmp/.buildx-cache
key:  buildx-${{ github.sha }}
```

**📖 Why Buildx + Cache?**

**setup-buildx-action** enables advanced Docker building features including layer caching and multi-platform builds.  
  
**Caching Docker layers** means if only your app code changed (not your base image or dependencies), Docker reuses cached layers and only rebuilds the changed parts. NovaPay's image build time dropped from 4 minutes to 45 seconds after adding this.

> **🔐 Never Hardcode Credentials in Workflow Files**

> A workflow YAML is committed to your Git repository. Anyone with repository access — and anyone who ever gains access — can read it. That means credentials in YAML are permanently compromised. Always use `${{ secrets.NAME }}` for passwords, API keys, tokens, and certificates. Store secrets in GitHub Settings → Secrets and variables → Actions. GitHub masks secret values in workflow logs automatically — even if you accidentally print one with `echo`, it appears as `***`.

### 4. Phase 4 — Deploy to Kubernetes

**Business Problem:** After the Docker image is built and pushed, NovaPay's Kubernetes Deployment needs to be updated to use the new image. Previously this was a manual `kubectl set image` command run by whoever was on-call. Automated CD means the cluster is always running whatever was last merged to main — no human needed, no step forgotten.

**Scene 4 — NovaPay Production Incident | "Who Deployed?"**

> **Roshan** _DevOps Architect_
> 
> We have a production issue. Payments are failing. I need to know — what version is running right now, who deployed it, and when? With our current manual process, nobody keeps records. Someone ran kubectl apply from their laptop and nobody logged it.

> **Tanvi** _Senior Platform Engineer_
> 
> With GitHub Actions, every deployment is a workflow run. GitHub logs who triggered it, when it ran, what commit it deployed, and every command it executed. It's a complete audit trail. And because deployments only happen through the workflow — never from someone's laptop — you always know exactly what's running and why.

#### 4.1 Give GitHub Actions Access to Kubernetes

```
      - name: Set up kubectl
uses: azure/setup-kubectl@v3

      - name: Configure kubeconfig
run: echo "${{ secrets.KUBECONFIG }}" | base64 -d > ~/.kube/config
```

**📖 Connecting the Runner to Your Cluster**

**azure/setup-kubectl@v3** installs kubectl on the GitHub runner.  
  
The runner needs credentials to talk to your cluster. You export the kubeconfig file, base64-encode it, and store it as a GitHub Secret. During the workflow, it's decoded back into `~/.kube/config` — the same file kubectl uses on your laptop. From this point, all kubectl commands in this job target your production cluster.

#### 4.2 Update the Deployment

```bash
      - name: Deploy to Kubernetes
run: |
          kubectl set image deployment/novapay-api \
            api=novapay/payment-api:${{ github.sha }} \
            -n novapay-prod
          kubectl rollout status deployment/novapay-api \
            -n novapay-prod
```

**📖 The Deployment Step**

**kubectl set image** updates the Deployment to use the new Docker image (tagged with the commit SHA built earlier).  
  
**kubectl rollout status** watches the rolling update and waits until all new Pods are healthy. If the new Pods crash, this command exits with an error — which fails the workflow and alerts the team. If you don't include this, the workflow finishes "successfully" even if the deployment is broken.

#### 4.3 Environments — Protect Production with Required Approvals

```
jobs:
  deploy-prod:
    runs-on:    ubuntu-latest
environment: production
needs:       deploy-staging
steps: [...]
```

**📖 Environments — Manual Approval Gate**

**environment: production** links this job to a GitHub Environment named "production."  
  
In GitHub Settings → Environments, you configure "production" to require approval from specific reviewers before the job runs. The workflow pauses, sends a notification, and waits. A senior engineer reviews and clicks "Approve." Only then does the deployment to prod proceed.  
  
NovaPay's rule: staging deploys automatically; production requires Tanvi's approval.

> **💡 Fresher Tip — Staging Before Production, Always**

> A mature CD pipeline deploys to **staging first**, automatically. Staging is a copy of production with real-looking data. If something breaks in staging, nobody's payment is affected. Only after staging is healthy (and after approval) does the workflow deploy to production. This is the difference between a junior pipeline and a professional one.

### 5. Phase 5 — Secrets, Environments & Matrix Builds

#### 5.1 GitHub Secrets — The Right Way to Store Credentials

```
# Referencing secrets in a workflow
env:
  RAZORPAY_KEY: ${{ secrets.RAZORPAY_KEY }}
DB_PASSWORD:  ${{ secrets.DB_PASSWORD }}
```

**📖 Secrets as Environment Variables**

You can expose secrets as environment variables for steps that need them. The app reads them like normal env vars (`process.env.RAZORPAY_KEY`).  
  
GitHub automatically masks secret values in logs. Even `run: echo $RAZORPAY_KEY` prints `***` — the real value is never exposed in the Actions log, even to administrators.

```
# Environment-scoped secrets
jobs:
  deploy:
    environment: production
steps:
      - run: deploy.sh
env:
          KEY: ${{ secrets.PROD_API_KEY }}
```

**📖 Environment Secrets — Prod vs Staging Keys**

GitHub lets you store different secrets per environment. `secrets.PROD_API_KEY` is only available when the job uses `environment: production`.  
  
NovaPay stores Razorpay's test key under the "staging" environment and the live key under "production." The workflow file is identical — but it automatically uses the right key based on which environment the job is running in. No if/else logic needed.

#### 5.2 Matrix Builds — Test Against Multiple Versions at Once

```
jobs:
  test:
    strategy:
      matrix:
        node-version: [18, 20, 22]
    runs-on: ubuntu-latest
```

**📖 Matrix — Run the Same Job Multiple Times**

**strategy.matrix** creates multiple parallel runs of the same job, each with a different value.  
  
This creates 3 jobs: one testing with Node 18, one with Node 20, one with Node 22 — all running at the same time in parallel. If the code breaks on Node 18 but works on 20, you see exactly which version failed. Takes the same time as one test run.

```
 steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

**📖 Using Matrix Values in Steps**

**matrix.node-version** gives you the current value for this particular run. In the Node 18 job it's `18`, in the Node 20 job it's `20`.  
  
The same workflow step works for all matrix entries — you write the step once and the matrix handles the variation. This is far cleaner than copy-pasting three separate jobs.

#### 5.3 Conditional Steps — Skip or Run Based on Context

```
      - name: Deploy to prod
if: github.ref == 'refs/heads/main'
run: ./deploy.sh

      - name: Notify on failure
if: failure()
run: ./notify-slack.sh
```

**📖 if — Conditional Logic**

**if: github.ref == 'refs/heads/main'** — only run this step when the workflow is triggered by the main branch. Useful in a workflow triggered by both push and PR — the deploy step only runs for actual merges, not for every PR.  
  
**if: failure()** — run this step only if a previous step failed. NovaPay uses this to send a Slack alert whenever a production deployment fails.

### 6. Putting It All Together — NovaPay Complete Pipeline

Here's how all the pieces connect. Each short block shows one stage of the full CI/CD pipeline from push to production.

```
NovaPay Complete GitHub Actions Pipeline
==========================================

  Developer pushes to main
         │
         ▼
  ┌─────────────┐      ┌──────────────┐      ┌────────────────┐
  │   JOB: test │ ───▶ │  JOB: build  │ ───▶ │ JOB: deploy-   │
  │             │      │              │      │    staging      │
  │ checkout    │      │ checkout     │      │                │
  │ setup-node  │      │ docker login │      │ kubectl set    │
  │ npm ci      │      │ docker build │      │ image...       │
  │ npm test    │      │ docker push  │      │ rollout status │
  │ npm lint    │      │ :sha tag     │      └────────┬───────┘
  └─────────────┘      └──────────────┘               │
                                               Staging healthy?
                                                       │
                                                       ▼
                                         ┌─────────────────────────┐
                                         │   JOB: deploy-prod      │
                                         │   environment:production│
                                         │   ← Requires approval   │
                                         │                         │
                                         │   Tanvi approves ✓      │
                                         │   kubectl set image...  │
                                         │   rollout status        │
                                         └─────────────────────────┘
```

```
name: NovaPay CD Pipeline
on:
  push:
    branches: [main]
env:
  IMAGE: novapay/payment-api
SHA:   ${{ github.sha }}
```

**📖 Workflow-Level env Variables**

Defining `env` at the top level makes those variables available to every job and step in the workflow.  
  
`IMAGE` and `SHA` are used in both the build job (for tagging) and the deploy jobs (for kubectl). Setting them once at the top prevents copy-paste mistakes and makes updates easy.

```
jobs:
  test:
    runs-on: ubuntu-latest
steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
with: { node-version: "20" }
      - run: npm ci && npm test
```

**📖 Job 1 — Test**

The first job. Every push to main runs this first. If tests fail, the entire pipeline stops here — the image is never built and never deployed. This is the safety gate.  
  
`npm ci && npm test` — the `&&` means "only run npm test if npm ci succeeds." One line, two commands, fail fast.

```
 build:
    needs:    test
runs-on: ubuntu-latest
steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
with:
          username: ${{ secrets.DOCKER_USERNAME }}
password: ${{ secrets.DOCKER_PASSWORD }}
      - uses: docker/build-push-action@v5
with:
          push: true
tags: ${{ env.IMAGE }}:${{ env.SHA }}
```

**📖 Job 2 — Build & Push**

**needs: test** — only starts after test passes.  
  
Logs in to Docker Hub using secrets (never hardcoded), then builds and pushes the image tagged with the commit SHA.  
  
The image tag is `novapay/payment-api:a3f8c1d` — where `a3f8c1d` is the unique git commit ID. This means you can always trace "which image is running" back to "which exact commit."

```bash
 deploy-prod:
    needs:       deploy-staging
environment: production
runs-on:    ubuntu-latest
steps:
      - uses: azure/setup-kubectl@v3
      - run: echo "${{ secrets.KUBECONFIG }}" | base64 -d > ~/.kube/config
      - run: |
          kubectl set image deployment/novapay-api \
            api=${{ env.IMAGE }}:${{ env.SHA }} -n novapay-prod
          kubectl rollout status deployment/novapay-api -n novapay-prod
```

**📖 Job 3 — Deploy to Production**

**needs: deploy-staging** — only runs after staging deployment succeeds.  
  
**environment: production** — pauses and waits for a reviewer to approve in GitHub UI.  
  
Once approved: installs kubectl, authenticates to the cluster, updates the Deployment image, and waits for the rollout to confirm all Pods are healthy. If the rollout fails, the workflow fails and the team is alerted.

### 7. Essential GitHub Actions Reference

Concept

What It Does

When to Use It

on: push

Trigger on code push

CD pipelines — deploy when code lands on main

on: pull_request

Trigger on PR open/update

CI pipelines — test before code is merged

on: workflow_dispatch

Manual trigger button in GitHub UI

Emergency rollbacks, on-demand jobs

runs-on: ubuntu-latest

Use GitHub's free Ubuntu runner

Almost every job — it's free and fast

uses: actions/checkout@v4

Clone the repository onto the runner

First step of every single job, always

needs: job-name

Wait for another job to pass first

test → build → staging → prod ordering

environment: production

Require manual approval before running

Any job that touches production

secrets.NAME

Inject a stored secret safely

Any password, key, token, or credential

github.sha

The commit ID of the current run

Docker image tags for traceability

strategy.matrix

Run job N times with different values

Test across multiple Node/Python versions

if: failure()

Only run if a previous step failed

Slack/email alerts on failure

actions/cache@v4

Save and restore files between runs

node_modules, pip packages, Docker layers

### 8. Interview Questions — GitHub Actions

##### Interview Q&A — Fresher Level (0–1 Year CI/CD Experience)

**Q: Q1. What is the difference between a Workflow, a Job, and a Step?**

A: A Workflow is the entire automation file — it's the YAML file in .github/workflows/. It defines when automation runs (the trigger) and what it does. A Job is a group of steps that run on one machine (runner). A workflow can have multiple jobs. If a workflow has a "test" job and a "deploy" job, each runs on its own separate, fresh Ubuntu machine. A Step is one individual action inside a job — either a pre-built Action (uses:) or a shell command (run:). Steps within a job share the same machine and can pass files to each other. Steps across different jobs cannot share files directly because they run on different machines.

**Q: Q2. How do you pass sensitive credentials like passwords and API keys in GitHub Actions?**

A: Using GitHub Secrets. Never hardcode a password or API key directly in the workflow YAML file — the YAML is committed to the repository and visible to anyone with access (and potentially forever in git history). Instead, go to GitHub Settings → Secrets and variables → Actions, and add the secret with a name like DOCKER_PASSWORD. In the workflow, reference it as ${{ secrets.DOCKER_PASSWORD }}. GitHub masks the secret value in all workflow logs, so even if a step accidentally prints it, it shows as ***. For environment-specific secrets (different keys for staging vs production), use GitHub Environments — each environment can have its own set of secrets accessible only when a job uses that environment.

**Q: Q3. What does `needs` do in a GitHub Actions workflow?**

A: The needs keyword creates a dependency between jobs. By default, all jobs in a workflow run in parallel at the same time. needs overrides this — needs: test means "don't start this job until the test job has finished successfully." This creates a sequential pipeline: test → build → deploy. If the test job fails, the build job never starts, which means you never waste time building or deploying broken code. You can chain multiple jobs: deploy-prod can need deploy-staging, which needs build, which needs test — creating a fully ordered pipeline.

**Q: Q4. What is a GitHub Actions Environment and why use it?**

A: A GitHub Environment is a named deployment target (like "staging" or "production") with its own settings. The key feature is protection rules: you can configure an environment to require approval from specific people before a job using that environment starts. The workflow pauses and sends a notification. The designated reviewer (a senior engineer or tech lead) reviews what's about to be deployed and clicks Approve. Only then does the job run. Environments also have their own scoped secrets, so the production Razorpay key is only accessible to jobs targeting the production environment — even if a developer can run the staging deployment, they can't access the production API key.

**Q: Q5. What is the purpose of `actions/checkout` and why is it always the first step?**

A: actions/checkout clones your GitHub repository onto the runner machine. GitHub's runner starts as a completely blank Ubuntu virtual machine — it has no files from your repository. Without checkout, there is no code, no package.json, no Dockerfile — nothing to build, test, or deploy. It must be the first step in every job that needs your code. One important subtlety: because each job runs on a separate machine, every job that needs your code must include checkout independently — even if a previous job already ran it.

**Q: Q6. How does GitHub Actions CI prevent bad code from reaching production?**

A: There are two layers. First, you add a workflow triggered by pull_request that runs your tests automatically. Every PR gets a status check showing pass or fail directly on the PR page. Second, in GitHub repository Settings → Branches → Branch protection rules, you add the CI job as a required status check for the main branch. With this setting, GitHub's merge button is completely greyed out if the status check is failing — it is physically impossible to merge without tests passing. No human can accidentally bypass it. This means the only code that can ever reach main (and therefore production) is code that has passed the full automated test suite.

**Quiz: Quiz 1 — A developer pushes to main. The test job passes but the build job fails with "denied: requested access to the resource is denied." What is the likely cause?**

- A) The Dockerfile has a syntax error
- B) The Docker Hub credentials stored in GitHub Secrets are wrong or missing
- C) The runner doesn't have internet access
- D) The tests are not actually passing, the log is wrong

> **Answer/explanation:** ✅ Answer: B. "Access denied" when pushing to a Docker registry means authentication failed. The `docker/login-action` step uses `secrets.DOCKER_USERNAME` and `secrets.DOCKER_PASSWORD` — check that both are set correctly in GitHub Settings → Secrets. Common mistakes: the secret is set at the repository level but the job uses an environment that has its own secrets (overriding the repo-level ones), or the Docker Hub password was recently rotated and the secret wasn't updated. Always verify by checking if the login step specifically shows an error, distinct from the push step.

**Quiz: Quiz 2 — Your deploy job runs before your test job finishes. What is missing from your workflow?**

- A) The workflow is missing `on: push`
- B) The deploy job is missing `needs: test`
- C) The test job needs `runs-on: ubuntu-latest`
- D) You need to add `if: success()` to the deploy job

> **Answer/explanation:** ✅ Answer: B. Without `needs: test`, GitHub runs all jobs in parallel by default. The deploy job starts at the same time as the test job — it doesn't wait. Adding `needs: test` to the deploy job creates the dependency: "wait until test passes, then start deploying." Without it, you can deploy code that hasn't been tested yet, which defeats the entire purpose of CI/CD.

**Quiz: Quiz 3 — What does `github.sha` give you and why use it for Docker image tags?**

- A) The SHA of the workflow file — used to check if the pipeline changed
- B) A random ID generated fresh each run — used to ensure uniqueness
- C) The git commit ID — used to tag images so each commit maps to exactly one image
- D) The SHA of the Docker image — used to verify the image wasn't tampered with

> **Answer/explanation:** ✅ Answer: C. `github.sha` is the full git commit SHA (a 40-character hex string like `a3f8c1d...`) for the commit that triggered the workflow. Using it as a Docker tag means every commit produces exactly one uniquely tagged image. If production is running `novapay/payment-api:a3f8c1d` and something breaks, you know exactly which commit is deployed. You can check git log for that SHA, see what changed, and roll back to the previous commit's image instantly. Never use `:latest` in production — it's meaningless for rollbacks because you can't tell what version "latest" pointed to before the update.

> **GitHub Actions Project — Core Takeaways for Freshers**

> - A workflow file is a set of instructions GitHub follows automatically when something happens in your repository. It lives in .github/workflows/ and you never have to "register" or "configure" it separately — GitHub finds it automatically.
> - Never put credentials, passwords, or API keys directly in a YAML file. Always use GitHub Secrets (${{ secrets.NAME }}). Secrets in code are a security incident waiting to happen — and git history is forever.
> - Always use `actions/checkout@v4` as the first step in every job. Runners start empty — no code, no files. Without checkout, every subsequent step fails.
> - Use `needs:` to create ordered pipelines. Without it, all jobs run in parallel. test → build → deploy must be sequential — never deploy before tests pass.
> - Tag Docker images with `github.sha`, never with `:latest`. Traceability is non-negotiable in production — you must be able to answer "which commit is running right now?" at any moment.
> - Use GitHub Environments with required reviewers for any job that touches production. Automated deployment is powerful; automated deployment to production without a human check is dangerous.

##### GitHub Actions Standards — NovaPay Platform Engineering Rules

- Pin all Action versions to a specific tag (actions/checkout@v4, not actions/checkout@latest) — unpinned actions can change behaviour on any day, breaking your pipeline without any change in your code
- Always add `kubectl rollout status` after `kubectl set image` — without it, the workflow reports success even if Kubernetes is still rolling out (or failing to roll out) the new version
- Keep CI and CD in separate workflow files — ci.yml triggers on pull_request, deploy.yml triggers on push to main. Mixing them makes debugging harder and can cause CI failures to block deployments
- Add a failure notification step (Slack, email, PagerDuty) to every production deploy job using `if: failure()` — silent failures in production deployments are unacceptable in a payment company
- Cache dependencies (node_modules, pip packages, Docker layers) from day one — CI pipelines that don't cache re-download gigabytes on every run, wasting minutes and GitHub Actions minutes quota
- Set up branch protection rules the same day you set up the workflow — a CI workflow with no branch protection is just a suggestion, not a gate. Make the test job a required status check so merging broken code is physically impossible

##### 🏋️ Hands-On Exercises — Extend the Pipeline

1. **Add a security scan job:** After the test job, add a job that runs `npm audit --audit-level=high`. It should block the build if any high-severity vulnerabilities are found in dependencies. Use `needs: test` and let it run in parallel with the build job using `needs: [test]` for both.
2. **Add Slack notifications:** Add a final step to the deploy-prod job that only runs on failure (`if: failure()`) and calls a Slack webhook URL stored as a secret. The message should include the commit SHA, the repository name (`github.repository`), and a link to the failed run (`github.server_url/${{ github.repository }}/actions/runs/${{ github.run_id }}`).
3. **Add matrix testing:** Modify the test job to run against both Node 18 and Node 20 using `strategy.matrix`. Observe in the GitHub Actions UI how two test jobs now appear in parallel, each clearly labelled with its Node version.
4. **Add a rollback workflow:** Create a separate workflow file triggered by `workflow_dispatch` with an input field for a git SHA. The workflow should run `kubectl set image` with the provided SHA, effectively allowing a one-click rollback from the GitHub UI to any previous version.
5. **Add Docker image scanning:** After building the Docker image (but before deploying), add a step using the `aquasecurity/trivy-action` to scan the image for known CVEs. Configure it to fail the workflow if any CRITICAL severity vulnerabilities are found.

### GitHub Actions Pipeline Complete 🎉

You have built NovaPay's complete CI/CD pipeline — automated testing on every PR, Docker image builds with layer caching, staging and production deployments with environment protection, secrets management, matrix testing, and conditional failure alerts. Every code push is now tested, built, and deployed automatically. No manual steps. No forgotten commands. Complete audit trail.

> **Tanvi**
> 
> "Priya, before you built this, deploying NovaPay took 45 minutes and four people. This week we shipped 23 times. Average deploy time: 4 minutes and 12 seconds. Zero human involvement after the PR is merged. Every deploy is identical. Every failure sends an alert. Every image is traceable to a commit. That is what CI/CD means — and you built it from scratch."

> **Roshan**
> 
> "The thing that surprises most freshers is this: the pipeline isn't just automation. It's documentation. Reading your workflow YAML, I can see exactly how NovaPay's software moves from a developer's laptop to a customer's phone. Every step is explicit. Every dependency is visible. A new engineer joining the team understands the entire delivery process in 10 minutes by reading one file."

> **Aditya**
> 
> "And as a developer — I just push code. I don't think about Docker. I don't think about kubectl. I don't think about which namespace or which cluster. I write a feature, open a PR, the tests go green, Tanvi approves the prod deploy, and two minutes later it's live. That confidence — knowing my code will be tested and deployed exactly the same way every time — is what GitHub Actions gives us."

> **Next: Advanced CI/CD — Helm Deployments, ArgoCD GitOps & Multi-Environment Pipelines**

> - Helm deployments in GitHub Actions — replace `kubectl set image` with `helm upgrade` for full chart-based deployments with values per environment
> - Argo CD GitOps integration — the pipeline updates a values.yaml in a separate GitOps repo; Argo CD detects the change and syncs the cluster automatically
> - Reusable workflows — extract common jobs (test, build, scan) into reusable workflow files that multiple repositories can call, keeping pipelines DRY
> - OIDC-based authentication — replace long-lived secrets (KUBECONFIG, cloud credentials) with short-lived tokens using OpenID Connect — no stored secrets at all
> - Advanced caching strategies — monorepo-aware caching that only rebuilds the services whose source files actually changed
> - GitHub Actions self-hosted runners — run workflows on your own VMs for faster builds, private network access, and larger machines
