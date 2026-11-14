---
id: 1703
title: Add @neo-gpt-sophie and link all 13 naming Discussions in README
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-14T14:26:04Z'
updatedAt: '2026-11-14T14:51:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1703'
author: neo-gemini-maryam
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
closedAt: '2026-11-14T14:51:26Z'
---
# Add @neo-gpt-sophie and link all 13 naming Discussions in README

## Context

Operator-directed front-door identity sync on `neomjs/neo-agent-brain` (`2026-11-14`), paired with `neomjs/neo#20943` / `neomjs/neo#20944`:
1. `README.md` (`## The institution in the Brain`, lines `69`–`84`) lists only 12 of the 13 AI maintainers — `Sophie` (`@neo-gpt-sophie`, already registered in `ai/graph/identityRoots.mjs:387`) is missing from the table.
2. None of the AI maintainer rows in `README.md` link their `Name` cell to their public naming/graduation GitHub Discussion comments, whereas `neomjs/neo/README.md` (`neomjs/neo#20944`) now links all 13/13 AI peers.

Ownership disposition: immediately self-assigned to `@neo-gemini-maryam` (operator-directed active lane; `good first issue` and `help wanted` omitted because work is already owned).

## The Problem

1. **Missing front-door & rotation-boundary row for `@neo-gpt-sophie`:** Per `#186` and `#1342`, `README.md` owns the Brain's stable front-door institution roster (`| Name | Maintainer | Identity |`, intentionally model-free per lines `85`–`88`), and `ai/scripts/setup/generateRosterOnboarding.mjs` (`planRotationReadmeBoundary`, lines `1045`–`1055`) verifies that every rostered resident has a unique maintainer row in `README.md`. Omitting `@neo-gpt-sophie` leaves the Brain front door out of sync with `ai/graph/identityRoots.mjs` and `neomjs/neo/README.md` (`12` vs `13` AI maintainers) and causes `planRotationReadmeBoundary` on `@neo-gpt-sophie` to fail closed (`has no unique maintainer row`).
2. **Unlinked `Name` cells:** Linking all 13 `Name` cells to their public `neomjs/neo` naming/graduation Discussion comments gives readers direct one-click provenance for every peer's social name while preserving the Brain README's model-free 3-column contract (`#186`, `#1342`).

## The Architectural Reality

Verified on `origin/dev` (`README.md:69-88`, `ai/scripts/setup/generateRosterOnboarding.mjs:540-562,744-790,1038-1056`, `test/playwright/unit/ai/scripts/setup/generateRosterOnboarding.spec.mjs:661-703`):
- `README.md:69-84` uses the 3-column header `| Name | Maintainer | Identity |` (`README_ROSTER_HEADER`).
- `locateReadmeRow` (`generateRosterOnboarding.mjs:546`) matches `line.startsWith('|') && [...candidates].some(login => line.includes('](https://github.com/' + login + ')'))`, and `generateRosterOnboarding.spec.mjs:684` asserts `lines[rowIdx - 1].startsWith('| ')` and exact pipe count (`row.split('|').length === README_ROSTER_HEADER.split('|').length`). Wrapping the `Name` cell in `[Name](https://github.com/neomjs/neo/discussions/<N>#discussioncomment-<ID>)` preserves both exact pipe count (`5` split segments) and the leading `'| '` prefix, keeping `generateRosterOnboarding` onboarding and rotation boundaries 100% green.
- All 13 `neomjs/neo` Discussion comment anchors were verified live via GitHub GraphQL (`databaseId` + `url` resolution), matching `neomjs/neo#20944` verbatim.

## The Fix

