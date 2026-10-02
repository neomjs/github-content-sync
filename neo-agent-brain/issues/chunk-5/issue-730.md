---
id: 730
title: A start's per-repository outcome stays on the seat's launch record and reaches the roster
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T20:27:12Z'
updatedAt: '2026-10-02T08:23:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/730'
author: neo-opus-ada
commentsCount: 1
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
closedAt: '2026-10-02T08:23:58Z'
---
# A start's per-repository outcome stays on the seat's launch record and reaches the roster

## Context

neomjs/neo-agent-institution#408 shows each of a seat's repositories with its last start outcome (prepared, or failed with the reason) on the Accounts Repositories card (neomjs/neo-agent-institution#412). The Brain computes that outcome at every provisioned start, then keeps it nowhere a later read can see.

## The Problem

`startAgentProvisioned` clones each of `metadata.repos` before the launch and returns `repos: [{repoSlug, state: 'prepared' | 'failed', reason?}]` in the start's answer only (`ai/services/fleet/startAgentProvisioned.mjs`: the loop at :284-291, the status at :348-351). No later read carries it:
- `FleetLifecycleService.status(id)` (:905) reports the launch, and its wake route since #705, but not the repositories;
- `inspectFleetRepos` inspects only the working repository (`metadata.repo`);
- the roster DTO (`fleetCockpitStatus`) joins only those two.

So a surface that did not press Start, or reads after it, can never show why a repository is missing from the seat's checkout.

## The Architectural Reality

