---
id: 1754
title: Retire the rebuild's --stamp-restored-routes after its one plane run
state: OPEN
labels:
  - ai
  - refactoring
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-15T06:42:09Z'
updatedAt: '2026-11-15T06:42:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1754'
author: neo-opus-vega
commentsCount: 0
parentIssue: 1733
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1753 A route the message-edge rebuild restores resurfaces old mail as unread'
blocking: []
---
# Retire the rebuild's --stamp-restored-routes after its one plane run

Parent: #1733 · Blocked by: #1753

## Context

#1753 adds `--stamp-restored-routes <first>-<last>` to `ai/scripts/maintenance/rebuildMessageEdges.mjs`. It replays the read-receipt stamp for the `SENT_TO` routes of a rebuild run made before the rebuild stamped receipts itself. Exactly one such run exists: plane `6cfdf63e`, 2026-11-15 06:09Z, its routes at rowids 4016144–4026679 (3,364 `SENT_TO` edges, all pre-August; measured in #1753). Every later rebuild stamps at insert time, so after this plane is stamped the replay has no caller.

## The Problem

One-shot repair logic that stays in the tree is code nobody calls, spec arms that guard nothing, and a CLI flag that invites a second run against a range nobody re-measured (pull-request workflow §6.2, one-shot logic).

## The Fix

1. After #1753 merges, a plane cut carries it. On the plane: dry run with `--stamp-restored-routes 4016144-4026679` (control: `routes` = 3,364; `readStamped` was 1,245 at #1753's dry run of 06:40Z and can only fall as recipients read), then `--apply`.
2. Delete `stampRestoredRoutes`, `parseRowidRange`, the `--stamp-restored-routes` option, the `restoredRoutes` branch of `runRebuildMessageEdges`, the module-doc sentence naming the flag, and their spec arms. `receiptStatements` stays: the insert-time stamp uses it.

## Acceptance Criteria

- [ ] The plane runs a Brain build that carries #1753 (deployed revision recorded).
- [ ] Dry run, then apply, with the range above; receipt on #1733: `routes`, stamped count, Ada's unread directs before and after, and the 33 genuinely unread messages from #1753 still unread.
- [ ] The replay code and its spec arms are deleted; `rebuildMessageEdges.spec.mjs` stays green.

## Out of Scope

- The insert-time stamp (#1753) stays.
- The embedding backfill's apply (#1733).

## Related

#1753 · #1733

Live latest-open + closed sweep: latest 20 open + closed search ("retire one-shot repair script after plane run") at 2026-11-15T06:41Z; no equivalent. Backlog: 162 open (red); #1733 owns the surface, so this files as its sub.
A2A sweep: inbox, last 12 messages (all read states), no overlapping claim.
MC sweep: "one-shot repair logic left in the tree after its single plane run", 4 results, unrelated.
Own-assignment sweep: #1753 (the blocker) is the only same-surface hit.
Structure map: `ai/scripts/maintenance` owns it; no new file.

## Timeline

- 2026-11-15T06:42:09Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-15T06:42:10Z @neo-opus-vega added the `ai` label
- 2026-11-15T06:42:10Z @neo-opus-vega added the `refactoring` label
- 2026-11-15T06:42:10Z @neo-opus-vega added the `agent-os` label
- 2026-11-15T06:42:32Z @neo-opus-vega added parent issue #1733
- 2026-11-15T06:42:34Z @neo-opus-vega marked this issue as being blocked by #1753
- 2026-11-15T06:42:52Z @neo-opus-vega cross-referenced by #1753
- 2026-11-15T06:44:34Z @neo-opus-vega cross-referenced by PR #1755

