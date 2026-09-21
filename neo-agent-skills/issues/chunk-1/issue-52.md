---
id: 52
title: Two workflows still route lane readiness through a retired label
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-06T12:34:30Z'
updatedAt: '2026-09-08T07:44:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/52'
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
blockedBy: []
blocking: []
closedAt: '2026-09-08T07:44:44Z'
---
# Two workflows still route lane readiness through a retired label

`Serves:` **none** — a live defect in shipped substrate, not a lane under an open epic. Stating that rather than claiming a parent this does not have; the honest `none` is the point of asking.

## Context

`not-code-ready` was **retired on purpose**. Operator, 2026-09-06:

> *"we fully on purpose removed the `not-code-ready` label. tickets can get blocked. and an unblocked ticket gets 'fresh' once a blocker is resolved. the label led to: adding the label, never removing it, team then ignored important tickets 'forever'."*

Two shipped workflows still route readiness through it. Verified absent from every repo it would be applied in:

| repo | `not-code-ready` | `needs-design` | `deferred-by-design` | `needs-re-triage` |
|---|---|---|---|---|
| `neomjs/neo` | absent | absent | absent | **present** |
| `neomjs/neo-agent-brain` | absent | — | — | present |
| `neomjs/neo-agent-skills` | absent | — | — | present |

Three of the four labels the substrate names do not exist.

## The Problem

Five live instructions on `origin/dev`, in two lane-critical workflows:

| site | instruction | effect today |
|---|---|---|
| `post-review-pickup-workflow.md:100` | survey excludes `-label:not-code-ready` | inert filter — matches nothing |
| `post-review-pickup-workflow.md:102` | *"Mark it `not-code-ready`"* | applies a label that does not exist |
| `ticket-intake-workflow.md:22` | Readiness Pre-Check keyed on the label | gates on a state nothing can be in |
| `ticket-intake-workflow.md:148` | *"Apply `not-code-ready` + `needs-re-triage`"* | one dead label, one mislabelled audience |
| `ticket-intake-workflow.md:152` | *"`manage_issue_labels (action: add)` to apply `not-code-ready`"* | an explicit tool call with an invalid label |

Two of them tell an agent to call `manage_issue_labels` with a name the taxonomy does not carry — which `ticket-create` §4 already forbids (*"Do not invent label names"*).

**`needs-re-triage` is a separate defect, not a survivor.** It is a live label but belongs to a **different audience**: public repos where external contributors open tickets that do not meet the quality or intent bar — the `ticket-triage` skill owns it. Instructing intake to stamp it on *our own* architecture tickets both mislabels the ticket and pollutes the queue a maintainer scans for contributor submissions.

## The Architectural Reality

The retirement rationale generalizes past this label, and is the reason a like-for-like replacement would be wrong:

> **A state that must be manually cleared will not be cleared.**

`not-code-ready` was add-only in practice. The replacement already exists and needs no clearing: `update_issue_relationship(relationship_type: 'blocked_by')`. **What self-clears is the readiness, not the edge** — the blocker closing is what makes the ticket claimable again, with nobody removing anything, which is the operator's *"an unblocked ticket gets fresh once a blocker is resolved"*. `ticket-create` §6 already names this tool for exactly this purpose, so nothing new is introduced.

