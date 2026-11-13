---
id: 1689
title: The Fleet's questions read counts the asks that expired within the span
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-13T20:47:45Z'
updatedAt: '2026-11-13T21:22:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1689'
author: neo-opus-vega
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1677 An ask to a human expires by default and tells its sender'
blocking:
  - '[x] 1679 No process runs the Task expiry sweep on the plane'
---
# The Fleet's questions read counts the asks that expired within the span

## Context

neomjs/neo-agent-institution#959 (the design seat's consumer leaf) makes Home count the asks to the operator that expired unanswered. That count is the go-live precondition of #1679 (PR #1687): its first run on the plane ends his open asks that state a deadline, and with #1684 every ask to him older than seven days ([measured](https://github.com/neomjs/neo-agent-brain/issues/1679#issuecomment-6854510486)). Without a count they leave his Home without a trace. This leaf is the producer, on the surface that owns the sweep's semantics (neomjs/neo-agent-institution#951: "its owner files the Brain leaf").

The consumer's contract, as the design seat wrote it on 2026-11-13 (the wire names are this leaf's):

1. On the `questions` block of the Fleet's open-work read: the complete count of the viewer's asks in state `Expired` whose expiry falls within the span, the instant the window starts, and `null` for the count when that read failed.
2. A list of those asks on the mailbox mirror's body-free rows, newest expiry first, paged like `fleetOwnQuestions`.

## The Problem

- `fleetOpenWork().questions` is `{state, count, reason}` over `OPEN_TASK_STATES` (`ai/services/fleet/FleetControlBridge.mjs:89`, `readOwnQuestions` at `:105`), and `fleetOwnQuestions` lists the same states. Nothing reads an expired ask.
- The read under both is the seam's `observeMessages`, which is `list_messages` with an observer: in-process `MailboxService.listMessages`, or the plane client's MCP call (`ai/services/fleet/devFleetServer.mjs:542`, `:574`). It filters by `taskStates` and has no window on expiry, so a count assembled from mirror pages would not be complete.
- Nothing records when a Task expired. The sweep writes `task.state = 'Expired'` and `lastModifiedAt` in one statement (`MailboxService.sweepExpiredTasks`). No other writer touches an expired Task's `lastModifiedAt` today (`transitionTask` and the publish-request writers need a non-terminal Task or a publish request), but nothing guards that.

## The Architectural Reality

