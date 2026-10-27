---
id: 1200
title: 'An FM update revokes every Codex seat: its Start lists ungranted servers'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-19T15:28:18Z'
updatedAt: '2026-10-19T17:19:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1200'
author: neo-opus-vega
commentsCount: 0
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-19T17:19:14Z'
---
# An FM update revokes every Codex seat: its Start lists ungranted servers

## Context

**Observed (installed, 2026-10-19).** An FM update at 15:17:50Z, from quitting Brain `48b8812` to successor `cf66245`, read from the FM's `main.log`:

- **The handoff sealed** after 4270 ms with 9 running seats.
  - Every `claude-desktop` and `antigravity` seat logged `survivor <id>: verified`.
  - All three `codex-desktop` seats logged `survivor neo-gpt-sophie: unresolved (server set changed)`; `neo-gpt-emmy` and `neo-gpt` logged the same.
- **The successor recovered** with `previous issuer handed-off …; … neo-gpt revoked, neo-gpt-emmy revoked …`.

**The issuer journal** (`launch-admission/issuer.json`, read for key names only):

| Seat class | Start `targets` | `profile.grants` | generation `servers` |
|---|---|---|---|
| `codex-desktop` (3 seats) | github-workflow, knowledge-base, memory-core, neural-link | neural-link | neural-link |
| `claude-desktop`, `antigravity` (6 seats) | the same four | the same four | the same four |

## The Problem

Every FM update revokes every Codex Desktop seat. The seat's running MCP children keep answering, but any reconnect is refused until the operator restarts that seat. Removing that restart round is the outcome of neomjs/neo-agent-institution#732. Today the three Codex seats needed a second restart within one hour.

**A separate cause hit the same seats on the same day.** At 15:29Z the host Bridge's agent list held none of the three Codex seats, and their NL app calls timed out. That persisted after a fresh restart, so it is not this ticket's revocation. The cause was the host Bridge itself: it ran a pre-#1154 revision (`9cd1528b`), on which the close of a seat's newest socket deletes the seat, orphaning its still-open long-lived socket. Codex spawns short-lived NL children at Start. The fix (`2d62f2e6`) is on `dev`, and the host Bridge refresh is the operator's call.

*(Corrected 2026-10-19: an earlier revision of this Problem inferred that the revocation removed the seat from the Bridge. Sophie's restart retest falsified that.)*

## The Architectural Reality

- **The check.** `McpLaunchAdmissionService.mjs:327` `recoverServers` throws `server set changed` unless three key sets all equal `saved.servers` (`:329–331`): the recovered capabilities, `start.profile.grants`, and `start.targets`.
- **The grants.** Grants are minted only for the Start's `servers` subset (`:565`, which skips `servers?.includes(key) === false`). A Codex Desktop Start admits only `neural-link`, because its other three servers come from the prepared remote adapter (`FleetLifecycleService.mjs`, `inspectPreparedRemoteMcpAdapter` / `REMOTE_MCP_SERVER_KEYS`, around `:3588`).
- **The targets.** `seatRuntimeRecord.mjs:135` records `targets: captureTargets(plan, …)`, which is the whole plan. Target rows (`key`, `target`, `args`, `slots`) carry nothing that says whether a row was admitted.
- So for a Codex seat the third comparison is false by construction, and no change ever happened.

Design authority: ADR 0034 §2.5 item 4, "an unproved fact about one surviving seat revokes only the authority that seat would have carried". A remote-adapter target never carried a grant. The check exists to catch a changed admitted set, not a server the generation never admitted.

## The Fix

The Start already records which servers its admission covered: `start.profile.grants`, whose keys `recoverServers` already compares with `saved.servers`. So `recoverServers` compares only the targets whose key is granted, and every other comparison stays as it is. No record field and no format change are needed.

This has one semantic edge. A planned target the issuer never granted is set aside instead of refused. It carries no grant, so no authority can return through it (ADR 0034 §2.5 item 4).

*(Revised 2026-10-19 before the PR: the first draft added an `admitted` field to the Start record, which would only duplicate `profile.grants`.)*

## Acceptance Criteria

- [ ] AC-1 (unit): a handoff over a `codex-desktop`-shaped survivor (four targets, `neural-link` admitted) verifies it and keeps its generation active.
- [ ] AC-2 (unit): a survivor whose admitted set really changed is still refused. A grant the issuer never minted is refused even earlier, by the launch-profile check (`row-set-mismatch`).
- [ ] AC-3 (unit): the granted servers stay the check. A granted server without its target row is still refused with `server set changed`, and the existing all-granted handoff arms stay green.
- [ ] AC-4 [L3-deferred — operator handoff needed]: on the next FM update with all three Codex seats running, each logs `survivor <id>: verified` and the successor lists them `active`. With the host Bridge on a revision carrying `2d62f2e6` (a shared prerequisite, not this fix), each also appears in the Bridge's agent list with its NL calls answering. This is witnessed in #1198's AC-5 run.

## Out of Scope

- How Codex Desktop serves its remote-adapter servers.
- The Codex stop delay the operator reported on 2026-10-19 (being traced separately).
- #1198's boot re-hash.

## Related

neomjs/neo-agent-institution#732 (epic), #1185 / #1186 (the handoff this check runs in), #1080 (the survivor reason lines that named this).

Live latest-open sweep: latest 20 open Brain and 10 open Institution issues at 2026-10-19T15:27Z, none equivalent; a title search over 100 Brain issues for Codex found only closed, different defects (`#1136`, `#1167`, `#1169`, `#1170`, `#1178`, `#1183`).
A2A sweep: last 15 messages, no claim on this scope.
MC sweep: "Codex Desktop seats revoked after FM update, server set changed, remote MCP adapter", 5 results, no prior decision found (the nearest, `#1178`, is a different Codex-after-update defect).
Own-assignment sweep: 11 open, none overlapping.
Structure map: `ai/services/fleet` owns both files; no new files.

Origin Session ID: 64acf30d-d59a-4080-b9b3-5343e568959a
Retrieval Hint: "codex-desktop survivor unresolved server set changed recoverServers targets grants REMOTE_MCP_SERVER_KEYS revoked every FM update"

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-10-19T15:28:19Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-19T15:28:20Z @neo-opus-vega added the `bug` label
- 2026-10-19T15:28:20Z @neo-opus-vega added the `ai` label
- 2026-10-19T15:28:20Z @neo-opus-vega added the `agent-os` label
- 2026-10-19T15:28:27Z @neo-opus-vega added parent issue #732
- 2026-10-19T16:24:31Z @neo-opus-vega cross-referenced by PR #1206
- 2026-10-19T16:54:48Z @neo-opus-vega cross-referenced by #1198
- 2026-10-19T17:19:14Z @tobiu closed this issue
- 2026-10-19T17:19:15Z @tobiu referenced in commit `eda14b9` - "fix(fleet): compare a survivor's granted targets, not every planned one (#1200) (#1206)

A Codex Desktop Start plans every server but the issuer admits Neural Link alone, so recoverServers'
target comparison failed by construction and every FM update revoked every Codex seat. The granted
servers stay the check; a planned target that never held a grant is set aside."
- 2026-10-19T19:26:43Z @neo-opus-vega cross-referenced by PR #1214
- 2026-10-20T10:00:49Z @neo-opus-ada cross-referenced by PR #1224

