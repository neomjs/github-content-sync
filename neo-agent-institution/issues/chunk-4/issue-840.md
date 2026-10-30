---
id: 840
title: Observatory's Open on GitHub opens nothing and drops the window's port
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-30T21:07:42Z'
updatedAt: '2026-10-30T21:07:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/840'
author: neo-opus-vega
commentsCount: 0
parentIssue: 312
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
# Observatory's Open on GitHub opens nothing and drops the window's port

## Context

Row 3's installed sitting (#485, 2026-10-30; installed Neo Harness: Institution `20d89be` / Brain `1bf8bd91` / engine `477b65bd9e`) ran check 7 of the packet (#312, comment 5966703991; #333 AC-7: *select an issue, a merged PR and a session and open each*).

On a freshly reloaded page the Selected node section held `neomjs/neo#issue-19515` (ISSUE · OPEN · source github) and its Open control was, read over the Neural Link (`query_vdom`): an `a` element, `href` `https://github.com/neomjs/neo/issues/19515`, target `_blank`, text "Open on GitHub". **The operator's click opened nothing; the app kept focus.** An earlier `simulate_event` click on the same control, before the reload, behaved the same way.

Both times the app worker's console then recorded, in order: `vdom update wedged: "neo-button-75" has been in-flight for over 5000ms` (the neomjs/neo#12946 class), then (human run) the same for `neo-button-1`, then `worker.Base#promiseMessage: no live port for destination "<windowId>" … a window closed?`. The Neural Link bridge listed no window for the app afterwards (`healthcheck` windows: []), the shared tooltip stayed on screen although the worker held it `hidden: true` (its hide never reached the DOM), and the Observatory stopped taking updates until a View → Reload. The roster view's awaiting-merge links open GitHub in the browser on the same build (operator, same sitting), so the external hand-off itself works.

## The Problem

Opening a GitHub item from the Observatory severs the window's port to the app worker instead of opening the page. Every later update for that window is lost until a reload; nothing tells the operator why the screen went still.

## The Architectural Reality

- `apps/agentos/view/fleet/goldenpath/ObservatorySelectionContainer.mjs:313–315` sets the Open button's `text`, `tooltip` and `url` from `facts.source`; `Neo.button.Base` (engine `src/button/Base.mjs`, `url_`/`urlTarget_`, `onClick` at :576) renders the button as an anchor with `href` and `target` and runs the handler, menu, route and ripple on click. `onOpenClick` (:269–272) handles only the session case.
- The harness decides every popup and `target="_blank"` anchor in `harness/main.mjs` `configureWebContents` (`setWindowOpenHandler` → `windowOpenDisposition`, `harness/contentPolicy.mjs:193`); an allowlisted `https:` host (`EXTERNAL_LINK_HOSTS`, `:161`, holds `github.com`) is meant to leave for the operator's browser while the shell opens nothing.
- The roster's links reach the browser; the Observatory's anchor-in-a-button does not and takes the port with it. The divergence between the two paths is the thing to find, in a red arm before any fix.

## The Fix

1. An NL/e2e arm on `dev`: select a GitHub item in the Observatory, click its Open control, assert the window stays connected (the bridge still lists it, no wedged update) and that one window-open request with the item's URL reaches the shell. Red first.
2. Route the Observatory's Open through the same mechanism the roster links use, or fix the anchor path in the button, whichever the arm names; the session case (`sessionOpen` → the Memories drill) stays as it is.
3. BEFORE/AFTER screenshot pair in the PR body (the Selected node section before and after a click).

## Acceptance Criteria

- [ ] The arm reproduces the port loss or the missing open on `dev` and goes green with the fix.
- [ ] On the installed build, an issue and a merged PR open in the operator's browser from the Observatory and the window stays live (Neural Link lists it; no `vdom update wedged` line).
- [ ] The tooltip of the Open control hides when the pointer leaves, with no reload.
- [ ] BEFORE/AFTER screenshot pair in the PR body.

## Out of Scope

The session case could not be exercised: the installed read carries no session with a uuid id (the 500-node list under "by relations" holds 33 seeded placeholders such as `session:current` with 6–60 relations; selecting `neomjs/neo#session:405a309b-…` cleared the selection), and the placeholders hide Open by design (#333 AC-5). That is a note on #312, not this leaf. The tooltip stickiness is a symptom of the severed port, not a second defect.

## Related

Parent: #312. Receipt: #485 (the 2026-10-30 sitting). #333 AC-5 and AC-7. neomjs/neo#12946 (the wedge class). #649 (stable Neural Link window names on the installed FM).

Live latest-open sweep: latest 20 open Institution issues read at 2026-10-30T21:06:46Z; no equivalent. A2A sweep: last 30 `lane-claim` messages, none on the Observatory. MC sweep: "Fleet Manager Open on GitHub link target _blank Electron window open handler nothing happens worker port disconnected", 5 results, no prior decision. Own-assignment sweep: 4 open (#732, #651, #649, #485), none overlapping.

Origin Session ID: 40918982-6f4d-45f6-8b01-c5f951c22edf
Retrieval Hint: "row 3 sitting check 7 Open on GitHub opens nothing window port lost vdom update wedged"

## Timeline

- 2026-10-30T21:07:42Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-30T21:07:44Z @neo-opus-vega added the `bug` label
- 2026-10-30T21:07:44Z @neo-opus-vega added the `agent-os` label
- 2026-10-30T21:07:44Z @neo-opus-vega added the `ai` label
- 2026-10-30T21:09:11Z @neo-opus-vega added parent issue #312
- 2026-10-30T21:10:05Z @neo-opus-vega cross-referenced by #485
- 2026-10-30T21:10:07Z @neo-opus-vega cross-referenced by #505
- 2026-10-30T21:10:45Z @neo-opus-vega cross-referenced by #312

