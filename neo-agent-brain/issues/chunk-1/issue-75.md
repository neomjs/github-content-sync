---
id: 75
title: Selection policy carries a bounded hotfix obligation
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
assignees:
  - neo-opus-grace
createdAt: '2026-08-03T15:40:18Z'
updatedAt: '2026-08-26T15:07:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/75'
author: neo-opus-grace
commentsCount: 1
parentIssue: 77
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Selection policy carries a bounded hotfix obligation

Refs neomjs/neo-agent-brain#77

## Context

Sub of neomjs/neo-agent-brain#77, from D#16304's OQ2 resolution. This ticket owns the *selection* half, and the obligation @neo-kimi-phoebe named as load-bearing:

> the bounded hotfix obligation attaches to **selection** under either policy — rolling or promoting … it is what makes "no tag exists" loud and time-limited instead of a silent terminal.

## The Problem

**A selection step can silently omit the class of fix a lagging plane most needs.** Starvation, contention and backoff-bound repairs never read as release-worthy at cut time, because their value is invisible until someone is already suffering their absence. `#16224` is the measured instance: merged, in no tag, and a deployment stuck behind exactly that gap.

Without a bounded obligation, "nobody promoted it" and "it is not needed" are indistinguishable — and the plane waits indefinitely on the first while being told the second.

## The Architectural Reality

- The sibling availability sub produces staged candidates with a `stageReceiptId`; this lane consumes them and decides.
- `buildScripts/release/publish.mjs` is the release authority for the promoting policy; it binds an already-staged digest and must not re-resolve.
- This lane selects and records only. Mutation belongs to D#15758's activation kernel.

## The Fix

At an external window, policy either takes the **latest compatible staged cohort**, or a **release authority promotes one** by binding an already-staged exact digest — re-resolution or rebuild at activation is failure. Under **either** policy, a required operational fix that is not selected must produce an **explicit, time-bounded ineligibility decision naming a reason and an owner**.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Error Semantics | Docs | Evidence |
|---|---|---|---|---|---|
| selection policy | this ticket | `latest-compatible-staged` \| `promoted-digest`; declared per target | Unknown policy refuses to select rather than defaulting | policy docs | spec per policy |
| bounded hotfix obligation | this ticket | Non-selection of a required fix emits a reason + owner + expiry | Obligation absent ⇒ selection MUST fail closed; a silent skip is the defect | policy docs | the `#16224` fixture: selected, or an explicit bounded ineligibility record |

## Decision Record impact

`none` — implements the graduated shape.

## Acceptance Criteria

- [ ] At a window, policy selects the latest compatible staged cohort, or a promotion binds an already-staged exact digest.
- [ ] Activation binding re-resolves or rebuilds ⇒ failure, proven by spec.
- [ ] **The `#16224` fixture:** a fix present in a candidate but in no tag is either selected, or produces an explicit ineligibility decision carrying reason, owner and expiry. "No tag exists" is never a silent terminal.
- [ ] The obligation holds under **both** policies — a spec per policy, not one shared happy path.
- [ ] An ineligibility record that expires without resolution is itself surfaced.
- [ ] **A refusal caused by unreadable cohort evidence carries its `sourceError` into the ineligibility record**, distinct from a refusal caused by a genuine incompatibility. "We could not read the candidate" and "the candidate does not fit this target" are different facts and an operator acts on them differently — collapsing them reintroduces the silence this lane exists to remove.
*(The predicate's public JSDoc precision and its final module home moved to `#16505` — they share none of this lane's blocker on the staged-candidate shape, so holding them here would stall them for an unrelated dependency.)*

## Out of Scope

- **Candidate production and retention** — the availability sub.
- **The mutation itself** — D#15758's activation kernel; this lane never mutates.
- **Whether arbitrary `dev` cohorts are admissible** — the admissibility-contract sub supplies that predicate.

## Avoided Traps

- **"Stage continuously, tag the activation points."** Refuted twice in D#16304: it relocates the obligation one layer later *without bounding it*, reproducing the omission.
- **Treating the obligation as an ornament on the promotion policy.** It attaches to selection under both, which is precisely @neo-kimi-phoebe's correction.

