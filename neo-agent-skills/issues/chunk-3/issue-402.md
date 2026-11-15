---
id: 402
title: 'A merge order is declared as a native blocked_by edge, not a body line'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-15T17:58:21Z'
updatedAt: '2026-11-15T21:25:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/402'
author: neo-opus-grace
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
  - '[x] 1768 A pull request whose close target is still blocked waits off the merge list'
blocking: []
closedAt: '2026-11-15T21:25:00Z'
---
# A merge order is declared as a native blocked_by edge, not a body line

## Context

neomjs/neo-agent-brain#1769 (Resolves neomjs/neo-agent-brain#1768) makes the Fleet hold an approved, green pull request off the operator's merge list while an issue it closes still has an open blocker. The open-work snapshot reads GitHub's `issueDependenciesSummary.blockedBy` for each close target, and `holderOf` names `author/await-prerequisite` until that issue's blockers close.

The hold fires only when the dependency exists as a native edge. Today authors state a merge order in the PR body, as "Merge order: X first" or "Draft until #N merges". On 2026-11-15 a pull request merged ahead of its engine prerequisite, although its body said so in bold. The pull-request workflow gives authors no other place to declare it.

Design authority: "The dependency is declared natively: the close-target ticket is `blocked_by` the prerequisite's ticket." — neomjs/neo-agent-brain#1768, The Fix, item 3 (merged in neomjs/neo-agent-brain#1769).

## The Problem

A rule with no slot does not fire. The producer can now read a declared dependency, but nothing tells an author, at the moment they write a dependent PR, to declare it where the producer reads.

## The Architectural Reality

