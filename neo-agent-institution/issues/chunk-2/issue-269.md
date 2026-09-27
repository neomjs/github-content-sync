---
id: 269
title: 'Brain pin 4 (dev@c6c92c2, fleetGraphScene) + engine pin (dev@a50ae57ce8, GraphScene''s rejecting setScene)'
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-09-26T22:49:47Z'
updatedAt: '2026-09-27T08:34:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/269'
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
blocking:
  - '[x] 271 Observatory draws the bounded graph read with one canonical selection'
  - '[x] 258 Build a bounded Observatory scene with stable selection'
closedAt: '2026-09-27T08:34:15Z'
---
# Brain pin 4 (dev@c6c92c2, fleetGraphScene) + engine pin (dev@a50ae57ce8, GraphScene's rejecting setScene)

## Context

#258 (the bounded Observatory scene) consumes Brain #533's `fleetGraphScene` read through the fleet bridge. The Institution pins Brain `1ac9492` (Brain pin 3, `#228`), which predates Brain PR #545. Its contract's `FLEET_WIRE_METHODS` has no `fleetGraphScene`, so the bridge refuses the verb in every topology. The operator's local plane runs Brain `c6c92c2` (current `dev` head, containing #545, #551 and #559).

#258 also draws through `Neo.canvas.GraphScene`, and its AC-6 names the engine contract that refuses a scene without positions (neomjs/neo#19290, PR neomjs/neo#19291). The engine pin `2965d82` predates that PR.

## The Problem

Without the pins, #258's read cannot be wired or tested against the real contracts. Pin 3 set the precedent: a pin bump is its own small PR, reviewed and smoked separately, because it moves every contract import at once.

## The Architectural Reality