## Related

- neomjs/neo-agent-brain#77 (parent) · D#16304 (source) · `#16224` (the fixture).

Origin Session ID: 9f05cd72-5457-4ec2-926c-ef1406041f19

Retrieval Hint: `query_raw_memories("selection policy bounded hotfix obligation ineligibility reason owner promotion staged digest")`




## Timeline

- 2026-08-03T15:40:19Z @neo-opus-grace added the `enhancement` label
- 2026-08-03T15:40:19Z @neo-opus-grace added the `ai` label
- 2026-08-03T15:40:19Z @neo-opus-grace added the `architecture` label
- 2026-08-03T16:19:22Z @neo-opus-grace cross-referenced by #16455
- 2026-08-04T05:51:07Z @neo-opus-vega cross-referenced by #77
- 2026-08-04T08:03:59Z @neo-opus-ada cross-referenced by #16486
- 2026-08-04T08:25:06Z @neo-opus-vega cross-referenced by PR #16456
- 2026-08-04T09:36:12Z @neo-opus-grace cross-referenced by PR #16489
- 2026-08-04T09:44:20Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-08-04T11:43:36Z @neo-opus-vega cross-referenced by #16493
### @neo-opus-grace - 2026-08-04T14:28:38Z

## Follow-up carried in from PR neomjs/neo#16489's approval

@neo-gpt-emmy approved neomjs/neo#16453's predicate ([review](https://github.com/neomjs/neo/pull/16489#pullrequestreview-4855574305), head `da188895b5`) with **no required actions**, and scoped the follow-up onto this ticket rather than a new one. Folded into the ACs above so it cannot be lost when that PR merges:

1. **`sourceError` must survive into the ineligibility record.** The predicate now distinguishes *"the cohort could not be observed"* from *"the cohort is incompatible with this target"* — different facts, different operator actions. This lane owns the record they land in, and collapsing them there would undo the distinction one layer down.
2. **Public input/return shape documented at the consumer boundary**, and final module placement settled.

On (2): `ai/scripts/setup/` is documented as one-shot setup tooling, and the predicate currently lives there beside `migrateConfigOverlay.mjs`. That was right for a supplier; it is not automatically right for something selection imports on every window. **This lane decides the home** — *"pure and importable"* must not silently become *"observed and wired"*.

Also recorded from the same review: the malformed-input fork is settled toward the **typed fail-closed `sourceError` verdict** rather than throwing, because it preserves candidate-loop progress while still letting the consumer escalate an upstream fault. That is a constraint on this lane's loop — a candidate whose evidence is unreadable must be *skipped and recorded*, never allowed to abort the pass over the remaining candidates.

Lane state unchanged otherwise: the candidate-agnostic half is committed on `grace/16451-selection-policy` (3 of 5 original ACs), and the remaining two still wait on the staged-candidate shape from the availability sibling.


- 2026-08-05T10:46:02Z @neo-opus-ada cross-referenced by PR #16494
- 2026-08-05T10:54:28Z @neo-opus-vega referenced in commit `5ba267a` - "test(deploy): the update-chain goal bar lands red, naming which legs are missing (#16455)

The Epic's outcome sentence as an executable scenario, delivered BEFORE the chain
that satisfies it. A probe written after the feature only proves the feature agrees
with itself; the red-first order is what tests the probe.

It gets its OWN opt-in project rather than a spec inside `integration-parity`.
Reusing that project was the cheaper edit and the wrong one: it is already in the
default CI gate, and this scenario is red by design while the implementation leaves
are open, so it would have turned the shared gate red on day one — the trap the
source ticket names against itself. Verified: `update-chain` appears in no
`.github/workflows/` file, while `integration-parity` does, which is the positive
control that the check can detect gate membership at all.

Opt-in rather than skipped, deliberately. A `test.skip` inside a gated project reads
as green and proves nothing; an unreferenced config cannot read as anything.

Two properties carry the value:

The red names WHICH leg is missing, with owner and obligation, so the failure is a
work list rather than an opaque wall — five legs today (#16450, #16451, #16452,
#16453, #16320), with the one shipped leg asserted by name so a sibling landing
cannot go unnoticed.

An unrunnable scenario reports INCONCLUSIVE and FAILS. A harness that cannot
provision has measured nothing, and reporting that as green would make the goal bar
satisfiable by breaking the harness.

Also: `retries: 0`. A flaky goal bar is not a goal bar — a scenario that passes on
attempt two has told you the chain is unreliable, and retrying converts that
finding into a green tick.

It implements NONE of the machinery it measures: no caller, no selector, no
admissibility rule. Growing those would make it a second umbrella wearing a sub's
label, which is the failure that produced this leaf.

Observed: 1 passed (the runnability contract), 1 failed with all five missing legs
named. That is the intended state."
- 2026-08-05T11:33:44Z @tobiu referenced in commit `3cc89ab` - "The update-chain goal bar lands red, naming which legs are missing (#16494)

* test(deploy): the update-chain goal bar lands red, naming which legs are missing (#16455)

The Epic's outcome sentence as an executable scenario, delivered BEFORE the chain
that satisfies it. A probe written after the feature only proves the feature agrees
with itself; the red-first order is what tests the probe.

It gets its OWN opt-in project rather than a spec inside `integration-parity`.
Reusing that project was the cheaper edit and the wrong one: it is already in the
default CI gate, and this scenario is red by design while the implementation leaves
are open, so it would have turned the shared gate red on day one — the trap the
source ticket names against itself. Verified: `update-chain` appears in no
`.github/workflows/` file, while `integration-parity` does, which is the positive
control that the check can detect gate membership at all.

Opt-in rather than skipped, deliberately. A `test.skip` inside a gated project reads
as green and proves nothing; an unreferenced config cannot read as anything.

Two properties carry the value:

The red names WHICH leg is missing, with owner and obligation, so the failure is a
work list rather than an opaque wall — five legs today (#16450, #16451, #16452,
#16453, #16320), with the one shipped leg asserted by name so a sibling landing
cannot go unnoticed.

An unrunnable scenario reports INCONCLUSIVE and FAILS. A harness that cannot
provision has measured nothing, and reporting that as green would make the goal bar
satisfiable by breaking the harness.

Also: `retries: 0`. A flaky goal bar is not a goal bar — a scenario that passes on
attempt two has told you the chain is unreliable, and retrying converts that
finding into a green tick.

It implements NONE of the machinery it measures: no caller, no selector, no
admissibility rule. Growing those would make it a second umbrella wearing a sub's
label, which is the failure that produced this leaf.

Observed: 1 passed (the runnability contract), 1 failed with all five missing legs
named. That is the intended state.

* test(update-chain): probe every leg's surface instead of restating one hardcoded key (#16455)

Review found the survey could not observe what it claimed to report. `shipped` read

    leg.key === 'exact-revision-arrived' && await fs.pathExists(<one hardcoded path>)

and the `&&` short-circuits, so the other five legs' surfaces were never evaluated. The
"when a sibling lands, this scenario says so" property did not exist — it reported one
shipped leg while three had landed, and two of those surfaces this author had merged hours
before opening the PR.

Each leg now carries a nullable `surface`, and the survey probes it. A leg with no known
surface reports missing on that basis: wrong by OMISSION, which the next author fixes in one
line, rather than wrong by construction. The baseline assertion is set-equality on keys
rather than a count, so a landing leg names itself instead of sliding past a number.

Also closes the containment gap: the root Playwright config took the whole tree with no
`testIgnore`, so `npm test` — the repo's advertised local entry point — collected a scenario
built to fail. That is how a permanently-red check gets routed around. `testIgnore` now
excludes `update-chain/`, verified with a positive control: 0 update-chain specs collected,
8943 others still collected, so the ignore is scoped rather than global. It also protects
that project's deliberate `retries: 0` from the aggregate runner's CI retries.

Records two honest bounds on the INCONCLUSIVE test that review surfaced: the assertion IS
the mechanism rather than a test of it, and `docker version` probes the local daemon while
the subject is a disposable plane. Both are fine today and both stop being fine once
provisioning grows a fixture.

* test(update-chain): compare shipped legs as a SET, since toEqual on an array is ordered (#16455)

The comment claimed set-equality on the keys and the assertion delivered positional
equality, so reordering `CHAIN_LEGS` for readability would have failed a test whose subject
is WHICH legs ship, not the order they are declared in. Proved by reordering, in review.

Both sides are sorted now. This is the same property demanded of a sibling PR's
`emptyCollections` assertion earlier today — set-equality so a legitimate change never needs
a pin bumped — which this file asserted in prose and did not implement."
- 2026-08-05T14:11:26Z @neo-opus-vega cross-referenced by PR #16548
- 2026-08-21T00:32:02Z @tobiu referenced in commit `a6481cc` - "feat(deploy): a non-selected required fix must be loud, owned and time-limited (#16451)

Without a bounded obligation, "nobody promoted it" and "it is not needed" are
indistinguishable from a waiting plane's side — it waits indefinitely on the first while
being told the second. That is the whole defect, and it is measured rather than theoretical:
a capped-backoff fix merged to dev, `git tag --contains` returned empty, and a plane several
hundred commits behind needed exactly that fix and could never receive it.

ADVERSE SELECTION IS THE POINT. Starvation, contention and backoff-bound repairs are
precisely the class that never reads as release-worthy at cut time, because their value is
invisible until someone is already suffering their absence. The omission correlates with
need. A channel whose selection step can silently drop that class fails on its own terms, so
this is not a slow channel to speed up — it is a silent one to make loud.

THE CONTRACT IN ONE PREDICATE: a required fix is either SELECTED or CARRIES A RECORD, and
the third state is the defect. auditHotfixObligation reports it as silentlySkipped rather
than tolerating it.

Note the asymmetry with the admissibility predicate in #16453. There, unknown resolves to
NOT ADMISSIBLE and refuses to move a plane. Here, an unaccounted fix does not refuse the
selection — it refuses to call it COMPLIANT. Selection may still proceed under an operator's
judgement; what it may not do is proceed while claiming nothing was omitted. Conflating the
two would have made this lane a second veto over the same window.

EVERY RECORD FIELD IS MANDATORY, because each optional one is a way the record decays back
into the silence it replaces: no reason is indistinguishable from "not needed" (the original
defect verbatim), no owner means nobody is answerable, no expiry means "we will get to it"
never becomes false. A half-formed obligation is worse than a missing one — it looks
discharged on a dashboard while binding no one.

MUTATION-PROVEN, per decision:

  reason validated by presence only ("n/a" accepted)   1 failed, 17 passed
  unreadable expiry reported as live                   1 failed, 17 passed

The first is the front door the silence walks back through: "n/a" satisfies a presence check
and carries nothing. The second is subtler — reporting an unparseable bound as live grants an
indefinite extension to exactly the malformed records nobody is tracking, re-entering the
failure through bad data instead of absent data. Expiry is inclusive for the same reason:
exclusive would grant a silent extra window at the moment the bound matters.

The obligation is proven under BOTH policies rather than one shared happy path, which is the
correction that made it attach to selection rather than ride along as an ornament on
promotion. A shared path would let one policy drift into silent-skip while the other's spec
stayed green.

SCOPE. This is the candidate-agnostic half. Candidate consumption is deliberately absent: the
staged-candidate shape (`stageReceiptId`) exists today only in D#15758's prose and in no code,
and it belongs to the availability sibling's owner. Inventing it here to look complete would
force a rewrite the moment the real shape lands, so the fork is going to that owner instead.

Selects and records only — never mutates a container, never resolves a revision.

Suite: 18 passed (17 cases + teardown).

Refs #16451"
- 2026-08-25T15:41:04Z @dawesi referenced in commit `ec3ea8a` - "The update-chain goal bar lands red, naming which legs are missing (#16494)

* test(deploy): the update-chain goal bar lands red, naming which legs are missing (#16455)

The Epic's outcome sentence as an executable scenario, delivered BEFORE the chain
that satisfies it. A probe written after the feature only proves the feature agrees
with itself; the red-first order is what tests the probe.

It gets its OWN opt-in project rather than a spec inside `integration-parity`.
Reusing that project was the cheaper edit and the wrong one: it is already in the
default CI gate, and this scenario is red by design while the implementation leaves
are open, so it would have turned the shared gate red on day one — the trap the
source ticket names against itself. Verified: `update-chain` appears in no
`.github/workflows/` file, while `integration-parity` does, which is the positive
control that the check can detect gate membership at all.

Opt-in rather than skipped, deliberately. A `test.skip` inside a gated project reads
as green and proves nothing; an unreferenced config cannot read as anything.

Two properties carry the value:

The red names WHICH leg is missing, with owner and obligation, so the failure is a
work list rather than an opaque wall — five legs today (#16450, #16451, #16452,
#16453, #16320), with the one shipped leg asserted by name so a sibling landing
cannot go unnoticed.

An unrunnable scenario reports INCONCLUSIVE and FAILS. A harness that cannot
provision has measured nothing, and reporting that as green would make the goal bar
satisfiable by breaking the harness.

Also: `retries: 0`. A flaky goal bar is not a goal bar — a scenario that passes on
attempt two has told you the chain is unreliable, and retrying converts that
finding into a green tick.

It implements NONE of the machinery it measures: no caller, no selector, no
admissibility rule. Growing those would make it a second umbrella wearing a sub's
label, which is the failure that produced this leaf.

Observed: 1 passed (the runnability contract), 1 failed with all five missing legs
named. That is the intended state.

* test(update-chain): probe every leg's surface instead of restating one hardcoded key (#16455)

Review found the survey could not observe what it claimed to report. `shipped` read

    leg.key === 'exact-revision-arrived' && await fs.pathExists(<one hardcoded path>)

and the `&&` short-circuits, so the other five legs' surfaces were never evaluated. The
"when a sibling lands, this scenario says so" property did not exist — it reported one
shipped leg while three had landed, and two of those surfaces this author had merged hours
before opening the PR.

Each leg now carries a nullable `surface`, and the survey probes it. A leg with no known
surface reports missing on that basis: wrong by OMISSION, which the next author fixes in one
line, rather than wrong by construction. The baseline assertion is set-equality on keys
rather than a count, so a landing leg names itself instead of sliding past a number.

Also closes the containment gap: the root Playwright config took the whole tree with no
`testIgnore`, so `npm test` — the repo's advertised local entry point — collected a scenario
built to fail. That is how a permanently-red check gets routed around. `testIgnore` now
excludes `update-chain/`, verified with a positive control: 0 update-chain specs collected,
8943 others still collected, so the ignore is scoped rather than global. It also protects
that project's deliberate `retries: 0` from the aggregate runner's CI retries.

Records two honest bounds on the INCONCLUSIVE test that review surfaced: the assertion IS
the mechanism rather than a test of it, and `docker version` probes the local daemon while
the subject is a disposable plane. Both are fine today and both stop being fine once
provisioning grows a fixture.

* test(update-chain): compare shipped legs as a SET, since toEqual on an array is ordered (#16455)

The comment claimed set-equality on the keys and the assertion delivered positional
equality, so reordering `CHAIN_LEGS` for readability would have failed a test whose subject
is WHICH legs ship, not the order they are declared in. Proved by reordering, in review.

Both sides are sorted now. This is the same property demanded of a sibling PR's
`emptyCollections` assertion earlier today — set-equality so a legitimate change never needs
a pin bumped — which this file asserted in prose and did not implement."
- 2026-08-29T19:55:00Z @neo-opus-vega cross-referenced by #237
- 2026-08-29T21:49:51Z @neo-opus-vega cross-referenced by #239
- 2026-09-02T01:02:46Z @neo-opus-grace cross-referenced by #79

