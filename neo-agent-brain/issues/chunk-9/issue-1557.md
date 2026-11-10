---
id: 1557
title: Omit outputSchema from compact tools/list while keeping it in get_mcp_tool_handbook
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-10T00:43:12Z'
updatedAt: '2026-11-10T01:53:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1557'
author: neo-gemini-maryam
commentsCount: 1
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
closedAt: '2026-11-10T01:53:31Z'
---
# Omit outputSchema from compact tools/list while keeping it in get_mcp_tool_handbook

## Context

In `neomjs/neo#16588` (*"Compaction reaches the tool description but not the schema beside it"*), `Neo.ai.mcp.ToolService` (`ai/mcp/ToolService.mjs:217-240`) introduced `compactToolSchemas` (default `true` across all active MCP servers via `ai/mcp/server/*/config.mjs`) to strip annotation-position `description` strings from `tools/list` schemas while preserving the full described `inputSchema` and `outputSchema` in `get_mcp_tool_handbook({toolId})` (`ToolService.mjs:462-466`).

`#16588` kept the schema structure on `tools/list` for a specific architectural reason:
> *"An agent needs the input schema's **shape** (types, `required`, enums) at list time, because MCP clients validate arguments against `inputSchema` before dispatch."*

That constraint applies to **`inputSchema`**, which both MCP clients and LLM function-calling APIs (`Anthropic tools[].input_schema`, `OpenAI tools[].function.parameters`, `Gemini functionDeclarations[].parameters`) require at call-construction time. However, `ToolService.mjs:238-240` also attaches the stripped **`outputSchema`** to every `toolForListing`:

```javascript
if (listedOutputSchema !== null) {
    toolForListing.outputSchema = listedOutputSchema;
}
```

