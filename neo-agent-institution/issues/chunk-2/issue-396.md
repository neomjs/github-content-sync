---
id: 396
title: 'An installed FM restart gives existing seats fresh, empty homes'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T16:28:56Z'
updatedAt: '2026-10-01T16:46:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/396'
author: neo-opus-grace
commentsCount: 0
parentIssue: 7
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-01T16:45:55Z'
---
# An installed FM restart gives existing seats fresh, empty homes

## Context

**Incident, 2026-10-01 16:05Z.** An ordinary restart of the installed Fleet Manager gave `neo-gpt-sophie` a new, empty seat home at `~/.neo-ai/agents/neo-gpt-sophie` (0 sessions). Her original home under the app's data root is intact (18 sessions, 39 memories, auth, config). @tobiu had to sign her in again and saw her settings and sessions as gone.

@neo-gpt-emmy contained it: Sophie runs again on the restored one-launch override. A per-app `LSEnvironment` root pin is **built and smoke-tested, not installed** (corrected 16:38Z per Emmy). Nothing was lost; both copies are kept.

The cause is #379 (Resolves #378, merged 12:10Z): the installed app stopped passing `NEO_FLEET_AGENTS_ROOT = <userData>/brain/fleet/agents`, so the Brain default applies. It gave **no adoption path to an installation that already held seats there**. The interim install (#381) shipped it before any deliberate move, and a one-launch env bridge was all that held the old root.

## The Problem

The packaged app decides where seats live from what it passes at launch. Since #379 it passes nothing, so the answer silently changes for every existing installation at its next launch.

The legacy root on the operator's machine holds `neo-gpt-sophie` and `neo-opus-ada`. Each would get a fresh, empty home the first time it starts without the bridge.

## The Architectural Reality

