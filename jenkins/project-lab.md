# Jenkins Project Mastery

#### What You Will Build & Learn in This Project

This project mastery takes you through building a complete, production-grade CI/CD pipeline using Jenkins as the orchestration core. You will deploy a full-stack e-commerce application — **ShopAxis** — using Jenkins to wire together Maven, Docker, SonarQube, Nexus, Kubernetes, Prometheus, and Grafana. No application is built with Jenkins alone; Jenkins is the conductor that connects every tool.

Jenkins Pipelines, Jenkinsfile, Maven Build, SonarQube, Docker, Nexus Registry, Kubernetes Deploy, Prometheus + Grafana

**Scene 1 — CI/CD War Room, TechAxis Solutions | Sprint Planning Monday**

> **Agastya** _Senior DevOps Engineer — Pipeline Architecture Lead_
> 
> Team, the engineering leadership has asked us to ship the ShopAxis e-commerce platform with zero-downtime deployments and full automated testing by end of quarter. Right now deployments are manual, take four hours, and break production 40% of the time. Jenkins is going to fix that — but only if we build the pipeline correctly.

> **Vashisht** _Junior DevOps Engineer — Three months on the team_
> 
> I have heard Jenkins mentioned before. Is it just a build tool? Like, it compiles code?

> **Nandan** _Systems Architect — DevOps transformation lead_
> 
> Jenkins is not a build tool. Jenkins is a CI/CD orchestrator. It does not compile Java — Maven does. It does not run tests — JUnit does. It does not scan for vulnerabilities — SonarQube does. It does not store artifacts — Nexus does. It does not deploy to Kubernetes — kubectl does. Jenkins watches your Git repository, triggers all of those tools in the right order, and tells you the result. Jenkins is the conductor of the orchestra, not a musician.

> **Aarya** _DevOps Engineer — CI/CD specialist_
> 
> Think of it this way: every tool in our stack is brilliant at exactly one thing. Jenkins knows how to talk to all of them, and it does so automatically every time a developer pushes code. That is CI/CD — continuous integration and continuous delivery.

### 1. Understanding Jenkins — The CI/CD Orchestrator

Jenkins is an open-source automation server written in Java, first released in 2011 as a fork of Hudson. It is the most widely deployed CI/CD server in the world, with over 300,000 active installations and a plugin ecosystem of 1,800+ plugins that allow it to integrate with virtually every tool in the DevOps ecosystem.

The core function of Jenkins is simple: detect a trigger (usually a Git push), execute a series of steps defined in a Jenkinsfile, and report the result. What makes Jenkins powerful is that those steps can invoke any tool — compilers, test frameworks, security scanners, artifact stores, container registries, and deployment platforms.

> **Jenkins Core Philosophy: Pipeline as Code**

> Modern Jenkins uses the Jenkinsfile — a Groovy DSL file committed to your repository alongside your source code. This means your pipeline definition is version-controlled, peer-reviewed, auditable, and reproducible. Every change to the pipeline goes through the same Git review process as your application code. This is Pipeline as Code.

```
Jenkins CI/CD Orchestration Architecture
==========================================

  Developer          Jenkins              Integrated Tools
  ─────────          ───────              ────────────────
  git push ────────► SCM Poll / Webhook
                          │
                          ▼
                    [Checkout Stage]─────────────► Git Repository
                          │
                          ▼
                    [Build Stage]────────────────► Maven / Gradle
                          │                        (compile, package)
                          ▼
                    [Test Stage]─────────────────► JUnit / TestNG
                          │                        (unit + integration)
                          ▼
                    [Code Quality]───────────────► SonarQube
                          │                        (SAST, coverage)
                          ▼
                    [Security Scan]──────────────► OWASP Dependency Check
                          │                        Trivy (container scan)
                          ▼
                    [Artifact Store]─────────────► Nexus Repository
                          │                        (WAR, Docker image)
                          ▼
                    [Docker Build]───────────────► Docker Engine
                          │                        (image creation)
                          ▼
                    [Push Image]─────────────────► Docker Hub / ECR
                          │
                          ▼
                    [Deploy Staging]─────────────► Kubernetes (kubectl)
                          │                        Helm charts
                          ▼
                    [Smoke Test]─────────────────► Selenium / Postman
                          │
                          ▼
                    [Approve?]─────── Manual Gate ──── Engineering Lead
                          │
                          ▼
                    [Deploy Prod]────────────────► Kubernetes Production
                          │
                          ▼
                    [Notify]─────────────────────► Slack / Email
                          │
                          ▼
                    [Monitor]────────────────────► Prometheus + Grafana

  Jenkins orchestrates ALL of this — it does none of it alone.
```

#### What Jenkins Is and Is Not

Role

Jenkins Responsibility

The Actual Tool

Source Control

Watches Git, triggers on push

Git, GitHub, GitLab, Bitbucket

Build / Compile

Invokes the build tool

Maven, Gradle, npm, Make

Unit Testing

Runs test phase, collects results

JUnit, TestNG, Jest, PyTest

Code Quality

Calls SonarQube Scanner, reads gate

SonarQube, PMD, Checkstyle

Security Scanning

Invokes scanner, parses output

OWASP DC, Trivy, Snyk

Artifact Storage

Pushes artifacts to repository

Nexus, Artifactory, S3

Containerization

Runs docker build / push commands

Docker Engine, Buildah

Deployment

Calls kubectl / helm

Kubernetes, Helm, ArgoCD

Monitoring

Exposes build metrics

Prometheus, Grafana, ELK

Notification

Sends status messages

Slack, Email, PagerDuty

**Scene 2 — Architecture Whiteboard Session**

> **Vashisht**
> 
> So Jenkins is like a general manager — it does not do the actual work but it tells everyone else what to do and when?

> **Agastya**
> 
> Exactly. And what is powerful is that Jenkins captures the result of every step. If Maven compilation fails, Jenkins stops the pipeline immediately and notifies the team. Code does not reach production if any gate fails. Before Jenkins, our lead developer manually tested on his laptop, and half the time something worked on his machine but not on the server.

> **Nandan**
> 
> That is the classic "works on my machine" problem. CI/CD eliminates it because every build happens on an identical, clean environment — the Jenkins agent. No developer laptop variables, no forgotten environment differences.

> **Aarya**
> 
> And the Jenkinsfile lives in the repository. When you onboard a new project, you clone it, and the entire pipeline definition comes with it. No clicking through Jenkins UI, no undocumented tribal knowledge. Everything is code.

### 2. Project Overview — ShopAxis E-Commerce Platform

ShopAxis is a microservices-based e-commerce platform consisting of a React frontend, a Spring Boot backend API, and a PostgreSQL database. The project will be used throughout this mastery to demonstrate every aspect of Jenkins CI/CD at production scale.

#### ShopAxis — Application Architecture

Full-stack e-commerce platform — product catalog, cart, checkout, order management, and admin dashboard.

React 18 (Frontend) Spring Boot 3.x (API) PostgreSQL 15 (Database) Docker (Containers) Kubernetes (Orchestration) Nginx (Reverse Proxy) Redis (Cache)

Jenkins Pipeline Tools: Maven · JUnit · SonarQube · OWASP Dependency Check · Trivy · Nexus · Docker Hub · Helm · Prometheus · Grafana · Slack

