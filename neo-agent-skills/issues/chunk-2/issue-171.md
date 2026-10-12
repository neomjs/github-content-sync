---
id: 171
title: 'release-notes runbook: the cut''s one human step is merging the release PR'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-12T12:46:10Z'
updatedAt: '2026-10-12T13:46:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/171'
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
closedAt: '2026-10-12T13:46:22Z'
---
# release-notes runbook: the cut's one human step is merging the release PR

## Context

neomjs/neo#19653 (resolves neomjs/neo#19637) changes how the Engine is released. The old `publish.mjs` pushed straight to `dev` and needed a ruleset bypass. The new one has two phases:
- `prepare` commits the release to `release/v<version>` and opens a PR to `dev` that resolves a cut ticket;
- the operator merges that PR once `dev`'s required checks and CodeQL pass;
- `finish` squashes the merge onto `main` and creates the GitHub release on the merge commit, with the atomic-commit line in the release body;
- `--rehearse` reports the gates. A re-run resumes either phase.

The operator's requirement behind it: the AI maintainers must be able to trigger the release.

The release-notes runbook still describes the old flow. `.agents/skills/release-notes/references/release-notes-workflow.md` §6, read on `origin/dev` at 0.1.40:
- item 3 says `publish.mjs` appends the atomic line to the note after the squash;
- item 7 says the cut is human-only: "`publish.mjs` execution and the dev→main release line belong to the operator";
- §7's last checklist line posts an operator "publish is yours" handoff.

Once #19653 merges, an agent following §6.7 would hand the operator steps the agents are now meant to run.

## The Fix

Rewrite those three lines in place, with no new section and no net byte growth:
- **§6.3:** `publish.mjs finish` puts the atomic line in the GitHub release body. It parses the note as merged, then runs `gh release create` on the release PR's merge commit.
- **§6.7:** the cut's one human step is merging the release PR. Agents run `prepare --rehearse`, then `prepare --ticket <cut ticket>`, and `finish` after the merge.
- **§7, last line:** `publish.mjs prepare --rehearse --ticket <cut ticket>` shows every gate ✅. This replaces the "publish is yours" handoff.

Bump the package version.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `release-notes-workflow.md` §6.3 (atomic line) | neomjs/neo#19637 AC-4; `publish.mjs finish` in #19653 | Names the release body, not a `dev` commit, as where the atomic line lands | — | Yes (this file) | Text matches `finish` at #19653's merge |
| `release-notes-workflow.md` §6.7 (who runs the cut) | neomjs/neo#19637 AC-2 (operator requirement); `AGENTS.md` §critical_gates #1/#8 | Agents run `prepare`/`finish`; the operator merges the release PR, the only human step | A phase that stops resumes on re-run (#19637 AC-5) | Yes | Text matches the `publish.mjs` header at #19653 |
| `release-notes-workflow.md` §7 checklist, last line | neomjs/neo#19637 AC-6 | The rehearsal replaces the operator handoff comment | Any ❌ row blocks the cut | Yes | `prepare --rehearse` output, run live on #19653 |

Surface anchors checked by reading the §6/§7 text on `origin/dev` and the `publish.mjs` header and CLI at #19653's head (`04e7a93c7f`).

## Acceptance Criteria

- [ ] §6.3, §6.7 and §7's last line describe the two-phase flow, and the file does not grow.
- [ ] The package version is bumped, and the repo's own checks pass.
- [ ] Merged only after neomjs/neo#19653, so the runbook never describes a flow `dev` doesn't have.

## Out of Scope

The rest of the release-notes skill. Consumers pick the change up through their normal package bump.

## Related

neomjs/neo#19637, neomjs/neo#19653, neomjs/neo#19652 (the 13.3.0 cut ticket)

Sweeps:
- Live open issues: latest 20 here at 2026-10-12T12:45Z; none touches the release runbook.
- Own assignments: none here.
- MC: the #19637 lane's own records.

Origin Session ID: 02040656-932a-48c8-8894-6f43d0368663

## Timeline

- 2026-10-12T12:46:11Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-12T12:46:13Z @neo-opus-grace added the `documentation` label
- 2026-10-12T12:46:13Z @neo-opus-grace added the `enhancement` label
- 2026-10-12T12:46:13Z @neo-opus-grace added the `ai` label
- 2026-10-12T12:48:07Z @neo-opus-grace cross-referenced by PR #172
- 2026-10-12T13:02:34Z @neo-gpt-sophie cross-referenced by PR #19653
- 2026-10-12T13:11:13Z @neo-opus-grace referenced in commit `22b2cdb` - "docs(release-notes): name the preparation leaf and --body-file the repaired prepare takes (#171)

Aligns §6.7 and the §7 rehearsal with neomjs/neo#19653 after its review: the release PR resolves a preparation leaf with an author-written body, and the cut record stays open until npm and the Portal verify. §6.3 shortened so the file still shrinks."
- 2026-10-12T13:11:31Z @neo-opus-grace referenced in commit `344055b` - "docs(release-notes): keep the runbook below its pre-change size (#171)"
- 2026-10-12T13:46:22Z @tobiu referenced in commit `8ae6219` - "docs(release-notes): the cut's one human step is merging the release PR (#171) (#172)

* docs(release-notes): the cut's one human step is merging the release PR (#171)

Mirrors neomjs/neo#19653: publish.mjs prepare opens the release PR, the operator merges it, finish squashes it onto main and puts the atomic-commit line in the release body. Rewrites §6.3, §6.7 and the last §7 checklist line in place (-34 bytes). Bumps 0.1.40 -> 0.1.41.

* docs(release-notes): name the preparation leaf and --body-file the repaired prepare takes (#171)

Aligns §6.7 and the §7 rehearsal with neomjs/neo#19653 after its review: the release PR resolves a preparation leaf with an author-written body, and the cut record stays open until npm and the Portal verify. §6.3 shortened so the file still shrinks.

* docs(release-notes): keep the runbook below its pre-change size (#171)"
- 2026-10-12T13:46:22Z @tobiu closed this issue

