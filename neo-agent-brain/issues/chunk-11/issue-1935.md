---
id: 1935
title: A directOnly list_messages walks every broadcast per row and stalls
state: CLOSED
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-25T18:25:28Z'
updatedAt: '2026-11-25T19:02:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1935'
author: neo-opus-ada
commentsCount: 0
parentIssue: 1843
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-25T19:02:33Z'
---
# A directOnly list_messages walks every broadcast per row and stalls

## Context

Both `mc-server` stalls on 2026-11-25 ran the same `list_messages` shape. The callers sent their arguments verbatim:

- 16:41Z, @neo-opus-grace: `{"box":"inbox","directOnly":true,"projection":"compact","limit":8,"observer":{"scope":"involves-me"}}`. The call ended in `Request timed out`, and #1843's metrics recorded a `list_messages` maximum of 246,516 ms.
- 17:48Z, @neo-gpt-sophie: `{"status":"all","box":"inbox","projection":"compact","directOnly":true,"observer":{"scope":"involves-me"},"limit":12}`. She killed the wrapper at ~42 s, and the plane was restarted at 17:52Z.

Both seats first sent `directOnly` without `observer`, got `MailboxService.listMessages: directOnly requires observer`, and added the observer. The validation error leads every caller who wants direct messages onto this path.

Measured read-only on 2026-11-25 around 18:20Z, inside `mc-server`, against the live store. A separate `better-sqlite3` process used `{readonly: true}` and `query_only`, and every slice ran under a `timeout`:

| Probe | Result |
|---|---|
| `SENT_TO` edges to Grace's identity variants (the candidate set) | 9,240, found in 44 ms |
| Edges targeting `AGENT:*` | 27,676 |
| The count query over the first 250 / 1,000 candidates | 7.4 s / 39.5 s (≈ 37 ms per candidate) |
| The same 1,000, broadcast check pinned to `idx_edges_source` | 16 ms |
| Full count / full page with the pin | 126 ms / 142 ms |

Unpinned, the full count extrapolates to ≈ 5.7 min, and the page query runs the same CTE again.

## The Problem

`readMailboxObservation` (`ai/services/memory-core/helpers/mailboxObservation.mjs`) answers `directOnly` with one correlated check per candidate: `NOT EXISTS (… observer_broadcast.source = n.id AND type = 'SENT_TO' AND target = 'AGENT:*')`. `EXPLAIN QUERY PLAN` on the live store plans it as `SEARCH observer_broadcast USING INDEX idx_edges_target (target=?)`. For each direct message, it walks all 27,676 broadcast edges instead of the message's own handful. The cost is candidates × broadcasts, both growing with the fleet. It runs synchronously on the event loop inside the observation transaction, which is the stall.

## The Architectural Reality

- `Edges` has two single-column indices, `idx_edges_source` and `idx_edges_target`. With `source` correlated and `target` constant, the planner takes the constant.
- **Prior art:** `MailboxService.listMessages`' counts projection already pins the same lookup, with `+st.target = 'AGENT:*'` (#1270).
- **The same plan, three more sites** (`EXPLAIN QUERY PLAN` on the live store; each walks the 27,676 per row):
  - the `legacyBroadcast` EXISTS that scopes `own` and non-team `all` add (`observer_legacy`);
  - `getWakeDeliverySeries`' two `fromBroadcast` / `broadcasts` sums (`MailboxService.mjs`);
  - `stampBroadcastCohortEdge`'s read, which plans `target=?` despite its bound `source = ?`.
- `legacyBroadcastPredicate` also drives the legacy candidate branch, a scan of all broadcasts by `target`, where the target index is right. A pin inside the shared predicate would turn that scan into a full table scan.
- Not this class: the UNION branches that drive *from* `target = 'AGENT:*'` (MailboxService's legacy inbox branch, MemoryService's unread counts, SwarmHeartbeatService). Their inner lookups are already keyed by `source`.
- Design authority: `MailboxService.listMessages`, "Explicit observation has its own bounded projection and admission."
- Owning folders `ai/services/memory-core` and `ai/services/memory-core/helpers` (structure map read). This is an in-place change.

## The Fix

Pin each correlated own-edge lookup to the message's edges with `INDEXED BY idx_edges_source` on its `FROM`. The pin goes on the correlated subquery, never on the shared predicate, so a missing index fails `prepare` loudly instead of falling back:

1. `mailboxObservation.mjs`: the `directOnly` `NOT EXISTS (… observer_broadcast …)` and the correlated `legacyBroadcast` EXISTS (`observer_legacy`). The legacy candidate branch keeps its target scan.
2. `MailboxService.mjs`: `getWakeDeliverySeries`' two correlated EXISTS, and `stampBroadcastCohortEdge`'s read.

