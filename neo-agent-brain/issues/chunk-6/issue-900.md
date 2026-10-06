---
id: 900
title: A relocated seat home converges its Fleet-owned path pins at the new root
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-06T12:12:17Z'
updatedAt: '2026-10-06T14:58:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/900'
author: neo-opus-ada
commentsCount: 1
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-06T14:58:58Z'
---
# A relocated seat home converges its Fleet-owned path pins at the new root

## Context

The Brain half of neomjs/neo-agent-institution#573: the installed shell moves its seats from the app-data root it adopted on 2026-10-03 to the Brain's default `~/.neo-ai/agents` (Gap 13 of #571). The mechanism is settled on #573: the move runs in the shell's boot before the fleet child starts. It copies each materialized seat home, verifies it, relocates the binding with `relocateSeatHome`'s exact `from`, then re-runs the Fleet's convergence at the destination ([Mnemosyne's design read 6015907840](https://github.com/neomjs/neo-agent-institution/issues/573#issuecomment-6015907840), [AC-1 result 6015949744](https://github.com/neomjs/neo-agent-institution/issues/573#issuecomment-6015949744)).

Sophie's seat is the materialized case today: a Codex seat whose `CODEX_HOME` and checkout live in the app-data home (her session witness, 2026-10-06 11:48Z).

## The Problem

After a byte copy, the moved home's Fleet-owned path pins still name the old home, and convergence refuses them rather than re-deriving them:
- **The Claude memory pin.** `convergeSeatMemory` writes `autoMemoryDirectory` into the checkout's `.claude/settings.local.json` through `convergeJsonSetting`, whose rule is "A value already there that differs is refused, never replaced: it is someone else's decision about the same thing."
- **The Codex trust block.** `convergeCodexRemoteTrust` keeps a Fleet-owned `[projects."<managed repo>"]` block between markers. For a remote-MCP seat whose block names another path, with no trusted row for the new one, it refuses with "Fleet trust block diverged".

So a moved seat's first Start refuses. Worse, if a pin were left as it is, a Claude seat would keep writing its memory into the old home, which is the rollback copy.

## The Architectural Reality

Read at Brain `origin/dev` `1b69ef75`:
- `prepareManagedAgentWorkspace.mjs`: `convergeSeatMemory`, `convergeJsonSetting` and `convergeCodexRemoteTrust` (`:1855-1935`) are the one writer of each pin. Their refusals guard against another writer's edits and must stay.
- `FleetRegistryService.relocateSeatHome(id, {from, to})` (`:795`) is the one compare-and-set writer of a row's `seatHome`. It moves no files.
- `startAgentProvisioned` refuses a row whose recorded `seatHome` differs from the derived one (`FLEET_SEAT_HOME_MISMATCH`, `:240-262`).
- The shell asks the Brain through one-shot scripts in the Brain root (Institution `harness/brain.mjs`, `runBrainScript`).

## The Fix

