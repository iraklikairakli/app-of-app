 # App of Apps — Scalable Structure

## Adding a new service

```bash
mkdir services/my-new-service
cp services/_template.yaml services/my-new-service/values.yaml
# Edit values.yaml — done. ArgoCD auto-discovers it.
git add . && git commit -m "add my-new-service" && git push
```

## Structure

```
├── applicationset.yaml        # Deploy once — auto-discovers services/*/
├── defaults.yaml              # Global defaults (shared)
├── chart/                     # Shared Helm chart (service-template)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
└── services/                  # Each service = 1 directory + 1 file
    ├── _template.yaml         # Copy this for new services
    ├── frontend/
    │   └── values.yaml        # Only overrides
    └── ...
```
