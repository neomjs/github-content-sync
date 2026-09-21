---
id: 398
title: Backup topology presents client paths as physical storage
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-20T04:32:45Z'
updatedAt: '2026-09-21T11:09:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/398'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 306
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-21T11:09:49Z'
---
# Backup topology presents client paths as physical storage

## Context

Delivery child of #306, limited to its backup-topology criterion. PR #397 isolates this correction; the parent stays open for physical compaction and daily orchestration. This is bookkeeping for the existing implementation lane, not a second implementation reservation or a graduation of the physical-maintenance design.

## The Problem

At `dev@0838bc3840431365309622c43d852d60c280418f`, `backup.mjs` copies client-process Chroma filesystem coordinates into `bundle-meta.json`. The JSONL exporter has not observed the server's filesystem. A configured path is consequently presented as physical topology without evidence.

## The Architectural Reality

[`buildTopologyDescriptor`](https://github.com/neomjs/neo-agent-brain/blob/0838bc3840431365309622c43d852d60c280418f/ai/scripts/maintenance/backup.mjs#L986) owns serialization. [Restore's topology check](https://github.com/neomjs/neo-agent-brain/blob/0838bc3840431365309622c43d852d60c280418f/ai/scripts/maintenance/restore.mjs#L1208) consumes `chromaUnified` / `shared_topology`; logical imports use the bundle's JSONL directories, not the source path fields. The existing unit specs exercise those producer/consumer boundaries.

## The Fix

Keep endpoint coordinates and topology flags. Record unobserved physical paths as null with a reason at the existing metadata writer. Cover actual written metadata and logical restore compatibility, and select the owning specs for hosted execution.

Prescription checked: `ai/scripts/maintenance/backup.mjs#buildTopologyDescriptor` owns the concern. Structure-map command `npm run ai:structure-map -- --files --loc` completed; existing maintenance scripts and existing specs remain their owners. No new module or configuration surface is needed.

### Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `bundle-meta.json.topology.kbChromaCoords.path` and `mcChromaCoords.dataDir` | Parent AC-3; producer above | Null when physical storage was not observed | `storageReason: physical-storage-not-observed` on each coordinate object | Writer JSDoc | Written-bundle assertions |
| Existing endpoint fields and shared-topology flags | Existing bundle/restore contract | Retained | Existing legacy topology handling retained | Existing restore contract | Logical restore import control |

Decision Record impact: aligned-with ADR 0019 — read resolved endpoint leaves at the use site; do not equate a client namespace with an observed server filesystem.

## Acceptance Criteria

- [ ] Written backup metadata retains endpoint coordinates and the shared-topology flag, but both unobserved physical path fields are null with an explicit reason.
- [ ] Logical restore accepts that descriptor and imports the expected KB/MC JSONL sources without changing target-selection behavior.
- [ ] The backup and restore regression specs execute in hosted CI; collection-only output is not execution evidence.

## Out of Scope

Physical defrag execution, scheduler changes, volume mounts, lifecycle exclusion, snapshot/recovery mechanics and closure of the parent issue.

## Avoided Traps

A plausible path, localhost endpoint or matching collection UUID cannot supply physical-storage authority. This repair does not manufacture such authority or remove maintenance capability.

## Sweep Evidence

Latest-20 open issues, exact open/closed backup-topology search and current A2A claims were checked immediately before filing. The overlap is the intentionally retained parent, #306; no separate delivery leaf was found. Own-assignment sweep found #306 and unrelated #48. Memory Core recall recovered the existing metadata decision and withdrawal of the broader retirement; current source and restore consumers were rechecked. No new Discussion decision is claimed.

Origin Session ID: f18d3aa0-4065-41ba-9e2f-04c6bc109d5f


## Timeline

- 2026-09-20T04:32:46Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-20T04:32:47Z @neo-gpt-emmy added the `bug` label
- 2026-09-20T04:32:47Z @neo-gpt-emmy added the `ai` label
- 2026-09-20T04:32:47Z @neo-gpt-emmy added the `testing` label
- 2026-09-20T04:32:47Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-20T04:35:25Z @neo-gpt-emmy cross-referenced by PR #397
- 2026-09-20T04:35:27Z @neo-gpt-emmy cross-referenced by #306
- 2026-09-21T10:49:50Z @neo-opus-grace cross-referenced by #201
- 2026-09-21T11:09:49Z @tobiu referenced in commit `881b2eb` - "Merge pull request #397 from neomjs/codex/306-backup-topology-observation

fix(backup): leave unobserved physical topology unspecified (#398)"
- 2026-09-21T11:09:49Z @tobiu closed this issue

