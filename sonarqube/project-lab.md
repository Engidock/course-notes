# SonarQube Project Mastery

> **👋 Hey Fresher — Read This First!**

> - SonarQube is the **industry-standard static code analysis tool** — used by TCS, Infosys, Wipro, Accenture delivery teams and every serious software product company. "Static analysis" means it reads your code without running it and finds bugs, security holes, and poor coding practices.
> - In DevOps, SonarQube sits inside your **CI/CD pipeline** as a quality gate — code that fails the quality check is automatically blocked from merging or deploying. No human review needed for common mistakes.
> - Every code block is **short and focused**. Every explanation is **bullet points**. SonarQube dashboard previews show exactly what you see in the browser after each configuration.
> - **Company in this project:** CodeCraft Technologies — a Pune-based software product company building a banking application. They ship code 10+ times a day. Before SonarQube, bugs and SQL injection vulnerabilities were reaching production. You just joined as a Junior DevOps Engineer. Your lead is Meera. You will set up SonarQube and integrate it into every team's CI/CD pipeline.

📦 Where SonarQube Lives in a Real DevOps Stack

Developer pushes code → GitHub / GitLab / Bitbucket → Jenkins / GitHub Actions → SonarQube Analysis → Quality Gate Pass/Fail → Deploy or Block

Maven / Gradle / npm build → sonar-scanner (SonarQube) → Results in SonarQube dashboard

JaCoCo / Istanbul coverage → SonarQube reads coverage report → Coverage % shown in Quality Gate

Pull Request opened → SonarQube PR decoration → Inline comments on PR with issues

SonarQube is the quality checkpoint between writing code and deploying code. It catches: bugs your tests might miss, security vulnerabilities like SQL injection, code that is overly complex (hard to maintain), and code paths with no test coverage. The pipeline blocks if quality gate fails — bad code literally cannot proceed.

#### What You Will Learn in This Project

> **🏗️ Phase 1 — Install & Configure SonarQube**

> Install SonarQube with Docker, configure PostgreSQL backend, create projects, generate tokens, and understand the SonarQube dashboard.

> Install sonar-scanner, configure sonar-project.properties, run analysis on a Java and Node.js project, understand what each metric means.

> Understand the default quality gate, create a custom quality gate with your team's standards, set coverage thresholds, and configure blocker/critical issue limits.

> Add SonarQube analysis to Jenkins pipelines and GitHub Actions workflows. Block builds that fail the quality gate. Configure PR decoration for inline code review comments.

> Understand OWASP rules in SonarQube, customise Quality Profiles for different languages, configure issue exclusions, and manage technical debt.

Installation, Quality Gates, Bugs & Vulnerabilities, Code Smell, Coverage, Jenkins CI, GitHub Actions, OWASP Security

**Scene 1 — CodeCraft Technologies, Pune | The Production Bug**

> **Meera** _Senior DevOps Engineer — CodeCraft Technologies_
> 
> Kiran, last Friday a SQL injection vulnerability reached production. A security auditor showed us how someone could dump the entire customer database through our login form. The code passed all unit tests. The tests never tested for malicious input. SonarQube would have caught this at the moment the developer committed — before it ever reached any environment. Today we're setting it up. Every commit, every pull request, every build — SonarQube analyses it. If it finds a blocker or critical security issue, the pipeline stops. The code does not merge. The deployment does not happen.

> **Kiran (You)** _Junior DevOps Engineer — Day 1 at CodeCraft_
> 
> How does SonarQube know what's a bug versus what's just a different coding style?

> **Meera** _Senior DevOps Engineer_
> 
> SonarQube has thousands of built-in rules organised into categories. Bugs are code that is definitely wrong — null pointer dereferences, resource leaks, division by zero. Vulnerabilities are security weaknesses — SQL injection, hardcoded passwords, insecure random number generation. Code Smells are maintainability problems — functions that are 300 lines long, duplicate code, unused variables. You configure which rules matter to your team, set thresholds, and SonarQube enforces them automatically on every commit.

### 1. Phase 1 — Installing and Configuring SonarQube

> **SonarQube Architecture — What the Components Are**

> - **SonarQube Server** — the web application where you see dashboards, configure rules, manage projects. Runs on port 9000 by default.
> - **SonarQube Database** — stores all analysis results, issues, quality gate history. Supports PostgreSQL (recommended for production), MySQL, or Oracle. Default embedded H2 database is for evaluation only — not production.
> - **sonar-scanner** — a CLI tool installed on build agents (Jenkins, GitHub Actions runner, developer laptops). It reads your source code, runs the analysis, and sends results to the SonarQube Server.
> - **Quality Gate** — a set of conditions (e.g. coverage > 80%, no new blocker bugs). If the analysis fails these conditions, the pipeline is marked as failed and blocks deployment.
> - **SonarLint** — an IDE plugin (VS Code, IntelliJ, Eclipse) that shows SonarQube issues in real time as you type, before you even commit. Connects to your SonarQube server for consistent rules.

🔧 Where Installation Fits in the Stack

Linux Server (Ubuntu 22.04) → SonarQube + PostgreSQL (Docker) → http://sonarqube.codecraft.in:9000

Jenkins Build Agent → sonar-scanner installed → Sends results to SonarQube Server

#### 1.1 Install SonarQube with Docker Compose

