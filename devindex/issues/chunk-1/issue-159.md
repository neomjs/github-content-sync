---
id: 159
title: 'DevIndex''s page tells search what it is: title, description and canonical URL'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-21T20:27:19Z'
updatedAt: '2026-11-21T20:50:30Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/159'
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
closedAt: '2026-11-21T20:50:30Z'
---
# DevIndex's page tells search what it is: title, description and canonical URL

## Context

In Search Console's three-month export for `sc-domain:neomjs.com`, `/apps/devindex/` (which redirects to `/devindex/`) is the second-highest-impression non-archive URL on the domain: 46 impressions, average position 5.57, 0 clicks. Julia read this export for neomjs/neo#20671. On 2026-11-21, the served page's head carries `<title>DevIndex</title>`, no meta description, no OpenGraph tags and no canonical link. That is true for a browser user agent and for Googlebot alike.

## The Problem

A searcher who sees the result on page one learns nothing from it: "DevIndex", with a snippet Google has to improvise. Nothing says that the page ranks the most active developers on GitHub, which is the reason to click.

## The Architectural Reality

- The head comes from `apps/devindex/index.html`. `buildScripts/assemblePagesSite.mjs` assembles the Pages site without touching `<title>` or `<meta>`, and middleware-v2 proxies `/devindex/` to that site. So humans and crawlers both read this file's head.
- The facts the copy may claim are in `learn/Methodology.md`: the ranking is the sum of lifetime public contribution metrics (`:7`, `:13`), and the index is capped at the top 50,000 developers (`:17`). `learn/Introduction.md` disclaims affiliation with GitHub, Inc.
- Design authority: n/a. This adds missing metadata and calls no existing behaviour wrong.

## The Fix

In `apps/devindex/index.html`'s head, add a search-intent `<title>`, a `<meta name="description">`, `og:title`, `og:description`, `og:url`, `og:type`, and `<link rel="canonical" href="https://neomjs.com/devindex/">`. The canonical link consolidates the Pages origin URL onto the domain. Every claim stays inside Methodology's facts, and the copy does not imply GitHub endorsement.

## Acceptance Criteria

- [ ] After the Pages deploy, `https://neomjs.com/devindex/` serves the new title, description, OpenGraph tags and canonical link, both to a browser user agent and to a Googlebot user agent.
- [ ] Every claim in the copy matches `learn/Methodology.md`.

## Out of Scope

Prerendered snapshots for DevIndex's own routes, and the Portal card that links to DevIndex.

## Related

neomjs/neo#20671 (search row) · #61

Live latest-open + closed sweep: the 3 open issues here plus an all-states search ("title description seo") at 2026-11-21T20:30Z found no equivalent. Band: 3 open (green). A2A claim sweep: no claim on DevIndex's head. Own-assignment sweep: #135 and #61 are mine, and neither covers metadata.

Origin Session ID: e869d0af-8242-40f0-bd1f-f90653208a16

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-11-21T20:27:20Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-21T20:27:21Z @neo-opus-grace added the `enhancement` label
- 2026-11-21T20:27:22Z @neo-opus-grace added the `ai` label
- 2026-11-21T20:28:04Z @neo-opus-grace cross-referenced by PR #160
- 2026-11-21T20:50:29Z @tobiu referenced in commit `8d20e69` - "feat(devindex): the page tells search what it is: title, description and canonical URL (#159) (#160)"
- 2026-11-21T20:50:31Z @tobiu closed this issue

