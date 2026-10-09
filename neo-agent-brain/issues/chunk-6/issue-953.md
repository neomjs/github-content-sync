---
id: 953
title: Late-joining agents see each connected app's registration and windows
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-09T16:23:12Z'
updatedAt: '2026-10-09T17:53:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/953'
author: neo-opus-vega
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
closedAt: '2026-10-09T17:53:24Z'
---
# Late-joining agents see each connected app's registration and windows

## Context

The operator, 2026-10-09 at 16:1xZ: *"FM should be reachable via FM"* (the Neural Link). The intake of neomjs/neo-agent-institution#649 ([comment](https://github.com/neomjs/neo-agent-institution/issues/649#issuecomment-6084862144)) measured that the installed Fleet Manager is connected to the bridge, yet no agent that started after it can see its windows.

Evidence, 16:18–16:25Z: my seat's Neural Link server started at 02:50:49.334Z.
- It holds an `agentos` session whose logs are live: the `brainHealth` poll every ~2 minutes, plus route changes. The session has no `appWorkerId`, `environment` or `userAgent`, and no windows.
- Portal, which connected at 15:21Z (after the agent), has all of them. `get_window_topology` lists Portal only.

## The Problem

A seat cannot drive any app that connected before the seat's own agent joined: the installed Fleet Manager, a long-lived dev tab, an open portal. Its agent never learns that app's windows. Every fresh seat session is such a late joiner, so in practice only apps opened after a seat starts can be driven.

## The Architectural Reality

- **App side.** The engine client sends `register` and one `window_connected` per window, once per socket open (neo `src/ai/Client.mjs`, `onSocketOpen`).
- **Bridge** (`ai/mcp/server/neural-link/Bridge.mjs`).
  - `handleAppMessage` (`:368`) relays each app message to the agents connected at that moment and caches nothing.
  - `registerAgent` (`:219`, replay at `:254`) sends a joining agent only `app_connected {appWorkerId, appName}` for each already-connected app.
- **Agent side** (`ai/services/neural-link/ConnectionService.mjs`).
  - `app_connected` creates a bare session stamped `connectedAt: Date.now()` (`:686`).
  - An `app_message` carrying `register` fills the worker fields (`:774`).
  - `window_connected` (`:783`) and `window_disconnected` (`:796`) maintain `windows`.
  - `activeApps` is `windows.size` (`:622`).

## The Fix

In `Bridge.mjs`, keep per connected app its last `register` message and its open windows, taken from the messages `handleAppMessage` already relays:
- `window_connected` adds a window, `window_disconnected` removes it;
- the app's close drops the entry, and a reconnect under the same id replaces it.

In `registerAgent`, after each `app_connected`, replay that app's `register` and its current `window_connected` messages as `app_message` envelopes. No new wire type; the agent side is unchanged.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `Bridge.registerAgent` replay | the app messages the bridge already relays (`register`, `window_connected`, `window_disconnected`) | A joining agent receives each connected app's `app_connected`, then its last `register` and one `window_connected` per open window, as `app_message` envelopes. | An app that never registered replays only `app_connected`, as today. A window closed before the join is not replayed. A reconnect replaces the app's entry. | `Bridge.mjs` JSDoc | unit: an agent joining after an app's registration sees its worker fields and windows |

## Acceptance Criteria

- [ ] AC-1: an agent that joins after an app registered receives the app's registration and its open windows, so its `get_window_topology` lists them. Unit-tested against the Bridge and ConnectionService.
- [ ] AC-2: a window closed before the join is not replayed. An app that reconnects under the same id replays only its new registration.

## Post-Merge Validation

- [ ] On the plane, a fresh seat session's `get_window_topology` lists the installed Fleet Manager's windows, and `set_route` reaches its rail. Residual-Owner: neomjs/neo-agent-institution#649

## Out of Scope

- The engine client and the agent side, which stay unchanged.
- Drag state: `drag_active` is transient, so an agent that joins mid-drag sees the next drag.
- Institution #649's System line.

## Avoided Traps

- **Asking each app to resend its registration when an agent joins.** That needs an engine change in every app, and a client in the middle of a reconnect answers late. The bridge already sees every message.
- **Caching every app message.** Logs and events are history, not state. Only the registration and the window topology are state.

## Related

neomjs/neo-agent-institution#649 · neomjs/neo-agent-institution#12 · neomjs/neo-agent-institution#485

Live latest-open sweep: checked the latest 20 open Brain issues at 16:22Z; no equivalent. Exact searches for "bridge replay" and "window topology agent joins" found nothing. A2A claim sweep (last 12, all states): no overlapping claim. MC sweep: no prior decision on a late-join replay; neo `#8213` (Dec 2025) only added `appName` to `app_connected`. Own-assignment sweep: nothing on the bridge. Structure map: owner `ai/mcp/server/neural-link`, agent side `ai/services/neural-link`. Decision Record impact: none.

Origin Session ID: 9a84c569-02eb-4f7c-b87d-43ebcb24593d
Retrieval Hint: "neural link bridge late joining agent replay app registration window topology"

## Timeline

- 2026-10-09T16:23:14Z @neo-opus-vega added the `bug` label
- 2026-10-09T16:23:14Z @neo-opus-vega added the `ai` label
- 2026-10-09T16:23:14Z @neo-opus-vega added the `agent-os` label
- 2026-10-09T16:23:26Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-09T16:29:17Z @neo-opus-vega cross-referenced by PR #954
- 2026-10-09T17:21:06Z @neo-opus-vega referenced in commit `2b3484c` - "fix(neural-link): a replaced app socket's late messages no longer speak for its successor (#953)

Review RA (neo-gpt, 5473190555): the current-socket guard covered close but not message. A socket replaced by a same-id reconnect could still deliver frames that overwrote the successor's registration, added stale windows or removed live ones, and relayed them to Agents. The message callback now acts only for the socket still registered under the id. The new test drives both sockets' installed handlers, with the successor as the positive control; it fails without the guard."
- 2026-10-09T17:53:24Z @tobiu referenced in commit `daff56b` - "fix(neural-link): late-joining agents see each connected app's registration and windows (#953) (#954)

* fix(neural-link): late-joining agents see each connected app's registration and windows (#953)

The Bridge relayed an App's register and window_connected messages only to the Agents connected at that moment, and replayed nothing but app_connected to an Agent that joined later. Every seat session that started after an App registered saw it connected but windowless, the installed Fleet Manager included. The Bridge now keeps each App's last registration and open windows from the messages it already relays, and replays them as app_message envelopes after app_connected. A reconnect under the same id starts afresh, and a replaced socket's close no longer evicts its successor.

* fix(neural-link): a replaced app socket's late messages no longer speak for its successor (#953)

Review RA (neo-gpt, 5473190555): the current-socket guard covered close but not message. A socket replaced by a same-id reconnect could still deliver frames that overwrote the successor's registration, added stale windows or removed live ones, and relayed them to Agents. The message callback now acts only for the socket still registered under the id. The new test drives both sockets' installed handlers, with the successor as the positive control; it fails without the guard."
- 2026-10-09T17:53:24Z @tobiu closed this issue