```
# docker-compose.yml — SonarQube + PostgreSQL
version: '3.8'
services:
sonarqube:
image: sonarqube:10.4-community
ports:
      - "9000:9000"
environment:
SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
SONAR_JDBC_USERNAME: sonar
SONAR_JDBC_PASSWORD: sonar_pass
depends_on:
      - db
db:
image: postgres:15
environment:
POSTGRES_DB: sonar
POSTGRES_USER: sonar
POSTGRES_PASSWORD: sonar_pass
```

**📖 Docker Compose Setup**

- **sonarqube:10.4-community** — the Community (free) edition. Supports Java, Python, JavaScript, TypeScript, Go, C#, and more. Enterprise edition adds security reports and portfolio management.
- **SONAR_JDBC_URL** — points SonarQube to PostgreSQL. Never use the default H2 embedded database in production — it doesn't support concurrent access and loses data on container restart.
- **depends_on: db** — ensures PostgreSQL starts before SonarQube tries to connect. SonarQube will fail to start if the database isn't ready.
- First start takes 2–5 minutes — SonarQube runs database migrations on startup. After that, access at `http://localhost:9000`.
- Default credentials: username `admin`, password `admin`. Change immediately on first login.

```bash
# Start SonarQube
docker compose up -d
# Watch logs until "SonarQube is operational"
docker compose logs -f sonarqube

# Required: set vm.max_map_count (Elasticsearch needs it)
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072
# Make it permanent across reboots
echo "vm.max_map_count=524288" >> /etc/sysctl.conf
```

**📖 Critical Linux Pre-requisites**

- **vm.max_map_count** — SonarQube uses Elasticsearch internally for full-text search. Elasticsearch requires this kernel parameter to be at least 524288. Without it, SonarQube exits silently.
- **docker compose logs -f** — follow logs until you see `SonarQube is operational`. Don't try to access the UI before this message.
- If SonarQube crashes on startup, 90% of the time it's either a missing vm.max_map_count or the database isn't ready yet. Check logs first.
- Production setup: use a **reverse proxy (nginx)** in front of SonarQube on port 80/443 with HTTPS. Expose port 9000 only on localhost, not publicly.

#### 1.2 Create a Project and Generate a Token

```bash
# Create project via SonarQube API
# (or use the web UI: Projects > Create Project)
curl -u admin:admin_password \
  -X POST \
  "http://localhost:9000/api/projects/create" \
  -d "name=banking-app&project=banking-app&visibility=private"
# Generate an analysis token
# (UI: User > My Account > Security > Generate Token)
curl -u admin:admin_password \
  -X POST \
  "http://localhost:9000/api/user_tokens/generate" \
  -d "name=jenkins-token&type=GLOBAL_ANALYSIS_TOKEN"
```

**📖 Projects and Tokens**

- **Project key** — a unique identifier for the project (banking-app). This is what sonar-scanner uses to link analysis results to the right project in the dashboard.
- **Token** — SonarQube authentication credential for CI/CD tools. Never use username/password in pipelines. Generate a project-specific or global analysis token and store it as a CI/CD secret.
- **Token types:** User Token (tied to a user account), Global Analysis Token (for CI/CD, not tied to a user — recommended), Project Analysis Token (limited to one project).
- Store the token immediately — SonarQube shows it only once. If you lose it, generate a new one.
- **Visibility: private** — only authenticated users can see this project's results. Use private for proprietary code; public for open-source.

### 2. Phase 2 — Running Your First Analysis

**Business Problem:** CodeCraft's banking application needs its first SonarQube analysis to establish a baseline. You'll see the current state of the code — how many bugs, vulnerabilities, and code smells exist — before making any rules mandatory.

🔧 Where sonar-scanner Fits in the Stack

Developer Laptop / Jenkins Agent → sonar-scanner (reads source + tests) → HTTP POST → SonarQube Server → Dashboard updates

sonar-scanner is separate from SonarQube server. It runs on the build machine, analyses code locally, then uploads the report. The server never pulls your code — only receives the analysis report.

#### 2.1 Configure sonar-project.properties

```
# sonar-project.properties — in project root
# Project identity
sonar.projectKey=banking-app
sonar.projectName=CodeCraft Banking Application
sonar.projectVersion=1.0
# Source code location
sonar.sources=src/main
sonar.tests=src/test
# Coverage report (generated by JaCoCo)
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
# SonarQube server URL
sonar.host.url=http://sonarqube.codecraft.in:9000
```

**📖 sonar-project.properties**

- **sonar.projectKey** — must match the project key you created in SonarQube. This links the scanner output to the correct dashboard project.
- **sonar.sources** — where your production source code is. Do NOT include test code here — it would inflate your technical debt numbers incorrectly.
- **sonar.tests** — where your test code is. SonarQube analyses tests separately — fewer rules apply, but it uses test paths to calculate coverage.
- **sonar.coverage.jacoco.xmlReportPaths** — path to the JaCoCo XML coverage report. SonarQube does NOT run tests — it reads the coverage report produced by your test tool. For JavaScript/Node.js: use `sonar.javascript.lcov.reportPaths=coverage/lcov.info`.
- The token is NOT in this file — it's passed as a command-line argument or environment variable to avoid committing it to Git.

#### 2.2 Run the Analysis

