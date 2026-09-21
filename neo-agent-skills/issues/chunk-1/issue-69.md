---
id: 69
title: An all-numeric CSS color fails the archaeology guard without a marker
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-15T02:07:03Z'
updatedAt: '2026-09-15T12:59:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/69'
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
closedAt: '2026-09-15T12:59:43Z'
---
# An all-numeric CSS color fails the archaeology guard without a marker

This ticket follows up one acceptance criterion of #18, and it is the published half of neomjs/neo#16553.

## Context

#18 required that "non-tracking numeric forms such as colors remain valid controls". A color with a letter in it passes: `#1234ff` reports nothing. An all-numeric color fails. Measured with `findArchaeology` at `dev@73017ef`:

| comment | reports |
|---|---|
| `// @member {String} backgroundColor_='#000000'` | `tracking-reference` |
| `// borderColor="#111111"` | `tracking-reference` |
| `` // fillStyle=`#123456` `` | `tracking-reference` |
| `// CSS color #123456` | `tracking-reference` |
| `// defaults to #000000` | `tracking-reference` |

So a consumer has two choices for such a color: annotate it with `[not-ticket-ref: css-color]`, or see its baseline job fail. neomjs/neo carries two of these annotations, at `src/component/Gallery.mjs:33` and `src/component/Helix.mjs:29`. neomjs/neo#16553 cannot remove them until this guard changes, is released, and is bumped there.

## The Problem

`findArchaeology` flags every `NUMERIC_REF_RE` match as a `tracking-reference`. The guard already recognizes color syntax: `CSS_COLOR_CONTEXT_RE` matches `color:`, `backgroundColor_=`, `fillStyle=`, `CSS color` and similar forms. That pattern only decides whether a marker is valid. It never decides whether the number is a ref.

## The Architectural Reality

- All of the logic is in `scripts/check-ticket-archaeology.mjs`: `findArchaeology`, `escapedColorOffsets`, `CSS_COLOR_CONTEXT_RE` and `NUMERIC_REF_RE`.
- Consumers run the guard through the `Source comment archaeology` job in `reusable-pr-baseline.yml`. That job installs a released version, so a consumer gets a change only after a release and a bump.

## The Fix

- A 3-, 4-, 6- or 8-digit number directly after color syntax is a color, with or without the marker.
- A number with a leading zero is never a ticket.
- Every other bare number still reports.
- The marker is still accepted on a color and still reports anywhere else, so consumers that carry it keep passing until they remove it.
- The README states the rule.

## Acceptance Criteria

- [ ] Without a marker, the five comments above report nothing.
- [ ] `see #16538`, `'see #16538'`, `#9473`, `tracked in #123456` and `issue='#9473'` still report.
- [ ] The marker controls still report: a marker without color syntax, a marker beside a real ref, and an unused marker.
- [ ] A numeric HTML entity reports nothing: `&#39;` and `&#8212;` in a comment are codepoints. Controls: a real ref beside an entity still reports, the hex form `&#x27;` is unaffected, and a bare `#39` stays a ticket.
- [ ] Post-merge: a release carries the change, so neomjs/neo#16553 can bump the dependency and remove its two markers.

## Out of Scope

- The engine's own guard and its two markers. Those belong to neomjs/neo#16553, after a release.
- Whether a deliberate ticket ref may stay in a durable comment (the legacy `ticket-ref-ok`). That is a policy question for both repositories, recorded on neomjs/neo#16553.

## Avoided Traps

neomjs/neo#16553 records the attempts that failed. Two of them limit this fix:

- Exempting a quoted or value-position match silences a quoted real ref such as `'see #16538'`.
- Exempting color-length numbers anywhere silences `#9473`, and six-digit tickets are coming. Here the length counts only after color syntax, so `tracked in #123456` still reports.

## Related

#18 (the guard) · neomjs/neo#16553 (the engine half) · neomjs/neo#18433 (the typed marker) · PR #68

Decision Record impact: none.
Structure map: N/A. An existing guard changes in place, and no file or directory is added.
Live latest-open sweep: checked the latest 20 open issues here at 2026-09-15T02:05Z, plus issues in all states matching "archaeology", "color" and "marker". No equivalent exists. #18 is the closed origin.
A2A in-flight claim sweep: checked the latest 30 messages in all read states and the latest 30 lane claims. The only claim on this guard is mine, from 2026-09-15T01:52Z.
Memory Core rationale sweep: no prior decision on color context. neomjs/neo#18433 made the colors' marker truthful and left the question of whether a color needs one to neomjs/neo#16553.
Own-assignment sweep: my one open assignment here, #64, is unrelated.

Retrieval Hint: "archaeology guard color context leading zero"
Origin Session ID: b9eefdf5-60e0-4156-860c-8cb4a628b8db



## Timeline

- 2026-09-15T02:11:30Z @neo-opus-ada cross-referenced by PR #68
- 2026-09-15T02:11:34Z @neo-opus-ada cross-referenced by #16553

