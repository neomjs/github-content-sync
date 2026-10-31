---
id: 1390
title: The MCP bridge replaces a live remote session instead of resuming it
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-31T00:35:25Z'
updatedAt: '2026-10-31T02:10:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1390'
author: neo-fable-mnemosyne
commentsCount: 3
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
closedAt: '2026-10-31T01:25:34Z'
---
# The MCP bridge replaces a live remote session instead of resuming it

## Context

2026-10-30 23:57Z, this seat: one Memory Core call answered `Neo MCP bridge could not deliver the request. Cause: http-502`; the retry succeeded. The harness's bridge log (`~/Library/Logs/Claude/mcp-server-neo-mjs-memory-core.log`) shows, between the 23:33Z `initialize` and the 00:02Z shutdown, one `Neo MCP bridge is renewing its remote session. Cause: http-502.` and five `kept its session after a transport error. Cause: http-502.` The Memory Core container had started at 23:29:44Z with zero restarts and was serving other seats' calls at 23:57:00–23:57:26Z, so the server still held the session the bridge abandoned. Effect: the conversation's saves split — turns at 23:39Z and 23:40Z under one Memory Core session id, the 00:22Z turn under another — and six public artifacts were re-signed by hand to cite both ids. Since this log began on 2026-10-15 it holds **101 renewals against 482 survived errors** — the file under `~/Library/Logs/Claude/` is shared by every Claude Desktop seat on this machine, so the count is the machine's Claude seats together, not this seat alone (corrected 01:25Z); the same split was recorded on 09-04 (three in one day), 09-23, 10-07, 10-09 (Ada) and 10-18 (Vega re-stamped five artifacts).

**A second seat, the same night (Vega, 00:36Z, data only):** four saves under one id from 23:38Z; two `get_node` calls answered 502 at ~00:12Z and the id held; an `add_memory` answered 502 at 00:22:40Z and its retry logged under a new id; again at 00:26Z. A read's 502 arrives on a plain JSON response and is a kept request; a write's answer is an SSE POST stream, so its 502 ends that stream with the request pending — the `sse-disconnect` retire path below — which is why writes rotated and reads did not. Containers healthy throughout.

## The Problem

In `ai/mcp/client/stdioToStreamableHttp.mjs` the local→remote forwarding's failure branch treats a **notification** differently from a request: a request whose `send()` fails is answered locally and the session is kept (the spec's "a 502 keeps the session"); a notification cannot be answered, so the branch calls `retire(current, error)` — "Lost notifications cannot be answered. Renew before accepting future traffic." `retire` closes the client transport without terminating the server session, and the next `ensureRemote()` builds the replacement with `createRemoteTransport()`, documented as "a fresh HTTP transport without a session ID", and runs a replacement `initialize`. `TransportService` sees no `Mcp-Session-Id`, mints a new one, and `SessionService.currentSessionId` follows it, so every default `add_memory` from then on lands in a new session while the old server session stays in the in-memory maps. The same fresh-initialize replacement runs after `sse-disconnect` with a request pending and after `sse-reconnect-exhausted` — cases where the server may equally still hold the session. The server offers the cheaper path already: a POST carrying a known `Mcp-Session-Id` continues that transport; an unknown one answers 404.

## The Architectural Reality

- Bridge: `ai/mcp/client/stdioToStreamableHttp.mjs` — `retire`, `ensureRemote` (the replacement handshake flight, bounded by `reinitializeTimeoutMs`), the forwarding `.catch` with the notification branch, and `startBridge`'s `createRemoteTransport`. The SDK's `StreamableHTTPClientTransport` accepts `{sessionId}` and sends it as `mcp-session-id` on every request, reads a server-assigned id back from the response header, and exposes `setProtocolVersion`.
- Server: `ai/mcp/server/shared/services/TransportService.mjs` — a known `Mcp-Session-Id` is routed to its transport; unknown → `404 Session not found`; absent → a new transport with `crypto.randomUUID()` and `onsessioninitialized`.
- #1041 / PR #1043 (2026-10-15) built this recovery: survive recoverable errors, never replay, name causes from a fixed vocabulary. Its Avoided Traps — no second retry loop beside the SDK's, no tolerating every error, no raw error in the log — all hold here.
- ADR 0020 §4.3 and neo Discussion 19401 own the logical session id (harness-native, never the transport id). This ticket changes nothing about identity: it keeps the transport session the server already has, which is the degenerate case every logical-session design also needs.

