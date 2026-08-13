# YAML Scripting Mastery

> **👋 Hey Fresher — Read This Before You Start!**

> YAML stands for **YAML Ain't Markup Language**. It's a way to write structured data that both humans and machines can read easily. You've already used YAML if you've written a `docker-compose.yml` file, a GitHub Actions workflow, or a Kubernetes manifest — they're all YAML. The problem is: YAML looks simple but has tricky rules. Indentation matters. Quotes matter. Types matter. This document explains every concept with **short 4–8 line code blocks** followed by a plain-English explanation. No 60-line files to get lost in. One concept at a time.

> **Company in this project:** CloudNest — a SaaS startup in Bengaluru building a cloud monitoring platform. They use YAML for their Kubernetes configs, CI/CD pipelines, Ansible playbooks, and application config files. You just joined as a Junior DevOps Engineer. Your lead is **Meera**. Let's learn YAML the way you'll actually use it at work.

#### What You Will Learn in This Course

You'll go from "I don't understand YAML" to writing production-grade config files confidently. Every section connects to a real use-case at CloudNest.

Syntax Basics, Data Types, Lists & Maps, Multi-line Strings, Anchors & Aliases, Multi-Document, CI/CD YAML, Kubernetes Configs, Validation & Linting

> **📄 Phase 1 — YAML Fundamentals**

> Indentation rules, comments, data types (string, number, boolean, null), and the difference between scalars, lists, and maps.

> Multi-line strings, anchors, aliases, merge keys, and multi-document YAML files — the features most freshers never learn.

> Write GitHub Actions and GitLab CI YAML. Understand triggers, jobs, steps, environment variables, and reusable workflows.

> Apply YAML skills to write Kubernetes manifests and Ansible playbooks — the two biggest uses of YAML in real DevOps jobs.

**Scene 1 — CloudNest Office, Bengaluru | First Day**

> **Meera** _Senior DevOps Engineer — CloudNest_
> 
> Welcome to CloudNest. Quick question — have you used YAML before?

> **Arjun (You)** _Junior DevOps Engineer — Day 1 at CloudNest_
> 
> A little. I've seen docker-compose files and some Kubernetes YAML. But honestly, I mostly copied them from examples without really understanding why the spacing has to be exactly right.

> **Meera** _Senior DevOps Engineer_
> 
> That's the right honest answer. YAML looks simple but has real rules. The biggest one: indentation is not decoration — it defines the structure. Two spaces versus four spaces means something completely different. Tab characters will break everything. Get the mental model right first, and everything else — Kubernetes manifests, GitHub Actions, Ansible playbooks — is just YAML with different key names. They're all the same grammar.

> **Vikram** _Platform Architect — CloudNest_
> 
> I'll add: the reason companies use YAML everywhere is because it's human-readable configuration. JSON is for machines. XML is too verbose. YAML is what humans use to describe structured data in a way that's both readable and parseable. Once you internalize the three concepts — scalars, sequences, and mappings — everything clicks.

### 1. Phase 1 — YAML Fundamentals

YAML has one core rule: **structure is defined by indentation, not by brackets or braces**. Get the indentation right and the rest is just learning which keys go where.

> **The 3 Building Blocks of All YAML**

> Every YAML file — no matter how complex — is made of just three things: **Scalars** (a single value like a string, number, or boolean), **Sequences** (a list of items, each starting with a dash `-`), and **Mappings** (key-value pairs, like `name: Arjun`). That's it. Everything you see in a 500-line Kubernetes manifest is just these three things nested inside each other.

#### 1.1 Your Very First YAML File

```
# CloudNest developer profile
name:  Arjun Sharma
role:  DevOps Engineer
level: Junior
city:  Bengaluru
```

**📖 Key-Value Pairs — The Simplest YAML**

Every line is `key: value`. The colon **must be followed by a space** — `name:Arjun` without the space will cause a parse error.  
  
Lines that start with `#` are comments — they're ignored by the parser but help humans understand the file.  
  
This whole block is called a **mapping** (also called a map or dictionary). Think of it as a Python dictionary or a JSON object.

#### 1.2 YAML Comments

```
# This is a full-line comment
app: cloudnest-monitor # inline comment
version: 2.1.0
# YAML has no multi-line comment syntax
# Each line needs its own #
```

**📖 Comments in YAML**

Use `#` for comments — either on their own line or at the end of a value line.  
  
YAML does **not** have a block comment syntax like `/* ... */`. To comment out multiple lines, add `#` to each line individually.  
  
Use comments to explain *why* a value is set, not *what* it is — the key name already tells you that.

#### 1.3 Data Types — YAML Figures Out the Type Automatically

YAML automatically detects whether a value is a string, number, boolean, or null — you don't declare types like in a programming language. But this auto-detection can surprise you.

```
# Numbers (no quotes needed)
port:     8080
replicas: 3
pi:       3.14159
# Booleans (no quotes)
enabled:  true
debug:    false
```

**📖 Numbers and Booleans**

Integers and floats are written without quotes. The parser treats them as numbers.  
  
Booleans are `true` or `false` (lowercase). Some older YAML parsers also accept `yes`/`no` and `on`/`off` — but avoid these as they cause confusion. Stick to `true`/`false`.  
  
If you write `port: "8080"` (with quotes), the value is the *string* "8080", not the number 8080. Some tools care about this difference.

```
# Strings (quotes optional — until they're not)
name:    cloudnest
message: "Hello, World!"
path:    "/usr/local/bin"
# These look like booleans — must quote them!
answer: "yes"
flag:   "true"
```

**📖 When to Quote Strings**

Simple strings don't need quotes. But you **must** use quotes when:  
  
— The value looks like a boolean: `"yes"`, `"no"`, `"true"`, `"false"`  
— The value looks like a number: `"8080"` (when you want the string, not the integer)  
— The value contains special characters: `:`, `#`, `{`, `[`  
— The value starts with a special character  
  
Use double quotes `"..."` when your string contains escape sequences like `\n`. Use single quotes `'...'` when you want the string taken literally.

```
# Null values
database_password: ~
optional_field:    null
also_null:

# The three above are all null/None
```

**📖 Null Values**

YAML has three ways to express null: `~`, `null`, or simply leaving the value empty after the colon.  
  
All three produce the same result — a null/None value when parsed in Python, a null in JavaScript, nil in Go.  
  
Most teams pick one style and stick to it for consistency. `null` is the most readable.

> **💡 The Most Common Fresher Mistake — The "Norway Problem"**

> In older YAML 1.1 parsers, `NO` was treated as boolean `false`. This meant that a configuration file with `country: NO` for Norway would silently become `country: false`. Always quote values that look like booleans: `country: "NO"`. Modern YAML 1.2 parsers only treat `true` and `false` as booleans, but be careful with tools that haven't been updated.

#### 1.4 Lists (Sequences)

```
# Block style list (most common)
services:
  - api-server
  - web-frontend
  - metrics-collector
# Flow style (same thing, compact)
services: [api-server, web-frontend, metrics-collector]
```

**📖 Two Ways to Write Lists**

**Block style** — each item on its own line, starting with a dash `-` and a space. The dashes must all be at the same indentation level. This is what you'll use 95% of the time.  
  
**Flow style** — compact, comma-separated in square brackets. Use this for short lists of simple values. Avoid for long lists or lists of objects — it becomes unreadable fast.

```
# List of objects (very common in K8s YAML)
team:
  - name: Meera
role: Senior DevOps
  - name: Arjun
role: Junior DevOps
```

**📖 List of Objects — The Most Important Pattern**

This is the pattern you'll see constantly in Kubernetes: a list where each item is a mapping (key-value pairs).  
  
The dash `-` marks the start of each object. The keys inside that object are indented **two more spaces** from the dash.  
  
In Kubernetes: `containers:` is a list of objects (each container is an object with `name`, `image`, `ports`, etc.). Same pattern. Different keys.

#### 1.5 Nested Maps — Maps Inside Maps

```
database:
  host:     db.cloudnest.in
port:     5432
credentials:
    username: admin
password: secret123
```

**📖 Nested Maps**

A map value can itself be a map — just indent it further. Each nesting level adds two spaces of indentation.  
  
Here: `database` is a map with three keys: `host`, `port`, and `credentials`. The value of `credentials` is itself a map with `username` and `password`.  
  
In code: `config['database']['credentials']['username']` gives you `'admin'`.

```
YAML Structure — How Indentation Creates Hierarchy
====================================================

database:             ← key at level 0
  host: db.cloudnest  ← key at level 1 (child of database)
  port: 5432          ← key at level 1 (child of database)
  credentials:        ← key at level 1, value is another map
    username: admin   ← key at level 2 (child of credentials)
    password: secret  ← key at level 2 (child of credentials)

Think of it like a folder structure:
database/
  host         = db.cloudnest
  port         = 5432
  credentials/
    username   = admin
    password   = secret
```

#### 1.6 Indentation Rules — The Most Important Rules in YAML

##### ⚠️ The 3 Indentation Rules That Break Everything

**Rule 1 — Never use tabs.** YAML parsers reject tab characters. Always use spaces. Configure your editor to insert spaces when you press Tab (VS Code: "Editor: Insert Spaces" → true, "Editor: Tab Size" → 2).

**Rule 2 — Be consistent.** Use 2 spaces everywhere. Don't mix 2-space and 4-space indentation in the same file.

**Rule 3 — Child items must indent further than parent.** You don't need to use exactly 2 spaces — 2, 3, or 4 all work — but every level must be more indented than its parent, and siblings must be at the same level.

```
# ❌ WRONG — tabs cause parse error
server:
	host: localhost # ← tab here
# ✅ CORRECT — 2 spaces
server:
  host: localhost # ← 2 spaces
```

**📖 Tabs vs Spaces**

This is the single most common cause of YAML errors for beginners.  
  
Tab characters look identical to spaces on screen but YAML parsers treat them completely differently — and will throw a cryptic error like `found character '\t' that cannot start any token`.  
  
Set VS Code to **auto-convert tabs to spaces** for `.yml` and `.yaml` files. You'll never see this error again.

**Quiz: 🧠 Quick Check — Which of these YAML snippets will cause a parse error?**

- `name: "cloudnest"` — quoted string
- `port: 8080` — unquoted integer
- `enabled: YES` — uppercase boolean (YAML 1.1 treats as true)
- `host:localhost` — no space after colon

> **Answer/explanation:** ✅ Answer: **D**. `host:localhost` — a colon must be followed by a space to be treated as a key-value separator. Without the space, the whole thing is treated as a single string, not a key-value pair. Option C won't error but will give unexpected behaviour — `YES` becomes boolean `true` in YAML 1.1 parsers.

### 2. Phase 2 — Advanced YAML Features

These are the features that separate someone who copy-pastes YAML from someone who actually *writes* YAML. They're used constantly in CI/CD and Kubernetes configs.

**Scene 2 — CloudNest Config Review | The 80% Duplication Problem**

> **Arjun (You)** _Junior DevOps Engineer_
> 
> Meera, I'm writing the deployment YAML for our three environments — dev, staging, and prod. They're nearly identical. 80% of the content is exactly the same — same image, same ports, same env vars. I'm copying and pasting and it feels wrong. If we change a port number we have to update it in three files.

> **Meera** _Senior DevOps Engineer_
> 
> That's exactly the problem YAML anchors solve. An anchor lets you define a block once and reuse it with a single alias reference anywhere else in the file. One change updates all three. It's YAML's version of a variable or a template. Let me show you.

