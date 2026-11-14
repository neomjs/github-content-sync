---
id: 1710
title: 'A sunset never rotates: the handover reader matches no message'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-14T16:27:08Z'
updatedAt: '2026-11-14T17:33:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1710'
author: neo-opus-ada
commentsCount: 2
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
closedAt: '2026-11-14T16:57:20Z'
---
# A sunset never rotates: the handover reader matches no message

## Context

The operator, 2026-11-14: Claude seats sunset at about 75 % of their window, and no fresh session follows. Gemini and GPT seats rarely sunset, so this lands almost entirely on Claude seats.

The orchestrator log since 12:00Z holds six rotation decisions, all refusals and no request. Five are `prompted-after-receipt` (Grace twice, Ada twice, Vega once) and one is `newer-session` (Clio). My own receipt (MESSAGE:7e67133d, 15:17:47Z) waited fourteen minutes without a log line. The operator then prompted the old session, and the lane refused the receipt at 15:32:00.

## The Problem

`decideRotation` (`ai/daemons/wake/sessionRotation.mjs`) requests a rotation only once the receipt carries `focusItemsReceipt`. Until then it returns `wait (receipt-count-pending)`, which writes nothing and logs nothing. Only the summary lane writes that field, in the `onSuccess` of its `sunset-handover` trigger (`ai/daemons/orchestrator/scheduling/summary.mjs`).

That trigger never fires. `getUnreadSunsetHandovers` (`ai/daemons/wake/queries.mjs`) selects `json_extract(data, '$.type') = 'MESSAGE'`, but the store writes message nodes as `{id, label, properties}`. Read-only counts on the live plane (`neo-local-canonical`, revision 0a1f4cf9), 2026-11-14 ~16:30Z:

| Population | Count |
|---|---|
| MESSAGE nodes; with a non-null `$.type` | 71,043; 0 |
| handovers (`sunset-protocol-handover`); with `$.label = 'MESSAGE'`; with `$.type = 'MESSAGE'` | 580; 580; 0 |
| handovers ever stamped `focusItemsReceipt` / `handoverSummaryProcessedAt` | 0 / 0 |

So every sunset waits until a prompt refuses it, or until the 3-hour window marks it `incomplete`. While it waits, the `poll-digest` wake hold keeps the seat's wakes from it. #1401's defect-note for a handover without a focus-items count has never published either.

The specs did not catch it. `queries.spec.mjs` and `summary.spec.mjs` build handover fixtures with `type: 'MESSAGE'`, a shape the store does not write.

## The Architectural Reality

- `ai/daemons/wake/queries.mjs` `getUnreadSunsetHandovers`: the only `$.type` read of a message in `ai/`. `sunsetReceipt.mjs` reads `id LIKE 'MESSAGE:%'`, and the wake daemon's turn-presence read takes `$.label` or `$.type`.
- `ai/daemons/orchestrator/scheduling/summary.mjs` `getDueTask`: the `sunset-handover` trigger. Its `onSuccess` stamps `focusItemsReceipt`, publishes focus-item defects and sets `handoverSummaryProcessedAt`. The registry (`scheduling/registry.mjs`) passes its timings from `intervals`, which `buildOrchestratorSchedulingOptions` (`scheduling/pipeline.mjs`) maps from `AiConfig`.
- The rest of the chain is live. `@neo-opus-ada` holds an active `SESSION_ROTATE` route (`WAKE_SUB:cd5a5a16-7ec5-46da-a1bc-2d7913f60e57`: push, osascript, launch `userDataDir`), which the Fleet armed at Start (`FleetManager#armSeatWake`, routeClass `rotation`). The lane evaluates it every minute, and the receiver last accepted at 15:48Z. That falsifies Institution #842's AC-3 deferral premise ("Claude seats have no SESSION_ROTATE route yet").
- Design authority: `getUnreadSunsetHandovers`' JSDoc, "Reads unread handovers and admitted receipt retries, until summary consumption finishes." And `decideRotation`'s: "The receipt count is read from `focusItemsReceipt`: counts or `'missing'` rotate, an absent count waits for the summary lane."

## The Fix

1. `getUnreadSunsetHandovers(db, {sinceIso})` selects `$.label = 'MESSAGE'`, and only handovers sent at or after `sinceIso`.
2. The summary lane's `getDueTask` passes `sinceIso = now - sessionRotation.windowMs`. The leaf reaches the lane through `buildOrchestratorSchedulingOptions`' `intervals`, as its sweep interval already does (ADR-0019: read at that boundary, with no new pass-along).
   - A receipt older than the window can no longer rotate.
   - Without the bound, the first run would release the backlog. The plane holds 54 unread, unprocessed handovers since 2026-06-20, and 40 of them carry no focus-items count, so each would publish a stale defect-note.
3. The fixtures in `queries.spec.mjs` and `summary.spec.mjs` take the stored shape `{id, label: 'MESSAGE', properties}`.

Decision Record impact: aligned-with `adr:0019-aiconfig-reactive-provider-ssot`.

## Acceptance Criteria

- [ ] AC-1: `getUnreadSunsetHandovers` reads a handover stored as `{id, label: 'MESSAGE', properties}`. Its spec fails on `dev`.
- [ ] AC-2: A handover sent before `now - sessionRotation.windowMs` is not read, so it is neither stamped nor flagged.
- [ ] AC-3: Every handover fixture in the touched specs uses the stored node shape.

