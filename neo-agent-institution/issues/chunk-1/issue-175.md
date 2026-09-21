---
id: 175
title: The activity stream says "streaming" over weeks-old rows
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-19T15:25:59Z'
updatedAt: '2026-09-19T16:42:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/175'
author: neo-fable-clio
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
closedAt: '2026-09-19T16:42:20Z'
---
# The activity stream says "streaming" over weeks-old rows

## Context

Observed three times on 2026-09-19 in the running cockpit against a live fleet server (`dev@d02fe83`, `dev@272e5af`, `dev@6d953f1`): the Live Activity header reads `● streaming` in the live ink while the newest row is dated **Aug 26** — 24 days old. Nothing on the surface says so. An operator reads a healthy, current feed.

The cause of the old rows is outside this ticket (that dev fleet server's pr-lane source reads content that stopped syncing). The defect here is the cockpit's: **its only liveness word describes the connection, and it lets the data's age go unsaid.**

## The Problem

`AgentOS.view.fleet.activity.Container#updateHeader()` derives the header's state cell from `adapterState` alone:

- `sample` → `sample · live feed pending`
- `stale` → `stale — reconnecting`
- anything else → `● streaming`

`adapterState` becomes `live` whenever the read succeeds (`cockpit/LivenessController.mjs`, the stream load's success branch). A successful read of old rows is therefore indistinguishable from a busy fleet. The pane holds the fact it needs: its store is newest-first on `occurredAt`.

For an operator who cannot read container logs, a surface that looks live and is not is worse than a terminal. Connection truth and data freshness are two facts, and the header states one.

## The Architectural Reality

- The pane: `apps/agentos/view/fleet/activity/Container.mjs` (`updateHeader()`, the `state` reference, header cls `is-live` · `is-stale` · `is-sample`); styles in `resources/scss/src/apps/agentos/fleet/activity/Container.scss`.
- The store: `apps/agentos/store/FleetActivityEvents.mjs`, sorted `occurredAt DESC`; rows format time through `ViewerTime.formatViewerTime()`.
- The test harness controls no clock, and the cockpit's goldens include this header. A relative age ("24 d ago") would re-render every day; an absolute stamp does not.
- None of the three cockpit files in the size warn band is touched.

## The Fix

In the live state, when the newest retained event is older than a threshold, the state cell states the fact beside the connection word — `● streaming · quiet since Aug 26` — in the attention ink, with a title that names both readings: the fleet was quiet, or an activity source stopped delivering. The header cannot know which, and must not guess.

- Absolute date through `ViewerTime`, never a relative age (golden stability, and no new timer: the header already re-renders on every store change, which the liveness cadence drives).
- The threshold is one named constant in the pane. 24 hours is the proposed value; challenge it on this ticket.
- `sample` and `stale` keep their words: they already say the rows are not current.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| The activity header's state cell (`.fm-stream-state`) and the header cls | The pane's own liveness contract: "live, honestly-labelled sample, or stale last-known data" (`adapterState_` JSDoc) | Live + newest event older than the threshold → `● streaming · quiet since <date>`, header cls gains `is-quiet`, the cell's `title` names both readings. Live + fresh, or an empty store → unchanged `● streaming` | An event without a parseable `occurredAt` never claims quiet | The `adapterState_` JSDoc and `updateHeader()`'s summary | Unit arms on `updateHeader()`: fresh, old, empty, unparseable, `sample`, `stale`; red on `dev` for the old-rows arm |

## Acceptance Criteria

- [ ] Live state + newest event older than the threshold: the state cell reads `● streaming · quiet since <absolute date>`, the header carries `is-quiet`, and the cell's title names both readings.
- [ ] Live state + a fresh newest event, and live state + an empty store: the cell reads `● streaming`, no `is-quiet`.
- [ ] `sample` and `stale` render exactly as today.
- [ ] Unit arms cover all of the above; the old-rows arm is red on `dev` first.
- [ ] Every golden that shows this header is re-rendered from a whole-suite run only if it changes, and read by eye.
- [ ] Post-merge, flagged: the live cockpit against the host dev fleet server shows `quiet since Aug 26`.

## Out of Scope

- Why that fleet server's activity sources are old (the dev recipe's plane topology).
- The roster's own freshness, and the mailbox meter's `0 / 24h`.
- A relative, ticking age.

## Avoided Traps

- **Replacing `● streaming`:** the connection IS streaming; hiding that trades one half-truth for another.
- **Calling it `stale`:** `stale` already means "the read failed, these are last-known rows" and carries a reconnect affordance. Old rows from a healthy read are a different fact.
- **A relative age:** it re-renders every golden daily, and there is no clock control in the harness.

## Related

#10 · #15 · #171 (the same journey's honesty bar)

Live latest-open sweep: all 16 open Institution issues read at 2026-09-19T15:26Z, plus four keyword searches (open and closed) — no equivalent. A2A in-flight sweep 15:23Z: no claim on this scope. Memory Core rationale sweep on the symptom: no prior decision surfaced (the semantic index returned unrelated rows, so this is a weak negative). Own open assignments: #10 (the epic) only.

Decision Record impact: none

Origin Session ID: bddb2360-a144-424d-b2b8-050ab386ff2e
Retrieval Hint: "activity stream header streaming quiet since newest event age data freshness"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session bddb2360-a144-424d-b2b8-050ab386ff2e

## Timeline

- 2026-09-19T15:25:59Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-19T15:26:01Z @neo-fable-clio added the `bug` label
- 2026-09-19T15:26:01Z @neo-fable-clio added the `agent-os` label
- 2026-09-19T15:26:01Z @neo-fable-clio added the `ai` label
- 2026-09-19T15:26:01Z @neo-fable-clio added the `design` label
- 2026-09-19T15:34:22Z @neo-fable-clio cross-referenced by PR #176
- 2026-09-19T15:43:44Z @neo-fable-clio cross-referenced by #17416
- 2026-09-19T16:42:20Z @tobiu referenced in commit `35f6ba9` - "Merge pull request #176 from neomjs/agent/175-activity-quiet-since

fix(activity): a live feed over old rows says so — "streaming · quiet since" beside the connection word (#175)"
- 2026-09-19T16:42:21Z @tobiu closed this issue

