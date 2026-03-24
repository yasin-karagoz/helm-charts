# helm-charts

Wrapper Helm charts for the [DevOps Local Lab](https://github.com/yasin-karagoz/devops-local-lab) platform stack.

Each chart in `charts/` wraps an official upstream Helm chart and applies the configuration needed to run it on a local Kind cluster. They are used by Terraform (project 02 in `local-platform-hub`) and will be watched by ArgoCD for GitOps deployments.

---

## Charts

| Chart | Upstream | Version | Purpose |
|---|---|---|---|
| `nginx-ingress` | [ingress-nginx](https://kubernetes.github.io/ingress-nginx) | 4.15.1 | HTTP/HTTPS ingress controller |
| `cert-manager` | [cert-manager](https://charts.jetstack.io) | v1.15.5 | TLS certificate management |
| `argocd` | [argo-cd](https://argoproj.github.io/argo-helm) | 9.4.15 | GitOps continuous delivery |
| `kube-prometheus-stack` | [kube-prometheus-stack](https://prometheus-community.github.io/helm-charts) | 82.12.0 | Prometheus + Grafana + Alertmanager |

---

## How wrapper charts work

Each wrapper chart contains exactly two files:

```
charts/nginx-ingress/
├── Chart.yaml    ← declares the upstream chart as a dependency
└── values.yaml   ← your overrides for the upstream chart
```

`Chart.yaml` declares the upstream as a dependency:

```yaml
# charts/nginx-ingress/Chart.yaml
dependencies:
  - name: ingress-nginx
    version: "4.15.1"
    repository: "https://kubernetes.github.io/ingress-nginx"
```

`values.yaml` overrides only what you need. All keys must be prefixed with the chart name:

```yaml
# charts/nginx-ingress/values.yaml
ingress-nginx:
  controller:
    kind: DaemonSet
    hostPort:
      enabled: true
```

**Why this pattern?**
- You only maintain your overrides — not a full copy of the upstream chart
- Upstream security patches arrive by bumping the version in `Chart.yaml`
- The chart is self-contained and installable with a single `helm install`

---

## What each chart configures

### nginx-ingress

Configures the controller to run as a **DaemonSet** on the Kind control-plane node, which has ports 80 and 443 mapped to your Mac. This is how traffic from your browser reaches pods inside the cluster.

Key settings:
- `kind: DaemonSet` — ensures the controller lands on the control-plane node
- `hostPort.enabled: true` — binds to ports 80/443 on the node directly
- `nodeSelector: ingress-ready: "true"` — targets the Kind control-plane node
- `admissionWebhooks.enabled: false` — not needed locally, simplifies setup

### cert-manager

Manages TLS certificates inside the cluster. Combined with a `ClusterIssuer`, it automatically provisions certificates for any `Ingress` resource that requests one.

Key settings:
- `crds.enabled: true` — installs CustomResourceDefinitions automatically
- `replicaCount: 1` — single replica saves memory on a laptop

**Note:** Pinned to v1.15.5 for compatibility with Kubernetes v1.29.2. cert-manager v1.16+ requires Kubernetes v1.30.

### argocd

GitOps controller that watches a Git repository and keeps the cluster state in sync with what is committed.

Key settings:
- `server.insecure: true` — disables TLS on the ArgoCD server; nginx-ingress handles TLS termination instead
- `ingress.enabled: true` — exposes the UI at `https://argocd.localhost`
- `cert-manager.io/cluster-issuer: selfsigned-issuer` — auto-provisions a certificate
- `dex.enabled: false` — SSO not needed locally
- All components set to 1 replica — saves memory on a laptop

### kube-prometheus-stack

Full observability stack: Prometheus scrapes metrics, Grafana visualises them, Alertmanager handles alerts.

Key settings:
- Grafana exposed at `https://grafana.localhost` with nginx ingress
- Prometheus exposed at `https://prometheus.localhost`
- `serviceMonitorSelectorNilUsesHelmValues: false` — Prometheus scrapes all `ServiceMonitor` resources across all namespaces, not just ones created by this chart
- `retention: 7d` — keeps 7 days of metrics (appropriate for local use)
- `adminPassword: "admin"` — simple password for local use

---

## Installing charts

### Via Terraform (recommended)

Project 02 in `local-platform-hub` installs all charts via the Terraform Helm provider. See `projects/02-platform-stack/main.tf`.

### Manually with Helm

```bash
# From this repo root — update dependencies first
helm dependency update charts/nginx-ingress

helm install nginx-ingress charts/nginx-ingress \
  --namespace ingress-nginx \
  --create-namespace

helm install cert-manager charts/cert-manager \
  --namespace cert-manager \
  --create-namespace

helm install argocd charts/argocd \
  --namespace argocd \
  --create-namespace

helm install kube-prometheus-stack charts/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

---

## Repository structure

```
helm-charts/
├── charts/
│   ├── nginx-ingress/
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   ├── cert-manager/
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   ├── argocd/
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   └── kube-prometheus-stack/
│       ├── Chart.yaml
│       └── values.yaml
└── README.md
```

Upstream chart archives (`charts/*/charts/*.tgz`) are excluded from git via `.gitignore`. Run `helm dependency update charts/<name>` to download them locally.

---

## Upgrading a chart

To upgrade an upstream chart version:

1. Edit the `version` field in `charts/<name>/Chart.yaml`
2. Run `helm dependency update charts/<name>`
3. Test with `helm upgrade` or `terraform apply`
4. Commit `Chart.yaml` and `Chart.lock` — do not commit the `.tgz` files
