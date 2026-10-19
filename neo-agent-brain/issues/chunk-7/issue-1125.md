---
id: 1125
title: Two unit arms read neo's Playwright fixture from the Brain tree
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T07:20:58Z'
updatedAt: '2026-10-19T02:51:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1125'
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
closedAt: '2026-10-19T02:51:45Z'
---
# Two unit arms read neo's Playwright fixture from the Brain tree

## Context

This is a #201 slice of the class #1108, #1110 and #1112 fixed: an engine-owned file read from the Brain tree. Two single reds, each `ENOENT` or `ERR_MODULE_NOT_FOUND` on neo's `test/playwright/fixtures.mjs`:

- `services/hostBarrelRuntimeReach.spec.mjs` › `the shared Playwright fixture survives the same denial — a consumer must not inherit the Brain` probes it;
- `services-resilient-load.spec.mjs` › `wires dock helpers through the whitebox Neural Link fixture` reads it.

A third red of the same class, `GuideToolParity.spec`'s `neural-link` arm, is not fixed here. Read from the installed engine, it fails for a real reason: neo's `NeuralLink.md` claims "59 operation IDs", and the registry has 60. The guide fix is neomjs/neo#19855, and the Brain arm follows once the pin includes it.

## The Architectural Reality

- Both arms check a cross-repo property: neo's shipped fixture against the Brain's denial loader and its dock service. The Brain owns its side. The engine's side ships in the installed engine, reachable through `import.meta.resolve('neo.mjs/…')`, because neo declares no `exports`.
- The engine fixture loads the Brain's Neural Link services only at runtime, from `NEO_AGENTOS_RUNTIME_ROOT`, so importing it alone must survive the cloud-plane denial.

## The Fix

1. `hostBarrelRuntimeReach`'s probe takes an absolute target, and its fixture arm targets the installed engine's `test/playwright/fixtures.mjs`.
2. `services-resilient-load` reads that same installed fixture, and its now-unused `repoRoot` goes.

## Acceptance Criteria

- [ ] **AC-1:** both specs pass at head. On dev each of the two arms fails.
- [ ] **AC-2:** neither arm reads a neo path from the Brain root.

## Out of Scope

- `GuideToolParity.spec`: neomjs/neo#19855 and its paired Brain follow-up.
- The other #201 singles.

## Related

- #201: the burn-down.
- #1108, #1110, #1112: the same class.
- neomjs/neo#19855: the guide's count.

Live latest-open sweep: the latest 20 open Brain issues, rechecked at 2026-10-18T07:20Z; no equivalent found.
A2A claim sweep: the latest inbox messages; no claim.
Own-assignment sweep: #201, #1117, #1119, #1121, #1123.
Structure map: N/A; this changes two specs.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "hostBarrelRuntimeReach and services-resilient-load read neo's fixtures.mjs from the installed engine via import.meta.resolve"


## Timeline

- 2026-10-18T07:20:59Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T07:21:00Z @neo-opus-ada added the `bug` label
- 2026-10-18T07:21:00Z @neo-opus-ada added the `ai` label
- 2026-10-18T07:21:00Z @neo-opus-ada added the `testing` label
- 2026-10-18T07:23:08Z @neo-opus-ada cross-referenced by #19855
- 2026-10-18T07:23:23Z @neo-opus-ada changed title from **Three unit arms read neo's Neural Link guide and Playwright fixture from the Brain tree** to **Two unit arms read neo's Playwright fixture from the Brain tree**
- 2026-10-18T07:24:07Z @neo-opus-ada cross-referenced by PR #1126
- 2026-10-18T07:24:28Z @neo-opus-ada cross-referenced by #201
- 2026-10-18T07:27:25Z @neo-opus-ada cross-referenced by #1127
- 2026-10-18T07:33:14Z @neo-opus-ada cross-referenced by #1129
- 2026-10-18T07:58:38Z @neo-opus-ada cross-referenced by #1131
- 2026-10-18T22:45:41Z @neo-opus-ada referenced in commit `80be824` - "chore(test): merge dev, picking up #1134's wake-daemon race fix that reddened this PR's unit job (#1125)"
- 2026-10-18T23:42:28Z @neo-opus-ada referenced in commit `1b9a427` - "chore(test): merge dev, picking up #1159 and #1163, the two cross-spec leaks that reddened this PR's unit job (#1125)"
- 2026-10-19T02:51:45Z @tobiu referenced in commit `7cc72ea` - "fix(test): the runtime-reach and dock-wiring arms read neo's fixture from the installed engine (#1125) (#1126)

Both arms looked for test/playwright/fixtures.mjs in the Brain checkout. The installed engine ships it, and the fixture reaches the Brain's Neural Link services only at runtime through NEO_AGENTOS_RUNTIME_ROOT, so the denial probe now exercises the real shipped fixture."
- 2026-10-19T02:51:46Z @tobiu closed this issue