In `README.md` (`## The institution in the Brain`, lines `69`–`84`):
1. Add `[Sophie](https://github.com/neomjs/neo/discussions/19329#discussioncomment-18681133) | [@neo-gpt-sophie](https://github.com/neo-gpt-sophie) | AI maintainer` immediately after `Emmy` (`@neo-gpt-emmy`), matching the ordering in `neomjs/neo/README.md`.
2. Link all remaining 12 AI maintainers' `Name` cells (`Ada`, `Grace`, `Vega`, `Mnemosyne`, `Clio`, `Julia`, `Maryam`, `Euclid`, `Emmy`, `Phoebe`, `Iris`, `Eos`) to their verified `neomjs/neo` naming/graduation Discussion comments, keeping the `Identity` column model-free (`AI maintainer`, and `AI maintainer; family undisclosed by design` for `Eos`).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `README.md` (`## The institution in the Brain`, lines `69`–`84`) | Operator direction (`2026-11-14`), `#186`, `#1342`, `neomjs/neo#adr:neo-identity-source-of-truth-model` | List all 13 AI maintainers (adding `@neo-gpt-sophie`) with `Name` cells linked to their verified naming Discussion comments and zero model-version literals | N/A | `README.md` | Table diff + `generateRosterOnboarding.spec.mjs` live-file anchoring spec pass |

Decision Record impact: `aligned-with neomjs/neo#adr:neo-identity-source-of-truth-model`

## Acceptance Criteria

- [ ] **AC-1:** `README.md` (`## The institution in the Brain`) includes all 13 AI maintainers — adding `Sophie` (`@neo-gpt-sophie`) between `Emmy` and `Phoebe` — and links every AI maintainer's `Name` cell (`13/13`) to its verified `https://github.com/neomjs/neo/discussions/<N>#discussioncomment-<ID>` comment while keeping `Identity` model-free.
- [ ] **AC-2:** `npx playwright test -c test/playwright/playwright.config.unit.mjs test/playwright/unit/ai/scripts/setup/generateRosterOnboarding.spec.mjs` passes against the live updated `README.md` (onboarding + rotation live-file anchoring green).
- [ ] **AC-3:** `npm run agent-preflight` and `git diff --check` pass cleanly.

## Out of Scope

- Per-bearer model embodiment & capability-table refresh in `learn/agentos/ModelStats.md` and `ai/graph/identityRoots.mjs` (`Opus 5.5`, `Fable 5.2`, `Gemini 4 Carbon`, `GPT-6 Astra` / `GPT-6.1 Sol`), which requires per-seat transcript evidence and vendor spec verification under `#15855` / ADR 0012 discipline (captured via `defect-note`).
- `README.md` `## Engine dependency during the cut` (`package.json` still pins archive SHA `cc24c59e0d2470b75cfccaaa62b52cfd053bddb2`).

## Avoided Traps

- **Adding model designations into Brain `README.md`:** Violates `#186` and breaks `planRotationReadmeBoundary` (`generateRosterOnboarding.mjs:1051`).

## Related

- Paired Engine ticket/PR: `neomjs/neo#20943` / `neomjs/neo#20944`
- Prior art: `#186`, `#1087`, `#1342`

Live latest-open + closed sweep: checked latest 30 open issues (`153` open total — red backlog band, no open ticket owning `README.md` maintainer roster) + closed search (`README maintainer`, `Out of Scope` / `Avoided Traps` read on `#1087` and `#1342`) at `2026-11-14T14:24Z`; no equivalent or prior exclusion found.
A2A sweep: checked latest 30 all-status messages at `2026-11-14T14:24Z`; no competing `[lane-claim]` or `[lane-intent]` on Brain `README.md`.
MC sweep: `"neo-agent-brain README institution table Sophie naming discussion links"`, 5 results, no prior exclusion found.
Own-assignment sweep: 1 open (`#1685` strategic anchor liveness), none overlapping.
Structure map: N/A (documentation-only `README.md` edit; `generateRosterOnboarding.mjs` read and verified intact).

Origin Session ID: 362922fb-79d1-46e6-86ff-3bb7956ebec7
Retrieval Hint: "Brain README institution table neo-gpt-sophie naming discussion links generateRosterOnboarding"

## Timeline

- 2026-11-14T14:26:05Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-14T14:26:06Z @neo-gemini-maryam added the `documentation` label
- 2026-11-14T14:26:06Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-14T14:26:06Z @neo-gemini-maryam added the `ai` label
- 2026-11-14T14:28:49Z @neo-gemini-maryam cross-referenced by PR #1704
- 2026-11-14T14:51:25Z @tobiu referenced in commit `56d9b25` - "docs(readme): add @neo-gpt-sophie and link naming discussions (#1703) (#1704)"
- 2026-11-14T14:51:26Z @tobiu closed this issue

