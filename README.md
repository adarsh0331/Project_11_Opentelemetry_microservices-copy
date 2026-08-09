# Micro Services Project — OpenTelemetry Astronomy Shop Demo

A microservice-based e-commerce app (18 services, 7 languages) instrumented end-to-end with OpenTelemetry, deployed to EKS via ArgoCD (GitOps), with CI in GitHub Actions and Jenkins.

**Looking for what a specific file does?** → [EXPLANATION.md](EXPLANATION.md) catalogs every file in this repo.
**Looking to deploy this?** → keep reading, it's a straight top-to-bottom walkthrough.

---

## Architecture

<img width="1166" height="1044" alt="image" src="https://github.com/user-attachments/assets/9ce5f0d0-c669-4a5a-a941-07c3937bed42" />

18 services talking over gRPC (internal calls), HTTP (user-facing), and Kafka (async events). `frontend-proxy` (Envoy) is the single entry point, fanning out to `frontend` (web UI), `flagd-ui` (feature-flag admin), and `image-provider` (static assets). `checkout` is the central write-path service, calling `shipping`, `quote`, `email`, `currency`, `payment`, and `fraud-detection`, then publishing to Kafka for `accounting` and `fraud-detection` to consume asynchronously. `flagd` serves feature flags to `cart`, `checkout`, `fraud-detection`, and `payment`. Every service exports OpenTelemetry traces/metrics/logs to a central collector, which fans out to Jaeger (traces) and Prometheus (metrics), both visualized in Grafana.

Full per-service breakdown (language, role, files): [EXPLANATION.md](EXPLANATION.md).

---

## Step-by-Step: Deploy This From Scratch

Everything below assumes you're deploying to AWS EKS. Total time: ~30-40 minutes, most of it waiting on `eksctl`.

### Step 1 — Provision an EKS cluster

On an EC2 instance or any machine with outbound internet access and the AWS CLI configured:

```bash
# aws cli
sudo apt update && sudo apt install unzip curl -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
aws configure   # access key, secret key, region, output format

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -sSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# eksctl
curl -sSL "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" -o eksctl.tar.gz
tar -xzf eksctl.tar.gz && sudo mv eksctl /usr/local/bin/
```

Then create the cluster:

```bash
eksctl create cluster \
  --name otel-demo \
  --region us-east-1 \
  --nodegroup-name workers \
  --node-type m7i-flex.large \
  --nodes 2 --nodes-min 2 --nodes-max 3 \
  --managed
```

Takes ~15-20 minutes. **`--node-type` matters — don't substitute a cheaper-looking instance without reading [why `m7i-flex.large`](#instance-type-what-actually-works-on-a-free-tier-restricted-aws-account) first**; `t3.micro` physically cannot schedule enough pods for this app regardless of node count.

Verify:
```bash
eksctl get cluster --region us-east-1
kubectl get nodes
```

### Step 2 — Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts
kubectl get pods -n argocd                  # wait until all Running
kubectl edit svc argocd-server -n argocd    # change type: ClusterIP -> LoadBalancer
kubectl get svc -n argocd                   # note the LoadBalancer address
```
`--server-side --force-conflicts` is required — a plain `kubectl apply` fails on the `applicationsets.argoproj.io` CRD because its rendered annotation exceeds Kubernetes' 256KB limit.

Get the admin password (username is `admin`):
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

### Step 3 — Point ArgoCD at this repo

This one Application deploys **both** manifests at once — `deployment-service.yml` (the app) and `observability.yml` (Jaeger/Prometheus/Grafana/OTel Collector) — because both live at the repo root and the Application's `path` is `.`.

Via `argocd` CLI:
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
Or via the Web UI (`http://<ARGOCD_LOADBALANCER_ADDRESS>`): **New App** → Repository URL = the URL above, Revision = `HEAD`, Path = `.`, Cluster = `https://kubernetes.default.svc`, Namespace = `ms`, enable **Auto-Sync** + **Auto-Create Namespace**.

Wait for it to sync:
```bash
kubectl -n ms get pods    # expect 23 pods, all 1/1 Running, within a couple minutes
```

> **No manual `kubectl apply`, no `helm install` — ArgoCD is the only thing that ever touches the cluster.** `selfHeal: true` means any live `kubectl edit` you make gets reverted on the next reconcile. Changes belong in git (`deployment-service.yml` / `observability.yml`), not `kubectl`.

### Step 4 — Access the app

