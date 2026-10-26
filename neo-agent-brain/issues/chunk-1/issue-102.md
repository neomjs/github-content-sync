---
id: 102
title: Bind community events to canonical A2A Tasks
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - security
assignees:
  - neo-gemini-maryam
createdAt: '2026-07-14T05:31:21Z'
updatedAt: '2026-10-26T22:41:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/102'
author: neo-gpt-euclid
commentsCount: 2
parentIssue: 106
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy:
  - '[x] 15152 Reconcile GitHub issue activity exhaustively'
  - '[x] 15151 Admit idempotent community batches into durable history'
blocking:
  - '[ ] 98 Prove the community-activity authority chain end to end'
  - '[x] 101 Project bounded tenant community-attention counts'
closedAt: '2026-10-26T22:41:20Z'
---
# Bind community events to canonical A2A Tasks

## Context

Grace's mandatory STEP_BACK partial and OQ5 require one unique atomic `sourceEventId -> taskId` binding before existing A2A Task authority begins. Existing Task ownership and transition event identity do not prevent two Tasks from being minted for one external response item.

This is one fully closeable PR leaf under Epic neomjs/neo-agent-brain#106. The live parent-child and blocked-by graph is authoritative; this body owns only this leaf's contract.

## The Problem

If Task creation and source binding are separate writes, concurrent peers can each create a canonical-looking Task and both win downstream assignment. A crash can also leave an orphan Task or binding.

## The Architectural Reality

The binding belongs in Memory Core and must compose with canonical A2A Task acceptance. `MailboxService.transitionTask` owns an existing Task's SQLite state/event transaction; it is not a Task-creation API. Current `MailboxService.addMessage` first accepts an immutable message WAL record, then projects it into the graph. The claim path must therefore establish its atomic Task/binding acceptance point explicitly, while keeping existing MailboxService assignment and transition rules downstream.

The Agent OS structure map was run on 2026-07-14. New service/script/test placement must use the named sibling-file-lift fast paths; no service logic moves into MCP server entrypoint directories.

## The Fix

Implement an atomic claim operation and unique binding table/index that validates tenant/source eligibility, creates or retrieves exactly one canonical A2A Task, commits Task plus `sourceEventId -> taskId` binding together, and returns the same Task to concurrent/retry callers.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Canonical claim | OQ5 STEP_BACK | One transaction binds eligible source event to one Task | Concurrent/retry callers receive existing canonical Task | Service/OpenAPI docs | Race and idempotency tests |
| Task authority handoff | MailboxService contracts | Existing Task owner/state authority begins only after binding commit | Failure exposes neither Task nor binding | JSDoc | Crash-point tests |
| Lifecycle exclusion | ADR 0035/OQ5 | Unclaimed community activity remains outside LifecycleFrontier | Seen or count state cannot promote it | Authority docs | Negative frontier tests |

## Decision Record impact

Depends on ADR 0036 and neutral admission; composes ADR 0035 and existing Task contracts without changing their state machine.

## Decision Record

**Required: ADR 0036.** This leaf is not code-ready until the ADR-0036 child of neomjs/neo-agent-brain#106 is accepted at the human merge gate.

## Discussion Criteria Mapping

| Upstream graduated criterion | This leaf's executable contract |
|---|---|
| OQ5 | Creates the missing admission-to-canonical-Task transition. |
| STEP_BACK point 4 | Proves unique atomic sourceEventId-to-taskId before Task authority. |
| OQ7/OQ8 | Claim is explicit, tenant-scoped, and independent of wake/read state. |

Source authority: Discussion #15139 body at the version-bound graduation anchor plus Grace's [STEP_BACK](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631120) and [GRADUATION_APPROVED](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631315).

## Acceptance Criteria

- [ ] **AC1** — Only an admitted, eligible, tenant/source-visible response item can be claimed.
- [ ] **AC2** — A unique constraint prevents more than one taskId for one scoped sourceEventId.
- [ ] **AC3** — Concurrent claim callers receive the same server-owned canonical Task.
- [ ] **AC4** — Task creation and binding commit atomically; every injected crash point leaves neither orphan authority nor duplicate Task.
- [ ] **AC5** — Retries after lost response are idempotent.
- [ ] **AC6** — Existing MailboxService assignment/RBAC/state transition rules take over only after commit.
- [ ] **AC7** — Seen state, count projection, and provider delivery ids cannot authorize claim.
- [ ] **AC8** — Unclaimed community items remain outside LifecycleFrontier and Golden Path ranking.
- [ ] **AC9** — Cross-tenant and revoked-source claim attempts fail closed.

## Out of Scope

Automatic assignment, automatic response, Task state-machine redesign, wake leases, or source acquisition.

## Avoided Traps

Do not use Task creation alone as dedup, bind after Task publication, overload occurrence identity with task id, or let seen imply claim.

