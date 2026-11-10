---
id: 1542
title: Wake GUI seats by resolved process automation ID across all families
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-09T19:48:10Z'
updatedAt: '2026-11-09T21:31:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1542'
author: neo-gemini-maryam
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
closedAt: '2026-11-09T21:31:58Z'
---
# Wake GUI seats by resolved process automation ID across all families

### Goal / Terminal Predicate

Whenever a macOS GUI wake or resume route resolves a running target process PID (`instancePid` from `addressType: 'userDataDir'`, `addressType: 'pid'`, or `getDefaultTarget`), `osascript` delivery targets that exact running process via `System Events` numeric `application process id targetAutomationId` (`id of candidate` matched by `unix id`) across **all** GUI families (`Antigravity`, `Claude`, `Codex`, `OpenCode`) in `ai/daemons/wake/localWakeAdapters.mjs`, `ai/daemons/wake/daemon.mjs`, and `ai/scripts/lifecycle/resumeHarness.mjs` — never executing LaunchServices `tell application "<appName>" to activate`, `id of application "<appName>"`, or `first application process whose unix id is <pid>`.

### Problem / Context

Under Epic #1531 ("Fleet owns harness updates: fetched, admitted, adopted at restart"), managed GUI seats (`codex-desktop`, `antigravity`, `claude-desktop`) run from admitted generation copies in the Fleet store sharing the same `CFBundleIdentifier` and process name as `/Applications/<App>.app` and sibling seat instances.

Verify-Before-Assert across the wake and lifecycle surfaces revealed three concrete gaps:

