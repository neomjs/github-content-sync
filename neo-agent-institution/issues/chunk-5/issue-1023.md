---
id: 1023
title: Completed session rotation renders "to unknown" on roster cards
state: CLOSED
labels:
  - bug
  - good first issue
  - help wanted
  - agent-os
  - ai
  - design
assignees:
  - thib2306681-spec
createdAt: '2026-11-18T15:38:06Z'
updatedAt: '2026-11-23T17:15:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1023'
author: neo-gemini-julia
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-23T17:15:49Z'
---
# Completed session rotation renders "to unknown" on roster cards

Please comment to claim before starting; check current assignees and linked PRs.

## Context

On `2026-11-18`, Fleet Manager roster cards for `@neo-opus-grace` and `@neo-opus-vega` displayed `rotated 03:20 PM · from a915466a to unknown` and `rotated 06:58 AM · from 8788cd0d to unknown`, making successful session rotations look broken or half-finished during a wider Agent OS incident when both rotations had actually completed (`state: 'completed'`, `successorMessageId` present).

## The Problem

Since `neomjs/neo-agent-brain#1712`, Brain's `runSessionRotation` intentionally writes `newSessionId: null` on `completed` rotation records because `[sunset successor]` self-DMs carry no sender Memory Core session ID (an MCP reconnect inside an existing transcript mints a session ID without opening a fresh session).

However, `AgentOS.util.SessionRotation.describe()` (`apps/agentos/util/SessionRotation.mjs:131-138`) passes `rotation.newSessionId` into `short = sessionId => ... : 'unknown'`, producing `rotated <time> · from <8-char> to unknown` on every successful plane-driven rotation.

## The Architectural Reality

- `apps/agentos/util/SessionRotation.mjs` (`describe`, lines 112–162): `case 'completed'` unconditionally formats `to ${short(rotation.newSessionId)}`, turning `null` into `'unknown'`.
- `test/playwright/unit/apps/agentos/view/fleet/roster/card/rotationLine.spec.mjs`: unit test suite for `SessionRotation.describe` and roster card rendering.
- `test/playwright/visual/FleetCockpitVisual.spec.mjs`: visual fixture assertions on `.fm-card-rotation`.

Design authority: `apps/agentos/util/SessionRotation.mjs:114-115` ("completed: the fresh session registered its wake route, so the old one is retired. The words name both sessions and raise nothing") vs `neomjs/neo-agent-brain#1712` ("newSessionId stays null, because the marker names no sender session").

## The Fix

Update `AgentOS.util.SessionRotation.describe()` (`apps/agentos/util/SessionRotation.mjs`) so `case 'completed':` omits `to unknown` when `rotation.newSessionId` is null/empty:
- When both `oldSessionId` and `newSessionId` are non-empty strings: keep `rotated <time> · from <old8> to <new8>`.
- When `oldSessionId` is present and `newSessionId` is null/empty: render `rotated <time> · from <old8>` (and include `decided?.title` / `from ${rotation.oldSessionId}` / `receipt ${rotation.receiptId}` in `title`, without `"to unknown"`).
- When neither session ID is present: render `rotated <time>` (or `'rotated'`).

Update JSDoc on `SessionRotation.describe()` and update/add unit specs in `test/playwright/unit/apps/agentos/view/fleet/roster/card/rotationLine.spec.mjs` (plus check `FleetCockpitVisual.spec.mjs` fixtures if affected).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `AgentOS.util.SessionRotation.describe` (`apps/agentos/util/SessionRotation.mjs`) | neomjs/neo-agent-brain#1712 / #842 | Omit `to unknown` when `newSessionId` is null/empty in `completed` state; keep `from <old8> to <new8>` when both are present | Bare `rotated <time>` if both IDs absent | JSDoc on `SessionRotation#describe` | `rotationLine.spec.mjs` |

Decision Record impact: none.

## Acceptance Criteria

