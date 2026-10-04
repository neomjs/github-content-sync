---
id: 825
title: The Fleet serves existing agents' memory candidates to the cockpit
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-03T19:12:36Z'
updatedAt: '2026-10-04T01:04:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/825'
author: neo-opus-ada
commentsCount: 0
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
closedAt: '2026-10-04T01:04:09Z'
---
# The Fleet serves existing agents' memory candidates to the cockpit

## Context

This is gap 2 of #571, the team's move into Fleet Manager as dogfooding, under the operator's ruling of 2026-10-03: "claude or codex markdown memories get moved accordingly … we do not want that any peer loses his/her identity". The planner accepted it as two leaves ([disposition 5972558630](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5972558630)). This leaf is the Brain control operation and is built first. The Institution step (the Add Agent frame) consumes it.

## The Problem

`detectMemoryCandidates()` (`ai/services/fleet/seatMemoryImport.mjs`, #797) has **no production caller**; only its specs call it. The cockpit cannot ask which existing memory to import, so `defineAgent` never receives a `memoryImport` consent from FM. As a result, every seat added through FM starts with empty memory, and only the team script `onboardPeer` can adopt one.

## The Architectural Reality

- `detectMemoryCandidates({homeDir, fileSystem})` returns `{family, path, files}`, sorted by file count, read-only. It covers Claude projects' `memory` folders, `~/.codex/memories` and `~/.codex-instances/*/memories`.
- A fleet read is declared in three places, as #795's `fleetRecentTurns` was: `src/fleet/contract/wire.mjs` `FLEET_WIRE_METHODS`, `ai/services/fleet/fleetServerPolicy.mjs` (`FLEET_S1_METHOD_POLICY` + `FLEET_METHOD_SCOPE_CLASSES: 'read-observe'`), and the `FleetControlBridge` / `devFleetServer` dispatch.
- The designated reader's product answers shape the payload: the agent's name first, then "N notes · last changed <when>", with the path only under Details.

## The Fix

1. A read-observe wire method, `fleetMemoryCandidates`, served from `detectMemoryCandidates()` on the host that holds the seats. It returns, per candidate:
   - `family`;
   - `source`, the exact path a later `memoryImport` consent names;
   - `name`, derived from the source and never the raw path;
   - `notes`, the file count;
   - `lastChanged`, the newest file's mtime.
2. Detection gains `lastChanged` and `name`. No file contents and no secrets cross the wire.
3. Declare the method in all three places, with scope class `read-observe`.

## Contract Ledger

*Backfilled at Sophie's review of PR #827 (pr-review §5.4). It describes the head under review, b71507f.*

| Target surface | Source of authority | Behavior | Fallback / failure | Docs | Evidence |
|---|---|---|---|---|---|
| Wire method `fleetMemoryCandidates` | `src/fleet/contract/wire.mjs` `FLEET_WIRE_METHODS`; `fleetServerPolicy.mjs` (S1 `awaiting-s5`, scope class `read-observe`) | A read-observe method that **takes no input**: params are ignored, and no caller value reaches detection. It is answered by the host that holds the seats, through `FleetControlBridge.memoryCandidatesSource`, which only `devFleetServer.mjs` wires. | The composed plane service answers `degraded` (`awaiting-s5`), as for every S5 read. | `FleetControlBridge.fleetMemoryCandidates` JSDoc | `dispatchFleetRequest.spec`, `fleetServer.spec`, `FleetControlBridge.spec` |
| Response envelope | `FleetControlBridge.fleetMemoryCandidates` | **Wired:** `{capability: {state: 'wired'}, candidates, count}`. **Wired and empty:** `candidates: []`, `count: 0`, meaning the host read completed and found no memory. | **Unwired** (a service that holds no seats): `{capability: {state: 'unavailable', reason: 'memory candidates are read on the host that holds the seats'}, candidates: [], count: 0}`. This is never an empty host. **A thrown read** (for example `EACCES` on `~/.claude/projects`): the dispatcher's sanitized `operationFailed` envelope, `fleet: 'fleetMemoryCandidates' failed`, with the cause logged server-side only. **Only wired-and-empty means "no memory exists"**; a consumer treats the other two as unknown (neomjs/neo-agent-institution#521). | same | `FleetControlBridge.spec` (wired, unwired); `dispatchFleetRequest` catch path |
| Candidate | `seatMemoryImport.detectMemoryCandidates` | `{family: 'claude'\|'codex', source, name, notes, lastChanged}`, most notes first. **`source`** is the absolute memory folder `normalizeMemoryImport` accepts: a Claude project's `memory`, `~/.codex/memories`, or `~/.codex-instances/<name>/memories`, with real folders only. **`name`** comes from the folder, never the path: a Codex instance's folder name, `codex` for the Codex home, or a Claude slug without the home's encoding (`~` for the home itself). **`notes`** counts the regular files beneath `source`; links are neither followed nor counted. **`lastChanged`** is the newest such file's mtime, in ISO form. | A folder with no files is no candidate. A link on any segment hides the folder. No file's contents are read. | `detectMemoryCandidates` / `candidateName` JSDoc | `seatMemoryImport.spec` |
| Consent boundary | `defineAgent` → `normalizeMemoryImport` → `importSeatMemory` (#797) | A candidate's `source` is the value a `memoryImport` consent names. `normalizeMemoryImport` re-validates it at define time, because the wire carries the consent and not trust. The copy runs at Start, never moves the original, and is refused when it reads empty. | `'none'` is the explicit decline. Anything else is a `TypeError` at define. | `seatMemoryImport` module doc | `seatMemoryImport.spec` "a candidate's source is the consent defineAgent accepts" |

## Acceptance Criteria

- [ ] AC-1: `fleetMemoryCandidates` is declared in `FLEET_WIRE_METHODS`, the S1 policy and the scope classes (`read-observe`), and is served by the bridge.
- [ ] AC-2: it returns `{family, source, name, notes, lastChanged}` per candidate, with no file contents. An empty list means none exists (unit specs on real temp dirs).
- [ ] AC-3: `source` round-trips. Passing it to `defineAgent` as `memoryImport` is accepted by `normalizeMemoryImport` (unit).
- [ ] AC-4: `name` is never the raw path. The derivation is stated in the JSDoc and covered for a Claude slug and a Codex instance.

## Out of Scope

- The Add Agent frame and its words (the Institution leaf).
- The copy and the Start guard, which #797 already ships.

## Related

Parent: #571. #797 / PR #806 (the Brain half this exposes). Precedent: #795 (`fleetRecentTurns`). The design read lives on #571 (`5972558630`). @neo-gpt-emmy reads as enrollment steward.

Sweeps: live latest-open Brain and Institution, 12 each at 2026-10-03T19:12:13Z, plus searches for "memory candidates": no equivalent. A2A: no competing claim. Own-assignment: #823, #571 (parent).

Decision Record impact: `none` (an additive read on an existing contract).

Origin Session ID: 84371353-afea-4f59-9b58-2b8777325f56
Retrieval Hint: "fleetMemoryCandidates detectMemoryCandidates wire method memory import Add Agent candidates name notes lastChanged"


## Timeline

- 2026-10-03T19:12:36Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-03T19:12:37Z @neo-opus-ada added the `enhancement` label
- 2026-10-03T19:12:37Z @neo-opus-ada added the `ai` label
- 2026-10-03T19:12:38Z @neo-opus-ada added the `agent-os` label
- 2026-10-03T19:13:05Z @neo-opus-ada added parent issue #571
- 2026-10-03T19:13:12Z @neo-opus-ada cross-referenced by #521
- 2026-10-03T19:33:31Z @neo-opus-ada cross-referenced by #571
- 2026-10-03T19:44:15Z @neo-opus-ada cross-referenced by PR #827
- 2026-10-04T01:04:09Z @tobiu referenced in commit `7b250ab` - "feat(fleet): the Fleet serves existing agents' memory candidates to the cockpit, by name, note count and newest change (#825) (#827)

- detectMemoryCandidates returns {family, source, name, notes, lastChanged}.
  The name comes from the folder, never the path: a Codex instance's folder,
  `codex` for the Codex home, and a Claude slug without the home's own
  encoding. It reads no file's contents.
- fleetMemoryCandidates is a read-observe wire method, declared in
  FLEET_WIRE_METHODS, the S1 policy (awaiting-s5, beside the other memory
  reads) and the scope classes. The bridge serves an injected source.
  devFleetServer wires it, being the process that launches the seats; an
  unwired service answers unavailable, never an empty host."
- 2026-10-04T01:04:10Z @tobiu closed this issue