#### 2.1 Multi-line Strings — Literal and Folded

Sometimes a string value is long — a script, a certificate, a SQL query. YAML has two ways to write multi-line strings.

```
# Literal block (|) — preserves newlines
startup_script: |
  #!/bin/bash
  echo "Starting CloudNest API"
  cd /app
  node server.js
```

**📖 Literal Block Scalar — |**

The pipe character `|` tells YAML: "everything indented under me is the string value, and keep all the newlines exactly as-is."  
  
When parsed, `startup_script` will be the full multi-line string:  
`#!/bin/bash\necho "Starting..."...`  
  
Use `|` for: shell scripts, SQL queries, certificates, any text where newlines are meaningful.

```
# Folded block (>) — collapses newlines to spaces
description: >
  This is the CloudNest monitoring
  platform. It collects metrics from
  all microservices automatically.
```

**📖 Folded Block Scalar — >**

The greater-than character `>` says: "keep the text but fold newlines into spaces — make it one long paragraph."  
  
When parsed, the value becomes: `"This is the CloudNest monitoring platform. It collects metrics from all microservices automatically."`  
  
Use `>` for: long description text, messages, anything where you want to wrap long lines for readability but the actual value should be one line.

> **💡 The Chomp Indicator — Controlling Trailing Newlines**

> By default, YAML keeps one trailing newline at the end of a block scalar. You can control this with a "chomp" indicator after the `|` or `>`: use `|-` or `>-` to **strip** all trailing newlines, or `|+` or `>+` to **keep** all trailing newlines. In CI/CD files you'll almost always see `|-` for inline scripts — it strips the final newline which can cause issues with some pipeline runners.

#### 2.2 Anchors and Aliases — DRY YAML (Don't Repeat Yourself)

Anchors let you define a block of YAML once and reference it multiple times. This is the single most powerful feature for reducing duplication in config files.

```
# Define an anchor with &anchor-name
defaults: &app-defaults
image:    cloudnest/api:v2.1
port:     8080
replicas: 2
# Use the anchor with *alias-name
staging:
  <<: *app-defaults
replicas: 1 # override just this
```

**📖 Anchors (&) and Aliases (*)**

`&app-defaults` — defines an anchor. The name after `&` is just a label you choose.  
  
`<<: *app-defaults` — the merge key. It says "copy all key-value pairs from the anchored block into this map." Think of it as "paste everything from defaults here."  
  
After the merge you can override individual keys. Here, `staging` gets the same `image` and `port` as defaults, but overrides `replicas` to 1.

```
# Full example: 3 envs, one source of truth
base: &base
image: cloudnest/api:v2.1
port:  8080
dev:
  <<: *base
debug: true
prod:
  <<: *base
replicas: 5
```

**📖 Why This Matters at Work**

Without anchors: if the image tag changes from `v2.1` to `v2.2`, you edit three places and risk forgetting one.  
  
With anchors: change the image in `base` and all three environments update automatically.  
  
Both `dev` and `prod` inherit `image` and `port` from base, then add their own unique keys. This pattern is used heavily in GitLab CI's `extends` feature and Docker Compose override files.

##### ⚠️ Anchors Only Work Within the Same File

Anchors cannot span across multiple YAML files. An anchor defined in `base.yml` cannot be referenced in `dev.yml`. If you need cross-file reuse, tools like Helm (for Kubernetes), Kustomize, or language-level templating (Jsonnet, CUE) are designed for that. Pure YAML anchors = same file only.

#### 2.3 Multi-Document YAML Files

A single YAML file can contain multiple independent documents, separated by `---`. This is used in Kubernetes constantly — a single file can contain a Deployment AND a Service.

```yaml
# Document 1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
---
# Document 2
apiVersion: v1
kind: Service
metadata:
  name: api-service
```

**📖 The --- Document Separator**

`---` on its own line signals the start of a new document. Everything after it is a completely independent YAML document.  
  
In Kubernetes: `kubectl apply -f app.yaml` reads all documents in the file and applies each one. You can bundle all your resources (Deployment, Service, ConfigMap, Ingress) in one file, separated by `---`.  
  
`...` marks the explicit end of a document — optional but sometimes seen at the end of a file.

#### 2.4 Flow Style vs Block Style — When to Use Each

**Block vs Flow — Rule of Thumb**

- **Block Style (Preferred)** — Uses indentation and newlines. Readable, easy to diff in Git, easy to add comments. Use for all production YAML files.  
  
`labels:  app: api  env: prod`

- **Flow Style (Compact)** — Uses brackets and commas, like JSON. Saves vertical space. Use only for short, simple values where readability isn't hurt.  
  
`labels: {app: api, env: prod}`

- **Mixed (Sometimes Seen)** — Flow style values inside a block style structure. Acceptable for short inline lists.  
  
`ports: [80, 443, 8080]`

- **The Golden Rule** — If the YAML is going into Git and being reviewed by humans, use block style. If it's machine-generated, flow style is fine. Human-written config = block style.

**Quiz: 🧠 Quick Check — What does `<<: *base` do in YAML?**

- Creates a reference to an external file called "base"
- Merges all key-value pairs from the anchor named "base" into the current map
- Imports another YAML document into this one
- Declares a variable called "base"

> **Answer/explanation:** ✅ Answer: **B**. `<<` is the YAML merge key. Combined with `*base` (an alias that points to whatever was anchored with `&base`), it copies all key-value pairs from that anchored block into the current mapping. This is YAML's way of doing inheritance or "extend this config with defaults."

#### 2.5 Special String Cases — Things That Look Wrong But Are Right

```
# Strings that MUST be quoted
version:   "1.0" # float without quotes
zip_code:  "560001" # leading zero — integer
colon_str: "host:port" # colon in value
hash_str:  "#tag" # hash starts a comment
```

**📖 Quoting Edge Cases**

`"1.0"` without quotes becomes the float `1.0`. If you want the string `"1.0"`, quote it.  
  
`"560001"` without quotes becomes the integer 560001 (losing the leading zero if there was one). Always quote numeric strings like phone numbers or zip codes.  
  
Values containing `:` or starting with `#` must be quoted — those characters have special meaning in YAML.

### 3. Phase 3 — YAML for CI/CD Pipelines

The most common use of YAML in DevOps jobs is writing CI/CD pipeline definitions. GitHub Actions uses `.github/workflows/*.yml`. GitLab CI uses `.gitlab-ci.yml`. CircleCI, Jenkins, and Azure DevOps all have their own YAML formats — but they all use the same YAML syntax you've just learned.

**Scene 3 — CloudNest Sprint Planning | "We Need Automated Pipelines"**

> **Vikram** _Platform Architect — CloudNest_
> 
> Right now, developers push code and manually run tests and deployment scripts. That's not sustainable. Arjun, your next task is to write our GitHub Actions workflow. Every push to main should automatically run tests, build the Docker image, and deploy to staging. All of that gets defined in a YAML file.

> **Arjun (You)** _Junior DevOps Engineer_
> 
> So the pipeline definition itself lives in the Git repository as a YAML file?

> **Meera** _Senior DevOps Engineer_
> 
> Exactly. That's the principle behind "pipeline as code." Your workflow file describes every step. Anyone cloning the repo gets the same pipeline. You can review pipeline changes in a pull request. You can roll back a pipeline by reverting a commit. It's the same Git workflow you use for code, applied to infrastructure and automation.

#### 3.1 GitHub Actions — Anatomy of a Workflow File

```
# .github/workflows/ci.yml
name: CI Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

**📖 Workflow Triggers — The on Key**

**name** — display name shown in the GitHub Actions UI. Choose something descriptive.  
  
**on** — defines what events trigger this workflow. `push` fires when someone pushes commits. `pull_request` fires when a PR is opened or updated.  
  
**branches:** — limits the trigger to specific branches. `[main]` means only pushes to `main` trigger this workflow. Remove this to trigger on all branches.

```
jobs:
  test:
    runs-on: ubuntu-latest
steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
run: npm install
      - name: Run tests
run: npm test
```

**📖 Jobs and Steps**

**jobs** — a workflow has one or more jobs. Each job runs on its own runner (virtual machine).  
  
**runs-on: ubuntu-latest** — run this job on an Ubuntu virtual machine provided by GitHub.  
  
**steps** — ordered list of actions inside the job. Steps run sequentially, top to bottom.  
  
**uses:** — runs a pre-built GitHub Action (reusable code). `actions/checkout@v4` clones your repo code into the runner.  
  
**run:** — runs a shell command directly.

```
# Environment variables in workflow
jobs:
  build:
    runs-on: ubuntu-latest
env:
      NODE_ENV: production
API_URL:  https://api.cloudnest.in
steps:
      - run: echo "Building for $NODE_ENV"
```

**📖 Environment Variables**

**env:** — sets environment variables available to all steps in the job. Use this for non-secret configuration values.  
  
Reference them in shell commands with `$VARIABLE_NAME`.  
  
For secrets (passwords, tokens), use GitHub Secrets instead: `${{ secrets.MY_SECRET }}`. Never put sensitive values directly in the YAML file — they'd be visible to anyone who can read the repo.

```
# Jobs that depend on each other
jobs:
  test:
    runs-on: ubuntu-latest
steps:
      - run: npm test
deploy:
    needs: test
runs-on: ubuntu-latest
steps:
      - run: ./deploy.sh
```

**📖 Job Dependencies — needs**

By default, all jobs in a workflow run **in parallel**. If you need jobs to run in order, use `needs`.  
  
`needs: test` means "don't start the `deploy` job until the `test` job has finished successfully."  
  
If `test` fails, `deploy` is skipped automatically. This is how you prevent broken code from being deployed — tests must pass before deployment runs.

#### 3.2 Multi-line Run Commands

```bash
# Multiple commands in one step
steps:
  - name: Build and tag image
run: |-
docker build -t cloudnest/api:${{ github.sha }} .
docker tag cloudnest/api:${{ github.sha }} cloudnest/api:latest
docker push cloudnest/api:latest
```

**📖 Multi-line run Commands**

Use the literal block scalar `|-` to write multi-line shell scripts in a single step. The `-` strips the trailing newline (important for some runners).  
  
Each line of the script runs in sequence. If any command fails (exits with non-zero), the step fails and the job stops.  
  
`${{ github.sha }}` is a GitHub Actions expression — it injects the commit SHA. Expressions use `${{ }}` syntax, not `$`.

#### 3.3 GitLab CI — Similar Concept, Different Structure

```
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy
run-tests:
  stage: test
image: node:20
script:
    - npm install
    - npm test
```

**📖 GitLab CI vs GitHub Actions**

GitLab CI uses `stages` to define the pipeline order. Jobs in the same stage run in parallel. Jobs in later stages wait for earlier stages to complete.  
  
`script:` in GitLab = `run:` in GitHub Actions — it's the list of shell commands to run.  
  
`image:` — run this job inside a specific Docker image. Here, the test job runs inside `node:20` so Node.js and npm are available without installing them.

```
# GitLab CI — reuse config with anchors
.node-defaults: &node-defaults
image: node:20
before_script:
    - npm install
test:
  <<: *node-defaults