```
ShopAxis Application Architecture
===================================

  Browser / Mobile
       │
       ▼
  ┌─────────────┐     ┌──────────────────────────────────┐
  │  Nginx       │────►│        React Frontend            │
  │  Load Balancer    │  Product List | Cart | Checkout   │
  └─────────────┘     └──────────────────────────────────┘
       │                            │ REST API calls
       ▼                            ▼
  ┌─────────────────────────────────────────────────────┐
  │            Spring Boot API Gateway                   │
  │  /api/products  /api/cart  /api/orders  /api/users   │
  └─────────────────────────────────────────────────────┘
         │               │                │
         ▼               ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────────┐
  │PostgreSQL│    │  Redis   │    │  Stripe API  │
  │ Products │    │  Cache   │    │  Payments    │
  │ Orders   │    │ Sessions │    └──────────────┘
  │ Users    │    └──────────┘
  └──────────┘

  Jenkins Pipeline manages build + test + scan + deploy
  of ALL components above in a single automated workflow
```

#### Project Repository Structure

```
# ShopAxis monorepo structure (Jenkins will build this)
shopaxis/
├── Jenkinsfile                   # ← The entire CI/CD pipeline definition
├── pom.xml                       # Maven parent POM
├── backend/
│   ├── src/main/java/com/shopaxis/
│   │   ├── api/                  # REST controllers
│   │   ├── service/              # Business logic
│   │   ├── repository/           # Spring Data JPA
│   │   └── model/                # Domain entities
│   ├── src/test/java/            # JUnit 5 tests
│   ├── Dockerfile
│   └── pom.xml
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   └── services/
│   ├── Dockerfile
│   └── package.json
├── k8s/                          # Kubernetes manifests
│   ├── helm/
│   │   └── shopaxis/
│   │       ├── Chart.yaml
│   │       ├── values.yaml
│   │       └── templates/
│   ├── namespace.yaml
│   ├── backend-deployment.yaml
│   ├── frontend-deployment.yaml
│   └── postgres-statefulset.yaml
├── sonar-project.properties      # SonarQube config
├── docker-compose.yml            # Local development
└── monitoring/
    ├── prometheus.yml
    └── grafana-dashboard.json
```

### 3. Jenkins Installation and Infrastructure Setup

Before writing a single Jenkinsfile, you need a Jenkins controller, one or more agents, and all integrated tool servers running and reachable. This section covers the complete infrastructure setup for the ShopAxis pipeline.

**Jenkins Infrastructure Components**

- **Jenkins Controller** — 

- **Jenkins Agents** — 

- **Integrated Servers** — 

#### Jenkins Installation on Ubuntu 22.04

```
# Step 1: Install Java (Jenkins requires Java 17+)
sudo apt update
sudo apt install fontconfig openjdk-17-jre -y
java -version
# Expected: openjdk version "17.x.x"
# Step 2: Add Jenkins repository and install
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y

# Step 3: Start and enable Jenkins service
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins

# Step 4: Retrieve initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Step 5: Add Jenkins user to Docker group (for Docker builds)
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

```
● jenkins.service - Jenkins Continuous Integration Server
     Loaded: loaded (/lib/systemd/system/jenkins.service; enabled)
     Active: active (running) since Mon 2026-01-13 09:00:12 UTC
   Main PID: 12345 (java)

Jenkins is accessible at: http://your-server-ip:8080
Initial admin password: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

#### Required Plugins for ShopAxis Pipeline

Plugin

Purpose

Used In Stage

Pipeline

Core Pipeline DSL support

All stages

Git Plugin

GitHub/GitLab checkout

Checkout

Maven Integration

Maven build automation

Build, Test

SonarQube Scanner

Code quality analysis

Code Quality

OWASP Dependency Check

Dependency vulnerability scan

Security

Docker Pipeline

Docker build/push in pipeline

Docker Build

Kubernetes CLI

kubectl commands in pipeline

Deploy

Nexus Artifact Uploader

Push artifacts to Nexus

Artifact Push

Slack Notification

Build status to Slack

Post-build

Blue Ocean

Visual pipeline dashboard

UI layer

Prometheus Metrics

Expose build metrics

Monitoring

Credentials Binding

Secure secret injection

All secure stages

**Scene 3 — The Plugin Conversation**

> **Vashisht**
> 
> Why does Jenkins need plugins for everything? Can it not just run Docker commands natively?

> **Agastya**
> 
> Jenkins is deliberately minimal by design. The core handles triggering and orchestration. Everything else is a plugin — this is what makes Jenkins so flexible. The Docker Pipeline plugin does not make Jenkins run Docker; Docker runs Docker. The plugin provides the withDockerRegistry() block in your Jenkinsfile so you can authenticate securely without hard-coding credentials. It is an abstraction layer, not the actual tool.

> **Nandan**
> 
> Think of plugins as language bindings. If you want to call Python from Java, you use a binding library. Jenkins plugins are bindings between Jenkinsfile DSL and the external tools. Without plugins, you would write raw shell commands everywhere, lose structured error handling, and have no UI integration.

### 4. Jenkins Credentials and Security Configuration

A CI/CD pipeline touches every sensitive system — source control, Docker registries, Kubernetes clusters, and databases. Jenkins Credentials Manager is the vault that stores all secrets and injects them into pipeline steps without ever exposing them in logs or code.

##### Critical Security Rule

Never hard-code passwords, API keys, tokens, or kubeconfig files in your Jenkinsfile or any version-controlled file. Always store secrets in Jenkins Credentials Manager and reference them by ID. Jenkins automatically masks credential values in build logs — if you hard-code them, they appear in plaintext logs accessible to anyone with build log access.

#### Credentials to Configure for ShopAxis

```
# Navigate: Jenkins Dashboard → Manage Jenkins → Credentials
# → System → Global credentials → Add Credentials
# 1. GitHub Access Token
Kind: Username with password
ID: github-credentials
Username: your-github-username
Password: ghp_xxxxxxxxxxxxxxxxxxxx (PAT token)

# 2. Docker Hub Credentials
Kind: Username with password
ID: dockerhub-credentials
Username: your-dockerhub-username
Password: your-dockerhub-password-or-token

# 3. SonarQube Token
Kind: Secret text
ID: sonar-token
Secret: squ_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 4. Nexus Credentials
Kind: Username with password
ID: nexus-credentials
Username: admin
Password: admin123

# 5. Kubernetes Config
Kind: Secret file
ID: kubeconfig
File: /path/to/kubeconfig (upload the file)

# 6. Slack Bot Token
Kind: Secret text
ID: slack-token
Secret: xoxb-xxxx-xxxx-xxxxxxxxxxxx
```

#### Using Credentials in Jenkinsfile

```
// Credentials Binding — multiple patterns
// Pattern 1: withCredentials block (most explicit)
withCredentials([usernamePassword(
    credentialsId: 'dockerhub-credentials',
    usernameVariable: 'DOCKER_USER',
    passwordVariable: 'DOCKER_PASS'
)]) {
    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
}

// Pattern 2: environment block (pipeline-wide)
environment {
    SONAR_TOKEN = credentials('sonar-token')
    NEXUS_CREDS = credentials('nexus-credentials')
    // NEXUS_CREDS_USR and NEXUS_CREDS_PSW auto-created
}

// Pattern 3: Secret file (kubeconfig)
withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
    sh 'kubectl get pods -n shopaxis'
}
```

