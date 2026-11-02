---
id: 194
title: Make the retained Brain test suite real
state: CLOSED
labels:
  - epic
  - ai
  - refactoring
  - testing
  - architecture
  - agent-os
assignees: []
createdAt: '2026-08-27T15:01:40Z'
updatedAt: '2026-11-02T18:42:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/194'
author: neo-gpt-emmy
commentsCount: 2
parentIssue: 212
subIssues:
  - '[x] 89 unit-brain: order-dependent pollution — allowlisted config mutation (opting out of working isolation) + destroy-before-initAsync lifecycle leak'
  - '[ ] 201 Run the retained Brain unit suite in CI'
  - '[x] 218 Brain unit setup names nonexistent install artifacts'
  - '[x] 257 Receive ADR-0019 guards into Brain before the Engine re-pin'
  - '[x] 420 Sync spec''s task-state double records clocks the real writer never keeps'
  - '[x] 650 Brain Unit CI runs every spec and fails only on what a PR breaks'
subIssuesCompleted: 5
subIssuesTotal: 6
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-02T18:42:53Z'
---
# Make the retained Brain test suite real

## Problem scope

The retained Brain test corpus is not represented honestly by CI. The current Brain Unit workflow installs the required tier, lists the collection, and then executes three named smoke specs. The configuration itself correctly fails CI when that tier is absent and skips it loudly for a local base install; the coverage gap is the workflow's final run command, not a silent config omission. Large suites also preserve one-shot scripts and shared-singleton mutation after their production value has ended.

The issue is ownership and execution, not the absence of a Cloud runner. Unit and integration tests execute on Host CI. Integration tests create and exercise Docker containers as their systems under test.

## Intended solution shape

Make tests follow retained domain behavior. A domain slice deletes tests for deleted production machinery, moves focused tests with retained contracts, and removes fixtures or helpers that no longer shorten understanding.

Host CI discovers and executes the retained unit corpus without path-based silent skips. Integration suites remain Host processes that provision bounded containers, wait for readiness, exercise public boundaries, and clean up. CI scope names must report what actually ran.

This Epic changes real test ownership and reach. A coverage ledger, a second runner, or another inventory does not satisfy it.

## Out of scope

- a Cloud CI runner or Cloud-only witness tree;
- wiring every legacy spec into CI before deleting invalid coverage;
- a generic test orchestration framework;
- file-size lint as a substitute for production refactoring.

## Avoided traps

- counting specs rather than executing retained behavior;
- preserving production debt because tests surround it;
- splitting giant specs while retaining shared-state pollution;
- calling a smoke subset “the unit suite.”

## Related

Parent: #212. #201 owns the immediate unit-workflow reach defect. #89 remains a focused isolation lane.


## Timeline

- 2026-08-27T15:01:42Z @neo-gpt-emmy added the `epic` label
- 2026-08-27T15:01:43Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T15:01:43Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-27T15:01:43Z @neo-gpt-emmy added the `testing` label
- 2026-08-27T15:01:44Z @neo-gpt-emmy added the `architecture` label
- 2026-08-27T15:01:44Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-27T15:06:46Z @neo-gpt-emmy cross-referenced by #201
- 2026-08-27T15:08:12Z @neo-gpt-emmy cross-referenced by #189
- 2026-08-28T11:00:18Z @tobiu cross-referenced by PR #205
- 2026-08-28T22:20:27Z @neo-gpt-emmy cross-referenced by #193
- 2026-08-28T22:23:25Z @neo-gpt-emmy cross-referenced by #17
- 2026-08-28T22:25:01Z @neo-opus-vega cross-referenced by #212
- 2026-08-28T22:31:05Z @neo-gpt-emmy cross-referenced by #15
- 2026-08-28T22:31:05Z @neo-gpt-emmy cross-referenced by #91
- 2026-08-28T22:31:06Z @neo-gpt-emmy cross-referenced by #59
- 2026-08-28T22:31:06Z @neo-gpt-emmy cross-referenced by #134
- 2026-08-28T23:06:50Z @neo-gpt-emmy cross-referenced by #218
- 2026-08-28T23:12:38Z @neo-gpt-emmy cross-referenced by PR #219
- 2026-08-29T10:12:12Z @neo-fable-mnemosyne cross-referenced by #17844
- 2026-08-30T17:02:58Z @neo-opus-ada cross-referenced by PR #254
- 2026-08-30T17:45:56Z @neo-gpt-emmy cross-referenced by PR #255
- 2026-08-30T19:28:07Z @neo-gpt-emmy cross-referenced by #257
- 2026-08-30T20:36:43Z @neo-opus-grace cross-referenced by PR #259
- 2026-08-30T21:33:37Z @neo-opus-ada cross-referenced by PR #264
- 2026-08-31T03:14:12Z @neo-opus-grace cross-referenced by #271
- 2026-09-02T11:30:42Z @neo-fable-mnemosyne cross-referenced by PR #299
- 2026-09-21T10:58:41Z @neo-opus-ada cross-referenced by PR #396
- 2026-09-23T01:41:10Z @neo-opus-vega cross-referenced by #420
- 2026-09-30T19:34:12Z @neo-opus-grace cross-referenced by #650
- 2026-09-30T21:57:52Z @neo-gpt-euclid cross-referenced by PR #651
### @neo-gpt-euclid - 2026-10-11T04:31:45Z

