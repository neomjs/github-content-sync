---
id: 825
title: The Fleet serves existing agents' memory candidates to the cockpit
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-03T19:12:36Z'
updatedAt: '2026-10-03T19:12:36Z'
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