### 5. The Complete ShopAxis Jenkinsfile

The Jenkinsfile is the heart of your CI/CD pipeline. Every stage, step, conditional, and notification is defined here. This is the complete production-grade Jenkinsfile for the ShopAxis application, with all eleven stages fully implemented.

##### Pipeline Stage Overview

1. Git Checkout 2. Compile 3. Unit Test 4. SonarQube 5. OWASP Scan 6. Trivy FS Scan 7. Build & Push JAR 8. Docker Build 9. Trivy Image Scan 10. Push to DockerHub 11. Deploy to K8s

```bash
// ShopAxis Complete Jenkinsfile — EngiDock DevOps Project Mastery
pipeline {
    agent any
    tools { maven 'maven3'; jdk 'jdk17' }
    environment {
        SCANNER_HOME  = tool 'sonar-scanner'
        APP_NAME      = 'shopaxis'
        IMAGE_TAG     = "${BUILD_NUMBER}"
        DOCKER_IMAGE  = "agastya/${APP_NAME}"
        NEXUS_URL     = 'http://nexus-server:8081'
        K8S_NAMESPACE = 'shopaxis-prod'
        SLACK_CHANNEL = '#shopaxis-deployments'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github-credentials',
                    url: 'https://github.com/techaxis/shopaxis.git'
                echo "✓ Code checked out — Build #${BUILD_NUMBER}"
            }
        }

        stage('Compile') {
            steps {
                dir('backend') { sh 'mvn compile -DskipTests=true' }
            }
        }

        stage('Unit Tests') {
            steps { dir('backend') { sh 'mvn test' } }
            post {
                always {
                    junit 'backend/target/surefire-reports/*.xml'
                    jacoco(execPattern: 'backend/target/jacoco.exec',
                           classPattern: 'backend/target/classes',
                           sourcePattern: 'backend/src/main/java')
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    dir('backend') {
                        sh '''$SCANNER_HOME/bin/sonar-scanner \
                          -Dsonar.projectKey=shopaxis-backend \
                          -Dsonar.projectName="ShopAxis Backend" \
                          -Dsonar.sources=src/main/java \
                          -Dsonar.tests=src/test/java \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                          -Dsonar.qualitygate.wait=true'''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP Dependency Scan') {
            steps {
                dir('backend') {
                    dependencyCheck additionalArguments: '--scan . --out . --format XML --failOnCVSS 7',
                                    odcInstallation: 'owasp-dc'
                    dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --exit-code 0 --severity HIGH,CRITICAL -o trivy-fs-report.txt .'
                archiveArtifacts artifacts: 'trivy-fs-report.txt'
            }
        }

        stage('Build JAR') {
            steps {
                dir('backend') {
                    sh 'mvn package -DskipTests=true'
                }
                echo "✓ JAR built: shopaxis-backend-${BUILD_NUMBER}.jar"
            }
        }

        stage('Push to Nexus') {
            steps {
                dir('backend') {
                    nexusArtifactUploader(nexusVersion: 'nexus3', protocol: 'http',
                        nexusUrl: 'nexus-server:8081', groupId: 'com.techaxis.shopaxis',
                        version: "${BUILD_NUMBER}", repository: 'shopaxis-releases',
                        credentialsId: 'nexus-credentials',
                        artifacts: [[artifactId: 'shopaxis-backend', classifier: '',
                                     file: "target/shopaxis-backend-0.0.1-SNAPSHOT.jar", type: 'jar']])
                }
                echo "✓ Artifact pushed to Nexus: build ${BUILD_NUMBER}"
            }
        }

        stage('Docker Build') {
            steps {
                dir('backend') {
                    sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
                    sh "docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} ${DOCKER_IMAGE}:latest"
                }
                echo "✓ Docker image built: ${DOCKER_IMAGE}:${IMAGE_TAG}"
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --exit-code 0 --severity HIGH,CRITICAL -o trivy-image-report.txt ${DOCKER_IMAGE}:${IMAGE_TAG}"
                archiveArtifacts artifacts: 'trivy-image-report.txt'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                    docker push ${DOCKER_IMAGE}:latest
                    docker logout'''
                }
                echo "✓ Image pushed: ${DOCKER_IMAGE}:${IMAGE_TAG}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                    helm upgrade --install shopaxis k8s/helm/shopaxis/ \
                      --namespace ${K8S_NAMESPACE} --create-namespace \
                      --set image.tag=${IMAGE_TAG} --set replicaCount=3 \
                      --wait --timeout 5m
                    kubectl rollout status deployment/shopaxis-backend -n ${K8S_NAMESPACE}
                    kubectl get pods -n ${K8S_NAMESPACE}
                    """
                }
                echo "✓ ShopAxis deployed to Kubernetes — Build ${IMAGE_TAG}"
            }
        }
    }

    post {
        always { sh "docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG} || true"; cleanWs() }
        success {
            slackSend channel: "${SLACK_CHANNEL}", color: 'good',
                message: "✅ ShopAxis #${BUILD_NUMBER} SUCCESS | ${DOCKER_IMAGE}:${IMAGE_TAG} | ${BUILD_URL}"
            emailext subject: "✅ ShopAxis Build #${BUILD_NUMBER} — SUCCESS",
                body: "Image: ${DOCKER_IMAGE}:${IMAGE_TAG}", to: 'devops-team@techaxis.com'
        }
        failure {
            slackSend channel: "${SLACK_CHANNEL}", color: 'danger',
                message: "❌ ShopAxis #${BUILD_NUMBER} FAILED | ${BUILD_URL}"
            emailext subject: "❌ ShopAxis Build #${BUILD_NUMBER} — FAILED",
                body: "Failed at: ${BUILD_URL}console", to: 'devops-team@techaxis.com'
        }
        unstable {
            slackSend channel: "${SLACK_CHANNEL}", color: 'warning',
                message: "⚠️ ShopAxis Build #${BUILD_NUMBER} UNSTABLE — check test results"
        }
    }
}
```

### 6. Backend Dockerfile — Spring Boot

The Dockerfile defines how the Spring Boot application is containerised. Jenkins calls `docker build` in the pipeline — it is Docker Engine that actually builds the image. Jenkins simply triggers the command and captures the output.

```dockerfile
# backend/Dockerfile — Multi-stage build for minimal production image
# ─── Stage 1: Build Stage ────────────────────────────────
FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder
LABEL maintainer="devops@techaxis.com"
LABEL app="shopaxis-backend"

WORKDIR /build

# Copy pom.xml first (layer caching — deps download only when pom changes)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source and build
COPY src ./src
RUN mvn package -DskipTests=true -B

# ─── Stage 2: Runtime Stage ──────────────────────────────
FROM eclipse-temurin:17-jre-alpine AS runtime

# Create non-root user (security best practice)
RUN addgroup -S shopaxis && adduser -S shopaxis -G shopaxis

WORKDIR /app

# Copy only the JAR from builder stage
COPY --from=builder /build/target/shopaxis-backend-*.jar app.jar

# Change ownership to non-root user
RUN chown -R shopaxis:shopaxis /app
USER shopaxis

# Expose application port
EXPOSE 8080

# Health check — Kubernetes uses this via livenessProbe
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

# JVM tuning for containers
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=70.0 -Djava.security.egd=file:/dev/./urandom"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

#### Frontend Dockerfile — React + Nginx

```dockerfile
# frontend/Dockerfile — Multi-stage React build
# Stage 1: Node build
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Stage 2: Nginx serve
FROM nginx:alpine AS runtime
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Scene 4 — Understanding Multi-Stage Builds**

