---
id: 1303
title: An outside agent is never told the Neural Link's tool docs are in the handbook
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-gemini-julia
createdAt: '2026-10-26T15:53:32Z'
updatedAt: '2026-10-26T17:59:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1303'
author: neo-opus-grace
commentsCount: 0
parentIssue: 20114
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-26T17:59:51Z'
---
# An outside agent is never told the Neural Link's tool docs are in the handbook

## Context

In the cloud cold walk for neomjs/neo#20114 (2026-10-26), an agent without our skills saw 60 Neural Link tools whose descriptions are just titles and whose parameters are undocumented. It called `get_dom_rect` with `componentId` instead of `componentIds: [...]`. The real documentation (for example `depth`: 1 = self, 2 = self + children, −1 = all) is reachable only through `get_mcp_tool_handbook`, and nothing points there.

## The Problem

Compaction is right for our token budget, but the one sentence that makes it work for an outsider, "the full docs are in `get_mcp_tool_handbook(toolId)`", is never sent.

## The Architectural Reality

- `ai/mcp/server/neural-link/toolService.mjs:84-85` opts into `compactToolDescriptions` and `compactToolSchemas` (the `ai/mcp/ToolService.mjs` mechanism: compact listing, full schema in the lazily loaded handbook).
- `BaseServer` sends no MCP `instructions` on `initialize`.

## The Fix

Send MCP server `instructions` on `initialize` that name `get_mcp_tool_handbook(toolId)` as the place for usage and parameter docs. At minimum the standalone Neural Link does this; the owner decides whether every compacted server should. Alternatively, ship the standalone profile uncompacted. Keep it to one deliberate choice, recorded.

## Acceptance Criteria

- [ ] AC-1: a client with no prior knowledge learns from the `initialize` result (or uncompacted tools) where each tool's parameters are documented (test on the response).
- [ ] AC-2: the team's compact listing budget is unchanged for our own seats.

## Contract Ledger

Backfilled 2026-10-26 by the ticket's author, after review 5656324502 on #1304. The rows hold the shape that review's RA-1 and RA-2 settle.

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `BaseServer#getServerInstructions()` (new) | This ticket; MCP `InitializeResult.instructions` | The one override hook. Its default returns one sentence naming `get_mcp_tool_handbook({toolId})`, and `createMcpServer()` passes it to the SDK as `ServerOptions.instructions` | A subclass returning `null` sends no `instructions` | JSDoc, listed in the class extension model | `ServerInstructions.spec.mjs`: an SDK `Client` reads it over `InMemoryTransport` |
| `initialize` result of every `BaseServer` subclass (all six compact `tools/list` today) | This ticket | Carries that sentence; `tools/list` bytes stay unchanged | Absent where the hook returns `null` | none (protocol field) | Same spec, plus its compact `listTools` check |
| `get_mcp_tool_handbook` dispatch in `BaseServer` | This ticket | Exempt from the health gate on every server: it reads static OpenAPI metadata | none | A one-line reason at the dispatch check | The spec's handbook arm, which fails without the exemption |

## Out of Scope

- Rewriting individual tool descriptions.

## Related

Parent: neomjs/neo#20114 · #1282 (the standalone package)

Sweeps (2026-10-26T15:52Z): live latest-open, the 20 newest `neomjs/neo-agent-brain` issues, no equivalent; keyword `get_mcp_tool_handbook` / `compactToolDescriptions`, none open; A2A, latest 30 messages, no claim. unowned-rationale: the MCP server-surface owner decides between instructions and an uncompacted profile; the filer is the #20114 epic owner.

Origin Session ID: 9853627c-2129-4d41-b905-0c74bec35a48
Retrieval Hint: "neural-link compactToolDescriptions compactToolSchemas MCP instructions get_mcp_tool_handbook outside agent parameters undocumented"


## Timeline

- 2026-10-26T15:53:33Z @neo-opus-grace added the `enhancement` label
- 2026-10-26T15:53:33Z @neo-opus-grace added the `ai` label
- 2026-10-26T15:53:51Z @neo-opus-grace added parent issue #20114
- 2026-10-26T15:54:12Z @neo-opus-grace cross-referenced by #20114
- 2026-10-26T16:12:04Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-26T16:23:44Z @neo-gemini-julia cross-referenced by PR #1304
- 2026-10-26T16:32:06Z @neo-gemini-julia referenced in commit `a045728` - "test(mcp): drop redundant autoConnect config mutation in ServerInstructions.spec (#1303)"
- 2026-10-26T17:41:57Z @neo-gemini-julia referenced in commit `93e7e6f` - "fix(mcp): keep single getServerInstructions channel and single handbook health exemption (#1303)"
- 2026-10-26T17:59:51Z @tobiu referenced in commit `4dd096d` - "feat(mcp): send initialize instructions pointing clients to get_mcp_tool_handbook (#1303) (#1304)

* feat(mcp): send initialize instructions pointing clients to get_mcp_tool_handbook (#1303)

* test(mcp): drop redundant autoConnect config mutation in ServerInstructions.spec (#1303)

* fix(mcp): keep single getServerInstructions channel and single handbook health exemption (#1303)"
- 2026-10-26T17:59:51Z @tobiu closed this issue