stage: test
script: [npm test]
```

**📖 GitLab CI Hidden Jobs + Anchors**

In GitLab CI, job names starting with a dot (`.`) are **hidden jobs** — they don't run on their own. They're used purely as templates to be merged into real jobs with anchors.  
  
Here, `.node-defaults` defines the shared Node.js setup. The `test` job merges in those defaults and adds its own `script`.  
  
This is the GitLab equivalent of GitHub Actions' `uses:` reusable workflows — same concept, YAML anchors as the mechanism.

**Quiz: 🧠 Quick Check — In a GitHub Actions workflow, what does `needs: test` under a job do?**

- The job will run before the "test" job
- The job will only run after the "test" job has finished successfully
- The job will share the same virtual machine as "test"
- The job will run in parallel with the "test" job

> **Answer/explanation:** ✅ Answer: **B**. `needs:` creates a dependency chain. The job with `needs: test` waits for the "test" job to complete successfully before starting. If "test" fails, the dependent job is automatically skipped. This is how you build sequential pipelines: test → build → deploy, where each stage only runs if the previous one passed.

### 4. Phase 4 — YAML for Kubernetes Manifests

Kubernetes is the biggest real-world use of YAML in DevOps. Every Kubernetes object — Pod, Deployment, Service, ConfigMap, Secret — is defined in YAML. Now that you know YAML deeply, the Kubernetes-specific key names are all that's left to learn.

**Scene 4 — CloudNest Infrastructure Review | "It's All YAML"**

> **Meera** _Senior DevOps Engineer_
> 
> You've seen our Kubernetes folder in the repo. Every file in there is a YAML file. A Deployment YAML, a Service YAML, a ConfigMap YAML, an Ingress YAML. But here's the thing — now that you actually understand YAML, reading those files should feel completely different to you.

> **Arjun (You)** _Junior DevOps Engineer_
> 
> It does. I used to see all the indentation and get overwhelmed. Now I see it as nested maps and lists. The Deployment spec is just a map. The containers list is a list of maps. Each container map has keys like image and ports.

> **Vikram** _Platform Architect — CloudNest_
> 
> Exactly. That mental model change is 80% of the battle. The kubectl error messages also become readable — "mapping values are not allowed here" means you have an indentation problem. "did not find expected key" means you're missing a required field. You can now debug Kubernetes YAML at the YAML level, not just guess at line numbers.

#### 4.1 The 4 Fields Every Kubernetes YAML Must Have

```yaml
# These 4 keys appear in EVERY K8s object
apiVersion: v1
kind:       Pod
metadata:
  name:      my-pod
namespace: default
spec:
  # what you want goes here
```

**📖 The Universal Kubernetes Structure**

**apiVersion** — which Kubernetes API group and version. `v1` for core objects (Pod, Service, ConfigMap). `apps/v1` for Deployments, StatefulSets. `networking.k8s.io/v1` for Ingress.  
  
**kind** — what type of object: Pod, Deployment, Service, ConfigMap, Secret, Ingress...  
  
**metadata** — data about the object: name (required), namespace, labels, annotations.  
  
**spec** — the desired state. Every kind has a different spec structure.

#### 4.2 Labels and Selectors — YAML's Connection System

```
# Labels on a Pod
metadata:
  name: api-pod
labels:
    app: cloudnest-api
env: production
version: v2.1
```

**📖 Labels — Key-Value Tags on Any Object**

Labels are arbitrary key-value pairs you attach to any Kubernetes object. The key names are completely up to you.  
  
Labels are not just cosmetic — they're functional. Services use labels to find which Pods to route traffic to. Deployments use labels to track which Pods they manage. HPA uses labels to find Deployments.  
  
Convention: always add at minimum `app: <app-name>` and `env: <environment>` to every object.

```
# Service using selector to find pods
spec:
  selector:
    app: cloudnest-api
ports:
  - port:       80
targetPort: 8080
```

**📖 Selectors — How Services Find Pods**

The Service's `selector` is a map of labels to match. Any Pod that has **all** the listed labels receives traffic from this Service.  
  
Here: the Service finds all Pods with `app: cloudnest-api` and load-balances traffic across them. New Pods with that label are automatically added to the pool. Pods without it are ignored.  
  
This is why the labels in your Pod template must exactly match the selector in your Service — if they don't match, the Service has zero pods and traffic goes nowhere.

#### 4.3 ConfigMap — Storing Config in YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL:   info
API_URL:     https://api.cloudnest.in
TIMEOUT_SEC: "30"
```

**📖 ConfigMap — Non-Secret Configuration**

A ConfigMap stores key-value pairs that your application needs as environment variables or config files.  
  
**data:** — the actual config values. Keys are environment variable names (by convention uppercase). Values are always strings — note that `"30"` is quoted to be explicit.  
  
ConfigMap values are visible to anyone who can access the cluster. **Never put passwords, API keys, or tokens in a ConfigMap** — use a Secret instead.

```
# Using ConfigMap in a Deployment
spec:
  containers:
  - name: api
image: cloudnest/api:v2.1
envFrom:
    - configMapRef:
        name: app-config
```

**📖 Injecting ConfigMap as Env Variables**

`envFrom:` — inject all keys from a ConfigMap as environment variables into the container. No need to list them one by one.  
  
The container will have `LOG_LEVEL=info`, `API_URL=https://...`, and `TIMEOUT_SEC=30` available as environment variables.  
  
Alternatively, use `env:` with `valueFrom.configMapKeyRef` to inject individual keys — useful when you only need specific values from a large ConfigMap.

#### 4.4 Secrets — Sensitive Data in YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=
```

**📖 Secrets — Base64 Encoded Values**

Secrets look like ConfigMaps but values must be **base64 encoded**. `cGFzc3dvcmQxMjM=` is `password123` in base64.  
  
Encode a value: `echo -n "password123" | base64`  
Decode a value: `echo "cGFzc3dvcmQxMjM=" | base64 -d`  
  
Important: base64 is encoding, not encryption. Secrets are **not secure by default** — they're just obfuscated. Use Kubernetes RBAC to restrict who can read Secrets, and consider external secret managers (HashiCorp Vault, AWS Secrets Manager) for production.

#### 4.5 Resource Requests and Limits in YAML

```
containers:
- name: api
image: cloudnest/api:v2.1
resources:
    requests:
      cpu:    "250m"
memory: "256Mi"
limits:
      cpu:    "500m"
memory: "512Mi"
```

**📖 CPU and Memory Units in YAML**

**CPU units:** `250m` = 250 millicores = 0.25 of one CPU core. `"1"` = 1 full core. `"2"` = 2 cores.  
  
**Memory units:** `Mi` = mebibytes (1 Mi = ~1.05 MB). `Gi` = gibibytes. Use `Mi`/`Gi` (base-2) not `MB`/`GB` (base-10) for precision.  
  
Note both are quoted strings in YAML — `"250m"` not `250m`. Without quotes, `250m` is a number in scientific notation (0.25) in some YAML parsers.

#### 4.6 Liveness and Readiness Probes — YAML Health Checks

```
livenessProbe:
  httpGet:
    path: /health
port: 8080
initialDelaySeconds: 15
periodSeconds:      10
```

**📖 Liveness Probe**

A liveness probe asks: *"Is this container still alive?"* Kubernetes sends an HTTP GET to `/health` on port 8080 every 10 seconds.  
  
If the app returns non-200 (or the request times out), Kubernetes **restarts** the container automatically.  
  
`initialDelaySeconds: 15` — wait 15 seconds after the container starts before sending the first probe. This gives the app time to start up before Kubernetes checks it.

```
readinessProbe:
  httpGet:
    path: /ready
port: 8080
initialDelaySeconds: 5
periodSeconds:      5
```

**📖 Readiness Probe**

A readiness probe asks: *"Is this container ready to receive traffic?"* If it returns non-200, the container is temporarily removed from the Service's load balancer — no traffic is sent to it.  
  
Unlike liveness, a failed readiness probe does **not** restart the container. It just stops traffic until the probe passes again.  
  
Use readiness for: database connections warming up, caches loading, any initialization that takes time after the container starts.

### 5. Phase 5 — YAML for Ansible Playbooks

Ansible is a configuration management tool that uses YAML to define automation tasks — installing packages, configuring servers, deploying applications. An Ansible playbook is a YAML file that describes what to do on which machines.

**Scene 5 — CloudNest Infrastructure Team | "Configure 20 Servers at Once"**

> **Vikram** _Platform Architect — CloudNest_
> 
> We have 20 new VM instances coming online for the monitoring backend. Each one needs Node.js 20 installed, the CloudNest agent deployed, and a systemd service configured. Doing this manually on 20 machines is 2 hours of work and one mistake away from a config drift. Arjun, write an Ansible playbook. You run one command, all 20 machines are configured identically in 3 minutes.

#### 5.1 Ansible Playbook — Basic Structure

```
# setup-servers.yml
- name: Configure CloudNest agent servers
hosts: monitoring_servers
become: true
tasks:
    - name: Update apt cache
apt:
        update_cache: true
```

**📖 Playbook Anatomy**

A playbook is a list of **plays**. Each play is a mapping with these important keys:  
  
**hosts:** — which machines to run on. Must match a group name in your Ansible inventory file.  
  
**become: true** — run tasks as root (sudo). Required for installing packages, modifying system files.  
  
**tasks:** — the list of things to do, in order. Each task has a `name` (description) and a module call (`apt`, `copy`, `template`, `service`, etc.).

```bash
# Install Node.js and a package
tasks:
    - name: Install Node.js 20
apt:
        name: nodejs
state: present

    - name: Install npm packages
npm:
        name: pm2
global: true
```

**📖 Ansible Modules — The Verbs of Automation**

Each task calls an Ansible **module** — a pre-built action. `apt:` installs packages on Ubuntu. `npm:` manages Node packages. `service:` starts/stops services. `copy:` copies files. `template:` renders Jinja2 templates.  
  
**state: present** means "make sure this is installed." If it's already installed, Ansible does nothing (idempotent — safe to run multiple times).  
  
**state: absent** would uninstall it. **state: latest** would upgrade it.

```
# Variables in a playbook
- name: Deploy CloudNest Agent
hosts: monitoring_servers
vars:
    agent_version: 2.1.0
install_dir:  /opt/cloudnest
tasks:
    - name: Create install directory
file:
        path:  "{{ install_dir }}"
