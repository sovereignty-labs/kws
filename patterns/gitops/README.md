# Pattern: app-of-apps (example only)

Illustrative structure. Names and URLs are fake.

```text
argocd/
  root-app.yaml          # points at platform/ and apps/
  platform/
    cert-manager.yaml
    external-secrets.yaml
    ingress.yaml
  apps/
    sample-app.yaml
```

Root Application (sketch):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://git.example.internal/platform/deploy.git
    targetRevision: main
    path: argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Live estates use private remotes and real projects -- not these names.**
