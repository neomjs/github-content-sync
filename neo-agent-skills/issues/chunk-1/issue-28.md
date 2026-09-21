---
id: 28
title: The PR-body anchor gate is satisfied by naming an anchor in prose
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-08-31T07:37:00Z'
updatedAt: '2026-08-31T12:00:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/28'
author: neo-opus-grace
commentsCount: 0
parentIssue: 14
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-31T12:00:24Z'
---
# The PR-body anchor gate is satisfied by naming an anchor in prose

## Context

Found while building the negative control for `neomjs/neo` PR #17917 (restoring `agent-pr-body-lint.yml`), and observed rather than reasoned: the documented mutation **failed to turn the check red**, twice, before I understood why.

The gate matches its six required anchors with `body.includes(anchor)` — a literal substring test against the whole body. That is not a heading test. Any occurrence anywhere satisfies it, including one inside a table cell, a code fence, or a sentence *about* the anchor.

The consequence runs the opposite direction from the known authoring gotcha. `neomjs/neo#14344` recorded the false-negative half — writing `## Evidence` as a heading does not satisfy the `Evidence:` anchor, because the substring differs. This ticket is the false-**positive** half, which nothing has recorded: a body that never carries the section at all still passes, provided it says the anchor's name somewhere.

Live evidence, all at unchanged head `9d2d23d12f` on `neomjs/neo` PR #17917:

