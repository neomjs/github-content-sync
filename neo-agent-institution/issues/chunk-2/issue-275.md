---
id: 275
title: The N-window film beat's mailbox vessel closes before its tear-out URL
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-09-27T00:41:57Z'
updatedAt: '2026-09-27T00:50:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/275'
author: neo-opus-grace
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
---
# The N-window film beat's mailbox vessel closes before its tear-out URL

## Context

Split from #274 (the local Neural Link battery's red specs). `FleetCockpitNWindowNL` › "detail vessel + mailbox mid-gesture vessel stay live, update and return as the same instances" fails identically on engine `2965d82` and dev `a50ae57ce8` (Brain checkout runtime root, private bridge port). CI never runs it.

## The Problem

Phase 2 drives the operator Mailbox tab outward in four 16 ms `mousemove` samples. The mailbox popup opens (`mailboxPopupPromise` resolves), then closes before its URL carries `tearout=operator`: `page.waitForURL: Target page, context or browser has been closed` at spec line 392. The spec's own `pageerror` capture records nothing before the close. It is not yet known whether the test's gesture or the engine's mid-gesture vessel lifecycle is at fault. The dock v13.2 lane changed this area recently (`#19254` converted drags, `#19299` cross-window proxy copies, `#19301` Escape across windows).

## The Architectural Reality

The mid-gesture vessel is the dock engine's tear-out: the Group holds the vessel as a provisional connection until the terminal commits (the spec's own notes after line 392). The popup's URL gains `tearout=operator` when the vessel adopts the pane.

## The Fix

Reproduce with a trace, find what closes the popup (a sensor end, a cancelled drop, the provisional connection's retirement), then fix the test's gesture or file the engine defect.

## Acceptance Criteria

- [ ] AC-1: the cause is named with evidence (trace or App-worker log), and the fix or the engine ticket follows from it.
- [ ] AC-2: `FleetCockpitNWindowNL` passes locally on the Institution's engine pin, or waits on a linked engine ticket.

## Out of Scope

- The battery's other specs (#274).

## Related

#274 (parent sweep), #10. Dock lane: `#19254`, `#19299`, `#19301`.

Origin Session ID: 6408fcd4-3571-4ec2-8009-b4dae5d18917
Retrieval Hint: "FleetCockpitNWindowNL mailbox mid-gesture vessel closes before tearout URL"

## Timeline

- 2026-09-27T00:41:58Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-27T00:41:58Z @neo-opus-grace added the `bug` label
- 2026-09-27T00:41:59Z @neo-opus-grace added the `agent-os` label
- 2026-09-27T00:41:59Z @neo-opus-grace added the `ai` label
- 2026-09-27T00:41:59Z @neo-opus-grace added the `testing` label
- 2026-09-27T00:42:24Z @neo-opus-grace cross-referenced by #274
### @neo-opus-grace - 2026-09-27T00:45:45Z

**It is a race, not a deterministic break.** The full battery on #274's branch (dev `1acb7ec` plus the test fixes, engine `a50ae57ce8`, Brain checkout runtime root, private port) passed `FleetCockpitNWindowNL` at 2026-09-27T00:5xZ. The same spec failed three earlier runs: the full battery on each engine pin, and one isolated run, all at the mailbox popup's `waitForURL` (line 392). A race points at the gesture's timing: four 16 ms samples, then a popup that must adopt the pane before it closes. Next: reproduce under CPU throttling (CDP `Emulation.setCPUThrottlingRate`) to make it deterministic, then read which side closes the popup.

🖖 Grace · @neo-opus-grace · session 6408fcd4-3571-4ec2-8009-b4dae5d18917

- 2026-09-27T00:46:24Z @neo-opus-grace cross-referenced by PR #276
### @neo-opus-grace - 2026-09-27T00:50:24Z

**Deterministic under CPU throttling.** A scratch copy of the spec with `Emulation.setCPUThrottlingRate` on the main page, same runtime root and port:

| CPU | Runs | Result |
|---|---|---|
| 1× | 3 | 1 pass; 2 fail at the popup's `waitForURL` with `net::ERR_ABORTED; maybe frame was detached?` |
| 4× | 3 | 3 fail: `Target page, context or browser has been closed` |

A probe inside the throttled run shows what happens around the popup:
- The popup opens already at `/apps/agentos/childapps/widget/index.html?tearout=operator`, so the tear-out URL is right from the first frame and line 392's wait should resolve at once.
- Within 2.5 s the page is gone, yet a `close` listener on that Page never fired. Either the window was replaced under the same name (the Page object detached rather than closed), or the vessel retired mid-gesture and the window went with it. The App worker logs only three `Window.onWindowConnect` lines and no error.
- At 1×, `ERR_ABORTED` / frame detached points the same way: the popup's frame is navigated or detached while the gesture is still live.

Next, for whoever takes this (the v13.2 dock lane knows the vessel lifecycle best): open the 4× trace (`test-results/e2e/artifacts/*ProbeNWindowThrott*/trace.zip` reproduces with the injected throttle). Read whether the engine closes and reopens the vessel window between the provisional connection and the commit. If it does, a slow machine loses its mid-gesture vessel, which is a product defect, not a test one. Note that this spec is the N-window film beat (#15650), which the operator has deprioritized ("not the video tour"), though the vessel behavior underneath is dock stability.

🖖 Grace · @neo-opus-grace · session 6408fcd4-3571-4ec2-8009-b4dae5d18917



