---
id: 736
title: A packaged candidate runs the seat checklist in isolation before install
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T23:02:15Z'
updatedAt: '2026-10-19T17:22:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/736'
author: neo-opus-ada
commentsCount: 1
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1164 A seat''s Start, restart and recovery run end to end against a stand-in harness in CI'
blocking: []
closedAt: '2026-10-19T17:22:07Z'
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

- **Isolation already exists.** The isolated Brain smoke (`NEO_HARNESS_SMOKE=1 NEO_HARNESS_BRAIN=1`) boots the candidate's organism under its own root, on runtime-allocated ports, with its own `userData` and registry (`harness/main.mjs#bootSmokeBrain`). Packaged, `buildPackagedBrainEnv` places every plane member under that root and gates the host lanes off. The fixture-plane variant (`harness/fixturePlane.mjs`, `walkControl.mjs`) adds a plane of its own. A second instance outside these profiles attaches to the live organism or takes it over (`harness/README.md`, "`start:brain` is PLANE-ATTACH / ATTACH / OWN").
- **The windows belong to the smoke, not to the Fleet.** `createHarnessWindow` is visible by design: hidden windows do not mount, because Neo's deltas ride rAF. The seat checklist needs no renderer, since Start, Stop, recovery and launch admission are all Fleet verbs. Start availability is the roster's own mapping: the `fleetRoster` rows through `RosterRow.mapRosterRow` (which applies `SourceHealth.mapFleetSessionHealth`), and the card's rule, Start for state `off`.
- **The journey and stand-in come from the organism.** neomjs/neo-agent-brain#1164 ships them in `ai/`. The journey drives a fleet server through its wire, given that server's endpoint, its bearer and a restart callback.
- **A bundle's identity** is `Contents/Resources/organism/organism-build-info.json`, which names the Engine, Brain and Institution revisions.

## The Fix

Add a windowless **rehearsal** arm to the packaged isolated Brain smoke (`NEO_HARNESS_SMOKE=1` with the Brain leg on), `NEO_HARNESS_REHEARSAL=1`. Anywhere else it is refused, the way `resolveSmokeHold` refuses a hold. The fixture-plane arm is refused too, for now: a plane-mode Fleet defines seats only after its plane registers a forge.

1. The candidate's host boots its organism as `smoke:brain` does: in its runtime generation, with `buildPackagedBrainEnv` gating the host lanes off, the NL Bridge lane included. It creates no window and shows no Dock icon.
2. It runs the organism's seat journey against its own fleet child. The restart callback is the host's own: stop the child uncleanly and boot a fresh one on the same root. The journey runs with the covered families (Claude Desktop, Codex Desktop) and with a Bridge the run owns on an allocated port, never the host's. The stand-in, a test double, runs on the host's Node (`NEO_HARNESS_STAND_IN_NODE` or `PATH`): the Fleet proves a seat's process against its shebang interpreter, and Electron runs as Node only under an environment variable a shebang cannot set. It never takes `NEO_HARNESS_NODE_BIN`, which a packaged shell points at its own Electron runtime, and every probe is bounded.
3. Right after the unclean restart, before the journey starts the seats again, the roster's Start offer is read as the cockpit reads it. That catches a recovered seat the roster still shows running. The mapping downgrades an `exited` lifecycle to `off`, so #1138's symptom is for the journey's `recover` row to name.
4. The verdict and receipt land beside the artifact, at `<artifact>.rehearsal.json`. The receipt holds the candidate's identity, the journey's step rows, the Start offers, the isolation facts, the timings and a teardown proof.
5. The packaged smoke root becomes the temp dir's real path (`resolveSmokeRoot`). The Brain's custody refuses `/var/folders`, which macOS serves through `/private/var`, so every packaged Brain smoke failed its launch admission before (measured 2026-10-19T00:50Z: `Launch admission journal refused: directory-custody`).