- The Brain pin appears in three places: `package.json` (`neo-agent-brain`), `package-lock.json` (the resolved git ref) and `.github/workflows/ci.yml` (the explicit Brain contract job's `ref`). The engine pin appears in two: `package.json` (`neo.mjs`) and `package-lock.json`.
- `harness/contentPolicy.mjs` allowlists the installed contract's modules, and `ContentPolicy.spec` parses the index's `export *` lines to prove each one is allowlisted. A new contract module fails there until allowlisted, which is what pin 3's packaged smoke caught.
- Brain between the two pins includes the healthcheck posture split (#557, `status` is the serving verdict) and the fleet wake-hook identity (#551). The cockpit reads the live plane's healthcheck regardless of the pin; the pin moves the contract, the local test Brain and the packaged organism.
- The engine between its two pins adds the rejecting `setScene`, the v13.2 dock, tab and draggable fixes, and two workstation popup fixes.

## The Fix

Move the Brain references to `c6c92c20857710676b3eb8566b49f59edb8ea7a8` and the engine references to `a50ae57ce82002251495c9892ac1bba95295634d`. Allowlist any new contract module, and fix whatever the unit, visual and NL suites surface against the new contracts. No feature code: the `fleetGraphScene` read itself is #258.

## Acceptance Criteria

- [ ] AC-1: `package.json`, `package-lock.json` and `ci.yml` name `c6c92c2`, and the installed contract lists `fleetGraphScene` in `FLEET_WIRE_METHODS`. `package.json` and `package-lock.json` name `a50ae57ce8`, and the installed `GraphScene#setScene` refuses a scene without positions.
- [ ] AC-2: unit (including `ContentPolicy.spec`), components and visual pass on the new pins. Every e2e and NL spec that fails on the new pins fails identically on the old ones.
- [ ] AC-3 (post-merge): a packaged smoke on the merged pins shows no content-policy 404 for a contract module (receipt under #7).

## Out of Scope

- Wiring `fleetGraphScene` into the cockpit (#258).

## Related

Blocks #258. Precedent: `#228` / `#233` (pin 3). Parent: #10.

Live latest-open sweep: the latest 20 open Institution issues at 2026-09-26T22:55Z hold no pin ticket (newest #267). A2A: no pin claim in the last hour. Memory Core: pin 3's session (Clio, 2026-09-26) documents the contract allowlist trap this AC-2 covers. The engine pin joined on 2026-09-26 after the Brain pin's PR opened: #258 needs both, and two pin PRs would conflict on adjacent `package.json` lines.

Origin Session ID: 6408fcd4-3571-4ec2-8009-b4dae5d18917
Retrieval Hint: "Brain pin 4 engine pin fleetGraphScene GraphScene setScene contract allowlist c6c92c2 a50ae57ce8"


## Timeline

- 2026-09-26T22:49:47Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-26T22:49:48Z @neo-opus-grace added the `enhancement` label
- 2026-09-26T22:49:48Z @neo-opus-grace added the `ai` label
- 2026-09-26T22:49:48Z @neo-opus-grace added the `build` label
- 2026-09-26T22:49:56Z @neo-opus-grace marked this issue as blocking #258
- 2026-09-26T22:57:13Z @neo-opus-grace cross-referenced by PR #270
- 2026-09-26T23:24:50Z @tobiu referenced in commit `bfa1dfa` - "chore(deps): engine pin → dev@a50ae57ce8, which carries GraphScene's rejecting setScene (#269)

#258's Observatory draws through Neo.canvas.GraphScene, and its AC-6 names the engine contract that refuses a scene without positions (neomjs/neo#19290, PR neomjs/neo#19291) instead of blanking the canvas. The previous pin 2965d82 predates it. The ten other engine commits since are the dock, tab and draggable fixes of the v13.2 blocker lane plus two workstation popup fixes."
- 2026-09-26T23:26:01Z @neo-opus-grace changed title from **Brain pin 4 — dev@c6c92c2 carries the fleetGraphScene wire method** to **Brain pin 4 (dev@c6c92c2, fleetGraphScene) + engine pin (dev@a50ae57ce8, GraphScene's rejecting setScene)**
- 2026-09-26T23:46:08Z @neo-opus-grace cross-referenced by #271
- 2026-09-26T23:46:17Z @neo-opus-grace marked this issue as blocking #271
- 2026-09-26T23:47:42Z @neo-opus-grace cross-referenced by PR #272
- 2026-09-27T08:11:53Z @tobiu referenced in commit `6590a3a` - "chore(deps): Brain pin 4 — dev@c6c92c2 carries the fleetGraphScene wire method (#269)

The Institution pinned Brain 1ac9492, which predates Brain #545; its
contract had no fleetGraphScene, so the fleet bridge refused the verb the
bounded Observatory scene (#258) reads. c6c92c2 is Brain dev's head and the
commit the local plane runs. The package, the lock and the CI Brain
checkout move together; the installed contract adds no module, so the
content policy's allowlist is unchanged."
- 2026-09-27T08:11:53Z @tobiu referenced in commit `799b0e4` - "chore(deps): engine pin → dev@a50ae57ce8, which carries GraphScene's rejecting setScene (#269)

#258's Observatory draws through Neo.canvas.GraphScene, and its AC-6 names the engine contract that refuses a scene without positions (neomjs/neo#19290, PR neomjs/neo#19291) instead of blanking the canvas. The previous pin 2965d82 predates it. The ten other engine commits since are the dock, tab and draggable fixes of the v13.2 blocker lane plus two workstation popup fixes."
- 2026-09-27T08:11:53Z @tobiu referenced in commit `4f44650` - "test(visual): re-stamp the baseline inputs on the new engine pin (#269)"
- 2026-09-27T08:31:27Z @neo-opus-grace cross-referenced by #277
- 2026-09-27T08:34:15Z @tobiu referenced in commit `dc171ce` - "Merge pull request #270 from neomjs/grace/269-brain-pin-4

chore(deps): Brain pin 4 (dev@c6c92c2, fleetGraphScene) + engine pin (dev@a50ae57ce8, GraphScene's rejecting setScene) (#269)"
- 2026-09-27T08:34:15Z @tobiu closed this issue
- 2026-09-27T08:46:46Z @neo-opus-grace cross-referenced by #278
- 2026-09-27T09:18:58Z @neo-opus-ada cross-referenced by #280

