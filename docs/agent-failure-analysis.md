# Error analysis: how AI agents fail at infrastructure work

A hand-labelled failure taxonomy from one working session on a production private
AI cloud, 2026-08-02 to 2026-08-03.

Not a benchmark and not a survey. Thirteen substantive claims made by three AI
systems doing real work on a live estate, each one audited against live state, each
failure labelled by hand and grouped afterwards rather than sorted into categories
chosen in advance.

Redacted per [`../SANITIZATION.md`](../SANITIZATION.md): hosts appear by role,
addresses and internal DNS are removed. Every number is measured, not estimated.

---

## Method

Three systems were working on the estate concurrently:

| system | role that session |
|---|---|
| **Agent A** — hosted coding agent | container CVE remediation, mirror hygiene |
| **Agent B** — hosted coding agent | repository redaction before public release |
| **Local model** — self-hosted reasoning model on the inference host | read-only security review of the cluster |

Every claim any of them made — including my own — was checked against the running
system: `kubectl` against live objects, PromQL against the metrics store, Hubble
flow records, `git` against actual history. A claim counts as an error if a
competent operator acting on it would have done the wrong thing.

**The audit is the dataset.** What follows is not "here are some bugs"; it is what
the failures had in common.

---

## Headline

**Of 13 audited claims, 8 contained a material error.**

The distribution matters more than the count:

- **6 of 8 were errors in verification or reporting**, not in the work. The
  underlying engineering was largely sound — real CVEs were closed, real images
  were mirrored, real redaction was applied — and then described inaccurately.
- **2 of 8 were errors in the work itself**, and both were caused by a verification
  error upstream.
- **3 of the 8 were mine.** An audit that only catches other systems is measuring
  the auditor's confidence, not the systems.

The single most useful finding: **7 of the 8 were an instrument pointed at the
wrong surface.** Not hallucination, not incompetence — a measurement that was
technically executed correctly against something other than the thing in question.

---

## The taxonomy

### 1. Unit mismatch across a before/after comparison

Agent A reported reducing critical CVEs **210 → 41, a 80% cut**. Both numbers were
real. They measured different things: 210 was *total occurrences* summed per image;
41 was *distinct CVE-plus-package pairs*. I measured 40 distinct pairs, so its
arithmetic was fine.

In consistent units the result was **210 → 151 (−28%)**, or **208 → 131 (−37%)**
scoped to the mirror manifest. Both defensible; neither defensible without naming
its scope. The agent had also changed the *after* number's scope three times while
holding the *before* number fixed, which is what produced each successive wrong
figure.

> **Countermeasure.** Take before and after from the same query, in the same
> scope, in the same run. A delta assembled from two measurements is a hypothesis.

### 2. Namespace mismatch during triage — *cost: real*

Agent A retired 52 images from the mirror manifest as "not currently running."
**15 of the 52 were running**, including the CNI (42 pods), the hardened Kubernetes
image (9), the Helm controller (6), `busybox` (6), etcd (3), and all three
cert-manager components.

The cause: workloads reference images by *upstream* name (`rancher/…`,
`quay.io/jetstack/…`) while the manifest lists them under the *internal mirror*
name. A registry-mirror configuration is precisely what makes those the same
image — so a literal string comparison reports "unused" for the most load-bearing
images on the platform.

Worse, the manifest is not an inventory of what runs today; it is the **airgap
baseline**. Images not running now are exactly what a reschedule or a cold start
needs. A prior full-power-loss recovery on this estate was extended by exactly this
condition — pods rescheduling onto nodes with no cached image and no route to the
internet — and the fix at the time was populating that manifest.

The accompanying "48 → 29 CVEs" was entirely an artefact of deleting the rows.

> **Countermeasure.** Normalise identifiers to a canonical form before comparing
> them. And know what a file *is*: "what runs" and "what must be pullable" are
> different sets, and only one of them protects a cold start.

### 3. Correct diagnosis, self-negating prescription

The local model correctly established that network policy existed in only 2 of 23
namespaces, that zero CNI-native policies existed, and that the CNI's enforcement
mode was the permissive default — "enforce only where a policy selects the pod."
All three verified true.

It then prescribed setting that same mode to `default`, describing it as
"enforce every endpoint, deny unless allowed." It had diagnosed the permissive
value and prescribed the permissive value. The parameter name was also wrong.

Applied as written, the top remediation for the top finding would have done
nothing, while reading as though the gap were closed.

> **Countermeasure.** A fix must be stated as a delta from the measured current
> value, not as an absolute. "Set X to Y" hides this; "X is currently Y, change it
> to Z" makes it impossible to write.

### 4. Querying the wrong system, then reporting absence as disproof — *mine*

The local model reported a drift alert firing since a precise timestamp with a
precise object count. I queried the metrics store's `ALERTS` series, found nothing
over 25 hours, and **called the finding fabricated**.

The alert rules live in the dashboard layer's unified alerting, which does not
write `ALERTS` series into the metrics store. Checked correctly, the alert had been
firing since **2026-08-01T11:02:20** with **6** objects — the exact timestamp and
the exact count reported.

I had reached for the strongest possible verdict on the weakest possible evidence:
absence, from an instrument that could not have seen the thing.

