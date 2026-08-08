# Micro Services Project

## OpenTelemetry Astronomy Shop Demo

This project contains the Open Telemetry Astronomy Shop, a microservice-based distributed system intended to illustrate the implementation of Open Telemetry in a near real-world environment.

### Project Goals
- Provide a realistic example of a distributed system that can be used to demonstrate OpenTelemetry instrumentation and observability.
- Build a base for vendors, tooling authors, and others to extend and demonstrate their OpenTelemetry integrations.
- Create a living example for OpenTelemetry contributors to use for testing new versions of the API, SDK, and other components or enhancements.


### Supported Languages
Open Telemetry supports many popular programming languages including:
- Python
- Java
- JavaScript / Node.js
- Go
- .NET
- Ruby
- PHP

## Micro Services Architecture

<img width="1166" height="1044" alt="image" src="https://github.com/user-attachments/assets/9ce5f0d0-c669-4a5a-a941-07c3937bed42" />

This architecture illustrates a microservices-based e-commerce architecture using various communication protocols (mainly gRPC, HTTP, and TCP), and integrates tools like Kafka, Envoy, Valkey, and Flagd for key functionalities.

### Top Layer – Entry Points & Proxy
- Internet, Load Generator, React Native App
  - These represent external users or automated testing tools accessing the application via HTTP.
- Frontend Proxy (Envoy)
  - Acts as an API Gateway or Ingress Controller.
  - Routes HTTP traffic to internal services like Frontend, Image Provider, and flagd-ui.
  - Offers load balancing, routing, and security features.

### Frontend Layer
- Frontend
  - Serves the web UI and connects users with backend services.
  - Communicates via gRPC to:
    - Ad, Cart, Checkout, Currency, Recommendation, Product Catalog.
- flagd-ui
  - UI dashboard for feature flag management (likely connects to Flagd).
- Image Provider (nginx)
  - Serves static assets (e.g., product images).

### Core Services
- Checkout
  - Central service in the buying workflow.
  - Communicates with:
    - Shipping, Quote, Email, Currency, Payment, Fraud Detection, Flagd.
- Cart
  - Maintains user carts, backed by Cache (Valkey) and integrated with:
    - Ad, Flagd.
- Ad
  - Provides ad content, possibly for recommendations or marketing.
- Recommendation
  - Suggests products to users based on various data points.
  - Talks to Product Catalog.
- Product Catalog
  - Stores product details, connects to both Recommendation and Frontend.

### Supporting Services
- Cache (Valkey)
  - Used for caching (probably Redis-compatible like Valkey).
  - Supports Cart and Ad.
- queue (Kafka)
  - Event streaming platform.
  - Accepts messages from Checkout, passes them to:
    - Accounting
    - Fraud Detection

### Backend Services
- Accounting
  - Processes financial data from Kafka.
- Fraud Detection
  - Analyzes data from Kafka to detect fraudulent activity.
  - Also talks to Flagd.
- Shipping
  - Handles shipping details; communicates over HTTP and gRPC.
- Quote
  - Generates shipping or price quotes.
- Email
  - Sends confirmation and update emails.
- Currency
  - Converts prices to different currencies.
- Payment
  - Processes transactions, talks to Flagd.

### Feature Flagging
- Flagd
  - Central service managing feature flags.
  - Communicated with by:
    - Cart, Checkout, Fraud Detection, Payment.

### Communication Types
- gRPC – Used extensively for internal service-to-service calls (fast & efficient).
- HTTP – Used for user-facing and REST-based services.
- TCP – Used where event-streaming or lower-level connections are needed (Kafka, service queues).

### Architecture Summary
- Microservices based – each component handles a specific responsibility.
- Event-driven – Kafka is used for decoupling & async processing.
- Service Mesh / Proxy – Envoy acts as the API gateway.
- Caching – via Valkey (Redis-like).
- Observability & Feature Flags – via Flagd and possibly frontend telemetry.
- Platform-Agnostic Access – UI via browser + mobile app (React Native).

### Benefits
- Unified approach to collecting metrics, logs, and traces.
- Vendor-neutral and widely adopted.
- Integrates easily with tools like Prometheus, Grafana, and Jaeger.
- Supports both manual and automatic instrumentation.

## Repository Layout

This repo is a **monorepo**: every microservice lives under `src/<service>/` on `main`, each with its own `Dockerfile`. There are 18 services; 17 are built as custom images, and `flagd` runs from the upstream `ghcr.io/open-feature/flagd` image (only its config, `src/flagd/demo.flagd.json`, lives here).

## CI: GitHub Actions

[.github/workflows/build-and-push.yml](.github/workflows/build-and-push.yml) is the primary CI pipeline. On every push to `main` that touches `src/**` (or via manual **Run workflow** dispatch, which builds all 17 services), it runs a matrix job **per changed service** with these stages:

1. **Checkout** — `actions/checkout@v4`.
2. **SonarQube Scan** — `SonarSource/sonarqube-scan-action`, scoped to that service's `src/<service>` dir. Wired against `SONAR_HOST_URL`/`SONAR_TOKEN` secrets; `continue-on-error: true` so it doesn't block the pipeline until a real server is configured.
3. **Build & Push** — Buildx build using each service's own `context`/`dockerfile` pair (several — `cart`, `accounting` — have it nested a level deeper). Pushes `docker.io/adarshbarkunta/<service>:latest` and `:<git-sha>`, with GHA layer caching.
4. **Trivy Vulnerability Scan** — scans the image just pushed; report-only (`exit-code: 0`, never blocks), results uploaded to the repo's **Security → Code scanning** tab.
5. **Update deployment-service.yml** — rewrites that service's `image:` line to the newly pushed tag and commits/pushes back to `main` (GitOps). Since matrix jobs run in parallel, this step retries with a rebase on push conflicts.

