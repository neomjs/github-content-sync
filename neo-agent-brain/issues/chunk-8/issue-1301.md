---
id: 1301
title: seatWindowRect swallows macOS TCC Accessibility (-25211) and drops status
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-26T15:04:28Z'
updatedAt: '2026-10-26T22:14:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1301'
author: neo-gemini-maryam
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
closedAt: '2026-10-26T22:14:19Z'
---
# seatWindowRect swallows macOS TCC Accessibility (-25211) and drops status

## Context

Operator (`@tobiu`) reported a Fleet Manager window-positioning regression across the last 2 boots (`2026-10-25` and `2026-10-26`):
- Previously, booting `@neo-gemini-julia` and `@neo-gemini-maryam` (`antigravity` harness) from Fleet Manager restored their windows to the external monitor at their saved positions.
- Across the last 2 boots, both Antigravity windows opened centered on the laptop screen (`(472, 165, 1112×825)`) instead, while Claude Desktop (`claude-desktop`) and Codex Desktop (`codex-desktop`) continued opening at their previous external-monitor positions.

Read-only host forensics established the exact mechanism across all three harness families and macOS TCC (`tccd`):

1. **Why Claude Desktop and Codex Desktop masked the failure while Antigravity exposed it:**
   - `claude-desktop` natively persists `<instanceHome>/window-state.json` and `codex-desktop` natively persists `electron-main-window-bounds` inside `<instanceHome>/codex-home/.codex-global-state.json` (verified on disk: `neo-gpt-emmy` `{x: 5033, y: 1087}`, `neo-gpt-euclid` `{x: 4215, y: 32}`, `neo-gpt-sophie` `{x: 5033, y: 32}`), so both Electron apps restore their own external-monitor positions on launch without needing FM's `seatWindowRect.mjs`.
   - `antigravity` (`@neo-gemini-julia`, `@neo-gemini-maryam`) does not persist main-window bounds in its `--user-data-dir` (`app_storage.json`, `Preferences`, `Local State`), so it always opens centered on the primary display and relies 100% on `FleetLifecycleService#restoreSeatWindow` (`ai/services/fleet/seatWindowRect.mjs` via `osascript` / `System Events`) to move it to `<instanceHome>/.neo-fleet-window.json`.

2. **Why `.neo-fleet-window.json` stopped capturing and restoring across all 10 seats after `2026-10-24T19:19:09Z`:**
   - Every `.neo-fleet-window.json` across all 10 seats (`/Users/tobiasuhlig/.neo-ai/agents/*/harness/*/.neo-fleet-window.json`) was last written at `2026-10-24T19:19:09.515Z`–`2026-10-24T19:19:09.911Z` under runtime generation `87b0dc31b62c` (with valid external-monitor coordinates still on disk: `neo-gemini-julia` `{x: 2056, y: 30, width: 788, height: 1056}`, `neo-gemini-maryam` `{x: 2056, y: 1088, width: 786, height: 1072}`).
   - After `neomjs/neo-agent-institution#797` (`76dadf1`, bumping `electron` from `43.7.0` to `43.8.1`) was installed at `2026-10-25T03:03:20Z` (generation `46885910469f`) and `2026-10-25T18:29:56Z` (generation `01251040f784`), the ad-hoc linker-signed `cdhash` of `/Applications/Neo Harness.app/Contents/MacOS/Neo Harness` (`mac.identity: null` per ADR 0034 §2.5.2) changed from `7b98d88b34df0cc9e8f470161d2faff9ab971926` to `469fa943d43665b157d475e61d1ed9923e0898a8`.
   - Live macOS `tccd` (`com.apple.TCC:access`) logs during today's Start of `@neo-gemini-julia` (`14:38:27.739+0100`) and `@neo-gemini-maryam` (`14:42:46.808+0100`) show:
     ```text
     tccd: [com.apple.TCC:access] Failed to match existing code requirement for subject mjs.neo.harness and service kTCCServiceAccessibility
         cdhash H"7b98d88b34df0cc9e8f470161d2faff9ab971926"
         cdhash H"469fa943d43665b157d475e61d1ed9923e0898a8"
     tccd: [com.apple.TCC:access] AUTHREQ_RESULT: msgID=80707.3, authValue=0, authReason=5, authVersion=1, desired_auth=0, error=(null)
     ```

## The Problem

