# Architecture (redacted)

Role-level view of KWS. No live hostnames or addresses.

## Layers

### 1. Compute & tenancy
- Bare-metal hosts running a KVM-based IaaS control plane
- VM catalogs for platform services and Kubernetes nodes
- Separate GPU hosts for model inference (not mixed casually with CP roles)

### 2. Kubernetes
- Managed Kubernetes distribution with a **multi-node HA control plane**
- Workers for platform and application workloads
- CNI with network policy support for least-privilege east-west traffic

### 3. GitOps & delivery
- ArgoCD **app-of-apps**: bootstrap -> platform -> apps
- Self-hosted Git as source of truth
- CI -> container registry -> deploy
- Per-PR sandbox environments; rollback = git revert + reconcile

### 4. Storage
- Replicated block for VM disks and Kubernetes volumes
- S3-compatible object storage for artifacts and model stages
- Bulk/ZFS-class tier under a 3-2-1 backup posture

### 5. Identity, secrets, PKI
- Single OIDC identity provider; SSO across services
- HA secrets platform (Vault-class): AppRole, transit unseal patterns, OIDC login
- Secrets projected into cluster via external-secrets style sync
- Internal CA + cert-manager for service TLS

### 6. AI layer
- Self-hosted LLM inference on the GPU fleet (OpenAI-compatible front door)
- Private chat/RAG and MCP tooling as internal consumers
- Platform itself operable with **directed agents** under human authority

## Trust model (agents)

| Agent class | Authority |
|-------------|-----------|
| Local / break-glass recovery | Operational (narrow, auditable) |
| External / consumer | No cluster admin; product surfaces only |
| Secret handoff | Response-wrapped / short-lived credentials where possible |

## What is intentionally omitted

Exact node counts beyond "HA control plane," physical layout, IP plan, and
vendor account identifiers. Those do not improve portfolio signal and increase risk.
