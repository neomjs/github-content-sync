---
id: 27
title: 'Deploy devindex as its own Pages site, served at neomjs.com/devindex/'
state: CLOSED
labels:
  - enhancement
  - ai
  - github_actions
assignees:
  - neo-opus-ada
createdAt: '2026-09-23T10:16:38Z'
updatedAt: '2026-09-24T14:25:54Z'
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
closedAt: '2026-09-24T13:19:46Z'
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

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Site entry and config (new): `rootEntry()` and `assembleSite()` in `buildScripts/assemblePagesSite.mjs` write `index.html` and `dist/production/neo-config.json` | Fix 1 · D#19050 C′ | The root `index.html` is the build's app entry with `<base href="./dist/production/">`, `src/MicroLoader.mjs`, the favicon under `apps/devindex/resources/images/`, and a click handler that keeps `#` links on the page. The config is the build's with `basePath: '../../'` and `workerBasePath: './'`. The page resolves it from `<base>` and the workers from their own directory, and both reach the mount | Any of the four entry rewrites that finds nothing to replace throws `shape changed`, and no site is written | JSDoc | `AssemblePagesSite.spec.mjs`: "the site root serves the app from one base…", "a build entry of another shape fails…" |
| Learn content base: `contentPath` in `apps/devindex/view/learn/MainContainerStateProvider.mjs` | Fix 3 | Production fetches `Neo.config.basePath + 'learn/'`, which is `<mount>/learn/`, where `assembleSite()` copies the guides | Development keeps `/learn/`, the dev server's workspace root | JSDoc | spec: the receipt's `contentBase` equals `basePath + 'learn/'` resolved from `dist/production/`; the PR's mounted-site learn and deep-hash requests |
| `deploy-receipt.json` (new), written by `assembleSite()` from the script's CLI | Fix 1 | `commit` (`GITHUB_SHA`, else `HEAD`), `neoVersion` (the installed `neo.mjs`), `dataSource` (the index URL), `publicBase`, `contentBase`, `dataDigest` (SHA-256 of the shipped `users.jsonl`), `dataRecords` (its non-blank lines) | A `publicBase` without its trailing slash is refused before anything is written | JSDoc | spec: the receipt arm and the trailing-slash arm; post-merge: the first run's receipt names the merge commit |
| Empty-index refusal: `assembleSite()`, fed by `pullDevIndexData.mjs` | Fix 2 | The workflow clears `CI` for `devindex:pull-data`. Zero non-blank records throws before the site directory exists, so upload and deploy never run | The pull itself is never fatal (`postinstall`, `skipReason()`), so the assembly is the gate | JSDoc and workflow comment | spec: the missing, empty and blank-lines-only arms; the PR's disabled-pull fresh build |

Rows anchored at `fff3cfe`: `rootEntry`/`assembleSite` read in full, `contentPath` at `MainContainerStateProvider.mjs:30`, `skipReason()` and the non-fatal `run()` in `pullDevIndexData.mjs`.

## Post-Merge Validation

AC-1 and AC-3 name the live site, so only the first deploy after merge can witness them: the first `Pages` run on `dev` serves `https://neomjs.github.io/devindex/` with a non-empty stream, and its `deploy-receipt.json` names the merge commit. Before merge, the PR witnesses both on the assembled site served under a `/devindex/` mount.

**Receipts, 2026-09-24 13:27Z** (first `Pages` run, [36004902381](https://github.com/neomjs/devindex/actions/runs/36004902381), on merge commit `c3d44f166c`, green):
- HTTP 200: `/devindex/` (the root entry), `deploy-receipt.json`, `apps/devindex/resources/data/users.jsonl` (24,084,949 bytes), `learn/tree.json`, `dist/production/neo-config.json`.
- The receipt names `commit: c3d44f166c…`, `dataRecords: 50000`, `dataDigest: dcf299d9…`, `publicBase` and `contentBase: https://neomjs.github.io/devindex/learn/`, so AC-3 is witnessed.
- **AC-1 fails live.** Witnessed in Chromium at 13:30Z: the page boots and reports `Visible Rows: 50,000`, but the grid body paints **no rows**, and the viewport's vdom update wedges. The byte-identical build painted all rows in the pre-merge rehearsal on `127.0.0.1`, so the cause is environmental. It is tracked in #34, with the fix in #35. **Met after #35 (14:25Z):** the redeployed site paints its grid, 50,000 rows streamed (#34's validation).

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
- 2026-09-24T11:48:43Z @neo-opus-ada cross-referenced by #10
- 2026-09-24T11:54:58Z @neo-opus-ada cross-referenced by PR #30
- 2026-09-24T12:01:08Z @neo-opus-ada cross-referenced by PR #31
- 2026-09-24T12:46:47Z @neo-gpt cross-referenced by PR #32
- 2026-09-24T13:01:57Z @neo-opus-ada cross-referenced by #33
- 2026-09-24T13:11:03Z @neo-opus-ada referenced in commit `fff3cfe` - "fix(pages): the deploy receipt names the content base the learn view fetches (#27)

#27 promises a receipt with the public base and the content base; it carried only the first. The assembler now
owns both: it takes the mount as `publicBase`, copies the guides to `learn/` and derives `contentBase` from that
same directory. A mount without its trailing slash is refused, since URL resolution would drop its last segment.

The witness checks the receipt against the app's own resolution: the written `basePath` plus `learn/`, from the
directory the workers run in, must be the receipt's `contentBase`."
- 2026-09-24T13:19:46Z @tobiu referenced in commit `c3d44f1` - "Merge pull request #31 from neomjs/ada/27-pages-deploy

feat(ci): devindex deploys as its own Pages site, the app at the mount's root (#27)"
- 2026-09-24T13:19:46Z @tobiu closed this issue
- 2026-09-24T13:40:53Z @neo-opus-ada cross-referenced by #34
- 2026-09-24T13:43:06Z @neo-opus-ada cross-referenced by #19163
- 2026-09-24T13:55:59Z @neo-gpt-emmy cross-referenced by PR #35
- 2026-09-24T15:40:35Z @neo-gpt-emmy cross-referenced by PR #36

