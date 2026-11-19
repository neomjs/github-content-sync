---
id: 1024
title: Drive cockpit and shell updates from Fleet push events over polling
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - architecture
  - performance
  - security
assignees: []
createdAt: '2026-11-19T18:12:47Z'
updatedAt: '2026-11-19T18:12:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1024'
author: neo-gemini-julia
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1865 Serve event-driven /fleet/events SSE push from devFleetServer bridge'
blocking: []
---
# Drive cockpit and shell updates from Fleet push events over polling

## Context

On `2026-11-19`, `@tobiu` reported `5+ minute` delays before PR approvals and state changes appear in Fleet Manager and challenged the cockpit/shell polling model directly: *"event driven architectures with PUSH clearly beat polling whereever possible."*

Paired producer ticket: `neomjs/neo-agent-brain#1865` (exposes authenticated `GET /fleet/events` SSE topic push on `devFleetServer` / `fleetBridgeServer`), alongside `neomjs/neo-agent-brain#1864` (keeps `openWorkReducer` head `ci` green across informative `pull_request_review` lint runs).

## The Problem

1. **Packaged desktop Fleet Manager is `100%` timer-polled across two independent loops:**
   - **Renderer cockpit (`apps/agentos/view/fleet/cockpit/LivenessController.mjs` + `apps/agentos/util/LivenessCadence.mjs`):** runs a `15 s` `setInterval` pass (`DEFAULT_PASS = 15000`) launching `8` blind polling loops (`activity`, `roster`, `openWork`, `operatorInbox`, `recentTurns` every `60 s`; `brainHealth`, `tasks`, `deploymentState` every `120 s`), completely uncoupled from when `HARNESS_BRAIN` (`devFleetServer`) actually observes state transitions (`0–75 s` artificial UI lag on top of producer pulse time, plus continuous idle load on `mc-server`).
   - **Electron `main` (`harness/operatorNotifications.mjs`):** runs a separate `60 s` `setInterval` calling `Promise.all([read('fleetRoster'), read('fleetOpenWork'), read('fleetOwnQuestions', {offset: 0})])` every minute regardless of whether anything changed.
2. **Packaged shell intentionally omits `openWakeStream`, and direct-browser `ViewerWakeController` ignores domain invalidations:**
   - `apps/agentos/fleet/installFleetBridge.mjs` (line 326: `shellTransport || Object.defineProperty(registryBridge, 'openWakeStream', ...)`) skips `openWakeStream` in the packaged Electron shell (`"Packaged-shell bridges carry NO such capability — main owns the push topology on its side of the boundary"`), so `ViewerWakeController#ensureViewerWakeStream` permanently stamps `"wake push not wired"` in `Neo Harness.app`.
   - Even on direct-browser bridges (`!shellTransport`), `ViewerWakeController#onViewerWakeSignal` (`apps/agentos/view/fleet/cockpit/ViewerWakeController.mjs`, lines 100–117) only appends rows to `viewerWakeFeed` and triggers **zero** cockpit surface reads (`loadOpenWork`, `loadRoster`, `loadActivity`, `refreshOperatorInbox`, `loadRecentTurns`).

## The Architectural Reality

- `Design authority: "Packaged-shell bridges carry NO such capability — main owns the push topology on its side of the boundary."` (`apps/agentos/fleet/installFleetBridge.mjs`, lines 319–320) + ADR 0034 §2.3.4 (`harness/preload.cjs`: `"one minimal contextBridge preload exposing named, capability-shaped affordances — never ipcRenderer raw, never Node ... capabilities land with their consuming leaves; additions amend the shell ADR §2.3 first"`) + ADR 0038 §2.5.1 rows 1 & 2 (Electron `main` holds class-1 bearer; renderer holds only a credential-free IPC session capability).
- `harness/main.mjs` (lines 1055, 1919) & `harness/fleetCapability.mjs`: `main` already owns the class-1 bearer to `HARNESS_BRAIN` (`127.0.0.1:<port>`), constructs `createOperatorNotifier`, and enforces `containsSensitiveValue` + sender-frame validation on every renderer crossing.
- `apps/agentos/view/fleet/cockpit/LivenessController.mjs` (lines 770–836) & `apps/agentos/util/LivenessCadence.mjs`: already own per-read `inFlight` caps and generation fences (`OpenWorkRead`, `RosterRow`, `OperatorInbox`) so push-triggered calls to `loadOpenWork()` / `loadRoster()` / `loadActivity()` safely coalesce and never race out-of-order responses.

## The Fix

1. **Amend `learn/agentos/decisions/0034-electron-shell-architecture.md` (§2.3.4):**
   - Record the narrow read-only push subscription capability on `neoShell` (`subscribeFleetEvents(listener) => unsubscribe`), carrying only validated `{topic, revision, observedAt, state}` invalidation envelopes from `main` to trusted same-origin cockpit windows — zero credentials, zero raw `ipcRenderer`.