### Required repo secrets
Set these under **Settings → Secrets and variables → Actions**:
- `DOCKERHUB_USERNAME` — Docker Hub username/namespace images are pushed under. *(already set)*
- `DOCKERHUB_TOKEN` — a Docker Hub [access token](https://hub.docker.com/settings/security) (not your account password).
- `SONAR_HOST_URL` / `SONAR_TOKEN` — your SonarQube server URL and auth token. Until these are set, the scan step just no-ops.

## CI: Jenkins (alternative)

[Jenkinsfile](Jenkinsfile) implements the same five stages for a Jenkins pipeline job pointed at this repo's `main` branch. It auto-detects which `src/<service>` directories changed since the last commit (or accepts an explicit `SERVICES_OVERRIDE` build parameter) and loops through: SonarQube scan → Docker build & push → Trivy scan → `deployment-service.yml` update, per service. Sonar-scanner and Trivy run as Docker containers, so nothing extra needs installing on the Jenkins agent beyond Docker itself.

Jenkins credentials required (**Manage Jenkins → Credentials**):
- `docker-cred` (Username/password) — Docker Hub push access.
- `github-cred` (Username/password: GitHub username + PAT with repo write access) — pushes the `deployment-service.yml` update back to `main`.
- `sonar-token` (Secret text) — SonarQube auth token.

And a `SONAR_HOST_URL` global environment variable (**Manage Jenkins → System**) pointing at your SonarQube server.

## Eks setup in Ubuntu server using Terraform

Terraform code Repository URL: 
https://github.com/adarsh0331/Project_10_Eks_Cluster_with_terraform.git

Complete terraform files to create EKS in AWS VPC is available in the eks-install folder of this repo. This includes remote backend and statelocking implementation as well.

- eks-install: Folder that holds the complete terraform hcl files.
- backend: Folder that holds hcl files for s3 bucket and dynamodb creation.
- modules: Terraform Modules for VPC and EKS.
- main.tf: Main file that invokes the modules to create EKS in VPC.
- variables.tf: Variables for main.tf
- Jenkinfile: Jenkins file to trigger pipeline
- outputs.tf: Output values you wish to see post terraform execution, For example - VPC ID.

### Connect to your provisioning EC2 server
```bash
ssh -i your-key.pem ubuntu@your-ec2-ip
```

### aws cli:
```bash
sudo apt update
sudo apt install unzip curl -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### kubectl installation:
```bash
curl -LO "https://dl.k8s.io/release/$(curl -sSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version –client
```

### eksctl installation:
```bash
curl -sSL "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" -o eksctl.tar.gz
tar -xzf eksctl.tar.gz
sudo mv eksctl /usr/local/bin/
eksctl version
```

### AWS Configure
Provide your AWS Access Key, Secret Key, region, and output format.

### Provision the cluster
From the `eks-install` folder of the [Terraform repo](https://github.com/adarsh0331/Project_10_Eks_Cluster_with_terraform.git):
```bash
terraform init
terraform apply
```

### Check:
- EKS cluster in AWS Console
- Nodes are in Ready state

## Argocd Installation:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argocd/stable/manifests/install.yaml
kubectl get pods -n argocd
kubectl get svc -n argocd
kubectl edit svc argocd-server -n argocd            # change type: LoadBalancer
kubectl get svc -n argocd
```

Take the LoadBalancer ip and access it in web

```bash
kubectl get secrets -n argocd
kubectl edit secret argocd-initial-admin-secret -n argocd
echo RDg3ZjNUaDg3S3ZmTDhpbw== | base64 –decode         # -------Replace it with ur secret
```

Username :  admin

Password : <Your passaword>

## Create Application via Web UI
1. Open Argo CD Web UI (e.g., http://<ARGOCD_SERVER>:80)
2. Login with username/password
3. Click “New App”
4. Fill the form:
   - Application Name: opentelemetry-demo
   - Project: default
   - Repository URL: https://github.com/open-telemetry/opentelemetry-demo.git
   - Revision: HEAD
   - Path: Kubernetes/complete
   - Cluster URL: https://kubernetes.default.svc
   - Namespace: argocd
5. Enable Auto-Sync if you want Argo CD to automatically apply changes
6. Click Create

To access the application in the web change frontend-proxy cluster ip to LoadBalancer

## Prometheus & Grafana & Jaeger :

### Helm installation:
```bash
wget https://get.helm.sh/helm-v3.17.2-linux-amd64.tar.gz
tar -xvf helm-v3.17.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

Add OpenTelemetry Helm repository:
```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

To install the chart with the release name my-otel-demo:
```bash
helm install my-otel-demo open-telemetry/opentelemetry-demo
```

TAKE THE FRONTEND-PROXY LOAD-BALANCER IP AND ACCESS PROMETHEUS & GRAFANA IN WEB

## CD: Deploying to Kubernetes

[deployment-service.yml](deployment-service.yml) holds the rendered Kubernetes manifests (Deployments + Services) for all 18 services. Once you have a cluster and `kubectl` context configured:

```bash
kubectl create namespace ms
kubectl apply -n ms -f deployment-service.yml
kubectl get pods -n ms
kubectl get svc -n ms
```

Update the image references in `deployment-service.yml` to `adarshbarkunta/<service>:latest` (or a specific `:<git-sha>` tag from a GitHub Actions run) to deploy the images built by CI. A GitHub Actions-based deploy job (targeting a live cluster via a `KUBE_CONFIG` secret) can be added here once a cluster is available.
