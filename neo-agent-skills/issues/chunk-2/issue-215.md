---
id: 215
title: App-audience skills skip getting connected and name a maintainer-only tool
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-26T15:53:14Z'
updatedAt: '2026-10-26T19:01:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/215'
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
closedAt: '2026-10-26T19:01:33Z'
---
# App-audience skills skip getting connected and name a maintainer-only tool

## Context

The cloud cold walk for neomjs/neo#20114 (2026-10-26) read both app-audience skills (`neural-link`, `neo-app-authoring`, from `neo-agent-skills@0.1.61`) in a fresh workspace.

## The Problem

- Both `SKILL.md` routers say the agent "MUST" use a `view_file` tool, which an outside harness does not have.
- `neural-link`'s handbook covers tool order once connected, but says nothing about getting connected:
  - the Bridge on `127.0.0.1:8081` (or `NEO_NL_PORT`);
  - that `healthcheck` "healthy" with zero sessions means no page is connected;
  - what "No active App Worker sessions found" means: the page loaded before the Bridge, and the app stopped dialing after about 17 s by design, so reload or focus it.
- It notes that a reload changes the session id, but not that window info can be missing, which breaks the obvious source of a `windowId` for `simulate_event`.
- It never mentions `get_mcp_tool_handbook`, although the server ships compact tool descriptions and schemas (Brain `ai/mcp/server/neural-link/toolService.mjs:84-85`), so parameter docs live only there.

## The Architectural Reality

The app audience gets exactly these two skills, linked into `.claude/skills` and `.agents/skills` by `neo-agent-skills-materialize --audience app`. A version bump ships the change to new workspaces.

## The Fix

- Use a harness-neutral read instruction in both app-audience `SKILL.md` files.
- In `neural-link`, add a short "getting connected" section (Bridge, empty sessions, the reload rule) and a pointer to `get_mcp_tool_handbook(toolId)` for parameters.
- Add the `windowId` caveat after a reload until the engine leaf of #20114 ships.
- Bump the package version.

## Acceptance Criteria

- [ ] AC-1: neither app-audience `SKILL.md` names a harness-specific tool.
- [ ] AC-2: the `neural-link` skill states the Bridge port, the zero-sessions meaning and its fix, and the handbook pointer.
- [ ] AC-3: the package version is bumped in the same PR, and the skill-corpus lint passes.

## Out of Scope

- Maintainer-audience skills.
- The engine and Brain behaviour behind these symptoms (sibling leaves of #20114).

## Related

Parent: neomjs/neo#20114

Sweeps (2026-10-26T15:52Z): live latest-open, the 20 newest `neomjs/neo-agent-skills` issues, no equivalent; A2A, latest 30 messages, no claim; own-assignment, none open in this repo.

Origin Session ID: 9853627c-2129-4d41-b905-0c74bec35a48
Retrieval Hint: "app audience skills view_file neural-link getting connected bridge 8081 No active App Worker sessions get_mcp_tool_handbook"

## Timeline

- 2026-10-26T15:53:14Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-26T15:53:15Z @neo-opus-grace added the `enhancement` label
- 2026-10-26T15:53:16Z @neo-opus-grace added the `ai` label
- 2026-10-26T15:53:47Z @neo-opus-grace added parent issue #20114
- 2026-10-26T15:54:12Z @neo-opus-grace cross-referenced by #20114
- 2026-10-26T17:32:24Z @neo-opus-grace cross-referenced by PR #1304
- 2026-10-26T18:05:07Z @neo-opus-grace cross-referenced by #217
- 2026-10-26T18:35:04Z @neo-opus-grace cross-referenced by PR #219
- 2026-10-26T18:47:03Z @neo-opus-grace referenced in commit `f22f681` - "docs(skills): merge the Neural Link handbook's two no-session bullets (#215)

Zero sessions in healthcheck and "No active App Worker sessions found" are
one condition seen from two tools, so they share one bullet.

[skill-growth-justified: #215 AC-2 adds the handbook's Getting Connected section, 881 B of on-demand payload that maintainer and app audiences both load; the two routers shrank]"
- 2026-10-26T19:01:33Z @tobiu referenced in commit `0c8e512` - "feat(skills): app-audience skills say how to get connected and name no harness's own tool (#215) (#219)

* feat(skills): app-audience skills say how to get connected and name no harness's own tool (#215)

Both app-audience routers told the agent to "use the `view_file` tool",
which an outside harness does not have, and neural-link's handbook covered
tool order only once connected. The routers now say "read"; the handbook
gains a Getting Connected section: the Bridge on 127.0.0.1:8081
(NEO_NL_PORT), what zero sessions means, the reload rule for "No active App
Worker sessions found", get_mcp_tool_handbook for parameters, and the
windowId fallback for engines up to 13.3.2. The corpus lint refuses an
app-audience router that names view_file or read_file. 0.1.64.

* docs(skills): merge the Neural Link handbook's two no-session bullets (#215)

Zero sessions in healthcheck and "No active App Worker sessions found" are
one condition seen from two tools, so they share one bullet.

[skill-growth-justified: #215 AC-2 adds the handbook's Getting Connected section, 881 B of on-demand payload that maintainer and app audiences both load; the two routers shrank]"
- 2026-10-26T19:01:34Z @tobiu closed this issue

