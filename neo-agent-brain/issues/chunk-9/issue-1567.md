---
id: 1567
title: 'Knowledge Base and file-system servers: each tool line says what the tool does and when to use it'
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-10T02:33:45Z'
updatedAt: '2026-11-11T00:56:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1567'
author: neo-fable-mnemosyne
commentsCount: 1
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
closedAt: '2026-11-11T00:56:50Z'
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
- 2026-11-10T16:50:04Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-10T17:00:56Z @neo-gemini-maryam cross-referenced by PR #1578
- 2026-11-10T17:05:47Z @neo-gemini-maryam referenced in commit `68c2125` - "feat(mcp): refine 6 tool summaries from AC-2 second-seat blind read (#1567)"
### @neo-gemini-maryam - 2026-11-10T17:07:20Z

### Session Sunset Handover — `neomjs/neo-agent-brain#1567` (`scope: solo-refresh`)

- **Owner:** `@neo-gemini-maryam` (Maryam)
- **Open PR:** [`neomjs/neo-agent-brain#1578`](https://github.com/neomjs/neo-agent-brain/pull/1578) (`agent/1567-kb-fs-tool-trigger-lines` @ `68c21256`)
- **Status:** Implementation + AC-2 second-seat blind read complete; CI running on `68c21256`.
- **Empirical Anchors:**
  - `AC-1` & `AC-3`: `npm run test-unit -- test/playwright/unit/ai/mcp/server` (`682 passed (6.2s)`) + `npm run test-unit -- test/playwright/unit/ai/mcp/server/McpServerListToolsSmoke.spec.mjs test/playwright/unit/ai/mcp/server/knowledge-base/classHierarchyScope.spec.mjs test/playwright/unit/ai/mcp/server/file-system/toolServiceDispatch.spec.mjs test/playwright/unit/ai/services/knowledge-base/IngestionService.spec.mjs` (`142 passed (2.5s)`).
  - `AC-2`: Blind 20-line summary-only read (without `operationId` or `description`) completed by `@neo-opus-grace` in `MESSAGE:c921e47b-dda2-433f-b13c-b9aa841a9aa8` (`20/20` placed; 6 summaries `#2`, `#4`, `#6`, `#8`, `#9`, `#20` refined in `68c21256`).
- **Pickup Protocol for Next Session:**
  1. Check `gh pr checks 1578 -R neomjs/neo-agent-brain` on head `68c21256`.
  2. Once CI is green, assign `@neo-opus-grace` (or another eligible cross-family reviewer) via `manage_pr_reviewers` and send the actionable `primary-reviewer` A2A DM per `ci-green-review-routing.md §3`.

Authored by Maryam (Gemini 4 Argon, Antigravity). Session 985cc2dd-9f98-42ad-b4d3-660e4e5336e5. 🧭

- 2026-11-10T22:33:29Z @neo-gemini-maryam referenced in commit `a609797` - "feat(mcp): add trigger summaries to knowledge-base and file-system tools (#1567)"
- 2026-11-10T22:33:30Z @neo-gemini-maryam referenced in commit `29d0436` - "feat(mcp): refine 6 tool summaries from AC-2 second-seat blind read (#1567)"
- 2026-11-10T23:50:00Z @neo-gemini-maryam referenced in commit `62f7f7c` - "feat(mcp): add trigger summaries to knowledge-base and file-system tools (#1567)"
- 2026-11-10T23:50:00Z @neo-gemini-maryam referenced in commit `6aa2bc1` - "feat(mcp): refine 6 tool summaries from AC-2 second-seat blind read (#1567)"
- 2026-11-11T00:08:25Z @neo-gemini-maryam referenced in commit `d53ef2f` - "fix(mcp): align KB trigger summaries with corpus scope and MCP volume gate (#1567)"
- 2026-11-11T00:56:50Z @tobiu referenced in commit `44726d7` - "feat(mcp): add trigger summaries to knowledge-base and file-system tools (#1567) (#1578)

* feat(mcp): add trigger summaries to knowledge-base and file-system tools (#1567)

* feat(mcp): refine 6 tool summaries from AC-2 second-seat blind read (#1567)

* fix(mcp): align KB trigger summaries with corpus scope and MCP volume gate (#1567)"
- 2026-11-11T00:56:51Z @tobiu closed this issue
- 2026-11-11T05:39:12Z @neo-gemini-maryam cross-referenced by PR #1600

