---
id: 562
title: Keep the Fleet roster available across dock layout changes
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-05T10:18:19Z'
updatedAt: '2026-10-05T11:06:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/562'
author: neo-gpt-emmy
commentsCount: 1
parentIssue: 505
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-05T11:06:10Z'
---
# Keep the Fleet roster available across dock layout changes

## Context

The operator reported that closing Fleet's dock tab removes the roster from the cockpit, leaving the main operating surface unavailable. On 2026-10-05 the operator explicitly selected the policy: “some views could never be closable. e.g. the roster.” This is the bounded safeguard within #505's view-reachability outcome.

## The Problem

FM declares the roster as a normal closable dock pane. The Engine's `closeItem` removes an item from the tree and catalog and retires its component. Group history and declared perspectives provide recovery primitives, but a recovery UI is not a prerequisite for protecting this essential pane.

A declaration-only fix leaves another entry: old or imported saved perspectives carry their own item flags, or may already omit the roster.

## The Architectural Reality

Design authority: the operator's 2026-10-05 direction above; #505's terminal predicate says each important view is reached “in one move.”

- `apps/agentos/view/fleet/cockpit/Container.mjs`: `panes.fleet` supplies the shared pane definition; `activatePerspective` admits declared layouts and full saved documents.
- `apps/agentos/util/CockpitPerspectives.mjs`: saved-layout normalization currently retires undeclared items, preserving surviving item flags.
- Engine `dashboard/dock/model/Operations.closeItem` rejects `closable: false`; `HeaderActionPolicy` hides the same action. No custom close handler or Engine default change is needed.
- Current source read: Institution `6561d1f17d64439d621ff7ccf291eb7f705a1099`; existing declaration and perspective-capture specs are the consumer test seams.

## The Fix

Declare the Fleet pane non-closable and enforce that current application policy when applying saved/imported perspectives. A saved document that contains Fleet has its stale close flag normalized to false. A document that omits Fleet is refused visibly before changing the current layout; offer a declared perspective as the recovery direction. Preserve all other dock actions and pane policies.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `panes.fleet.closable` | Operator's essential-pane policy | `false` in every declared perspective; Engine hides and refuses Close | Other dock actions retain their existing policy | Pane declaration JSDoc | Declaration/reducer test and visible header |
| Saved perspective admission | `activatePerspective` | Normalize Fleet's close flag before applying the saved document | Missing Fleet: visible refusal; existing layout retained | Method JSDoc | Legacy flags and missing-roster controls |

## Acceptance Criteria

- [ ] Fleet has no Close action and Engine `closeItem` refuses it in Overview, Focus and Review.
- [ ] Applying a saved/imported perspective with Fleet's close flag true or absent preserves non-closability; other pane flags remain unchanged.
- [ ] Applying a saved layout without Fleet produces a visible reason and leaves the live document unchanged.
- [ ] Roster movement, resizing, pop-out and the other header actions retain their existing behavior; optional panes are not globally locked or made non-closable.
- [ ] Focused unit and browser evidence covers the admission rules and the visible Fleet header.

## Post-Merge Validation

- [ ] Verify the protected roster on the next installed #12 candidate; installed view-quality acceptance remains with #505. Source/browser tests do not constitute that installed receipt.

## Out of Scope

The general Views/reopen interface, Group Undo/Redo controls, default-layout redesign, confirmation dialogs and Engine changes. Those can be designed independently; this ticket does not graduate a broader recovery proposal.

## Avoided Traps

A visual-only veto would leave the model operation admitted. Locking the roster would remove useful docking behavior. Reconstructing an arbitrary old layout's missing roster would invent placement policy; a visible admission refusal keeps the current usable layout.

## Related

Parent: #505. Adjacent: #507 (default arrangement), #12 (installed candidate). These scopes stay separate.

Decision Record impact: none — consumer policy using the existing dock item contract.
handoff: @neo-gpt-emmy

Live latest-open sweep: latest 20 open Institution issues checked immediately before filing; no equivalent. Exact all-state search and #505/#507 scope reads find no roster-protection leaf.
A2A in-flight sweep: last 30 messages, recent claim window, no roster-close claim.
MC sweep: “Fleet roster closed dock close missing roster perspective restore prior decision”; three results include today's operator discussion and older popup/perspective work, not a prior conflicting ruling.
Own-assignment sweep: #42 is the view-layer investigation and explicitly excludes repairs; body read.
Structure map: Brain command exited 0; this change owns existing Institution cockpit files, no new file or service.

Origin Session ID: 0ed3f43c-0e36-46b9-94fd-9ccb43b7edb1
Retrieval Hint: "Fleet roster close essential pane saved perspective closable"


## Timeline

- 2026-10-05T10:18:20Z @neo-gpt-emmy added the `bug` label
- 2026-10-05T10:18:21Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-05T10:18:21Z @neo-gpt-emmy added the `ai` label
- 2026-10-05T10:18:21Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-05T10:18:25Z @neo-gpt-emmy added parent issue #505
### @neo-gpt-emmy - 2026-10-05T10:25:13Z

Prescription checked: `apps/agentos/view/fleet/cockpit/Container.mjs` — owns the essential-pane declaration and saved-layout admission. The independent source read confirmed `closable: false` is lowered into the item and enforced by the Engine reducer; workspace-wide disable/presentation veto is broader and weaker. The saved path must normalize the flag and refuse a missing Fleet item before commit. This is a same-session authored ticket; the other creation checks remain current.

Epic preflight: https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-5992602187. Existing #42 is an investigation, not an implementation authority for this repair.

The new issue was not yet in the Native Graph at intake (`pre_brief_session` returned `NODE_NOT_FOUND`); the live issue, fresh Memory Core problem sweep and current source supplied the bounded brief. Implementation uses `codex/562-protect-fleet-roster` in the existing Institution checkout. Worktree creation was denied by the local execution policy; Candidate B's packaged artifact and recorded source revisions remain preserved, while this checkout now contains the new branch.

- 2026-10-05T10:25:13Z @neo-gpt-emmy cross-referenced by #505
- 2026-10-05T10:52:21Z @neo-gpt-emmy cross-referenced by PR #565
- 2026-10-05T11:06:10Z @tobiu referenced in commit `5f3275b` - "fix(agentos): keep the Fleet roster available (#562) (#565)"
- 2026-10-05T11:06:10Z @tobiu closed this issue

