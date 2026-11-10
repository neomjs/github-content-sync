---
id: 1584
title: Fleet admits Claude Desktop builds with Claude's updater off by policy
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-10T23:09:55Z'
updatedAt: '2026-11-10T23:09:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1584'
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
---
# Fleet admits Claude Desktop builds with Claude's updater off by policy

Sub of #1531, the Claude leg: its points 2 and 5 for `claude-desktop`, plus the operator's 2026-11-10 requirement that no update path launches the default Claude app. It also delivers neomjs/neo-agent-institution#682's outcome: no Claude update stops a running seat.

## Context

- **Claude still updates itself under the seats.** ShipIt installed 5.1231.0 into `/Applications/Claude.app` at 2026-11-10 22:31:50Z, as soon as a restart left no Claude instance running. At 22:31:51Z it launched the unmanaged default-profile app itself. The operator's verdict: "That makes no sense, since we only use seats."
- **5.1231.0 shows every seat a modal "Fix required for auto-updates" dialog**, because the bundle is `root:wheel`; see #1531 comment 6781224453.
- **The seats' updater is still on.** Every seat boot since logs `[updater] App is installed, enabling initial check and auto-updates` in `~/Library/Logs/Claude/main.log` (last at 22:51:29Z). `/Library/Managed Preferences/` does not exist on the host, so the policy profile staged for the operator is not installed yet.
- **Claude cannot opt in today.** `FAMILY_PROBES` in `ai/services/fleet/harnessGeneration.mjs` has no `claude-desktop` entry. Admission therefore rejects every Claude candidate with `family-probe-unavailable`, and an opted-in `fleet.harnessGenerations.claudeDesktop` refuses every Start.

## The Problem

Epic point 5: no seat may run its vendor's updater. That updater rewrites the bundle all seats share, and it relaunches the default app. Claude Desktop has one off-switch, the enterprise policy `disableAutoUpdates` in the domain `com.anthropic.claudefordesktop`. The Help Center documents it for fleets whose versions an MDM manages, and Fleet is that manager here. Today nothing generates the profile, nothing checks that it is installed, and no probe lets a Claude build into the generation store.

## The Architectural Reality

