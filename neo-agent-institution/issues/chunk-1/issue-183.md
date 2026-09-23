---
id: 183
title: Instance manager intents carry `source` as an id string
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-23T09:22:45Z'
updatedAt: '2026-09-23T10:39:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/183'
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
closedAt: '2026-09-23T10:39:10Z'
---
# Instance manager intents carry `source` as an id string

## Context

Seen 2026-09-23 ~09:20Z on `dev@aea24bf` (headless Chromium, #181's witness). Two App-worker console errors from the instance manager:

- **Add:** `TypeError: source.onClearClick is not a function` at `ViewportController.onSaveInstance` (:299) — the row IS added first; the editor never clears.
- **Connect + switch:** `TypeError: Cannot create property 'notice' on string 'neo-container-227'` at `ViewportController.onConnectInstance` (:364) — the switch never runs, so a bearer-backed instance switch is unreachable from the UI.

Defect-note `MESSAGE:3e302ba6-1382-44fa-a073-dc07c50df43f`, promoted by triage: Connect + switch is the only credentialed switch path, and #181's switch-back witness needs it. Sweeps 2026-09-23T09:21Z: latest open Institution issues (no equivalent), A2A (no claim), Memory Core (no prior decision).

## The Problem

`ManagerContainer` fires `saveinstance` (`onSaveClick`, the fire at :438) and `connectinstance` (`onConnectClick`, :466) with `source: me`; the ViewportController handlers call `source.onClearClick()` and write `source.notice` and receive the component's id string instead of the instance. First V-B-A: log `typeof source` at the handler and read the listener resolution at the pinned engine (`d850607`) — whether the engine's event path stamps a component `source` as its id, or the manager's own `fire` goes through a listener that re-stamps it. The same app already handles the id form: `cockpit/Controller.mjs` `onAgentLifecycleIntent` resolves `Neo.getComponent(data.source)`.

## The Architectural Reality

- `apps/agentos/view/ViewportController.mjs`: `onSaveInstance()` :267–:299, `onConnectInstance()` :357–:364, the domain listeners :250–:254.
- `apps/agentos/view/fleet/instances/ManagerContainer.mjs`: `onSaveClick()`, `onConnectClick()` :459, `onAdmitClick()` (the third intent — check it the same way).
- Precedent: `apps/agentos/view/fleet/cockpit/Controller.mjs` `onAgentLifecycleIntent()`.

## The Fix

One convention for the manager's intents: resolve the instance at the handler (`Neo.getComponent(source)` when a string arrives) or fire the id and resolve — the same for `saveinstance`, `connectinstance` and the plane-admission intent. Red-first in `ViewportController.spec.mjs` (it already builds a manager fake): the string-source arm.

## Acceptance Criteria

- [ ] Unit, red-first: `onSaveInstance` and `onConnectInstance` with `source` as the manager's id string clear the editor / write the notice on the manager instance; no TypeError.
- [ ] Unit control: the instance form of `source` keeps working if a caller passes it.
- [ ] Live: "Add" leaves an empty editor; "Connect + switch" with the dev server's handshake bearer lands the switch.

## Out of Scope

The switch's custody semantics — a bearer-less switch stays fail-closed.

unowned-rationale: found while witnessing #181; the roster-list node defect (#182) comes first on my seat. Claimable by any seat — one file plus its spec.

## Related

#181 · #182

Origin Session ID: f34cbeb6-fd44-4060-b31f-e05332e62aee
Retrieval Hint: "instance manager saveinstance connectinstance source id string TypeError"

## Timeline

- 2026-09-23T09:22:45Z @neo-fable-clio added the `bug` label
- 2026-09-23T09:22:46Z @neo-fable-clio added the `agent-os` label
- 2026-09-23T09:22:46Z @neo-fable-clio added the `ai` label
- 2026-09-23T09:23:56Z @neo-fable-clio cross-referenced by PR #184
- 2026-09-23T10:06:07Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-23T10:11:48Z @neo-fable-clio cross-referenced by PR #186
- 2026-09-23T10:39:10Z @tobiu referenced in commit `4e88d6d` - "Merge pull request #186 from neomjs/agent/183-manager-intent-source

fix(viewport): the instance manager's intents resolve the manager behind the firer id the engine stamps onto source (#183)"
- 2026-09-23T10:39:10Z @tobiu closed this issue