> **Vashisht**
> 
> Why two stages in the Dockerfile? Could we not just install Maven in the final image and build there?

> **Aarya**
> 
> You could, but your production image would be 700 MB instead of 180 MB. The Maven build image includes Maven itself, the JDK, all the source code, and all the test dependencies. None of that should be in production. The multi-stage build discards everything from the builder stage and only copies the compiled JAR into the minimal runtime image. Smaller images mean faster pulls, less attack surface, and lower storage costs.

> **Agastya**
> 
> And Jenkins does not care about this at all — it just runs docker build. The multi-stage logic is entirely inside Docker. That is the principle of separation of concerns. Jenkins orchestrates; Docker builds; Kubernetes runs. Each tool does exactly what it is best at.

### 7. SonarQube Integration — Code Quality Gate

SonarQube performs Static Application Security Testing (SAST), measures code coverage, detects code smells, bugs, and security vulnerabilities. The Jenkins SonarQube Scanner plugin sends your source code to the SonarQube server for analysis. The pipeline then waits for the Quality Gate result — if it fails, the build fails.

```
SonarQube Quality Gate Flow
==============================

  Jenkins Pipeline
        │
        ├─ [SonarQube Analysis stage]
        │        │
        │        ▼
        │   sonar-scanner sends code + coverage to SonarQube server
        │        │
        │        ▼
        │   SonarQube processes:
        │    • Code smells (maintainability issues)
        │    • Bugs (reliability issues)
        │    • Vulnerabilities (security issues)
        │    • Code coverage (test coverage %)
        │    • Duplication (copy-paste code %)
        │        │
        ├─ [Quality Gate stage]
        │        │
        │        ▼
        │   waitForQualityGate()
        │        │
        │   ┌────┴────┐
        │   │         │
        │  PASS      FAIL
        │   │         │
        │   ▼         ▼
        │ Next      Pipeline
        │ Stage     ABORTED
        │           Slack Alert
        │           Email sent
```

#### SonarQube Quality Gate — Default Conditions

Metric

Condition for FAIL

Recommended Threshold

Coverage on New Code

Less than 80%

80% minimum

Duplicated Lines on New Code

Greater than 3%

< 3%

Maintainability Rating

Worse than A

A rating

Reliability Rating

Worse than A

A rating (0 bugs)

Security Rating

Worse than A

A rating (0 vulnerabilities)

Security Hotspots Reviewed

Less than 100%

100% reviewed

```
# sonar-project.properties — ShopAxis SonarQube configuration
sonar.projectKey=shopaxis-backend
sonar.projectName=ShopAxis Backend
sonar.projectVersion=1.0.${BUILD_NUMBER}

sonar.sources=src/main/java
sonar.tests=src/test/java
sonar.java.binaries=target/classes
sonar.java.test.binaries=target/test-classes

# Coverage report path (generated by JaCoCo Maven plugin)
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

# Exclusions — don't scan generated/configuration code
sonar.exclusions=**/generated/**,**/model/**,**/dto/**,**/*Config.java

# Quality Gate — wait for result before pipeline continues
sonar.qualitygate.wait=true
sonar.qualitygate.timeout=300
```

### 8. Nexus Repository — Artifact Management

Nexus Repository Manager is the artifact store where built JARs, WARs, and Docker images are stored before deployment. Jenkins pushes artifacts to Nexus after a successful build, creating an immutable, versioned record of every release. If a deployment fails, you can roll back to any previous artifact version stored in Nexus.

**Nexus Repository Types for ShopAxis**

- **Maven Hosted (Releases)** — 

- **Maven Hosted (Snapshots)** — 

- **Maven Proxy** — 

#### Maven settings.xml — Point to Nexus

```
<!-- /etc/maven/settings.xml — Point Maven to Nexus proxy -->
<settings>
  <servers>
    <server><id>nexus-releases</id>
      <username>${NEXUS_USER}</username><password>${NEXUS_PASS}</password></server>
  </servers>
  <mirrors>
    <mirror><id>nexus-public</id><mirrorOf>*</mirrorOf>
      <url>http://nexus-server:8081/repository/maven-public/</url></mirror>
  </mirrors>
  <profiles>
    <profile><id>nexus</id>
      <repositories>
        <repository><id>nexus-releases</id>
          <url>http://nexus-server:8081/repository/shopaxis-releases/</url></repository>
      </repositories>
    </profile>
  </profiles>
</settings>
```

### 9. Kubernetes Deployment — Helm Charts

Jenkins calls `helm upgrade --install` to deploy ShopAxis to Kubernetes. Helm is the package manager for Kubernetes — it templates the deployment YAML, manages releases, and enables rollbacks. Jenkins provides the image tag from the build and passes it to Helm as a value override.

#### Helm Chart Structure

```
# k8s/helm/shopaxis/
Chart.yaml          # Chart metadata
values.yaml         # Default values (overridden by Jenkins)
templates/
  ├── deployment.yaml      # Backend pod definition
  ├── service.yaml         # Kubernetes Service
  ├── ingress.yaml         # Nginx Ingress
  ├── hpa.yaml             # Horizontal Pod Autoscaler
  ├── configmap.yaml       # Application configuration
  └── secret.yaml          # DB password, API keys (sealed)
```

```
# k8s/helm/shopaxis/values.yaml
replicaCount: 3

image:
  repository: agastya/shopaxis
  tag: "latest"          # Jenkins overrides this with BUILD_NUMBER
  pullPolicy: Always

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  className: nginx
  host: shopaxis.techaxis.com
  tls: true

resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"

livenessProbe:
  path: /actuator/health/liveness
  initialDelaySeconds: 60
  periodSeconds: 30

readinessProbe:
  path: /actuator/health/readiness
  initialDelaySeconds: 30
  periodSeconds: 10

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

postgresql:
  enabled: true
  auth:
    database: shopaxis_db
    username: shopaxis_user
    existingSecret: shopaxis-db-secret
```

#### Kubernetes Deployment Manifest (Generated by Helm)

```yaml
# templates/deployment.yaml — Helm templated Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-backend
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Release.Name }}
    version: {{ .Values.image.tag }}
    managed-by: jenkins
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # One extra pod during update
      maxUnavailable: 0     # Zero downtime guarantee
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
        version: {{ .Values.image.tag }}
    spec:
      containers:
      - name: shopaxis-backend
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "shopaxis-postgresql"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: shopaxis-db-secret
              key: password
        resources: {{- toYaml .Values.resources | nindent 10 }}
        livenessProbe:
          httpGet: { path: /actuator/health/liveness, port: 8080 }
          initialDelaySeconds: 60; periodSeconds: 30
        readinessProbe:
          httpGet: { path: /actuator/health/readiness, port: 8080 }
          initialDelaySeconds: 30; periodSeconds: 10
```

### 10. Jenkins Agent Configuration — Docker-Based Agents

Using Docker-based agents means each pipeline build runs in a fresh, isolated container. No build pollution between runs, no dependency conflicts, and agents can be scaled on demand. Jenkins controller provisions Docker containers as agents automatically.

**Scene 5 — Why Docker Agents?**

