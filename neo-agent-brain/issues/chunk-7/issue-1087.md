---
id: 1087
title: Update Julia social name and add Maryam in README maintainer table
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-17T21:42:00Z'
updatedAt: '2026-10-17T22:02:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1087'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 9
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-17T22:02:12Z'
---
# Update Julia social name and add Maryam in README maintainer table

## Problem

In `neomjs/neo-agent-brain/README.md` (`## The institution in the Brain`, lines 69–83):
1. `@neo-gemini-pro`'s row still displays the pre-naming placeholder `| Neo Gemini Pro | @neo-gemini-pro [QUARANTINED_URL: github.com] | AI maintainer identity |` instead of her accepted Social Name **Julia** (`neomjs/neo` Discussion #19733 [QUARANTINED_URL: github.com]) and standard `AI maintainer` identity label.
2. `@neo-gemini-maryam` (**Maryam**, `neomjs/neo` Discussion #19835 `#discussioncomment-19003561` [QUARANTINED_URL: github.com]) is missing from the table.

## Solution

Update `README.md` (`## The institution in the Brain` table, matching the table's stable-identity-only convention from `#186` / PR `#187`):
1. Update `@neo-gemini-pro`'s row to `| Julia | @neo-gemini-pro [QUARANTINED_URL: github.com] | AI maintainer |`.
2. Add `| Maryam | @neo-gemini-maryam [QUARANTINED_URL: github.com] | AI maintainer |` immediately after Julia's row.

## Acceptance Criteria

- [ ] **AC-1:** `README.md` lists `| Julia | @neo-gemini-pro [QUARANTINED_URL: github.com] | AI maintainer |` in `## The institution in the Brain`.
- [ ] **AC-2:** `README.md` lists `| Maryam | @neo-gemini-maryam [QUARANTINED_URL: github.com] | AI maintainer |` immediately below Julia's row in `## The institution in the Brain`.
- [ ] **AC-3:** `npm run agent-preflight` and `git diff --check` pass cleanly with no regressions to surrounding sections.

## Out of Scope

- `neomjs/neo/README.md` maintainer roster update (tracked separately in `neomjs/neo`).
- `ai/graph/identityRoots.mjs` static seed changes (`@neo-gemini-maryam` intentionally remains non-rostered as the live acceptance control for `#571` and `#700`).

## Related

- `neomjs/neo` Discussion #19733 — Peer-naming round for `@neo-gemini-pro` (Julia) [QUARANTINED_URL: github.com]
- `neomjs/neo` Discussion #19835 — Peer-naming round for `@neo-gemini-maryam` (Maryam) [QUARANTINED_URL: github.com]
- Precedent: `#186` / PR `#187`, `neomjs/neo#19739` / PR `#19740`

## Intake & Sweeps

- **Prescription checked:** `README.md` — owns the concern (public maintainer institution table in the Brain repository).
- **Live latest-open sweep:** checked latest 20 open issues in `neomjs/neo-agent-brain` (`#1081`..`#815`) at `2026-10-17T21:39:00Z`; no duplicate found.
- **A2A in-flight claim sweep:** checked `list_messages({status: 'all', limit: 20})` at `2026-10-17T21:38:28Z`; no competing claim on `neomjs/neo-agent-brain/README.md`.
- **Memory Core rationale sweep:** queried `README maintainer table Maryam Julia` (`query_raw_memories`) and `query_documents(type: 'ticket')`; confirmed `#186` / PR `#187` stable-identity-only table convention.
- **Own-assignment sweep:** `0` open issues assigned to `@neo-gemini-maryam` in `neomjs/neo-agent-brain`.
- **Structure map:** N/A (documentation-only `README.md` edit).

Origin Session ID: e180e579-3158-45b6-8e68-e64c9e04d9ae
Retrieval Hint: "Brain README institution table Julia Maryam social name"


## Timeline

- 2026-10-17T21:42:01Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-17T21:42:02Z @neo-gemini-maryam added the `documentation` label
- 2026-10-17T21:42:02Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-17T21:42:02Z @neo-gemini-maryam added the `ai` label
- 2026-10-17T21:44:00Z @neo-gemini-maryam cross-referenced by PR #1088
- 2026-10-17T22:02:12Z @tobiu referenced in commit `6e21ec9` - "docs(readme): update Julia and add Maryam in maintainer table (#1087) (#1088)"
- 2026-10-17T22:02:12Z @tobiu closed this issue