2. **Electron `main` single SSE consumer (`harness/main.mjs`, `harness/fleetCapability.mjs`, `harness/operatorNotifications.mjs`, `harness/preload.cjs`):**
   - Open one class-1 SSE connection from `main` to `HARNESS_BRAIN` `GET /fleet/events` (`neomjs/neo-agent-brain#1865`) while the Fleet child is live (reconnecting with backoff on restart/cutover).
   - Drive `createOperatorNotifier` (`harness/operatorNotifications.mjs`) directly from `open-work` / `questions` / `roster` push events (coalesced within ~`200 ms`), demoting its `60 s` `setInterval` to a disconnected/fallback safety sweep.
   - Forward sanitized (`containsSensitiveValue`-checked) `{topic, revision, observedAt}` push frames and stream-liveness state to validated harness `BrowserWindow`s via `neoShell.subscribeFleetEvents`.
3. **Unified bridge + event-driven cockpit invalidation (`apps/agentos/fleet/installFleetBridge.mjs`, `ViewerWakeController.mjs`, `LivenessController.mjs`, `LivenessCadence.mjs`):**
   - Wire `shellTransport` in `installFleetBridge.mjs` to `neoShell.subscribeFleetEvents` and map `fleet-topic` frames on both shell and direct-browser (`openWakeStream`) bridges into coalesced/debounced (~`150–250 ms`, respecting window visibility — marking dirty topics while hidden and flushing once on `visibilitychange`) calls to the matching read verbs (`open-work` → `loadOpenWork`, `roster` → `loadRoster` + `loadPlaneAdmission`, `activity` → `loadActivity`, `questions` / `operator-inbox` → `refreshOperatorInbox` + `loadOpenWork`, `recent-turns` → `loadRecentTurns` when detail is open, `health` → `loadBrainHealth`, `deployment` → `loadDeploymentState`).
   - In `LivenessCadence.mjs`, when the push stream is `live` (`armed`/connected), widen routine background reconciliation intervals (e.g. `5 min` safety sweep + immediate catch-up on stream connect/reconnect) instead of firing `60 s`/`120 s` blind timers every minute; automatically restore standard polling intervals whenever the push stream is disconnected or `unavailable`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge | Docs | Evidence |
|---|---|---|---|---|---|
| `neoShell.subscribeFleetEvents` (`harness/preload.cjs`, `harness/main.mjs`, ADR 0034 §2.3.4) | ADR 0034 §2.3.4 + ADR 0038 §2.5.1 row 2 | Push sanitized `{topic, revision, observedAt}` + stream state to validated harness frames; returns `unsubscribe()` | Untrusted frame / sensitive payload refused fail-closed | ADR 0034 §2.3.4 + JSDoc | `harness` unit specs |
| `createOperatorNotifier` (`harness/operatorNotifications.mjs`) | #932 + this ticket | Trigger `tick()` immediately on `open-work` / `questions` / `roster` push topics (debounced) | Fall back to interval sweep when push stream is disconnected | JSDoc on `createOperatorNotifier` | `operatorNotifications` unit specs |
| `installFleetBridge` + `ViewerWakeController` + `LivenessController` + `LivenessCadence` (`apps/agentos/**`) | `neomjs/neo-agent-brain#1865` + `fleetWakeFanout` ("push for latency, poll for truth") | Sub-second coalesced surface refresh on `fleet-topic` push; demote blind `60 s`/`120 s` polling while push stream is live; flush dirty topics on window unhide + stream reconnect | Restore `DEFAULT_INTERVALS` polling automatically when stream is down/unwired | Anchor & Echo JSDoc | Unit + Whitebox E2E specs |

Decision Record impact: `amends adr:0034-electron-shell-architecture` (§2.3.4 `neoShell` capability allowlist) and `aligned-with adr:0038-fm-client-topology` (§2.5.1 rows 1 & 2 — `main` retains class-1 bearer custody; renderer receives credential-free invalidation signals only).

## Acceptance Criteria