Decided at intake ([6016008039](https://github.com/neomjs/neo-agent-brain/issues/900#issuecomment-6016008039)): a relocation is one more bounded history entry, the same kind convergence already honours for an earlier placement (`previousPlacementPlan`: "upgrading only an untouched Fleet projection… unrelated operator edits remain a divergence").
- `relocateSeatHome` records the previous home on the row as `previousSeatHome`. The compare-and-set itself is unchanged.
- Start passes the previous root to `prepareManagedAgentWorkspace`. Every Fleet-owned, path-bearing artifact also accepts exactly its previous-home rendering as upgradable: the Claude memory pin, the Codex trust block, and the Kimi and OpenCode configs. Any other value still refuses, as today.
- **One move script for the shell's one-shot seam:**
  - it inventories the rows (bound home, materialized, destination state), and a dry run stops there;
  - it copies each materialized home, never moves it, with a clone copy where the volume allows, into a staging folder beside its destination, verifies it identical, and only then publishes it under its own name;
  - it relocates each binding with the exact `from`, reading `seatHome === to` as already done;
  - it refuses, with nothing changed, while a seat's lease names a live process: an app seat (Codex Desktop, Claude Desktop) outlives the Fleet server and would keep writing into the home it started in;
  - it reports per-row state as JSON, naming the sockets and pipes it could not copy.

  It writes no artifact, because the first Start's own convergence re-derives them. It changes no root record either; that is the shell's commit point (neomjs/neo-agent-institution#573).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `convergeSeatMemory` / `convergeJsonSetting` (Claude memory pin) | #573's AC-1 result | re-derives a pin equal to the previous home's derivation | any other differing value refuses, as today | JSDoc | red-first fixture |
| `convergeCodexRemoteTrust` (Codex trust block) | same | re-derives a Fleet block naming the previous home's repo path | any other divergence refuses, as today | JSDoc | red-first fixture |
| the move script | #573's mechanism | inventory, copy and verify, exact-`from` relocation; JSON report; resumable | an occupied or divergent target, or a seat whose lease names a live process, stops the run, and nothing changes | script header | unit on real temp roots |
| the move's `moveId` (helper ↔ shell admission) | #901 review, RA-6 | the caller's id for one consented move. A stage (`.moving-<id>`) and its marker (`.moving-<id>.owner`) are that move's own only when the marker is a plain file carrying it, and only those are ever discarded or removed. The caller passes the same id on every run of one consent, which is what lets an interrupted run resume. | a missing id, another id, a link or any other file at either path stops the row untouched. A new consent does not adopt an earlier attempt's leftovers; the operator removes them | `moveSeatHomes` JSDoc | unit on real temp roots |

## Acceptance Criteria

- [ ] AC-1: For every launchable harness family that prepares a workspace, with local and remote MCP, a seat folder copied to a new root converges when prepared there with the previous root: every Fleet-owned artifact names the new home (one parametrized spec, red-first against today's refusals). Antigravity prepares none, and a spec pins that refusal.
- [ ] AC-2: Without the previous root, the same copy refuses as today. With it, an artifact edited away from both renderings still refuses.
- [ ] AC-3: `relocateSeatHome` records `previousSeatHome`, and Start passes the previous root to the workspace preparation.
- [ ] AC-4: The move script classifies every row, a home counting as materialized when it holds the harness homes the Fleet provisions. It stops, without changing anything, on any of these:
  - roots that resolve to the same folder, or one inside the other;
  - an occupied or divergent destination, or a staging folder the move did not leave;
  - a source folder the Fleet did not provision;
  - a seat whose lease names a live process or cannot be read. It copies and verifies materialized homes, and relocates bindings with the exact `from`.
- [ ] AC-5: Run again after an interruption at any step, the script completes: verified copies are kept, and rows already at `to` read as done.

## Out of Scope

- The shell's transition record, the boot orchestration, the root record and the System view (neomjs/neo-agent-institution#573).
- Deleting the old homes (a later step, with one receipt per seat).
- Seat roots other than the Brain default.

## Related

Parent: #571 · Blocks neomjs/neo-agent-institution#573 · #704 · #797

Decision Record impact: none. The pins stay Fleet-owned with one writer, and relocation is the declared `relocateSeatHome` act.

Sweeps (2026-10-06 12:12Z, immediately before filing):
- Live latest-open: the latest 20 open issues here (#599 to #898). No equivalent.
- A2A: the last 30 inbox messages, all read states. No lane-claim on this scope; Mnemosyne's design read names the rule this ticket builds.
- MC sweep: one query on the problem's nouns ("moved seat home first Start refuses, settings.local.json autoMemoryDirectory names old path, Codex trust block diverged after copy"), 5 results. Clio's 09-30 hand-move runbook (copy, re-key Codex trust rows by hand) is the manual precedent this replaces; no prior decision against re-derivation.
- Own-assignment sweep: #571 and #898 here; neither touches convergence pins.

Ownership: @neo-opus-ada (the #571 and neomjs/neo-agent-institution#573 owner).

Origin Session ID: 4095e966-9503-44c4-a759-5e7293dc3daa

Retrieval Hint: "relocated seat home convergence re-derive Fleet-owned path pin autoMemoryDirectory Codex trust block diverged move script relocateSeatHome"


## Timeline

- 2026-10-06T12:12:19Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-06T12:12:19Z @neo-opus-ada added the `enhancement` label
- 2026-10-06T12:12:20Z @neo-opus-ada added the `ai` label
- 2026-10-06T12:12:20Z @neo-opus-ada added the `agent-os` label
- 2026-10-06T12:12:31Z @neo-opus-ada added parent issue #571
- 2026-10-06T12:12:43Z @neo-opus-ada cross-referenced by #573
### @neo-opus-ada - 2026-10-06T12:15:05Z

## Intake: the census, and the design it points to (owner, 2026-10-06)

**Census, measured at Brain `1b69ef75`** with the real `prepareManagedAgentWorkspace` on temp roots. For every launchable family, local and remote MCP, a seat is prepared under root A, its folder is copied to root B, and it is prepared again at B:

| Family | Local MCP | Remote MCP |
|---|---|---|
| `claude-code`, `claude-desktop` | refuses at `.claude/settings.local.json` (`autoMemoryDirectory` names A) | same |
| `codex`, `codex-desktop` | converges (all `MATCH`) | refuses at the Codex `config.toml` (the Fleet trust block names A's checkout) |
| `kimi-code` | refuses at `harness/kimi-code/config.toml` ("divergent Fleet-owned content") | same |
| `opencode` | refuses at `opencode.jsonc` (transport projection) | same |

Five of six families refuse after a byte copy; only a local-MCP Codex seat survives. Sophie's seat is `codex-desktop` on a remote target, so her first Start after the move would refuse on the trust block, exactly as predicted on neomjs/neo-agent-institution#573.

**Design: a relocation is one more bounded history entry.** Convergence already upgrades a file that equals an earlier Fleet rendering, and refuses everything else. The function is `previousPlacementPlan`: "Exact pre-placement slot vocabulary for upgrading only an untouched Fleet projection… unrelated operator edits remain a divergence". The previous home's rendering is the same kind of history:
1. `relocateSeatHome` records the previous home on the row (`previousSeatHome`, as provenance). Today's compare-and-set is unchanged.
2. Start passes it to `prepareManagedAgentWorkspace` as the previous instance root.
3. Each Fleet-owned, path-bearing artifact also accepts exactly its previous-home rendering as upgradable:
   - the Claude memory pin, as a previous value for `convergeJsonSetting`;
   - the Codex trust block, as the previous repo path for `convergeCodexRemoteTrust`;
   - the Kimi and OpenCode configs, appended to their legacy-content lists.

   Any other value still refuses, as today.
4. The move script copies, verifies and relocates only. The re-derivation is the first Start's own convergence, with Start's full context. So nothing outside convergence writes an artifact, and nothing duplicates Start's inputs.

The record needs no clearing: the effect applies only to bytes that still equal the old home's Fleet rendering, which is the same "untouched Fleet projection" rule the placement history already follows.

**Fixtures, red-first:** the census becomes a spec parametrized over every launchable family, local and remote. Prepare at A, copy, then prepare at B with the previous root: it converges. Two controls: without the previous root it refuses (today's behaviour), and an artifact edited away from both renderings still refuses.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


- 2026-10-06T12:44:44Z @neo-opus-ada cross-referenced by PR #901
- 2026-10-06T12:48:02Z @neo-opus-ada referenced in commit `527b5a2` - "feat(fleet): the move adopts no seat folder the Fleet did not provision (#900)

A seat home counts as materialized only when it holds the harness homes the
Fleet derives. A folder at the bound home without them stops the move, naming
it to rename aside; once it is gone, the row's binding moves alone. This is the
inventory rule of the design read on neomjs/neo-agent-institution#573."
- 2026-10-06T12:51:17Z @neo-opus-ada cross-referenced by #571
- 2026-10-06T13:47:55Z @neo-opus-ada referenced in commit `8d7b73a` - "fix(fleet): a relocation moves only what the Fleet owns, and the move admits only an independent, quiet, owned path (#900)

Sophie's round 1 on #901:
- RA-1: relocated lines must start inside the ranges each policy owns: the
  neo-mjs-* entries, OpenCode's instructions and the seat's moved permission
  grants, Kimi's owned blocks, and whole-file hooks. An operator's entry that
  copies a Fleet entry stays byte-identical.
- RA-2: a destination checkout the operator distrusts refuses the moved Codex
  trust block, without writing.
- RA-3: roots that resolve to the same folder, or one inside the other, refuse
  before anything is written.
- RA-4: the manifest records the seat folder's own mode.
- RA-5: a lease that cannot be read or names no process refuses as unknown.
- RA-6: a staging folder is the move's own only when its marker carries the
  caller's moveId; any other stays untouched and stops the move."
- 2026-10-06T13:58:17Z @neo-gpt-emmy cross-referenced by #582
- 2026-10-06T14:00:20Z @neo-opus-ada referenced in commit `da5a1bb` - "fix(fleet): the move admits a staging marker only as its own plain file, and never writes through one (#900)

Sophie's round 2 on #901 (RA-6): the marker the round-1 repair introduced
was written through, and removed, whatever stood at its path. Now:
- the plan admits the stage and the marker together. Only a plain file
  carrying this move's id is the move's own; a link, another id or any other
  file stops the row untouched;
- the marker is created exclusively, so an existing path is never written
  through;
- only an owned marker or stage is ever removed: after publishing, on a failed
  verification, and in a published copy's recovery."
- 2026-10-06T14:58:58Z @tobiu referenced in commit `a8dd1ae` - "feat(fleet): a moved seat starts at its new root, and the shell's move of the seat homes (#900) (#901)

* feat(fleet): a relocated seat's first Start re-derives its Fleet-owned files (#900)

relocateSeatHome records the home a move left as previousSeatHome, and Start
passes that home's root to the workspace preparation. Each Fleet-owned file
still exactly as Fleet rendered it there moves to the new home: the Claude
memory pin, the Codex trust block, and the Kimi and OpenCode generated files
(line by line, verified against their owned projections). Anything else still
refuses, as before.

check-block-alignment re-aligned pre-existing import blocks in the touched
files (whitespace only).

* feat(fleet): move the seat homes to another agents root (#900)

moveSeatHomes classifies every registry row before anything changes. It copies
each materialized home into a staging folder, proves it identical and publishes
it, then relocates the binding with its exact from. A destination holding
anything but a verified copy, or a seat whose lease names a live process, stops
the run with nothing changed. Run again after an interruption, it completes.
Sockets and pipes are left behind and named per row. FleetLifecycleService
exports its lease file name for the check.

* feat(fleet): the move adopts no seat folder the Fleet did not provision (#900)

A seat home counts as materialized only when it holds the harness homes the
Fleet derives. A folder at the bound home without them stops the move, naming
it to rename aside; once it is gone, the row's binding moves alone. This is the
inventory rule of the design read on neomjs/neo-agent-institution#573.

* fix(fleet): a relocation moves only what the Fleet owns, and the move admits only an independent, quiet, owned path (#900)

Sophie's round 1 on #901:
- RA-1: relocated lines must start inside the ranges each policy owns: the
  neo-mjs-* entries, OpenCode's instructions and the seat's moved permission
  grants, Kimi's owned blocks, and whole-file hooks. An operator's entry that
  copies a Fleet entry stays byte-identical.
- RA-2: a destination checkout the operator distrusts refuses the moved Codex
  trust block, without writing.
- RA-3: roots that resolve to the same folder, or one inside the other, refuse
  before anything is written.
- RA-4: the manifest records the seat folder's own mode.
- RA-5: a lease that cannot be read or names no process refuses as unknown.
- RA-6: a staging folder is the move's own only when its marker carries the
  caller's moveId; any other stays untouched and stops the move.

* fix(fleet): the move admits a staging marker only as its own plain file, and never writes through one (#900)

Sophie's round 2 on #901 (RA-6): the marker the round-1 repair introduced
was written through, and removed, whatever stood at its path. Now:
- the plan admits the stage and the marker together. Only a plain file
  carrying this move's id is the move's own; a link, another id or any other
  file stops the row untouched;
- the marker is created exclusively, so an existing path is never written
  through;
- only an owned marker or stage is ever removed: after publishing, on a failed
  verification, and in a published copy's recovery."
- 2026-10-06T14:58:58Z @tobiu closed this issue

