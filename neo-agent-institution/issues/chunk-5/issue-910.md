---
id: 910
title: Retire the legacy thought alias after the Brain pin moves
state: OPEN
labels:
  - enhancement
  - good first issue
  - help wanted
  - agent-os
  - ai
  - refactoring
assignees: []
createdAt: '2026-11-11T01:23:53Z'
updatedAt: '2026-11-13T22:37:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/910'
author: neo-fable-clio
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
  - '[x] 908 Memories reader accepts decisionContext alongside legacy thought'
blocking: []
---
# Retire the legacy thought alias after the Brain pin moves

## Context

neomjs/neo-agent-brain#1582 renamed the middle field of a turn from `thought` to `decisionContext` on every Memory Core read surface. `#908` (PR `#909`) lets the Fleet Manager's Memories reader accept both shapes: `AgentOS.model.SessionTurn` declares the field `decisionContext` with `mapping: 'thought'`, and the engine reads a mapping only when a row lacks the field's own key (`RecordFactory#assignDefaultValues`).

The alias is a bridge for one window. This repository pins the Brain at `690dc8df` (`package.json`), which predates the rename's commit on Brain `dev`, `4f3b8fbb`. A plane provisioned from the pin therefore still emits `thought`, while the team's plane emits `decisionContext` after its next cut.

- Live latest-open + closed sweep: checked latest 20 open (51 open, red band; `#908`..`#649`) + closed search (`thought decisionContext alias SessionTurn Memories reader`; `legacy alias retire` in titles) at 2026-11-11T01:22:31Z; no equivalent or prior exclusion found. No owning ticket to fold into: `#908` closes on merge, and `#505` already carries 25 sub-issues.
- A2A in-flight claim sweep: checked the 14 newest messages across all read states (00:43Z to 01:23Z on 2026-11-11); no overlapping `[lane-claim]` or `[lane-intent]`.
- MC sweep: not run from this seat. A Fable seat reads no memory rows until the plane carries the fix of neomjs/neo-agent-brain#1580. KB ticket sweep instead (`ask_knowledge_base`, type `ticket`, 4 hits, none on the alias).
- Own-assignment sweep: 4 open in this repository (`#351`, `#505`, `#507`, `#889`), none on this surface besides `#505`.

## The Problem

A bridge without a retirement trigger stays. Once the pinned Brain contains `4f3b8fbb`, no supported plane emits `thought`. The alias, any other read of that key, and the fixtures shaped like the old wire then describe a producer that no longer exists, and the next reader of `SessionTurn` has to find out whether the fallback still matters.

## The Architectural Reality

- `apps/agentos/model/SessionTurn.mjs`: the `mapping: 'thought'` line on the `decisionContext` field (line 62 at PR `#909`'s head `1c7a00e4`; not on `dev` until that PR merges).
- At that head a second read of the key sits in `apps/agentos/view/fleet/memories/ReaderComponent.mjs` (`turnFieldValue`). The review of PR `#909` asks to remove it there; whatever remains at merge is in scope here.
- Fixtures that still feed the old key: `test/playwright/unit/apps/agentos/view/fleet/memories/reading.spec.mjs` (`TURN`, and the legacy rows of the dual-key tests), `test/playwright/unit/apps/agentos/view/fleet/memories/container.spec.mjs:293`, `test/playwright/component/apps/agentos/MemoriesGridSeam.spec.mjs:119-120`.
- The trigger is the `neo-agent-brain` dependency in `package.json`. In a Brain checkout, `git merge-base --is-ancestor 4f3b8fbb <pinned commit>` decides it.
- Any change under `apps/agentos/**` needs the visual baseline stamp refreshed (`buildScripts/checkVisualBaselines.mjs`).

## The Fix

After the pin contains `4f3b8fbb`:

1. Delete the `mapping` line and the fallback clause of the field's JSDoc in `SessionTurn.mjs`.
2. Delete every remaining read of a turn row's `thought` key under `apps/agentos`.
3. Flip the fixtures above to `decisionContext` and drop the tests that exist only for the old shape.
4. Re-stamp the visual baseline inputs.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `AgentOS.model.SessionTurn`, field `decisionContext` (the surface as PR `#909` merges it) | neomjs/neo-agent-brain#1580, `#908` | Reads the wire's `decisionContext` only | None: a row without it reads as missing, and the reader names it with the record id | Field JSDoc | Playwright unit spec |

## Decision Record impact

`none`

## Acceptance Criteria

- [ ] Precondition: the Brain commit pinned in `package.json` contains `4f3b8fbb`. This ticket does not move the pin.
- [ ] No code under `apps/agentos` reads a key named `thought` from a turn row. The search names the tree it ran against and carries a positive control.
- [ ] The Memories unit and component specs feed `decisionContext` only, and pass.
- [ ] A turn row without `decisionContext` still renders the missing-field sentence with its record id.
- [ ] `npm run check-visual-baselines` passes.

## Out of Scope

- Moving the Brain pin.
- The Agent Detail's "Thought stream" pane and its name.
- The Brain's stored metadata key, which stays `thought`.

## Avoided Traps

- **Removing the alias when the team's plane is cut:** the pin decides what a freshly provisioned plane emits, the team's plane does not.
- **Removing it together with the pin bump without the search:** a read of the old key outside the model would then fail silently, as a missing block.

## Related

- Blocked by #908
- Related: #505, neomjs/neo-agent-brain#1580, neomjs/neo-agent-brain#1582

Ownership: unowned-rationale: Ready for a contributor once the precondition holds; maintainers provide planning and review.

Origin Session ID: 0b200952-7e03-4bbc-9790-106b51a875d9
Retrieval Hint: "SessionTurn decisionContext mapping thought alias retirement Brain pin 4f3b8fbb"

## Timeline

- 2026-11-11T01:23:54Z @neo-fable-clio added the `enhancement` label
- 2026-11-11T01:23:55Z @neo-fable-clio added the `good first issue` label
- 2026-11-11T01:23:55Z @neo-fable-clio added the `agent-os` label
- 2026-11-11T01:23:55Z @neo-fable-clio added the `ai` label
- 2026-11-11T01:23:55Z @neo-fable-clio added the `refactoring` label
- 2026-11-11T01:24:03Z @neo-fable-clio marked this issue as being blocked by #908
- 2026-11-11T01:32:39Z @neo-gemini-julia cross-referenced by PR #909
- 2026-11-13T22:37:06Z @neo-gpt-emmy added the `help wanted` label
- 2026-11-13T22:37:42Z @neo-gpt-emmy cross-referenced by #936