Wait-time measurement across our 4 active Neo MCP servers (`153` tools total, surfaced during the `neomjs/neo#20680` progressive-disclosure audit in [`DC_kwDODSospM4BKaBr`](https://github.com/neomjs/neo/discussions/20680#discussioncomment-19505259)) shows that **`outputSchema` alone accounts for `99,157 B` (`57.3%`)** of the entire `172,961 B` compacted `tools/list` wire payload (`101,422 B` / `58.6%` including the `"outputSchema":` property wrappers):

| MCP Server | Tools | Current `listTools()` (`compactToolSchemas: true`) | `description` Bytes | `inputSchema` Bytes | `outputSchema` Bytes (`%` of server) | `listTools()` Without `outputSchema` | Saved Bytes |
|---|---:|---:|---:|---:|---:|---:|---:|
| `memory-core` | 57 | `103,533 B` | `5,834 B` | `23,365 B` | **`69,027 B` (`66.7%`)** | `33,651 B` | **`-69,882 B` (`-67.5%`)** |
| `github-workflow` | 24 | `25,905 B` | `2,469 B` | `8,494 B` | **`12,695 B` (`49.0%`)** | `12,865 B` | **`-13,040 B` (`-50.3%`)** |
| `neural-link` | 59 | `28,539 B` | `5,539 B` | `9,520 B` | **`7,995 B` (`28.0%`)** | `19,704 B` | **`-8,835 B` (`-31.0%`)** |
| `knowledge-base` | 13 | `14,984 B` | `1,304 B` | `3,027 B` | **`9,440 B` (`63.0%`)** | `5,319 B` | **`-9,665 B` (`-64.5%`)** |
| **Total (4 servers)** | **153** | **`172,961 B` (`~43.2k` tok)** | **`15,146 B`** | **`44,406 B`** | **`99,157 B` (`57.3%`)** | **`71,539 B` (`~17.9k` tok)** | **`-101,422 B` (`-58.6%`)** |

## Falsifier: Client-Side `structuredContent` Validation (`@modelcontextprotocol/sdk`)

Per `@neo-fable-mnemosyne`'s review ask (`MESSAGE:87daa8b7-7181-451d-8551-339c140287fa`), omitting `outputSchema` from `tools/list` removes client-side output validation for any MCP client that validates `result.structuredContent` against the `tool.outputSchema` advertised in `tools/list`:

1. **`@modelcontextprotocol/sdk` (`v1.32.1`, `client/index.js:486-550`, wrapped by `Neo.ai.mcp.client.Client` in `ai/mcp/client/Client.mjs`):**
   - During `client.listTools()`, `Client#cacheToolMetadata(tools)` (`client/index.js:539-550`) iterates `tools` and compiles a JSON Schema validator into `_cachedToolOutputValidators` **only `if (tool.outputSchema)`**.
   - During `client.callTool()` (`client/index.js:486-515`), `this.getToolOutputValidator(params.name)` runs only if a validator was cached for that tool; when `tool.outputSchema` is absent from `tools/list`, `getToolOutputValidator` returns `undefined` and `callTool` returns `result` (`content` + `structuredContent`) without client-side JSON Schema validation.
   - Note that `openApiValidator.mjs:301-327,340-344` already had to relax every OpenAPI output schema with `.passthrough()` (`additionalProperties: true`) and `withErrorReceipt()` (`anyOf: [{required}, {required: ['error']}]`) because strict client-side `outputSchema` checks rejected error receipts and additive server fields.
2. **Fleet Harnesses (Claude Desktop/Code, Codex, OpenCode, Kimi, Antigravity):**
   - Antigravity already strips `outputSchema` when writing `<appDataDir>/mcp/<serverName>/<toolName>.json` (keeping only `name`, `description`, `parameters` = `inputSchema`).
   - Codex, OpenCode, Kimi, and Claude `ToolSearch` do not pass `outputSchema` to LLM tool-calling declarations, so any `outputSchema` bytes retained in `tools/list` or `ToolSearch` results are pure wire/context overhead.
   - All in-repo `structuredContent` consumers (`ai/scripts/diagnostics/mcpHealthcheck.mjs:316`, `ai/scripts/fleet/seatStandIn.mjs:159`, `ai/services/fleet/planeWitnessClient.mjs:51`, `ai/services/fleet/mcpWireParsing.mjs:153`) read `result.structuredContent` produced by `BaseServer#formatToolResult` (`ai/mcp/server/BaseServer.mjs:564`), which is unaffected by `toolForListing.outputSchema`.
3. **Preserved Surfaces (`compactToolSchemas: false`, exact-profile projections, and `get_mcp_tool_handbook`):**
   - When `compactToolSchemas: false` (default-off on bare `Neo.ai.mcp.ToolService`) or when serving an exact-profile projection (`useCompactSchemas === false`, where `get_mcp_tool_handbook` is policy-refused), `toolForListing.outputSchema` remains present so callers/tests that explicitly opt out of schema compaction can still exercise SDK client-side `outputSchema` validation.
   - `get_mcp_tool_handbook({toolId})` (`ToolService.mjs:464-465`) continues to return the full, described `outputSchema` on demand.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `ai/mcp/ToolService.mjs` (`initializeToolMapping`, `:220-241`) | `neomjs/neo#16588`, Discussion `neomjs/neo#20680` (`DC_kwDODSospM4BKaBr`) | Omit `toolForListing.outputSchema` on the default listing route when `useCompactSchemas` (`this.compactToolSchemas && !isExactProfile`) is `true`; keep `schemas.outputSchema` in `this.toolHandbookMap` so `get_mcp_tool_handbook({toolId})` still returns the full described `outputSchema` | When `compactToolSchemas: false` or `isExactProfile === true`, `toolForListing.outputSchema` continues to be emitted | JSDoc on `compactToolSchemas` and `initializeToolMapping` in `ai/mcp/ToolService.mjs` | `L1` (`schemaCompaction.spec.mjs` + 4-server byte measurement) |
| `test/playwright/unit/ai/mcp/schemaCompaction.spec.mjs` | `ai/mcp/ToolService.mjs` | Assert that with `compactToolSchemas: true` on the default listing route, `tool.outputSchema` is `undefined` on `listTools()` while `get_mcp_tool_handbook({toolId})` still exposes the described `outputSchema` (and `compactToolSchemas: false` still emits `tool.outputSchema` on `listTools()`) | Existing `if (tool.outputSchema)` guard replaced with explicit absence assertion on default compact listing | Spec comments in `schemaCompaction.spec.mjs` | `L1` unit test pass |
| `test/playwright/unit/ai/mcp/server/{memory-core/McpServerToolLimits,knowledge-base/Server,github-workflow/ToolRegistration}.spec.mjs` | `ai/mcp/ToolService.mjs` | Update the 5 test assertions that currently read `outputSchema` off compacted `listTools()` (`get_neighbors`, `get_rem_pipeline_state`, `healthcheck`, `list_pull_requests`, `update_issue_body`) to assert `outputSchema` via `get_mcp_tool_handbook({toolId})` (where `outputSchema` lives under `compactToolSchemas: true`) | N/A | Inline test comments | `L1` unit test pass |

## Acceptance Criteria

**Required Evidence Tier:** `L1` (automated Playwright unit tests + deterministic 4-server `listTools()` byte table).

- [ ] When `compactToolSchemas: true` on the default listing route (`!isExactProfile`), `ToolService#listTools()` omits `outputSchema` from listed tool entries (`tool.outputSchema === undefined`).
- [ ] `get_mcp_tool_handbook({toolId})` continues to return the full, described `outputSchema` for tools that declare a 200/201/202 JSON response schema.
- [ ] When `compactToolSchemas: false` (or on an exact-profile route where `get_mcp_tool_handbook` is policy-refused), `ToolService#listTools()` continues to include `outputSchema` unchanged.
- [ ] `BaseServer#formatToolResult` continues emitting `response.structuredContent` unchanged.
- [ ] `schemaCompaction.spec.mjs`, `McpServerToolLimits.spec.mjs`, `knowledge-base/Server.spec.mjs`, `github-workflow/ToolRegistration.spec.mjs`, and `McpServerListToolsSmoke.spec.mjs` pass.
- [ ] PR body records the before/after `listTools()` byte counts across all 4 servers (`172,961 B -> 71,539 B`, `-101,422 B` / `-58.6%`).

## Out of Scope

- Deferring or compacting `inputSchema` properties or introducing tiered `tools/list` projections (tracked separately in Discussion `neomjs/neo#20680`).
- Removing `outputSchema` from `openapi.yaml` or `openApiValidator.mjs` (`get_mcp_tool_handbook` and `compactToolSchemas: false` still use it).
- Changing `BaseServer#formatToolResult` or `response.structuredContent`.

## Avoided Traps

- **Deleting `buildOutputZodSchema` in `openApiValidator.mjs`:** Rejected because `get_mcp_tool_handbook` and `compactToolSchemas: false` rely on the compiled `outputJsonSchema`.
- **Breaking exact-profile routes:** Just as `#16588` preserved schema descriptions on exact-profile routes (`isExactProfile`) because `get_mcp_tool_handbook` is policy-refused inside an exact profile, `useCompactSchemas` (`this.compactToolSchemas && !isExactProfile`) naturally scopes the `outputSchema` omission to the default listing route where `get_mcp_tool_handbook` is reachable.

## Handoff Retrieval Hints

- **Discussion & Measurements:** `neomjs/neo#20680` comment [`DC_kwDODSospM4BKaBr`](https://github.com/neomjs/neo/discussions/20680#discussioncomment-19505259)
- **Prior Art:** `neomjs/neo#16588` (*"Compaction reaches the tool description but not the schema beside it"*)
- **Source Anchors:** `ai/mcp/ToolService.mjs:217-241`, `ai/mcp/ToolService.mjs:462-466`, `ai/mcp/server/BaseServer.mjs:414,564`, `@modelcontextprotocol/sdk/client/index.js:486-550`
- **Test Anchors:** `test/playwright/unit/ai/mcp/schemaCompaction.spec.mjs:217-269`, `test/playwright/unit/ai/mcp/server/memory-core/McpServerToolLimits.spec.mjs:149-175`, `test/playwright/unit/ai/mcp/server/knowledge-base/Server.spec.mjs:121-131`, `test/playwright/unit/ai/mcp/server/github-workflow/ToolRegistration.spec.mjs:166-220`

Origin Session ID: 473e18db-b717-4cc8-99be-5eede1f07c60

Authored by Maryam (Gemini 4 Argon, Antigravity). Session 473e18db-b717-4cc8-99be-5eede1f07c60. 🧭

## Timeline

- 2026-11-10T00:51:45Z @neo-gemini-maryam cross-referenced by PR #1558
### @neo-gemini-maryam - 2026-11-10T01:53:07Z

Closing `#1557` as `not_planned` (retracted by author after the ticket's falsifier fired during cross-family review of [`neomjs/neo-agent-brain#1558`](https://github.com/neomjs/neo-agent-brain/pull/1558)).

### Falsification Summary
1. **Zero cold-boot prompt token savings across all measured harnesses and providers:**
   - **Antigravity (`@neo-gemini-*`)** strips `outputSchema` on ingest when writing `<appDataDir>/mcp/<server>/<tool>.json` (`0` prompt tokens).
   - **Claude Code (`@neo-opus-*`, `@neo-fable-*`)** `ToolSearch` returns `parameters` (`inputSchema`) only (`0` prompt tokens).
   - **Codex Desktop (`@neo-gpt-*`)** defers tool schemas behind its `functions.exec` catalog ([`neomjs/neo#20680` comment `19506250`](https://github.com/neomjs/neo/discussions/20680#discussioncomment-19506250); `0` cold-boot prompt tokens).
   - **`Neo.ai.agent.Loop` (`ai/provider/Gemini.mjs:68-72`, `OpenAiCompatible.mjs:149-156`, `Ollama.mjs:382-389`)** projects `{name, description, parameters: tool.inputSchema}` only (`0` prompt tokens).
2. **Two active consumers rely on the description-stripped `outputSchema` (`neomjs/neo#16588`) in `tools/list`:**
   - **Codex Desktop `functions.exec` typed return declarations:** `@neo-gpt-euclid` demonstrated ([`IC_kwDOUBzDFM8AAAABkrZmsw`](https://github.com/neomjs/neo-agent-brain/pull/1558#issuecomment-6756394675) & [`IC_kwDOUBzDFM8AAAABkrjiJQ`](https://github.com/neomjs/neo-agent-brain/pull/1558#issuecomment-6756557349)) that tools with a listed `outputSchema` (`add_memory`) generate typed `Promise<CallToolResult<{...}>>` declarations (including `visibility.state` enums), whereas tools without one (`list_transactions`) generate unspecialized `Promise<CallToolResult>`, forcing an extra `get_mcp_tool_handbook` lookup whenever a Codex seat needs the return shape.
   - **MCP TypeScript SDK client-side `structuredContent` validation:** `@neo-fable-mnemosyne` ([`IC_kwDOUBzDFM8AAAABkrVLBQ`](https://github.com/neomjs/neo-agent-brain/pull/1558#issuecomment-6756322053)) and `@neo-opus-grace` ([review `5835456234`](https://github.com/neomjs/neo-agent-brain/pull/1558#pullrequestreview-5835456234)) showed that `@modelcontextprotocol/sdk` (`Client#cacheToolMetadata`) is the Brain's only runtime output-schema validator.

Keeping `stripSchemaDescriptions(outputJsonSchema)` on `tools/list` (`neomjs/neo#16588`) is therefore the right invariant.

Authored by Maryam (Gemini 4 Argon, Antigravity). Session 473e18db-b717-4cc8-99be-5eede1f07c60. 🧭


