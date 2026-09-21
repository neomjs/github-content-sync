---
id: 46
title: Retire the tracked MicroLoader fork — reference the engine loader directly
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-08-28T23:44:39Z'
updatedAt: '2026-08-29T09:55:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/46'
author: neo-fable-clio
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
closedAt: '2026-08-29T09:55:41Z'
---
# Retire the tracked MicroLoader fork — reference the engine loader directly

## Context

Cross-repo finding, relayed with verified anchors (Grace → Ada → this lane, A2A 2026-08-29): `src/MicroLoader.mjs` in this repo is a **tracked fork** of the engine's loader, committed 2026-08-26 from an already-stale source. Engine `src/MicroLoader.mjs` moved to the JSON-module import shape (`import(new URL('./neo-config.json', document.baseURI).href, {with: {type: 'json'}})`, neomjs/neo#13909); our copy still carries the older `fetch` + `self.Neo` shape. Grace places it in the same defect class as neomjs/devindex#10: a copied upstream file with a decay clock and no mechanism that notices drift.

## The Problem

Two loader generations diverge silently. The fork exists only to be referenced by three repo HTMLs — and the repo already proves the fork is unnecessary: `test/playwright/component/apps/empty-viewport/index.html` loads the ENGINE loader directly (`node_modules/neo.mjs/src/MicroLoader.mjs`) and has run the entire component battery through it. Config resolution is page-relative in both shapes (the engine's via `document.baseURI`), so a directly-referenced engine loader picks up each page's own `neo-config.json` unchanged.

## The Fix (the class fix: no copy, so nothing can drift)

1. Point the three fork consumers at the engine loader: `apps/agentos/index.html`, `apps/agentos/childapps/widget/index.html`, `docs/index.html`.
2. Delete `src/MicroLoader.mjs`.
3. Verify each page boots (config resolution is page-relative; the component testbed is the working precedent).

No drift detector is needed in THIS repo once the copy is gone — that is the durable answer to the class here; devindex#10 remains the devindex instance's own call.

## Acceptance Criteria

- [ ] The three HTMLs reference `node_modules/neo.mjs/src/MicroLoader.mjs`; `src/MicroLoader.mjs` is deleted; zero references remain.
- [ ] The agentos app + widget childapp + docs page boot in dev mode (loader resolves each page's `neo-config.json`).
- [ ] Full unit + component batteries green.

## Out of Scope

- devindex#10 (the sibling instance, Grace's repo call).
- Any dist/build pipeline change (no buildScript references the fork — grepped).

## Related

Engine shape: neomjs/neo#13909 · sibling class instance: neomjs/devindex#10 · finding provenance: Grace (via Ada's relayed A2A, anchors verified engine-side).

Live latest-open sweep: checked latest 10 open issues at 2026-08-29T00:00Z — no equivalent. A2A in-flight: the finders explicitly filed nothing ("yours to judge").

Origin Session ID: 41859592-b7ee-4bce-bee3-f25644d9003b

Authored by Clio (Fable 5, Claude Code). Session 41859592-b7ee-4bce-bee3-f25644d9003b.


## Timeline

- 2026-08-28T23:44:41Z @neo-fable-clio added the `bug` label
- 2026-08-28T23:44:41Z @neo-fable-clio added the `ai` label
- 2026-08-28T23:44:45Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-08-28T23:48:39Z @neo-fable-clio cross-referenced by PR #47
### @neo-fable-clio - 2026-08-29T09:55:40Z

Closed not-planned per operator ruling on PR #47: src/MicroLoader.mjs is the intentional Neo-workspaces customization seam (official scaffolder: neomjs/create-app tasks/createSrcFolder.mjs — default pulls Main + neo-config.json; client boot requirements like env.json belong exactly there). The stale-copy premise was wrong; a shape-sync inside the seam would be a scaffolder-convention question, never a deletion.

- 2026-08-29T09:55:41Z @neo-fable-clio closed this issue
- 2026-08-29T10:07:17Z @neo-opus-grace cross-referenced by #10

