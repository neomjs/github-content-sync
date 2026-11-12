---
id: 1619
title: A merge wakes the seats whose blocked tickets it unblocks
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T23:00:49Z'
updatedAt: '2026-11-12T08:07:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1619'
author: neo-opus-grace
commentsCount: 1
parentIssue: 759
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
# A merge wakes the seats whose blocked tickets it unblocks

## Context

On 2026-11-11 the operator confirmed that merge messages should wake: "in case one merge unblocks other lanes and peers are idle waiting for this to happen." Since #1610, every open-work message wakes its recipient, the `post-merge` kind included.

A `post-merge` wake goes to one seat, the merged PR's author (`ai/services/fleet/openWorkWakes.mjs:213`: `wakeOf(transition, transition.owner.seat, 'author', 'post-merge', …)`). The seats the operator means hold tickets that the merge unblocked, and nothing wakes them. Memory Core shows the cost recurring: seats park "until PR #11704 merges to unblock the next phase" and stay parked after it merges.

## The Problem

A merge that unblocks another lane is invisible to that lane's owner until a human or a heartbeat happens to look. The open-work producer reads no dependency: nothing under `ai/services/fleet/` names `blocked_by`, `blockedBy` or `blocking`.

## The Architectural Reality

- The producer already observes each merge as a terminal transition (`openWorkReducer.mjs`, `transition(…, 'merged', 'open', 'merged', pulse)`), and the round turns it into the author's `post-merge` wake.
- GitHub records the dependency natively, and one query resolves it. Receipt at 2026-11-11T23:0xZ: neomjs/neo PR #20791 (`merged=true`) → `closingIssuesReferences` #20788 → `blocking` #20790 (OPEN, assignee `neo-opus-grace`).
- #1607 (PR #1613) moves open-work wakes to a plane intake that composes fixed text from tokens and sends as `@system`. A new kind rides it.

## The Fix

On a merged transition, the producer also resolves what the merge unblocks:
1. Merged PR → `closingIssuesReferences` → each closed issue's `blocking` issues that are still open.
2. A blocked issue counts as unblocked only when its `blockedBy` holds no other open issue. One merge of two blockers wakes nobody yet.
3. Each assignee of an unblocked issue that maps to a registered seat gets one `unblocked` wake, with fixed text naming the issue and the merged PR, through the #1607 intake. The episode is keyed by issue and merge, so a re-observed merge does not wake twice.
4. Cost: one query per merged PR, outside the per-pulse snapshot.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `OPEN_WORK_WAKE_REASONS` and the observation check (existing, `ai/services/fleet/openWorkWakeMessage.mjs`) | #1607: tokens, never text | a new kind, `unblocked`, and one new token, `unblocks`: the unblocked issue as `owner/name#N`, required for that kind and refused for every other | the episode is the merge's episode plus the issue key, so two issues unblocked by one merge are two tasks | the module's JSDoc | `openWorkWakeMessage.spec.mjs` |
| `admit_open_work_wake`'s input (existing, `ai/mcp/server/memory-core/openapi.yaml`, `additionalProperties: false`) | #1607 | accepts the optional `unblocks` string | an older host never sends it; a newer host against an older plane is refused, so the plane rolls first, as for #1613 | the tool's description | `openWorkWakeIntake.spec.mjs`, `OpenApiValidatorCompliance.spec.mjs` |
| the producer's unblock read (new: one static GraphQL document per merged PR, outside the snapshot) | Fix 1–2 and the receipt above | merged PR → `closingIssuesReferences` → each closing issue's open `blocking` issues → each one's open `blockedBy` | an open `blockedBy` entry that is one of the merge's own closing issues does not count, since GitHub closes them just after the merge; a failed read plans nothing and retries on the next pulse | the document's JSDoc and its `OPEN_WORK_QUERY_COSTS` entry | `openWorkProducer.spec.mjs` |
| `planOpenWorkWakes`, merged transitions (existing, `ai/services/fleet/openWorkWakes.mjs`) | #761: one wake per holder change | each assignee of an unblocked issue that maps to a seat gets one `unblocked` wake | an unassigned issue, or an assignee with no seat, plans none; a re-observed merge plans nothing new; a quiet round sends nothing | the module's JSDoc | `openWorkWakes.spec.mjs` |

## Acceptance Criteria

- [ ] Unit, on the producer's resolution and plan:
  - A merge that closes the last open blocker of an assigned issue plans one `unblocked` wake to the assignee seat.
  - A merge that leaves another open blocker plans none.
  - A blocker that is another closing issue of the same merge, still open when read, does not count.
  - An unassigned blocked issue, or an assignee with no seat, plans none.
  - Re-observing the same merge plans nothing new.
- [ ] Unit, on the intake contract: `unblocked` validates and composes like the other kinds (fixed text, tokens only, normal priority, wake on); its `unblocks` token is required for it and refused for every other kind.
- [ ] *(post-merge, installed)* A real merge that closes a blocker wakes the blocked ticket's assignee seat, as seen in the FM feed.

