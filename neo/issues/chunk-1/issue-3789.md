---
id: 3789
title: 'cross reference: import maps for the worker scope'
state: OPEN
labels:
  - enhancement
  - no auto close
assignees: []
createdAt: '2023-01-05T15:11:51Z'
updatedAt: '2026-11-05T10:30:48Z'
githubUrl: 'https://github.com/neomjs/neo/issues/3789'
author: tobiu
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy: []
blocking: []
---
# cross reference: import maps for the worker scope

this feature is super important for the neo framework, so we should keep track on the progress:
https://github.com/WICG/import-maps/issues/2

## Timeline

- 2023-01-05T15:11:51Z @tobiu added the `enhancement` label
- 2024-08-27T20:45:09Z @tobiu added the `no auto close` label
### @TomDakan - 2025-08-02T03:15:44Z

It looks like this repo has been shutdown in favor of the whatwg/html repo. This pull request seems to be taking over from issue:
[QUARANTINED_URL: github.com]

- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-julia - 2026-11-05T10:30:48Z

### Backlog Premise Read (`2026-11-05` · `neomjs/neo#15000`)

- **Verdict:** **holds** (external browser/spec dependency)
- **Premise check against `dev`:**
  - `src/worker/Manager.mjs:127-143` still branches on `useMjsFiles` (`Neo.config.environment === 'development' && !Neo.config.useThreadAddons`) between raw `.mjs` worker entrypoints (`App.mjs`, `Canvas.mjs`, `Data.mjs`, `Task.mjs`, `VDom.mjs`) and bundled worker chunks (`appworker.js`, `canvasworker.js`, `dataworker.js`, `taskworker.js`, `vdomworker.js`), instantiated at `src/worker/Manager.mjs:271` (`new cls(filePath, {name, type: 'module'})`).
  - Upstream `WICG/import-maps#2` was folded into `whatwg/html#10858` (import maps in `Worker` / `SharedWorker` / `ServiceWorker` scopes), which has not yet shipped across browser engines, so `dist/esm` and `dist/production` still require bundled worker entrypoints whenever bare-specifier main-thread addons are imported inside workers.

🏛️ **Julia** (`@neo-gemini-julia` · Gemini 4 Argon · Antigravity)


