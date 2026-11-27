---
id: 1066
title: Upgrade Monaco to 0.58.0 and retire the DOMPurify override
state: CLOSED
labels:
  - bug
  - ai
  - dependencies
assignees:
  - neo-opus-grace
createdAt: '2026-11-27T14:09:33Z'
updatedAt: '2026-11-27T14:27:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1066'
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
blocking: []
closedAt: '2026-11-27T14:27:14Z'
---
# Upgrade Monaco to 0.58.0 and retire the DOMPurify override

## Context

Dependabot's Monaco bump #1060 (0.57.0 → 0.58.0) passes `Isolated Institution`, `Explicit Brain contract` and CodeQL, and its auto-merge hand-off is armed. The required `Shared PR Baseline / PR baseline` fails: `check-npm-overrides: 1 of 3 override(s) no longer hold a floor.`

The Engine retired the same rule for the same reason in neomjs/neo#22355 (Resolves neomjs/neo#22354), merged today.

## The Problem

On `dev`, the root override `"dompurify": "^3.4.17"` still holds a floor: `monaco-editor` 0.57.0 pins `dompurify` 3.4.15, while `mermaid` requires `^3.4.17`. Monaco 0.58.0 pins 3.4.17 itself, so after the bump nothing needs the rule. The guard then fails it, and Dependabot never removes an override, so #1060 stays red.

## The Architectural Reality

Design authority: the shared guard ([`check-npm-overrides.mjs`](https://github.com/neomjs/neo-agent-skills/blob/v2.11.0/scripts/check-npm-overrides.mjs)) fails "an npm `overrides` rule that no longer does anything, or that now forces a downgrade". `package.json` owns the policy and `package-lock.json` the resolved tree.

GitHub's DOMPurify advisories all fall at or below 3.4.16, and the newest three are fixed in 3.4.17, so the floor survives in the dependents' own pins.

## The Fix

In one PR: pin `monaco-editor` to 0.58.0, remove the root `dompurify` override, and regenerate `package-lock.json` against current `dev`. The two other overrides (`@huggingface/transformers > sharp`, `lodash-es`) stay.

Dependabot's #1060 becomes redundant once this lands.

## Acceptance Criteria

- [ ] `package.json` pins `monaco-editor` 0.58.0 and has no `dompurify` override. The other overrides stay, and the shared guard passes.
- [ ] The locked tree holds a single `dompurify` 3.4.17, shared by Monaco and Mermaid.
- [ ] `Isolated Institution` and `Explicit Brain contract` pass on the PR head.

## Out of Scope

- Monaco API changes.
- Other dependency upgrades.
- The plist pin (#1064).

## Avoided Traps

- **Pushing the removal onto Dependabot's branch.** A push by another account disarms auto-merge, and Dependabot stops rebasing the branch.
- **Keeping the rule "for safety".** The guard fails any rule that holds no floor, so keeping it means every Monaco bump goes red.

## Related

#1060 · neomjs/neo#22355 · neomjs/neo#22354 · #1064

Decision Record impact: none.

Live latest-open + closed sweep: checked the latest open issues and an open + closed search for `monaco`, `dompurify` and `npm overrides` at 2026-11-27T14:09Z; no equivalent found. #149, #657 and #151 are closed and cover other dependency surfaces. 58 open (red band); no open ticket owns this surface. The A2A sweep (latest 30) showed no claim on #1060.

Owner: @neo-opus-grace.

Origin Session ID: 4f976ef7-5f53-44f3-b7da-da23e0288b29
Retrieval Hint: "Institution Monaco 0.58.0 DOMPurify override redundant check-npm-overrides #1060"

## Timeline

- 2026-11-27T14:09:34Z @neo-opus-grace added the `bug` label
- 2026-11-27T14:09:34Z @neo-opus-grace added the `ai` label
- 2026-11-27T14:09:35Z @neo-opus-grace added the `dependencies` label
- 2026-11-27T14:09:38Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-27T14:11:21Z @neo-opus-grace cross-referenced by PR #1067
- 2026-11-27T14:27:14Z @tobiu referenced in commit `0764fa4` - "chore(deps-dev): bump Monaco to 0.58.0 and retire the DOMPurify override (#1066) (#1067)

Monaco 0.58.0 pins DOMPurify 3.4.17 and Mermaid requires ^3.4.17, so
the root "dompurify" override no longer holds a floor and the shared
check-npm-overrides guard fails it on Dependabot's #1060. On dev the
rule was still load-bearing (Monaco 0.57.0 pinned 3.4.15), so the
removal rides with the bump, as neomjs/neo#22355 did for the Engine.

The regenerated lockfile is byte-identical to #1060's head, the tree
its Isolated Institution and Explicit Brain contract jobs passed on."
- 2026-11-27T14:27:16Z @tobiu closed this issue

