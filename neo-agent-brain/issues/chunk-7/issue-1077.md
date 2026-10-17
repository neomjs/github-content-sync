---
id: 1077
title: Fleet keeps a seat's window rectangle across stop and start
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-17T16:02:59Z'
updatedAt: '2026-10-17T17:30:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1077'
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
closedAt: '2026-10-17T17:30:43Z'
---
# Fleet keeps a seat's window rectangle across stop and start

## Context

Operator-observed (Tobi, 2026-10-17 ~15:00Z, to the team): a seat booted from the Fleet Manager does not come back at its previous screen position when its harness is Antigravity or Codex Desktop; Claude Desktop seats do. His proposal: the stop-harness logic (Stop fleet included) should save the latest position when the stop succeeds. Traced the same afternoon on the maintainer host, read-only: live rectangles via `CGWindowListCopyWindowInfo` (no permission needed) show nine seats in two rows on the external display, each at its slot after the operator's manual arrangement — the baseline a restore must reproduce.

## The Problem

Fleet launches a GUI seat and lets the application put its own window where it was. Three harness families, three behaviours, measured today against the live windows:

| Harness | Where the app keeps its main-window rectangle | State today |
|---|---|---|
| Claude Desktop | `window-state.json` in the seat's user-data-dir, written on move and resize | equals the live window (mine: 3398 · 1087, 816 × 1073) → restores |
| Codex Desktop | `electron-main-window-bounds` in the seat's `codex-home/.codex-global-state.json` | the file is rewritten continuously while the seat works, the rectangle is stale for 3 of 3 seats: one holds the Electron default on the laptop display (79 · 39, 1280 × 820) while its window sits bottom-right of the external display; the other two hold earlier slots. The app records the rectangle at a lifecycle moment Fleet's stop never gives it, not on move. |
| Antigravity | nowhere found: `app_storage.json` (layouts, projects), Preferences, Local State, Local Storage, the `com.google.antigravity` defaults (one sheet frame only), no saved application state | nothing to restore from |

Fleet's stop (`ai/services/fleet/FleetLifecycleService.mjs` `stop`, :1385–1476) is SIGTERM at :1473 and SIGKILL after `sigkillTimeoutMs` 5000 (:446, :1463). Whether a longer grace would let Codex Desktop flush is not Fleet's to rely on, and it cannot help Antigravity at all. So the operator re-arranges up to four windows after every fleet start, and the sentence in Institution #687 — "preserving an already chosen window position matters now" — has no owner: #687 (Emmy, 2026-10-14) is the UI for *target* positions and keeps "the separate current-position restoration diagnosis" out of its scope.

## The Architectural Reality

- The GUI families are already a named set: `deriveHarnessLaunchSpec.mjs` `deriveWakeUserDataDir` (:427: `antigravity`, `claude-desktop`, `codex-desktop`); the launch argv carries no coordinates (antigravity `--user-data-dir` only, :194; codex-desktop :289–307).
- Fleet already writes per-seat facts into the harness home: `writeSeatLease` (:2037–2060) writes `.neo-fleet-seat-lease.json` through `writeFileAtomicSync` (mode 0o600, fsync). The lease is per launch (pid, start time) — the neighbour, not the holder, of a rectangle that must outlive a launch.
- The Fleet process already drives System Events through the osascript adapter: `ai/daemons/wake/localWakeAdapters.mjs` `buildDialogGateArgs` (:594–640) is one read-only `tell application "System Events"` query addressed by unix id, and :759 sets a process frontmost. Reading and setting a window's position and size use the same `first process whose unix id is <pid>` address and the same Accessibility grant the dialog gate already works through. For all three families the tracked child is the window's owner process.
- Host-side effects of the fleet service live in `ai/services/fleet/hostEffects.mjs`; the wake adapter's injectable `spawnAsync` effect is the test-seam precedent.
- Institution `VesselContainer.mjs` positions Fleet's own tear-outs only (#687's reality section); it is not this seam.

## The Fix

Fleet remembers the rectangle itself; the application's own restoration stays the first word, Fleet's the last.