## Out of Scope

- Stacked pull requests whose base was the merged branch: their authors' retarget is a different action.
- Unassigned blocked tickets, and dependencies recorded only in prose (no `blocked_by` relationship).
- The author's own `post-merge` wake, which stays as it is.

## Related

Parent #759 · #1610 (every kind wakes) · #1607 / PR #1613 (the intake) · #807 (escalations, same producer)

Decision Record impact: none.

Live latest-open + closed sweep: the latest 20 open (red band; the owning epic is #759, so this is filed as its leaf) and a state-all search (`unblock wake OR "blocked_by" merged wake OR dependents merge wake`; no equivalent) at 2026-11-11T23:00Z. A2A in-flight claim sweep: the latest 12 messages, back to 22:09Z, carry no claim on this surface. MC sweep: the waiting pattern above, no prior decision. Own-assignment sweep: #807, #1607, #1614, no overlap.

Intake, 2026-11-12: the drift probe since creation hit `openWorkWakes.mjs` and `openWorkWakeMessage.mjs`, because PR #1613 moved the wake to the plane intake, so the full gate ran. The premise holds: nothing in the producer reads a dependency. Prescription checked: `ai/services/fleet/openWorkWakes.mjs` owns the plan and `ai/services/fleet/openWorkWakeMessage.mjs` the contract. The intake's tokens all name a pull request, which is why the ledger adds `unblocks`. #1629 (PR #1638) lands first in `openWorkWakes.mjs`.

handoff: @neo-opus-grace self-assigns, as owner of #759.

Origin Session ID: 15ea444b-707c-4458-bba5-79b4f61fb280

Retrieval Hint: "merge wakes the assignees of tickets it unblocks; blocked_by closingIssuesReferences blocking; idle peers waiting for a merge"

## Timeline

- 2026-11-11T23:00:49Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-11T23:00:51Z @neo-opus-grace added the `enhancement` label
- 2026-11-11T23:00:51Z @neo-opus-grace added the `ai` label
- 2026-11-11T23:00:51Z @neo-opus-grace added the `agent-os` label
- 2026-11-11T23:01:05Z @neo-opus-grace added parent issue #759
- 2026-11-12T00:15:49Z @neo-opus-grace cross-referenced by #1620
- 2026-11-12T03:19:08Z @neo-opus-grace cross-referenced by #1628
- 2026-11-12T04:28:15Z @neo-opus-grace cross-referenced by PR #1633
- 2026-11-12T04:49:39Z @neo-opus-grace cross-referenced by PR #1638
### @neo-opus-grace - 2026-11-12T08:07:11Z

**Handover (Grace, session sunset, 12 Nov 08:1xZ).** Owner: Grace; the next session of this seat continues.

**Done, on `grace/1619-merge-wakes-unblocked-seats` @ `b8b1cb69`, pushed with no PR yet:** the intake half, as the Contract Ledger's first two rows describe.
- `unblocked` is a kind in `OPEN_WORK_WAKE_REASONS`.
- `unblocks` (`owner/name#N`) is required for that kind and refused for every other.
- `unblockedEpisodeOf(mergeEpisode, issue)` mints `<merge transition id>|<issue>`, which `validEpisode` checks.
- The composed wake leads with the issue and names the merge after it.
- `admit_open_work_wake`'s strict schema declares `unblocks`.
- Specs: `openWorkWakeMessage.spec.mjs` (two new arms; the per-kind arms build a valid observation through `valid(kind)`), and `openWorkWakeIntake.spec.mjs` end to end through `callTool`. The red-first run failed for the right reason: "Unrecognized key: unblocks".

**Left, in order:**
1. **Rebase onto `dev` once #1638 (#1629) merges.** It changes `openWorkWakes.mjs` and `openWorkQueries.mjs` (it adds a cost-table entry), and both files are next on this branch.
2. **The producer's unblock read.** One static GraphQL document (merged PR → `closingIssuesReferences` → each closing issue's open `blocking` → each one's open `blockedBy`, plus assignees), run per newly merged transition with the merged PR author's reader, recorded in the quota tally. Give it its own `OPEN_WORK_QUERY_COSTS` entry and keep the `rateLimit` block, so `wakeGateOf` prices it. Ledger row 3 has the race rule: an open blocker that is one of the merge's own closing issues does not count.
3. **The planner.** In `planOpenWorkWakes`, for merged transitions: one `unblocked` wake per (assignee seat, issue), deduped like `pendingMerges`; a quiet round sends nothing. Then pass `unblocks` through `observationOf` in `wireFleetOpenWorkWakes.mjs`.
4. Write the AC-1 specs red-first, then open the PR. Deploy order: the plane first, then the FM host, as for #1613.

🖖 Grace (Claude Opus 5.5, Claude Code) · session a0126f58-6ebb-41be-83b7-1371cf98480f


