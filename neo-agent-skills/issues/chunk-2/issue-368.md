---
id: 368
title: Eight npm test suites fail on a Windows core.autocrlf checkout
state: OPEN
labels:
  - bug
  - good first issue
  - windows
  - contributor-experience
  - ai
  - testing
assignees: []
createdAt: '2026-11-12T02:10:24Z'
updatedAt: '2026-11-12T02:10:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/368'
author: neo-gemini-maryam
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
# Eight npm test suites fail on a Windows core.autocrlf checkout

## Context

During the Cycle-1 review of `#355` (`#367`), running the repository's full `npm test` suite against a `git -c core.autocrlf=true` checkout of `#355` head `986327b4acde06ee40fc538ce5b7918107dd8103` confirmed that `#355`'s five touched files pass `npm run lint`, `scripts/test-lint-skill-corpus.mjs`, `scripts/test-agents-md-pointer.mjs`, and `scripts/test-materialize-harness-skills.mjs`, while eight other repository test suites still fail because `neomjs/neo-agent-skills` ships no `.gitattributes` file.

## The Problem

Git for Windows enables `core.autocrlf=true` by default. Because `neomjs/neo-agent-skills` has no `.gitattributes`, cloning the repository on Windows checks out `.github/workflows/*.yml`, `.agents/skills/**/*.md`, and `agents-md/**/*.md` with `\r\n` line endings:

1. **Eight of the 23 suites in `package.json#scripts.test` fail on a `core.autocrlf=true` checkout** (producer: `git config core.autocrlf true && git rm --cached -r . && git reset --hard` followed by per-suite `node scripts/test-*.mjs` execution over all 23 commands in `package.json#scripts.test` at commit `986327b4acde06ee40fc538ce5b7918107dd8103`; scope: `scripts/test-*.mjs`; sampled hits below; positive control: adding `* text=auto eol=lf` in `.gitattributes` before `git rm --cached -r . && git reset --hard` reduces fast-suite failures from `7/22` to `0/22` and keeps `pull-request-workflow.md` LF for `scripts/test-generate-agents-md.mjs`):
   - `scripts/test-generate-agents-md.mjs` (line 587 fails extracting the fenced PR body template from CRLF `pull-request-workflow.md` inside the `npm pack` consumer tarball)
   - `scripts/test-check-pr-review-body.mjs` (line 53 fails matching `"- **Decision**: Drop+Supersede\n"` against CRLF `pr-review-template.md`)
   - `scripts/test-reusable-pr-baseline.mjs` (CRLF `.github/workflows/reusable-pr-baseline.yml`)
   - `scripts/test-reusable-dependabot-automerge.mjs` (line 77, CRLF workflow YAML)
   - `scripts/test-check-pr-body.mjs` (CRLF `.github/workflows/skill-corpus.yml`)
   - `scripts/test-reusable-pr-review.mjs` (CRLF `.github/workflows/reusable-pr-review.yml`)
   - `scripts/test-check-ticket-body.mjs` (line 94 `/^permissions:\n  contents: read\n/m` against CRLF `.github/workflows/skill-corpus.yml`)
   - `scripts/test-release-origin.mjs` (line 68, CRLF release workflow YAML)

```text
FAIL: node scripts/test-generate-agents-md.mjs
  TypeError: Cannot read properties of null (reading '1')
    at scripts/test-generate-agents-md.mjs:587:108

FAIL: node scripts/test-check-pr-review-body.mjs
  AssertionError [ERR_ASSERTION]: fixture lost "- **Decision**: Drop+Supersede\n"
    at swap (scripts/test-check-pr-review-body.mjs:53:12)
```

2. **Mixed-EOL output and byte-budget inflation on CRLF `agents-md/` checkouts:** While `scripts/generate-agents-md.mjs#parseSection` intentionally preserves `\r` in section bodies (`scripts/test-generate-agents-md.mjs` lines 70–86), checking out `agents-md/**/*.md` as CRLF causes `generate()` to emit mixed `\r\n` + `\n\n` output and inflates `neo-agent-brain/maintainer` from `23,404 B` to `23,557 B` (`+153 B` against the `23,800 B` cap), and packing from a CRLF working tree ships CRLF Markdown inside the npm tarball.

## The Architectural Reality

Design authority: `"Count text bytes using the repository's canonical LF line endings."` (`scripts/lint-skill-corpus.mjs` / `#367`: `"The budgets measure the repository's canonical LF bytes, which is what the package publishes."`).

