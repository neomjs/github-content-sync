---
id: 1518
title: 'The Brain''s PR baseline still pins v0.1.84, and every check name repeats its caller'
state: OPEN
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-11-08T19:59:27Z'
updatedAt: '2026-11-08T19:59:27Z'
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
# The Brain's PR baseline still pins v0.1.84, and every check name repeats its caller

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

1. Pin `shared-pr-baseline.yml` to the release that ships `guides` (≥ 1.2.0). Drop the caller job's `name:`, so the checks read `baseline / PR baseline`, `baseline / Skills materialized` and `baseline / guards`.
2. Move the Brain's PR-gating guards into the `guards` list and retire their standalone workflows.
3. Turn on `guides: true` for the Brain's own guides. Retire `ai/scripts/lint/lint-guides.mjs` and its spec, with a `spec-retired:` account, and point `ai:lint-guides` at the bin.
4. Lint the Engine's guides at the Brain's `neo.mjs` pin for the two Brain-data rules, `dead-script-ref` and `openapi-tool-parity`. A Brain change that breaks an Engine guide then reds the Brain PR that made it.
5. One Brain ruleset edit (operator) at merge: remove the ten `Shared PR Baseline / …` contexts, and add `baseline / PR baseline`, `baseline / Skills materialized` and `baseline / guards`.

## Acceptance Criteria

- [ ] AC-1: A Brain PR's checks show the three baseline contexts under one caller prefix, with no doubled name.
- [ ] AC-2: No PR-gating guard runs outside the `guards` list. The census sits in the PR body, with each retired workflow named.
- [ ] AC-3: `ai/scripts/lint/lint-guides.mjs` and its spec are gone, and the Brain's guides fail a required context on a HARD finding.
- [ ] AC-4: A Brain PR that renames an `ai:*` script named by an Engine guide at the pin fails Brain CI.
- [ ] AC-5 (post-merge, operator): `gh api repos/neomjs/neo-agent-brain/rules/branches/dev` shows the new set.

## Out of Scope

- What any guard checks.
- The Engine side: neomjs/neo#20598 is done, and neomjs/neo#20597 is open.

## Related

Blocked by neomjs/neo-agent-skills#321 (release 1.2.0). Related: neomjs/neo#17171 (the Decision) · neomjs/neo#20598 (the Engine side) · neomjs/neo#20597 (the guide lint's Engine side) · #523 (the Brain's first baseline caller).

Decision Record impact: none (aligned with neomjs/neo#17171's Decision).

Live latest-open + closed sweep: the latest 20 open Brain issues, plus searches for "baseline guards", `reusable-pr-baseline`, `v0.1.84`, "Shared PR Baseline" and "baseline pin". Hits read: #523, #353 and #970, all closed predecessors. A2A claims over the last 12 messages and Memory Core also checked, 2026-11-08T19:58Z. No equivalent found.

Owner: @neo-opus-grace (author of #523). After the Engine cut; not an Engine release blocker (operator).

Origin Session ID: 3041b574-18af-44d7-840f-2baf12f52c3a
Retrieval Hint: "Brain shared-pr-baseline v0.1.84 Shared PR Baseline doubled check names guards guides lint-guides retire ruleset"

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

