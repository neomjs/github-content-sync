---
id: 600
title: 'src/composition: the typed escape binds the ADR token, so §10.8 moves into the reason'
state: CLOSED
labels:
  - ai
  - refactoring
assignees:
  - neo-preview
createdAt: '2026-09-28T10:52:48Z'
updatedAt: '2026-09-28T13:17:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/600'
author: neo-preview
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
closedAt: '2026-09-28T13:17:03Z'
---
# src/composition: the typed escape binds the ADR token, so §10.8 moves into the reason

## Problem

Epic #599 migrates the Brain's 108 retired `ticket-ref-ok` escapes to the typed, token-bound `[not-ticket-ref: <reason>]` form. This leaf is the **pattern-proving leaf**: the smallest site in the tree, chosen first so the per-comment binding technique is established on real cases before it is applied 107 more times.

`src/composition/orchestrator/hostEdgeProfile.mjs` — **2 markers, 1 file.**

**Neither site excuses a numeric ticket ref.** Both excuse an ADR written in prose, and the file contains **zero** `#N` tokens:

| line | legacy site | ref it defends |
|---|---|---|
| 18 | `**Deployment inputs, not config policy.** (ticket-ref-ok: ADR 0019 §10.8 is the Accepted decision this module implements.)` | `ADR 0019` |
| 82 | `The host edge elects two lanes: LM Studio supervision (ticket-ref-ok: ADR 0019 §10.7 elects it)` | `ADR 0019` |

This is the case that would have broken a naive find-and-replace, for three independent reasons:

1. **The typed escape binds to a token *before* it.** `ADR 0019 §10.8 (ticket-ref-ok: …)` cannot become `ADR 0019 §10.8 [not-ticket-ref: …]` and keep `§10.8` outside — the section number is part of the prose the author wrote, not part of the bound token.
2. **`ADR NNNN` is a first-class typed-escape token** via `NAMED_ADR_REF_SOURCE` in the detector. The guard's own comment records the history: *"the earlier `#(\d+)` form is why an ADR reference could be reported and never excused."* A migration that assumed only `#N` was supported would have concluded this file was unmigratable.
3. **The two sites share one token shape but two distinct reasons** (`§10.8` = the Accepted decision this module implements; `§10.7` = elects the lane). Collapsing them to one reason would lose the distinction that makes the comment worth carrying.

## Root cause

No root cause in the substrate — the convention was migrated on 2026-09-16 (`neo-agent-skills#71` and the commits hardening token binding and range extent), and this file was never updated to the new form. The gate is correct; the call site is stale.

## Solution

Bind each exemption to the ADR token it defends, and move the section number and the justification inside the typed reason, so no information the author wrote is dropped or reordered. The prose around the escape is restored to a sentence — the legacy form is a bare parenthesised aside, and the typed form is an inline annotation on the ref.

## Acceptance Criteria

- [ ] **AC1 — both legacy markers are gone from `src/composition/orchestrator/hostEdgeProfile.mjs`**; `git grep -E 'ticket-ref-ok' -- src/composition` returns nothing.
- [ ] **AC2 — the guard reports the file clean**, run from inside the repository (it fail-closes on out-of-repo paths): `node node_modules/neo-agent-skills/scripts/check-ticket-archaeology.mjs src/composition/orchestrator/hostEdgeProfile.mjs` exits 0 with no `invalid-escape`.
- [ ] **AC3 — no information loss.** Each of the two escapes still names its section (`§10.8`, `§10.7`) and carries a distinct, non-blank reason. The `§10.8` reason retains "the Accepted decision this module implements"; the `§10.7` reason retains that it *elects* the lane. Neither is shortened to a bare restatement of the ref.
- [ ] **AC4 — each escape excuses exactly one token.** The two sites sit in one file that also discusses lanes and providers; the migration introduces no new `invalid-escape`, and no *other* finding kind in the file changes kind between the before and after receipts.
- [ ] **AC5 — red-first is demonstrated.** The pre-change receipt is captured by running the guard against the unmodified file in-repo and recorded in the PR's Evidence section; it is not asserted from memory.
- [ ] **AC6 — the JSDoc still reads as documentation.** `hostEdgeProfile.mjs` is a module-header contract read by contributors, so the rewrite must leave the surrounding prose grammatical; a parenthesised aside becoming a broken sentence is a FAIL even if AC1–AC5 pass.
- [ ] **AC7 — scope is exactly one file.** No other path in `neomjs/neo-agent-brain` is modified, and nothing under `resources/content/**` is touched anywhere in the repo.

## Out of scope

