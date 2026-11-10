---
id: 1563
title: 'ToolService lists one line per tool: summary is its only source, title and x-neo-tool-summary go'
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-10T02:33:02Z'
updatedAt: '2026-11-10T13:16:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1563'
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
blockedBy: []
blocking:
  - '[ ] 1568 Each server''s instructions carry a catalog of its tool families and the tools nobody finds'
  - '[ ] 1567 Knowledge Base and file-system servers: each tool line says what the tool does and when to use it'
  - '[ ] 1566 GitHub and GitLab workflow servers: each tool line says what the tool does and when to use it'
  - '[ ] 1565 Neural Link: each of its 60 tool lines says what the tool does and when to use it'
  - '[ ] 1564 Memory Core: each of its 55 tool lines says what the tool does and when to use it'
closedAt: '2026-11-10T13:16:18Z'
---
# ToolService lists one line per tool: summary is its only source, title and x-neo-tool-summary go

## Context

First leaf of #1562 (graduated from neomjs/neo#20680, OQ-7). It changes the code and moves the existing texts; it rewrites no line. The six server leaves depend on it.

## The Problem

A listed tool's one line has two sources and is sent twice. `buildToolListDescription` takes `x-neo-tool-summary`, then `summary`, then the description (`ai/mcp/ToolService.mjs:356`), and the same `summary` also goes out as the listed `title` (`:203`, passed on at `:234` and by the ListTools handler in `ai/mcp/server/BaseServer.mjs`). That double use is why `x-neo-tool-summary` was added (neomjs/neo#16602): a sentence in `summary` would have appeared in both fields. 45 operations now carry both fields (memory-core 30, github-workflow 7, knowledge-base 6, file-system 2; counted at `10deb831`). A line over `toolListDescriptionMaxLength` (120 on every server) is cut with `...` and nothing fails.

## The Architectural Reality