Results are unchanged; only the plan changes.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `list_messages` with `observer` (every scope, `directOnly`) | `mailboxObservation.mjs` | Same rows and counts. Each candidate's broadcast check reads only its own edges | — | JSDoc | unit + PMV |
| `getWakeDeliverySeries`, `stampBroadcastCohortEdge` | `MailboxService.mjs` | Same results, own-edge lookup by `source` | — | JSDoc | unit |

## Acceptance Criteria

- [ ] AC-1 (unit): on the Brain's own graph schema, the plans of the `directOnly` observation query and of the `own` / non-team `all` observation query search each correlated `AGENT:*` check through `idx_edges_source`, never `idx_edges_target`. The legacy candidate branch still searches `idx_edges_target`.
- [ ] AC-2 (unit): the plans of `getWakeDeliverySeries`' queries and of `stampBroadcastCohortEdge`'s read search the message's `SENT_TO AGENT:*` edge through `idx_edges_source`.
- [ ] AC-3 (unit): the existing observation, mailbox and wake-delivery-series specs pass unchanged, so rows, counts and pagination are the same.
- [ ] PMV-1 (post-merge, after `mc-server` restarts on the merged code): Grace's 16:41Z shape answers in under 1 s in the container, and `get_memory_core_tool_metrics` records no `list_messages` call with `directOnly` above 10 s over the next 24 h.

## Out of Scope

- `box: 'all'` and wide `status: 'all'` reads, and `query_recent_turns` (#1843's remaining hot paths).
- An `Edges(source, type, target)` composite index. It would fix the class in the schema, but it needs a migration across every store. Not filed.
- `sqlite_stat1` / `ANALYZE`. Per-index averages cannot show that one target (`AGENT:*`) is hot.

## Avoided Traps

- **`+target` in `legacyBroadcastPredicate`.** It would also strip the index from the legacy candidate scan that drives from `target`.
- **Rejecting `directOnly`, or widening its error message.** The callers' intent is right, and the cost lives in the plan.

## Related

Parent: #1843 · Sibling: #1933 (names a stall's query shape) · Prior art: #1270 (`+st.target`), #563 (page-first SQL; its Out of Scope left the match-set count to "a later ticket if the plane metric asks")

Live latest-open + closed sweep:
- Ran at 2026-11-25T18:25Z: the latest 20 open, plus open searches for "list_messages slow / directOnly / AGENT:* index / query plan" and closed searches for "list_messages slow / mailbox observation / idx_edges_target / broadcast edge index".
- #563's and #962's Out of Scope and Avoided Traps were read. No equivalent or prior exclusion found.
- 157 open issues, which is the red band. #1843 owns the `list_messages` hot path, and this leaf is its measured root cause for both stalls.
- A2A sweep: the last 30 messages, all read states. No claim on this scope.
- MC sweep: "list_messages request timed out mc-server stall minutes mailbox query slow broadcast edges", 8 results, no prior decision found.
- Own-assignment sweep: 9 open. #1843 is the parent, and #1933 is a sibling on attribution, not on this query.

Decision Record impact: none.

Retrieval Hint: "list_messages directOnly observer stall AGENT:* broadcast idx_edges_target correlated subquery plan"
Origin Session ID: 86f1f927-fcc8-4839-adab-bb17cfb5c265

## Timeline

- 2026-11-25T18:25:28Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-25T18:25:30Z @neo-opus-ada added the `bug` label
- 2026-11-25T18:25:30Z @neo-opus-ada added the `ai` label
- 2026-11-25T18:25:31Z @neo-opus-ada added the `performance` label
- 2026-11-25T18:25:31Z @neo-opus-ada added the `agent-os` label
- 2026-11-25T18:25:45Z @neo-opus-ada added parent issue #1843
- 2026-11-25T18:25:48Z @neo-opus-ada cross-referenced by #1843
- 2026-11-25T18:39:07Z @neo-opus-ada cross-referenced by PR #1936
- 2026-11-25T18:51:36Z @neo-opus-ada cross-referenced by PR #1937
- 2026-11-25T19:02:33Z @tobiu referenced in commit `c806ea2` - "fix(mailbox): read a message's AGENT:* edge by source (#1935) (#1936)

Four correlated lookups of a message's own SENT_TO AGENT:* edge name idx_edges_source: the directOnly broadcast check and the legacy-broadcast check in the observation query, getWakeDeliverySeries' two sums and stampBroadcastCohortEdge's read. With source correlated and target constant, the planner took idx_edges_target and walked every broadcast edge once per row; a directOnly list_messages over 9,240 direct messages took minutes. Plan assertions pin each lookup."
- 2026-11-25T19:02:35Z @tobiu closed this issue
- 2026-11-25T19:09:06Z @neo-opus-ada cross-referenced by #1939
- 2026-11-25T19:09:13Z @neo-opus-ada cross-referenced by #1940

