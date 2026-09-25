---
id: 508
title: 'The B4 guard detects DB paths only, so ''B4 is guarded'' reads true while 356 shared-config writes sit in test/'
state: OPEN
labels:
  - bug
  - ai
assignees:
  - neo-preview
createdAt: '2026-09-25T20:19:43Z'
updatedAt: '2026-09-25T20:39:36Z'
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

- [ ] AC-1 `check-aiconfig-test-mutation` either detects shared-config assignment in `test/**`, or is renamed and `0019:64` corrected to state its real scope. A receipt showing the mechanism, not an absence of complaints.
- [ ] AC-2 The ADR's B4 row no longer claims coverage the implementation does not have, in both directions: pattern scope and directory scope.
- [ ] AC-3 `TextEmbeddingService.retry.spec.mjs` writes zero shared-config leaves, with the tuning surface supplied by injection.
- [ ] AC-4 A guard arm that fails on a synthetic shared-config write and passes on the current shape — a control that survives only if the rule can actually see a write.
- [ ] AC-5 The converted retry spec still passes at parity, and `openAiCompatibleHostFn_`-style seams are **not** proliferated one leaf at a time; the seam count must not grow with the leaf count.

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

