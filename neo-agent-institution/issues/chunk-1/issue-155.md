---
id: 155
title: The marked postinstall bridge has met its own retirement rule
state: CLOSED
labels:
  - enhancement
  - ai
  - build
  - tech-debt
assignees:
  - neo-fable-clio
createdAt: '2026-09-18T13:40:12Z'
updatedAt: '2026-09-18T14:15:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/155'
author: neo-fable-clio
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
closedAt: '2026-09-18T14:15:28Z'
---
# The marked postinstall bridge has met its own retirement rule

## Context

@neo-opus-ada's non-blocking note on PR #154 (review 5248362768): `buildScripts/linkMarkedForWorkerScope.mjs` names its own retirement condition, and the pinned engine meets it.

## The Problem

The script is a `postinstall` bridge: it placed `marked` under `node_modules/neo.mjs/node_modules/` because the engine reached it through a specifier that left the package. Its docblock ends: *"The moment the engine ships `dist/marked.mjs`, the condition below goes false and this becomes a no-op that announces why … Delete it once the engine version in `package.json` carries the fix."*

Measured at the pin (engine dev@70c2c94618, 2026-09-18) — corrected while building the fix:
- **a github-SHA install of the engine has no `dist/` at all** (`rm -rf node_modules/neo.mjs && npm install` → `ls node_modules/neo.mjs/dist`: no such directory). The `dist/marked.mjs` this ticket first cited (47,913 B) was written by the filer's own `build-all` runs, so the script's sunset condition is false on every fresh clone and in CI — there it still copies `marked` to the nested path;
- that copy serves nothing: the engine's two Markdown modules import `../../dist/marked.mjs` now, and
- `grep -rlE "node_modules/marked"` over the engine's `src apps examples docs` → 0 files (the last three went with neomjs/neo#18851);
- the script's only reference is `package.json`'s `postinstall`.

- **this workspace has no Markdown consumer**: no file under `apps` or `docs` imports `component/Markdown.mjs` or `app/content/Component.mjs` (the docblock's "the entire Learn section" left with the repo split).

So every `npm install` either copies 476 KB nothing imports (fresh clone, CI) or prints that it is obsolete (after a `build-all`).

## The Architectural Reality

- `package.json:39` — `"postinstall": "node ./buildScripts/linkMarkedForWorkerScope.mjs && neo-agent-skills-materialize"`.
- `buildScripts/linkMarkedForWorkerScope.mjs` — the bridge; nothing else imports or names it.

## The Fix

Delete the script and take it out of `postinstall` (`neo-agent-skills-materialize` stays). Check whether `marked` is still a direct dependency this workspace needs for anything else (the engine's `build-all` resolves some browser dependencies from the workspace) before touching `dependencies`.

## Acceptance Criteria

- [ ] `buildScripts/linkMarkedForWorkerScope.mjs` is gone and no file names it.
- [ ] `npm install` from a clean `node_modules/neo.mjs` ends green, skills materialized, and leaves no `node_modules/neo.mjs/node_modules/marked`.
- [ ] No file under `apps` or `docs` imports the engine's Markdown or content component (grep receipt), and the cockpit boots in dev mode from the clean install.

## Out of Scope

Removing `marked` from `dependencies` unless the check above shows nothing needs it — then it is one more line in the same PR, said in its Deltas.

## Related

PR #154 (where the note was made), neomjs/neo#18851, neomjs/neo#18849.

Decision Record impact: none.

Live latest-open sweep: all open issues read at 2026-09-18T13:39:42Z, plus `gh issue list --state all --search 'linkMarkedForWorkerScope OR "marked" postinstall'` → no hit. A2A in-flight sweep: no claim on this surface; the origin is the reviewer's note.
MC sweep: `linkMarkedForWorkerScope postinstall bridge places marked under node_modules/neo.mjs until the engine ships dist/marked.mjs, retire the script`, 2 results — the bridge was recorded as "sunset-bound when Neo bundles dist/marked.mjs" at its birth (2026-08-20); no contrary decision.
Own-assignment sweep: #127, #128, #129, #10 — none overlapping.

Origin Session ID: 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59
Retrieval Hint: "linkMarkedForWorkerScope retirement dist/marked.mjs postinstall bridge"


## Timeline

- 2026-09-18T13:40:13Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-18T13:40:14Z @neo-fable-clio added the `enhancement` label
- 2026-09-18T13:40:15Z @neo-fable-clio added the `ai` label
- 2026-09-18T13:40:15Z @neo-fable-clio added the `build` label
- 2026-09-18T13:40:15Z @neo-fable-clio added the `tech-debt` label
- 2026-09-18T13:44:06Z @neo-fable-clio cross-referenced by PR #156
- 2026-09-18T14:15:28Z @tobiu referenced in commit `603d5ae` - "Merge pull request #156 from neomjs/agent/155-retire-marked-bridge

chore(build): the marked postinstall bridge is retired (#155)"
- 2026-09-18T14:15:29Z @tobiu closed this issue

