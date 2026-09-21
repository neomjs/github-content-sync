---
id: 71
title: 'The typed escape binds only to colours, so a deliberate reference has no state both guards accept'
state: CLOSED
labels:
  - bug
assignees:
  - neo-opus-grace
createdAt: '2026-09-15T13:59:23Z'
updatedAt: '2026-09-15T14:39:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/71'
author: neo-opus-grace
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
closedAt: '2026-09-15T14:39:27Z'
---
# The typed escape binds only to colours, so a deliberate reference has no state both guards accept

## Context

`0.1.5` taught `check-ticket-archaeology` three narrowing rules and rejected the legacy `ticket-ref-ok` marker in favour of the typed `[not-ticket-ref: …]` form. Consumers moved: `neomjs/neo` at [#18734](https://github.com/neomjs/neo/pull/18734), `neomjs/neo-agent-institution` at [#139](https://github.com/neomjs/neo-agent-institution/pull/139).

That surfaced a defect the colour work could not see, now owned Engine-side by [`neomjs/neo#18740`](https://github.com/neomjs/neo/issues/18740): **there is no accepted state for a deliberate ticket reference in a durable comment.**

Live sweeps 2026-09-15T14:1xZ: latest 8 open read; full-text over open and closed for `typed escape reference marker ticket-ref-ok` returns only `#14`, which is the governance umbrella, not this. No competing A2A claim.

## The Problem

Measured, one comment, four candidate repairs, both implementations run directly:

| repair | engine `buildScripts/util/check-ticket-archaeology.mjs` | this package at `0.1.5` |
|---|---|---|
| keep `ticket-ref-ok` | green | **RED** |
| migrate to `[not-ticket-ref: implementing ticket]` | **RED** | **RED** |
| remove the marker, keep `@see #11133` | **RED** | **RED** |
| delete the reference itself | green | green |

**The obvious migration is the worst available option** — it reds both where the current state reds one. That is the trap worth recording: an author following the deprecation notice makes their PR less mergeable.

The cause is structural, not a regex slip. `escapedColorOffsets()` admits a marker only when `CSS_COLOR_ESCAPE_RE` matches **and** the 48-char lookbehind is colour syntax. `findArchaeology()` then requires `markers.length === escaped.size`, so **every typed marker that does not bind to an escaped colour is itself `invalid-escape`**. The vocabulary has one word in it.

The consequence, at the consumer: ten files in `neomjs/neo` red on touch, and the only both-green repair deletes lines like `@see #11133 — the ticket this script implements` — provenance the comment exists to carry, in a file whose purpose is implementing `#11133`.

## The Architectural Reality

- `ANY_TYPED_ESCAPE_RE` (`\[not-ticket-ref:[^\]]*\]`) already accepts free-text reasons. The restriction is not in the marker grammar; it is in the **binding**.
- `escapedColorOffsets()` is offset-scoped by design, so a marker excuses one token rather than a whole row. That precision is right and should be preserved, not traded for a row-level escape hatch.
- `colorContextOffsets()` and `htmlEntityOffsets()` are *inference* — the guard decides. A typed marker is *declaration* — the author decides, visibly and greppably. Extending declaration is the smaller change.

## The Fix

Let a typed `[not-ticket-ref: <reason>]` bind to a numeric reference token the same way the `css-color` form binds to a colour:

- a new offset set for `#\d+` tokens directly carrying a typed marker whose reason is not `css-color`;
- those offsets skipped in the `NUMERIC_REF_RE` loop, so the token stops raising `tracking-reference`;
- the marker-count equality widened to count colour escapes **and** reference escapes, so a marker binding to neither is still `invalid-escape`.

The reason text stays free-form and is not validated. The guard's job is to require a deliberate, visible, greppable justification — not to grade its prose. `LEGACY_ESCAPE_RE` keeps rejecting `ticket-ref-ok`, so this widens the accepted set without re-admitting the untyped form.

## Acceptance Criteria

- [ ] **AC-1** — `#11133 [not-ticket-ref: implementing ticket]` raises neither `tracking-reference` nor `invalid-escape`.
- [ ] **AC-2** — a bare `#11133` still raises `tracking-reference`, and `#11133 ticket-ref-ok` still raises `invalid-escape`. The change adds one accepted form and removes none of the rejections.
- [ ] **AC-3** — a typed marker binding to no token at all (no colour, no reference) is still `invalid-escape`; the count equality is widened, not dropped.
- [ ] **AC-4** — the colour, entity and leading-zero rules are unchanged, asserted by the existing spec arms plus the eight-probe differential against the engine implementation still agreeing 8/8.
- [ ] **AC-5** — red-first: each new arm fails against the current implementation before the change, and the mutation is on the **binding**, not only on the old tree — reverting only the offset-set wiring must red the new arms.

## Out of Scope

- `NAMED_TRACKING_PATTERNS` prose references with no `#`, such as `pull request 11982`. Offset binding cannot reach them; one of the ten consumer files carries this shape and it needs its own read.
- Rewriting the consumer comments. That is `neomjs/neo#18740`, which gates on this.
- The engine's second implementation taking the same rule — its own step, in that ticket.
- Validating reason vocabulary, or a registry of accepted reasons.

## Avoided Traps

- **Do not make the marker excuse the whole comment row.** It would silence genuine archaeology sharing a line with one justified reference; the existing colour binding is offset-scoped precisely to avoid that.
- **Do not drop the `markers.length === escaped.size` check** to make the new form pass. That equality is what stops a decorative marker from being a free pass; widen what counts, keep the equality.
- **Do not re-admit `ticket-ref-ok`.** The legacy form carries no binding at all, which is why it was rejected.

## Decision Record impact

This changes the escape contract both implementations honour. If a Decision Record documents the `0.1.5` rule set, it names one more accepted form.

## Related

[`neomjs/neo#18740`](https://github.com/neomjs/neo/issues/18740) (the consumer deadlock, blocked on this) · `neomjs/neo#16553` (the parent rule; this is its deferred policy row) · [#68](https://github.com/neomjs/neo-agent-skills/pull/68) / `#69` (the 0.1.5 rules) · `#14`

Origin Session ID: 148d12cc-9777-46e4-bfb9-a422479153a5

Retrieval Hint: `query_raw_memories("typed escape binds only to colours reference deadlock ticket-ref-ok")`

## Timeline

- 2026-09-15T14:04:28Z @neo-opus-grace cross-referenced by PR #72

