---
id: 1321
title: Make the steward shadow window executable and retainable
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - agent-os
assignees:
  - neo-gpt-euclid
createdAt: '2026-10-27T19:36:59Z'
updatedAt: '2026-10-27T21:45:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1321'
author: neo-gpt-euclid
commentsCount: 0
parentIssue: 106
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-27T21:45:06Z'
---
# Make the steward shadow window executable and retainable

## Context

Parent #106 retains the real source-bound steward observation window after the closed `#100` instrumentation and merged `#1312`. At Brain `436aec637e1d72123d2ebfd894d29d8c64260489`, `CommunityActivityService.evaluateStewardWake` has only its service wrapper and test callers; the MCP surface exposes counts, not steward evaluation or a retained window. The [parent custody record](https://github.com/neomjs/neo-agent-brain/issues/106#issuecomment-6430221168) records the live readiness result: missing registered sources, not measured zero demand.

## The Problem

The existing helper can record aggregate vacancy/would-wake counters, but the owner cannot run a bounded, reproducible source-bound window that retains the actual count input, evaluation, denominator, coverage and clock for every observation. Acquisition `--mode shadow` does not admit ledger records or invoke the steward. The calibration runner's temporary SQLite fixtures are not the missing operational window.

This is the executable observation contribution, not a reopening of `#100` or a claim that its fixtures failed. The real retained window and activation disposition stay on #106.

## The Architectural Reality

- `ai/services/memory-core/CommunityActivityService.mjs:1072` projects tenant-scoped counts and invokes the existing evaluator.
- `ai/services/memory-core/communityStewardWakeLease.mjs:569-660` preserves missing/degraded coverage and records vacancy only for eligible fresh demand; null lease TTL prevents acquisition.
- `ai/scripts/maintenance/reconcileGithubCommunity.mjs` is the existing trusted, co-located operator boundary with explicit tenant/source arguments and validation before loading storage.
- `CommunitySourceRunbook.md` §1–§3 reserves registration and reconciliation for the deployment operator; §8 describes steward internals but no retained-window command.
- The documented source-operator npm alias is absent from `package.json`; its existing CLI file can be invoked directly.

Design authority: ADR 0036 §5: “a steward experiment adds vacancy and wake-cost evidence.” Preserve ADR 0019 use-site configuration authority and ADR 0036 tenant/source isolation.

## The Fix

Provide one co-located, finite shadow-window entry under `ai/scripts/maintenance/`, following the reconciliation/calibration entrypoint pattern. Explicit deployment-owned tenant, source selectors, sample count, observation spacing and fresh output identify one experiment; they do not set scheduler cadence or production thresholds.

Use the existing service/evaluator and retain the exact attention snapshot actually consumed by each evaluation, its timestamp/elapsed time, source epochs/coverage/watermarks, decision and count-only bytes. Extend the service's observation seam only as needed to bind that input to its output; do not duplicate count projection or the evaluator.

Refuse live activation and preserve unset leaves. Do not acquire a lease using an invented TTL, select an internal roster, register/admit sources, claim Tasks, or dispatch messages. Missing/empty/degraded observations remain explicit and cannot become an accepted fresh-demand window. Keep unavailable occupancy and acknowledge/claim/respond latency slots unknown with reasons.

Write the finite report without replacing existing files or following unsafe output paths. Partial/failing runs retain an honest terminal result. Document the executable source bootstrap and window procedure, including the accurate existing source-control command.

Prescription checked: the co-located maintenance entry owns operator invocation/report custody; Memory Core owns scoped counts/evaluation, and the existing registry/connector retain source administration/acquisition.

## Contract Ledger

| Surface | Authority / behavior | Refusal / fallback | Docs / evidence |
|---|---|---|---|
| Operator entry | Existing trusted co-located tenant boundary; finite explicit observation controls | Invalid arguments/output refuse before storage loading; no MCP tenant override | Runbook + real CLI entry tests |
| Observation binding | Actual service count projection consumed by the existing evaluator | Coverage or source-generation gaps remain explicit; no synthetic count replacement | Timestamped report + input/output identity controls |
| Shadow policy | Resolved `communityStewardWake` leaves at use sites | Live activation refuses; false/null values stay unchanged; no lease/dispatch/claim | Policy, isolation and no-effect controls |
| Report | Actual runtime/caller provenance and each observation, measured/unknown distinction | Partial terminal outcome; no prose, secrets or lease tokens | Retained CLI receipt and report validation |
| Outcome | #106 owns the real source window/calibration after this capability lands | Fixture/CLI tests cannot discharge the real-window obligation | Explicit parent handoff |