- Repository-checkout line endings belong in `.gitattributes` (`* text=auto eol=lf`), not in scattered `.replace(/\r\n/g, '\n')` regexes across workflow YAML and template test suites.
- `#355`'s runtime normalizations (`lint-skill-corpus.mjs` `fileBytes`, `generate-agents-md.mjs` front-matter delimiter, and `materialize-harness-skills.mjs` `'junction'` + `realpathSync`) remain complementary and necessary: they handle Windows consumer `postinstall` junctions and unstaged CRLF working-tree files written by Windows editors before Git's clean filter runs.

## The Fix

1. Add a root `.gitattributes` file declaring `* text=auto eol=lf` so `git clone` / `git checkout` under `core.autocrlf=true` materializes tracked text files (`agents-md/**`, `.agents/skills/**`, `.github/workflows/**`, `scripts/**`) with canonical LF line endings, and `npm pack` on Windows packs canonical LF files.
2. Add a real-git regression check (e.g., in `scripts/test-agents-md-pointer.mjs` or a dedicated check in `scripts/test-lint-skill-corpus.mjs`) that clones/checks out the repository with `git -c core.autocrlf=true` and asserts checked-out Markdown/YAML files contain no `\r\n`.
3. Bump `package.json` and `package-lock.json` version only if authored on an internal maintainer branch; fork contributors leave the base version untouched per `#358`.

## Acceptance Criteria

- [ ] Root `.gitattributes` enforces `eol=lf` for text files in `neomjs/neo-agent-skills`.
- [ ] On a fresh `git -c core.autocrlf=true` clone/checkout of the repository, tracked `.md`, `.yml`, and `.mjs` files check out with LF (`!\r\n`), `generate({audience: 'maintainer', repos: ['neo-agent-brain']}).bytes` matches an LF checkout, and `npm test` + `npm run lint` exit `0`.
- [ ] A regression test exercises `git -c core.autocrlf=true` against `.gitattributes` and fails when `.gitattributes` is removed.

## Out of Scope

- Replacing `#355`'s runtime CRLF / `'junction'` handling in `lint-skill-corpus.mjs`, `generate-agents-md.mjs`, or `materialize-harness-skills.mjs` (those protect Windows consumer installs and unstaged editor buffers).
- Adding a hosted Windows GitHub Actions runner.

## Avoided Traps

- **Patching `.replace(/\r\n/g, '\n')` across all 8 failing workflow/template test files instead of `.gitattributes`:** treats the checkout symptom file-by-file, bloats test harnesses, and still lets `npm pack` on a Windows checkout ship CRLF Markdown payloads.
- **Normalizing `\r\n` inside `parseSection` / `readPreamble` in `scripts/generate-agents-md.mjs`:** contradicts the explicit contract in `scripts/test-generate-agents-md.mjs` lines 70–86 (`trimFinalLineFeeds` strips `\n` only).

## Related

#367 · #355 · #358

Decision Record impact: none.

Live latest-open + closed sweep: checked latest 25 of 37 open (orange band; `#367` scopes `#355`'s five-file diff via AC-1, so folding `.gitattributes` there would violate `#367` AC-1) + closed search (`gitattributes OR autocrlf OR CRLF OR Windows OR eol`, 11 hits; read `Out of Scope` / `Avoided Traps` on `#353` and `#346`) at 2026-11-12T02:08Z; no equivalent or prior exclusion found.
A2A in-flight claim sweep: checked latest 30 messages (`status: 'all'`) at 2026-11-12T02:09Z; no in-flight claim on `.gitattributes` or Windows `npm test`.
MC sweep: `"npm test fails on Windows core.autocrlf checkout CRLF workflow regex assertion"`, 5 results, no prior decision found.
Own-assignment sweep: 0 open in `neomjs/neo-agent-skills`, none overlapping.
Structure map: ran `npm run ai:structure-map -- --files --loc` (4,571 lines); root `.gitattributes` matches `neomjs/neo-stage/.gitattributes` and consumer `.gitattributes` fixture in `scripts/test-agents-md-pointer.mjs:46`.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: 3fcf378b-7eba-44ca-9bd4-7e185731aace

Retrieval Hint: "neo-agent-skills Windows core.autocrlf=true checkout eight npm test suites fail CRLF .gitattributes eol=lf"

## Timeline

- 2026-11-12T02:10:26Z @neo-gemini-maryam added the `bug` label
- 2026-11-12T02:10:26Z @neo-gemini-maryam added the `good first issue` label
- 2026-11-12T02:10:27Z @neo-gemini-maryam added the `windows` label
- 2026-11-12T02:10:27Z @neo-gemini-maryam added the `contributor-experience` label
- 2026-11-12T02:10:27Z @neo-gemini-maryam added the `ai` label
- 2026-11-12T02:10:27Z @neo-gemini-maryam added the `testing` label
- 2026-11-12T02:13:41Z @neo-gemini-maryam cross-referenced by PR #355

