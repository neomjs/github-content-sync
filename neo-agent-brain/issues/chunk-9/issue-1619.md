---
id: 1619
title: A merge wakes the seats whose blocked tickets it unblocks
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T23:00:49Z'
updatedAt: '2026-11-12T14:49:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1619'
author: neo-opus-grace
commentsCount: 2
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
closedAt: '2026-11-12T14:49:17Z'
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
| the producer's unblock read (new: one static GraphQL document per merged PR, outside the snapshot) | Fix 1–2 and the receipt above | merged PR → `closingIssuesReferences` → each closing issue's open `blocking` issues → each one's open `blockedBy` | an open `blockedBy` entry that is one of the merge's own closing issues does not count, since GitHub closes them just after the merge; a closing, dependent or blocker list past its page makes the answer incomplete, a failed attempt that is never kept; a failed read plans nothing and retries on the next pulse, three attempts at most; only a merge observed in the last ten minutes is read, at most ten per pulse, so retained history is never read | the document's JSDoc and its `OPEN_WORK_QUERY_COSTS` entry | `openWorkProducer.spec.mjs` |
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

- 2026-11-12T13:46:07Z @neo-opus-grace referenced in commit `4f5c674` - "feat(fleet): the open-work intake admits an unblocked wake that names its issue (#1619)

The intake's tokens all named a pull request, so a wake could not say which ticket a merge unblocked. A new kind, unblocked, carries one more token, unblocks: the issue as owner/name#N, required for that kind and refused for every other. Its episode is the merge transition, then the issue, so one merge that unblocks two issues is two tasks. The composed wake leads with the issue and names the merge after it; admit_open_work_wake's strict schema declares the token. The producer's read and the planner follow on this branch."
- 2026-11-12T13:46:07Z @neo-opus-grace referenced in commit `52e3b67` - "feat(fleet): a merge wakes the assignees of the tickets it unblocked (#1619)

The producer reads, once per merge it observed in the last ten minutes, what that merge unblocked: the issues it closes, the open issues they block, and each one's assignees and remaining blockers. An issue counts as unblocked when every other blocker is closed or is another issue the same merge closes; a blocker list past its first page keeps it blocked. The answer rides on the merged transition as `unblock`. A failed read retries on the next pulse, three attempts at most, and at most ten merges are read per pulse; the read has its own cost entry so the wake gate prices it.

The planner turns each resolved issue into one `unblocked` wake per assignee seat, held with the merge's post-merge wake until its route answers and keyed by the pulse that resolved it, so a re-observed merge plans nothing new. The host round passes the issue token to the intake, accepts the new wakes in its ledger, and treats them like post-merge for freshness, since their pull request has left the open snapshot."
- 2026-11-12T13:46:11Z @neo-opus-grace cross-referenced by PR #1644
- 2026-11-12T14:19:04Z @neo-opus-grace referenced in commit `bd44bb7` - "fix(fleet): the unblock read keeps every dependent, and a list past its page is never a result (#1619)

Review RA-1 (Euclid): blocking(first: 10) dropped every dependent after the tenth and cached the shortened answer as the merge's result, so those assignees were never woken. The read now asks for 50 dependents and 50 blockers per issue, and every dependency list carries totalCount. A closing, dependent or blocker list past its page makes the answer incomplete: it counts as a failed attempt and is read again, never kept. The formula cost is 10, which a live read measured."
### @neo-opus-grace - 2026-11-12T14:40:50Z

**Handover (Grace, session sunset, 12 Nov 14:4xZ).** Owner: Grace; the next session of this seat continues.

**State:** PR #1644 at `bd44bb79` is approved (Euclid's Round 2, after RA-1: every dependency list is read whole, or the read is incomplete and retried). It is CLEAN, the required set is green, and the merge handoff went to the operator at 14:40Z. I am not merging it.

**After the merge, in order:**
1. **Plane cut first.** `admit_open_work_wake` gains `unblocks`, and an older plane refuses a newer host. The cut is the operator's go. If Ada runs the batched cut again, coordinate with her rather than cutting in parallel.
2. **Then the FM host update,** which carries the producer and host-round change.
3. **AC-3 receipt:** the first real merge that closes a blocker should wake the blocked ticket's assignee seat. Its `unblocked` wake must reach that seat, and the FM feed should show it. Post the receipt on #759.
4. **Clean up the worktree** `neo-agent-brain-1619` and its branch.

**Contract:** Ledger row 3 states the shipped bounds:
- 3 attempts per merge;
- 10 reads per pulse;
- only merges observed in the last 10 minutes;
- a list past its page makes the read incomplete.

The cost is 10 per merge, measured.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 5b51ef21-69c0-4872-8b70-31584800d698

- 2026-11-12T14:49:17Z @tobiu referenced in commit `fddae3b` - "feat(fleet): a merge wakes the assignees of the tickets it unblocked (#1619) (#1644)

* feat(fleet): the open-work intake admits an unblocked wake that names its issue (#1619)

The intake's tokens all named a pull request, so a wake could not say which ticket a merge unblocked. A new kind, unblocked, carries one more token, unblocks: the issue as owner/name#N, required for that kind and refused for every other. Its episode is the merge transition, then the issue, so one merge that unblocks two issues is two tasks. The composed wake leads with the issue and names the merge after it; admit_open_work_wake's strict schema declares the token. The producer's read and the planner follow on this branch.

* feat(fleet): a merge wakes the assignees of the tickets it unblocked (#1619)

The producer reads, once per merge it observed in the last ten minutes, what that merge unblocked: the issues it closes, the open issues they block, and each one's assignees and remaining blockers. An issue counts as unblocked when every other blocker is closed or is another issue the same merge closes; a blocker list past its first page keeps it blocked. The answer rides on the merged transition as `unblock`. A failed read retries on the next pulse, three attempts at most, and at most ten merges are read per pulse; the read has its own cost entry so the wake gate prices it.

The planner turns each resolved issue into one `unblocked` wake per assignee seat, held with the merge's post-merge wake until its route answers and keyed by the pulse that resolved it, so a re-observed merge plans nothing new. The host round passes the issue token to the intake, accepts the new wakes in its ledger, and treats them like post-merge for freshness, since their pull request has left the open snapshot.

* fix(fleet): the unblock read keeps every dependent, and a list past its page is never a result (#1619)

Review RA-1 (Euclid): blocking(first: 10) dropped every dependent after the tenth and cached the shortened answer as the merge's result, so those assignees were never woken. The read now asks for 50 dependents and 50 blockers per issue, and every dependency list carries totalCount. A closing, dependent or blocker list past its page makes the answer incomplete: it counts as a failed attempt and is read again, never kept. The formula cost is 10, which a live read measured."
- 2026-11-12T14:49:19Z @tobiu closed this issue
- 2026-11-12T16:49:06Z @neo-opus-grace cross-referenced by #807
- 2026-11-12T17:31:03Z @neo-opus-grace cross-referenced by #759

