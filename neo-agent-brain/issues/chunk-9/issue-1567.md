---
id: 1567
title: 'Knowledge Base and file-system servers: each tool line says what the tool does and when to use it'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-11-10T02:33:45Z'
updatedAt: '2026-11-10T02:33:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1567'
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
# Knowledge Base and file-system servers: each tool line says what the tool does and when to use it

## Context

A server leaf of #1562 (graduated from neomjs/neo#20680, OQ-7). Blocked by the `ToolService` leaf, which makes `summary` the only source of a listed line. The two small servers share one leaf: 20 operations together.

## The Problem

9 of the 13 Knowledge Base summaries and 1 of the 7 file-system summaries repeat the operation id. The Knowledge Base has the pair a seat most often confuses, `ask_knowledge_base` and `query_documents`, and its lines do not say which one to take.

## The Architectural Reality

- `ai/mcp/server/knowledge-base/openapi.yaml` (13 operations, 6 curated lines today) and `ai/mcp/server/file-system/openapi.yaml` (7 operations, 2 curated lines).
- The Knowledge Base records calls per tool, so the measurement leaf can read its counts.

## The Fix

Rewrite each `summary` of both servers in the form `<what>; use when <trigger>`, at most 120 bytes, written by a seat that uses them. The lines of `ask_knowledge_base`, `query_documents`, `get_document_by_id` and `list_documents` say which question each answers.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `summary` of each operation on both servers | #1562, "The contract of the line" | `<what>; use when <trigger>`, at most 120 bytes | none | the OpenAPI files | the byte test of the `ToolService` leaf; the stranger read below |

## Acceptance Criteria

- [ ] All 20 summaries have the form; none repeats the operation id; the byte test is green.
- [ ] A second seat reads all 20 lines without the descriptions and says for each when it would call the tool; a line it cannot place is rewritten.
- [ ] The compatibility gate: this leaf changes no `inputSchema` and no `outputSchema`; the listed `outputSchema` of every tool is byte-identical before and after.

## Out of Scope

Descriptions and the handbook. The catalog.

## Related

#1562 (parent), the `ToolService` leaf (blocks this one).

unowned-rationale: ready for a seat that uses the Knowledge Base daily. No `good first issue`: the lines need use of a populated plane.

Live latest-open + closed sweep and A2A, MC and own-assignment sweeps: as on #1562 at 2026-11-10T02:27Z; this leaf is filed with it and has no equivalent.

Origin Session ID: fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

Retrieval Hint: "Knowledge Base file-system tool trigger lines summary what and when"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

## Timeline

- 2026-11-10T02:33:45Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-10T02:33:46Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-10T02:33:46Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-11-10T02:33:46Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-10T02:34:41Z @neo-fable-mnemosyne added parent issue #1562
- 2026-11-10T02:35:06Z @neo-fable-mnemosyne marked this issue as being blocked by #1563
- 2026-11-10T02:35:13Z @neo-fable-mnemosyne marked this issue as blocking #1569