```
# For Java with Maven — simplest integration
mvn clean verify \
    sonar:sonar \
    -Dsonar.host.url=http://sonarqube.codecraft.in:9000 \
    -Dsonar.login=$SONAR_TOKEN
# For JavaScript / Node.js projects
sonar-scanner \
  -Dsonar.projectKey=banking-app \
  -Dsonar.sources=src \
  -Dsonar.host.url=http://sonarqube.codecraft.in:9000 \
  -Dsonar.login=$SONAR_TOKEN
```

**📖 Running the Scanner**

- **mvn sonar:sonar** — Maven plugin. Runs tests + coverage (via `verify`) then uploads to SonarQube. The easiest approach for Java projects.
- **sonar-scanner** — standalone CLI for non-Maven projects. Download from sonarqube.org and add to PATH. Same parameters as Maven plugin.
- **-Dsonar.login=$SONAR_TOKEN** — pass the token as an environment variable, never hardcoded in the command. In CI/CD, set `SONAR_TOKEN` as a secret variable.
- Analysis duration depends on codebase size: 10,000 lines = ~30 seconds, 500,000 lines = ~10 minutes. Keep the report size manageable by excluding generated code and third-party libraries.
- For Python: use `sonar-scanner` with `sonar.python.coverage.reportPaths=coverage.xml`. For .NET: use `dotnet-sonarscanner` wrapper.

SonarQube

Projects › banking-app

3

Bugs

2

Vulnerabilities

47

Code Smells

62%

Coverage

3.2%

Duplication

2d 4h

Technical Debt

Top Issues Found

CRITICAL SQL injection vulnerability — string concatenation in query LoginService.java:47

BLOCKER Null pointer dereference — result never checked for null AccountDAO.java:89

MAJOR Method has cyclomatic complexity of 24 (threshold: 10) PaymentService.java:112

#### 2.3 Understanding Issue Types and Severities

- **🐛 Bug** — 

- **🔐 Vulnerability** — 

- **🌀 Code Smell** — 

- **🔒 Security Hotspot** — 

### 3. Phase 3 — Quality Gates

**Business Problem:** CodeCraft needs an automated enforcement mechanism — no human should have to manually check if code meets quality standards before merging. The Quality Gate is this enforcement: configure it once, and every analysis automatically passes or fails against it.

🔧 How Quality Gates Block the Pipeline

sonar-scanner finishes analysis → Quality Gate evaluated (SonarQube) → Pipeline waits for result

Quality Gate: PASSED → Jenkins continues → Deploy to staging

Quality Gate: FAILED → Jenkins fails build → No deployment → Developer gets email alert

The Quality Gate is what makes SonarQube a mandatory gate rather than an optional dashboard. Without it, developers can ignore the issues and deploy anyway. With a blocking Quality Gate in the pipeline, they cannot.

#### 3.1 Default Quality Gate vs Custom

```bash
# Create a custom Quality Gate via API
curl -u admin:password -X POST \
  "http://localhost:9000/api/qualitygates/create" \
  -d "name=CodeCraft-Gate"
# Add condition: block if new Blocker bugs > 0
curl -u admin:password -X POST \
  "http://localhost:9000/api/qualitygates/create_condition" \
  -d "gateId=2&metric=new_blocker_violations&op=GT&error=0"
# Block if coverage on NEW code < 80%
curl -u admin:password -X POST \
  "http://localhost:9000/api/qualitygates/create_condition" \
  -d "gateId=2&metric=new_coverage&op=LT&error=80"
```

**📖 Quality Gate Conditions**

- **Default "Sonar Way" gate** — focuses on *new code* only: no new blocker/critical bugs, coverage on new code ≥ 80%, new duplications ≤ 3%. Good starting point for most teams.
- **new_blocker_violations GT 0** — "Greater Than 0 blockers on new code = fail." This catches the most severe issues without blocking legacy code debt.
- **new_coverage LT 80** — "Less Than 80% coverage on new code = fail." New code you write must have at least 80% test coverage. Forces test discipline without requiring fixing all legacy untested code at once.
- **Metrics on new code vs overall** — SonarQube distinguishes between new code (written since a reference date) and overall. New code gates are more realistic for teams with legacy debt.
- Always configure the gate to apply to a specific project: `api/qualitygates/select`.

#### 3.2 Check Quality Gate Status from CLI

```bash
# Poll quality gate result after analysis
curl -u admin:password \
  "http://localhost:9000/api/qualitygates/project_status?projectKey=banking-app"
# Result:
{
  "projectStatus": {
    "status": "ERROR",
    "conditions": [
      {
        "status":         "ERROR",
        "metricKey":      "new_coverage",
        "comparator":     "LT",
        "errorThreshold": "80",
        "actualValue":    "62"
      }
    ]
  }
}
```

**📖 Gate Status API**

- **status: "ERROR"** — Quality Gate failed. Pipeline should fail the build.
- **status: "OK"** — Quality Gate passed. Pipeline continues.
- In Jenkins and GitHub Actions, the SonarQube plugin calls this API automatically after the scanner finishes. You don't need to call it manually in pipelines — but it's useful for custom scripts.
- **actualValue: "62"** vs **errorThreshold: "80"** — clearly shows WHY the gate failed (coverage 62% < required 80%). Developers see this in the pipeline logs.
- The pipeline must use `waitForQualityGate()` (Jenkins) or `sonarqube-quality-gate-check` action (GitHub Actions) — without this step, the pipeline doesn't wait for results and always passes.

