# Amazon EKS (Elastic Kubernetes Service) Mastery

> **👋 Hey Fresher — Read This First!**

> - **EKS = Amazon's managed Kubernetes service** — AWS runs the control plane (API server, etcd, scheduler) across 3 AZs automatically; you manage the worker nodes
> - Without EKS: you install Kubernetes manually on EC2 instances — upgrading, HA setup, etcd backups, load balancer wiring — weeks of work per cluster
> - With EKS: cluster is ready in ~15 minutes; AWS guarantees 99.95% SLA on the control plane; you never SSH into a master node
> - EKS integrates natively with IAM, ECR, ALB, CloudWatch, Secrets Manager, and every AWS service — no third-party glue needed
> - **Company:** ShipFast — a Bengaluru-based e-commerce startup processing 50,000 orders/day, with flash sales that spike traffic 20x in seconds. They run on raw EC2 with manual deployments. Deployments take 2 hours, flash sale crashes happen monthly. You just joined as Junior DevOps Engineer. Lead: **Deepa**. Mission: migrate ShipFast to EKS and make the platform uncrashable

#### What You Will Learn and Build

- Provision EKS cluster with eksctl — VPC, subnets, IAM roles, managed node groups, all in one command
- Connect kubectl to EKS using aws-auth and IAM identity mapping
- Push images to ECR and deploy ShipFast's order-api and storefront to EKS
- Configure AWS Load Balancer Controller for ALB Ingress with TLS from ACM
- Set up IRSA (IAM Roles for Service Accounts) — pod-level AWS permissions without static credentials
- Integrate AWS Secrets Manager via CSI Driver — no secrets in YAML or environment variables
- Configure HPA, Cluster Autoscaler, and Karpenter for instant flash-sale scaling
- Monitor with CloudWatch Container Insights and set up alarms
- Implement RBAC with IAM, Network Policies with Calico, and upgrade EKS zero-downtime

EKS Cluster, Managed Node Groups, ECR Integration, ALB Ingress, IRSA, Karpenter, CloudWatch, IAM RBAC, Secrets Manager, Fargate

#### ShipFast EKS Architecture — Full Picture

```
ShipFast E-Commerce — EKS Production Architecture on AWS
══════════════════════════════════════════════════════════════════════════

  AWS Account: shipfast-prod  |  Region: ap-south-1 (Mumbai)

  ┌─────────────────────────────────────────────────────────────────────┐
  │  VPC: vpc-shipfast  10.0.0.0/8                                      │
  │                                                                     │
  │  Public Subnets (ALB lives here)         Private Subnets (pods)    │
  │  10.1.1.0/24  ap-south-1a               10.2.1.0/24 ap-south-1a   │
  │  10.1.2.0/24  ap-south-1b               10.2.2.0/24 ap-south-1b   │
  │  10.1.3.0/24  ap-south-1c               10.2.3.0/24 ap-south-1c   │
  │                                                                     │
  │  ┌──────────────────────────────────────────────────────────────┐   │
  │  │  EKS Cluster: eks-shipfast-prod  (K8s 1.29)                 │   │
  │  │                                                              │   │
  │  │  Control Plane — AWS Managed (FREE, 3-AZ HA)                │   │
  │  │  [API Server]  [Scheduler]  [Controller Mgr]  [etcd×3]      │   │
  │  │                                                              │   │
  │  │  Managed Node Group: system-ng (3×m5.large)                 │   │
  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │   │
  │  │  │ 1a node  │  │ 1b node  │  │ 1c node  │ ← CoreDNS, CNI   │   │
  │  │  └──────────┘  └──────────┘  └──────────┘                   │   │
  │  │                                                              │   │
  │  │  Managed Node Group: app-ng  autoscale 3–20 (m5.xlarge)     │   │
  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │   │
  │  │  │order-api │  │storefront│  │  worker  │ ← ShipFast Pods   │   │
  │  │  │  pods    │  │  pods    │  │  pods    │                   │   │
  │  │  └──────────┘  └──────────┘  └──────────┘                   │   │
  │  └──────────────────────────────────────────────────────────────┘   │
  │                                                                     │
  │  ┌───────────┐  ┌────────────┐  ┌──────────┐  ┌────────────────┐   │
  │  │  AWS ECR  │  │ Secrets Mgr│  │   ACM    │  │   CloudWatch   │   │
  │  │  (images) │  │ (secrets)  │  │  (certs) │  │ (logs/metrics) │   │
  │  └───────────┘  └────────────┘  └──────────┘  └────────────────┘   │
  │                                                                     │
  │  Internet → ALB (public subnet) → order-api / storefront pods       │
  └─────────────────────────────────────────────────────────────────────┘
```

### 1. Phase 1 — Prerequisites and Cluster Provisioning with eksctl

**Business Problem:** ShipFast needs a production-grade EKS cluster across 3 AZs with managed node groups, proper IAM roles, and VPC networking — set up in under 30 minutes.

**Scene 1 — ShipFast Engineering Standup | "Flash Sale Took Down Orders Again"**

> **Deepa** _Lead DevOps — ShipFast_
> 
> Diwali flash sale — traffic spiked 18x in 90 seconds. Our EC2-based order API couldn't handle it. Orders dropped for 11 minutes. Revenue loss: ₹47 lakhs. With EKS and Karpenter, when traffic spikes, new nodes provision in 45 seconds, pods schedule immediately, and traffic distributes automatically. We will never miss a flash sale window again because of infrastructure. Arjun, pair with the new engineer on provisioning today.

> **Arjun** _Senior DevOps — ShipFast_
> 
> And no more 2-hour deployment windows. With EKS rolling updates: deploy order-api v2 — 3 old pods, Kubernetes brings up 1 new pod, checks health, shifts traffic, removes 1 old pod. Repeat. Zero downtime. Zero manual steps. Total time: 4 minutes. We can deploy 10 times a day if we need to.

#### 1.1 Install Required Tools

```bash
# 1. AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install
aws configure   # enter Access Key, Secret, region, output
# 2. eksctl — EKS cluster management tool
curl --silent --location \
  "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /usr/local/bin

# 3. kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**📖 The EKS Toolchain**

- **aws CLI** — interacts with all AWS services; authentication via IAM access keys or instance profile
- **eksctl** — the official CLI for EKS; creates clusters, node groups, OIDC providers, IAM mappings — all in one command; abstracts CloudFormation stacks underneath
- **kubectl** — Kubernetes API client; works identically on EKS, AKS, GKE — the commands in this guide work on any Kubernetes cluster
- Verify: `aws --version`, `eksctl version`, `kubectl version --client`

#### 1.2 Create EKS Cluster with eksctl

```yaml
# cluster-config.yaml — full cluster definition for ShipFast
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: eks-shipfast-prod
region: ap-south-1
version: "1.29"
vpc:
  cidr: 10.0.0.0/8
clusterEndpoints:
    publicAccess: true
privateAccess: true
managedNodeGroups:
  - name: system-ng
instanceType: m5.large
minSize: 3
maxSize: 3
desiredCapacity: 3
availabilityZones: [ap-south-1a, ap-south-1b, ap-south-1c]
    taints:
      - key: CriticalAddonsOnly
value: "true"
effect: NoSchedule

  - name: app-ng