- [ ] AC-1: `learn/agentos/decisions/0034-electron-shell-architecture.md` §2.3.4 documents `neoShell.subscribeFleetEvents`, and `harness/preload.cjs` + `harness/main.mjs` enforce sender-frame origin validation and `containsSensitiveValue` filtering on pushed payloads.
- [ ] AC-2: Electron `main` (`harness/main.mjs` + `harness/operatorNotifications.mjs`) consumes `HARNESS_BRAIN` `GET /fleet/events` (`neomjs/neo-agent-brain#1865`), triggering `operatorNotifications` within `<1 s` of an `open-work` / `questions` push event without blind `60 s` polling while connected, and falling back cleanly when disconnected.
- [ ] AC-3: Cockpit (`installFleetBridge.mjs`, `ViewerWakeController.mjs`, `LivenessController.mjs`, `LivenessCadence.mjs`) refreshes the targeted surface (`loadOpenWork`, `loadRoster`, `loadActivity`, `refreshOperatorInbox`, `loadRecentTurns`, `loadBrainHealth`, `loadDeploymentState`) on push events (coalescing bursts, deferring while window-hidden until visible, and catching up on reconnect), while suppressing `60 s`/`120 s` blind polling while the push stream is live.
- [ ] AC-4: Unit and E2E specs cover push-driven refresh, hidden-window dirty-topic flush, stream-disconnect fallback to polling, and sensitive-value rejection.
- [ ] AC-5 `[L3-deferred — operator handoff needed]`: Post-Merge Validation on installed `Neo Harness.app` (Residual-Owner: #414 / #516) — approving a green PR updates the cockpit `awaitingMerge` badge within `<2 s` of the producer pulse completing, and `WakeRoutePane` shows live push stream status in packaged desktop FM.

## Out of Scope

- Producer-side `GET /fleet/events` SSE endpoint and `devFleetServer` topic emission (`neomjs/neo-agent-brain#1865`).
- Filtering informative `pull_request_review` check suites out of `openWorkReducer` head `ci` (`neomjs/neo-agent-brain#1864`).
- Replacing GitHub's upstream GraphQL Search poll with org-wide webhooks (requires external ingress outside local-only desktop deployments).

## Avoided Traps

- **Exposing `bearerToken` or `EventSource` URL directly to the packaged renderer:** violates ADR 0034 §2.3 and ADR 0038 §2.5.1 row 2; `main` owns the single loopback SSE socket and bridges credential-free topic events over `contextBridge`.
- **Adding a class-3 MC operator wake subscription in `main`:** explicitly rejected by #551 and #932; subscribing to `HARNESS_BRAIN` `GET /fleet/events` uses the existing class-1 process bearer.
- **Triggering cockpit HTTP reads while the window is minimized/hidden:** wastes plane RPCs; record dirty topics on push while `livenessHidden === true` and flush once in `onLivenessVisibility`.
- **Removing fallback polling completely when the stream drops:** violates *"push for latency, poll for truth"*; `LivenessCadence` must automatically resume `DEFAULT_INTERVALS` whenever the push stream is disconnected.

## Related

- `neomjs/neo-agent-brain#1865` (Brain `/fleet/events` SSE producer — prerequisite)
- `neomjs/neo-agent-brain#1864` (review-lint `ci: pending` fix)
- `neomjs/neo-agent-brain#1842` (`Out of Scope` explicitly pointed to this Institution push ticket)
- #932 (`operatorNotifications.mjs` — `Out of Scope` deferred packaged-shell stream)
- #467 (`ViewerWakeController.mjs` extraction)
- #919 (cold-start activity feed readiness)

---

Live latest-open + closed sweep: checked latest 20 open of 57 (`red` band; `#1023`..`#819`) + open and closed searches (`LivenessCadence`, `polling`, `SSE`, `openWakeStream`, `fleetOpenWork`, `#919`, `#932`, `#467`, `#449`, Out of Scope / Avoided Traps read on `#919`, `#932`, `#467`, and `neomjs/neo-agent-brain#1842`) at `2026-11-19T18:10Z`; no open ticket owns replacing cockpit/shell polling with push events.
A2A in-flight claim sweep: latest 30 messages across all read-states (`17:32Z–18:09Z`); no overlapping claim.
MC sweep: `"cockpit polling LivenessCadence setInterval heavy caller plane list_messages SSE push"`, 8 results, no prior exclusion found.
Own-assignment sweep: 0 open in `neomjs/neo-agent-institution`.
Structure map (`npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc`, exit 0; Institution sibling controllers `ViewerWakeController.mjs` / `LivenessController.mjs` verified): no new directory role required.

unowned-rationale: Curated labels omitted because this amends ADR 0034 §2.3.4 (`contextBridge` allowlist), depends on `neomjs/neo-agent-brain#1865`, and touches the Electron security boundary; ready for maintainer pickup once `neomjs/neo-agent-brain#1865` lands.

Origin Session ID: a4bf6af6-691f-41a3-a060-2f8ab8b4ff81
Retrieval Hint: "LivenessCadence operatorNotifications replace polling with SSE push events subscribeFleetEvents ADR 0034"

## Timeline

- 2026-11-19T18:12:49Z @neo-gemini-julia added the `enhancement` label
- 2026-11-19T18:12:49Z @neo-gemini-julia added the `agent-os` label
- 2026-11-19T18:12:50Z @neo-gemini-julia added the `ai` label
- 2026-11-19T18:12:50Z @neo-gemini-julia added the `architecture` label
- 2026-11-19T18:12:50Z @neo-gemini-julia added the `performance` label
- 2026-11-19T18:12:50Z @neo-gemini-julia added the `security` label
- 2026-11-19T18:13:08Z @neo-gemini-julia marked this issue as being blocked by #1865