state: directory
```

**📖 Variables in Ansible YAML**

**vars:** — define variables for this play. Reference them anywhere in the play with `{{ variable_name }}`.  
  
The double-curly-brace `{{ }}` syntax is Jinja2 templating — it's interpolated at runtime. When Ansible runs the task, `{{ install_dir }}` becomes `/opt/cloudnest`.  
  
Note the quotes around `"{{ install_dir }}"` — when a value starts with `{{`, YAML requires it to be quoted to avoid being parsed as a map.

#### 5.2 Ansible Conditionals and Loops in YAML

```
# Run only on Ubuntu servers
    - name: Install apt packages
apt:
        name:  curl
state: present
when: ansible_os_family == "Debian"
```

**📖 Conditional Execution — when**

`when:` adds a condition to a task. The task only runs if the condition is true.  
  
`ansible_os_family` is a **fact** — Ansible automatically discovers information about each server (OS, CPU, RAM, IP) before running tasks. You can use these facts in `when` conditions.  
  
Other common facts: `ansible_distribution` (Ubuntu, CentOS), `ansible_architecture` (x86_64), `ansible_memtotal_mb`.

```
# Loop over a list of packages
    - name: Install required packages
apt:
        name:  "{{ item }}"
state: present
loop:
        - curl
        - git
        - unzip
```

**📖 Loops — Installing Multiple Things at Once**

`loop:` iterates over a list. On each iteration, `{{ item }}` is replaced by the current list value.  
  
This runs the `apt` module three times — once for curl, once for git, once for unzip.  
  
This is cleaner than writing three separate tasks. In newer Ansible versions, the `apt` module also accepts a list in `name:` directly — but `loop` works universally across any module.

### 6. Phase 6 — YAML Validation, Linting, and Common Errors

YAML errors are often cryptic. Understanding what the parser is actually saying — and knowing how to validate YAML before applying it — saves hours of debugging.

#### 6.1 The Most Common YAML Errors and What They Mean

**YAML Error Messages Decoded**

- **mapping values are not allowed here** — Indentation error. A key-value pair appears where a list item or scalar is expected. Check the indentation of the line before the error — something is misaligned.

- **found character '\t' that cannot start any token** — You used a tab character. Replace all tabs with spaces. In VS Code: View → Command Palette → "Convert Indentation to Spaces".

- **could not find expected ':'** — A key is missing its colon, or a colon has no space after it. Check the line number in the error and the lines immediately above it.

- **did not find expected key** — A required field is missing or incorrectly spelled. In Kubernetes: you misspelled a field name or the indentation put a key at the wrong level.

#### 6.2 Validating YAML — Tools Every DevOps Engineer Uses

1. yamllint — catch syntax errors before applying
A command-line YAML linter that checks syntax and style rules. Install once, use everywhere.

```bash
# Install yamllint
pip install yamllint

# Lint a single file
yamllint deployment.yaml

# Lint all YAML files in a directory
yamllint k8s/
```

> **yamllint** catches syntax errors (tabs, missing colons, wrong indentation) before you try to apply the YAML. The output shows the line and column number of each error. Configure it with a `.yamllint` file in your repo to set your team's rules (max line length, indentation width, etc.). Add it to your CI pipeline so YAML files are validated on every pull request — broken YAML never reaches main.

2. kubectl dry-run — validate Kubernetes YAML without applying
Test if Kubernetes accepts your manifest without actually creating any resources.

```bash
# Dry run — validate the YAML against the cluster API
kubectl apply -f deployment.yaml --dry-run=client

# Server-side validation (more thorough)
kubectl apply -f deployment.yaml --dry-run=server

# Check what the cluster would actually see
kubectl apply -f deployment.yaml --dry-run=server -o yaml
```

> **--dry-run=client** validates the YAML structure without talking to the cluster. Fast, works offline, but only catches structural errors.  
**--dry-run=server** sends the request to the cluster API server which validates it fully — including custom resource definitions, admission controllers, and schema validation. It doesn't create anything, but it tells you exactly what the cluster would accept or reject. Always dry-run before applying to production.

3. Python — parse and verify YAML programmatically
A quick way to check if a YAML file can be parsed without errors.

```python
# Quick parse check with Python (no install needed)
python3 -c "import yaml; yaml.safe_load(open('config.yaml'))"

# Parse and pretty-print the structure
python3 -c "import yaml, pprint; pprint.pprint(yaml.safe_load(open('config.yaml')))"
```

> Python's `yaml` module (PyYAML) is available on almost every Linux system used for DevOps. This one-liner tries to parse the file — if it succeeds with no output, the YAML is valid. The `pprint` version shows you the parsed Python data structure, which is excellent for understanding what your YAML actually evaluates to. For example, you can check whether a value is being parsed as a string or a number.

#### 6.3 VS Code Setup for YAML

> **Configure VS Code for Zero-Mistake YAML Writing**

> Install the **"YAML" extension by Red Hat** (the most popular VS Code YAML extension). It provides: syntax highlighting, inline error detection (red underlines on bad YAML), auto-completion for Kubernetes resource fields when you set the schema, and format-on-save that fixes indentation automatically.

> Add this to your VS Code `settings.json` to enable Kubernetes schema validation for all `.yaml` files:

```
# VS Code settings.json
{
  "yaml.schemas": {
    "kubernetes": "*.yaml"
  },
  "editor.insertSpaces": true,
  "editor.tabSize": 2,
  "[yaml]": {
    "editor.formatOnSave": true
  }
}
```

> With `"kubernetes": "*.yaml"`, VS Code downloads the Kubernetes JSON schema and validates your YAML against it in real time. If you type `kind: Deploymnet` (typo), VS Code immediately underlines it in red and suggests the correct spelling. Auto-complete works too — type `spec:` under a Deployment and VS Code suggests all valid child keys. This setup alone eliminates 80% of YAML errors before you even run a command.

### 7. Phase 7 — YAML Patterns You'll See at Every Company

These are real patterns used in production YAML files. Understanding them helps you read unfamiliar codebases quickly.

#### 7.1 Environment-Specific Overrides Pattern

```
# base.yaml — shared across all envs
app:
  name:     cloudnest-api
port:     8080
replicas: 2
log:
    level: info
```

**📖 Base + Override Pattern**

Most projects have a base YAML with common settings, then environment-specific overrides that change just the values that differ.  
  
This is how tools like **Kustomize** work for Kubernetes — you have a `base/` folder and `overlays/dev/`, `overlays/prod/` that patch only the differences.  
  
The same principle applies to Helm values: `values.yaml` has defaults, `values-prod.yaml` overrides specific keys.

```
# prod-override.yaml — only the differences
app:
  replicas: 5 # override base value
log:
    level: warn # less verbose in prod
# name and port come from base.yaml
```

**📖 Only Override What's Different**

The override file only contains the keys that change. Everything else is inherited from base.  
  
In production, you typically want more replicas and less verbose logging. Rather than duplicating the entire config, you express only the delta.  
  
Tools that support this pattern: Kustomize (`patchesStrategicMerge`), Helm (`-f prod-values.yaml`), Ansible (`group_vars/` directory), Docker Compose (`-f docker-compose.yml -f docker-compose.prod.yml`).

#### 7.2 The Template Pattern — Parameterized YAML

```
# Helm values.yaml
image:
  repository: cloudnest/api
tag:        latest
service:
  port: 80
replicaCount: 3
```

**📖 Helm Values — YAML as Input to Templates**

Helm is Kubernetes' package manager. A `values.yaml` file provides variables to Go template files. The template files use `{{ .Values.image.tag }}` syntax to interpolate values.  
  
This makes it possible to deploy the same application to different environments by just changing the values file — not the template files.  
  
Understanding this pattern means you understand how every major Helm chart in the world works: the chart is the template, `values.yaml` is the YAML configuration that drives it.

#### 7.3 Health Check Pattern in Docker Compose YAML

```
# docker-compose.yml healthcheck
services:
  api:
    image: cloudnest/api:v2.1
healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
timeout:  10s
retries:  3
```

**📖 Health Check in Docker Compose**

The `test:` key takes a list — the first element is the execution mode (`CMD` = run directly, `CMD-SHELL` = run in shell).  
  
`interval` — how often to check. `timeout` — how long to wait for a response before marking it failed. `retries` — how many consecutive failures before the container is marked `unhealthy`.  
  
Other services can use `depends_on: condition: service_healthy` to wait for this container to be healthy before starting — prevents database connection errors on startup.

### 8. Phase 8 — Complete CloudNest YAML Example

Let's bring everything together. This is a real, complete multi-document Kubernetes YAML file — every line uses a concept you've learned. Read it top to bottom and see how it all fits.

**Scene 8 — CloudNest Production Deploy | Putting It All Together**

> **Meera** _Senior DevOps Engineer_
> 
> Arjun, write me the full Kubernetes manifest for the CloudNest API. ConfigMap for config, Deployment with probes and resource limits, and a ClusterIP Service. Put it all in one file separated by ---. That's our production YAML pattern.

> **Arjun (You)** _Junior DevOps Engineer_
> 
> Got it. Three documents, one file.

#### Document 1 — ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudnest-config
namespace: production
data:
  LOG_LEVEL: warn
API_URL:   https://api.cloudnest.in
MAX_CONN:  "100"
```

**📖 ConfigMap Recap**

Non-sensitive configuration stored in the cluster. All values in `data:` are strings — note `"100"` is quoted to be explicit about it being a string, not an integer.  
  
Using `namespace: production` scopes this ConfigMap to the production namespace. If you forget the namespace, it lands in `default` and your Deployment in `production` won't find it.

#### Document 2 — Deployment

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudnest-api
namespace: production
spec:
  replicas: 3
selector:
    matchLabels:
      app: cloudnest-api
```

**📖 Deployment Header**

The `---` separator starts a new YAML document — this Deployment is a separate object from the ConfigMap above, even though they're in the same file.  
  
`spec.replicas: 3` — run 3 identical Pods at all times.  
  
`selector.matchLabels` — the Deployment finds and manages Pods with `app: cloudnest-api`. This must match the `labels` in the Pod template below, or the Deployment will never manage any Pods.

```
 template:
    metadata:
      labels:
        app: cloudnest-api
spec:
      containers:
      - name: api
image: cloudnest/api:v2.1.3
ports:
        - containerPort: 8080
```

**📖 Pod Template**

The `template:` section is the blueprint for each Pod the Deployment creates. It has its own `metadata` (for labels) and `spec` (for containers).  
  
The labels here must match `selector.matchLabels` above — they're how the Deployment identifies its own Pods.  
  
Use a specific image tag (`v2.1.3`) not `latest` — in production, `latest` makes it impossible to know which version is running or to roll back reliably.

```
 envFrom:
        - configMapRef:
            name: cloudnest-config
resources:
          requests:
            cpu: "250m"
memory: "256Mi"
limits:
            cpu: "500m"
memory: "512Mi"
```

**📖 Config Injection + Resource Limits**

`envFrom.configMapRef` — inject all keys from the `cloudnest-config` ConfigMap as environment variables. The container gets `LOG_LEVEL`, `API_URL`, `MAX_CONN` automatically.  
  
Resource requests + limits — required in production. Without requests, the scheduler may place the Pod on a node with insufficient resources. Without limits, a memory leak can consume all node memory and evict every other Pod.

```
 livenessProbe:
          httpGet:
            path: /health
port: 8080
initialDelaySeconds: 20
periodSeconds:      10
readinessProbe:
          httpGet:
            path: /ready
port: 8080
initialDelaySeconds: 5
```

**📖 Both Probes Together**

Liveness: if `/health` returns non-200, Kubernetes restarts the container. Catches hangs and deadlocks.  
  
Readiness: if `/ready` returns non-200, the Pod is removed from the Service's endpoints — no traffic until it's ready again. Catches startup delays and temporary unavailability.  
  
Both probes together mean: zero traffic to starting containers, and automatic restart of broken containers. This is the minimum viable health setup for any production container.

#### Document 3 — Service

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: cloudnest-api-svc
namespace: production
spec:
  type: ClusterIP
selector:
    app: cloudnest-api
ports:
  - port:       80
targetPort: 8080
```

**📖 Service Completes the Trio**

The Service is the stable network endpoint for the Pods. Other services inside the cluster call `http://cloudnest-api-svc` (port 80) — the Service routes to port 8080 on whichever Pods match `app: cloudnest-api`.  
  
`type: ClusterIP` — internal only. Not reachable from outside the cluster. An Ingress or LoadBalancer would be added in front to expose it externally.  
  
The three documents together form a complete, production-ready application deployment — Config, Compute, and Network.

```
$ kubectl apply -f cloudnest-app.yaml
configmap/cloudnest-config created
deployment.apps/cloudnest-api created
service/cloudnest-api-svc created

$ kubectl get all -n production
NAME                                 READY   STATUS    RESTARTS   AGE
pod/cloudnest-api-7d9f6b8c4-xkp2s   1/1     Running   0          45s
pod/cloudnest-api-7d9f6b8c4-mnq7r   1/1     Running   0          45s
pod/cloudnest-api-7d9f6b8c4-w4tj9   1/1     Running   0          45s

NAME                        TYPE        CLUSTER-IP      PORT(S)   AGE
service/cloudnest-api-svc   ClusterIP   10.100.45.123   80/TCP    45s

NAME                            READY   UP-TO-DATE   AVAILABLE
deployment.apps/cloudnest-api   3/3     3            3
```

##### ❓ Common Fresher Questions — Answered

**Q: Q: Should I use .yml or .yaml extension?**

A: Both are identical — YAML parsers treat them the same. The YAML specification recommends `.yaml`. In practice, GitHub Actions uses `.yml` by default, Kubernetes manifests are often `.yaml`. Pick one and be consistent within a project. When in doubt, `.yaml`.

**Q: Q: Why do some YAML values need quotes and others don't?**

A: YAML auto-detects types. Unquoted values that look like numbers, booleans (true/false/yes/no), or null (~) are parsed as those types. If you want the string "true" rather than the boolean true, quote it. If your string contains characters with special YAML meaning (: # { [ |), quote it. When unsure, quoting always makes a value a string.

**Q: Q: What's the difference between a YAML null and an empty string?**

A: `value:` (empty after colon) = null. `value: ""` (empty quotes) = empty string. These are different and some tools treat them differently. If you explicitly want an empty string (to override a default with nothing), use `""`. If you want null/undefined, use `null` or `~`.

**Q: Q: Can I have a list as the top-level structure of a YAML file?**

A: Yes. Ansible playbooks are a perfect example — the entire file is a list of play objects, each starting with `- name:`. You don't have to start with a map. The top level can be a list, a map, or even a scalar. In practice, almost all YAML config files start with a mapping.

**Q: Q: How does YAML handle duplicate keys?**

A: YAML technically prohibits duplicate keys in a mapping, but most parsers accept them and use the last value. This is dangerous — the first definition is silently overwritten. yamllint will flag this as an error. Always treat duplicate keys as a bug to fix, not a feature to rely on.

> **Key Takeaways — What You Must Always Remember**

> - Indentation defines structure in YAML — two spaces per level, never tabs. Every YAML error in your career will either be a tab character, wrong indentation level, or a missing space after a colon.
> - YAML auto-detects types. Quote values that look like booleans (true, false, yes, no), numbers you want to keep as strings, and anything containing colons, hashes, or brackets.
> - The three building blocks are scalars (single values), sequences (lists with -), and mappings (key: value pairs). Every YAML file — Kubernetes, GitHub Actions, Ansible — is just these three things nested.
> - Use anchors (&name) and merge keys (<<: *name) to avoid duplicating shared configuration. Define once, reference everywhere in the same file.
> - Separate multiple Kubernetes objects in one file with ---. One kubectl apply -f deploys all of them. This is the standard pattern for grouping related resources (Deployment + Service + ConfigMap).
> - Always use yamllint and kubectl --dry-run before applying YAML to a production cluster. Fix errors in your editor, not in the middle of an incident.

##### CloudNest YAML Standards — Rules the Team Follows

- All YAML files are stored in Git and reviewed via pull request. Never edit a YAML file directly in the cluster with kubectl edit — the change won't be tracked and will be overwritten on the next deploy.
- Use specific image tags in Kubernetes manifests (image:v2.1.3), never image:latest in production. Latest makes rollbacks impossible and deployments unpredictable.
- Every Kubernetes container must have resource requests and limits. Every production container must have both liveness and readiness probes. These are non-negotiable in code review.
- Use anchors to DRY up repeated blocks within a file. If you copy-paste a block more than once, consider an anchor. If the duplication spans files, use Kustomize or Helm instead.
- Add yamllint to the CI pipeline. Broken YAML should fail the pipeline immediately, before any tool even tries to parse it. Fail fast, fail clearly.
- For multi-environment config, use a base file + override files rather than duplicating the entire config. The diff between environments should be minimal and obvious.

##### 🏋️ Hands-On Exercises — Apply What You Learned

1. **Write a docker-compose.yml with anchors:** Create a compose file for a three-service application (api, worker, scheduler). All three use the same base environment variables and healthcheck. Use a YAML anchor to define the common config once and merge it into all three services with `<<: *base`.
2. **Write a complete GitHub Actions workflow:** Create a workflow that triggers on push to main, runs `npm install && npm test` in a test job, then if tests pass, builds and pushes a Docker image in a build job (use `needs: test`). Use a GitHub Secret for the Docker Hub password.
3. **Write a GitLab CI pipeline with a hidden job template:** Define a `.deploy-defaults` hidden job anchor with common deploy steps. Create three real jobs (deploy-dev, deploy-staging, deploy-prod) that each merge the defaults and add their own environment-specific commands.
4. **Fix the broken YAML:** This snippet has three bugs — find and fix them:  
`server:  host: localhost  port:8080  debug: TRUE  tags: - api - web`
5. **Write an Ansible playbook for server setup:** Create a playbook that configures a group of servers called `app_servers`. It should install nginx, copy a custom nginx config file from your local machine to `/etc/nginx/nginx.conf`, start and enable the nginx service, and run only on Debian-family systems using `when:`.

**Quiz: 🧠 Final Check — Which statement about YAML is FALSE?**

- A YAML file can contain multiple documents separated by ---
- Indentation must use spaces, never tabs
- YAML anchors can reference values across different files
- An unquoted value of "true" is parsed as a boolean, not a string

> **Answer/explanation:** ✅ Answer: **C**. YAML anchors only work *within* the same file. An anchor defined in one file cannot be referenced in another. Cross-file reuse requires tooling built on top of YAML: Helm templates, Kustomize overlays, or language-level config systems like Jsonnet or CUE.

### Quick Reference — YAML Commands Every DevOps Engineer Uses Daily

What You Want to Do

Command

Validate YAML syntax

yamllint filename.yaml

Parse YAML with Python

python3 -c "import yaml; yaml.safe_load(open('f.yaml'))"

Dry-run Kubernetes apply

kubectl apply -f file.yaml --dry-run=client

See what cluster would apply

kubectl apply -f file.yaml --dry-run=server -o yaml

Convert tabs to spaces (VS Code)

Command Palette → "Convert Indentation to Spaces"

Base64 encode a secret value

echo -n "mypassword" | base64

Base64 decode a value

echo "bXlwYXNzd29yZA==" | base64 -d

Pretty-print parsed YAML structure

python3 -c "import yaml,pprint; pprint.pprint(yaml.safe_load(open('f.yaml')))"

Check YAML with yq (jq for YAML)

yq eval '.spec.replicas' deployment.yaml

Merge two YAML files with yq

yq eval-all 'select(fi==0) * select(fi==1)' base.yaml override.yaml

### YAML Mastery Complete 🎉

You now understand YAML at the level that professional DevOps engineers use every day — scalars, sequences, mappings, multi-line strings, anchors, aliases, multi-document files, CI/CD pipelines, Kubernetes manifests, and Ansible playbooks. YAML is no longer something you copy from Stack Overflow and hope it works. You can write it, debug it, and explain it.

> **Meera**
> 
> "Arjun, the playbook you wrote configured all 20 servers in 4 minutes. Identical config on every single one. And the Kubernetes manifests you reviewed last week — you caught three indentation bugs and one missing readiness probe before they went to production. That's what YAML fluency buys the team."

> **Vikram**
> 
> "The mental model shift: YAML is just three data structures — scalar, sequence, mapping. When you see a 300-line Kubernetes manifest, you're not reading a wall of text anymore. You're reading nested maps and lists. Every complex file is the same three things, deeply nested. Once that clicks, you can read any YAML file in any tool on Day 1."

> **Arjun**
> 
> "The anchor pattern was the biggest unlock for me. I used to copy-paste environment variable blocks across three files. Now I define them once and merge. When the API URL changes, I update one line. That single YAML feature probably saves me 30 minutes of error-prone editing every sprint."

> **Next: Advanced Configuration Management — Helm, Kustomize & Jsonnet**

> - Helm — package Kubernetes YAML into reusable, versioned charts with Go templating for true multi-environment support
> - Kustomize — layer YAML patches over a base without modifying the original files; built into kubectl
> - Jsonnet — a data templating language that compiles to JSON/YAML; used at Google-scale for generating thousands of Kubernetes manifests
> - CUE — type-safe configuration language that validates data structure and generates YAML; increasingly used in large mono-repos
> - Secrets management — integrating YAML configs with HashiCorp Vault, AWS Secrets Manager, and sealed-secrets for safe secret handling in Git
> - Schema validation — writing and enforcing JSON Schema and OPA policies on YAML files in CI/CD pipelines

### 9. Phase 9 — YAML Schema Validation in Depth

Schema validation means checking that your YAML not only parses correctly but also has the right structure — required fields are present, values are the right type, no unexpected keys are used. This is how you catch logic errors before deployment.

#### 9.1 What is a Schema?

```
# A schema is a contract: "valid YAML looks like this"
# Example: JSON Schema in YAML format
type: object
required: [name, port, image]
properties:
  name:
    type: string
port:
    type: integer
minimum: 1024
maximum: 65535
```

**📖 JSON Schema — The Standard for Validation**

A JSON Schema (also expressible in YAML) describes the structure your data must have. It defines: which fields are required, what type each field must be, constraints like minimum/maximum for numbers, allowed string patterns, and more.  
  
Tools like VS Code's YAML extension, the `jsonschema` Python library, and `kubeconform` use schemas to validate your YAML files. You write the schema once, then every file is automatically checked against it in your editor and CI pipeline.

1. kubeconform — validate Kubernetes YAML against the official schema
A fast Kubernetes manifest validator. Catches field name typos, missing required fields, and wrong value types.

```
# Install kubeconform
brew install kubeconform  # Mac
go install github.com/yannh/kubeconform/cmd/kubeconform@latest  # Linux
# Validate one file
kubeconform deployment.yaml

# Validate all YAML in a directory
kubeconform k8s/
```

> **kubeconform** downloads the official Kubernetes API schemas and checks your YAML manifests against them. If you spell `containerPort` as `containerport` (lowercase p), kubeconform catches it. If you forget the `selector.matchLabels` field in a Deployment, it catches that too. Add it to your CI pipeline alongside yamllint for full coverage: yamllint catches YAML syntax errors, kubeconform catches Kubernetes semantic errors.

#### 9.2 Custom Schema Validation with Python

```python
# validate_config.py
import yaml, jsonschema
schema = {
  "required": ["app", "port"],
  "properties": {
    "port": {"type": "integer"}
  }
}
```

**📖 Validate Your Own Config Format**

For application config files (not Kubernetes manifests), you define your own schema. The `jsonschema` Python library validates a parsed YAML document against a schema you write.  
  
This is extremely useful for: validating `config.yaml` before your app starts, checking pipeline parameter files in CI, or validating Ansible variable files before a playbook run.  
  
Install with `pip install jsonschema pyyaml`.

```python
# Full validation script
import yaml, jsonschema

schema = {
    "required": ["app", "port", "image"],
    "properties": {
        "port": {"type": "integer", "minimum": 1024},
        "image": {"type": "string"},
        "replicas": {"type": "integer", "default": 1}
    }
}

config = yaml.safe_load(open("config.yaml"))
jsonschema.validate(config, schema)
print("Config is valid!")
```

> This script: loads your YAML file with `yaml.safe_load()`, then passes the parsed Python object to `jsonschema.validate()`. If the config is missing required fields or has wrong types, it raises a `ValidationError` with a clear message: "port is required" or "port must be an integer". If validation passes, you get "Config is valid!" Add this check as a pre-flight step in your application startup to catch config errors before the app crashes mid-operation.

### 10. Phase 10 — YAML in Docker Compose

Docker Compose is often the first real-world YAML tool freshers encounter. Understanding its structure deeply helps you transition to Kubernetes YAML — the concepts map directly.

**Scene 10 — CloudNest Dev Environment | Local Multi-Service Setup**

> **Meera** _Senior DevOps Engineer_
> 
> Before we deploy to Kubernetes, every developer should be able to run the full CloudNest stack locally. That means the API, the database, the Redis cache, and the metrics collector — all running together with one command. We use Docker Compose for local development. The Compose YAML is your local environment definition.

> **Arjun (You)** _Junior DevOps Engineer_
> 
> So Compose is like a simpler, local version of Kubernetes YAML?

> **Vikram** _Platform Architect_
> 
> Exactly. Services in Compose = Deployments + Services in Kubernetes. Volumes in Compose = PersistentVolumeClaims in Kubernetes. Networks in Compose = Services with selectors in Kubernetes. Learn the Compose model first — it's simpler — and the Kubernetes version will make more sense because you already understand what each piece is trying to achieve.

#### 10.1 Docker Compose File Structure

```
# docker-compose.yml — top level keys
version: "3.9"
services:
  # your containers go here
volumes:
  # named volumes go here
networks:
  # custom networks go here
```

**📖 Compose File Top-Level Keys**

**version:** — the Compose file format version. `"3.9"` is the latest stable version. Note it's a string (quoted) not a float.  
  
**services:** — the containers you want to run. Each service is a named mapping under this key.  
  
**volumes:** — declares named volumes that persist data across container restarts. Used for databases.  
  
**networks:** — custom networks for service-to-service communication. By default, all services in a compose file can reach each other.

```
services:
  api:
    build: .
ports:
      - "8080:8080"
environment:
      - NODE_ENV=development
      - DB_HOST=database
depends_on:
      - database
```

**📖 Defining a Service**

**build: .** — build the image from the Dockerfile in the current directory. Use `image:` instead to pull from a registry.  
  
**ports:** — map host port to container port. `"8080:8080"` = host:container. Visit `localhost:8080` to reach the container.  
  
**environment:** — list of environment variables. Note the `KEY=VALUE` string format — different from Kubernetes where env vars are maps.  
  
**depends_on:** — start this service only after `database` starts. Doesn't wait for healthy, just for started.

```
services:
  database:
    image: postgres:15
environment:
      POSTGRES_DB:       cloudnest
POSTGRES_USER:     admin
POSTGRES_PASSWORD: dev_password
volumes:
      - db_data:/var/lib/postgresql/data
```

**📖 Database Service with Persistent Volume**

Environment variables can also be written as a map (key: value) instead of a list (`KEY=VALUE`). Both formats are valid Compose YAML — the map format is cleaner and more readable for multiple variables.  
  
`volumes: db_data:/var/lib/postgresql/data` — mount the named volume `db_data` at the container path where Postgres stores data. The data survives `docker compose down` and is restored on the next `up`. Without this, your database resets every restart.

#### 10.2 Docker Compose Overrides for Different Environments

```
# docker-compose.override.yml (auto-loaded for dev)
services:
  api:
    volumes:
      - .:/app # mount source code for hot reload
environment:
      - DEBUG=true
command: npm run dev # dev mode with file watching
```

**📖 Override Files — Local Dev vs Production**

Compose automatically merges `docker-compose.yml` with `docker-compose.override.yml` when you run `docker compose up`. Use the base file for production-like config and the override for development conveniences.  
  
**The source code volume** `.:/app` mounts your local directory into the container — code changes immediately reflect without rebuilding. Never do this in production.  
  
For production: `docker compose -f docker-compose.yml -f docker-compose.prod.yml up`

### 11. Phase 11 — Real Debugging Scenarios

The best way to learn YAML debugging is to see broken YAML and understand exactly what went wrong. These are scenarios from real CloudNest incidents.

#### 11.1 Debugging Scenario: The Missing Space After Colon

##### ❌ Error: yaml.scanner.ScannerError — could not find expected ':' at line 3

This error means the parser expected a colon in a specific position but didn't find one — usually because of a formatting problem on or near the indicated line.

```
# ❌ BROKEN — no space after colon
server:
  host:localhost
port: 8080
# ✅ FIXED — space after colon
server:
  host: localhost
port: 8080
```

**📖 Root Cause + Fix**

Without the space after `:`, YAML doesn't see a key-value separator. It sees the whole thing as a scalar string `"host:localhost"` — not a key-value pair.  
  
When the parser then finds the next proper key-value pair on the following line, the structure doesn't make sense and it errors.  
  
Fix: always `key: value` (space after colon). VS Code with the YAML extension will highlight this in red before you even save.

#### 11.2 Debugging Scenario: Wrong Indentation Level

##### ❌ Kubernetes Error: error validating data — ValidationError(Deployment.spec.template.spec): unknown field "image"

The field exists in the Kubernetes schema but is being placed at the wrong level — it's a child of the wrong parent.

```
# ❌ BROKEN — image at wrong indent level
spec:
  containers:
  - name: api
image: myapp:v1 # ← wrong level!
# ✅ FIXED — image is a child of the container
spec:
  containers:
  - name: api
image: myapp:v1 # ← 4 spaces
```

**📖 Root Cause + Fix**

In the broken version, `image:` is indented at the same level as the `-` (list marker), making it a sibling of the container object rather than a child of it.  
  
In a list of objects, the fields of each object must be indented **more** than the dash. The dash occupies 2 spaces, so container fields need 4 spaces.  
  
This is the single most common Kubernetes YAML error — always count spaces carefully when working with lists of objects.

#### 11.3 Debugging Scenario: Boolean Type Confusion

##### ❌ App Error: Expected string for environment variable DEBUG, got boolean

The application received a boolean `true` when it expected the string `"true"`. YAML auto-detected the type incorrectly for this use case.

```
# ❌ BROKEN — unquoted, becomes boolean true
env:
  DEBUG: true
VERBOSE: false
# ✅ FIXED — quoted, stays as string
env:
  DEBUG:   "true"
VERBOSE: "false"
```

**📖 Root Cause + Fix**

Most applications read environment variables as strings — that's how environment variables work at the OS level. But YAML parses unquoted `true`/`false` as booleans.  
  
When a YAML parser reads `DEBUG: true` and serializes it for the app, some tools pass the boolean `true`; others pass the string `"true"`. Behavior varies between tools.  
  
The safe rule: always quote boolean-looking values in config files intended to be read as strings.

#### 11.4 Debugging Scenario: Tab Character in CI Pipeline

##### ❌ GitHub Actions Error: Invalid workflow file — found character '\t' that cannot start any token

A tab character exists somewhere in the YAML file. The line number in the error points to the first line with a tab, but there may be more.

```
# Find tab characters in a file
grep -Pn "\t" .github/workflows/ci.yml

# Replace all tabs with 2 spaces
sed -i 's/\t/  /g' .github/workflows/ci.yml

# Or use expand command
expand -t 2 ci.yml > ci_fixed.yml
```

> `grep -Pn "\t"` shows every line that contains a tab character (with line numbers). This is faster than manually searching in an editor.  
`sed -i 's/\t/  /g'` replaces every tab with two spaces in-place. Always verify the file looks correct after running this — sometimes tabs were intentional in inline scripts inside the YAML and replacing them changes the script behaviour.  
Prevention: configure your editor to show tab characters as a visible symbol and to auto-convert them on paste.

### 12. Phase 12 — YAML for Application Config Files

Beyond DevOps tools, YAML is widely used as the format for application configuration files. Languages like Python, Ruby, Java (Spring Boot), Go, and Node.js all have YAML config libraries. Understanding how applications read YAML makes you a better DevOps engineer.

#### 12.1 Python Application Reading YAML Config

```
# config.yaml for a Python service
database:
  host: localhost
port: 5432
name: cloudnest_db
server:
  host: 0.0.0.0
port: 8080
debug: false
```

**📖 Structured Config for Applications**

Grouping related config under a top-level key (database, server, logging) is the standard pattern for application YAML config files.  
  
It keeps the file organized, prevents name collisions between sections, and makes it easy to pass just one section to a function: `db_config = config['database']`.  
  
In Python with PyYAML: `config = yaml.safe_load(open('config.yaml'))`. Then access any value with `config['database']['host']`.

```python
# Python: load and use YAML config
import yaml

with open("config.yaml") as f:
    config = yaml.safe_load(f)

db_host = config["database"]["host"]    # "localhost"
db_port = config["database"]["port"]    # 5432  (integer, not string)
debug   = config["server"]["debug"]     # False (boolean, not string)
```

> `yaml.safe_load()` is the safe way to parse YAML in Python — it only creates standard Python objects (dict, list, str, int, float, bool, None) and won't execute arbitrary Python code the way `yaml.load()` could. Always use `safe_load()`.  
  
Notice the types: `port` is the Python integer `5432`, not the string `"5432"`. `debug` is Python's `False`, not the string `"false"`. YAML's auto-typing flows through to your application — this is why quoting matters.

#### 12.2 Spring Boot Application YAML (application.yml)

```
# Spring Boot application.yml
spring:
  datasource:
    url:      jdbc:postgresql://localhost:5432/cloudnest
username: ${DB_USER}
password: ${DB_PASS}
jpa:
    hibernate:
      ddl-auto: validate
```

**📖 Spring Boot YAML + Environment Variable Injection**

Spring Boot natively supports `application.yml`. The deeply nested structure mirrors Spring's configuration property hierarchy.  
  
`${DB_USER}` — Spring reads the value from the `DB_USER` environment variable at runtime. This lets you store config in YAML while keeping secrets in environment variables (injected by Kubernetes Secrets).  
  
This pattern — YAML for structure, env vars for secrets — is used in production by virtually every Java Spring Boot application running on Kubernetes.

#### 12.3 Node.js Application Config with YAML

```
# config/default.yaml (node-config library)
server:
  port:    3000
timeout: 30000
redis:
  host: localhost
port: 6379
ttl:  3600
```

**📖 Node.js — node-config Library**

The `node-config` npm package loads YAML (or JSON) config files from a `config/` directory. `default.yaml` is loaded for all environments. `production.yaml` overrides just the production-specific values.  
  
Usage in code: `const config = require('config'); const port = config.get('server.port');`  
  
The dotted path `'server.port'` navigates the nested YAML structure. This is a common pattern in Node.js microservices — every service has a `config/` folder with YAML files per environment.

### 13. Phase 13 — YAML Gotchas That Trip Up Everyone

These are subtle YAML behaviours that cause bugs even for experienced engineers. Learn them now so they don't bite you in production.

#### 13.1 Gotcha: The Octal Number Problem

```
# YAML 1.1 octal interpretation
file_mode: 0755 # ← parsed as 493 (decimal)!
port:      0022 # ← parsed as 18 (decimal)!
# ✅ Safe: quote it
file_mode: "0755" # ← stays as string "0755"
file_mode_int: 0o755 # ← YAML 1.2 explicit octal
```

**📖 Octal Numbers in YAML**

In YAML 1.1 (used by many tools), a number starting with `0` is interpreted as **octal**. So `0755` = decimal 493. This catches people writing Unix file permissions in YAML.  
  
YAML 1.2 removed this behaviour — only `0o755` (explicit octal prefix) is octal. But many tools still use 1.1 parsers.  
  
Safe rule: always quote strings that start with `0` unless you explicitly want octal interpretation.

#### 13.2 Gotcha: Implicit Type Coercion in Specific Tools

```
# These values are dangerous if unquoted
country:  NO # → boolean false in YAML 1.1
status:   on # → boolean true in YAML 1.1
version:  1.0 # → float 1.0 not string "1.0"
# ✅ Always quote these
country:  "NO"
status:   "on"
version:  "1.0"
```

**📖 The Full List of Dangerous Unquoted Values**

In YAML 1.1 parsers, these unquoted values become booleans: `y`, `Y`, `yes`, `Yes`, `YES`, `true`, `True`, `TRUE`, `on`, `On`, `ON` → `true`.  
  
And: `n`, `N`, `no`, `No`, `NO`, `false`, `False`, `FALSE`, `off`, `Off`, `OFF` → `false`.  
  
The safe default: if a value could be interpreted as something other than a plain string, quote it.

#### 13.3 Gotcha: Multi-line String Trailing Newline Behaviour

```
# |  → keeps final newline (clip)
script_a: |
  echo hello

# |- → strips final newline (strip)
script_b: |-
  echo hello

# |+ → keeps ALL trailing newlines
script_c: |+
  echo hello
```

**📖 Chomp Indicator — Trailing Newline Control**

`|` (clip, default) — strips trailing blank lines but keeps one final `\n`.  
`|-` (strip) — removes all trailing newlines entirely. The string ends with the last content character.  
`|+` (keep) — preserves all trailing newlines exactly as written.  
  
In practice: use `|-` for inline shell scripts in GitHub Actions and GitLab CI — some runners behave differently with trailing newlines in script blocks. Use `|` for file content that needs a final newline.

#### 13.4 Gotcha: Flow Sequences and Trailing Commas

```
# ✅ Valid flow sequence
ports: [80, 443, 8080]

# ✅ Also valid — trailing comma allowed in YAML
ports: [80, 443, 8080,]

# ❌ Invalid — double comma
ports: [80,, 443]
```

**📖 Flow Sequence Quirks**

Unlike JSON, YAML flow sequences **allow trailing commas** — `[80, 443,]` is valid YAML (it would be an error in JSON).  
  
However, a double comma `,,` creates a null element in the sequence — `[80,, 443]` gives you `[80, null, 443]`.  
  
These edge cases rarely matter in practice, but if you're writing YAML generation scripts or templating YAML programmatically, watch out for accidentally generating double commas.

### 14. Phase 14 — Working with YAML at Scale

When a project has hundreds of YAML files, you need tools beyond a text editor. This section covers the tools professional DevOps engineers use to manage YAML at scale.

#### 14.1 yq — The jq for YAML

`yq` is a command-line YAML processor — think of it as `jq` (for JSON) but for YAML. It lets you query, filter, and transform YAML files from the terminal.

```
# Install yq
brew install yq   # Mac
snap install yq   # Linux
# Read a specific value
yq eval '.spec.replicas' deployment.yaml

# Update a value in-place
yq eval -i '.spec.replicas = 5' deployment.yaml

# Extract a key from all files in a directory
yq eval '.metadata.name' k8s/*.yaml
```

> `yq eval '.spec.replicas'` uses a dot-notation path to navigate the YAML structure. This is the same syntax as `jq` for JSON.  
`yq eval -i` edits the file in-place — extremely useful in CI/CD pipelines to update image tags: `yq eval -i '.spec.template.spec.containers[0].image = "myapp:v2.0"' deployment.yaml`. This replaces manual `sed` commands for YAML updates and is much safer since it understands YAML structure instead of doing text replacement.

```
# Merge two YAML files
yq eval-all 'select(fi==0) * select(fi==1)' base.yaml override.yaml

# Convert YAML to JSON
yq eval -o=json deployment.yaml

# Convert JSON to YAML
cat data.json | yq eval -P -
```

> `yq eval-all 'select(fi==0) * select(fi==1)'` merges two files — the `*` operator deep-merges mappings. This is the CLI equivalent of YAML anchors but across files.  
`-o=json` outputs as JSON — useful for piping to other tools that only understand JSON.  
The `-P` flag enables "pretty print" mode. Piping JSON into yq with `-P` converts it to YAML — handy when you have JSON config that you need to convert to YAML format.

#### 14.2 Kustomize — YAML Without Templates

```yaml
# kustomize/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

**📖 Kustomize — Built Into kubectl**

Kustomize is a YAML overlay system built into `kubectl`. You define a `kustomization.yaml` that lists your base resources, and then create overlay files for each environment that patch specific values.  
  
No templates, no programming language — just YAML patching YAML. The base files are valid Kubernetes YAML. The patches are also valid YAML. Kustomize merges them intelligently.  
  
Apply with: `kubectl apply -k kustomize/overlays/prod/`

```yaml
# kustomize/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
patchesStrategicMerge:
  - replicas-patch.yaml
images:
  - name: cloudnest/api
newTag: v2.1.3
```

**📖 Prod Overlay — Override Without Modifying Base**

`bases:` — point to the base folder. All resources defined there are included.  
  
`patchesStrategicMerge` — list of patch files. Each patch file is a partial Kubernetes manifest that describes only the changes.  
  
`images:` — a convenience feature to update image tags across all resources in one place. Change the tag here and it's updated in all Deployments that reference that image name. This is the cleanest way to manage image updates in a GitOps workflow.

#### 14.3 Checking YAML Diff Before Applying

```bash
# See what changes kubectl apply would make (without applying)
kubectl diff -f deployment.yaml

# Example output:
# --- a/Deployment/cloudnest-api
# +++ b/Deployment/cloudnest-api
#  spec:
#    replicas: 3
# -  image: cloudnest/api:v2.0
# +  image: cloudnest/api:v2.1
```

> `kubectl diff` compares your local YAML file against what's currently running in the cluster and outputs a standard diff. Green lines (+) are additions, red lines (-) are deletions. This is your safety check before any production apply — see exactly what will change before it changes. Add this step to your CI pipeline as a review step: generate the diff, attach it to the pull request, require a human approval before the actual apply runs.

### Quick Reference — YAML Syntax Cheatsheet

> **Every YAML Pattern in One Reference**

**Quiz: 🧠 Scenario Quiz — Arjun's CI pipeline fails with "found character '\t' that cannot start any token" on line 23. He checks line 23 and it looks fine. What should he do?**

- Delete and rewrite only line 23
- Run `grep -Pn "\t" .github/workflows/ci.yml` to find ALL lines with tabs, then fix all of them
- Change the indentation on line 23 to 4 spaces instead of 2
- The problem must be in a different file — YAML errors always point to the correct line

> **Answer/explanation:** ✅ Answer: **B**. YAML parsers report the *first* tab they encounter, but there may be many more throughout the file. Running `grep -Pn "\t"` finds every occurrence at once. Fix them all in one shot with `sed -i 's/\t/  /g' filename.yaml` or VS Code's "Convert Indentation to Spaces" command. Then re-run your linter to confirm zero tabs remain before pushing.

### 15. Interview Prep — YAML Questions You'll Actually Be Asked

These are real questions that appear in DevOps, SRE, and Platform Engineering interviews. Know these cold.

##### ❓ Real Interview Q&A — YAML Edition

**Q: Q: What is the difference between `|` and `>` in YAML?**

A: Both are block scalar indicators. `|` (literal) preserves newlines exactly as written — use for shell scripts, certificates, or any text where line breaks matter. `>` (folded) folds newlines into spaces — use for long description text that wraps for readability but should be one line when parsed.

**Q: Q: How do you handle sensitive values like passwords in YAML files that go into Git?**

A: Never put plaintext secrets in YAML files committed to Git. Use environment variable references (`${DB_PASSWORD}` in Spring, `{{ secrets.DB_PASSWORD }}` in GitHub Actions). For Kubernetes, use Secrets objects with values injected from an external secret manager (HashiCorp Vault, AWS Secrets Manager) using tools like External Secrets Operator. The YAML file contains the reference to the secret, not the secret itself.

**Q: Q: What's the difference between YAML anchors and Kubernetes ConfigMaps for sharing config?**

A: YAML anchors work at the file level — they deduplicate repeated blocks within a single YAML file at parse time. ConfigMaps work at the cluster level — they store configuration in the cluster that multiple Pods can reference at runtime. Anchors solve copy-paste duplication in your YAML source. ConfigMaps solve the problem of giving the same runtime configuration to multiple running containers.

**Q: Q: Why does `kubectl apply` sometimes fail with "unknown field" even though your YAML looks correct?**

A: Three common causes: (1) Indentation error — the field is at the wrong level in the hierarchy. (2) Typo in the field name — Kubernetes field names are case-sensitive and exactly specified. (3) The field exists in the spec but you're using the wrong `apiVersion` for it — newer fields require newer API versions. Use `kubectl explain` to see the exact valid fields for any resource: `kubectl explain deployment.spec.template.spec.containers`.

**Q: Q: A colleague asks: "Can I just use JSON instead of YAML for Kubernetes manifests?" What do you say?**

A: Yes — kubectl accepts both JSON and YAML. Kubernetes internally stores everything as JSON. YAML is preferred for human-written files because it's less verbose (no quotes around keys, no braces, comments are supported). JSON is sometimes preferred for machine-generated manifests. In practice, 99% of Kubernetes documentation, tutorials, and tooling uses YAML, so using JSON would make it harder to work with community resources.

**Q: Q: What is the purpose of `---` in a YAML file, and when would you use it?**

A: `---` is the document separator — it marks the start of a new, independent YAML document within the same file. Common uses: (1) Multiple Kubernetes resources in one file (Deployment + Service + ConfigMap) — `kubectl apply -f` applies all documents. (2) Ansible playbooks start with `---` by convention. (3) Some YAML parsers use it to distinguish multiple independent configs. It also signals to human readers "new top-level object starts here."

### 16. Phase 16 — YAML in GitOps with Argo CD

GitOps is the practice of using Git as the single source of truth for your infrastructure and applications. Argo CD is the most popular GitOps tool for Kubernetes — it watches a Git repository and automatically applies any YAML changes to the cluster. Understanding the YAML files that drive Argo CD is an essential modern DevOps skill.

**Scene 16 — CloudNest Platform Review | The GitOps Shift**

> **Vikram** _Platform Architect — CloudNest_
> 
> We've been running kubectl apply manually. Every deploy is someone SSH-ing into a jump server and running a command. That's fragile, unaudited, and not reproducible. We're moving to Argo CD. Every YAML change goes through a pull request. Argo CD watches the main branch and syncs the cluster automatically. No more manual deploys.

> **Meera** _Senior DevOps Engineer_
> 
> The key concept: Git becomes the cluster. If it's in Git, it's in the cluster. If it's not in Git, Argo CD will detect the drift and either alert or auto-correct it. Your YAML files are not just documentation — they are the actual running state of production.

#### 16.1 Argo CD Application YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name:      cloudnest-api
namespace: argocd
spec:
  project: default
source:
    repoURL:        https://github.com/cloudnest/k8s-config
targetRevision: HEAD
path:           apps/cloudnest-api
```

**📖 Argo CD Application — The Bridge Between Git and Cluster**

An Argo CD `Application` is itself a YAML object that tells Argo CD: "Watch this Git repo, at this path, and keep the cluster in sync with whatever YAML files are there."  
  
`repoURL` — the Git repository containing your Kubernetes manifests.  
`targetRevision: HEAD` — track the latest commit on the default branch.  
`path` — the subdirectory within the repo that contains the manifests for this application.

```
 destination:
    server:    https://kubernetes.default.svc
namespace: production
syncPolicy:
    automated:
      prune:    true
selfHeal: true
```

**📖 Sync Policy — Automated GitOps**

`destination` — which cluster and namespace to deploy to.  
  
`syncPolicy.automated` — enable automatic sync when Git changes are detected.  
  
`prune: true` — if a YAML file is deleted from Git, Argo CD deletes the corresponding resource from the cluster. Without this, removed resources linger.  
  
`selfHeal: true` — if someone manually changes a resource in the cluster (breaking GitOps), Argo CD detects the drift and reverts it to match Git. Git wins, always.

#### 16.2 YAML Naming Conventions for GitOps Repos

> **Repository Structure for YAML Manifests**

> A well-structured GitOps repository makes YAML files easy to find and review. CloudNest uses this layout, which is a common industry standard:

> Each application has its own folder. All resources for that application live in that folder. Environment-specific differences are handled by Kustomize overlays in the `environments/` directory. The `main` branch = production state. `staging` branch = staging state. Pull requests = change reviews before deploy.

### 17. Phase 17 — Advanced Debugging Tools

These tools go beyond basic linting and help you understand complex YAML structures, trace values through templates, and catch drift between what's in Git and what's running.

#### 17.1 kubectl explain — Learn Kubernetes Fields Without Documentation

```bash
# What fields can a Deployment's spec have?
kubectl explain deployment.spec

# What about the container spec?
kubectl explain deployment.spec.template.spec.containers

# Deep dive on probes
kubectl explain deployment.spec.template.spec.containers.livenessProbe

# See all fields recursively
kubectl explain deployment --recursive
```

> `kubectl explain` is your offline Kubernetes API documentation. It shows every valid field for any resource type, the field's type (string, integer, map, list), whether it's required, and a plain-English description of what it does. This is faster than searching the web and always accurate for the exact Kubernetes version you're connected to. When you're unsure of a field name or what values it accepts — don't guess, don't Google — run `kubectl explain` and get the authoritative answer in 2 seconds.

#### 17.2 Helm Template Debugging — See the Generated YAML

```bash
# Render Helm templates without deploying
helm template my-release ./my-chart --values values-prod.yaml

# Render and validate against the cluster
helm install my-release ./my-chart --dry-run --debug

# See the computed values after merging all values files
helm install my-release ./my-chart --debug 2>&1 | head -50
```

> `helm template` renders all the Go templates in a Helm chart with your values file and outputs the resulting YAML to stdout. This lets you see exactly what Kubernetes YAML will be created before you apply it — invaluable for debugging "why is my image tag wrong" or "where is this environment variable coming from?"  
`--dry-run --debug` goes further — it sends the rendered YAML to the Kubernetes API server for validation (same as `kubectl --dry-run=server`) and shows you the full computed values including defaults.

#### 17.3 Finding Drift — What's Running vs What's in Git

```bash
# Get the YAML of what's currently running in the cluster
kubectl get deployment cloudnest-api -o yaml > current.yaml

# Compare to what's in Git
diff current.yaml git_version.yaml

# Or use kubectl diff directly
kubectl diff -f deployment.yaml
```

> `kubectl get -o yaml` retrieves the full YAML of any running resource. The output includes Kubernetes-added fields (status, resource version, timestamps) but also all the spec fields you set.  
`diff current.yaml git_version.yaml` shows you exactly what changed — useful when you suspect someone edited a resource directly in the cluster (bypassing Git). With Argo CD's `selfHeal: true`, this drift is auto-corrected. Without GitOps, manual drift is a constant operational headache.

### 18. Phase 18 — Complete GitHub Actions CI/CD Workflow

Let's write a complete, production-ready GitHub Actions workflow for CloudNest. This brings together every CI/CD YAML concept from Phase 3 in one coherent file.

#### The Full Pipeline — Test, Build, Push, Deploy

```
# .github/workflows/deploy.yml
name: Build and Deploy
on:
  push:
    branches: [main]
  workflow_dispatch:  # manual trigger
env:
  IMAGE: ghcr.io/cloudnest/api
TAG:   ${{ github.sha }}
```

**📖 Triggers and Global Variables**

`workflow_dispatch:` — adds a "Run workflow" button in the GitHub UI so you can trigger the pipeline manually without pushing code. Essential for emergency deploys or re-runs.  
  
`env:` at the workflow level — variables available to ALL jobs in the workflow. `${{ github.sha }}` is the commit hash — using it as the Docker image tag means every push produces a uniquely tagged image, making rollbacks trivial.

```
jobs:
  test:
    runs-on: ubuntu-latest
steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
with:
          node-version: "20"
      - run: npm ci
      - run: npm test
```

**📖 The Test Job**

`actions/setup-node@v4` installs the specific Node.js version. The `with:` key passes parameters to the action — different from `env:` which sets shell environment variables.  
  
`npm ci` instead of `npm install` — in CI always use `npm ci`. It installs exactly what's in `package-lock.json`, fails if the lockfile is out of date, and is faster. `npm install` can silently update packages, producing non-reproducible builds.

```
 build-push:
    needs: test
runs-on: ubuntu-latest
permissions:
      packages: write
steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
with:
          registry: ghcr.io
username: ${{ github.actor }}
password: ${{ secrets.GITHUB_TOKEN }}
```

**📖 Build Job with Registry Login**

`needs: test` — only runs if the test job passed. Tests guard the build.  
  
`permissions.packages: write` — explicitly grants this job permission to push to GitHub Container Registry (ghcr.io). GitHub Actions uses least-privilege by default — you must explicitly request each permission your job needs.  
  
`secrets.GITHUB_TOKEN` — an automatic token GitHub creates for every workflow run. It authenticates to the repo's container registry without needing a manually created API token.

```
      - name: Build and push image
uses: docker/build-push-action@v5
with:
          push: true
tags: ${{ env.IMAGE }}:${{ env.TAG }}
update-manifest:
    needs: build-push
runs-on: ubuntu-latest
steps:
      - uses: actions/checkout@v4
with:
          repository: cloudnest/k8s-config
token:      ${{ secrets.GITOPS_TOKEN }}
```

**📖 The GitOps Update Step**

The `update-manifest` job checks out the separate k8s-config GitOps repository (using a PAT stored as a secret) and updates the image tag in the deployment YAML.  
  
This is the GitOps deploy pattern: instead of running `kubectl apply` in the pipeline, you update the YAML file in Git. Argo CD detects the change and syncs the cluster. The pipeline never directly touches the cluster — Git is the intermediary.  
  
The tag `${{ env.TAG }}` uses the workflow-level variable defined at the top.

```bash
      - name: Update image tag
run: |-
yq eval -i '.spec.template.spec.containers[0].image = "${{ env.IMAGE }}:${{ env.TAG }}"' apps/cloudnest-api/deployment.yaml
git config user.email "bot@cloudnest.in"
git config user.name "CloudNest CI Bot"
git commit -am "deploy: update api to ${{ env.TAG }}"
git push
```

**📖 yq + git in a Pipeline Step**

This step uses `yq eval -i` to update the image tag in the Kubernetes Deployment YAML in the GitOps repo, then commits and pushes the change.  
  
The commit message follows the convention `deploy: action description` — Argo CD shows this message in its UI as the reason for the sync.  
  
After this push, Argo CD detects the change in the GitOps repo and applies the updated Deployment to the cluster within ~2 minutes. Complete audit trail: who changed what, when, through which pipeline run.

> **The 10 YAML Rules Every Professional DevOps Engineer Lives By**

> - Never use tabs — always spaces. Set your editor to show whitespace characters and auto-convert tabs. This rule alone eliminates 40% of all YAML parse errors.
> - Colon must always be followed by a space. `key:value` without a space is not a key-value pair — it's a string. The space is not optional.
> - Quote anything that could be auto-detected as a non-string type — booleans (true/false/yes/no), numbers you want to keep as strings, and values containing special characters.
> - Use anchors to avoid copy-pasting config blocks. If you copy-paste more than once within a file, use `&anchor` and `<<: *alias`. If you need cross-file reuse, use Kustomize or Helm.
> - Always validate before applying — run yamllint for syntax, kubeconform or kubectl --dry-run for semantic validation. Never apply to production without a dry run.
> - Use `---` to group related Kubernetes resources in one file. One kubectl apply deploys everything. One git diff shows all changes for one feature.
> - Never put secrets in YAML files committed to Git. Reference environment variables. Use Kubernetes Secrets with external secret managers in production.
> - Use specific image tags in Kubernetes manifests — never `:latest`. Specific tags make rollbacks possible and deployments reproducible.
> - All YAML config files belong in Git and change through pull requests, not manual editing. The Git history is your audit log and rollback mechanism.
> - When a YAML error message is confusing, use Python's `yaml.safe_load()` to parse the file and `pprint` the structure — seeing the data as Python objects immediately reveals indentation and type problems.

##### CloudNest YAML Engineering Standards — Final Checklist

- Every YAML file passes yamllint before it can be merged to main. This is enforced in the CI pipeline — not a suggestion.
- Every Kubernetes YAML file passes kubeconform validation before merge. Field name typos and missing required fields are caught at review time, not deploy time.
- Kubernetes Deployments in production always have: resource requests + limits, liveness probe, readiness probe, and a specific image tag. PRs that add a Deployment without these are rejected in code review.
- No secrets in YAML files in Git — ever. The linter is configured to flag anything that looks like a password or API key pattern. Human reviewers double-check sensitive sections.
- The GitOps repo (k8s-config) has branch protection: no direct pushes to main, required review from one senior engineer, required CI checks to pass. The cluster mirrors main. Main is sacred.
- New team members spend their first sprint reading and understanding the YAML files for two existing services before writing any themselves. Reading YAML is a skill that compounds — the more manifests you read, the faster you write your own.

##### 🏋️ Final Hands-On Challenge — Build the Full CloudNest YAML Stack

1. **Write a complete multi-document Kubernetes manifest:** One YAML file with `---` separators containing: a Namespace, a ConfigMap, a Secret (base64-encoded values), a Deployment with 3 replicas + resource limits + both probes, and a ClusterIP Service. Apply it with one `kubectl apply -f` command.
2. **Use anchors to eliminate duplication:** Write a docker-compose.yml for a 3-tier app (frontend, backend, database). Define a `&logging-defaults` anchor with shared logging configuration and merge it into all three services. Change the log driver name in one place and confirm all three services update.
3. **Write a complete GitHub Actions workflow:** Triggers on push to main. Job 1: run tests. Job 2 (needs Job 1): build and push a Docker image to GitHub Container Registry. Job 3 (needs Job 2): use yq to update the image tag in a Kubernetes deployment.yaml file and commit the change. Use GitHub Secrets for the registry credentials.
4. **Create a Kustomize overlay:** Start with a base Deployment YAML (2 replicas, image tag v1.0). Create a production overlay that changes replicas to 5 and image tag to v1.5 using a strategic merge patch. Run `kubectl apply -k overlays/production/ --dry-run=client` to verify the rendered output.
5. **Validate your YAML with all three tools:** Take any YAML file from this course. Run yamllint on it. Run kubeconform on it (if it's a Kubernetes manifest). Parse it with `python3 -c "import yaml,pprint; pprint.pprint(yaml.safe_load(open('f.yaml')))"` and verify the data types match what you expect. Fix any issues found.
