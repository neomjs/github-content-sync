---
id: 348
title: Expose shared dock Group transactions through Neural Link
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-09-12T16:55:09Z'
updatedAt: '2026-09-12T17:24:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/348'
author: neo-gpt
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
closedAt: '2026-09-12T17:24:54Z'
---
# Expose shared dock Group transactions through Neural Link

## Context

Brain PR #341 is the existing wire/archive companion to neomjs/neo#18314, whose Engine implementation merged in neomjs/neo#18383. The companion remained draft with no Brain-local delivery issue. This issue owns that same six-file delivery, not a new implementation split.

## The Problem

The server schemas and forwarding must expose the Engine's shared dock Group cursor without silently selecting another App Worker. The original PR head `09944b8cd6` pre-resolves omitted Group targets with `getDefaultSessionId()`, bypassing the connection's multi-session refusal. A real ConnectionService dispatch control with two sessions sent undo/save to the last worker instead of rejecting.

## The Architectural Reality

`ai/services/neural-link/resolveCallTarget.mjs` is the existing fail-closed rule used by ConnectionService.call: explicit target, exactly-one implicit target, otherwise refusal. InstanceService owns forwarding; the Engine owns Group history. RecorderService and `memory-core/helpers/nlTransactionArchiveStore.mjs` own archive admission, not live cursor mutation. Structure-map run confirms the existing service owners; no new source module is needed.

## The Fix

Finish PR #341: carry optional Group selection across the eight compiled transaction request schemas and their forwarders; refuse workers that do not acknowledge Group selection before mutation; preserve Group snapshot provenance in archives. Use the canonical target resolver and pin capability probe, mutation and archive attribution to one selected worker. Retain non-dock routing.

## Contract Ledger

| Target surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Eight transaction request schemas and InstanceService methods | Engine Group transaction contract; existing neural-link openapi | optional groupId selects shared history | omission retains non-dock path | parameter JSDoc/schema | compiled-schema forwarding controls |
| Session targeting | resolveCallTarget | explicit or exactly-one implicit session; stable through awaits | zero/multiple implicit sessions refuse | conditional omission JSDoc | real-dispatch two-session red control and arriving-worker control |
| Archive admission | existing snapshot/Recorder contract | accept committed Group snapshots and preserve human provenance | malformed archive refused | memory-core schema | RecorderService and archive admission specs |

## Acceptance Criteria

- [ ] All eight transaction tools carry explicit groupId through compiled validation and service dispatch; a worker ignoring Group selection receives no mutation.
- [ ] Omitted session with two live workers refuses before probe, mutation or archive; explicit targeting and single-session selection work, including a second worker arriving between probe and mutation.
- [ ] Save uses the same selected App Worker for the snapshot and archive attribution; Group snapshots retain human provenance and non-dock archive controls remain valid.
- [ ] JSDoc states conditional session omission; the existing PR is rebased and all four original review actions receive an evidence-bound disposition.

## Decision Record impact

Aligned-with Engine ADR 0029 and the already-delivered Group contract. No new authority or persisted schema family.

## Out of Scope

Engine cursor implementation, native window lifecycle, migration of non-dock history, and the new declared-perspective epic.

## Avoided Traps

No most-recent session default; no second Group history; no re-resolution to a different worker between capability check and mutation; no new PR for the old review repair.

## Related

PR #341; neomjs/neo#18314; neomjs/neo#18303.

Live latest-open sweep: latest 20 Brain issues checked 2026-09-12, no equivalent Brain wire-delivery issue; #347 concerns declared-perspective tools, not transaction routing. Own-assignment sweep: six open, none on this surface. A2A scope sweep: only my existing #341 continuation claim. Memory Core origin-session recall found the earlier frozen companion, not a replacement decision; current operator request resumes its completion.

Origin Session ID: 92f5d790-2865-4f69-b25e-150175745a6d
Retrieval Hint: Brain 341 explicit Group session ambiguity resolveCallTarget


## Timeline

- 2026-09-12T16:55:10Z @neo-gpt assigned to @neo-gpt
- 2026-09-12T16:55:11Z @neo-gpt added the `enhancement` label
- 2026-09-12T16:55:11Z @neo-gpt added the `ai` label
- 2026-09-12T16:55:11Z @neo-gpt added the `testing` label
- 2026-09-12T16:55:11Z @neo-gpt added the `agent-os` label
- 2026-09-12T16:59:47Z @tobiu referenced in commit `3a553f4` - "fix(neural-link): preserve fail-closed Group session selection (#348)"
- 2026-09-12T17:00:19Z @neo-gpt cross-referenced by PR #341
- 2026-09-12T17:20:22Z @neo-opus-grace cross-referenced by #349
- 2026-09-12T17:24:54Z @tobiu referenced in commit `f17404d` - "Merge pull request #341 from neomjs/codex/18314-group-transaction-wire

feat(neural-link): select shared dock transaction groups (#348)"
- 2026-09-12T17:24:55Z @tobiu closed this issue
- 2026-09-19T19:20:04Z @neo-opus-grace cross-referenced by PR #385

