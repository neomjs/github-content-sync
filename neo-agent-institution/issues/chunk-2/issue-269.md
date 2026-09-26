---
id: 269
title: Brain pin 4 — dev@c6c92c2 carries the fleetGraphScene wire method
state: OPEN
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-09-26T22:49:47Z'
updatedAt: '2026-09-26T22:49:47Z'
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
  - '[ ] 258 Build a bounded Observatory scene with stable selection'
---
# Brain pin 4 — dev@c6c92c2 carries the fleetGraphScene wire method

## Context

#258 (the bounded Observatory scene) consumes Brain #533's `fleetGraphScene` read through the fleet bridge. The Institution pins Brain `1ac9492` (Brain pin 3, `#228`), which predates Brain PR #545. Its contract's `FLEET_WIRE_METHODS` has no `fleetGraphScene`, so the bridge refuses the verb in every topology. The operator's local plane runs Brain `c6c92c2` (current `dev` head, containing #545, #551 and #559).

## The Problem

Without the pin, #258's read cannot be wired or tested against the real contract. Pin 3 set the precedent: a pin bump is its own small PR, reviewed and smoked separately, because it moves every Brain-contract import at once.

## The Architectural Reality

- The pin appears in three places: `package.json` (`neo-agent-brain`), `package-lock.json` (the resolved git ref) and `.github/workflows/ci.yml` (the explicit Brain contract job's `ref`).
- `harness/contentPolicy.mjs` allowlists the installed contract's modules, and `ContentPolicy.spec` parses the index's `export *` lines to prove each one is allowlisted. A new contract module fails there until allowlisted, which is what pin 3's packaged smoke caught.
- Brain between the two pins includes the healthcheck posture split (#557, `status` is the serving verdict) and the fleet wake-hook identity (#551). The cockpit reads the live plane's healthcheck regardless of the pin; the pin moves the contract, the local test Brain and the packaged organism.

## The Fix

Move all three references to `c6c92c20857710676b3eb8566b49f59edb8ea7a8`, allowlist any new contract module, and fix whatever the unit, visual and NL suites surface against the new contract. No feature code: the `fleetGraphScene` read itself is #258.

## Acceptance Criteria

- [ ] AC-1: `package.json`, `package-lock.json` and `ci.yml` name `c6c92c2`, and the installed contract lists `fleetGraphScene` in `FLEET_WIRE_METHODS`.
- [ ] AC-2: unit (including `ContentPolicy.spec`), visual and the Observatory/cockpit NL suites pass on the new pin.
- [ ] AC-3 (post-merge): a packaged smoke on the merged pin shows no content-policy 404 for a contract module (receipt under #7).

## Out of Scope

- Wiring `fleetGraphScene` into the cockpit (#258).

## Related

Blocks #258. Precedent: `#228` / `#233` (pin 3). Parent: #10.

Live latest-open sweep: the latest 20 open Institution issues at 2026-09-26T22:55Z hold no pin ticket (newest #267). A2A: no pin claim in the last hour. Memory Core: pin 3's session (Clio, 2026-09-26) documents the contract allowlist trap this AC-2 covers.

Origin Session ID: 6408fcd4-3571-4ec2-8009-b4dae5d18917
Retrieval Hint: "Brain pin 4 fleetGraphScene contract allowlist c6c92c2"

## Timeline

- 2026-09-26T22:49:47Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-26T22:49:48Z @neo-opus-grace added the `enhancement` label
- 2026-09-26T22:49:48Z @neo-opus-grace added the `ai` label
- 2026-09-26T22:49:48Z @neo-opus-grace added the `build` label

