---
id: 392
title: ticket-create's sweep misses an open owner older than the latest 20
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-opus-vega
createdAt: '2026-11-14T22:34:36Z'
updatedAt: '2026-11-15T00:19:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/392'
author: neo-opus-vega
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
closedAt: '2026-11-15T00:19:15Z'
---
# ticket-create's sweep misses an open owner older than the latest 20

## Context

neomjs/neo-agent-brain#1438, the Brain backlog sweep, has all three ACs met: 123 premise reads, each verdict on its own ticket. Its "awareness fix" section names two changes, so that a forgotten backlog does not grow back. The first shipped with neomjs/neo-agent-brain#762: the wake digest renders a seat's own open work. This ticket is the second, the one that is still open: the filing protocol steers toward fresh tickets. #1438 opened on the operator's 2026-11-02 words: "the team mostly creates new tickets on the fly, while there are hidden gems inside the already very big backlog."

## The Problem

The mandatory content sweep, `ticket-create-workflow.md` §1a(i), reads the latest 20 open issues and searches closed state on the problem's nouns. No mandatory command reaches an open ticket on the same subject that is older than the latest 20. Only the semantic sweep might find it, and that sweep is listed after the mandatory ones. #296 added the backlog band, and in orange or red it says to fold the work into an owning ticket or epic. The sweep never looks for that owner where it most likely is: open, and older.

Measured on this repository at 2026-11-14T22:34Z (38 open, orange): the open search for `ticket-create sweep` returns 15 hits. Five of them (#5, #6, #298, #299, #302) sit below the latest 20, and #6 is the open ticket that owns this same file's other concern.

## The Architectural Reality

- `.agents/skills/ticket-create/references/ticket-create-workflow.md:22-35`: §1a(i) holds its two commands, the attestation line and the backlog band. The file is skill-loaded at ticket creation, not turn-loaded.
- `.agents/skills/pr-review/references/reviewer-instrument-audit.md:27` quotes the attestation line only as an example of a routine receipt, so it needs no change.

## The Fix

In §1a(i), add `gh issue list --state open --search "<problem nouns>" --limit 15` between the two commands. The step's lead sentence says why, in one clause: an open owner older than the latest 20 is the one a fold needs. The attestation line records an "open and closed search". The package version is bumped.

## Acceptance Criteria

- [ ] AC-1: §1a(i) runs an open-state search on the problem's nouns beside the latest-20 read and the closed search, says why in one clause, and its attestation line records the search.
- [ ] AC-2: `npm run lint`, `npm run lint -- --base origin/dev` and `npm test` pass, and the version is bumped.

## Out of Scope

- Making the semantic sweep mandatory, or adding a lint.
- #6's incident mode (same file, another concern).

## Avoided Traps

- **`--state all` in place of the closed search.** It returns one created-descending list of 15, so recent closed hits push out exactly the old open owner the step is for.

## Related

neomjs/neo-agent-brain#1438 (its "awareness fix" section names this as part two) · neomjs/neo-agent-brain#762 (part one) · #296 (the backlog band)

Decision Record impact: none.
Structure map: N/A. The change places no file, and `ai:structure-map` is Brain-hosted.
Live latest-open + closed sweep: checked latest 20 open + open and closed search (`ticket-create sweep`; Out of Scope read on #296) at 2026-11-14T22:34Z; no equivalent or prior exclusion found. Backlog: 38 open (orange); no open ticket or epic owns the sweep's shape.
A2A sweep: the last 30 messages up to 22:31Z; no claim on ticket-create.
MC sweep: "a new ticket filed while an older open ticket on the same subject sits in the backlog", 5 results; no decision on the sweep's shape.
Own-assignment sweep: 4 open (#388, #6, #5, #302). #6 touches the same file for incident mode and does not overlap.

Origin Session ID: 7b2f3090-eee3-44a8-bcc6-3cb93187adf8
Retrieval Hint: "ticket-create sweep open owner older than latest 20, backlog gems"

## Timeline

- 2026-11-14T22:34:36Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-14T22:34:38Z @neo-opus-vega added the `enhancement` label
- 2026-11-14T22:34:38Z @neo-opus-vega added the `ai` label
- 2026-11-14T22:34:38Z @neo-opus-vega added the `model-experience` label
- 2026-11-14T22:35:11Z @github-actions added the `needs-design-authority` label
- 2026-11-14T22:38:48Z @neo-opus-vega cross-referenced by PR #393
- 2026-11-14T22:38:58Z @neo-opus-vega cross-referenced by #1438
- 2026-11-14T22:45:09Z @neo-opus-vega referenced in commit `ed10523` - "chore: merge dev after #389 and keep version 1.9.5 (#392)"
- 2026-11-14T23:18:37Z @neo-opus-vega referenced in commit `d606b78` - "chore: merge dev after #391 and move to version 1.9.6 (#392)"
- 2026-11-15T00:19:15Z @tobiu referenced in commit `cbbd6fb` - "feat(ticket-create): the sweep searches open issues by the problem's nouns (#392) (#393)

Step (i) of the content sweep read the latest 20 open issues and searched
closed state on the problem's nouns, so an open ticket on the same subject
older than the latest 20 was reached by no mandatory command. The backlog
band's fold-into-an-owner rule needs exactly that ticket. The step now runs
an open-state noun search between the two, says why in one clause, and its
attestation records "open and closed search".

Two sentences of the same section are compressed to keep the payload under
its 25,000-byte budget. Version 1.9.5: #389 holds 1.9.4."
- 2026-11-15T00:19:16Z @tobiu closed this issue

