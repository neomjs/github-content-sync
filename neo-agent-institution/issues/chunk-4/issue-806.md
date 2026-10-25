---
id: 806
title: 'Pin the engine to a neo dev commit that carries #20100'
state: CLOSED
labels:
  - bug
  - ai
  - dependencies
assignees:
  - neo-opus-ada
createdAt: '2026-10-25T01:20:07Z'
updatedAt: '2026-10-25T02:42:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/806'
author: neo-opus-ada
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
closedAt: '2026-10-25T02:42:02Z'
---
# Pin the engine to a neo dev commit that carries #20100

### Context
neo #20099 ("A visible document without frames strands Main's DOM write queue") is the installed Fleet Manager's freeze. neo #20100 fixed it and merged at 2026-10-25T00:59:42Z as `0c1a373918`.

Tuesday's walk (Oct 27) runs a candidate built from this repository's lock. The lock resolves `neo.mjs` to `d454606718` (`package-lock.json`, `node_modules/neo.mjs` → `resolved`), which predates the fix. Walk candidate 1 (Institution `f8bfa66`, Brain `a60f9ea7`, Engine `d454606718`) passed the packaged rehearsal, but it ships the engine without #20100.

### The Problem
`package.json` declares `"neo.mjs": "github:neomjs/neo#dev"`, and the lock fixes the commit. A packaged build (`harness` → `npm run dist`) installs from the lock. So a candidate carries the lock's engine commit, not dev's head, until the lock moves.

### The Architectural Reality
Delta census for `d454606718..0c1a373918`, read row by row (`git diff --stat`):
- `src/Main.mjs` and `src/main/addon/DockFlip.mjs`: #20100's fix, a frame-independent drain of Main's write queue (`renderQueueDamDelay`).
- `apps/portal/**` and `buildScripts/docs/seo/generate.mjs`: the portal and the SEO build (#20096, #20097). Neither ships into the cockpit.
- Everything else is tests.

There is no `package.json` change, no `resources/scss` change, and no engine version change (13.3.1). No Institution consumer moves. The Brain pin does not move: CI checks out Brain `dev` (`.github/workflows/ci.yml`).

### The Fix
A lock-only move. Run `npm install neo.mjs` so `#dev` resolves to a commit that contains `0c1a373918`, then walk the pin checklist:
1. Install for real: remove `node_modules/neo.mjs` first, then grep `renderQueueDamDelay` in the installed `src/Main.mjs`.
2. Run the Brain-bound suites as CI runs them, with the shared-engine link: unit, the e2e list, the Brain-bound e2e and the Neural Link battery.
3. Build the themes non-interactively and run the visual capture. Restamp only if the goldens pass unchanged. The stamp records the engine revision, and engine drift is a notice, not a failure (`buildScripts/checkVisualBaselines.mjs#engineDrift`).
4. If a spec breaks on a new reactive read, declare that read on its spy-host double.

### Acceptance Criteria
- [ ] `package-lock.json` resolves `neo.mjs` to a dev commit that contains `0c1a373918`, and the installed `src/Main.mjs` carries `renderQueueDamDelay`.
- [ ] Institution CI is green at the new pin.
- [ ] The engine move changes no visual result: every visual test gives the same result and the same pixel count at `d454606718` and at the new engine. The stamp is not restamped, because 6 of the 59 goldens already differ from dev's own render at both engines (captured as a defect note on 2026-10-25; re-capture is separate work).
- [ ] *(post-merge)* The re-cut walk candidate passes the packaged rehearsal, and its `organism-build-info.json` names the new engine revision.

### Out of Scope
- The reveal pop-out (neo #20103, open) and the follow-up that deletes `pinRevealedDetail` here. That is the next pin, after #20103 merges.
- Any Brain pin or Institution source change.

### Related
- neo #20099, neo #20100: the freeze and its fix.
- #516: row 5's installed walkthrough, which the walk candidate serves.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-25T01:19Z; no equivalent found. A2A claim sweep (latest 30 `lane-claim` messages) and a Memory Core sweep found no competing claim or prior decision.

Origin Session ID: bb7f4705-9365-478d-879f-5276a3839427
Retrieval Hint: "Institution engine pin move walk candidate render queue freeze"


## Timeline

- 2026-10-25T01:20:08Z @neo-opus-ada added the `bug` label
- 2026-10-25T01:20:08Z @neo-opus-ada added the `ai` label
- 2026-10-25T01:20:08Z @neo-opus-ada added the `dependencies` label
- 2026-10-25T01:20:16Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-25T01:33:50Z @neo-opus-ada cross-referenced by PR #807
- 2026-10-25T02:42:02Z @tobiu referenced in commit `84e86b5` - "build(deps): pin the engine to neo dev 0c1a373918, which carries #20100 (#806) (#807)

The lock resolved neo.mjs to d454606718, which predates neo #20100 (Main's DOM write queue
drains without frames). The engine delta to 0c1a373918 is src/Main.mjs and
src/main/addon/DockFlip.mjs plus portal, SEO-build and test paths: no package.json, scss or
version change, so no Institution consumer moves."
- 2026-10-25T02:42:02Z @tobiu closed this issue
- 2026-10-25T02:44:24Z @neo-opus-ada cross-referenced by #808

