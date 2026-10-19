---
id: 192
title: Retire superseded work when publishing a replacement decision
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-gpt-euclid
createdAt: '2026-10-19T12:26:45Z'
updatedAt: '2026-10-19T20:42:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/192'
author: neo-gpt-euclid
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
blocking: []
closedAt: '2026-10-19T20:42:31Z'
---
# Retire superseded work when publishing a replacement decision

## Context

The operator asked for a sweep when an accepted research direction is published or graduated, and when tickets/epics are created, so superseded implementation trails stop remaining executable work. The incident was verified against live predecessor bodies, an abandoned draft and the current decision; private project details stay outside this public ticket. The correction already applied there preserves historical evidence and leaves the quality outcome open.

## The Problem

Creation's adjacency sweep discovers related work, but its exit only prevents a duplicate new filing. A replacement decision can coexist with older open, assigned prescriptions and unchecked ACs. Intake can reject one when picked up; it does not make the publisher reconcile the predecessors. A closed draft alone leaves the older tickets actionable. A second operator-caught recurrence after ticket cleanup revived a historical research/tool recommendation and a saved next action instead; those entry points must be reconciled against current interaction requirements too.

## The Architectural Reality

At Skills dev `efa11b5230c1ef25c64598d9224808cb2da2b695`, `ticket-create/references/decision-substrate-sweeps.md` owns rationale/outcome discovery and same-turn porting. `ticket-create-workflow.md` §10 owns post-create work; `ideation-sandbox-workflow.md` §§3/6.7 own accepted folds and graduation; `epic-create-workflow.md` owns decomposition. Intake §0/§4 already rejects stale prescriptions and preserves unresolved outcomes. No corresponding producer cleanup exists in those payloads or the inspected corpus scripts.

## The Fix

Extend the existing shared sweep/porting payload with a bounded reconciliation of predecessors found by the same adjacency reads. Trigger it when an accepted fold changes direction, at graduation and after successor ticket/epic IDs exist. Record which related work stays current, narrows, transfers or retires; preserve unique obligations/evidence/dissent before removing old actionability. Update current heads and successor links, honoring authorship or scoped authorization, and verify readback. Unresolved outcomes and research stay open; a finally rejected narrow implementation closes as not planned. A new exploratory option alone retires nothing.

Keep the procedure in one conditional payload, with short pointers from its producer workflows and intake's existing rejection step. Leave intake's validation pass, routers, manifests and global turn instructions unchanged. Compress repeated explanation in that payload to offset the new operation.

## Contract Ledger

| Surface | Authority | Behavior | Fallback / ownership | Docs | Evidence |
|---|---|---|---|---|---|
| Accepted decision / graduation | Source decision and existing §11 authorship rules | Reconcile affected predecessor actionability before reporting the transition complete | New idea without accepted authority changes no predecessor; foreign edits need owner application or scoped authorization | Existing shared sweep payload | Cold read distinguishes accepted replacement from exploratory alternative |
| Ticket / epic publication | Existing adjacency results and new artifact ID | Move surviving obligations first, then link and disposition obsolete prescriptions | Unresolved parent outcome stays open; partial overlap narrows rather than closes | Existing creation payload pointers | Partial-overlap and rejected-method cases |
| Future session loading | ADR 0007 / skill Map-Atlas contract | Conditional detail; zero router/global load increase | Source merge is not installed adoption | PR load-effect audit | Corpus lint, byte delta, source-based cold reads |

## Acceptance Criteria

- [ ] One shared producer reconciliation runs at accepted direction-changing folds, graduation and successor ticket/epic creation, reusing adjacency results rather than adding a global scan.
- [ ] It preserves outcome, surviving obligations, evidence/provenance and authorship; speculative alternatives cannot retire accepted work, and completed-but-not-selected candidates cannot claim the outcome achieved.
- [ ] Related current heads/checklists and successor links receive an explicit disposition; owner-pending actions remain unresolved until verified. Narrow rejected prescriptions and unresolved outcome authorities have different close behavior.
- [ ] Existing corpus/reference lint passes; router and turn-loaded bytes do not grow. Net skill Markdown change is non-positive or its conditional retirement rationale is explicitly justified.
- [ ] Bounded cold reads cover a rejected method with an open outcome, a partially overlapping predecessor, a foreign-owned artifact, an unaccepted exploratory alternative, and a historical research/tool recommendation that conflicts with newer accepted interaction requirements even after old method tickets close.

## Out of Scope

New skills, registries, hooks, MCP fields, automatic branch deletion, historical evidence erasure, global intake expansion, general closure-policy changes, publishing/installing a package, or resolving the underlying visual-quality goal.

## Avoided Traps

Adding an intake warning while leaving producer artifacts contradictory; treating all old research as invalid; closing an outcome because its method was rejected; adding another required review seat; editing another author's ACs without authority.

## Existing-work disposition

`#62` covers reading closed decisions, `#59/#60` body currency/comment triggers, `#10` within-body fold sufficiency and `#13` source-to-epic fidelity. This is the bounded producer-side reconciliation missing between those existing contracts; no unrelated scope is added to them.

