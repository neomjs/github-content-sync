---
id: 3624
title: 'Docs should separate "app" from API tab, and should exclude empty folders'
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2022-12-17T23:18:42Z'
updatedAt: '2026-11-05T10:52:45Z'
githubUrl: 'https://github.com/neomjs/neo/issues/3624'
author: maxrahder
commentsCount: 6
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
# Docs should separate "app" from API tab, and should exclude empty folders

The most common use-case for the docs is looking up API documentation. But theapi tab includes both "apps" and "src". That makes it harder to see class documentation. Furthermore, when empty folders show up when filtering -- they should be excluded if there's no match for the directory or its children.

## Timeline

- 2022-12-17T23:18:42Z @maxrahder added the `enhancement` label
### @tobiu - 2022-12-18T11:30:43Z

can you please break it down into 2 tickets? the headline items are not related. thx!

### @github-actions - 2024-08-30T02:27:33Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-30T02:27:34Z @github-actions added the `stale` label
### @github-actions - 2024-09-14T02:26:35Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-14T02:26:36Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:25:55Z

#19489 set C · T4 · Grace · 2026-10-09: **confirm-close (obsolete).** The Docs app is gone from `apps/`; the Portal replaced it. Thank you @maxrahder.

- 2026-10-09T20:26:18Z @neo-opus-grace cross-referenced by #19489
### @neo-opus-grace - 2026-10-09T21:01:25Z

#19489 set C · correction · Grace · 2026-10-09: **Correction.** My earlier note here said the Docs app is gone. It is not: it lives at `docs/`, and the Portal embeds it at `/docs` (`apps/portal/view/Viewport.mjs`). Thank you @maxrahder. **Reopened, valid but later.** App classes still share the API tree (a collapsed `apps` branch, `buildScripts/docs/generateDocsJson.mjs`). The empty folders while filtering are a `tree.List#filter` defect, pinned on #3240.

- 2026-10-09T21:01:40Z @neo-opus-grace cross-referenced by #3239
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:52:45Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. Half of this shipped yesterday. #20244, merged on 11-04 and closing #3240, filters `tree.List` per item and collapses unmatched child folders, so empty folders no longer show while filtering. The other half still holds: app classes share the API tree as a collapsed `apps` branch built by `buildScripts/docs/generateDocsJson.mjs`.

**Verdict: holds**, narrowed to separating "apps" from the API tab.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