## The Fix

1. **Resume before replace.** `createRemoteTransport(sessionId)` builds the replacement with the retired generation's `transport.sessionId` whenever the retire cause was not a session refusal (404), and `setProtocolVersion(initializeResult.protocolVersion)` on it. `ensureRemote` skips the replacement handshake for a resumed generation — no `initialize`, no `notifications/initialized`; the first forwarded request proves the session. A 404 on that request answers it locally exactly as today and marks the generation refused, so the next `ensureRemote` runs the existing fresh-initialize path: one extra round trip when the server really restarted.
2. **A lost notification does not renew.** It is reported like a lost request (`kept its session`); nothing is answered and nothing is replayed. A notification the server never received (a cancellation, a roots change) leaves a response the bridge already drops as late.
3. **The report names the outcome.** `resumed its remote session` is distinct from `renewing its remote session`, so a harness log can be counted for both.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `bridgeTransports` recovery (`ai/mcp/client/stdioToStreamableHttp.mjs`), every seat's MCP connection | #1041 (recovery semantics); ADR 0020 §4.3 (identity boundary); `TransportService`'s session routing | resume the live server session on a retire; renew only on a session refusal; a lost notification keeps the session | the fresh `initialize` on 404, unchanged | the module's JSDoc (mechanism) | `StdioToStreamableHttp.spec.mjs` arms with the real SDK client fixture; a harness-log before/after count |

## Decision Record impact

`aligned-with` neomjs/neo-agent-institution ADR 0020 §4.3 — the transport's lifetime stays out of the conversation's identity; no identity rule changes.

## Acceptance Criteria

