# App-of-Apps — Multi-Environment, Multi-Cluster ArgoCD Platform

Scalable GitOps structure for managing 100+ Kubernetes services across multiple environments and clusters using ArgoCD ApplicationSets.

Migrated from [zero-to-hero-v2](https://gitlab.crocobet.com/kubernetes/zero-to-hero-v2.git) to a clean, scalable app-of-apps pattern.

## Architecture Overview

```
app-of-apps/
├── Chart.yaml                         # Root Helm chart metadata
├── values.yaml                        # Global config: repo URL, project, environments
│
├── chart/                             # Shared service Helm chart (single template for all services)
│   ├── Chart.yaml
│   ├── values.yaml                    # Default values for all services
│   └── templates/
│       ├── _helpers.tpl               # Name/label helpers
│       ├── deployment.yaml            # Deployment with env vars, volumes, probes, strategy
│       ├── service.yaml               # ClusterIP service
│       ├── ingress.yaml               # Nginx Ingress
│       ├── configmap.yaml             # Optional ConfigMap
│       ├── secret.yaml                # Optional Secret skeleton
│       ├── hpa.yaml                   # HPA with CPU + Memory metrics
│       ├── httproute.yaml             # Gateway API HTTPRoute (alternative to Ingress)
│       ├── serviceaccount.yaml        # ServiceAccount
│       └── servicemonitor.yaml        # Prometheus ServiceMonitor
│
├── environments/                      # Per-environment/cluster defaults
│   ├── dev.yaml
│   ├── qa.yaml
│   ├── stage.yaml
│   ├── prod-c9.yaml                   # Production cluster C9 (10.4.12.100)
│   └── prod-co.yaml                   # Production cluster CO (10.4.12.200)
│
├── services/                          # One directory per service
│   ├── _template.yaml                 # Copy this to onboard a new service
│   └── yggdrasil-integration/
│       ├── base.yaml                  # Shared config (all envs): image, ports, ingress, volumes
│       ├── dev.yaml                   # Dev: image tag, ingress hosts, env vars
│       ├── qa.yaml                    # QA: different registry, resources
│       ├── stage.yaml                 # Stage: staging hosts
│       └── prod.yaml                  # Prod: HPA, resources, prod hosts
│
├── applicationsets/                   # Standalone ApplicationSet YAMLs
│   ├── dev.yaml
│   ├── qa.yaml
│   ├── stage.yaml
│   ├── prod-c9.yaml                   # C9 cluster (10.4.12.100)
│   └── prod-co.yaml                   # CO cluster (10.4.12.200)
│
└── templates/                         # Helm template that generates ApplicationSets
    └── applicationsets.yaml           # Renders from values.yaml environments
```

## How It Works

### Values Merge Order (lowest to highest priority)

```
chart/values.yaml              ← Chart defaults (all services, all envs)
  ↓
environments/<env>.yaml        ← Environment-wide overrides (replicas, HPA)
  ↓
services/<name>/base.yaml      ← Service-specific shared config (image, ports, ingress, volumes)
  ↓
services/<name>/<env>.yaml     ← Service + environment specific overrides (optional)
```

Each layer only defines what it overrides. Helm deep-merges everything.

### Auto-Discovery

Each ApplicationSet uses a **Git directory generator** that scans `services/*`. Adding a new directory under `services/` causes ArgoCD to automatically create Applications for it in every environment.

---

## Environment Strategy

| Environment | Cluster | Server | Namespace | Sync Policy |
|---|---|---|---|---|
| `dev` | c9 | 10.4.12.100:6443 | default | Auto-prune, self-heal |
| `qa` | c9 | 10.4.12.100:6443 | default | Auto-prune, self-heal |
| `stage` | c9 | 10.4.12.100:6443 | default | Auto-prune, self-heal |
| `prod-c9` | c9 | 10.4.12.100:6443 | default | Manual (no auto-prune/heal) |
| `prod-co` | co | 10.4.12.200:6443 | default | Manual (no auto-prune/heal) |

### Two Independent Production Clusters

Both prod clusters are **completely independent**:
- Separate ApplicationSets (`apps-prod-c9`, `apps-prod-co`)
- Separate cluster server URLs (c9: `10.4.12.100`, co: `10.4.12.200`)
- Separate Application names (`prod-c9-<service>`, `prod-co-<service>`)
- Share the same `prod.yaml` overrides (via `envKey: prod` in values.yaml)
- Can have cluster-specific defaults (`environments/prod-c9.yaml` vs `environments/prod-co.yaml`)
- Production sync policy is **manual** (no auto-prune, no self-heal) for safety

You can safely deploy to one cluster without affecting the other.

### Application Naming Convention

Format: `<env>-<service-name>`

Examples:
- `dev-yggdrasil-integration`
- `qa-yggdrasil-integration`
- `prod-c9-yggdrasil-integration`
- `prod-co-yggdrasil-integration`

This matches the zero-to-hero-v2 pattern (`{env}-{cluster}-{service}`).

---

## Chart Features

The shared service template (`chart/`) supports all features from zero-to-hero-v2:

| Feature | Controlled By |
|---|---|
| Deployment with rolling update strategy | `deploymentStrategy` |
| Direct env vars (SPRING_PROFILES_ACTIVE, etc.) | `env[]` |
| Nginx Ingress with multiple hosts | `ingress` |
| Gateway API HTTPRoute | `httpRoute` |
| HPA with CPU + Memory metrics | `hpa` |
| ConfigMap (envFrom) | `configMap` |
| Secret skeleton (envFrom) | `secret` |
| ServiceAccount | `serviceAccount` |
| Prometheus ServiceMonitor | `serviceMonitor` |
| Volume mounts (certs, configmaps) | `volumes`, `volumeMounts` |
| Lifecycle hooks (update-ca-certificates) | `lifecycle` |
| Pod anti-affinity | `affinity` |
| Init containers | `initContainers` |
| Extra ports (debug 5005) | `extraPorts` |
| Image pull secrets (regcred) | `imagePullSecrets` |

---

## Onboarding a New Service

```bash
# 1. Create service directory and config
mkdir services/my-new-service
cp services/_template.yaml services/my-new-service/base.yaml

# 2. Edit base.yaml — set image, ports, ingress, volumes
vim services/my-new-service/base.yaml

# 3. Create environment-specific overrides
# Copy from yggdrasil-integration as a reference:
cp services/yggdrasil-integration/dev.yaml services/my-new-service/dev.yaml
cp services/yggdrasil-integration/qa.yaml services/my-new-service/qa.yaml
cp services/yggdrasil-integration/stage.yaml services/my-new-service/stage.yaml
cp services/yggdrasil-integration/prod.yaml services/my-new-service/prod.yaml
# Edit each file — update image tags, ingress hosts, env vars

# 4. Commit and push — ArgoCD auto-discovers it
git add services/my-new-service/
git commit -m "onboard: my-new-service"
git push
```

ArgoCD creates Applications for `my-new-service` across all 5 environments automatically.

## Onboarding a New Cluster

```bash
# 1. Add to values.yaml under environments:
#    prod-new:
#      enabled: true
#      server: "https://<new-cluster-ip>:6443"
#      namespace: default
#      envKey: prod          # reuses services/*/prod.yaml
#      labels:
#        environment: prod
#        cluster: new

# 2. Create cluster defaults
cp environments/prod-c9.yaml environments/prod-new.yaml

# 3. Create standalone ApplicationSet
cp applicationsets/prod-c9.yaml applicationsets/prod-new.yaml
# Update: name, server, labels

# 4. Commit and push
git add environments/prod-new.yaml applicationsets/prod-new.yaml values.yaml
git commit -m "add prod-new cluster"
git push
```

---

## Promoting Changes Across Environments

### dev -> qa -> stage -> prod

This structure uses **directory-per-environment** (not branches). Promotion is done through Git commits:

```bash
# 1. Deploy to dev: edit services/<svc>/dev.yaml
vim services/yggdrasil-integration/dev.yaml
# Update image tag: v3.114 → v3.115
git commit -am "yggdrasil-integration: deploy v3.115 to dev"
git push
# → ArgoCD syncs to dev

# 2. Promote to qa: update services/<svc>/qa.yaml
vim services/yggdrasil-integration/qa.yaml
# Update image tag
git commit -am "yggdrasil-integration: promote v3.115 to qa"
git push

# 3. Promote to stage: update services/<svc>/stage.yaml
vim services/yggdrasil-integration/stage.yaml
git commit -am "yggdrasil-integration: promote to stage"
git push

# 4. Promote to prod: update services/<svc>/prod.yaml (via Pull Request!)
vim services/yggdrasil-integration/prod.yaml
git commit -am "yggdrasil-integration: promote to prod"
# Create PR → code review → merge → ArgoCD syncs to both C9 and CO
```

**Best practice**: Require pull request reviews for stage and prod promotions.

---

## Deployment

### Option A: Helm Bootstrap (Recommended for initial setup)

```bash
# Render and apply all ApplicationSets at once
helm template app-of-apps . --namespace argocd | kubectl apply -f -

# Or bootstrap via ArgoCD itself:
argocd app create app-of-apps \
  --repo https://github.com/iraklikairakli/app-of-app.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --sync-policy automated
```

### Option B: Direct kubectl apply (per-environment)

```bash
# Apply all at once
kubectl apply -f applicationsets/

# Or selectively
kubectl apply -f applicationsets/dev.yaml
kubectl apply -f applicationsets/qa.yaml
kubectl apply -f applicationsets/stage.yaml
kubectl apply -f applicationsets/prod-c9.yaml
kubectl apply -f applicationsets/prod-co.yaml
```

### ArgoCD CLI

```bash
# List all applications
argocd app list

# Filter by environment or cluster
argocd app list -l environment=prod
argocd app list -l cluster=c9
argocd app list -l environment=dev

# Sync a specific application
argocd app sync dev-yggdrasil-integration
argocd app sync prod-c9-yggdrasil-integration

# Check app status
argocd app get prod-c9-yggdrasil-integration

# Diff before sync
argocd app diff prod-c9-yggdrasil-integration

# Hard refresh (re-read from Git)
argocd app get dev-yggdrasil-integration --hard-refresh

# List ApplicationSets
argocd appset list

# View ApplicationSet details
argocd appset get apps-dev
argocd appset get apps-prod-c9
```

### ArgoCD Web UI

1. **View all apps**: Navigate to the ArgoCD dashboard. Use label filters `environment=dev` or `cluster=c9`.
2. **Sync**: Click any app, then "Sync" to trigger manual sync.
3. **App details**: Click an app to see resources, events, logs, and diff.
4. **ApplicationSets**: Go to "Settings" -> "ApplicationSets" to see the generators.
5. **Filtering**: Use search or label filters to navigate 100+ services efficiently.

---

## Quick Reference

| Task | Command |
|---|---|
| Add a service | `mkdir services/foo && cp services/_template.yaml services/foo/base.yaml` |
| Add an environment | Add entry to `values.yaml`, create `environments/X.yaml` + `applicationsets/X.yaml` |
| Add a prod cluster | Same as above with `envKey: prod` |
| Deploy to dev | Edit `services/*/dev.yaml`, push |
| Promote to prod | Edit `services/*/prod.yaml`, push via PR |
| View apps | `argocd app list -l environment=prod` |
| Sync an app | `argocd app sync dev-yggdrasil-integration` |
| Bootstrap | `kubectl apply -f applicationsets/` |
