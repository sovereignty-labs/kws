# Sanitization policy — `kws` public repo

This repository is a **curated portfolio extract**. It is not the operational
source of truth.

## Never commit

- Real IP addresses, VIPs, or site-specific subnets from a live estate
- Real hostnames, internal DNS zones, or node nicknames
- Inventories, full topology that would aid an attacker
- Secrets, tokens, sealed-secret blobs, private keys, kubeconfigs
- Unredacted identity-provider realm exports
- Raw operational dumps / context files from live clusters

## Always prefer

- Role names: `control-plane`, `worker`, `gpu-node`, `gitops-server`
- Example DNS: `*.platform.example.internal`
- Documentation IPs: `203.0.113.0/24` (TEST-NET-3) or `${VAR}` placeholders
- Incident stories that keep **technical truth** but drop targeting detail

## Before every push

```bash
./scripts/check-redaction.sh
```

Fix any hits by **rewriting**, not by light scrubbing of a live file.

## Relationship to private repos

Private Gitea/infra catalogs may contain full fidelity. Content moves **here** only
after a human (or directed agent) rewrite against this policy — never via bulk copy.
