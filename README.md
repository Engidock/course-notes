# EngiDock Course Notes & Project Labs

Condensed notes and **hands-on project labs** that accompany the courses on [engidock.com](https://www.engidock.com). Each course folder has a `project-lab.md` with an objective, prerequisites, step-by-step tasks, and a concrete deliverable — meant to be done, not just read.

These are **not** replacements for the courses — they're the practical, hands-on complement to the video/lesson content.

## Course index

### Artificial Intelligence

| Course | Description | Lab |
|---|---|---|
| Generative AI | Build a small app that uses an LLM API to summarize or generate content. | [`generative-ai/project-lab.md`](./generative-ai/project-lab.md) |
| Prompt Engineering | Iterate on prompts for a specific task and document what actually improved output. | [`prompt-engineering/project-lab.md`](./prompt-engineering/project-lab.md) |

### Cloud Computing

| Course | Description | Lab |
|---|---|---|
| AWS Networking | Design a custom VPC with public/private subnets and secure routing. | [`aws-networking/project-lab.md`](./aws-networking/project-lab.md) |
| Serverless & Event-Driven Architecture | Build an S3-to-Lambda-to-DynamoDB event pipeline. | [`serverless-event-driven/project-lab.md`](./serverless-event-driven/project-lab.md) |
| Cloud Security | Harden an AWS account against the most common misconfigurations. | [`cloud-security/project-lab.md`](./cloud-security/project-lab.md) |
| AWS | Deploy a 3-tier web application on AWS. | [`aws/project-lab.md`](./aws/project-lab.md) |
| Azure | Deploy a secured web app on Azure App Service with Azure SQL. | [`azure/project-lab.md`](./azure/project-lab.md) |

### Container Orchestration

| Course | Description | Lab |
|---|---|---|
| Kubernetes | Deploy a multi-tier application on Kubernetes with Ingress. | [`kubernetes/project-lab.md`](./kubernetes/project-lab.md) |

### Containerization

| Course | Description | Lab |
|---|---|---|
| Podman | Run a rootless multi-container pod with Podman. | [`podman/project-lab.md`](./podman/project-lab.md) |
| Docker | Containerize a full app with a multi-stage Dockerfile and docker-compose. | [`docker/project-lab.md`](./docker/project-lab.md) |

### Data Engineering

| Course | Description | Lab |
|---|---|---|
| Apache Hadoop | Run a MapReduce word-count job on a local Hadoop cluster. | [`apache-hadoop/project-lab.md`](./apache-hadoop/project-lab.md) |
| Big Data | Design and run a batch pipeline over a large CSV dataset. | [`big-data/project-lab.md`](./big-data/project-lab.md) |
| Apache Spark | Write a PySpark job that cleans, aggregates, and writes data to Parquet. | [`apache-spark/project-lab.md`](./apache-spark/project-lab.md) |
| Databricks | Build an ingest-transform-visualize notebook pipeline in Databricks. | [`databricks/project-lab.md`](./databricks/project-lab.md) |
| Microsoft Excel | Build a sales dashboard using pivot tables and lookup formulas. | [`microsoft-excel/project-lab.md`](./microsoft-excel/project-lab.md) |
| SQL | Answer real business questions against a sample schema using joins and window functions. | [`sql/project-lab.md`](./sql/project-lab.md) |

### Design & Creative

| Course | Description | Lab |
|---|---|---|
| Interior Designing | Design a full floor plan and mood board for a small studio apartment. | [`interior-designing/project-lab.md`](./interior-designing/project-lab.md) |

### DevOps

| Course | Description | Lab |
|---|---|---|
| Spinnaker | Build a multi-stage deployment pipeline in Spinnaker. | [`spinnaker/project-lab.md`](./spinnaker/project-lab.md) |
| Microservices & Kubernetes | Split a monolith into microservices and deploy them on Kubernetes. | [`microservices-kubernetes/project-lab.md`](./microservices-kubernetes/project-lab.md) |
| DevSecOps | Integrate security scanning into a CI pipeline. | [`devsecops/project-lab.md`](./devsecops/project-lab.md) |
| Apache Maven | Build and package a Java project with Maven, including a custom profile. | [`apache-maven/project-lab.md`](./apache-maven/project-lab.md) |
| Nexus Repository | Stand up Nexus as a private artifact repository. | [`nexus-repository/project-lab.md`](./nexus-repository/project-lab.md) |
| SRE Fundamentals | Define SLIs/SLOs for a sample service and alert on error budget burn. | [`sre-fundamentals/project-lab.md`](./sre-fundamentals/project-lab.md) |
| Github Actions | Build a full CI/CD workflow: lint, test, build, and deploy. | [`github-actions/project-lab.md`](./github-actions/project-lab.md) |
| Azure DevOps | Build a YAML pipeline for build and release to Azure App Service. | [`azure-devops/project-lab.md`](./azure-devops/project-lab.md) |
| Terraform | Provision cloud infrastructure with reusable Terraform modules. | [`terraform/project-lab.md`](./terraform/project-lab.md) |
| Ansible | Automate provisioning and configuration of multiple servers with a playbook. | [`ansible/project-lab.md`](./ansible/project-lab.md) |
| GIT | Practice a real branching workflow: feature branches, rebase, and conflict resolution. | [`git/project-lab.md`](./git/project-lab.md) |
| Jenkins | Build a declarative Jenkins pipeline with a webhook trigger. | [`jenkins/project-lab.md`](./jenkins/project-lab.md) |

### GitOps

| Course | Description | Lab |
|---|---|---|
| ArgoCD | Deploy an app to Kubernetes via ArgoCD, synced from Git, with a rollback. | [`argocd/project-lab.md`](./argocd/project-lab.md) |

### Infrastructure as Code

| Course | Description | Lab |
|---|---|---|
| Cloud Formation | Provision a VPC, EC2 instance, and S3 bucket with a CloudFormation template. | [`cloudformation/project-lab.md`](./cloudformation/project-lab.md) |
| Pulumi | Provision the same style of infrastructure as Terraform/CloudFormation, using real code. | [`pulumi/project-lab.md`](./pulumi/project-lab.md) |

### Monitoring

| Course | Description | Lab |
|---|---|---|
| Grafana | Build a monitoring dashboard with alerts on a Prometheus data source. | [`grafana/project-lab.md`](./grafana/project-lab.md) |
| Prometheus | Instrument an app, scrape it with Prometheus, and write alerting rules. | [`prometheus/project-lab.md`](./prometheus/project-lab.md) |

### Package Management

| Course | Description | Lab |
|---|---|---|
| Helm | Package a Kubernetes app as a configurable Helm chart. | [`helm/project-lab.md`](./helm/project-lab.md) |

### Programming

| Course | Description | Lab |
|---|---|---|
| Java | Build a CLI inventory management app using core OOP principles. | [`java/project-lab.md`](./java/project-lab.md) |

### Scripting

| Course | Description | Lab |
|---|---|---|
| Groovy Scripting | Write a Groovy shared-library function for a Jenkins pipeline. | [`groovy-scripting/project-lab.md`](./groovy-scripting/project-lab.md) |
| YAML Scripting | Author a complex, valid multi-document YAML file with anchors and references. | [`yaml-scripting/project-lab.md`](./yaml-scripting/project-lab.md) |
| Python Scripting | Automate a repetitive task with a robust, error-handled Python script. | [`python-scripting/project-lab.md`](./python-scripting/project-lab.md) |
| Shell Scripting | Automate a sysadmin task with a properly argument-parsed bash script. | [`shell-scripting/project-lab.md`](./shell-scripting/project-lab.md) |

### Security

| Course | Description | Lab |
|---|---|---|
| Cybersecurity | Perform a basic OWASP Top 10 assessment of a sample web app. | [`cybersecurity/project-lab.md`](./cybersecurity/project-lab.md) |
| SonarQube | Set up SonarQube, scan a codebase, and get it to pass the quality gate. | [`sonarqube/project-lab.md`](./sonarqube/project-lab.md) |

### System Administration

| Course | Description | Lab |
|---|---|---|
| Linux | Practice user management, permissions, systemd, and log troubleshooting. | [`linux/project-lab.md`](./linux/project-lab.md) |

### Web Development

| Course | Description | Lab |
|---|---|---|
| Django | Build a CRUD app with models, views, templates, and authentication. | [`django/project-lab.md`](./django/project-lab.md) |
| JavaScript | Build an interactive to-do app with vanilla JavaScript. | [`javascript/project-lab.md`](./javascript/project-lab.md) |
| CSS | Build a responsive landing page with Flexbox and Grid. | [`css/project-lab.md`](./css/project-lab.md) |
| HTML | Build a semantic, accessible multi-section page from scratch. | [`html/project-lab.md`](./html/project-lab.md) |

## Contributing

Completed a lab and want to add your notes alongside it? Add a `notes.md` in the same course folder and open a pull request. Spotted an outdated step? PRs welcome there too.

## License

[MIT](./LICENSE)
