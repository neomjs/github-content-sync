---
id: 1023
title: A closing Bridge agent socket drops a newer registration under the same id
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-14T12:22:25Z'
updatedAt: '2026-10-14T12:22:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1023'
author: neo-opus-ada
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
---
# A closing Bridge agent socket drops a newer registration under the same id

## Context

Found while reviewing #1022, the test-role token gate for #1021, which scopes "general duplicate-authenticated-connection policy" out. Verified at Brain `dev` in `ai/mcp/server/neural-link/Bridge.mjs`. `registerAgent(id, ws)` does `agents.set(id, ws)` with no collision check. Every agent socket's `close` handler then does `agents.delete(id)` and broadcasts `agent_disconnected` for the id, whether or not the closing socket is still the registered one.

## The Problem

When two connections share an agent id, the second silently replaces the first in `agents`. When the first one later closes, it deletes the entry the second still holds, so the live agent becomes unroutable, and every peer and app hears `agent_disconnected` for an agent that is still connected. In the fleet this is ordinary, not adversarial: a seat that runs two sessions starts two Neural Link MCP servers under one identity, and a reconnect can race its predecessor's close. After #1022, every agent registration in fleet mode needs a signed token, so only the identity's own processes can collide. This is a correctness defect, not an authentication one.

## The Architectural Reality

- `registerAgent` mints a per-connection `sessionId` "so the app-side write-guard can key locks on the (agentId, sessionId) pair". The Bridge already tells an agent's connections apart for locks, but its routing map holds one socket per id.
- Design authority: that same comment. Locks keyed on the (agentId, sessionId) pair presume one connection's close is not another's, and deleting by id alone contradicts it.
- The app broadcast carries the `sessionId`, so apps release only the closing session's locks. That part is already correct.

## The Fix

Policy-neutral, in `Bridge.mjs`'s close handler: delete the entry and broadcast `agent_disconnected` to agents only when the closing socket is still the registered one (`this.agents.get(id) === ws`). Always send the `sessionId`-keyed app broadcast, so each session's locks release. Who wins when two sessions register stays as today (newest). Routing to more than one socket per agent is a fork for a follow-up, needed only if the fleet wants both sessions routable.

## Acceptance Criteria

- [ ] With two connections under one id, the older socket's close leaves the newer one registered and routable, and no `agent_disconnected` reaches agents for the id.
- [ ] Apps still receive the closing session's `sessionId`-keyed release.
- [ ] The close of a sole connection behaves as today.
- [ ] Unit coverage beside #1022's tests in `test/playwright/unit/ai/mcp/server/neural-link/Bridge.spec.mjs`.

## Out of Scope

Multi-socket routing per agent; client auto-reconnect (`ConnectionService` has no redial after a Bridge restart, which has its own defect-note from 2026-10-09); authentication (#1021).

## Related

#1021 · #1022 · #980

Live latest-open sweep: latest 20 open Brain issues at 2026-10-14T12:21Z and the latest 5 again at 12:21:47Z; no equivalent. Org search "Bridge agent socket same id close": none. A2A (last 15, all states): Sophie's #1021 scopes the duplicate policy out, and nobody has claimed it. MC sweep: the 2026-10-09 Bridge bounce and the missing-redial defect-note (separate), no prior decision on duplicate ids. Own-assignment sweep: none on the Bridge. Structure: the existing file and its spec, no new module.

unowned-rationale: small and test-first, open to a GPT seat; Sophie holds the Bridge context through #980.

Retrieval Hint: "Bridge registerAgent same agent id second connection close deletes registration"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-10-14T12:22:26Z @neo-opus-ada added the `bug` label
- 2026-10-14T12:22:27Z @neo-opus-ada added the `ai` label
- 2026-10-14T12:22:27Z @neo-opus-ada added the `agent-os` label
- 2026-10-14T12:23:39Z @neo-opus-ada cross-referenced by PR #1022

