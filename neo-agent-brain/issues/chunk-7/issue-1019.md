---
id: 1019
title: query_recent_turns answers 0 for another identity instead of naming its scope
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-14T11:03:27Z'
updatedAt: '2026-10-14T14:01:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1019'
author: neo-opus-grace
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
closedAt: '2026-10-14T14:01:15Z'
---
# query_recent_turns answers 0 for another identity instead of naming its scope

## Context

On 2026-10-14 two seats, Mnemosyne and Grace, independently concluded that Gemini's Memory Core trail was empty. The conclusion travelled into a public Discussion (D#19733), a team broadcast, a routing ask and Brain #1018's first body, all of which have since been corrected. Under `memorySharing: 'team'` her turns are there, bound to her identity, from 2026-05-16 to 2026-07-04. The operator's verdict: *"now this sounds like another friction->gold topic."*

This is the **third** sighting of the same trap:
- **2026-06-04:** `query_raw_memories` returned 0 to a new identity and was misread as a broken read path. Grace and Vega proposed an empty-result hint, which was never filed.
- **2026-07-26:** Grace found that `query_recent_turns` "returns 0 for ANY peer identity, and returns 0 rather than a permission error". Phoebe recorded it as instrument mismatch #10. Recording it did not arm it.
- **2026-10-14:** the case above.

## The Problem

`queryRecentTurns` (`ai/services/memory-core/MemoryService.mjs` ~1688) defaults to the `private` policy. That adds an owner clause, `json_extract(memory.data, '$.properties.userId') = caller`, so a request for *another* identity returns `count: 0` by construction. `legacy` has the same effect for that identity's own rows (`userId IN (caller, shared)`). The empty result looks exactly like "this agent has no turns", which is the wrong conclusion and the one a recovering or investigating agent will draw.

## The Architectural Reality

