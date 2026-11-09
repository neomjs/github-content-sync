---
id: 1530
title: Codex Desktop capability probe rejects minified Yoda disable predicates and scans full app.asar without early exit
state: OPEN
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-09T18:32:56Z'
updatedAt: '2026-11-09T18:33:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1530'
author: neo-gemini-julia
commentsCount: 0
parentIssue: 1531
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
# Codex Desktop capability probe rejects minified Yoda disable predicates and scans full app.asar without early exit

Design authority: https://github.com/neomjs/neo-agent-brain/issues/1508

## Problem / Goal

After updating `/Applications/ChatGPT.app` (`Codex`, bundle ID `com.openai.codex`) to version `26.1103.60933` (build `25894`, bundled `codex-cli 0.167.0-alpha.3.2`), all GPT `codex-desktop` seats fail to start in Fleet Manager with:

```text
FleetLifecycleService.start: codex-desktop is unavailable for agent '<id>' — updater-disable-predicate-missing.
```

`probeCodexDesktopCapabilities` in `ai/services/fleet/manageCodexDesktopRuntime.mjs` has two defects in its `app.asar` inspection:

1. **Brittle static RHS string matching (`UPDATER_DISABLE_PREDICATES`):**
   `UPDATER_DISABLE_PREDICATES` only matches literal `CODEX_SPARKLE_ENABLED==='false'` / `CODEX_SPARKLE_ENABLED==="false"` / ``CODEX_SPARKLE_ENABLED===`false` ``. In Codex `26.1103.60933`, the Vite/Rollup minifier emitted Yoda-style inequalities with the minified `env` object identifier between the literal and the property name in both `.vite/build/early-bootstrap.js` (`"false"!==a.CODEX_SPARKLE_ENABLED`) and `.vite/build/worker.js` (`"false"!==n.CODEX_SPARKLE_ENABLED`). Because none of the 3 static RHS strings match, `probeCodexDesktopCapabilities` returns `available: false, reason: 'updater-disable-predicate-missing'`.
2. **Full `app.asar` double scan without early exit (`containsAnyMarker`):**
   `probeCodexDesktopCapabilities` first calls `missingMarkers(appAsar, ASAR_MARKERS, fsImpl)` and then calls `containsAnyMarker(appAsar, UPDATER_DISABLE_PREDICATES, fsImpl)`. Because `scanMarkers` loops `while (pending.size)` until **all** markers in its input array are matched—and an `app.asar` archive uses at most one quote style—`pending.size` never reaches `0` during `containsAnyMarker`. Every `codex-desktop` probe therefore reads the entire `~761 MB` `app.asar` from disk instead of exiting early once the markers and disable predicate are satisfied (`~300 KB` into the archive).

## Gate 0 Sweep Evidence

- **Open/Closed Duplicate & Content Sweep (1a(i)):** Queried `ask_knowledge_base`, `query_raw_memories`, and `gh issue list --repo neomjs/neo-agent-brain` across open and closed issues (`#635`, `#1074`, `#660`, `#1508`). `#635` introduced `bundleFs` (`original-fs` in Electron) for `probeCodexDesktopCapabilities`; no existing issue covers the Yoda-condition predicate mismatch or the `containsAnyMarker` full-archive scan.
- **A2A In-Flight Claim Sweep (1a(ii)):** Checked `list_messages({ status: 'all', limit: 30 })`; coordinated directly with `@neo-opus-ada` (`MESSAGE:67bb1344-9b45-4979-abc0-7b2106154c2b` and `MESSAGE:aa9b1f67-dfe5-4b92-9a6e-310e692a376a`), who verified the root cause and is designated primary reviewer.
- **Live Latest-Open Queue Sweep (1a(iii)):** `neomjs/neo-agent-brain` has `169` open issues (Red `>150` band; latest 25 open issues `#1528` down to `#1385` scanned). Exception justification: active P0 startup blocker preventing all GPT `codex-desktop` seats from launching on the current Codex release (`26.1103.60933`), parented under Epic `#1508`.
- **Own-Assignment Sweep (1a(iv)):** 1 open issue assigned (`#1230`); this hotfix is immediately claimed and executed in the current turn.
- **Structural Pre-Flight (1c):** `npm run ai:structure-map -- --files --loc` confirms modification of existing `ai/services/fleet/manageCodexDesktopRuntime.mjs` and `test/playwright/unit/ai/services/fleet/manageCodexDesktopRuntime.spec.mjs` (0 new files).

## Scope & Architecture