- [ ] `SessionRotation.describe({state: 'completed', decidedAt, oldSessionId: 'd915466a-eeed-432b-bbd6-f294497a7bad', newSessionId: null, receiptId})` returns text without `'unknown'` (`rotated <time> · from d915466a`) and retains `receipt <receiptId>` in `title`.
- [ ] When `newSessionId` is a non-empty string, `from <old8> to <new8>` still renders unchanged.
- [ ] `npm run test-unit` covering `test/playwright/unit/apps/agentos/view/fleet/roster/card/rotationLine.spec.mjs` (and any updated cockpit fixtures) passes cleanly.

## Out of Scope

- Changing Brain's `sessionRotation` payload schema or synthesizing a synthetic `newSessionId`.
- Fixing Claude rotation holds or receiver CPU spin (neomjs/neo-agent-brain#1841 and neomjs/neo-agent-brain#1840).

## Avoided Traps

- Deriving a pseudo-session ID from `successorMessageId`: a message UUID prefix is not a session ID and misleads operators and diagnostics.

## Related

#842, neomjs/neo-agent-brain#1692, neomjs/neo-agent-brain#1712, neomjs/neo-agent-brain#1841

---

Live latest-open + closed sweep: checked latest 20 open of 56 (`red` band) + open/closed searches (`SessionRotation`, `to unknown`, `rotationLine`, #842) at 2026-11-18T15:36:25Z; Out of Scope read on #842; no open owner for `SessionRotation.mjs` `"to unknown"` rendering.
A2A in-flight sweep: latest 30 messages read across all read states (`2026-11-18T14:37Z–15:35Z`); no peer claim on `SessionRotation.mjs`.
MC sweep: `"rotation refused prompted-after-receipt claude session sunset fresh session"`, 8 results (#1712/#842 context, no FM card formatting ticket).
Own-assignment sweep: 0 open in `neomjs/neo-agent-institution`.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: 613064f8-74aa-4baa-8ee7-6a0b8540f31b
Retrieval Hint: "SessionRotation.describe completed newSessionId null to unknown roster card"


## Timeline

- 2026-11-18T15:38:39Z @neo-gemini-julia added the `bug` label
- 2026-11-18T15:38:39Z @neo-gemini-julia added the `good first issue` label
- 2026-11-18T15:38:39Z @neo-gemini-julia added the `help wanted` label
- 2026-11-18T15:38:40Z @neo-gemini-julia added the `agent-os` label
- 2026-11-18T15:38:40Z @neo-gemini-julia added the `ai` label
- 2026-11-18T15:38:40Z @neo-gemini-julia added the `design` label
- 2026-11-20T13:39:53Z @neo-opus-grace cross-referenced by #1031
### @thib2306681-spec - 2026-11-23T10:07:28Z

Hi! I would like to work on this issue. I'll reproduce the "to unknown" rendering on roster cards locally, fix the logic with unit tests, and submit a PR. Could you please assign it to me?

- 2026-11-23T10:07:40Z @github-actions assigned to @thib2306681-spec
### @github-actions - 2026-11-23T10:07:41Z

Thanks for claiming this, @thib2306681-spec, and welcome back! Assigned to you — you can start now.

Your claim reserves this issue for 3 days (any comment here resets the clock; we still merge the first approve-worthy PR if another lands first or a claim goes quiet).

<sub>An automatic reply. How claims work [QUARANTINED_URL: github.com]</sub>
<!-- claim-responder:@thib2306681-spec -->

- 2026-11-23T13:14:55Z @thib2306681-spec cross-referenced by PR #1041
- 2026-11-23T14:55:29Z @neo-gpt-sophie referenced in commit `9752fee` - "test(agentos): refresh the rotation label golden stamp (#1023)"
- 2026-11-23T17:15:48Z @tobiu referenced in commit `660629b` - "fix(agentos): omit 'to unknown' on completed session rotation when ne… (#1041)

* fix(agentos): omit 'to unknown' on completed session rotation when newSessionId is null

* test(agentos): refresh the rotation label golden stamp (#1023)

---------

Co-authored-by: Sophie <neo-gpt-sophie@neomjs.com>
Co-authored-by: Maryam <neo-gemini-maryam@neomjs.com>"
- 2026-11-23T17:15:50Z @tobiu closed this issue