## Post-Merge Validation

- [ ] After the next plane cut, a Claude seat's sunset handover gets `focusItemsReceipt`, the orchestrator logs `session-rotation: requested`, and the receiver opens the fresh session. The record (`sessionRotation.state: 'completed'`, both session ids) is quoted here.

## Out of Scope

- The receipt stamp still waits for a heavy summary run (`onSuccess`), so a failed run leaves it unstamped until the window. Decoupling the stamp from the run is separate work.
- #1383's guards (`prompted-after-receipt`, minimum interval, live turn).
- Rotation for Codex and Antigravity, whose fresh-session keystrokes are unmeasured.
- Backfilling or retro-flagging historical handovers.

## Avoided Traps

- **Calling the missing route the cause:** the route exists and is deliverable.
- **Dropping the receipt gate from `decideRotation`:** it is #1383's contract. The broken part is the stamp.
- **An unbounded fix:** it publishes about 40 stale defect-notes on its first run.
- **Reading `$.label` or `$.type`:** no message carries `$.type`, so the second arm would only preserve the fixture analogue.

## Related

#1383 (the rotation lane) · #1401 (the focus-items receipt) · #1682 (the receiver) · #1692 (the Fleet's rotation read) · neomjs/neo-agent-institution#842 (AC-3 deferral)

Live latest-open + closed sweep: latest 20 open, plus closed searches ("sunset handover receipt", "focusItemsReceipt", "session rotation"; #1383 and #1401 read), checked 2026-11-14 ~16:30Z; no equivalent found. Open backlog: 154 (red); no open owner to fold into. A2A claim sweep: the last 30 messages hold no claim on this scope. Memory Core: no prior finding of the dead query. Own assignments read.

Ownership: self-assigned (Ada). Operator focus item, fix PR follows today.

Origin Session ID: d831ebc0-f252-4b4c-be63-03d12a613118

Retrieval Hint: "sunset handover focusItemsReceipt never stamped, getUnreadSunsetHandovers $.type vs label, rotation waits receipt-count-pending"


## Timeline

- 2026-11-14T16:27:09Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-14T16:27:10Z @neo-opus-ada added the `bug` label
- 2026-11-14T16:27:10Z @neo-opus-ada added the `ai` label
- 2026-11-14T16:27:10Z @neo-opus-ada added the `agent-os` label
- 2026-11-14T16:35:22Z @neo-opus-ada cross-referenced by PR #1711
- 2026-11-14T16:57:20Z @tobiu referenced in commit `f9b0044` - "fix(wake): read handovers by label, inside the rotation window (#1710) (#1711)

A message node is stored {id, label, properties}, so the handover reader's $.type filter matched none of the plane's 580 handovers: no receipt count was ever stamped, and every rotation waited until a prompt refused it. The read now takes the label, and only handovers inside the session-rotation window, which keeps the 54-handover backlog out of the first run."
- 2026-11-14T16:57:21Z @tobiu closed this issue
- 2026-11-14T17:03:30Z @neo-opus-ada cross-referenced by #1712
- 2026-11-14T17:03:50Z @neo-opus-vega cross-referenced by #385
- 2026-11-14T17:05:48Z @neo-opus-vega cross-referenced by PR #386
### @neo-opus-ada - 2026-11-14T17:17:05Z

**Post-merge receipt: the stamp lands.** The plane was cut to `f9b0044b` (#1711) at 17:16:11Z; all four services are healthy at that revision, and chroma and ingress were untouched.
- **17:16:14Z** `Starting session summarization (sunset-handover:2)`: the trigger's first firing on this plane.
- **17:16:25Z** stamped, read-only graph check. Handovers with `focusItemsReceipt` went from 0 to 2:
  - MESSAGE:5527eebb (Grace): `{named: 7, placed: 7}`;
  - MESSAGE:6702a2ed (Clio): `{named: 4, placed: 4}`.

  Both are `handoverSummaryProcessedAt` 17:16:25Z.
- The bound held: the 54-handover backlog stayed out, and no defect-note was published.

Still open in the same chain: a Memory Core session-id change inside one transcript still reads as a fresh session. That is #1712, with the skill side in neomjs/neo-agent-skills#386. The rotation itself (`session-rotation: requested` → the receiver opens the session) is observed on the next Claude sunset; mine follows this comment.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-14T17:33:01Z @neo-opus-ada cross-referenced by #1713
### @neo-opus-ada - 2026-11-14T17:33:27Z

**Rotation outcome for my sunset: refused, `prompted-after-receipt`, at 17:22:34Z.** Receipt `MESSAGE:95cb2238…`, sent 17:18:57Z.

- **17:19:06Z:** the summary lane stamped `{named: 4, placed: 4}`. The fix holds live.
- **17:19:06Z–17:21:56Z:** the lane ticked each minute and never requested. A `wait` writes nothing. Its inputs at those ticks left one guard open, `live-turn`: the sunset turn's interval read `active` until 18:19:07Z.
- **17:21:44Z:** the operator started the successor session by hand. Its turn interval opened at 17:21:56Z, and the next tick refused the rotation.

**Cause:** the Claude PostToolUse hook re-opens the interval `add_memory` just closed. This is a regression from my #1686, filed as #1713.

**Where the chain stands:** the receipt read, the stamp and the guards now run live. The signed `session/rotate` and the host receiver have still never been exercised.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


