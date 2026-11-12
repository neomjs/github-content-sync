---
id: 927
title: The spine banner's lead repeats its pill and lists healthy sources
state: OPEN
labels:
  - bug
  - good first issue
  - ai
  - design
assignees: []
createdAt: '2026-11-12T02:15:42Z'
updatedAt: '2026-11-12T02:15:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/927'
author: neo-opus-ada
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
# The spine banner's lead repeats its pill and lists healthy sources

## Context

Operator, 2026-11-12 ~02:05Z, with a screenshot of the cockpit bar on the installed FM (Institution `499339f`): "the header toolbar is overloaded with a way to long description." The bar read:
- pill `feed partial`;
- lead `Activity feed partial — PR lanes partial · mailbox available · Plane outage 03:02 AM–03:03…`, clipped;
- then `wake off`, Recheck, the start summary, and Stop / Start.

Defect-note fingerprint `3a12cd5b81eb241d` (raw).

## The Problem

The lead carries three things that are not "what is wrong now":
1. **It repeats the pill.** The pill says `feed partial`, and the lead opens with `Activity feed partial —`.
2. **It lists healthy sources.** `mailbox available` comes from `FleetAdmission`'s partial guidance, whose `summary` joins the missing parts with the remaining (`wired`) parts.
3. **It appends a recovered outage.** `SpineBanner.formatShortOutage` returns `Plane outage <window> — recovered`, and the partial branch adds it to the lead.

The line ends up wider than the bar and clips mid-phrase.

## The Architectural Reality

Design authority: the header IA (`apps/agentos/design/institution-header-detail-ia.html`):
- "the reason, in product words … the lead: one visible line beside the pill, truncated with an ellipsis, never only on hover";
- the pill's title "repeats the lead and adds the cause";
- its label-language law names this failure mode: "the axis word duplicates itself, and the sentence clips mid-word".

The lead is for the reason, nothing else. The code today:
- `apps/agentos/util/SpineBanner.mjs`, partial branch: `lead = note ? \`Activity feed partial — ${summary} · ${note.lead}\` : \`Activity feed partial — ${summary}\``, and `cause = [reason, note?.reason, note?.windowTitle]`.
- `apps/agentos/util/FleetAdmission.mjs`, partial guidance: `missing` holds the degraded slots (`PR lanes partial`), `remaining` holds the wired ones (`mailbox available`), `summary = [missing, remaining].join(' · ')`, and `head = partial — ${missing}`.

## The Fix

1. The partial lead names only the degraded sources (`missing`, e.g. `PR lanes partial`), without the `Activity feed partial —` prefix.
2. The full summary, healthy sources included, and any recovered outage move to the pill's title (`cause`), where the outage's reason and window already ride.
3. Specs in the `SpineBanner` spec pin the partial lead: it carries neither the pill's words, nor a `wired` source, nor a recovered outage, while the title still carries all three.

## Acceptance Criteria

- [ ] With PR lanes partial and the mailbox wired, after a recovered plane outage, the lead reads `PR lanes partial`, and the title carries `mailbox available` and the outage window.
- [ ] No lead in the partial branch starts with the pill's words.
- [ ] The specs pass, and the header IA's rule holds: one visible line carrying the reason only.

## Out of Scope

- Why the feed is partial after boot (#919).
- The wake chip, which has its own ticket.
- The other verdict branches (stale, booting), which stay unchanged.

## Related

#919, #424, and the header IA (#533 / #542).

Live latest-open + closed sweep: latest 20 open (58 open, red band; `#925`..`#819`) plus all-state searches (`spine banner lead long partial`, `activity feed cadence boot roster pass`) at 2026-11-12T02:12Z. The hits were the row and epic umbrellas #424, #516, #12 and #10, read by title; none owns this line.
A2A in-flight claim sweep: latest messages, all states, at 02:10Z; no claim.
MC sweep: `cockpit activity feed empty after FM restart; banner text too long; wake shows off and never success`, 6 results; no prior decision.
Own-assignment sweep: 3 open (#917, #516, #424), none on this surface.

unowned-rationale: Ready for a contributor. Two pure utilities with specs (`SpineBanner`, `FleetAdmission`); maintainers provide review.

Origin Session ID: c56bdafb-bd1d-4a67-b91d-3c37931e0abb
Retrieval Hint: "spine banner lead repeats the pill, lists wired sources and a recovered outage: feed partial overloaded cockpit bar"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T02:15:44Z @neo-opus-ada added the `bug` label
- 2026-11-12T02:15:44Z @neo-opus-ada added the `good first issue` label
- 2026-11-12T02:15:44Z @neo-opus-ada added the `ai` label
- 2026-11-12T02:15:45Z @neo-opus-ada added the `design` label

