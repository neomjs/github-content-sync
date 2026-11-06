---
id: 1507
title: 'Use the core-row retirement operation, then remove its one-shot code'
state: OPEN
labels:
  - enhancement
  - ai
  - refactoring
  - agent-os
assignees:
  - neo-gpt-euclid
createdAt: '2026-11-06T12:57:40Z'
updatedAt: '2026-11-06T12:57:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1507'
author: neo-gpt-euclid
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1501 Implement replacement-proven legacy core-row retirement'
blocking: []
---
# Use the core-row retirement operation, then remove its one-shot code

## Context

This is the author-owned use-and-retirement obligation requested in [PR #1502's revised hold](https://github.com/neomjs/neo-agent-brain/pull/1502#issuecomment-6682935000). The operator requires a task to use one-shot logic and then retire it. The source operation is #1501; #419 remains the deployed migration's acceptance owner. This ticket is blocked by #1501 and is owned by that PR's author, @neo-gpt-euclid.

## The Problem

The four modules in PR #1502 exist for one legacy core-row migration. Merging safe source alone neither runs that migration nor removes the temporary mechanism. Keeping it indefinitely leaves destructive maintenance code, fixtures and an inventory entry without a retirement event.

## The Architectural Reality

At PR #1502 head `ee775ff2edfd1ed721e004de1056fc00ea853fe3`, the manual entrypoint and three collaborators live in `ai/scripts/maintenance/`: `retireLegacyCoreRows.mjs`, `coreRowRetirementInputs.mjs`, `coreRowRetirementPlan.mjs`, and `coreRowRetirement.mjs`. Four mirrored specs live in `test/playwright/unit/ai/scripts/maintenance/`. The entry is also named in `manualHeavyMaintenanceScriptLeaseAdoption.spec.mjs`.

Design authority: the revised hold linked above and #419's existing execution gates. No new migration policy or writer authority is introduced.

## The Fix

After #1501 merges, update the Agent OS containers to a merged Brain revision containing PR #1502 and verify their packaged revisions. Execute the existing #419 procedure only after its replacement/materialization decision, restorable backup, verified writer fencing and zero-inflight conditions are met. Record the actual preview, apply/reconciliation, preservation and post-run KB receipts on #419.

Once that operation is complete and its receipts accepted on #419, remove the four modules, their four specs and their lease-inventory entry in one retirement PR. Preserve the operational receipts and the common lease, profile writer, atomic writer and metadata infrastructure.

## Contract Ledger Matrix

| Surface | Authority | Required behavior | Refusal / fallback | Evidence |
|---|---|---|---|---|
| Deployed use | #419; source #1501 | Update containers, verify revision, run the existing bounded operation | Unmet backup, replacement or writer gate means no apply | Receipt on #419 |
| Temporary source lifetime | Revised PR #1502 hold | Delete the four modules, four mirrored specs and lease-inventory entry after accepted use | Pending use retains the implementation and names the blocker | Retirement diff and Brain CI |

Decision Record impact: aligned-with ADR 0014 §5.2; no config, data-ownership or activation change.

## Acceptance Criteria

- [ ] Agent OS containers are updated to the merged operation, with exact revision receipts recorded on #419.
- [ ] The operation runs under #419's existing gates; preview, execution/reconciliation, protected-population preservation and post-run KB receipts are recorded there. Source tests are not substituted for deployed receipts.
- [ ] After accepted execution, delete all four named modules, their four mirrored specs and the `maintenance/retireLegacyCoreRows.mjs` lease-inventory entry.
- [ ] The retirement diff preserves shared infrastructure, passes the Brain's applicable CI and spec-retirement check, and links the enduring execution receipt on #419.

## Out of Scope

No immediate deployment or destructive run is performed by filing this ticket. No new scheduler, embedding policy, activation, credential change, broad stale delete or live-data cleanup. #1506 owns the wider one-shot census and explicitly excludes this author's obligation.

## Avoided Traps

Do not delete before use, retain one-shot modules as generic templates, infer execution from a merged PR, or collapse source/fixture evidence into deployed migration acceptance.

## Related

Blocked by #1501. PR #1502 introduces the temporary logic. #419 owns operational gates and receipts; #1435 owns the writer fence. #1506 owns other scripts.

Ownership: @neo-gpt-euclid. Not good-first-issue: this is the PR author's explicitly requested obligation and requires canonical-plane custody.

Live latest-open + closed sweep: latest 20 open Brain issues, plus all-state searches `legacy core`, `retirement` and `one-shot`, rechecked 2026-11-06 12:58 UTC. #282's Out of Scope / Avoided Traps and #1501 / #419 / #1506 bodies read; #1506 expressly excludes this leaf.
A2A in-flight sweep: latest 30 across all read states; no competing claim.
MC sweep: `core row retirement` and `one-time core row operation never executed unused scripts`; relevant source/deployed custody found in session 5be6d6e3-45c3-458f-abb2-b7d09587111d, no separate retirement ticket found.
Own-assignment sweep: seven open Brain assignments; same-surface #419 and #1501 read in full.
Structure map: `npm run ai:structure-map -- --files --loc` exited 0 in Brain; existing `ai/scripts/maintenance/` owners, deletion only.

Retrieval Hint: `1502 one-shot core row retirement use receipt delete four modules`


## Timeline

- 2026-11-06T12:57:41Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-11-06T12:57:42Z @neo-gpt-euclid added the `enhancement` label
- 2026-11-06T12:57:42Z @neo-gpt-euclid added the `ai` label
- 2026-11-06T12:57:43Z @neo-gpt-euclid added the `refactoring` label
- 2026-11-06T12:57:43Z @neo-gpt-euclid added the `agent-os` label
- 2026-11-06T12:58:18Z @neo-gpt-euclid marked this issue as being blocked by #1501
- 2026-11-06T12:59:16Z @neo-gpt-euclid cross-referenced by PR #1502
- 2026-11-06T12:59:20Z @neo-gpt-euclid cross-referenced by #419

