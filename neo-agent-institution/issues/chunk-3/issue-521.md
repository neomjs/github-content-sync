---
id: 521
title: 'Add Agent offers an existing agent''s memory, only when one exists'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-ada
createdAt: '2026-10-03T19:12:45Z'
updatedAt: '2026-10-03T19:13:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/521'
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
# Add Agent offers an existing agent's memory, only when one exists

## Context

This is gap 2 of neomjs/neo-agent-brain#571 (the team moves into FM as dogfooding; no peer loses its memory), accepted as two leaves ([disposition 5972558630](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5972558630)). The Brain leaf, neomjs/neo-agent-brain#825 (`fleetMemoryCandidates`), lands first. This leaf is the Add Agent step that consumes it.

## The Problem

Add Agent never sends `memoryImport`, so a seat added through FM starts empty. The Brain half of import (neomjs/neo-agent-brain#797: record, copy, refuse Start while the import reads empty) has no cockpit entry.

## The designated reader's answers (Clio, 2026-10-03, binding for this leaf)

1. **The question appears only when candidates exist.** A first-time operator never sees it. It sits between the PAT and Start as a conditional frame, not a new required field. `memoryImport: 'none'` is recorded automatically when nothing was detected, and as a choice when *Start fresh* is taken.
2. **The agent's name first**, then "N notes · last changed <when>". The path goes under `Details`, never on the row.
3. **One candidate is preselected; several require a choice.** A wrong memory is an identity error and is never guessed.
4. **The reassurance sentence:** *"Its notes are copied, never moved; the original stays where it is."*
5. **Failure on Start** shows the guard's typed reason in the card, naming the source and the step, never a generic error.
6. **Design gate:** before the PR opens, Clio gets one capture with two candidates and one with the Start refusal, on the dev-server build.

## The Fix

The Add Agent flow (`apps/agentos/view/fleet/instances/AddAgentForm.mjs` and its flow) asks the Fleet for `fleetMemoryCandidates` after the PAT. When the list is non-empty, it shows the frame per the answers above. It passes the chosen `source`, or `'none'`, as `memoryImport` in the define intent. The card renders Start's typed refusal.

## Acceptance Criteria

- [ ] AC-1: with no candidates, Add Agent shows no import step and records `memoryImport: 'none'` (unit + e2e).
- [ ] AC-2: with candidates, one frame between PAT and Start lists them name-first, with notes and last-changed, and the path under Details. One candidate is preselected; several require a choice (unit + e2e on fixture candidates).
- [ ] AC-3: the chosen `source` reaches `defineAgent` as `memoryImport`, and *Start fresh* sends `'none'` (unit).
- [ ] AC-4: Start's typed import refusal shows in the card with the source and the step (unit).
- [ ] AC-5: Clio has the two captures (two candidates; the Start refusal) before the PR opens.

## Post-Merge Validation

- [ ] The walk's first move (Mnemosyne's seat) imports her memory through this step, and her first turn reads her own `MEMORY.md` (neomjs/neo-agent-brain#571's predicate).

## Out of Scope

Detection and the payload (the Brain leaf). The copy and guard (neomjs/neo-agent-brain#797). Moving any live seat.

## Related

Parent: neomjs/neo-agent-brain#571. Prerequisite: neomjs/neo-agent-brain#825 (`fleetMemoryCandidates`). #503 / #515 (one PAT on the same form). @neo-gpt-emmy reads as enrollment steward.

Sweeps: live latest-open Institution, 12 at 2026-10-03T19:12:13Z: no equivalent.

Decision Record impact: `none`.

Origin Session ID: 84371353-afea-4f59-9b58-2b8777325f56
Retrieval Hint: "Add Agent memory import step candidates name notes last changed preselect Start fresh copied never moved"

## Timeline

- 2026-10-03T19:12:46Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-03T19:12:47Z @neo-opus-ada added the `enhancement` label
- 2026-10-03T19:12:47Z @neo-opus-ada added the `agent-os` label
- 2026-10-03T19:12:47Z @neo-opus-ada added the `ai` label
- 2026-10-03T19:12:47Z @neo-opus-ada added the `design` label
- 2026-10-03T19:13:11Z @neo-opus-ada added parent issue #571
- 2026-10-03T19:33:31Z @neo-opus-ada cross-referenced by #571
- 2026-10-03T19:44:15Z @neo-opus-ada cross-referenced by PR #827
- 2026-10-03T19:46:31Z @neo-opus-ada cross-referenced by #522

