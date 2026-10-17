---
id: 1075
title: Neural Link sessions say who addressed them and what they run
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-17T00:27:42Z'
updatedAt: '2026-10-17T00:27:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1075'
author: neo-fable
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
# Neural Link sessions say who addressed them and what they run

## Context
The Neural Link's one session namespace per bridge is the design: peers tap into the same app instances to collaborate — a conversational UI, the Fleet Manager instance itself (operator, 2026-10-17 00:2xZ). Tonight two seats drove and read one avatar-lab scene without either knowing the other was there (neo-stage#93 receipt 6226483181; neo-stage#54 comment 6227246554): a session row today is `{id, connectedAt, activeApps}` and nothing else. The operator's quick win: when listing entry points, show who opened them. This leaf delivers the part the bridge can know by itself — who addressed a session, first and last — plus what the app already announces about itself.

## The Problem
`healthcheck().sessions[]` (`ai/services/neural-link/ConnectionService.mjs:633–662`, `getStatus`) exposes `id`, `connectedAt` and a window count. A peer choosing a `sessionId` cannot tell a Fleet Manager window from a lab, its own tab from a peer's, or whether another agent is already writing to it. The auto-target refusal (`ai/services/neural-link/resolveCallTarget.mjs:33–37`) names only the count of live sessions, so the caller's next move is a second call and a guess.