## Acceptance Criteria

- [ ] The supported CLI validates explicit scope, finite controls and a fresh safe output before opening storage; it uses the existing operator boundary and documented commands actually exist.
- [ ] Each retained sample binds the exact real-service count input to its steward result, timestamp, source identity/generation, coverage and watermark. Denominators and partial/failure outcomes are retained.
- [ ] Live activation is refused, governed leaves remain unchanged, no invented TTL or global roster is introduced, and the path creates/assigns no Tasks or live/fallback dispatch.
- [ ] Missing/degraded/empty demand and unavailable lease/latency evidence remain distinct from measured zeros; the report cannot claim a completed real window from fixtures or missing-source inputs.
- [ ] Focused tests cover actual CLI execution, source/tenant isolation, policy refusal, input/output binding, safe output/partial failure and zero dispatch/claim effects. Fixture evidence is labelled; all owning checks pass.
- [ ] Runbook/report instructions and the #106 handoff give exact execution/provenance and the deployment-owned source prerequisite. The real production window remains open on #106 until separately observed.

## Decision Record impact

Aligned with ADR 0036 §5, ADR 0019 and the existing SQLite operational owner. No new ADR, scheduler, public MCP surface or activation authority.

## Out of Scope

Source grants or live registration/admission without deployment-owner authority; periodic activation, threshold calibration from fixtures, Task claims, a second event ledger, new wake transport, and closing #106.

## Avoided Traps

Do not call acquisition shadow mode the steward window; do not call a temporary fixture a live tenant; do not manufacture lease occupancy from an unset TTL; do not add a public tenant selector; do not repeat the count/evaluator algorithm in the CLI.

## Related and Ownership

Child of #106; follows closed `#100` and merged `#1312` / `#1313`. Owner: @neo-gpt-euclid.

Origin Session ID: 607243d1-f786-4b5c-b0a2-028defcc0310
Retrieval Hint: "steward retained observation window no production caller exact count evaluation"

Creation checks: current source and independent caller census at `436aec63`; structure map executed successfully with `--files --loc`; maintenance sibling lift (`reconcileGithubCommunity.mjs`, `calibrateCommunityActivityPolicy.mjs`). Live latest-20 open, last-30 all-state A2A, all-state steward/observation titles, own-assignment bodies and problem-noun Memory Core/KB sweeps immediately before filing found no equivalent. Existing #106 is the outcome owner; `#100` intentionally transferred the real window. MC prior custody: `f6594d79-d5eb-4479-addc-cc5a6e04c610`.
Live final freshness: latest 20 open issues and last 30 all-state A2A claims checked at 2026-10-27 19:36:58 UTC; no equivalent active leaf or foreign claim. This is one coherent implementation contribution to the existing parent, not a new outcome.


## Timeline

- 2026-10-27T19:37:00Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-10-27T19:37:02Z @neo-gpt-euclid added the `enhancement` label
- 2026-10-27T19:37:02Z @neo-gpt-euclid added the `ai` label
- 2026-10-27T19:37:02Z @neo-gpt-euclid added the `testing` label
- 2026-10-27T19:37:02Z @neo-gpt-euclid added the `agent-os` label
- 2026-10-27T19:37:30Z @neo-gpt-euclid added parent issue #106
- 2026-10-27T20:35:25Z @neo-gpt-euclid cross-referenced by #106
- 2026-10-27T20:38:03Z @neo-gpt-euclid cross-referenced by PR #1322
- 2026-10-27T20:53:02Z @neo-gpt-euclid referenced in commit `d0ccdaa` - "docs(memory-core): clarify shadow window prerequisites (#1321)"
- 2026-10-27T21:45:07Z @tobiu closed this issue
- 2026-10-27T21:45:07Z @tobiu referenced in commit `06f5574` - "feat(memory-core): retain steward shadow windows (#1321) (#1322)

* feat(memory-core): retain steward shadow windows (#1321)

* docs(memory-core): clarify shadow window prerequisites (#1321)"

