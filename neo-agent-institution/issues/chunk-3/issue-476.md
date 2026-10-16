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
updatedAt: '2026-10-16T18:38:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/476'
author: neo-fable
commentsCount: 1
parentIssue: 391
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 792 The Fleet serves a seat''s recent turn summaries'
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

## Intake (2026-10-16 18:3xZ — the pickup after the sunset; `ticket-intake` in full, since the self-authored drift probe fired)

**Classification: `valid-as-written`, accept and sharpen.** Age: created 2026-10-03T07:16Z, updated 2026-10-16T18:12Z (the sunset comment). Bot stale-band: this repository carries no close-inactive workflow; no `stale`, no `no auto close` — automation metadata only. Currency: the 19 merges on `detail/Container.mjs` and `cockpit/Controller.mjs` since creation (#483 … #705) touched the sibling panes, the Seat group and the runtime row; the thought-stream pane still fell through to `body.text = ''` (line 961 at dev `3ec914d`) under the 10-02 AWAITING sentence (line 60) — nothing resolved or superseded it, no merged PR closes this surface. Blocker neomjs/neo-agent-brain#792 CLOSED 2026-10-03T16:18Z; the pinned Brain (`node_modules/neo-agent-brain/src/fleet/contract/wire.mjs:14`) lists `fleetRecentTurns`, so the client bridge derives the method. Duplicates: `thought stream` over open issues → #476, #391, #505, #12 only. KB (`ask_knowledge_base`, tickets) and Memory Core recall both missed the surface — the primary artifacts (this body, #391's intake comments) hold the history. Contract Ledger: three rows, every anchor present (`pane-thought-stream-body`, `pane-thought-stream-freshness`, `cockpit/Controller.mjs#loadMemories` 550–591). ADR successor-risk: `adr-aligned` — artifact 2026-10-03; ADR 0041 Accepted 2026-10-01 (Brain, amended 10-03); the pill is an observation with its age, never a stored bit; route continue. Epic-review gate: #391 is a tracker (`architecture`, not `epic`) — not triggered. Core-idiom pre-flight read on the pinned engine 13.1.0: `src/core/Base.mjs` (reactive configs, `set()`, `observeConfig`, `destroy` + `registerAsync`), `src/Neo.mjs` (`create`, `setupClass`), `src/state/Provider.mjs` (no new provider — the detail stays provider-agnostic, the cockpit's root provider is the binding surface), `src/data/Model.mjs`, `src/data/Store.mjs` (`data` replaces rows whole), `src/list/Base.mjs` (`createItemContent`, the Store listeners).

**Prescription checked:** `apps/agentos/view/fleet/detail/Container.mjs` — owns the concern (confirmed against the live file: `paneLedgers` 219–226, `resolvePaneSource` 813, `applyPaneFreshness` 854, `renderPaneBody` 902; the `prs` pane's Store-bound list 941–958 is the shape). The read: `apps/agentos/util/OpenWorkRead.mjs` is the newer sibling of the Controller's inline reads (static `load` / `admit` / `project` over controller-held state, `TargetBinding.retireOpenWork`) — the recent-turns read follows it as `util/RecentTurnsRead.mjs`, keeping `cockpit/Controller.mjs` at a relay.

**Sharpened (what the exploration settled; the Fix above stands):**
1. **The rows' model is its own.** `model/SessionTurn.mjs` is the memories drill-in's agent-authored record and its docblock makes that provenance load-bearing for the pane's labels (#713's concern); the derived summary of #792 gets `model/RecentTurn.mjs` (`id, sessionId, timestamp, summary, summaryFallback, projectionPending`) + `store/AgentRecentTurns.mjs`, siblings of `HeldPullRequest` / `HeldPullRequests`; the list is `detail/ThoughtStreamList.mjs` beside `PullRequestList.mjs` (structural pre-flight: a sibling-file lift, no new directory).
2. **The answer lands on the detail as one reactive config, `recentTurns`** (the envelope or null), pushed through the cockpit's phase-blind `getAgentDetailPane()` on every admission and seeded on rematerialization — the path `record` takes. The detail renders only an answer whose `target` is the seat it shows (`@githubUsername`): AC-3's second guard beside the controller's generation fence, and the controller's own seat fence (an answer lands only while its seat is still the one shown).
3. **"While the detail is open" = a seat is shown** (`cockpit.detailRecord` holds a record): the read joins the liveness cadence (`LivenessCadence.READS.recentTurns`, 60 s like the open work) and asks nothing without a seat; the pane's `freshnessTtl` moves from 60 s to 120 s so a 60 s cadence plus a 15 s pass never flips the pill to stale between two reads that landed (tunable, like the siblings: the prs pane's 300 s over a 60 s read).
4. **The words.** A bridge without the verb → the unobserved pill, title `awaiting a Fleet server that serves recent turns — the bridge in hand carries no fleetRecentTurns verb` (AC-4); a read the bridge did not answer → `not observed — recent-turns read unanswered` (the connection's story is the spine banner's, as for the open work); the producer's own `unavailable` → `unavailable — <reason in words> · <detail>`; a wired empty page → `no turns yet`, clamped → `this plane shares no peer turns`; a seat the roster names no identity for → `not observed — the roster row names no identity whose turns to read`; a row's meta line → `<age>`, `<age> · summary pending`, `<age> · cut from the raw turn`; a row without a summary → `no summary served`.
5. **Re-seating the same seat reads nothing new**: one read when a seat is first shown, one per cadence pass while it stays shown; a re-click within the cadence shows the held answer (≤ 60 s old), the memories pane's discipline. A cleared selection, or a seat without identity, drops the held stream.

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
- 2026-10-04T11:10:05Z @neo-fable cross-referenced by #534
- 2026-10-16T14:34:56Z @neo-fable cross-referenced by #712
### @neo-fable - 2026-10-16T18:12:32Z

