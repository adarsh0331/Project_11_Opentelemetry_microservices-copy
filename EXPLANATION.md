# File-by-File Explanation

This is a map of every file and directory in this repo and what it's for. [README.md](README.md) is the "how to deploy and operate this" guide; this file is the "what is this thing sitting in the repo" reference.

---

## Root files

| File | What it is |
|---|---|
| [`.env`](.env) | The **source of truth for image versions and env-var naming**. Defines which upstream image tag each infra component uses (`COLLECTOR_CONTRIB_IMAGE`, `JAEGERTRACING_IMAGE`, `PROMETHEUS_IMAGE`, `GRAFANA_IMAGE`, `FLAGD_IMAGE`, etc.) and documents the `<SERVICE>_PORT` / `<SERVICE>_ADDR` naming convention every app service actually expects. Not consumed directly by anything in this repo's deploy path (no `docker compose` here) — it's the reference used to catch/fix manifest bugs (see README's "Manifest bugs found and fixed" table) and to keep observability image versions in sync. |
| [`README.md`](README.md) | Step-by-step deployment guide: provision EKS, install ArgoCD, deploy, access the app and observability tools, set up CI, operate/roll back/tear down. |
| [`EXPLANATION.md`](EXPLANATION.md) | This file. |
| [`deployment-service.yml`](deployment-service.yml) | **The main Kubernetes manifest.** One file, multi-document YAML (`---`-separated), containing a `Deployment` + `Service` pair for each of the 18 app services (frontend, cartservice, checkoutservice, etc.). This is what ArgoCD actually syncs to run the app. CI rewrites the relevant `image:` line here after every successful build (GitOps). |
| [`observability.yml`](observability.yml) | The second Kubernetes manifest, same pattern as above but for the **observability stack**: OTel Collector (+ its config as a `ConfigMap`), Jaeger, Prometheus (+ config), Grafana (+ datasource provisioning config). Picked up automatically by the same ArgoCD Application as `deployment-service.yml` since both live at the repo root. |
| [`Jenkinsfile`](Jenkinsfile) | Declarative Jenkins pipeline — alternative to the GitHub Actions workflow below, same five stages (checkout, SonarQube scan, build+push, Trivy scan, GitOps manifest update), for teams running self-hosted Jenkins instead of/alongside GitHub Actions. |

## `.github/workflows/`

| File | What it is |
|---|---|
| [`.github/workflows/build-and-push.yml`](.github/workflows/build-and-push.yml) | The GitHub Actions CI pipeline. Detects which `src/<service>` directories changed on a push to `main` (or builds all 17 on manual dispatch), then per changed service: checkout → SonarQube scan → Docker build & push (Buildx, to Docker Hub) → Trivy vulnerability scan → commit the new image tag back into `deployment-service.yml`. |

## `src/` — one directory per microservice

18 services total. 17 have their own `Dockerfile` and are built as custom images by CI; `flagd` runs the upstream image unmodified. Every service directory also has its own `README.md` (from the upstream OpenTelemetry demo) with language-specific dev instructions — worth opening directly if you're changing that service's code.

| Service | Language | Role | Key files |
|---|---|---|---|
| [`accounting`](src/accounting/) | C# / .NET | Consumes checkout events from Kafka, does the "accounting" side-effect (no external API). | `accounting/Accounting.csproj` |
| [`ad`](src/ad/) | Java (Gradle) | Serves contextual ad content to the frontend, gRPC. | `build.gradle`, `src/main/java/...` |
| [`cart`](src/cart/) | C# / .NET | Maintains per-user shopping carts, backed by the Valkey cache. | `src/cart.csproj` (build context is `src/cart/src`, not `src/cart` — note for anyone touching CI) |
| [`checkout`](src/checkout/) | Go | Orchestrates the buy flow: calls shipping, currency, payment, email, fraud-detection; publishes to Kafka. Central service in the write path. | `main.go`, `go.mod` |
| [`currency`](src/currency/) | C++ | Converts prices between currencies. | `src/server.cpp`, `CMakeLists.txt` |
| [`email`](src/email/) | Ruby | Sends the order-confirmation email (logs it; no real SMTP in the demo). | `email_server.rb`, `Gemfile` |
| [`flagd`](src/flagd/) | N/A — upstream image | Feature-flag evaluation service; this directory holds only its config, not source. Image comes straight from `ghcr.io/open-feature/flagd` (version pinned in `.env`). | `demo.flagd.json` — the actual flag definitions |
| [`flagd-ui`](src/flagd-ui/) | Node/TS (Next.js) | Web UI for toggling the feature flags `flagd.json` defines, at `/feature` via frontend-proxy. | `package.json`, `next.config.mjs` |
| [`fraud-detection`](src/fraud-detection/) | Java (Gradle Kotlin DSL) | Consumes checkout events from Kafka, flags suspicious orders. | `build.gradle.kts` |
| [`frontend`](src/frontend/) | Node/TS (Next.js) | The web storefront UI. Calls ad, cart, checkout, currency, recommendation, product-catalog over gRPC from server-side code. | `index.js`, `pages/`, `components/`, `opentelemetry.js` (instrumentation bootstrap) |
| [`frontend-proxy`](src/frontend-proxy/) | Envoy (config only, no app code) | The single entry point / reverse proxy in front of everything (API gateway). Routes `/` → frontend, `/jaeger` → Jaeger, `/grafana` → Grafana, `/feature` → flagd-ui, `/images/` → image-provider, `/otlp-http/` → OTel Collector, `/loadgen` → load-generator's Locust web UI. | `envoy.tmpl.yaml` — the actual routing table; edit this to add/change routes |
| [`image-provider`](src/image-provider/) | nginx (config only) | Serves static product images. | `nginx.conf.template`, `static/` |
| [`load-generator`](src/load-generator/) | Python (Locust) | Simulates user traffic against the frontend continuously — this is what generates the trace/metric data you see in Jaeger/Prometheus. | `locustfile.py` — the actual user-behavior script |
| [`payment`](src/payment/) | Node/TS | Processes (fake) payment charges for checkout. | `charge.js`, `index.js` |
| [`product-catalog`](src/product-catalog/) | Go | Product listing/lookup; backing store for frontend and recommendation. | `main.go` |
| [`quote`](src/quote/) | PHP | Calculates shipping quotes for checkout. | `composer.json` |
| [`recommendation`](src/recommendation/) | Python | Suggests related products, calls product-catalog. | `recommendation_server.py` |
| [`shipping`](src/shipping/) | Rust | Calculates shipping cost/tracking for checkout. | `Cargo.toml`, `src/main.rs` (via `build.rs`) |

### Two services not in the table above
- **`cache` (Valkey)** and **`queue` (Kafka)** are infrastructure dependencies referenced by the architecture diagram in README, not custom-built services — they run from upstream images directly inside `deployment-service.yml`, with no corresponding `src/` directory.

---

## How the two manifests relate to `src/`

```
src/<service>/Dockerfile  --[CI builds & pushes]-->  docker.io/adarshbarkunta/<service>:<tag>
                                                              |
                                                    [CI writes the new tag]
                                                              v
                                                deployment-service.yml (image: ... line)
                                                              |
                                                   [ArgoCD watches this repo]
                                                              v
                                                    Deployment updated on EKS
```

`observability.yml` isn't touched by CI at all — its images are pinned upstream versions (from `.env`), not built from this repo's source.
