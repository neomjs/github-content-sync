---
id: 1518
title: Every Brain baseline check name repeats its caller
state: OPEN
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-11-08T19:59:27Z'
updatedAt: '2026-11-22T03:20:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1518'
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
blockedBy:
  - '[x] 320 The guide lint runs in the shared PR baseline, not from a Brain checkout'
blocking: []
---
# Every Brain baseline check name repeats its caller

## Context

The operator, 2026-11-08, on a Brain pull request's checks: "inside the brain repo, the titles look odd now => duplication." Each check reads `Shared PR Baseline / Shared PR Baseline / PR body`, `… / Release ref`, `… / Secrets`, and so on.

On the same day the Engine finished its side of neomjs/neo#17171's Decision (neomjs/neo#20598). That Decision makes the Brain's side a ticket of its own: "Brain follows under its own ticket." The guide lint's move into the shared baseline (neomjs/neo-agent-skills#321, for neomjs/neo#20597) leaves a Brain half as well. Each of these changes the Brain's required contexts, so one change and one ruleset edit should carry them together.

## The Problem

- **Doubled names.** `.github/workflows/shared-pr-baseline.yml` sets both the workflow `name:` and the caller job's `name:` to `Shared PR Baseline` (`dev`, lines 5 and 22). GitHub renders a check as `<workflow> / <caller job> / <called job>`. The Engine's caller leaves its job unnamed (id `baseline`), so its checks read `PR Baseline / baseline / PR body`.
- **An old pin.** The caller pins `reusable-pr-baseline.yml@v0.1.84`. The Brain still starts the ten pre-split baseline jobs on every PR, under the 20-slot organization cap the split exists for. The ruleset requires all ten as `Shared PR Baseline / …`, beside `unit`, `substrate`, `integration-unified` and `integration-parity` (`GET /repos/neomjs/neo-agent-brain/rules/branches/dev`, 2026-11-08).
- **Standalone guards.** PR-gating guard workflows still run as their own jobs: `config-template-ssot-lint.yml`, `identity-vocabulary-lint.yml`, `openapi-service-parity-lint.yml` and `script-plane-lint.yml`. That is a first census; intake completes it.
- **A second copy of the guide lint.** `ai/scripts/lint/lint-guides.mjs` and its spec now duplicate the skills bin. They also carry seven quadratic paths on guide input that the skills port removed (#321, `e54f994`).

## The Architectural Reality

- `reusable-pr-baseline.yml` from v0.1.90 on runs three jobs: `PR baseline`, `Skills materialized`, and `guards` when a caller declares a list. 1.2.0, once #321 is released, adds the `Guides` step behind `guides: true`.
- The Engine's caller at `dev` `b6758e9518` is the pattern: an unnamed job `baseline`, a pinned tag, a `guards` list. Its parity checker credits only that list.
- The Brain's `neo.mjs` dependency is the Engine at a pinned commit, and `node_modules/neo.mjs/learn/` holds the Engine's guides (#20597, Ada's refinement).

## The Fix

1. The pin is done (`v2.3.0`, see the re-measure). Drop the caller job's `name:`, so the checks read `baseline / PR baseline` and `baseline / Skills materialized`.
2. Resolved differently: the standalone guard workflows are gone, and their lints run as steps of `brain-lint.yml`.
3. and 4. Moved to #1893: `guides: true`, the Engine-guide check at the `neo.mjs` pin, and retiring the copy.
5. One Brain ruleset edit (operator) at merge: replace the required `Shared PR Baseline / PR baseline` and `Shared PR Baseline / Skills materialized` with `baseline / PR baseline` and `baseline / Skills materialized` (the required set on 2026-11-22).

## Acceptance Criteria

- [ ] AC-1: A Brain PR's checks show the baseline contexts under one caller prefix, with no doubled name.
- [x] AC-2 (resolved differently): no standalone guard workflow remains; the lints run as steps of `brain-lint.yml`.
- AC-3 and AC-4 moved to #1893.
- [ ] AC-5 (post-merge, operator): `gh api repos/neomjs/neo-agent-brain/rules/branches/dev` shows the new set.

## Out of Scope

- What any guard checks.
- The Engine side: neomjs/neo#20598 is done, and neomjs/neo#20597 is open.

## Related

neomjs/neo-agent-skills#321 has shipped, and the caller pins `v2.3.0`. #1893 takes the guide-lint half. Related: neomjs/neo#17171 (the Decision) · neomjs/neo#20598 (the Engine side) · neomjs/neo#20597 (the guide lint's Engine side) · #523 (the Brain's first baseline caller).

Decision Record impact: none (aligned with neomjs/neo#17171's Decision).

Live latest-open + closed sweep: the latest 20 open Brain issues, plus searches for "baseline guards", `reusable-pr-baseline`, `v0.1.84`, "Shared PR Baseline" and "baseline pin". Hits read: #523, #353 and #970, all closed predecessors. A2A claims over the last 12 messages and Memory Core also checked, 2026-11-08T19:58Z. No equivalent found.

Owner: @neo-opus-grace (author of #523). After the Engine cut; not an Engine release blocker (operator).

Origin Session ID: 3041b574-18af-44d7-840f-2baf12f52c3a
Retrieval Hint: "Brain shared-pr-baseline v0.1.84 Shared PR Baseline doubled check names guards guides lint-guides retire ruleset"

## Intake re-measure (2026-11-22)

Measured at Brain `dev` `d6f43108`. Two of the four problems are gone, and two stand.
- **Pin: resolved.** `shared-pr-baseline.yml:24` calls `reusable-pr-baseline.yml@v2.3.0`, and Dependabot moves the tag.
- **Standalone guards: resolved differently.** The four workflows are gone, and their lints run as steps of the one `brain-lint.yml` job.
- **Doubled names: stands.** Both the workflow `name:` and the caller job `name:` are still `Shared PR Baseline`.
- **Guide-lint copy: stands, and is unwired.** `ai/scripts/lint/lint-guides.mjs` (651 lines) and its spec (338) run in no CI step and from no package script; only `isEntryModule.spec.mjs` uses the file, as its subject. The Brain caller sets no `guides:` input either, so the Brain guides get no guide lint at all. The shared bin run on `dev` reports 26 full guides, 0 hard findings and 22 warnings, so `guides: true` would be green.

**This changes the ticket's reason to bundle.** `guides: true` adds a step inside the existing `PR baseline` job, so retiring the copy changes no required context. Only the name fix needs the ruleset edit, and it renames the contexts every in-flight Brain PR reports, so it is better scheduled outside the v14 cut weekend. Plan: split. This ticket keeps the doubled names and the ruleset edit. A new ticket takes the guide-lint retirement: delete the copy and its spec, set `guides: true`, and point `isEntryModule.spec.mjs` at another lint. Before deleting, read closed #1360, #1366 and #1368 for any caller of the copy outside the Brain.

**Split filed (2026-11-22):** #1893. Those three tickets and a code search of nine other organization repositories found no caller outside the Brain.

## Timeline

- 2026-11-08T19:59:27Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-08T19:59:29Z @neo-opus-grace added the `enhancement` label
- 2026-11-08T19:59:29Z @neo-opus-grace added the `ai` label
- 2026-11-08T19:59:29Z @neo-opus-grace added the `build` label
- 2026-11-08T19:59:33Z @neo-opus-grace marked this issue as being blocked by #321
- 2026-11-08T19:59:52Z @neo-opus-grace removed the block by #321
- 2026-11-08T20:00:01Z @neo-opus-grace marked this issue as being blocked by #320
- 2026-11-08T20:16:02Z @neo-opus-grace cross-referenced by #20597
- 2026-11-08T20:19:05Z @neo-opus-grace cross-referenced by PR #321
- 2026-11-08T20:38:42Z @neo-opus-grace cross-referenced by PR #20612
- 2026-11-08T21:01:20Z @neo-opus-grace cross-referenced by #20328
- 2026-11-11T03:48:52Z @neo-opus-grace cross-referenced by #1594
- 2026-11-11T19:05:32Z @neo-opus-grace cross-referenced by #1609
- 2026-11-12T03:19:08Z @neo-opus-grace cross-referenced by #1628
- 2026-11-22T03:18:13Z @neo-opus-grace cross-referenced by #1893
- 2026-11-22T03:20:14Z @neo-opus-grace changed title from **The Brain's PR baseline still pins v0.1.84, and every check name repeats its caller** to **Every Brain baseline check name repeats its caller**

