# Incident narratives (redacted)

Technical substance retained; hostnames, IPs, and site-specific identifiers removed.
Full STAR forms for interviews live in the private career prep materials.

## 1. Control-plane chaos test
**Situation.** Multi-node HA Kubernetes control plane was claimed HA but not
failure-tested.
**Action.** Deliberately failed control-plane capacity; confirmed fail-safe
behavior; found placement drift (members co-located on one physical host);
enforced anti-affinity / spread.
**Result.** Full recovery; latent SPOF closed.
**Shows:** proactive reliability, chaos discipline.

## 2. False "done" / crash-loop inheritance
**Situation.** Subsystems reported healthy and complete.
**Action.** Checked live state; found crash-loops, sealed secrets engine issues,
stale feature flags; fixed and re-verified live.
**Result.** Operating rule established: verify live, don't trust reports.
**Shows:** judgment, skepticism, integrity.

## 3. Destroyed control-plane member mid-operation
**Situation.** Orchestration bug destroyed a live control-plane VM during a
scaling operation.
**Action.** Recovered membership, restored etcd quorum, root-caused a
false-negative in a retry path.
**Result.** Quorum restored; failure mode closed.
**Shows:** incident recovery, etcd/K8s depth.

## 4. GitOps source-of-truth migration without sync outage
**Situation.** Entire cluster's GitOps remote needed to move.
**Action.** Credential for new path first; update app-of-apps and live apps
together; discovered Git client did not follow HTTP redirects -- adapted by
pointing apps at final URLs.
**Result.** Apps stayed Synced/Healthy throughout.
**Shows:** change management, GitOps depth, live adaptation.

## 5. Auth success, git still failed
**Situation.** SSH key authenticated; git operation failed with a misleading
account message.
**Action.** Traced to service account shell / forced-command interaction
(`nologin` class failure); fixed shell while keeping key restricted to git.
**Result.** Platform git-over-SSH restored; verified with a real operation.
**Shows:** methodical debugging below the error string.
