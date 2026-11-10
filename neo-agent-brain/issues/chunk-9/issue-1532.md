---
id: 1532
title: Fleet admits vendor harness builds into immutable generations
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-09T18:36:40Z'
updatedAt: '2026-11-09T19:12:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1532'
author: neo-opus-ada
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
closedAt: '2026-11-09T19:12:47Z'
---
# Fleet admits vendor harness builds into immutable generations

## Context

This is the first leaf of #1531: the generation engine that every family's update path feeds. This leaf adds no seat behaviour. The seat-Start wiring and per-family fetching are later leaves.

## The Problem

Every seat of a family launches the one mutable bundle at `fleet.harnessBinaries.*`. When a new build fails its family probe, there is no other copy to start: on 2026-11-09 all GPT seats were refused with `updater-disable-predicate-missing` (#1530). Fleet also has no record of which vendor build a seat runs.

## The Architectural Reality

- `FleetLifecycleService` resolves a seat's binary as `this.harnessBinaryPaths?.[harnessType] || AiConfig.fleet.harnessBinaries[leafKey]` (`ai/services/fleet/FleetLifecycleService.mjs`, the method documented as reading the `fleet.harnessBinaries.*` leaf).
- The family probes already exist:
  - `probeCodexDesktopCapabilities` (`manageCodexDesktopRuntime.mjs`);
  - `probeAntigravityRuntime` (`probeAntigravityRuntime.mjs`);
  - Claude Desktop has only its `--version` probe (`deriveHarnessLaunchSpec.mjs`).
- Precedent to mirror, not to fork: `runtimeGeneration.mjs` (identity, format and read-only verifier) and ADR 0034 §2.5 items 1, 2 and 5. Its layout is `<userData>/brain/runtime/<id>/`, which is content-addressed and read-only.
- Measured 2026-11-09 against `ChatGPT.app` 26.1103.60933:
  - `cp -c -R` cloned the 1.9 GB bundle in 0.63 s using about 3 MB of free space.
  - `codesign --verify --deep --strict` passes at the clone path, and `spctl -a -t exec` reports `Notarized Developer ID`.
  - The live probe run against the clone reproduces `updater-disable-predicate-missing`. A family probe therefore works on a generation path.

## The Fix

A new module, `ai/services/fleet/harnessGeneration.mjs`, sits beside `runtimeGeneration.mjs`.

1. **Identity:** SHA-256 of the UTF-8 JSON array `[bundleId, shortVersion, buildVersion, cdhash]`. The values come from `Info.plist` and `codesign -dvvv`; location and time are excluded.
2. **Materialise:** copy the candidate bundle into a staging directory under the store root. Use an APFS clone (`cp -c`) when on the same volume, else a plain copy. Then verify, run the family probe against the staged copy, and rename atomically into `<root>/<family>/<id>/<App>.app`.
   - Re-admitting a known build is a no-op.
   - A published copy whose seal no longer verifies is refused as `existing-generation-invalid` and never replaced. The cdhash in the identity seals the contents, so one id cannot name two payloads that both verify.
   - An interrupted run leaves nothing selectable.
3. **Verify:** each check refuses with its own named reason.
   - `codesign --verify --deep --strict`;
   - the TeamIdentifier equals the family's vendor (`claude-desktop` → `Q6L2SF6YDW`, `codex-desktop` → `2DC432GLL2`, `antigravity` → `EQHXZ8M8AV`);
   - `spctl -a -t exec` accepts the bundle;
   - the family probe answers `available`. A family with no probe yet (`claude-desktop`) is refused as `family-probe-unavailable`, never admitted on its signature alone.
4. **Ledger:** one owner-only record per family, holding admitted builds and the last 20 rejections. Selection is the most recently admitted new build whose copy is present. Re-admitting a known build never reorders the ledger, so seeing the untouched `/Applications` bundle again cannot displace a newer admitted build. A rejection never changes the selection.
5. **Store root:** the caller passes it as an absolute path (ADR 0019 §5.4 rung iii: a pure operation's scalar). Its AiConfig leaf (`planeMember: false`, kept out of plane backups) lands with its first reader, the seat-Start leaf, so no unread leaf ships here.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `materializeHarnessGeneration({root, family, sourceBundle, source?, seams?})` (new) | this module | resolves `{admitted, id, appBundle, binaryPath, reason}` | named refusal recorded in the ledger; selection unchanged | JSDoc `@summary` | unit specs |
| `selectHarnessGeneration({root, family})` (new) | the family ledger | returns the selected generation or `null` | `null`: the caller keeps today's `harnessBinaries` path | JSDoc | unit specs |
| `HARNESS_GENERATION_FAMILIES`, `harnessGenerationId` (new) | this module | vendor bundle id + Team ID per family; the identity hash | none | JSDoc | unit specs |

## Acceptance Criteria

- [ ] Identity is stable across locations and changes when the version, build or cdhash changes (unit).
- [ ] A wrong Team ID, an invalid signature, a Gatekeeper rejection and a probe failure each refuse with their named reason, and the previous selection is unchanged (unit, with injected seams).
- [ ] Re-admitting a known build is a no-op that never displaces a newer selection. A published copy whose seal no longer verifies is refused as `existing-generation-invalid` and left in place (unit).
- [ ] A run interrupted after staging, or after the rename but before the ledger write, leaves nothing selectable, and the next run recovers (unit).
- [ ] Host witness from a scratch run with the default host tools, not committed: the installed Codex build is rejected as `probe-refused:updater-disable-predicate-missing`, and the installed Antigravity build is admitted and selected. The receipt is in the PR.

## Out of Scope

- Seat Start pinning, adoption and retirement (the next #1531 leaf).
- Fetching from vendor feeds (the per-family leaves).
- The cockpit surface.
- Any change to the `harnessBinaries` semantics.

## Avoided Traps

- A version allowlist. #1030 already removed that shape for Antigravity.
- Writing into `/Applications`.
- Importing code from the bundle under test, the same rule as `runtimeGeneration.mjs`: bundle bytes are data only.

## Related

Parent #1531 · #1530 · #980 · neomjs/neo-agent-institution#674

Origin Session ID: 1ef4ba01-d8ab-4b0e-bb7c-1ee8426dc5fe

Retrieval Hint: `query_raw_memories("harness generation materialize admit Team ID cdhash identity")`

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-11-09T18:36:40Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-09T18:36:42Z @neo-opus-ada added the `enhancement` label
- 2026-11-09T18:36:42Z @neo-opus-ada added the `ai` label
- 2026-11-09T18:36:43Z @neo-opus-ada added the `architecture` label
- 2026-11-09T18:36:43Z @neo-opus-ada added the `agent-os` label
- 2026-11-09T18:36:45Z @neo-opus-ada added parent issue #1531
- 2026-11-09T18:43:32Z @neo-gemini-julia cross-referenced by #1531
- 2026-11-09T18:46:08Z @neo-opus-ada cross-referenced by PR #1534
- 2026-11-09T18:52:59Z @neo-opus-ada cross-referenced by PR #1533
- 2026-11-09T19:12:47Z @tobiu referenced in commit `2d0f7a6` - "feat(fleet): admit harness builds into immutable generations (#1532) (#1534)

Adds ai/services/fleet/harnessGeneration.mjs, the vendor-bundle sibling of runtimeGeneration.mjs: a candidate .app is copied (APFS clone, ditto fallback) into <root>/<family>/<id>/ only after its bundle id, Team ID, deep codesign, Gatekeeper assessment and family capability probe pass on the copy. Identity is sha256 of [bundleId, shortVersion, buildVersion, cdhash]; a per-family ledger records admissions and the last 20 rejections; selection is the most recently admitted new build. No seat behaviour changes yet."
- 2026-11-09T19:12:47Z @tobiu closed this issue
- 2026-11-09T19:18:46Z @neo-gemini-julia cross-referenced by #1537
- 2026-11-09T19:32:00Z @neo-gemini-julia cross-referenced by #1539
- 2026-11-09T19:36:27Z @neo-opus-ada cross-referenced by PR #1538
- 2026-11-09T19:51:08Z @neo-opus-ada cross-referenced by #1543

