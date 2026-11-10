---
id: 1565
title: 'Neural Link: each of its 60 tool lines says what the tool does and when to use it'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-11-10T02:33:23Z'
updatedAt: '2026-11-10T02:33:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1565'
author: neo-fable-mnemosyne
commentsCount: 0
parentIssue: 1562
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1563 ToolService lists one line per tool: summary is its only source, title and x-neo-tool-summary go'
blocking:
  - '[ ] 1569 Read the call counts four weeks after the trigger lines land: did saying when move use?'
---
# Neural Link: each of its 60 tool lines says what the tool does and when to use it

## Context

A server leaf of #1562 (graduated from neomjs/neo#20680, OQ-7). Blocked by the `ToolService` leaf, which makes `summary` the only source of a listed line.

## The Problem

The Neural Link has the most tools and the fewest lines: 46 of its 60 summaries repeat the operation id, and no operation has a curated line. A seat that wants to inspect or drive a running app has to tell `query_component`, `find_instances`, `get_component_tree`, `inspect_component_render_tree` and `query_vdom` apart by name.

## The Architectural Reality

- `ai/mcp/server/neural-link/openapi.yaml`: 60 operations (32 read, 23 write-locked, 5 admin).
- 17 of them declare no JSON success schema and are listed without an `outputSchema` (`list_transactions` among them); that is unchanged by this leaf.
- #143 owns the Neural Link's curated surface; this leaf writes lines and curates nothing.

## The Fix

Rewrite each `summary` in the form `<what>; use when <trigger>`, at most 120 bytes, written by a seat that drives apps through the Neural Link. The lines of look-alike tools say which one answers which question (a component by id, instances by class, the tree, the rendered tree, the vdom).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `summary` of each Neural Link operation | #1562, "The contract of the line" | `<what>; use when <trigger>`, at most 120 bytes | none | the OpenAPI file | the byte test of the `ToolService` leaf; the stranger read below |

## Acceptance Criteria

- [ ] All 60 summaries have the form; none repeats the operation id; the byte test is green.
- [ ] A second seat reads ten lines chosen by the reviewer, the five look-alike inspection tools among them, without the descriptions, and says for each when it would call the tool; a line it cannot place is rewritten.
- [ ] The compatibility gate: this leaf changes no `inputSchema` and no `outputSchema`; the listed `outputSchema` of every tool is byte-identical before and after, and the 17 tools without one still have none.

## Out of Scope

The curated surface and locking (#143). Descriptions and the handbook. The catalog.

## Related

#1562 (parent), the `ToolService` leaf (blocks this one), #143.

unowned-rationale: for a seat that drives apps through the Neural Link. No `good first issue`: the lines need use of the tools against a running app.

Live latest-open + closed sweep and A2A, MC and own-assignment sweeps: as on #1562 at 2026-11-10T02:27Z; this leaf is filed with it and has no equivalent.

Origin Session ID: fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

Retrieval Hint: "Neural Link tool trigger lines summary what and when"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

## Timeline

- 2026-11-10T02:33:24Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-10T02:33:24Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-10T02:33:25Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-11-10T02:33:25Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-10T02:34:37Z @neo-fable-mnemosyne added parent issue #1562
- 2026-11-10T02:35:03Z @neo-fable-mnemosyne marked this issue as being blocked by #1563
- 2026-11-10T02:35:11Z @neo-fable-mnemosyne marked this issue as blocking #1569

