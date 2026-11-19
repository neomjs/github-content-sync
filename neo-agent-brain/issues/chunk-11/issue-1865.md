---
id: 1865
title: Serve event-driven /fleet/events SSE push from devFleetServer bridge
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - performance
  - security
  - agent-os
assignees: []
createdAt: '2026-11-19T18:11:50Z'
updatedAt: '2026-11-19T18:11:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1865'
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
blockedBy: []
blocking:
  - '[ ] 1024 Drive cockpit and shell updates from Fleet push events over polling'
---
# Serve event-driven /fleet/events SSE push from devFleetServer bridge

## Context

On `2026-11-19`, `@tobiu` flagged `5+ minute` end-to-end Fleet Manager UI lag and challenged the cockpit/shell polling model directly: *"event driven architectures with PUSH clearly beat polling whereever possible."*

Today, the composed container `fleetServer.mjs` owns an SSE wake fan-out (`ai/services/fleet/fleetWakeFanout.mjs`), whereas the desktop Fleet bridge (`ai/services/fleet/devFleetServer.mjs` + `ai/services/fleet/fleetBridgeServer.mjs`) that `Neo Harness.app` spawns as `HARNESS_BRAIN` serves **zero push stream** — `fleetBridgeServer.mjs` (line 209) rejects everything except `POST /fleet`, `GET /fleet/probe`, and `GET /fleet/handshake` with HTTP `404` (`fleet: POST /fleet only`).

## The Problem

1. **`devFleetServer.mjs` discards exact state-transition instants that it already owns in-process:**
   - `wireFleetOpenWorkSource` (`devFleetServer.mjs:775`) finishes an open-work pulse (`transitions.length > 0` or `coverage`/`state` change) at an exact millisecond (`e.g. 17:39:07Z` when `neo#21492` & `neo#21464` flipped red, or `17:41:06Z` when `neo#21487` cleared CI), yet notifies neither Electron `main` (`operatorNotifications`) nor the cockpit UI — forcing both to blind-poll on uncoupled `60 s` (+ `15 s` pass) timers.
   - Local seat lifecycle mutations (`start`, `stop`, `bench`, `bind`, `harnessAcquisition`) and `bridgeSupervisor.onStateChange` (`devFleetServer.mjs:794`) happen inside `devFleetServer.mjs` with zero outbound push notification.
   - Plane-side A2A / turn-presence / operator-question transitions arrive on the composed plane's `/fleet/events` SSE lane (`fleetWakeSseConsumer.mjs`), yet `devFleetServer.mjs` neither relays typed invalidation topics over `fleetBridgeServer.mjs` nor refreshes a cached `questions` count — so every `FleetControlBridge#fleetOpenWork` call (`FleetControlBridge.mjs:1489-1492`) still triggers two live `readOwnQuestions` (`list_messages`) RPCs against `mc-server` (measured at `4–14.8 s` with `McpError` drops under `#1843` load).
2. **Duplicate polling load on `mc-server`:** Without a bridge push stream, the cockpit (`8` `LivenessCadence` loops) and Electron `main` (`operatorNotifications` `60 s` loop) independently hammer `POST /fleet` and `mc-server` every minute even when zero state changed.

## The Architectural Reality

- `Design authority: "**Push for latency, poll for truth.** This module owns the latency half's server side: it keeps the identity-keyed registry of connected SSE streams ... a dropped stream loses nothing durable, so this fan-out deliberately owns NO replay state."` (`ai/services/fleet/fleetWakeFanout.mjs`, lines 6–11) + `neomjs/neo-agent-institution` ADR 0038 §2.5.1 row 1 (class-1 process bearer guards `/fleet` and `/fleet/probe`).
- `ai/services/fleet/fleetBridgeServer.mjs` (lines 110–216, 270–295): `createFleetIngressGuard` already enforces exact `Host` allowlist, exact cockpit `Origin` CORS policy, and constant-time class-1 `bearerToken` verification before dispatching requests, and `drainFleetRequests(timeoutMs)` owns clean shutdown.
- `ai/services/fleet/fleetWakeFanout.mjs` (lines 47–200): already implements bounded per-identity/total SSE stream registration (`maxStreamsPerIdentity = 8`, `maxStreamsTotal = 64`), `256 KB` backpressure eviction (`MAX_BUFFERED_BYTES`), `:hb` keep-alive comments, and idempotent disposal without holding the Node event loop open (`unref()`).
- `ai/services/fleet/devFleetServer.mjs` (lines 729–805): wires `startFleetBridgeServer`, `wireFleetOpenWorkSource` (`onPulse`), `superviseFleetBridge` (`onStateChange`), `FleetManager` lifecycle operations, and `planeClient`.

## The Fix

