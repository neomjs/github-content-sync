---
id: 1358
title: Reconcile deleted source conversations in the corpus
state: OPEN
labels:
  - bug
  - ai
  - build
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-29T15:49:08Z'
updatedAt: '2026-10-29T17:34:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1358'
author: neo-gpt-sophie
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
---
# Reconcile deleted source conversations in the corpus

## Problem
Ordinary corpus synchronization fetches changed conversations by an update-time delta. A deleted GitHub discussion supplies no later update, so its cached Markdown, content-index entry and sync-metadata row can survive indefinitely. The same deletion-detection question applies to issues; verify each facet's semantics before extending the fix.

## Corrected disposition
The operator rejected PR #1359. Hard-coding incident numbers into Brain configuration is not the solution. That PR is closed without merge.

The immediate item was removed directly from the corpus in [f047ee3a](https://github.com/neomjs/github-content-sync/commit/f047ee3a12d979bea4f92dc925f01d6730345f8b): one Markdown file, one origin-qualified index entry and one metadata entry, in the same commit. Readback confirmed file 404 and both references absent. This is current-tree removal; Git history was not rewritten.

## Required behavior
Brain's corpus synchronization must reconcile confirmed source deletions without a maintained list of item numbers. Preserve content on incomplete pagination, failed requests or ambiguous access/transfer results. Remove all applicable corpus copies, the origin-qualified index entry and sync metadata together after deletion is established. Keep the existing delta path for changed content.

Prescription checked: `DiscussionSyncer.syncDiscussions` owns emission and cached removal; the rejected config default treated the symptom. `DiscussionReconciliationService` is prior art for distinguishing vanished inventory members from proved deletions. `TenantRepoSyncService` and `IngestionService.applyDeletionSignals` already own downstream corpus-path deletion; do not add a parallel database-deletion mechanism.

## Acceptance criteria
- [ ] A previously mirrored discussion confirmed deleted at GitHub disappears during ordinary corpus synchronization without a per-item config change.
- [ ] Its Markdown copies, index entry and metadata row are removed together; unrelated repository/type/number identities remain untouched.
- [ ] Failed or incomplete source reads and ambiguous access/transfer states cannot silently remove cached content.
- [ ] Focused tests cover confirmed deletion, unchanged delta-skipped items, incomplete inventory and repository-qualified removal.
- [x] The ordinary corpus-to-KB consumer removes deleted corpus paths. Independently verified in the local orchestrator log on 29 October at 17:25:16 UTC: `neo-shared/github-content-sync completed: head=f047ee3a ingested=56481 deleted=5`; the corresponding materialization reported `errors=0`. This verifies the immediate cleanup on this deployment plane. The upstream source-reconciliation criteria above remain open.

## Scope and ownership
Sophie owns this corrected deletion-reconciliation outcome and its downstream verification. The immediate direct removal is complete; generic source-deletion detection is not implemented by that commit. No new configuration leaf, per-item denylist additions, Git-history rewrite, force push or manual database mutation.

Decision Record impact: existing sync and deletion lifecycle; no new policy authority.
Live issue/source and A2A duplicate sweeps on 29 October found no other current owner for this corpus gap. Vega's independent sweep agrees. Implementation must distinguish provider-confirmed deletion from ordinary NOT_FOUND ambiguity.

Origin Session ID: 8d829855-3e60-4921-833f-766417f5c2a8
Retrieval Hint: corpus automatic source deletion reconciliation delta discussion issue


## Timeline

- 2026-10-29T15:49:08Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-29T15:49:09Z @neo-gpt-sophie added the `bug` label
- 2026-10-29T15:49:09Z @neo-gpt-sophie added the `ai` label
- 2026-10-29T15:49:10Z @neo-gpt-sophie added the `build` label
- 2026-10-29T15:56:02Z @neo-gpt-sophie cross-referenced by PR #1359
- 2026-10-29T16:20:08Z @neo-gpt-sophie changed title from **Quarantine deleted discussion 20175 from the corpus** to **Reconcile deleted source conversations in the corpus**
### @neo-gpt-sophie - 2026-10-29T16:25:16Z

Direct cleanup is complete in [github-content-sync f047ee3a](https://github.com/neomjs/github-content-sync/commit/f047ee3a12d979bea4f92dc925f01d6730345f8b): only the target Markdown file (244 lines), one index entry (8 lines), and one metadata entry (8 lines) were removed. Live readback confirms file 404 and no corresponding index/metadata row. PR #1359 is closed, unmerged; its configuration prescription was rejected.

Downstream verification remains open. The last scheduled consumer receipt was `2026-10-29T16:17:29Z`, `head=322112a5 ingested=56486 deleted=0`, before it saw the removal. A scoped call to the installed `syncTenantRepos.mjs --repo-slug github-content-sync` hit the default Node heap limit (exit 134). The scheduler itself runs with `--max-old-space-size=6144`; retrying the same CLI with that observed budget respected the existing lease and deferred behind the summary task (exit 4). No service restart, lease override, per-item ingestion payload or manual database deletion occurred.

The existing consumer's next receipt must show the removal revision or a descendant and its deletion result. Vega has a passive watch on the ordinary scheduled sweep. The generic upstream deletion gap remains separate from this immediate corpus-to-KB verification.

Sophie (GPT-6 Astra, ultra, Codex Desktop) · session 8d829855-3e60-4921-833f-766417f5c2a8


