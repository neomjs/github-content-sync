---
id: 445
title: 'epic-resolution: all subs closed means close or file the missing subs'
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-11-24T20:52:40Z'
updatedAt: '2026-11-25T12:33:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/445'
author: neo-opus-ada
commentsCount: 1
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
---
# epic-resolution: all subs closed means close or file the missing subs

## Context

The operator ruled on 24 Nov, after closing neomjs/neo#21033: "IF all subs are resolved for an epic, literally 2 options are acceptable: CLOSING it, or if not done CREATING NEW SUBS that must get resolved to reach completion." He also noted that comment threads arguing to keep such epics open burn weekly limits and bloat every context that reads them.

An audit at 2026-11-24T20:45Z found 33 open parents whose subs were all closed: 8 in neo, 17 in Brain, 6 in Institution and 2 in Skills.

## The Problem

`epic-resolution-workflow.md` permits the state the ruling retires:
- **§4 keeps `RECOMMEND_KEEP_OPEN` for `BLOCKER` and recoverable `LOST` rows even when no sub is open.** "Recoverable in existing subs" cannot apply once every sub is closed.
- **§4 `RECOMMEND_CREATE_MISSING_SUBS` lists gaps "or A2A the team to volunteer. An owner who accepts files via `/ticket-create`".** The gaps then sit in comments with no sub.
- **§1 skips "non-epic issues"**, yet most of the 33 are plain parents.
- **§2's claim comment and §5's long template** add comments to every closeout.

## The Architectural Reality

- The workflow's §§1, 2, 4 and 5, and the verdict list in the `SKILL.md` description.

Design authority: "IF all subs are resolved for an epic, literally 2 options are acceptable: CLOSING it, or if not done CREATING NEW SUBS that must get resolved to reach completion." — the operator, 2026-11-24, after closing neomjs/neo#21033.

## The Fix

- **§4:** when every sub is closed, the verdict is `CLOSE_COMPLETED`, `CREATE_MISSING_SUBS` or `RETIRE_OR_SUPERSEDE`. For `CREATE_MISSING_SUBS`, the reviewer files each missing sub in the same turn, owned or with an unowned-rationale, and links it. `KEEP_OPEN` applies only while an open sub carries the residual.
- **§1:** the trigger covers any open parent issue whose sub-issues are all closed.
- **§2:** the assignee runs the closeout with no claim comment; the claim applies only to a non-assignee.
- **§5:** the verdict comment is the matrix plus one line of terminal action.
- **`SKILL.md`:** the description drops "keep open".

## Acceptance Criteria

- [ ] No path in the workflow keeps a parent open when all of its subs are closed and no new sub has been filed.
- [ ] Plain parents are covered.
- [ ] The workflow plus `SKILL.md` shrink in bytes (`lint-skill-corpus.mjs --base origin/dev`).
- [ ] The rule is named once in `SKILL.md` as the trigger.

## Out of Scope

The mechanical trigger: a Brain sub of neomjs/neo-agent-brain#759, filed next.

## Related

#231 (the reviewer closes a retired epic), #298 (artifact bloat), neomjs/neo#21033.

Live latest-open + closed sweep: checked the latest 20 open issues plus an open and closed search (`epic-resolution keep open subs closed`); #231 and #13 are related and closed, with no exclusion. 43 open (orange band); no owning ticket, so this one is filed.
A2A in-flight claim sweep: no claim on epic-resolution.
MC sweep: earlier closeout lanes, no ruling against this.
Own-assignment sweep: #298 is adjacent (bloat in emitted artifacts), not this rule.

ownership: self-assigned.

Origin Session ID: 36593ade-49d5-4bf2-92e6-6e86d12beb9b
Retrieval Hint: "epic-resolution all subs closed close or create missing subs keep open retired operator ruling"


## Timeline

- 2026-11-24T20:52:40Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-24T20:52:42Z @neo-opus-ada added the `enhancement` label
- 2026-11-24T20:52:42Z @neo-opus-ada added the `ai` label
- 2026-11-24T20:52:55Z @github-actions added the `needs-design-authority` label
- 2026-11-25T08:52:33Z @neo-opus-ada cross-referenced by #463
### @neo-opus-ada - 2026-11-25T12:32:33Z

**Contract Ledger (T3)**, posted before the PR opens.

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `epic-resolution` trigger (workflow §1, `SKILL.md` description) | Operator ruling 2026-11-24 (Design authority above) | The workflow fires on any open parent issue, epic-labeled or not, whose sub-issues are all closed. | A sub merging while others stay open does not fire it. | The workflow, `SKILL.md`, and the manifest mirror | skill-corpus lint (manifest coherence, description ≤ 220 B) |
| `epic-resolution` verdict set (workflow §3.5, §4, §3.5 audit exit line) | Same | When every sub is closed, the verdict is `CLOSE_COMPLETED`, `CREATE_MISSING_SUBS` or `RETIRE_OR_SUPERSEDE`. `CREATE_MISSING_SUBS` files and links each sub in the same turn. | `KEEP_OPEN` holds only while an open sub carries the residual (`LOST`, `BLOCKER`). | The workflow and `source-discussion-reconciliation-audit.md` | The diff |
| Closeout comment (workflow §2, §5) | Same, and #298 (artifact bloat) | The assignee posts no claim comment. The verdict comment is the matrix plus one terminal-action line, plus the Origin Session ID. | A non-assignee on an unassigned parent still claims, and updates the claim into the verdict. | The workflow | The diff |

Surface-anchor V-B-A: every section named above exists at `origin/dev` `f82c625`+ (read in full before the edit). The verdict names are unchanged.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-25T12:32:36Z @github-actions removed the `needs-design-authority` label
- 2026-11-25T12:33:05Z @neo-opus-ada cross-referenced by PR #466

