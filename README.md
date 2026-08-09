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

## CD: ArgoCD (only deployment path — points at this repo)

This repo is the single source of truth ArgoCD watches. The GitOps loop is:

**CI builds an image → CI commits the new tag into [deployment-service.yml](deployment-service.yml) on `main` → ArgoCD notices the commit and auto-syncs the cluster.**

No manual `kubectl apply` and no `helm install` — ArgoCD is the only thing that ever touches the cluster.

> **Image state**: verified working as of the deployment below — all 17 custom images build, push, and run correctly. See [Verified Live Deployment](#verified-live-deployment-eks--argocd) for the full record, including manifest bugs that had to be fixed to get there.

### 1. Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts
kubectl get pods -n argocd
kubectl edit svc argocd-server -n argocd    # change type: ClusterIP -> LoadBalancer
kubectl get svc -n argocd                   # note the LoadBalancer external IP
```
Note the repo is `argo-cd`, not `argocd` — a plain `kubectl apply` (without `--server-side`) on the same URL will also fail on the `applicationsets.argoproj.io` CRD ("metadata.annotations: Too long") because that CRD's rendered `last-applied-configuration` annotation exceeds Kubernetes' 256KB limit; `--server-side` avoids the annotation entirely.

Get the initial admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```
Username: `admin`

### 2. Create the Application (points at THIS repo)
Via the Web UI (`http://<ARGOCD_LOADBALANCER_IP>`) or `argocd` CLI, create an app with:
- **Application Name**: `opentelemetry-demo`
- **Project**: `default`
- **Repository URL**: `https://github.com/adarsh0331/Project_11_Opentelemetry_microservices-copy.git`
- **Revision**: `HEAD` (tracks `main`)
- **Path**: `.` (repo root — `deployment-service.yml` is the only manifest there)
- **Cluster URL**: `https://kubernetes.default.svc`
- **Namespace**: `ms`
- **Sync Policy**: enable **Auto-Sync** (+ **Auto-Create Namespace**) — this is what makes CI's image-tag commits actually redeploy without you touching anything

Equivalent CLI form:
```bash
argocd app create opentelemetry-demo \
  --repo https://github.com/adarsh0331/Project_11_Opentelemetry_microservices-copy.git \
  --revision main \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace ms \
  --sync-policy automated \
  --auto-prune --self-heal
```

### 3. Access the app
`frontend-proxy`'s Service is already set to `type: LoadBalancer` in `deployment-service.yml` — ArgoCD provisions the ELB automatically on sync. Get its address with:
```bash
kubectl -n ms get svc opentelemetry-demo-frontendproxy
# hit http://<EXTERNAL-IP or hostname>:8080
```
Don't `kubectl edit`/`kubectl patch` this Service directly — with `selfHeal: true` (set above), ArgoCD reverts any change that isn't committed to `deployment-service.yml` on the next reconcile. Change the `type:` (or anything else about it) in git instead.

### Observability (optional, separate ArgoCD app)
`deployment-service.yml` does **not** include Prometheus/Grafana/Jaeger. To add them, create a second ArgoCD Application (Helm source) pointing at `https://open-telemetry.github.io/opentelemetry-helm-charts`, chart `opentelemetry-demo`, rather than running `helm install` by hand — keeps everything under ArgoCD.

## Verified Live Deployment (EKS + ArgoCD)

This full CI → Docker Hub → GitOps → ArgoCD → EKS pipeline has been run end-to-end and confirmed working: **all 19 pods `1/1 Running` with zero restarts, ArgoCD reporting `Synced / Healthy`, app reachable through the LoadBalancer.** Getting there surfaced a few real issues worth recording so they don't get re-discovered the hard way.

### Instance type: what actually works on a free-tier-restricted AWS account
Some AWS accounts (e.g. newer accounts under AWS's revamped free tier) reject any `eksctl create nodegroup` with a non-free-tier instance type outright (`InvalidParameterCombination - The specified instance type is not eligible for Free Tier`). If you hit that:
- Check what your account actually allows: `aws ec2 describe-instance-types --filters "Name=free-tier-eligible,Values=true"` — the eligible list is broader than just `t2.micro`/`t3.micro`; it can include `t3.small`, `t4g.small`, `c7i-flex.large`, and **`m7i-flex.large`**.
- **Avoid `t3.micro`/`t4g.micro` for this app.** They cap out at ~4 pods/node (EKS's max-pods is derived from ENI/IP capacity, and these instances barely have any), so even ArgoCD alone (7 pods) won't schedule across 2 nodes, let alone the full 19-pod app.
- **`m7i-flex.large`** (2 vCPU, 8GB RAM, ~29 pods/node) is what this was actually deployed on and comfortably fits everything across 2 nodes.
- The EKS control plane itself (~$0.10/hr) is **never** free-tier eligible, regardless of node instance type — factor that in either way.

```bash
eksctl create cluster \
  --name otel-demo --region us-east-1 \
  --nodegroup-name workers --node-type m7i-flex.large \
  --nodes 2 --nodes-min 2 --nodes-max 3 --managed
```

### Manifest bugs found and fixed
`deployment-service.yml` was a `helm template` export that had partially drifted from a newer chart's env-var naming convention (`<SERVICE>_SERVICE_<PORT|ADDR>`) while the actual service source in `src/` (consolidated from the original per-service branches) reads the plain `<SERVICE>_<PORT|ADDR>` form. This crashed 9 of the 17 custom-built services on first real deploy — confirmed one at a time via live pod logs, not guessed:

| Service | Was | Fixed to | Symptom |
|---|---|---|---|
| accounting, checkout, fraud-detection | `KAFKA_SERVICE_ADDR` | `KAFKA_ADDR` | `KAFKA_ADDR is not supplied` / `ArgumentNullException` |
| checkout | `CHECKOUT_SERVICE_PORT` + 6 `*_SERVICE_ADDR` vars | `CHECKOUT_PORT` + `*_ADDR` | `panic: environment variable "CHECKOUT_PORT" not set` |
| currency | `CURRENCY_SERVICE_PORT` | `CURRENCY_PORT` | `Usage: currency <port>` |
| payment | `PAYMENT_SERVICE_PORT` | `PAYMENT_PORT` | `Failed to parse DNS address dns:0.0.0.0:undefined` |
| quote | `QUOTE_SERVICE_PORT` | `QUOTE_PORT` | `Invalid URI "tcp://0.0.0.0:"` |
| recommendation | `RECOMMENDATION_SERVICE_PORT`, `PRODUCT_CATALOG_SERVICE_ADDR` | `RECOMMENDATION_PORT`, `PRODUCT_CATALOG_ADDR` | `PRODUCT_CATALOG_ADDR environment variable must be set` |
| product-catalog | `PRODUCT_CATALOG_SERVICE_PORT` | `PRODUCT_CATALOG_PORT` | `Environment Variable Not Set: "PRODUCT_CATALOG_PORT"` |
| shipping | `SHIPPING_SERVICE_PORT` | `SHIPPING_PORT` | `$SHIPPING_PORT is not set: NotPresent` |
| ad | `AD_SERVICE_PORT` | `AD_PORT` | `IllegalStateException: environment vars: AD_PORT must not be null` |
| frontendproxy | `GRAFANA_SERVICE_HOST/PORT`, `JAEGER_SERVICE_HOST/PORT` | `GRAFANA_HOST/PORT`, `JAEGER_HOST/PORT` | envoy config parse failure (unresolved `${GRAFANA_HOST}` template placeholder), immediate exit 1, no explicit error text |
| frontend | `AD_SERVICE_ADDR`, `CHECKOUT_SERVICE_ADDR`, `RECOMMENDATION_SERVICE_ADDR` | `AD_ADDR`, `CHECKOUT_ADDR`, `RECOMMENDATION_ADDR` | didn't crash frontend itself, but its calls to those services would've failed |

All of these are already fixed on `main` — this table is a record of what happened, not a to-do list. If a future `deployment-service.yml` regeneration reintroduces the `_SERVICE_` naming, this is the pattern to search for (`grep -n "name: .*_SERVICE_PORT\|name: .*_SERVICE_ADDR"`, then cross-check each against `.env`'s `<SERVICE>_PORT`/`<SERVICE>_ADDR` values — excluding `OTEL_SERVICE_NAME`/`WEB_OTEL_SERVICE_NAME`, which are correct as-is).

### Cost reminder
A live EKS cluster on `m7i-flex.large` nodes costs real money continuously (control plane + EC2, roughly a few dollars/day). Tear it down when not actively using it:
```bash
eksctl delete cluster --name otel-demo --region us-east-1
```
