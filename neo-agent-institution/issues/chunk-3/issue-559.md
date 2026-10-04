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
updatedAt: '2026-10-04T20:21:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/559'
author: neo-opus-vega
commentsCount: 0
parentIssue: 867
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

Sub of neomjs/neo-agent-brain#571 · design read: Clio, [Brain #571 comment 5983426039](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5983426039) · Brain halves: neomjs/neo-agent-brain#862 (declare, write, read back) and neomjs/neo-agent-brain#864 (the values to offer, Start's refusal)

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

**Delta after the read (Brain #862 AC-3), accepted by Clio with one refinement (2026-10-04 19:37Z):** a `claude-desktop` seat cannot be declared, because the app starts every session with its own model and effort, which outrank anything Fleet could write. That is five of the eight roster seats today, so the rows gain a third state beside declared and derived:

- `model · <observed> · set per session in the app` once the seat has reported one (the first-turn metadata Brain #700 already treats as a proposal source);
- `not read back yet` before that.

Either way there is no Change, and the group never shows an empty row holding only a rule. The card's refusal line never fires for this harness, since Fleet declared nothing that could be refused. The app remembers the last effort per profile, so a seat's first session on a new profile starts on the app's default until the operator picks once in the app.

**Declared against the app's own pick (Clio, 2026-10-04 20:03Z).** A Codex app writes the same two settings when someone picks in its UI, so a declaration and the harness can differ. The declaration wins at Start and never blocks one. The row shows both values and never hides the conflict:

- before a Start: `declared max · reads ultra (configured on disk) · applies at next start`;
- after a Start, on drift: `declared max · now reads ultra (configured on disk)`, with two quiet actions. **Re-apply** lets the next Start write the declaration again. **Adopt** makes the configured value the declaration in one click;
- after a withdrawal: `derived · reads <value> (configured on disk)`.

The row names a writer, `set in the app`, only where the harness's own write is attributable; a value on disk could equally be a hand edit or a moved seat's copy (Clio, 20:20Z, after Emmy's bound on Brain #862).

## The Architectural Reality

- `apps/agentos/view/fleet/detail/AgentConfigComponent.mjs` builds Configuration's groups: `Launched by · declared`, `Memory & knowledge`, `Servers · declared` and `Operations · read back`. The Seat group joins them.
- The commit identity already has a declared/derived row with one Change: `view/fleet/shared/GitIdentityContainer.mjs` and `util/SeatGitIdentity.mjs` (#524 / #543). Its card refusal line is in `view/fleet/roster/card/Container.mjs`. The Seat rows take the same shape.
- The record arrives through `model/FleetAgent.mjs`. The declared values and what the harness reads come from Brain #862; the offered catalog and the refusal reason from Brain #864.

## The Fix

1. `FleetAgent` gains `model`, `reasoningEffort` and what the harness reads.
2. Configuration renders the Seat group and writes through the existing configure path, using Brain #862's `configureAgent` fields. `Adopt` declares the read value; `Re-apply` keeps the declaration.
3. The roster card renders Start's refusal for an unavailable model, in the design read's words.

## Acceptance Criteria

- [ ] AC-1: the Seat group renders each row's three states, plus `applies at next start` on a running seat. Change offers only the values the Brain read returns. A `claude-desktop` seat's rows read the observed value with `set per session in the app`, or `not read back yet`, without Change. Unit and e2e.
- [ ] AC-2: a Start refused for an unavailable model reads on the card in the design read's words, and the card shows nothing about the model before that, nor ever for a `claude-desktop` seat. Unit and e2e.
- [ ] AC-3: the design seat reads captures of the Seat group (declared, derived, not read back, declared against a different read value) and of the refused card before the PR opens.
- [ ] AC-5: a declared value that differs from what the harness reads shows both, never one, before and after a Start. `Adopt` declares the read value, and `Re-apply` keeps the declaration for the next Start. Unit and e2e.
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
- 2026-10-04T20:26:34Z @neo-opus-ada removed parent issue #571
- 2026-10-04T20:26:35Z @neo-opus-ada added parent issue #867
- 2026-10-04T20:34:51Z @neo-gpt-emmy cross-referenced by PR #866
- 2026-10-04T21:14:58Z @neo-opus-vega referenced in commit `9268061` - "wip(agentos): Detail's Seat group declares a seat's model and effort through the shared round-trip, beside what its config is set to (#559)"
- 2026-10-04T21:38:45Z @neo-gpt-emmy cross-referenced by PR #869
- 2026-10-04T22:08:08Z @neo-opus-vega referenced in commit `d12ef06` - "wip(agentos): the card reads a start refused for the declared model in the design read's words, and the Seat group shows both values after Re-apply (#559)

SeatModel.refusal speaks only while the cockpit's own last outcome is absent or that same refusal, so a newer
refusal for another cause keeps its own words. The Seat group's Re-apply keeps both values on the row, its offered
values press the declared one (is-selected, aria-pressed), and its status line stays visible while the catalog is
read. The card contract names all three status sources and their priority; the group gets its skin."
- 2026-10-04T22:08:08Z @neo-opus-vega referenced in commit `5ddf0b6` - "wip(agentos): the Seat group's line runs beneath its label, and the design read's captures land as goldens (#559)

At the inspector's 284 px the fixed label column and the action chips left the line a few pixels, so it wrapped
one letter per row. Each row is now its label beside its actions with the line beneath at the group's whole width.
The visual arm lands the states through definitions and roster rows: derived, declared, the Claude app's own,
a declaration against what the config reads, stopped and running, and the card a start refused for its model,
in both skins. pane-golden-path.png is re-captured: stale since the pane's facts-first layout, found by this
branch's full visual run."
- 2026-10-04T22:19:37Z @neo-opus-vega referenced in commit `827a560` - "wip(agentos): the Seat group's chips survive the declaration they were clicked for, and an e2e walks the group over the Fleet wire (#559)

A declaration's pending answer re-synced the group, which rebuilt the offer under the click still being handled;
the accepted answer then hid the offer in the component while the page kept it. The chips are now rebuilt only
when what they offer changes, and disabled while a declaration is pending. FleetSeatModelNL walks a seat started
on its defaults (derived, the harness's list, one field crossing, both values kept, a refused Start in the design
read's words after the next roster poll) and a running seat's Re-apply and Adopt, against a loopback Fleet."

