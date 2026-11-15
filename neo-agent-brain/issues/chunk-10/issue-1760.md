---
id: 1760
title: Retire the legacy issue-edge upgrade CLI once every store has run it
state: OPEN
labels:
  - enhancement
  - ai
  - refactoring
  - agent-os
  - tech-debt
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-11-15T10:38:27Z'
updatedAt: '2026-11-15T10:38:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1760'
author: neo-fable-mnemosyne
commentsCount: 0
parentIssue: 122
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1729 Retire one-shot IssueIngestor legacy anchor weight reset after #1527 deploy'
blocking: []
---
# Retire the legacy issue-edge upgrade CLI once every store has run it

## Context

PR #1757 takes the one-time weight normalization out of `IssueIngestor.mjs` and keeps its logic as a step an operator runs at upgrade: `ai/scripts/migrations/normalizeLegacyIssueEdgeWeights.mjs`, with a spec and one section in two deployment guides. That step is one-shot logic too. A store runs it once, and its `SchemaVersion` marker turns every later run into a no-op. `pull-request` §6.2 asks for one retirement ticket per one-shot logic. This is that ticket, filed by the reviewer of PR #1757 as the follow-up of its approval.

## The Problem

The step exists for graph stores that the old write-time anchor wrote and that did not finish an ingest pass while the inline pass was on `dev` (from the merge of PR #1728 at 2026-11-14T23:28:58Z UTC, `06e16078be`, to the merge of PR #1757). Nobody has counted those stores. `deploy/cloud/docker-compose.yml` turns corpus projection on in every deployment of that compose (lines 513 to 515 at `6cfdf63e`), so the count can be above zero. The one store with a receipt is the canonical local plane (`#1729`, comment 6890553296).

Until the count is known, 869 lines of one-time code (417 in the CLI, 452 in its spec) and two guide sections stay in the tree, and nothing says when they leave. If the count is zero they can go at once.

## The Architectural Reality

Read at the head of PR #1757, `fa5278eacc`:

- `ai/scripts/migrations/normalizeLegacyIssueEdgeWeights.mjs`: reads `Nodes` and `Edges` through `better-sqlite3`, rewrites `Edges.data` for the edges it normalizes, and stamps `SchemaVersion.id = 'issue_ingestor_legacy_anchor_reset_v1'`. A store that carries the marker answers `status: "already-normalized"` with no write. The default is a dry run.
- `test/playwright/unit/ai/scripts/migrations/normalizeLegacyIssueEdgeWeights.spec.mjs`: four tests, each of which starts from the CLI.
- `learn/agentos/cloud-deployment/Configuration.md` and `learn/agentos/DeploymentCookbook.md`: one section each, titled "Persistent graph-store upgrade: legacy issue-edge weights".
- `ai/graph/issueEdgeWeightSchema.mjs` stays. It holds the decay exemptions and the read-time issue emphasis, and `GraphService.mjs`, `computedGoldenPathRouting.mjs`, `issueFocusSections.mjs` and `IssueIngestor.mjs` consume it. One branch in it serves the CLI alone: `readInternalAuthorLogins()` accepts a direct `better-sqlite3` handle (`typeof graphOrDb?.prepare === 'function'`). Its other caller, `GoldenPathSynthesizer.mjs:1252`, passes the graph service.
- The step can run beside live services: the `edge_update` trigger (`ai/graph/storage/SQLite.mjs:192`) writes each change to `GraphLog`, and every process drops its cached copy of a changed edge in `Database#syncCache()`.
- Structure map (`npm run ai:structure-map -- --files --loc` at `dev` `6cfdf63e`): `ai/scripts/migrations` is the folder of one-time store migrations; `normalizeGraphIdentities.mjs` is the sibling of this shape.

Decision Record impact: none.

## The Fix

1. **The population.** A comment here names every deployment, besides the canonical local plane, whose graph store was written with corpus projection on by a revision older than `06e16078be`. Or it says there is none.
2. **The runs.** Each named store runs the step once, dry run first, then `--apply`. Its output is recorded here.
3. **The deletion**, in one PR: the CLI, its spec, both guide sections, and the direct-handle branch of `readInternalAuthorLogins()`.
4. **The lost reasons**, in the same PR unless PR #1757 restores them before it merges. Three declarations lost their reasons when they moved into `ai/graph/issueEdgeWeightSchema.mjs`:
   - `PROTECTED_EDGE_TYPES`: the reason beside each of `ADVANCED_BY`, `ATTRIBUTED_TO`, `DELIVERED_TO`, `SENT_TO` and `SENT_BY` (for example: pruning `DELIVERED_TO` shows read messages as unread again).
   - `isMessageRecordEdge()`: message ids carry the prefix by construction.
   - `normalizeLabels()`: labels arrive from graph payloads and from frontmatter, and one normalizer keeps a difference in case or whitespace from routing a ticket that is not actionable back into computed work.

   The source of all three is `dev` at `6cfdf63e`: `ai/services/memory-core/GraphService.mjs` and `ai/services/graph/issueFocusSections.mjs`.

## Acceptance Criteria

- [ ] **AC-1 (a human step; owner: @tobiu).** A comment on this ticket lists the deployments of step 1, or states that the canonical local plane is the only one. No seat can produce this list: it is in no repository.
- [ ] **AC-2.** For each listed store a comment carries the JSON the step printed with `--apply`: `status` (`applied` or `already-normalized`), `candidateUpdates`, `updatedEdges` and `markerUpdatedAt`. An empty list meets this criterion without a comment. A listed store that will never run the step (it is decommissioned, or its owner declines) counts once a comment says so and names who decided.
- [ ] **AC-3.** One PR deletes the four items of step 3. After it, `git grep -n -e normalizeLegacyIssueEdgeWeights -e issue_ingestor_legacy_anchor_reset_v1` returns nothing, and the unit suite is green.
- [ ] **AC-4.** The reasons of step 4 stand beside their declarations in `ai/graph/issueEdgeWeightSchema.mjs`.

## Out of Scope

- Hardening the CLI. It reads its candidates before it opens its write transaction and then replaces the whole `data` document of an edge, where the inline pass read inside the transaction. A change another process makes to the same edge in between would be overwritten. A `json_set` on `$.properties.weight` would close that window. It is named here so that an operator who runs the step on a busy store knows it, and it is not worth a change to code this ticket deletes.
- The read-time emphasis and the decay exemptions themselves.
- A clause in `pull-request` §6.2 that says which stores a receipt must cover. It is worth a ticket in the Skills repository only if AC-1 names a second store.

## Avoided Traps

- Deleting now on the assumption that no other store exists. The first head of PR #1757 did that; the count is unknown.
- Keeping the step with no end condition. One-shot code that has no retirement ticket is not removed.

## Related

- Blocked by #1729, which closes with PR #1757. Parent: #122.
- PR #1757 and its Round 1 review: https://github.com/neomjs/neo-agent-brain/pull/1757#pullrequestreview-5909576218
- The same shape for another one-shot: `#1754`.

Live latest-open + closed sweep: checked the latest 20 open issues and a closed search (legacy anchor weight reset, retire one-shot, upgrade CLI; no closed hit) at 2026-11-15T10:36:49Z; the A2A claims of the last 60 minutes (none on this scope); one Memory Core query on the symptom (nothing equivalent); my own open tickets in this repository (`#122` is the parent, nothing overlaps). Backlog: 164 open (red). No owning ticket exists to fold this into: `#1729` closes with PR #1757, and the one-shot rule asks for one retirement ticket per one-shot logic.

Ownership: self-assigned. I asked the population question in the review of PR #1757 and carry it to the operator. `good first issue` and `help wanted` are left off because AC-1 depends on knowledge outside the repository. The deletion PR (AC-3, AC-4) is free for any seat once AC-1 and AC-2 are met.

Origin Session ID: 38b238a1-cb32-44a5-9a4d-4972cd7e2656

Retrieval Hint: `query_raw_memories("one-shot migration receipt covers one store; which deployments ran corpus projection")`

Authored by Mnemosyne (Claude Fable 5.2, Claude Code).

## Timeline

- 2026-11-15T10:38:27Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-11-15T10:38:28Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-15T10:38:29Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-15T10:38:29Z @neo-fable-mnemosyne added the `refactoring` label
- 2026-11-15T10:38:29Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-15T10:38:29Z @neo-fable-mnemosyne added the `tech-debt` label
- 2026-11-15T10:39:02Z @neo-fable-mnemosyne added parent issue #122
- 2026-11-15T10:39:20Z @neo-fable-mnemosyne marked this issue as being blocked by #1729
- 2026-11-15T10:39:52Z @neo-fable-mnemosyne cross-referenced by PR #1757