## Related

- Parent: neomjs/neo-agent-brain#106
- Source: Discussion #15139 OQ5 and Grace STEP_BACK
- Precedents: neomjs/neo#15106/PR neomjs/neo#15111, neomjs/neo#15114/PR neomjs/neo#15121

Origin Session ID: 837ad74b-c2d2-413d-9aab-b7165a93a82a

## Handoff Retrieval Hints

- `sourceEventId taskId atomic canonical Task claim`
- `community activity claim race MailboxService transaction`


## Creation Freshness

Creation duplicate sweep: immediately before filing at 2026-07-14T05:31:21.422Z, checked the latest 20 open issues and last 30 all-state A2A messages. The independent broader audit at 2026-07-14T05:13:00Z covered open and closed issues, pull requests, A2A, ADRs, and code; no equivalent owner or foreign claim existed.

## Intake evidence — 2026-09-20

Prescription checked: `ai/services/memory-core/MailboxService.mjs` owns downstream Task rules, but wrapping `addMessage()` in a SQLite transaction would not make Task acceptance atomic with the source binding. At dev `0838bc3840431365309622c43d852d60c280418f`, `addMessage` awaits `appendWalMessage` before graph projection; `transitionTask` updates a Task that already exists. The existing `MailboxService.spec.mjs` arm `returns its durable WAL receipt before graph projection` reproduces a successful receipt with no graph MESSAGE yet. The same implementation marks ordinary message and routing projections `sharedEntity: true`, which cannot be adopted as tenant-private community-claim authority without an explicit visibility contract.

The uniqueness/race/crash acceptance criteria remain unchanged. Before code, resolve the one durable acceptance point for canonical Task creation plus binding, including replay/publication behavior and tenant-private visibility. A binding write followed by an ordinary `addMessage` call, or the reverse order, is not the accepted fix. This is a current-source correction within this leaf, not a reason to close the unresolved claim capability.



## Timeline

- 2026-07-14T05:31:23Z @neo-gpt-euclid added the `enhancement` label
- 2026-07-14T05:31:23Z @neo-gpt-euclid added the `ai` label
- 2026-07-14T05:31:23Z @neo-gpt-euclid added the `architecture` label
- 2026-07-14T05:31:23Z @neo-gpt-euclid added the `security` label
- 2026-07-14T05:32:55Z @neo-gpt-euclid marked this issue as being blocked by #15151
- 2026-07-14T05:32:57Z @neo-gpt-euclid marked this issue as being blocked by #15152
- 2026-07-14T05:34:32Z @neo-gpt-euclid cross-referenced by #106
- 2026-08-26T15:11:06Z @neo-gpt-euclid marked this issue as blocking #98
- 2026-08-26T15:13:15Z @tobiu added the `enhancement` label
- 2026-08-26T15:13:15Z @tobiu added the `ai` label
- 2026-08-26T15:13:15Z @tobiu added the `architecture` label
- 2026-08-26T15:13:16Z @tobiu added the `security` label
- 2026-08-26T15:13:18Z @neo-gpt-euclid marked this issue as being blocked by #15151
- 2026-08-26T15:13:18Z @neo-gpt-euclid marked this issue as being blocked by #15152
- 2026-08-26T15:13:19Z @neo-gpt-euclid marked this issue as blocking #101
- 2026-08-26T15:13:19Z @neo-gpt-euclid marked this issue as blocking #98
- 2026-08-26T15:14:31Z @tobiu added parent issue #106
- 2026-09-19T23:01:07Z @neo-gpt-euclid cross-referenced by PR #392
- 2026-10-26T20:04:05Z @neo-gemini-maryam assigned to @neo-gemini-maryam
### @neo-gemini-maryam - 2026-10-26T20:04:08Z

*Previously reviewed this epic: neomjs/neo-agent-brain#106#issuecomment-6426058715 [QUARANTINED_URL: github.com] (`IC_kwDOUBzDFM8AAAABfwXf2w`). Proceeding with sub pickup per `ticket-intake`.*

## Ticket Intake & Acceptance-Point Resolution (`#102`)

1. **Duplicate & Blocker Sweep:**
   - Upstream blockers `neomjs/neo#15151` (`PR #15513`) and `neomjs/neo#15152` (`PR #15524`) are both `CLOSED` and merged on `dev`.
   - Checked open PRs, issues, and recent A2A messages; no competing claim or duplicate implementation exists.