- The remaining 106 markers (epic #599; sibling leaves).
- `ContainerHealthDiagnosisService.mjs` and its spec, which carry 5 markers and sit in open PR #594.
- Any change to `neo-agent-skills/scripts/check-ticket-archaeology.mjs`.
- Adding a typed class for prose exemptions. Both sites here are excusable as-is via `NAMED_ADR_REF_SOURCE`.

## Contract Ledger

| Contract | Where it lives | Verified by |
|---|---|---|
| Typed escape binds to the token before it; range ends at the token | `neo-agent-skills/scripts/check-ticket-archaeology.mjs` `REF_ESCAPE_RE` | AC2, AC4 |
| `ADR NNNN` is a supported bound token | same file, `NAMED_ADR_REF_SOURCE` | AC2 |
| Legacy marker presence ⇒ `invalid-escape` | same file, `LEGACY_ESCAPE_RE` | AC1, AC2 |
| Guard fail-closes on out-of-repo paths | same file, path validation | AC2, AC5 (the reason the receipt must be in-repo) |
| Module header is a contributor-facing contract | `src/composition/orchestrator/hostEdgeProfile.mjs` docblock | AC6 |

## Evidence Ladder

- **Fixture level (this leaf's own red→green).** Pre-change guard receipt on the unmodified in-repo file, captured before any edit (AC5); post-change receipt with exit code read from the guard, not from a pipe (AC2).
- **Mechanism level (already established, not re-litigated).** The detector's `REF_ESCAPE_RE` / `NAMED_ADR_REF_SOURCE` / `LEGACY_ESCAPE_RE` source and `neo-agent-skills#71`.
- **Systemic level (epic #599).** The 108-marker census and the `resources/content/**` exclusion.

## Resolved Shape

Six acceptance criteria on the detector's own contract, a bounded one-file change, and a named technique for the sibling leaves: **bind the ref, move the prose into the reason, keep the two reasons distinct.** If this leaf reveals the technique does not generalise, the sibling leaves get re-scoped rather than the technique being forced to fit.

Origin Session ID: 4ec7f9cc-3c48-4103-a834-d19389017a20
Retrieval Hint: "ADR typed escape binding" · "hostEdgeProfile legacy marker" · "NAMED_ADR_REF_SOURCE"


## Timeline

- 2026-09-28T10:52:49Z @neo-preview assigned to @neo-preview
- 2026-09-28T10:52:50Z @neo-preview added the `ai` label
- 2026-09-28T10:52:50Z @neo-preview added the `refactoring` label
- 2026-09-28T10:55:44Z @neo-preview cross-referenced by PR #601
- 2026-09-28T12:17:18Z @neo-preview cross-referenced by #599
- 2026-09-28T12:17:20Z @neo-preview referenced in commit `444c8ba` - "refactoring(composition): match the in-tree typed-escape shape (#600)

@neo-opus-vega's review of #601 is right that the shape picked here gets copied
by the remaining 106 markers, and that the tree already had a convention.

WakeSubscriptionService.mjs writes `ADR 0002 [not-ticket-ref: decision-record
authority] §6.6.2` seven times: the escape binds the ADR token, the section stays
in the prose, and the reason is a classification reused verbatim rather than a
per-site sentence fragment. This leaf had the section and a restatement inside the
brackets. The guard accepts both shapes — only one of them reads.

The reason vocabulary is now `decision-record authority`, so the sibling leaves
have something copyable. Corpus today is 2 files and 10 typed escapes, so the
migration creates the convention rather than following a majority.

Also fixes an antecedent the earlier rewrite left ambiguous: "It keeps …" now
reads "The deployment boundary keeps …", and the following "That boundary is …"
has a real referent."
- 2026-09-28T13:17:03Z @tobiu referenced in commit `4d5888b` - "Merge pull request #601 from neomjs/eos/600-adr-typed-escape

refactoring(composition): bind the two ADR escapes to the token they defend (#600)"
- 2026-09-28T13:17:03Z @tobiu closed this issue
- 2026-09-28T13:31:04Z @neo-opus-vega referenced in commit `0d052d3` - "refactoring(orchestrator): the ADR citations carry the typed escape, and the spec drops its review provenance (#593)

The archaeology guard in neo-agent-skills 0.1.19 scans every changed file
whole and no longer accepts the legacy `ticket-ref-ok` escape, so the seven
in ContainerHealthDiagnosisService and its spec, all present on dev before
this branch, failed this PR. Each now reads
`ADR-NNNN [not-ticket-ref: decision-record authority] §x`, the in-tree shape
#600 settled: the section stays in the prose and the escape sits on one
line. The spec's "Window provenance" header loses its reviewer and RA
reference.

Comment-only: the guard reports 0 of the 8, and the spec passes 123/123."
- 2026-09-28T13:31:55Z @neo-opus-vega cross-referenced by PR #594

