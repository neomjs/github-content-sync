---
id: 199
title: Move Dream into a Cloud-owned domain
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
  - architecture
  - performance
  - agent-os
assignees: []
createdAt: '2026-08-27T15:06:41Z'
updatedAt: '2026-08-28T22:22:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/199'
author: neo-gpt-emmy
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 197 Establish deploy/host and independent deploy/cloud packages'
blocking:
  - '[ ] 202 Replace Engine-era learning folders with one Brain journey'
closedAt: '2026-08-28T22:22:05Z'
---
# Move Dream into a Cloud-owned domain

## Context

Brain #193 owns domain refactoring under parent Epic #189. Dream behavior currently centers on `ai/daemons/orchestrator/services/DreamService.mjs` (1,391 LOC, 24 methods, 28 direct dependencies) and is scattered across orchestrator scheduling, graph synthesis, ingestion, memory, and provider readiness.

## The Problem

Dream is treated as an orchestrator helper despite owning durable REM processing, graph digestion, concept ingestion, garbage collection, and Golden Path synthesis. Host scheduling and Cloud domain behavior are coupled in one large singleton, making placement, testing, and ownership opaque.

## The Architectural Reality

Durable memory, graph, ingestion, and Dream behavior are Cloud-owned. Host Edge may schedule or request a Dream cycle through a contract, but must not import Cloud internals.

Pre-Flight (structural full): considered root `src/dream`, Host orchestration, and `deploy/cloud/src/dream`. Because Dream reads/writes durable memory/graph/ingestion authority, chose `deploy/cloud/src/dream`. The orchestrator keeps only a thin trigger/contract adapter. This accommodates future Dream capabilities without growing a service junk drawer and reduces future Cloud extraction friction. ADR 0040's plane ownership binds; the architecture map needs the new domain.

## The Fix

Move Dream behavior into the Cloud dream domain. Split orchestration trigger/scheduling from domain application flow. Replace direct cross-domain imports with small explicit capabilities owned by memory, graph, and ingestion domains. Delete duplicate scheduling/Golden-Path wrappers and tests that pin private implementation.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Dream domain | Cloud durable-intelligence boundary | owns REM/graph/evolution flow | failure propagates to caller | Dream guide | domain tests |
| Host trigger | Host scheduler | invokes one contract | no internal imports | Host runbook | static/runtime boundary |
| memory/graph capabilities | owning domains | narrow callable contracts | fail loud | domain docs | focused tests |

## Decision Record impact

Aligned with ADR 0040's Cloud durable-state ownership.

## Acceptance Criteria

- [ ] Dream production behavior lives under `deploy/cloud/src/dream/**`; the legacy Dream service path is absent.
- [ ] Host orchestration contains scheduling/trigger logic only and imports no Dream storage, graph, or ingestion internals.
- [ ] Dream's application service is below 1,500 LOC because responsibilities were removed, not split mechanically.
- [ ] REM, graph digestion, concept ingestion, garbage collection, and Golden Path flows have explicit domain owners.
- [ ] Tests assert Dream public behavior and boundary failures, not singleton internals or method-by-method mutation.
- [ ] Obsolete Dream/scheduling/Golden-Path wrappers and their tests are deleted.
- [ ] No new facade, registry, ledger, or orchestration layer is introduced.

## Out of Scope

Embedding-path simplification; general Memory Core decomposition; new Dream features.

## Avoided Traps

- Moving the existing class unchanged.
- Extracting 24 one-method helpers.
- Keeping Host and Cloud coupled through a “shared” service locator.

## Related

Parent: #193 · Goal: #189 · Related: #71

Origin Session ID: d39e8182-295f-418a-82cd-a96be9c08e4f

Retrieval Hint: `DreamService Cloud dream domain thin Host orchestrator REM graph ingestion Golden Path`


## Timeline

- 2026-08-27T15:06:43Z @neo-gpt-emmy added the `enhancement` label
- 2026-08-27T15:06:43Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T15:06:44Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-27T15:06:44Z @neo-gpt-emmy added the `architecture` label
- 2026-08-27T15:06:44Z @neo-gpt-emmy added the `performance` label
- 2026-08-27T15:06:44Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-28T11:41:19Z @neo-opus-vega cross-referenced by PR #205
- 2026-08-28T22:17:05Z @neo-gpt-emmy cross-referenced by #215
### @neo-gpt-emmy - 2026-08-28T22:22:05Z

Superseded by #215. REM digestion is an Evolution use case scheduled by Orchestration, not a Cloud-owned Dream domain. The corrective leaf preserves one shared source contract for both executable profiles.

- 2026-08-28T22:22:05Z @neo-gpt-emmy closed this issue

