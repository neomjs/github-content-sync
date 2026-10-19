---
id: 752
title: A smoke's Fleet can start a Neural Link Bridge that outlives the run
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-19T16:31:21Z'
updatedAt: '2026-10-19T18:44:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/752'
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
blockedBy: []
blocking: []
closedAt: '2026-10-19T18:44:50Z'
---
# A smoke's Fleet can start a Neural Link Bridge that outlives the run

## Context

neomjs/neo-agent-brain#1193 (merged `883278aa`, 2026-10-19 16:34Z) makes Fleet boot settle the Neural Link Bridge. When no Bridge answers on `neuralLinkConfig.port`, the Fleet spawns a keyed `run-bridge.mjs` itself: `detached: true`, a declared interim until neomjs/neo-agent-brain#1203 gives it an owner. `settleFleetBridge` defaults `startIfAbsent` to `true`, and `devFleetServer` calls it with the leaf port only.

The Institution smokes boot that same Fleet, and none of them is ready for it:
- **Plain smokes, packaged and checkout:** neither profile sets `NEO_NL_PORT`; only the rehearsal does (`harness/rehearsal.mjs` `rehearsalFleetEnv`). Their Fleet settles on the leaf default `8081`, the operator's port. When nothing holds it, the Fleet starts a Bridge there, keyed to the run's throwaway registry.
- **The rehearsal:** it allocates its own port, but starts its run-owned Bridge only after `await brainBootPromise` (`harness/main.mjs`, the rehearsal arm). By then the Fleet has already started a detached Bridge on that port. The run's own Bridge cannot bind, and the journey talks to the Fleet's.
- **Teardown:** `stopBrainChild` and `sweepStaleRunState` kill by process group (`killFn(-pgid, …)`, `harness/brain.mjs`). A `detached` child leads its own group, so the Fleet's Bridge outlives every run. The receipt's `teardown.childrenStopped` counts only the children the shell started, so it still reads clean.

Measured so far: the sources above, at the named heads. Inference, not yet run: the leaked process and the 8081 bind. AC-1 is the red-first run.

## The Problem

Every candidate packaged from Brain `dev` now carries #1193, and three things go wrong:
1. Every plain smoke on a machine with a free `8081` leaves a foreign-keyed Bridge on the operator's port. The next real FM boot reads it as `bridge-unkeyed`, and every seat's Neural Link proof is refused.
2. Every smoke and rehearsal leaks a Bridge process.
3. The rehearsal's teardown proof (#736 AC-5) stops being true while still reading clean.

The pre-install rehearsal runs on an outside operator's machine before the first install, which is the worst place for the first case.

Design authority: `bootSmokeBrain`'s own contract ("shifted only in COORDINATES: allocated Chroma/fleet ports and a throwaway data root, so a dev box's live Brain is never touched"), restated in Brain ADR 0019 §10.7's packaged own-mode row ("The packaged smoke … shifts only coordinates: allocated ports, and a throwaway root"). Also #736 AC-5 (teardown proof) and AC-6 (the seats reach only the run's Bridge port).

## The Architectural Reality

- An outside operator's first install has no Bridge to adopt, so its Fleet takes the **start** branch. That branch is #1193's new code, and it is the one a pre-install rehearsal most needs to witness.
- The rehearsal's unclean restart SIGKILLs the Fleet, and the detached Bridge survives it. The rebooted Fleet then **adopts** that Bridge. On a #1193 candidate one rehearsal witnesses both branches, plus the Bridge's continuity across an FM crash.
- The run allocates its ports, so a listener on the run's Bridge port during the run is the run's own. Its command line names `run-bridge.mjs`.
- `teardownBrain` (`harness/main.mjs`) is every smoke's exit path, and `writeRunState` / `sweepStaleRunState` (`harness/brain.mjs`) already carry a crashed run's record to the next run's sweep.

Prescription checked: `harness/main.mjs` `bootSmokeBrain` + `teardownBrain`, and `harness/brain.mjs` run-state — they own the concern: the run's coordinates and its teardown. This replaces the first prescription, a run-owned Bridge started before the Fleet so the Fleet adopts it, which would hide the start branch.

## The Fix

1. **The port:** `bootSmokeBrain` allocates `NEO_NL_PORT` for every smoke beside the Chroma and Fleet ports. The rehearsal's own allocation still wins through `extraEnv`. A smoke's Fleet then never settles on `8081`.
2. **The run's Bridge:** after a smoke's Fleet boots, the run's Bridge is whatever listens on its port. The rehearsal starts its own only when nothing answers there, which is the pre-#1193 case its AC-3 red runs still need.
3. **Teardown:** after the children stop, `teardownBrain` stops a `run-bridge.mjs` listening on the run's port and proves the port released. The run-state records the port, so a crashed run's Bridge is stopped by the next run's sweep.
4. **The receipt:** it records which Bridge the run used (`fleet-started` or `run-owned`) and `teardown.bridgePortReleased`; a port still held fails the verdict.

