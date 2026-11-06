---
id: 1506
title: One-shot scripts leave the Brain once their run is done or impossible
state: OPEN
labels:
  - ai
  - refactoring
  - agent-os
  - tech-debt
assignees:
  - neo-opus-grace
createdAt: '2026-11-06T10:53:30Z'
updatedAt: '2026-11-06T12:16:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1506'
author: neo-opus-grace
commentsCount: 1
parentIssue: 191
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
# One-shot scripts leave the Brain once their run is done or impossible

## Context

This is a leaf of #191, "Delete legacy Brain surfaces within domain slices". @tobiu, 2026-11-06, on #1502, verbatim: "there MUST be a task to USE the logic, and then retire it." And: "i would even guess some of them got never executed."

Going forward, neomjs/neo-agent-skills#292 / #293 has every new one-shot PR file its retirement ticket before review. This leaf retires the backlog.

The epic review is Maryam's premise read, verdict "holds": https://github.com/neomjs/neo-agent-brain/issues/191#issuecomment-6591189720

## The Problem

A census on `origin/dev` (2026-11-06). It uses a Perl import grep with a positive control, so a mention in JSDoc is not counted as a caller.

- **`ai/scripts/migrations/`:** 17 of the 18 modules (7,321 lines) have no production importer and no npm script.
- **`ai/scripts/maintenance/`:** 13 of the 54 scripts have no reference outside their own tests. Among them are clear one-shots:
  - `retireLegacyConversationRows.mjs` (397 lines; #417 closed 2026-10-12, then still edited by #1369);
  - `repairUnprojectedSessions.mjs` (412 lines; its JSDoc calls it a "one-time hard-orphan repair");
  - `repairRawMemoryRecencyKeys.mjs` (268 lines).
- **Code that only these scripts use:** `SyncService` documents three methods as "invoked out-of-band" by `refetchStalePulls`, `refetchStaleDiscussions` and `rebucketArchive`. `GraphService.mjs` l.178–180 registers three migration paths.
- **At least one may never have run:** the ADR identity migration (`migrateLegacyAdrIdentities.mjs` + `adrMigrationManifest.mjs`, 2,175 lines after #1237 compacted them instead of deleting them) has no live-cut receipt on #1010. That issue's last comments (2026-10-13, 2026-10-21) still describe the cut as gated.

Static-import silence is not proof. #191 names that as an avoided trap, so every deletion needs its own evidence.

## The Architectural Reality

- `bootstrapWorktree.mjs` sits in `migrations/` but is product code: `ai/services/fleet/prepareManagedAgentWorkspace.mjs` imports it.
- `repairMemoryCoreStoredEmbeddings.mjs` is product code too: `reEmbedMissingHeal.mjs` imports it.
- The heavy-maintenance lease census and the `GraphService` registry name script paths, so a deletion updates them in the same change.

## The Fix

The PR carries one row per script, with its evidence and its disposition:
- **Run done** (a receipt on a ticket or PR, or a Memory Core turn): delete the script, its specs, the service code only it uses, and its registry or census entries.
- **Can never run** (its precondition is gone, such as a data shape that no longer exists): delete the same set.
- **Run still pending** (for example #1010's ADR cut, #419's core rows): keep it, and name its run ticket and its retirement ticket. Missing ones are filed, and the retirement ticket is `blocked_by` the run.

## Acceptance Criteria

- [ ] **AC-1:** every module in `ai/scripts/migrations/`, and every one-shot in `ai/scripts/maintenance/` (retire, repair, backfill, purge, rebuild, recreate, migrate), has a row: evidence, then disposition.
- [ ] **AC-2:** the "run done" and "can never run" rows are deleted, together with their specs, service-only code and registry or census entries. The Brain unit suite stays green.
- [ ] **AC-3:** every "run still pending" row names its run ticket and its retirement ticket.
- [ ] **AC-4:** no deletion rests on static-import silence alone; each row cites a run receipt or the reason it cannot run.

## Out of Scope

- Diagnostics without a consumer (other #191 slices).
- Moving the product code misfiled in `migrations/`.
- #1502's own retirement ticket, which belongs to its author.

## Related

#191 (parent), #1502, #1237, #1010, #419, #417, neomjs/neo-agent-skills#292.

Live latest-open + closed sweep at 2026-11-06 ~10:56Z: the latest 8 and 20 open, plus an all-state search for "retire migrations / one-shot scripts / migration scripts". No equivalent leaf exists; #191 is the parent, and #1237 closed as a compaction.
- A2A: no claim on this scope in the last 30 messages.
- Memory Core: no prior decision.
- Own assignments: #684 and #54, on other surfaces.

Ownership: self-assigned.

Origin Session ID: a33fc3c4-7e98-4300-985d-25a10e216b50
Retrieval Hint: "one-shot migration scripts retire run receipt ai/scripts/migrations census #191"

🖖 Grace (Opus 5.5, Claude Code)

## Timeline

- 2026-11-06T10:53:30Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-06T10:53:32Z @neo-opus-grace added the `ai` label
- 2026-11-06T10:53:32Z @neo-opus-grace added the `refactoring` label
- 2026-11-06T10:53:32Z @neo-opus-grace added the `agent-os` label
- 2026-11-06T10:53:33Z @neo-opus-grace added the `tech-debt` label
- 2026-11-06T10:53:38Z @neo-opus-grace added parent issue #191
### @neo-opus-grace - 2026-11-06T12:16:53Z

**Handover (Grace, session a33fc3c4, 2026-11-06): filed and claimed; the census has not started.**

Pickup:
1. List every module in `ai/scripts/migrations/` and every one-shot in `ai/scripts/maintenance/` (retire, repair, backfill, purge, rebuild, recreate, migrate).
   - Find importers with a Perl `git grep -P` plus a positive control. POSIX `-E` has no `\s`.
   - A path string in a registry is not an import: `GraphService.mjs` l.178–180 and the lease census name scripts without importing them.
2. For each script, find its run receipt on the ticket or PR that introduced it, or in Memory Core. One evidence line each: run done, can never run, or pending.
3. "Pending" known so far: `migrateLegacyAdrIdentities.mjs` + `adrMigrationManifest.mjs` (#1010 has no live-cut receipt), and #419's core rows. Ask their owners (Sophie, Euclid) before touching them.
4. Delete the "done" and "never" rows, together with:
   - their specs;
   - service code only they use: `SyncService`'s three out-of-band methods for `refetchStalePulls`, `refetchStaleDiscussions` and `rebucketArchive`;
   - their registry and census entries.

   Every deleting commit carries `spec-retired:` lines, and `check-spec-retirement` runs before the first push.

Measured starting point: 17 of the 18 migration modules (7,321 lines) have no production importer. `bootstrapWorktree.mjs` is product code and stays.


- 2026-11-06T12:57:41Z @neo-gpt-euclid cross-referenced by #1507
- 2026-11-06T12:59:20Z @neo-gpt-euclid cross-referenced by #419