- **Design authority:** the function already names its scope whenever its own filters force an empty answer. `return {_channelSeparation: channelSeparation, count: 0, turns: [], nextCursor: null, scope: 'fail-closed: no resolvable tenant'}` (`MemoryService.mjs:1701`), and the same for `'fail-closed: no resolvable agent identity'` (`:1729`). The silent case breaks that contract.
- **Policy semantics stay as they are.** `private` is the deliberate recovery default ("Default recovery is tenant-scoped; explicit wider sharing is policy-clamped and public-summary-only", the method's JSDoc).
- **Tests:** `test/playwright/unit/ai/services/memory-core/QueryRecentTurns.spec.mjs` runs on the real in-memory SQLite graph.

## The Fix

In `queryRecentTurns`, when a page comes back **empty** for an identity other than the caller's under an owner-scoped policy (`private` or `legacy`), the response carries a **`hint`**. It names the restriction and the next step: `memorySharing: 'team'`, or, when the deployment clamps sharing, that the rows are unreadable there.
- Rows, `count` and the policy are unchanged.
- **Why `hint` and not `scope` (author correction at implementation):** `ai/services/fleet/fleetRecentTurnsSource.mjs` treats any `scope` string as the operation's refusal and discards the page as `recent-turns-scope-refused`. `scope` therefore stays reserved for refusals. `hint` is the existing informational key, already used in `add_memory`'s visibility envelope.
- **Why only on empty pages:** `userId` is the tenant, so identities that share a tenant read each other's rows under `private`. A non-empty page for another identity is real and needs no hint.

**On a `team` deployment, a public-summary read of another identity without `memorySharing` takes the `team` policy** (operator direction, 2026-10-14: *"we have team sharing mode enabled … you should get results"*; friction → gold).
- The read takes the shared path in full: the `memorySharing` verdict is returned, and an unreadable WAL fails rather than claim an empty stream.
- Unchanged:
  - recovery of one's own identity stays tenant-scoped;
  - full or private projections of another identity stay `private`;
  - on `private` and `legacy` deployments, the no-parameter path stays exactly #741's (its isolated probe stays green).
- Opt-in for a read every caller is already entitled to on a `team` deployment is no security boundary; it only produced silent zeros. Access stays governed by the configured policy.
- This amends the default-path line of #741 and #12671's AC-7a for `team` deployments only. Isolation under `private` is still asserted.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Evidence |
|---|---|---|---|---|
| `query_recent_turns` response `hint` | `MemoryService.queryRecentTurns` | present on an empty page for another identity under `private`/`legacy`; names the restriction and `memorySharing: 'team'` (or the clamp) | absent on non-empty pages, self reads and `team` reads | `QueryRecentTurns.spec.mjs` (#1019 case) |
| `scope` (unchanged) | the same method, as a refusal marker | no new emitters | — | Fleet recent-turns specs stay green |
| No-parameter read of another identity | `MemoryService.queryRecentTurns` + `aiConfig.memorySharing.defaultPolicy` | on `team` deployments, public summaries resolve as `team` (verdict returned) | `private`/`legacy` deployments, self reads and full/private projections are unchanged | `QueryRecentTurns.spec.mjs` AC7a, #1019 and unreadable-WAL cases; #741's private/legacy probe |

## Acceptance Criteria

- [ ] Caller B reading identity A (a different tenant) under `private` gets `count: 0` and a `hint` naming the owner restriction and `memorySharing: 'team'`, and no `scope`.
- [ ] The same read under `team` returns A's turn, with no `hint`.
- [ ] B reading `@me` gets no `hint` (no regression).
- [ ] `legacy` plus a foreign identity carries its own `hint` text.
- [ ] A tenant-mate's rows (same `userId`, different identity) return under `private` with no `hint`.
- [ ] The existing `QueryRecentTurns` and Fleet recent-turns specs stay green.
- [ ] On a `team` deployment, a no-parameter public-summary read of another identity returns its turns with the `team` verdict; an explicit `private` read stays isolated.
- [ ] Under the `team` default, an implicit foreign read fails on an unreadable WAL like an explicit shared read, while recovery keeps its graph rows.
- [ ] On `private` and `legacy` deployments, the no-parameter foreign read is unchanged (#741's isolated probe).
- [ ] With the graph absent, an implicit `team` foreign read fails like an explicit one (`RECENT_TURNS_ERROR`), while recovery keeps its empty fallback (Euclid's owned-surface read).

## Out of Scope

- Changing the configured default policy or the owner clauses.
- The same silent-zero class on other tools: `query_raw_memories` own-scoping (June), and `get_node` returning null for another agent's memory. Those are follow-ups if this shape is accepted.

## Avoided Traps

- **Applying the configured default on `legacy` deployments too:** that would widen the no-parameter foreign read to unowned rows, which no deployment asked for. Only `team` declares peers' summaries shared.
- **Turning the 0 into an error, or into a `scope`:** callers that page or count would break, and the Fleet source would discard real pages as refusals.

## Related

Brain #1018 (where today's instance surfaced) · D#19733

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-14T11:02:52Z; no equivalent. `gh search issues --owner neomjs` for empty-result hint / scope / zero signal: none. A2A in-flight sweep (last 8): no overlapping claim. MC rationale sweep: the 06-04 hint proposal (never filed) and Phoebe's 07-26 record. Own-assignment sweep: #1018 is related but a different surface.
Decision Record impact: none.
Retrieval Hint: "query_recent_turns private policy foreign identity silent zero scope hint"
Origin Session ID: 530763bc-91d2-4aea-926a-00bf6ab58030

🖖 Grace (Opus 5.5, Claude Code)


## Timeline

- 2026-10-14T11:03:27Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-14T11:03:28Z @neo-opus-grace added the `bug` label
- 2026-10-14T11:03:28Z @neo-opus-grace added the `ai` label
- 2026-10-14T11:03:29Z @neo-opus-grace added the `model-experience` label
- 2026-10-14T11:03:29Z @neo-opus-grace added the `agent-os` label
- 2026-10-14T11:08:21Z @neo-opus-grace cross-referenced by PR #1020
- 2026-10-14T11:19:12Z @neo-opus-grace referenced in commit `b8c8839` - "feat(memory-core): on a team deployment, another identity's recent turns follow the sharing default (#1019)"
- 2026-10-14T11:50:52Z @neo-opus-grace referenced in commit `785b82d` - "fix(memory-core): an implicit team recency read fails like an explicit one when the graph is absent (#1019)"
- 2026-10-14T14:01:15Z @tobiu referenced in commit `5af8fd2` - "feat(memory-core): another identity's recent turns follow the team default, and empty owner-scoped pages carry a hint (#1019) (#1020)

* fix(memory-core): an empty owner-scoped recency page for another identity carries a hint (#1019)

* feat(memory-core): on a team deployment, another identity's recent turns follow the sharing default (#1019)

* fix(memory-core): an implicit team recency read fails like an explicit one when the graph is absent (#1019)"
- 2026-10-14T14:01:15Z @tobiu closed this issue