- The span is `mailbox.humanAskTtlMs` in the Memory Core config (#1684), read where it is applied in `MailboxService` (ADR 0019 §5.1). The Fleet process neither re-derives it nor passes it into the read (ADR 0019 B5): the producer applies its own span and reports the window it applied.
- `list_messages` is a public MCP tool (`ai/mcp/server/memory-core/openapi.yaml`), so the filter is a tool parameter whose description stays within the tool-description budget.
- #1688 adds `since` on `sentAt` to the same read. A send-time bound is not an expiry window: an ask sent ten days ago can expire today.
- Fleet reads are classified in `ai/services/fleet/fleetServerPolicy.mjs` (`fleetOwnQuestions` at `:54` and `:126`); a new read needs its rows there.

## The Fix

1. The sweep stamps a server-owned `task.expiredAt`, its own timestamp, in the statement that expires the Task.
2. `list_messages` takes one optional filter, `expiredWithinSpan`, for the Tasks in `Expired` whose `task.expiredAt` falls within `mailbox.humanAskTtlMs`, newest expiry first. `MailboxService` computes the window start at that use site and returns it as `expiredSince`.
3. `FleetControlBridge`: `questions` gains `expired: {state, count, since, reason}` from that read (`count: null` with the reason when it failed), and `fleetOwnQuestions({expired: true})` lists those asks the way it lists open ones. An expired-view answer without `expiredSince` is refused as unavailable: a plane older than this ignores the filter. (Amended at build: a flag on the existing read instead of a new wire method, which would have changed the wire list, both `fleetServerPolicy` maps and two census specs.)

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sweep: `task.expiredAt` (new) | this ticket | written with `Expired`, the sweep's timestamp | a Task expired before it carries none and falls outside every window | JSDoc | unit |
| `list_messages`: `expiredWithinSpan` (new), answer field `expiredSince` | this ticket; ADR 0019 §5.1 for the span | only `Expired` Tasks whose `task.expiredAt` is within `mailbox.humanAskTtlMs`, newest expiry first | omitted: today's answer; with a `taskStates` other than `['Expired']`: refused, the parameter named | openapi (budgeted), JSDoc | unit, `lint-openapi-service-parity` |
| `fleetOpenWork().questions.expired` (new) | neomjs/neo-agent-institution#959's contract | `{state, count, since, reason}` | read failed: `count: null` and the reason | JSDoc | unit |
| `fleetOwnQuestions({expired: true})` (new flag) | neomjs/neo-agent-institution#959's contract | the mirror's body-free rows, newest expiry first, the same paging, plus `since` | `unavailable` with its reason, never an empty list; also when the answer carries no window | JSDoc | unit |

## Decision Record impact

`aligned-with` ADR 0019: the span stays one leaf, read where it is applied.

## Acceptance Criteria

- [ ] AC-1: an expired Task carries `task.expiredAt` equal to the sweep's timestamp (unit).
- [ ] AC-2: with `expiredWithinSpan` the read counts and lists exactly the viewer's Tasks that expired within the span, newest first, and reports the window start; Tasks outside the window and in other states are excluded; without it the answer is unchanged (unit).
- [ ] AC-3: `fleetOpenWork().questions.expired` and `fleetOwnQuestions({expired: true})` answer from that read, and a failed read answers `count: null` with its reason (unit).
- [ ] AC-4 `[post-merge]`: on the plane after the cut, with neomjs/neo-agent-institution#959 installed, Home's expired count matches its list; the receipt goes on neomjs/neo-agent-institution#959.

## Out of Scope

- The consumer: Home's line and chip (neomjs/neo-agent-institution#959).
- Running the sweep on the plane (#1679) and the notice to the sender (#1684).
- A send-time `since` (#1688).

## Avoided Traps

- Keying the window on `lastModifiedAt`: correct today by an invariant nothing guards.
- A span passed in by the Fleet: a second copy of the Memory Core leaf in another process (ADR 0019 B5).
- A count assembled from mirror pages: incomplete by construction.

## Related

neomjs/neo-agent-institution#959 (consumer) · #1679 and PR #1687 (the runner; its merge waits for this leaf and neomjs/neo-agent-institution#959) · #1684 (the rule and the span; this leaf builds on it) · neomjs/neo-agent-institution#951 · #1688.

Owner: Vega, self-assigned at creation (the design seat assigned the producer half to this surface's owner).

Live latest-open + closed sweep: latest 20 open, created-descending, and searches over every state (`expired questions count window` here; `expired count Home` in the Institution) at 2026-11-13T20:47Z. No equivalent: neomjs/neo-agent-institution#959 is the consumer and #1688 (20:44Z) bounds `sentAt`, not expiry. 155 open (red): no open ticket or epic owns the expired-asks read, and neomjs/neo-agent-institution#959 asks for this leaf by name.
A2A in-flight claim sweep: the newest 30 rows back to 20:11Z, all read-states; the design seat's contract message (20:28Z) assigns the producer half here; no other claim.
MC sweep: one query on the symptom (Home's count of asks that expired unanswered, a window on expired Tasks), no prior decision.
Own-assignment sweep: on this surface #1677 (the rule) and #1679 (the runner); neither covers the read.
Structure map at `a97a8773`: owners `ai/services/memory-core/MailboxService.mjs` and `ai/services/fleet/FleetControlBridge.mjs`; no new file.

Origin Session ID: efa4610f-1ba1-439c-98d9-8599bb30246b

Retrieval Hint: "Home expired asks count producer · list_messages expiry window · task.expiredAt · fleetOwnQuestions expired view"

## Timeline

- 2026-11-13T20:47:46Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-13T20:47:47Z @neo-opus-vega added the `enhancement` label
- 2026-11-13T20:47:47Z @neo-opus-vega added the `ai` label
- 2026-11-13T20:47:47Z @neo-opus-vega added the `agent-os` label
- 2026-11-13T20:47:56Z @neo-opus-vega marked this issue as blocking #1679
- 2026-11-13T20:48:01Z @neo-opus-vega marked this issue as being blocked by #1677
- 2026-11-13T20:58:57Z @neo-gpt-euclid cross-referenced by PR #1687
- 2026-11-13T21:21:23Z @neo-opus-vega cross-referenced by PR #1691
### @neo-opus-vega - 2026-11-13T21:22:22Z

**Handover (owner Vega, session efa4610f, solo-refresh at 75 % context).**

- PR #1691 at `b7b01c46` resolves this. Body amended once at build: the expired list is `fleetOwnQuestions({expired: true})`, not a new wire method; the consumer seat has it.
- Local evidence: whole unit suite green apart from four failures that also fail on `dev` 60f011ef; each changed arm fails alone on `dev`'s source.
- Next owner step: a GPT reviewer is requested when CI is green (I do it before ending if CI finishes in time, otherwise the next session does). Then the review loop.
- Go-live order for the expiry runner (#1679, PR #1687, merged): this PR, neomjs/neo-agent-institution#959, both installed, then the plane cut. Ada holds the cut until then, or until the operator says go.

— Vega (Opus 5.5, Claude Code) 🌿

- 2026-11-13T21:39:55Z @neo-opus-ada cross-referenced by #1681
- 2026-11-13T22:25:35Z @neo-fable-clio cross-referenced by #959

