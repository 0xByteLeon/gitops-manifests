# GitOps Multi-Environment Infrastructure

Production-grade GitOps repository supporting **Dev**, **QA**, and **Prod** environments with ArgoCD, Helm, Kustomize, and OTEL LGTM observability stack.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              GitOps Repository                               │
├─────────────────┬─────────────────────────┬─────────────────────────────────┤
│    argocd/      │        apps/            │          platform/              │
│  (ArgoCD Config)│   (Business Apps)       │    (Infrastructure: LGTM)       │
└────────┬────────┴───────────┬─────────────┴──────────────┬──────────────────┘
         │                    │                            │
         ▼                    ▼                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ArgoCD (App-of-Apps)                                │
│   Watches Git repo and syncs desired state to Kubernetes clusters           │
└─────────────────┬─────────────────────────┬─────────────────────────────────┘
                  │                         │
    ┌─────────────┴─────────────┐ ┌─────────┴─────────────────────┐
    ▼             ▼             ▼ ▼             ▼                 ▼
┌────────┐  ┌────────┐  ┌────────┐ ┌────────┐  ┌────────┐  ┌────────┐
│  Dev   │  │   QA   │  │  Prod  │ │  Dev   │  │   QA   │  │  Prod  │
│Cluster │  │Cluster │  │Cluster │ │  LGTM  │  │  LGTM  │  │  LGTM  │
└────────┘  └────────┘  └────────┘ └────────┘  └────────┘  └────────┘
```

## Directory Structure

```
gitops-manifests/
├── README.md                    # This file
├── .gitignore
├── argocd/                      # ArgoCD configuration
│   ├── base/                    # ArgoCD deployment base
│   ├── overlays/                # Environment-specific ArgoCD config
│   │   ├── dev/
│   │   ├── qa/
│   │   └── prod/
│   └── apps/                    # App-of-Apps definitions
│       ├── root-app.yaml        # Bootstrap application
│       ├── dev-apps.yaml
│       ├── qa-apps.yaml
│       └── prod-apps.yaml
├── apps/                        # Business applications
│   └── my-app/
│       ├── base/                # Helm Chart + base config
│       └── overlays/            # Environment overrides
│           ├── dev/
│           ├── qa/
│           └── prod/
└── platform/                    # Infrastructure components (LGTM)
    ├── otel-collector/          # OpenTelemetry Collector
    ├── grafana/                 # Visualization
    ├── loki/                    # Log storage
    ├── tempo/                   # Trace storage
    └── prometheus/              # Metrics storage
```

## Quick Start

### Prerequisites

- Kubernetes clusters (Dev, QA, Prod)
- `kubectl` configured for target clusters
- ArgoCD CLI installed

### 1. Install ArgoCD (First Time Only)

```bash
# For each cluster
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Bootstrap with App-of-Apps

```bash
# Apply the root application to start the bootstrap process
kubectl apply -f argocd/apps/root-app.yaml
```

### 3. Access ArgoCD UI

```bash
# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Environment Configuration

| Environment | Cluster | Namespace | Replicas | Resources |
|-------------|---------|-----------|----------|-----------|
| Dev | dev-cluster | dev | 1 | Low |
| QA | qa-cluster | qa | 2 | Medium |
| Prod | prod-cluster | prod | 3+ | High |

## OTEL LGTM Stack

The observability stack consists of:

| Component | Purpose | Helm Chart |
|-----------|---------|------------|
| **OpenTelemetry Collector** | Telemetry data collection | `open-telemetry/opentelemetry-collector` |
| **Grafana** | Visualization & dashboards | `grafana/grafana` |
| **Loki** | Log aggregation | `grafana/loki` |
| **Tempo** | Distributed tracing | `grafana/tempo` |
| **Prometheus** | Metrics storage | `prometheus-community/kube-prometheus-stack` |

### Data Flow

```
Applications ──OTLP──▶ OTel Collector ──┬──metrics──▶ Prometheus ──┐
                                        ├──logs────▶ Loki ────────┼──▶ Grafana
                                        └──traces──▶ Tempo ───────┘
```

## CI/CD Workflow

1. **CI (GitHub Actions)**: Build → Test → Push Image → Update GitOps Repo
2. **CD (ArgoCD)**: Detect Git changes → Sync to Kubernetes

### Image Tag Update Strategy

- **Dev**: Auto-commit on merge to `main`
- **QA**: Create PR for review
- **Prod**: Create PR with approval required

## Configuration Management

This repository uses **Kustomize overlays on top of Helm charts** for maximum flexibility:

- `base/`: Contains Helm chart references and default values
- `overlays/{env}/`: Environment-specific patches and value overrides

### Example: Customizing for Production

```yaml
# apps/my-app/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: patches/deployment.yaml
```

## Security Best Practices

1. **RBAC**: Limit ArgoCD permissions per environment
2. **Secrets**: Use External Secrets Operator or Sealed Secrets
3. **Network Policies**: Restrict pod-to-pod communication
4. **Image Scanning**: Integrate with CI pipeline

## Contributing

1. Create a feature branch
2. Make changes following the existing patterns
3. Test with `kustomize build` locally
4. Submit PR for review

## License

MIT

