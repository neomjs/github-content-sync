---
id: 666
title: Mailbox misses new messages while its freshness label stays live
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gpt
createdAt: '2026-10-10T18:03:05Z'
updatedAt: '2026-10-10T22:31:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/666'
author: neo-gpt
commentsCount: 0
parentIssue: 477
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 602 Loading older Mailbox rows resets the reading position'
blocking: []
closedAt: '2026-10-10T22:31:25Z'
---
# Mailbox misses new messages while its freshness label stays live

## Context

Design authority: the operator's 2026-10-10 report that Activity shows a new direct A2A message while Mailbox has no messages after 9:56am. The installed observation confirmed that timestamp is **October 9**, not today. The owning Mailbox JSDoc promises an honest live/stale freshness classification.

This is a reproduced gap under #477's installed truthful-state outcome. It is separate from #596's Activity observer modes and #602's older-page scroll reset.

## The Problem

At 17:53–17:58 UTC, the operator's installed FM showed today's 17:48 UTC direct message in Activity. Mailbox still showed the October 9 07:56 UTC message as its newest row and claimed `updated 5s ago`. Switching Activity → Mailbox kept the old list.

Live Neural Link inspection established:

| Observation | Before explicit inbox read | After the same existing read |
|---|---|---|
| Admitted viewer / subject | `@tobiu` / `@tobiu`, granted | Same, granted |
| Snapshot capture | `2026-10-09T22:40:52.406Z` | `2026-10-10T17:59:11.841Z` |
| Window | Offset 0, limit/count 50, more available | Same bounds |
| Newest message | October 9 07:56 UTC | Today's 17:48 UTC message |
| Visible result | Old rows, `updated 5s ago` | New row visible, `updated 1s ago` |

The control invoked the existing `Controller.loadOperatorInbox({offset:0})`. No code patch, message read receipt, Task resolution, send, service restart or app reload was used. The successful read and rendering isolate this incident to a retained consumer snapshot and freshness presentation; they do not establish that every possible mailbox failure has the same cause.

## The Architectural Reality

The actual installed build metadata names Institution `b089d215b2f863d95a13b6f7792607e936d9ce49`, bundled Brain `03da5025f18ca00bf83dd9d7fbdae69d5667bbdd`, Engine `e1b8fb0b1ad4631edef9e6a2892373aa11599fdd`, staged October 8. The served plane is Brain `f61bba44614e11d02f008c9dc6de32a3dce06030`. A newer built candidate is not the installed application.

At that installed Institution source:

- [LivenessCadence.mjs](https://github.com/neomjs/neo-agent-institution/blob/b089d215b2f863d95a13b6f7792607e936d9ce49/apps/agentos/util/LivenessCadence.mjs) registers Activity/roster/health/tasks/deployment/open-work reads, but no operator-inbox read.
- [cockpit/Controller.mjs](https://github.com/neomjs/neo-agent-institution/blob/b089d215b2f863d95a13b6f7792607e936d9ce49/apps/agentos/view/fleet/cockpit/Controller.mjs#L941) owns the generation-fenced inbox read. Construction/identity binding, paging, own actions and Reconnect drive it. Merely selecting the Mailbox tab does not.
- [mailbox/Container.mjs](https://github.com/neomjs/neo-agent-institution/blob/b089d215b2f863d95a13b6f7792607e936d9ce49/apps/agentos/view/fleet/mailbox/Container.mjs#L505) computes freshness in `applySnapshot()` using the held capture time and `now ?? Date.now()`. The label is written when that projection runs; no clock subsequently ages it. Its initial five-second label can therefore survive overnight.
- The pane projects into its existing `AgentMailbox` Store; that Store does not fetch. The installed mailbox mirror performs a fresh source read when requested, and the runtime control reached it successfully.

Current Institution `dev` `7867b7f92e86db7d847f82509320b578cd96ceed` still has no operator-inbox entry in the shared cadence. Its later `OperatorInbox` helper retains the same explicit-read ownership; upgrading alone is not the verified repair.

Structure map: Brain's `npm run ai:structure-map -- --files --loc` completed. The repair belongs in the existing Institution cockpit cadence/inbox and Mailbox projection owners, with their existing tests; no new subsystem or module is proposed.

## The Fix

Give the operator inbox a bounded refresh through the existing cockpit liveness owner, using the existing authenticated read and profile/generation fences. Age the freshness display from the real held capture time even when no new snapshot lands or a read fails. Preserve the last known rows with an honest stale/degraded state.

Use current source ownership rather than copying the older installed controller implementation. Reuse the shared cadence; do not add a pane-owned poller, change A2A authorization or stamp messages as read.

## Contract Ledger

| Surface | Authority | Behavior / fallback | Docs | Evidence |
|---|---|---|---|---|
| Operator inbox read | Existing `loadOperatorInbox` / `OperatorInbox`, transport-stamped viewer | Bounded refresh, one in-flight read, old-profile answers discarded; failure retains truthful last-known state | Owning controller/helper JSDoc | Delayed-response and failed-read controls |
| Mailbox freshness | Mirror `capability.capturedAt`, existing pane `now` input and state vocabulary | Clock advances without a new snapshot; old data cannot remain labeled freshly updated | Existing pane freshness contract | Held-snapshot clock advancement and rendered label |
| Mailbox Store / paging | Existing pane, Store/Grid and #602 boundary | New rows visible once published, coherent selection/thread facts; automatic refresh must not reset an older-page reading window | Existing projection contract | New-message, duplicate, selection and page controls |

Prescription checked against Institution `fd7b9471`: the shared cadence, owned inbox reader and existing pane clock/projection seams own this concern. The unchanged authenticated read exposed the missing messages. Automatic replacement defers for an older/pending/scrolled window; selecting the existing all-mail chip again explicitly returns to page zero.

Decision Record impact: none; restore the existing freshness and read ownership.

## Acceptance Criteria

- [ ] A new direct message to the admitted operator appears in an already-open first Mailbox window within its declared bounded cadence, without tab cycling, app reload or a write action.
- [ ] A held snapshot ages from live to stale and its displayed age advances when reads fail or no new snapshot arrives; it never remains `updated 5s ago` indefinitely.
- [ ] Failed/refused reads preserve the appropriate truthful state and last-known-data boundary; a failed read is not presented as an empty inbox or a fresh successful read.
- [ ] Reads are bounded and fenced: one in flight per source, no boot drain or page loop, no late previous-profile result, and none after teardown.
- [ ] Refresh preserves coherent row identities/thread facts and selection. For an older-page reading window, preserve its anchor or defer first-page replacement; do not worsen #602's separately owned continuation defect.
- [ ] Focused tests include a red-before-green new-message refresh and a frozen-label clock control, using the production owners.
## Post-Merge Validation

- [ ] A named installed candidate repeats the already-open Mailbox → new real operator message → automatic visibility journey and records its source/plane pins under #12 / #479. This leaf does not mark parent row 2 passed.

## Out of Scope

Activity observer modes (#596), the older-page append repair (#602), message/Task mutations, new permissions, cadence redesign across every cockpit surface, shared-service changes and live FM replacement during implementation.

## Related and ownership

Parent outcome: #477. Related: #12, #479, #551, #596, #602.
Owner: `neo-gpt`; the current view was refreshed as a diagnostic workaround, not a shipped fix.

Creation checks: latest 20 live open Institution issues and latest 30 all-read-state A2A messages checked immediately before filing; no same-scope ticket/claim. Own assignments #477/#596/#657 checked, and #477/#596/#602 bodies read. All-state Mailbox and org refresh searches found the distinct older-page/instance-switch work. Three symptom-shaped MC queries returned unrelated old watchdog entries; KB returned no useful match. Live source and the installed control provide the diagnosis rather than treating those retrieval misses as proof of no prior history.

Origin Session ID: 2c648caa-d36c-413c-99b3-0083aaad4b30
Retrieval Hint: "FM Mailbox October 9 09:56 updated 5s Activity new message loadOperatorInbox capturedAt"



## Timeline

- 2026-10-10T18:03:06Z @neo-gpt assigned to @neo-gpt
- 2026-10-10T18:03:07Z @neo-gpt added the `bug` label
- 2026-10-10T18:03:07Z @neo-gpt added the `agent-os` label
- 2026-10-10T18:03:07Z @neo-gpt added the `ai` label
- 2026-10-10T18:03:40Z @neo-gpt added parent issue #477
- 2026-10-10T19:07:12Z @neo-fable-clio cross-referenced by #667
- 2026-10-10T19:33:23Z @neo-gpt-sophie cross-referenced by #602
- 2026-10-10T19:50:56Z @neo-opus-ada cross-referenced by #669
- 2026-10-10T20:07:48Z @neo-gpt cross-referenced by PR #672
- 2026-10-10T20:38:19Z @neo-gpt referenced in commit `5f582d3` - "feat(mailbox): refresh the inbox and age retained observations (#666)"
- 2026-10-10T20:45:24Z @neo-gpt marked this issue as being blocked by #602
- 2026-10-10T21:40:36Z @neo-gpt referenced in commit `a3555e1` - "feat(mailbox): refresh the inbox and age retained observations (#666)"
- 2026-10-10T22:01:56Z @neo-gpt referenced in commit `74cea08` - "fix(mailbox): publish inbox read outcomes as one config batch (#666)"
- 2026-10-10T22:31:26Z @tobiu referenced in commit `cb82cb2` - "feat(mailbox): refresh the inbox and age retained observations (#666) (#672)

* feat(mailbox): refresh the inbox and age retained observations (#666)

* fix(mailbox): publish inbox read outcomes as one config batch (#666)"
- 2026-10-10T22:31:26Z @tobiu closed this issue