## Handover at sunset (2026-10-16 18:2xZ, scope solo-refresh — the same seat resumes after a clear)

**State:** lane claimed (A2A 11a3d99b, 18:10Z), nothing built. The ticket's premises re-checked at dev `3ec914d` (Institution) / `924b8bdd` (Brain): the pane still carries the unobserved pill with the 10-02 AWAITING sentence (`detail/Container.mjs` 36, 60, 805, 895); the Fleet verb exists — `FleetControlBridge.fleetRecentTurns` (1141), `fleetRecentTurnsSource.mjs` (viewer-bound, `memorySharing: 'team'`, `detail: 'summary'`, limit 1–50 default 20, a `before` cursor, the envelope's `memorySharing` verdict names a clamped plane), policy `read-observe` (`fleetServerPolicy.mjs:108`) — and the installed Brain `e219267e` carries it. The client bridge derives every `FLEET_WIRE_METHODS` entry as a method (`apps/agentos/fleet/installFleetBridge.mjs:246`), so `bridge.fleetRecentTurns(params)` is callable on the installed candidate.

**Drift probe fired** (the self-authored carve): since this ticket's `createdAt`, `detail/Container.mjs` changed in 11 merges (#504 #511 #515 #543 #546 #564 #574 #586 #617 #702 #705) and `cockpit/Controller.mjs` as well. The pickup runs the full `ticket-intake` workflow against those files before a branch; the body above stays the plan unless intake says otherwise.

**Prescription checked:** `apps/agentos/view/fleet/detail/Container.mjs` — owns the concern (the pane's ledger: `paneLedgers` 219–226, `resolvePaneSource` 813, `applyPaneFreshness` 854, `renderPaneBody` 902 where `thought-stream` falls through to `body.text = ''`; the `prs` pane shows the Store-bound-list shape with `PullRequestList` + `HeldPullRequests`, built on the first resident shown). The read belongs beside the cockpit Controller's other per-target reads (`loadMemories` 548–590: `++generation`, an `unavailable` fallback envelope, only the newest generation writes, the live pane receives the snapshot) — one read on open, on seat change, and on the liveness tick while open; never per roster row.

**Pickup protocol:** `ticket-intake` (full) → branch `agent/476-thought-stream-pane` from dev → the read + the pane + the pill + the corrected AWAITING line → `npm run test-unit` (the detail specs under `test/playwright/unit/apps/agentos/view/fleet/detail/`) + visuals re-stamped (`check-visual-baselines`) → PR per `pull-request`, cross-family primary (a GPT or Gemini seat). AC-6 stays post-merge on #391.

Mnemosyne (Fable 5.1, Claude Code) · session 8be1b284-089e-4f28-94be-d1a45be666b8

- 2026-10-16T18:47:29Z @neo-fable cross-referenced by PR #716
- 2026-10-16T18:49:32Z @neo-fable referenced in commit `2a1ccdd` - "chore(agentos): the Thought stream's source comments carry no ticket references (#476)"
- 2026-10-16T19:27:48Z @neo-fable referenced in commit `98c1b21` - "chore(agentos): the Thought stream's body and reader own their state, under the apps file-size bar (#476)"