> **Vashisht**
> 
> Our builds are getting slow because multiple pipelines are fighting over Maven dependencies and Java versions on the same agent. Is there a cleaner way?

> **Agastya**
> 
> Docker agents. Each pipeline stage can run in its own container with exactly the tools it needs. The Maven build runs in a maven:3.9-eclipse-temurin-17 container. The Node frontend build runs in node:20-alpine. The kubectl deploy runs in bitnami/kubectl. Each container is pulled fresh, used, and destroyed. No pollution, no conflicts, perfect isolation.

> **Nandan**
> 
> And you can specify this at the pipeline level or the stage level. If your backend and frontend builds need different environments, use a parallel pipeline with per-stage Docker agents. The build time drops dramatically because Maven and Node builds run simultaneously.

```bash
// Docker agent — per pipeline (single container for all stages)
pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17-alpine'
            args '-v /root/.m2:/root/.m2'  // Cache Maven dependencies across builds
        }
    }
    stages {
        stage('Build') { steps { sh 'mvn package -DskipTests' } }
    }
}

// Docker agent — per stage (different containers per stage)
pipeline {
    agent none
    stages {
        stage('Backend Build') {
            agent { docker { image 'maven:3.9.6-eclipse-temurin-17-alpine' } }
            steps { sh 'mvn package -DskipTests' }
        }
        stage('Frontend Build') {
            agent { docker { image 'node:20-alpine' } }
            steps { sh 'npm ci && npm run build' }
        }
        stage('Deploy') {
            agent { docker { image 'bitnami/kubectl:latest' } }
            steps { sh 'kubectl apply -f k8s/' }
        }
    }
}
```

### 11. Parallel Pipeline Stages

Jenkins supports running multiple stages in parallel. For ShopAxis, unit tests and frontend build can run simultaneously, cutting total pipeline time from 18 minutes to 10 minutes. Parallel stages declare a failure policy — if one branch fails, the parallel block fails immediately.

```
// Parallel stages — run simultaneously
stage('Build & Test') {
    parallel {
        stage('Backend Tests') {
            agent { docker { image 'maven:3.9.6-eclipse-temurin-17' } }
            steps {
                dir('backend') {
                    sh 'mvn test'
                }
            }
            post {
                always {
                    junit 'backend/target/surefire-reports/*.xml'
                }
            }
        }
        stage('Frontend Tests') {
            agent { docker { image 'node:20-alpine' } }
            steps {
                dir('frontend') {
                    sh 'npm ci'
                    sh 'npm run test -- --watchAll=false --passWithNoTests'
                    sh 'npm run build'
                }
            }
        }
        stage('Security Scan') {
            agent { docker { image 'aquasec/trivy:latest' } }
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL --exit-code 0 .'
            }
        }
    }
}
```

```
Parallel Pipeline Execution Timeline
========================================

  Without Parallel (Sequential):
  ─────────────────────────────────────────────
  Backend Tests    ████████████ 8 min
  Frontend Build             ████████ 6 min
  Security Scan                      ████ 4 min
                   ──────────────────────────── 18 min total

  With Parallel (Concurrent):
  ─────────────────────────────────────────────
  Backend Tests    ████████████ 8 min
  Frontend Build   ████████     6 min
  Security Scan    ████         4 min
                   ──────────── 8 min total (55% faster!)
```

### 12. Jenkins Shared Libraries

When you have multiple applications — ShopAxis, PaymentService, UserService — duplicating the Jenkinsfile in every repository is wasteful and inconsistent. Jenkins Shared Libraries let you define reusable pipeline functions once and import them into any Jenkinsfile. This enforces standardisation across all pipelines in the organisation.

```
// Shared Library structure (in separate Git repository: jenkins-shared-library)
jenkins-shared-library/
├── vars/
│   ├── buildJavaApp.groovy       // Reusable Maven build function
│   ├── dockerBuildPush.groovy    // Reusable Docker build/push
│   ├── deployToKubernetes.groovy // Reusable Helm deploy
│   ├── sonarAnalysis.groovy      // Reusable SonarQube step
│   └── notifySlack.groovy        // Reusable Slack notification
└── resources/
    └── sonar-quality-gate.json
```

```bash
// vars/buildJavaApp.groovy — Shared Library function
def call(Map config = [:]) {
    def mavenImage = config.mavenImage ?: 'maven:3.9.6-eclipse-temurin-17'
    def appDir = config.appDir ?: 'backend'

    docker.image(mavenImage).inside('-v /root/.m2:/root/.m2') {
        dir(appDir) {
            stage("${config.appName} — Compile") {
                sh 'mvn compile -DskipTests=true'
            }
            stage("${config.appName} — Test") {
                sh 'mvn test'
                junit 'target/surefire-reports/*.xml'
            }
            stage("${config.appName} — Package") {
                sh 'mvn package -DskipTests=true'
                archiveArtifacts artifacts: 'target/*.jar'
            }
        }
    }
}
```

```
// Jenkinsfile using Shared Library — clean, minimal
@Library('techaxis-jenkins-library@main') _

pipeline {
    agent any
    stages {
        stage('Build Backend') {
            steps {
                buildJavaApp(appName: 'ShopAxis', appDir: 'backend')
            }
        }
        stage('Code Quality') {
            steps {
                sonarAnalysis(projectKey: 'shopaxis-backend')
            }
        }
        stage('Docker') {
            steps {
                dockerBuildPush(
                    imageName: 'agastya/shopaxis',
                    imageTag: env.BUILD_NUMBER,
                    credentialsId: 'dockerhub-credentials'
                )
            }
        }
        stage('Deploy') {
            steps {
                deployToKubernetes(
                    namespace: 'shopaxis-prod',
                    helmChart: 'k8s/helm/shopaxis',
                    imageTag: env.BUILD_NUMBER
                )
            }
        }
    }
    post {
        always { notifySlack(channel: '#deployments') }
    }
}
```

### 13. Monitoring Jenkins with Prometheus and Grafana

The Prometheus Metrics plugin exposes Jenkins build metrics at `/prometheus`. Prometheus scrapes these metrics and Grafana visualises them — giving you dashboards for build success rates, queue depth, agent utilisation, and deployment frequency across all pipelines.

```
# prometheus.yml — scrape Jenkins metrics
scrape_configs:
  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['jenkins-server:8080']
    scrape_interval: 30s

  - job_name: 'shopaxis-backend'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['shopaxis-service:8080']
    scrape_interval: 15s
```

#### Key Jenkins Metrics to Monitor

Metric

Description

Alert Threshold

jenkins_builds_failed_total

Total failed builds

3 failures in 30 min

jenkins_builds_duration_milliseconds

Build duration histogram

p95 > 20 minutes

jenkins_executor_in_use_value

Active build executors

> 80% for 10 min

jenkins_queue_size_value

Jobs waiting in queue

> 5 queued jobs

jenkins_plugins_active

Active plugin count

Alert if drops

jenkins_builds_success_build_count

Successful build count

Track for DORA metrics

### 14. Branch-Based Pipeline Strategy

Different Git branches require different pipeline behaviour. Feature branches should run only tests; the main branch runs the full pipeline including production deployment; release branches require a manual approval gate. Jenkins Multibranch Pipeline handles this automatically.

