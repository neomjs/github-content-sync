---
id: 83
title: 'The typed not-ticket-ref escape works only for a bare #N — every named form is inescapable'
state: CLOSED
labels: []
assignees:
  - neo-opus-vega
createdAt: '2026-09-16T14:07:55Z'
updatedAt: '2026-09-16T19:35:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/83'
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
closedAt: '2026-09-16T19:35:26Z'
---
# The typed not-ticket-ref escape works only for a bare #N — every named form is inescapable

## Context

`check-ticket-archaeology`'s failure message prescribes one remedy for a deliberate reference:

> put a typed marker directly after the ref: `"#1234 [not-ticket-ref: <reason>]"`

It works for exactly one shape. Measured against `findArchaeology()` at `dev` `3503c35`, four comments each carrying the same valid marker:

| comment | escape honoured? | kinds reported |
|---|---|---|
| `#1234 [not-ticket-ref: deliberate, the contract]` | ✅ yes | — no finding |
| `Epic #1234 [not-ticket-ref: deliberate, the contract]` | ❌ **no** | `tracking-reference` |
| `Discussion #1234 [not-ticket-ref: deliberate, the contract]` | ❌ **no** | `tracking-reference` |
| `ADR 0004 [not-ticket-ref: deliberate, the contract]` | ❌ **no** | `tracking-reference`, **`invalid-escape`** |

So the escape excuses a **bare** `#N` and is powerless against all eight named forms the same message enumerates — `issue`, `ticket`, `bug`, `PR`, `pull request`, `Epic`, `Discussion`, `ADR`. Naming the ref you are citing removes your only way to declare it deliberate.

The ADR row is the sharpest: the marker is not ignored, it is counted as a **defect**. `invalid-escape` fires on `markers.length !== escaped.size + refs.size`, so following the message's own instruction produces *two* findings where there was one.

Filed from `neomjs/neo#18794`, which recorded the ADR half. That body scoped it to ADRs and was wrong; the correction and this measurement are on its thread.

## The Problem

Two mechanisms built on different models, which never met.

The escapes are **offset-based**. `escapedRefOffsets()` collects the index of each `#N` token carrying a marker, and the bare-numeric loop at the end of `findArchaeology()` skips those offsets. That loop is the only consumer.

`NAMED_TRACKING_PATTERNS.some(pattern => pattern.test(comment))` is a **whole-comment boolean**, evaluated before that loop and independent of it. A boolean has no token position, so there is nothing an offset could excuse. The named branch cannot honour an escape even in principle.

This is not "widen `REF_ESCAPE_RE` to accept an `ADR NNNN` token". That change would leave `Epic #1234` exactly as inescapable, because the named pattern fires on the phrase regardless of what the escape says about the token.

## The Architectural Reality

- `scripts/check-ticket-archaeology.mjs` — `NAMED_TRACKING_PATTERNS` (two regexes, one for the seven `<word> #N` forms, one for `ADR NNNN`), `REF_ESCAPE_RE` (requires `#(\d+)`), `escapedRefOffsets()`, and the `invalid-escape` count in `findArchaeology()`.
- The engine keeps a mirrored copy in `neomjs/neo:buildScripts/util/check-ticket-archaeology.mjs`, whose comment states the reason: *"Both patterns mirror the published guard rather than paraphrasing it, because the ONLY thing this change buys is the two agreeing."* CI in `neomjs/neo` runs the **published** binary, so a local-only widening there passes the hook and fails CI. **This repository is where the fix starts.**
- `scripts/test-ticket-archaeology.mjs` is `node:assert`-based and already mutation-sensitive, so the arms belong there.

## The Fix

Make the named branch token-scoped and escape-aware, the same shape the numeric loop already has:

1. Named patterns match **per token with an index** (`matchAll`, global flags) instead of testing the whole comment.
2. A named match whose token carries a valid typed marker is skipped, exactly as an escaped `#N` is.
3. `REF_ESCAPE_RE` accepts a named token, including the `ADR NNNN` form that has no `#`.
4. `invalid-escape` counts a marker bound to a named token as valid, so a correct escape can no longer manufacture a second finding.

The reason requirement does not move: a marker with a blank reason stays a defect, because that is what made the legacy `ticket-ref-ok` marker worthless.

## Acceptance Criteria

- [ ] AC-1: each of `Epic #N`, `Discussion #N`, `issue #N`, `PR #N` and `ADR NNNN` carrying a valid typed marker produces **no** finding.
- [ ] AC-2: red control — each of those **without** a marker still produces `tracking-reference`. A fix that made named refs pass unconditionally would delete the detection rather than complete the escape.
- [ ] AC-3: a marker with a blank or missing reason still produces `invalid-escape`, on named and bare forms alike.
- [ ] AC-4: an escape stays bound to **its own token** — a comment carrying an escaped ref AND an unescaped one still reports the unescaped one. The marker must not become a line-wide bypass.
- [ ] AC-5: `npm test` green, and the engine-side mirror is updated in a companion `neomjs/neo` PR so the two agree line for line.
- [ ] AC-6: package version bumped, per README line 33.

## Out of Scope

- Whether ADR or Epic refs should be detected at all — they rot like any other ref, and that is settled.
- The CSS-colour escape and its context rule.
- The review-archaeology patterns, which have no escape story and are not claimed to.

## Avoided Traps

- ⛔ **Do not widen `REF_ESCAPE_RE` alone.** It is the obvious one-line change and it fixes only the ADR row; `Epic #1234` stays inescapable because the named branch never consults the escape.
- ⛔ **Do not make the escape line-scoped.** The existing design binds a marker to its token deliberately — a line may carry an annotated ref and a genuine one, and only the first is excused. AC-4 pins it.
- ⛔ **Do not fix the engine copy first.** CI there runs this package's binary; a local widening passes the hook and fails CI, which is a divergence already recorded on `neomjs/neo#18794`.
- ⛔ **Do not treat "zero findings in the corpus today" as low value.** The cost is paid by authors at the keyboard: the guard refuses the ref, the author follows the instruction, and it refuses again with an extra finding. Both engine authors who hit this rephrased rather than escaped, which is exactly why the corpus stays clean and the defect stays invisible.

## Related

`neomjs/neo#18794` (origin, with the corrected scope on its thread) · `neomjs/neo#18793` and `neomjs/neo#18806` (the two PRs it blocked) · README line 33 (version bump)

Origin Session ID: 5bf0b816-b919-40fc-9c18-fee751fd1635

Retrieval Hint: "ticket archaeology named tracking pattern whole-comment test ignores typed escape offsets Epic Discussion ADR inescapable"


## Timeline

- 2026-09-16T14:12:20Z @neo-opus-vega cross-referenced by PR #84
- 2026-09-18T10:36:55Z @neo-opus-vega cross-referenced by PR #89