```bash
kubectl -n ms get svc opentelemetry-demo-frontendproxy
# hit http://<EXTERNAL-IP or hostname>:8080
```

### Step 5 — Access the observability tools

Jaeger and Grafana are reachable through the **same LoadBalancer** as the app — no extra cost, they're pre-wired into `frontend-proxy`'s routing table:
```
http://<same-LB-address-as-step-4>:8080/jaeger
http://<same-LB-address-as-step-4>:8080/grafana
```
Prometheus has no route through the proxy (by upstream design — it's meant to be queried via Grafana), so reach it with a tunnel instead:
```bash
kubectl -n ms port-forward svc/opentelemetry-demo-prometheus 9090:9090   # http://localhost:9090
```

### Step 6 — (Optional) Turn on CI so future commits deploy automatically

Right now the cluster is running whatever's already committed in `deployment-service.yml`. To make `git push` → new image → auto-redeploy actually happen, set up one of the two CI pipelines below (GitHub Actions or Jenkins) — see [CI Pipelines](#ci-pipelines).

### Step 7 — Tear down when you're done

```bash
eksctl delete cluster --name otel-demo --region us-east-1
```
The EKS control plane bills ~$0.10/hr regardless of node type, and `m7i-flex.large` nodes add more on top — don't leave this running idle.

---

## CI Pipelines

Two independent, equivalent pipelines — use whichever fits your setup.

### GitHub Actions

[.github/workflows/build-and-push.yml](.github/workflows/build-and-push.yml) runs on every push to `main` that touches `src/**` (or via manual **Run workflow** dispatch, which builds all 17 services regardless of what changed).

**Job structure — this is why the Actions UI looks like it's "just" `changes` + `build`:** GitHub Actions only shows *jobs* as top-level rows on a run's summary page. This workflow has exactly two jobs — `changes` (detects which services touched `src/**`) and `build` (one instance per changed service, run in parallel as a matrix). SonarQube, Trivy, and the manifest update are **steps inside each `build (<service>)` job**, not separate jobs — click into any `build (<service>)` row to see them. In execution order, per service:

| # | Step | What it does |
|---|---|---|
| 1 | Checkout | `actions/checkout@v4` |
| 2 | **SonarQube Scan** | `sonar-scanner` against the self-hosted server below, `continue-on-error: true` (report-only) |
| 3 | Docker Buildx setup | `setup-qemu-action` + `setup-buildx-action` |
| 4 | Docker Hub Login | `docker/login-action` |
| 5 | **Build & Push** | builds that service's image, pushes `:latest` + `:<git-sha>` to Docker Hub |
| 6 | **Trivy Vulnerability Scan** | scans the image just pushed, `exit-code: 0` (report-only) |
| 7 | Upload Trivy results | SARIF → repo's **Security → Code scanning** tab |
| 8 | **Update deployment-service.yml** | rewrites that service's `image:` line, commits + pushes to `main` (GitOps — ArgoCD picks this up automatically) |

Required repo secrets (**Settings → Secrets and variables → Actions**):
- `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` — Docker Hub push access ([access token](https://hub.docker.com/settings/security), not your password).
- `SONAR_HOST_URL` / `SONAR_TOKEN` — already set, pointing at the self-hosted SonarQube server below. `continue-on-error: true` keeps this report-only regardless.

### SonarQube server (self-hosted, EC2)
SonarQube Community Edition runs as a single Docker container on its own EC2 instance (`sonarqube-server`, `m7i-flex.large`, tag `Name=sonarqube-server`) — kept off the EKS cluster deliberately since the cluster was already at 63-70% memory and SonarQube's Elasticsearch-backed indexer wants 2GB+ of headroom on its own.

```bash
# provisioned via SSM (no SSH key needed) - to check on it:
aws ssm start-session --target <instance-id> --region us-east-1

# the container itself:
docker ps                 # container name: sonarqube, port 9000
docker logs sonarqube
```
Bootstrap did three things beyond `docker run sonarqube:lts-community`: bumped `vm.max_map_count` (Elasticsearch bootstrap check requirement), rotated the default `admin/admin` password on first boot via the REST API, and generated a CI token — both stored only in the `SONAR_TOKEN` GitHub secret, never committed.

Each service scans as its own project, keyed `otel-demo-<service>` (matching the `-Dsonar.projectKey` arg in the workflow) — browse them at `http://<instance-public-ip>:9000`. One gotcha specific to this setup: CI runs `sonar-scanner` directly against source, without a full compile step first, so Java services with raw `.java` files (only `ad`; `fraud-detection` is Kotlin and unaffected) need `-Dsonar.java.binaries=.` supplied as a placeholder or the scan hard-fails with `AnalysisException: ... please provide compiled classes`. Already added to both pipelines' scan args.

> **Security note**: the instance's security group allows `9000`/`22` from `0.0.0.0/0` for demo convenience, matching this repo's other public endpoints (ArgoCD, frontend LB). Restrict to your IP if this stops being a throwaway demo. Costs the same as an EKS worker node (~$0.10/hr on `m7i-flex.large`) — terminate it when not in use: `aws ec2 terminate-instances --instance-ids <instance-id> --region us-east-1`.

### Jenkins (alternative)

[Jenkinsfile](Jenkinsfile) implements the same pipeline for a Jenkins job pointed at `main`. Unlike GitHub Actions, Jenkins' Blue Ocean / stage view shows every stage below by name directly — no clicking into a job to find them. Top-level stages, in order:

| # | Stage | What it does |
|---|---|---|
| 1 | `Checkout` | pulls `main` |
| 2 | `Detect Changed Services` | diffs `src/<service>` dirs since the last commit, or uses the explicit `SERVICES_OVERRIDE` build parameter |
| 3 | `Build, Scan & Deploy Services` | parent stage; per changed service, runs the four nested stages below |

Nested per-service, repeated for each detected service:

| # | Stage | What it does |
|---|---|---|
| 3a | `SonarQube: <service>` | `sonar-scanner` container against the self-hosted server below (`catchError` keeps it report-only) |
| 3b | `Build & Push: <service>` | `docker build` + push via `withDockerRegistry` |
| 3c | `Trivy Scan: <service>` | Trivy container scan |
| 3d | `Update Manifest: <service>` | rewrites `deployment-service.yml`'s `image:` line, commits + pushes to `main` |

SonarQube and Trivy run as Docker containers, so nothing extra needs installing on the agent beyond Docker itself.

Jenkins credentials required (**Manage Jenkins → Credentials**):
- `docker-cred` (Username/password) — Docker Hub push access.
- `github-cred` (Username/password: GitHub username + PAT with repo write access) — pushes the manifest update.
- `sonar-token` (Secret text) — SonarQube auth token.

Plus a `SONAR_HOST_URL` global environment variable (**Manage Jenkins → System**).

---

## Operating This Deployment

### Rolling back a single service's image
`deployment-service.yml` is the source of truth, so a rollback happens in git, not `kubectl`: edit that service's `image:` line back to a known-good tag (Docker Hub keeps every `:<git-sha>` tag CI ever pushed — check `docker.io/adarshbarkunta/<service>/tags`), commit, push. ArgoCD picks it up on the next auto-sync, or force it immediately:
```bash
kubectl -n argocd patch application opentelemetry-demo --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'
```
Example already done in this repo's history: `frontend` was rolled back from a bad build to the last verified-healthy tag this way.

### Observability stack details
[observability.yml](observability.yml) wires: every service's OTLP telemetry → OTel Collector → (`memory_limiter` + `batch` processors) → Jaeger (traces) + `spanmetrics` connector (derives RED metrics from spans) → Prometheus (scrapes the collector) → Grafana (both wired as datasources). Image versions are pinned to match `.env` exactly. The collector config is deliberately simpler than upstream's current `main`-branch version, which uses experimental "profiles" pipelines and receivers (Docker/Postgres/Redis) that don't apply to this K8s-only setup.

Getting Jaeger/Grafana working through the shared LoadBalancer (Step 5 above) required telling each app it's mounted under a subpath, not root — otherwise the index page loads but every asset request 404s:
- Jaeger: `QUERY_BASE_PATH=/jaeger` env var.
- Grafana: `GF_SERVER_SERVE_FROM_SUB_PATH=true` + `GF_SERVER_ROOT_URL=%(protocol)s://%(domain)s:8080/grafana/`.

---

## Verified Live Deployment (reference / troubleshooting log)

This full CI → Docker Hub → GitOps → ArgoCD → EKS pipeline has been run end-to-end and confirmed working: all 23 pods `1/1 Running`, zero restarts, ArgoCD `Synced / Healthy`, app + Jaeger + Grafana reachable through the LoadBalancer, traces/metrics confirmed flowing. Getting there surfaced real issues, recorded here so they don't get re-discovered the hard way.

### Instance type: what actually works on a free-tier-restricted AWS account
Some AWS accounts reject any `eksctl create nodegroup` with a non-free-tier instance type outright (`InvalidParameterCombination - not eligible for Free Tier`). If you hit that:
- Check what your account actually allows: `aws ec2 describe-instance-types --filters "Name=free-tier-eligible,Values=true"` — the eligible list is broader than `t2.micro`/`t3.micro`; it can include `t3.small`, `t4g.small`, `c7i-flex.large`, and **`m7i-flex.large`**.
- **Avoid `t3.micro`/`t4g.micro` for this app.** They cap out at ~4 pods/node (EKS's max-pods is derived from ENI/IP capacity, not CPU/memory), so even ArgoCD alone (7 pods) won't schedule across 2 nodes.
- **`m7i-flex.large`** (2 vCPU, 8GB RAM, ~29 pods/node) is what this was actually deployed on and comfortably fits all 23 pods across 2 nodes.
- The EKS control plane itself (~$0.10/hr) is never free-tier eligible, regardless of node type.

### Manifest bugs found and fixed
`deployment-service.yml` was a `helm template` export that had partially drifted from a newer chart's env-var naming convention (`<SERVICE>_SERVICE_<PORT|ADDR>`) while the actual service source in `src/` reads the plain `<SERVICE>_<PORT|ADDR>` form (confirmed against `.env`). This crashed 9 of 17 custom services on first real deploy — diagnosed one at a time via live pod logs:

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
| frontendproxy | `GRAFANA_SERVICE_HOST/PORT`, `JAEGER_SERVICE_HOST/PORT` | `GRAFANA_HOST/PORT`, `JAEGER_HOST/PORT` | envoy config parse failure, immediate exit 1 |
| frontend | `AD_SERVICE_ADDR`, `CHECKOUT_SERVICE_ADDR`, `RECOMMENDATION_SERVICE_ADDR` | `AD_ADDR`, `CHECKOUT_ADDR`, `RECOMMENDATION_ADDR` | didn't crash frontend itself, but calls to those services would've failed |

All already fixed on `main` — this table is a record, not a to-do list. If a future `deployment-service.yml` regeneration reintroduces `_SERVICE_` naming: `grep -n "name: .*_SERVICE_PORT\|name: .*_SERVICE_ADDR"`, cross-check each against `.env`'s convention (excluding `OTEL_SERVICE_NAME`/`WEB_OTEL_SERVICE_NAME`, which are correct as-is).

### loadgenerator OOMKilled under sustained load
After ~50 minutes of continuous load, `loadgenerator` was OOMKilled — `1500Mi` wasn't enough once Locust had accumulated enough in-memory stats. Fixed: raised to `3000Mi`.

### Observability stack OOM-crash-looping under sustained load
Two separate issues surfaced after the observability stack had been running under continuous `loadgenerator` traffic for a while:
- **OTel Collector & Jaeger OOMKilled** — 300Mi/500Mi limits were too low, and the collector's `memory_limiter` was configured with `limit_percentage` (relative to the *node's* total memory — useless against a much smaller container cgroup limit that gets hit first). Fixed: switched to absolute `limit_mib`/`spike_limit_mib` below the container limit, and raised both containers' limits (otelcol → 512Mi, Jaeger → 1000Mi + capped its in-memory trace storage to 50k traces via `MEMORY_MAX_TRACES`).
- **Prometheus OOM-crash-looping** (8 restarts in under an hour, worsening each time) — root cause was `resource_to_telemetry_conversion.enabled: true` on the collector's Prometheus exporter, which promotes *every* OTLP resource attribute into a Prometheus label on every metric. Under continuous varying traffic this caused unbounded time-series cardinality growth. Fixed: disabled that setting (not needed for the RED-metric dashboards this stack supports) and gave Prometheus headroom (1000Mi → 1500Mi) to recover cleanly.

### Cost reminder
A live EKS cluster on `m7i-flex.large` nodes costs real money continuously (control plane + EC2, roughly a few dollars/day), and the standalone SonarQube EC2 instance adds another `m7i-flex.large` on top of that. Tear both down when not actively using them:
```bash
eksctl delete cluster --name otel-demo --region us-east-1
aws ec2 terminate-instances --instance-ids <sonarqube-instance-id> --region us-east-1
```
