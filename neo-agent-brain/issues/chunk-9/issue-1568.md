---
id: 1568
title: Each server's instructions carry a catalog of its tool families and the tools nobody finds
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-11-10T02:33:58Z'
updatedAt: '2026-11-10T02:33:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1568'
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
blocking: []
---
# Each server's instructions carry a catalog of its tool families and the tools nobody finds

## Context

The catalog leaf of #1562 (graduated from neomjs/neo#20680, option H). Blocked by the `ToolService` leaf; its text improves as the server leaves land, because it is generated from their lines.

## The Problem

A line on a tool helps once a seat looks at that tool. On a harness that defers schemas a seat first has to know the tool exists. `explore_lane_landscape` answers what a seat asks at every lane pickup and had no call in the 7 days to 2026-11-10; `pre_brief_session` is called because a workflow names it. The one text a deferring harness does show early is the server's `initialize` instructions: Claude Code holds them in context from attach, Antigravity writes them to one file per server.

## The Architectural Reality

- Each server's instructions today are one generic sentence about compacted listings and the handbook.
- Every OpenAPI operation has `tags`; they are the tool families.
- A catalog of every tool would cost about 4k tokens on a harness that always loads instructions; the Discussion rejected that.

## The Fix

Each server's instructions gain 10 to 15 lines: one per tool family with the two or three tools to start from, and the tools a seat does not find by name. The lines are generated from the operations' `tags` and `summary`, with one marker on an operation that the catalog should name, so the catalog has no text of its own.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `initialize` instructions of each MCP server | #1562, "The catalog" | the existing sentence plus 10 to 15 generated lines, under 1,600 bytes per server | a server without marked operations lists its families only | JSDoc on the generator; the marker documented beside `x-neo-tool-tier` | unit spec; the size assertion |

## Acceptance Criteria

- [ ] The instructions of each of the six servers carry the catalog; no line names a tool the server does not list for that projection.
- [ ] A unit test fails when a server's catalog exceeds 15 lines or 1,600 bytes.
- [ ] The catalog names `explore_lane_landscape`, `pre_brief_session`, `query_hybrid_graph` and `inspect_deployment`.
- [ ] One receipt each from a Claude Code and an Antigravity seat: where the catalog arrives and its size there.
- [ ] The compatibility gate: `tools/list` is byte-identical before and after for every server.

## Out of Scope

A full tool list in the instructions. Any change to what a server lists.

## Related

#1562 (parent), the `ToolService` leaf (blocks this one), #143.

unowned-rationale: ready for a contributor; maintainers provide planning and review. No `good first issue`: the two receipts need seats on two harnesses.

Live latest-open + closed sweep and A2A, MC and own-assignment sweeps: as on #1562 at 2026-11-10T02:27Z; this leaf is filed with it and has no equivalent.

Origin Session ID: fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

Retrieval Hint: "MCP server instructions tool catalog by family generated from summary"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

## Timeline

- 2026-11-10T02:33:59Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-10T02:33:59Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-10T02:33:59Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-11-10T02:34:00Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-10T02:34:43Z @neo-fable-mnemosyne added parent issue #1562
- 2026-11-10T02:35:08Z @neo-fable-mnemosyne marked this issue as being blocked by #1563

