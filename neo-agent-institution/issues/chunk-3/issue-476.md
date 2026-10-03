---
id: 476
title: The Agent Detail's Thought stream pane reads the seat's recent turns
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-03T07:16:54Z'
updatedAt: '2026-10-03T07:16:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/476'
author: neo-fable
commentsCount: 0
parentIssue: 391
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 792 The Fleet serves a seat''s recent turn summaries'
blocking: []
---
# The Agent Detail's Thought stream pane reads the seat's recent turns

## Context

A leaf of #391 (the Agent Detail's live panes). The Repository and Current lane panes are live (#435, #418). The Thought stream pane still reads `not observed`, and its pill names a producer that has since landed in two steps: the Memory Core's recency read is policy-aware (neomjs/neo-agent-brain#741), and the Fleet verb that serves it to the cockpit is neomjs/neo-agent-brain#792. This leaf is the consumer.

## The Problem

`apps/agentos/view/fleet/detail/Container.mjs` renders four panes; `thought-stream` has no ledger writer, so it takes the unobserved pill with `AWAITING['thought-stream']` on its title: *"awaiting a policy-aware read of this resident's turns — the plane's recency read answers the caller's own turns only"*. That sentence was true on 2026-10-02 and is not any more. With the team entering the roster (neomjs/neo-agent-brain#571), every seat's detail opens on this pane.

## The Architectural Reality

- `detail/Container.mjs`: `PANES` (`thought-stream`, 60 s freshness), `paneLedgers` (an explicit entry wins), the pane body references built by `paneConfig`.
- The cockpit `Controller`'s per-target reads are the pattern: a bridge-verb presence check, a generation guard, an `unavailable` fallback, and an instance switch retiring the previous profile's answer (the session-memories drill-in, `fleetSessionMemories`).
- neomjs/neo-agent-brain#792's envelope: `{capability {state, reason, capturedAt, detail}, viewer, target, page, turns, count, nextCursor, memorySharing {policy, clamped}}`; a turn is `{id, sessionId, timestamp, summary, summaryFallback, projectionPending}`.
- The client bridge derives its methods from the Brain's `FLEET_WIRE_METHODS`, so the verb exists on the bridge once a pin carries it.
- App-work gate: the pane binds a `data.Store` of `data.Model` records; the provider stays at the view root; styles stay in SCSS.

## The Fix

1. **The read.** Opening the detail for a seat, changing the seat, and the liveness tick while the detail is open read `fleetRecentTurns({agentIdentity, limit})` for the selected seat, guarded like the cockpit's other per-target reads. An instance switch retires the answer.
2. **The pane.** A Store of turn records bound to a list in the pane body: the time and the one-line summary per turn, newest first; a turn whose summary is a fallback or still pending says so.
3. **The pill.** The pane's ledger entry is the read's own observation: `capturedAt` as `observedAt`, the pane's TTL, and the envelope's state. `unavailable` shows the reason; a wired empty page reads "no turns yet"; a wired empty page with `memorySharing.clamped` reads that this plane shares no peer turns.
4. `AWAITING['thought-stream']` names what the pane waits for when the bridge lacks the verb: a Fleet server that serves recent turns.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Thought stream pane body (existing frame) | neomjs/neo-agent-brain#792's envelope | A Store-bound list of the seat's newest turn summaries. | Bridge without the verb → the unobserved pill with the corrected `AWAITING` text. `unavailable` → the reason on the pill, no rows. | JSDoc | unit + visual |
| Thought stream pane pill (existing) | the read's `capability` and `memorySharing` | `live` aged from `capturedAt`; a clamped policy and an empty stream are told apart in words. | A read older than the pane's TTL reads stale, like the sibling panes. | JSDoc | unit |
| The detail's read cadence (new for this pane) | the cockpit Controller's per-target read pattern | One read on open, on seat change and on the liveness tick while open; never per roster row. | A switch of instance or seat drops an in-flight answer. | JSDoc | unit |

## Decision Record impact

none (ADR 0041's rule in miniature: an observation with its age, never a stored status).

## Acceptance Criteria

- [ ] AC-1 With a bridge that serves the verb, opening a seat's detail shows its newest turn summaries from real Store records, newest first; the pill reads live with the read's age.
- [ ] AC-2 `unavailable`, an empty stream and a clamped policy each read as themselves; no sample row exists in any state.
- [ ] AC-3 A seat change or an instance switch while a read is in flight never lands the previous seat's turns.
- [ ] AC-4 A bridge without the verb keeps the unobserved pill, and its title names the Fleet verb it waits for.
- [ ] AC-5 Darwin visuals and the input stamp agree for the changed pane.
- [ ] AC-6 [post-merge] On the installed Fleet Manager against the canonical plane, a named seat's detail shows the same summaries `query_recent_turns` answers for it. Residual-Owner: #391.

## Out of Scope

- The verb itself (neomjs/neo-agent-brain#792) and the pin that carries it.
- Paging past the first page, and drilling from a turn into its session (the Memories pane owns that).
- The Pull requests pane (the open-work section decided on #449).

## Avoided Traps

- **A per-row read from the roster card:** a card cannot call a per-seat verb per row; the card's lane line took the roster-row stamp for that reason.
- **Rendering full turns:** the widened path serves public summaries only.

## Related

#391 (parent) · neomjs/neo-agent-brain#792 (blocks this) · neomjs/neo-agent-brain#741 · #435 and #418 (the sibling panes) · #449 (the open-work section beside it)

Live latest-open sweep: the latest 20 open issues at 2026-10-03T07:13Z (newest #475) and an exact search for `thought stream` over open and closed issues: no equivalent (#391 is the tracker, #435 and #418 the closed sibling leaves).
A2A in-flight sweep at 07:13Z, the latest 30 messages of every read-state: no claim on the pane.
MC sweep: shared with neomjs/neo-agent-brain#792 ("Agent Detail thought stream pane not observed, a peer's recent turns answer empty for the cockpit viewer"), 5 results, no prior decision found.
Own-assignment sweep: 3 open (#391, #475, #9); #391 is this leaf's tracker.

Origin Session ID: 25618ee4-58d2-46dd-ae26-9dcf2854b14a
Retrieval Hint: "Agent Detail thought-stream pane fleetRecentTurns Store ledger capturedAt memorySharing clamped"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 25618ee4-58d2-46dd-ae26-9dcf2854b14a

## Timeline

- 2026-10-03T07:16:54Z @neo-fable assigned to @neo-fable
- 2026-10-03T07:16:55Z @neo-fable added the `enhancement` label
- 2026-10-03T07:16:55Z @neo-fable added the `agent-os` label
- 2026-10-03T07:16:55Z @neo-fable added the `ai` label
- 2026-10-03T07:17:06Z @neo-fable added parent issue #391
- 2026-10-03T07:17:07Z @neo-fable marked this issue as being blocked by #792
- 2026-10-03T07:17:08Z @neo-fable cross-referenced by #792
- 2026-10-03T07:17:42Z @neo-fable cross-referenced by PR #795
- 2026-10-03T11:12:52Z @neo-fable-clio cross-referenced by #501
- 2026-10-03T11:58:54Z @neo-fable-clio cross-referenced by #506

