# Cloud-native automation architecture

[![CI](https://github.com/ahmedbadawy4/cloud-native-automation-architecture/actions/workflows/pr-checks.yml/badge.svg)](https://github.com/ahmedbadawy4/cloud-native-automation-architecture/actions/workflows/pr-checks.yml)

A reference setup for GitOps with Argo CD: a small web application is deployed to Kubernetes in two environments (dev and prod) with Helm, and a monitoring stack is installed next to it.
It shows deterministic environments, environment separation and Git-driven promotion without imperative cluster changes. It is a reference setup for local clusters, not a production system.

The repository was formerly named `sre-gitops`. The Argo CD application `sre-gitops-apps`, the default kind cluster and minikube profile name `sre-gitops`, and the `app.kubernetes.io/part-of: sre-gitops` label keep that name.

## Architecture (high level)

High-level flow:
```
Git Repo (app + Helm)
          |
          v
      Argo CD
          |
          v
 Kubernetes Cluster
   - web-app-dev
   - web-app-prod
   - monitoring (kube-prometheus-stack)
```

- Git repo holds application code, the Helm chart and the Argo CD definitions.
- Argo CD watches the repo and syncs desired state into the cluster.
- Two environments (dev/prod) are separated by namespaces, Argo CD projects and values files.
- The `monitoring` application installs Prometheus and Grafana. The chart renders a ServiceMonitor so Prometheus scrapes both environments.

## What this demonstrates

- GitOps deployment using Argo CD (app-of-apps pattern).
- Pinned Kubernetes node image, Argo CD chart and kube-prometheus-stack versions. The Traefik chart is pinned when `TRAEFIK_CHART_VERSION` is set.
- Dev/Prod separation via namespaces, Argo CD projects and Helm values.
- Git-based promotion and rollback.
- Metrics scraped through a ServiceMonitor, a Grafana dashboard, and basic reliability controls.

## How to review this repo

Suggested review order:
1. README (this file) for system intent and workflow.
2. Makefile for reproducible bootstrap and operational commands.
3. deploy/argocd/ for GitOps application and project definitions.
4. charts/ for environment-specific configuration and reliability settings.
5. .github/workflows/ for CI-driven image build and promotion.

This repository is intentionally scoped to core GitOps mechanics rather than full production hardening. See [Limits](#limits).

## Prereqs

Required:
- One Kubernetes option: Docker Desktop, kind, minikube, or an existing cluster.
- `kubectl`, `docker`, `helm`.
- Internet access for the chart fetches (Argo CD, Traefik, kube-prometheus-stack).

Optional:
- `pre-commit` (and `hadolint`) for local linting.

## Cluster setup (pinned versions)

Choose one of the following so all subsequent steps target the right cluster:

```bash
make k8s-kind-up KIND_CLUSTER=sre-gitops
make k8s-minikube-up MINIKUBE_PROFILE=sre-gitops
make k8s-use-context K8S_CONTEXT=<existing-context>
```

Pinned Kubernetes versions (default, overrideable):

- Kind defaults to `KIND_NODE_IMAGE=kindest/node:v1.35.0` (set in `Makefile`).
- Minikube defaults to `MINIKUBE_K8S_VERSION=v1.35.0` (set in `Makefile`).

You can override these defaults if required for your environment. The defaults are pinned to keep the cluster reproducible for reviewers.

## Quickstart (Makefile-first)

After this section, Argo CD, monitoring, and both environments are running.

Reviewer-friendly shortcuts:

```bash
# Kind: build the app locally, load it into kind, no GHCR access required.
make bootstrap-kind

# Minikube: build the app locally, load it into minikube, no GHCR access required.
make bootstrap-minikube
```

The values files pin one image tag per environment (`main` for dev, the release tag for prod). `kind-images` and `minikube-images` build the app once and tag the build with both, so the local cluster starts both environments from that build.

The same steps one by one:

```bash
# Verify Docker is installed and running.
make docker-check
# Verify kubectl context.
make k8s-check
# Build the dev and prod tags locally and load them into the cluster.
make kind-images          # or: make minikube-images
# Alternatively build and push multi-arch to GHCR (requires docker login to ghcr.io).
make docker-build-image
# Install Traefik ingress controller (if not already present).
# Set TRAEFIK_CHART_VERSION=<version> to pin the chart.
make traefik-install
# Install Argo CD via Helm using tools/argocd-values.yaml.
make argocd-install
# Deploy Argo CD apps (app-of-apps).
make argocd-deploy-apps
# Port-forward Argo CD locally.
make argocd-url
```

The web-app charts render a ServiceMonitor, and its CRD comes from the `monitoring` application. On the first sync the two web-app applications can report a failed sync until the CRD exists; their `retry` policy syncs them again.

## Expected results

Successful setup is indicated by all Argo CD applications being Healthy and Synced, by `/environment` returning `dev` or `prod`, and by `/version` returning the image tag the environment runs.

Argo CD Applications should show all four apps (monitoring, sre-gitops-apps, web-app-dev, web-app-prod) Healthy and Synced. The screenshots below were taken before the repository rename, so they show the earlier repository name and the `default` project on the web-app applications; they now belong to the `web-app-dev` and `web-app-prod` projects.

![Argo CD applications](docs/expected-results/argocd-applications.png)

App URLs (port-forward):

```bash
make app-urls
```

Web app ingress (default cert):

- Dev: `https://web-app-dev.local` (add `127.0.0.1 web-app-dev.local` to `/etc/hosts`).
- Prod: `https://web-app-prod.local` (add `127.0.0.1 web-app-prod.local` to `/etc/hosts`).

Expected pages (the version shown is the tag of the image that is running):

![Web app dev](docs/expected-results/web-app-dev.png)

![Web app prod](docs/expected-results/web-app-prod.png)

Monitoring (Grafana + Prometheus):

```bash
make monitoring-urls
```

In Prometheus (Status, Targets) each environment shows a `serviceMonitor/web-app-dev/web-app/0` or `serviceMonitor/web-app-prod/web-app/0` target. The metric is `sre_app_requests_total`.

Grafana login: `admin` and the password stored in the Secret:

```bash
kubectl -n monitoring get secret monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

Dashboard: in Grafana choose Dashboards, New, Import, upload `tools/grafana-web-app-dashboard.json` and pick the Prometheus data source. It shows the request rate per path and the number of scrape targets that are up, per namespace.

## Argo CD access

Local: `make argocd-url` (port-forward) or `https://argocd.local` via ingress (add `127.0.0.1 argocd.local` to `/etc/hosts`).

For a shared cluster, use a real DNS name with valid TLS (e.g., cert-manager + Let’s Encrypt) and configure ingress settings in `tools/argocd-values.yaml`.

## GitOps promotion and rollback

- On push to `main`, CI builds and pushes `ghcr.io/ahmedbadawy4/cloud-native-automation-architecture:main`. Dev runs that tag.
- On Git tag push (e.g., `v0.0.4`), the release workflow builds and pushes that tag, then commits the new tag into `charts/values-prod.yaml` on `main`. Prod runs that tag.
- Argo CD Helm chart version is pinned via `ARGOCD_CHART_VERSION` in `Makefile`.

Production changes are auditable, reversible, and driven through Git history rather than imperative cluster actions.
If a bad image is promoted, the blast radius is limited to the target environment, and recovery is a Git revert.

Rollback (GitOps):

- Revert the commit that bumped `charts/values-prod.yaml` and push.
- Argo CD auto-sync will reconcile back to the previous image tag.
- Validate by checking `argocd app get web-app-prod` (Synced/Healthy), and by hitting the `/version` endpoint to confirm the running version.
- If a rollout is flapping, pause auto-sync temporarily, fix the values file, then re-enable auto-sync to resume reconciliation.

## Reliability and security

Reliability:
- Readiness and liveness probes are configured in the Helm chart.
- Resource requests/limits are set per environment in `charts/values-*.yaml`.
- Autoscaling (HPA) is enabled in dev/prod via `autoscaling` values (requires `metrics-server`, which this repository does not install).
- The Argo CD applications for the web app retry a failed sync with backoff.

Observability:
- `/metrics` endpoint exposes basic Prometheus-style counters.
- The `monitoring` application deploys `kube-prometheus-stack` through Argo CD.
- The chart renders a ServiceMonitor (`serviceMonitor.enabled`, on in dev and prod) labelled `release: monitoring`, which the Prometheus of that stack selects. The `prometheus.io/*` pod annotations are kept for annotation-based scrapers; kube-prometheus-stack does not use them.
- `tools/grafana-web-app-dashboard.json` is a dashboard for the app metrics (import steps above).

Security:
- Argo CD RBAC defaults to readonly; admin is explicitly mapped in `tools/argocd-values.yaml`.
- Argo CD Projects (`web-app-dev`, `web-app-prod`) restrict repos and destinations to enforce environment boundaries.
- The pod runs as a non-root user with a read-only root filesystem, dropped capabilities and the `RuntimeDefault` seccomp profile (`charts/web-app/values.yaml`).
- Web app sets basic security headers (CSP, X-Frame-Options, etc.) in `app/httpd.conf`.

Not included:
- Least-privileged service accounts for the app (currently uses the default SA).
- Secrets management integration (see below).

## Secrets strategy (design only)

The web app needs no secrets, so this repository manages none. For an application that does, the intended approach is to store runtime secrets in a secret manager (e.g., AWS Secrets Manager/Vault) and sync them with External Secrets, with only the `ExternalSecret` manifest in Git. Nothing here implements that.

## Production safeguards

- The release workflow runs in the `production` environment of GitHub Actions, which only allows deployments from Git tags, to reduce human error.
- A manual approval step is added by setting required reviewers on that environment in the repository settings.

## Design decisions

- Helm chosen for app templating to keep environment differences explicit and versioned.
- App-of-apps used to scale environments cleanly under a single Argo CD parent.
- Git-based promotion chosen over in-cluster automation to keep changes auditable.
- Kubernetes node image, Argo CD chart and kube-prometheus-stack are pinned for reproducibility while remaining overrideable.
- No service mesh or progressive delivery tooling (e.g., Argo Rollouts) to keep the scope focused on core GitOps primitives.

## Assumptions and tradeoffs

- Uses local clusters (kind/minikube/Docker Desktop) for simplicity.
- Argo CD is installed via Helm to keep installation declarative and reproducible.
- Prod's image tag is written into `charts/values-prod.yaml` by the release workflow; dev follows the moving `main` tag.
- Argo CD Image Updater is intentionally not used to keep all state changes Git-driven.

## Private repo credentials (optional)

If the repo is private, Argo CD needs credentials. One approach is to create a repo secret:

```bash
kubectl -n argocd create secret generic private-repo \
  --from-literal=type=git \
  --from-literal=url=https://github.com/ORG/REPO.git \
  --from-literal=username=GIT_USERNAME \
  --from-literal=password=GIT_TOKEN

kubectl -n argocd label secret private-repo \
  argocd.argoproj.io/secret-type=repository
```

## Cleanup

```bash
# Stop local port-forward processes and remove temp pid/log files.
make pf-stop
# Delete Argo CD Application objects so Argo CD stops reconciling them.
make apps-delete
# Delete only this repo's apps and namespaces (safe for shared clusters).
make cleanup
```

## PR checks and local linting

GitHub Actions runs these checks on pull requests:

- The image name in `charts/web-app/values.yaml`, `charts/values-dev.yaml`, `charts/values-prod.yaml` and `IMAGE_NAME` in the `Makefile` equals `ghcr.io/<owner>/<repository>`, the name the workflows publish to.
- `helm lint` for the default, dev and prod values, and `helm template` for dev and prod.
- Dockerfile lint (Hadolint).
- pre-commit (YAML syntax, end of file, trailing whitespace, `helm lint`).

```bash
pre-commit install
pre-commit run --all-files
```

Useful targets:

```bash
# Sync Argo CD apps (requires argocd CLI).
make argocd-sync
# Show namespaces and Argo CD apps.
make status
# Run lint checks locally.
make lint
```

## Bootstrap and configuration

- Argo CD installation is via Helm with pinned chart version and values in `tools/argocd-values.yaml`.
- Applications and projects are defined as YAML in `deploy/argocd/applications.yaml` and `deploy/argocd/apps/*.yaml`.
- After `make argocd-deploy-apps`, Argo CD continuously syncs from Git. The only step outside Git is the optional Grafana dashboard import.

## Limits

- The Argo CD projects allow every cluster-scoped resource kind (`clusterResourceWhitelist` is `*`). Dev and prod separation is by namespace and project destination inside one cluster, not by cluster.
- Releases need no manual approval: the `production` environment limits deployments to tags and has no required reviewers. The release workflow pushes the tag bump straight to `main`, so branch protection on `main` would need a bypass for that push.
- Dev runs the moving tag `main` with `imagePullPolicy: IfNotPresent`, so a node keeps the first `main` image it pulled. Set `image.pullPolicy: Always` for a registry-backed cluster that should follow new builds; that setting does not work with images loaded into kind or minikube.
- The chart has no `imagePullSecrets`, so the GHCR package must be public, or the images must be loaded into the cluster with `make kind-images` or `make minikube-images`.
- The image name follows the repository name. A repository rename creates a new GHCR package, which needs a new push and its own visibility setting; the PR check above flags a mismatch.
- The Traefik chart is installed unpinned unless `TRAEFIK_CHART_VERSION` is set.
- `metrics-server` is not installed, so the HPA reports unknown CPU until it is.
- Argo CD runs with `server.insecure` (plain HTTP behind the ingress) and the built-in admin user, and has no SSO.
- The request counters are kept by the CGI scripts in a file under `/tmp` of each pod. They reset when a pod restarts and concurrent requests can lose counts.

## TODOs (production hardening)

- Enable SSO or OIDC for Argo CD and disable the default admin user.
- Enforce TLS on Argo CD server and restrict network access (ingress + firewall).
- Define least-privilege RBAC for users and automation.
- Add centralized logging and alerts for sync failures.
- Add alerting rules for Prometheus/Grafana.
