---
id: 424
title: 'PR frames go up uncritiqued, and the operator finds their defects'
state: CLOSED
labels:
  - enhancement
  - design
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-19T17:02:13Z'
updatedAt: '2026-11-19T17:57:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/424'
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
closedAt: '2026-11-19T17:57:51Z'
---
# PR frames go up uncritiqued, and the operator finds their defects

## Context

On 19 Nov 2026 the operator found two defects, hours apart, in frames the author had embedded in PR bodies:

- neomjs/neo#21473: the tab strip's overflow control rendered as a filled default button (39 × 32 px) beside a plain 24 × 24 "+". A cold reader had noted that the "+" sits small at the strip's right end; the body carried the remark as colour.
- neomjs/neo#21487: the tree's drag proxy sat at the grab point, over the target folder's label and the left edge of its outline. The author had looked at that frame and judged the overlap acceptable.

His words: "it is nice that you add screenshots so that i can spot defects. it would be even nicer if you would notice them too." Both defects are fixed (`a938add320` on #21473, `647ede4caf` on #21487). This ticket is the substrate half.

## The Problem

The 📸 audit (#409) makes the author embed the whole surface as released and as changed, and a new surface also gets a `## Cold look`: two readers grade the unlabelled pair from 1 to 10. Both answer whether the new whole beats the old one. Neither asks the author what is wrong in their own frames. So the frames reach review unexamined, and the reviewer (here the operator) does the defect hunt. A grade does not name a clipped outline, and nothing makes the author answer a reader's adjacent remark.

The body has no slot for it, and a rule without a slot does not fire: an author fills what the template asks for. Both catches came three days after #409's rule was in place.

## The Architectural Reality

- `.agents/skills/pull-request/references/pull-request-workflow.md`, the PR-body template's `## Test Evidence` placeholder: "A changed rendered surface ⇒ the whole page or tab as released and as changed; new or re-composed ⇒ also `## Cold look` (pr-review 📸 audit)." Nothing about the frames' own defects.
- `.agents/skills/pr-review/audits/rendered-surface-audit.md`: item 1 (the pair), item 2 (the Cold look), item 3 (the verdict first). Its retirement trigger already plans the move into the shared check: "Item 1 retires when the shared PR-body check requires the pair".
- `scripts/check-pr-body.mjs`: `findBodyViolations` joins the anchor checks, `findCloseTargetViolations` and `findImageEmbedViolations`. The last already walks every image embed through `markdownContentLines` and `lineImageEmbedUrls`, so the check already knows when a body embeds a frame.
- `.github/workflows/reusable-pr-baseline.yml` runs the full check only on agent pull requests (a `neo-` login or the `ai` label). Every other pull request gets `--close-target-only`, so a new rule there never refuses an outside contributor.

## The Fix

1. **Template.** `## Test Evidence` asks for one `Frame critique:` line whenever the body embeds a frame. The line names:
   - what a reviewer could fault in each frame: overlap, clipping, a cut-off outline, a truncated label, siblings that differ in size or style, a transitional state;
   - each item's disposition: fixed in `<sha>`, or out of scope with its owner;
   - an answer to each defect a cold reader named.
2. **Check.** `findFrameCritiqueViolations(body)` sits beside `findImageEmbedViolations`. It refuses a body whose content lines embed an image and carry no content line starting `Frame critique:`. Its failure names the line and what it holds. As for the other rules, content lines only: a fenced or indented mention does not count.
3. **Audit.** Item 1 gains the reviewer's half: read the `Frame critique:` line against the frames. A defect a frame shows that the line leaves out is a finding.
4. **Version.** The package version is bumped in the same PR.

Load cost stays small: the template sentence stays short, because the check's failure text carries the full instruction.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| PR-body `## Test Evidence` slot | this ticket; operator report, 19 Nov | one `Frame critique:` line whenever the body embeds a frame | no image ⇒ no line | `pull-request-workflow.md` template | AC-1 |
| `check-pr-body.mjs` | the same | an image embed without the line is a visible violation | fenced and indented mentions do not count; non-agent PRs run `--close-target-only` | the rule's JSDoc | AC-2 specs |
| `rendered-surface-audit.md` item 1 | #409 and this ticket | the reviewer reads the line against the frames | an outside contributor's pair is captured by the reviewer (#409), who writes the line with it | the audit | AC-3 |

**Decision Record impact:** none.

## Acceptance Criteria

- [ ] AC-1: The template's `## Test Evidence` placeholder asks for a `Frame critique:` line whenever the body embeds a frame. The line gives each frame's visible defects with a disposition, and answers each defect a cold reader named.
- [ ] AC-2: `check-pr-body.mjs` refuses a body that embeds an image (Markdown or `<img>`) and has no `Frame critique:` content line, and its failure names the line. Specs cover four cases:
  - an image without the line: refused;
  - an image with the line: passes;
  - no image: passes;
  - the line only inside a fence: refused.
- [ ] AC-3: `rendered-surface-audit.md` item 1 tells the reviewer to read the line against the frames and to raise any defect a frame shows that the line leaves out.
- [ ] AC-4: The package version is bumped in the PR.

## Post-Merge Validation

- [ ] The first rendered-surface PR in neomjs/neo after the consumer bump carries a `Frame critique:` line.

## Out of Scope

- Judging a critique's quality mechanically, or diffing images.
- The Cold look's grading and its readers' prompt.
- Evidence that is not an image.

## Avoided Traps

- **Leave it to review.** Review is where these defects were found, and the cost was the operator's time. The author already has the frame open.
- **A sentence in the audit only.** Nothing in the body asks for a critique, so none gets written; both catches came after #409.
- **One counted line per image.** Frame tables pair released and changed shots, and some frames are reference-only. The check asks for the line; the template asks it to cover each changed frame.

## Related

- #409: the 📸 audit widened to every rendered surface, and the Cold look.
- #271: the same check refuses image embeds that 404 on private repos.
- #14: PR governance, which owns where the checks run; this ticket changes what one check asks.
- neomjs/neo#21473 and neomjs/neo#21487: the two reports.

Live latest-open + closed sweep: checked the latest 20 open issues, plus open and closed searches (frame critique, screenshot defect, Test Evidence frame, rendered surface frame, check-pr-body image), at 2026-11-19T16:58Z. I re-checked at 17:01Z. I read the scope lines of #409, #271 and #14; no equivalent or prior exclusion found. Backlog: 38 open (orange). #14 owns governance infrastructure, not this rule's content, so I am filing this standalone rather than folding it in. A2A: no overlapping claim. Memory Core: no prior decision on an author-side frame critique. My own open assignments: no same-surface hit. Structure map: no new file; the rule sits beside its sibling `findImageEmbedViolations`.

Ownership: self-assigned, @neo-opus-grace implements it. No curated labels, because the work is owned.

Origin Session ID: 01029166-914d-45da-ab32-9adf4cd59857
Retrieval Hint: "author critiques own PR frames for visible defects before posting" · "Frame critique line check-pr-body"

## Timeline

- 2026-11-19T17:02:13Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-19T17:02:15Z @neo-opus-grace added the `enhancement` label
- 2026-11-19T17:02:15Z @neo-opus-grace added the `design` label
- 2026-11-19T17:02:16Z @neo-opus-grace added the `ai` label
- 2026-11-19T17:02:16Z @neo-opus-grace added the `model-experience` label
- 2026-11-19T17:03:53Z @github-actions added the `needs-design-authority` label
- 2026-11-19T17:11:33Z @neo-opus-grace cross-referenced by PR #425
- 2026-11-19T17:57:51Z @tobiu referenced in commit `c31d48b` - "feat(pull-request): a PR body that embeds a frame critiques it (#424) (#425)

The operator found two defects on 19 Nov 2026 in frames their author had
embedded: neomjs/neo#21473 (mismatched header buttons) and neomjs/neo#21487
(a drag proxy over the target's label and outline). The author had looked at
the second and judged it acceptable. Nothing in the body asked for a critique,
so the defect hunt fell to review.

- check-pr-body.mjs: findFrameCritiqueViolations refuses an agent body that
  embeds an image with no `Frame critique:` content line. Fenced, indented and
  inline-code mentions do not count. --close-target-only (human PRs) is
  unchanged.
- pull-request-workflow.md: the Test Evidence placeholder asks for the line.
- rendered-surface-audit.md: the reviewer reads the line against the frames.
  The raw-embed rule shrinks to a pointer, since the check has refused such
  embeds since #271.
- 2.1.0: a new refusal rule for consumers."
- 2026-11-19T17:57:52Z @tobiu closed this issue

