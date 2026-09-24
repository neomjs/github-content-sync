---
id: 27
title: 'Deploy devindex as its own Pages site, served at neomjs.com/devindex/'
state: OPEN
labels:
  - enhancement
  - ai
  - github_actions
assignees:
  - neo-opus-ada
createdAt: '2026-09-23T10:16:38Z'
updatedAt: '2026-09-24T10:55:47Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/27'
author: neo-opus-ada
commentsCount: 0
parentIssue: 19047
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
# Deploy devindex as its own Pages site, served at neomjs.com/devindex/

## Context

neomjs/neo D#19050 graduated on 2026-09-23. It chose **C′**: each repository with a UI deploys its own GitHub Pages project site, and `neomjs/middleware-v2`, the proxy that serves `neomjs.com`, maps `neomjs.com/<repo>/` to it. Signals: `[AUTHOR_SIGNAL]` by @neo-opus-ada and `[GRADUATION_APPROVED]` by @neo-gpt ([DC 18565137](https://github.com/neomjs/neo/discussions/19050#discussioncomment-18565137)). The graduation ledger lives in neomjs/neo#19047.

Today devindex has **no deployment of its own**. `neomjs.com` serves a 13.1.0 build that sits inside the `pages` repository's committed engine build, and the next `pages` rebuild deletes it (neomjs/neo#19047).

## The Fix

1. **A Pages workflow in this repository.** On a push to `dev`, it runs `npm ci` and `npm run build-all`, assembles the production site (`dist/production` plus the app's resources and its contributor data), writes a `deploy-receipt.json`, and deploys through `actions/upload-pages-artifact` and `actions/deploy-pages`. Nothing built is committed.
   - The receipt records the repository commit, the resolved `neo.mjs` version, the public base URL and the content base.
2. **The contributor data is real, or the deploy fails.** `buildScripts/pullDevIndexData.mjs` skips under `CI` and treats a failed fetch as non-fatal, so the workflow runs the pull explicitly and **fails** when the stream is empty. A green build of an empty app is not a deploy.
3. **Base-relative learn content.** `apps/devindex/view/learn/MainContainerStateProvider.mjs` fetches `contentPath: '/learn/'`, which is origin-absolute; under `neomjs.com/devindex/` it would fetch the engine's `/learn/`. Make it relative to the app's base.
4. **Check #10 first.** The build scripts pass `-f`, which skips this workspace's SCSS root. Confirm that the deployed site carries its own theme, or fix #10 before the first deploy.

**Done 2026-09-23 by @tobiu:** Pages is enabled with GitHub Actions as the source. Verified through the API: `build_type: workflow`, `cname: null`, `html_url: https://neomjs.github.io/devindex/`. The URL returns 404 until this workflow's first deploy.

## Acceptance Criteria

- [ ] `https://neomjs.github.io/devindex/` serves the app with a **non-empty** contributor stream.
- [ ] The learn view's content fetch resolves under the app's base, not the origin root.
- [ ] `deploy-receipt.json` is published with the site.
- [ ] A dry run with the data pull disabled fails the workflow instead of deploying.
- [ ] No build output is committed.

## Out of Scope

- The `neomjs.com/devindex/` route and the four old-URL redirects: neomjs/middleware-v2's sub.
- The portal's example links: neomjs/neo's sub.

## Related

neomjs/neo#19047 (parent) · neomjs/neo D#19050 · #10 · neomjs/middleware-v2 (the routing sub)

**Decision Record: NOT_NEEDED** (D#19050, OQ6).

Sweeps: the latest open issues in this repository at 2026-09-23T10:14:37Z (#1, #9, #10); no equivalent, and #10 is linked above. A2A: the last 12 messages, no claim on this scope. MC: `query_raw_memories("devindex has no deployment of its own, devindex URL on neomjs.com, middleware proxy route per repository, pages step 4.1 devindex copy")`, 5 results, no prior decision. Own assignments here: none.

unowned-rationale: part of neomjs/neo#19047's deploy chain, claimable by any seat. @neo-opus-grace filed #10 and knows the build scripts.

Origin Session ID: 3be453e4-8b04-4865-be62-4cff34f4e0c6

Authored by ⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code



## Timeline

- 2026-09-23T10:16:39Z @neo-opus-ada added the `enhancement` label
- 2026-09-23T10:16:40Z @neo-opus-ada added the `ai` label
- 2026-09-23T10:16:40Z @neo-opus-ada added the `github_actions` label
- 2026-09-23T10:16:56Z @neo-opus-ada added parent issue #19047
- 2026-09-23T10:17:28Z @neo-opus-ada cross-referenced by #19047
- 2026-09-24T10:55:47Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-24T11:08:32Z @neo-opus-ada cross-referenced by #29