instanceType: m5.xlarge
minSize: 3
maxSize: 20
desiredCapacity: 3
availabilityZones: [ap-south-1a, ap-south-1b, ap-south-1c]
    labels: { pool: app, env: prod }
    iam:
      withAddonPolicies:
        autoScaler: true
albIngress: true
cloudWatch: true
addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
iam:
  withOIDC: true # Required for IRSA — pod-level IAM
```

> **eksctl.io/v1alpha5** — eksctl's ClusterConfig spec; single YAML file replaces dozens of manual AWS console clicks and CloudFormation templates
**clusterEndpoints: privateAccess: true** — kubectl can reach API server from within the VPC; combined with public: true for developer access during setup; lock down to private-only in hardened environments
**managedNodeGroups** — AWS manages the Auto Scaling Group, node launch template, OS patching, and node lifecycle; you only declare the desired state
**withAddonPolicies: autoScaler, albIngress, cloudWatch** — eksctl attaches the correct IAM policies to the node group IAM role automatically; no manual policy attachment
**withOIDC: true** — creates the OIDC identity provider; required for IRSA (pod-level IAM roles) — the most important security feature in EKS
Apply: `eksctl create cluster -f cluster-config.yaml` — takes ~15 minutes

#### 1.3 Connect kubectl

```bash
# Update kubeconfig for the new EKS cluster
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name eks-shipfast-prod

# Verify nodes are ready
kubectl get nodes -o wide
```

**📖 aws eks update-kubeconfig**

- Writes EKS cluster credentials into `~/.kube/config` using your IAM identity
- Authentication is done via `aws eks get-token` — a short-lived token signed by IAM; no static passwords
- Run this on every developer machine and CI/CD runner that needs cluster access
- Switch between clusters: `kubectl config use-context arn:aws:eks:ap-south-1:123456789:cluster/eks-shipfast-prod`

```
NAME                                      STATUS   ROLES    AGE   VERSION   ZONE
ip-10-2-1-45.ap-south-1.compute.internal  Ready    <none>   5m    v1.29.2   ap-south-1a
ip-10-2-2-78.ap-south-1.compute.internal  Ready    <none>   5m    v1.29.2   ap-south-1b
ip-10-2-3-12.ap-south-1.compute.internal  Ready    <none>   5m    v1.29.2   ap-south-1c
```

### 2. Phase 2 — ECR (Elastic Container Registry) Integration

**Business Problem:** ShipFast's Docker images need a private AWS registry — not Docker Hub. ECR integrates with EKS via IAM — no Docker credentials in YAML, no imagePullSecrets.

**Scene 2 — ShipFast On-Call Slack | "Image Pull Failed — Docker Hub Rate Limit"**

> **Deepa** _Lead DevOps — ShipFast_
> 
> Production deploy blocked at 3 AM — Docker Hub rate-limited our image pulls. 6-hour window wasted. ECR is inside AWS — pulls never hit the public internet, no rate limits, private by default. And because EKS node IAM roles already have ECR pull permissions, there are zero credentials to manage. The kubelet on every node authenticates to ECR automatically using the node's IAM role. We are moving all images to ECR this week.

#### 2.1 Create ECR Repositories

```
# Create ECR repos for ShipFast services
aws ecr create-repository \
  --repository-name shipfast/order-api \
  --region ap-south-1 \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

aws ecr create-repository \
  --repository-name shipfast/storefront \
  --region ap-south-1 \
  --image-scanning-configuration scanOnPush=true
```

**📖 ECR Repository Options**

- **scanOnPush=true** — ECR automatically scans every pushed image for CVEs using AWS Inspector; critical for compliance
- **encryptionType=AES256** — images encrypted at rest using AWS-managed keys; upgrade to KMS for customer-managed keys
- ECR URI format: `123456789012.dkr.ecr.ap-south-1.amazonaws.com/shipfast/order-api:tag`
- Set lifecycle policy to auto-delete images older than 30 days — prevents unlimited storage cost

#### 2.2 Build and Push to ECR

```bash
# Authenticate Docker to ECR (token valid 12 hours)
aws ecr get-login-password --region ap-south-1 \
  | docker login --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com

# Build, tag, and push
docker build -t shipfast/order-api:v2.1.0 .
docker tag shipfast/order-api:v2.1.0 \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/shipfast/order-api:v2.1.0
docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/shipfast/order-api:v2.1.0
```

**📖 ECR Authentication**

- **get-login-password** — generates a 12-hour token from AWS STS; pipe directly to docker login
- In CI/CD: use IAM Role attached to the build agent — no access keys needed; role must have `ecr:GetAuthorizationToken` and `ecr:BatchGetImage`
- EKS worker nodes pull images automatically — node IAM role has `AmazonEC2ContainerRegistryReadOnly` policy attached by eksctl
- No `imagePullSecrets` in pod spec — IAM handles authentication transparently

### 3. Phase 3 — Deploying ShipFast Applications

**Business Problem:** Deploy order-api (5 replicas) and storefront (3 replicas) to EKS with resource limits, health probes, PodDisruptionBudgets, and proper namespace isolation.

#### 3.1 Namespace and Resource Quota

```yaml
# Create namespace with resource limits
kubectl create namespace shipfast-prod
kubectl label namespace shipfast-prod \
  env=prod team=platform

# quota.yaml — cap total namespace resources
apiVersion: v1
kind: ResourceQuota
metadata:
  name: shipfast-quota
namespace: shipfast-prod
spec:
  hard:
    requests.cpu: "40"
requests.memory: 80Gi
limits.cpu: "80"
limits.memory: 160Gi
pods: "100"
```

**📖 Namespace Governance**

- All ShipFast app pods run in `shipfast-prod` — separate from `kube-system` and monitoring namespaces
- ResourceQuota prevents a single team or service from consuming all cluster capacity
- Kubernetes rejects pod creation if it would exceed the quota — deploy fails with a clear error rather than silently degrading other services
- Check usage: `kubectl describe quota shipfast-quota -n shipfast-prod`

#### 3.2 order-api Deployment

```yaml
# order-api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
namespace: shipfast-prod
spec:
  replicas: 5
selector:
    matchLabels: { app: order-api }
  strategy:
    type: RollingUpdate
rollingUpdate: { maxSurge: 2, maxUnavailable: 0 }
  template:
    metadata:
      labels: { app: order-api }
    spec:
      nodeSelector: { pool: app }
      topologySpreadConstraints:
        - maxSkew: 1
topologyKey: topology.kubernetes.io/zone
whenUnsatisfiable: DoNotSchedule
labelSelector:
            matchLabels: { app: order-api }
      containers:
        - name: order-api
image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/shipfast/order-api:v2.1.0
ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: "500m", memory: "512Mi" }
            limits:   { cpu: "2",     memory: "2Gi" }
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 10
livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 30
failureThreshold: 3
```

> **topologySpreadConstraints** — spreads pods evenly across AZs; maxSkew: 1 means no AZ can have more than 1 extra pod vs another; ensures 5 pods are never all on one AZ
**maxSurge: 2, maxUnavailable: 0** — during rolling update, add 2 new pods before removing any old ones; 5 pods stay available throughout the entire deploy
**nodeSelector: pool=app** — pods land only on the app-ng node group, not the system node group
**ECR image URI** — includes AWS account ID, region, repo name, and exact tag; no :latest in production

#### 3.3 PodDisruptionBudget

```yaml
# pdb-order-api.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-order-api
namespace: shipfast-prod
spec:
  minAvailable: 4