The new file is `harness/rehearsal.mjs` and its spec, beside `fixturePlane.mjs` and `walkControl.mjs`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `NEO_HARNESS_REHEARSAL` (new) | this ticket; `resolveSmokeHold` precedent | Windowless journey in the isolated Brain smoke only | Refused elsewhere and on the fixture plane, with its reason | `harness/README.md` | AC-1 |
| `npm run rehearse`, `rehearse:packaged` (new) | `smoke:brain` | Run the arm against a checkout or a packaged artifact | — | README, "Updating an installed app" | AC-2 |
| `<artifact>.rehearsal.json` (new) | this ticket | Receipt bound to the artifact's identity | No receipt means not rehearsed | README | AC-5 |
| Seat journey + stand-in (consumed) | neomjs/neo-agent-brain#1164 | Run from the candidate's organism, never from a checkout | — | — | AC-2, AC-3 |

## Decision Record impact

Aligned with ADR 0034 §2.5: the installer stays the only swap. No Brain contract changes beyond neomjs/neo-agent-brain#1164.

## Acceptance Criteria

- [ ] AC-1: `NEO_HARNESS_REHEARSAL=1` is refused outside the isolated Brain smoke and on the fixture plane, with its reason. Inside it, no `BrowserWindow` is created and no Dock icon shows. Evidence: a unit seam plus one packaged run.
- [ ] AC-2: On a candidate packaged at a Brain revision that ships neomjs/neo-agent-brain#1164's journey, the journey passes for Claude Desktop and Codex Desktop, and the roster offers Start for both right after the unclean restart.
- [ ] AC-3 (red first): a candidate packaged at Brain `d6a905dc`, the 15:43Z install, fails. The verdict names the #1138 symptom (the seat recovered as `exited`) and the #1140 symptom (NL launch refused), by step and family. For this run, the journey from #1164 is used at the candidate's own revision; if it is not present there, the PR records how the red candidate got it.
- [ ] AC-4: In the packaged run, the operator's content after the second Start equals that after the first (the journey's `operator-content` row). Byte-equal config digests cannot hold, because each Start writes fresh launch grants (#1164's avoided trap).
- [ ] AC-5: The receipt names the artifact's `organism-build-info.json` revisions and digest, every journey step with its reason, and a teardown proof: every child the run started stopped with its process group empty, no seat process alive, and no window. The smoke root stays, by `resolveSmokeRoot`'s design: it is kept per user, so a later run's sweep can reap a crashed run's children.
- [ ] AC-6: Nothing outside the run's root is the run's to write. Every path the profile resolves sits under the run's root (the isolation matrix, asserted before any child starts), the Fleet runs in own mode with no plane, and the seats reach only the run's Bridge port (the journey's port guard). The receipt records all three. A custody hash taken before and after is no witness while the installed app runs, because the app rewrites `brain/fleet/open-work.json` on its own (measured 2026-10-19T13:45:51Z). Evidence: the isolation-matrix unit test plus the packaged run's receipt.

Corrected at implementation (2026-10-19): AC-2's `9cd1528b` predates the journey; AC-4's byte-equal digests contradict fresh launch grants; AC-5's root removal contradicts the per-user smoke root; AC-6's custody hash moves with the live app's own writes; the stand-in runs on the host's Node.

## Out of Scope

- **The upgrade leg,** where the installed build starts the seats and the candidate inherits them. It catches the #1136 class on packaged builds and is the next leaf, reusing this arm.
- **The installer running the rehearsal before `--quit`:** its own leaf after #733, which owns `install.mjs`'s refusal paths.
- **Plane mode** (the fixture-plane arm). A plane-mode Fleet defines seats only after its plane registers a forge, so it is a later row.
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
- 2026-10-18T23:15:54Z @neo-fable-clio cross-referenced by #732
- 2026-10-18T23:33:05Z @neo-fable-clio cross-referenced by #1144
- 2026-10-18T23:41:51Z @neo-opus-ada cross-referenced by PR #1165
### @neo-opus-ada - 2026-10-19T00:46:27Z

**Intake (prescription check, Ada).** This leaf is where the 10-18 P0s get their red-first. The CI journey (neomjs/neo-agent-brain#1165) cannot reach them, measured:
- `admissionOwnsBridgeToken` (#1140's owner) and custody recovery (#1138's `exited`) both require `runtime.generation`.
- With #1139 reverted, the checkout journey stayed green.
- The packaged smoke does run generation mode. `resolveLauncherRuntime` (`harness/brain.mjs`) materializes a generation under the run's own `userData` (`brain/runtime`) and hands `runtimeGenerationArguments(generation)` to the fleet child (`bootSmokeBrain`, `launcherRuntime.launchArgs`). So AC-3 at Brain `d6a905dc` is reachable here as written.

One packaged-specific point is open: **the stand-in launcher's runtime.** The Fleet proves a seat's process by its command line: the launch binary itself, or a script behind exactly its shebang interpreter (`runsSeatLaunch`). Electron runs as Node only with `ELECTRON_RUN_AS_NODE=1`; the seat env's ambient allowlist drops that, and a shebang cannot set it without changing the interpreter line the proof compares. Plan: the rehearsal runs the stand-in on the host's Node, located the way `NEO_HARNESS_NODE_BIN` is. Without a usable Node it refuses with that reason; it never falls back. The stand-in is the test double, so the product's runtime stays untouched.

Origin Session ID: 3616f5a6-7356-44a2-b974-7415da5f3f65

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-10-19T02:57:49Z @neo-opus-vega cross-referenced by PR #1168
- 2026-10-19T02:57:51Z @neo-opus-vega cross-referenced by #1167
- 2026-10-19T03:07:08Z @neo-opus-vega cross-referenced by PR #1171
- 2026-10-19T03:13:35Z @neo-opus-vega cross-referenced by PR #1172
- 2026-10-19T03:15:20Z @neo-opus-vega cross-referenced by #1170
- 2026-10-19T03:42:46Z @neo-opus-vega cross-referenced by PR #740
- 2026-10-19T10:56:22Z @neo-opus-vega cross-referenced by #1178
- 2026-10-19T11:00:21Z @neo-opus-vega cross-referenced by PR #1179
- 2026-10-19T11:48:14Z @neo-opus-vega cross-referenced by #1183
- 2026-10-19T11:49:37Z @neo-opus-vega cross-referenced by PR #1184
- 2026-10-19T14:10:20Z @neo-opus-ada cross-referenced by #1194
- 2026-10-19T14:10:48Z @neo-opus-ada cross-referenced by PR #1195
- 2026-10-19T15:09:05Z @neo-opus-ada referenced in commit `af94e93` - "feat(harness): the rehearsal receipt records what kept the run off the operator's organism (#736)"
- 2026-10-19T15:09:05Z @neo-opus-ada referenced in commit `ec61be4` - "fix(harness): a packaged rehearsal probes only a Node it may use, and its smoke root is the temp dir's real path (#736)

The first packaged run hung: the stand-in probe took NEO_HARNESS_NODE_BIN,
which a packaged shell points at its own Electron runtime, ran it without
ELECTRON_RUN_AS_NODE, and spawnSync waited on an app that never answers.
The stand-in now has its own NEO_HARNESS_STAND_IN_NODE and every probe is
bounded. The packaged smoke root is realpath'd: the Brain's custody checks
refuse /var/folders, which macOS serves through /private/var."
- 2026-10-19T15:09:05Z @neo-opus-ada referenced in commit `62e741f` - "fix(harness): the rehearsal waits for a slow Fleet as the product does, and settles a failed boot as its verdict (#736)"
- 2026-10-19T15:09:05Z @neo-opus-ada referenced in commit `51dc042` - "fix(harness): the rehearsal's roster mapping runs the packaged runtime as Node (#736)"
- 2026-10-19T15:09:06Z @neo-opus-ada referenced in commit `de1bda2` - "fix(harness): a Brain child or script on a pinned runtime always answers as Node (#736)

A packaged shell pins NEO_HARNESS_NODE_BIN to its own Electron runtime;
started without ELECTRON_RUN_AS_NODE it is the whole app, with the shell's
run modes inherited. The first packaged rehearsal's probe did exactly that
and respawned itself until a peer killed the chain. runBrainScript and
startBrainChild now add the flag after every caller's env whenever the
runtime is pinned, so no caller can start the app by omission; a checkout's
plain node is unchanged. The README asks seats sharing a machine to give
each packaged run its own NEO_HARNESS_BRAIN_ROOT."
- 2026-10-19T15:18:01Z @neo-opus-ada referenced in commit `3a1d9b6` - "fix(harness): every rehearsal starts from an empty Fleet, so a reused root cannot replay its seats' configs (#736)

A rerun on the same root appended the journey's operator table to the Codex seat's
config.toml a second time, and Codex refused the redefined table. The rehearsal now
removes the Fleet dir the isolation matrix certified before it seeds anything."
- 2026-10-19T15:37:27Z @neo-opus-ada cross-referenced by PR #1193
- 2026-10-19T15:54:38Z @neo-opus-ada cross-referenced by PR #750
- 2026-10-19T16:27:20Z @neo-opus-ada referenced in commit `9dc5cdf` - "docs(harness): the rehearsal's signing-key format reads PKCS8, which the archaeology gate does not take for a ticket (#736)"
- 2026-10-19T16:31:22Z @neo-opus-ada cross-referenced by #752
- 2026-10-19T17:22:07Z @tobiu referenced in commit `32d3e34` - "feat(harness): a pre-install rehearsal runs the seat journey against the packaged candidate (#736) (#750)

* feat(harness): the pre-install rehearsal's gate, stand-in runtime, candidate identity and receipt (#736)

* feat(harness): the rehearsal runs in the isolated Brain smoke, not yet on the fixture plane (#736)

* feat(harness): the rehearsal runs the organism's seat journey windowless against the smoke's own Fleet (#736)

NEO_HARNESS_REHEARSAL=1 in the isolated Brain smoke boots the organism with
the stand-in as each desktop family's harness, the rehearsal's own viewer
and a Bridge on an allocated port, then runs Brain's seatJourney against the
smoke's Fleet. The unclean restart is the shell's: it kills the fleet child
and boots a fresh one on the same root, and reads the roster's Start offer
the cockpit's way right after. The receipt lands beside the candidate and
the exit code is the verdict. No window, no Dock icon, no renderer setup.

* feat(harness): the rehearsal receipt records what kept the run off the operator's organism (#736)

* fix(harness): a packaged rehearsal probes only a Node it may use, and its smoke root is the temp dir's real path (#736)

The first packaged run hung: the stand-in probe took NEO_HARNESS_NODE_BIN,
which a packaged shell points at its own Electron runtime, ran it without
ELECTRON_RUN_AS_NODE, and spawnSync waited on an app that never answers.
The stand-in now has its own NEO_HARNESS_STAND_IN_NODE and every probe is
bounded. The packaged smoke root is realpath'd: the Brain's custody checks
refuse /var/folders, which macOS serves through /private/var.

* fix(harness): the rehearsal waits for a slow Fleet as the product does, and settles a failed boot as its verdict (#736)

* fix(harness): the rehearsal's roster mapping runs the packaged runtime as Node (#736)

* fix(harness): a Brain child or script on a pinned runtime always answers as Node (#736)

A packaged shell pins NEO_HARNESS_NODE_BIN to its own Electron runtime;
started without ELECTRON_RUN_AS_NODE it is the whole app, with the shell's
run modes inherited. The first packaged rehearsal's probe did exactly that
and respawned itself until a peer killed the chain. runBrainScript and
startBrainChild now add the flag after every caller's env whenever the
runtime is pinned, so no caller can start the app by omission; a checkout's
plain node is unchanged. The README asks seats sharing a machine to give
each packaged run its own NEO_HARNESS_BRAIN_ROOT.

* fix(harness): every rehearsal starts from an empty Fleet, so a reused root cannot replay its seats' configs (#736)

A rerun on the same root appended the journey's operator table to the Codex seat's
config.toml a second time, and Codex refused the redefined table. The rehearsal now
removes the Fleet dir the isolation matrix certified before it seeds anything.

* docs(harness): the rehearsal's signing-key format reads PKCS8, which the archaeology gate does not take for a ticket (#736)"
- 2026-10-19T17:22:07Z @tobiu closed this issue
- 2026-10-19T18:01:36Z @neo-opus-ada cross-referenced by PR #1212
- 2026-10-19T18:44:51Z @tobiu referenced in commit `764c085` - "fix(harness): every smoke settles its Fleet's Bridge on its own port, and stops the Bridge its Fleet started (#752) (#753)

* feat(harness): the pre-install rehearsal's gate, stand-in runtime, candidate identity and receipt (#736)

* feat(harness): the rehearsal runs in the isolated Brain smoke, not yet on the fixture plane (#736)

* feat(harness): the rehearsal runs the organism's seat journey windowless against the smoke's own Fleet (#736)

NEO_HARNESS_REHEARSAL=1 in the isolated Brain smoke boots the organism with
the stand-in as each desktop family's harness, the rehearsal's own viewer
and a Bridge on an allocated port, then runs Brain's seatJourney against the
smoke's Fleet. The unclean restart is the shell's: it kills the fleet child
and boots a fresh one on the same root, and reads the roster's Start offer
the cockpit's way right after. The receipt lands beside the candidate and
the exit code is the verdict. No window, no Dock icon, no renderer setup.

* feat(harness): the rehearsal receipt records what kept the run off the operator's organism (#736)

* fix(harness): a packaged rehearsal probes only a Node it may use, and its smoke root is the temp dir's real path (#736)

The first packaged run hung: the stand-in probe took NEO_HARNESS_NODE_BIN,
which a packaged shell points at its own Electron runtime, ran it without
ELECTRON_RUN_AS_NODE, and spawnSync waited on an app that never answers.
The stand-in now has its own NEO_HARNESS_STAND_IN_NODE and every probe is
bounded. The packaged smoke root is realpath'd: the Brain's custody checks
refuse /var/folders, which macOS serves through /private/var.

* fix(harness): the rehearsal waits for a slow Fleet as the product does, and settles a failed boot as its verdict (#736)

* fix(harness): the rehearsal's roster mapping runs the packaged runtime as Node (#736)

* fix(harness): a Brain child or script on a pinned runtime always answers as Node (#736)

A packaged shell pins NEO_HARNESS_NODE_BIN to its own Electron runtime;
started without ELECTRON_RUN_AS_NODE it is the whole app, with the shell's
run modes inherited. The first packaged rehearsal's probe did exactly that
and respawned itself until a peer killed the chain. runBrainScript and
startBrainChild now add the flag after every caller's env whenever the
runtime is pinned, so no caller can start the app by omission; a checkout's
plain node is unchanged. The README asks seats sharing a machine to give
each packaged run its own NEO_HARNESS_BRAIN_ROOT.

* fix(harness): every rehearsal starts from an empty Fleet, so a reused root cannot replay its seats' configs (#736)

A rerun on the same root appended the journey's operator table to the Codex seat's
config.toml a second time, and Codex refused the redefined table. The rehearsal now
removes the Fleet dir the isolation matrix certified before it seeds anything.

* docs(harness): the rehearsal's signing-key format reads PKCS8, which the archaeology gate does not take for a ticket (#736)

* fix(harness): every smoke settles its Fleet's Bridge on its own port, and stops the Bridge its Fleet started (#752)

A Fleet that finds no Bridge on its NL port now starts one, detached. The plain smokes
set no NEO_NL_PORT, so their Fleet settled on the operator's 8081, and the detached
Bridge outlived every smoke's group teardown while the receipt read clean. Each smoke
now allocates its Bridge port; teardown stops the run-bridge.mjs on it and proves the
port released, and the run-state records the port for a crashed run's next sweep. The
rehearsal starts a Bridge of its own only when its Fleet has not."
- 2026-10-19T19:32:13Z @neo-opus-vega cross-referenced by #758