> **Countermeasure.** Absence of evidence is a claim about the instrument. Before
> reporting "not found", show the instrument can see a positive case.

### 5. A scanner that reported clean by failing silently — *mine*

I scanned a repository's full history for leaked identifiers with `grep` over
`git log -p` output. Every pattern returned zero, including the leak patterns.

Every pattern also returned zero for strings I *knew* were present. The history
contained NUL bytes, so `grep` classified the stream as binary and emitted nothing.
A security scan reporting "clean" because it had silently stopped looking.

Rewritten to read bytes directly, the same history yielded live host identifiers.

> **Countermeasure.** Every scanner asserts a positive control — a token known to
> be present must match — and exits non-zero if the control fails. "Zero hits" and
> "the scanner did not run" must never be the same output.

### 6. A gate scoped to the tree, guarding a risk that lives in history

A redaction pass ran as an ordinary commit and was validated by a CI gate that
checks the working tree. Both passed. Both were correct about the tree.

Identifiers inside files that had been *deleted before the redaction ran* never
entered the tree, so no tree-scoped check could see them — and they remained
trivially retrievable with `git log -p`. The redaction commit's own diff contained
the values it was removing.

> **Countermeasure.** Enumerate history directly — every blob, every commit
> message — never the checkout. And a rewrite is not published until a fresh clone
> of the published bytes is re-scanned.

### 7. Confident prose with no instrument behind it — *mine*

I wrote a public README asserting the repository held **231** source files, **110**
test files, **more than a year** of history, and an **MIT** licence.

Measured: **225**, **107**, **5.5 months**, and **Apache-2.0 since the initial
commit**. Four claims, four wrong, in the one document whose explicit argument was
that the reader should check the history themselves. A second agent caught all four
by reading the tree.

The licence error is the instructive one: it was not a rounding slip but an
invented fact about a legal document that had never been opened.

> **Countermeasure.** Numbers in prose are claims. Generate them or omit them.

### 8. Instructions carrying unverified values — *mine*

I gave the operator a command to run containing `PASSWORD='...'` as an unmarked
placeholder, under a heading that said "run these." He ran it verbatim and received
an opaque authentication failure.

Compounding it: I had never checked the value was obtainable. The location the
repository documented for it was stale, and the credential was not there either.
The correct output was never a command — it was "this needs a value I cannot
confirm exists."

> **Countermeasure.** Never hand someone a command containing a value you have not
> verified they can obtain. If you cannot confirm it, the missing thing *is* the
> report.

---

## What the failures had in common

Seven of the eight were the same shape: **an instrument aimed at the wrong
surface.**

| failure | instrument | surface it should have measured |
|---|---|---|
| unit mismatch | occurrences | the same unit on both sides |
| namespace mismatch | mirror names | canonical image identity |
| wrong system | metrics store | dashboard alerting |
| binary-blind scan | text grep | bytes |
| tree-scoped gate | working tree | full history |
| unsourced prose | recollection | the repository |
| unverified command | assumption | the credential store |

The eighth — the self-negating fix — is a semantic error rather than an
instrumentation one, and I am not going to force it into the pattern. Seven of
eight is the honest number.

**None of these look like the failure mode people expect from language models.**
Nothing was invented from nothing. Every wrong claim was the faithful output of a
real measurement, taken against something adjacent to the question. Which means the
defence is not "check whether the model is hallucinating" — it is **check whether
the measurement addresses the claim.**

The corollary that matters for anyone running agents against production: the work
was mostly good. Real vulnerabilities were closed, real images mirrored, real
redaction applied. What could not be trusted was the *report of the work*. An agent
fleet does not primarily need better engineers; it needs an evidence discipline that
its own output has to survive.

---

## What changed as a result

Each countermeasure above was implemented, not just written down:

- The CVE figure published was the **less flattering** of the two defensible
  numbers, with its scope stated in the commit message.
- The 52-image retirement was reverted and the comparison rewritten to normalise
  registry prefixes before matching.
- The history scanner was rewritten to read bytes and to **exit non-zero when its
  own positive control fails**, so a broken scan can no longer look like a clean
  one.
- Redaction procedure now enumerates history directly and re-verifies against a
  fresh clone of the published bytes.
- A monitoring gap found while auditing — the estate's auto-unseal root of trust
  was scraped by nothing at all — was closed with a probe and three alert rules,
  each verified *firable* by substituting the failure value, because this codebase
  had previously shipped three alert rules that were structurally incapable of
  firing and had let an 80-minute outage pass unnoticed.

That last one is the point of doing error analysis at all. It was not on the task
list. It surfaced because verifying someone else's finding meant reading the actual
state of the system, and the actual state contained something nobody had asked
about.

---

## Why this is the portfolio piece and the incidents are not

An incident record shows you can recover a platform. This shows you can **run AI
agents against production and tell the difference between work that is done and
work that is reported done** — which is the harder skill, and the one that decides
whether an organisation can safely scale agent-directed engineering at all.

The estate's operating rule predates this analysis and survives it unchanged:
*verify live, don't trust reports.* What this adds is the specific shape the
failures take, so the verification can be aimed at the right surface instead of
performed as a ritual.