1. **Expose authenticated `GET /fleet/events` (`text/event-stream`) on `fleetBridgeServer.mjs`:**
   - Behind the existing `createFleetIngressGuard` (`admit(req)` — same class-1 `bearerToken`, `Host`, and exact `Origin` CORS headers; zero new credential or unauthenticated surface), accept `GET /fleet/events` when an optional `eventFanout` / `registerEventStream(viewerContext, res, corsOrigin)` seam is supplied to `startFleetBridgeServer`.
   - Integrate stream disposal into `server.drainFleetRequests(timeoutMs)` so active SSE responses close cleanly on `SIGTERM`/`SIGINT` instead of hanging shutdown until `timeoutMs` expires.
2. **Publish typed, secret-free change topics from `devFleetServer.mjs`:**
   - Emit lightweight invalidation frames (`event: fleet-topic`, `data: {topic, revision, observedAt, changed}`) — carrying **invalidation metadata only, never credentials or cross-tenant bodies** — on:
     - `open-work`: inside `wireFleetOpenWorkSource`'s `onPulse` whenever `state.observedAt` advances with `transitions.length > 0` or `coverage`/`state` changes;
     - `roster`: after seat lifecycle mutations (`startAgent`, `stopAgent`, `bench`, `setPlaneCredential`, `harnessAcquisition` adoption) and `bridgeSupervisor.onStateChange`;
     - `questions` / `activity` / `operator-inbox` / `recent-turns`: when `devFleetServer`'s plane wake consumer (`createFleetWakeSseConsumer` against `<planeBase>/fleet/events`, when armed) observes a wake frame or state change;
     - `health` / `deployment`: on bridge/plane connectivity state changes.
3. **Event-driven retained `questions` cache in `FleetControlBridge#fleetOpenWork` (`ai/services/fleet/FleetControlBridge.mjs`):**
   - Cache the `{open, expired, settledAtMs}` `readOwnQuestions` summary with invalidation on `questions` / mailbox push events (and bounded background refresh when stale, bounded by `AiConfig.fleet.rosterAxisTimeoutMs` / `rosterRetainedMaxAgeMs` per ADR-0019), so `fleetOpenWork()` returns both `openWork` and fresh retained `questions` in `<1 ms` without firing two synchronous `list_messages` RPCs on every UI read.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge | Docs | Evidence |
|---|---|---|---|---|--|
| `GET /fleet/events` (`ai/services/fleet/fleetBridgeServer.mjs`) | ADR 0038 §2.5.1 row 1 + `fleetWakeFanout.mjs` | Authenticated class-1 SSE stream emitting `state` handshake + `fleet-topic` `{topic, revision, observedAt}` frames; evicted on `>256 KB` buffer or `drainFleetRequests` | Unauthenticated/bad-Origin refused fail-closed (`401`/`403`) before stream registration; `404` when `eventFanout` unwired | JSDoc on `startFleetBridgeServer` | `fleetBridgeServer.spec.mjs` |
| `devFleetServer.mjs` topic emitters (`open-work`, `roster`, `questions`, `activity`, `health`) | `fleetWakeFanout.mjs` ("push for latency, poll for truth") | Emit monotonic `revision` per topic on `onPulse` transitions, seat lifecycle mutations, `bridgeSupervisor` transitions, and plane SSE wakes | Disconnected clients catch up via standard read verbs on reconnect | JSDoc in `devFleetServer.mjs` | `devFleetServer` / fanout unit specs |
| `FleetControlBridge#fleetOpenWork` (`ai/services/fleet/FleetControlBridge.mjs`) | #1689 + #1842 + ADR-0019 | Serve retained `questions` within freshness window, invalidated immediately on `questions`/wake push events | Cold/expired cache falls back to bounded `readOwnQuestions` (`rosterAxisTimeoutMs`) | JSDoc on `fleetOpenWork` | `wireOperatorComposeWriter.spec.mjs` |

Decision Record impact: `aligned-with neomjs/neo-agent-institution#adr:0038-fm-client-topology` (§2.5.1 row 1 — reuses class-1 process bearer on `/fleet/events`; mints no new credential class) and `aligned-with` ADR-0019 (reuses existing `AiConfig.fleet` bounds at consumer sites; zero new env vars or B5 pass-through).

## Acceptance Criteria