```
// Branch-conditional Jenkinsfile — abbreviated key logic
pipeline {
    agent any
    stages {
        stage('Build & Test') {
            steps { dir('backend') { sh 'mvn test' } }
        }
        stage('Code Quality') {
            when { anyOf { branch 'main'; branch 'release/*'; branch 'develop' } }
            steps {
                withSonarQubeEnv('sonar-server') { sh 'mvn sonar:sonar' }
            }
        }
        stage('Docker Build & Push') {
            when { branch 'main' }
            steps {
                sh "docker build -t agastya/shopaxis:${BUILD_NUMBER} backend/"
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh 'echo $P | docker login -u $U --password-stdin'
                    sh "docker push agastya/shopaxis:${BUILD_NUMBER}"
                }
            }
        }
        stage('Deploy Staging') {
            when { branch 'main' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh "helm upgrade --install shopaxis k8s/helm/shopaxis/ --set image.tag=${BUILD_NUMBER} -n shopaxis-staging"
                }
            }
        }
        stage('Manual Approval — Production') {
            when { branch 'release/*' }
            steps {
                input message: 'Deploy to PRODUCTION?', ok: 'Approve', submitter: 'agastya,nandan'
            }
        }
        stage('Deploy Production') {
            when { branch 'release/*' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh "helm upgrade --install shopaxis k8s/helm/shopaxis/ --set image.tag=${BUILD_NUMBER} -n shopaxis-prod"
                }
            }
        }
    }
}
```

```
Branch Pipeline Strategy
===========================

  feature/login-page         ─► [Test Only] → PR to develop
  feature/checkout-redesign  ─► [Test Only] → PR to develop

  develop          ─► [Test] → [SonarQube] → [Docker Build] → [Deploy Staging]

  main             ─► [Test] → [SonarQube] → [OWASP] → [Docker Build]
                       → [Push DockerHub] → [Deploy Staging] → [Smoke Test]

  release/v2.1.0   ─► [Full Pipeline] → [Manual Approval] → [Deploy Production]
                       → [Smoke Test Prod] → [Tag Release] → [Notify]
```

### 15. Webhook Configuration and Multibranch Pipelines

A Multibranch Pipeline automatically discovers all branches in your repository that contain a Jenkinsfile and creates a separate pipeline job for each. GitHub webhooks trigger builds within seconds of a push rather than relying on polling.

#### GitHub Webhook Setup

1. Install GitHub Integration Plugin in Jenkins
Navigate: Manage Jenkins → Plugins → Available → search "GitHub Integration" → Install without restart.

2. Configure Webhook in GitHub Repository
Go to your GitHub repository → Settings → Webhooks → Add webhook. Set Payload URL to `http://jenkins-server:8080/github-webhook/`, Content type to application/json, and select "Just the push event." Save.

3. Create Multibranch Pipeline Job
Jenkins Dashboard → New Item → Multibranch Pipeline. Under Branch Sources add GitHub, select your credentials, enter repository URL. Under Build Configuration select "by Jenkinsfile." Save and click "Scan Repository Now."

