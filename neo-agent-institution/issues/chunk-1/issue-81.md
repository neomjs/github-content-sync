---
id: 81
title: 'Engine pin bump to dev@4e0e9dd8c3: the cockpit reads the reveal slide, the native-drag shield lift and the flat inline header before the rehearsal'
state: CLOSED
labels:
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-02T14:46:00Z'
updatedAt: '2026-09-02T15:38:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/81'
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
blockedBy: []
blocking: []
closedAt: '2026-09-02T15:38:31Z'
---
# Engine pin bump to dev@4e0e9dd8c3: the cockpit reads the reveal slide, the native-drag shield lift and the flat inline header before the rehearsal

## Context

Institution `dev` (`577e03bd`) pins `neo.mjs` at `fabb61f001` (PR #77, the third bump of the demo chain after #76 / PR #79). Engine `dev` has moved five commits since, and three of them change what the cockpit's docks look like or do — which the operator's rehearsal reads next to the WebStudio, which runs on current engine `dev`:

- `0c847b0b86` — neomjs/neo#18074 / PR #18079: a rail reveal slides in from its own edge (the root fades in place, its content enters from the strip side as one panel; `--dock-transition-duration-reveal` = the panel tier, 280ms).
- `b59a171562` — neomjs/neo#18087 / PR #18091: a native drag lifts the rail reveal's frame shield for its lifetime (a drop from a `nativeDragZone` row into an outside iframe no longer loses its payload while a reveal is open).
- `4e0e9dd8c3` — neomjs/neo#18095 / PR #18097: the inline tab header paints a flat surface with an inset hairline in both neo themes (the gradient band the cockpit inherited is gone; the gradient hook stays public).
- `29cbdb7467` (PR #18090) and `5a08fb9a81` (PR #18093) are test-only.

## The Problem

The cockpit cannot read any of it until the pin moves: the engine is consumed as a GitHub-SHA dependency, so `dev` advancing is invisible to the institution by design. Without the bump the rehearsal shows the old 8px reveal nudge and the gradient header band.

## The Architectural Reality

- `package.json` → `dependencies["neo.mjs"]` = `github:neomjs/neo#<sha>`; `package-lock.json` carries the resolved commit; both are visual-baseline stamp inputs (`test/playwright/visual/__screenshots__/baseline-inputs.json`, `npm run check-visual-baselines`).
- The same shape as PR #79 and PR #77: pin, lock, stamp — no product code.
- Battery: `test-e2e:nl` (34 witnesses) under the Brain root, with the two stated reds (`FleetGridScaleNL`, #78; the `FleetCockpitDockNL` presets arm, the headless FLIP-settle hold from #66) — any other red is the bump's.

## The Fix

Pin `neo.mjs` at `4e0e9dd8c3` (full SHA), refresh the lock, restamp, run unit + the NL battery under the Brain root, read the cockpit's reveal motion and inline header live in both themes.

## Acceptance Criteria

- [ ] AC-1 `package.json` + `package-lock.json` pin `neo.mjs` at `4e0e9dd8c3` (full SHA); `check-visual-baselines` green on the pushed tree.
- [ ] AC-2 Unit suite green; `test-e2e:nl` under the Brain root reads 32/34 with only the two stated reds (#78, the #66 hold).
- [ ] AC-3 Live read on the cockpit: a rail reveal enters as one panel from its strip side; the inline tab header paints flat with a hairline in neo-dark and neo-light, no gradient band.
- [ ] AC-4 The define-agent zone still materializes lazily from the rail tab and from the bootstrap CTA (PR #77's contract): `AddAgentJourneyNL` green.

## Out of Scope

- Any cockpit SCSS reconciliation the flat header may invite (the cockpit's own `Container.scss` reveal paint is a separate leaf if it drifts).
- Consumer-side adoption of #18091's body class beyond what the engine does on its own.

## Related

#74 / PR #77 (pin 3) · #76 / PR #79 (pin 1) · #10 (parent) · neomjs/neo#18074 · neomjs/neo#18087 · neomjs/neo#18095

Live latest-open sweep: checked the latest 20 open institution issues at 2026-09-02T14:44Z (#80 … #10); no pin-bump ticket open; an `engine pin` search returns the closed #76 / #39 and no open equivalent. A2A: no competing claim; the operator merged #77 and three engine PRs at 14:40Z. Structure-map gate: N/A (no `ai/` touch). Structural pre-flight: N/A (no new `.mjs`).

Origin Session ID: 91f83b9c-df95-4f72-a68f-d33f470792ac

Retrieval Hint: "institution engine pin bump 4e0e9dd8c3 reveal slide flat inline header rehearsal"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 91f83b9c-df95-4f72-a68f-d33f470792ac

## Timeline

- 2026-09-02T14:46:00Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-02T14:46:53Z @neo-fable-clio added the `agent-os` label
- 2026-09-02T14:46:53Z @neo-fable-clio added the `ai` label
- 2026-09-02T14:52:33Z @neo-fable-clio cross-referenced by PR #82
- 2026-09-02T15:28:45Z @neo-fable-clio cross-referenced by PR #83
- 2026-09-02T15:38:31Z @tobiu referenced in commit `64389fc` - "Merge pull request #82 from neomjs/agent/81-engine-pin-4e0e9dd8c3

chore(agentos): engine pin to dev@4e0e9dd8c3 — the cockpit reads the reveal slide, the drag-shield lift and the flat inline header (#81)"
- 2026-09-02T15:38:31Z @tobiu closed this issue
- 2026-09-04T09:27:32Z @neo-fable-clio cross-referenced by #90
- 2026-09-04T09:55:22Z @neo-fable-clio cross-referenced by PR #91
- 2026-09-04T09:57:11Z @neo-fable-clio cross-referenced by #92
- 2026-09-04T13:04:46Z @neo-fable-clio cross-referenced by #98
- 2026-09-12T10:10:47Z @neo-fable-clio cross-referenced by #120
- 2026-09-18T10:40:56Z @neo-fable-clio cross-referenced by #151
- 2026-09-18T14:17:27Z @neo-fable-clio cross-referenced by #157

