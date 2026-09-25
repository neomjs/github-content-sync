---
id: 486
title: Delete the eleven src/ai unit specs once the engine executes them
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
assignees:
  - neo-opus-vega
createdAt: '2026-09-25T12:42:04Z'
updatedAt: '2026-09-25T14:47:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/486'
author: neo-opus-vega
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
  - '[x] 19215 Eleven src/ai unit specs live only in the Brain, executed by nothing'
blocking: []
closedAt: '2026-09-25T14:47:44Z'
---
# Delete the eleven src/ai unit specs once the engine executes them

## Context

Eleven unit specs under `test/playwright/unit/ai/` test the engine's `src/ai/**` (three `Client*`, eight `InstanceService*`), importing it as a package. neomjs/neo#19215 adopts them into the engine's own unit tree, where `npm run test-unit` executes them on every engine PR. This ticket is the Brain half of that custody move: once the engine holds them, the Brain's copies go, so no subject has two test authorities.

The trigger is on #482: the #484 pin bump (`17b59aad` → `7f16355a`) turned nine arms of `ClientAgentDisconnect.spec.mjs` and `ClientWindowRegistration.spec.mjs` red, because neo `4802862b73` (neomjs/neo#18956) made `Client.isConnected` a getter and both specs assign it. The break was 20 days old before a manual full-suite run found it: none of the eleven is on `brain-unit.yml`'s run list (#201), so no CI on either side could fail on it.

The same run list is missing `test/playwright/unit/ai/scripts/setup/prepare.spec.mjs`, the seven arms that pin the install lifecycle #484 introduced (Residual-Owner on PR #484; #201 owns the class). The Brain PR that deletes the eleven adds that one line.

## The Problem

A spec whose subject lives in another repository can fail only when that repository's contract changes, and then only in a run nobody schedules. Keeping the copies after the engine adopts them would recreate the state neomjs/neo#17968 found on 2026-09-01: two authorities for one behaviour, one of them executed by nothing.

## The Architectural Reality

- Census (2026-09-25, `origin/dev`): `git grep -l 'neo\.mjs/src/ai/' origin/dev -- test/playwright/unit` → 11 files, all under `test/playwright/unit/ai/`, none with an engine spec of the same name at `7f16355a`.
- `.github/workflows/brain-unit.yml` — the "Run the move-first Brain smoke" step names the specs CI executes; `--list` above it only collects.
- `ai/scripts/setup/prepare.mjs` ↔ `test/playwright/unit/ai/scripts/setup/prepare.spec.mjs` (PR #484).

## The Fix

One Brain PR, opened after neomjs/neo#19215 merges:

1. Delete the eleven `test/playwright/unit/ai/{Client*,InstanceService*}.spec.mjs` files.
2. Add `test/playwright/unit/ai/scripts/setup/prepare.spec.mjs` to `brain-unit.yml`'s run list.

**Decision Record impact:** none. Custody precedent: neomjs/neo#17968 / PR #18002.

## Acceptance Criteria

- [ ] neomjs/neo#19215 is merged and its PR body names each of the eleven specs as adopted or as a duplicate of a restored engine arm, before this PR opens.
- [ ] `git grep -l 'neo\.mjs/src/ai/' HEAD -- test/playwright/unit` returns nothing at the PR head.
- [ ] `brain-unit.yml`'s run list names `prepare.spec.mjs`, and the PR's `unit` check log shows its seven arms executed.
- [ ] `npm run test-unit -- --list` at the PR head collects without an import error (the deletion removes no module another spec imports).

## Out of Scope

- The Brain's remaining retained-suite reds: missing-input, order-pollution and obsolete-surface classes stay with #201 / #191 / #89.
- Replacing the run list with full retained execution: #201's own ACs.

## Related

- neomjs/neo#19215 — the engine adoption this waits for.
- #482 / PR #484 — the pin bump and the full-suite receipt that surfaced the reds.
- #201 — the run list; Residual-Owner of PR #484's `prepare.spec.mjs` gap.
- neomjs/neo#17968 / PR #18002 — the first twelve restored.

Live latest-open sweep: checked the latest 20 open issues in neomjs/neo-agent-brain and neomjs/neo at 2026-09-25T12:40:25Z; no equivalent found. A2A in-flight sweep: the last 30 messages (12:04–12:35Z), no claim on this scope.
MC sweep: two `query_raw_memories` calls on the symptom's nouns; the only prior decision is neomjs/neo#17968's custody ruling, which this continues.
Own-assignment sweep: 6 open in neo-agent-brain (#471, #442, #417, #23, #64, #65), none on this surface.
Structure map: `npm run ai:structure-map -- --files --loc` run (exit 0); no row owns unit-test placement — deletions only.

Origin Session ID: d19add67-d33c-489d-99aa-27ad2782ed5e
Retrieval Hint: "Brain deletes src/ai unit specs after engine adoption; prepare.spec.mjs run list"

## Timeline

- 2026-09-25T12:42:04Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-25T12:42:06Z @neo-opus-vega added the `enhancement` label
- 2026-09-25T12:42:07Z @neo-opus-vega added the `ai` label
- 2026-09-25T12:42:07Z @neo-opus-vega added the `testing` label
- 2026-09-25T12:43:00Z @neo-opus-vega cross-referenced by #482
- 2026-09-25T12:43:01Z @neo-opus-vega marked this issue as being blocked by #19215
- 2026-09-25T12:54:06Z @neo-opus-vega cross-referenced by PR #19216
- 2026-09-25T14:11:01Z @neo-opus-vega cross-referenced by PR #489
- 2026-09-25T14:47:44Z @tobiu referenced in commit `251fafa` - "Merge pull request #489 from neomjs/vega/486-brain-drops-its-src-ai-spec-copies

test(unit): the Brain drops its eleven src/ai spec copies and runs prepare.spec.mjs in CI (#486)"
- 2026-09-25T14:47:44Z @tobiu closed this issue

