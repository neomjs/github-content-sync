---
id: 252
title: The Observatory renders through Neo.canvas.GraphScene; its app-local WebGL2 machinery retires
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-26T10:45:52Z'
updatedAt: '2026-09-26T20:32:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/252'
author: neo-opus-ada
commentsCount: 0
parentIssue: 10034
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 19261 Extract a WebGL2 graph-scene renderer into src/canvas as Neo.canvas.GraphScene'
blocking: []
closedAt: '2026-09-26T20:32:12Z'
---
# The Observatory renders through Neo.canvas.GraphScene; its app-local WebGL2 machinery retires

## Context

neomjs/neo#19261 extracts the generic WebGL2 graph-scene machinery (the point-and-line program, the orbit camera with `fit`, drag and wheel, `pick`, frames on demand, `getStats`, and the host's pixel ratio) into the engine as `Neo.canvas.GraphScene`. That ticket closes only once this twin is filed, so the app-local copy has a named sunset.

## The Problem

`apps/agentos/canvas/Observatory.mjs` carries its own copy of that machinery beside what is FM-specific (the palette via `canvas/fmPalette.mjs`, the currency tones, the scene derived from the Golden Path envelope). Every graph follow-up on neomjs/neo#10034 (LOD over the Brain's scene feed, the team lens, node selection, gravity wells) would otherwise extend app code the engine cannot test. The copy also assumes `dpr: 2` because the canvas worker is told no pixel ratio.

## The Architectural Reality

- `apps/agentos/canvas/Observatory.mjs`: `extends Neo.canvas.Base`, `contextType: 'webgl2'`, `dpr: 2`, `getStats`, `pick({x, y})`, `fit()`, the orbit camera.
- `package.json`: `neo.mjs` is pinned by commit (`github:neomjs/neo#87ac80a…`), so the engine class is importable only after a pin bump to a commit that carries it.
- The consumers of the renderer: `view/fleet/goldenpath/ObservatoryCanvas.mjs` (the `SharedCanvas` host), `FleetObservatoryNL.spec.mjs`, the `observatory-pane-*` goldens.

## The Fix

1. Bump the `neo.mjs` pin (and CI's engine ref) to a dev commit carrying `Neo.canvas.GraphScene`.
2. `AgentOS.canvas.Observatory extends Neo.canvas.GraphScene`, keeping only the palette, the currency tones and the scene derivation. It inks the layout scene into the engine's flat scene, as plain arrays, and hands it to the engine's `setScene`. The engine's normalizer turns them into typed arrays before the upload. *(Corrected after review RA-2, Euclid on PR #256: this line first said "hands the engine typed arrays". Sending typed arrays from the App Worker only pays off at H3-e's scale, not for this route of a few dozen nodes.)*
3. Delete the duplicated program, camera, pick and stats code from the app, and the assumed `dpr`.

## Contract Ledger

