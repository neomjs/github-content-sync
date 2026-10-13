---
id: 684
title: Fleet names the PR holder's reviewer-routing and rebase actions
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-13T18:41:50Z'
updatedAt: '2026-10-13T19:51:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/684'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 477
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-13T19:51:42Z'
---
# Fleet names the PR holder's reviewer-routing and rebase actions

## Context
Brain's action-specific holder contract is being delivered by neomjs/neo-agent-brain#761. The current Institution consumer must understand it before a package adopts that Brain pin.

## The Problem
At Institution `e5aa4238`, `OpenWorkSeat.kindOf()` returns `changes-requested` for every non-red author-held row. Applying it to a green `route-review` or `rebase` holder therefore tells the operator that changes were requested, although the next action differs. The source probe and earlier notification are recorded in [the producer intake](https://github.com/neomjs/neo-agent-brain/issues/761#issuecomment-6150177749).

## The Architectural Reality
Design authority: `OpenWorkSeat` documents: “The Brain's open-work producer decides who holds each PR (`holder`); this module only reads that verdict.”
The existing utility owns the shared roster chip and detail-row vocabulary. Brain owns the holder decision. Existing `#501` and `#449` deliveries remain the consumer surfaces; no second holder table belongs in Institution.

## The Fix
Consume the producer's explicit `holder.kind` for red-CI repair, requested changes, review, reviewer routing and rebasing. Give roster chips and detail rows the same action wording and ranking. Preserve compatibility with the currently shipped producer that omits `kind`; an explicit unsupported string kind remains a held row with neutral wording; it must not become a fabricated change request or disappear from the count. Do not change the Brain pin in this wording-only delivery.

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `OpenWorkSeat.kindOf/held/describe/describeRow` | Brain `holder.kind`; existing utility JSDoc | Author routing and rebase actions retain their meaning in both views | Absent kind keeps the shipped legacy contract; unknown explicit string kind remains held with neutral wording | Utility JSDoc | Existing utility controls plus composed card/detail projection cases |

## Acceptance Criteria
- [ ] AC-1: Explicit `repair-ci`, `address-changes`, `review`, `route-review` and `rebase` produce their corresponding words; routing/rebase never read “changes requested.”
- [ ] AC-2: Roster chip text/accessibility and detail-row wording share the mapping, including mixed-kind ranking and unchanged age/stale behavior.
- [ ] AC-3: Old rows without `kind` remain compatible. An unsupported explicit string kind remains held and counted with neutral wording at the lowest rank; it does not invent an action. Non-holders, operator/rotation/unknown holders and malformed non-string kinds remain excluded.

## Out of Scope
Holder selection, wake delivery, automatic enablement, Brain-pin adoption, installed acceptance and unrelated detail-pane work.

Decision Record impact: none; consumer adoption of the existing producer authority.

## Related
Parent: #477. Producer: neomjs/neo-agent-brain#761. Existing detail surface: #391.

Sweeps: latest 20 open Institution issues and all-state OpenWorkSeat search checked 2026-10-13 18:4x UTC; no equivalent. Latest 30 A2A messages contain no competing claim. MC search in the current session recovered the existing producer intake and consumer gap; KB found no relevant owner. Own-assignment #42 covers a controller/provider inventory, not this mapping. Brain-hosted structure map verified the existing `apps/agentos/util/OpenWorkSeat.mjs` placement; no new module.

Origin Session ID: 807642f8-fc33-41f7-acb0-be52fed54672
Retrieval Hint: "OpenWorkSeat green author route-review rebase wording"


## Timeline

- 2026-10-13T18:41:51Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-13T18:41:52Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-13T18:41:53Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-13T18:41:53Z @neo-gpt-emmy added the `ai` label
- 2026-10-13T18:42:13Z @neo-gpt-emmy added parent issue #477
- 2026-10-13T18:43:52Z @neo-gpt-emmy cross-referenced by #761
- 2026-10-13T18:48:32Z @neo-gpt-emmy cross-referenced by PR #1009
- 2026-10-13T18:49:03Z @neo-gpt-emmy cross-referenced by PR #685
- 2026-10-13T19:16:07Z @neo-gpt-emmy referenced in commit `369272c` - "test(fleet): bind verified holder wording captures (#684)

Co-Authored-By: Emmy <neo-gpt-emmy@neomjs.com>"
- 2026-10-13T19:38:36Z @neo-gpt-emmy referenced in commit `0a583cc` - "fix(fleet): keep unfamiliar holder actions counted (#684)

Co-Authored-By: Emmy <neo-gpt-emmy@neomjs.com>"
- 2026-10-13T19:51:42Z @tobiu closed this issue
- 2026-10-13T19:51:43Z @tobiu referenced in commit `20138c4` - "feat(fleet): render action-specific holder wording (#684) (#685)

* feat(fleet): render action-specific holder wording (#684)

Co-Authored-By: Emmy <neo-gpt-emmy@neomjs.com>

* test(fleet): bind verified holder wording captures (#684)

Co-Authored-By: Emmy <neo-gpt-emmy@neomjs.com>

* fix(fleet): keep unfamiliar holder actions counted (#684)

Co-Authored-By: Emmy <neo-gpt-emmy@neomjs.com>"