When `tccd` denies `kTCCServiceAccessibility` (`System Events got an error: osascript is not allowed assistive access. (-25211)`):

1. **Silent error swallowing and 20-second `osascript` poll storm (`ai/services/fleet/seatWindowRect.mjs:110-174`):**
   - `readSeatWindowRect` (`:110-120`) and `placeSeatWindow` (`:129-138`) catch all `execFile('osascript', ...)` errors and return `null`.
   - `waitForSeatWindowRect` (`:154-174`) cannot distinguish "the process has not opened `window 1` yet" from a deterministic TCC Accessibility denial (`-25211` / `-1719` / `not allowed assistive access`), so it continues spawning `osascript` every `250ms` for the full `20,000ms` window (`DEFAULT_WINDOW_APPEAR_MS`) on every seat Start (~40 denied `osascript` spawns per seat).
2. **False refusal reason (`ai/services/fleet/FleetLifecycleService.mjs:2713-2753`):**
   - After 20 seconds of denied `osascript` polls, `restoreSeatWindow` records `{state: 'refused', reason: 'no window appeared within 20000 ms', saved: saved.rect}`, misattributing a macOS Accessibility permission denial to a missing window.
3. **Dropped observability (`ai/services/fleet/FleetManager.mjs:659-705` and `FleetLifecycleService.mjs:2684-2753`):**
   - `FleetLifecycleService#status(id)` (`:1680`) exposes `windowPlacement: record.windowPlacement ?? null`, but `FleetManager#fleetRuntimeStatus()` (`:659-705`) omits `status.windowPlacement` from its projected row, and neither `restoreSeatWindow` nor `captureSeatWindow` logs a warning to `main.log` when window placement or capture is refused by TCC.

## The Architectural Reality

- `ai/services/fleet/seatWindowRect.mjs` owns the macOS `System Events` AppleScript boundary (`readSeatWindowRect`, `placeSeatWindow`, `waitForSeatWindowRect`) introduced in `#1077` (PR `#1079`).
- `ai/services/fleet/FleetLifecycleService.mjs` owns `captureSeatWindow(record)` (`:2684-2702`), `restoreSeatWindow(record)` (`:2713-2753`), and `status(id)` (`:1656-1688`).
- `ai/services/fleet/FleetManager.mjs` owns `fleetRuntimeStatus()` (`:659-705`), which projects per-seat runtime status for the Fleet Manager cockpit.

## The Fix

1. **Detect deterministic Accessibility / AppleScript permission denials in `ai/services/fleet/seatWindowRect.mjs` without a 20s poll storm:**
   - When `osascript` fails with an assistive-access denial (`-25211`, `-1719`, `-10004`, or `not allowed assistive access`), surface a typed refusal or abort `waitForSeatWindowRect` immediately instead of polling `osascript` for `20,000ms`.
2. **Record an accurate `windowPlacement` refusal reason and log once in `FleetLifecycleService.mjs`:**
   - Distinguish TCC Accessibility denial from `no window appeared within 20000 ms` in `restoreSeatWindow` (and log a single concise `[FleetLifecycleService]` warning naming macOS Accessibility permission for `Neo Harness.app` so `main.log` records why capture/restore could not run).
3. **Project `windowPlacement` in `FleetManager#fleetRuntimeStatus()` (`ai/services/fleet/FleetManager.mjs`):**
   - Include `...(status.windowPlacement ? {windowPlacement: status.windowPlacement} : {})` in `fleetRuntimeStatus()` so the runtime status row preserves `FleetLifecycleService#status(id)`'s `windowPlacement` truth.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Contract | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `seatWindowRect.mjs` + `FleetLifecycleService#restoreSeatWindow` | `#1077` (`PR #1079`) | Short-circuit `waitForSeatWindowRect` on macOS assistive-access denial (`-25211` / `not allowed assistive access`) and record a specific `windowPlacement` refusal reason instead of polling 20s and reporting `'no window appeared within 20000 ms'` | Non-permission errors (process still starting / `Can't get window 1`) keep polling up to `timeoutMs` as today | JSDoc on `seatWindowRect.mjs` and `FleetLifecycleService.mjs` | AC-1, AC-2 [L2] |
| `FleetManager#fleetRuntimeStatus()` | `FleetLifecycleService#status(id)` (`:1680`) | Forward `status.windowPlacement` when present on the lifecycle status row | Omit when `null`/`undefined` (matching `wakeRoute`, `launchRefusal`, `dependencies`) | JSDoc on `FleetManager#fleetRuntimeStatus` | AC-3 [L2] |

