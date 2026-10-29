---
id: 1360
title: lintGuides' discovery arm reads neo's guides from the Brain root
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-29T16:12:48Z'
updatedAt: '2026-10-29T17:34:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1360'
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
closedAt: '2026-10-29T17:34:01Z'
---
# lintGuides' discovery arm reads neo's guides from the Brain root

## Context

`test/playwright/unit/ai/scripts/lint/lintGuides.spec.mjs` › "discoverGuideSeries: recursively reaches nested guides without widening the full-guide surface" is the last #201 red without a fix in flight (classified in #201 comment 6482850355).

## The Problem

The arm calls `discoverGuideSeries()`, which walks `learn/guides` under `ROOT_DIR` (the script's own repository root, `ai/scripts/lint/lint-guides.mjs:49`). It expects neo's real guides there (`learn/guides/testing/ComponentTesting.md`, `learn/guides/userinteraction/events/CustomEvents.md`). The Brain has no `learn/guides`, so discovery returns `[]` and the arm fails. It is engine-independent: neo's guides are not the Brain's input.

## The Architectural Reality

- Custody of the script is settled. The cut manifest (`agentOsExtractionInventory.json` at neo `91ae3604b4^`) gives `ai/scripts/lint/lint-guides.mjs` the disposition `edge`, so the Brain holds it. Its subjects (`learn/benefits`, nested `learn/guides`, the Engine half of `learn/agentos`) live in neo.
- The arm tests discovery semantics, recursion and scope boundary, not neo's content. A fixture tree tests the same semantics in any repository.
- Out of scope here: the lint runs nowhere since neomjs/neo#17806 (defect-note `a03b68c6`, 2026-10-29). Wiring it into neo's CI needs a caller-supplied root. This change adds that root parameter for discovery and no more.

## The Fix

`discoverGuideSeries(rootDir = ROOT_DIR)` walks `learn/guides` under the root it is given. The arm builds a temp tree with two nested guides and one `learn/agentos/*.md` file, then asserts that both nested guides are found and the `learn/agentos` file is not. Nothing in the CLI path changes.

## Acceptance Criteria

- [ ] AC-1: the discovery arm passes in the Brain and reads no file outside its fixture.
- [ ] AC-2: the arm still fails if discovery stops recursing or starts including `learn/agentos`.
- [ ] AC-3: default discovery (no argument) is unchanged for the CLI arm.
- [ ] AC-4: Brain Unit on the PR reports 0 introduced reds.

## Out of Scope

- Wiring `lint-guides` into any CI, or the root option for the full-guide rules.

## Related

Parent: #201. Wiring gap: defect-note `a03b68c6`. Custody: the neomjs/neo#17806 cut manifest.

Live latest-open + closed sweep: latest open Brain issues + a search for `lintGuides` / `lint-guides` / `discoverGuideSeries` at 2026-10-29T16:12:32Z, with no equivalent. A2A: no claim in the unread queue at 16:09Z. Own assignments: #201, #1346, #1353, #1356. Structure map: N/A, since this changes one script function and its spec.

Retrieval Hint: "lintGuides discoverGuideSeries Brain root neo learn/guides hermetic fixture edge custody"

## Timeline

- 2026-10-29T16:12:49Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-29T16:12:51Z @neo-opus-ada added the `bug` label
- 2026-10-29T16:12:51Z @neo-opus-ada added the `ai` label
- 2026-10-29T16:12:51Z @neo-opus-ada added the `testing` label
- 2026-10-29T16:14:27Z @neo-opus-ada cross-referenced by PR #1361
- 2026-10-29T17:34:01Z @tobiu referenced in commit `1f90af5` - "fix(test): lintGuides' discovery arm walks a fixture tree, not neo's guides from the Brain root (#1360) (#1361)

The arm expected neo's real learn/guides files under the script's own
root, the Brain's, which has none, so discovery returned [] and the
arm failed. discoverGuideSeries now takes the root to walk (default
unchanged), and the arm builds two nested guides plus one learn/agentos
file in a temp tree. A no-recursion mutant fails it.

The cut manifest keeps lint-guides.mjs in the Brain (disposition
edge); wiring it to the Engine's guides is a separate question."
- 2026-10-29T17:34:02Z @tobiu closed this issue

