# Incident narratives (redacted)

Real incidents on a production private AI cloud, written the way I'd want to read them:
what happened, what I did, what it cost, and what I got wrong.

Technical substance retained; hostnames, addresses, VM identifiers and site-specific
details removed — see [`../SANITIZATION.md`](../SANITIZATION.md). Full STAR forms for
interviews live in private material.

---

## 1. Full power loss — whole platform down, back in 1h50m with zero data loss

**The one that counts, because nobody scheduled it.**

**Situation.** Total power loss to the estate. Every physical host, the distributed
storage layer, the database cluster, the hypervisor control plane and the Kubernetes
cluster went down hard, at once, with no graceful shutdown. One host then failed to come
back at all — its BIOS boot order had been lost and needed repairing at the console.

**Action.** Recovered bottom-up in dependency order, verifying each layer before starting
the next. That order is the whole lesson, and I had it written down because a *planned*
drill six days earlier had produced it:

1. **Physical hosts** — power on; repair the lost boot entry at the console.
2. **Distributed block storage** — nothing to do. A hardening change made days earlier
   (disabling automatic eviction) meant zero evictions; satellites reconnected and
   resynced unaided. Verify only.
3. **Database cluster** — self-formed into a primary component, 3/3 synced, no manual
   bootstrap, thanks to an earlier hardening pass. I still treat manual bootstrap as the
   fallback: self-forming is not guaranteed when nodes die at different times.
4. **Hypervisor control plane** — came up on its own once the database was there.
5. **Virtual machines** — resumed in waves: gateway, then control plane, then storage and
   workers, then service VMs. **Expected and hit a retry pass** — resumes issued while
   replicated disks are still reconnecting fail, because the volume has no connected
   up-to-date peer yet. Six of sixteen hit it; a second pass ten minutes later succeeded
   for all.
6. **Network gateway pair** — the hard part. See below.
7. **Kubernetes** — started by hand on every node, by design. Control-plane nodes took
   5–15 minutes each for quorum and defragmentation; one was IO-starved by three VMs
   cold-booting simultaneously against network-remote disks. Patience, not a defect.
8. **In-cluster convergence** — largely automatic once the API was up, except the secrets
   chain and an image-availability trap.

**Result.** Everything recovered. **No data loss. No split-brain.** All VMs running, all
Kubernetes nodes ready, secrets management HA and unsealed, every GitOps application
synced and healthy, storage fully up-to-date, monitoring confirmed fresh, CI runner
re-registered on its own.

| Elapsed | Phase |
|---|---|
| 0:00–0:15 | Assess; verify storage, database and hypervisor self-recovery |
| 0:15–0:35 | VM resume waves including the retry pass; start Kubernetes |
| 0:35–1:05 | Wait out an IO storm; diagnose and fix the gateway failure |
| 1:05–1:50 | In-cluster recovery, secrets unseal, image steering, verification |

I then replaced the theory-written runbook with a **verified** one, and fixed what the
outage exposed the same evening.

**What it exposed** — the value of an unplanned outage is that it finds what a drill
cannot:

- **The "HA" gateway pair was a cold-start single point of failure.** A known reload race
  killed the load balancer on *both* gateways during boot, and the watchdog meant to catch
  exactly that had never started — its log was empty. The active gateway held the virtual
  IP as a black hole: answering ARP, serving nothing. **Fixed the same evening** with
  file-locking around config regeneration and the watchdog rebuilt under a supervisor with
  heartbeat logging, then validated by hard-rebooting both gateways. Failover measured at
  ≤13 seconds.
- **Egress lockdown plus an incomplete internal mirror is a reschedule trap.** Pods
  rescheduled onto a node without a cached image cannot pull, because the outside world is
  deliberately unreachable. Hit the CNI operator, cert-manager, the ingress controller and
  more. Worked around live by locating the node holding the cache and steering pods to it.
  **Fixed** by populating the mirror and configuring every node to mirror upstream
  registries.
- **A certificate was missing a DNS subject alternative name**, so the cluster had to be
  reached by address rather than name mid-recovery. **Fixed** at the template level so new
  nodes inherit it.
- **A race between disk reconnection and VM resume**, self-inflicted by recovering faster
  than storage could reconnect. Now documented: expect a retry pass, or wait for
  replication links to establish.

**The honest comparison.** My previous-generation platform, on an immutable Kubernetes
distribution, recovered from the same outage **completely hands-off**. This one needed
manual gates. Most are deliberate — I want a human deciding when a control plane rejoins.
One was not: an automatic VM-resume hook I had specified and never built would have
removed roughly twenty minutes of manual work and the disk race entirely.

**Shows:** disaster recovery under real conditions; dependency-order reasoning; converting
an incident into a runbook and permanent fixes; and telling a deliberate manual gate apart
from an unbuilt one.

---

## 2. Control-plane chaos test

**Situation.** A multi-node HA Kubernetes control plane was *claimed* HA and had never
been failure-tested. Claimed and tested are different words.

**Action.** Deliberately failed control-plane capacity — hard-killing a host running two
of three members — to find out what would happen before it happened on its own. Confirmed
fail-safe behaviour; found placement drift, with members co-located on one physical host;
enforced anti-affinity and spread.

**Result.** Full recovery, and a latent single point of failure closed before it could
choose its own timing.

**Shows:** proactive reliability, chaos discipline.

---

## 3. Destroyed control-plane member mid-operation

**Situation.** An orchestration bug destroyed a live control-plane VM during a scaling
operation.

**Action.** Recovered membership and restored database quorum, then root-caused it rather
than stopping at "it works again" — a false-negative in a retry path, where empty output
was read as failure and "corrected" by destroying a healthy node.

**Result.** Quorum restored; the failure mode closed at source.

**Shows:** incident recovery, etcd and Kubernetes depth, and the difference between
restoring service and fixing a bug.

---

## 4. Work reported "done" that wasn't

**Situation.** Multiple subsystems were reported healthy and complete — by tooling, by
dashboards, and by AI assistants working on the platform.

**Action.** Checked live state instead of reading the reports. Found a crash-looping
control plane, a secrets engine sealed for **27 hours** with its auto-unseal chain
silently dead, and a storage feature flag reported off that was on. Fixed and re-verified
each against the running system.

**Result.** The operating rule for everything since: **verify live, don't trust reports.**
A status document, a dashboard, or an AI's "done" is a claim. Only a query against the
running system is a fact.

**Shows:** the judgment that makes AI-directed infrastructure safe to run. The doctrine as
a scar, not a slogan.

---

## 5. GitOps source-of-truth migration without sync outage

**Situation.** The entire cluster's GitOps remote had to move while continuous delivery
stayed live against it. Applications were auto-synced, so a mistake would deploy itself.

**Action.** New-path credential first; updated the app-of-apps and the live applications
together. Discovered mid-migration that the Git client did not follow HTTP redirects, and
adapted by pointing applications at final URLs rather than relying on the redirect.

**Result.** Applications stayed Synced and Healthy throughout.

**Shows:** change management on a system that deploys itself, GitOps depth, and adapting
to a discovery mid-operation instead of restarting.

---

## 6. Authentication succeeded, git still failed

**Situation.** An SSH key authenticated successfully, but the git operation failed with a
misleading account-level message that pointed nowhere useful.

**Action.** Traced it below the error string to an interaction between the service
account's shell and a forced command — a `nologin`-class failure that reports as an
authorization problem. Fixed the shell while keeping the key restricted to git only.

**Result.** Platform git-over-SSH restored, verified with a real operation rather than a
login test.

**Shows:** methodical debugging beneath the error message, and not trading a security
control away to make an error disappear.
