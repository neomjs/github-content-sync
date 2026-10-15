---
id: 696
title: Open an observed merge-queue record in a Stage dev window
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-gpt
createdAt: '2026-10-15T10:39:34Z'
updatedAt: '2026-10-15T13:52:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/696'
author: neo-gpt
commentsCount: 0
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
closedAt: '2026-10-15T13:52:13Z'
---
# Open an observed merge-queue record in a Stage dev window

## Context
This is the Institution consumer half of neomjs/neo-stage#59, within neomjs/neo-stage#32. It records the repository-local delivery and close target for the already-authorized sibling-first lane, not another product outcome. The Stage issue owns its record receiver, development composition and overall launch journey.

Design authority: [the seed's adopted entry](https://github.com/neomjs/neo-stage/issues/59) says "capability-gated Open in Stage secondary action in the awaiting-merge row menu"; [the independent design read](https://github.com/neomjs/neo-stage/issues/59#issuecomment-6189050082) binds keyboard access, one held source snapshot, lifetime fencing and truthful coverage.

## The Problem
At Institution afabdef60face9a02a0841c43ed18f8f33fdac60, `AwaitingMergeMenuList.createItemContent` renders the forge anchor and stale marker, with no sibling action. `OpenWorkRead.mergeRows` strips raw producer fields and the model supplies defaults, so passing its projected record as original source would overclaim the data held.

## The Architectural Reality
`Controller.openWorkSnapshot` and `openWorkProfileId` hold the one answering source. The root Provider owns shared outcome state; roster/button/menu only render and relay. `Neo.Main.windowOpen` admits a physical window with a Boolean; it does not establish module load or paint. Stage uses a separate App worker and an ephemeral same-origin JSON channel. `OperatorInbox` establishes the owned helper lifecycle in `apps/agentos/util`; native opening/closure remains Engine Main's responsibility.

Decision Record impact: aligned-with the accepted sibling-first Stage decision; no new runtime-extension loader, topology authority or AiConfig surface.

## The Fix
Add the optional development entry and its lifecycle owner within existing FM composition. Resolve the clicked internal list record to its repository-qualified Store key, then resolve raw row, envelope observation and profile together. Retire a pending launch before closing its unique native window on timeout, source/profile replacement or destruction. Keep received/admitted/drawn stages distinct. Retain addressed failure feedback across ordinary menu dismissal, without adding default closed-header pixels.

## Contract Ledger
| Target surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Optional `stageLaunchUrl` development config / menu action | Stage59 and design read above | Same-origin explicit sibling; existing forge anchor remains | Omitted/invalid capability adds no action; denied/missing/refused app reports addressed unavailability | README | Mounted keyboard and unavailable controls |
| `StageWindow` owned lifecycle | Held `openWorkSnapshot` / profile; existing Main native APIs | JSON of raw row plus envelope provenance, one token-bound child and completed-draw receipt | Changed source/profile, closed child and late completion retire without reopening | JSDoc | Real Provider unit controls and separate-worker journey |
| Provider `stageLaunch` result | Launch owner only | Row-addressed state/message rendered by existing trigger/menu | Never infer paint from open or coverage | JSDoc / README | Focus/dismissal and source-replacement controls |

## Acceptance Criteria
- [ ] AC-1: The explicit dev capability adds a secondary Open in Stage action while preserving the forge link and default header density; keyboard Enter/Space activate once, Escape returns focus, and the narrow menu remains readable.
- [ ] AC-2: Activation resolves the raw source row, observation metadata and profile from one held answer, retaining unknown fields and coverage without manufacturing original history or model defaults.
- [ ] AC-3: Denied, absent and refused Stage report addressed unavailability while FM remains usable. Close, profile/source replacement, destruction and late results cannot recreate the window or replace newer context.
- [ ] AC-4: A mounted dev-composition journey binds separate App/Canvas identities, observes the admitted context's completed draw, returns/closes through the native owner, and leaves host topology and another window unchanged.

## Out of Scope
Installed extension loading/packaging, backend changes, a second history reader, original review-request production, credential/profile mutation, avatar source changes and the full recorded handoff outcome. That original-event source remains neomjs/neo-stage#32's named next step.

## Avoided Traps
A same origin is not a shared App heap. An open Boolean is not paint. A list's internal Neo ID is not the producer key. Producer coverage is not original review history. The URL carries only an ephemeral token; no retained localStorage record is added.

## Recovery and duplicate sweep
Live latest 20 open Institution issues and historical Stage matches checked 2026-10-15T10:39Z; no matching entry task. Current A2A slice contains this lane's existing quiet claim and no competing entry claim. MC problem-noun query returned three unrelated earlier cockpit records; the explicit current design authority is linked above. Own-assignment sweep: three open; the row-2 body's installed-state scope does not deliver this optional sibling entry. Structure map: frontend owned-helper pattern beside `OperatorInbox`, no new directory or Agent OS runtime service.

Related: neomjs/neo-stage#59 · neomjs/neo-stage#32.
Origin Session ID: ed9c23e6-01d5-4dc2-8031-cb7c5a631f62
Retrieval Hint: "Stage59 optional awaiting-merge secondary action one held raw observation sibling draw lifetime"


## Timeline

- 2026-10-15T10:39:34Z @neo-gpt assigned to @neo-gpt
- 2026-10-15T10:39:35Z @neo-gpt added the `enhancement` label
- 2026-10-15T10:39:35Z @neo-gpt added the `agent-os` label
- 2026-10-15T10:39:35Z @neo-gpt added the `ai` label
- 2026-10-15T10:53:53Z @neo-gpt cross-referenced by #19751
- 2026-10-15T11:03:34Z @neo-gpt cross-referenced by PR #19752
- 2026-10-15T11:19:48Z @neo-gpt cross-referenced by PR #697
- 2026-10-15T11:53:45Z @neo-gpt referenced in commit `5449925` - "fix(fleet): report the Stage close request truthfully (#696)"
- 2026-10-15T11:53:46Z @neo-gpt referenced in commit `4062605` - "fix(fleet): retain addressed Stage feedback after queue changes (#696)"
- 2026-10-15T12:37:28Z @neo-gpt referenced in commit `093ada1` - "chore(fleet): integrate the merged harness fixture (#696)"
- 2026-10-15T13:52:13Z @tobiu referenced in commit `0f01c91` - "feat(fleet): open observed work in a Stage sibling (#696) (#697)

* feat(fleet): open observed work in a Stage dev sibling (#696)

* fix(fleet): retain Stage action focus while opening (#696)

* fix(fleet): report the Stage close request truthfully (#696)

* fix(fleet): retain addressed Stage feedback after queue changes (#696)"
- 2026-10-15T13:52:13Z @tobiu closed this issue

