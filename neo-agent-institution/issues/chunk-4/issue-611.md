---
id: 611
title: Carry Brain 03da5025 in the next Fleet package so Mnemo can move
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - build
assignees:
  - neo-opus-vega
createdAt: '2026-10-08T23:07:52Z'
updatedAt: '2026-10-08T23:19:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/611'
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
blockedBy: []
blocking: []
closedAt: '2026-10-08T23:19:59Z'
---
# Carry Brain 03da5025 in the next Fleet package so Mnemo can move

## Context

The operator asked on 2026-10-08 at 23:0xZ to move Mnemo (`@neo-fable`) into Fleet Manager now, with a container and app rebuild approved. Grace's [fix-first ledger](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6066806393) gated that move on F1 (a Start installs each seat checkout's dependencies), which ships in Brain. The installed Candidate E bundles Brain `aab9e2a0`, which predates it.

## The Problem

Institution `dev@b78173f` pins Brain `aab9e2a0722c3032ddd81873b76308e27e4b1dfb`. Since then Brain merged:
- `4248494d` (#938): a Start installs each checkout's locked dependencies, so `.agents/skills` exists at first boot (F1).
- `c30d9215` (#941): a working pull route reads deliverable and armed (F3).
- `03da5025` (#939): the Fleet stops arming Claude Desktop on `osascript`, and the routes verb reads a seat's own pull route (F2).
A package built from today's pin carries none of them, so Mnemo's first Start would again open without skills.

## The Architectural Reality

As in #606/#607: `package.json` and `package-lock.json` own the installed contract, the cross-repository job in `.github/workflows/ci.yml` checks out its own Brain ref, and `test/playwright/visual/__screenshots__/baseline-inputs.txt` stamps the inputs. `harness/pack.mjs` stages the product with an explicit Brain runtime root.

## The Fix

Move the Brain pin to `03da5025f18ca00bf83dd9d7fbdae69d5667bbdd` in `package.json`, `package-lock.json`, the CI Brain checkout and the visual input stamp. The Engine stays at `e1b8fb0b`. No application change.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Brain pin (`package.json`, `package-lock.json`) | Brain `dev` at `03da5025` | Resolves exactly that commit | Resolution fails; no other checkout is used | — | Lock inspection, exact-head CI |
| CI Brain checkout (`.github/workflows/ci.yml`) | same | Checks out the same commit | CI failure is kept | — | Exact-head CI |

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1: package, lockfile, CI checkout and the visual stamp resolve Brain `03da5025` consistently.
- [ ] AC-2: Institution CI passes at the PR head on that pin.
- [ ] AC-3 (post-merge, installed): Candidate F, packed from merged `dev`, installs with a receipt on #12, and Mnemo's Start lands on a fresh root with dependencies installed and a consented memory import. Residual owner: neomjs/neo-agent-brain#571.

## Out of Scope

The Engine pin, the effort picker (#600), Mnemo's retain-aside and Start (#12, neomjs/neo-agent-brain#571), and any application change.

## Related

#606 / #607 (Candidate E's pin move) · #12 · neomjs/neo-agent-brain#571 · neomjs/neo-agent-brain#937 · neomjs/neo-agent-brain#768 · neomjs/neo-agent-brain#940

Live latest-open sweep: the latest 20 open Institution issues at 23:07Z; no equivalent.
A2A claim sweep: no competing claim; my 23:06Z broadcast announces this.
MC sweep: "installed FM app still bundles an old Brain; next Fleet package candidate pin move", 5 results: the #606/#607 and #430/#433 pin-move precedents; no contrary decision.
Own-assignment sweep: 1 open (#485), not overlapping.

Origin Session ID: 7d3fc6b2-cee6-4f82-ba2c-103729d4047a
Retrieval Hint: "Candidate F Brain pin 03da5025 Mnemo move install-at-Start"

## Timeline

- 2026-10-08T23:07:53Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-08T23:07:54Z @neo-opus-vega added the `enhancement` label
- 2026-10-08T23:07:54Z @neo-opus-vega added the `agent-os` label
- 2026-10-08T23:07:54Z @neo-opus-vega added the `ai` label
- 2026-10-08T23:07:54Z @neo-opus-vega added the `build` label
- 2026-10-08T23:09:19Z @neo-opus-vega cross-referenced by PR #612
- 2026-10-08T23:19:59Z @tobiu referenced in commit `b089d21` - "feat(fleet): carry Brain 03da5025 in the next Fleet package (#611) (#612)"
- 2026-10-08T23:20:00Z @tobiu closed this issue
- 2026-10-09T00:54:58Z @neo-fable cross-referenced by #571

