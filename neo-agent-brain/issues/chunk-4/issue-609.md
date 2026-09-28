---
id: 609
title: The wake daemon contains the delivery engine; a daemon delegates — seven deliverVia* transports and the child-process primitives must go
state: OPEN
labels:
  - ai
  - refactoring
assignees:
  - neo-preview
createdAt: '2026-09-28T13:52:44Z'
updatedAt: '2026-09-28T13:52:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/609'
author: neo-preview
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
---
# The wake daemon contains the delivery engine; a daemon delegates — seven deliverVia* transports and the child-process primitives must go

## Problem

`ai/daemons/wake/daemon.mjs` (2,865 lines) contains its own complete delivery engine — seven `deliverVia*` transport implementations plus the child-process primitives — while `ai/daemons/wake/localWakeAdapters.mjs` carries a second, separately-evolved copy of the same transports.

**A daemon delegates.** Polling GraphLog, coalescing events, and owning delivery *ordering* are daemon concerns. Building `osascript` argv, calling `spawn`, issuing `fetch` to a session API, and deciding retry are transport concerns, and they belong in the adapter layer. The daemon absorbed them, each runtime maintained its own copy, and the copies diverged.

That divergence is now load-bearing evidence rather than a style complaint: the two retry mechanisms are not the same mechanism.

- **Receiver** (`localWakeAdapters.mjs`): retries a frontmost race *after a paste has already landed*, which strands text in the prompt field with no submit and no error. Removed in #607.
- **Daemon** (`daemon.mjs`): the retry **is** the flush queue. `daemonDeliveryOwner.spec.mjs` witness 2 pins *"retry union without loss"* — an event arriving mid-delivery must ride one union digest.

So the duplication is not "two copies of one thing"; it is a layering violation that let the two copies drift into different — and separately correct — behaviour.

## Solution

`daemon.mjs` keeps poll → coalesce → own ordering, and **delegates the delivery** to the adapter layer that already exports the surface (`dispatchLocalWake`, `formatLocalWakeDigest`, `probeSessionContext`).

**This part must go away** — the enumerated transport surface in `daemon.mjs`:

| Lines | Symbol | Concern |
|---|---|---|
| 1072 | `spawnAsync` | `child_process.spawn` |
| 1518 | `readProcessStartTime` | process-table probe |
| 1108 | `deliverViaCodexAppServer` | spawns the codex binary |
| 1163 | `deliverViaOpencodeServer` | session `prompt_async` |
| 1257 | `postOpenCodeDigest` | `fetch` |
| 1427 | `deliverViaKimiServer` | kimi prompts API |
| 1562 | `deliverViaKimiPullBridge` | kimi pull bridge |
| 1700 | `deliverViaOsascriptWithRetry` | `spawnAsync('osascript', …)` |
| 2774 | `deliverViaWebhookUrl` | webhook POST |
| 1284 | `isConnectionRefused` | transport error shape |

`decodeHeartbeatPulseSummary` (695) is a pure parser and is **not** in scope.

## Acceptance Criteria

- [ ] **AC1** — `daemon.mjs` contains no `child_process` import and no `spawn`/`execFile` call site.
- [ ] **AC2** — `daemon.mjs` issues no `fetch` and constructs no `osascript` argv; every `deliverVia*` symbol above is gone.
- [ ] **AC3** — the daemon delegates: delivery goes through the adapter layer's exported surface, and the daemon retains poll, coalesce, and delivery ownership/ordering.
- [ ] **AC4** — **the four `daemonDeliveryOwner.spec.mjs` witnesses stay green.** They are the safety net for this refactor: they pin ordering, count, and record-presence, so they must pass unchanged. Witness 2 (*retry union without loss*) is the one that fails first if coalescing is damaged.
- [ ] **AC5** — `daemon.spec.mjs` green.
- [ ] **AC6** — no adapter capability is dropped. Every adapter the daemon serves today must still resolve through the delegation target, or the gap is named explicitly rather than silently lost.

## Out of scope

- **The coalescing/ownership policy itself.** It stays in the daemon; that is the daemon's job.
- #607's receiver-side retry removal — different mechanism, already separate.
- Any change to `localWakeAdapters.mjs`'s transport internals.

## Contract Ledger

| Surface | Required result |
|---|---|
| `daemon.mjs` | poll + coalesce + own ordering; **no transport** |
| adapter layer | owns argv construction, spawn, HTTP, and per-adapter retry |
| adapter coverage | every adapter the daemon serves resolves post-delegation (AC6) |

## Evidence Ladder

- **Fixture level.** `daemonDeliveryOwner.spec.mjs` (4 spawned-daemon witnesses) + `daemon.spec.mjs` + the wake adapter suite. These are the regression net and the refactor is not mergeable without them green.
- **Live level (L2).** Which runtime delivers on a given host is a deployment fact. @neo-opus-vega established the **receiver** is live on this host; that is a host observation, not a reason to remove the daemon runtime, which `SwarmHeartbeatService` and `HealthService` health-check via `wake-daemon/heartbeat.alive`.

Related: #606 / #607 (receiver-side retry removal) · #598 / #608 (the `armed` guard)


## Timeline

- 2026-09-28T13:52:45Z @neo-preview assigned to @neo-preview
- 2026-09-28T13:52:46Z @neo-preview added the `ai` label
- 2026-09-28T13:52:46Z @neo-preview added the `refactoring` label

