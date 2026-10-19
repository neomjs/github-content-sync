---
id: 758
title: 'The FM window stays black: the shell refuses seatLifecycle.mjs'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - regression
assignees:
  - neo-opus-vega
createdAt: '2026-10-19T19:32:12Z'
updatedAt: '2026-10-19T19:59:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/758'
author: neo-opus-vega
commentsCount: 0
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-19T19:59:28Z'
---
# The FM window stays black: the shell refuses seatLifecycle.mjs

## Context

**Observed 2026-10-19 by the operator, shortly after 19:15Z.** The FM candidate was installed at 19:14:41Z: Brain `0fa6deb8`, Engine `fbb213d4`, Product `764c0858`. Its window then shows only a black background: no shell and no roster. The Fleet child booted normally, and its transport was listening at 19:15:35Z.

## The Problem

Brain #1202 (`2a69eabd`) added `src/fleet/contract/seatLifecycle.mjs` and re-exports it from the contract index (`index.mjs:14`, `export * from './seatLifecycle.mjs'`). The installed bundle carries the file. But the shell's `ALLOWED_EXACT_PATHS` (`harness/contentPolicy.mjs:35`) lists only the seven earlier contract modules, so the protocol handler refuses it. Five renderer utilities import the contract: `SeatModel`, `SourceHealth`, `HarnessChoice`, `ActivityObserver` and `DeploymentStateRead`. Because one of the index's re-exports cannot load, the module graph fails and the app never renders.

The Institution's own guard catches exactly this. `test/playwright/unit/harness/ContentPolicy.spec.mjs:183`, "every module the installed Brain contract index re-exports is allowlisted", fails with `seatLifecycle.mjs is allowlisted` once `resolve-org-dev` takes Brain dev. That is the red on #757 at `c2ec4227` (1 failed, 2007 passed).

This candidate was built locally from resolved dev, and that suite never ran on it. The pre-install rehearsal (#736) is windowless, so it never loads the renderer.

## The Architectural Reality

- `ALLOWED_EXACT_PATHS` is the one allowlist for what the renderer may load, and the packaging pipeline derives its bundle manifest from it (`contentPolicy.mjs:33–34`).
- Contract modules are pinned by exact path on purpose: widening the contract has to be a deliberate line here.

Design authority: `harness/contentPolicy.mjs:34` — *"one authority for 'what the renderer may load' and 'what the artifact must carry'"*. The contract index the renderer loads re-exports `seatLifecycle.mjs`, so the renderer must be allowed to load it.

## The Fix

Add `'/node_modules/neo-agent-brain/src/fleet/contract/seatLifecycle.mjs'` to `ALLOWED_EXACT_PATHS`, in its sorted place.

## Acceptance Criteria

- [ ] AC-1 (unit): `ContentPolicy.spec.mjs` › "every module the installed Brain contract index re-exports is allowlisted" passes against resolved Brain dev. It is red on `dev` today.
- [ ] AC-2 [L3-deferred — operator handoff needed]: an FM candidate built with this fix shows the shell and roster after install.

## Out of Scope

- A pre-install gate that runs the content-policy suite, or loads the renderer once, against the resolved candidate. Its absence is why this reached an install; that is a follow-up.
- #749's roster reading of the new lifecycle states.

## Related

neomjs/neo-agent-brain#1202 (the new module), #749, #757 (red on the same guard), #736 (the windowless rehearsal), #732 (epic).

Live latest-open sweep: latest 10 open Institution issues at 2026-10-19T19:31Z, plus every open Institution PR's file list; none touches `contentPolicy.mjs`. A2A sweep: last 12 messages, no claim on this scope. MC sweep on the problem's nouns: 5 results, no prior decision on this allowlist. Own-assignment sweep: #733, #732, #651, #649, #485, none overlapping. Structure map: N/A, no new file; `harness/contentPolicy.mjs` owns the list.

Origin Session ID: e992c365-0539-4b95-b42c-bf0dffebfdbc
Retrieval Hint: "FM window black renderer content policy ALLOWED_EXACT_PATHS seatLifecycle contract index re-export"

Authored by Vega (Opus 5.5, Claude Code) 🌿

## Timeline

- 2026-10-19T19:32:12Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-19T19:32:14Z @neo-opus-vega added the `bug` label
- 2026-10-19T19:32:14Z @neo-opus-vega added the `agent-os` label
- 2026-10-19T19:32:14Z @neo-opus-vega added the `ai` label
- 2026-10-19T19:32:15Z @neo-opus-vega added the `regression` label
- 2026-10-19T19:32:26Z @neo-opus-vega added parent issue #732
- 2026-10-19T19:34:19Z @neo-opus-vega cross-referenced by PR #759
- 2026-10-19T19:59:28Z @tobiu referenced in commit `be8ab1b` - "fix(harness): the shell serves the Brain's seatLifecycle contract module (#758) (#759)"
- 2026-10-19T19:59:29Z @tobiu closed this issue

