---
id: 255
title: Stage before/after audit passes pairs that 404 on the private repo
state: OPEN
labels:
  - bug
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-11-01T18:09:32Z'
updatedAt: '2026-11-01T18:09:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/255'
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
# Stage before/after audit passes pairs that 404 on the private repo

## Context

The pr-review template's **📸 Before/After Audit** (#244) asks that a `neomjs/neo-stage` PR body embed one before and one after screenshot. `neo-stage` is private. An embed at `raw.githubusercontent.com/neomjs/neo-stage/<sha>/<path>` carries no token. That host never sees the viewer's github.com session, so the image 404s for everyone, the operator included. The pair is "in the body" and passes the line as written, but nobody can see it.

There are two independent occurrences:

- neomjs/neo-stage#519 (2026-10-31): the operator found every image in the body broken. All 12 embeds used the full-SHA `raw.githubusercontent.com` form.
- neomjs/neo-stage#630 (2026-11-01): the pair had been captured, hash-bound and committed, and both embeds used the same form. [Round-2 review 5729485544](https://github.com/neomjs/neo-stage/pull/630#pullrequestreview-5729485544) kept RA1 open for rendering alone.

The census covers the last 60 Stage PRs; 43 of their bodies reference images. Nine use the raw form: 7 merged (neomjs/neo-stage#574, neomjs/neo-stage#577, neomjs/neo-stage#578, neomjs/neo-stage#583, neomjs/neo-stage#636, neomjs/neo-stage#643, neomjs/neo-stage#645), plus closed neomjs/neo-stage#604 and open neomjs/neo-stage#630. The rest use `https://github.com/neomjs/neo-stage/blob/<sha>/<path>?raw=true` (e.g. neomjs/neo-stage#641), which resolves through the viewer's session.

## The Problem

Authors and reviewers check that the pair is present, not that it renders. Nothing names the route that works on a private repository, so each author picks one. A 404 never shows up as a failed check. Instruments: an anonymous fetch of the neomjs/neo-stage#630 URL returns 404, and the PR's rendered `body_html` carries the bare URL with no token.

## The Architectural Reality

- The line lives in `.agents/skills/pr-review/assets/pr-review-template.md` (`📸 Before/After Audit`, last touched `f5c9760`). It is an asset line; the `SKILL.md` router is untouched. Structure map: N/A, because this amends one existing line in its owning template and changes no placement.
- Design authority: the operator's Stage merge rule as recorded in #244's Context: "i will not merge a single new PR without a before and after screenshot in the PR body. at least one." A screenshot he cannot see does not meet it.

## The Fix

Amend the audit line: both images must render for a signed-in viewer. On the private repository that means `https://github.com/neomjs/neo-stage/blob/<full-sha>/<path>?raw=true`. A `raw.githubusercontent.com` embed counts as a missing pair.

Load impact: about +150 bytes on a template loaded once per Stage review. Retire the clause when `neo-stage` turns public (raw URLs then render) or when a body lint enforces the route.

## Acceptance Criteria

- [ ] The Before/After Audit line requires both images to render, names the `?raw=true` route, and treats a `raw.githubusercontent.com` embed as a missing pair.
- [ ] `npm run lint` passes in a Skills checkout (corpus byte budgets).

## Out of Scope

- Rewriting the merged bodies in the census.
- A mechanical PR-body lint (a successor, if the line proves insufficient).
- Authoring guidance in `neo-stage`'s own `AGENTS.md`.

## Related

#244 (the audit line) · neomjs/neo-stage#519 · neomjs/neo-stage#630 · neomjs/neo-stage#641 (the working route)

Live latest-open + closed sweep: checked latest 20 open + closed search (`screenshot OR image OR raw.githubusercontent OR before after`; #244 read) at 2026-11-01T18:08Z. A2A claims (last 30, all read-states), the Memory Core rationale sweep and own open assignments were checked too. No equivalent or prior exclusion found.

Decision Record impact: none.

Origin Session ID: 532727df-bad5-4b83-bf57-3878087ac8f0

Retrieval Hint: "Stage PR body images raw.githubusercontent.com 404 private repo before/after audit"

## Timeline

- 2026-11-01T18:09:32Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-01T18:09:34Z @neo-opus-ada added the `bug` label
- 2026-11-01T18:09:34Z @neo-opus-ada added the `ai` label