1. **Capture at stop.** In `stop(id)` (and `stopAdoptedSeat`) for a GUI family, before the signal: one read-only System Events query for the seat's main window rectangle by pid, through a new injectable host effect (`readWindowRect(pid)` → `{x, y, width, height}` or `null`). A clean read — a visible window with positive size — is written to `.neo-fleet-window.json` beside the lease: `{version, agentId, harnessType, rect, capturedAt}`, atomic, 0o600. A failed read, a minimised or missing window leaves the previous record untouched. The record is written for every managed GUI seat with an `instanceHome`, independent of the lease: `writeSeatLease` runs only for `survivesFleetExit` seats (:1013, :1172), a condition the window record does not share. A seat move carries it for free — `moveSeatHomes` stages and renames the whole home (`moveSeatHomes.mjs:182`, recursive readdir :286) and re-checks only the lease by name (:321). *(Intake input: Mnemosyne, 2026-10-17 16:06Z.)* This runs on the stop *request*, not on the stop's success: the window exists before the signal and not after, and the rectangle is true whatever the Crashpad finalizer (#1074) decides later.
2. **Restore at start.** After spawn, for a GUI family with a record: wait for the pid's first window (bounded, ≤ 20 s, through the same effect), read its rectangle; if it equals the record the app restored itself (Claude Desktop today) and nothing happens; otherwise place it once with the recorded rectangle and read back. The result travels on the lifecycle record as `windowPlacement: {requested, observed, state: 'restored' | 'kept' | 'refused' | 'unsupported', reason}`. A read-back that differs from the request (display disconnected, the OS clamped it) is `refused` with the reason and the record is kept for the next start. No loop, no re-apply on later moves.
3. Nothing is written into a vendor file, no coordinates join the launch argv, and a platform without the adapter reports `unsupported` instead of silence.

#687's control later edits the same `.neo-fleet-window.json` — "adopt the current rectangle" is this capture on demand.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetLifecycleService#stop(id)` (:1385) and `stopAdoptedSeat` (:2121) | the stop path as it is | before SIGTERM, capture the GUI seat's window rectangle and persist a clean read | a failed read leaves the previous record; the stop proceeds unchanged | JSDoc on `stop` | AC-1 |
| `.neo-fleet-window.json` in the harness home (new, beside `.neo-fleet-seat-lease.json`, `writeSeatLease` :2037) | Fleet-owned seat-home files (#571) | `{version: 1, agentId, harnessType, rect: {x, y, width, height}, capturedAt}`, written atomically, mode 0o600 | absent = no restore (today's behaviour) | the seat-home layout note in `prepareManagedAgentWorkspace.mjs` JSDoc | AC-1, AC-2 |
| `FleetLifecycleService#start(id)` (:743 ff.) → the `lifecycleStatus(id)` row (:1540) | the existing row shape | after the first window appears, place once when observed ≠ recorded, read back; `windowPlacement: {requested, observed, state, reason}` on the row | `refused` / `unsupported` with a reason; the record kept; never a silent success | JSDoc; CARD-CONTRACT.md gains the field once the Institution shows it (sibling leaf under #687) | AC-2, AC-3, AC-4 |
| `hostEffects` window seam (new: `readWindowRect`, `placeWindow`, injectable) | the osascript adapter precedent (`localWakeAdapters.mjs` :594–640, :759) | one System Events query / one set, by unix id; darwin only | `unsupported` elsewhere | JSDoc | the AC-1–AC-4 unit seams |

Decision Record impact: none (host-side lifecycle behaviour inside the existing stop/start contract; aligned with #687's requested / observed / refused wording).

## Acceptance Criteria

- [ ] AC-1 Unit (Brain): `stop()` on a GUI-family record calls the read seam before the signal and writes `.neo-fleet-window.json` with the rectangle; with a `null` read the previous file is byte-identical afterwards; a non-GUI family never calls it.
- [ ] AC-2 Unit (Brain): `start()` with a record and an observe seam that first reports no window, then a window at another rectangle, places once with the recorded rectangle, reads back, and `lifecycleStatus(id).windowPlacement` reads `{state: 'restored', requested, observed}` with observed equal to requested.
- [ ] AC-3 Unit (Brain): observe equals the record on the first read → no place call, `state: 'kept'`; read-back ≠ request → `state: 'refused'` with the reason, the file unchanged, no second attempt.
- [ ] AC-4 Unit (Brain): a non-darwin platform reports `state: 'unsupported'` and the stop path writes nothing.
- [ ] AC-5 Post-merge, installed: on the next Institution #12 candidate, Stop fleet → Start fleet with all seats — every Antigravity and Codex Desktop seat returns to the row slot it had, Claude Desktop seats unchanged; receipt on the Row 1 walk (Fri Oct 23), rectangles read by `CGWindowListCopyWindowInfo` or the lifecycle rows.

## Out of Scope

#687's UI for choosing target positions (it edits this record later); any write into `window-state.json`, `.codex-global-state.json` or Antigravity's stores; a Fleet Manager quit without Stop (no capture point exists — Vega's 2026-10-09 census that a quit does not stop seats); Linux and Windows adapters; display enumeration (the read-back is the display check).

## Avoided Traps

A longer SIGTERM grace as the fix (helps one harness at most, none for Antigravity, and Fleet would still not know where the window was); matching windows by title (pid only); treating spawn as placement (the read-back is the proof); a continuous tiling loop (one capture at stop, one placement after start); a record without provenance (`capturedAt` and `harnessType` travel with the rectangle); recording a minimised or off-screen window; the launch-admission journal as the record's owner (`persistRuntimeState` :2379 only calls `persistAdmission()` — generation-scoped, revoked on stop, under the issuer lease — while a rectangle outlives a generation).

## Related

Institution #687 (the target-position UI; shares the record), Institution #12 (the walk that proves AC-5), Brain #1074 (the Codex Desktop stop finalizer; independent of the capture point), Brain #571 (the seat-home layout the record joins), Brain #936 (wake raises the addressed Codex instance — the same System Events ground), Brain #30 (the wake router's move off osascript for Claude seats — placement stays a host window operation either way).

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-17 15:57:10Z (created-descending); nearest #936 (Codex wake activation) and #1074 (stop finalizer) — no equivalent. Exact sweeps: Institution `window` (#687, #649, #12, #7 — none restores a harness window), `window position`; Brain `window state`, `placement` (inference placement only). A2A in-flight claims (all read-states, the last hour, at 15:59Z): none on this scope; Mnemosyne's 15:50Z note names it mine. MC sweep on the problem's nouns ("harness window opens at the wrong screen position after a Fleet Manager restart"): 8 rows, none on window placement. Own-assignment sweep: Brain #977 #850 #50 #51 #53, Institution #507 #505 #351 — none overlapping. Structure map: ran at Brain `dev` (exit 0); no new module — the owners are `ai/services/fleet/FleetLifecycleService.mjs` and `ai/services/fleet/hostEffects.mjs`, with `ai/daemons/wake/localWakeAdapters.mjs` as the adapter precedent.

unowned-rationale: filed from the operator's report by the design seat; the Fleet lifecycle steward self-selects (the stop/start phases are Grace's and Vega's prior ground; the operator opened Brain/FM backlog work to every peer today). The AC-5 witness on the Row 1 walk is mine.

Origin Session ID: 22306624-a5da-4d61-ac1f-2468c66f61e7

Retrieval Hint: "Fleet seat window rectangle capture at stop restore at start Antigravity Codex Desktop screen position"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 22306624-a5da-4d61-ac1f-2468c66f61e7


## Timeline

- 2026-10-17T16:03:01Z @neo-fable-clio added the `enhancement` label
- 2026-10-17T16:03:01Z @neo-fable-clio added the `ai` label
- 2026-10-17T16:03:01Z @neo-fable-clio added the `agent-os` label
- 2026-10-17T16:11:54Z @neo-fable-clio cross-referenced by #1078
- 2026-10-17T16:36:55Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-17T16:50:26Z @neo-fable-clio cross-referenced by PR #1079
- 2026-10-17T17:25:41Z @neo-fable-clio cross-referenced by #1081
- 2026-10-17T17:30:43Z @tobiu referenced in commit `d1a7402` - "feat(fleet): a seat's window rectangle survives stop and start (#1077) (#1079)"
- 2026-10-17T17:30:44Z @tobiu closed this issue
- 2026-10-17T18:17:43Z @neo-fable-clio cross-referenced by PR #1082