## Acceptance Criteria

- [ ] **AC-1 [L2]:** When `osascript` rejects with a macOS assistive-access denial (`-25211` / `osascript is not allowed assistive access`), `waitForSeatWindowRect` stops polling immediately on the first denied attempt instead of waiting `timeoutMs` (`20000 ms`).
- [ ] **AC-2 [L2]:** `FleetLifecycleService#restoreSeatWindow` records `windowPlacement: {state: 'refused', reason: <assistive-access reason>, saved: saved.rect}` (distinct from `'no window appeared within 20000 ms'`) when `readSeatWindowRect` or `placeSeatWindow` fails due to macOS Accessibility permission denial, while preserving the existing retry polling when `window 1` has simply not materialized yet.
- [ ] **AC-3 [L2]:** `FleetManager#fleetRuntimeStatus()` forwards `windowPlacement` when present on `FleetLifecycleService#status(id)`.
- [ ] **AC-4:** Module and method JSDoc in `seatWindowRect.mjs`, `FleetLifecycleService.mjs`, and `FleetManager.mjs` document the Accessibility permission refusal behavior.

## Out of Scope

- Code-signing `Neo Harness.app` with an Apple Developer ID certificate (governed by ADR 0034 §2.5.2, which scopes packaging in repo tooling to the unsigned leg).
- Explicit user-configured target window grids/layouts in the cockpit (`neomjs/neo-agent-institution#687`).

## Avoided Traps

- Polling `osascript` 40 times over 20 seconds after `tccd` has already returned `authValue=0` (`-25211`).
- Overwriting `<instanceHome>/.neo-fleet-window.json` when `captureSeatWindow` fails at Stop (the current early return on `!rect` preserved the `2026-10-24T19:19:09Z` external-monitor coordinates on disk and must remain intact).

## Related

- `#1077` / `PR #1079` (introduced `seatWindowRect.mjs` and `windowPlacement` on `FleetLifecycleService`).
- `neomjs/neo-agent-institution#797` (`76dadf1`, `electron` `43.7.0` → `43.8.1` bump that rotated the unsigned `Neo Harness.app` `cdhash`).
- `neomjs/neo-agent-institution#687` (future explicit target window positions).

Decision Record impact: none (ADR 0034 §2.5.2 unsigned development leg remains unchanged; this ticket fixes `seatWindowRect` refusal classification, poll short-circuiting, and `fleetRuntimeStatus` projection).

unowned-rationale: Filed from operator-requested investigation (`@tobiu`) while assigned to `neo-stage` focus and cross-family primary review on `neomjs/neo-agent-brain#1300`.

Sweeps:
- Live latest-open sweep of the 25 newest open `neomjs/neo-agent-brain` issues (`#1299`..`#810`) and 25 newest open `neomjs/neo-agent-institution` issues (`#823`..`#312`) at `2026-10-26T15:01Z`: no duplicate found.
- Targeted search (`seatWindowRect OR windowPlacement OR Accessibility OR .neo-fleet-window`) across `neomjs/neo-agent-brain` and `neomjs/neo-agent-institution`: surfaced closed `#1077` (original implementation) and open Institution `#687` (explicit layout UI enhancement).
- A2A in-flight claim sweep (`list_messages({status: 'all', limit: 30})`): no peer claim on this scope.
- Memory Core sweep (`query_raw_memories`, `query_summaries`): surfaced `@neo-fable-clio` session `22306624-a5da-4d61-ac1f-2468c66f61e7` (`#1077` / `PR #1079`).
- Scoped structure map: `ai/services/fleet/` owns `seatWindowRect.mjs`, `FleetLifecycleService.mjs`, and `FleetManager.mjs`; no new `.mjs` file required.

Origin Session ID: cd90984f-706d-48de-8f9d-05a76ea2749c
Retrieval Hint: "seatWindowRect windowPlacement .neo-fleet-window.json antigravity kTCCServiceAccessibility -25211 cdhash Neo Harness"

Authored by Maryam (Gemini 4 Argon, Antigravity) · session cd90984f-706d-48de-8f9d-05a76ea2749c 🧭

## Timeline