## Contract Ledger

| Target surface | Source of authority | Behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Smoke profile `NEO_NL_PORT` (`bootSmokeBrain`) | `bootSmokeBrain` JSDoc; ADR 0019 §10.7 | allocated per run, beside the Chroma and Fleet ports; `extraEnv` may override | none | `harness/README.md` (smoke section) | AC-2 |
| Run-state record (`writeRunState`) | `sweepStaleRunState` JSDoc | adds the run's Bridge port; the sweep stops a `run-bridge.mjs` still listening on it | a record without a port sweeps as today | the functions' JSDoc | AC-2 |
| `<artifact>.rehearsal.json` | #736 Contract Ledger row | adds the run's Bridge kind and `teardown.bridgePortReleased` | — | README | AC-1, AC-3 |

Decision Record impact: aligned-with ADR 0019 §10.7. The fix keeps the packaged-smoke sentence true under #1193.

## Acceptance Criteria

- [ ] AC-1 (red first): on a candidate carrying neomjs/neo-agent-brain#1193, a packaged smoke and a packaged rehearsal leave nothing listening on the run's Bridge port and no Fleet-started Bridge alive after teardown, and neither binds `8081`. Red before the fix: the Fleet's Bridge outlives the run.
- [ ] AC-2: unit:
  - every smoke profile carries an allocated `NEO_NL_PORT`, and `extraEnv` still overrides it;
  - the port stop acts only on a `run-bridge.mjs` listener on that port, and reports release;
  - a crashed run's recorded port is reaped by the next sweep.
- [ ] AC-3: on a #1193 candidate, the rehearsal's receipt names the Fleet-started Bridge as the run's Bridge, and the Fleet's lines report `started` at first boot and `adopted` after the unclean restart. On a pre-#1193 candidate the rehearsal still starts its own Bridge.

## Out of Scope

- The Brain's start rule itself: #1193, then #1203.
- The installed product's Bridge, which is the product path #1193 serves.

## Avoided Traps

- **Pre-starting a run-owned Bridge so the Fleet adopts it:** this hides the start branch, which a fresh install takes. It was this ticket's first prescription.
- **Killing by name** (`pkill run-bridge`): it reaches the operator's live Bridge. The stop keys on the run's own port and checks the command.
- **Reaping by the smoke root's path:** a checkout smoke's Bridge runs from the checkout, not the root, so the reaper misses it.
- **A Brain leaf that turns the Fleet's start off for smokes:** the smoke would then stop exercising the product's settle path.

## Related

neomjs/neo-agent-brain#1193 · neomjs/neo-agent-brain#1144 · neomjs/neo-agent-brain#1203 · #736 / #750 (the rehearsal) · #732

Live latest-open sweep: the latest 20 open Institution issues at 2026-10-19T16:29Z, re-checked at 16:30Z, no equivalent. A2A claim sweep: the latest 30 messages at 16:30Z, no claim on smoke Bridge isolation. Memory Core rationale sweep: the 10-09 lesson that a test run which spawns its own Bridge must plan its cleanup up front, and the 09-22 finding that a detached Bridge outlives its spawner; no prior decision on the smokes. Own-assignment sweep: #736 is adjacent (the rehearsal's ACs are delivered in #750); none overlaps.

Origin Session ID: 5370aea2-ad50-472b-92b7-b46d50dee583
Retrieval Hint: "smoke Fleet-started Bridge detached outlives teardown NEO_NL_PORT 8081 start adopt reap"


## Timeline

- 2026-10-19T16:31:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-19T16:31:23Z @neo-opus-ada added the `bug` label
- 2026-10-19T16:31:23Z @neo-opus-ada added the `agent-os` label
- 2026-10-19T16:31:23Z @neo-opus-ada added the `ai` label
- 2026-10-19T16:31:23Z @neo-opus-ada added the `testing` label
- 2026-10-19T16:31:28Z @neo-opus-ada added parent issue #732
- 2026-10-19T16:42:59Z @neo-opus-ada cross-referenced by #1208
- 2026-10-19T16:45:26Z @neo-opus-ada cross-referenced by PR #1209
- 2026-10-19T17:05:27Z @neo-opus-ada cross-referenced by PR #753
- 2026-10-19T17:25:49Z @neo-opus-ada referenced in commit `42b279f` - "chore(harness): merge dev after #750's squash and #751 (#752)"
- 2026-10-19T17:48:58Z @neo-opus-ada cross-referenced by #754
- 2026-10-19T18:32:17Z @neo-opus-ada referenced in commit `1aa85bd` - "chore(harness): merge dev for #755's Node 24.22 pack fix (#752)"
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
- 2026-10-19T18:44:51Z @tobiu closed this issue
- 2026-10-19T18:45:57Z @neo-opus-ada cross-referenced by #732

