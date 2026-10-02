---
id: 421
title: The setup card's design contract — four states from the recipe's output
state: CLOSED
labels:
  - documentation
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-10-02T08:57:56Z'
updatedAt: '2026-10-02T10:06:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/421'
author: neo-fable-clio
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 384 The cockpit projects the first-run recipe inline, never as a gate'
closedAt: '2026-10-02T10:06:47Z'
milestone: FM v1
---
# The setup card's design contract — four states from the recipe's output

## Context

#384 (the Create door: the cockpit projects the first-run recipe inline, never as a gate) names the design seat's deliverable before a builder starts: a capture or mock on the ticket. This cockpit keeps its designs as pages under `apps/agentos/design/` — the mailbox, memories, tasks and catch-up contracts, the two cockpit plans — and the builder reads the page, not a chat. This leaf is that page for the setup card. Parent: #351 (the wizard epic); it blocks #384's build.

## The Problem

A builder picking up #384 has the recipe's step JSON (neomjs/neo-agent-brain#679 → PR #732), the probe (neomjs/neo-agent-brain#685), the presets (neomjs/neo-agent-brain#686) and the shell rule (#12), and no picture of the card: which content is primary at a cold boot, what the frame shows once the card is dismissed, how a `reconcile-required` row reads beside a plane that is not the target's, what first persistence looks like. Without the page each of those is decided at implementation time and reviewed from pixels — the review surface #13 exists to prevent.

## The Architectural Reality

- The recipe's output is the design's input: an `evaluate()` row is `{id, kind, status, reason, summary, effectId, receipt, observedAt, answer, placement, served, observed}` (Brain `ai/services/fleet/firstRunRecipe.mjs`), the budgets are the probe's, the preset facts are the table's. A design that writes its own status text re-creates the stored-status anti-anchor ADR 0041 is written against — every row the page shows must be a line the CLI printed.
- The card family is `apps/agentos/view/PlaneSetupPanel.mjs` with `resources/scss/src/apps/agentos/PlaneSetupPanel.scss` (the Connect card): the signal left mark, the quiet / primary button pair, the body-role lede, the FM font on the field. The Create door is the same family.
- Tokens: `apps/agentos/TOKENS.md` — the five type roles; `--fm-ink-faint` is decorative only (D4), so no indicator sits on it; the stamp input set excludes `apps/agentos/design` (`buildScripts/checkVisualBaselines.mjs`), so a page there moves no golden.

## The Fix

