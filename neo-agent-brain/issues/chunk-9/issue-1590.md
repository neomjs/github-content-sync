---
id: 1590
title: Antigravity seats start from the Fleet's admitted builds
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-11T01:13:40Z'
updatedAt: '2026-11-11T04:03:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1590'
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
closedAt: '2026-11-11T04:03:25Z'
---
# Antigravity seats start from the Fleet's admitted builds

## Context

The #1531 re-plan (comments 6782962688 and 6783708271) switches families on one at a time, Antigravity first. Its in-seat updater is off, the Fleet's store already holds a feed-admitted build, and #1588 runs acquisition on its own. Today `fleet.harnessGenerations.antigravity` defaults to `false`, so both running Antigravity seats launch `/Applications/Antigravity.app` and nothing updates them.

Measured on the operator's Mac, 2026-11-11:
- `~/.neo-ai/harnesses/antigravity/ledger.json` holds one admitted build: 2.31.0 (`b43c5434…`), `source` the vendor feed, admitted 2026-11-10T21:35Z.
- `/Applications/Antigravity.app` is 2.30.0, which ShipIt installed on 2026-11-10. Both running Antigravity main processes run from that path.
- Both seat profiles hold `autoCheckForUpdates: "false"` (#1539).

## The Problem

Switching the flag on as it stands would downgrade the seats. To measure it, I ran the real `FleetLifecycleService.resolveHarnessGeneration('antigravity')`, the call every Antigravity Start makes through `resolveAntigravityStartContext`. It ran with `harnessGenerationOptIns = {antigravity: true}` on Brain `origin/dev` @ `0bd80723`, against an APFS clone of that store. The live ledger's hash did not change.

| | Selected generation |
|---|---|
| before the Start | 2.31.0 (`b43c5434…`) |
| the Start | admits the installed 2.30.0 (`ce5773cb…`, 1.2 s) and returns it |
| after | 2.30.0 |

The condition: a feed-admitted build is newer than an installed bundle that the ledger has never seen. That holds on this host today, and on any host where acquisition admits before the first opted-in Start.

The cause is in two places:
- `resolveHarnessGeneration` admits the installed bundle whenever the ledger lacks its version.
- `selectHarnessGeneration` picks the most recently *admitted* build.

`selectHarnessGeneration`'s docblock covers an older bundle seen *again* (a known build never reorders the ledger), not one seen for the first time. After that, `checkAntigravityUpdate` reports 2.31.0 as `alreadyAdmitted`, so #1588's job fetches nothing. The seats stay on 2.30.0 until the vendor ships something newer than 2.31.0. The Codex and Claude opt-ins go through the same path.

## The Architectural Reality

- `ai/configBase.mjs`: `fleet.harnessGenerations.antigravity` is `leaf(false, 'NEO_FLEET_HARNESS_GENERATIONS_ANTIGRAVITY', 'boolean')`. Its JSDoc says an opted-in family "admits the installed build first when that build is new to the family's ledger".
- `ai/services/fleet/FleetLifecycleService.mjs`:
  - `resolveHarnessGeneration` checks the installed bundle by `shortVersion` + `buildVersion`, then calls `materialize` with `source: 'installed'`.
  - `selectedHarnessGeneration` takes the first entry of `harnessGenerationsOf`.
- `ai/services/fleet/harnessGeneration.mjs`: `listHarnessGenerations` reverses the ledger, and `selectHarnessGeneration` takes its head.
- Production callers, both live once the flag is on:
  - seat Start: `resolveAntigravityStartContext` → `resolveHarnessGeneration('antigravity')`;
  - the background job: `wireFleetHarnessAcquisition` (#1588, in review) → `acquireAntigravityUpdate`.
- The epic's own design, point 4: "The installed `/Applications` bundle is only the bootstrap candidate." #1543 named this step *Bootstrap*, but it admits every installed version it has not seen.
- Unit isolation: `isolateUnitEnvironment` strips every inherited `NEO_*_ROOT`, so a unit worker resolves the store root to its default, the operator's `~/.neo-ai/harnesses`. Only `FleetLifecycleService.spec.mjs` overrides the store root (`/nonexistent/…`), and its reset sets opt-ins to `null`, which means the leaf. Once the leaf is `true`, any spec that resolves an Antigravity generation without that fixture reads the real store, or admits into it.

## The Fix

1. **The installed bundle bootstraps only.** `resolveHarnessGeneration` admits it only while the family holds no admitted generation whose copy is present. From then on, the Fleet's acquisition is the family's only source. Update the leaf JSDoc and the docblocks of `resolveHarnessGeneration`, `selectedHarnessGeneration` and `selectHarnessGeneration` to say so.
2. **Opt Antigravity in:** `fleet.harnessGenerations.antigravity` defaults to `true`.
3. **Isolate tests from the store by construction** (ADR-0019 §5.4, rung (ii)): the unit run binds `NEO_FLEET_HARNESS_GENERATIONS_ROOT` to a throwaway path before the first config import. Every Antigravity Start spec states its opt-in.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `fleet.harnessGenerations.antigravity` (`NEO_FLEET_HARNESS_GENERATIONS_ANTIGRAVITY`) | `ai/configBase.mjs` leaf | default `true`: Antigravity seats start from the selected admitted generation | env `false` restores the `/Applications` launch | leaf JSDoc | AC-3, AC-5 |
| Installed-bundle admission at an opted-in Start | `FleetLifecycleService.resolveHarnessGeneration` | only while the family holds no present admitted generation | a family with none still bootstraps from the installed bundle | method JSDoc | AC-1, AC-2 |

**Decision Record impact:** aligned-with `neomjs/neo-agent-institution#adr:0034-electron-shell-architecture` (the #1531 amendment) and `adr:0019-aiconfig-reactive-provider-ssot` (a default changes; tests are isolated by construction).

## Acceptance Criteria

- [ ] AC-1: A Start whose family already holds an admitted generation never admits the installed bundle. With a newer admitted build and an older installed bundle the ledger has not seen, the Start selects the newer build and never calls `materialize`. Red on `dev`, green at head (unit).
- [ ] AC-2: A family with no present admitted generation still admits its installed bundle and starts from it (unit; the existing first-admission spec stays green).
- [ ] AC-3: `fleet.harnessGenerations.antigravity` defaults to `true`. The leaf JSDoc and the three docblocks state the bootstrap-only rule.
- [ ] AC-4: No unit worker resolves the store root to `~/.neo-ai/harnesses`. A spec reads the resolved leaf under the unit run and asserts the path.
- [ ] AC-5 `[L4-deferred — operator handoff needed]` (post-merge, operator's Mac, an FM build carrying this and #1588): a restarted Antigravity seat runs from `~/.neo-ai/harnesses/antigravity/<id>/Antigravity.app` at the store's newest build (`ps` receipt). Its lifecycle record carries that pin, and an A2A wake reaches it.
- [ ] AC-6 `[L4-deferred — operator handoff needed]` (post-merge): the next Antigravity release reaches a restarted seat with no hand install. Receipt: the ledger entry's `source` is the feed URL, the seat's pin is that build, and ShipIt's log shows no new install into `/Applications`. This is #1588's AC-5 for this family (Residual-Owner #1531).

## Out of Scope

- The Codex opt-in, whose app-server CLI readers must follow the pin first, and the Claude source and opt-in.
- Retiring superseded generations, and the cockpit.
- The Institution smoke profile binding the store root under its throwaway root. A smoke Fleet reaches the store only if it lives past the job's two-minute first run or starts an Antigravity seat. The profile's own contract binds every mutable path its tree consumes.

## Avoided Traps

- **Selecting by version** (compare dotted versions, newest wins). Rejected because it brings vendor version semantics into selection, which the epic keys on admitted identity. It also still copies every older installed bundle into the store. The defect is that the installed copy is admitted at all once the Fleet owns acquisition.
- **Opting in through the packaged FM's env** (`buildPackagedBrainEnv`). Rejected because ADR-0019 §10.8 keeps only secrets, provider and tenant choices, network placement and privileged runtime capabilities as deployment inputs, and an update opt-in is none of them. The canonical default would then misstate what the product runs.
- **Opting in before every seat's updater is off.** Epic review, Stage 5: an in-seat Squirrel updater targets the running bundle URL, which would be the read-only generation copy. Both seat profiles hold `"false"` today.

## Related

- Parent: #1531. Prerequisite for AC-6: #1588 (#1587). The flip ships with a Fleet Manager build, and plane cuts are held until `neomjs/neo-agent-institution#908` is installed.
- #1543 (seat Start pins the generation), #1532 (the store), #1539 (Antigravity suppression and acquisition).

Live latest-open + closed sweep: I checked the latest 20 open issues and ran two closed searches (`installed bundle admitted generation selected`, `Antigravity opt-in harness generation`), reading Out of Scope and Avoided Traps on #1543 and #1539, at 2026-11-11 ~01:15Z. No equivalent or prior exclusion turned up; #1543 scoped out "flipping any opt-in default". Backlog band: 159 open (red), so this is filed as a #1531 leaf. A2A claim sweep (last 30, all read-states): no claim on this scope. MC sweep (`opted-in Start admits older installed harness bundle…`): 6 results, no prior decision. Own-assignment sweep: 5 open (#1587, #1531, #571, #133, #1508), none overlapping. Structure map: `ai/services/fleet` owns the change (sibling `harnessGeneration.mjs`), and no new file is added.

Origin Session ID: 3ac6b9a1-a988-4aed-86f0-c8640799f150 (the re-plan). The probe receipts above are from its successor session, 2026-11-11.

Retrieval Hint: `query_raw_memories("opted-in Start admits older installed harness bundle over newer admitted generation")`

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-11T01:13:40Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-11T01:13:41Z @neo-opus-ada added the `enhancement` label
- 2026-11-11T01:13:41Z @neo-opus-ada added the `ai` label
- 2026-11-11T01:13:41Z @neo-opus-ada added the `agent-os` label
- 2026-11-11T01:14:59Z @neo-opus-ada cross-referenced by #1531
- 2026-11-11T01:29:43Z @neo-opus-ada cross-referenced by PR #1591
- 2026-11-11T02:09:03Z @neo-opus-ada cross-referenced by #1587
- 2026-11-11T02:11:20Z @neo-opus-ada cross-referenced by PR #1588
- 2026-11-11T02:33:32Z @neo-opus-ada referenced in commit `8c0b068` - "fix(fleet): a removed copy no longer blocks the bootstrap (#1590)

The installed bundle bootstraps whenever the family holds no generation whose copy is present, even when the ledger still lists its version; admission restores the copy under the same id. The Antigravity generation spec now runs on the shipped default, so reverting that default fails it."
- 2026-11-11T03:01:39Z @neo-opus-ada referenced in commit `d47212c` - "chore(fleet): merge dev, resolve the spec fixture with #1588 (#1590)

Keeps this branch's pinned opt-ins in the spec fixture, and states antigravity: false in #1588's acquisition-context test, which otherwise falls back to the new true default."
- 2026-11-11T04:03:25Z @tobiu referenced in commit `7f1cd69` - "feat(fleet): Antigravity seats start from admitted builds (#1590) (#1591)

* feat(fleet): Antigravity seats start from admitted builds (#1590)

An opted-in Start admits the installed bundle only while the family holds no admitted generation, so an older installed build never displaces a newer admitted one. Antigravity opts in by default, and unit runs bind the harness-generation store root beneath their own Fleet root.

* fix(fleet): a removed copy no longer blocks the bootstrap (#1590)

The installed bundle bootstraps whenever the family holds no generation whose copy is present, even when the ledger still lists its version; admission restores the copy under the same id. The Antigravity generation spec now runs on the shipped default, so reverting that default fails it."
- 2026-11-11T04:03:26Z @tobiu closed this issue

