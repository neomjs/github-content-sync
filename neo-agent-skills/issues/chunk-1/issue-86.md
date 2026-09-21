---
id: 86
title: 'The AC table is what the merge gate reads, and nothing checks it against the body it sits in'
state: OPEN
labels: []
assignees: []
createdAt: '2026-09-17T18:13:55Z'
updatedAt: '2026-09-17T18:13:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/86'
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
---
# The AC table is what the merge gate reads, and nothing checks it against the body it sits in

## Context

The `## AC Evidence` table is the row a merge gate and a reviewer act on. Nothing — substrate or CI — checks a row against the rest of the body it sits in, and `pull-request-workflow.md` §9 says so plainly:

> The shared job enforces **anchor presence** — that the heading opens a content line. It does not read the rows.

The Brain preflight reads them where it resolves, for AC *coverage*. Internal consistency is checked nowhere.

**Measured on my own output, 2026-09-17, twice in one session, both caught by @neo-opus-grace reading the table against the prose:**

| PR | the AC row claimed | the same body said, a few lines below |
|---|---|---|
| `neomjs/neo#18825` | AC-1 **"CI-covered by the arm"** | `## Test Evidence`: *Outside-CI*. The `e2e-engine` job log has **0** occurrences of the spec name and prints `NEO_AGENTOS_RUNTIME_ROOT is unset — 88 of 116 spec files are EXCLUDED`. |
| `neomjs/create-app#35` | AC-3 filed under **`## Post-Merge Validation`** | my own disposition comment called it *the un-draft gate* — i.e. blocking |

Both times the table was wrong and the prose right. Neither is a typo: the prose gets written while looking at the evidence, the table afterwards in a summarising pass that reaches for the conventional row — "CI-covered", "post-merge" — because that is what rows usually say.

Live latest-open sweep of this repository at 2026-09-17T18:05Z, plus a targeted search on AC table / evidence / body contradiction across all states: nothing equivalent. `#65` is about `check-pr-body` hanging in a non-TTY shell — its execution, not its content.

## The Problem

The two halves of a PR body are written in different modes and read in different orders, and the cheaper-to-read half is the one that is wrong.

1. **A "CI-covered" row tells a reviewer not to look for a manual receipt.** On `#18825` the arm cannot run in CI by construction. A reviewer trusting the row would have recorded CI as the witness for an arm CI never executes.
2. **A "post-merge" row is follow-up; a gate is blocking.** On `create-app#35` the difference is whether a PR can merge before the thing it depends on exists. That PR would have merged four scripts calling build files no published release contains.
3. **No reader re-derives the row.** Grace did, twice, in one day, on one author. That is a reviewer absorbing a defect the author should have caught, and it does not scale past a reviewer who happens to be that careful.

The failure is cheap to prevent and invisible to every existing gate: shape-lint passes, anchors are present, and the row is internally well-formed.

## The Architectural Reality

- `pull-request/references/pull-request-workflow.md` §9 — the body structure, the `## AC Evidence` template rows (`<CI-covered: owning spec reference>` / `<outside-CI: command + receipt>`), and the paragraph stating the job does not read rows.
- Same file §9, the six unconditional anchors — `## AC Evidence`, `## Test Evidence`, `## Post-Merge Validation`, `## Deltas`, `Evidence:`, `Authored by ` — all presence-only.
- `pr-review/` — the reviewer template carries an Evidence Audit, but nothing directs a reviewer to cross-read the AC table against the body's own sections. Both catches above came from one reviewer's habit, not from the template.
- `neomjs/neo`'s `pr-baseline.yml` — records that the shared `pr-body` job REPORTS and is not a required context, so a mechanical check is not available here anyway.

## The Fix

Author side, in §9, beside the template that produces the rows: read the AC table **last**, against the finished body, and check each row against the sections that contradict it — a `CI-covered` row against `## Test Evidence` and the job log, a `Post-Merge Validation` item against anything calling it a gate.

Reviewer side, in the `pr-review` template: one checkable item for cross-reading the table against the body, so catching it does not depend on which reviewer drew the seat.

Discipline-only, deliberately: the enforceable surface is a shared reusable workflow in another corpus, and the two instances here are semantic (does *this* row's claim survive *this* body) rather than pattern-matchable.

## Acceptance Criteria

- [ ] **AC-1** — §9 instructs that the AC table is written or re-read LAST, against the finished body, with the two measured contradiction shapes named as examples rather than described abstractly.
- [ ] **AC-2** — the rule states WHY: the table is what the merge gate and the next reader act on, so a wrong row outranks correct prose.
- [ ] **AC-3** — the `pr-review` template carries a checkable cross-read item, so the catch does not depend on the reviewer.
- [ ] **AC-4** — no claim of mechanical enforcement. The shared `pr-body` job does not read rows and this does not change that.
- [ ] **AC-5** — net loaded-bytes delta reported per the accretion rule; this should be a short addition to an existing section, not a new one.

## Out of Scope

- Any change to the shared `pr-body` workflow or its anchor set. Different corpus, different owner, and these contradictions are semantic rather than shape-detectable.
- The Brain preflight's AC-coverage check, which answers a different question (are all ACs present) than this one (does a row's claim survive its own body).
- Enrolling anything new in CI.

## Avoided Traps

- ⛔ **Do not add a seventh anchor or a new section.** The sections exist; the defect is that their contents disagree. Another heading adds bytes and catches nothing.
- ⛔ **Do not write it as "be careful".** The two shapes are specific and repeat: a coverage claim contradicted by the evidence section, and a gate filed as follow-up. Name them.
- ⛔ **Do not claim the shape-lint covers it.** It passed on both PRs. Substrate implying a gate that does not exist is the same defect class this ticket is about.

## Related

- `neomjs/neo#18825` and `neomjs/create-app#35` — the two measured instances, same author, same day
- `#85` — the other substrate gap surfaced by review this week, same pattern of a rule living only in memories

Origin Session ID: 2b78af80-54a8-4e14-afd8-85c6dfe41004

Retrieval Hint: "AC Evidence table contradicts Test Evidence section, CI-covered row on a spec CI never runs, post-merge item that is really a pre-merge gate"


## Timeline

- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90

