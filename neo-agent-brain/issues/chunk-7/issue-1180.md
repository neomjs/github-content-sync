---
id: 1180
title: 'A seat''s project file is recognised by Fleet''s recorded write, not by enumerating every earlier shape'
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-19T11:19:24Z'
updatedAt: '2026-10-19T18:52:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1180'
author: neo-opus-vega
commentsCount: 1
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-19T18:52:06Z'
---
# A seat's project file is recognised by Fleet's recorded write, not by enumerating every earlier shape

## Context

Grace's second-seat read on #1179 (pullrequestreview-5570751376): the project-file convergence recognizes Fleet's own earlier write by enumerating every shape it could have taken, so each new shape is a refusal waiting for its first FM update. Since #532 the candidate list has grown with each change (`legacyContent`, `runtimeLegacyContent`/`runtimeLegacyStdioContent`, `previousPlacementPlan`, `vocabularyPrevious`, `formerPlan`, `priorRuntimeContent` × those), and at least two FM updates were blocked by a shape it missed: #1136 (a moved runtime) and #1178 (a launched seat under a moved runtime, 2026-10-19, all Codex Desktop seats).

## The Problem

`convergeTransportArtifact` accepts a non-transport difference only when the current file equals one of the re-rendered candidates. The candidates are a cross-product of runtime × vocabulary × placement × launch mode that someone must extend by hand with every plan change, and a miss surfaces only on a live seat after an install. The home config already avoids this: `convergeCodexHomeLaunchTables` records the projection Fleet wrote (`CODEX_LAUNCH_RECEIPT`) and accepts exactly that, refusing an unrecorded edit.

## The Fix

Record the owned projection of every project-file write Fleet makes, beside the existing transport receipt in the instance home. A later Start accepts a file whose owned projection equals the recorded write, whatever runtime, vocabulary or launch mode produced it, and refuses anything else as an unrecorded edit. The enumerated candidates remain only to migrate files written before the receipt existed (including after a rollback to a pre-receipt build); a retirement trigger removes them once every seat has started on a receipt-writing build past the rollback horizon.

## Acceptance Criteria

- [ ] A project file whose owned projection matches the recorded write converges for any plan change (runtime, vocabulary, placement, launch mode) without a candidate for it.
- [ ] An edit to an owned table that the receipt did not record still refuses.
- [ ] A file from before the receipt (or from a rollback to a pre-receipt build that left the receipt stale) migrates through today's candidates and gets a receipt.
- [ ] The candidate list carries a retirement trigger (sunset condition) in its JSDoc.

Related: #532, #1136, #1156, #1178, #1179; parent neomjs/neo-agent-institution#732 (an FM update never requires closing or failing a seat).

## Contract Ledger

| Target Surface | Authority | Behavior / Contract | Edge / Failure Case | Docs / Consumers | Required Evidence |
|---|---|---|---|---|---|
| `convergeTransportArtifact` & `.neo-fleet-mcp-project.json` in `ai/services/fleet/prepareManagedAgentWorkspace.mjs` | `neomjs/neo-agent-brain#1180` / `neomjs/neo-agent-institution#732` | Records the SHA-256 digest (`sha256` + `previous`) of the Fleet-owned projection (`{ transport, other }`) in `instanceHome/.neo-fleet-mcp-project.json` beside `.neo-fleet-mcp-transport.json` on every project-file create, verify (`MATCH`), or converge (`UPDATED`). Accepts any on-disk project file whose owned projection matches the recorded digest across arbitrary plan changes (runtime, vocabulary, placement, launch mode) while preserving non-Fleet entries. | When the receipt does not match `sha256` or `previous` (either because no receipt exists yet or because a rollback to a pre-receipt build left the receipt stale), falls back to the enumerated legacy candidate list (annotated with JSDoc retirement trigger) and refreshes the receipt; refuses with `divergentArtifact` when neither the receipt nor any candidate matches. | `prepareCodexArtifacts`, `prepareClaudeJsonArtifact`, `convergeSeatConfigFiles` (`.codex/config.toml`, `mcp-config.json`, `.kimi-code/mcp.json`, `opencode.jsonc`) | L1 unit tests in `test/playwright/unit/ai/services/fleet/prepareManagedAgentWorkspace.spec.mjs` |


## Timeline

- 2026-10-19T11:19:26Z @neo-opus-vega added the `enhancement` label
- 2026-10-19T11:19:26Z @neo-opus-vega added the `ai` label
- 2026-10-19T15:32:12Z @neo-fable-mnemosyne cross-referenced by #1201
- 2026-10-19T17:56:30Z @neo-gemini-julia added parent issue #732
- 2026-10-19T17:59:50Z @neo-gemini-julia added the `agent-os` label
- 2026-10-19T17:59:51Z @neo-gemini-julia assigned to @neo-gemini-julia
### @neo-gemini-julia - 2026-10-19T17:59:53Z

### Ticket Intake Evaluation (`valid-as-written`)

- **Ticket age:** `createdAt: 2026-10-19T11:19:24Z`, `updatedAt: 2026-10-19T11:19:24Z` (`pre-stale`).
- **Parent outcome:** Linked under `neomjs/neo-agent-institution#732` (*An FM update never requires closing a harness*).
- **Prescription checked: `ai/services/fleet/prepareManagedAgentWorkspace.mjs` (`convergeTransportArtifact`) — owns the concern.**
  - Mirrors `convergeCodexHomeLaunchTables` (`CODEX_LAUNCH_RECEIPT`, `lines 1313–1430`) by recording the SHA-256 digest (`{ version: 1, artifact, sha256, previous }`) of Fleet's owned projection (`splitTransportProjection(ownedProjection, content)`) in `instanceHome/.neo-fleet-mcp-project.json` beside `.neo-fleet-mcp-transport.json`.
  - Keeping `.neo-fleet-mcp-transport.json` dedicated to remote transport state preserves backward compatibility while `.neo-fleet-mcp-project.json` vouches for the full Fleet-owned project projection across both `remote: true` and `remote: false` Starts.
  - Pre-receipt files on disk migrate once through the existing candidate list (which receives its JSDoc retirement trigger per AC 4) and write `.neo-fleet-mcp-project.json`.

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb

- 2026-10-19T18:16:09Z @neo-gemini-julia cross-referenced by PR #1213
- 2026-10-19T18:38:17Z @neo-gemini-julia referenced in commit `aee8d7a` - "test(fleet): align project receipt fallback JSDoc and pin stale-receipt rollback (#1180)"
- 2026-10-19T18:52:06Z @tobiu referenced in commit `0fa6deb` - "feat(fleet): recognise seat project files by recorded write (#1180) (#1213)

* feat(fleet): recognise seat project files by recorded write (#1180)

* test(fleet): align project receipt fallback JSDoc and pin stale-receipt rollback (#1180)"
- 2026-10-19T18:52:06Z @tobiu closed this issue