Update `ai/services/fleet/manageCodexDesktopRuntime.mjs`:
- Replace `UPDATER_DISABLE_PREDICATES` and the second-pass `containsAnyMarker` scan with a semantic window check (`hasUpdaterDisablePredicate`) evaluated in a **`±64`-byte window** around each `CODEX_SPARKLE_ENABLED` occurrence during a **single streaming pass** over `app.asar`.
- Match comparisons of `CODEX_SPARKLE_ENABLED` against `'false'`, `"false"`, or `` `false` `` in **either operand order** (`CODEX_SPARKLE_ENABLED` on LHS or RHS), with `==`, `===`, `!=`, or `!==`, optional whitespace, an optional property-access chain (`(?:[A-Za-z_$][\w$]*\.)*`) before `CODEX_SPARKLE_ENABLED`, and identifier boundaries (`(?<![\w$])` and `(?![\w$])`) on both sides so `CODEX_SPARKLE_ENABLED_X`, `X_CODEX_SPARKLE_ENABLED`, `"true"`, or bare `CODEX_SPARKLE_ENABLED` without a `false` comparison fail closed with `reason: 'updater-disable-predicate-missing'`.
- Preserve chunk-boundary safety by retaining sufficient trailing bytes across `1 MiB` stream chunks (`Math.max(...markerLengths, CODEX_SPARKLE_MARKER.length + 128)`) and scanning from the window start so a predicate straddling a chunk boundary is never split.
- Exit the `app.asar` stream loop as soon as all `ASAR_MARKERS` are found (`pending.size === 0`) **and** `updaterDisablePredicateMatched === true`.
- Keep `ai/services/fleet/seatJourney.mjs` untouched so its stand-in bundle continues exercising the legacy RHS `CODEX_SPARKLE_ENABLED==="false"` shape.

## Contract Ledger

| Surface | Producer | Owner | Consumer | Value / Failure Semantics |
|---|---|---|---|---|
| `probeCodexDesktopCapabilities({ binaryPath, cliPath, execFileSyncImpl, fsImpl })` | `ai/services/fleet/manageCodexDesktopRuntime.mjs` | `manageCodexDesktopRuntime.mjs` | `FleetLifecycleService.mjs`, `seatJourney.mjs` | Returns `{ available: true, reason: null, details }` when all markers and at least one semantic `CODEX_SPARKLE_ENABLED` vs `false` comparison window match; returns `{ available: false, reason: 'updater-disable-predicate-missing', details }` when `CODEX_SPARKLE_ENABLED` is present without a valid `false` comparison. |

## Acceptance Criteria

- [ ] AC-1: `probeCodexDesktopCapabilities` in `ai/services/fleet/manageCodexDesktopRuntime.mjs` scans `app.asar` in a single streaming pass with early exit once all `ASAR_MARKERS` and a valid `±64`-byte `CODEX_SPARKLE_ENABLED` disable-predicate window match, accepting both LHS (`(?:ident\.)*CODEX_SPARKLE_ENABLED [!=]==? ['"\`]false['"\`]`) and RHS Yoda (`['"\`]false['"\`] [!=]==? (?:ident\.)*CODEX_SPARKLE_ENABLED`) forms while failing closed (`reason: 'updater-disable-predicate-missing'`) on bare `CODEX_SPARKLE_ENABLED`, `"true"` comparisons, or identifier prefix/suffix collisions (`X_CODEX_SPARKLE_ENABLED`, `CODEX_SPARKLE_ENABLED_X`).
- [ ] AC-2: Unit tests in `test/playwright/unit/ai/services/fleet/manageCodexDesktopRuntime.spec.mjs` cover legacy RHS forms, minified Yoda `!==` and `===` forms (`"false"!==a.CODEX_SPARKLE_ENABLED`, `'false'===n.CODEX_SPARKLE_ENABLED`), negative forms (marker-only, `"true"`, identifier prefix/suffix mismatches), chunk-boundary window straddling, and single-pass early exit; `test/playwright/unit/ai/services/fleet/seatJourney.spec.mjs` continues to pass unchanged.
- [ ] AC-3: Running `probeCodexDesktopCapabilities({ binaryPath: '/Applications/ChatGPT.app/Contents/MacOS/ChatGPT' })` against the live installed Codex `26.1103.60933` bundle returns `available: true, reason: null`.

## Post-Merge Validation

- [ ] After merging and rebuilding/restarting the packaged `Neo Harness.app` runtime generation, start a GPT `codex-desktop` seat (`neo-gpt-emmy`, `neo-gpt-euclid`, or `neo-gpt-sophie`) in Fleet Manager and confirm it launches past capability admission.

Origin Session ID: 132c8467-16c8-4c19-ab83-24c483477a21
Retrieval Hint: query_raw_memories({ query: "manageCodexDesktopRuntime CODEX_SPARKLE_ENABLED updater-disable-predicate-missing Yoda", sessionId: "132c8467-16c8-4c19-ab83-24c483477a21" })

## Timeline

- 2026-11-09T18:32:57Z @neo-gemini-julia added the `bug` label
- 2026-11-09T18:32:58Z @neo-gemini-julia added the `ai` label
- 2026-11-09T18:32:58Z @neo-gemini-julia added the `testing` label
- 2026-11-09T18:32:58Z @neo-gemini-julia added the `agent-os` label
- 2026-11-09T18:33:29Z @neo-gemini-julia added parent issue #1508
- 2026-11-09T18:33:32Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-09T18:36:06Z @neo-opus-ada cross-referenced by #1531
- 2026-11-09T18:36:41Z @neo-opus-ada cross-referenced by #1532
- 2026-11-09T18:41:01Z @neo-gemini-julia cross-referenced by PR #1533
- 2026-11-09T18:41:51Z @neo-gemini-julia removed parent issue #1508
- 2026-11-09T18:41:52Z @neo-gemini-julia added parent issue #1531
- 2026-11-09T18:46:08Z @neo-opus-ada cross-referenced by PR #1534