2. **Resolution of the `2026-09-20` Intake Evidence (One-Transaction Acceptance Point):**
   - As recorded in the `2026-09-20` intake note, wrapping `MailboxService.addMessage()` in a SQLite transaction cannot make canonical Task creation atomic with the `sourceEventId -> taskId` binding because `addMessage()` first writes to the JSONL WAL (`appendWalMessage`) before graph projection (`_projectMessageWalRecord`) and stamps ordinary messages `sharedEntity: true`.
   - Meanwhile, `GraphService.requireDb(...).storage.db`, `MailboxService.transitionTask()` (`ai/services/memory-core/MailboxService.mjs:6216-6450`), `CommunityActivityService.db`, `CommunityBatchAdmissionService.db`, and `SourceRegistryService.db` all share the single Memory-Core SQLite database (`config.storagePaths.graph` under `ADR 0015`).
   - **Concrete single-transaction design (`AC1`–`AC9`, `ADR 0036 §2.7 & §4 Gate 1–2`):**
     1. **Dedicated Binding Table (`mc_community_task_binding`):**
        - `PRIMARY KEY (tenant_id, occurrence_identity)` (`AC2`: at most one `taskId` per scoped `sourceEventId`) and `UNIQUE (task_id)`, storing `tenant_id`, `source_instance_id`, `occurrence_identity`, `task_id`, `claimed_by`, and `claimed_at`.
     2. **One `IMMEDIATE` SQLite Transaction on `GraphService.requireDb(...).storage.db`:**
        - Resolve `tenantId = SourceRegistryService.resolveTenantId()` and authenticated claimant identity (`RequestContextService.getAgentIdentityNodeId()`); fail closed if either is unbound (`AC9`).
        - Verify inside the transaction that `(tenantId, sourceEventId)` exists in `mc_community_observation`, has `attention_disposition = 'eligible'`, belongs to `SUPPORTED_GITHUB_COMMUNITY_KINDS`, has no valid tombstone (`absence = 'deleted'` with parseable `deletion_evidence`), and joins to an `ACTIVE` `mc_source_registration` (`provider = 'github'`, `canonical_provider_host = 'github.com'`, `resource_kind = 'repository'`) (`AC1`, `AC7`, `AC9`).
        - Query `mc_community_task_binding` for `(tenantId, sourceEventId)`:
          - If a binding row already exists, read the committed `MESSAGE` Task node from `Nodes` (`id = binding.task_id`) and return the existing canonical Task idempotently (`status: 'existing'`, `created: false` — `AC3`, `AC5`).
          - If no binding row exists, mint canonical `taskId = 'MESSAGE:' + crypto.randomUUID()`, insert the `mc_community_task_binding` row, upsert the claimant endpoint node if needed, insert the tenant-private `MESSAGE` Task node into `Nodes` (`sharedEntity: false, userId: tenantId`, `task: { id: taskId, state: 'Working', assignee: canonicalClaimant, sourceEventId, sourceInstanceId }`, `taskAssignmentAuthority: TASK_ASSIGNMENT_AUTHORITY`, `lastModifiedAt: claimedAtIso`), insert the `SENT_BY` and `SENT_TO` edges into `Edges` (`sharedEntity: false, userId: tenantId`), and append the initial `task_state_changed` `GraphLog` row (`appendGraphLogEvent`) **all inside the same SQLite transaction** (`AC4`).
        - After commit, sync `GraphService`'s in-memory node/edge cache (`db.getAdjacentNodes(taskId, 'both')` / `db.acknowledgeLocalMutations?.()`) so `MailboxService.transitionTask` (`AC6`) and `admitClaimedTask` (`ai/services/graph/lifecycleAdmission.mjs`, `AC8`) operate directly on the committed canonical Task, while any injected crash point inside the transaction rolls back both `mc_community_task_binding` and `Nodes`/`Edges`/`GraphLog` atomically (`AC4`).

Authored by Maryam (Gemini 4 Argon, Antigravity).
Origin Session ID: cd90984f-706d-48de-8f9d-05a76ea2749c 🧭

- 2026-10-26T21:16:48Z @neo-gemini-maryam cross-referenced by PR #1307
- 2026-10-26T21:46:28Z @neo-gemini-maryam cross-referenced by PR #1308
- 2026-10-26T22:32:08Z @neo-gemini-maryam cross-referenced by PR #1311
- 2026-10-26T22:38:32Z @tobiu referenced in commit `ba773d4` - "feat(memory-core): project bounded tenant community-attention counts (#101) (#1308)

* feat(memory-core): bind community events to canonical A2A Tasks (#102)

* feat(memory-core): project bounded tenant community-attention counts (#101)

* feat(memory-core): ship query-only attention counts and bound watermark (#101)"
### @neo-gemini-maryam - 2026-10-26T22:41:19Z

Resolved on `dev` in `ba773d4e9996f572bd7ccf1e6bbee323c27653de` (stacked PR #1307 / #1308).

- 2026-10-26T22:41:20Z @neo-gemini-maryam closed this issue
- 2026-10-26T23:34:23Z @neo-gemini-maryam cross-referenced by PR #1313

