---
id: 370
title: Curated tickets carry `good first issue` and `help wanted` together
state: CLOSED
labels:
  - enhancement
  - contributor-experience
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T04:01:14Z'
updatedAt: '2026-11-12T05:01:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/370'
author: neo-fable-mnemosyne
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
closedAt: '2026-11-12T05:01:27Z'
---
# Curated tickets carry `good first issue` and `help wanted` together

## Context

The ticket-create workflow labels most new non-critical tickets `good first issue` + `ai` (§4, from #199) and adds: "Honor first-time-only conventions with `help wanted` for returning contributors." The Engine's `CONTRIBUTING.md` sends a contributor to `help wanted` after the first merged pull request and reserves good first issues for newcomers.

Measured on `neomjs/neo` on 2026-11-12:

| Read | Value |
|---|---|
| `good first issue` created in the last 30 days | 66 |
| Open: `good first issue` / `help wanted` | 22 / 1 |
| `help wanted`, open and unassigned | 0 |
| Outside accounts with a merged pull request since Jul 1, and those with exactly one | 47, of which 39 |
| Curated issues closed in the last 30 days: by an outside pull request / by a maintainer seat | 27 / 17 |
| Open and unassigned `good first issue`, and their age | 14, median two days, none older than three |

## The Problem

- The second label depends on a judgment per ticket ("is this one for a returning contributor?") that no seat makes. The queue shows it: 22 to 1.
- A contributor who follows the guide after a first merge finds no issue. An agent applies the guide literally: its search is `is:open label:"help wanted" no:assignee`, the result is empty, and it moves to another repository.
- The reservation answered a scarce entry supply (neomjs/neo#19105: "returning contributors consume the entry supply"). The supply is no longer scarce: maintainer seats close 17 of 45 curated issues themselves, and 14 sit unassigned.

## The Architectural Reality

- `.agents/skills/ticket-create/references/ticket-create-workflow.md` §4: the default at line 133, the returning-contributor sentence at line 138.
- `neomjs/neo` `CONTRIBUTING.md` lines 117–121 hold the reservation. The Brain's and the Fleet Manager's guides carry the same sentence. neomjs/neo#20803 covers the guide and the issues open today.
- `neomjs/neo` `buildScripts/util/claimResponse.mjs` answers a claim on either label and prefers `good first issue` when an issue carries both. One open curated claim per account (neomjs/neo#20750) bounds what a single contributor can hold.
- What the labels mean. GitHub's default-label table: `good first issue` "Indicates a good issue for first-time contributors", `help wanted` "Indicates that a maintainer wants help on an issue or pull request" ([Managing labels](https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels)). #199 defines the first as first-contribution readiness, not difficulty: a contributor new to the repository can finish it from the ticket alone, whatever its size.

Design authority: "someone who lands their first PR is steered to the second label, not into another entry slot" (neomjs/neo#18985, restated in neomjs/neo#19105, September 2026). The same record's Out of Scope names today's gap: "pointing returning contributors at a stale label only moves the gap." The refresh it asked for did not happen. The operator proposed this change on 2026-11-12 as a peer's input; the measurement above is the reason, and the pull request's review decides.

## The Fix

In §4 of the workflow:

- Line 133 becomes: **Most new non-critical standalone tickets our agents create: `good first issue` + `help wanted` + `ai`.**
- Line 138, "Honor first-time-only conventions with `help wanted` for returning contributors.", becomes: "Both curated labels go on together: a first-time and a returning contributor draw on one list."
- Any other line under `.agents/skills/` that tells a seat to choose between the two labels follows (`grep -rn "help wanted" .agents/skills`).

The exclusions of §4 stay: work on a release's critical path, and unresolved, private-dependent or already-owned work, carries neither label. `help wanted` alone is no case of its own. A ticket's trail, the code's intent documentation and the installed skills give an outside agent the repository's context. What it cannot reach is private context, and that is already excluded.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| ticket-create §4, the label default (existing, line 133) | this ticket's measurement; the review | a curated ticket carries both labels | excluded classes carry neither | the section itself | `npm run lint`; the grep above |

Decision Record impact: none.

## Acceptance Criteria

- [ ] §4 names the three labels as the default and holds no sentence that reserves a curated label for a class of contributor.
- [ ] §4 states two cases: both labels, or neither.
- [ ] `grep -rn "help wanted" .agents/skills` shows no line that contradicts it.
- [ ] `npm run lint` passes, and the pull request states the payload's byte count before and after; it does not grow.
- [ ] The pull request names its version bump.

## Out of Scope

- The guide's sentence and the label on the issues that are open today: the Engine ticket in Related.
- A separate stream for returning contributors. Since Jul 1, a later pull request of a returning outside account drew a change request more often than a first one (7 of 22 against 7 of 47; a small sample, and one account holds half of the later ones), so tenure is no reason to route work.
- Recognition for contributors (badges, standing).

## Avoided Traps

- **Stocking `help wanted` by hand.** neomjs/neo#19105 asked for it in September. The label holds one issue.
- **Dropping the reservation and keeping one label.** A returning contributor's agent searches `help wanted`; the label has to be on the issue.
- **Assuming the slack lasts.** Returning contributors draw on the same list as newcomers. The triager's tripwire: fewer than five unassigned curated issues for a day, or fewer first-time authors in a month than in the month before, means label more work or split the labels again.

## Related

#199 (the label default) · neomjs/neo#19105 and neomjs/neo#18985 (the reservation) · neomjs/neo#20750 (one curated claim per account) · neomjs/neo#20803 (the guide and the issues open today)

handoff: @neo-opus-ada is asked. She built the one-claim cap and holds #364, which touches the same contributors. Not a curated ticket: its direction is a policy that maintainers review, so no outside author should spend a round on it.

Live latest-open + closed sweep: the latest 20 open by creation date (37 open, orange band; no open ticket or epic owns the label default) and a closed search (`"help wanted" OR "good first issue" label`) at 2026-11-12T03:58Z; #199 is the prior decision and is cited, its Out of Scope read. A2A in-flight claim sweep: the latest 30 messages, all read states, at 03:55Z; no claim on this scope. MC sweep: "outside contributors good first issue claim responder retention, do contributors come back", 8 results; the records of 2026-07-13, 2026-10-21 and 2026-11-06 are the prior art. Own-assignment sweep: 2 open here (#359, #191), not overlapping. Structure map: N/A, no file is placed. Meta-skill sweep: `create-skill` read; the change edits two lines of a reference payload and leaves the router untouched.

Origin Session ID: f73a4ac5-6f47-4cc7-9942-dc0bfd6c211d

Retrieval Hint: "curated labels together, good first issue plus help wanted, returning contributor finds no issue, first-timer reservation"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session f73a4ac5-6f47-4cc7-9942-dc0bfd6c211d


## Timeline

- 2026-11-12T04:01:15Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-12T04:01:15Z @neo-fable-mnemosyne added the `contributor-experience` label
- 2026-11-12T04:01:15Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-12T04:01:54Z @neo-fable-mnemosyne cross-referenced by #20803
- 2026-11-12T04:06:23Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T04:10:03Z @neo-opus-ada cross-referenced by PR #371
- 2026-11-12T04:11:24Z @neo-opus-ada cross-referenced by PR #20804
- 2026-11-12T04:19:35Z @neo-opus-ada referenced in commit `527c9a6` - "feat(ticket-create): section 4 names the three label cases (#370)

#370 gained an acceptance line at 04:13Z: both labels for a ticket finished from the ticket alone, help wanted alone for work that needs the repository's context, neither for the excluded classes. The exclusion line folds into that sentence, so the payload still shrinks (24,977 to 24,972 bytes)."
- 2026-11-12T04:27:18Z @neo-opus-ada referenced in commit `452bff2` - "feat(ticket-create): section 4 states two label cases, both or neither (#370)

The help-wanted-alone sentence goes (the operator's objection, and Mnemosyne's numbers on PR #371: a returning account's PR drew a change request in 7 of 22 cases, a first one in 7 of 47, so tenure is no reason to route work). Neither stays for the excluded classes."
- 2026-11-12T04:36:33Z @neo-opus-ada referenced in commit `96b73ea` - "feat(ticket-create): curated tickets carry both labels (#370)

Section 4's default becomes good first issue + help wanted + ai, and the sentence that reserved help wanted for returning contributors goes: newcomers and returning contributors draw on one list. The payload shrinks by 10 bytes. Minor bump to 1.6.0."
- 2026-11-12T04:36:33Z @neo-opus-ada referenced in commit `87ca47a` - "feat(ticket-create): section 4 names the three label cases (#370)

#370 gained an acceptance line at 04:13Z: both labels for a ticket finished from the ticket alone, help wanted alone for work that needs the repository's context, neither for the excluded classes. The exclusion line folds into that sentence, so the payload still shrinks (24,977 to 24,972 bytes)."
- 2026-11-12T04:36:34Z @neo-opus-ada referenced in commit `80e769f` - "feat(ticket-create): section 4 states two label cases, both or neither (#370)

The help-wanted-alone sentence goes (the operator's objection, and Mnemosyne's numbers on PR #371: a returning account's PR drew a change request in 7 of 22 cases, a first one in 7 of 47, so tenure is no reason to route work). Neither stays for the excluded classes."
- 2026-11-12T04:36:35Z @neo-opus-ada referenced in commit `1d1978f` - "chore(release): 1.7.0, since #369 took 1.6.0 (#370)"
- 2026-11-12T05:01:27Z @tobiu referenced in commit `e4c7797` - "feat(ticket-create): curated tickets carry both labels (#370) (#371)

* feat(ticket-create): curated tickets carry both labels (#370)

Section 4's default becomes good first issue + help wanted + ai, and the sentence that reserved help wanted for returning contributors goes: newcomers and returning contributors draw on one list. The payload shrinks by 10 bytes. Minor bump to 1.6.0.

* feat(ticket-create): section 4 names the three label cases (#370)

#370 gained an acceptance line at 04:13Z: both labels for a ticket finished from the ticket alone, help wanted alone for work that needs the repository's context, neither for the excluded classes. The exclusion line folds into that sentence, so the payload still shrinks (24,977 to 24,972 bytes).

* feat(ticket-create): section 4 states two label cases, both or neither (#370)

The help-wanted-alone sentence goes (the operator's objection, and Mnemosyne's numbers on PR #371: a returning account's PR drew a change request in 7 of 22 cases, a first one in 7 of 47, so tenure is no reason to route work). Neither stays for the excluded classes.

* chore(release): 1.7.0, since #369 took 1.6.0 (#370)"
- 2026-11-12T05:01:28Z @tobiu closed this issue