### 4. Phase 4 — Jenkins and GitHub Actions Integration

**Business Problem:** Setting up SonarQube is useless if developers can bypass it. The Quality Gate must be mandatory — triggered automatically on every pull request and every push to main. Manual analysis that developers run themselves is optional and gets skipped. Automated pipeline integration is not optional.

🔧 Full CI/CD + SonarQube Integration Flow

PR created on GitHub → GitHub Actions triggered → mvn test (run tests + coverage) → SonarQube analysis + gate check → PR blocked or approved

Jenkins push to main → Maven build + test → sonar:sonar + waitForQualityGate → Deploy to staging (if passed)

#### 4.1 Jenkins Pipeline Integration

```
// Jenkinsfile — SonarQube + Quality Gate
pipeline {
  agent any
environment {
    SONAR_TOKEN = credentials('sonar-token')
  }
  stages {
    stage('Test') {
      steps {
        sh 'mvn clean verify'
      }
    }
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh 'mvn sonar:sonar'
        }
      }
    }
    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }
  }
}
```

**📖 Jenkins SonarQube Integration**

- **credentials('sonar-token')** — Jenkins Credentials store. Never paste the token directly in Jenkinsfile. Store it as a Secret Text in Jenkins → Manage Jenkins → Credentials.
- **withSonarQubeEnv('SonarQube')** — Jenkins SonarQube plugin wrapper. Automatically sets SONAR_HOST_URL and authentication. Requires the SonarQube server to be configured in Jenkins settings (Manage Jenkins → System).
- **waitForQualityGate abortPipeline: true** — the critical step. Jenkins pauses here (up to 5 minutes) waiting for SonarQube to finish evaluation. If gate fails, the pipeline is aborted and marked red. Without this step, the build always passes regardless of code quality.
- **timeout(time: 5, MINUTES)** — prevents the pipeline from hanging forever if SonarQube is slow to respond. Adjust based on your analysis duration.

#### 4.2 GitHub Actions Integration

```
# .github/workflows/sonarqube.yml
name: SonarQube Analysis
on:
push:
branches: [main, develop]
  pull_request:
types: [opened, synchronize]

jobs:
sonarqube:
runs-on: ubuntu-latest
steps:
    - uses: actions/checkout@v4
with:
fetch-depth: 0
    - name: Run Tests with Coverage
run: mvn clean verify
    - name: SonarQube Scan
uses: SonarSource/sonarqube-scan-action@master
env:
SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

**📖 GitHub Actions Integration**

- **fetch-depth: 0** — SonarQube needs full Git history to determine "new code" (what changed since the last analysis). Without this, GitHub Actions does a shallow clone and new code detection fails.
- **pull_request trigger** — runs on every PR open/update. SonarQube adds inline comments directly on the PR showing exactly which lines have issues — developers see problems in code review without visiting the SonarQube dashboard.
- **secrets.SONAR_TOKEN** — stored in GitHub repository Settings → Secrets and variables → Actions. Never hardcode in the YAML file.
- **SonarSource/sonarqube-scan-action** — official SonarQube GitHub Action. Handles sonar-scanner installation, configuration, and quality gate check. Fails the workflow step if quality gate fails.
- For GitHub.com (not self-hosted): use **sonarcloud** (SonarQube cloud version) — same concepts, different action: `SonarSource/sonarcloud-github-action`.

SonarQube

Projects › banking-app › Activity

Analysis date 29 Mar 2026 14:32 IST

New code 342 lines added / 18 files changed

New bugs 1 (threshold: 0)

New vulnerabilities 1 (threshold: 0)

Coverage on new code 61.4% (threshold: 80%)

Why gate failed Coverage 61.4% < required 80% on new code | 1 new critical vulnerability

⛔ Pipeline blocked — PR cannot merge until quality gate passes. Jenkins build #247 marked FAILED.

### 5. Phase 5 — Security Analysis, Quality Profiles and Advanced Configuration

**Business Problem:** CodeCraft's banking application must comply with OWASP Top 10 security standards. Different teams use different languages (Java backend, React frontend). Quality Profiles let you apply different rule sets per language. Some generated code and third-party code must be excluded from analysis.

🔧 Advanced SonarQube in the Organisation Stack

Java backend team → Quality Profile: Java-Banking (strict OWASP) → banking-api project

React frontend team → Quality Profile: JS-Standard → banking-frontend project

Generated code (Protobuf, Swagger) → sonar.exclusions (excluded from analysis) → Not counted in metrics

#### 5.1 Quality Profiles — Rules Per Language

```bash
# Copy the default profile and customise it
# (UI: Quality Profiles > Java > Copy > Edit Rules)
# Via API: activate a specific rule in a profile
curl -u admin:password -X POST \
  "http://localhost:9000/api/qualityprofiles/activate_rule" \
  -d "key=java-banking-profile&rule=java:S3649&severity=CRITICAL"
# S3649 = SQL injection rule (OWASP A01:2021)
# Assign profile to a project
curl -u admin:password -X POST \
  "http://localhost:9000/api/qualityprofiles/add_project" \
  -d "language=java&qualityProfile=java-banking-profile&project=banking-app"
