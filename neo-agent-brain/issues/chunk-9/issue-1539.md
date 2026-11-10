---
id: 1539
title: Fleet suppresses Antigravity seat auto-updates and acquires updates from app-update.yml
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-09T19:31:59Z'
updatedAt: '2026-11-09T21:31:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1539'
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
closedAt: '2026-11-09T21:31:00Z'
---
# Fleet suppresses Antigravity seat auto-updates and acquires updates from app-update.yml

## Context

Epic #1531 (`Fleet owns harness updates: fetched, admitted, adopted at restart`, AC-5) establishes that Antigravity seat profiles suppress in-seat vendor auto-updates and that Fleet acquires Antigravity updates from the `electron-updater` feed named in `Contents/Resources/app-update.yml` into immutable harness generations (#1532 / #1534).

Currently:
1. Antigravity's `StorageManager` (`Contents/Resources/app.asar` -> `out/main/storage/storageManager.js` and `out/main/settings/settingsService.js`) reads `<instanceHome>/app_storage.json` as a flat JSON map of string values and defaults `SettingKey.AUTO_CHECK_FOR_UPDATES` (`"autoCheckForUpdates"`) to `"true"`. On startup, `initAutoUpdater` evaluates `items['autoCheckForUpdates'] === 'true'` and starts `electron-updater` + Squirrel `ShipIt`, which stages updates into `~/Library/Caches/com.google.antigravity.ShipIt` and rewrites the running bundle on quit (`ShipIt_stderr.log` recorded an unmanaged swap on `2026-11-09`).
2. `prepareAntigravityArtifacts` in `ai/services/fleet/prepareManagedAgentWorkspace.mjs` (`lines 1683-1727`) converges `.neo-fleet-mcp/*.sh` invokers into `<instanceHome>`, but does not converge `"autoCheckForUpdates": "false"` into `<instanceHome>/app_storage.json`.
3. Fleet has no Antigravity update-acquisition module to read `Contents/Resources/app-update.yml` (`url: https://antigravity-hub-auto-updater-974169037036.us-central1.run.app/manifest/`), fetch `latest-arm64-mac.yml`, verify the `Antigravity.zip` `sha512` (base64) and byte `size`, extract `Antigravity.app`, and admit it via `materializeHarnessGeneration({root, family: 'antigravity', sourceBundle, source})`.

## The Problem

1. **In-seat Squirrel auto-updater remains enabled per seat profile:** Without `"autoCheckForUpdates": "false"` in `<instanceHome>/app_storage.json`, every managed Antigravity seat polls the update feed in-process and spawns `ShipIt`, which either mutates `/Applications/Antigravity.app` or fails against an immutable generation clone while relaunching an unmanaged default-profile window. Furthermore, existing Antigravity seat profiles already contain `"autoCheckForUpdates": "true"` seeded by Antigravity on first launch alongside user/profile settings (`hubSurfaceColor`, `wcv-*` flags), so convergence must set `"autoCheckForUpdates": "false"` while preserving all unrelated keys in `app_storage.json`.
2. **Version-dependent Cloud Run manifest response on `latest-arm64-mac.yml`:** Empirical probe of `https://antigravity-hub-auto-updater-974169037036.us-central1.run.app/manifest/latest-arm64-mac.yml` (`vary: x-user-staging-id, x-app-version`) shows:
   - Requesting `latest-mac.yml` returns `HTTP/2 404`; the endpoint is arch-qualified (`latest-${arch}-mac.yml`, i.e. `latest-arm64-mac.yml` on Apple Silicon).
   - Sending `x-app-version` equal to the latest version (`2.29.1`) returns a 16-byte stub (`version: 2.29.1`) with no `files:`, `sha512`, or `size` fields.
   - Sending `x-app-version: 0.0.0` always returns the complete `electron-updater` YAML manifest (`version`, `files[0].url`, `files[0].sha512`, `files[0].size`, `path`, `sha512`).
3. **Unverified archive extraction risk:** Extracting `Antigravity.zip` without first verifying its byte `size` and base64 `sha512` digest against `latest-arm64-mac.yml` would expose staging to truncated or tampered downloads before `materializeHarnessGeneration` runs `codesign`, `spctl`, and `probeAntigravityRuntime`.

## The Architectural Reality