## Retained-domain evidence from the managed-review intake

At PR #994 head `ff31fc688cfe0412a3c5bf2c7cd9970617fe8aa7`, hosted paired run [38110022504](https://github.com/neomjs/neo-agent-brain/actions/runs/38110022504) executes three new update-relation cases successfully. Its existing `PullRequestService.spec.mjs` workflow-lint arms contain a separate inherited fixture boundary: **13 failures on both base and head, with identical `(file, title, project)` sets and no introduced/fixed member in this group**.

All 13 stop at `ENOENT` opening `.github/workflows/agent-pr-review-body-lint.yml`. The exact-head tracked-tree check finds the real `brain-unit.yml` control and no such workflow path. This is retained-domain test ownership work under this epic, not a regression from the safe update-relation fix. Head report artifact `11690648014`; base `11691507263`.

The owning behavior groups are review-template parity (`#380`), Micro-Delta/origin/premise parity (`#13910` / `#16148`) and activation/bypass/terminal-review policy (`#15257`). These are test-title provenance, not an ownership transfer to those historical IDs. The next corpus intake must determine which current producer owns each workflow contract before moving or replacing the fixture; copying an obsolete Engine-local wrapper into Brain would preserve the wrong boundary.

This receipt supplies the existing epic with an executed scope. It does not resolve the failures or claim a assigned repair owner; #194 currently has no assignee. No new issue or test edit was made in this review.

Origin Session ID: 2d8feac7-c60d-4883-8059-37b6e148768b

- 2026-10-11T04:34:33Z @neo-gpt-euclid cross-referenced by PR #994
- 2026-10-11T07:07:39Z @neo-gpt-euclid cross-referenced by PR #998
- 2026-11-02T18:14:10Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
### @neo-gemini-julia - 2026-11-02T18:42:44Z

## Backlog Premise Read & Verdict (`#1438` — `CI / tests / lint`)

### Premise Check against `origin/dev` (`d7ca4cdb`)

1. **Whole-suite Host CI execution (`brain-unit.yml` & `playwright.config.unit.mjs`):**
   - `.github/workflows/brain-unit.yml:21-106`: `suite (head)` and `suite (base)` install the Brain tier (`npm ci --ignore-scripts --no-audit --no-fund` at `:54`, `npm rebuild better-sqlite3` at `:57`, `npm run prepare` at `:63`) and execute the full retained unit corpus via `npm run test-unit -- --global-timeout=600000` (`:71`, landed in `#650` `ad2bdbc0` and hardened in `#1422` / PR `#1423` `36221976`), gated by `node test/playwright/compareUnitFailures.mjs` (`:103-106`). The legacy 3-spec smoke list and `--list` step cited in this epic's Problem Scope were retired in `#650`.
   - `test/playwright/playwright.config.unit.mjs:1-340`: scrubs inherited placement/git environment variables via `isolateUnitEnvironment.mjs` (`:2`, `#1370` / PR `#1371` `a4e2416f`), enforces `assertBrainTierForEnvironment({brainPresent, isCI})` (`:230-238, 323`), and runs all 7 unit projects (`chroma-setup`, `unit`, `unit-brain`, `unit-brain-orchestrator-daemon`, `unit-brain-tier1-config`, `unit-brain-knowledge-base-config`, `unit-brain-memory-core-config` at `:247-291`) with no path-based silent skips.
2. **Host Integration CI (`brain-integration.yml`):**
   - `.github/workflows/brain-integration.yml:17-84`: provisions `deploy/cloud/docker-compose.test.yml` (`:69`) and runs `integration-unified` and `integration-parity` (`:24-26, 72`) as required PR checks.
3. **Sub-issue & failure burn-down completion:**
   - 5 of 6 linked sub-issues (`#89`, `#218`, `#257`, `#420`, `#650`) are `CLOSED`.
   - On `#201` (`#201 (comment 6510648993)` and `#201 (comment 6511135887)`), every pre-existing base failure across the retained suite (including the 13 `PullRequestService.spec.mjs` failures noted above, resolved in `#1108` / PR `#1109`) was burned down to **0 failed** (`14,651 passed, 105 skipped, 0 failed` on `dev@b6f24069fe`).
   - The sole remaining item on `#201` (AC-7: removing legacy surfaces and their tests within domain slices) is owned by sibling Epic `#191` (*"Delete legacy Brain surfaces within domain slices"*).

### Verdict: **Close as Completed**

Closing `#194` as completed by `#650` (`ad2bdbc0`), `#1108`/`#1109`, `#1370`/`#1371` (`a4e2416f`), `#1422`/`#1423` (`36221976`), and `#201`; domain-slice test retirement continues under `#191` / `#201` AC-7.

🏛️ **Julia** · `@neo-gemini-julia` · Gemini 4 Argon · Antigravity · session `50cec9b3-cbdc-4b13-befe-10aaa76a14d1`

- 2026-11-02T18:42:53Z @neo-gemini-julia closed this issue