selector:
    matchLabels: { app: order-api }
```

**📖 PDB — Non-Negotiable in Production**

- **minAvailable: 4** — at least 4 of 5 order-api pods must be running at all times during voluntary disruptions
- Protects against: node drains during upgrades, Cluster Autoscaler scale-down, and manual node maintenance
- Without PDB: EKS upgrade can evict all 5 pods from a node simultaneously → 100% order-api outage
- PDB does not protect against involuntary disruption (EC2 instance failure) — that is what multiple replicas + AZ spread handles

### 4. Phase 4 — AWS Load Balancer Controller and ALB Ingress

**Business Problem:** ShipFast needs HTTPS on orders.shipfast.in and www.shipfast.in, path-based routing, WAF protection, and SSL certificates from AWS Certificate Manager — all managed by AWS Application Load Balancer.

#### 4.1 Install AWS Load Balancer Controller

```bash
# Create IAM policy for the controller
curl -o alb-iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://alb-iam-policy.json

# Create IRSA for the controller
eksctl create iamserviceaccount \
  --cluster eks-shipfast-prod \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::123456789012:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eks-shipfast-prod \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

**📖 ALB Controller vs Classic ELB**

- **AWS LB Controller** watches Kubernetes Ingress objects → creates/updates ALBs automatically in AWS
- Older approach: Kubernetes creates a classic ELB per Service (type: LoadBalancer) — expensive; one ELB per service
- ALB Controller: one ALB for all services with path-based routing → dramatically cheaper
- **IRSA** gives the controller IAM permissions to create ALBs without storing AWS credentials in the cluster

#### 4.2 ALB Ingress with TLS from ACM

```yaml
# shipfast-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shipfast-ingress
namespace: shipfast-prod
annotations:
    kubernetes.io/ingress.class: alb
alb.ingress.kubernetes.io/scheme: internet-facing
alb.ingress.kubernetes.io/target-type: ip
alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443},{"HTTP":80}]'
alb.ingress.kubernetes.io/ssl-redirect: "443"
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:123456789012:certificate/abc-def
alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:ap-south-1:123456789012:regional/webacl/shipfast-waf
spec:
  rules:
    - host: orders.shipfast.in
http:
        paths:
          - path: /api
pathType: Prefix
backend:
              service: { name: order-api-svc, port: { number: 8080 } }
          - path: /
pathType: Prefix
backend:
              service: { name: storefront-svc, port: { number: 3000 } }
```

**📖 Key ALB Annotations**

- **scheme: internet-facing** — ALB has a public IP; use `internal` for private APIs
- **target-type: ip** — routes directly to pod IP (VPC CNI gives pods real IPs); faster than `instance` mode which routes through NodePort
- **certificate-arn** — ACM certificate; TLS terminates at ALB; pods receive plain HTTP internally
- **wafv2-acl-arn** — attaches AWS WAF to the ALB; blocks SQL injection, XSS, bad bots before traffic reaches pods
- **ssl-redirect: 443** — ALB redirects all HTTP to HTTPS automatically

### 5. Phase 5 — IRSA (IAM Roles for Service Accounts)

**Business Problem:** order-api needs to read from S3 (order attachments), write to SQS (order events), and query DynamoDB (product catalog). Old approach: embed AWS access keys in environment variables — keys rotate, developers leave, keys leak. IRSA: pod gets temporary IAM credentials automatically via its Kubernetes service account.

**Scene 5 — ShipFast Security Alert | "AWS Keys Found in GitHub"**

> **Deepa** _Lead DevOps — ShipFast_
> 
> A developer committed AWS_ACCESS_KEY_ID to GitHub. The keys had S3 and DynamoDB permissions. Within 6 minutes, GitHub's secret scanner flagged it — but those 6 minutes are enough for an attacker to exfiltrate all order data. With IRSA, there are no access keys to commit. The order-api pod gets short-lived credentials through the pod's service account. Credentials last 15 minutes. Automatically refreshed. Cannot be committed to Git because they don't exist as static values anywhere.

#### 5.1 Create IAM Role for order-api

```
# Create IRSA — IAM role bound to a K8s service account
eksctl create iamserviceaccount \
  --cluster eks-shipfast-prod \
  --name order-api-sa \
  --namespace shipfast-prod \
  --attach-policy-arn arn:aws:iam::123456789012:policy/OrderAPIPolicy \
  --approve \
  --override-existing-serviceaccounts
```

**📖 How IRSA Works**

- eksctl creates an IAM role with a trust policy that allows the EKS OIDC provider to assume it
- The trust policy is scoped to the specific namespace + service account name — only pods using `order-api-sa` in `shipfast-prod` can assume this role
- At pod start, EKS injects a projected service account token as a file; the AWS SDK exchanges this for temporary STS credentials automatically
- No code change needed — AWS SDK v2 reads the token file and calls STS transparently

#### 5.2 Use the Service Account in the Deployment

