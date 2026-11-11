---
id: 912
title: The installed Fleet switches on open-work wakes
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T01:54:57Z'
updatedAt: '2026-11-11T02:11:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/912'
author: neo-opus-grace
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
closedAt: '2026-11-11T02:11:16Z'
---
# The installed Fleet switches on open-work wakes

## Context

On 2026-11-09 the operator asked that a seat hear when its PR's CI finishes, and with what outcome. The Brain's open-work producer and its holder-change wakes do that (neomjs/neo-agent-brain#759). They sit behind `fleet.openWorkWakes`, declared in Brain `ai/configBase.mjs:488` as `leaf(false, 'NEO_FLEET_OPEN_WORK_WAKES', 'boolean')`, so they are off by default.

AC1 of neomjs/neo-agent-brain#1535 is now certified on the installed state: `wakeGateOf` holds over a 24 h window, and every quota bucket stays inside its share (neomjs/neo-agent-brain#759, comment 6784623704). AC2 asks for `fleet.openWorkWakes` to be on in the installed Fleet, and nothing in the harness sets it.

## The Problem

The Fleet server reads the leaf when it boots (Brain `ai/services/fleet/devFleetServer.mjs:739-747`): `AiConfig.fleet.openWorkWakes && planeClient ? wireFleetOpenWorkWakes(…)`. The packaged harness passes no such variable. So the installed producer observes and certifies, and wakes nobody.

## The Architectural Reality

- **`bootProductBrain`** (`harness/main.mjs:1095`) builds `packagedEnv` (`:1150-1157`) only when `app.isPackaged`, and spawns the Fleet server with it (`:1241`).
- **The other boots:** `bootSmokeBrain` (`:1503`) and `bootUiFleetTransport` (`:1419`) build their own environments.
- **The rule for config values (Brain ADR 0019, §10.7 and §10.8):** the deployment profile declares a value and the leaf resolves it. The harness serializes the value into the child's environment and never re-derives the leaf.
- **Test precedent:** `test/playwright/unit/harness/pack.spec.mjs:330` already pins the Fleet spawn environment by reading the source.

## The Fix

1. **`harness/main.mjs`:** the `packagedEnv` of `bootProductBrain` gains `NEO_FLEET_OPEN_WORK_WAKES: 'true'`, beside `ELECTRON_RUN_AS_NODE`. It is the packaged product's own fragment, so the smoke and unpackaged boots never see it.
2. **`pack.spec.mjs`:** a new arm pins the variable in that fragment and its absence from `bootSmokeBrain` and `bootUiFleetTransport`. Each absence check has a must-contain control on the same function body.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| The environment of `bootProductBrain`'s packaged Fleet child | Brain leaf `fleet.openWorkWakes`; Brain ADR 0019 §10.7 | carries `NEO_FLEET_OPEN_WORK_WAKES=true` | the unpackaged and smoke boots set nothing, and the leaf's default `false` holds | this ticket; neomjs/neo-agent-brain#759 | the `pack.spec.mjs` arm |

## Decision Record impact

aligned-with neomjs/neo-agent-brain#adr:0019-aiconfig-reactive-provider-ssot (§10.7, §10.8)

## Acceptance Criteria

- [ ] `bootProductBrain`'s packaged environment carries `NEO_FLEET_OPEN_WORK_WAKES: 'true'`, and the smoke and UI-transport boots carry none. A unit arm in `pack.spec.mjs` checks both.
- [ ] `npm run test-unit` is green.
- [ ] `[L4-deferred — operator handoff needed]` After the install, the Fleet log reports the open-work wake round starting (`devFleetServer.mjs:747`). The first live wakes are receipted on neomjs/neo-agent-brain#759, as AC3 and AC4 of neomjs/neo-agent-brain#1535 ask.

## Out of Scope

- **A runtime off switch in the cockpit.** Until one exists, a reinstall without this line turns the wakes off. Quota certification can refuse the first activation; once activated, the wakes latch (Brain `wireFleetOpenWorkWakes.mjs:296-311`), so a later quota overflow does not switch them off.
- **The rollup that still reads red after a re-run passes** (the open defect-note on neomjs/neo-agent-brain#759).
- **The unpackaged (dev) and smoke boots.**

## Avoided Traps

- **Flipping the leaf's default in the Brain.** Every Fleet server would then wake seats, a peer's dev Fleet included, and seats would get each wake twice.
- **Gating the switch on the wake receiver (`wakeReceiverEnv`).** Open-work wakes travel through the plane's `add_message` sender (`wireFleetOpenWorkWakes.mjs:248`). The receiver is not their transport.

## Related

neomjs/neo-agent-brain#759 (epic; residual owner of these receipts) · neomjs/neo-agent-brain#1535 (AC2) · neomjs/neo-agent-brain#1541 · neomjs/neo-agent-brain#1546

**Ownership:** Grace (self-assigned; the residual owner on neomjs/neo-agent-brain#759).

**Sweeps:**
- **Live latest-open and closed sweep:** the latest 20 open issues (#911 to #657) and a closed search (open-work wakes, openWorkWakes, packagedEnv, bootProductBrain env) at 2026-11-11T01:55Z. #211's Out of Scope and Avoided Traps were read: it governs where plane-attach config comes from, not a declared product switch. No equivalent found. 53 issues are open, the red band; the owning epic is Brain #759, so this is filed as its leaf.
- **A2A in-flight claim sweep:** the latest 30 rows in all read states, with no claim on this scope.
- **Memory Core sweep** ("open-work wakes switched off in the installed Fleet"): no prior decision.
- **Own-assignment sweep:** #635, #633, #490, #414; none on this surface.
- **Structure map:** N/A. This is one key in an existing fragment; no placement is decided.

Origin Session ID: e1e2f65b-eba3-4251-a947-d4a1b58b9e06

Retrieval Hint: "installed Fleet open-work wakes switch NEO_FLEET_OPEN_WORK_WAKES packaged env bootProductBrain"

🖖 Grace (Claude Opus 5.5, Claude Code) · session e1e2f65b-eba3-4251-a947-d4a1b58b9e06

## Timeline

- 2026-11-11T01:54:57Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-11T01:54:59Z @neo-opus-grace added the `enhancement` label
- 2026-11-11T01:54:59Z @neo-opus-grace added the `agent-os` label
- 2026-11-11T01:55:00Z @neo-opus-grace added the `ai` label
- 2026-11-11T01:56:57Z @neo-opus-grace cross-referenced by PR #913
- 2026-11-11T02:11:17Z @tobiu closed this issue
- 2026-11-11T02:11:47Z @tobiu referenced in commit `6f00d3d` - "feat(fleet): the installed Fleet switches on open-work wakes (#912) (#913)"

