# Decisions (ADR-style, public)

## Storage: replicated block (LINSTOR/DRBD-class), not full Ceph
**Chose:** Dedicated replicated block for VM + K8s volumes; object storage as a
separate S3 service.
**Rejected:** All-in Ceph fabric at this scale (ops weight vs benefit).
**Why it matters:** Matches small multi-host reality; clear failure domains.

## DNS: delegation-forwarding, not replacement
**Chose:** Existing network resolver stays authoritative; forwards internal zone
to a dedicated DNS service.
**Rejected:** Big-bang cutover that replaces the edge resolver.
**Why it matters:** Additive and reversible.

## Repo boundary: GitOps deploy tree != knowledge/IaC tree
**Chose:** ArgoCD watches a lean Kubernetes-only deploy repo; VM templates,
non-K8s IaC, and engineering docs live apart.
**Rejected:** One monorepo where a docs edit can touch production sync paths.
**Why it matters:** Blast-radius control for GitOps.

## DR: agent-assisted recovery over static-only scripts
**Chose:** Local recovery capability that is also used day-to-day (stays
exercised) plus documented procedures.
**Rejected:** Runbooks that only exist for the day the world ends.
**Why it matters:** Bit-rot kills DR; dual-use keeps it honest.

## Identity: one IdP, SSO everywhere, least privilege
**Chose:** Single OIDC provider; group-driven access; scoped credentials; egress
locked down by default.
**Rejected:** Per-app local users and long-lived shared passwords.
**Why it matters:** Security is structural, not bolted on.

## Verification: live state over status reports
**Chose:** "Done" means checked in reality (API, pods, failover drill), not a
checklist claim.
**Rejected:** Trusting agent or session "all green" summaries without proof.
**Why it matters:** Multiple false-done events made this non-negotiable.