| attempt | mutation | result |
|---|---|---|
| 1 | deleted the `## Deltas` **heading** | **green** — 3 surviving prose occurrences kept the anchor satisfied |
| 2 | removed **all** occurrences of `Authored by ` | [red](https://github.com/neomjs/neo/actions/runs/33366874780) — `missing required template anchors: ` + the anchor |
| 3 | restored, body edit only | [green](https://github.com/neomjs/neo/actions/runs/33366986918) |

Only the total-erasure mutation reds the gate. A heading-only deletion is invisible to it.

## The Problem

`neomjs/neo#11501` introduced the **invisible anchor layer** (`Authored by `, `## Deltas`) with an explicit purpose, still stated in the workflow's own header comment: to defeat Goodhart anchor-stuffing, where an agent that has read the CI failure text hallucinates a body containing exactly the anchors the failure names, and nothing else. The layer works by keeping two anchors *out* of the failure output, so a body reconstructed from the error message alone stays red.

That defends against **omission from an enumeration**. It does not defend against **naming in prose**, because invisible anchors are literal substrings too and are stuffable by exactly the same move. The layer raises the cost of the attack by two strings; it does not close it.

The reachable end state is a PR body that carries `Resolves #N` (regex-checked, genuinely structural) plus one paragraph mentioning `Evidence:`, `## AC Evidence`, `## Test Evidence`, `## Post-Merge Validation`, `Authored by ` and `## Deltas` — with none of those sections present — and it is green. The gate then certifies that the author knows the anchor names, which is the one thing that never needed certifying.

This is not hypothetical prose: PR #17917's own body discusses the anchor set, so every anchor occurs 2–4 times there. Every governance PR that documents the gate is structurally in this position.

## The Architectural Reality

`neomjs/neo` `.github/workflows/agent-pr-body-lint.yml` at `9d2d23d12f`:

- `:87–92` — `VISIBLE_PR_BODY_ANCHORS` = `Evidence:`, `## AC Evidence`, `## Test Evidence`, `## Post-Merge Validation`
- `:94–97` — `INVISIBLE_PR_BODY_ANCHORS` = `Authored by `, `## Deltas`
- `:99–100` — both filtered with `!body.includes(a)`; **both layers substring, no line anchoring, no multiline regex**
- the ticket-reference check is separate and *is* regex-based, which is why `Resolves #N` cannot be satisfied by prose

Three of the six anchors are markdown headings by intent (`## AC Evidence`, `## Test Evidence`, `## Post-Merge Validation`, plus `## Deltas`); two are inline forms (`Evidence:`, `Authored by `) that legitimately appear mid-line and must **not** be line-anchored. So the fix is not a uniform regex swap — the anchor set needs a per-anchor match **kind**, which it does not currently carry.

Ownership is this repository's, not the Engine's. Epic #14 makes `neo-agent-skills` the source of truth for non-product PR governance, and `neomjs/neo#17916` AC-7 already routes the anchor policy here by design; the Engine copy is a verbatim restoration whose scope is explicitly "restore the rules as they were".

## The Fix

1. Give the anchor set a **match kind** per entry rather than one flat string list — `heading` (line-anchored, `^## Name` with multiline matching) for the four `##` anchors, `substring` for `Evidence:` and `Authored by `. The list stops being `string[]` and becomes `{anchor, kind}[]`.
2. Keep the visible/invisible split unchanged. It is orthogonal to match kind and still does its (narrower, correctly-scoped) job of bounding the failure output.
3. Land it in the shared governance surface Epic #14 is building, so all five repositories move together and no consumer keeps a divergent copy.
4. Add a spec that **mutates both ways**: a body whose `## Test Evidence` heading is deleted but whose prose still names it must go **red** (this is the arm that fails today), and a body with the real heading must stay green. Without the first arm the change is untested against the defect it exists to fix.
5. Correct `pull-request-workflow.md` §9 to state the match kind per anchor. Today it lists anchors without saying how they are matched, which is why both the false-negative (`neomjs/neo#14344`) and this false-positive were discovered empirically rather than read.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| PR-body anchor set | Epic #14 (Skills owns non-product PR governance) | each anchor carries an explicit match kind: `heading` (line-anchored) or `substring` | none — an anchor with no declared kind is a lint error, never a silent substring default | `pull-request-workflow.md` §9 | spec arm: heading deleted + name still in prose ⇒ red |
| `## AC Evidence`, `## Test Evidence`, `## Post-Merge Validation`, `## Deltas` | `pull-request-workflow.md` §9 / §5 | matched line-anchored at start of line | none | §9 table | mutation control per anchor |
| `Evidence:`, `Authored by ` | `pull-request-workflow.md` §5 | remain substring — they are inline forms by design, and `neomjs/neo#14344` is the receipt that heading-shaping them is the *other* defect | none | §5 | existing green bodies stay green |
| Failure output | `neomjs/neo#11501` | unchanged: one diagnostic visible anchor, invisible anchors withheld | none | workflow header | existing behavior, not re-litigated |

## Decision Record impact

`none` — this completes Epic #14's already-selected Skills-SSOT shape and changes no runtime ADR. It corrects the implementation of `neomjs/neo#11501`'s stated intent rather than reversing its decision.

## Acceptance Criteria

- [ ] The shared anchor set expresses a match kind per anchor; no anchor is matched by an undeclared default.
- [ ] The four `##` anchors are line-anchored; `Evidence:` and `Authored by ` remain substring.
- [ ] A spec arm proves the defect is closed: a body that deletes a `##` heading while still naming it in prose goes **red**. The arm must be shown failing against the current implementation before the fix lands.
- [ ] A spec arm proves no regression: a conforming body — including one that *discusses* the anchor set, as governance PR bodies do — stays green.
- [ ] `pull-request-workflow.md` §9 states the match kind per anchor.
- [ ] Post-merge only: the first agent-authored PR after rollout reports the shared check under its stable name in every consumer repository.

## Out of Scope

- Renegotiating **which** anchors are required. The set is unchanged; only how it is matched changes.
- The visible/invisible split and the failure-output bounding from `neomjs/neo#11501`.
- Re-homing the `agent-preflight` AC-Evidence **content** check (#24) or the stacked-PR guard.
- Making the shared job a required status context — `neomjs/neo#17171` owns that.
- Any content-quality judgement of the sections. A real `## Test Evidence` heading over empty prose still passes; this gate is a floor, and #22 is where quality lives.

## Avoided Traps

- **Uniform line-anchoring of all six.** It would red every currently-green body: `Evidence:` and `Authored by ` are inline by mandate (§5), and forcing them to line-start re-creates `neomjs/neo#14344` from the other side.
- **Fixing it in the Engine's restored copy.** That is the duplication Epic #14 exists to end, and `neomjs/neo#17916` AC-7 already routes the policy here. A restoration PR is the wrong place to change the rules.
- **Treating this as a `neomjs/neo#11501` reversal.** The invisible layer is sound for its stated job; its premise is simply narrower than the anti-stuffing claim made for it.
- **Shipping without the failing arm.** A spec written after the fix passes trivially; the arm that must be seen red first is the heading-deleted-but-named-in-prose case.

## Related

- #14 — Epic: unify PR governance across Neo repositories (parent)
- #22 — reusable agent PR-**review** policy; sibling surface, different body
- #24 — the mandated PR preflight is not runnable outside the Brain
- `neomjs/neo#17916` / PR `#17917` — the Engine-side restoration where this was found; AC-7 routes anchor policy here
- `neomjs/neo#11501` — introduced the visible+invisible anchor layers
- `neomjs/neo#14344` — the false-negative half: heading shape does not satisfy an inline anchor
- `neomjs/neo#17431` — the live-body-fetch property of the same gate

Live latest-open sweep: checked all 16 open `neo-agent-skills` issues at 2026-08-31T07:35:42Z plus the A2A claim window (last 60 min, 15 messages) — no equivalent found; nearest are #22 (review body, not PR body) and #24 (preflight runnability), both distinct. Structure-map gate: N/A — `ai:structure-map` is a Brain-owned script with no entry in the Engine `package.json`, and this ticket introduces no `.mjs` placement decision.

Retrieval Hint: `query_raw_memories("PR-body anchor substring match heading line-anchored Goodhart anchor stuffing")`; run receipts 33366874780 / 33366986918 on `neomjs/neo` PR #17917.

Origin Session ID: 77316fa6-874f-4773-9505-7a3ea378f040

Authored by Grace (Anthropic Claude Opus 5, Claude Code).

## Timeline

- 2026-08-31T07:37:01Z @neo-opus-grace added the `bug` label
- 2026-08-31T07:37:02Z @neo-opus-grace added the `ai` label
- 2026-08-31T07:37:02Z @neo-opus-grace added the `agent-os` label
- 2026-08-31T07:37:12Z @neo-opus-grace added parent issue #14
- 2026-08-31T08:07:23Z @neo-opus-grace cross-referenced by #29
- 2026-08-31T08:17:30Z @neo-opus-grace cross-referenced by PR #30
- 2026-08-31T08:21:07Z @neo-opus-grace referenced in commit `a8fc793` - "fix(ci): per-anchor match kinds, and section 9 states them (#28)

Conforming to #28's design rather than my own. The first commit line-anchored all
six anchors with one rule; #28 AC-1/AC-2 specify a declared match kind PER anchor,
with the four `##` headings line-anchored and `Evidence:` / `Authored by `
remaining substring.

That distinction is right and my uniform rule was not. Those two are line
PREFIXES, not headings, and tightening them re-opens the false-negative half
already recorded as neomjs/neo#14344 -- a body carrying `- **Evidence:** ...`
would start failing for formatting. The ticket had made this call before I
arrived at the file; conforming, not overriding.

No default. An anchor with no declared kind throws at the declaration rather than
silently picking one, because an unnamed default is precisely how the substring
rule survived unexamined -- nobody chose it, so nobody reviewed it.

Two arms added for #28's remaining ACs: an undeclared kind throws, and a
governance body that DISCUSSES the whole anchor set while carrying the sections
stays green. That second one keeps the fix from overshooting; bodies in this
repository routinely name every anchor in prose, and refusing those would trade
one false verdict for another.

Section 9 now states the match kind per anchor in a table, names the measured
receipt, and records why the two prefixes stay substring -- the promise a seat
reads matches what runs. Its stale AC-Evidence CONTENT claim is re-scoped to the
Brain preflight (#24) rather than left asserting this job does it.

JSDoc and the CLI failure message corrected in the same pass: both still
described uniform line-anchoring after the code stopped doing that."
- 2026-08-31T08:30:29Z @neo-opus-grace referenced in commit `7a87fe1` - "docs(pull-request): compress section 9's match-kind contract into the corpus budget (#28)

The first statement of the per-anchor contract grew the corpus 978 bytes against a
250-byte cap. The cap is not a hard ceiling — it offers [skill-growth-justified] —
but the budget existed to force the question, and the answer was yes: the same
contract, both receipts and the re-scoping fit in 248 bytes once the table markup
became two bullets and the prose stopped restating what the receipts already prove.

Nothing load-bearing was dropped: the six anchors, the per-anchor match kinds, the
#17917 receipt that stops the anchors being re-loosened, the #14344 rationale that
stops the two prefixes being tightened, and the #24 re-scoping all survive."
- 2026-08-31T08:53:20Z @neo-opus-grace referenced in commit `9cd57a3` - "fix(ci): anchor the workflow-contract path assertions so a suffixed root reds (#25)

RA-2 on PR #26, from @neo-gpt. The workspace pin was present in the workflow and
asserted in the contract suite — as an UNANCHORED substring, so
`working-directory: ${{ github.workspace }}/docs` satisfied the assertion that
exists to forbid exactly that. A wrong root is the silent failure the pin was
added for: the guard resolves its targets from cwd, so outside the checkout every
target is ENOENT, the negative-space contract reports N/A, and the job greens
having measured nothing.

Three assertions shared the defect, not one. Every path-valued assertion in this
suite was substring-matched, so a suffix produced a different root that passed:
`working-directory`, the substrate `SKILLS_ROOT`, the archaeology `SKILLS_ROOT`,
and the substrate `ref:`. All are now anchored to end-of-line.

Non-vacuity, which is the only thing that makes this a fix rather than a claim:
the three new mutations append a suffix rather than replacing the value. A
wholesale replacement reds against an unanchored pattern too, so it cannot tell
an anchored assertion from an unanchored one — it would have passed before this
change and after it. Reverting ONLY the workspace anchor fails the suite at
exit 1 on exactly `substrate workspace pin — suffixed root` and nothing else;
restored, the suite is green at 32 mutations.

Same class as the #28 defect fixed on PR #30: substring presence cannot express
a positional rule. I shipped it in one guard while fixing it in another."
- 2026-08-31T08:54:18Z @neo-opus-grace cross-referenced by PR #26
- 2026-08-31T09:58:14Z @neo-opus-grace referenced in commit `fdcce10` - "fix(ci): code is not content, and the close target is a standalone line (#28)

Five required actions from @neo-gpt-emmy. Three were P1 and all three reproduced
before anything was changed.

RA-1 — the heading rule stopped at prose. A fenced or four-space-indented
`## Test Evidence` satisfied the anchor, so the fix for "a sentence naming the
anchor is not the section" left "a code block naming it" wide open: the same
defect one level in. My own "indentation is formatting, not evasion" tolerance is
what opened the indented-code door. `markdownContentLines()` now removes fenced
(``` and ~~~, closed or unterminated) and indented blocks, and both match kinds
read content rather than the raw body. Headings keep Markdown's ≤3-space
indentation, asserted so the fix cannot overshoot into a false negative.

RA-3 — the close target was a substring search where the skill promises a
standalone line. Mid-prose, fenced, table-cell, indented and colon forms all
satisfied a rule §9.1 forbids, and the comma form now earns its own message: it
is a close target the author wrote deliberately, so reporting it as "missing"
sends them hunting the wrong bug.

RA-2 — the contract ran nowhere. `test-check-pr-body.mjs` was in neither
`package.json#test` nor the corpus workflow, so the PR's only green check never
executed the suite proving the #28 fix. Wired into both — the workflow runs
suites by name, so the package script alone would still not have selected it —
and the pack assertions now prove the CLI ships and the test does not.

RA-4 — both anchor arrays were documented `{String[]}` while carrying
`{anchor, match}` records, contradicting the very declaration AC-1 exists to make
explicit.

Non-vacuity: pointing the classifier back at raw lines reds the fenced arm by
name at exit 1. The first attempt at that control broke the file syntactically
and went red for the wrong reason, which is not evidence — redone.

§9 states the content rule within the 250-byte corpus budget (245)."
- 2026-08-31T10:02:36Z @neo-opus-grace referenced in commit `3c255c8` - "feat(ci): the PR-body anchor validator as a portable guard (#29)

First slice of the relocation: the decision half only. The reusable-workflow job
follows once #26 lands, because that PR owns reusable-pr-baseline.yml and
stacking the two would conflict on one file.

`pull-request-workflow.md` §9 tells every agent in every consuming repository
that `agent-pr-body-lint.yml` enforces six anchors as unconditional. That
workflow was deleted from the Engine on 2026-08-26 and re-homed nowhere, so the
promise is currently false everywhere. It has a measured victim: neomjs/neo PR
review rounds and green CI.

The port closes #28 by construction. The old gate matched `body.includes(anchor)`
against the whole body, and every anchor also appears in bodies that DISCUSS the
anchors -- a table cell naming `## Test Evidence`, or a sentence explaining why a
section was omitted, satisfied it while the section was absent. Measured on the
restoration PR: deleting the `## Deltas` heading left three surviving prose
occurrences and the check stayed green.

Anchors are now matched as LINES. That covers both shapes without a second rule:
`## AC Evidence` is a heading and `Evidence:` / `Authored by ` are line-initial
prefixes, so all three open a line, while a mid-sentence mention, a table row
(which opens with `|`), or a fenced snippet does not. Leading whitespace is
tolerated because indentation is formatting, not evasion.

Pure and transport-free: it receives a body and returns findings, never fetching
a pull request or posting a comment. That is the property the inline home could
not have -- 253 lines of `github-script` inside YAML cannot carry a test
contract, which is why the substring defect survived there.

The visible/invisible split is preserved and so is its reason: the failure
message names at most ONE anchor and never an invisible one, because a message
enumerating the set is a template an agent can satisfy without writing the
sections. A spec arm asserts the invisible anchors never reach the output.

Mutation-verified: restoring `body.includes` reds the prose-mention arm for every
anchor, and only that arm."
- 2026-08-31T10:02:36Z @neo-opus-grace referenced in commit `6dcbc6f` - "fix(ci): per-anchor match kinds, and section 9 states them (#28)

Conforming to #28's design rather than my own. The first commit line-anchored all
six anchors with one rule; #28 AC-1/AC-2 specify a declared match kind PER anchor,
with the four `##` headings line-anchored and `Evidence:` / `Authored by `
remaining substring.

That distinction is right and my uniform rule was not. Those two are line
PREFIXES, not headings, and tightening them re-opens the false-negative half
already recorded as neomjs/neo#14344 -- a body carrying `- **Evidence:** ...`
would start failing for formatting. The ticket had made this call before I
arrived at the file; conforming, not overriding.

No default. An anchor with no declared kind throws at the declaration rather than
silently picking one, because an unnamed default is precisely how the substring
rule survived unexamined -- nobody chose it, so nobody reviewed it.

Two arms added for #28's remaining ACs: an undeclared kind throws, and a
governance body that DISCUSSES the whole anchor set while carrying the sections
stays green. That second one keeps the fix from overshooting; bodies in this
repository routinely name every anchor in prose, and refusing those would trade
one false verdict for another.

Section 9 now states the match kind per anchor in a table, names the measured
receipt, and records why the two prefixes stay substring -- the promise a seat
reads matches what runs. Its stale AC-Evidence CONTENT claim is re-scoped to the
Brain preflight (#24) rather than left asserting this job does it.

JSDoc and the CLI failure message corrected in the same pass: both still
described uniform line-anchoring after the code stopped doing that."
- 2026-08-31T10:02:36Z @neo-opus-grace referenced in commit `99831b2` - "docs(pull-request): compress section 9's match-kind contract into the corpus budget (#28)

The first statement of the per-anchor contract grew the corpus 978 bytes against a
250-byte cap. The cap is not a hard ceiling — it offers [skill-growth-justified] —
but the budget existed to force the question, and the answer was yes: the same
contract, both receipts and the re-scoping fit in 248 bytes once the table markup
became two bullets and the prose stopped restating what the receipts already prove.

Nothing load-bearing was dropped: the six anchors, the per-anchor match kinds, the
#17917 receipt that stops the anchors being re-loosened, the #14344 rationale that
stops the two prefixes being tightened, and the #24 re-scoping all survive."
- 2026-08-31T10:02:36Z @neo-opus-grace referenced in commit `94ae70a` - "fix(ci): code is not content, and the close target is a standalone line (#28)

Five required actions from @neo-gpt-emmy. Three were P1 and all three reproduced
before anything was changed.

RA-1 — the heading rule stopped at prose. A fenced or four-space-indented
`## Test Evidence` satisfied the anchor, so the fix for "a sentence naming the
anchor is not the section" left "a code block naming it" wide open: the same
defect one level in. My own "indentation is formatting, not evasion" tolerance is
what opened the indented-code door. `markdownContentLines()` now removes fenced
(``` and ~~~, closed or unterminated) and indented blocks, and both match kinds
read content rather than the raw body. Headings keep Markdown's ≤3-space
indentation, asserted so the fix cannot overshoot into a false negative.

RA-3 — the close target was a substring search where the skill promises a
standalone line. Mid-prose, fenced, table-cell, indented and colon forms all
satisfied a rule §9.1 forbids, and the comma form now earns its own message: it
is a close target the author wrote deliberately, so reporting it as "missing"
sends them hunting the wrong bug.

RA-2 — the contract ran nowhere. `test-check-pr-body.mjs` was in neither
`package.json#test` nor the corpus workflow, so the PR's only green check never
executed the suite proving the #28 fix. Wired into both — the workflow runs
suites by name, so the package script alone would still not have selected it —
and the pack assertions now prove the CLI ships and the test does not.

RA-4 — both anchor arrays were documented `{String[]}` while carrying
`{anchor, match}` records, contradicting the very declaration AC-1 exists to make
explicit.

Non-vacuity: pointing the classifier back at raw lines reds the fenced arm by
name at exit 1. The first attempt at that control broke the file syntactically
and went red for the wrong reason, which is not evidence — redone.

§9 states the content rule within the 250-byte corpus budget (245)."
- 2026-08-31T12:00:24Z @tobiu referenced in commit `c9fa4e8` - "Merge pull request #30 from neomjs/feat/pr-body-lint-shared-29

fix(ci): the PR-body anchor gate refuses a prose mention (#28)"
- 2026-08-31T12:00:24Z @tobiu closed this issue
- 2026-08-31T13:29:56Z @neo-opus-grace cross-referenced by PR #31

