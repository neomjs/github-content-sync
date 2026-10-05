---
id: 867
title: A seat the Fleet starts runs on the model and reasoning effort declared for it
state: OPEN
labels:
  - epic
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-04T20:26:12Z'
updatedAt: '2026-10-04T20:26:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/867'
author: neo-opus-ada
commentsCount: 0
parentIssue: 571
subIssues:
  - '[x] 862 A Fleet seat starts on its declared model and reasoning effort'
  - '[x] 864 Configuration offers the models and efforts a seat''s harness names, and Start refuses one it lacks'
  - '[x] 559 Detail''s Seat group declares a seat''s model and reasoning effort'
subIssuesCompleted: 3
subIssuesTotal: 3
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# A seat the Fleet starts runs on the model and reasoning effort declared for it

Terminal predicate: a seat the Fleet starts runs on the model and reasoning effort declared for it. The operator declares them in the cockpit's Detail from the harness's own catalog, Start applies them through that harness's own surface or refuses one it lacks, and a peer moved into the Fleet keeps the level it ran at before the move.

## Problem scope

On 2026-10-04, while the peers joined the Fleet Manager roster, the operator asked that a Fleet-started harness start on its seat's model and thought level (for example Opus 5.5 at Max). Today every Fleet start runs on the harness default. No seat record carries a model or a reasoning effort, so a seat can run below its intended level until someone adjusts it inside the harness, and a moved peer silently loses the level it had.

For #571's move this is one of decision A's classes: the model and the preferences each get an explicit preserve, redeclare or retire decision, with a functional proof at the destination.

**Why an epic.** Three leaves across two repositories converge on this one outcome: the record and Start's write; the harness catalog and Start's refusal; the cockpit's declaration. It was split out of #571 to keep that epic's sub-set within bound (Vega's agreement, 2026-10-04).

## Intended solution shape

Clio's design read, [#571 comment 5983426039](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5983426039):
- The seat record carries a declared model and reasoning effort. Add stays as it is: a stranger gets the harness default and never meets the question.
- The values offered are the harness's own catalog, never a Fleet-side list: Codex's `model/list`, the claude-code CLI's named levels. A family that takes no declaration (claude-desktop) offers nothing.
- Start applies the declaration through each family's own surface (claude-code flags, Codex's `config.toml`) and reads back what it set. A value the harness lacks refuses the Start, naming it.
- Detail › Configuration gains a Seat group that declares both.

## Out of scope

- The move itself and the other preference classes of #571's gap 10.
- Model selection for families that take no declaration.

## Avoided traps

- **A Fleet-side model list.** It drifts from the harnesses, and ADR-0019 §3 rules out hidden defaults.
- **Asking at Add.** A stranger would meet a question about a harness they have not started yet.

Owner: @neo-opus-vega, who holds every leaf. Parent: #571.

Origin Session ID: 6b13f348-5848-47a1-8740-c4a9d1dfaea7
Retrieval Hint: "seat declared model reasoning effort harness catalog Start applies refuses Detail Seat group moved peer keeps level"

## Timeline

- 2026-10-04T20:26:12Z @neo-opus-ada assigned to @neo-opus-vega
- 2026-10-04T20:26:13Z @neo-opus-ada added the `epic` label
- 2026-10-04T20:26:13Z @neo-opus-ada added the `ai` label
- 2026-10-04T20:26:13Z @neo-opus-ada added the `agent-os` label
- 2026-10-04T20:26:21Z @neo-opus-ada added sub-issue #862
- 2026-10-04T20:26:23Z @neo-opus-ada added sub-issue #864
- 2026-10-04T20:26:25Z @neo-opus-ada added parent issue #571
- 2026-10-04T20:26:35Z @neo-opus-ada added sub-issue #559
- 2026-10-04T20:34:51Z @neo-gpt-emmy cross-referenced by PR #866
- 2026-10-04T21:38:45Z @neo-gpt-emmy cross-referenced by PR #869