Decision Record impact: aligned-with ADR 0007 and the existing skill-shape contract; no new architectural primitive or ADR.
Source authority: direct operator request, recorded in the Origin Session below; no ungraduated Discussion is claimed as implementation authority.
Structure map: N/A — existing Markdown skill payloads, no Agent OS module or placement change.
Live latest-open sweep: latest20 read immediately before creation; no equivalent producer reconciliation ticket found. Same-surface closed query had no matching replacement.
MC rationale sweep: rejected approach/resumed draft research trails; own/peer current records found, no separate implementation owner.
Own-assignment sweep: `#140` is integration adoption, not this source change. Recent all-state claims contain no competing Skills write-surface claim.

Origin Session ID: ba608f22-df6f-4096-ac55-98ea38dc2cb1
Retrieval Hint: "superseded implementation trails accepted research decision publisher reconciliation"




## Timeline

- 2026-10-19T12:26:45Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-10-19T12:26:46Z @neo-gpt-euclid added the `enhancement` label
- 2026-10-19T12:26:46Z @neo-gpt-euclid added the `ai` label
- 2026-10-19T12:26:46Z @neo-gpt-euclid added the `model-experience` label
### @neo-gpt-euclid - 2026-10-19T12:30:55Z

### Source contract and load-effect evidence

The T3 Contract Ledger is in the canonical ticket body; its existing surfaces were checked at `efa11b5` before editing. One operational owner lives in `ticket-create/references/decision-substrate-sweeps.md#same-turn-reconciliation`, referenced by accepted ideation folds/graduation, ticket/epic publication and intake's existing rejection step. No additional intake pass is introduced.

Placement: `rewrite` conditional detail + `compress-to-trigger` caller pointers. Event-triggered, discipline-only guidance for accepted replacements; no router, manifest, global turn text, runtime schema or new skill. The final five-payload Markdown diff is **−307 bytes** against `origin/dev`, with zero router delta. `npm run lint -- --base origin/dev` passes corpus/reference and budget checks; historical explanations are compressed with their provenance retained.

A cold-context reader distinguished all four cases: rejected method/open outcome, partial overlap, foreign authority and unaccepted alternative. It identified successor-ID timing, pending-versus-complete reporting and the intake-rejection ingress. Those were corrected; a fresh delta read found no remaining ambiguity. This is source interpretation evidence, separate from installed-session adoption. Required release metadata is 0.1.52; no manual publish or consumer update.

- 2026-10-19T12:34:39Z @neo-gpt-euclid cross-referenced by PR #193
### @neo-gpt-euclid - 2026-10-19T13:22:27Z

### Additional measured supersession case: research entry points

A follow-up operator correction exposed a recurrence after old method tickets had closed: an earlier research/tool recommendation plus my saved next action still revived an obsolete path. The reference was reusable evidence, but no longer answered current interaction requirements.

The shared reconciliation explicitly reads research recommendations, tool assumptions and active handovers/checklists, comparing them against the current outcome/required interactions. Immutable memories/handovers retain history with exact superseded-ID → current-authority mappings. AC-5 includes this fifth case. Five-payload net Markdown change is **−124 bytes** against the current base; routers remain unchanged. Corpus/body/version checks pass locally. The base advanced to 0.1.52, so release metadata is now 0.1.53.

The actual recommendation heads and current research head were reconciled with original evidence preserved. A fresh-context recovery began from the obsolete research handover and two old recommendation URLs; it rejected the saved action and recovered current requirements without selecting a replacement implementation. This proves the record correction, not package loading or general recurrence prevention. The latest PR head is 286722d8d3177c470cbe3bb4d63e5a4be2d1725c; new-head CI follows the ordinary gate.

- 2026-10-19T13:24:46Z @neo-gpt-euclid referenced in commit `f9d2492` - "fix(skills): reconcile research against current interaction goals (#192)"
- 2026-10-19T13:24:46Z @neo-gpt-euclid referenced in commit `baebdb6` - "chore(skills): integrate current base for supersession fix (#192)"
- 2026-10-19T13:27:16Z @neo-gpt-euclid referenced in commit `286722d` - "fix(skills): preserve supersession links for immutable handovers (#192)"
- 2026-10-19T16:21:54Z @neo-gpt-euclid referenced in commit `e32a88f` - "fix(skills): restore executable reconciliation steps (#192)"
- 2026-10-19T16:43:14Z @neo-gpt-euclid referenced in commit `d6c707d` - "fix(skills): advance unpublished release candidate (#192)"
- 2026-10-19T16:51:29Z @neo-gpt-euclid referenced in commit `a957ef7` - "chore(skills): integrate current dev for reconciliation (#192)"
- 2026-10-19T20:42:31Z @tobiu referenced in commit `c6df0ef` - "feat(skills): reconcile superseded work at publication (#192) (#193)

* feat(skills): reconcile superseded work at publication (#192)

* fix(skills): reconcile research against current interaction goals (#192)

* fix(skills): preserve supersession links for immutable handovers (#192)

* fix(skills): restore executable reconciliation steps (#192)

* fix(skills): advance unpublished release candidate (#192)"
- 2026-10-19T20:42:31Z @tobiu closed this issue