Measured in 5.1231.0's `app.asar` on 2026-11-10:
- The app reads its policy from `/Library/Managed Preferences/com.anthropic.claudefordesktop.plist` (machine) and `/Library/Managed Preferences/<user>/com.anthropic.claudefordesktop.plist` (user).
- With `autoUpdate.disabled` set, startup returns before the first update check and logs `[updater] Auto-updates disabled by enterprise policy`. Each later check returns with `[updater] Skipping update check: auto-updates disabled by enterprise policy`. The self-restart-to-install path after an unclean exit returns early too, and the ownership check behind the dialog runs only when the policy is off.
- Seat isolation rests on Chromium's `--user-data-dir` switch, which `deriveHarnessLaunchSpec.mjs` passes. The same build contains the warning `CLAUDE_USER_DATA_DIR is set and signed builds ignore it`, so the probe pins the switch, not the environment variable.
- Precedent: `probeCodexDesktopCapabilities` scans `app.asar` and the framework binary for named markers and fails closed (#1530). It sits beside `probeAntigravityRuntime` in `FAMILY_PROBES`.
- At Start, `FleetLifecycleService.resolveHarnessGeneration` admits the installed build, and `selectedHarnessGeneration` refuses with the family's latest rejection.

## The Fix

1. **Probe:** `ai/services/fleet/probeClaudeDesktopRuntime.mjs`, registered as `FAMILY_PROBES['claude-desktop']`. Like the Codex probe it is static, and it requires these markers:
   - in `app.asar`: the policy path `/Library/Managed Preferences/com.anthropic.claudefordesktop.plist`, the key `disableAutoUpdates`, and the gate's log literal `Auto-updates disabled by enterprise policy`;
   - in `Electron Framework`: `user-data-dir`.

   A missing marker refuses with its name, as `app-contract-marker-missing:<marker>` or `chromium-contract-marker-missing:user-data-dir`.
2. **Policy:** `ai/services/fleet/claudeDesktopUpdatePolicy.mjs`.
   - `readClaudeDesktopUpdatePolicy()` reads the machine-level managed plist, the one a System-scope profile writes.
   - `claudeDesktopUpdatePolicyProfile()` returns that profile: `com.apple.ManagedClient.preferences`, then `com.anthropic.claudefordesktop`, Forced, `disableAutoUpdates: true`. It keeps the identifiers `com.neomjs.fleet.claude-desktop-updates` and `com.neomjs.fleet.claude-desktop-updates.preferences` of the profile staged on 2026-11-10, so an install made before this lands stays the Fleet's profile.
3. **CLI:** `ai/scripts/fleet/claudeDesktopUpdatePolicy.mjs` writes the profile to a path the operator names, and reports whether the policy is in force.
4. **Start refusal:** an opted-in `claude-desktop` Start refuses while the policy is not in force and names the CLI. It runs no admission and writes no ledger row.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FAMILY_PROBES['claude-desktop']` | #1531 point 2 | Admits a Claude build that carries every marker | Rejection naming the marker; the selected generation stays | JSDoc | Unit fixtures + live admission of 5.1231.0 |
| `readClaudeDesktopUpdatePolicy()` | Help Center "Enterprise Configuration"; the read path above | `{inForce, path}` | A missing or unreadable plist means not in force | JSDoc | Unit test via an injected reader; live read after install |
| `claudeDesktopUpdatePolicyProfile()` | #1531 point 5 | Deterministic profile XML | — | JSDoc | `plutil -lint`; payload test |
| `ai/scripts/fleet/claudeDesktopUpdatePolicy.mjs` | — | Write the profile; report the policy | — | `--help` | Live run |
| `claude-desktop` Start refusal | #1531 point 5: refuse Claude's opt-in while the policy is absent | Start throws with the reason | — | JSDoc | Unit tests |

## Decision Record impact

`aligned-with neomjs/neo-agent-institution#adr:0034-electron-shell-architecture`, through #1531's amendment of §2.5. ADR-0019: the change adds no AiConfig leaf. The new modules take no config, and the refusal reads the opt-in where `harnessGenerationsOptedIn` already reads it. Tests use the `harnessGenerationOptIns` instance override and never write the shared config.

## Acceptance Criteria

- [ ] The probe admits the installed 5.1231.0: `materializeHarnessGeneration` run against `/Applications/Claude.app` into a scratch root records an admitted generation with `teamId: Q6L2SF6YDW` (receipt in the PR).
- [ ] For each marker, a fixture bundle without it is rejected with that marker named, and the family's selected generation stays unchanged.
- [ ] The generated profile passes `plutil -lint` and carries System scope, the Forced `disableAutoUpdates: true` payload, and the two identifiers above.
- [ ] An opted-in `claude-desktop` Start without the policy refuses, names the CLI, spawns nothing and writes no ledger row. With the policy in force, it starts from the selected generation.
- [ ] **Post-merge, after the operator installs the profile:** the machine-level managed plist holds `disableAutoUpdates = 1`. Every seat boot logs `[updater] Auto-updates disabled by enterprise policy`, none logs `enabling initial check and auto-updates`, and no seat shows the ownership dialog.
- [ ] **Post-merge, over the next Claude release:** `ShipIt_stderr.log` records no install, and no Claude process runs without a seat's `--user-data-dir`.

## Out of Scope

- Acquiring new Claude builds (#1531 point 4). The candidates are measured in #1531 comment 6781813120, and the source sub follows this one. Until then, a new build enters only as the installed bootstrap candidate.
- Switching `fleet.harnessGenerations.claudeDesktop` on.
- The cockpit surface (#1531 point 7).
- User-scope profiles.
- `CLAUDE_USER_DATA_DIR` in the launch spec, harmless while it equals the switch.

## Avoided Traps

- **"Fix ownership"** (the bundle goes to `root:admin`) keeps the vendor updater working. The policy is the fix, and its gate runs before the dialog.
- **Checking the policy during admission:** the policy is a host fact, while the ledger records bundles. A host without the policy would write rejections that outlive the install.
- **`CLAUDE_E2E_MANAGED_PLIST` per seat** is a vendor test hook (rejected in #1531).
- **A version allowlist** (#1030).

## Related

#1531 (parent) · #1532 (generation engine) · #1543 (Start pin) · #1530 (Codex probe precedent) · neomjs/neo-agent-institution#682 (the outcome) · neomjs/neo-agent-institution#732

Not `good first issue`: it is owned, and it needs a host with Claude Desktop and an operator-installed profile.

Live latest-open + closed sweep: checked the latest 20 open and searched closed issues (`claude desktop update`, `disableAutoUpdates OR probe claude-desktop`; Out of Scope read on #1532, #1530, #1543) plus open issues in Brain and Institution (`claude update OR ShipIt OR stealth update`) at 2026-11-10T23:10Z. No equivalent found; #1531 lists this leaf as unfiled. Backlog band: 162 open (red), so the leaf folds under #1531. A2A claim sweep: only my own `[lane-intent]` of 22:44Z. Own-assignment sweep: #1531, #571, #133 and #1508 do not cover it. Memory Core sweep: no prior art beyond the #1531 thread. Structure map: `ai/services/fleet` (siblings `probeAntigravityRuntime.mjs`, `harnessGeneration.mjs`); CLI siblings in `ai/scripts/fleet`.

Origin Session ID: 3ac6b9a1-a988-4aed-86f0-c8640799f150

Retrieval Hint: `query_raw_memories("Claude Desktop disableAutoUpdates profile probe claude-desktop admission ShipIt default app")`

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-10T23:09:55Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-10T23:09:56Z @neo-opus-ada added the `enhancement` label
- 2026-11-10T23:09:57Z @neo-opus-ada added the `ai` label
- 2026-11-10T23:09:57Z @neo-opus-ada added the `architecture` label
- 2026-11-10T23:09:57Z @neo-opus-ada added the `agent-os` label
- 2026-11-10T23:10:00Z @neo-opus-ada added parent issue #1531

