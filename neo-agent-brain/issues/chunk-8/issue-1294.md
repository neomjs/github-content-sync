---
id: 1294
title: Record class-9 reservation custody in ADR 0041
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-26T05:02:13Z'
updatedAt: '2026-10-26T12:28:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1294'
author: neo-gpt-sophie
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 1296 Admit protected family history and migrate its readers'
closedAt: '2026-10-26T12:28:30Z'
---
# Record class-9 reservation custody in ADR 0041

## Context

[Institution D811 C7](https://github.com/neomjs/neo-agent-institution/discussions/811) reached the [GPT author signal](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188256) and [Claude approval/final Step-Back](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188284), with the two expressly acknowledged raw-writer census controls folded.

Its manual-confirmation delivery needs both the credential record in Institution ADR 0038 ([Institution #816](https://github.com/neomjs/neo-agent-institution/issues/816), [PR #817](https://github.com/neomjs/neo-agent-institution/pull/817)) and this narrow Brain ADR 0041 amendment. This is a documentation prerequisite; it implements no reservation, signer or recovery mechanism.

## The Problem

ADR 0041 correctly keeps pre-plane consent and host effects in one secret-free record per run. D811 additionally needs a post-plane class-9 reservation that lets a new setup/update run locate the original generation attempt. A new empty run cannot prove that another run never generated a key; an absent consumption pin cannot prove that a staged key never existed.

The relation must be recorded without turning MC into the owner of pre-plane bootstrap history or turning an ordinary PAT into provisioner authority.

## The Architectural Reality

Design authority: [ADR 0041 §2.5](https://github.com/neomjs/neo-agent-brain/blob/563262b5d560e0d5f1207ea260a1502c9708431a/learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md) says authenticated plane observations own current readiness while the host record owns prior consent/effects.

- `setupRunRecord.mjs` retains history within a run; `setupRecordPath` deterministically locates a run. A new first-run invocation reads its own UUID record. Those facts do not supply a cross-run index.
- `hostEffects.applyEffect` persists dispatch before a handler and reconciles an ambiguous pending effect. Its generic failed/changed-input paths do not by themselves provide class-9 create-once admission.
- C7's L3 plane reservation is admitted from trusted MC boot config after a capable plane exists. It retains `originRunId`; the host remains the writer and owner of that original run record.
- The same current/previous-head distinction must survive lost acknowledgements. “Plane ahead” can be a committed pending transition, not proof of rollback.

The owning document remains `learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md`. The file and cited host-effect sources are unchanged between `533724f0` and refreshed `origin/dev` `563262b5`.

## The Fix

Amend the existing record, preserving its accepted history and principles:

1. Add the post-plane reservation → original-host-run relation for class 9, with exact plane-id/data-root binding. Clarify that the rejection of a plane-owned **bootstrap record** does not prohibit this later, class-specific index.
2. Record separate `class9.reserve` / `class9.generate` dispatches; the plane reservation carries attempt, stable issuer/kid and original run ID before key generation. Fingerprint binding follows durable custody.
3. Require every later CLI/shell/update run to read the original record's current and retired attempt dispositions. Missing/unreadable history, changed root, failed/interrupted generation or a fresh current record cannot authorize a replacement mint.
4. Record expected-previous and exact intended-next heads for pending transitions. A matching target reconciles a lost acknowledgement normally; unexplained divergence refuses/fences. Receipts remain historical provenance, not readiness.
5. Add the class-specific witness: new run after staged-key loss, cross-renderer resume, lost reserve/bind reply, wrong target, ambiguous generation and supported restoration fence. Link ADR 0038/C7 for credential, clock and fence ownership instead of duplicating the whole publication protocol.
6. Record this amendment as a required decision record before the dependent manual-source implementation merges.

## Contract Ledger

| Surface | Authority | Behavior | Refusal / fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Host/plane ownership split | Existing ADR 0041; C7 L3 | Host owns per-run consent/effects; MC holds post-plane class-specific reservation/index | No plane replacement of pre-plane record; no PAT-derived provisioning privilege | Existing ADR 0041 | Exact amendment review preserves §§2.1–2.10 intent |
| Original-run relation and generation dispatch | C7, `setupRecordPath` / host-effect ownership | Reserved attempt names original run; separate reserve/generate; retained dispositions remain readable | Missing/ambiguous original history refuses mint | New bounded decision item | Source-backed witness table and cross-family review |
| Pending-phase reconciliation | ADR 0041 §2.6/§3; C7 | Matching pending target handles lost acknowledgement without replay | Unexplained divergence remains unavailable/recovery-required | Decision item + witness | Reserve/bind lost-reply cases explicitly distinguish normal reconciliation |
| Acceptance boundary | C7; ADR 0005 | Accepted decision record precedes implementation merge | Documentation alone proves no runtime or installed behavior | Merge-gate paragraph and sibling link | Human-merged reviewed amendment; later source/installed evidence stays separate |

## Decision Record impact

`amends` Brain ADR 0041's narrow post-plane relation and witness; preserves its host ownership, one-writer, target binding, historical receipt and observation rules. `aligned-with` ADR 0019; complements the Institution ADR 0038 class-9 amendment.

Decision Record: **Required — ADR 0041 amendment**, quorum-backed D811 C7. Its original author/history stay intact; acceptance follows the human merge of its reviewed PR.

## Acceptance Criteria

- [ ] AC-1: the amended record distinguishes the post-plane class-9 index from the still-host-owned pre-plane/per-run record, with both target comparisons and no new PAT privilege.
- [ ] AC-2: reservation/original-run binding, split dispatch, retained generation history, create-once refusal and fingerprint-after-custody order match C7.
- [ ] AC-3: expected-previous/intended-next reconciliation and the witness distinguish lost acknowledgement from rollback, and explicitly refuse new-run, missing-history, wrong-target and ambiguous-generation substitutions.
- [ ] AC-4: original decisions/history remain intact, the bounded class-9 exception/relation and ADR 0038/C7 links are coherent, and the implementation merge gate/source-versus-installed boundary are explicit and cross-family reviewed.

## Discussion Criteria Mapping

C7's original-run and provisioning contract → AC-1/2. Lost-acknowledgement and failure/restore controls → AC-3. Required accepted decision record → AC-4. Executable admission, readers, fence machinery and installed witness remain in the manual-source delivery; this leaf cannot close them.

## Signal Ledger

- `gpt`: [Sophie AUTHOR_SIGNAL](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188256), C7 body `2026-10-26T04:12:12Z`.
- `claude`: [Ada GRADUATION_APPROVED](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188284), explicitly extended to the two acknowledged census-AC tightenings.

## Unresolved Dissent

No unresolved formal DEFERRED/VETO at the checked C7 anchor. New substantive evidence follows the discussion's dissent/revalidation path.

## Unresolved Liveness

Clio's combined Monday read remains on her lane; no silence is approval. No Gemini signal was requested or received. The active GPT/Claude signers supply the applicable family quorum; no consensus rule/core value changes here.

## Out of Scope / Avoided Traps

No executable config, signer, API, restore/migration, runtime tests or installed acceptance. No second generic host ledger, graph-first bootstrap owner, new credential question, “never generated” inferred from an empty current run, or universal rollback-detection claim.

## Sweeps and placement

Own Brain assignments `#700` and `#1010` were body-read: the former owns the family outcome; the latter concerns ADR namespace/custody migration, not this semantic amendment. `#1005` is the unrelated summary-backfill defect. Live exact/history and created-descending open scans found no equivalent amendment. KB/MC returned budget-truncated or unrelated prior context, so absence is not inferred from them. Final live created-descending 20-open, all-state 30-message A2A and problem-noun MC sweeps at 2026-10-26 05:01:23 UTC found no competing amendment/claim. The MC rows were unrelated; no absence claim depends on them.

`npm run ai:structure-map -- --files --loc` passed; no new module/directory is prescribed. The existing ADR owns this relation. Decay rationale: amend one canonical record and link the credential contract; do not create a parallel bootstrap guide.

Related: #700
Decision companion: https://github.com/neomjs/neo-agent-institution/issues/816
Origin Session ID: 5032dade-507a-4d14-ac0f-15d78670a93c
Retrieval Hint: D811 C7 ADR0041 original bootstrap run class9 reservation lost acknowledgement

Sophie (GPT-6 Astra · ultra, Codex Desktop)

## Timeline

- 2026-10-26T05:02:13Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-26T05:02:15Z @neo-gpt-sophie added the `documentation` label
- 2026-10-26T05:02:15Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-26T05:02:15Z @neo-gpt-sophie added the `ai` label
- 2026-10-26T05:02:15Z @neo-gpt-sophie added the `architecture` label
### @neo-gpt-sophie - 2026-10-26T05:02:16Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-gpt-sophie`
**New assignees:** `neo-gpt-sophie`
**Reason:** This newly created issue was atomically assigned to neo-gpt-sophie; confirming the same self-claim for the tracked-file gate, with no other ownership replaced.

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

- 2026-10-26T05:02:16Z @neo-gpt-sophie added the `agent-os` label
### @neo-gpt-sophie - 2026-10-26T05:05:32Z

Prescription checked: `learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md` owns the host-record/plane-observation boundary. Its existing §§2.1–2.10 and rejected plane-owned-bootstrap-record alternative were read; the amendment adds only the post-plane class-9/original-run relation, preserving those invariants. D811 C7's non-author Step-Back/approval confirms this owner and split. This same-session documentation leaf needs no new module or competing guide.

- 2026-10-26T05:21:38Z @neo-gpt-sophie cross-referenced by PR #1295
- 2026-10-26T07:55:12Z @neo-gpt-sophie cross-referenced by #1296
- 2026-10-26T07:55:16Z @neo-gpt-sophie marked this issue as blocking #1296
- 2026-10-26T08:24:29Z @neo-gpt-sophie cross-referenced by #1297
- 2026-10-26T08:27:28Z @neo-gpt-sophie cross-referenced by #1298
- 2026-10-26T08:31:10Z @neo-gpt-sophie cross-referenced by #819
- 2026-10-26T12:28:30Z @tobiu referenced in commit `afba233` - "docs(agentos): bind publication reservations to host runs (#1294) (#1295)"
- 2026-10-26T12:28:30Z @tobiu closed this issue

