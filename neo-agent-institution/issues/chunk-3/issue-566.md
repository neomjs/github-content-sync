---
id: 566
title: The activity recipient gets its avatar and the new-events pill its skin
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-ada
createdAt: '2026-10-05T11:05:31Z'
updatedAt: '2026-10-05T13:12:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/566'
author: neo-opus-ada
commentsCount: 0
parentIssue: 505
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-05T13:12:59Z'
---
# The activity recipient gets its avatar and the new-events pill its skin

## Context

The operator reported two problems in the Activity pane on 2026-10-05, in-session, from a screenshot of an older Electron shell:
- the "N new events ↑" pill "lacks styling (looks alien)";
- in A2A rows the "sender gets a github avatar icon, receiver does not get it (even though we have one for most receivers)".

Both are verified on Institution `dev`.

## The Problem

1. **The pill is a stock engine button.** `apps/agentos/view/fleet/activity/Container.mjs` declares `{module: Button, cls: ['fm-stream-new-events'], hidden: true, reference: 'new-events'}` with no `ui`, and `.fm-stream-new-events` (activity `Container.scss`) only positions it. So it renders as the engine's default filled primary button, floating over the FM's dark rail.
2. **The recipient is plain text.** `RowContainer.mjs` renders `→ <to>` or `⇒ fleet` in a plain `Component` (`fm-ev-recipient`). The sender is an `ActorChip` carrying the roster avatar from the same `actorDirectory`.

## The Architectural Reality

- Design authority, pill: `Viewport.scss`, *"THE GHOST SKIN — one for the whole shell: dim ink on no surface, a soft field under the pointer, and a pressed choice carried by ink plus a hairline."*
- Design authority, recipient: the operator, 2026-10-05, in-session. Asked whether a direct recipient should render as the sender's avatar chip, he answered "Yes, avatar chip": `→ [avatar] name`, with broadcasts keeping `⇒ fleet`.
- `ActorChip` (`ActorChipComponent.mjs`) renders an optional avatar and a label. The row looks up the sender's facts in `actorDirectory` by id, with or without `@`.
- The row's anatomy is fixed: five cells, and *"the component and DOM tree never changes shape."* The pill floats over pooled `list.Buffered` rows, so it needs an opaque surface.
- Goldens `activity-stream-chips.png` and `pane-activity.png` (`FleetCockpitVisual`) cover the pane. The visual stamp entry is shared with #561 and #564, so this PR re-stamps after whichever of those lands first.

## The Fix

- **Pill:** `ui: 'ghost'`. `.fm-stream-new-events` re-values the ghost variables to an opaque overlay (`--fm-panel-2` surface, `--fm-line` hairline, `--fm-ink` text) and keeps its position and shadow.
- **Recipient:** the `recipient` cell becomes an `ActorChip` that renders a lead arrow before the avatar (`→` direct, `⇒` broadcast).
  - A direct recipient takes its avatar and label from `actorDirectory`, as the sender does; the label is the display name, else the handle without `@`.
  - A broadcast keeps `fleet` and no avatar.
  - A row with no recipient keeps the empty cell.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `ActorChip.lead` (`ActorChipComponent.mjs`, reactive `lead_`), consumed by `RowContainer`'s recipient cell | the operator's 2026-10-05 recipient ruling (Architectural Reality) | optional `String` or `null`, default `null`; renders one mark before the avatar and text: `→` for a direct recipient, `⇒` for a broadcast | `null` renders no mark (the sender chip, and every existing consumer) | the config's JSDoc | `container.spec.mjs` recipient test (unit); the activity goldens (visual) |

## Acceptance Criteria

- [ ] AC-1: the new-events pill renders with the shell's ghost skin on an opaque overlay surface, never the engine's default primary fill (unit: the rendered pill carries the shell's ghost class; the live pill is checked on the next installed candidate, because the visual harness cannot raise it: the pill needs live arrivals while the list is scrolled away from the top).
  Residual-Owner: #505 (the installed pill read is deferred to the next installed candidate)
- [ ] AC-2: a direct A2A row renders `→ [avatar] name` for a recipient the actor directory knows, and `→ name` without an image for one it does not (unit).
- [ ] AC-3: a broadcast row renders `⇒ fleet` without an avatar, and a non-A2A row keeps the empty recipient cell (unit).
- [ ] AC-4: the row keeps its five-cell anatomy and its pooled-row behaviour (the existing activity specs stay green), and the two goldens are re-captured and stamped.

## Out of Scope

- How the mailbox pane renders recipients.
- Avatars for identities the roster does not carry (e.g. `@tobiu` today).

## Related

Parent: #505 (renders correctly, can be read). It shares the visual stamp entry with #561 and #564.

Sweeps:
- Live latest-open: the latest 20 open Institution issues at 11:05Z, no equivalent.
- Keyword: `activity avatar recipient` and `new events button activity` returned nothing.
- A2A: the last 30 messages in all read states, no claim on the activity pane.
- Memory Core: no prior decision.
- Own assignments: #424 and #516, neither overlapping.

Origin Session ID: 5267f5db-e1d4-4297-8570-4981234133aa
Retrieval Hint: "activity feed recipient avatar actor chip new-events pill ghost skin"




## Timeline

- 2026-10-05T11:05:32Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-05T11:05:33Z @neo-opus-ada added the `bug` label
- 2026-10-05T11:05:33Z @neo-opus-ada added the `agent-os` label
- 2026-10-05T11:05:33Z @neo-opus-ada added the `ai` label
- 2026-10-05T11:05:33Z @neo-opus-ada added the `design` label
- 2026-10-05T11:05:38Z @neo-opus-ada added parent issue #505
- 2026-10-05T11:28:17Z @neo-opus-ada cross-referenced by PR #567
- 2026-10-05T12:01:29Z @neo-opus-ada referenced in commit `7f4ea11` - "fix(agentos): the new-events pill keeps its overlay field and hairline while pressed (#566)

Euclid's RA-1: the pill inherited the shell ghost skin's active tokens, so a
pressed pill turned translucent (alpha 0.22) with a transparent border over
the rows. The active background now mixes onto --fm-panel-2 and the active
border keeps the overlay hairline, like the normal and hover states."
- 2026-10-05T13:12:59Z @tobiu referenced in commit `5426c89` - "feat(agentos): the activity recipient gets its avatar chip and the new-events pill the shell's skin (#566) (#567)

* feat(agentos): the activity recipient gets its avatar chip and the new-events pill the shell's skin (#566)

A direct A2A recipient renders as the sender's ActorChip, with the arrow
before the roster avatar; a broadcast keeps '⇒ fleet'. ActorChip gains an
optional lead mark. The new-events pill takes ui 'ghost' and re-values the
ghost variables onto an opaque overlay surface. The fixture's first A2A
row is now a direct message, so the goldens show both recipient kinds.
Five goldens re-captured; the stamp is regenerated.

* fix(agentos): the new-events pill keeps its overlay field and hairline while pressed (#566)

Euclid's RA-1: the pill inherited the shell ghost skin's active tokens, so a
pressed pill turned translucent (alpha 0.22) with a transparent border over
the rows. The active background now mixes onto --fm-panel-2 and the active
border keeps the overlay hairline, like the normal and hover states."
- 2026-10-05T13:13:00Z @tobiu closed this issue

