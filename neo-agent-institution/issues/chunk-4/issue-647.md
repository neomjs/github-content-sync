---
id: 647
title: 'The Institution pins Brain 2445eb36: the open questions read on a plane'
state: OPEN
labels:
  - enhancement
  - ai
  - dependencies
assignees:
  - neo-opus-vega
createdAt: '2026-10-09T15:41:27Z'
updatedAt: '2026-10-09T15:41:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/647'
author: neo-opus-vega
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
# The Institution pins Brain 2445eb36: the open questions read on a plane

## Context

The Institution pins Brain `fb8c11ee`, set by #623 (merged 2026-10-09 as `41068a4`). Brain `dev` is at `2445eb36`, one merge later:

| Brain merge | What reaches the Institution | Read by |
|---|---|---|
| neomjs/neo-agent-brain#952 | The plane serves an explicit observer read that writes no receipts, and `devFleetServer` binds the Fleet's `observeMessages` to it. `fleetOwnQuestions` and `fleetOpenWork.questions` then answer on a plane instead of returning `unavailable`. | #490 (its own-question AC); #596 (All / involves-me) |

The operator decided on 2026-10-09 to bump this pin now. Replacing hand-bumped pins with tracking `dev` is #646.

## The Problem

Until the pin carries #952, the operator's open questions and Home's question count read `unavailable` on every plane-attached Institution. The feature is merged in the Brain and absent from every candidate.

## The Architectural Reality

The delta census `git diff --stat fb8c11ee 2445eb36 -- src package.json` is empty. The whole delta is #952's 14 files under `ai/` and `test/`.

The Institution consumes it through the Fleet wire with no source change. `readOwnQuestions` in the Brain's `ai/services/fleet/FleetControlBridge.mjs` already reads through `seam.observeMessages`, and `devFleetServer` now binds that seam on the plane.

## The Fix

- **Point the pin** at `2445eb360f284c3e7208e2b02371f826a1d98c5c` in `package.json`, `package-lock.json` and `.github/workflows/ci.yml` (the Brain checkout `ref`).
- **Reinstall** the dependency and check one marker: the installed `ai/services/fleet/planeMailboxClient.mjs` exports `MAILBOX_OBSERVER_UNAVAILABLE`.
- **Run CI's jobs locally:** unit, components, E2E, the visual suite, and the cross-repository job (Brain at the pin, `NEO_AGENTOS_RUNTIME_ROOT`).

## Acceptance Criteria

- [ ] AC-1: `package.json`, `package-lock.json` and `ci.yml` all name `2445eb36`, and the installed module carries the marker.
- [ ] AC-2: unit, components, E2E, the visual suite and the cross-repository job pass on the new pin.

## Post-Merge Validation

- [ ] The next #12 candidate carries this pin, and #490's own-question AC reads Home's count and the `for you` list on the installed plane. Residual-Owner: #490

## Out of Scope

- The engine pin, which is unchanged.
- Tracking `dev` instead of pinning; that is #646.

## Related

#623 · #599 · #490 · #596 · #646 · neomjs/neo-agent-brain#952 · neomjs/neo-agent-brain#921

Live latest-open sweep: checked the latest 20 open issues at 15:39Z; no equivalent. A2A claim sweep (last 30, all states): no competing claim. Own-assignment sweep: #485 only, unrelated. Structure map: N/A (dependency pin and CI ref only).

Origin Session ID: 9a84c569-02eb-4f7c-b87d-43ebcb24593d
Retrieval Hint: "Institution Brain pin 2445eb36 plane-mode open questions observer"

## Timeline

- 2026-10-09T15:41:29Z @neo-opus-vega added the `enhancement` label
- 2026-10-09T15:41:29Z @neo-opus-vega added the `ai` label
- 2026-10-09T15:41:29Z @neo-opus-vega added the `dependencies` label
- 2026-10-09T15:41:46Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-09T15:47:44Z @neo-opus-vega cross-referenced by PR #648

