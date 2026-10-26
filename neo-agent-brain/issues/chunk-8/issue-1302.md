---
id: 1302
title: Stopping the Neural Link leaves its Bridge and server running
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-gemini-julia
createdAt: '2026-10-26T15:53:23Z'
updatedAt: '2026-10-26T19:56:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1302'
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
closedAt: '2026-10-26T19:56:43Z'
---
# Stopping the Neural Link leaves its Bridge and server running

## Context

The cloud cold walk for neomjs/neo#20114 (2026-10-26) ran the published `neo-neural-link@1.0.0` through its `.mcp.json` command (`npx -y neo-neural-link`). Observed:
- `manage_connection {action:'stop'}` replied "Neural Link Bridge stopped.", yet the Bridge kept running and the app stayed connected.
- When the stdio client went away, the MCP server did not exit.
- SIGTERM to the top-level `npm exec` process, the one a harness starts, left the MCP server and the Bridge running, with port 8081 still open. The agent had to kill them one by one.

## The Problem

An outside user's harness session ends and leaves node processes holding port 8081. The `stop` action also reports success for something it did not do.

## The Architectural Reality

- `ai/services/neural-link/ConnectionService.mjs:1235-1237` spawns the Bridge through a **detached `npm`** process (a new process group). The `stop` branch at `:1120` calls `bridgeProcess.kill()` on that npm wrapper only, so `run-bridge.mjs` survives.
- `ai/mcp/server/neural-link/standalone.mjs:76-80` forwards SIGINT/SIGTERM to its child server. Whether `npm exec` delivers them, and whether the server exits on stdin EOF, is not established in source yet.
- The Bridge is designed to be long-lived and shared (detached + `unref`). Whether it should outlive its last client in the standalone product is a design question for the owner, not part of these ACs.

Design authority: the MCP specification (2025-06-18), *Lifecycle → Shutdown → stdio*. A client's first shutdown step is closing the server's input stream; SIGTERM and then SIGKILL follow only if the server does not exit. So the server exits on stdin EOF. For `stop`, the authority is `manageConnection`'s own reply, "Neural Link Bridge stopped.", which claims a stopped Bridge.

## The Fix

- `stop` ends the Bridge it spawned (its process group) and replies only once the port is closed.
- The MCP server exits when its stdio transport closes or it receives SIGTERM.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `manage_connection({action:'stop'})`, Bridge spawned by this server | AC-1; the reply's own claim | Signals the Bridge's process group. Replies "Neural Link Bridge stopped." only once nothing listens on the port; otherwise rejects and names the port. | None: no success reply while the port is open | `openapi.yaml` `manage_connection` ("start/stop"), unchanged | `ConnectionService.spec.mjs`, #1302 arms |
| `manage_connection({action:'stop'})`, Bridge found running | AC-3 | Disconnects only, and its reply says it killed nothing | — | as above | `ConnectionService.spec.mjs`, external-Bridge arm |
| Neural Link MCP server process, stdio mode | AC-2; the MCP stdio shutdown above | Exits on stdin EOF, on SIGTERM, and when its parent dies. HTTP mode is unaffected. | — | — | `standalone.spec.mjs` |
| `neo-neural-link` bin (`standalone.mjs`) | AC-2 | Forwards termination signals, and ends its server when its own parent dies | — | — | `standalone.spec.mjs` |
| `run-bridge.mjs` | AC-1 | On SIGTERM or SIGINT, closes its server and exits 0 | — | — | `ConnectionService.spec.mjs`, run-bridge arm |

## Acceptance Criteria

- [ ] AC-1: after `manage_connection stop` on a Bridge this server spawned, nothing listens on the Bridge port, and the reply reflects the actual outcome (test with a spawned Bridge).
- [ ] AC-2: closing the server's stdin, or SIGTERM to the `npx` process, ends the MCP server process (test).
- [ ] AC-3: a Bridge found already running (not spawned here) is still never stopped by this server.

## Out of Scope

- Whether a spawned Bridge should outlive its last MCP client (owner decision; record it if made).

## Related

Parent: neomjs/neo#20114 · #1282 (the standalone package)

Sweeps (2026-10-26T15:52Z): live latest-open, the 20 newest `neomjs/neo-agent-brain` issues, no equivalent; keyword "manage_connection stop" / "bridge keeps running", none open; A2A, latest 30 messages, no claim. unowned-rationale: Brain Neural Link lifecycle owner to pick up; the filer is the #20114 epic owner, not the NL server owner.

The design-authority line and the Contract Ledger were added by the ticket's author on 2026-10-26, during the review of #1305. The ACs are unchanged.

Origin Session ID: 9853627c-2129-4d41-b905-0c74bec35a48
Retrieval Hint: "neural-link manage_connection stop detached npm bridge survives MCP server does not exit stdin close SIGTERM npx port 8081"


## Timeline

- 2026-10-26T15:53:25Z @neo-opus-grace added the `bug` label
- 2026-10-26T15:53:26Z @neo-opus-grace added the `ai` label
- 2026-10-26T15:53:49Z @neo-opus-grace added parent issue #20114
- 2026-10-26T15:54:12Z @neo-opus-grace cross-referenced by #20114
- 2026-10-26T18:09:06Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-26T18:51:23Z @neo-gemini-julia cross-referenced by PR #1305
- 2026-10-26T19:36:23Z @neo-gemini-julia referenced in commit `2705830` - "fix(neural-link): require explicit port on bridge stop and centralize disconnect (#1302)"
- 2026-10-26T19:56:44Z @tobiu closed this issue
- 2026-10-26T19:56:44Z @tobiu referenced in commit `4b5577d` - "fix(neural-link): stop spawned bridge process group and exit server on stdio close or SIGTERM (#1302) (#1305)

* fix(neural-link): stop spawned bridge process group and exit server on stdio close or SIGTERM (#1302)

* fix(neural-link): require explicit port on bridge stop and centralize disconnect (#1302)"

