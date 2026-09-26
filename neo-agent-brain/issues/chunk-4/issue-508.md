---
id: 508
title: 'The B4 guard detects DB paths only, so ''B4 is guarded'' reads true while 356 shared-config writes sit in test/'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-preview
createdAt: '2026-09-25T20:19:43Z'
updatedAt: '2026-09-26T08:58:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/508'
author: tobiu
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-26T08:58:18Z'
---
# The B4 guard detects DB paths only, so 'B4 is guarded' reads true while 356 shared-config writes sit in test/

## Context

Raised out of PR #501's Round 2 (@neo-opus-vega, review `5322011589`), where I had claimed the B4 guard "never flagged the writes it exists to catch." That claim was **wrong in its mechanism**, and the truth is worse than what I wrote.

While discharging #501's B4 half I ran `ai/scripts/lint/check-aiconfig-test-mutation.mjs` before and after removing four shared-config writes and got the same answer both times:

```
check-aiconfig-test-mutation: 825 test file(s) scanned, 0 new violations.
```

I read that as the guard failing to catch what it exists to catch. It is not failing. **It was never looking.**

## The Problem

`ADR 0019`'s forbidden-pattern table (`learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md:64`) states:

> **B4 ⭐** | **SAFETY-CRITICAL — runtime writes to `AiConfig`** (see §4) | `[guarded: check-aiconfig-test-mutation]` — scans `test/**` only, so `ai/**` remains unenforced (§4)

The implementation of that guard implements a strictly narrower rule:

- `check-aiconfig-test-mutation.mjs:407` — `const B4_RULE = ADR_0019_RULES[0]`
- `:404` — that rule's `detect` is `findDbPathMutations`
- `:290` — `findDbPathMutations` matches only `DB_PATH_MUTATION_GLOBAL`
- `:430` — the result field is literally named `dbPathHits`

So the rule carrying the **B4** id detects **DB-path** writes. `grep -cE "openAiCompatible|aiConfig\."` over the guard's own source returns **2** — incidental, not detection logic. Nothing in it recognises a shared-config write as such.

**Consequence:** anyone auditing B4 against the ADR's own table gets a false assurance. The ADR honestly discloses the guard's *directory* gap (`ai/**` unenforced); it does not disclose the *pattern* gap, which is the larger one.

## The Architectural Reality

The surface the guard cannot see, measured in `test/**` at this head:

- **356** `aiConfig.*` assignments across **81** distinct config leaves.
- `TextEmbeddingService.retry.spec.mjs` alone: **80** assignments across **12** distinct leaves — `openAiCompatible.{host, embeddingModel, unloadRetryCount, unloadRetryDelayMs, contentionRetryCount, contentionRetryDelayMs, contentionTimeoutMs, batchEmbeddingChunkSize, batchEmbeddingTimeoutMs, batchEmbeddingYieldMs}`, plus `localModels.embedding.parallel` and `orchestrator.lms.port`.
- Only 3 specs mutate `openAiCompatible`: this one, `checkAiConfigTestMutation.spec.mjs` (the guard's own spec, which necessarily contains the patterns), and `RemDigestion.spec.mjs`.

ADR-0019 §4 says the danger is mechanical, not stylistic: the hierarchical data proxy's set-trap **routes assignments to the owning provider's `setData`**, so a write to the shared singleton propagates. A spec that mutates it is therefore a cross-test hazard by construction — a missed restore, a shared process, or plain test order hands the next consumer the previous test's endpoint. #501 hit exactly this and the fix was an injectable consumer seam (`TextEmbeddingService.openAiCompatibleHostFn_`).

The retry spec's shape is *not* careless — its `beforeEach` sets the tuning surface and its `afterEach` restores ten of the twelve leaves, which is a legitimate pattern. The defect is only that it mutates the singleton instead of injecting, and that at 12 leaves a per-leaf seam is the wrong answer: the right shape is an injected config snapshot (or a per-instance overlay) for the whole surface.

## The Fix

Two halves, and the order matters — the guard first, or the conversion has nothing holding it in place.

1. **Close the pattern gap.** Either generalise the rule to detect shared-config assignment in `test/**` and report the real count, or — if a general rule is judged too noisy to land at once — rename the id so it stops overclaiming, and correct `0019:64` to say what is actually enforced. **A lint whose label reads broader than its detection is worse than no lint**, because it converts an audit question into a false assurance.
2. **Convert `TextEmbeddingService.retry.spec.mjs`** to an injected config snapshot for its 12 leaves, once the rule above can see whether the conversion actually held.

If the general rule lands first it will report ~356 pre-existing hits; those need an explicit, justified baseline rather than a silent grandfather, or the rule will be muted the moment it is inconvenient.

## Acceptance Criteria

- [x] AC-1 `check-aiconfig-test-mutation` either detects shared-config assignment in `test/**`, or is renamed and `0019:64` corrected to state its real scope. A receipt showing the mechanism, not an absence of complaints.
- [x] AC-2 The ADR's B4 row no longer claims coverage the implementation does not have, in both directions: pattern scope and directory scope.
- [ ] AC-3 `TextEmbeddingService.retry.spec.mjs` writes zero shared-config leaves, with the tuning surface supplied by injection. → **moved to #541** (phase 2; not delivered here)
- [x] AC-4 A guard arm that fails on a synthetic shared-config write and passes on the current shape — a control that survives only if the rule can actually see a write.
- [ ] AC-5 The converted retry spec still passes at parity, and `openAiCompatibleHostFn_`-style seams are **not** proliferated one leaf at a time; the seam count must not grow with the leaf count. → **moved to #541** (phase 2; not delivered here)

**Scope note.** This ticket covers the DETECTION half only. AC-3 and AC-5 are the CONVERSION half — they change specs, not the guard — and were split to #541 before this PR opened, so closing this ticket cannot close the only pointer to them.

## Out of Scope

`ai/**` runtime enforcement (the ADR's own disclosed directory gap, separate decision). The #480 identity-guard work, which is delivered in #501. `RemDigestion.spec.mjs` and the guard's own spec, which are instances for a later pass once the rule can see them.

## Related

#480 (the served-model identity guard whose B4 half surfaced this) · PR #501 · `checkAiConfigTestMutation.spec.mjs` · `TextEmbeddingService.openAiCompatibleHostFn_`, added in #501 as the first consumer seam.

Origin Session ID: d19add67-d33c-489d-99aa-27ad2782ed5e



## Timeline

- 2026-09-25T20:19:45Z @tobiu added the `bug` label
- 2026-09-25T20:19:45Z @tobiu added the `ai` label
- 2026-09-25T20:20:02Z @tobiu cross-referenced by PR #501
- 2026-09-25T20:39:36Z @tobiu assigned to @neo-preview
- 2026-09-25T20:45:32Z @neo-opus-vega cross-referenced by #509
- 2026-09-25T22:15:18Z @neo-preview cross-referenced by PR #525
- 2026-09-26T07:15:30Z @tobiu cross-referenced by PR #529
- 2026-09-26T07:22:11Z @neo-preview cross-referenced by #532
- 2026-09-26T07:30:09Z @neo-preview referenced in commit `04db28d` - "fix(lint): B4's two halves share one catalog id, and the ownership check is pinned (#508)

The red-first spec I wanted for the ids is the one that was missing: a rule id
that is executable and self-consistent can still be unexpressible in the ADR
catalog it claims to serve. `lint-config-template-ssot.mjs` parses catalog id
cells with `/\b([A-C]\d+)\b/` and resolves every registry id back to a row, so
`B4-DB-PATH` parses as `B4` and a second suffixed rule collides into
`duplicate-row`. Splitting B4's halves was right; suffixing their ids was not.

The halves now share the id `B4` — one antipattern, one catalog row, one key —
and are told apart by a `scope` field plus the `gating` flag the scanner already
branches on. The ADR row keeps the honesty this PR exists for: the DB-path
subset GATES, the full scope is REPORT-ONLY, `ai/**` stays unenforced. Its tag
was also `[gated: …]`, which the parser does not read, so the row parsed as
zero guards and `unverifiable-tag` fired; the parser only accepts `[guarded: …]`.

Two specs, because the drift cost a 44s lint job in CI and a peer had to report
it:

- the two-way ownership relation, asserted against the real ADR source (0
  violations) — milliseconds instead of a CI round-trip;
- a non-vacuity arm feeding the same check the suffixed ids this PR originally
  shipped, which must red in both directions (`overstates-enforcement` on the
  row, `guard-id-missing-from-adr` per half). An arm asserting only `[]` passes
  just as happily against a validator that stopped checking.

48/48 in the guard spec; `lint-config-template-ssot` reports 0 ownership
mismatches.

The substantive part of this PR is unchanged: the guard that carried the B4 id
detected DB paths only, so auditing "is B4 enforced?" against the ADR's own table
returned a false assurance. It now scans the full clause, prints its measured
count on every run, and says in the table which half is not yet gating."
- 2026-09-26T07:37:25Z @neo-preview referenced in commit `a6990bb` - "docs(lint): the catalog-key rule reads as mechanism, not provenance (#508)

The ownership mechanism is unchanged and still pinned by the specs; only the prose
changes. A durable comment that cites the decision record by number decays into
a second, unmaintained copy of the rule — the check-ticket-archaeology gate says
so, and it caught three such refs I had just introduced in this same PR while
documenting a different mechanism."
- 2026-09-26T08:18:22Z @neo-preview cross-referenced by #541
- 2026-09-26T08:20:10Z @neo-preview referenced in commit `8bc38dd` - "fix(lint): the B4 detector sees Object.assign too, and a `full` label may not outrun it (#508)

@neo-opus-ada's Round 1, four RAs, all correct. The second one is this PR's own
thesis aimed at this PR's own rule: I widened the detector because a label
broader than its detection is a false assurance, then shipped a `scope: 'full'`
whose detection was also narrower than its label.

`Object.assign(<config root>, …)` is a write. Every key of that call is still a
`[[Set]]` on the hierarchical proxy and still routes to the owning provider's
`setData`, so the previous grammar — which required `=` after the leaf — could
not see it. It is now its own pattern rather than an alternation on the
assignment pattern, because a bare `,<object literal>` alternative also matches
any call that merely PASSES a config value beside a literal. That was my first
implementation and it produced 177 false positives on a count that looked
entirely plausible; anchoring on the callee is the only thing that separates the
write from the read, and an arm now pins it.

Each hit carries its `form`, and the guard prints the split rather than a bare
total: one `assign-call` can write many leaves, so a per-hit count is a lower
bound on leaves touched, never a census. The reviewer's independent `git grep`
measured 57 such sites in 12 files; the detector reports 57. Two instruments
agreeing to the unit is the receipt, and neither number is a receipt alone.

The true total is therefore 655 / 150, not the 598 / 146 this branch previously
recorded — the earlier census was itself an undercount by exactly the form it
could not see. That is now stated in the PR body next to the earlier correction
rather than quietly replaced.

RA-3: the point-in-time census is gone from the B4 tag cell, the Status-row
amendment and the detector's comment. ADR 0019 §4's tag contract keeps figures
with the guard that can re-measure them, and a count in a tag is stale the
moment the surface moves. The PR body keeps its numbers, because a per-PR
measurement is evidence rather than durable prose — and they are current.

RA-4: "What this PR does" item 1 still named `B4-DB-PATH` after the head
stopped shipping it.

RA-1 is its own correction and the one I am least comfortable writing: a
`Resolves` on a ticket holding ACs this PR does not deliver is the exact mistake
I was corrected on with #510, repeated one session later. AC-3 and AC-5 are the
CONVERSION half — they change specs, not the guard — and are now #541, with
#508's body naming the successor, filed and assigned before this push rather
than promised under Post-Merge Validation.

Verified: guard spec 51/51 (four new arms — the assign form, the read/write
discrimination, the two-way ownership relation, and its non-vacuity feed);
`lint-config-template-ssot` 0 ownership mismatches; archaeology 0."
- 2026-09-26T08:58:18Z @tobiu closed this issue
- 2026-09-26T08:58:18Z @tobiu referenced in commit `7786f01` - "Merge pull request #525 from neomjs/agent/508-b4-guard-scope

fix(lint): the B4 guard stops claiming scope it did not enforce (#508)"