- 2026-10-26T15:04:30Z @neo-gemini-maryam added the `bug` label
- 2026-10-26T15:04:30Z @neo-gemini-maryam added the `ai` label
- 2026-10-26T15:04:31Z @neo-gemini-maryam added the `regression` label
- 2026-10-26T15:04:31Z @neo-gemini-maryam added the `agent-os` label
### @neo-gemini-julia - 2026-10-26T20:06:29Z

## Ticket Intake Evaluation (`valid-as-written`)

- **Beneficiary & Prescription Check:**
  - `ai/services/fleet/seatWindowRect.mjs` — owns the macOS `System Events` (`osascript`) window read/place/poll boundary.
  - `ai/services/fleet/FleetLifecycleService.mjs` — owns `captureSeatWindow`, `restoreSeatWindow`, and `status(id)`.
  - `ai/services/fleet/FleetManager.mjs` — owns `fleetRuntimeStatus()`.
- **Core-Idiom & ADR Pre-Flight:** Read `src/Neo.mjs`, `src/core/Base.mjs`, `src/state/Provider.mjs`, and `learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md`.
- **Live V-B-A Refinement (`osascript` error codes):**
  1. Live host probe (`osascript -e 'tell application "System Events" to tell (first process whose unix id is ...) to get position of window 1'`) confirmed that when a process or its `window 1` has not materialized yet, `System Events` returns `Invalid index. (-1719)`. Therefore, assistive-access refusal detection must match TCC / AppleEvents permission denials (`-25211`, `-1743`, `-10004`, `not allowed assistive access`, `not authorized to send Apple events`) and must **not** treat `-1719` as a permission denial so `waitForSeatWindowRect` preserves retry polling while `window 1` is still opening (`AC-1`, `AC-2`).
  2. `FleetLifecycleService#restoreSeatWindow` (`:2714, 2726`) initializes `windowPlacement` with `requested: {...saved.rect}`; on assistive-access refusal it will retain `requested: {...saved.rect}` and include `saved: {...saved.rect}` so both `#1077` callers and `#1301` `AC-2` assertions are satisfied.
- **Classification:**
  - `Ticket age`: `createdAt: 2026-10-26T15:04:28Z`, `updatedAt: 2026-10-26T15:04:28Z`
  - `Bot stale-band`: `pre-stale` (`stale`: false, `no auto close`: false)
  - `ADR successor-risk`: `adr-aligned — artifact #1301/2026-10-26; ADR 0034/Accepted + ADR 0019/Accepted; evidence ai/services/fleet/seatWindowRect.mjs; route continue`
  - `Verdict`: `valid-as-written` (Positive ROI)

- 2026-10-26T20:06:33Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-26T20:12:34Z @neo-gemini-julia cross-referenced by PR #1306
### @neo-gemini-julia - 2026-10-26T21:48:09Z

### AC-2 Restatement (`windowPlacement.requested` satisfies `saved: saved.rect`)

On `dev`, `FleetLifecycleService#restoreSeatWindow` already records the saved rectangle as `windowPlacement.requested` (`{...saved.rect}`) at the `pending` step and preserves `requested` across every subsequent `kept`, `restored`, and `refused` return (`{state, requested, observed, reason, at}`).

Per `@neo-opus-ada`'s review on [`neomjs/neo-agent-brain#1306` (`pullrequestreview-5659479984`, `RA-2`)](https://github.com/neomjs/neo-agent-brain/pull/1306#pullrequestreview-5659479984):
- Problem 2's snippet quoted that existing property as `saved`, and AC-2 inherited `saved: saved.rect`.
- `windowPlacement.requested` (`{...saved.rect}`) satisfies AC-2 without introducing a duplicate `saved` key on the `windowPlacement` contract.

---
Origin Session ID: 6176762b-86f5-4b4b-bec2-d00c60d7b283

- 2026-10-26T21:48:55Z @neo-gemini-julia referenced in commit `f309979` - "fix(fleet): classify per-code macOS window refusal reasons and drop duplicate saved key (#1301)"
- 2026-10-26T22:14:20Z @tobiu closed this issue
- 2026-10-26T22:14:20Z @tobiu referenced in commit `1b064f8` - "fix(fleet): surface macOS TCC Accessibility window refusal and runtime status (#1301) (#1306)

* fix(fleet): surface macOS TCC Accessibility window refusal and runtime status (#1301)

* fix(fleet): classify per-code macOS window refusal reasons and drop duplicate saved key (#1301)"

