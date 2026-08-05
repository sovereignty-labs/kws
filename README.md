# KWS — a private AI cloud, operated by agents

Bare metal to Kubernetes: GitOps, replicated storage, SSO, HA secrets, internal PKI,
and a self-hosted GPU inference tier — built and run as a sole-architect system, and
operated day to day by local agents under direction.

**It survived a total power loss with zero data loss and no split-brain.** Storage,
database quorum, hypervisors and Kubernetes cold-started in dependency order. The
disaster-recovery plan of record here is an agent with a recovery skill rather than a
human runbook, because a runbook is useless if the person holding it is asleep. Two
steps needed a human, and both were things an agent physically cannot do: repairing a
lost BIOS boot entry at the console, and unsealing Vault with shamir keys.

| | |
|--|--|
| **Role** | Sole architect & operator |
| **Status** | Production |
| **Scale** | 19 monitored hosts · 6-node RKE2 · ~104 pods · 8 GitOps applications · 3-node HA Vault |

---

## Read these three first

| | |
|--|--|
| **[docs/agent-failure-analysis.md](docs/agent-failure-analysis.md)** | **How AI agents fail at infrastructure work.** 13 claims from three AI systems audited against live state; 8 contained a material error, 3 of them mine. Seven of the eight were a real measurement taken correctly against the wrong surface — so the defence isn't "check for hallucination," it's "check whether the measurement addresses the claim." |
| **[docs/incidents.md](docs/incidents.md)** | Real incidents, including what I got wrong. The power-loss recovery, a deliberate chaos test that killed two of three control-plane nodes, and a secrets engine sealed for 27 hours behind green dashboards. |
| **[docs/decisions.md](docs/decisions.md)** | ADRs with the **rejected alternatives kept in**, so the reasoning is auditable rather than just the outcome. |

Also here: [architecture](docs/architecture.md) (roles, not hostnames) ·
[operating model](docs/operating-model.md) · [patterns/](patterns/) (example manifests)

---

## How it's built, stated plainly

I design the system and make the decisions. AI implements under direction. I **verify
against live state** and course-correct — and when it's wrong, I write that down too.
Every major primitive carries judgment from earlier generations rather than an agent
inventing the architecture; the lineage runs
[Asgard](https://github.com/sovereignty-labs/asgard) → KWS.

The operational knowledge lives in the repo the agents read: a runbook of traps that
already cost real time, dated incident records, and proven primitives. Live state is
always a tool call, never a document — when the two disagree, the tool wins and the
stale document is itself a finding.

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

## Stack

OpenNebula/KVM · RKE2 Kubernetes · ArgoCD · Gitea + CI · Terraform · Ansible ·
LINSTOR/DRBD · MinIO · ZFS · HashiCorp Vault · Keycloak · cert-manager · Cilium ·
llama.cpp / GPU fleet · MCP

---

## What this repository is

A **curated extract, not a mirror.** Architecture, decisions and incidents are public;
live inventories, addresses and secrets are not — see
[SANITIZATION.md](SANITIZATION.md). Hosts appear by role. Example manifests use
TEST-NET addresses and are illustrative rather than a turnkey deploy.

That line is itself part of the work: knowing what can be published, and proving it
with a gate rather than a promise.

**Related** · [Asgard](https://github.com/sovereignty-labs/asgard) — the platform
generation before this one · [Sovereignty Labs](https://github.com/sovereignty-labs)

## License

Documentation and example patterns: MIT (see [LICENSE](LICENSE)).
