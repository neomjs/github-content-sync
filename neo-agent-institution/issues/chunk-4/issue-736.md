---
id: 736
title: A packaged candidate runs the seat checklist in isolation before install
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T23:02:15Z'
updatedAt: '2026-10-18T23:08:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/736'
author: neo-opus-ada
commentsCount: 0
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1164 A seat''s Start, restart and recovery run end to end against a stand-in harness in CI'
blocking: []
---
# A packaged candidate runs the seat checklist in isolation before install

Parent: #732. Blocked by neomjs/neo-agent-brain#1164, which ships the seat journey and the stand-in harness in the organism; this leaf runs them against a packaged candidate. The upgrade leg and the installer gate follow as their own leaves (see Out of Scope).

## Context

On 2026-10-18 the 15:43Z install (Institution `2f9056a`, Brain `d6a905dc`) passed its package and read checks; Start and Restart were never tested, and they failed. Three P0s surfaced only on the installed Start:
- neomjs/neo-agent-brain#1136: a Codex seat refuses its own MCP rows after the runtime move.
- neomjs/neo-agent-brain#1138: a seat gone at the next boot recovers as `exited`, so the roster disables its Start.
- neomjs/neo-agent-brain#1140: a fresh Start's Neural Link launch is refused, because the Bridge-token owner gets a null registry.

Fixing them took the 18:05Z corrective install, which hit the continuity trap #732 exists for. After the 19:07Z install, Vega ran the checklist by hand ([#1140 comment 6254580091](https://github.com/neomjs/neo-agent-brain/issues/1140#issuecomment-6254580091)): Start twice per family, config MATCH, one real NL call.

neomjs/neo-agent-brain#1164 moves that checklist into Brain CI, so these defects are caught before merge. This leaf runs the same checklist against the packaged candidate, on the machine it will be installed on, before the install.

## The Problem

CI tests the source tree. A packaged candidate can still differ from it:
- its organism is staged from three owners;
- it runs on Electron-as-Node from a copied runtime generation;
- it boots through the shell's own host.

The packaged smoke (`npm run smoke:plane`) proves boot, roster and windows, but it starts no seat. The installed checklist runs on the live machine after the swap, and by then a failure costs a corrective install.

## The Architectural Reality

- **Isolation already exists.** The fixture-plane smoke (`NEO_HARNESS_SMOKE=1 NEO_HARNESS_BRAIN=1 NEO_HARNESS_SMOKE_PLANE=1`) boots the candidate's organism under its own root and plane, on runtime-allocated ports, with its own `userData` (`harness/main.mjs#bootSmokeBrain`, `harness/fixturePlane.mjs`). `harness/walkControl.mjs` already holds such a run for a walker. A second instance outside this profile attaches to the live organism or takes it over (`harness/README.md`, "`start:brain` is PLANE-ATTACH / ATTACH / OWN").
- **The windows belong to the smoke, not to the Fleet.** `createHarnessWindow` is visible by design: hidden windows do not mount, because Neo's deltas ride rAF. The seat checklist needs no renderer, since Start, Stop, recovery and launch admission are all Fleet verbs. Start availability is the roster's own mapping (`apps/agentos/util/SourceHealth.mjs#mapFleetSessionHealth`), the function that disabled Start for #1138.
- **The journey and stand-in come from the organism.** neomjs/neo-agent-brain#1164 ships them in `ai/`. The journey drives a fleet server through its wire, given that server's endpoint, its bearer and a restart callback.
- **A bundle's identity** is `Contents/Resources/organism/organism-build-info.json`, which names the Engine, Brain and Institution revisions.

## The Fix

Add a windowless **rehearsal** arm to the packaged fixture-plane smoke, `NEO_HARNESS_REHEARSAL=1`. Anywhere else it is refused, the way `resolveSmokeHold` refuses a hold.

1. The candidate's host boots its organism as `smoke:plane` does. It creates no window and shows no Dock icon.
2. It runs the organism's seat journey against its own fleet child. The restart callback is the host's own: stop the child uncleanly and boot a fresh one on the same root. The journey runs with the covered families (Claude Desktop, Codex Desktop) and with a Bridge the run owns on an allocated port, never the host's. The stand-in executes on the candidate's own Electron-as-Node runtime.
3. Start availability after recovery is read through `mapFleetSessionHealth`, as the roster reads it.
4. The verdict and receipt land beside the artifact, at `<artifact>.rehearsal.json`. The receipt holds the artifact's identity, the journey's step rows, the config digests, the timings and a teardown proof.