4. Verify Branch Discovery
Jenkins scans the repository and creates child jobs for every branch containing a Jenkinsfile: main, develop, feature/*, release/*. Each branch gets an isolated pipeline with its own build history.

5. Test End-to-End Webhook
Push a trivial commit to any branch. Within 5 seconds, the GitHub webhook triggers and the Jenkins pipeline starts automatically. Check the GitHub webhook delivery log to confirm 200 OK response from Jenkins.

#### Multibranch Pipeline — Key Configuration Options

Setting

Value for ShopAxis

Purpose

Branch Discovery Strategy

All branches

Discover every branch with Jenkinsfile

Pull Request Discovery

Merge with target branch

Test PR merged state before merging

Orphaned Item Strategy

Discard after 7 days

Remove deleted branch jobs automatically

Scan Interval (fallback)

Every 5 minutes

Polling fallback if webhook fails

Number of Builds to Keep

10

Limit disk usage for build history

#### Jenkins Job DSL — Pipeline as Infrastructure Code

For teams with dozens of repositories, creating Jenkins jobs manually is unsustainable. The Job DSL plugin lets you generate Jenkins jobs programmatically using a Groovy script — pipeline infrastructure as code.

```
// jobs/shopaxis-multibranch.groovy — Job DSL to create the pipeline job
multibranchPipelineJob('shopaxis-cicd') {
    displayName('ShopAxis CI/CD Pipeline')
    description('End-to-end pipeline: Maven → SonarQube → Docker → K8s')

    branchSources {
        github {
            id('shopaxis-github')
            repoOwner('techaxis')
            repository('shopaxis')
            credentialsId('github-credentials')
            buildForkPRHead(false)
            buildForkPRMerge(false)
            buildOriginBranch(true)
            buildOriginPRHead(false)
            buildOriginPRMerge(true)
        }
    }

    orphanedItemStrategy {
        discardOldItems {
            numToKeep(10)
            daysToKeep(7)
        }
    }

    triggers {
        periodic(5)  // Scan every 5 minutes as webhook fallback
    }

    factory {
        workflowBranchProjectFactory {
            scriptPath('Jenkinsfile')  // Where to find the Jenkinsfile
        }
    }
}
```

#### Jenkins Configuration as Code (JCasC)

JCasC lets you define the entire Jenkins configuration — tools, credentials, plugins, security — in a YAML file. This makes Jenkins itself reproducible and version-controlled.

```
# jenkins.yaml — Jenkins Configuration as Code (JCasC)
jenkins:
  systemMessage: "EngiDock ShopAxis Jenkins — Managed by JCasC"
  numExecutors: 0
  mode: EXCLUSIVE
  scmCheckoutRetryCount: 2

  globalNodeProperties:
    - envVars:
        env:
          - key: NEXUS_URL
            value: "http://nexus-server:8081"
          - key: SONAR_URL
            value: "http://sonarqube-server:9000"
          - key: K8S_NAMESPACE
            value: "shopaxis-prod"

tool:
  maven:
    installations:
      - name: "maven3"
        home: "/usr/share/maven"
  jdk:
    installations:
      - name: "jdk17"
        home: "/usr/lib/jvm/java-17-openjdk-amd64"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              id: "github-credentials"
              username: "${GITHUB_USER}"
              password: "${GITHUB_TOKEN}"
              description: "GitHub PAT"
          - string:
              id: "sonar-token"
              secret: "${SONAR_TOKEN}"
              description: "SonarQube analysis token"

unclassified:
  sonarGlobalConfiguration:
    installations:
      - name: "sonar-server"
        serverUrl: "http://sonarqube-server:9000"
        credentialsId: "sonar-token"

  slackNotifier:
    teamDomain: "techaxis"
    tokenCredentialId: "slack-token"
    room: "#shopaxis-deployments"
```

### 16. Jenkins Pipeline Best Practices

##### Pipeline Design Best Practices — EngiDock Standard

- Always use declarative pipeline syntax — avoid scripted pipeline for new pipelines
- Commit Jenkinsfile to the repository root — pipeline is part of the codebase
- Never store secrets in Jenkinsfile or source code — use Jenkins Credentials Manager
- Use timeout() wrappers on every stage to prevent hung builds from blocking agents
- Always run cleanWs() in post { always { } } to free disk space between builds
- Use parallel stages for independent tasks — cut build time by 30-60%
- Archive build artifacts and test reports — use archiveArtifacts and junit plugins
- Set specific tool versions — never rely on whatever is installed on the agent
- Use Docker agents for isolation — eliminate "works on my machine" completely
- Implement Shared Libraries — DRY principles apply to pipelines too
- Use when conditions — don't run expensive stages on feature branches
- Always include post build notifications — success, failure, and unstable separately
- Pin plugin versions in production — uncontrolled plugin updates break pipelines
- Tag Docker images with BUILD_NUMBER — never overwrite the latest tag only
- Use rolling deployments with maxUnavailable: 0 — zero-downtime deployments

### 17. Knowledge Check — Jenkins Mastery Quiz

**Quiz: 1. A developer pushes to GitHub and the Jenkins pipeline starts automatically. The Maven compile stage fails. What happens to the Docker build stage?**

- A) Docker build runs with whatever was previously compiled
- B) Docker build is skipped automatically — pipeline aborts at failed stage
- C) Docker build runs but posts a warning
- D) Jenkins restarts the entire pipeline from the beginning

> **Answer/explanation:** **Correct: B** — Jenkins declarative pipeline fails fast. When a stage fails, all subsequent stages are skipped and the post { failure { } } block runs. This is by design — you should never push an artifact built from broken code. The Slack notification goes out immediately so the team knows.

**Quiz: 2. Your Jenkins pipeline has a SonarQube Quality Gate that requires 80% test coverage. The developer wrote new code with no tests — coverage drops to 62%. What is the correct pipeline behaviour?**

- A) Pipeline continues with a warning in the build summary
- B) SonarQube stage is marked unstable but pipeline continues to Docker build
- C) waitForQualityGate abortPipeline: true causes the pipeline to fail and stop
- D) Jenkins retries the SonarQube stage three times before failing

> **Answer/explanation:** **Correct: C** — The waitForQualityGate(abortPipeline: true) call blocks until SonarQube returns the gate result. If the gate fails (coverage below threshold), Jenkins marks the build as FAILED and stops. The Docker image is never built and never deployed. Code quality gates are hard gates, not suggestions.

**Quiz: 3. Which statement correctly describes Jenkins' relationship to Maven in the ShopAxis pipeline?**

- A) Jenkins compiles Java code internally; Maven manages dependencies only
- B) Jenkins calls the mvn package command; Maven actually compiles and packages the JAR
- C) Maven controls the pipeline; Jenkins only stores the output
- D) Jenkins and Maven both compile the code — Jenkins compiles first, Maven verifies

> **Answer/explanation:** **Correct: B** — Jenkins is the orchestrator, not the build tool. In the Build stage, Jenkins executes sh 'mvn package -DskipTests=true' — that is the full extent of Jenkins' role in compilation. Maven handles everything: downloading dependencies from Nexus, compiling Java, running tests, and packaging the JAR. Jenkins captures the exit code and logs the result.

**Quiz: 4. You want to run the backend unit tests and frontend build simultaneously to save time. Which Jenkinsfile construct achieves this?**

- A) concurrent { stages { } }
- B) parallel { stage('Backend') { } stage('Frontend') { } }
- C) async { stage() stage() }
- D) background { run('mvn test') run('npm test') }

> **Answer/explanation:** **Correct: B** — The parallel { } block inside a stage is Jenkins declarative pipeline's construct for concurrent execution. Each sub-stage inside parallel runs simultaneously on available agents. If one parallel branch fails, the whole parallel block fails (configurable with failFast: true/false). This is how you cut a 20-minute sequential pipeline to a 10-minute parallel one.

**Quiz: 5. A production deployment fails. The Kubernetes pods are in CrashLoopBackOff. What is the fastest recovery path using your Jenkins + Helm pipeline?**

- A) Manually edit the Kubernetes deployment YAML on the cluster
- B) Run helm rollback shopaxis 0 from the Jenkins agent or trigger the previous successful build in Jenkins
- C) Delete the namespace and redeploy from scratch
- D) SSH into the pods and restart the Java process

> **Answer/explanation:** **Correct: B** — Helm maintains release history. helm rollback shopaxis 0 rolls back to the immediately previous Helm release, which uses the previous Docker image stored in Nexus/DockerHub. Alternatively, re-run the last successful Jenkins build — it will push the same Docker image and Helm will update the Kubernetes deployment back to the working version. This is why immutable versioned artifacts stored in Nexus and Docker registries are critical.

**Quiz: 6. You have 10 microservices, each with their own Jenkinsfile. The Docker push step is duplicated across all 10 files. When Docker Hub changes their authentication method, you must update 10 Jenkinsfiles. The correct solution is:**

- A) Use a global environment variable in Jenkins configuration for all pipelines
- B) Create a Jenkins Shared Library with a dockerBuildPush.groovy function and import it
- C) Copy-paste the fix manually across all 10 Jenkinsfiles
- D) Create a Jenkins Folder and use the same Jenkinsfile for all 10 services

> **Answer/explanation:** **Correct: B** — Jenkins Shared Libraries are the solution to DRY (Don't Repeat Yourself) in CI/CD. You define the dockerBuildPush function once in a shared library Git repository. All 10 Jenkinsfiles import it with @Library('techaxis-jenkins-library'). When Docker Hub changes authentication, you update one file in the shared library and all 10 pipelines automatically use the fix at next build. This is enterprise-grade pipeline management.

**Quiz: 7. Trivy scans your Docker image and finds a HIGH severity CVE in the base image eclipse-temurin:17-jre-alpine. Your pipeline has --exit-code 0 in the Trivy command. What happens?**

- A) Pipeline fails immediately — HIGH CVE is a hard blocker
- B) Trivy exits with code 0 (success), report is saved, pipeline continues
- C) Jenkins quarantines the Docker image and sends it for manual review
- D) Docker Hub refuses to accept the image push

> **Answer/explanation:** **Correct: B** — --exit-code 0 means Trivy always exits successfully regardless of findings. The report is still generated and archived. This is a "report only" mode used when you have known vulnerabilities in base images that cannot be immediately patched. Changing to --exit-code 1 would fail the build on any HIGH/CRITICAL CVE, which is the strict enforcement mode used when you have control over all dependencies.

**Quiz: 8. Your ShopAxis pipeline takes 25 minutes. Management wants it under 12 minutes. The stages are: Checkout (1 min), Compile (2 min), Unit Tests (8 min), SonarQube (5 min), OWASP Scan (6 min), Docker Build (2 min), Deploy (1 min). Which restructuring gives the most improvement?**

- A) Use a faster server for the Jenkins controller
- B) Skip the SonarQube stage on weekdays
- C) Run Unit Tests, SonarQube, and OWASP Scan in parallel after Compile
- D) Combine Docker Build and Deploy into a single stage

> **Answer/explanation:** **Correct: C** — Unit Tests (8 min) + SonarQube (5 min) + OWASP Scan (6 min) are completely independent — none depends on the others' output. Running them in parallel, the slowest one (8 min) determines the total. New timeline: Checkout (1) + Compile (2) + Parallel group (8) + Docker (2) + Deploy (1) = 14 minutes. Further optimisation with Docker layer caching on the build stage can bring it under 12. Parallelisation is always the highest-impact optimisation.

##### Common Questions — Jenkins Project Mastery

**Q: Q: Can Jenkins deploy to production without any human approval?**

A: Yes — this is called Continuous Deployment (CD). The input() step in Jenkinsfile is optional. For fully automated production deployments, remove the input step. Whether to require human approval depends on your organisation's risk tolerance and deployment maturity. Netflix and Amazon deploy to production fully automatically hundreds of times per day. Many organisations require manual approval for production but automate staging completely. Jenkins supports both models through the presence or absence of the input() step.

**Q: Q: What is the difference between a Freestyle Job and a Pipeline in Jenkins?**

A: Freestyle Jobs are configured through the Jenkins UI — click-through forms that define build steps. They are not version-controlled, not easily auditable, and difficult to share. Pipelines (using Jenkinsfile) define the entire CI/CD workflow as code committed to your repository. Pipelines support parallel stages, conditional logic, loops, shared libraries, and advanced error handling. In 2026, no new CI/CD project should use Freestyle Jobs — they are a legacy concept. All modern Jenkins implementations use Pipeline as Code via Jenkinsfile.

**Q: Q: Why do we need Nexus if Docker Hub already stores our images?**

A: Nexus serves a different purpose than Docker Hub. Nexus stores Maven artifacts — the JAR files that are intermediate build outputs. These are stored with precise versioning (tied to BUILD_NUMBER), enabling artifact traceability: for any deployed Docker image, you can identify the exact JAR it was built from, which Git commit produced it, and which test run passed it. Nexus also acts as a Maven proxy — caching all Maven Central dependencies locally, which speeds up builds and protects against Maven Central outages. Docker Hub stores the final container images. Both are needed in a mature pipeline.

**Q: Q: Should I use Jenkins or switch to GitHub Actions / GitLab CI?**

A: Jenkins, GitHub Actions, and GitLab CI are all CI/CD orchestrators — they solve the same fundamental problem. Jenkins is the right choice when you need maximum flexibility, self-hosted infrastructure, complex multi-tool integration, and the largest plugin ecosystem. GitHub Actions is ideal if your code is already on GitHub and you want zero infrastructure management. GitLab CI is excellent if you use GitLab for source control and want native integration. The core concepts — triggers, stages, agents, artifacts, notifications — are identical across all three. Master Jenkins and you can migrate to any other CI/CD system in days.

**Q: Q: How does Jenkins know when to trigger a new build?**

A: Jenkins supports three trigger mechanisms. Webhook triggers — GitHub/GitLab sends an HTTP POST to Jenkins when code is pushed, triggering a build within seconds. SCM Polling — Jenkins checks the repository on a schedule (every 5 minutes) and triggers if changes are detected. This is less efficient but works when webhooks cannot reach Jenkins due to firewalls. Scheduled triggers — like cron, for nightly full builds. Most production pipelines use webhooks for fast feedback. The webhook URL is typically http://jenkins-server:8080/github-webhook/ and is configured in the GitHub repository settings under Webhooks.

##### Hands-On Exercise — Build the ShopAxis Pipeline

1. Install Jenkins on Ubuntu 22.04 using the instructions in Section 3. Verify it is accessible at port 8080.
2. Install all required plugins listed in the Plugin table. Configure Global Tool settings for Maven 3.9.6 and JDK 17.
3. Set up SonarQube (Docker: docker run -d -p 9000:9000 sonarqube:community). Create a project and generate a token. Add the token to Jenkins Credentials.
4. Set up Nexus (Docker: docker run -d -p 8081:8081 sonatype/nexus3). Create the shopaxis-releases hosted Maven repository. Add Nexus credentials to Jenkins.
5. Fork or create a Spring Boot application. Add a Jenkinsfile with at minimum: Checkout, Compile, Test, SonarQube, Docker Build, Docker Push stages.
6. Create a Multibranch Pipeline job in Jenkins. Point it to your repository. Watch it detect the Jenkinsfile and trigger the first build.
7. Verify every stage passes. Check: SonarQube shows the project, Nexus shows the JAR artifact, Docker Hub shows the pushed image, JUnit results appear in Jenkins.
8. Break the pipeline deliberately: introduce a Java compilation error and push. Verify Jenkins detects the failure, stops at the Compile stage, and sends a notification.
9. Add a parallel block — run unit tests and a frontend npm build simultaneously. Measure the pipeline time before and after and record the improvement.
10. Create a Shared Library with at minimum a dockerBuildPush() function. Refactor your Jenkinsfile to use the shared library. Confirm the pipeline still passes.

> **Jenkins Project Mastery — Core Takeaways**

> - Jenkins is a CI/CD orchestrator — it calls tools, it does not replace them. Maven compiles, Docker containerises, Kubernetes deploys, SonarQube analyses — Jenkins coordinates all of them.
> - The Jenkinsfile is your pipeline as code — version-controlled, peer-reviewed, auditable, and self-documenting. It is committed to the repository alongside the application code.
> - Every secret — tokens, passwords, kubeconfig — belongs in Jenkins Credentials Manager, never in the Jenkinsfile or any version-controlled file.
> - Quality gates are hard gates — SonarQube quality gate failure, OWASP CVSS-7+ findings, and test failures must stop the pipeline. Warnings without consequences are useless.
> - Parallel stages cut pipeline time by 30-60% — independent steps like tests, security scans, and frontend builds should always run concurrently.
> - Docker agents provide complete build isolation — no dependency pollution between builds, no "works on my machine" issues, and perfect reproducibility.
> - Shared Libraries enforce organisational standards — one update in the library fixes all pipelines that import it. DRY principles apply to CI/CD code.
> - Nexus provides artifact traceability — every deployed image can be traced to a specific JAR, a specific Git commit, and a specific test run. This is critical for incident investigation.
> - Helm enables zero-downtime deployments — RollingUpdate with maxUnavailable: 0 ensures Kubernetes always has running pods during a deployment. Rollback is one command.
> - Prometheus + Grafana give pipeline observability — DORA metrics (deployment frequency, lead time, MTTR, change failure rate) become measurable and improvable.

### Jenkins Project Mastery Complete

You have built a production-grade Jenkins CI/CD pipeline for a real full-stack application — Java + React + PostgreSQL + Docker + Kubernetes — with quality gates, security scanning, artifact management, parallel execution, and monitoring.

> **Agastya**
> 
> "You have gone from zero to a full production CI/CD system. Every tool in our stack — Maven, SonarQube, OWASP, Trivy, Nexus, Docker, Helm, Kubernetes, Prometheus — connects through Jenkins. When a developer pushes code, the entire pipeline runs automatically, validates quality, ensures security, and deploys with zero human intervention. That is what DevOps looks like at scale. Well done."

> **Nandan**
> 
> "Remember: Jenkins is the conductor. Every brilliant musician in your DevOps orchestra — Maven, Docker, Kubernetes, SonarQube — plays their part perfectly. Jenkins makes sure they all start at the right time, in the right order, and that the whole performance succeeds. That mental model will serve you throughout your entire DevOps career."

> **Next: Advanced Jenkins — Multi-Cluster, ArgoCD GitOps & Jenkins X**

> - ArgoCD integration — GitOps deployment model replacing kubectl in Jenkins
> - Jenkins X — cloud-native CI/CD on Kubernetes with automated environments
> - Multi-cluster deployments — blue/green across AWS and GCP clusters
> - DORA metrics dashboard — measuring deployment frequency and MTTR with Grafana
> - Tekton pipelines — Kubernetes-native alternative pipeline engine
