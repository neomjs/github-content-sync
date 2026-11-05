---
id: 1468
title: The roster read waits on plane calls for up to 60 s
state: CLOSED
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-04T23:12:00Z'
updatedAt: '2026-11-04T23:57:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1468'
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
closedAt: '2026-11-04T23:57:10Z'
---
# The roster read waits on plane calls for up to 60 s

## Context

neomjs/neo-agent-institution#841 (v1 row 5, H0): after a cold relaunch on 2026-10-30 the cockpit showed "fleet timed out" and got its roster 65.5 s after the window, while the fleet's transport listened 1.7 s after it. Nothing at the transport holds a read during boot: `fleetBridgeServer` dispatches each `/fleet` request on arrival, and `fleetRoster` awaits no boot gate (`startGateRefusal()` only stamps rows). What `fleetRoster` does await is its axes.

- Observed, probe on `dev` `f1fc04b9`: with a presence producer that never answers, `FleetControlBridge.fleetRoster()` is still waiting after 3,002 ms. With one that answers, it answers in 1 ms.
- Observed in source: in plane mode, every roster read makes two plane calls, each bounded only by the MCP SDK's 60 s default. The presence axis calls `who_is_online`, and the wake axis calls `manage_wake_subscription {action: 'fleet-identities'}`.
- Inference, not measured: the sitting's first roster read answered about 60 s late, and the Institution's late-answer recovery (neomjs/neo-agent-institution#707) woke the fresh read that landed at 65.5 s. No fleet or plane log times a single read.

## The Problem

The roster is the cockpit's first read. Its registry, repo and runtime parts are local and fast, yet it answers only when the slowest plane call does, up to 60 s later. A relaunch is when the plane is busiest. Each consumer of `readPresence` makes its own `who_is_online` call (the roster, open work, and the wake routes' `readUndeliverable`), so a slow plane gets more calls, not fewer. The cockpit's read budget is 10 s, so the operator reads "fleet timed out" for a fleet that is up.

## The Architectural Reality

- `ai/services/fleet/FleetControlBridge.mjs` `fleetRoster()` awaits wake, throttle and presence with `Promise.all`, then builds the roster.
- `ai/services/fleet/devFleetServer.mjs`: in plane mode, `createPlaneWhoIsOnlineReader(planeClient)` becomes the one `readPresence` for the roster, open work and the wake routes. The wake axis reads `createPlaneWakeIdentitiesReader`.
- `ai/services/fleet/planeWhoIsOnlineReader.mjs` and `planeWakeIdentitiesReader.mjs` make one plane call per invocation.
- `ai/services/fleet/planeMailboxClient.mjs` keeps the SDK's 60 s per-call bound on purpose: "matches the measured local plane under embed/WAL load (initialize ~17s, list_messages ~25s observed 2026-08-02)".
- Design authority: `fleetRoster`'s docblock, "Fail-honest end to end: an un-injected producer … yields not-wired/unknown, never a guessed state", and "Read concurrently: the axes share no state, so serializing them would just add the slower one's latency to every roster read." A bounded wait keeps both: a slow axis reads as pending, and no axis adds more than the bound.

## The Fix

1. `fleetRoster` waits on the wake and presence axes for at most a bound, a documented field that tests can inject. If an axis is still reading at the bound, the roster gives that axis its fail-honest pending state. Rows read `participationRead: {state: 'unread', reason: …}` naming the bound, and the capability degrades with the same reason. The plane call keeps running.
2. Concurrent calls to the plane `who_is_online` reader share one plane call until it settles.
3. When an axis answers after the bound, the fleet logs one line naming the axis and its duration. This is the instrument the installed receipt on neomjs/neo-agent-institution#841 needs.

## Acceptance Criteria

