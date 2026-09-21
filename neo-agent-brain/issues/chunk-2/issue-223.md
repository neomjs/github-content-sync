---
id: 223
title: 'A repo that has never once succeeded reports uninitialized, not failed'
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - architecture
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-08-29T00:21:57Z'
updatedAt: '2026-08-29T09:59:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/223'
author: neo-opus-vega
commentsCount: 0
parentIssue: 64
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-29T09:59:02Z'
---
# A repo that has never once succeeded reports uninitialized, not failed

Leaf of neomjs/neo-agent-brain#64 (epic). Implementation open as PR neomjs/neo-agent-brain#221.

## Problem

`classifyTenantRepoCheckpoint` returns `UNINITIALIZED` for any state without a `lastIngestedRev`, and **that branch precedes every other return**:

```js
if (!normalizedState?.lastIngestedRev)          return UNINITIALIZED;   // ← wins first
if (ingestVersion === CONTRACT && committedId)  return COMPLETE;
if (attemptVersion === CONTRACT)                return FAILED;          // ← unreachable without a rev
```

So a repo that attempted at the current contract version and failed **every single time** can only report *"nobody has started this."* `FAILED` is reachable only for repos that previously succeeded — the narrower population, and the opposite of where the status matters.

**Live specimen (2026-08-20):** `b17f44b388a1`, `lastIngestedRev: null`, **41 consecutive failures**, `stopReasonCode: KB_VECTOR_EMBED_INPUT_TRUNCATED`, reporting `checkpointStatus: uninitialized`.

`uninitialized` is the reading that invites no investigation. Collapsing the two hid the alarming state behind the boring one.

## Architectural reality

`lastAttemptedIngestContractVersion` already carries the distinction and is populated **independently** of `lastIngestedRev`. The classifier's own JSDoc describes using it exactly this way for the has-a-rev case; the absent-rev case never inherited the same rule.

🔴 **A downstream invariant is load-bearing only because of this defect.** `TenantRepoSyncService`'s revalidation-deferred row dereferences `priorState.lastIngestedRev.slice(0, 8)` unguarded. That is safe today **only** because `requiresTenantRepoCheckpointRevalidation` is `PENDING || FAILED` and both are unreachable without a rev — so the field is guaranteed to be a string. Fixing the classifier breaks that invariant and the line throws on null, **but only once the per-sweep admission cap binds** — under contention, never in a light test. Five sibling rows already guard the same field; this one does not.

**The reclassification also closes a scheduling gap rather than opening one.** The admission cap exists to bound *"automatic null-base replays"* per sweep. A never-succeeded repo performs one every sweep and, while classified `uninitialized`, bypassed that bound entirely — brand-new repos are spread by jitter, legacy revalidations by the cap, and this population by neither.

Replay semantics are provably unchanged: the envelope's `lastIngestedRev` is `fullReplay || revalidationRequired ? null : priorState?.lastIngestedRev || null`, and this population has no rev, so **both branches already yield `null`**.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `classifyTenantRepoCheckpoint` | the existing `lastAttemptedIngestContractVersion` contract | a null-rev checkpoint classifies `failed` when the current contract was attempted | a genuinely untouched repo stays `uninitialized`; a prior-contract attempt stays `uninitialized` | classifier JSDoc | both polarity arms as focused cases |
| `TenantRepoSyncService` revalidation-deferred row | the five sibling rows that already guard | `lastIngestedRev` guarded like every sibling | a newly reachable null-rev repo defers with `null` rather than throwing | in-line comment naming the retired invariant | sibling-parity read |
| `requiresTenantRepoCheckpointRevalidation` | the admission cap's stated purpose | the reclassified population becomes revalidation-eligible | — | — | explicit assertion on both arms |

## Acceptance criteria

- [ ] A repo that attempted at the current ingest contract and never succeeded reports `failed`.
- [ ] A repo nobody has started still reports `uninitialized` — the fix must not trade one wrong reading for another.
- [ ] An attempt at a **prior** contract version does not count as a current-contract failure.
- [ ] Every field newly reachable by the reclassification is guarded; specifically the revalidation-deferred row cannot throw on a null rev.
- [ ] Focused cases cover both polarity arms plus revalidation eligibility, and run **locally** — Brain CI's `unit` check does not execute this spec (`brain-unit.yml` runs `--list` plus a three-spec smoke).

## Out of scope

- **Scheduling fairness** — #64 AC-1, and its premise is separately in question: the live starvation has *no* lease holder while the AC is worded around a long-running holder.
- **`status: completed` over a null `lastIngestedRev`** — the other half of #64's reporting AC. Traced separately: the sweep verdict already routes the all-nothing-landed case to `deferred`, and the mixed-sweep residual is a documented deliberate choice.
- Changing the admission cap's own population rules beyond what the reclassification implies.

## Avoided traps

- **Treating this as a reporting-layer bug.** It is a check *order* in a pure classifier; nothing downstream was mislabelling anything.
- **Shipping the classifier change alone.** The null-deref above is reachable only after the fix, and only under contention — the "small local fix" ships a crash.
- **Weakening a pinning assertion to make room for a new case.** The spec's order-exact sequence assertions are the thing, not the obstacle.

**Live latest-open sweep:** checked the latest 20 open issues in `neomjs/neo-agent-brain` at 2026-08-29T00:20:47Z, created-descending; no equivalent found. **A2A in-flight claim sweep:** 25 most recent messages scanned by recency and scope (not read-status) across the ~60-minute herd window; the live claims are #212 (@neo-gpt-emmy), DockLayouts #17836 (@neo-gpt), Institution #46 (@neo-fable-clio) and #17839/#17840 (@neo-opus-ada) — none overlapping.

Origin Session ID: 96836c41-0a29-415d-aa36-6ac60b81c782

Retrieval Hint: `query_raw_memories("tenant repo checkpoint uninitialized vs failed check order lastAttemptedIngestContractVersion null rev deferral guard")`


## Timeline

- 2026-08-29T00:21:57Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-08-29T00:23:27Z @neo-opus-vega cross-referenced by PR #221
- 2026-08-29T00:27:06Z @neo-opus-vega cross-referenced by #23
- 2026-08-29T00:27:24Z @neo-gpt-emmy added the `bug` label
- 2026-08-29T00:27:24Z @neo-gpt-emmy added the `ai` label
- 2026-08-29T00:27:24Z @neo-gpt-emmy added the `testing` label
- 2026-08-29T00:27:24Z @neo-gpt-emmy added the `architecture` label
- 2026-08-29T00:27:24Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-29T09:59:02Z @tobiu closed this issue
- 2026-08-29T11:37:10Z @neo-opus-vega cross-referenced by #233
- 2026-08-29T16:52:22Z @neo-gpt cross-referenced by PR #234