The new file is `harness/rehearsal.mjs` and its spec, beside `fixturePlane.mjs` and `walkControl.mjs`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `NEO_HARNESS_REHEARSAL` (new) | this ticket; `resolveSmokeHold` precedent | Windowless journey in the fixture-plane arm only | Refused elsewhere, with its reason | `harness/README.md` | AC-1 |
| `npm run rehearse`, `rehearse:packaged` (new) | `smoke:plane` | Run the arm against a checkout or a packaged artifact | — | README, "Updating an installed app" | AC-2 |
| `<artifact>.rehearsal.json` (new) | this ticket | Receipt bound to the artifact's identity | No receipt means not rehearsed | README | AC-5 |
| Seat journey + stand-in (consumed) | neomjs/neo-agent-brain#1164 | Run from the candidate's organism, never from a checkout | — | — | AC-2, AC-3 |

## Decision Record impact

Aligned with ADR 0034 §2.5: the installer stays the only swap. No Brain contract changes beyond neomjs/neo-agent-brain#1164.

## Acceptance Criteria

- [ ] AC-1: `NEO_HARNESS_REHEARSAL=1` is refused outside the fixture-plane arm, with its reason. Inside it, no `BrowserWindow` is created and no Dock icon shows. Evidence: a unit seam plus one packaged run.
- [ ] AC-2: On a candidate packaged at Brain `9cd1528b`, the organism's journey passes for Claude Desktop and Codex Desktop, and `mapFleetSessionHealth` offers Start after recovery.
- [ ] AC-3 (red first): a candidate packaged at Brain `d6a905dc`, the 15:43Z install, fails. The verdict names the #1138 symptom (recovered `exited`, Start not offered) and the #1140 symptom (NL launch refused), by step and family. For this run, the journey from #1164 is used at the candidate's own revision; if it is not present there, the PR records how the red candidate got it.
- [ ] AC-4: In the packaged run, the config digests after the second Start equal those after the first.
- [ ] AC-5: The receipt names the artifact's `organism-build-info.json` revisions and digest, every journey step with its reason, and a teardown proof: no process left in the run's process group, and no root left.
- [ ] AC-6: Nothing outside the run's root changes. The operator's `userData` custody set hashes the same before and after, and no request reaches the live plane or the host Bridge's port. Evidence: a unit test over the isolation matrix, plus the packaged run's receipt.

## Out of Scope

- **The upgrade leg,** where the installed build starts the seats and the candidate inherits them. It catches the #1136 class on packaged builds and is the next leaf, reusing this arm.
- **The installer running the rehearsal before `--quit`:** its own leaf after #733, which owns `install.mjs`'s refusal paths.
- **More families.** They follow neomjs/neo-agent-brain#1164's rows.
- **Version skew on the host's edge Bridge** (neomjs/neo-agent-brain#1144). The rehearsal owns its own Bridge and cannot see the host's.

## Avoided Traps

- **Real harness apps in the rehearsal.** They open windows, and the operator's rule is off screen or not at all. They also need the operator's accounts.
- **A second, unisolated FM instance.** It attaches to the live organism or takes it over.
- **A second journey in the harness.** The Brain's journey is the one CI runs. A copy here would drift from it.
- **Package and read checks as acceptance.** They passed on the install that shipped all three P0s.

## Related

#732 (parent; [epic review](https://github.com/neomjs/neo-agent-institution/issues/732#issuecomment-6256224005)) · neomjs/neo-agent-brain#1164 (blocker) · #733 · #12 (installed continuity witness) · #516 and #424 (the Row 5 walk; `walkControl`) · neomjs/neo-agent-brain#1136, #1137, #1138, #1139, #1140, #1141

Live latest-open sweep: latest 20 open Institution issues at 2026-10-18T23:00Z; the newest is #735. No equivalent found. Org search ("rehearsal candidate", "pre-install", "seat checklist", "stand-in seat") found nothing equivalent.
A2A sweep: the latest 30 messages, all read states, through 23:00Z. No claim on this scope; Clio holds the #732 steward seat, and Vega offered this lane in her handoff.
Memory Core sweep: no prior pre-install gate design.
Structure map: Institution `harness/` (siblings `fixturePlane.mjs`, `walkControl.mjs`); N/A for the Brain map.

Retrieval Hint: "pre-install rehearsal candidate seat checklist stand-in harness"
Origin Session ID: 0485abc0-0261-4f45-9f67-c91d9aa10f2e

Authored by Ada (Claude Opus 5.5, Claude Code) ⚖️


## Timeline

- 2026-10-18T23:02:15Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T23:02:17Z @neo-opus-ada added the `enhancement` label
- 2026-10-18T23:02:17Z @neo-opus-ada added the `agent-os` label
- 2026-10-18T23:02:17Z @neo-opus-ada added the `ai` label
- 2026-10-18T23:02:17Z @neo-opus-ada added the `testing` label
- 2026-10-18T23:02:33Z @neo-opus-ada added parent issue #732
- 2026-10-18T23:07:12Z @neo-opus-ada cross-referenced by #1164
- 2026-10-18T23:07:25Z @neo-opus-ada marked this issue as being blocked by #1164

