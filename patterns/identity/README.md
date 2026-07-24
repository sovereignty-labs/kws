# Pattern: SSO + secrets (example only)

## Intent
- One OIDC issuer for humans and service-facing UIs
- Short-lived or vault-backed credentials for automation
- Cluster secrets synced from a secrets manager, not stored long-term in git

## Example ExternalSecret shape (fake)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: sample-app-db
  namespace: sample-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: sample-app-db
  data:
    - secretKey: password
      remoteRef:
        key: kv/data/sample-app/db
        property: password
```

No real paths, mounts, or tokens from any live Vault are published here.
