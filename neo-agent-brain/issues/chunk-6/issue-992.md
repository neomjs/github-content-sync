---
id: 992
title: Projection submission admits async denials before its transaction
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-gpt-euclid
createdAt: '2026-10-11T03:21:44Z'
updatedAt: '2026-10-11T03:57:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/992'
author: neo-gpt-euclid
commentsCount: 0
parentIssue: 107
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-11T03:47:52Z'
---
# Projection submission admits async denials before its transaction

## Context

The source/admission audit for #107 found a concrete primitive failure before production registration. At Brain `dddc1bcc`, an isolated real SQLite probe of `submitProjectionChannel` accepts `async () => false` for either admission callback and inserts a channel row. Both ordinary callbacks observe `db.inTransaction === false`.

Design authority: the function's own JSDoc promises `targetId => Boolean` / `({targetId, channel}) => Boolean`; ADR 0035 §2.6.1 requires transaction-level target admission and producer identity validation.

## The Problem

The primitive tests callback truthiness, so a Promise or other truthy non-boolean becomes a grant. It also resolves both grants before `submit.immediate()`, leaving the authorization read outside the write transaction. The claimed single-transaction contract is not met.

This is a library-level reproduction, not a deployed exploit claim. No production writer caller is registered yet.

## The Architectural Reality

`ai/services/memory-core/hookProjectionSubmission.mjs#submitProjectionChannel` owns submission policy and SQL mutation. Its current owning spec uses a real in-memory better-sqlite3 store. The writer delegates to this function; the future source-owned admission callbacks remain a separate boundary.

## The Fix

Keep the synchronous API and existing outcomes. Require literal `true` from each callback, and resolve both callbacks inside the existing immediate write transaction before reading or mutating the channel. Keep schema and watermark behavior intact.

| Surface | Authority | Behavior | Failure | Evidence |
| --- | --- | --- | --- | --- |
| `isTargetAdmitted` | Source-owned synchronous callback | Only `true` grants, checked inside submission transaction | Existing `target-not-admitted`; no channel mutation | Promise/non-boolean and transaction-state arms |
| `mayProduceChannel` | Source-owned synchronous callback | Only `true` grants, checked in the same transaction | Existing `foreign-producer`; no channel mutation | Promise/non-boolean and grant-revocation arms |
| Channel schema/watermark | Existing pinned contracts | Preserve replay, conflict, regression and newer-watermark semantics | Existing outcomes | Owning and adjacent writer/lease tests |

## Decision Record impact

Aligned with ADR 0035 §2.6.1; no new permission, registration or instance-attestation contract.

## Acceptance Criteria

- [x] Async-false and truthy non-boolean results from either callback are refused without inserting, advancing or conflicting a channel.
- [x] Both callback reads run inside the immediate transaction; a grant revoked before transaction entry prevents mutation.
- [x] Existing true/false, schema, watermark, replay and conflict behavior stays green in the owning and adjacent writer/lease specs.
- [x] Changes stay within the submission helper and its existing canonical spec; #107 retains tenant mapping, registration/publication admission, launcher proof and installed acceptance.

## Out of Scope

A production service caller, target registry, tenant mapping, publication API/gate, config, deployment, lease/schema changes, remote transport and the #958 language matcher.

## Avoided Traps

Do not await network authorization inside a SQLite transaction. The callback contract remains synchronous. A literal-true check does not itself establish the callback's authority; source-owned predicates must query the relevant current authority using the same store transaction. This fix does not certify the dormant writer as integrated.

## Related

Parent: #107
Related: ADR 0035

Origin Session ID: 2d8feac7-c60d-4883-8059-37b6e148768b
Retrieval Hint: `hookProjectionSubmission async false truthy admission before transaction`

Creation sweep: final live tracker, recent all-state mailbox, own-assignment and rationale checks immediately before filing. Semantic recall is currently unavailable due `EMBEDDING_PROBE_TIMEOUT`; earlier framing searches were off-topic. Source and the isolated real-store reproduction establish the defect. This independently actionable primitive defect cannot be folded into the closed documentation leaf #989; it stays under the existing #107 outcome.
Independent prescription check: Grace's source-fork reply (2026-10-11T03:17:19Z) confirms the transaction and strict synchronous true contract; exception rollback, transaction observation and malformed/thenable controls are carried into the owning spec.


## Timeline

- 2026-10-11T03:21:44Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-10-11T03:21:45Z @neo-gpt-euclid added the `bug` label
- 2026-10-11T03:21:45Z @neo-gpt-euclid added the `ai` label
- 2026-10-11T03:21:45Z @neo-gpt-euclid added the `testing` label
- 2026-10-11T03:21:49Z @neo-gpt-euclid added parent issue #107
- 2026-10-11T03:28:30Z @neo-gpt-euclid cross-referenced by PR #993
- 2026-10-11T03:29:44Z @neo-gpt-euclid cross-referenced by #107
- 2026-10-11T03:32:21Z @neo-gpt-euclid referenced in commit `222466f` - "test(memory): guard admission fixture cleanup (#992)

Co-Authored-By: Euclid <neo-gpt@neomjs.com>"
- 2026-10-11T03:47:52Z @tobiu closed this issue
- 2026-10-11T03:47:52Z @tobiu referenced in commit `7ca6602` - "fix(memory): enforce transactional projection admission (#992) (#993)

* fix(memory): enforce transactional projection admission (#992)

Co-Authored-By: Euclid <neo-gpt@neomjs.com>

* test(memory): guard admission fixture cleanup (#992)

Co-Authored-By: Euclid <neo-gpt@neomjs.com>"
- 2026-10-11T07:23:16Z @neo-opus-grace cross-referenced by PR #998

