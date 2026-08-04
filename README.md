# KWS — Private AI Cloud (portfolio)

**Curated public repository.** Architecture, decisions, operating model, and
pattern sketches for **KWS** — a production private AI cloud built and operated
as a sole-architect system.

This is **not** a dump of live infrastructure. Operational catalogs, inventories,
and secrets stay private. What you see here is deliberately rewritten for
portfolio and hiring review.

| | |
|--|--|
| **Role** | Sole architect & operator |
| **Status** | Completed platform · operating in production |
| **Org** | [Sovereignty Labs](https://github.com/sovereignty-labs) |
| **Related** | [Asgard](https://github.com/sovereignty-labs/asgard) (prior enterprise platform generation) · product work (Hirdforge, anvil) publishes separately |

## What KWS is

A private cloud for AI workloads: bare metal compute, managed Kubernetes, GitOps,
replicated storage, object storage, SSO, secrets, internal PKI, and a self-hosted
LLM inference tier — designed so the platform can be operated with agent
assistance under human authority and live verification.

```
            +---------- GitOps (ArgoCD app-of-apps) ----------+
 Git SoT -->|  bootstrap · platform · apps · pipelines        |--> Kubernetes (HA control plane)
   ^        +-------------------------------------------------+
   | CI -> registry -> deploy (per-PR sandboxes, git-revert rollback)
   |
 Hypervisor / bare metal  --  replicated block · S3 object · bulk backup tier
   |
 SSO (one OIDC IdP) · Vault (HA secrets) · internal PKI/CA
   |
 GPU fleet -- self-hosted LLM inference -- private AI surface + MCP tooling
```

## How it was built (stated plainly)

I design the system and make the decisions. AI implements under direction. I
**verify against live state** and course-correct. Every major primitive reflects
judgment from earlier builds (lab cluster -> enterprise platform generation --
see [Asgard](https://github.com/sovereignty-labs/asgard)), not an agent inventing
the architecture.

## Start here

| Doc | Contents |
|-----|----------|
| [docs/architecture.md](docs/architecture.md) | System shape (roles, not hostnames) |
| [docs/decisions.md](docs/decisions.md) | ADRs with rejected alternatives |
| [docs/operating-model.md](docs/operating-model.md) | Verify-live, GitOps, security posture |
| [docs/incidents.md](docs/incidents.md) | Real incident narratives (redacted) |
| [docs/agent-failure-analysis.md](docs/agent-failure-analysis.md) | How AI agents fail at infrastructure work — hand-labelled taxonomy from one audited session |
| [patterns/](patterns/) | Example-only manifests (TEST-NET / placeholders) |
| [SANITIZATION.md](SANITIZATION.md) | What never ships in this repo |

## Stack (representative)

OpenNebula/KVM · RKE2 Kubernetes · ArgoCD · Gitea + CI · Terraform · Ansible ·
LINSTOR/DRBD · MinIO · ZFS · HashiCorp Vault · Keycloak · cert-manager · Cilium ·
llama.cpp / GPU fleet · MCP

## License

Documentation and example patterns: MIT (see [LICENSE](LICENSE)).
Example manifests are illustrative only -- not a turnkey deploy of any live estate.
