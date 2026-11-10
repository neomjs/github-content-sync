---
id: 1537
title: Fleet acquires Codex Desktop updates from Sparkle appcast with Ed25519 verification
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-09T19:18:45Z'
updatedAt: '2026-11-09T20:02:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1537'
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
closedAt: '2026-11-09T20:02:19Z'
---
# Fleet acquires Codex Desktop updates from Sparkle appcast with Ed25519 verification

## Context

This is the Codex acquisition leaf (AC-4) of Epic #1531 (`Fleet owns harness updates: fetched, admitted, adopted at restart`), building directly on the capability probe fix (#1530 / #1533) and the immutable generation store (#1532 / #1534).

## The Problem

`deriveHarnessLaunchSpec.mjs` launches every `codex-desktop` seat with `CODEX_SPARKLE_ENABLED=false` so running seats never rewrite their bundle in place. Because nothing outside the seats checked OpenAI's Sparkle feed, Codex Desktop sat a month behind until a manual `/Applications/ChatGPT.app` update took down the GPT lab (#1530). Fleet needs to read the Sparkle appcast from the Codex bundle's own feed metadata, verify the archive's Ed25519 signature against `SUPublicEDKey` before extraction, and admit new builds into `<root>/codex-desktop/<id>/ChatGPT.app` via `materializeHarnessGeneration`.

## The Architectural Reality

- Verified on `/Applications/ChatGPT.app` (`26.1103.60933`, build `25894`):
  - `Contents/Info.plist` carries `CFBundleIdentifier = com.openai.codex`, `CFBundleShortVersionString = 26.1103.60933`, `CFBundleVersion = 25894`, and `SUPublicEDKey = mNfr1v9t63BfgDtlw4C8lRvSY6uMggIXABDOCi3tS6k=` (a 32-byte raw Ed25519 public key encoded in base64).
  - `Contents/Resources/app.asar`'s embedded `package.json` carries `codexSparkleFeedUrl: "https://persistent.oaistatic.com/codex-app-prod/appcast.xml"` and `codexSparklePublicKey: "mNfr1v9t63BfgDtlw4C8lRvSY6uMggIXABDOCi3tS6k="`.
  - The live `appcast.xml` publishes `<item>` entries with `<sparkle:version>`, `<sparkle:shortVersionString>`, `<sparkle:minimumSystemVersion>`, `<sparkle:hardwareRequirements>arm64</sparkle:hardwareRequirements>`, a primary full-archive `<enclosure url="https://persistent.oaistatic.com/codex-app-prod/ChatGPT-darwin-arm64-<version>.zip" length="..." sparkle:edSignature="..." />`, and nested `<sparkle:deltas>` `<enclosure sparkle:deltaFrom="..." />` elements.
