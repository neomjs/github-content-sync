---
id: 559
title: Detail's Seat group declares a seat's model and reasoning effort
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-10-04T19:26:57Z'
updatedAt: '2026-10-04T19:32:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/559'
author: neo-opus-vega
commentsCount: 0
parentIssue: 571
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
# Detail's Seat group declares a seat's model and reasoning effort

Sub of neomjs/neo-agent-brain#571 · design read: Clio, [Brain #571 comment 5983426039](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5983426039) · Brain half: neomjs/neo-agent-brain#862

## Context

On 2026-10-04 the operator asked that Start set each seat's model and thought level. A harness started on its defaults can run a seat below its intended level until someone adjusts it inside the harness. Clio's design read puts the setting in Agent Detail, never in Add.

## Journey delta (design read: Clio, 2026-10-04 19:12Z)

Add stays as it is: name, one PAT, play. A stranger gets the harness default and never meets the question.

Detail › Configuration gains a **Seat** group with two rows, `model` and `reasoning effort`:

- Each row reads `declared <value>`, `derived from <harness default>` or `not read back yet`.
- Each row has one inline **Change**, which offers the harness's own catalog.
- A change on a running seat reads `applies at next start`.
- A moved seat brings its own values, prefilled.

The roster card says nothing until a Start is refused. Then it reads `start refused: model <x> is not available — change it in Detail › Configuration`.

**Delta after the read (Brain #862 AC-3; it needs the design seat's read before the PR):** a `claude-desktop` seat cannot be declared. That is five of the eight roster seats. The app starts every session with its own model and effort, and those outrank anything Fleet could write. The rows therefore read `set per session in the app`, with no Change. The app remembers the last effort per profile, so a seat's first session on a new profile starts on the app's default until the operator picks once in the app.

## The Architectural Reality

- `apps/agentos/view/fleet/detail/AgentConfigComponent.mjs` builds Configuration's groups: `Launched by · declared`, `Memory & knowledge`, `Servers · declared` and `Operations · read back`. The Seat group joins them.
- The commit identity already has a declared/derived row with one Change: `view/fleet/shared/GitIdentityContainer.mjs` and `util/SeatGitIdentity.mjs` (#524 / #543). Its card refusal line is in `view/fleet/roster/card/Container.mjs`. The Seat rows take the same shape.
- The record arrives through `model/FleetAgent.mjs`. The values, the offered catalog and the refusal reason come from Brain #862.

## The Fix

1. `FleetAgent` gains `model` and `reasoningEffort`.
2. Configuration renders the Seat group and writes through the existing configure path, using Brain #862's `configureAgent` fields.
3. The roster card renders Start's refusal for an unavailable model, in the design read's words.

## Acceptance Criteria

- [ ] AC-1: the Seat group renders each row's three states, plus `applies at next start` on a running seat. Change offers only the values the Brain read returns. A `claude-desktop` seat's rows read `set per session in the app`, without Change. Unit and e2e.
- [ ] AC-2: a Start refused for an unavailable model reads on the card in the design read's words, and the card shows nothing about the model before that. Unit and e2e.
- [ ] AC-3: the design seat reads captures of the Seat group (declared, derived, not read back) and of the refused card before the PR opens.
- [ ] AC-4 (post-merge, installed): on the next #12 candidate, the operator sets a `codex-desktop` seat's model and effort in Configuration, and the next Start reads them back.

## Out of Scope

- Add Agent, which stays unchanged.
- The family row's derivation from a declared model: Brain #700 (Sophie).
- Writing the launch configuration: Brain #862.

## Related

Brain #571 (epic) · Brain #862 · #524 / #543 (the identity row this mirrors) · Brain #700 · #12

Live latest-open sweep: latest 20 open issues in `neo-agent-institution` and `neo-agent-brain` at 19:26Z, plus a title/body search for model and effort; no equivalent found.
A2A in-flight sweep: the last 30 messages (16:57Z–19:23Z); no claim on a seat's model or effort.
MC sweep: "harness starts on its default thought level, peers less smart unless the operator adjusts the model inside the harness", 6 results, no prior decision found.
Own-assignment sweep: 3 open in Institution and 10 in Brain, none on Configuration.

Origin Session ID: 15ff44b9-9b0e-48b5-af34-9833bdfdf2f1
Retrieval Hint: "Configuration Seat group model reasoning effort declared derived"


## Timeline

- 2026-10-04T19:26:58Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-04T19:26:59Z @neo-opus-vega added the `enhancement` label
- 2026-10-04T19:26:59Z @neo-opus-vega added the `agent-os` label
- 2026-10-04T19:26:59Z @neo-opus-vega added the `ai` label
- 2026-10-04T19:26:59Z @neo-opus-vega added the `design` label
- 2026-10-04T19:27:13Z @neo-opus-vega cross-referenced by #862
- 2026-10-04T19:27:16Z @neo-opus-vega added parent issue #571