- `ai/mcp/ToolService.mjs`: `initializeToolMapping` (`title` at `:203`, `toolForListing` at `:231`), `buildToolListDescription` (`:354`), `buildToolHandbookEntry` (`title` at `:455`).
- `ai/mcp/server/BaseServer.mjs`: `setupRequestHandlers` (`ListToolsRequestSchema` handler at `:409-416`).
- Six OpenAPI files with operations: `ai/mcp/server/{file-system,github-workflow,gitlab-workflow,knowledge-base,memory-core,neural-link}/openapi.yaml`, 174 operations, every one with a `summary`, none over 120 bytes today.
- Five unit specs assert the old fields; they are listed with line numbers in the step-back on the Discussion ([comment](https://github.com/neomjs/neo/discussions/20680#discussioncomment-19506839)), plus `BaseServer.spec.mjs` (`ListTools` handler mock).
- #1560 (Maryam) changes `buildToolHandbookEntry` too; the two leaves touch neighbouring lines.

## The Fix

1. `buildToolListDescription` reads `operation.summary` and nothing else for the listed line.
2. The 45 `x-neo-tool-summary` texts move into `summary`, unchanged, and the field is deleted from the OpenAPI files and from the code.
3. A listed tool carries no `title`. The handbook entry's `title` becomes the tool's name.
4. A unit test reads all six OpenAPI files and fails on an operation without a `summary` or with one over 120 bytes.
5. The specs that assert `x-neo-tool-summary` or the listed `title` assert the new shape.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `tools/list` entry (`Neo.ai.mcp.ToolService#initializeToolMapping`, `Neo.ai.mcp.ToolService#buildToolListDescription`, `Neo.ai.mcp.server.BaseServer#setupRequestHandlers`) | #1562, neomjs/neo#20680 OQ-7 | `name`, `description` (the operation's `summary`), `inputSchema`, `outputSchema` and `annotations` as today; no `title` | none: the new test refuses an operation without a `summary` | JSDoc on `buildToolListDescription` and `toolListDescriptionMaxLength` | `McpServerListToolsSmoke.spec.mjs`, `BaseServer.spec.mjs`, `CommentGuidanceTiers.spec.mjs`, `classHierarchyScope.spec.mjs`; the same-tool check below |
| OpenAPI operation (`ai/mcp/server/{file-system,github-workflow,gitlab-workflow,knowledge-base,memory-core,neural-link}/openapi.yaml`) | #1562, neomjs/neo#20680 OQ-7 | `summary` is the listed line (`<= 120` bytes); `x-neo-tool-summary` does not exist (45 curated texts moved into `summary` verbatim) | none | the test's failure message names the operation and its byte count | `McpServerListToolsSmoke.spec.mjs`, `ToolRegistration.spec.mjs`, `IngestionService.spec.mjs` |
| `get_mcp_tool_handbook` answer (`Neo.ai.mcp.ToolService#buildToolHandbookEntry`) | #1562, neomjs/neo#20680 OQ-7 | `title` is the tool's name (`toolName`) | none | JSDoc on `buildToolHandbookEntry` | `schemaCompaction.spec.mjs` |

## Acceptance Criteria

- [ ] No file under `ai/` or `test/` contains `x-neo-tool-summary`.
- [ ] `listTools()` returns no `title` on any tool of any server, and each `description` equals that operation's `summary`.
- [ ] The listed line of each of the 45 operations that had a curated text is byte-identical before and after.
- [ ] The new test fails when a `summary` is missing or over 120 bytes, shown by a fixture.
- [ ] The compatibility gate: `outputSchema` of every listed tool is byte-identical before and after; and on a Codex client one tool with an output schema shows the same typed result declaration before and after (a GPT seat's receipt, or the probe packet named on neomjs/neo-agent-brain#1558).
- [ ] The unit suite is green.

## Out of Scope

Rewriting any line into the trigger form (the server leaves). The catalog. The duplicate text in handbook answers (#1560).

## Avoided Traps

- Retiring the field without moving its texts: 45 tools would fall back to a `summary` that often repeats the name.
- Keeping `title` and filling it from the name: a client already shows the name when `title` is absent.

## Related

#1562 (parent), #1560, neomjs/neo#16602, neomjs/neo#16588.

handoff: @neo-gemini-maryam, first refusal: her step-back found the cause and the five specs. No `good first issue`: it changes the listing of every server and carries a cross-client check.

Live latest-open + closed sweep: the latest 20 open (172 open in all, red band; this leaf belongs to #1562, filed the same hour) and searches over open and closed issues (tool, summary, trigger, `x-neo-tool-summary`, title) at 2026-11-10T02:27Z; no equivalent found. A2A in-flight claim sweep: to 02:22Z, only #1559 and #1560 on this surface. MC sweep: as on #1562, no prior decision. Own-assignment sweep: none on this surface.

Origin Session ID: fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

Retrieval Hint: "ToolService listed description summary only, title omitted, x-neo-tool-summary retired"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session fda09208-5d6a-4fc7-a0d4-69a53fdf2dec


## Timeline

- 2026-11-10T02:33:03Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-10T02:33:03Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-10T02:33:03Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-11-10T02:33:04Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-10T02:48:46Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-10T03:25:28Z @neo-gemini-maryam referenced in commit `29b7a8c` - "feat(mcp): source listed tool descriptions from summary alone and drop title (#1563)

Co-authored-by: neo-fable <neo-fable@neomjs.com>"
- 2026-11-10T03:25:58Z @neo-gemini-maryam cross-referenced by PR #1572
- 2026-11-10T03:34:56Z @neo-gemini-maryam referenced in commit `13d2786` - "feat(mcp): source listed tool descriptions from summary alone and drop title (#1563)

Co-authored-by: neo-fable <neo-fable@neomjs.com>"
- 2026-11-10T12:20:53Z @neo-gemini-maryam referenced in commit `a8ebda1` - "test(mcp): restore BaseServer spec ADR/ticket citations and import digest label (#1563)

Co-Authored-By: Maryam <neo-gemini-maryam@neomjs.com>"
- 2026-11-10T12:24:27Z @neo-fable-mnemosyne cross-referenced by #1562
- 2026-11-10T13:16:18Z @tobiu referenced in commit `7549b60` - "feat(mcp): source listed tool descriptions from summary alone and drop title (#1563) (#1572)

* feat(mcp): source listed tool descriptions from summary alone and drop title (#1563)

Co-authored-by: neo-fable <neo-fable@neomjs.com>

* test(mcp): restore BaseServer spec ADR/ticket citations and import digest label (#1563)

Co-Authored-By: Maryam <neo-gemini-maryam@neomjs.com>

---------

Co-authored-by: neo-fable <neo-fable@neomjs.com>"
- 2026-11-10T13:16:18Z @tobiu closed this issue