One page, `apps/agentos/design/first-run-setup-card.html`, in the sibling pages' format (the `:root` mirror of the dark skin's literal values and type roles, the eyebrow / lede / `col-label` sections, the field → element table, the falsifier drills). It renders four states from the recipe's real `--json` output on fake hosts: a cold boot on the probe's own 32 GiB fixture (nothing recommended, each preset with its reason), the card dismissed with the frame operable and every empty pane labelled, a resume with a `pending` receipt turned `reconcile-required` while another plane answers, and first persistence with the quiet confirmation. Then the field → element contract (every element to a step, probe or preset field), the main-process IPC table (the six channels, what never crosses), the implementation destination (#24's idiom: a Panel of the `PlaneSetupPanel` family, a `Neo.list` over a step Store, the run on the view root's provider) and the density count for the path. Captured headlessly; the author's design pass before the PR leaves draft.

## Acceptance Criteria

- [ ] AC-1 The page exists at that path in the sibling format and renders the four states; every step row's status and reason is a line of the recipe's output at a named Brain commit (the #732 → #736 stack or later).
- [ ] AC-2 Every element of the card maps to a step JSON field, a probe field or a preset field in the page's contract table; the page carries no invented state.
- [ ] AC-3 The page mirrors the FM dark skin's literal token values and the five type roles; every status reads with two channels (glyph + word); no indicator uses `--fm-ink-faint`; the progress count equals the `ok` rows shown.
- [ ] AC-4 The author's design pass on the four frames is recorded on the PR with the capture recipe that reproduces them, and #384 links the page as the design it waits for. *(Reworded 2026-10-02 by the author before the PR: the frames are reproducible from the recipe rather than attached as binaries.)*

## Out of Scope

The build (#384): the Panel, the Store, the IPC, the e2e witnesses, the goldens. Any change to `PlaneSetupPanel.mjs` or its SCSS. The ROADMAP's design-SSOT list.

## Avoided Traps

Status text written for the mock. A second skin for the page instead of the SCSS's literal values. A path the stamp reads (a golden would move for a page). A capture posted as the deliverable without the page (the builder needs the contract tables, not only the picture).

## Related

#384 (the build this blocks) · #351 (parent) · #12 (the shell rule) · #13 (design conformance) · #24 (the idiom gate) · neomjs/neo-agent-brain#679 / PR #732 · neomjs/neo-agent-brain#686 / PR #715 · neomjs/neo-agent-brain#685 / PR #707 · neomjs/neo-agent-brain#696 / PR #736 · ADR 0041 · `#20` (the sibling pages' ticket)

Decision Record impact: aligned-with ADR 0041 (the cockpit projects, never writes).

Sweeps: live latest-open sweep — the latest 20 open Institution issues read at 2026-10-02T08:56Z (newest #418), no equivalent (#384 is the build, #12 the rule); A2A in-flight sweep — the last 30 messages at 08:57Z, all read-states, no claim on a setup card beyond the author's own 08:18Z design-half claim on #384; Memory Core rationale sweep — nothing on this surface beyond today's lane; own-assignment sweep — #384, #351; structure map — N/A (a design page, no `.mjs`).

Origin Session ID: 1efa16ff-bd83-41e5-87dc-4c186b03b451
Retrieval Hint: "setup card design contract four states recipe output create door"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1efa16ff-bd83-41e5-87dc-4c186b03b451


## Timeline

- 2026-10-02T08:57:56Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-02T08:57:57Z @neo-fable-clio added the `documentation` label
- 2026-10-02T08:57:58Z @neo-fable-clio added the `enhancement` label
- 2026-10-02T08:57:58Z @neo-fable-clio added the `agent-os` label
- 2026-10-02T08:57:58Z @neo-fable-clio added the `ai` label
- 2026-10-02T08:57:58Z @neo-fable-clio added the `design` label
- 2026-10-02T08:58:12Z @neo-fable-clio added parent issue #351
- 2026-10-02T08:58:14Z @neo-fable-clio marked this issue as blocking #384
- 2026-10-02T08:58:30Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-10-02T08:59:46Z @neo-fable-clio cross-referenced by PR #422
- 2026-10-02T09:22:24Z @neo-fable-clio referenced in commit `e3c2092` - "docs(agentos): the setup card's density count is a defined target, not a measured path (#421)

Review round 1: a decision is an answered question or a consent to an effect, a manual action a step whose action is an operator instruction; placement is defaulted; the 32 GiB path targets 3 + 3 decisions since hosted is its only possible preset and asks for its key; the measured count stays the completed run's receipt on the epic."
- 2026-10-02T09:22:46Z @neo-fable-clio cross-referenced by #384
- 2026-10-02T10:06:47Z @tobiu referenced in commit `3324bc4` - "docs(agentos): the setup card's design contract, four states from the recipe's output (#421) (#422)

* docs(agentos): the setup card's design contract, four states from the recipe's output (#421)

The Create door of the PlaneSetupPanel family as a design page in the sibling format: a cold boot on the probe's 32 GiB fixture, the card dismissed with the frame operable, a resume with a pending receipt turned reconcile-required while another plane answers, and first persistence — every step row a line the recipe's CLI printed on Brain dbb9d53 — then the field-to-element contract, the six main-process IPC channels, the implementation destination and the density count.

* docs(agentos): the setup card's density count is a defined target, not a measured path (#421)

Review round 1: a decision is an answered question or a consent to an effect, a manual action a step whose action is an operator instruction; placement is defaulted; the 32 GiB path targets 3 + 3 decisions since hosted is its only possible preset and asks for its key; the measured count stays the completed run's receipt on the epic."
- 2026-10-02T10:06:47Z @tobiu closed this issue
- 2026-10-02T10:32:09Z @neo-fable-clio cross-referenced by PR #432

