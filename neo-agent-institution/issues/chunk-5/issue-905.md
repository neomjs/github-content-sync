---
id: 905
title: Catch up reads a failed source as failed and scrolls to its end
state: OPEN
labels:
  - bug
  - good first issue
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-11-10T19:04:24Z'
updatedAt: '2026-11-10T19:05:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/905'
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
milestone: FM v1
---
# Catch up reads a failed source as failed and scrolls to its end

## Context

A design sweep of the installed Fleet Manager on 2026-11-10 read the Catch up pane on the team's own plane while both of its sources were degraded: the memory read answered without a synthesis, and the pull-request read could not sign in to GitHub. Texts, rects and timings are in the `catchUp` section of the [receipt](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6776500148) (Institution `86ca206d`, Brain `67537f93`). Everything below was observed there; nothing is inferred from source alone.

## The Problem

1. **A failed read is printed as an absence.** The pull-request card shows `No source records in this window.` and, one line below, `Coverage degraded · retrieval-failed: Could not authenticate with GitHub…`. The first sentence is false: the read failed. `sourceCardConfig()` picks the body by `coverage.totalResolved === 0` before it looks at the failure.
2. **The pane cannot be read to its end.** `.fm-catch-up-sources` computes `overflow: hidden` at 533 px while its two cards need 586 px, so the second card's body and reason are cut off and nothing scrolls. Inside a card every child computes `flex: 1 1 0%`: single lines sit in 68 px slots, and 20 citations share 71 px, so 16 are clipped.
3. **A pending read looks like no read.** A read took 8 to 16 s on this plane, and the pane showed its previous state unchanged until the answer landed.

## The Architectural Reality

- `apps/agentos/view/fleet/catchup/Container.mjs`: `sourceCardConfig()` selects a card's body; `applySnapshot()` projects an answer; a read is a fired `historyRequest`, answered when the owning cockpit writes `snapshot`.
- `resources/scss/src/apps/agentos/fleet/catchup/Container.scss`: the root declares `overflow: auto` and, in its header, "inner-body scroll ownership". The measured boxes show the sources container clipping instead.
- Design authority: `apps/agentos/design/institution-catchup-pane.html` §6, "Partial: a band whose source failed says so in one line with the source's reason, and the other bands render. An unavailable source never reads as "nothing happened"", and "Reading: one line under the head, and no bands yet." `ROADMAP.md` row 2: "every surface names cold · live · stale · degraded · unreachable with its reason and the next step". #309 lists "silently equating an unavailable source with no changes" among its avoided traps.

## The Fix

In the installed pane, without redesigning it:

- a card whose source is `unavailable`, or `degraded` with a retrieval failure, prints that failure as its body and never the empty-window sentence;
- the sources area scrolls on the vertical axis, and a card's lines keep their natural height;
- from a fired read until its answer, one line under the head says that a read is in flight.

## Acceptance Criteria

- [ ] AC-1 For a slot whose envelope carries `coverage.totalResolved: 0` together with a `retrieval-failed` reason, the card's body is the failure. `No source records in this window.` appears only when the read succeeded and found nothing. One unit arm covers both cases.
- [ ] AC-2 With two cards and 20 citations in a pane 660 px high, every card and every rendered citation can be scrolled into view, and no card child is taller than its content. The arm asserts containment against the scrolling parent, not an element's own box.
- [ ] AC-3 Between `historyRequest` and the next `snapshot` write the pane shows one in-flight line; it leaves when the answer lands, also when that answer is `unavailable`.
- [ ] AC-4 Post-merge: a design-sweep receipt on the next installed candidate records the three states.

## Out of Scope

The accepted redesign (v2 of the design page), the window that is lost at the next read, the partition chips, and the six source gaps the page lists. They belong to the v2 build recorded on #505. No Brain change.

## Avoided Traps

- Restyling the first-use row or the action row. #309 names "reproducing the current controls with nicer CSS" as a trap; this leaf repairs truth and reach only.
- Asserting an element's own height. A child can measure right and still be clipped by its parent.

## Related

#505 (the views epic; its sub-issue list is full, so this leaf links by reference) · #309 and PR #318 (the design) · #898 (the same class on the Golden Path pane).

Live latest-open + closed sweep: checked the latest 20 open issues and closed searches ("Catch up", "Chat rail placeholder", "Accounts card read back hooks", "side by side 28rem"; Out of Scope and Avoided Traps read on #309 and #245) at 2026-11-10T19:03Z; no equivalent or prior exclusion found. Backlog band: 47 open, orange; no open ticket owns this pane. A2A claim sweep over the last 40 minutes at 19:03Z: none on this scope. Memory Core rationale sweep on the symptom: no prior decision. Own assignments (#505, #507, #351, #889): #505 is the parent by reference.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: 0c3a6369-4269-4b0b-8ad5-0c9441248925

Retrieval Hint: "Catch up pane failed pull-request read printed as No source records"

## Timeline

- 2026-11-10T19:04:26Z @neo-fable-clio added the `bug` label
- 2026-11-10T19:04:26Z @neo-fable-clio added the `good first issue` label
- 2026-11-10T19:04:26Z @neo-fable-clio added the `agent-os` label
- 2026-11-10T19:04:26Z @neo-fable-clio added the `ai` label
- 2026-11-10T19:04:27Z @neo-fable-clio added the `design` label
- 2026-11-10T19:05:52Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-11-10T19:07:33Z @neo-fable-clio cross-referenced by #505

