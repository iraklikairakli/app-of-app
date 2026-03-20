# App-of-Apps — Multi-Environment, Multi-Cluster ArgoCD Platform

Scalable GitOps structure for managing 100+ Kubernetes services across multiple environments and clusters using ArgoCD ApplicationSets.

## Architecture Overview

```
app-of-apps/
├── Chart.yaml                         # Root Helm chart metadata
├── values.yaml                        # Global config: repo URL + environment definitions
│
├── chart/                             # Shared service Helm chart (single template for all services)
│   ├── Chart.yaml
│   ├── values.yaml                    # Default values for all services
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── configmap.yaml
│       ├── secret.yaml
│       ├── hpa.yaml
│       └── httproute.yaml
│
├── environments/                      # Per-environment/cluster default overrides
│   ├── dev.yaml                       # Dev: low resources, 1 replica
│   ├── stage.yaml                     # Stage: mirrors prod sizing
│   ├── prod-cluster-1.yaml            # Prod cluster 1: full resources
│   └── prod-cluster-2.yaml            # Prod cluster 2: full resources
│
├── services/                          # One directory per service
│   ├── _template.yaml                 # Copy this to onboard a new service
│   ├── frontend/
│   │   ├── base.yaml                  # Shared config (all envs)
│   │   ├── dev.yaml                   # Dev-specific overrides
│   │   └── prod.yaml                  # Prod-specific overrides
│   ├── api-gateway/
│   │   ├── base.yaml
│   │   ├── dev.yaml
│   │   └── prod.yaml
│   └── ...
│
├── applicationsets/                   # Standalone ApplicationSet YAMLs (alternative to Helm)
│   ├── dev.yaml
│   ├── stage.yaml
│   ├── prod-cluster-1.yaml
│   └── prod-cluster-2.yaml
│
└── templates/                         # Helm template that generates ApplicationSets
    └── applicationsets.yaml           # Renders from values.yaml environments
```

## How It Works

### Values Merge Order (lowest to highest priority)

```
chart/values.yaml              ← Chart defaults (all services, all envs)
  ↓
environments/<env>.yaml        ← Environment-wide overrides (resources, replicas)
  ↓
services/<name>/base.yaml      ← Service-specific shared config (image, ports, routes)
  ↓
services/<name>/<env>.yaml     ← Service + environment specific overrides (optional)
```

Each layer only needs to define what it overrides. Helm deep-merges everything.

### Auto-Discovery

Each ApplicationSet uses a **Git directory generator** that scans `services/*`. When you add a new directory under `services/`, ArgoCD automatically creates Applications for it in every environment.

---

## Environment Strategy

| Environment | Namespace | Purpose | Sync Policy |
|---|---|---|---|
| `dev` | `dev` | Development and testing | Auto-sync, self-heal |
| `stage` | `stage` | Pre-production validation | Auto-sync, self-heal |
| `prod-cluster-1` | `production` | Production (us-east-1) | Auto-sync, self-heal |
| `prod-cluster-2` | `production` | Production (eu-west-1) | Auto-sync, self-heal |

### Two Independent Production Clusters

Both prod clusters are **completely independent**:
- Separate ApplicationSets (`apps-prod-cluster-1`, `apps-prod-cluster-2`)
- Separate cluster server URLs
- Separate Application names (`<service>-prod-1`, `<service>-prod-2`)
- Share the same `prod.yaml` overrides (via `envKey: prod` in values.yaml)
- Can have cluster-specific environment files (`environments/prod-cluster-1.yaml` vs `environments/prod-cluster-2.yaml`)

You can safely update one cluster without affecting the other.

---

## Onboarding a New Service

```bash
# 1. Create service directory and config
mkdir services/my-new-service
cp services/_template.yaml services/my-new-service/base.yaml

# 2. Edit base.yaml — set your image, ports, routes
vim services/my-new-service/base.yaml

# 3. (Optional) Add environment-specific overrides
# Only create these if the service needs different config per env
cat > services/my-new-service/prod.yaml << 'EOF'
hpa:
  enabled: true
  maxReplicas: 8
EOF

# 4. Commit and push — ArgoCD auto-discovers it
git add services/my-new-service/
git commit -m "onboard: my-new-service"
git push
```

That's it. ArgoCD creates Applications for `my-new-service` in all environments automatically.

## Onboarding a New Cluster / Environment

### Adding a new environment (e.g., `qa`)

```bash
# 1. Add to values.yaml
cat >> values.yaml << 'EOF'
  qa:
    enabled: true
    server: "https://qa.k8s.example.com"
    namespace: qa
    labels:
      environment: qa
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
      syncOptions:
        - CreateNamespace=true
EOF

# 2. Create environment defaults
cp environments/stage.yaml environments/qa.yaml
vim environments/qa.yaml

# 3. (Optional) Create standalone ApplicationSet
cp applicationsets/stage.yaml applicationsets/qa.yaml
# Update: name, server, namespace, valueFiles path

# 4. Commit and push
git add environments/qa.yaml applicationsets/qa.yaml values.yaml
git commit -m "add qa environment"
git push
```

### Adding a third production cluster

