---
id: 404
title: Five NL arms fail on dev for reasons inside the specs
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T18:15:18Z'
updatedAt: '2026-10-01T19:20:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/404'
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
closedAt: '2026-10-01T19:20:59Z'
---
# Five NL arms fail on dev for reasons inside the specs

## Context

Measured 2026-10-01 while verifying #402 (Brain pin 7). The full NL battery, with Brain `2e46930` bound, ran 43 passed, 7 failed, 2 did not run. Two of the reds are stale goldens that #401 re-captures. The other five arms fail inside their own specs. No Brain version changes their outcome, and no CI job runs the NL battery, so each one has turned into a "known red" that every peer's local run carries (defect-notes `a890b7118f273be4` and Fable's 14:31Z note).

## The Problem

A battery that is never fully green trains everyone to skim its reds, and a real regression hides among them, the way #287's stale goldens did.

## The Architectural Reality

- **`FleetLandingNL`, both arms** ("landing the sample renders eleven cards and six events…", "a spec's own rows land through the same seam…"). `readProvider` takes `stateProvider` from the cockpit Container and reads `stores.fleetRoster`. #244 (`0a929b57d4`) moved the roster store up to the Viewport provider (`apps/agentos/view/Viewport.mjs:128`), so the read is `undefined` and the arm throws reading `count`.
- **`FleetFirstLaunchNL`** ("a seat added in the cockpit starts from…") **and `AddAgentJourneyNL`** ("bootstrap CTA → S5 zone → readback-confirmed…"). Both fill `.fm-add-agent-form input[type="text"]`, which now resolves to two inputs, `githubUsername` and `repoSlug`, so Playwright's strict mode refuses. `AddAgentJourneyNL` also defines an agent without a credential.
- **`FleetCardNameSlotNL`** ("a live-registry resident renders the folded name…"). Its setup calls `FleetRegistryService.defineAgent` without a credential. Since neomjs/neo-agent-brain#576/#577 the registry refuses that: "'credential' is required — every agent holds its GitHub PAT."

## The Fix

Each fix stays in its spec:
- read the stores from the provider that owns them;
- select the add-agent fields by name;
- give every test-defined agent a fixture credential.

No product code changes.

## Acceptance Criteria

- [ ] AC-1: the five arms pass in the NL battery on Darwin, with the pinned Brain bound.
- [ ] AC-2: the diff touches specs (and test fixtures) only, and each selector names the field it means.
- [ ] AC-3: with #401's goldens in place, the full NL battery passes on this head.

## Out of Scope

- A CI job for the NL battery, the class that let these go stale.
- The two stale goldens (#401).

## Related

#244 · #401 · #402 · #287 (the same class) · neomjs/neo-agent-brain#576

## Sweeps

- Live latest-open sweep: the latest 20 open Institution issues at 2026-10-01T18:15:01Z, plus a search for each spec's name. Only closed hits, none for these failures.
- A2A: the defect-notes above. No claim.
- Memory Core: covered by the #287 sweep; there is no prior decision on these specs.
- Own assignments: #399 and #402 (open, not overlapping).

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a

Authored by Ada (Claude Opus 5.5, Claude Code).

## Timeline

- 2026-10-01T18:15:18Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T18:15:20Z @neo-opus-ada added the `bug` label
- 2026-10-01T18:15:20Z @neo-opus-ada added the `ai` label
- 2026-10-01T18:15:20Z @neo-opus-ada added the `testing` label
- 2026-10-01T18:25:17Z @neo-opus-ada cross-referenced by PR #406
- 2026-10-01T18:38:41Z @neo-opus-ada cross-referenced by PR #403
- 2026-10-01T18:41:35Z @neo-opus-ada cross-referenced by #407
- 2026-10-01T19:08:21Z @neo-opus-ada cross-referenced by #408
- 2026-10-01T19:20:59Z @tobiu referenced in commit `a26b8ae` - "test(agentos): five NL arms read the providers, fields and credentials the app now has (#404) (#406)"
- 2026-10-01T19:21:00Z @tobiu closed this issue

