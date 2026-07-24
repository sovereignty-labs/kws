# Operating model

## Architect -> direct AI -> verify live -> course-correct

Implementation is AI-directed. Ownership of design, prioritization, and
acceptance is human. AI output is a proposal until live state agrees.

## Principles

1. **Verify live, don't trust reports.**
2. **GitOps + IaC by default** -- declarative, versioned, reversible.
3. **Security is structural** -- one IdP, real secrets manager, least privilege, egress control.
4. **Generated-or-banned** -- inventories/catalogs that can rot are generated from
   live state in CI; drift fails the build.
5. **Escalate loudly, degrade never silently.**

## Day-2 signals (portfolio-relevant)

- HA control plane, secrets tier, and edge gateway designed for failure
- Chaos and recovery drills performed (see [incidents.md](incidents.md))
- GitOps source migrations planned as multi-step cutovers with health checks

## What "production" means here

Not a hyperscaler multi-tenant SaaS. A **real estate that runs real workloads**,
with incidents, backups, identity, and change control -- operated continuously by
one architect with agent assistance.