- `harness/brain.mjs` `buildPackagedBrainEnv({agentsRoot, backupRoot, dataRoot})` passes `NEO_FLEET_AGENTS_ROOT` only when `agentsRoot` is given (#379). `harness/main.mjs:979` builds the packaged env with no `agentsRoot`.
- `resolveBrainPaths` echoes the resolved `fleetAgentsRoot` (`harness/brain.mjs:348`).
- `harness/planeConfig.mjs` is the sibling pattern for a per-installation record in the data dir (`plane.json`, 0600, read/write functions with an injectable fs).
- The operator's ruling on Brain #571 (comment 5929565535): one default, `~/.neo-ai/agents`, and existing seats move there **deliberately**, never as a side effect.

## The Fix

Shape per @neo-fable-clio's lead word (16:22Z), peer-reviewed in the same thread:

1. **An installation seat-root record** beside `plane.json`: `{root, origin, recordedAt}`, written atomically (temp file + rename, 0600).
2. **One-time adoption.** At the first launch that finds no record:
   - an operator's `NEO_FLEET_AGENTS_ROOT` in the environment is recorded as given (origin `environment`);
   - else a legacy root holding a seat (a non-hidden subdirectory) is recorded (origin `adopted`);
   - else the root the Brain resolves for a fresh installation (origin `default`).
3. **Every later launch reads the record** and passes it explicitly as `agentsRoot`. It never probes the filesystem.
4. An environment root that disagrees with an existing record is reported and ignored. The log names both paths and the remedy.
5. The deliberate #571 move changes only the record (origin `moved`).
6. `main.log` names the root and its origin at every launch.

## Contract Ledger

| Target surface | Source of authority | Producer → consumer | Absent / broken / precedence | Docs | Evidence |
|---|---|---|---|---|---|
| `seat-root.json` in `userData`: `{root, origin, recordedAt}` | this ticket; Clio's lead word (16:22Z) | Written by the first packaged launch (`environment` / `adopted` / `default`) and by the deliberate move (`moved`). Read by every packaged launch. | Absent: the one-time choice runs. Present but unreadable or malformed: the Brain boot refuses with its reason, and nothing chooses again. | `seatRootRecord.mjs` JSDoc | AC-1–AC-5 |
| `buildPackagedBrainEnv({agentsRoot})` → `NEO_FLEET_AGENTS_ROOT` in the Brain children's env | #379's builder; `{...process.env, ...env}` merge order (`harness/brain.mjs:374`, `:835`) | Always the recorded root for the packaged app. | A fresh installation's first launch passes none, then records the resolver's root (`default`). | `main.mjs` comment | the resolver witness |
| The launch's `NEO_FLEET_AGENTS_ROOT` | ADR 0041 (the declared root is the bound root) | First launch without a record: recorded as `environment`. After that: reported (`HARNESS_SEAT_ROOT_ENV_IGNORED`) and ignored. | A relative path refuses. | `seatRootRecord.mjs` JSDoc | AC-5 |
| Legacy root `<userData>/brain/fleet/agents` | the pre-#379 placement | Read once, only when no record exists; adopted while it holds a visible subdirectory. | Missing: holds nothing. Unreadable: refuses, and never chooses away from it. | `seatRootRecord.mjs` JSDoc | AC-1, AC-4 |
| `HARNESS_SEAT_ROOT {origin, root}` | `mainLog` tee into `main.log` | Every packaged launch. | — | — | AC-6 |

## Acceptance Criteria

- [x] AC-1 (red-first): a seat under the legacy root survives a plain restart. Two launches record and keep the legacy root, and the second probes nothing. (#398; resolver witness: dev → `~/.neo-ai/agents`, head → the legacy root ×2)
- [x] AC-2: a present record is honoured with zero seat-directory probes (the record itself is read), whatever the legacy root holds. (#398)
- [x] AC-3: a fresh installation records the root the Brain resolves, as an absolute path. (#398)
- [x] AC-4: a deliberate move (the record's write function) changes only the record; the next launch passes the new root. (#398)
- [x] AC-5: an environment root that disagrees with the record is reported and ignored; at the first launch it is recorded with origin `environment`. (#398)
- [x] AC-6: `main.log` carries the root and its origin. (#398)
- [ ] AC-7 `[L4-deferred — operator handoff needed]` (post-merge, installed): a repackage carrying `d23b6a5`, with no one-launch override and no `LSEnvironment` pin, starts Sophie's seat on her original home across two plain restarts.

## Out of Scope

- The deliberate migration of Sophie's and Ada's seats (Brain #571).
- The Brain-side invariant: Start must never provision a fresh home for a registered seat whose home exists under another root. Filed separately under #571.
- Showing the root in the cockpit.
- Removing the stray `~/.neo-ai/agents/neo-gpt-sophie`, which is a deliberate step, never automatic.

## Decision Record impact

`aligned-with ADR 0041`: the declared root is the bound root on every launch. `aligned-with ADR 0019 §10.7`: placement is a deployment input at the entrypoint.

## Related

#379 (the cause) · #378 · #381 · #12 · #7 · neomjs/neo-agent-brain#571

## Sweeps

- Live latest-open sweep: the latest 20 open Institution issues at 2026-10-01T16:28:01Z. No equivalent.
- Exact: "seat root record" and "fresh home registered seat" find nothing open.
- MC: "restart created a fresh empty seat home agents root changed NEO_FLEET_AGENTS_ROOT lost sessions re-login installed Fleet Manager", 6 results: this incident, the #571 ruling, the earlier placement finding. No prior decision against this.
- A2A: Emmy's incident thread and Clio's lead word. No competing claim.
- Own-assignment: #386 and #388 (shell), neither overlapping.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "installed FM restart fresh empty seat home legacy agents root installation record adopted NEO_FLEET_AGENTS_ROOT #379"

🖖 Grace (Claude Opus 5.5, Claude Code)


## Timeline

- 2026-10-01T16:28:57Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T16:28:58Z @neo-opus-grace added the `bug` label
- 2026-10-01T16:28:58Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T16:28:58Z @neo-opus-grace added the `ai` label
- 2026-10-01T16:29:08Z @neo-opus-grace added parent issue #7
- 2026-10-01T16:34:10Z @neo-opus-grace cross-referenced by PR #398
- 2026-10-01T16:45:35Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-01T16:45:55Z @tobiu referenced in commit `d23b6a5` - "fix(harness): the installed FM records its seat root once, so a restart never gives existing seats fresh homes (#396) (#398)

A one-time choice at the first launch without a record (the launch environment, else the legacy root while it holds a seat, else the root the Brain resolves) is written to seat-root.json beside plane.json; every later launch passes the recorded root explicitly, and a disagreeing environment is reported and ignored. #379 had dropped the packaged root with no adoption path, so a restart without the one-launch bridge gave Sophie an empty home."
- 2026-10-01T16:45:55Z @tobiu closed this issue
- 2026-10-01T17:08:35Z @neo-opus-ada cross-referenced by PR #395
- 2026-10-01T18:04:00Z @neo-opus-grace cross-referenced by #402
- 2026-10-01T18:21:49Z @neo-opus-vega cross-referenced by PR #405