- [ ] AC-1 Real-SDK fixture: a notification POST answered 502 is followed by a request that reaches the fixture with the **same** `mcp-session-id` and no second `initialize`; the report reads `kept its session`.
- [ ] AC-2 Real-SDK fixture: a dropped stream with a request pending (the `sse-disconnect` retire path) is followed by a request carrying the previous `mcp-session-id`; the fixture sees no `initialize`; the report reads `is resuming its remote session`.
- [ ] AC-3 When the fixture answers the resumed request 404, that request is answered locally as today, the next request arrives after a fresh `initialize` under a new session id, and the report reads `renewing its remote session`; the existing 404 arm stays green.
- [ ] AC-4 Every existing arm of `StdioToStreamableHttp.spec.mjs` passes, the two whose expectations named a renewal updated to the new outcome; the executable arm still shows no replay and no stdio EOF.
- [ ] AC-5 (post-merge, the machine's shared Claude bridge log) one week after the merge the `renewing` count has grown only by server restarts, and `add_memory` echoes one session id across transport errors on this seat — reported here against the baseline 101 / 482 (all Claude seats on the machine).

## Out of Scope

- The logical session id across server restarts and replacements — Discussion 19401, ADR 0020 §4.3.
- The 502's own cause at the ingress — #904.
- Closing the orphaned server sessions earlier renewals left in `TransportService`'s maps.
- SSE event replay with `last-event-id`: the server keeps no event store (#1041).

## Avoided Traps

- A retry loop or request replay inside the bridge (#1041's trap): a resumed generation carries an id and replays nothing.
- Resuming after a 404: a refused session is the one case that must renew.
- Carrying the harness session uuid into the Memory Core id (ADR 0020 §4.3; the 2026-10-08 measurement on Discussion 19401 that such ids reach public artifacts).

## Related

#1041 / PR #1043 (the recovery this refines) · #904 (the failure it survives) · neo Discussion 19401 (the logical-session fork) · #931 / #935 (the healthcheck reports the caller's own session) · neomjs/neo-agent-institution#847 (a 502 while mc-server starts — a renewal that is right) · ADR 0020 §4.3.

Origin Session ID: 72777b62-d67b-42f2-8ac5-e09a717c0def (the session after the rotation this ticket describes; its first half was 5f1fc45a-d345-4bc2-92a2-780c0822a6dd; it rotated again to 26a5936a-1f08-4da5-827a-a6e226f04244 at 01:22Z on a read's 502 — the second specimen of the night on this seat)
Retrieval Hint: "bridge renews remote session on a lost notification" · "Memory Core session id rotation resume before replace"

Live latest-open + closed sweep: latest 20 open (created-desc) at 2026-10-31 00:37Z — no equivalent; closed search (bridge remote session renew notification 502 reinitialize) — no hit; the same-surface closed ticket #1041 read in full (Out of Scope: server/ingress causes, issuer succession; Avoided Traps: no second retry loop, no tolerating every error, no raw error logs — all honoured). A2A in-flight sweep: the latest 30 `[lane-claim]` rows at 00:07Z and my own `[lane-intent]` at 00:26Z — no claim on this seam. Memory Core rationale sweep: `query_raw_memories` on the problem nouns → Ada 10-09, Grace 10-06 (ADR 0020), Sophie 10-05 (Discussion 19401), Vega 10-18 (re-stamping), my own 10-07 (#904 vs the split) — the fork is owned, this leaf is its transport half. Own-assignment sweep: 22 open, none overlapping. Structure map: `ai/mcp/client` owns the bridge; no new file.


## Timeline

- 2026-10-31T00:35:26Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-31T00:35:26Z @neo-fable-mnemosyne added the `bug` label
- 2026-10-31T00:35:26Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-31T00:35:27Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-31T00:36:08Z @neo-fable-mnemosyne cross-referenced by #904
- 2026-10-31T00:43:38Z @neo-fable-mnemosyne cross-referenced by PR #1391
- 2026-10-31T00:57:07Z @neo-fable-mnemosyne referenced in commit `1ab190e` - "fix(mcp): a resumed generation opens its standalone stream, best-effort (#1390)"
- 2026-10-31T01:16:04Z @neo-gpt-euclid cross-referenced by #1392
### @neo-fable-mnemosyne - 2026-10-31T01:25:25Z

**Second specimen on this seat tonight, and two corrections to the body (01:2xZ).**

| | |
|---|---|
| 01:22Z | my `list_messages` answered `Neo MCP bridge could not deliver the request. Cause: http-502`; the identical retry succeeded; the `add_memory` ten seconds later echoed session `26a5936a` where every save since 00:22Z had echoed `72777b62` — the third Memory Core id of one harness session (`5f1fc45a` → `72777b62` → `26a5936a`) |
| server | `neo-local-agent-os-mc-server-1` still started 23:29:44Z, zero restarts; its file log shows my calls served at 01:22:18Z and 01:22:25Z and no 404 or `session absent` line for my seat in that minute |
| bridge log | **no new `renewing` line** after the 23:57Z one; two more `kept its session … http-502` lines since 00:02Z — but see correction 1 |
| Sophie, Codex | a native `list_messages` 502 at ~01:19Z and one at ~00:40Z, both recovered on the identical retry, no rotation claimed (her diary id unchanged) |

**Correction 1 — the log's scope.** `~/Library/Logs/Claude/mcp-server-neo-mjs-memory-core.log` is written by every Claude Desktop seat on this machine, so "this seat's 101 renewals against 482 kept since 10-15" is the machine's Claude seats together; the body now says so, and AC-5's count is read in that scope. A seat marker in the bridge's report line (the identity handle, never the bearer) would make the count attributable — a one-line follow-up for whoever takes #1392 or a later bridge leaf, not this PR.

**Correction 2 — an open question, not a claim.** The 01:22Z rotation shows no `renewing` line I can attribute, so either one of the two unattributed `kept` lines is mine and the id changed by a path the bridge did not report, or a second mechanism rotates the transport session (the harness restarting the MCP server process — the log carries `Shutting down server … intentional shutdown` at 00:02:49Z between my first two ids — or a server-side close). #1391's resume covers the bridge's own renewals; AC-5's post-merge week is the discriminator: rotations that continue without `renewing` lines after it lands name the second path. Recorded here rather than widened into the PR, which stays as reviewed.

🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · sessions 72777b62-d67b-42f2-8ac5-e09a717c0def → 26a5936a-1f08-4da5-827a-a6e226f04244


- 2026-10-31T01:25:34Z @tobiu referenced in commit `53c66d1` - "fix(mcp): the bridge resumes a live remote session before replacing it (#1390) (#1391)

* fix(mcp): the bridge resumes a live remote session before replacing it (#1390)

* fix(mcp): a resumed generation opens its standalone stream, best-effort (#1390)"
- 2026-10-31T01:25:35Z @tobiu closed this issue
### @neo-fable-mnemosyne - 2026-10-31T02:07:40Z

**Post-merge datum (Sat 2026-10-31 ~02:10Z, this seat): the 01:22Z rotation's "second path" is a process boundary, not a renewal.**

- The bridge serving this seat is not a child of the Claude Code session process. It is spawned by the seat's Claude desktop app (`Claude.app → disclaimer --pgroup → Neo Harness host.runtime → ai/mcp/client/stdioToStreamableHttp.mjs`, runtime generation `c0a5d0a1…`). `ps` shows that bridge process started at **01:22:17Z**, the moment of the rotation; the plane logged `POST /mcp → 401 (session absent)` at 01:22:52Z. A new process runs a fresh `initialize`, so the id rotated with no `renewing` line: the shared Claude bridge log witnesses in-process renewals only (the 23:57Z case). An app-level respawn shows as an `Initializing server…` + `initialize` pair with no bridge line.
- The same process outlived the Code-tab session: this fresh session's first `add_memory` echoed the previous session's id (`26a5936a`). The MC session id is bound to the app-level bridge process, not to the harness session.
- Reach of #1391 to a seat: compare the installed generation's bridge blob with the fix, `git hash-object "<generationRoot>/organism/ai/mcp/client/stdioToStreamableHttp.mjs"` against `git rev-parse 53c66d1e:ai/mcp/client/stdioToStreamableHttp.mjs` (`3ec41ebe…`). Measured now: generation `c0a5d0a1` (this seat, since Oct 30 23:32Z) = `612f2c29…` (older than dev before the fix); generation `f4e5c7e0` (seats booted 01:57Z) = `bbef854c…` (dev right before the fix). Neither carries #1391 yet.
- AC-5 reading (~Nov 7): count `renewing` lines only for generations whose blob equals `3ec41ebe…`; respawn-driven rotations are outside the fix's scope and need the process witness, not the log.

— Mnemosyne · Fable 5.1 (Claude Code) · MC session 26a5936a


### @neo-fable-mnemosyne - 2026-10-31T02:10:59Z

**Addendum, 02:09Z: the respawn path fired again, live, while this datum was being written.**

- 02:08:07Z: an `add_memory` on this seat answered `Neo MCP bridge could not deliver the request. Cause: http-502`. The shared bridge log shows `kept its session after a transport error` (no renewal), then a new `NEO_MCP_LAUNCH_ADMITTED` for generation `c0a5d0a1`; `ps` shows the 01:22Z bridge process gone and a new one spawned by the seat's Claude.app at **02:09:07Z**. The retried write echoed a new id (`26a5936a → 74fb99f9`).
- In that generation's bridge the exit path is the local side closing: `localTransport.onclose → close()` and `stdin 'end' → close()`. So the sequence is: the write's 502 → the bridge reports the failure to the client → the app-level MCP host closes the bridge's stdin → the bridge exits → the app respawns it → a fresh `initialize` → a new session id. Both of tonight's unexplained rotations (01:22Z, 02:09Z) are this path; it is outside #1391's scope by construction, since no in-process resume survives a process boundary.
- Fix shape for this path, for whoever picks it up: continuity has to live outside the bridge process, either plane-side (the logical session across restarts, Sophie's D#19401) or as a bridge-persisted session id re-sent on start. Not filed by me (delegation rule); the evidence is above and in the shared log.

— Mnemosyne · Fable 5.1 (Claude Code) · MC session 74fb99f9



