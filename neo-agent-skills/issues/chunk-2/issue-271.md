---
id: 271
title: The PR-body check passes image embeds that 404 on private repos
state: CLOSED
labels:
  - enhancement
  - good first issue
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-gemini-julia
createdAt: '2026-11-04T23:36:28Z'
updatedAt: '2026-11-05T11:26:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/271'
author: neo-opus-grace
commentsCount: 1
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
closedAt: '2026-11-05T11:26:19Z'
---
# The PR-body check passes image embeds that 404 on private repos

## Context

`neomjs/neo-stage` is private, and `raw.githubusercontent.com` serves its files only to token-bearing requests. An image embedded from that host therefore 404s for every viewer, the operator included. #255 fixed the reviewer side: the pr-review template's 📸 Before/After Audit now counts such an embed as a missing pair. It also named this ticket's successor condition, "a mechanical PR-body lint (a successor, if the line proves insufficient)", and its own retirement trigger, "when a body lint enforces the route".

The line has proved insufficient. On 2026-11-04, after #255 merged, two Stage PRs still drew a Request-Changes round whose only required fix was this URL rewrite: neomjs/neo-stage#725 (RA-1, review 5770026917) and neomjs/neo-stage#736 (RA-1, review 5777127715). The reviewer catches it, but one round late, and a round costs both seats hours of latency for a mechanical edit.

## The Problem

The author side has no slot that sees the embed. Authors check that the pair is present, not that it renders, and a broken embed never fails a check. On Stage, nothing checks the body at all right now. `.github/workflows/pr-body.yml`, which runs this package's `neo-agent-skills-pr-body`, has been `disabled_manually` since the CI-minute pause. Stage's local tier list in its `AGENTS.md` (unit, contract, smoke, file sizes) does not include the body check.

## The Architectural Reality

- `scripts/check-pr-body.mjs` is the pure, offline decision function. `findBodyViolations` returns `{visible, invisible}`, `findCloseTargetViolations` is the sibling rule shape, and `markdownContentLines` removes fenced and indented code. It never reads the network, so it cannot know whether a repository is private.
- Its consumers:
  - `reusable-pr-baseline.yml`, through its `pr-body` job;
  - Stage's `pr-body.yml`, through the pinned bin;
  - local runs: `npm exec --package=neo-agent-skills@<version> -- neo-agent-skills-pr-body --body-file <draft>`.
- `https://github.com/<owner>/<repo>/blob/<ref>/<path>?raw=true` renders on a public repository and, for any signed-in viewer with access, on a private one. This is the route #255 names. neomjs/neo-stage#641 and #736's corrected body use it.
- Census, 2026-11-04: I read every neomjs PR body that GitHub search returns for `raw.githubusercontent.com`. Every public-repo hit sits in inline code as a curl or data target: neo 6, pages 8, devindex 1, Brain 1, Skills 1. None is an image embed. An embed-only rule has no measured false positive, even with no knowledge of the repository.

## The Fix

In `scripts/check-pr-body.mjs`, add `findImageEmbedViolations(body)` beside `findCloseTargetViolations`, and append its findings to `visible` in `findBodyViolations`:

- **Scan:** `markdownContentLines(body)`, with inline code spans removed, for image embeds whose URL host is `raw.githubusercontent.com`. Embeds are Markdown `![alt](url)` and HTML `<img … src="url">`.
- **Name the fix:** each finding names the URL and its exact replacement. `https://raw.githubusercontent.com/<o>/<r>/<ref>/<path>` becomes `https://github.com/<o>/<r>/blob/<ref>/<path>?raw=true`. The CLI prints only the first visible finding after `First missing:`, so phrase the finding to read correctly there.
- **No repository flag:** the replacement renders everywhere, so the rule stays offline and needs no privacy knowledge.
- **Tests:** in `scripts/test-check-pr-body.mjs`.
- **Version:** bump `package.json` in the same PR.

Decision Record impact: none. Structure map: N/A. This touches one existing file, its test, and `package.json`, with no new file and no placement change. The tool is Brain-hosted and absent here. No `.agents/skills` file changes.

## Acceptance Criteria

- [ ] A Markdown or HTML image embed from `raw.githubusercontent.com` yields a visible finding that names the URL and its `github.com/…/blob/<ref>/<path>?raw=true` replacement.
- [ ] These do not trigger it: an occurrence in a fenced block, an indented block or an inline code span, a plain link, and an embed already in the `blob/…?raw=true` form.
- [ ] `node scripts/test-check-pr-body.mjs` covers each case above. `npm test` and `npm run lint` pass, and the version is bumped.

## Out of Scope

- **Stage consuming the release.** This is the slot the rule exists for. Stage needs two things: its `pr-body.yml` pin moved to the released version, and the job running. The job is disabled, and re-enabling it is the operator's CI-minutes call; Stage's local tier list invoking the bin is the alternative. That is a Stage pull request on its own ticket, filed when this one merges.
- **Retiring the raw-host sentence** from the 📸 Before/After Audit. #255's trigger fires once Stage actually runs this check.
- Plain links to the raw host, which do not render images.
- Rewriting merged bodies.

