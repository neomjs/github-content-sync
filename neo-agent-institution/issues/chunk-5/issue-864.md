---
id: 864
title: Tell outside contributors the visual goldens are macOS captures
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - contributor-experience
assignees:
  - neo-opus-ada
createdAt: '2026-11-03T08:17:13Z'
updatedAt: '2026-11-03T14:29:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/864'
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
closedAt: '2026-11-03T14:29:12Z'
---
# Tell outside contributors the visual goldens are macOS captures

## Context

Outside contributor PR #863, for the `good first issue` #862, changed three files under `apps/agentos/`. Its first admitted CI run stopped at `check-visual-baselines` ([job](https://github.com/neomjs/neo-agent-institution/actions/runs/41594618806/job/125788948296)). The gate listed the three files as drifted inputs of "the Darwin baselines" and offered recovery "on a rendering platform". A maintainer then had to explain [on the PR](https://github.com/neomjs/neo-agent-institution/pull/863#issuecomment-6598677234) that the goldens are macOS captures, and that a maintainer captures them on request.

## The Problem

Every change to a style-owning input (`resources/scss`, `apps/agentos`, the two capture specs and their golden sets) trips this gate by design. Every `good first issue` in the Fleet Manager's views is such a change. A contributor learns from neither the gate nor CONTRIBUTING that only a Mac produces the goldens, or what to do without one. The likely outcomes are a stalled PR or a stamp written without a capture.

Design authority: the gate's own docblock (the goldens are captured on Darwin, and the capture configs are deliberately absent from Ubuntu CI), and the `good first issue` label, which invites contributors who may not own a Mac.

## The Architectural Reality

At `dev` `5a60811`:
- `buildScripts/checkVisualBaselines.mjs` is a render-free input-identity gate. Its no-stamp message already says "Capture (Darwin)". Its drift message, the one a contributor meets, says only "Recovery (on a rendering platform)".
- `CONTRIBUTING.md`'s browser-suites section lists `npm run test-visual` "guarded by npm run check-visual-baselines" and says nothing about the platform or the recovery.
- `test/playwright/unit/buildScripts/checkVisualBaselines.spec.mjs` covers the gate's helpers (`diffStamp`, `parseStamp`, and the others), not its message text.

## The Fix

- `checkVisualBaselines.mjs`: the drift message's recovery block names macOS. It adds one line for a contributor without a Mac: ask in the pull request, and a maintainer captures and stamps the goldens on the branch.
- `CONTRIBUTING.md`: two or three lines in the browser-suites section. They say the goldens are macOS captures and give both paths: on a Mac, the recovery steps (both capture suites with `--update-snapshots`, stage, then `npm run stamp-visual-baselines`); without one, ask in the pull request.

## Acceptance Criteria

- [ ] AC-1: The drift message names macOS and tells a contributor without one to ask in the pull request.
- [ ] AC-2: CONTRIBUTING says the goldens are macOS captures and gives both paths.
- [ ] AC-3: What the gate checks is unchanged: `checkVisualBaselines.spec.mjs` passes, and a drifted input still exits 1 with the same file listing.

## Out of Scope

- Rendering or capturing goldens in CI, including on a macOS runner (#278 and #852 exclude it too).
- Making the gate non-blocking for pull requests from forks. That changes what the gate guarantees.

## Related

#863 (where it surfaced), #862, #278, #852.

Live latest-open + closed sweep: checked the latest 20 open, plus closed searches for `visual baselines golden` and `contributor CONTRIBUTING macOS` (Out of Scope / Avoided Traps read on #278, #852 and #11), at 2026-11-03T08:16Z; no equivalent or prior exclusion found. A2A claim sweep: the last 30 messages hold no claim on this surface. Memory Core sweep: no prior decision on contributor guidance for the gate. Own-assignment sweep: #843, #516 and #424 touch other surfaces.

Ownership: self-assigned (@neo-opus-ada, Institution steward); one PR follows now.

Origin Session ID: 0d705b6b-6706-4013-b114-91e7490cf576
Retrieval Hint: "outside contributor visual baselines Darwin goldens maintainer capture"

## Timeline

- 2026-11-03T08:17:13Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-03T08:17:15Z @neo-opus-ada added the `documentation` label
- 2026-11-03T08:17:15Z @neo-opus-ada added the `enhancement` label
- 2026-11-03T08:17:15Z @neo-opus-ada added the `ai` label
- 2026-11-03T08:17:16Z @neo-opus-ada added the `contributor-experience` label
- 2026-11-03T08:19:01Z @neo-opus-ada cross-referenced by PR #865
- 2026-11-03T14:29:12Z @tobiu referenced in commit `5046f84` - "docs: tell contributors the goldens are macOS captures (#864) (#865)

An outside pull request that touches a style-owning input stops at
check-visual-baselines, whose recovery said only "on a rendering
platform", and CONTRIBUTING never said where the goldens come from. The
drift message now names macOS and tells a contributor without a Mac to
ask in the pull request; CONTRIBUTING gives both paths. What the gate
checks is unchanged."
- 2026-11-03T14:29:13Z @tobiu closed this issue

