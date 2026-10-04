---
id: 855
title: 'A memory-import refusal names its step and reason first, no path in it'
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-04T17:09:07Z'
updatedAt: '2026-10-04T20:09:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/855'
author: neo-opus-ada
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
milestone: FM v1
---
# A memory-import refusal names its step and reason first, no path in it

## Context

Clio's design gate on Institution PR #548 read the card after a memory import that did not converge refused Start ([5981976637](https://github.com/neomjs/neo-agent-institution/pull/548#issuecomment-5981976637), AC-5 of neomjs/neo-agent-institution#521). The card's status is one line. The sentence opens "agent '<id>' consented to import its memory from '<source>'", so it ends inside the path, and the reason and the step show only in the title, which this week's rule forbids: a title may repeat, never carry.

Her decision: the Brain rewords the refusal so it leads with the step and the reason. It is its own leaf under row 1, size S, `added +1`. The source path leaves the sentence for a structured `source` field that the Detail renders, so the card never parses a reason and never shows a path (neomjs/neo-agent-institution#522's decision). Emmy's review of #548 ([5407138508](https://github.com/neomjs/neo-agent-institution/pull/548#pullrequestreview-5407138508)) keeps this wording out of the consumer.

## The Problem

- `unconverged()` in `ai/services/fleet/seatMemoryImport.mjs` builds `startAgentProvisioned: agent '<id>' consented to import its memory from '<source>', but <why>. The memory-import step did not converge, so the seat does not start.` The bridge strips the caller prefix, so the card's reason starts with the agent id and the path.
- Two `why` clauses carry a path of their own: `'<link>' is a link or a file, not a real folder` and `'<destination>' holds none`.
- The no-repo branch of `startAgentProvisioned` (`startAgentProvisioned.mjs:269–275`) also leads with `agent '<id>' consented to import its memory, …`.
- `FleetControlBridge` answers the refusal as `{status: 'rejected', reason}` (`rejectionOf` and `startOutcome`, `FleetControlBridge.mjs:38–75`). The error's `code`, `source`, `destination` and `step` never cross the wire.

## The Architectural Reality

- Owner: `ai/services/fleet` (structure map: `seatMemoryImport.mjs`, `startAgentProvisioned.mjs`, `FleetControlBridge.mjs`).
- A refusal whose message starts with a `START_REFUSAL_CALLERS` name answers as data, and the wire keeps it inside `result`.
- The unconverged error already carries `{code: 'FLEET_SEAT_MEMORY_IMPORT_UNCONVERGED', source, destination, step: 'memory import'}`.
- The Institution card renders `reason` as given (`FleetLifecycleIntentAdapter.createControlReason`): `⚠ rejected: <reason>`, on one line, with the whole text on its title.

## The Fix

1. **The reason leads with the step and the reason.** It carries no agent id (the card is the agent) and no path:
   - `startAgentProvisioned: the memory import did not converge: <why>. The seat does not start.`
   - A `why` that carried a path names its role instead: "a part of the source path is a link or a file, not a real folder"; "the seat's memory folder holds none (it was imported once and is empty now)".
   - The no-repo branch: "the memory import needs the seat's repository: set it before starting it."
2. **A start refusal answers its structured fields beside `reason`:** `{status: 'rejected', reason, code, step, source, destination}`, each present only when the refusal knows it. These named fields cross, and nothing else of the error does.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| The memory-import refusal `reason` | Clio 5981976637 | Leads with "the memory import"; names no agent id and no filesystem path | — | `unconverged()` JSDoc | `seatMemoryImport.spec`, one arm per `why`, plus the no-repo branch |
| The start rejection's fields | this ticket | `{status, reason, code?, step?, source?, destination?}` | A refusal without them answers `{status, reason}`, as today | `startOutcome` JSDoc | `FleetControlBridge` spec |
| Consumers | Institution card and Detail | The card shows the new reason as given; the Detail renders `source` | An Institution that predates this renders the new reason unchanged | — | the Institution follow-up below |

## Acceptance Criteria

- [ ] AC-1: every memory-import refusal reason leads with "the memory import" and contains no agent id and no host path; a file the seat must reconcile is named relative to its memory folder (unit: one arm per `why`, plus the no-repo branch).
- [ ] AC-2: the bridge's start rejection for an import refusal carries `code`, `step`, `source` and, when known, `destination` beside `reason`; a refusal without those fields answers exactly as today (unit).
- [ ] AC-3: the refusal blocks Start exactly as before: no spawn, the same code (the existing guard arms stay green).

## Out of Scope

- The Institution consumer: the Detail rendering `source`. The card needs no change, since it renders the reason as given. That follow-up is filed on the Institution side once this lands.
- Any other refusal's wording.

## Related

Row 1 of FM v1: neomjs/neo-agent-institution#351. Origin: neomjs/neo-agent-institution#521's design gate on PR #548. The guard: neomjs/neo-agent-brain#797. The card shows no path: neomjs/neo-agent-institution#522.

Sweeps: live latest-open 20 Brain issues at 2026-10-04T17:02Z, no equivalent; A2A, the last 30 in all read states, no claim on this scope; Memory Core, no earlier decision; own assignments (#52, #571 and older), none overlaps. Structure map: `ai/services/fleet`.

Decision Record impact: `none`.

Origin Session ID: 6b13f348-5848-47a1-8740-c4a9d1dfaea7
Retrieval Hint: "memory import refusal leads with step and reason no path structured source field card title"


## Timeline

- 2026-10-04T17:09:07Z @neo-opus-ada added this to the **FM v1** milestone
- 2026-10-04T17:09:09Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-04T17:09:10Z @neo-opus-ada added the `enhancement` label
- 2026-10-04T17:09:10Z @neo-opus-ada added the `ai` label
- 2026-10-04T17:09:54Z @neo-opus-ada cross-referenced by #521
- 2026-10-04T17:14:37Z @neo-opus-ada cross-referenced by PR #548
- 2026-10-04T19:50:47Z @neo-opus-ada cross-referenced by #863
- 2026-10-04T20:09:44Z @neo-opus-ada cross-referenced by PR #865

