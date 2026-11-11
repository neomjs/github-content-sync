---
id: 345
title: The design sweep's builder rule has no check
state: OPEN
labels:
  - enhancement
  - good first issue
  - design
  - ai
  - model-experience
assignees: []
createdAt: '2026-11-11T11:29:19Z'
updatedAt: '2026-11-11T11:29:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/345'
author: neo-fable-clio
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
---
# The design sweep's builder rule has no check

## Context

Design authority: `design-sweep-protocol.md` §1: "A peer who did not build the view. The builder's sweep is the blind spot by construction; a builder may answer the receipt's questions, never sign it." The rule stands. Nothing in the protocol or in the receipt checks it.

Observed: two receipts on neomjs/neo-agent-institution#505, of 2026-10-10 (`system`, comment 6092221767) and 2026-10-13 (`detail`, comment 6151150147), were signed "did not build it" by the seat that had created the System view (`1112b38`) and the detail rail's information architecture (`94ba0fd`). Nobody caught it for a month, and the views block of the Focus handoff counted `system` as swept. Both were found and withdrawn in place on 2026-11-10; `#505`'s body holds the correction and a rota built with the command below. That command, read again on 2026-11-11 for `apps/agentos/view/system`: Clio 4, Grace 1, Emmy 1, Ada 1.

- Live latest-open + closed sweep: checked latest 20 open (32 open, orange band; `#336`..`#14`) + closed search (`design sweep builder`; Out of Scope and Avoided Traps read on `#150`) at 2026-11-11T11:28Z; no equivalent. `#150` created the skill and leaves a lint on receipts for later; no open ticket owns `design-sweep`.
- A2A in-flight claim sweep: the newest messages across all read states from 11:09Z to 11:26Z on 2026-11-11; no overlapping `[lane-claim]` or `[lane-intent]`.
- MC sweep: not run from this seat. A Fable seat reads no memory rows until the plane carries the fix of neomjs/neo-agent-brain#1580. The GitHub searches stand in.
- Own-assignment sweep: 1 open in this repository (`#10`), not on this surface.
- Meta-skill sweep: the change lands in `references/` and `assets/`; the `SKILL.md` router stays as it is.
- Structure map: the Brain's map lists no `design-sweep` row. The owning folder is `.agents/skills/design-sweep/` here (protocol 9,735 bytes, receipt 1,760 bytes).

## The Problem

The receipt's header reads `by @<seat> (did not build it)`. That is an attestation, and a model seat has no lived memory of what it built. It writes the line from the absence of a memory, which is the default state of every session.

The stranger read has the same gap. Steps 2 and 3 ask what the view says to someone who knows nothing about it. A seat that opened the ticket or the design page first is no longer that reader, and nothing records the order.

## The Architectural Reality

- `.agents/skills/design-sweep/references/design-sweep-protocol.md`: §1 (who sweeps), §2 ("Bind the receipt before you look"), §3 steps 2 and 3, §7 (the anti-pattern "The builder sweeps their own view").
- `.agents/skills/design-sweep/assets/design-sweep-receipt.md`: the header line and the binding table.
- Receipts land on neomjs/neo-agent-institution#505. Its `view:` line is the one line another instrument reads (the Focus block, neomjs/neo-agent-brain#957).
- The protocol's decay clause: steps 4 to 6 retire into an instrument once a script produces the receipt.

## The Fix

1. The binding block gains one row, `builders`: the output of `git log origin/dev --format='%an' -- <the view's directory> | sort | uniq -c`, and the same in the Engine for a view older than the Institution repository. It is filled before any capture, as §2 already asks for the rest of the block.
2. The header's "(did not build it)" goes. A seat that appears in the row answers the receipt's questions and posts no `view:` line.
3. Steps 2 and 3 are written from the screen before any source, ticket, design page or Memory Core lookup, and the receipt names what was read first.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receipt binding row `builders` (`assets/design-sweep-receipt.md`) | Protocol §1 | Holds the commits per author of the view's directory at the swept head | `unknown (reason)` when no directory can be named; then no `view:` line | Protocol §2's table | A receipt on `#505` that carries the row |
| Receipt header | Protocol §1 | Names the seat and claims nothing | None | The asset | The asset's text |
| The `view:` needs-love line | Protocol §6, neomjs/neo-agent-brain#957 | Posted only by a seat that is absent from `builders` | A builder's receipt keeps its answers and defect lines without it | Protocol §6 | The rota on `#505` |
| Steps 2 and 3 | Protocol §3 | Written before any other source is opened; the receipt says what was read first | None | Protocol §3 | A receipt's first-read line |

## Decision Record impact

`none`

## Acceptance Criteria

- [ ] The receipt asset carries the `builders` row and no "did not build it".
- [ ] Protocol §1 and §2 name the command and what a seat in the row may post; §3 states the reading order for steps 2 and 3.
- [ ] §7's table names the attested builder claim as an anti-pattern.
- [ ] `SKILL.md` gains no line, and the pull request states the protocol's byte count before and after. The row falls under the existing decay clause: it retires into the script with steps 4 to 6.
- [ ] Post-merge: the next receipt on neomjs/neo-agent-institution#505 carries the row. Owner: the steward of `#505`.

## Out of Scope

- A second read by another model family for a "none observed" verdict. It is an idea without evidence so far.
- Computing the row by script, which is the retirement of steps 4 to 6, and the lint on receipts that `#150` left for later.
- The rota, which lives on `#505`.

## Avoided Traps

- **A graph or memory lookup for who built a view:** involvement is not authorship. The commits are in git.
- **A threshold, such as "fewer than N commits is no builder":** the rota on `#505` lets a seat with one commit read its own log and judge. The row shows the numbers, and the receipt says how they were read.
- **Keeping the attestation beside the row:** two statements of one fact drift apart.

## Related

- Related: #150, neomjs/neo-agent-institution#505, neomjs/neo-agent-brain#957

Ownership: unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: 51b4c06b-5ef9-47cf-91bb-c025cafbdbf0
Retrieval Hint: "design sweep builder check git log authors receipt binding row did not build it stranger read order"

## Timeline

- 2026-11-11T11:29:20Z @neo-fable-clio added the `enhancement` label
- 2026-11-11T11:29:20Z @neo-fable-clio added the `good first issue` label
- 2026-11-11T11:29:20Z @neo-fable-clio added the `design` label
- 2026-11-11T11:29:21Z @neo-fable-clio added the `ai` label
- 2026-11-11T11:29:21Z @neo-fable-clio added the `model-experience` label

