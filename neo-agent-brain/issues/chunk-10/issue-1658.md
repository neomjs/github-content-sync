---
id: 1658
title: The Fleet keeps Claude Desktop seats on the newest admitted build
state: OPEN
labels:
  - enhancement
  - ai
  - security
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-13T01:56:39Z'
updatedAt: '2026-11-13T02:12:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1658'
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
blocking:
  - '[ ] 1660 Antigravity and Codex acquire through the shared archive pipeline'
---
# The Fleet keeps Claude Desktop seats on the newest admitted build

## Context

The operator, 2026-11-13: *"FM was supposed to handle harness updates for claude, codex and antigravity. since updates arrive multiple times a week, we need a permanent solution to handle it."* On 2026-11-11 he had already ruled out the alternative: an update path that means "there will never be updates" is a failure. #1584 turned Claude's own updater off by policy and left two things to this leaf (its Out of Scope): acquiring new builds, and switching `fleet.harnessGenerations.claudeDesktop` on.

Measured on the operator's Mac, 2026-11-13 ~01:50Z:
- The policy holds (`disableAutoUpdates = true`). All 63 Claude boots since its install logged `[updater] Auto-updates disabled by enterprise policy` (receipt on #1584).
- Claude has shipped twice since: 5.1231.2 and 5.11492.0. Homebrew's cask record moved at 2026-11-11T03:46Z and 2026-11-12T15:48Z.
- All four Claude seats run `/Applications/Claude.app` 5.1231.0. The store holds no `claude-desktop` generation, and the acquisition job skips the family as `not-opted-in`.

Every Claude seat is frozen two releases behind, the state #1531 exists to prevent.

## The Problem

Two pieces are missing, and each is useless alone:

1. **No source.** `HARNESS_ACQUIRERS` (`ai/services/fleet/wireFleetHarnessAcquisition.mjs`) has entries for `antigravity` and `codex-desktop` only. Claude publishes no feed an unattended client can read; #1531 comment 6781813120 measured each candidate. The app's Squirrel feed needs the installation's device id. The fleet download link answers `403` with a bot challenge. The versioned zip `https://downloads.claude.ai/releases/darwin/universal/<version>/Claude-<commit>.zip` serves `200`, but nothing first-party announces its `<commit>` (the build's Sentry release id). Homebrew's cask record (`https://formulae.brew.sh/api/cask/claude.json`) names that URL and its SHA-256 within hours of a release.
2. **No owner switch.** `claudeDesktop` defaults to `false`, and the packaged FM sets no override (Institution `harness/brain.mjs#buildPackagedBrainEnv`). As the leaf works today, an opted-in Start on a host without the policy refuses (#1584 AC-4), so a default of `true` would stop every Claude seat whose operator has not installed the profile.

## The Architectural Reality

- `materializeHarnessGeneration` (`ai/services/fleet/harnessGeneration.mjs`) already admits Claude: Team ID `Q6L2SF6YDW`, `codesign --verify --deep --strict`, Gatekeeper, and `probeClaudeDesktopRuntime`. A source only has to hand it a bundle.
- The siblings are `antigravityUpdate.mjs` (an `app-update.yml` manifest, size and SHA-512 checked before extraction) and `codexDesktopUpdate.mjs` (a Sparkle appcast with Ed25519). Both serialize runs per family, skip a version the same admitter already rejected, and extract into `<root>/<family>/.update-staging-<uuid>`, which they always remove.
- `readHarnessHostGate({family: 'claude-desktop'})` reads the managed plist, and `FleetLifecycleService#assertHarnessHostGate` refuses an opted-in Start without it. The gate protects immutability: on macOS a packaged Claude copy updates itself wherever it sits, so a generation copy run with the updater on would rewrite the store.
- `harnessAcquisitionContext` and `resolveHarnessGeneration` read one opt-in (`harnessGenerationsOptedIn`), so the job and a Start agree.

## The Fix

1. **Source.** `ai/services/fleet/claudeDesktopUpdate.mjs` names the candidate: `checkClaudeDesktopUpdate` reads Homebrew's cask record as an **untrusted pointer**, and `acquireClaudeDesktopUpdate` downloads only when all of these hold:
   - the URL is `https://downloads.claude.ai/releases/darwin/universal/<version>/Claude-<hex>.zip`, and `<version>` equals the record's version (the part before its comma);
   - the version is newer than the selected generation, or than the installed bundle while none is selected;
   - the body stays under a size cap, and its SHA-256 equals the record's before anything is extracted.

   The download half (`ditto -x -k`, exactly one `.app`, `materializeHarnessGeneration({family: 'claude-desktop', source: <zip URL>})`) lives once, in `ai/services/fleet/acquireHarnessArchive.mjs`, instead of a third copy of what `antigravityUpdate.mjs` and `codexDesktopUpdate.mjs` each carry; #1660 moves those two onto it. A stale, missing or malformed record fetches nothing, and seats keep the last admitted build.
2. **Wiring.** `HARNESS_ACQUIRERS['claude-desktop']`.
3. **Owner.** `fleet.harnessGenerations.claudeDesktop` defaults to `true`, and the host gate decides which updater owns the family instead of refusing a Start. With the policy, seats start from the selected generation and the Fleet acquires. Without it, seats start from the installed bundle, whose own updater keeps it current, and the job records why it skipped the family. One updater owns Claude on any host, and neither path freezes. This amends #1584 AC-4, and the leaf's JSDoc says so.

## Acceptance Criteria

- [ ] A record naming a newer version with a conforming URL and a matching SHA-256 is admitted, with `source` set to the zip URL (unit, with fetch, extract and materialize seams).
- [ ] Each of these refuses by name and admits nothing: a URL outside `downloads.claude.ai/releases/darwin/universal/`, a record version that disagrees with the URL's, a missing or non-hex digest, a digest mismatch, a body over the cap, and zero or two `.app` bundles (unit).
- [ ] A version that is not newer than the selected generation, or is already admitted, downloads nothing; a version the same admitter rejected is skipped until the admitter changes (unit).
- [ ] `wireFleetHarnessAcquisition` runs the Claude acquirer on a host the Fleet owns, and records the skip with the host gate's reason otherwise (unit).
- [ ] `claudeDesktop` defaults to `true`. With the policy, a Start launches the selected generation; without it, a Start launches the installed bundle and writes no ledger row (unit, the second red on `dev`).
- [ ] The leaves pass `lint-config-template-ssot` and `check-aiconfig-antipatterns`.
- [ ] **Post-merge** `[L4-deferred — operator handoff needed]` (an FM build carrying this): the store admits the cask's newest Claude, 5.11492.0 or later, with `source` set to its zip URL. A restarted Claude seat runs from `~/.neo-ai/harnesses/claude-desktop/<id>/Claude.app` (`ps`), its boot logs `[updater] Auto-updates disabled by enterprise policy`, and its lifecycle record carries the pin.

## Out of Scope

- Codex, which gets its own leaf: its app-server CLI readers must follow the seat's pin before its opt-in.
- The cockpit surface (#1531 point 7) and retiring old generations.
- Generating or installing the policy profile.

## Avoided Traps

- **Trusting the pointer.** The record is third-party. Admission proves the bundle (Team ID, codesign, Gatekeeper, probe); the URL and digest checks only decide whether to download.
- **Anthropic's own feeds.** The Squirrel feed needs a device id the Fleet does not hold, and the fleet download link sits behind a bot challenge that an automated client must not pass.
- **A refusing default.** Default `true` with #1584's refusal would stop every Claude seat on a host without the profile.
- **Checking the policy during admission** (#1584's trap): the policy is a host fact, while the ledger records bundles.
- **A hand install.** A path nobody takes means "never" (the operator, 2026-11-11).

## Related

#1531 (parent) · #1584 (admission and policy; its Out of Scope) · #1543 (the Start pin) · #1587 (the acquisition job) · #1590 (Antigravity's default-on owner, the precedent) · neomjs/neo-agent-institution#682

Decision Record impact: aligned-with `adr:0019-aiconfig-reactive-provider-ssot` (one leaf's default changes; the host gate is a host fact read at the use site, never mirrored into config) and with `neomjs/neo-agent-institution#adr:0034-electron-shell-architecture` §2.5 (immutable generations; a seat moves to a new build only at its next Start).

Not a `good first issue`: it is owned, and it needs a host with Claude Desktop and the policy profile.

Live latest-open + closed sweep: latest 20 open plus closed searches (`claude desktop update`, `harness acquisition claude`; Out of Scope and Avoided Traps read on #1584) at 2026-11-13T01:56Z; no equivalent or prior exclusion found. Backlog band: 154 open (red), so the leaf folds under #1531. A2A claim sweep: latest 30 messages, all read states; no claim on Claude harness updates. MC sweep: "Claude seats never update after the updater policy turned off; harness frozen on an old version; no source for new Claude builds", 6 results; no prior decision on a Claude source (earlier updates were manual app restarts, Oct 13–20). Own-assignment sweep: #1531 (the parent), #571, #133 and #1508; none overlapping. Structure map: `--files --loc` run; `ai/services/fleet` holds the siblings `antigravityUpdate.mjs` and `codexDesktopUpdate.mjs`.

Origin Session ID: 776d12fd-202a-486b-88c0-41605d050556

Retrieval Hint: "Claude Desktop seats frozen behind the updater policy; Homebrew cask record as an untrusted pointer; the host gate decides which updater owns the family"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-13T01:56:39Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-13T01:56:41Z @neo-opus-ada added the `enhancement` label
- 2026-11-13T01:56:41Z @neo-opus-ada added the `ai` label
- 2026-11-13T01:56:42Z @neo-opus-ada added the `security` label
- 2026-11-13T01:56:42Z @neo-opus-ada added the `agent-os` label
- 2026-11-13T01:56:44Z @neo-opus-ada added parent issue #1531
- 2026-11-13T01:56:49Z @neo-opus-ada cross-referenced by #1584
- 2026-11-13T02:12:25Z @neo-opus-ada cross-referenced by #1660
- 2026-11-13T02:12:32Z @neo-opus-ada marked this issue as blocking #1660
- 2026-11-13T02:14:13Z @neo-opus-ada cross-referenced by PR #1661
- 2026-11-13T02:16:01Z @neo-opus-ada cross-referenced by #1662
- 2026-11-13T02:30:11Z @neo-opus-ada cross-referenced by #1665
- 2026-11-13T02:44:06Z @neo-opus-ada cross-referenced by #1531
- 2026-11-13T05:20:11Z @neo-opus-ada referenced in commit `a9d4c50` - "fix(fleet): a build must carry the version its source named (#1658)

Admission refuses a copy whose signed short version differs from the one its source advertised (expectedShortVersion), before Gatekeeper, the probe and publication, so the selection and the admission-ordered store stay as they were. acquireHarnessArchive passes the source's version; the refusal is memoized, so the zip is not fetched again. Addresses Euclid's RA-1: a record advertising 3.0.0 over a 1.0.0 bundle no longer displaces a selected 2.0.0.

The vendor's current archive (5.11492.2) carries CFBundleShortVersionString 5.11492.2, read from its zip by range request, so exact equality holds for real records."

