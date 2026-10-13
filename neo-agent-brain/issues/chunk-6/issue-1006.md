---
id: 1006
title: Preserve WAL-backed turns during miniSummary backfill
state: OPEN
labels:
  - bug
  - ai
  - regression
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-13T04:08:52Z'
updatedAt: '2026-10-13T04:16:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1006'
author: neo-gpt-sophie
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
blocking:
  - '[ ] 1005 miniSummary backfill archives turns before their embed lands'
---
# Preserve WAL-backed turns during miniSummary backfill

## Context

Source delivery for the recency incident in #1005. The incident stays open for deployed verification and historical reconciliation; this leaf delivers the preventive guard and the bounded operator repair capability. [Vega's ownership review](https://github.com/neomjs/neo-agent-brain/issues/1005#issuecomment-6137980212) separates those delivery authorities.

## The Problem

At deployed revision `7ca66028`, a graph-projected accepted turn can reach `backfillMiniSummaries` before its Chroma embed. The Chroma-only read classifies the turn as `no-content` and archives it despite its durable WAL payload. The later embed does not restore recency. The controlled SQLite/WAL reproduction and incident chronology are retained on #1005; this is independent of generation-timeout policy.

## The Architectural Reality

Design authority: `MemoryService.addMemory` accepts the durable WAL write while graph projection and embedding reconcile asynchronously. `readPendingWalRecords` already supplies recoverable payloads before embedding. A content miss in one derived store is insufficient evidence of durable absence.

The existing `ai/services/memory-core/MemoryService.mjs` owns content admission and archive reconciliation. The existing `ai/scripts/lifecycle/backfill-memory-summaries.mjs` owns manual execution under its heavy-maintenance lease. `QueryRecentTurns.spec.mjs` owns accepted-write/recall regression coverage; no new file, service, config leaf or scheduler is needed. The Brain structure-map command passed.

## The Fix

Snapshot pending WAL payloads before reading Chroma, preserving the handoff if an embed marker advances between reads. Defer the batch on unavailable evidence. Expose exact-ID, tenant-and-agent-bound, dry-run-default reconciliation through the existing lifecycle entry, with positive content, archive reason, durable identity tombstone and unchanged-row checks before applying repairs.

## Contract Ledger

| Surface | Authority | Behavior | Refusal / fallback | Docs and evidence |
| --- | --- | --- | --- | --- |
| Backfill content admission | accepted-write WAL contract | use durable payload across delayed embedding | unavailable reads defer without archiving or inference | service JSDoc; controlled handoff and read-failure tests |
| Recency | existing tenant/identity archive boundary | accepted turn remains recallable before and after embed | intentional archives remain excluded | actual SQLite recency assertions |
| Historical reconciliation | matching graph and payload owners, exact automatic reason | `reconcileNoContentArchives` inspects or repairs 1–100 exact IDs | no positive payload, tombstones, wrong owner or changed row refuse; no retry reset | service JSDoc; mixed-owner, late-archive, idempotence and preservation tests |
| Operator entry | existing lifecycle lease, explicit local scope | `--reconcile-no-content`, `--user-id`, `--agent-identity`, optional `--apply` | no args keeps normal backfill; local scope selection is not remote authentication | entry JSDoc; parser and actual request-context routing tests |
| Container WAL placement | existing shared SQLite volume | MC and orchestrator resolve the same WAL directory; MC remains the sole in-process drainer | no second drainer or private-root fallback | Compose mount test and resolved-leaf diagnostic |

## Acceptance Criteria

- [ ] AC-1 A graph-projected, WAL-backed turn with an absent Chroma row survives backfill and remains visible through recency before and after embedding.
- [ ] AC-2 Crossing the WAL/embed reconciliation boundary during reads cannot produce a false no-content decision. Read failures defer safely; the test uses controlled barriers, not elapsed sleeps.
- [ ] AC-3 Truly missing content retains its bounded handling, and ordinary summary, tenant isolation and intentional-archive controls remain correct. The generation-timeout policy is unchanged.
- [ ] AC-4 The exact-ID reconciliation restores only content-positive false no-content archives, is idempotent, and preserves raw content, unrelated graph properties, intentional identity tombstones and genuinely missing rows.
- [ ] AC-5 Documentation explains operator scope, dry-run limits and the separate deployed verification/reconciliation owned by #1005; source tests do not claim live recovery.
- [ ] AC-6 Base and canonical-local Compose give the orchestrator backfill consumer the same shared-volume WAL path as Memory Core without enabling a second in-process drainer. Preserve standalone parity placement.

The deployment preflight found a second necessary edge: the running orchestrator resolves `/app/.neo-ai-data/memory-wal`, while MC resolves `/app/.neo-ai-data/sqlite/memory-wal`. Both already mount the shared SQLite volume, but base Compose places the WAL only on MC. Bind the existing reader to that channel; service-level tests alone cannot prove cross-process placement.

## Decision Record impact

None: restore the accepted-write contract without changing privacy, ownership, scheduling or model configuration. ADR-0019's resolved-provider boundary remains unchanged.

## Avoided Traps

No age threshold, blocking embed on acceptance, blanket unarchive, retry-budget reset or new maintenance service. Either durable source can establish positive repair evidence; missing or unreadable evidence cannot. The existing WAL parser's skipped malformed entries are not certified absence.

## Out of Scope

Live-plane deployment and incident-row recovery (retained by #1005), generation-timeout restoration in #81, provider/window tuning and archive-policy redesign.

## Admission evidence

This replaces the source ACs on the existing incident; it does not create another implementation lane. Sophie retains both scopes. Fresh latest-20 open, all-state A2A last-30 and own-assignment sweeps found only the existing incident on this surface; the other own ticket was unrelated. Memory/KB searches recovered the distinct retry-exhaustion policy and the accepted-write race evidence, with no competing source owner.

Related: #1005, #81, #125.

Origin Session ID: bcad2075-59c0-49e7-9676-628af19c3033
Retrieval Hint: miniSummary no-content archive before embed; WAL snapshot order; exact-owner historical reconciliation.

Sophie (GPT-6, Codex).


## Timeline

- 2026-10-13T04:08:53Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-13T04:08:55Z @neo-gpt-sophie added the `bug` label
- 2026-10-13T04:08:55Z @neo-gpt-sophie added the `ai` label
- 2026-10-13T04:08:55Z @neo-gpt-sophie added the `regression` label
- 2026-10-13T04:09:12Z @neo-gpt-sophie marked this issue as blocking #1005
- 2026-10-13T04:10:00Z @neo-gpt-sophie cross-referenced by #1005
- 2026-10-13T04:11:04Z @neo-gpt-sophie cross-referenced by PR #1007
- 2026-10-13T04:19:34Z @neo-gpt-sophie referenced in commit `0525fb2` - "feat(memory): bind backfill to the shared WAL volume (#1006)"
- 2026-10-13T04:51:43Z @neo-gpt-sophie referenced in commit `d7addcb` - "docs(memory): clarify strict WAL read boundary (#1006)"

