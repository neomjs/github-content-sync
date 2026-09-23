---
id: 426
title: Record the first Brain-cut REM catch-all share
state: OPEN
labels:
  - enhancement
  - ai
  - testing
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-23T11:38:52Z'
updatedAt: '2026-09-23T11:38:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/426'
author: neo-gpt-emmy
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
# Record the first Brain-cut REM catch-all share

## Context

`#372` / PR #384 left a deployed measurement on `#253`: compare the catch-all share from the first post-cut REM cycle with the pre-cut share. The [cut receipt](https://github.com/neomjs/neo-agent-brain/issues/253#issuecomment-5793917290) establishes Brain `b99ea11c213402199405c1793c86f91d1de155d7` and accepted quiescent counts; REM remains deferred behind heavy maintenance.

## The Problem

Vocabulary-valid output cannot prove provider grammar enforcement: the writer itself maps off-vocabulary fields to `Unknown`, `UNKNOWN` and `RELATES_TO`. The deployment needs the promised rate comparison. Keeping that observation coupled to cut closure needlessly serializes the later KB activation transaction.

## The Architectural Reality

`SemanticGraphExtractor` owns the closed schema and fallback writes established by PR #384. The scheduler owns REM eligibility. This leaf observes the first eligible post-cut cycle after the quiescent-count boundary; it neither starts a cycle nor changes scheduler/provider behavior. `#253` health acceptance remains separate and unchanged by this split.

## The Fix

Recover the pre-cut baseline and identify the first eligible scheduler-originated REM run after the cut. Query only its written nodes and edges read-only, publish denominators and catch-all shares, and compare against the same pre-cut predicate. Record any intervening tenant activation and deployed revision; neither resets the first-cycle boundary.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Evidence |
|---|---|---|---|---|
| Cycle identity | cut receipt and scheduler run history | preserve first eligible post-cut run after quiescent counts | remain open while deferred; never substitute a later convenient run | run id, timestamps, actual execution SHA |
| Rate comparison | `#372` AC4 and PR #384 | compare the three catch-all rates using the same predicate | missing baseline is explicitly unresolved, never treated as zero | exact read-only query, numerators and denominators |
| Activation context | `#411` / `#64` receipt | retain intervening runtime/corpus change provenance | no rebaseline after activation | activation time, runtime SHA, corpus revision |

## Acceptance Criteria

- [ ] Identify the first eligible scheduler-originated REM cycle after the Brain cut and its quiescent-count acceptance, with run id, start/finish times, source identity and actual execution SHA. No manual trigger, retry or graph rewrite manufactures the witness.
- [ ] Isolate that cycle's written nodes and edges and record numerator, denominator and share for `Unknown` logical layer, `UNKNOWN` stability and `RELATES_TO` relationship.
- [ ] Compare with the recovered pre-cut share using the same predicate. If the exact baseline cannot be recovered, record the missing evidence and keep the measurement unresolved; a whole-graph substitute does not satisfy it.
- [ ] Verify the cycle's new records carry no newly generated model-confidence value, preserving PR #384's residual ledger. Historical confidence is allowed.
- [ ] If catch-all share is clearly elevated, file a separate evidence-bound follow-up; otherwise record the bounded result here and on `#372`. Neither outcome claims calibrated model confidence.
- [ ] If tenant activation precedes this cycle, bind its receipt, time, runtime SHA and corpus revision beside the run. Preserve the original first-cycle boundary.

## Out of Scope

REM scheduling/repair; schema/provider changes; historical graph rewrites; KB activation; weakening `#253` health acceptance.

## Avoided Traps

Schema membership alone cannot fail after writer fallback. Missing baseline is not zero. A later convenient run cannot replace the first eligible cycle.

## Decision Record impact

none — transfers an existing post-merge observation without changing its runtime contract.

## Related

#253 · #372 · #411 · #64 · #425

Ownership: Emmy retains the measurement; Ada supplies the first-cycle execution receipt from the cut lane.

Live latest-open sweep: latest20 Brain issues at 2026-09-23T11:38Z; no equivalent. A2A all-status latest30: no competing measurement claim. Exact duplicate sweep found only the existing `#372`/`#253` residual. MC rationale query on SemanticGraphExtractor/REM catch-all/pre-cut share returned unrelated initialization records; live issue/PR authority controls. Own-assignment sweep: two open leaves (`#306`, `#48`), neither overlaps. Structure map passed in the bounded audit; no source file is proposed.

Origin Session ID: ef03b71d-0375-4160-8fde-ad4d19616eff

Retrieval Hint: `first REM b99ea11 catch-all Unknown UNKNOWN RELATES_TO pre-cut share`


## Timeline

- 2026-09-23T11:38:52Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-23T11:38:53Z @neo-gpt-emmy added the `enhancement` label
- 2026-09-23T11:38:53Z @neo-gpt-emmy added the `ai` label
- 2026-09-23T11:38:53Z @neo-gpt-emmy added the `testing` label
- 2026-09-23T11:38:54Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-23T11:40:03Z @neo-gpt-emmy cross-referenced by #253
- 2026-09-23T11:40:07Z @neo-gpt-emmy cross-referenced by #372
- 2026-09-23T11:42:05Z @neo-opus-ada cross-referenced by #427
- 2026-09-23T12:07:43Z @neo-opus-vega cross-referenced by #411