## The Architectural Reality
- The Bridge (`ai/mcp/server/neural-link/Bridge.mjs`) owns both rosters: `agents` Map<agentId, ws> (`:42–45`; the agentId is FM-signed, verified by `verifyBridgeToken.mjs:34–39`) and `apps` Map<appWorkerId, ws> (`:47–50`), with `appState` (`:51–57`: the app's last `register` message and one `window_connected` per window, replayed to agents that join later, `:286–298`). Every agent→app message passes `handleAgentMessage(agentId, data)` (`:398–438`) with `payload.target` — the bridge is the one place that knows which agent addressed which app, and it does not record it.
- The per-seat server's `ConnectionService` mirrors the bridge's events into `sessionData` (`:270–273`; `handleAppConnected` `:700–708` stores `appName`; `handleNotification` `:792–798` assigns the app's `register` params onto the row — `environment`, `userAgent`, `isSharedWorker` are already there, unshown). `getStatus` builds the healthcheck rows from it; `call()` resolves the target through `resolveCallTarget` (`:340`).
- Design authority: the operator's statement above (shared access is intended; visibility, not gating) and neo-stage#54's seed ("the identity and authority model" of the tool surface). This leaf adds description, no refusal.

## The Fix
1. **Bridge** — in `handleAgentMessage`, when the target app exists, record on `appState.get(target).addressed`: `first {agentId, at}` once, `last {agentId, at}` every time, `agents` Map<agentId, {count, lastAt}>; broadcast `app_addressed {appWorkerId, agentId, at, first}` to all agents; include the `addressed` snapshot in the `app_connected` payload a late-joining agent receives (`:286–298`) so replay carries it.
2. **ConnectionService** — handle `app_addressed` (`handleBridgeMessage` switch, `:759–776`) onto `meta.addressed`; take `addressed` from `app_connected`; `getStatus().sessions[]` rows become `{id, appName, environment, userAgent, isSharedWorker, connectedAt, activeApps, firstAddressedBy, lastAddressedBy, addressedBy: [agentId…], addressedByMe}` (`addressedByMe` = `this.agentId` is among them).
3. **resolveCallTarget** — accept the rows instead of bare ids; the refusal lists each live session as `<id> · <appName> · last addressed by <agentId> <ago>` or `· untouched`, so the caller can pick in the same turn. Single-session and zero-session behaviour unchanged.
4. **Docs** — the healthcheck and the refusal in `openapi.yaml` / the tool handbook name the new fields; one sentence in the Neural Link skill's reference if it lists the session row.

Out of this leaf, named for the Decision Record (neo-stage#54): the agentId → seat handle label (the FM's launch admission journal holds `agentId` + `identity`, `ai/services/fleet/launchAdmissionJournal.mjs:157`; the bridge sees only the signed id) and "opened by" for windows a seat's browser opened itself (only the first addresser is knowable there).

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `healthcheck().sessions[]` row | `ConnectionService#getStatus` | adds `appName, environment, userAgent, isSharedWorker, firstAddressedBy, lastAddressedBy, addressedBy, addressedByMe` | fields `null` / `[]` for a session nobody addressed or that sent no `register` | `openapi.yaml` healthcheck description, the handbook | `ConnectionService.spec.mjs` rows |
| Bridge → agents `app_addressed` notice; `app_connected.addressed` | `Bridge#handleAgentMessage`, `#registerAgent` replay | emitted per forwarded agent message; snapshot on late join | none (additive) | `BridgeProtocol.mjs` doc line | `Bridge.spec.mjs` |
| `resolveCallTarget(sessionId, liveSessions)` refusal text | `resolveCallTarget.mjs` | lists the live sessions with app name and last addresser | ids only when rows carry no facts | the error message is the doc | `resolveCallTarget.spec.mjs` |

## Acceptance Criteria
- [ ] AC-1 Bridge unit: after two agents address one app, `appState.addressed` has the first and the last with timestamps and both agents' counts; the `app_addressed` notice reaches every agent; a late-joining agent's `app_connected` carries the snapshot.
- [ ] AC-2 ConnectionService unit: `getStatus().sessions[]` rows carry the app's register facts and the addressers; `addressedByMe` is true only for the server's own agentId; a session nobody addressed reads `firstAddressedBy: null`.
- [ ] AC-3 resolveCallTarget unit: with two live sessions and no explicit target, the thrown message names each session's id, app name and last addresser (or "untouched"); one live session still resolves implicitly; zero throws as before.
- [ ] AC-4 Docs: `openapi.yaml` and the handbook describe the new row; the Neural Link skill reference, if it shows a session row, shows the new one.
- [ ] AC-5 (local, L3) one live run: two seats' servers against one bridge, one addresses an app, the other's `healthcheck` shows `lastAddressedBy` with the first seat's agentId — recorded in the PR body.

## Out of Scope
Any refusal or lease on cross-seat writes (the design is shared access); the agentId → seat handle label and FM-launched "opened by" (the #54 Decision Record); engine changes (`src/ai/Client.mjs` already sends what the row needs).

## Avoided Traps
A second registry beside the bridge's `appState`; stamping ownership from timing (a window that appears after my `open_component_window` is not thereby mine); an unsigned identity claim shown as verified.

## Related
neo-stage#54 (the tool surface Decision Record; the seat-label follow-up lands there) · neo-stage#93 (the walk that measured the gap) · #1072 / PR #1073 (the Neural Link's image channel, same server, merged) · Institution #590 / #655 (launch admission, where agentId ↔ identity lives)

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-17 00:24Z; no equivalent (`gh search issues "neural link session"` open: none). A2A in-flight claim sweep at 00:3xZ (last 20 claims): none on the bridge or the session list; Grace's #1072/#1073 touched `BaseServer.mjs` / `imageContent.mjs` and is merged. MC sweep: "Neural Link healthcheck sessions which agent or seat opened the app window, session owner visibility", 5 results, all healthcheck transcripts, no prior decision. Own-assignment sweep: 9 open in the Brain (#812 #810 #95 #108 #109 #116 #117 #122 #144), none on the Neural Link.

Decision Record impact: aligned-with the operator's shared-access design; the seat-label and authority model remain neo-stage#54's Decision Record.

Origin Session ID: 772d7fb3-824a-46f5-9525-ea8c264da5e0

Retrieval Hint: "neural link session row who addressed app_addressed lastAddressedBy resolveCallTarget refusal"

🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · session 772d7fb3-824a-46f5-9525-ea8c264da5e0

## Timeline

- 2026-10-17T00:27:43Z @neo-fable assigned to @neo-fable
- 2026-10-17T00:27:44Z @neo-fable added the `enhancement` label
- 2026-10-17T00:27:44Z @neo-fable added the `ai` label
- 2026-10-17T00:27:44Z @neo-fable added the `agent-os` label
- 2026-10-17T00:33:48Z @neo-fable cross-referenced by PR #1076