```
# In order-api Deployment spec
spec:
  serviceAccountName: order-api-sa
containers:
    - name: order-api
image: ...ecr.../order-api:v2.1.0
# NO env AWS_ACCESS_KEY_ID — not needed
# SDK auto-reads token at /var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

**📖 IRSA in Action**

- Assign `serviceAccountName: order-api-sa` — that is the only change needed in the Deployment
- AWS SDK v2 automatically discovers credentials from the injected token file via the `AWS_WEB_IDENTITY_TOKEN_FILE` and `AWS_ROLE_ARN` env vars that EKS injects
- Credentials expire every 15 minutes and are refreshed automatically — no rotation needed
- storefront pod uses a different service account with different (read-only S3) permissions — least privilege per pod

#### 5.3 The IAM Policy for order-api

```
# OrderAPIPolicy — least privilege
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject", "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::shipfast-orders/*"
    },
    {
      "Effect": "Allow",
      "Action": ["sqs:SendMessage"],
      "Resource": "arn:aws:sqs:ap-south-1:123456789012:order-events"
    }
  ]
}
```

**📖 Least Privilege Principle**

- Only S3 Get/Put on `shipfast-orders` bucket — not list, not delete, not all buckets
- Only SQS SendMessage on the specific order-events queue — not read, not manage
- storefront pod's policy: S3 GetObject on `shipfast-static-assets` only — completely separate permissions
- If order-api is compromised, the attacker can only read/write to that one S3 bucket — cannot access databases, other queues, or other services

### 6. Phase 6 — AWS Secrets Manager Integration (CSI Driver)

**Business Problem:** order-api needs a PostgreSQL password, Razorpay API key, and JWT secret. Kubernetes Secrets are base64-encoded — not encrypted at rest unless you enable envelope encryption. AWS Secrets Manager encrypts with KMS and has automatic rotation built in.

#### 6.1 Install Secrets Store CSI Driver for AWS

```bash
# Install Secrets Store CSI Driver
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  -n kube-system

# Install AWS Provider
kubectl apply -f \
  https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

**📖 CSI Driver for AWS**

- Runs as a DaemonSet — one pod per node; mounts Secrets Manager values as files at pod start
- Pod uses IRSA to authenticate to Secrets Manager — no credentials needed in the CSI driver itself
- Secret fetched fresh at every pod start — if the secret is rotated in Secrets Manager, new pods get the new value automatically
- Running pods can get refreshed values with `rotationPollInterval` setting

#### 6.2 SecretProviderClass for ShipFast

```yaml
# secretproviderclass.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: shipfast-secrets
namespace: shipfast-prod
spec:
  provider: aws
parameters:
    objects: |
      - objectName: "shipfast/prod/pg-password"
        objectType: "secretsmanager"
      - objectName: "shipfast/prod/razorpay-key"
        objectType: "secretsmanager"
      - objectName: "shipfast/prod/jwt-secret"
        objectType: "secretsmanager"
  secretObjects:
    - secretName: shipfast-db-secret
type: Opaque
data:
        - objectName: shipfast/prod/pg-password
key: password
```

**📖 secretObjects — Sync to K8s Secret**

- **objectName** — matches the secret name in AWS Secrets Manager; use a path convention like `app/env/secret-name`
- **secretObjects** — optionally syncs the fetched value to a Kubernetes Secret object; allows existing apps that read from env vars to work without code changes
- The service account mounting this must have IRSA with `secretsmanager:GetSecretValue` permission on these specific secrets
- Create a secret in Secrets Manager: `aws secretsmanager create-secret --name shipfast/prod/pg-password --secret-string "SuperSecretPass"`

### 7. Phase 7 — Autoscaling: HPA + Cluster Autoscaler + Karpenter

**Business Problem:** ShipFast's Diwali flash sale: 18x traffic spike in 90 seconds. Standard Cluster Autoscaler takes 3–4 minutes to provision new nodes. Karpenter provisions nodes in under 60 seconds — fast enough to absorb the spike before orders drop.

**Scene 7 — ShipFast Flash Sale Post-Mortem | "Karpenter vs Cluster Autoscaler"**

> **Arjun** _Senior DevOps — ShipFast_
> 
> Cluster Autoscaler: detects Pending pods, calls Auto Scaling Group API, ASG launches EC2, EC2 boots, node joins cluster, pods schedule. That's 3 to 5 minutes. In a flash sale, the first 3 minutes is when 40% of orders come in. Karpenter: watches Pending pods directly, calls EC2 Fleet API with exact instance type needed, skips ASG entirely, node ready in 45 seconds. For ShipFast, Karpenter is not optional — it is the difference between capturing the flash sale and losing it.

#### 7.1 Horizontal Pod Autoscaler

```yaml
# hpa-order-api.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-order-api
namespace: shipfast-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
kind: Deployment
name: order-api
minReplicas: 5
maxReplicas: 50
metrics:
    - type: Resource
resource:
        name: cpu
target:
          type: Utilization
averageUtilization: 60
    - type: Resource
resource:
        name: memory
target:
          type: Utilization
averageUtilization: 70
```

**📖 Dual-Metric HPA**

- HPA scales on whichever metric requires more replicas — CPU or memory; the higher target wins
- **minReplicas: 5** — always 5 pods; at least one per AZ, two in primary AZ for peak pre-positioning
- **maxReplicas: 50** — maximum pod count; when Karpenter provisions enough nodes, HPA can scale to 50 order-api pods for the flash sale
- metrics-server must be installed — EKS does not include it by default: `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`

#### 7.2 Install Karpenter

```bash
# Create Karpenter IAM role via IRSA
eksctl create iamserviceaccount \
  --cluster eks-shipfast-prod \
  --name karpenter \
  --namespace karpenter \
  --attach-policy-arn arn:aws:iam::123456789012:policy/KarpenterControllerPolicy \
  --approve
# Install Karpenter via Helm
helm repo add karpenter https://charts.karpenter.sh
helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --set clusterName=eks-shipfast-prod \
  --set clusterEndpoint=$(aws eks describe-cluster \
    --name eks-shipfast-prod --query cluster.endpoint --output text) \
  --set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile
```

**📖 Karpenter vs Cluster Autoscaler**

- **Cluster Autoscaler** — works with ASGs; node provision time 3–5 min; must pre-define instance types in ASG
- **Karpenter** — calls EC2 Fleet API directly; node ready in ~45 seconds; selects the cheapest instance type that fits the pending pods automatically
- Karpenter also consolidates nodes — if 3 underutilised nodes can fit onto 2, it drains and terminates one automatically (saving cost)
- Install Karpenter only on the **app-ng** node group; keep Cluster Autoscaler for the system node group

#### 7.3 Karpenter NodePool and EC2NodeClass

```yaml
# karpenter-nodepool.yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: app-pool
spec:
  template:
    spec:
      nodeClassRef:
        name: app-nodeclass
requirements:
        - key: karpenter.sh/capacity-type
operator: In
values: [spot, on-demand]
        - key: kubernetes.io/arch
operator: In
values: [amd64]
  limits:
    cpu: 200
disruption:
    consolidationPolicy: WhenUnderutilized
consolidateAfter: 30s
```

**📖 NodePool Config**

- **capacity-type: spot, on-demand** — Karpenter tries Spot first (70% cheaper); falls back to On-Demand if no Spot available — best cost/reliability balance for flash sale scaling
- **limits.cpu: 200** — Karpenter will not provision more than 200 vCPUs total; prevents runaway scaling during bugs
- **consolidationPolicy: WhenUnderutilized** — after flash sale, Karpenter drains underutilised nodes and terminates them within 30 seconds — costs drop with traffic
- EC2NodeClass defines AMI family, subnet IDs, security groups, and instance profile for nodes Karpenter launches

### 8. Phase 8 — EKS Fargate (Serverless Pods)

**Business Problem:** ShipFast runs batch jobs for generating daily sales reports and sending email receipts. These jobs run for 5 minutes each, irregularly. Paying for dedicated EC2 nodes to sit idle 23 hours/day is wasteful. Fargate runs pods without nodes — pay per second of pod runtime.

#### 8.1 Create a Fargate Profile

```bash
# Create Fargate profile for batch workloads
eksctl create fargateprofile \
  --cluster eks-shipfast-prod \
  --name batch-profile \
  --namespace shipfast-batch \
  --labels workload=batch

# Any pod in shipfast-batch with label workload=batch
# runs on Fargate — no EC2 nodes needed
kubectl create namespace shipfast-batch
```

**📖 Fargate — Serverless Kubernetes**

- Pods run on Fargate — AWS provisions micro-VMs per pod; no EC2 instances to manage, patch, or pay for when idle
- Fargate profile: namespace + label selector determines which pods go to Fargate vs EC2 nodes
- Billing: vCPU-hours + GB-hours of actual pod runtime — idle time costs nothing
- Limitation: no DaemonSets on Fargate, no GPU, no privileged containers — fine for batch jobs
- ShipFast savings: batch jobs ran 6 hours/day on a dedicated m5.large; Fargate costs 80% less

#### 8.2 Deploy a Batch Job on Fargate

```yaml
# daily-report-job.yaml — runs on Fargate
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-sales-report
namespace: shipfast-batch
spec:
  schedule: "0 1 * * *" # 1 AM IST daily
jobTemplate:
    spec:
      template:
        metadata:
          labels: { workload: batch }  # Fargate selector
spec:
          serviceAccountName: report-sa # IRSA for S3 access
restartPolicy: Never
containers:
            - name: report-generator
image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/shipfast/reporter:v1.0
resources:
                requests: { cpu: "1", memory: "2Gi" }
```

**📖 Fargate Pod Sizing**

- Fargate selects the micro-VM size based on `resources.requests` — always set them; Fargate defaults to 0.25 vCPU + 0.5 GB which is too small for report generation
- **label workload=batch** — matches the Fargate profile selector; without this label, the pod would try to schedule on EC2 nodes instead
- Fargate pods run in private subnets with outbound internet access via NAT Gateway — they can call S3, DynamoDB, RDS, external APIs
- IRSA works identically on Fargate — report-sa service account gets S3 write access to save generated reports

### 9. Phase 9 — CloudWatch Container Insights and Alarms

**Business Problem:** ShipFast needs real-time visibility into pod failures, memory pressure, order-api latency, and node capacity — with automated alerts before customers notice issues.

#### 9.1 Enable Container Insights

```bash
# Install CloudWatch agent as DaemonSet
ClusterName=eks-shipfast-prod
RegionName=ap-south-1

curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml \
  | sed "s/{{cluster_name}}/${ClusterName}/g; \
    s/{{region_name}}/${RegionName}/g" \
  | kubectl apply -f -
```

**📖 What Container Insights Collects**

- **CloudWatch agent** — runs on every node; collects pod CPU, memory, disk, and network metrics
- **Fluent Bit** — runs on every node; ships container stdout/stderr logs to CloudWatch Logs groups
- Log groups created automatically: `/aws/containerinsights/eks-shipfast-prod/application`, `/host`, `/dataplane`
- All metrics queryable in CloudWatch Metrics under namespace `ContainerInsights`

#### 9.2 CloudWatch Alarm — OOMKill Alert

```
# Create alarm for OOMKilled containers
aws cloudwatch put-metric-alarm \
  --alarm-name "ShipFast-OrderAPI-OOMKill" \
  --namespace ContainerInsights \
  --metric-name pod_number_of_container_restarts \
  --dimensions \
    Name=PodName,Value=order-api \
    Name=ClusterName,Value=eks-shipfast-prod \
  --threshold 3 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --period 300 \
  --statistic Sum \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:shipfast-oncall
```

**📖 CloudWatch Alarms for EKS**

- **pod_number_of_container_restarts > 3** in 5 minutes — fires SNS → PagerDuty/Slack alert to on-call
- Other critical ShipFast alarms: node CPU > 80%, node memory > 85%, pending pods > 0 for 5 min, ALB 5xx rate > 1%
- Use CloudWatch Logs Insights to query logs: `filter @message like /Exception/ | stats count() by bin(5m)`
- Set up a CloudWatch Dashboard showing order-api pod count, CPU, memory, and ALB requests per second on one screen

### 10. Phase 10 — IAM RBAC and aws-auth ConfigMap

**Business Problem:** ShipFast has 3 teams — Platform (full cluster access), Backend (deploy to shipfast-prod only), Data (read-only to view pod status). IAM groups + aws-auth ConfigMap controls who can do what in the cluster.

#### 10.1 Grant IAM Users/Roles Cluster Access

```yaml
# aws-auth-patch.yaml — map IAM to K8s RBAC
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/ShipFastPlatformRole
username: platform-team
groups: [system:masters]
    - rolearn: arn:aws:iam::123456789012:role/ShipFastBackendRole
username: backend-team
groups: [shipfast-developers]
    - rolearn: arn:aws:iam::123456789012:role/KarpenterNodeRole
username: system:node:{{EC2PrivateDNSName}}
groups: [system:bootstrappers, system:nodes]
```

**📖 aws-auth ConfigMap**

- EKS's bridge between IAM and Kubernetes RBAC — maps IAM ARNs to Kubernetes usernames and groups
- **system:masters** — cluster-admin; platform team only
- **shipfast-developers** — maps to a K8s ClusterRole/Role you define; deploy but not delete
- Karpenter-provisioned nodes must be in aws-auth so the kubelet can join the cluster
- Edit carefully: a corrupt aws-auth ConfigMap locks everyone out — always keep a backup

#### 10.2 Kubernetes RBAC for Backend Team

```yaml
# backend-role.yaml — deploy permissions only
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: shipfast-backend-role
namespace: shipfast-prod
rules:
  - apiGroups: ["", "apps"]
    resources: [pods, deployments, services, configmaps]
    verbs: [get, list, watch, create, update, patch]
  - apiGroups: [""]
    resources: [pods/log, pods/exec]
    verbs: [get, create]
---
kind: RoleBinding
metadata:
  name: bind-backend-role
namespace: shipfast-prod
subjects:
  - kind: Group
name: shipfast-developers
roleRef:
  kind: Role
name: shipfast-backend-role
```

**📖 RBAC Chain**

- aws-auth maps `ShipFastBackendRole` IAM role → K8s group `shipfast-developers`
- RoleBinding binds group `shipfast-developers` → Role `shipfast-backend-role`
- Role allows deploy/update/patch but NOT delete — developers cannot delete production pods accidentally
- Backend devs use: `aws sts assume-role --role-arn arn:aws:iam::123456789012:role/ShipFastBackendRole` then `aws eks update-kubeconfig`

### 11. Phase 11 — Network Policy with VPC CNI

**Business Problem:** All pods in ShipFast's cluster can reach all other pods by default. A compromised storefront pod should never be able to talk to the payment database. Network Policies enforce service-level isolation.

#### 11.1 Enable Network Policy on EKS

```bash
# Enable VPC CNI network policy (built into EKS 1.25+)
aws eks update-addon \
  --cluster-name eks-shipfast-prod \
  --addon-name vpc-cni \
  --configuration-values '{"enableNetworkPolicy": "true"}'

# Or install Calico for advanced policy features
kubectl apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

**📖 EKS Network Policy Options**

- **VPC CNI Network Policy** — built into EKS VPC CNI since 1.25; no extra install; good for basic allow/deny rules
- **Calico** — richer policy features (CIDR-based rules, namespace selectors, egress DNS rules); community standard for complex policy requirements
- Without enabling network policy support, NetworkPolicy objects are accepted by Kubernetes but have no effect — a silent misconfiguration
- Verify Calico: `kubectl get pods -n kube-system -l k8s-app=calico-node`

#### 11.2 Default-Deny and Service Allowlist

```yaml
# default-deny.yaml — deny all in shipfast-prod
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
namespace: shipfast-prod
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# allow only order-api to reach postgres
kind: NetworkPolicy
metadata:
  name: allow-order-api-to-pg
namespace: shipfast-prod
spec:
  podSelector:
    matchLabels: { app: postgresql }
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: order-api }
      ports:
        - { protocol: TCP, port: 5432 }
```

**📖 Policy-First Design**

- Start with default-deny — all traffic blocked by default in the namespace
- Add explicit allow rules for each required path: order-api → postgres, storefront → order-api, ALB → all app pods
- Always add a policy to allow DNS egress to kube-system on port 53 — otherwise CoreDNS stops working for all pods
- Test policies with: `kubectl exec -it <storefront-pod> -- nc -zv postgres-svc 5432` — should time out if policy is correct

### 12. Phase 12 — EKS Cluster Upgrade (Zero Downtime)

**Business Problem:** EKS 1.27 is approaching end of support. Upgrade to 1.29 without interrupting order processing.

#### 12.1 Upgrade Sequence

1. Upgrade Control Plane

AWS handles HA upgrade of API server, etcd, scheduler — zero downtime to running pods
Command: `aws eks update-cluster-version --name eks-shipfast-prod --kubernetes-version 1.29`
Takes ~10–15 minutes; monitor with: `aws eks describe-update --name eks-shipfast-prod --update-id <id>`

2. Update EKS Add-ons

vpc-cni, kube-proxy, coredns — must be compatible with the new K8s version
Command: `aws eks update-addon --cluster-name eks-shipfast-prod --addon-name vpc-cni --addon-version v1.16.0-eksbuild.1`
Check compatible versions: `aws eks describe-addon-versions --kubernetes-version 1.29 --addon-name vpc-cni`

3. Upgrade Managed Node Groups

EKS rolls nodes one at a time — cordons, drains (respecting PDBs), launches new node on new AMI, deregisters old node
Command: `aws eks update-nodegroup-version --cluster-name eks-shipfast-prod --nodegroup-name app-ng`
PDB `minAvailable: 4` on order-api ensures at least 4 pods stay running during every drain

#### 12.2 Managed Node Group Upgrade Command

```
# Upgrade node group to new K8s version + latest AMI
aws eks update-nodegroup-version \
  --cluster-name eks-shipfast-prod \
  --nodegroup-name app-ng \
  --kubernetes-version 1.29 \
  --force
# Watch node group update status
aws eks describe-nodegroup \
  --cluster-name eks-shipfast-prod \
  --nodegroup-name app-ng \
  --query nodegroup.status
```

**📖 Managed Node Group Upgrade**

- EKS handles the entire rollout — no manual cordon/drain commands needed
- EKS respects PDB — if draining a node would violate minAvailable, the drain waits up to 15 minutes before forcing
- **--force** — forces upgrade even if some pods can't be evicted (PDB violations); use only in last resort
- Upgrade one node group at a time: system-ng first, then app-ng — never both simultaneously
- Karpenter-provisioned nodes auto-upgrade when Karpenter detects they are outdated and consolidates them

### 13. Phase 13 — EKS CI/CD with CodePipeline and GitHub Actions

**Business Problem:** ShipFast developers push code; CI/CD must build, scan, push to ECR, and deploy to EKS automatically — with staging promotion and manual prod gate.

#### 13.1 GitHub Actions — EKS Deploy

```bash
# .github/workflows/deploy.yml
name: ShipFast EKS Deploy
on:
  push:
    branches: [main]

env:
  AWS_REGION: ap-south-1
  ECR_REGISTRY: 123456789012.dkr.ecr.ap-south-1.amazonaws.com
  IMAGE_TAG: ${{ github.sha }}
permissions:
  id-token: write # For OIDC — no static AWS keys in GitHub Secrets
contents: read
jobs:
  build-and-push:
    runs-on: ubuntu-latest
steps:
      - uses: actions/checkout@v4
      - name: Configure AWS via OIDC
uses: aws-actions/configure-aws-credentials@v4
with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
aws-region: ap-south-1
      - name: Login to ECR
uses: aws-actions/amazon-ecr-login@v2
      - name: Build, Scan, Push
run: |
          docker build -t $ECR_REGISTRY/shipfast/order-api:$IMAGE_TAG .
          docker push $ECR_REGISTRY/shipfast/order-api:$IMAGE_TAG
      - name: Deploy to Staging
run: |
          aws eks update-kubeconfig --name eks-shipfast-staging
          kubectl set image deployment/order-api \
            order-api=$ECR_REGISTRY/shipfast/order-api:$IMAGE_TAG \
            -n shipfast-prod
          kubectl rollout status deployment/order-api -n shipfast-prod
```

> **OIDC — no static AWS keys** — GitHub Actions uses OIDC federation to assume the GitHubActionsRole; no AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY stored in GitHub Secrets — the biggest security upgrade in CI/CD
**github.sha as IMAGE_TAG** — every image is tagged with the exact commit SHA; if a prod issue appears, `git log` immediately shows what changed
**kubectl rollout status** — waits for the rolling update to complete successfully; if the new image crashes, the step fails and the pipeline stops — staging never goes into a broken state silently
Add a second job with `environment: production` and required reviewers for the prod deploy gate

### 14. Phase 14 — EKS Storage with EBS and EFS

**Business Problem:** ShipFast's PostgreSQL needs block storage (EBS) that survives pod restarts. The product image CDN worker needs shared storage accessible from multiple pods simultaneously (EFS).

#### 14.1 EBS CSI Driver — Dynamic PVC for PostgreSQL

```yaml
# Install EBS CSI Driver via EKS add-on
aws eks create-addon \
  --cluster-name eks-shipfast-prod \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789012:role/EBSCSIDriverRole

# pvc-postgres.yaml — GP3 SSD storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
namespace: shipfast-prod
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
resources:
    requests:
      storage: 200Gi
```

**📖 EBS on EKS**

- **aws-ebs-csi-driver** — EKS managed add-on; handles EBS volume lifecycle (create, attach, detach, delete)
- **gp3 StorageClass** — GP3 EBS volumes; 3000 IOPS and 125 MB/s throughput by default; 20% cheaper than GP2
- **ReadWriteOnce** — EBS volumes attach to exactly one node/pod; cannot be shared across nodes
- For GP3 custom IOPS: create a custom StorageClass with `parameters: iops: "6000" throughput: "250"`
- EBS volumes are AZ-specific — PostgreSQL pod and its EBS volume must be in the same AZ; use node affinity to pin the pod to the correct AZ

#### 14.2 EFS — Shared Storage Across Pods

```yaml
# Install EFS CSI Driver
aws eks create-addon \
  --cluster-name eks-shipfast-prod \
  --addon-name aws-efs-csi-driver

# pvc-shared.yaml — ReadWriteMany for multi-pod access
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: product-images
namespace: shipfast-prod
spec:
  accessModes: [ReadWriteMany]
  storageClassName: efs-sc
resources:
    requests:
      storage: 1Ti
```

**📖 EFS vs EBS**

- **EFS = ReadWriteMany** — multiple pods on multiple nodes simultaneously; all 5 order-api pods can read product images from the same EFS volume
- **EBS = ReadWriteOnce** — one pod only; suitable for databases but not shared file systems
- EFS storage grows and shrinks automatically — no pre-provisioning or capacity planning
- EFS is AZ-resilient — data replicated across all AZs in the region; pod can be on any AZ
- Cost: EFS is ~5x more expensive per GB than EBS — use it only for genuinely shared data

### 15. Phase 15 — EKS Production Best Practices Summary

**Scene 15 — ShipFast Diwali Post-Mortem | "The Platform That Didn't Break"**

> **Deepa** _Lead DevOps — ShipFast_
> 
> Diwali flash sale — 6 PM. Traffic spiked 22x. Karpenter provisioned 8 new nodes in 52 seconds. HPA scaled order-api from 5 to 47 pods. PDB ensured zero pods were evicted during the Karpenter consolidation that happened at 8 PM when traffic dropped. ALB with WAF blocked 1.2 million bot requests before they hit any pod. IRSA meant no credentials leaked from the 3 incidents where developers pushed to public repos. Revenue: ₹3.2 crore in 4 hours. Zero downtime. Zero on-call pages. Zero manual interventions.

> **Arjun** _Senior DevOps — ShipFast_
> 
> And the cost: after the flash sale, Karpenter terminated 6 of those 8 new nodes within 4 minutes of traffic dropping. We paid for exactly the compute we needed, for exactly the time we needed it. Spot instances for 70% of the nodes meant the flash sale infrastructure cost half of what we budgeted. That is what EKS with Karpenter looks like in production.

> **EKS Production Checklist — ShipFast Engineering Standards**

> - Always deploy managed node groups across 3 AZs — a single AZ failure must not drop availability below 2/3 capacity; use topologySpreadConstraints in every Deployment
> - Always set a PodDisruptionBudget for every production Deployment — without PDB, EKS node upgrades evict all replicas simultaneously
> - Always use IRSA for AWS service access — never embed AWS access keys in pods, environment variables, or ConfigMaps; they will eventually leak
> - Always set resources.requests on every container — HPA, Cluster Autoscaler, and Karpenter all make decisions based on requests; without them, scheduling is random
> - Never use image tag :latest in production — tag every image with the commit SHA; enables instant rollback and clear audit trail of what is running where
> - Always enable ECR image scanning (scanOnPush=true) — catch CVEs at build time, not after a breach; integrate with AWS Security Hub for centralised findings
> - Always use the EKS managed add-ons (vpc-cni, coredns, kube-proxy) — AWS keeps them compatible with the cluster version; self-managed add-ons break on cluster upgrades
> - Always upgrade staging cluster first — same sequence, same commands; fix issues in staging before they block the production upgrade window

##### EKS Cost Optimisation — ShipFast FinOps Rules

- Use Karpenter with `capacity-type: spot` first — Spot instances cost 70–80% less than On-Demand; Karpenter automatically diversifies across instance families to improve Spot availability
- Use Fargate for batch jobs and periodic tasks — zero node cost when jobs are not running; no idle EC2 capacity
- Enable Karpenter consolidation (`consolidationPolicy: WhenUnderutilized`) — after flash sale traffic drops, underutilised nodes are drained and terminated within minutes
- Use AWS Compute Optimizer to right-size node group instance types — it analyses 14 days of CloudWatch metrics and recommends cheaper instance types with equivalent performance
- Use VPA in recommender mode for 2 weeks — right-sized resource requests lead to better bin-packing on nodes; same workload on fewer nodes
- Set EBS lifecycle policies — automatically delete unattached PVs (orphaned after pod deletion) using a CronJob that runs `kubectl get pv | grep Released` and deletes them

##### ⚠️ Common EKS Mistakes — ShipFast Incident Log

- **Corrupted aws-auth ConfigMap** — accidentally deleted the node IAM role entry; all nodes lost connectivity to the API server → cluster split-brain; recovery required AWS support (fix: always keep a backup, use eksctl to manage aws-auth)
- **No PDB on order-api** — EKS node group upgrade evicted all 5 replicas in 30 seconds → full order processing outage for 4 minutes (fix: minAvailable: 4)
- **IRSA not configured — access keys in env vars** — developer committed Deployment YAML with AWS_ACCESS_KEY_ID to GitHub → key rotated, all pods broke in prod simultaneously (fix: IRSA + remove all static credentials)
- **EBS PVC and pod in different AZs** — pod rescheduled to different AZ after node failure; EBS volume couldn't attach → PostgreSQL stuck in Pending for 20 minutes (fix: node affinity to pin pod and volume to same AZ)
- **No readinessProbe** — order-api received traffic during 20-second startup → HTTP 503s for every deploy (fix: readinessProbe httpGet /healthz with initialDelaySeconds: 15)
- **Upgrading node group before control plane** — kubelet version 1.30 on nodes, API server on 1.28 → nodes refused to join; upgrade command failed, cluster partially updated (fix: always upgrade control plane first, then add-ons, then node groups)

### Quick Reference — All EKS Commands at ShipFast

Command

What It Does

eksctl create cluster -f cluster-config.yaml

Provision full EKS cluster from config file

aws eks update-kubeconfig --name eks-shipfast-prod --region ap-south-1

Configure kubectl for the EKS cluster

eksctl create nodegroup --cluster eks-shipfast-prod -f ng-config.yaml

Add a new managed node group

eksctl scale nodegroup --cluster eks-shipfast-prod --name app-ng --nodes 8

Manually scale node group to 8 nodes

eksctl create iamserviceaccount --cluster ... --name ... --attach-policy-arn ...

Create IRSA — pod-level IAM role

aws eks update-cluster-version --name ... --kubernetes-version 1.29

Upgrade EKS control plane version

aws eks update-nodegroup-version --cluster-name ... --nodegroup-name app-ng

Upgrade managed node group to new AMI

aws eks create-addon --cluster-name ... --addon-name vpc-cni

Install or enable an EKS managed add-on

aws eks describe-addon-versions --kubernetes-version 1.29 --addon-name coredns

Find compatible add-on versions for a K8s release

eksctl create fargateprofile --cluster ... --namespace shipfast-batch

Create Fargate profile for serverless pods

aws ecr get-login-password | docker login --username AWS --password-stdin <ecr-url>

Authenticate Docker to ECR

aws ecr describe-image-scan-findings --repository-name shipfast/order-api --image-id imageTag=v2.1.0

View CVE scan results for an ECR image

kubectl rollout status deployment/order-api -n shipfast-prod

Monitor rolling update progress

kubectl rollout undo deployment/order-api -n shipfast-prod

Roll back to previous Deployment revision

kubectl top nodes

Show CPU and memory usage per node

kubectl top pods -n shipfast-prod --sort-by=memory

Show pods sorted by memory usage

kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

Evict pods from a node before maintenance

eksctl get cluster --region ap-south-1

List all EKS clusters in the region

eksctl delete cluster --name eks-shipfast-prod

Delete cluster and all associated resources

aws eks list-nodegroups --cluster-name eks-shipfast-prod

List all node groups in the cluster

##### 🏋️ Hands-On Exercises — Extend ShipFast's EKS Setup

1. **Configure KEDA for SQS-based scaling:** Install KEDA on EKS (`helm install keda kedacore/keda -n keda`), create a ScaledObject targeting order-api Deployment and scaling based on the `order-events` SQS queue depth. Set minReplicas: 0 and maxReplicas: 30 — during flash sales, queue depth explodes and KEDA scales pods to drain it; at night, queue is empty and KEDA scales to 0. Create the KEDA IRSA to give KEDA SQS read access without static credentials.
2. **Set up GuardDuty for EKS runtime threat detection:** Enable EKS Audit Log Monitoring in AWS GuardDuty (`aws guardduty update-detector --detector-id <id> --features Name=EKS_AUDIT_LOGS,Status=ENABLED`). Enable EKS Runtime Monitoring for behavioural detection inside pods. Configure a GuardDuty finding event rule in EventBridge to send findings to SNS → Slack. Simulate a finding by running `kubectl exec -it <pod> -- curl http://169.254.169.254/latest/meta-data/` (IMDS access from pod — GuardDuty should flag it).
3. **Implement GitOps with Flux:** Install Flux on EKS (`flux bootstrap github --owner=shipfast --repository=k8s-gitops --path=clusters/prod`). Commit the order-api Deployment YAML to the GitOps repo. Update the image tag in the YAML and push — Flux auto-applies the change. Remove `kubectl apply` from the GitHub Actions pipeline — the pipeline only pushes the new tag to the GitOps repo; Flux syncs the cluster. Full audit trail: Git history shows every cluster change with author and timestamp.
4. **Configure VPC Endpoints to eliminate NAT Gateway costs:** EKS worker nodes in private subnets call ECR, S3, Secrets Manager, and CloudWatch over the internet via NAT Gateway — expensive at high volume. Create VPC Interface Endpoints for ECR (ecr.api, ecr.dkr), S3 Gateway Endpoint, Secrets Manager endpoint, and CloudWatch endpoint. All EKS traffic to these AWS services now stays inside the VPC — zero NAT Gateway data processing charges. Measure cost savings in Cost Explorer after 1 week.
5. **Set up multi-cluster with EKS Blueprints:** Use the AWS EKS Blueprints Terraform module to provision a second EKS cluster (eks-shipfast-dr) in ap-southeast-1 (Singapore) as a disaster recovery cluster. Configure the same node groups, add-ons, and IRSA roles using Terraform modules. Set up AWS Global Accelerator to route 10% of traffic to Singapore for active-active testing. Simulate an ap-south-1 cluster outage by setting all node groups to 0 — verify Global Accelerator routes 100% traffic to Singapore within 60 seconds.

**Quiz: ❓ Interview Question: ShipFast's order-api needs to write to an S3 bucket and send messages to SQS. A junior engineer put the AWS Access Key and Secret in environment variables in the Deployment YAML. What is wrong with this and what is the correct solution?**

- A) Environment variables are fine — they are stored encrypted in etcd
- B) The keys should be in a Kubernetes Secret instead of env vars directly
- C) Static access keys in any form are wrong — use IRSA so pods get temporary, auto-rotating credentials via the service account without any static keys
- D) The keys should be base64-encoded before putting in environment variables

> **Answer/explanation:** ✅ **Answer: C.** Static AWS access keys have multiple failure modes: developer commits the Deployment YAML to a public repo → key exposed; key rotation breaks all pods simultaneously; same key used across dev and prod — one breach exposes both. IRSA (IAM Roles for Service Accounts) gives the order-api pod a temporary STS token valid for 15 minutes, auto-refreshed, scoped only to the specific S3 bucket and SQS queue. If a pod is compromised, the attacker gets a credential that expires in 15 minutes and can only access two specific AWS resources. There is no static secret to commit, rotate, or leak. This is the correct answer in every EKS security interview — static keys vs IRSA is the most common EKS security question.

##### Common Fresher Questions — EKS at ShipFast

**Q: Q: What is the difference between eksctl, aws CLI, and kubectl for EKS operations?**

A: **eksctl** — EKS-specific tool; creates/deletes clusters, node groups, Fargate profiles, IRSA service accounts; abstracts CloudFormation; use for cluster lifecycle
**aws CLI** — general AWS tool; can manage EKS resources directly (slower, more verbose); also used for ECR, IAM, CloudWatch, Secrets Manager — everything outside the cluster
**kubectl** — Kubernetes API client; everything inside the cluster — pods, deployments, services, ingress, RBAC; works identically on EKS, AKS, GKE, on-prem
Rule: use eksctl for EKS infrastructure, aws CLI for AWS services, kubectl for Kubernetes resources

**Q: Q: Why does a PDB block Karpenter consolidation? How do you fix it?**

A: Karpenter tries to drain a node to consolidate — evicting pods one by one
If evicting a pod would violate the PDB (go below minAvailable), Kubernetes rejects the eviction
Karpenter backs off and retries after a configurable period
Fix: ensure minReplicas in HPA ≥ minAvailable in PDB + 1 — so there is always at least one pod that can be evicted without violating the PDB
Example: HPA minReplicas=5, PDB minAvailable=4 → Karpenter can always evict 1 pod, drain the node, reschedule on a smaller node

**Q: Q: EKS charges ₹0 for the control plane, right? What do you actually pay for?**

A: **EKS cluster fee** — $0.10/hour per cluster (~₹200/day); this covers the managed control plane
**EC2 nodes** — the actual worker nodes; m5.xlarge = ~$0.192/hour; this is the largest cost
**EBS volumes** — persistent storage for pods; GP3 = $0.08/GB/month
**NAT Gateway** — outbound internet from private subnets; $0.045/GB processed; use VPC endpoints to eliminate this for AWS service traffic
**ALB** — Application Load Balancer; $0.008/LCU-hour + $0.0085/hour fixed
**ECR** — $0.10/GB/month after 500 GB; set lifecycle policies to delete old images
Fargate: $0.04048/vCPU-hour + $0.004445/GB-hour — only while pod is running

### EKS Mastery Complete 🎉

You have built ShipFast's complete production EKS platform — cluster provisioning with eksctl across 3 AZs, managed node groups, ECR integration, ALB Ingress with WAF and ACM TLS, IRSA for zero-credential AWS access, Secrets Manager CSI Driver, HPA + Karpenter for 45-second flash-sale scaling, Fargate for serverless batch, CloudWatch Container Insights with alarms, IAM RBAC via aws-auth, Network Policies with Calico, EBS/EFS storage, and GitHub Actions CI/CD with OIDC authentication.

> **Deepa**
> 
> "Diwali flash sale — 22x traffic spike, ₹3.2 crore in 4 hours, zero downtime, zero on-call pages. Karpenter provisioned 8 nodes in 52 seconds. HPA scaled order-api to 47 pods. WAF blocked 1.2 million bots. IRSA meant no credential leaked even when two developers accidentally pushed to public repos. We went from 11-minute outages and manual restarts to a platform that heals, scales, and secures itself. That is what EKS done right looks like."

> **Arjun**
> 
> "And cost: flash sale infrastructure cost 52% of budget because Karpenter used Spot instances and terminated nodes within 4 minutes of traffic dropping. The finance team asked why the AWS bill for Diwali week was lower than a normal Wednesday. That is what Karpenter consolidation looks like at scale. We saved more than the EKS cluster fee in the first week."

> **Next: Advanced EKS — Service Mesh with App Mesh, EKS Anywhere, Multi-Cluster with GitOps & eBPF with Cilium**

> - **AWS App Mesh / Istio** — mTLS between all ShipFast services, traffic shifting for canary releases, distributed tracing to AWS X-Ray, circuit breaking for payment API dependencies
> - **EKS Anywhere** — run EKS on on-premises VMware or bare metal; same eksctl commands, same add-ons, same IAM model — hybrid cloud for ShipFast's Bengaluru data centre
> - **Cilium with eBPF** — replace kube-proxy with Cilium for 40% lower latency and packet-level network policies; Hubble for real-time network flow visibility without sidecars
> - **Multi-cluster GitOps with Argo CD** — manage 3 EKS clusters (prod, staging, dr) from one Argo CD instance; ApplicationSets deploy the same app to all clusters with environment-specific values
> - **EKS Pod Identity** — next-generation replacement for IRSA; simpler setup, no OIDC provider needed, one IAM role shared across multiple clusters; available in EKS 1.29+
> - **Amazon VPC Lattice** — service-to-service networking without Ingress or service mesh; automatic mutual TLS, IAM auth between services across VPCs and accounts; zero sidecar overhead
