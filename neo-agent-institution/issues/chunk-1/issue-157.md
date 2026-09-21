---
id: 157
title: 'Engine pin 11 — dev@78c3e2f916: 14 commits, the measured-list item heights'
state: CLOSED
labels:
  - enhancement
  - ai
  - dependencies
assignees:
  - neo-fable-clio
createdAt: '2026-09-18T14:17:26Z'
updatedAt: '2026-09-18T14:49:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/157'
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
closedAt: '2026-09-18T14:49:15Z'
---
# Engine pin 11 — dev@78c3e2f916: 14 commits, the measured-list item heights

## Context

The Institution pins the engine at `github:neomjs/neo#70c2c94618` (pin 10, 2026-09-18). Engine `dev` is at `78c3e2f916` — 14 commits later (`git log --oneline 70c2c94618..origin/dev | wc -l`). #128 is blocked on one of them: neomjs/neo#18886 (a measured animated list sizes every item to its row).

## The Problem

#128's two config lines only give equal roster cards on an engine that carries `DomAccess.getNaturalRect`. The range holds no deleted or renamed `src/` / `resources/scss` file (`git diff --diff-filter=DR`), so the drift is behavioural. The eight `src`-touching commits, by what a cockpit consumer can feel:

- neomjs/neo#18886 — the Animate plugin writes the row height into every item; in fixed mode (`itemHeight: 126`, the roster today) that is the value the items already carried, so no pixel should move at this pin.
- neomjs/neo#18888 / neomjs/neo#18860 — `menu.List` lost 48 net lines: a submenu is owned by the level that opens it, and a level that closes under focus hands it back. **`apps/agentos/view/fleet/instances/MenuList.mjs` is the one FM consumer** — the drift check reads what it overrides.
- neomjs/neo#18873 — `grid.Container#scrollByColumns` rewritten geometrically (the registers and the mailbox grid navigate by arrow keys).
- neomjs/neo#18852 — a picker field owns its floating picker (`parentComponent`), focus moving into it no longer leaves the field.
- neomjs/neo#18878, neomjs/neo#18877, neomjs/neo#18876 — grid cell editing (Tab traversal, `selectText`, `aria-readonly` while editing): no FM grid edits today; the pooled-cell render path is shared.

## The Architectural Reality

As in #151: `package.json` + the lock entry are the pin; the visual-baseline stamp hashes the engine lock entry; `npm install` does not re-extract a `github:` SHA over an existing `node_modules/neo.mjs` (remove it first, verify by a code marker). New since pin 10: `build-all` runs in consumer mode (#153), 24 s, and a github-SHA install carries no engine `dist/` until it ran.

## The Fix

1. Drift check: FM-used engine identifiers ∩ the range — `instances/MenuList.mjs` first.
2. Move the pin + lock; marker: `grep -c getNaturalRect node_modules/neo.mjs/src/main/DomAccess.mjs` ≥ 1.
3. Follow each real consumer drift in the consumer; engine defects go to `neomjs/neo`.
4. `test-unit` (both modes), `test-components`, `test-e2e`, the NL battery, `test-visual` alone; goldens only from a full visual run, each drift read by eye; restamp as the last staged change.
5. `build-all` once at the new pin, consumer mode.

## Acceptance Criteria

- [ ] `package.json` + lock pin `neo.mjs` at the target SHA; the marker proves the installed tree is that SHA.
- [ ] Unit tier green in both CI modes; components + e2e green in CI.
- [ ] NL battery at the new pin: every red is either fixed, one of the six recorded host reds with identical errors, or shown to be independent of the pin. (Measured: two more arms are red — `AccountsConfigSurface` and `FleetNavFamilyPin`. Both name `neuralLink`, so the isolated CI job ignores them; neither matches `test-e2e:nl`'s file pattern; the Brain-contract job only lists. No script and no job ever executes them, and they rotted: the keeper-rail golden from 2026-08-27 has four tabs, the app has had five since the System view landed on 2026-09-05, and the harness-type expectation is one short. Neither can come from 14 engine commits in `menu` / `grid` / `form` / `list`; they get their own ticket.)
- [ ] Visual tier: no golden moves, or every drift is named with its cause and re-rendered from a full run; stamp fresh.
- [ ] `build-all` completes at the new pin in consumer mode.

## Out of Scope

#128 itself (the roster's measured rows — the next PR, on this pin). The Brain pin. New FM features.

## Avoided Traps

The three of #151, unchanged: no regolden from an isolated `-g` visual run; no "pin regression" verdict before the arm ran alone ×3 and against a pin-10 control (`npm install --no-save neo.mjs@github:neomjs/neo#70c2c94618`); no engine defect absorbed in a consumer constant.

## Related

#128 (blocked by this) · #151 (pin 10) · #153 · neomjs/neo#18886 · neomjs/neo#18888 · neomjs/neo#18873 · neomjs/neo#18852

Decision Record impact: none.

Live latest-open sweep: all open issues + an all-state `engine pin in:title` search read at 2026-09-18T14:16:52Z — prior pins (#81, #90, #98, #136, #151) are closed, none open. A2A in-flight sweep: newest inbox rows to 14:13Z, no claim on this surface.
MC sweep: `neo-agent-institution engine pin bump instances MenuList menu.List subclass broke after engine menu refactor`, 2 results, both session-boot noise, no prior decision found.
Own-assignment sweep: #10, #127, #128, #129 — #128 is the dependent, not a duplicate.

Origin Session ID: 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59
Retrieval Hint: "Institution engine pin 11 78c3e2f916 getNaturalRect menu.List MenuList drift consumer-mode build-all"


## Timeline

- 2026-09-18T14:17:26Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-18T14:17:27Z @neo-fable-clio added the `enhancement` label
- 2026-09-18T14:17:27Z @neo-fable-clio added the `ai` label
- 2026-09-18T14:17:28Z @neo-fable-clio added the `dependencies` label
- 2026-09-18T14:29:51Z @neo-fable-clio cross-referenced by PR #158
- 2026-09-18T14:49:15Z @tobiu referenced in commit `8b75572` - "Merge pull request #158 from neomjs/agent/157-engine-pin-11

chore(deps): engine pin 11 → dev@78c3e2f916 (#157)"
- 2026-09-18T14:49:16Z @tobiu closed this issue
- 2026-09-18T15:21:07Z @neo-fable-clio cross-referenced by #160
- 2026-09-18T15:46:37Z @neo-fable-clio cross-referenced by #163

