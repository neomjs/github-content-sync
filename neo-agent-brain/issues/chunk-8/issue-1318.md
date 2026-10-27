---
id: 1318
title: Keep diary field meanings in add_memory's compact description
state: CLOSED
labels:
  - bug
  - documentation
  - ai
  - model-experience
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-27T14:37:35Z'
updatedAt: '2026-10-27T15:14:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1318'
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
closedAt: '2026-10-27T15:12:30Z'
---
# Keep diary field meanings in add_memory's compact description

## Context

The compact `add_memory` listing says to write a diary but leaves the required `thought` and `response` fields unexplained. The full handbook already defines an authored retrospective. The operator requested that this existing meaning remain visible at the compact call boundary.

Design authority: `#955` defines `thought` as an authored decision retrospective and `response` as outcomes and continuation. This is a bounded follow-up to that completed contract.

## The Problem

Memory Core enables compact descriptions and compact schemas. Property descriptions are omitted from the compact listing, so the handbook's field meanings disappear unless the caller separately requests it.

## The Architectural Reality

- `ai/mcp/server/memory-core/openapi.yaml`, `POST /memories`, owns `add_memory.x-neo-tool-summary` and the full description.
- `ai/mcp/ToolService.mjs` selects that summary and caps it at 120 characters for Memory Core.
- Existing `McpServerListToolsSmoke.spec.mjs` requires the short description to retain the distinction between immediate acceptance and semantic recall after embedding.
- The scoped structure map confirms `ai/mcp/server/memory-core` owns this surface.

## The Fix

Replace the short summary with a positive, newly authored diary instruction that maps decisions/evidence to `thought`, outcomes/next steps to `response`, and retains the semantic-recall caveat within 120 characters.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Compact `add_memory` description | OpenAPI operation and `#955` | Carries the diary field meanings within the existing cap | Full handbook retains its detailed guidance | `x-neo-tool-summary` | Inspect generated listing and existing compact-description checks |
| Request and persistence contracts | Existing `AddMemoryRequest` and service | Unchanged | Existing validation and visibility semantics | Existing handbook | One-value YAML diff and OpenAPI parity |

## Acceptance Criteria

- [x] Generated compact listing contains the fresh-diary instruction, field meanings, and semantic/embedding caveat without truncation.
- [x] Existing description-limit/listing checks and OpenAPI parity pass.
- [x] The diff changes only the short description; field names, types, requiredness, storage, visibility and full handbook are unchanged.

## Decision Record impact

None; clarifies the existing content contract established by `#955`.

## Out of Scope

Runtime deployment/restart, client refresh, memory migration, field renaming, additional policies, or claims about provider behavior.

## Related and intake evidence

Related: #955. MC prior-art anchor: `059323f1-d04d-48a5-93c0-6022367cd2ec`; its original diary contract and the live handbook agree. The current operator audit is `fcd1477d-52cd-40cb-be3e-41af1faf74f5`.

Live latest-open sweep: latest 20 Brain issues checked on 2026-10-27 immediately before creation; no overlapping follow-up. Open/closed diary and compact-description searches found completed `#955`, which was read in full. Latest 30 A2A messages across all read states contain no competing claim. Own-assignment sweep: six open issues; the MC transport/diagnostic work is distinct. KB independently returned `#955` as the content authority.

Origin Session ID: 8006c2c0-4928-4fa9-83d1-c02e697b05f4
Retrieval Hint: `add_memory compact diary field meanings semantic embedding`



## Timeline

- 2026-10-27T14:37:35Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-27T14:37:37Z @neo-gpt-emmy added the `bug` label
- 2026-10-27T14:37:37Z @neo-gpt-emmy added the `documentation` label
- 2026-10-27T14:37:37Z @neo-gpt-emmy added the `ai` label
- 2026-10-27T14:37:37Z @neo-gpt-emmy added the `model-experience` label
- 2026-10-27T14:46:04Z @neo-gpt-emmy cross-referenced by PR #1319
- 2026-10-27T15:12:30Z @tobiu referenced in commit `436aec6` - "docs(memory-core): clarify compact diary fields (#1318) (#1319)"
- 2026-10-27T15:12:30Z @tobiu closed this issue

