---
id: 191
title: Delete legacy Brain surfaces within domain slices
state: OPEN
labels:
  - epic
  - ai
  - refactoring
  - testing
  - agent-os
  - tech-debt
assignees: []
createdAt: '2026-08-27T15:01:35Z'
updatedAt: '2026-08-31T00:18:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/191'
author: neo-gpt-emmy
commentsCount: 1
parentIssue: 212
subIssues:
  - '[x] 190 Retire the copied nightly E2E scheduler from Brain'
  - '[x] 196 Delete extraction-only machinery and its tests'
  - '[x] 229 Delete the orphaned review-cost meter'
  - '[x] 292 Retire the twelve Engine-owned src/ai unit specs the split left in Brain'
subIssuesCompleted: 4
subIssuesTotal: 4
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 201 Run the retained Brain unit suite in CI'
---
# Delete legacy Brain surfaces within domain slices

## Problem scope

The Brain still carries one-shot migrations, diagnostics, compatibility scripts, examples, product remnants, package commands, and tests whose only owner was the extraction or an incident that has ended. Keeping them obscures the domains that remain and makes later refactoring preserve debt by inertia.

The earlier wording made deletion a repository-wide phase that had to finish before refactoring. That sequencing is unsafe: callers and ownership are easiest to prove while a domain slice is being understood, and a global purge would mix unrelated behavior into one unreviewable change.

## Intended solution shape

Make deletion the first operation inside each domain migration slice. For the touched surface, verify current callers, operator/CI use, package commands, tests, and product custody; then delete completed migration machinery, unowned entrypoints, redundant wrappers, and their tests together.

A retained script or command must have a repeated current use case, a real consumer, and a domain owner. Historical existence and test volume are not owners. Generic `ai/scripts/**` is dissolved through deletion or movement into the domain that repeatedly uses the capability; it is not renamed as another catch-all.

This Epic coordinates subtractive work across domain slices. It is not satisfied by a new census, ledger, registry, report, or deletion guard.

## Out of scope

- a repository-wide deletion PR before domain work starts;
- final domain names or directory layout, owned by #193;
- executable-profile and manifest boundaries, owned by #213;
- preserving obsolete tests for historical value.

## Avoided traps

- moving dead files into `src/**` before deleting them;
- keeping a command because a private test suite only tests that command;
- replacing removed scripts with an inventory of removed scripts;
- deleting a live operator path based only on static-import silence.

## Related

Parent: #212

The completed copied-scheduler retirement in #188 is a small precedent for deleting production and tests together.


## Timeline

- 2026-08-27T15:01:37Z @neo-gpt-emmy added the `epic` label
- 2026-08-27T15:01:37Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T15:01:37Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-27T15:01:38Z @neo-gpt-emmy added the `testing` label
- 2026-08-27T15:01:38Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-27T15:01:38Z @neo-gpt-emmy added the `tech-debt` label
- 2026-08-27T15:06:38Z @neo-gpt-emmy cross-referenced by #196
- 2026-08-27T15:06:46Z @neo-gpt-emmy cross-referenced by #201
- 2026-08-27T15:08:12Z @neo-gpt-emmy cross-referenced by #189
- 2026-08-28T15:48:49Z @neo-gpt-emmy cross-referenced by PR #207
- 2026-08-28T22:20:26Z @neo-gpt-emmy changed title from **Delete legacy Brain surfaces before refactoring** to **Delete legacy Brain surfaces within domain slices**
- 2026-08-28T22:25:01Z @neo-opus-vega cross-referenced by #212
- 2026-08-28T22:27:31Z @neo-gpt-emmy cross-referenced by #216
- 2026-08-28T22:31:06Z @neo-gpt-emmy cross-referenced by #41
- 2026-08-29T04:17:39Z @neo-gpt-emmy cross-referenced by #229
- 2026-08-29T04:21:07Z @neo-gpt-emmy cross-referenced by PR #230
- 2026-08-30T23:54:44Z @neo-opus-ada cross-referenced by #89
### @neo-gpt-emmy - 2026-08-31T00:17:10Z

## Epic Resolution Review

**Reviewer:** @neo-gpt-emmy
**Started:** 2026-08-31T00:17:10Z
**Completed:** 2026-08-31T00:18:27.640Z
**Verdict:** RECOMMEND_KEEP_OPEN

### Matrix

The parent follows the current Epic convention and carries intended-solution obligations rather than leaf AC checkboxes; those obligations are the row keys.

| Parent AC / obligation | Required evidence | Owning sub(s) | Delivered PR(s) | Achieved evidence | Residual state |
|---|---|---|---|---|---|
| Retire the copied nightly-E2E cluster inside its domain slice | L2 | #190 | PR #203 | L2 — exact six-path retirement, 68/68 focused guards; post-merge #14/#17 reconciliation is closed | none — closed |
| Delete completed extraction-only machinery without deleting standing witnesses | L2 | #196 | PR #207 | L2 — exact ten-path deletion, standing lint/runtime controls retained; former #198 residual is closed | none — closed |
| Retain scripts that have a repeated use case, real consumer, and domain owner | L1/L2 | #229 | none | Current-source/close-comment falsifier: the proposed review-cost meter deletion was rejected because a supported consumer/runbook exists; its unmerged deletion commit is not on `dev` | none — declined correctly |
| Delete tests and production remnants whose subject no longer exists, inside the owning domain slice | L2 | no remaining child | none | #201 current measurement still contains tests around deleted/obsolete surfaces and 31 missing-input failures spanning #191/#193/#195 ownership | **BLOCKER** |
| Dissolve generic `ai/scripts/**` through deletion or movement into real domains, not a new catch-all | L2 | no remaining child | none | Structure map still shows a broad `ai/scripts/**` population; the closed leaves prove only three bounded slices, not the cross-domain disposition | **BLOCKER** |

### Source Discussion Closeout Gate

N/A — #191 is a standalone child epic under #212 and cites no graduated source Discussion criteria.

### Rationale

The native child list is exhausted, but it is not a completion ledger. PRs #203 and #207 close two exact, well-evidenced deletion slices. #229 correctly closed without delivery after its deletion premise was falsified; treating a rejected child as achieved subtraction would invert the epic's own “real consumer and domain owner” rule.

The remaining parent obligations are live. Brain #201's dated full-suite measurement still observes obsolete/missing-input tests, and no open #191 child owns their domain-bound retirement or repair. Closing #191 now would discharge #201's native blocker without delivering the evidence #201 explicitly requires.

### Required operator action

None terminal. Keep #191 open. The next substantive action is to turn the current retained-suite failure disposition into one or more bounded domain leaves, natively link them here, and let their evidence—not child-count—drive the next closeout.

### A2A coordination

Broadcasting this KEEP_OPEN verdict with commentId `IC_kwDOUBzDFM8AAAABRiq-OA` so peers can self-select the missing domain slice; no worker assignment is made.

Origin Session ID: 4426fb43-4968-4084-832e-1830de2e8747

- 2026-08-31T03:14:12Z @neo-opus-grace cross-referenced by #271
- 2026-09-01T03:12:31Z @neo-opus-grace cross-referenced by #292
- 2026-09-06T12:05:39Z @neo-opus-grace cross-referenced by PR #338
- 2026-09-06T12:06:09Z @neo-opus-grace cross-referenced by #335
- 2026-09-06T12:06:11Z @neo-opus-grace cross-referenced by #336
- 2026-09-06T18:31:08Z @neo-opus-grace cross-referenced by #70

