---
id: 995
title: 'ADR 0035: retain obligations when lifecycle clocks are unavailable'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-10-11T05:16:35Z'
updatedAt: '2026-10-11T06:44:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/995'
author: neo-gpt
commentsCount: 0
parentIssue: 107
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-11T06:44:52Z'
---
# ADR 0035: retain obligations when lifecycle clocks are unavailable

## Context

The source-fact audit under `#107` found that a known lifecycle obligation can have an unprovable start time. [The bounded draft](https://github.com/neomjs/neo-agent-brain/issues/107#issuecomment-6105561313) and [Grace's conditional approval](https://github.com/neomjs/neo-agent-brain/issues/107#issuecomment-6105571021) converge this record amendment before implementation.

## The Problem

At Brain `7ca66028`, a pure `normalizeLifecycleClocks` → `admitOwnPrReviewerRouting` control returned the old 10:30 check time when the source transition was omitted or bound to an old head; the correctly bound 12:00 transition returned 12:00. Current state alone cannot prove uninterrupted readiness. v1 requires a date: throwing loses the pass, dropping hides an obligation, and a fallback can invent its age. Both the `headCommittedAt` floor and zero-required-check fallback also confuse commit creation with PR head activation.

## The Architectural Reality

Design authority: ADR 0035 §2.3 states “re-entry starts a new clock”; §6 trigger4 requires reopening for new consumer fields.

The source owners remain GitHub Workflow and Memory Core A2A. `ai/services/graph/{lifecycleAdmission,lifecycleFrontier,produceLifecycleFrontier}.mjs` owns admission/composition; `hookProjectionLease.mjs` pins publication schemas; `ai/scripts/lifecycle/hookProjectionReader.mjs` validates/renders for both hooks. The executed structure map confirms these existing folders. No new owner or file is needed.

Prescription checked by Grace: ADR0035 §2.3 owns the incompatible clock contract; runtime-only null acceptance would bypass its shape/order/version boundary. Her C1 requires naming both obsolete commit-time uses, and the zero-required/existing-SHA evidence cases.

## The Fix

Amend only `learn/agentos/decisions/0035-live-lane-awareness-composition.md`:

- v2 separates eligibility from source-declared clock unavailability, with explicit basis and bounded reason;
- order remains stage-first, then dated oldest-first, unavailable, stable id;
- date every routing conjunct, current uninterrupted obligation and actual head activation under the policy read at `checkedAt`;
- retire both commit-time clock shortcuts; label observation dates in the renderer;
- require coherent admission/builder/validator/publication/shared-renderer adoption and matching rejection behavior;
- replace v1 outright without prescribing a dual-read window; preserve source-census versus installed-evidence limits.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Lifecycle clock | ADR0035 §2.3; source facts | v2 explicit date/basis or null/unavailable/reason; missing/malformed/wrong-head declaration invalid | No inferred date or pulse-derived age | §2.3 | Measured control; AC1/2 |
| Order and freshness | `lifecycleFrontier.mjs` builder/validator | Dated before unavailable within unchanged stage priority; clock knowledge separate from read freshness | Failed source reads retain coverage degradation | §2.3 | Consumer audit; AC2 |
| Publication/read | `PROJECTION_CHANNEL_SCHEMAS`; shared hook reader | Coherent v2 pin/validation/rendering, observation-labelled dates, both append paths | Version rejection and existing sibling-channel/bare-policy boundaries | §2.3/§5 | Grace C1; AC3/4 |

## Decision Record impact

`amends ADR 0035 §2.3` and its §5 evidence row under §6 trigger4. It preserves the graduated D15090 federation, source ownership, capabilities and stage priority.

## Acceptance Criteria

- [ ] **AC1:** Specify v2 dated/unavailable clock declarations, strict missing/malformed/wrong-head handling, and bounded source reasons.
- [ ] **AC2:** Specify uninterrupted/re-entry timing, all Stage2 conjuncts, actual PR head activation, both obsolete commit-time uses, current-policy/observation basis, and dated/undated ordering.
- [ ] **AC3:** Require coordinated publication/read adoption, explicit version rejection and both pure hook paths; no runtime/deployment completion claim.
- [ ] **AC4:** Add the discriminating implementation evidence cases, including zero required checks and an existing SHA whose checks predate PR entry; pass applicable docs/preflight checks and independent review.

## Out of Scope

Runtime code/tests, source adapter, admission/attester, registry/scorer/stateful federation, Fleet-read grants, deployment, installation and parent closure.

## Avoided Traps

An evergreen check cannot date re-entry. Commit creation is not push time. Null without a source declaration hides a wiring error. Degrading the entire envelope for an unavailable date suppresses known obligations.

## Related

Parent: #107 · ADR0035 · [Converged draft](https://github.com/neomjs/neo-agent-brain/issues/107#issuecomment-6105561313).

Origin Session ID: 2d8feac7-c60d-4883-8059-37b6e148768b
Retrieval Hint: `lifecycleAdmission reviewerRoutingSince headCommittedAt same-head re-entry unavailable clock ADR0035`

Creation sweeps at2026-10-11T05:16Z: live latest20 open(created-descending), all-state30 A2A claims, exact/historicalclock search and3 problem-keyed MC results; no standalone equivalent or competing claim. Own-assignment sweep:6 open; the same-surface107body is the owning outcome, not a competing clock leaf. Structure map:93 folders; existing graph/lifecycle/Memory-Core owners inspected. MC returned older own diaries; the live draft/disposition and current source supply the decision authority.


## Timeline

- 2026-10-11T05:16:35Z @neo-gpt assigned to @neo-gpt
- 2026-10-11T05:16:36Z @neo-gpt added the `documentation` label
- 2026-10-11T05:16:36Z @neo-gpt added the `enhancement` label
- 2026-10-11T05:16:37Z @neo-gpt added the `ai` label
- 2026-10-11T05:16:37Z @neo-gpt added the `architecture` label
- 2026-10-11T05:16:37Z @neo-gpt added the `agent-os` label
- 2026-10-11T05:17:07Z @neo-gpt added parent issue #107
- 2026-10-11T05:25:41Z @neo-gpt cross-referenced by PR #996
- 2026-10-11T05:32:53Z @neo-gpt cross-referenced by #107
- 2026-10-11T06:44:52Z @tobiu referenced in commit `4dd2054` - "docs(agentos): define unavailable lifecycle clocks (#995) (#996)"
- 2026-10-11T06:44:53Z @tobiu closed this issue
- 2026-10-11T06:47:50Z @neo-gpt cross-referenced by #997
- 2026-10-11T07:07:39Z @neo-gpt cross-referenced by PR #998

