---
id: 305
title: 'The Observatory head counts the read''s edges, not the ones it draws'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-09-27T16:20:01Z'
updatedAt: '2026-09-27T16:42:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/305'
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
closedAt: '2026-09-27T16:42:10Z'
---
# The Observatory head counts the read's edges, not the ones it draws

## Context
The Observatory head counts what the read's envelope carries, not what the pane draws. `GraphSceneEnvelope.describe` takes its "N nodes · M edges" from `envelope.scene`, while `ObservatorySceneLayout.fromGraphScene` keeps an edge only when both ends are node ids of the same read, and keeps a repeated edge once. A producer that sends index pairs, or edges to nodes a budget cut dropped, gets a drawing without those edges under a head that still counts them all. Found in the #291 ↔ Brain #583 seam review (@neo-preview, https://github.com/neomjs/neo-agent-brain/pull/587#issuecomment-5857576777). With the live read at about 244k nodes, the head is the one number a reader checks.

## The Problem
A count that survives the transformation that should reduce it looks right while it is wrong, and a pane that counts wrong is worse than one that visibly fails. This is the mirror image of the producer's `unlinked` count on Brain #587, which a trim never decrements.

## The Architectural Reality
- `apps/agentos/util/GraphSceneEnvelope.mjs` `describe(envelope, formatStamp)`: `holds` counts `scene.nodes.length` and `scene.edges.length`.
- `apps/agentos/util/ObservatorySceneLayout.mjs` `fromGraphScene`: an edge needs `at.has(from) && at.has(to)` and is kept once per `from`/`to`/`type`; a node needs its id.
- `ObservatoryContainer.updateLine` writes `describe(me.envelope, …)` into the head and holds the layout scene as `me.scene`.

## The Fix
`describe` takes the counts the pane drew as an optional third argument. The head counts what is drawn, and it names what arrived but is not drawn ("3 edges not drawn"). Without the argument, `describe` counts the envelope as it does today. `updateLine` passes the layout scene's node and edge counts.

## Acceptance Criteria
- [ ] AC-1: A read whose edges all reach nodes of the read reads as today.
- [ ] AC-2: A read with an edge to a node it does not hold, or a repeated edge, draws without it, and the head counts the drawn edges and says how many were not drawn (unit on `describe` and on the pane).

## Out of Scope
- Refusing such a read, or changing how the layout admits edges.
- The producer's own counts (Brain #587).

## Related
- #291 (the Observatory), #288, Brain #583 and #587 (the live read and its contract).

Live latest-open sweep: checked the latest 20 open issues at 16:19Z and searched "not drawn / dropped edge / edge count"; no equivalent. A2A: no claim on this surface in the last hour. Own assignments: none on this surface.

Origin Session ID: 0dc6daad-2744-44c9-91cb-38d82e9e82e6


## Timeline

- 2026-09-27T16:20:02Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-27T16:20:03Z @neo-opus-grace added the `bug` label
- 2026-09-27T16:20:03Z @neo-opus-grace added the `ai` label
- 2026-09-27T16:23:09Z @neo-opus-grace cross-referenced by PR #306
- 2026-09-27T16:29:29Z @tobiu referenced in commit `ff6d551` - "fix(agentos): the Observatory head counts what it draws and names what the read carried beyond it (#305)"
- 2026-09-27T16:42:10Z @tobiu referenced in commit `0e1250b` - "Merge pull request #306 from neomjs/grace/305-head-counts-drawn

fix(agentos): the Observatory head counts what it draws and names what the read carried beyond it (#305)"
- 2026-09-27T16:42:10Z @tobiu closed this issue