*(Backfilled after review RA-2 on PR #256.)*

| Surface | Authority | Behaviour | Fallback | Evidence |
|---|---|---|---|---|
| `Observatory.setScene({scene, windowId})` (remote) | `ObservatoryCanvas#pushScene` | takes the layout scene `{currency, empty, nodes, edges, route}` and inks it into the engine's flat scene: route items in the signal (current) or dim (withheld), citations dim, the route as one path | an empty or missing scene clears the surface. A scene the engine refuses throws in the canvas worker, and the drawn scene stays with its `sourceScene`. The App Worker's call rejects only from an engine pin that carries neo `#19291`; this PR's pin, `2965d82fca`, predates it, so there the call gets no reply | unit arms |
| `Observatory.pick({x, y})` (remote) | `ObservatoryCanvas#onMouseMove` → `reportHover` | the node itself, `{id, kind, label, rank, score}`, mapped from the engine's index | no scene or no surface size → `null` | unit arm |
| `Observatory.getStats()` (remote) | `ObservatoryCanvas#readStats`, `FleetObservatoryNL` | the engine's stats, with the pane's `counts: {nodes, edges, route}` and `currency` | an empty scene counts zeros; no scene counts `null` | unit arm, NL spec |
| a theme change | `Neo.canvas.Base#afterSetTheme` → `updateResources` | inks the drawn scene again in the new palette | no scene → one frame | unit arm |

## Acceptance Criteria

- [ ] AC-1 `Observatory` extends `Neo.canvas.GraphScene`; a grep of `apps/agentos/canvas/Observatory.mjs` finds no shader source, VAO upload or camera math of its own.
- [ ] AC-2 `FleetObservatoryNL` is green (node counts, idle frames stable, orbit and wheel), and the `observatory-pane-*` goldens hold or are re-captured with the before/after in the PR.
- [ ] AC-3 The drawing buffer follows the host's pixel ratio, not a constant.

## Out of Scope

- The engine primitive itself (neomjs/neo#19261).
- Moving the Observatory to a left-rail view (#243).

## Related

neomjs/neo#19261 (blocks this) · neomjs/neo#10034 (the epic) · #230 (the slice) · #243

unowned-rationale: blocked by neomjs/neo#19261 and a pin bump; claimable once that merges (the engine leaf's author, or the Observatory's).

Live latest-open sweep: the latest 20 open issues of this repository at 2026-09-26T10:45Z — no equivalent. A2A in-flight sweep: none. Memory Core sweep: Grace's Stage-4 note on neomjs/neo#10034 asks for exactly this sunset AC. Own-assignment sweep: none covers it.

Origin Session ID: 1b945fcf-1142-475f-8007-ac18d51c069a
Retrieval Hint: `query_raw_memories("Observatory extends GraphScene pin bump app-local renderer sunset")`



## Timeline

- 2026-09-26T10:45:53Z @neo-opus-ada added the `enhancement` label
- 2026-09-26T10:45:54Z @neo-opus-ada added the `agent-os` label
- 2026-09-26T10:45:54Z @neo-opus-ada added the `ai` label
- 2026-09-26T10:45:54Z @neo-opus-ada added parent issue #10034
- 2026-09-26T10:45:55Z @neo-opus-ada marked this issue as being blocked by #19261
- 2026-09-26T12:22:21Z @neo-opus-ada cross-referenced by PR #19274
- 2026-09-26T18:48:41Z @neo-gpt cross-referenced by PR #19291
- 2026-09-26T18:59:52Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-26T19:17:15Z @neo-opus-ada cross-referenced by PR #256
- 2026-09-26T19:43:25Z @neo-gpt cross-referenced by #258
- 2026-09-26T19:58:43Z @neo-opus-ada referenced in commit `d325b98` - "fix(agentos): the Observatory's scene changes only once the engine took the inked one (#252)"
- 2026-09-26T20:09:18Z @neo-opus-ada referenced in commit `255a5c6` - "test(visual): the stamp follows the Observatory's setScene order, goldens re-verified unchanged (#252)"
- 2026-09-26T20:21:09Z @neo-opus-ada referenced in commit `5177735` - "test(visual): the stamp follows the Observatory's setScene order, goldens re-verified unchanged (#252)"
- 2026-09-26T20:24:01Z @neo-opus-ada referenced in commit `aa40a5e` - "fix(agentos): the Observatory renders through Neo.canvas.GraphScene, its buffer at the host's pixel ratio (#252)

The app-local WebGL2 copy (program, camera, pick, upload, stats) retires:
AgentOS.canvas.Observatory extends the engine's GraphScene and keeps only
the ink - the palette, the currency tones and the layout scene marshalled
into the engine's flat scene. pick maps the engine's index back to the
node, getStats keeps the pane's counts and currency, a theme change inks
the scene again, and the fixed dpr: 2 gives way to the size message's
devicePixelRatio. Sizes move to the engine's unit (pixels at the camera
distance) so the pane keeps its look. The engine pin moves to 2965d82fca,
the GraphScene merge."
- 2026-09-26T20:24:01Z @neo-opus-ada referenced in commit `7508201` - "fix(agentos): the Observatory's scene changes only once the engine took the inked one (#252)"
- 2026-09-26T20:24:01Z @neo-opus-ada referenced in commit `67e8be8` - "test(visual): the stamp follows the Observatory's setScene order, goldens re-verified unchanged (#252)"
- 2026-09-26T20:24:01Z @neo-opus-ada referenced in commit `87abd6d` - "test(visual): the stamp covers #251's goldens beside the Observatory's after the rebase (#252)"
- 2026-09-26T20:32:12Z @tobiu referenced in commit `975d16d` - "Merge pull request #256 from neomjs/ada/252-observatory-graphscene

fix(agentos): the Observatory renders through Neo.canvas.GraphScene, its buffer at the host's pixel ratio (#252)"
- 2026-09-26T20:32:12Z @tobiu closed this issue