*Correction, 2026-09-08 (@neo-gpt, PR #53 RA-2):* an earlier revision of this section said the **edge** resolves. It does not. Live REST dependencies API: open `neomjs/neo#17834` still carries a `blocked_by` edge to **closed** `#17920`; positive control `#18379` → open `#18376`. The relationship persists and is provenance — it records what the ticket waited on, which is the one thing a label could never do. Only the blocking stops.

**Readiness is a relationship, not a label.**

## The Fix

Delete the retired-label machinery; route readiness through the blocked-by edge that already exists.

- `post-review-pickup` §6 — drop the inert filter; replace *"mark it not-code-ready"* with recording the blocker.
- `ticket-intake` Readiness Pre-Check — key on a `blocked_by` edge to an open issue.
- `ticket-intake` Close Policy + Autonomous Protocol — record the blocker instead of applying labels, and state why `needs-re-triage` is not the tool here.

**Second change, same §6 list, disclosed as scope rather than smuggled:** the lane survey's only epic reference excludes epics, so the skill that decides *what to work on next* never surfaces the artifacts encoding project direction. One added line makes them **readable but not claimable**. Empirical anchor: on 2026-09-06 I filed four tickets and two PRs into `neo-agent-brain`, whose open epics are a deletion-and-consolidation programme; two of them forbade the work in prose, I never saw them, and all six artifacts were closed unmerged after burning two cross-family peers' review cycles. Discussion of the general shape is on #6; this ticket carries only the one-line survey change.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `not-code-ready` | operator retirement 2026-09-06 | all references removed | none — label is gone | both workflows | absent in 3/3 repos |
| `needs-re-triage` | `ticket-triage` skill | reserved for external-contributor triage | unchanged | `ticket-intake` §Close Policy | operator audience ruling |
| `blocked_by` edge | `update_issue_relationship` | the readiness mechanism | comment stating the blocker | both workflows | tool already contracted in `ticket-create` §6 |
| lane survey epic filter | this ticket | epics readable, not claimable | unchanged | `post-review-pickup` §6 | 0 milestone/roadmap mentions across all lane skills |

## Decision Record impact

`none`.

## Acceptance Criteria

- [ ] No occurrence of `not-code-ready`, `needs-design` or `deferred-by-design` remains in `.agents/skills/**`. Greppable, and it must go red if any returns.
- [ ] No workflow instructs applying a label absent from the target repo's taxonomy. A reviewer can check each named label against `gh label list`.
- [ ] `ticket-intake` states that `needs-re-triage` is for externally-authored tickets below the quality/intent bar, and is not the tool for internal readiness.
- [ ] Readiness is read from the linked blockers' **live states**, and the text says why: claimability recomputes when a blocker closes, where a label must be manually cleared. The edge itself is retained as provenance and is not deleted on resolution.
- [ ] The lane survey reads open epics and milestones as direction, without making them claimable.
- [ ] `npm run lint` and `npm test` hold. Note the suite is 24/25 on unmodified `dev` in a checkout without full `node_modules` — dangling `.claude/skills` links — so that arm is pre-existing and environmental; confirmed by stash-and-rerun.

## Out of Scope

- The general "creation is the ungated stage" discussion — #6 owns it.
- Retiring or renaming any label that currently exists.
- Any change to `ticket-triage`, which correctly owns `needs-re-triage`.

## Avoided Traps

- **A like-for-like replacement label.** Reintroduces the exact add-only failure the operator retired; the retirement rationale is the design input, not an obstacle.
- **Leaving the inert filter because it is harmless.** It is not: it teaches the next reader that the label exists, which is how instruction 102 came to tell agents to apply it.
- **Treating `needs-re-triage` as the survivor.** It survives as a label and dies as *this* instruction — different audience.

## Related

- #6 (creation-stage plan authority — the general case; my session notes are in its thread)
- #8 (friction→gold retrospectives)
- neomjs/neo-agent-brain#136 (mechanical enforcement replaces prompt-machinery)

Origin Session ID: 70502f9a-5b14-4dcf-bcdf-4a29b546df77

Retrieval Hint: `query_raw_memories` — "not-code-ready retired label lane readiness blocked_by relationship self-clearing"; grep `not-code-ready` in `.agents/skills`.



## Timeline

- 2026-09-06T12:34:30Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-06T12:34:31Z @neo-opus-grace added the `bug` label
- 2026-09-06T12:34:32Z @neo-opus-grace added the `ai` label
- 2026-09-06T12:34:32Z @neo-opus-grace added the `model-experience` label
- 2026-09-06T12:34:32Z @neo-opus-grace added the `agent-os` label
- 2026-09-06T12:35:14Z @neo-opus-grace cross-referenced by PR #53
- 2026-09-06T13:35:06Z @neo-opus-grace referenced in commit `42e4ac9` - "chore(skills): the replacement says the same thing in fewer bytes (#52)

The corpus guard caught +676 bytes against a 250 budget and offered two exits:
trim, or justify in the commit message. Trimming was the honest one — the growth
was not justified, it was flab. I had replaced five short wrong instructions with
longer right ones and never asked whether the right ones needed the words.

Nothing was cut but prose. The blocked-by mechanism, the needs-re-triage audience
boundary, and the epics-as-direction-frame line all survive intact; several read
better short. `pr_diff_equals_pr_body` already demands this pass before posting
and I skipped it, which is why a guard had to ask.

Not raising maxPositiveDeltaBytes: the ceiling protects every future PR, and
spending it on my own wordiness would be the worst possible reason to move it."
- 2026-09-07T00:13:06Z @neo-opus-grace cross-referenced by PR #55
- 2026-09-07T00:14:47Z @neo-opus-grace cross-referenced by #56
- 2026-09-08T05:53:41Z @neo-opus-grace referenced in commit `35a7cbf` - "fix(skills): readiness is the blockers' states, not the edge's absence (#52)

Addresses @neo-gpt's two required actions on PR #53.

RA-2: "the edge clears itself" was false, and I verified the falsifier
independently rather than adopting it. Live REST dependencies API:
neomjs/neo#17834 is OPEN and still carries a blocked_by edge to CLOSED
#17920; positive control #18379 -> OPEN #18376. The relationship persists;
only the blocker's state changes. Worse than the wrong sentence was the
instruction next to it -- "correct the edge if the blocker already closed"
told an agent to delete exactly the #17834 edge, which is provenance.

What self-clears is readiness, not the edge. That is still the operator's
retirement rationale ("an unblocked ticket gets fresh once a blocker is
resolved") and it is strictly better than a label: the edge survives to say
what the ticket waited on, while claimability recomputes from live state.

RA-1: the human-guided rejection path still routed internal tickets to
needs-re-triage, contradicting the Close Policy this same PR introduced
eight lines above it. It now defers to that policy. Second defect in the
same line, not in the review: it named `status: needs-re-triage`, which
404s -- the label is `needs-re-triage`. Both are gone with the clause.

Net corpus delta +180 bytes against the 250 budget (was +117). Lint green."
- 2026-09-08T07:44:44Z @tobiu referenced in commit `81f3c92` - "Merge pull request #53 from neomjs/grace/retire-not-code-ready-refs

fix(skills): lane readiness is a blocked-by edge, not a retired label (#52)"
- 2026-09-08T07:44:44Z @tobiu closed this issue
- 2026-09-08T15:21:42Z @neo-opus-grace cross-referenced by #18484
- 2026-09-08T15:25:19Z @neo-opus-grace cross-referenced by PR #18485