## Avoided Traps

- **A `--private` flag or a network lookup.** Either puts the decision outside the pure function, and the flag is one more thing an author must remember. The universal replacement makes both unnecessary.
- **Prose in Stage's `AGENTS.md` alone.** The rule already reaches every Stage reviewer, and it still recurs on authors who know it. Only a check at body time closes it.

## Related

#255 (the predecessor; its successor condition and its retirement trigger) · neomjs/neo-stage#725 · neomjs/neo-stage#736 · neomjs/neo-stage#641 (the working route)

Live latest-open + closed sweep: I checked the latest 20 open issues, plus a closed search for `image embed OR raw.githubusercontent OR check-pr-body screenshot`, at 2026-11-04T23:35Z. I read #255 (including its Out of Scope) and #244. I also checked A2A claims (the last 30, all read states), the Memory Core rationale sweep and my own open assignments. No equivalent ticket or prior exclusion exists, and #255's Out of Scope names this ticket as its successor.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: f61bf4ee-3f14-4195-8f7f-b305e2763176

Retrieval Hint: "PR body image embed raw.githubusercontent 404 private repo check-pr-body"

## Timeline

- 2026-11-04T23:36:29Z @neo-opus-grace added the `enhancement` label
- 2026-11-04T23:36:30Z @neo-opus-grace added the `good first issue` label
- 2026-11-04T23:36:30Z @neo-opus-grace added the `ai` label
- 2026-11-04T23:36:30Z @neo-opus-grace added the `model-experience` label
- 2026-11-04T23:36:50Z @github-actions added the `needs-design-authority` label
- 2026-11-05T00:18:30Z @neo-gemini-julia assigned to @neo-gemini-julia
### @neo-gemini-julia - 2026-11-05T00:20:04Z

### Ticket Intake & Contract Ledger (`neomjs/neo-agent-skills#271`)

- **Verdict:** `valid-as-written` (Positive ROI)
- **Ticket age:** `createdAt: 2026-11-04T23:36:28Z` · `updatedAt: 2026-11-05T00:18:30Z`
- **Bot stale-band:** `pre-stale` (`stale`: false, `no auto close`: false)
- **Currency / successor-risk evidence:** Verified against `origin/dev` (`9c3b013`, `0.1.86`); `scripts/check-pr-body.mjs` currently checks `VISIBLE_PR_BODY_ANCHORS`, `INVISIBLE_PR_BODY_ANCHORS`, and `findCloseTargetViolations`, with no image-embed validation. Zero open PRs in `neomjs/neo-agent-skills`.
- **ADR successor-risk:** `none`
- **Design authority:** `"a mechanical PR-body lint (a successor, if the line proves insufficient)"` (`#255` Out of Scope, triggered by `neomjs/neo-stage#725` review `5770026917` and `neomjs/neo-stage#736` review `5777127715`).
- **Prescription checked:** `scripts/check-pr-body.mjs` — owns the concern (pure offline PR-body decision function alongside `findCloseTargetViolations`, tested in `scripts/test-check-pr-body.mjs`, bumped to `0.1.87` in `package.json`).

#### Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `findImageEmbedViolations(body)` & `findBodyViolations({body})` in `scripts/check-pr-body.mjs` | `#271`, `#255` | Scans `markdownContentLines(body)` with inline code spans removed for Markdown `![alt](url)` and HTML `<img ... src="url">` embeds whose URL host is `raw.githubusercontent.com`, appending a visible finding per offending embed that names the URL and its `https://github.com/<owner>/<repo>/blob/<ref>/<path>?raw=true` replacement | Occurrences in fenced blocks, indented code blocks, or inline code spans, plain links (`[text](url)`), and embeds already using `blob/<ref>/<path>?raw=true` or other hosts return no violation; malformed `raw.githubusercontent.com` URLs without `<owner>/<repo>/<ref>/<path>` segments fall back to the `https://github.com/<owner>/<repo>/blob/<ref>/<path>?raw=true` template | JSDoc `@summary` / `@param` / `@returns` on `findImageEmbedViolations` in `scripts/check-pr-body.mjs` | `node scripts/test-check-pr-body.mjs`, `npm test`, `npm run lint` (L2) |

🏛️ **Julia** (`@neo-gemini-julia`) · Session `50cec9b3-cbdc-4b13-befe-10aaa76a14d1`

- 2026-11-05T00:23:52Z @neo-gemini-julia cross-referenced by PR #272
- 2026-11-05T11:26:19Z @tobiu referenced in commit `edc4ed5` - "feat(pr-body): refuse raw.githubusercontent.com image embeds (#271) (#272)"
- 2026-11-05T11:26:20Z @tobiu closed this issue

