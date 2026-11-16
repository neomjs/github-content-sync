---
id: 991
title: The seat card names a PR that waits on its close target's blocker
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-15T17:58:39Z'
updatedAt: '2026-11-15T23:02:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/991'
author: neo-opus-grace
commentsCount: 0
parentIssue: 477
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1768 A pull request whose close target is still blocked waits off the merge list'
blocking: []
closedAt: '2026-11-15T23:02:43Z'
---
# The seat card names a PR that waits on its close target's blocker

## Context

neomjs/neo-agent-brain#1769 (Resolves neomjs/neo-agent-brain#1768) adds a holder kind to the open-work producer. An approved, green PR whose close target still has an open blocker is held as `{role: 'author', kind: 'await-prerequisite', ids: [seat], targets: ['owner/name#N']}` instead of `operator/merge`. The awaiting-merge button reads the operator rows, so it already omits such a PR. The author's seat card reads the same holder and has no words for the new kind.

## The Problem

`apps/agentos/util/OpenWorkSeat.mjs` maps producer kinds through `DISPLAY_KIND` (lines 20–27 at `dev`). `kindOf` turns any kind missing from it into `'unknown'` (line 91). So once neomjs/neo-agent-brain#1769 merges, a held PR shows on its author's card as "next action unknown", which is a false state: the PR's next step is known, and it waits on a named ticket.

## The Architectural Reality

- `KIND_WORDS` is ranked worst first ("The order IS the rank"). A PR that waits on a prerequisite asks nothing of the seat now, so it ranks last.
- The holder's `targets` names the blocked close targets. The card shows words, not data, today.

## The Fix

Add `'await-prerequisite'` to `DISPLAY_KIND`, mapped to a display kind with words such as "waits on a blocked ticket". Place it last in `KIND_WORDS`, after `route-review` and before `unknown`.

## Acceptance Criteria

- [ ] A row held `author/await-prerequisite` counts on its author's card and reads in words that name the wait, never "next action unknown" (unit, `OpenWorkSeat` spec).
- [ ] It ranks below every action the seat can take, so a red or changes-requested PR still names the card's worst.
- [ ] The awaiting-merge button is unchanged: the row is not the operator's.
- [ ] [Post-merge] the installed Fleet Manager shows the held PR on its author's card in those words.

## Out of Scope

- Naming the blocked issue on the card (`targets`): words first; data later if a reader needs it.
- The producer's hold (neomjs/neo-agent-brain#1769).

## Related

neomjs/neo-agent-brain#1768 (natively blocking this ticket), neomjs/neo-agent-brain#1769, neomjs/neo-agent-skills#402 (the slot that makes authors declare the edge). Parent outcome: neomjs/neo-agent-institution#477 (every cockpit surface names its state with a reason and a next step).

Decision Record impact: none.

Live latest-open + closed sweep: latest 20 open, plus open and closed search ("OpenWorkSeat kind words seat card held"), read at 2026-11-15T17:55Z; 58 open (red); no equivalent. neomjs/neo-agent-institution#869 owns the list's overflow, not the card's words, and #477's open leaves (#479, #887, #898) do not cover it. A2A claims of the last hour: none on this scope. Own assignments checked.

Origin Session ID: cf286bdf-2e66-492f-8b70-3661af17a29e

Retrieval Hint: "seat card await-prerequisite kind words OpenWorkSeat DISPLAY_KIND"

## Timeline

- 2026-11-15T17:58:41Z @neo-opus-grace added the `enhancement` label
- 2026-11-15T17:58:41Z @neo-opus-grace added the `ai` label
- 2026-11-15T17:59:00Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-15T17:59:04Z @neo-opus-grace added parent issue #477
- 2026-11-15T17:59:05Z @neo-opus-grace marked this issue as being blocked by #1768
- 2026-11-15T17:59:33Z @neo-opus-grace cross-referenced by PR #1769
- 2026-11-15T22:27:26Z @neo-opus-grace cross-referenced by PR #992
- 2026-11-15T22:46:32Z @neo-opus-grace referenced in commit `3d39d47` - "fix(agentos): the seat card counts pull requests on the seat, not ones waiting on it (#991)

A pull request that waits on a blocked ticket asks nothing of its author, so the chip's
accessible name and title no longer say every held pull request waits on the seat:
"N pull requests on this seat", with the worst kind still named first."
- 2026-11-15T23:02:42Z @tobiu referenced in commit `0c5fcac` - "feat(agentos): the seat card names a PR that waits on a blocked ticket (#991) (#992)

* feat(agentos): the seat card names a PR that waits on a blocked ticket (#991)

The open-work producer holds an approved, green head whose close target still has an
open blocker as author/await-prerequisite. The card mapped that unknown kind to "next
action unknown"; it now reads "waits on a blocked ticket" and ranks below every action
the seat can take, above an unknown one.

* fix(agentos): the seat card counts pull requests on the seat, not ones waiting on it (#991)

A pull request that waits on a blocked ticket asks nothing of its author, so the chip's
accessible name and title no longer say every held pull request waits on the seat:
"N pull requests on this seat", with the worst kind still named first."
- 2026-11-15T23:02:44Z @tobiu closed this issue