```bash
# 1. Add to values.yaml under environments:
#    prod-cluster-3:
#      enabled: true
#      server: "https://prod-3.k8s.example.com"
#      namespace: production
#      envKey: prod        # reuses services/*/prod.yaml
#      labels:
#        environment: prod
#        cluster: prod-3

# 2. Create cluster defaults
cp environments/prod-cluster-1.yaml environments/prod-cluster-3.yaml

# 3. Create standalone ApplicationSet
cp applicationsets/prod-cluster-1.yaml applicationsets/prod-cluster-3.yaml
# Update: name → apps-prod-cluster-3, server, labels

# 4. Commit and push
```

---

## Promoting Changes Across Environments

### dev → stage → prod

This structure uses **directory-per-environment** (not branches). Promotion is done through Git commits:

```bash
# 1. Deploy to dev: edit services/<svc>/dev.yaml or base.yaml
vim services/frontend/dev.yaml
git commit -am "frontend: update API_URL for dev"
git push
# → ArgoCD syncs to dev cluster

# 2. Promote to stage: create/update services/<svc>/stage.yaml
cp services/frontend/dev.yaml services/frontend/stage.yaml
vim services/frontend/stage.yaml  # adjust env-specific values
git commit -am "frontend: promote to stage"
git push
# → ArgoCD syncs to stage cluster

# 3. Promote to prod: create/update services/<svc>/prod.yaml
vim services/frontend/prod.yaml
git commit -am "frontend: promote to prod"
git push
# → ArgoCD syncs to both prod clusters
```

**Best practice**: Use pull requests for stage and prod promotions. Require code review before merging.

### Image tag promotion

For most deployments, you only update the image tag:

```bash
# Update in base.yaml (deploys everywhere) or env-specific file
vim services/frontend/base.yaml
# Change: tag: "v1.2.0" → tag: "v1.3.0"

git commit -am "frontend: release v1.3.0"
git push
```

---

## Deployment: Two Approaches

You can deploy the ApplicationSets either via **Helm** (renders from `values.yaml`) or **directly** (applies standalone YAMLs).

### Option A: Helm Bootstrap (Recommended for initial setup)

```bash
# Install the root chart — generates all ApplicationSets
helm template app-of-apps . --namespace argocd | kubectl apply -f -

# Or with ArgoCD managing itself:
argocd app create app-of-apps \
  --repo https://github.com/iraklikairakli/app-of-app.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --sync-policy automated
```

### Option B: Direct kubectl apply (simpler, per-environment)

```bash
# Apply specific environment ApplicationSets
kubectl apply -f applicationsets/dev.yaml
kubectl apply -f applicationsets/stage.yaml
kubectl apply -f applicationsets/prod-cluster-1.yaml
kubectl apply -f applicationsets/prod-cluster-2.yaml

# Or all at once
kubectl apply -f applicationsets/
```

### ArgoCD CLI

```bash
# List all applications
argocd app list

# Filter by environment
argocd app list -l environment=prod
argocd app list -l cluster=prod-1

# Sync a specific application
argocd app sync frontend-dev
argocd app sync frontend-prod-1

# Check app status
argocd app get frontend-prod-1

# Diff before sync
argocd app diff frontend-prod-1

# Hard refresh (re-read from Git)
argocd app get frontend-prod-1 --hard-refresh

# List ApplicationSets
argocd appset list

# View ApplicationSet details
argocd appset get apps-dev
```

### ArgoCD Web UI

1. **View all apps**: Navigate to the ArgoCD dashboard. Filter by label `environment=dev` or `cluster=prod-1` to see apps per environment.
2. **Sync**: Click any app → "Sync" button to trigger manual sync.
3. **App details**: Click an app to see its resources, events, logs, and diff.
4. **ApplicationSets**: Go to "Settings" → "ApplicationSets" to see the generators.
5. **Filtering**: Use the search bar or label filters (`environment:prod`, `cluster:prod-1`) to navigate 100+ services efficiently.

---

## Secrets Management

**This repository does NOT store secret values.** The `secret.enabled: true` flag creates a Kubernetes Secret resource skeleton, but actual values must be injected by one of:

- **External Secrets Operator** (recommended) — syncs secrets from AWS Secrets Manager, Vault, GCP Secret Manager, etc.
- **Sealed Secrets** — encrypts secrets for safe Git storage
- **HashiCorp Vault** with the Vault Agent sidecar
- **SOPS** — encrypts YAML values in-place

Configure your chosen solution per-cluster, and reference the secret name in your service's `base.yaml`.

---

## Anti-Patterns Avoided

| Anti-Pattern | What We Do Instead |
|---|---|
| Branch-per-environment | Directory-per-environment in a single branch |
| Secrets in Git | External Secrets Operator / Sealed Secrets |
| Duplicate chart directories | Single `chart/` directory |
| Manual `kubectl edit` in prod | All changes via Git commits |
| Monolithic values file | Per-service directory with base + env overlays |
| Hardcoded cluster URLs | Configurable via `values.yaml` environments |

---

## Quick Reference

| Task | Command |
|---|---|
| Add a service | `mkdir services/foo && cp services/_template.yaml services/foo/base.yaml` |
| Add an environment | Add entry to `values.yaml`, create `environments/X.yaml` |
| Add a prod cluster | Same as above with `envKey: prod` |
| Deploy to dev | Edit `services/*/dev.yaml` or `base.yaml`, push |
| Promote to prod | Edit `services/*/prod.yaml`, push via PR |
| View apps | `argocd app list -l environment=prod` |
| Sync an app | `argocd app sync frontend-prod-1` |
| Bootstrap | `kubectl apply -f applicationsets/` |