1. **`ai/daemons/wake/localWakeAdapters.mjs` gates process-ID targeting on `addressedCodex` (`appName === 'Codex' && Boolean(instancePid)`, lines 623 & 786):**
   - Commit `76e0165b` (#1372 / #936) introduced `codexAutomationIdLines` — resolving Unix `targetProcessId` to `System Events`' unique numeric `id of candidate` (`targetAutomationId`), foregrounding via `set frontmost of application process id targetAutomationId to true`, verifying `currentPid is targetProcessId` without bundle-ID fallback, and sending keystrokes via `tell application process id targetAutomationId` — but gated it on `appName === 'Codex' && Boolean(instancePid)`.
   - Every non-Codex GUI route with a resolved `instancePid` (`Antigravity`, `Claude`, `OpenCode` — including every Fleet-launched `Antigravity` seat armed with `addressType: 'userDataDir'` by `ai/services/fleet/armFleetSeatWake.mjs`) still emits:
     - `set targetBundleId to id of application "<appName>"` and `tell application "<appName>" to activate` (which resolve through LaunchServices by app name and can activate or launch the unmanaged `/Applications/<App>.app` copy once seats run from admitted generation clones),
     - `if currentBundleId is not targetBundleId` fallback inside `assertTargetFrontmost` (which passes when a same-bundle sibling window holds focus), and
     - `set targetProcess to first application process whose unix id is (targetProcessId as integer)` -> `tell targetProcess`.
2. **Live-host macOS `System Events` falsifier receipt (2026-11-09):**
   - Read-only `osascript` probe against the two live `Antigravity` instances (`39747` `@neo-gemini-julia`, `43104` `@neo-gemini-maryam`) and two live `Claude` instances (`59823` `@neo-opus-ada`, `34587` `@neo-opus-grace`) on the host:
     `stored Antigravity: 39747->39747 43104->39747 | direct id: 165649904->39747 166395558->43104 | stored Claude: 59823->59823 34587->59823 | direct id: 165789202->59823 166264454->34587`
   - Storing `first application process whose unix id is 43104` into `targetProcess` serializes to a name-keyed specifier (`application process "Electron"` / `"Claude"`) whose `unix id` reads back as `39747` (the first instance), and storing `34587` reads back as `59823`. Only `application process id targetAutomationId` (`166395558` and `166264454`) preserves the exact process across `Antigravity`, `Claude`, and `Codex`.
3. **`ai/daemons/wake/daemon.mjs` (lines 2348–2488) and `ai/scripts/lifecycle/resumeHarness.mjs` (lines 509–557):**
   - Even when `instancePid` is resolved (including for `Codex`, `Antigravity`, and `Claude`), both still emit `tell application "<appName>" to activate`, `first process whose unix id is <instancePid>`, and `tell frontmostProcess`.
4. **`ai/services/fleet/armFleetSeatWake.mjs` JSDoc (lines 32–36):**
   - Describes `GUI_WAKE_DISPATCH[harnessType].appName` as *"the automation `appName` the `osascript` adapter targets"*, obscuring that Fleet seats are addressed by `userDataDir` (`deriveHarnessWakeAddress`) -> `instancePid` -> `System Events` `application process id`, while `appName` selects the harness composer/submit dialect (`Antigravity` `Cmd+I` + Space-before-Return; `Codex` `focusSeedKey: 'r'` + `Cmd+Z` + Escape-before-Return).

### Intended Solution

1. **`ai/daemons/wake/localWakeAdapters.mjs`:**
   - Generalize `codexAutomationIdLines` -> `processAutomationIdLines` and `addressedCodex` -> `addressedProcess = Boolean(instancePid)` in `buildDialogGateArgs` and `buildOsascriptArgs`.
   - Remove the dead/broken `!addressedCodex && Boolean(instancePid)` branches (`instanceFrontmostLine`, `if targetProcessId is not ""` bundle-fallback in `assertTargetFrontmost`, and `first application process whose unix id is (targetProcessId as integer)` in `resolveTargetProcessLines`).
   - Retain `appName` for harness composer/submit dialect selection (`applyHarnessMetadataDefaults`, `Codex` `focusSeedKey: 'r'` undo + `key code 53`, `Antigravity` `key code 49`) and unaddressed single-instance fallback (`instancePid === null`).
2. **`ai/daemons/wake/daemon.mjs` and `ai/scripts/lifecycle/resumeHarness.mjs`:**
   - When `Boolean(instancePid)` is true, resolve `targetProcessId` to `targetAutomationId` (`id of candidate`), foreground via `tell application "System Events" to set frontmost of application process id targetAutomationId to true`, assert `currentPid is targetProcessId` + `((unix id of application process id targetAutomationId) as string) is targetProcessId`, and target keystrokes via `tell application process id targetAutomationId` without calling `tell application "<appName>" to activate` or `id of application "<appName>"`.
3. **`ai/services/fleet/armFleetSeatWake.mjs`:**
   - Update `GUI_WAKE_DISPATCH` JSDoc to state that `appName` selects the family's composer/submit dialect and process-identity lookup while the OS wake target is the seat's `userDataDir`-resolved running process (`instancePid` -> `System Events` `application process id`).
4. **Unit tests (`test/playwright/unit/ai/daemons/wake/localWakeAdapters.spec.mjs`, `daemon.spec.mjs`):**
   - Verify that addressed `Antigravity`, `OpenCode`, `Claude`, and `Codex` routes (and PID/default-PID routes in `daemon.spec.mjs`) resolve `targetAutomationId` and never emit `tell application "<appName>" to activate`, `id of application "<appName>"`, or `first process whose unix id is`, while unaddressed single-instance (`instancePid: null`) routes retain bundle/name fallback.

### Acceptance Criteria

- [ ] AC-1: In `ai/daemons/wake/localWakeAdapters.mjs`, any `osascript` wake with a resolved `instancePid` (`Antigravity`, `Claude`, `Codex`, `OpenCode`) resolves `targetAutomationId` in both the dialog gate and delivery script, activates via `set frontmost of application process id targetAutomationId to true`, types into `tell application process id targetAutomationId`, and emits no `tell application "<appName>" to activate`, `id of application "<appName>"`, or `first (application )?process whose unix id is`.
- [ ] AC-2: In `ai/daemons/wake/daemon.mjs` and `ai/scripts/lifecycle/resumeHarness.mjs`, any `osascript` dispatch with a resolved `instancePid` uses `targetAutomationId` (`application process id targetAutomationId`) instead of `tell application "<appName>" to activate` and `first process whose unix id is`.
- [ ] AC-3: Unaddressed single-instance `osascript` routes (`instancePid === null`) continue to use bundle/name activation and `tell targetProcess`.
- [ ] AC-4: `ai/services/fleet/armFleetSeatWake.mjs` `GUI_WAKE_DISPATCH` JSDoc accurately documents that `appName` selects the composer/submit dialect and process-identity table while the OS target is the `userDataDir`-resolved running process.
- [ ] AC-5: Emitted AppleScript across addressed and unaddressed shapes compiles cleanly under `osacompile`, and Playwright unit tests in `localWakeAdapters.spec.mjs`, `daemon.spec.mjs`, and `resumeHarness.spec.mjs` pass.

### Out of Scope / Avoided Traps

- **Out of scope:** Changing `armFleetSeatWake.mjs` wire contract or `GUI_WAKE_DISPATCH` keys (`appName` remains required by `deliverOsascript` for composer keystroke dialect and `resolveGuiAppProcessIdentity`).
- **Out of scope:** Seat-Start generation pinning (`@neo-opus-ada`) and family update fetchers (`#1537` / `#1539`, `@neo-gemini-julia`).
- **Avoided trap:** Using `first application process whose unix id is <pid>` in `System Events` (empirically proven on macOS to serialize as a name-keyed specifier that aliases the second running instance of `Antigravity` or `Claude` back to the first).

### Pre-Authoring Verification Receipts

- **Epic Review Citation:** Previously reviewed by Gemini family on parent epic #1531: https://github.com/neomjs/neo-agent-brain/issues/1531#issuecomment-6748368888 (`@neo-gemini-julia`).
- **Live Open & Closed Issue Sweep (2026-11-09T19:40Z):** Checked latest 20 open issues (`#1539`, `#1537`, `#1536`, `#1535`, `#1531`, ...) and closed issues (`#1532`, `#1530`, `#1372`, `#936`); `#1372` (`#936`) scoped `targetAutomationId` only to `addressedCodex` in `localWakeAdapters.mjs`.
- **A2A & Own-Assigned Sweep:** Checked `list_messages` (`MESSAGE:cc61c96d-2251-4355-996f-399e83913cbb` from `@neo-opus-ada` delegating this leaf under `#1531`) and verified 0 open issues assigned to `@neo-gemini-maryam`.
- **Structure Map:** `npm run ai:structure-map -- --files --loc` verified existing owners in `ai/daemons/wake/`, `ai/services/fleet/`, and `ai/scripts/lifecycle/` (0 new `.mjs` files created).

Origin Session ID: 57368f18-9698-4ce8-b73a-b9517e65bd7d
Retrieval Hint: `query_raw_memories("targetAutomationId addressedCodex localWakeAdapters armFleetSeatWake unix id alias")`

🧭 **Maryam** · `@neo-gemini-maryam` · Gemini 4 Argon · Antigravity

## Timeline

- 2026-11-09T19:48:10Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-09T19:48:11Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-09T19:48:12Z @neo-gemini-maryam added the `ai` label
- 2026-11-09T19:48:12Z @neo-gemini-maryam added the `architecture` label
- 2026-11-09T19:48:12Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-09T19:48:42Z @neo-gemini-maryam added parent issue #1531
- 2026-11-09T19:51:08Z @neo-opus-ada cross-referenced by #1543
- 2026-11-09T19:56:51Z @neo-gemini-maryam cross-referenced by PR #1545
- 2026-11-09T20:26:20Z @neo-gemini-maryam referenced in commit `b2ff3d6` - "fix(wake): share processAutomationIdLines and assert frontmost on resume (#1542)"
- 2026-11-09T20:27:32Z @neo-opus-ada cross-referenced by #1531
- 2026-11-09T21:31:58Z @tobiu referenced in commit `27f46b6` - "feat(wake): target GUI seats by resolved process automation ID across all families (#1542) (#1545)

* feat(wake): target GUI seats by resolved process automation ID across all families (#1542)

* fix(wake): share processAutomationIdLines and assert frontmost on resume (#1542)"
- 2026-11-09T21:31:58Z @tobiu closed this issue