- `.agents/skills/pull-request/references/pull-request-workflow.md` §9.1 holds the close-target rules authors follow when they write `Resolves` / `Related`. This is the moment a merge order is known.
- `update_issue_relationship` (github-workflow MCP) sets a native `blocked_by` edge within one repository. It takes one `repo` and resolves the blocking issue's number there, so it cannot write a cross-repository edge. GitHub's REST route `POST /repos/{owner}/{repo}/issues/{issue_number}/dependencies/blocked_by` (body: the blocker's numeric `issue_id`) writes one across repositories.
- Cross-repository edges read through `blockedBy`: neomjs/neo-agent-skills#401 returns its 8 blockers from `neo`, `neo-agent-brain`, `neo-agent-institution` and `neo-agent-skills`. A private issue blocked by a public one counts too: private neo-pro #321 `blocked_by` neomjs/neo#21028 read 1 open blocker a few seconds after the edge was written (0 on the very first read).
- `issueDependenciesSummary.blockedBy` counts only open blockers. neomjs/neo-agent-brain#1013 lists 3 blockers, all closed, and reads 0.

## The Fix

Add one rule to §9.1. A merge order is a native `blocked_by` edge from the PR's close target to the prerequisite's ticket. Within one repository, set it with `update_issue_relationship`; across repositories, with GitHub's dependency route. The Fleet holds the merge until that ticket closes, and a body line alone is not read. The draft-as-hold workaround can then go.

Load: about one line added to a reference payload read on trigger, not to a router. Retirement trigger: when the PR-body lint flags merge-order prose that has no matching edge, compress the rule to that lint's pointer.

## Contract Ledger

| Target surface | Source of authority | Behaviour | Fallback / edge | Evidence |
|---|---|---|---|---|
| Edge direction | Design authority above | The PR's close target is `blocked_by` the prerequisite's ticket, never the reverse and never the PR itself | A merge order with no ticket on either side gets no edge; it stays out of scope | §9.1 text |
| Same-repository writer | `update_issue_relationship` (one `repo`, numeric `parent_issue`/`child_issue`) | Writes the edge when both tickets share a repository | Qualified cross-repository input is refused, so the author uses the REST route | §9.1 text |
| Cross-repository writer | GitHub REST issue dependencies | `POST /repos/{owner}/{repo}/issues/{n}/dependencies/blocked_by` with the blocker's `issue_id` | Verify both repository identities after writing (`blockedBy` read) | §9.1 link; the private/public receipt above |
| Reader | neomjs/neo-agent-brain#1769 (`issueDependenciesSummary.blockedBy` via `holderOf`) | An approved, green head with a blocked close target is `author/await-prerequisite`, off the merge list | The count updates seconds after a write; an unread count certifies no merge | Brain's own unit arms; AC-4 here |

## Acceptance Criteria

- [ ] §9.1 states the rule in at most two lines: the edge (close target → prerequisite's ticket), its writers (same- and cross-repository), and that body prose is not read.
- [ ] No router (`SKILL.md`) grows; the old merge-order prose habit appears nowhere else as guidance.
- [ ] `npm run lint` and `node scripts/lint-skill-corpus.mjs --base origin/dev` pass, and the package version is bumped in the PR.
- [ ] [Post-merge, owner Grace] [L3-deferred — the installed Fleet Manager must run neomjs/neo-agent-brain#1769's producer] the next dependent PR in the swarm carries the edge, and the Fleet's awaiting-merge list omits it until the prerequisite closes. Link that receipt on neomjs/neo-agent-brain#1768. Residual-Owner: neomjs/neo-agent-brain#759.

## Out of Scope

- The Fleet hold itself (neomjs/neo-agent-brain#1769).
- A lint that enforces the edge, and ordering between PRs with no declared dependency.
- A cross-repository mode for `update_issue_relationship` (an MCP change, not this rule).

## Avoided Traps

- **A stronger body sentence:** the incident's body already carried one in bold.
- **A new section or router entry:** the rule belongs where the close target is written.
- **Naming one writer for both cases:** `update_issue_relationship` cannot write across repositories, and the incident was a cross-repository merge order.

## Related

neomjs/neo-agent-brain#1768, neomjs/neo-agent-brain#1769. neomjs/neo-agent-skills#399 edits the same file (`pull-request-workflow.md`, one of its six owners); whichever lands second rebases.

Decision Record impact: none.

Live latest-open + closed sweep: latest 20 open, plus open and closed search ("merge order blocked_by prerequisite"), read at 2026-11-15T17:55Z; 38 open (orange); no equivalent or prior exclusion. A2A claims of the last hour: none on this scope. Memory Core sweep (from neomjs/neo-agent-brain#1768's filing): 5 merge-order relays as prose, none a mechanism. Own assignments checked.

Origin Session ID: cf286bdf-2e66-492f-8b70-3661af17a29e

Retrieval Hint: "merge order declared as native blocked_by edge on the close target; Fleet await-prerequisite hold"


## Timeline

- 2026-11-15T17:58:23Z @neo-opus-grace added the `documentation` label
- 2026-11-15T17:58:23Z @neo-opus-grace added the `enhancement` label
- 2026-11-15T17:58:23Z @neo-opus-grace added the `ai` label
- 2026-11-15T17:58:30Z @github-actions added the `needs-design-authority` label
- 2026-11-15T17:58:40Z @neo-opus-grace cross-referenced by #991
- 2026-11-15T17:58:58Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-15T17:59:06Z @neo-opus-grace marked this issue as being blocked by #1768
- 2026-11-15T17:59:33Z @neo-opus-grace cross-referenced by PR #1769
- 2026-11-15T19:58:34Z @neo-opus-grace cross-referenced by PR #404
- 2026-11-15T20:52:36Z @github-actions removed the `needs-design-authority` label
- 2026-11-15T20:52:59Z @neo-opus-grace referenced in commit `6449ba7` - "fix(pull-request): name both writers of a merge-order edge (#402)

update_issue_relationship writes within one repository; a cross-repository edge goes through GitHub's dependency route.

[skill-growth-justified: the rule must name the cross-repository writer, since update_issue_relationship cannot write across repositories and the motivating merge order crossed them]"
- 2026-11-15T21:25:00Z @tobiu referenced in commit `4d30c8c` - "feat(pull-request): a merge order is declared as a native blocked_by edge (#402) (#404)

* feat(pull-request): a merge order is declared as a native blocked_by edge (#402)

* fix(pull-request): name both writers of a merge-order edge (#402)

update_issue_relationship writes within one repository; a cross-repository edge goes through GitHub's dependency route.

[skill-growth-justified: the rule must name the cross-repository writer, since update_issue_relationship cannot write across repositories and the motivating merge order crossed them]"
- 2026-11-15T21:25:01Z @tobiu closed this issue

