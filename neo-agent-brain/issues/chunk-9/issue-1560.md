---
id: 1560
title: Omit duplicate description prose from get_mcp_tool_handbook entries in ToolService#buildToolHandbookEntry
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-10T02:11:46Z'
updatedAt: '2026-11-10T03:24:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1560'
author: neo-gemini-maryam
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
closedAt: '2026-11-10T03:24:45Z'
---
# Omit duplicate description prose from get_mcp_tool_handbook entries in ToolService#buildToolHandbookEntry

## Problem Statement

In [`ai/mcp/ToolService.mjs` (lines 447–470)](https://github.com/neomjs/neo-agent-brain/blob/dev/ai/mcp/ToolService.mjs#L447-L470), `buildToolHandbookEntry(toolName, operation, fullDescription, schemas=null)` constructs the response payload for `get_mcp_tool_handbook`:

```javascript
buildToolHandbookEntry(toolName, operation, fullDescription, schemas=null) {
    const
        dedicated = operation['x-neo-tool-handbook'],
        handbook  = dedicated || fullDescription || operation.summary || toolName;

    const entry = {
        toolId     : toolName,
        found      : true,
        title      : operation.summary || toolName,
        description: operation.description || operation.summary || '',
        handbook,
        source     : dedicated ? 'x-neo-tool-handbook' : (operation.description ? 'description' : 'summary')
    };
```

Across the 4 fleet MCP servers (`153` tools total), `151` operations do not define `x-neo-tool-handbook` (only `read_file` in `file-system/openapi.yaml:113` defines it). Consequently, when `dedicated` is absent (`source !== 'x-neo-tool-handbook'`), `entry.description` and `entry.handbook` contain the **exact same multi-kilobyte `operation.description` string twice** in every `get_mcp_tool_handbook` tool result:
- Across all 4 fleet servers (measured by iterating `toolService.getToolHandbook(tool.name)` over the listed tools of `memory-core`, `github-workflow`, `knowledge-base`, and `neural-link`), `get_mcp_tool_handbook` duplicates **`75,831 B`** of identical description text (`40,420 B` in `memory-core` alone; `2,798 B × 2` on `add_memory`).

Graduated from [`neomjs/neo` Discussion #20680](https://github.com/orgs/neomjs/discussions/20680) (Track 2 — Option I).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `Neo.ai.mcp.ToolService#buildToolHandbookEntry` (`ai/mcp/ToolService.mjs:447-469`) & `get_mcp_tool_handbook` `'200'` response schema across the 6 MCP server `openapi.yaml` specs (`memory-core`, `github-workflow`, `gitlab-workflow`, `knowledge-base`, `neural-link`, `file-system`) | `neomjs/neo#20680` Track 2 Option I | Omit `description` from the returned handbook entry and from `get_mcp_tool_handbook`'s `'200'` response schema across all 6 MCP server `openapi.yaml` specs so `handbook` is the single prose field, eliminating `75,831 B` of duplicate prose across the 4 fleet servers | `handbook` (`entry.handbook`) remains populated on every found tool (`dedicated \|\| fullDescription \|\| operation.summary \|\| toolName`), `found`, `toolId`, `title`, `source`, `inputSchema`, and `outputSchema` remain unchanged, and tool-not-found responses (`{toolId, found: false, code: 'TOOL_NOT_FOUND', message}`) remain unchanged | `ai/mcp/ToolService.mjs`, `ai/mcp/server/{file-system,github-workflow,gitlab-workflow,knowledge-base,memory-core,neural-link}/openapi.yaml` | `test/playwright/unit/ai/mcp/schemaCompaction.spec.mjs`, `test/playwright/unit/ai/mcp/server/McpServerListToolsSmoke.spec.mjs` |

## Avoidance / Falsification Receipt

- **What would falsify this change?** If any runtime caller or test read `handbook.description` instead of `handbook.handbook` when calling `get_mcp_tool_handbook`.
- **V-B-A verification (`git grep -n "handbook\.description"` across `ai/` and `test/`):**
  - **0 callers** in `ai/` or `test/` read `handbook.description`.
  - Every consumer and unit spec (`McpServerListToolsSmoke.spec.mjs:325,364,481,606,638,705`, `toolServiceDispatch.spec.mjs:148,163`, `CommentGuidanceTiers.spec.mjs:116-123,135-140`, `ToolRegistration.spec.mjs:223-225`, `classHierarchyScope.spec.mjs:82-89,106-112`) reads `handbook.handbook`.
  - Only `test/playwright/unit/ai/mcp/schemaCompaction.spec.mjs:268` asserted `expect(Object.keys(handbook).sort()).toEqual(['description', 'found', 'handbook', 'source', 'title', 'toolId'])`.

## Acceptance Criteria

- [ ] In `ai/mcp/ToolService.mjs` (`buildToolHandbookEntry`, lines `447–469`) and the OpenAPI `'200'` response schema for `get_mcp_tool_handbook` across all 6 MCP servers (`memory-core`, `github-workflow`, `gitlab-workflow`, `knowledge-base`, `neural-link`, `file-system`), omit the duplicate `description` field from the handbook entry so `handbook` is the single prose field.
- [ ] Update `test/playwright/unit/ai/mcp/schemaCompaction.spec.mjs` (`line 268`) and add an explicit assertion that `getToolHandbook` omits `description` and populates `handbook` across all listed tools.
- [ ] Verify all MCP unit tests pass (`schemaCompaction.spec.mjs`, `McpServerListToolsSmoke.spec.mjs`, `toolServiceDispatch.spec.mjs`, `CommentGuidanceTiers.spec.mjs`, `ToolRegistration.spec.mjs`, `classHierarchyScope.spec.mjs`, `OpenApiValidatorCompliance.spec.mjs`).

---
Authored by Maryam (Gemini 4 Argon, Antigravity). Session `473e18db-b717-4cc8-99be-5eede1f07c60`. 🧭


## Timeline

- 2026-11-10T02:11:47Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-10T02:11:48Z @neo-gemini-maryam added the `ai` label
- 2026-11-10T02:11:48Z @neo-gemini-maryam added the `performance` label
- 2026-11-10T02:11:48Z @neo-gemini-maryam added the `model-experience` label
- 2026-11-10T02:11:49Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-10T02:30:28Z @neo-fable-mnemosyne cross-referenced by #1562
- 2026-11-10T02:33:03Z @neo-fable-mnemosyne cross-referenced by #1563
- 2026-11-10T02:40:37Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-10T02:44:24Z @neo-gemini-maryam cross-referenced by PR #1570
- 2026-11-10T03:24:45Z @tobiu referenced in commit `300162a` - "fix(mcp): omit duplicate description field from get_mcp_tool_handbook entries (#1560) (#1570)"
- 2026-11-10T03:24:45Z @tobiu closed this issue
- 2026-11-10T03:25:58Z @neo-gemini-maryam cross-referenced by PR #1572

