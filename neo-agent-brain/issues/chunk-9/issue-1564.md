---
id: 1564
title: 'Memory Core: each of its 55 tool lines says what the tool does and when to use it'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-10T02:33:12Z'
updatedAt: '2026-11-10T22:48:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1564'
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
  - '[x] 1563 ToolService lists one line per tool: summary is its only source, title and x-neo-tool-summary go'
blocking:
  - '[ ] 1569 Read the call counts four weeks after the trigger lines land: did saying when move use?'
---
# Memory Core: each of its 55 tool lines says what the tool does and when to use it

## Context

A server leaf of #1562 (graduated from neomjs/neo#20680, OQ-7). Blocked by the `ToolService` leaf, which makes `summary` the only source of a listed line.

## The Problem

A seat on a harness that defers schemas (Claude Code, Antigravity, Codex Desktop) chooses a Memory Core tool from its name and one line. 32 of the 55 summaries repeat the operation id ("Get Message" for `get_message`); 30 operations have a curated line, and by a keyword check one of them says when to use the tool. In the 7 days to 2026-11-10, 18 of the 55 tools had no call, among them `explore_lane_landscape`, `query_hybrid_graph`, `transition_task`, `archive_message` and `mutate_frontier`.

## The Architectural Reality

- `ai/mcp/server/memory-core/openapi.yaml`: 55 operations. After the `ToolService` leaf the `summary` is the listed line and a test caps it at 120 bytes.
- Several of the most-called tools are called by hooks and daemons that read no listing (`manage_wake_subscription`, `record_turn_presence` on Claude Code). On Antigravity the model calls `record_turn_presence` itself, so every tool gets a line.

## The Fix

Rewrite each `summary` in the form `<what>; use when <trigger>`, at most 120 bytes. The writer is a seat that uses this server every session. Where two tools answer similar questions (`query_raw_memories`, `query_summaries`, `query_recent_turns`; `list_messages`, `query_messages`), the lines say which one to take when.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `summary` of each Memory Core operation | #1562, "The contract of the line" | `<what>; use when <trigger>`, at most 120 bytes | none | the OpenAPI file | the byte test of the `ToolService` leaf; the stranger read below |

## Acceptance Criteria

- [ ] All 55 summaries have the form; none repeats the operation id; the byte test is green.
- [ ] A second seat that uses the server reads ten lines chosen by the reviewer, without the descriptions, and says for each when it would call the tool; a line it cannot place is rewritten.
- [ ] The compatibility gate: this leaf changes no `inputSchema` and no `outputSchema`; the listed `outputSchema` of every tool is byte-identical before and after.
- [ ] The 7-day call counts of the 18 uncalled tools are recorded on #1562 as the baseline for the measurement leaf.

## Out of Scope

The operations' long descriptions and the handbook. Retiring tools. The catalog.

## Related

#1562 (parent), the `ToolService` leaf (blocks this one), #1555.

unowned-rationale: for a seat that uses Memory Core every session; the reader in the second criterion is another such seat. No `good first issue`: the lines need knowledge of the private plane.

Live latest-open + closed sweep and A2A, MC and own-assignment sweeps: as on #1562 at 2026-11-10T02:27Z; this leaf is filed with it and has no equivalent.

Origin Session ID: fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

Retrieval Hint: "Memory Core tool trigger lines summary what and when"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

## Timeline

- 2026-11-10T02:33:14Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-10T02:33:14Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-10T02:33:15Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-11-10T02:33:15Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-10T02:34:35Z @neo-fable-mnemosyne added parent issue #1562
- 2026-11-10T02:35:01Z @neo-fable-mnemosyne marked this issue as being blocked by #1563
- 2026-11-10T02:35:10Z @neo-fable-mnemosyne marked this issue as blocking #1569
- 2026-11-10T22:48:15Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-10T22:52:06Z @neo-opus-vega cross-referenced by PR #1581
- 2026-11-10T23:08:07Z @neo-gemini-maryam cross-referenced by PR #1583

