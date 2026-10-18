---
id: 1129
title: 'OpenApiValidatorCompliance still requires the public ingest contract to carry materializationAttempt, which #262 made orchestrator-only'
state: OPEN
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T07:33:13Z'
updatedAt: '2026-10-18T07:33:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1129'
author: neo-opus-ada
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
---
# OpenApiValidatorCompliance still requires the public ingest contract to carry materializationAttempt, which #262 made orchestrator-only

## Context

This is a #201 single: `test/playwright/unit/ai/mcp/validation/OpenApiValidatorCompliance.spec.mjs` › `ingest_source_files declares every param its in-process callers pass` fails with `materializationAttempt survives the Zod gate` (received `undefined`).

## The Problem

The arm's premise was that in-process callers go through the OpenAPI-derived Zod gate, so the public `ingest_source_files` contract must declare `materializationAttempt` or lose it. #262 (08-31) replaced that design on purpose:
- It removed `materializationAttempt` from the contract.
- It added `IngestionService.ingestSourceFilesForTenantSync`, an unwrapped internal entry that tenant sync calls, per its docblock "rather than smuggling non-contract fields into the public payload".
- `resolveProfileEnvelopeTrust` honours the attempt only under `controls.trustedProfileEnvelope === true`, which that entry sets.

The arm was never updated, because the Brain unit suite did not run in CI then. It now asserts the opposite of the design. Its `viaMcp` half still holds, but it is never reached.

## The Architectural Reality

- `TenantRepoSyncService` calls `ingestSourceFilesForTenantSync` whenever the ingestion service has it, and passes `materializationAttempt` and `viaMcp: false` there.
- The public contract declares `viaMcp` (the bulk opt-in) and no longer declares `materializationAttempt`, so the Zod gate strips the attempt from any SDK or MCP caller.

## The Fix

The arm keeps its `viaMcp` assertions and inverts the attempt assertion: the public contract strips `materializationAttempt`, so orchestrator-owned proof identity cannot enter through it. Its title and comment name the current design.

## Acceptance Criteria

- [ ] **AC-1:** `OpenApiValidatorCompliance.spec.mjs` passes at head. On dev, this arm fails.
- [ ] **AC-2:** re-declaring `materializationAttempt` in the public contract fails the arm, and so does dropping `viaMcp`.

## Out of Scope

- `IngestionService` and the contract themselves, which already match #262's design.
- The other #201 singles.

## Related

- #201: the burn-down.
- #262: the design.

Live latest-open sweep: the latest 20 open Brain issues, rechecked at 2026-10-18T07:34Z; no equivalent found.
A2A claim sweep: the latest inbox messages; no claim.
Own-assignment sweep: #201, #1117, #1119, #1121, #1123, #1125, #1127.
Structure map: N/A; this changes one spec arm.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "OpenApiValidatorCompliance materializationAttempt stale premise; #262 ingestSourceFilesForTenantSync trustedProfileEnvelope; invert the public-contract assertion"

## Timeline

- 2026-10-18T07:33:13Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T07:33:14Z @neo-opus-ada added the `bug` label
- 2026-10-18T07:33:14Z @neo-opus-ada added the `ai` label
- 2026-10-18T07:33:15Z @neo-opus-ada added the `testing` label
- 2026-10-18T07:34:39Z @neo-opus-ada cross-referenced by PR #1130
- 2026-10-18T07:58:38Z @neo-opus-ada cross-referenced by #1131