- [ ] AC-1: `startFleetBridgeServer` serves `GET /fleet/events` (`text/event-stream`) behind `createFleetIngressGuard` (refusing missing/invalid bearer, disallowed `Host`, or disallowed `Origin` fail-closed), echoes exact `Access-Control-Allow-Origin` + `Vary: Origin` when admitted, and closes open SSE streams immediately inside `drainFleetRequests` without waiting out the drain timeout.
- [ ] AC-2: `devFleetServer.mjs` emits `fleet-topic` frames (`{topic, revision, observedAt}`) when `openWork` pulses produce transitions or coverage changes, when seat lifecycle / `bridgeSupervisor` state mutates, and when the plane wake consumer observes wake/state frames — verified by unit tests with zero secret/token leakage in emitted payloads.
- [ ] AC-3: `FleetControlBridge#fleetOpenWork` serves retained `questions` (`open` + `expired`) in memory while fresh and invalidates on push/mailbox wake, avoiding duplicate synchronous `list_messages` RPCs on back-to-back `fleetOpenWork` reads while preserving `#1689` and `#1842` fallback semantics.
- [ ] AC-4: `npm test` (unit suite for `fleetBridgeServer`, `fleetWakeFanout`, `wireOperatorComposeWriter`) and `npm run ai:lint-config-template-ssot` pass cleanly.

## Out of Scope

- Consuming `GET /fleet/events` inside `neomjs/neo-agent-institution` (`main.mjs` `operatorNotifications`, `preload.cjs` `neoShell`, and cockpit `LivenessController` / `LivenessCadence`) — tracked in the paired `neomjs/neo-agent-institution` consumer ticket.
- Excluding `pull_request_review` check suites from `openWorkReducer` head `ci` state (tracked in sibling Brain bug ticket).

## Avoided Traps

- **Creating an operator wake subscription on `mc-server` or requiring a class-3 MC credential in Electron `main` for desktop push:** explicitly rejected in `neomjs/neo-agent-institution#551` and `#932`. Pushing invalidation topics over `devFleetServer`'s existing class-1 loopback bridge requires zero new credentials or plane wake routes.
- **Pushing full multi-megabyte snapshots or raw mailbox bodies over SSE without replay state:** violates `fleetWakeFanout.mjs`'s *"push for latency, poll for truth"* contract; lightweight `{topic, revision, observedAt}` invalidations keep memory bounded (`256 KB` cap) and reuse the single canonical read verbs + generation fences in the cockpit.
- **Leaving SSE sockets open during `drainFleetRequests`:** long-lived `keep-alive` SSE responses never finish on their own and would force every `Neo Harness` quit to hit the shutdown timeout unless explicitly closed on drain.

## Related

- #1842 (`Out of Scope` explicitly deferred SSE push invalidation)
- #1689 (`fleetOpenWork().questions.expired`)
- #1843 (`mc-server` / `fleet-server` resource pressure amplified by polling)
- `neomjs/neo-agent-institution#932` and `#551` (credential boundary on Electron `main`)

---

Live latest-open + closed sweep: checked latest 20 open of 160 (`red` band; `#1863`..`#1719`) + open and closed searches (`/fleet/events`, `SSE`, `fleetBridgeServer`, `fleetOpenWork`, `#1842`, `#1689`, Out of Scope / Avoided Traps read on `#1842`, `#1689`, and `neomjs/neo-agent-institution#932`) at `2026-11-19T18:10Z`; no open ticket owns `/fleet/events` push on `devFleetServer` / `fleetBridgeServer`.
A2A in-flight claim sweep: latest 30 messages across all read-states (`17:32Z–18:09Z`); no overlapping claim.
MC sweep: `"cockpit polling LivenessCadence setInterval heavy caller plane list_messages SSE push"`, 8 results, no prior decision blocking desktop bridge SSE push.
Own-assignment sweep: 1 open (`#1230`), none overlapping.
Structure map (`npm run ai:structure-map -- --files --loc`, exit 0): owning folder `ai/services/fleet/` (`fleetBridgeServer.mjs`, `devFleetServer.mjs`, `fleetWakeFanout.mjs`, `FleetControlBridge.mjs`).

unowned-rationale: Curated labels omitted because this touches the fail-closed Fleet ingress auth and shutdown-drain boundary (`ADR 0038 §2.5.1` row 1); ready for maintainer pickup.

Origin Session ID: a4bf6af6-691f-41a3-a060-2f8ab8b4ff81
Retrieval Hint: "devFleetServer fleetBridgeServer /fleet/events SSE push invalidation openWork roster questions polling"

## Timeline

- 2026-11-19T18:11:52Z @neo-gemini-julia added the `enhancement` label
- 2026-11-19T18:11:52Z @neo-gemini-julia added the `ai` label
- 2026-11-19T18:11:52Z @neo-gemini-julia added the `architecture` label
- 2026-11-19T18:11:52Z @neo-gemini-julia added the `performance` label
- 2026-11-19T18:11:53Z @neo-gemini-julia added the `security` label
- 2026-11-19T18:11:53Z @neo-gemini-julia added the `agent-os` label
- 2026-11-19T18:13:08Z @neo-gemini-julia marked this issue as blocking #1024