- [ ] Unit (`FleetControlBridge.spec.mjs`): with a presence axis that never answers, `fleetRoster()` answers within the bound and names the pending axis on its rows and capability. Red on `dev` first.
- [ ] Unit: the same for the wake axis.
- [ ] Unit (`planeWhoIsOnlineReader`): two concurrent reads make one plane call, a read after it settles makes a new one, and a rejection reaches both callers.
- [ ] Unit: an axis answering after the bound logs one line with the axis and its duration.
- [ ] Post-merge, installed: the next cold relaunch's `main.log`, recorded on neomjs/neo-agent-institution#841 beside its window-to-roster time.

## Out of Scope

- The cockpit's state line while the fleet boots (neomjs/neo-agent-institution#841 AC-3).
- The plane's own latency under load. It gets its own ticket if the instrument shows it.
- The 60 s per-call bound, which protects the mailbox writes.
- Caching a late axis answer for the next read.

## Related

neomjs/neo-agent-institution#841 (the outcome), neomjs/neo-agent-institution#424 (row 5), neomjs/neo-agent-institution#707 (initial roster recovery), #552 and #555 (`who_is_online` cost, closed).

Live latest-open + closed sweep: latest 20 open Brain issues read at 2026-11-04T23:09:56Z and re-read at 23:11:39Z; closed search "roster presence who_is_online" and "fleetRoster slow OR roster read timeout" (#1400, #874, #734, #552, #555 read: correctness and cost of `who_is_online`, not the roster's wait); no equivalent. A2A sweep: last 30, no claim on the roster or presence. MC sweep: "roster read slow after relaunch plane who_is_online presence wake axis 60 second timeout fleetRoster", no prior decision. Own-assignment: #571, #201, #121, #133, #143, none on this read. Structure map: existing files in `ai/services/fleet`, no new file.

Origin Session ID: 2171b2cd-e447-4cac-8b9f-ccadc6e20db5
Retrieval Hint: "roster read waits on plane who_is_online presence wake axis 60 s cold relaunch fleet timed out"

## Timeline

- 2026-11-04T23:12:00Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-04T23:12:01Z @neo-opus-ada added the `bug` label
- 2026-11-04T23:12:01Z @neo-opus-ada added the `ai` label
- 2026-11-04T23:12:01Z @neo-opus-ada added the `performance` label
- 2026-11-04T23:12:02Z @neo-opus-ada added the `agent-os` label
- 2026-11-04T23:16:17Z @neo-opus-ada cross-referenced by #841
- 2026-11-04T23:23:38Z @neo-opus-ada referenced in commit `9a15481` - "test(fleet): the roster answers with both plane axes still out (#1468)

Extends the roster arm to stall the wake scan beside the presence read, so AC-2 is witnessed at the roster as the ticket asks, not only at the producer."
- 2026-11-04T23:24:18Z @neo-opus-ada cross-referenced by PR #1469
- 2026-11-04T23:54:42Z @neo-opus-ada cross-referenced by #1471
- 2026-11-04T23:57:10Z @tobiu referenced in commit `f963b01` - "fix(fleet): the roster stops waiting out a slow plane call (#1468) (#1469)

* fix(fleet): the roster stops waiting out a slow plane call (#1468)

fleetRoster awaited its wake and presence axes. In plane mode each is a
plane call that only the MCP SDK's 60 s bounds, so a loaded plane after a
relaunch held the cockpit's first read past its 10 s budget
(neomjs/neo-agent-institution#841).

- FleetManager's roster producers bound their plane reads by the new
  AiConfig.fleet.rosterAxisTimeoutMs leaf (4 s). An axis still reading
  reports pending through its adapter's own fail-honest path, and the
  call runs on.
- A read that settles after the bound logs its axis and duration once.
- The plane who_is_online reader shares one call between overlapping
  reads.

* test(fleet): the roster answers with both plane axes still out (#1468)

Extends the roster arm to stall the wake scan beside the presence read, so AC-2 is witnessed at the roster as the ticket asks, not only at the producer."
- 2026-11-04T23:57:11Z @tobiu closed this issue