- `ai/services/fleet/prepareManagedAgentWorkspace.mjs` (`lines 1683-1727`): `prepareAntigravityArtifacts` owns `<instanceHome>` profile preparation for `antigravity` seats.
- `ai/services/fleet/harnessGeneration.mjs` (`lines 33-44, 151-256`): `materializeHarnessGeneration` and `listHarnessGenerations` manage the `antigravity` family (`bundleId: 'com.google.antigravity'`, `teamId: 'EQHXZ8M8AV'`, `FAMILY_PROBES['antigravity'] -> probeAntigravityRuntime`).
- `ai/services/fleet/codexDesktopUpdate.mjs` (#1537 / PR #1538): Establishes the sibling per-family update acquisition pattern (`read*UpdateConfig`, `parse*Manifest`, `verify*Archive`, `check*Update`, `acquire*Update`).

## The Fix

1. **Converge `"autoCheckForUpdates": "false"` in `<instanceHome>/app_storage.json` (`ai/services/fleet/prepareManagedAgentWorkspace.mjs`):**
   - Add `convergeAntigravityAppStorage({instanceHome, fileSystem})` and invoke it inside `prepareAntigravityArtifacts` (`ai/services/fleet/prepareManagedAgentWorkspace.mjs:1683-1727`).
   - Verify no symlink segments along `<instanceHome>/app_storage.json`, create `{"autoCheckForUpdates": "false"}\n` (`mode: 0o600`) when absent, return `MATCH` when `parsed.autoCheckForUpdates === 'false'`, update `parsed.autoCheckForUpdates = 'false'` atomically while preserving all other keys when `parsed.autoCheckForUpdates` is absent or `'true'`, and throw `FLEET_WORKSPACE_DIVERGENT` if `app_storage.json` is malformed JSON, not a plain object, or holds a non-boolean-string `autoCheckForUpdates` value.
2. **Create `ai/services/fleet/antigravityUpdate.mjs`:**
   - `readAntigravityUpdateConfig(appBundle, {fsImpl, architecture})`: Reads `Contents/Resources/app-update.yml` and `Contents/Info.plist` (`CFBundleIdentifier`, `CFBundleShortVersionString`, `CFBundleVersion`), validates `com.google.antigravity` and `https://` feed base URL, and derives `manifestUrl` (`<baseUrl>/latest-${architecture}-mac.yml`).
   - `parseAntigravityUpdateManifest(yamlText, {manifestUrl})`: Parses `electron-updater` YAML (`version`, `files` entry + top-level `path`/`sha512`/`size`), validates `https://` archive URL, base64 `sha512` (64 decoded bytes / 512 bits), and positive integer `size`.
   - `verifyAntigravityArchive(archiveBuffer, {expectedSha512, expectedSize})`: Verifies archive byte length and SHA-512 digest (`crypto.createHash('sha512').update(archiveBuffer).digest('base64')`) using constant-time byte comparison (`crypto.timingSafeEqual`).
   - `checkAntigravityUpdate({root, appBundle, fetchImpl, ...})`: Reads bundle update config, fetches `manifestUrl` with `x-app-version: 0.0.0`, compares candidate `version` against admitted generations in `listHarnessGenerations({root, family: 'antigravity'})` and the installed bundle version, and reports `{updateAvailable, alreadyAdmitted, currentVersion, candidate, feedUrl}`.
   - `acquireAntigravityUpdate({root, appBundle, fetchImpl, extractArchive, materializeGeneration, ...})`: Short-circuits with `{status: 'up-to-date'}` when no newer build is needed, otherwise downloads `Antigravity.zip`, verifies `size` and `sha512`, extracts `Antigravity.app` via `/usr/bin/ditto -x -k` in a temporary directory inside `root`, and calls `materializeHarnessGeneration({root, family: 'antigravity', sourceBundle, source: {kind: 'feed', feedUrl, archiveUrl, sha512}})`, cleaning up staging in `finally`.
3. **Add unit tests:**
   - Add `test/playwright/unit/ai/services/fleet/antigravityUpdate.spec.mjs` covering `app-update.yml` parsing, `latest-arm64-mac.yml` parsing, SHA-512 + size verification, tampered archive refusal, up-to-date short-circuit, and end-to-end extraction + `materializeHarnessGeneration` admission.
   - Add unit test coverage in `test/playwright/unit/ai/services/fleet/generateAntigravityMcpFiles.spec.mjs` (or `prepareManagedAgentWorkspace.spec.mjs`) verifying `prepareAntigravityArtifacts` creates `<instanceHome>/app_storage.json` with `"autoCheckForUpdates": "false"`, preserves unrelated keys when flipping `"true"` -> `"false"`, and reports `MATCH` when already `"false"`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `prepareAntigravityArtifacts` (`ai/services/fleet/prepareManagedAgentWorkspace.mjs:1683-1727`) | Epic #1531 AC-5 | Converges `"autoCheckForUpdates": "false"` into `<instanceHome>/app_storage.json` while preserving unrelated profile keys; refuses malformed JSON or non-boolean-string `autoCheckForUpdates` with `FLEET_WORKSPACE_DIVERGENT` | Refuses seat preparation on malformed `app_storage.json` | JSDoc on `prepareAntigravityArtifacts` and `convergeAntigravityAppStorage` | L2 unit test |
| `readAntigravityUpdateConfig` / `parseAntigravityUpdateManifest` / `verifyAntigravityArchive` (`ai/services/fleet/antigravityUpdate.mjs`) | Epic #1531 AC-5 | Reads `app-update.yml`, parses `latest-arm64-mac.yml`, and verifies SHA-512 base64 digest + byte length before extraction | Throws error with deterministic `.code` (`update-config-invalid`, `manifest-empty`, `sha512-mismatch`, `archive-size-mismatch`) | Module & function JSDoc | L2 unit test + live host witness |
| `checkAntigravityUpdate` / `acquireAntigravityUpdate` (`ai/services/fleet/antigravityUpdate.mjs`) | Epic #1531 AC-5 + `ai/services/fleet/harnessGeneration.mjs:151-256` | Fetches `latest-arm64-mac.yml` with `x-app-version: 0.0.0`, short-circuits when version is already admitted, or downloads + SHA-512-verifies + extracts + admits via `materializeHarnessGeneration` | Cleans up temporary staging directory and leaves prior generation selected on failure | Module & function JSDoc | L2 unit test (`antigravityUpdate.spec.mjs`) |

## Decision Record impact

`amends neomjs/neo-agent-institution#adr:0034-electron-shell-architecture` (implements the Antigravity update-suppression and feed-acquisition slice of Epic #1531).

## Acceptance Criteria

- [ ] `prepareAntigravityArtifacts` (`ai/services/fleet/prepareManagedAgentWorkspace.mjs`) converges `"autoCheckForUpdates": "false"` into `<instanceHome>/app_storage.json`, preserving existing keys when updating from `"true"` or missing, returning `MATCH` when already `"false"`, and throwing `FLEET_WORKSPACE_DIVERGENT` on malformed JSON.
- [ ] `readAntigravityUpdateConfig` reads `Contents/Resources/app-update.yml` and `Contents/Info.plist` from an Antigravity bundle and derives the `https://` `latest-${arch}-mac.yml` manifest URL.
- [ ] `parseAntigravityUpdateManifest` parses `electron-updater` YAML (`version`, `files[0].url`, `files[0].sha512`, `files[0].size`) and rejects non-HTTPS URLs or malformed SHA-512 base64 digests.
- [ ] `verifyAntigravityArchive` checks both byte length and SHA-512 digest before any archive extraction occurs, throwing `archive-size-mismatch` or `sha512-mismatch` on mismatch.
- [ ] `acquireAntigravityUpdate` short-circuits with `{status: 'up-to-date'}` when the feed version is already admitted in `listHarnessGenerations`, and otherwise verifies, extracts, and hands `Antigravity.app` to `materializeHarnessGeneration({root, family: 'antigravity', ...})` while always removing its temporary staging directory.
- [ ] Unit tests pass in `test/playwright/unit/ai/services/fleet/antigravityUpdate.spec.mjs` and workspace preparation tests.

## Out of Scope

- Seat-Start generation pinning in `FleetLifecycleService.mjs` and `armFleetSeatWake.mjs` process-targeted wakes (separate leaves under #1531).
- Cockpit UI for harness generations (separate leaf under #1531).

## Avoided Traps

- **Using whole-file receipt convergence on `<instanceHome>/app_storage.json`:** Antigravity's `StorageManager` writes runtime UI state (`hubSurfaceColor`, `wcv-*`) and seeds `"autoCheckForUpdates": "true"` on first launch into `app_storage.json`. Whole-file receipt convergence would fail every existing seat start with `FLEET_WORKSPACE_DIVERGENT`; key-scoped convergence of `autoCheckForUpdates` preserves runtime state while enforcing update suppression.
- **Sending the installed `x-app-version` when fetching `latest-arm64-mac.yml`:** The Cloud Run updater omits `files:`, `sha512`, and `size` when `x-app-version` matches the latest release; sending `x-app-version: 0.0.0` guarantees the full manifest payload.

## Related

- Parent epic: #1531 (`Fleet owns harness updates: fetched, admitted, adopted at restart`)
- Foundation: #1532 / PR #1534 (`ai/services/fleet/harnessGeneration.mjs`)
- Sibling leaf: #1537 / PR #1538 (`ai/services/fleet/codexDesktopUpdate.mjs`)

Live latest-open + closed sweep: checked latest 30 open + closed search (`antigravity update autoCheckForUpdates app-update`, Out of Scope / Avoided Traps read on #1531, #1532, #1537) at 2026-11-09T19:30Z; no equivalent or prior exclusion found. Backlog band: 171 open (red; filed as required leaf under active epic #1531). A2A claim sweep: checked latest 30 messages (`status: 'all'`); split agreed with `@neo-opus-ada` in `MESSAGE:3db54d1c-1399-421e-bbee-dde1718b0d32`. Memory Core sweep: `query_raw_memories("antigravity autoCheckForUpdates app_storage app-update latest-arm64-mac")`. Own-assignment sweep: checked `#1537` and `#1230`; no overlap. Structure map: `ai/services/fleet/antigravityUpdate.mjs` + `ai/services/fleet/prepareManagedAgentWorkspace.mjs`.

Origin Session ID: 132c8467-16c8-4c19-ab83-24c483477a21

Retrieval Hint: `query_raw_memories("antigravityUpdate app-update.yml latest-arm64-mac.yml autoCheckForUpdates app_storage.json")`

## Timeline

- 2026-11-09T19:31:59Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-09T19:32:01Z @neo-gemini-julia added the `enhancement` label
- 2026-11-09T19:32:01Z @neo-gemini-julia added the `ai` label
- 2026-11-09T19:32:01Z @neo-gemini-julia added the `architecture` label
- 2026-11-09T19:32:01Z @neo-gemini-julia added the `agent-os` label
- 2026-11-09T19:32:22Z @neo-gemini-julia added parent issue #1531
- 2026-11-09T19:51:08Z @neo-opus-ada cross-referenced by #1543
- 2026-11-09T19:51:12Z @neo-gemini-julia cross-referenced by PR #1544
- 2026-11-09T20:01:41Z @neo-gemini-julia referenced in commit `46d9988` - "fix(fleet): inject copyBundle seam in Linux unit tests and align fetch signature (#1539)"
- 2026-11-09T20:20:24Z @neo-gemini-julia referenced in commit `d899ece` - "feat(fleet): suppress Antigravity seat auto-updates and acquire updates from app-update.yml (#1539)"
- 2026-11-09T20:20:24Z @neo-gemini-julia referenced in commit `c653520` - "fix(fleet): inject copyBundle seam in Linux unit tests and align fetch signature (#1539)"
- 2026-11-09T20:20:25Z @neo-gemini-julia referenced in commit `3afca48` - "fix(fleet): stream Antigravity archive with SHA-512 and byte-length limiter (#1539)"
- 2026-11-09T20:20:57Z @neo-opus-grace cross-referenced by PR #1545
- 2026-11-09T20:27:32Z @neo-opus-ada cross-referenced by #1531
- 2026-11-09T21:31:00Z @tobiu referenced in commit `8f06a59` - "feat(fleet): suppress Antigravity seat auto-updates and acquire updates from app-update.yml (#1539) (#1544)

* feat(fleet): suppress Antigravity seat auto-updates and acquire updates from app-update.yml (#1539)

* fix(fleet): inject copyBundle seam in Linux unit tests and align fetch signature (#1539)

* fix(fleet): stream Antigravity archive with SHA-512 and byte-length limiter (#1539)"
- 2026-11-09T21:31:00Z @tobiu closed this issue

