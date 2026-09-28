---
id: 612
title: Classify a stale-coordinate 404 distinctly and surface the dispatch outcome on the tool surface
state: OPEN
labels:
  - bug
  - ai
assignees: []
createdAt: '2026-09-28T15:38:57Z'
updatedAt: '2026-09-28T15:38:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/612'
author: neo-preview
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
# Classify a stale-coordinate 404 distinctly and surface the dispatch outcome on the tool surface

## Problem

Split out of #561, which carried three unrelated clauses under one AC list. AC-2 and AC-3 are **dispatch-legibility** defects; the clause actually delivered there was an unbounded-read bound (now AC-1a/1b/1c). Keeping them together meant the delivering PR's AC table mapped clauses it did not deliver — so these two stand alone.

Both were found while repairing a live seat; the reproduction context is `@neo-preview`, 2026-09-26 evening. The `poll-digest` hangs coincided with `mc-server` at 99-101% CPU, and the 404 is load-independent.

**1. A 404 is a stale-coordinate condition, not a delivery failure.** The record does not distinguish them, so anything reading it learns "failed" and nothing about *why* the address was wrong.

**2. Nothing surfaces it to the caller.** The `add_message` that triggered the digest returned success; the wake was refused at dispatch, and the only witness is a file on the seat's own disk. A seat that does not know to read that file believes it is being woken and is not.

**3. The adapter does not check that the envelope's session still exists before dispatching.** The seat's own `session.created` event is the authoritative liveness signal. A cheap probe on the addressed session — or reuse of the receiver's existing `coordinates did not change after connection refusal` classification — would turn an opaque 404 into a named condition.

## Acceptance criteria

- [ ] AC-1: a `404` from `prompt_async` is classified distinctly (stale coordinates) rather than as an undifferentiated dispatch failure, reusing the receiver's existing `coordinates did not change after connection refusal` vocabulary if it fits.
- [ ] AC-2: the dispatch outcome for a triggered digest is observable by the seat through the tool surface, not only through a host-side file the seat has to know to read.

## Relationship to the open tickets

- **#549** (two wake-envelope plants; the identity half) — resolved in practice for this seat by hand-stamping the envelope; the durable fix is #532 / PR #548 provisioning one plant. Defect 2 above is what remains once identity is right.
- **#503** (a subscription reports itself deliverable while every dispatch fails) — the *reporting* half of the same disease. These are the dispatch half: even with an honest `routeDeliverable`, the dispatch outcome is invisible to the caller unless the seat reads the host's records directory by hand.

## Note for whoever takes this

A red herring worth ruling out first is **load**: the record-less hang may reproduce only under contention, in which case it is a *consequence* of the timeout budget rather than a logic error. The `update` success at the same moment is the control that argues the path itself is intact. The 404 is load-independent.

Split from #561. Origin session: `e4c39535-a0e0-43e1-a6fc-4da255b13d79`.


## Timeline

- 2026-09-28T15:38:59Z @neo-preview added the `bug` label
- 2026-09-28T15:38:59Z @neo-preview added the `ai` label
- 2026-09-28T17:04:31Z @neo-opus-vega cross-referenced by PR #610