```

**📖 Quality Profiles**

- **Quality Profile** — a collection of active rules for a specific programming language. SonarQube comes with "Sonar way" (the default profile for each language).
- Create custom profiles by copying and modifying "Sonar way." Add banking-specific OWASP rules, increase severity of SQL injection rules to BLOCKER, deactivate rules irrelevant to your domain.
- **Rule S3649** — SQL injection detection in Java. By activating it at CRITICAL severity, any SQL injection found will fail the quality gate.
- **OWASP Top 10 rules** — SonarQube has built-in rules tagged with OWASP categories. Filter by tag "owasp-a1" (injection), "owasp-a2" (broken authentication) etc. in Quality Profiles to see and activate all relevant rules.
- Each project can have one profile per language. A Java+JavaScript project gets a Java profile AND a JavaScript profile.

#### 5.2 Excluding Files from Analysis

```
# sonar-project.properties — exclusions
# Exclude generated code and third-party
sonar.exclusions=\
  **/generated/**,\
  **/vendor/**,\
  **/node_modules/**,\
  **/*.pb.java
# Exclude from coverage (generated/test infra)
sonar.coverage.exclusions=\
  **/config/**,\
  **/*Config.java,\
  **/*Application.java
# Exclude from duplication check
sonar.cpd.exclusions=\
  **/dto/**,\
  **/entity/**
```

**📖 Exclusions**

- **sonar.exclusions** — these files are completely ignored. Generated code (Protobuf, Swagger, JOOQ), third-party libraries, and vendor code should be excluded. Otherwise technical debt numbers are inflated by code you didn't write.
- **sonar.coverage.exclusions** — still analysed for bugs/smells but not counted in coverage. Use for boilerplate Spring Boot configuration classes and main application entry points that can't easily be unit tested.
- **sonar.cpd.exclusions** — excluded from duplication check. DTOs and entities often have similar field definitions — marking them as duplication is a false positive.
- **Glob patterns** — `**/` matches any directory depth. `*.pb.java` matches any file ending in .pb.java (Protobuf generated files).
- Don't over-exclude — excluding real application code hides real problems. Only exclude code you have no control over.

#### 5.3 SonarLint — Catch Issues Before Committing

```
# SonarLint VS Code — settings.json
{
  "sonarlint.connectedMode.connections.sonarqube": [{
    "connectionId": "codecraft",
    "serverUrl": "http://sonarqube.codecraft.in:9000"
  }],
  "sonarlint.connectedMode.project": {
    "connectionId":  "codecraft",
    "projectKey":   "banking-app"
  }
}
```

**📖 SonarLint Connected Mode**

- **SonarLint** — a free IDE extension (VS Code, IntelliJ, Eclipse). Shows SonarQube issues in real time as you type — underlines the exact line, shows the rule description and fix suggestion.
- **Connected mode** — SonarLint syncs with your SonarQube server to use the same Quality Profile and rule configuration. Without this, SonarLint uses its own default rules which may differ from what SonarQube enforces.
- A developer sees the SQL injection warning in their IDE before they even save the file — the feedback loop is seconds, not minutes waiting for a CI pipeline.
- **Shift-Left** — the DevOps principle of catching problems as early as possible in the development cycle. SonarLint shifts quality checks to before the commit, SonarQube catches what SonarLint missed at CI time.
- SonarLint works offline and is always free — even if SonarQube Community isn't available, SonarLint still enforces many rules locally.

### 6. SonarQube Concepts & Commands Quick Reference

Concept / Command

What It Does

Where It's Used

sonar-scanner

CLI tool that analyses code and uploads to SonarQube

Runs in CI/CD pipeline on build agents

mvn sonar:sonar

Maven plugin to run SonarQube analysis

Java Maven projects in Jenkins/GitHub Actions

sonar.projectKey

Unique identifier linking scanner to SonarQube project

sonar-project.properties

sonar.sources

Directory containing production source code

sonar-project.properties

sonar.tests

Directory containing test code

sonar-project.properties

sonar.login / SONAR_TOKEN

Authentication token for scanner

CI/CD secret variable — never hardcode

sonar.coverage.jacoco.xmlReportPaths

Path to JaCoCo coverage XML for Java projects

sonar-project.properties

sonar.exclusions

Files completely excluded from all analysis

Generated code, vendor, node_modules

Quality Gate

Set of conditions — pass/fail determines pipeline outcome

SonarQube UI → Quality Gates

waitForQualityGate

Jenkins step that pauses pipeline until gate result arrives

Jenkinsfile — after sonar:sonar stage

withSonarQubeEnv

Jenkins step that injects SonarQube server configuration

Jenkinsfile — wraps sonar:sonar

Quality Profile

Collection of active rules for one language

SonarQube UI → Quality Profiles

Bug

Definite code error — will cause wrong behaviour or crash

SonarQube Issues dashboard

Vulnerability

Security weakness exploitable by attackers

SonarQube Issues → Vulnerability tab

Code Smell

Maintainability problem — code works but is hard to maintain

SonarQube Issues dashboard

Security Hotspot

Sensitive code needing human review to confirm if vulnerable

SonarQube Security Hotspots tab

Coverage

% of code executed by test suite

Requires JaCoCo/Istanbul report

Technical Debt

Estimated time to fix all code smells

SonarQube project overview

Blocker

Most severe issue — must fix immediately

Configured in Quality Gate as threshold

Critical

High severity — fix before next release

SQL injection, NPE in critical path

Major / Minor / Info

Lower severities — fix in backlog

Code smells, style issues

SonarLint

IDE plugin for real-time code quality feedback

VS Code, IntelliJ, Eclipse

fetch-depth: 0

Full Git history needed for new code detection

GitHub Actions checkout step

PR Decoration

Inline comments on GitHub/GitLab PRs with issues

Requires SonarQube + GitHub App setup

New Code

Code added since last analysis baseline

Quality Gate typically targets new code only

### 7. Interview Questions — SonarQube

##### Interview Q&A — Fresher Level (0–1 Year SonarQube Experience)

**Q: Q1. What is SonarQube and what problem does it solve?**

A: SonarQube is a **static code analysis platform** — it reads source code without executing it and identifies bugs, security vulnerabilities, and code quality issues using thousands of built-in rules.
The problem it solves: human code review misses issues at scale. When a team ships 50+ pull requests per day, no reviewer has time to check every line for null pointer dereferences, SQL injection, or copy-paste code duplication. SonarQube automates this.
It integrates into CI/CD pipelines and acts as a mandatory quality gate — code that violates configured rules cannot be merged or deployed.
Unlike unit tests which test behaviour, SonarQube tests code structure — it catches entire categories of bugs that tests may not cover (resource leaks, security patterns, complexity that makes bugs more likely).

**Q: Q2. What is the difference between a Bug, Vulnerability, Code Smell, and Security Hotspot in SonarQube?**

A: **Bug** — code that is definitively wrong and will cause incorrect behaviour or crashes: null dereferences, resource leaks, logical errors. Fix immediately — these break functionality.
**Vulnerability** — security weakness that could be exploited: SQL injection, hardcoded credentials, weak encryption, XSS. Classified by OWASP/CWE standards. Fix before production — these cause data breaches.
**Code Smell** — code that works but is hard to maintain: 300-line methods, duplicate code blocks, unused imports, deeply nested conditionals. Accumulates as technical debt — estimate in days/hours to fix.
**Security Hotspot** — security-sensitive code that MIGHT be a vulnerability but requires human review to determine. Example: using MD5 is flagged, but whether it's a problem depends on what it's being used for. Developers must mark hotspots as "reviewed" (safe or fixed).

**Q: Q3. What is a Quality Gate and how does it work in a CI/CD pipeline?**

A: A Quality Gate is a set of conditions that analysis results must meet. Example: "no new blocker bugs, coverage on new code ≥ 80%, no new critical vulnerabilities."
After the sonar-scanner finishes, SonarQube evaluates the project against the configured Quality Gate and returns either PASSED or FAILED.
In Jenkins: `waitForQualityGate abortPipeline: true` — the pipeline pauses, gets the result, and if FAILED, marks the build as failed and stops the deployment stage from running.
In GitHub Actions: the SonarQube action step fails, which causes the entire workflow job to fail, which blocks the PR from being merged (if branch protection rules require the workflow to pass).
Without `waitForQualityGate` or equivalent, the pipeline doesn't pause for the gate result — quality gate failures go unnoticed and code still deploys. This step is mandatory for enforcement.

**Q: Q4. Why is fetch-depth: 0 important in GitHub Actions for SonarQube?**

A: By default, GitHub Actions performs a **shallow clone** — it only downloads the most recent commit, not the full Git history. This is faster but loses information about what changed between commits.
SonarQube uses the Git history to determine what is **"new code"** — lines added since the last analysis. Quality Gates typically apply conditions only to new code (not legacy debt), so SonarQube needs to know which lines are new.
Without full history (`fetch-depth: 0`), SonarQube treats ALL code as new — your quality gate fails on all existing issues, not just what the current PR introduced.
This also affects PR decoration — SonarQube needs history to accurately annotate only the changed lines with issues in the pull request view.

**Q: Q5. How do you exclude generated code from SonarQube analysis?**

A: Add `sonar.exclusions` to `sonar-project.properties` with glob patterns for the paths to exclude.
Common exclusions: `**/generated/**` (Protobuf, Swagger generated code), `**/vendor/**` (Go vendor directory), `**/node_modules/**` (npm packages), `**/*.pb.java` (Protobuf Java files).
Use `sonar.coverage.exclusions` for files you want analysed for bugs but NOT counted in coverage (e.g. Spring Boot Application.java, configuration classes).
Use `sonar.cpd.exclusions` for files to exclude from duplicate code detection (DTOs and entity classes legitimately have similar patterns).
Don't use exclusions to hide real problems — only exclude code you genuinely don't own or can't change (generated, vendor, legacy untouchable modules).

**Q: Q6. What is SonarLint and how does it complement SonarQube?**

A: **SonarLint** is an IDE extension (VS Code, IntelliJ, Eclipse) that shows SonarQube-style issues as you type — highlighting the exact line with the problem and showing a description of the rule.
It implements the "shift-left" principle — catching quality issues at the moment of writing code rather than after committing, saving the full CI/CD round-trip time (minutes to hours depending on queue).
**Connected mode** — SonarLint can synchronise with your SonarQube server to use the exact same Quality Profile and rules. This ensures consistency: what SonarLint warns about is exactly what SonarQube will flag in the pipeline.
SonarLint is free for all IDEs regardless of SonarQube edition. It works standalone (default rules) or connected to a SonarQube/SonarCloud server (team rules).
Combined workflow: SonarLint prevents most issues at development time → SonarQube catches anything that slipped through → Quality Gate blocks the rare case where a developer ignored SonarLint warnings.

**Quiz: Quiz 1 — A Jenkins pipeline has mvn sonar:sonar but NO waitForQualityGate step. The SonarQube analysis finds 3 critical vulnerabilities. What happens?**

- A) Jenkins fails the build because SonarQube sent a failure signal
- B) Jenkins continues and deploys successfully — the pipeline never checks the Quality Gate result
- C) Jenkins pauses for 5 minutes then fails
- D) The deployment is blocked by SonarQube's firewall integration

> **Answer/explanation:** ✅ Answer: **B — Jenkins continues and deploys successfully**
`mvn sonar:sonar` only sends the analysis to SonarQube — it doesn't check the Quality Gate result. The Maven command succeeds (exit code 0) as long as the upload was successful.
Without `waitForQualityGate`, the pipeline never fetches the Quality Gate status from SonarQube.
The 3 vulnerabilities appear in the SonarQube dashboard (and the gate is marked FAILED there) but the Jenkins pipeline has no knowledge of this — it already moved on to the deploy stage.
This is the most common SonarQube misconfiguration in interview discussions. The `waitForQualityGate abortPipeline: true` step is the enforcement mechanism — without it, SonarQube is a reporting tool only, not a gate.

**Quiz: Quiz 2 — The Quality Gate is set to "new coverage ≥ 80%." A developer adds 200 lines of new code but no tests. Current overall coverage is 85%. What is the Quality Gate result?**

- A) PASSED — overall coverage is 85% which exceeds 80%
- B) FAILED — new code (the 200 new lines) has 0% coverage, which is less than 80%
- C) PASSED — the condition only applies if total new lines exceed 500
- D) FAILED — any new code without tests causes automatic failure

> **Answer/explanation:** ✅ Answer: **B — FAILED because new code coverage is 0%**
SonarQube distinguishes between **overall coverage** (all code in the project) and **coverage on new code** (lines added since the last analysis).
The 200 new lines have no tests → new code coverage = 0%. The condition "new_coverage LT 80" is met → Quality Gate FAILS.
Overall coverage being 85% is irrelevant when the condition targets new code specifically. This is intentional — teams with legacy untested code can still enforce standards on new code without having to retroactively add tests to old code.
This is why "new code" quality gates are more practical than "overall" quality gates when working on projects with legacy technical debt.

**Quiz: Quiz 3 — A developer runs sonar-scanner and the analysis takes 45 minutes. The most likely cause is:**

- A) SonarQube server is overloaded
- B) The scanner is analysing node_modules, generated code, and test frameworks in addition to the actual source code (missing sonar.exclusions)
- C) The quality gate has too many conditions
- D) The project has too many blocker issues

> **Answer/explanation:** ✅ Answer: **B — Missing exclusions causing unnecessary file analysis**
SonarQube analysis time scales with the number of files analysed. node_modules can contain 100,000+ JavaScript files. generated/ directories can have thousands of auto-generated Java/Python files.
Without `sonar.exclusions=**/node_modules/**,**/generated/**`, SonarQube analyses all of these files — multiplying analysis time 10–100x unnecessarily.
Fix: add proper exclusions, then re-run. Analysis for a typical project should take 30 seconds to 5 minutes, not 45 minutes.
Quality gate conditions don't affect analysis time — they're evaluated server-side after the scanner finishes. The number of issues found also doesn't significantly affect scanner duration.

> **SonarQube Project — Core Takeaways for Freshers**

> - **waitForQualityGate is mandatory** — running sonar:sonar without waitForQualityGate makes SonarQube a reporting tool, not a gate. The pipeline must pause and check the gate result, with abortPipeline: true, to block bad code from deploying.
> - **Never use H2 database in production** — SonarQube's embedded H2 database loses data on container restart and doesn't support concurrent access. Always configure PostgreSQL (or MySQL) as the backend database.
> - **Use vm.max_map_count=524288** — SonarQube will silently fail to start without this Linux kernel parameter set. It's needed by Elasticsearch which SonarQube uses internally. Add it to /etc/sysctl.conf to survive reboots.
> - **fetch-depth: 0 in GitHub Actions** — without full Git history, SonarQube treats all code as new and quality gate conditions fire on legacy issues too. Always set this in the checkout step.
> - **Add sonar.exclusions for generated code** — node_modules, Protobuf generated files, and vendor directories inflate analysis time and report false issues on code you don't own. Exclude them from the start.
> - **Configure Quality Gate on NEW code** — teams with legacy technical debt can't fix everything at once. Setting Quality Gate conditions on new_coverage and new_bugs (not overall) lets you enforce standards on new development without being blocked by old problems.
> - **Install SonarLint in every developer's IDE** — catching issues at typing time is cheaper than catching them at commit time which is cheaper than catching them at PR review time which is much cheaper than catching them in production.
> - **Security Hotspots require human review** — they are NOT automatic fails. Every hotspot must be reviewed by a developer who marks it "safe" or "fixed." Don't ignore them — unreviewed hotspots accumulate and hide real vulnerabilities.

##### SonarQube Engineering Standards — CodeCraft Rules

- Store the SonarQube token ONLY in CI/CD secret variables (Jenkins Credentials, GitHub Actions Secrets) — never in sonar-project.properties, Jenkinsfile, or any file committed to Git
- Set separate Quality Profiles per language — Java rules don't apply to JavaScript. Use custom profiles for each language that enforce your organisation's specific security and style requirements
- Start with Quality Gate on new code only — retroactively requiring 80% coverage on all legacy code will block every PR until the entire codebase has tests. New code gates are achievable immediately
- Run SonarQube analysis AFTER tests complete — SonarQube needs the JaCoCo/Istanbul coverage report to show coverage metrics. Skipping tests to speed up the pipeline means coverage always shows 0%
- Configure SonarQube PR decoration with your Git provider (GitHub/GitLab/Bitbucket) — developers see inline issues directly on the PR diff without having to visit the SonarQube dashboard separately
- Schedule a monthly SonarQube review meeting — review the overall security hotspot backlog, trending technical debt, and discuss rules that generate too many false positives. SonarQube governance requires human attention, not just automation

##### 🏋️ Hands-On Exercises — CodeCraft SonarQube Setup

1. **Full Installation and First Analysis:** Deploy SonarQube + PostgreSQL using Docker Compose. After startup, change the default admin password. Create a project called "practice-java-app." Clone any Java project from GitHub (e.g. Spring PetClinic). Run `mvn clean verify sonar:sonar` with your local SonarQube URL and token. View the dashboard and find: the number of bugs, the worst vulnerability, and the method with the highest cyclomatic complexity.
2. **Custom Quality Gate:** Create a Quality Gate named "Strict-Gate" with these conditions: new bugs = 0, new vulnerabilities = 0, new coverage on new code ≥ 85%, new duplicated lines ≤ 2%. Assign it to your practice project. Re-run the analysis and check whether the gate passes or fails. If it fails, identify which condition failed and why. Modify a test to improve coverage and re-run until coverage passes.
3. **Jenkins Pipeline Integration:** Install the SonarQube Scanner plugin in Jenkins. Configure your SonarQube server in Jenkins (Manage Jenkins → System → SonarQube installations). Create a Jenkinsfile with stages: Checkout, Test, SonarQube Analysis, Quality Gate. Store the SonarQube token as a Jenkins credential. Run the pipeline and observe the output — especially the Quality Gate stage waiting for results. Deliberately lower the coverage threshold and confirm the pipeline fails the Quality Gate stage.
4. **GitHub Actions Integration:** Fork a Java repository on GitHub. Add the SonarQube workflow YAML to `.github/workflows/sonarqube.yml`. Add SONAR_TOKEN and SONAR_HOST_URL as GitHub repository secrets. Open a pull request and observe: (1) the GitHub Actions workflow runs, (2) SonarQube analysis appears in the dashboard, (3) if issues are found, they appear as inline comments on the PR diff (requires SonarQube PR decoration setup). Configure branch protection rules to require the SonarQube workflow to pass before merging.
5. **Security Rule Deep-Dive:** In SonarQube, go to Rules, filter by Type=Vulnerability and Tag=owasp-a1 (Injection). Read the description of the SQL injection rule (java:S3649). Find the CodeCraft banking application code that triggered this rule in the first analysis. Understand why SonarQube flagged it — trace the issue from the user input through to the SQL query. Fix it using a PreparedStatement. Re-run analysis and confirm the vulnerability disappears. Write a test that proves the fix works (send a SQL injection string as input and verify it doesn't break the query).

### SonarQube Project Complete 🎉

You have set up CodeCraft's complete SonarQube infrastructure — Docker installation with PostgreSQL, project creation and token management, first analysis with bug and vulnerability detection, custom Quality Gate configuration, Jenkins pipeline integration with waitForQualityGate, GitHub Actions workflow with PR decoration, security rule configuration, file exclusions, and SonarLint IDE setup. The SQL injection vulnerability that reached production last month? SonarQube would have blocked that pull request automatically.

> **Meera**
> 
> "Kiran, since we set up SonarQube last month: 12 critical vulnerabilities caught before merging, 3 SQL injection issues blocked by the quality gate, test coverage went from 58% to 79% because developers started writing tests to pass the gate. The banking team's last security audit found zero new vulnerabilities — the auditor specifically asked what changed. SonarQube is the answer."

> **Vikram**
> 
> "And the developers actually like it now. SonarLint in the IDE shows them the issue the moment they write it — they fix it immediately and it never becomes a pipeline failure. The ones who ignored SonarLint warnings learned quickly when their PR was blocked and they had to go back and fix 8 issues while the rest of the team waited for their feature. Peer pressure from the quality gate is more effective than any code review comment."

> **Next: Advanced Code Quality — OWASP ZAP, Dependency Check & SAST/DAST in DevSecOps**

> - OWASP Dependency-Check — scan your project's dependencies (Maven, npm, pip) for known CVE vulnerabilities before they reach production
> - OWASP ZAP (Zed Attack Proxy) — dynamic application security testing (DAST) — unlike SonarQube which analyses static code, ZAP actively attacks a running application to find runtime vulnerabilities
> - Trivy — container image vulnerability scanner; scan your Docker images in CI/CD before pushing to ECR/ACR
> - Snyk — developer-first security platform combining dependency scanning, container scanning, and IaC misconfiguration detection
> - DevSecOps pipeline — combining SonarQube (SAST) + Dependency-Check + Trivy + ZAP into a single security pipeline that runs on every deployment
> - SonarQube Enterprise — portfolio views, security reports, application branching analysis, and organisation-wide governance dashboards
