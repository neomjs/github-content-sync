---
id: 451
title: Sandman handoff reader reads the wrong config provider
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-24T11:58:23Z'
updatedAt: '2026-09-24T13:13:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/451'
author: neo-gpt-emmy
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
closedAt: '2026-09-24T13:13:06Z'
---
# Sandman handoff reader reads the wrong config provider

## Context

Clio observed the deployed `get_sandman_handoff` reader returning `handoff-path-unconfigured` while the configured handoff file exists. Independent diagnosis is recorded in [D#19151](https://github.com/neomjs/neo/discussions/19151#discussioncomment-18580534). This leaf repairs the existing reader; it does not graduate that Discussion's proposed selectors or Fleet/3D views.

## The Problem

At deployed Brain `b99ea11c213402199405c1793c86f91d1de155d7`, the MCP response is `{content: null, path: null, stale: true, reason: 'handoff-path-unconfigured'}`. A read-only process in that same container, with Neo/core initialized before importing the configuration modules, resolves:

```json
{"rootHandoff":null,"memoryHandoff":"/app/.neo-ai-data/handoff/sandman_handoff.md"}
```

The reader therefore cannot serve an existing artifact. This does not establish whether the Golden Path generator is healthy or its recommendation current.

## The Architectural Reality

- `ai/mcp/server/memory-core/toolService.mjs` imports Tier-1 `AiConfig` and the Memory Core `mcConfig`, but `readSandmanHandoffTool` supplies `AiConfig.handoffFilePath`.
- The production/test leaves and active `handoffFilePath` formula belong to `ai/mcp/server/memory-core/configBase.mjs` (`:567–579,925` at the measured deployment).
- `GoldenPathSynthesizer` already consumes `Memory_Config`; its writer and this reader currently use different Provider levels.
- `sandmanHandoffStore.mjs` correctly distinguishes an unconfigured path, missing file, oversized/read-failed content and age. It should continue doing so.

## The Fix

Bind the existing MCP handler to the owning Memory Core Provider at the use site. Preserve the current tool arguments, full-handoff output and store behavior. Add a focused binding regression witness that would fail when the Tier-1 read is restored; a helper-only test supplied an already-correct path cannot catch this defect.

## Contract Ledger

| Surface | Authority | Delivered behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `get_sandman_handoff` handler path | ADR-0019; Memory Core `handoffFilePath` formula | reads the same resolved owner as the writer, retaining prod/test selection by construction | existing store reasons remain unchanged | handler comment/JSDoc | configured owning-provider fixture returns its file; root-provider mutation fails |
| freshness/size/error envelope | `sandmanHandoffStore.mjs` | unchanged | unavailable remains unavailable; stale content is never freshened | existing helper contract | existing helper tests plus binding witness |

## Acceptance Criteria

- [ ] The handler reads the resolved Memory Core handoff path without a new Tier-1 alias, env read, caller-supplied path, or secondary resolver.
- [ ] An isolated configured-path witness exercises the handler binding, returns the expected file/content metadata, and fails against the former Tier-1 binding. Test setup uses provider construction rather than shared-singleton mutation.
- [ ] Existing handoff age, missing/read-failure and size-limit behavior remains intact.
- [ ] Test evidence distinguishes a successful read from generator recovery and route freshness.

## Out of Scope

GP generation/scheduling, deployment changes, new section selectors, structured-route APIs, Fleet Manager panes, and graph visualization.

## Avoided Traps

An env-present file is not evidence that the handler read its owning Provider. Do not repair the mismatch by duplicating the leaf into Tier-1 or parsing the stale checkout copy. A read fix must not report the expired route as fresh.

## Decision Record impact

Aligned-with ADR-0019: consumers read resolved leaves from their owning Provider.

## Related and admission

Related: neomjs/neo#15599 · neomjs/neo#15604 · D#19151 · Clio defect fingerprint `54b1380f`.

Live latest-open sweep: latest 20 Brain issues at 2026-09-24 11:57Z plus exact Sandman/tool searches; no equivalent open repair. Latest 30 all-state A2A messages include Clio's explicit handoff to Emmy and no competing claim. MC rationale query returned unrelated initialization records; live source and the container discriminator supply the evidence. Own-assignment sweep: #426, #306, #48; none owns this binding. Brain structure-map executed successfully; existing MCP handler and existing handoff helper/test family own the change. No new module placement is proposed.

Origin Session ID: 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2

Emmy 🪡 (GPT-6 Astra, Codex).


## Timeline

- 2026-09-24T11:58:23Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-24T11:58:24Z @neo-gpt-emmy added the `bug` label
- 2026-09-24T11:58:25Z @neo-gpt-emmy added the `ai` label
- 2026-09-24T11:58:25Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-24T12:11:24Z @neo-gpt-emmy cross-referenced by PR #453
- 2026-09-24T12:54:27Z @neo-gpt-emmy referenced in commit `7c9c0d4` - "ci(memory-core): run the Sandman handoff regression witness (#451)"
- 2026-09-24T13:13:07Z @tobiu referenced in commit `353deb1` - "Merge pull request #453 from neomjs/codex/451-sandman-reader

fix(memory-core): read the owning handoff config (#451)"
- 2026-09-24T13:13:07Z @tobiu closed this issue
- 2026-09-24T14:05:32Z @neo-opus-vega cross-referenced by #442

