---
id: 1631
title: Merge readiness shows no earlier open PR on the same issue
state: OPEN
labels:
  - enhancement
  - good first issue
  - help wanted
  - contributor-experience
  - ai
assignees: []
createdAt: '2026-11-12T04:12:01Z'
updatedAt: '2026-11-12T05:32:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1631'
author: neo-fable-mnemosyne
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy: []
blocking: []
---
# Merge readiness shows no earlier open PR on the same issue

## Context

On 2026-11-11 two outside pull requests resolved the same curated issue, neomjs/neo#20782. neomjs/neo#20784 opened at 16:54Z. neomjs/neo#20787 opened at 17:17Z and was reviewed, approved and merged at 19:20Z while the first was still open. The first was closed unmerged the next morning. Nothing a reviewer reads named the earlier pull request.

## The Problem

- Merge readiness reads one pull request. `GET_MERGE_READINESS` (`ai/services/github-workflow/queries/pullRequestQueries.mjs:81`) asks for its state, review decision, body, review requests and reviews, and nothing about another pull request that closes the same issue.
- The pr-review guide's Close-Target Audit (§5.2 in `neomjs/neo-agent-skills`) checks what a pull request closes, never who else closes it.
- The read exists elsewhere. The claim responder asks the issue for `closedByPullRequestsReferences(first: 20, includeClosedPrs: false)` (`neomjs/neo` `.github/workflows/claim-responder.yml:69–77`) and `holders()` in `buildScripts/util/claimResponse.mjs` uses it.

## The Architectural Reality

- `ai/services/github-workflow/PullRequestService.mjs:833` and `:862` run the query; the payload they build is what `get_conversation` returns as a pull request's merge readiness.
- The pull request's body is already read, and its close target is parsed from it.
- `neomjs/neo` `CONTRIBUTING.md`, paragraph 3: "we merge the first approve-worthy PR for a ticket". An earlier open pull request therefore blocks nothing. The reviewer should know it is there and say so to both authors.

Design authority: n/a — calls no existing behaviour wrong; it adds a read.

## The Fix

The readiness payload gains the other open pull requests that close the same issue or issues: number, author, creation time and draft state, in creation order. One more GraphQL read per close target, with the field the claim responder already uses.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| merge readiness in `get_conversation` (existing, `PullRequestService.mjs:833`) | this ticket | `competingPullRequests`: for each close target, the open pull requests other than this one as `{number, author, createdAt, isDraft}` | no close target, or none open: an empty list. A failed read says unknown and never returns an empty list | the tool's handbook entry | a unit spec over a stubbed query |

Decision Record impact: none.

## Acceptance Criteria

- [ ] For a pull request whose close target has another open pull request, readiness lists it with number, author and creation time (unit, stubbed query).
- [ ] With none open the list is empty. When the read fails, the payload says so instead of returning an empty list (unit).
- [ ] The tool's handbook entry names the field.
- [ ] Post-merge: a readiness read of a live pull request that shares a close target shows its sibling. The receipt goes on this ticket.

## Out of Scope

- Deciding which of two pull requests merges. The guide already says: the first approve-worthy one.
- The review guide's wording. Once the field exists, §5.2 of the pr-review guide can name it.
- Waking a seat for the earlier pull request: #1629.

## Avoided Traps

- **A guide line instead of a read.** A reviewer under load runs no extra search. The field arrives with the readiness they already read.

## Related

#1629 · neomjs/neo#20782, neomjs/neo#20784, neomjs/neo#20787 · neomjs/neo#20750 (the claim responder's cap, which reads the same field)

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: the latest 20 open by creation date (156 open, red band; no open ticket or epic owns merge readiness) and a search over all states (`"close target"`, `"competing pull request"`, `"same issue" pull request readiness`) at 2026-11-12T04:08Z; no equivalent found. A2A in-flight claim sweep: the latest 30 messages, all read states; Julia's defect-note of Nov 12 names this gap, no claim. MC sweep: "outside contributor pull request waits unseen: runs await approval, no rotation wake, baseline slot, bot review counted as touched", 5 results, no prior decision. Own-assignment sweep: 5 open here (#1562, #1258, #144, #122, #95), not overlapping. Structure map: run at 04:08Z; the owning folder is `ai/services/github-workflow`, with `queries/pullRequestQueries.mjs` and `PullRequestService.mjs`; no file is placed.

Origin Session ID: f73a4ac5-6f47-4cc7-9942-dc0bfd6c211d

Retrieval Hint: "merge readiness competing pull request same close target, earlier open PR on the same issue, Close-Target Audit sibling"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session f73a4ac5-6f47-4cc7-9942-dc0bfd6c211d


## Timeline

- 2026-11-12T04:12:02Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-12T04:12:02Z @neo-fable-mnemosyne added the `good first issue` label
- 2026-11-12T04:12:02Z @neo-fable-mnemosyne added the `help wanted` label
- 2026-11-12T04:12:02Z @neo-fable-mnemosyne added the `contributor-experience` label
- 2026-11-12T04:12:03Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-12T04:12:38Z @neo-fable-mnemosyne cross-referenced by #759
- 2026-11-12T04:13:11Z @neo-fable-mnemosyne cross-referenced by #1629
### @jay23606 - 2026-11-12T05:32:44Z

I'd like to take this one. My plan is to add the sibling lookup in the merge readiness path in `PullRequestService.mjs` (around lines 833 and 862), with a companion GraphQL read next to `GET_MERGE_READINESS` in `pullRequestQueries.mjs`. I'd cover it with a unit spec over a stubbed query, including the failed-read case. Could you assign it to me?

### @github-actions - 2026-11-12T05:32:55Z

Thanks for claiming this, @jay23606, and welcome! You're first in line, so you can start now.

A maintainer confirms the assignment when one is online, and that can take a while. You don't need to wait for it.

<sub>An automatic reply. A maintainer follows up. How claims work [QUARANTINED_URL: github.com]</sub>
<!-- claim-responder:@jay23606 -->


