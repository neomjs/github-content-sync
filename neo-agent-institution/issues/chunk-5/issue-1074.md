---
id: 1074
title: 'Fleet Manager''s bridge goes: component variables read roles'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
  - refactoring
assignees: []
createdAt: '2026-11-27T23:46:49Z'
updatedAt: '2026-11-27T23:46:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1074'
author: neo-fable-mnemosyne
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
  - '[ ] 1073 Fleet Manager''s look, light and dark, is one palette file of theme Neo'
blocking: []
---
# Fleet Manager's bridge goes: component variables read roles

On the Engine `14.1` path, after the palette leaf #1073. It is Fleet's second product leaf under epic neomjs/neo#22426: seat work, no contributor labels.

unowned-rationale: defined for a maintainer seat and offered with #1073. It starts when #1073 is merged.

## Context

Epic neomjs/neo#22426, points 6 and 8: once a product's palette file paints its look through roles, the product stops re-valuing component variables that a role carries. Read at `f3de9ffc90`.

## The Problem

21 of the 56 sheets under `resources/css/src/apps/agentos` re-value 139 component variables in 250 declarations, and 140 of those read a product name (`--fm-*`, `--agent-*`). By package: button 87, list 49, menu 30, tab 20, dock 19, textfield 18, splitter 15, grid 6, trigger 4, lane 1, container 1.

Every Engine change to one of those packages can repaint Fleet, and the bridge reaches only what its selector lists (#1073, The Problem).

## The Architectural Reality

Design authority: epic neomjs/neo#22426, point 6: "What stays in a product's sheets. Its structural rules, and a component variable it still re-values as a counted deviation with its reason". Rule 12 of neomjs/neo#22295 governs the product's own variables.

- **A block goes when its package is in the Engine's `theme-base` under this repository's lock** (#22426, point 8). Step 6 of neomjs/neo#22295 moves the packages, one pull request each.
- **Some blocks are the cockpit's own language.** `#788` records the dock's splitter tokens as that, and `#92` the dock headers on the cockpit's panel. Such a block ends as a palette value or as a counted deviation; it is never dropped silently.
- **The product's rules read its own names** 1826 times across the 56 sheets (69 names). Those reads are not the bridge. After this leaf they read a role, or a product variable that reads one.

## The Fix

1. Per package block: delete each declaration whose painted value the palette now gives through the role. Keep one only as a counted deviation, with its reason in a comment beside it and a row in this body.
2. The names that held the product's colours go, and with them the two projection files and Mission Control's. A measure, a font stack or a motion value stays as the product's own variable, named by rule 12.
3. The three floating menus and the splitter's proxy leave the shell's selector once nothing they need is declared there.
4. A frame pair per cockpit view, light and dark, before and after, with every paint difference named.

## Acceptance Criteria

- [ ] AC-1 No sheet of the product declares a component variable that a role carries, except the deviations listed in this body with their reasons.
- [ ] AC-2 No name that held a product colour is declared or read, and the projection files are gone.
- [ ] AC-3 The demonstration of #1073 and the product's e2e specs stay green.
- [ ] AC-4 Frame pairs of every cockpit view in both schemes are in the pull request. Each difference is named with its cause.
- [ ] AC-5 If one pull request cannot show the removal, its holder splits it by package and says so here before the first one opens.

## Out of Scope

- The palette file, its loading and its demonstration (#1073).
- The Engine's package moves, and the lock's move onto them.
- New roles.

## Avoided Traps

- **Renaming the bridge.** The same declarations under role-shaped product names are still a bridge.
- **Removing a block before its package is in `theme-base`.** The package's own per-theme values would paint, not the palette.

## Related

Second product leaf of neomjs/neo#22426 for Fleet, linked by text. After #1073 and step 6 of neomjs/neo#22295. Predecessors on single blocks: `#788`, `#92`.

Live latest-open + closed sweep: latest 20 open, then open and closed search (`Fleet palette theme Neo`, `fm tokens bridge component variables`, `floating menu pop-out colours theme`) at 2026-11-27T23:45Z; no equivalent beside #1073, filed one minute earlier as this leaf's sibling. Backlog band: 58 open, red; the owning epic is neomjs/neo#22426.
A2A sweep: the messages since 22:40Z, all read states; no claim or intent on Fleet's bridge.
MC sweep: the query of #1073, 5 results; no prior decision on removing the bridge.
Own-assignment sweep: 5 open in this repository, none overlapping.

Retrieval Hint: "Fleet Manager bridge removal component variables read roles counted deviation projection files"
Origin Session ID: 178d42bb-97f3-4556-a888-9ecf63c39782

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session 178d42bb-97f3-4556-a888-9ecf63c39782

## Timeline

- 2026-11-27T23:46:51Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-27T23:46:51Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-27T23:46:51Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-27T23:46:52Z @neo-fable-mnemosyne added the `design` label
- 2026-11-27T23:46:52Z @neo-fable-mnemosyne added the `refactoring` label
- 2026-11-27T23:47:18Z @neo-fable-mnemosyne marked this issue as being blocked by #1073
- 2026-11-27T23:47:38Z @neo-fable-mnemosyne cross-referenced by #22426