- `ai/services/fleet/harnessGeneration.mjs` (#1532) provides `materializeHarnessGeneration`, `selectHarnessGeneration`, and `readHarnessGenerationLedger`. Because `materializeHarnessGeneration` derives `appName = path.basename(sourceBundle)`, the extracted candidate bundle passed to admission must preserve its canonical `ChatGPT.app` directory name.

## The Fix

Add `ai/services/fleet/codexDesktopUpdate.mjs` beside `harnessGeneration.mjs` and `manageCodexDesktopRuntime.mjs`, and export `readHarnessGenerationLedger` from `ai/services/fleet/harnessGeneration.mjs`:

1. **`readHarnessGenerationLedger({root, family})` (`ai/services/fleet/harnessGeneration.mjs`)**:
   - Exports the canonical ledger reader so update acquisition reads `{version, admitted, rejected}` through the ledger owner rather than re-parsing `ledger.json`.
2. **`readCodexSparkleConfig({appBundle, fsImpl, readPlist})`**:
   - Reads `CFBundleIdentifier`, `CFBundleShortVersionString`, `CFBundleVersion`, and `SUPublicEDKey` from `Contents/Info.plist`, plus `codexSparkleFeedUrl` and `codexSparklePublicKey` from `Contents/Resources/app.asar`'s `package.json` via the ASAR header.
   - Refuses closed if `bundleId !== 'com.openai.codex'`, if `feedUrl` is not `https:`, if `SUPublicEDKey` is not a valid 32-byte base64 Ed25519 key, or if `codexSparklePublicKey` disagrees with `SUPublicEDKey`.
3. **`parseCodexSparkleAppcast(xml, {hardwareRequirements = 'arm64', hostSystemVersion} = {})`**:
   - Parses `<item>` entries from the Sparkle RSS feed, strips nested `<sparkle:deltas>` blocks before selecting the full-archive `<enclosure>` (ignoring any `sparkle:deltaFrom` enclosure), filters by `hardwareRequirements` and `minimumSystemVersion` (`<= hostSystemVersion`), validates `https:` `archiveUrl`, positive integer `length`, and 64-byte base64 `edSignature`, and returns candidates ordered newest-first by `buildVersion`.
4. **`verifySparkleEdSignature({archivePath, edSignature, publicKey, length, fsImpl})`**:
   - Verifies that the downloaded archive's byte size matches `length` (`archive-length-mismatch`), wraps the 32-byte raw `publicKey` in the RFC 8410 Ed25519 SPKI DER header (`302a300506032b6570032100`), and verifies `edSignature` over the archive bytes via `node:crypto.verify(null, ...)` (`sparkle-signature-invalid` on mismatch).
5. **`acquireCodexDesktopUpdate({root, appBundle, checkOnly, retryRejected, seams})`**:
   - Serializes per family directory, resolves the reference bundle (`selectHarnessGeneration({root, family: 'codex-desktop'})?.appBundle ?? appBundle`), reads its Sparkle config, fetches `appcast.xml`, and selects the newest compatible release.
   - If the release `(shortVersion, buildVersion)` is already admitted in `readHarnessGenerationLedger({root, family: 'codex-desktop'})`, returns `{status: 'up-to-date', admitted: false, candidate, selected}` without downloading.
   - If the release was already rejected by the current `admitterId` in this process (and `retryRejected` is `false`), returns `{status: 'already-rejected', admitted: false, candidate, selected, reason}`; when the admitting code (`admitterId`) changes or a new Fleet process boots, prior rejections expire automatically so a probe fix immediately recovers without manual intervention.
   - If `checkOnly: true`, returns `{status: 'update-available', admitted: false, candidate, selected}`.
   - Otherwise downloads `candidate.archiveUrl` into a staging directory under `<root>/codex-desktop/.acquire-<uuid>/`, verifies byte length and Ed25519 signature before extraction, extracts `ChatGPT.app` via `/usr/bin/ditto -x -k`, and hands the extracted `ChatGPT.app` to `materializeHarnessGeneration({root, family: 'codex-desktop', sourceBundle, source: candidate.archiveUrl})`, cleaning up the download staging directory in `finally`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `readHarnessGenerationLedger({root, family})` (`ai/services/fleet/harnessGeneration.mjs:164`) | `ai/services/fleet/harnessGeneration.mjs` (#1532) | Reads one family's generation ledger (`{version, admitted, rejected}`) through the canonical ledger owner | Returns empty ledger on `ENOENT`; throws `ledger-invalid` on malformed JSON | JSDoc `@summary` | unit specs |
| `readCodexSparkleConfig({appBundle, fsImpl?, readPlist?})` (new) | `ai/services/fleet/codexDesktopUpdate.mjs` | Reads `feedUrl`, `publicKey` (`SUPublicEDKey`), `shortVersion`, `buildVersion` from `Info.plist` + `app.asar` `package.json` | Throws named refusal (`sparkle-config-invalid`, `sparkle-public-key-mismatch`) | JSDoc `@summary` | unit specs + live bundle witness |
| `parseCodexSparkleAppcast(xml, options?)` (new) | `ai/services/fleet/codexDesktopUpdate.mjs` | Extracts non-delta `<item>` releases compatible with `hardwareRequirements` and `hostSystemVersion` (`minimumSystemVersion`), ordered newest-first by `buildVersion` | Throws `appcast-invalid` when malformed or no compatible enclosure exists | JSDoc `@summary` | unit specs + live appcast witness |
| `verifySparkleEdSignature({archivePath, edSignature, publicKey, length, fsImpl?})` (new) | `ai/services/fleet/codexDesktopUpdate.mjs` | Checks exact byte length and Ed25519 signature over archive bytes against 32-byte `SUPublicEDKey` | Returns `{verified: false, reason}` (`archive-length-mismatch`, `sparkle-signature-invalid`) | JSDoc `@summary` | unit specs |
| `acquireCodexDesktopUpdate({root, appBundle, checkOnly?, retryRejected?, seams?})` (new) | `ai/services/fleet/codexDesktopUpdate.mjs` | Checks feed, skips already-admitted or same-admitter rejected builds, downloads/verifies/extracts `ChatGPT.app`, and calls `materializeHarnessGeneration` | Returns `{status, admitted, candidate, selected, generation, reason}` | JSDoc `@summary` | unit specs |

## Decision Record impact

`aligned-with` `neomjs/neo-agent-institution#adr:0034-electron-shell-architecture` (via parent Epic #1531).

## Acceptance Criteria

- [ ] `readCodexSparkleConfig` extracts `feedUrl`, `publicKey`, `shortVersion`, and `buildVersion` from a Codex bundle's `Info.plist` + `app.asar` `package.json`, and refuses mismatched `SUPublicEDKey` vs `codexSparklePublicKey` or non-HTTPS feed URLs (unit + host witness on `/Applications/ChatGPT.app`).
- [ ] `parseCodexSparkleAppcast` ignores `<sparkle:deltas>` (`sparkle:deltaFrom`) enclosures, filters by `hardwareRequirements` and `minimumSystemVersion` (`<= hostSystemVersion`), and selects the newest full-archive `.zip` release by `buildVersion` (unit + live `appcast.xml` fixture).
- [ ] `verifySparkleEdSignature` verifies a real Ed25519 signature against a 32-byte base64 `SUPublicEDKey` and rejects tampered bytes, wrong keys, or length mismatches before extraction (unit).
- [ ] `acquireCodexDesktopUpdate` reads the family ledger via `readHarnessGenerationLedger`, short-circuits with `up-to-date` when the newest feed build is already admitted, short-circuits with `already-rejected` when the build was rejected by the current `admitterId` (expiring automatically when the admitting code changes), supports `checkOnly: true`, refuses tampered archives before extraction, and hands a verified extracted `ChatGPT.app` to `materializeHarnessGeneration` while cleaning its `.acquire-*` directory (unit).

## Out of Scope

- Seat-Start generation pinning and retirement (separate #1531 leaf).
- Antigravity `autoCheckForUpdates` profile suppression and `latest-arm64-mac.yml` acquisition (separate #1531 leaf).
- Claude Desktop managed policy and acquisition (separate #1531 leaf).
- Cockpit UI rendering (separate #1531 leaf).

## Avoided Traps

- **Matching `<sparkle:deltas>` enclosures:** Every `<item>` in OpenAI's `appcast.xml` embeds 4–5 `<sparkle:deltas><enclosure sparkle:deltaFrom="..." /></sparkle:deltas>` elements after the primary `.zip` enclosure. Naive `<enclosure>` regexes without stripping `<sparkle:deltas>` or rejecting `sparkle:deltaFrom` can select a binary delta instead of the full `.app` archive.
- **Unexpiring rejection skips across probe fixes:** Keying the `already-rejected` skip only on `(shortVersion, buildVersion)` in persistent storage would block a build forever even after a probe bug (like #1530) is fixed. Scoping the skip to `admitterId` (the SHA-256 fingerprint of the admitting modules) + process lifetime prevents 815 MB download loops while automatically retrying after code updates.
- **Extracting under a non-canonical bundle name:** `materializeHarnessGeneration` uses `path.basename(sourceBundle)` as `appName`; extraction must preserve `ChatGPT.app`.

## Related

- Parent Epic: #1531
- Depends on: #1530 (#1533), #1532 (#1534)

Live latest-open + closed sweep: checked latest 20 open + closed search (`Sparkle appcast SUPublicEDKey codex update generation`, Out of Scope / Avoided Traps read on #1531, #1532, #1530) at 2026-11-09T19:17Z; no equivalent or prior exclusion found. Backlog band: 172 open (red — folded as sub-issue under Epic #1531).
MC sweep: `Codex Desktop stale month behind CODEX_SPARKLE_ENABLED appcast SUPublicEDKey update`, 5 results, no prior leaf ticket found (only parent #1531 and #1530/#1532).
Own-assignment sweep: 1 open (#1230), none overlapping.
A2A claim sweep: checked last 30 messages (`status: 'all'`); #1531 split assigns Codex fetch leaf to `@neo-gemini-julia`.
Structure map: `ai/services/fleet/codexDesktopUpdate.mjs` beside `harnessGeneration.mjs` and `manageCodexDesktopRuntime.mjs`.

Origin Session ID: 132c8467-16c8-4c19-ab83-24c483477a21
Retrieval Hint: `query_raw_memories("codexDesktopUpdate Sparkle appcast SUPublicEDKey acquireCodexDesktopUpdate #1531")`


## Timeline

- 2026-11-09T19:18:46Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-09T19:18:47Z @neo-gemini-julia added the `enhancement` label
- 2026-11-09T19:18:47Z @neo-gemini-julia added the `ai` label
- 2026-11-09T19:18:47Z @neo-gemini-julia added the `architecture` label
- 2026-11-09T19:18:48Z @neo-gemini-julia added the `agent-os` label
- 2026-11-09T19:19:05Z @neo-gemini-julia added parent issue #1531
- 2026-11-09T19:23:12Z @neo-gemini-julia cross-referenced by PR #1538
- 2026-11-09T19:32:00Z @neo-gemini-julia cross-referenced by #1539
- 2026-11-09T19:47:12Z @neo-gemini-julia referenced in commit `57866e9` - "fix(fleet): scope rejection skip to admitter, share ledger reader, check minimumSystemVersion (#1537)"
- 2026-11-09T19:51:08Z @neo-opus-ada cross-referenced by #1543
- 2026-11-09T19:51:12Z @neo-gemini-julia cross-referenced by PR #1544
- 2026-11-09T20:02:19Z @tobiu referenced in commit `23b5156` - "feat(fleet): acquire Codex updates from Sparkle appcast (#1537) (#1538)

* feat(fleet): acquire Codex updates from Sparkle appcast (#1537)

* fix(fleet): scope rejection skip to admitter, share ledger reader, check minimumSystemVersion (#1537)"
- 2026-11-09T20:02:19Z @tobiu closed this issue
- 2026-11-09T20:04:23Z @neo-gemini-julia cross-referenced by #1531
- 2026-11-09T20:20:57Z @neo-opus-grace cross-referenced by PR #1545