- **Precedent:** #705 keeps a launch's wake route on the lifecycle's process record: `setWakeRoute(id, route, {pid, startedAt})` (`FleetLifecycleService.mjs:962`) records it only for the launch it was armed for, and `status(id)` reports it.
- The process record lives in the Fleet owner process, so it survives roster reads and app reloads. A Fleet restart drops it, and the next start writes it again.
- `fleetCockpitStatus` builds each roster row from `runtimeByAgentId` (the `status` answers), so a field `status` reports can ride the row.
- The failed reason is already credential-redacted and bounded at the source (`redactReadFailure`, #683).

## The Fix

- `FleetLifecycleService.setRepoOutcomes(id, repos, {pid, startedAt})` beside `setWakeRoute`, with the same launch binding: it is refused for an unknown seat or a launch the seat has since replaced.
- `status(id)` reports `repos` (`null` when the launch recorded none).
- The provisioned start records its `repos` with the pid and `startedAt` of the status it returns.
- The roster row carries the outcome through the runtime join.

**Sequencing:** `FleetLifecycleService.mjs` is in the write-surface of #727, #728 and #729. This leaf is one additive method plus a `status` field, so it lands after them or rebases onto whichever merged first.

## Acceptance Criteria

- [ ] AC-1: after a provisioned start, `status(id).repos` holds that launch's `[{repoSlug, state, reason?}]`. A later start replaces it, and a write for a replaced launch or an unknown seat is refused.
- [ ] AC-2: the roster DTO row carries it, so a cockpit roster read shows each repository's last outcome.
- [ ] AC-3: no reason carries a credential (the source's redaction, asserted on the recorded value).

## Contract Ledger

The Brain produces and keeps the outcome. neomjs/neo-agent-institution#408 only renders it.

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Writer: `startAgentProvisioned` → `setRepoOutcomes(id, repos, {pid, startedAt})` | the start's own loop over `metadata.repos` | once `spawnPermitted` returns, records `[{repoSlug, state, reason?}]` on the process record of the launch that returned that `pid` + `startedAt` | no other repositories → no write. A preparation error, a released seat or a throwing spawn ends the start before the write, so nothing is recorded. | `setRepoOutcomes` JSDoc | `startAgentProvisioned.spec`: the start hands its `repos` over with pid 4242 |
| Process record → `status(id).repos` | the launch binding of #705's `setWakeRoute` | keeps only `repoSlug`, `state` and `reason`; `status` returns copies | an unknown seat or a replaced launch → `false`, nothing written. Before a recorded start, or with no record at all → `null`. | `setRepoOutcomes` JSDoc | `FleetLifecycleService.spec`, the `setRepoOutcomes` describe (2 arms) |
| Lifetime: reset and adoption | the record, held in the Fleet owner process's memory | a fresh start writes a fresh record (`processes.set`), so the next spawned start replaces the outcome. Stop and exit keep the record, so a stopped seat still reports its last start. | a Fleet restart drops it. A re-adopted seat (`adoptLeasedSeat`; the lease carries no outcome) reports `null` until its next start, as `wakeRoute` does. | this ledger | replacement: AC-1's arm. Stop/exit and adoption: code only (`adoptLeasedSeat` builds its record without `repos`, and no path deletes a record); no arm |
| `fleetRuntimeStatus` `repos` → roster row `repoOutcomes` | `status(id)` | passed through whole | the runtime row omits `repos` when it is `null`; the roster row then carries `repoOutcomes: null` | the roster row's inline comment | `FleetManager.spec` pass-through arm; `fleetCockpitStatus.spec` `repoOutcomes` arm |
| Reason redaction | the source: `redactReadFailure` (#683) | credentials redacted, whitespace collapsed, bounded to 240 characters; `'no legible error'` when nothing legible remains | `setRepoOutcomes` drops every other key, so a `cloneUrl` carrying `user:token@` never reaches the record | `redactReadFailure` JSDoc | `startAgentProvisioned.spec`: "a failed repository's reason carries no credential and stays bounded…". `FleetLifecycleService.spec`: the extra `cloneUrl` reads back as nothing |

## Out of Scope

- The Institution card (neomjs/neo-agent-institution#408).
- An on-disk check of each declared repository (`inspectFleetRepos` for `metadata.repos`). The start outcome answers #408. A disk check would be its own leaf if a surface needs presence between starts.

## Related

#683 / #682 (`repos` at start) · #705 (the `setWakeRoute` precedent) · #727 / #728 / #729 (same file) · neomjs/neo-agent-institution#408 (consumer) · neomjs/neo-agent-institution#412

## Sweeps

- Live latest-open sweep: the latest 20 open Brain issues at 2026-10-01T20:26:30Z. No equivalent.
- Keyword: "start outcome repository" across neomjs. Only neomjs/neo-agent-institution#408, the consumer.
- A2A: recent claims are #727 / #729 (Grace) and #728 (Vega), all in the same file and none on this concern.
- MC sweep: one query on the problem's nouns (a start's per-repository outcome lost after the answer), 4 results, no prior decision.
- Own-assignment sweep (taken 18:54Z, not re-run): 20 open Brain issues. #571 and #142 touch the Fleet, but not this concern.

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a

Authored by Ada (Claude Opus 5.5, Claude Code).



## Timeline

- 2026-10-01T20:27:14Z @neo-opus-ada added the `enhancement` label
- 2026-10-01T20:27:14Z @neo-opus-ada added the `ai` label
- 2026-10-01T20:27:14Z @neo-opus-ada added the `agent-os` label
- 2026-10-01T20:27:15Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T20:27:35Z @neo-opus-ada cross-referenced by #408
- 2026-10-01T20:32:22Z @neo-opus-ada cross-referenced by PR #731
- 2026-10-01T20:39:48Z @neo-opus-ada referenced in commit `c742171` - "docs(fleet): the activity-identity comment names its aliasing without a ticket-shaped number (#730)

The shared baseline's archaeology gate reads every changed file whole, and this pre-existing comment's example read as two ticket references. The wording keeps its meaning."
### @neo-opus-ada - 2026-10-01T21:05:31Z

**Handover (session sunset, 2026-10-01 ~21:15Z) · owner: @neo-opus-ada**

PR #731 at `c742171` is in review with @neo-gpt-emmy as primary: 19/19 green, CLEAN. The Contract Ledger in the body was added at her intake request. One gap is already disclosed to her: `FleetLifecycleService.status()`'s `@returns` list doesn't name `repos` yet. Fold it into the one commit that answers her review.

Downstream: neomjs/neo-agent-institution#408's Institution half is pushed (`ada/408-repo-outcomes`) and waits on this merge and a pin.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-10-02T08:10:55Z @neo-opus-ada referenced in commit `34e0401` - "feat(fleet): a start's per-repository outcome stays on its launch record and reaches the roster (#730)

FleetLifecycleService.setRepoOutcomes records the provisioned start's repos on the launch's process record, bound by pid and startedAt like setWakeRoute; status(id) reports it, fleetRuntimeStatus passes it through, and the cockpit roster row carries it as repoOutcomes. The start's own answer is unchanged."
- 2026-10-02T08:10:56Z @neo-opus-ada referenced in commit `12e444b` - "docs(fleet): the activity-identity comment names its aliasing without a ticket-shaped number (#730)

The shared baseline's archaeology gate reads every changed file whole, and this pre-existing comment's example read as two ticket references. The wording keeps its meaning."
- 2026-10-02T08:10:56Z @neo-opus-ada referenced in commit `2a915b6` - "docs(fleet): status() names the repos outcome it reports (#730)"
- 2026-10-02T08:23:58Z @tobiu referenced in commit `ba68ac3` - "feat(fleet): a start's per-repository outcome stays on its launch record and reaches the roster (#730) (#731)

* feat(fleet): a start's per-repository outcome stays on its launch record and reaches the roster (#730)

FleetLifecycleService.setRepoOutcomes records the provisioned start's repos on the launch's process record, bound by pid and startedAt like setWakeRoute; status(id) reports it, fleetRuntimeStatus passes it through, and the cockpit roster row carries it as repoOutcomes. The start's own answer is unchanged.

* docs(fleet): the activity-identity comment names its aliasing without a ticket-shaped number (#730)

The shared baseline's archaeology gate reads every changed file whole, and this pre-existing comment's example read as two ticket references. The wording keeps its meaning.

* docs(fleet): status() names the repos outcome it reports (#730)"
- 2026-10-02T08:23:58Z @tobiu closed this issue

