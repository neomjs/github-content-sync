---
id: 828
title: A PAT revoked or remapped mid-session never reaches the spine banner
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-28T12:59:10Z'
updatedAt: '2026-10-28T21:00:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/828'
author: neo-opus-ada
commentsCount: 1
parentIssue: 424
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-28T20:09:17Z'
---
# A PAT revoked or remapped mid-session never reaches the spine banner

## Context

On 2026-10-28, Julia walked row 5's peer-side half (#516 §1) on the installed build: Institution `20d89be`, Brain `1bf8bd91`, engine `477b65bd9e`. Her receipt is [#424 comment 6468175916](https://github.com/neomjs/neo-agent-institution/issues/424#issuecomment-6468175916). Two provocations fail:
- **5**, a PAT revoked while the shell runs;
- **6**, a PAT now admitted as another account.

In both, the spine keeps `fleet degraded`, the action stays Reconnect, and neither the pill nor the banner names the credential. #424's `Row state:` line reads `failed` on these two.

## The Problem

#446 (PR #447) gave the spine the connect card's words and Connect for a PAT the plane refuses while the shell runs. On the installed build that path never fires mid-session. The operator sees a generic degraded feed and a Reconnect that cannot help. H3 of the `[human]` half (#516, comment 6457088113) revokes a real PAT, and it is likely to fail the same way.

## The Architectural Reality

Design authority: "a PAT the plane refuses while the shell runs gets Connect, not Reconnect" (ROADMAP row 5, the #446 leaf).

Source at `20d89be`:
- `apps/agentos/util/PlaneCredentialCheck.mjs`: `ask()` queries the shell's `verifyPlane` once per failed roster-read episode, and only for `fleetConnectionState` `failed-upstream` or `refused`. Its JSDoc assumes that "a PAT that expires mid-session makes every roster read fail upstream". `settle()` clears `planeCause`.
- `apps/agentos/view/fleet/cockpit/LivenessController.mjs`: `loadRoster()` calls `ask()` only in its `catch`. A resolved read goes `admitRoster()` → `settle()`.

What the receipt observed:
- The roster read resolves `ok` after a revoke, with the plane-backed capability slots degraded, and after a remap, with every slot still `wired`. So `ask()` never runs, and every answer clears the cause again.
- In a held run (`NEO_HARNESS_SMOKE_PLANE=1`), `attachSmokePlane` in `harness/main.mjs` leaves `launchedPlane` null, so `verifyPlane()` answers `{cause: null}` even when asked.
- `walkControl.mjs remap` after `revoke` rewrites an empty `rows`, so a walk cannot restore or remap a seat after a revoke.

Decision Record impact: aligned-with the shell boundary of ADR 0034. The fix uses the existing `verifyPlane` capability and adds no `neoShell` key.

## The Fix

1. The cockpit asks the shell whether the stored credential still holds on three occasions: a rejected roster read (as today); a resolved read whose plane-backed capabilities report a plane failure; and a bounded periodic re-check, no more often than the plane's PAT cache TTL (300 s by default). `settle()` clears `planeCause` only when the shell's answer says the credential holds.
2. `attachSmokePlane` records the launched plane as `bootProductBrain` does, so `verifyPlane()` answers in a held run. #812 reverted a version that changed only the boot baseline; this one pairs with the ask path.
3. `walkControl.mjs remap` rebuilds the seat row when `rows` is empty after a `revoke`.

## Acceptance Criteria

- [x] AC-1 (unit) Three cases:
  - a resolved roster answer whose plane-backed capability reports a plane failure triggers one ask per episode;
  - the periodic re-check catches an account remap;
  - a resolved answer never clears a cause the shell still reports.
  - *Met by #832 (merged `a4420fdb1`): `planeCredentialCheck.spec.mjs` and `livenessLifecycle.spec.mjs`, CI green at `3142c29`. The review added a fourth case: the wired read that ends an episode asks once, so a restored PAT clears on the shell's answer.*
- [x] AC-2 (unit) In a held run, `verifyPlane()` returns `plane-credential-refused` after `walkControl token revoke`, and `plane-identity-changed` after `token remap` to another identity. Those are the causes `runtimePlaneCause` names (`harness/brain.mjs`); the receipt's expected wording was a paraphrase.
  - *Unit part met by #832: `planeConfig.spec.mjs` pins the launch record `attachSmokePlane` now composes. The held-run answers themselves need a running shell; AC-4's re-walk witnesses them.*
  - *Held-run answers witnessed in the re-walk receipt [6480226592](https://github.com/neomjs/neo-agent-institution/issues/424#issuecomment-6480226592): `window.neoShell.verifyPlane()` returned `{"cause":"plane-credential-refused"}` after the revoke and `{"cause":"plane-identity-changed"}` after the remap, then `{"cause":null}` on each restore.*
- [x] AC-3 (unit) `walkControl remap` after `revoke` restores the seat row.
  - *Met by #832: `walkControl.spec.mjs` › a remap after a revoke re-admits the revoked token.*
- [x] AC-4 (post-merge, on a new candidate) A non-builder walker re-walks provocations 5 and 6, and both pass with receipts on #424.
  - *Met: Julia re-walked both on the candidate Institution `a4420fdb1` / Brain `05e428558e` / engine `95e63df6db` (receipt [6480226592](https://github.com/neomjs/neo-agent-institution/issues/424#issuecomment-6480226592), 2026-10-28 20:24–20:41Z). Each showed the credential and Connect, and each restored.*

## Out of Scope

Provocations 1 and 4 at boot, which a held run cannot provoke because it always starts its own fixture plane, and the `[human]` half's other rows.

## Related

#424 (the row's epic) · #446 / PR #447 · #516 · #812 · receipt 6468175916

Live latest-open sweep: the latest 20 open issues at 2026-10-28T12:58Z; no equivalent. A2A in-flight sweep: the last 30 messages, all read states; no claim on this scope. Memory Core rationale sweep: no prior decision on mid-session credential detection beyond #446. Own-assignment sweep: #424 and #516 are this row's epic and walk. Not `good first issue`: owned by the row's steward and on row 5's critical path.

Origin Session ID: 3cb14c72-da24-49bb-83e5-010490fecc56
Retrieval Hint: "PAT revoked mid-session spine banner Reconnect PlaneCredentialCheck asks only on a failed roster read"





## Timeline

- 2026-10-28T12:59:11Z @neo-opus-ada added the `bug` label
- 2026-10-28T12:59:12Z @neo-opus-ada added the `agent-os` label
- 2026-10-28T12:59:12Z @neo-opus-ada added the `ai` label
- 2026-10-28T12:59:16Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-28T12:59:20Z @neo-opus-ada added parent issue #424
- 2026-10-28T12:59:56Z @neo-opus-ada cross-referenced by #424
- 2026-10-28T13:27:59Z @neo-opus-ada cross-referenced by #516
- 2026-10-28T13:45:35Z @neo-opus-ada cross-referenced by PR #832
- 2026-10-28T15:38:43Z @neo-opus-ada referenced in commit `8726ed8` - "test(visual): stamp the baseline inputs after the cockpit credential-check change, goldens unchanged (#828)"
### @neo-opus-ada - 2026-10-28T16:28:07Z

## Handover (session sunset 2026-10-28 ~16:1xZ, Ada continues)

**State:** PR #832 is open and green at `8726ed8` (15 checks pass, merge state `CLEAN`). Its primary reviewer is Julia (`@neo-gemini-julia`), requested 16:01Z.
- `e940b2b`: the three fixes and their unit coverage.
- `8726ed8`: the visual-baseline stamp, after a Darwin re-capture showed no golden delta.

**Pickup, in order:**
1. Answer Julia's review if she requests changes; it is a full-form review of `apps/agentos`.
2. After the operator merges, build a walk candidate from Institution `dev` (`fm-build-walk.sh`). Run its packaged rehearsal for both families, then send the packet to Julia so she can re-walk provocations 5 and 6 (AC-4, receipts on #424).
3. The deadline is before **Fri 10-30 ~13:00Z**. In the operator's sitting, H4 installs that candidate and H3 runs on it (#516 step list, comment 6457088113).

**Work surface:** the branch `ada/828-mid-session-credential-check` was built in a scratch `--local` clone, because the Institution clone stays detached at `7acb9e5` for the film rig until tonight.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-10-28T16:46:26Z @neo-opus-ada referenced in commit `74a78d3` - "fix(agentos): a wired read that ends a credential episode asks the shell once more, so a restored PAT clears on its answer, not the next re-check (#828)

Julia's review challenge on #832: observe() ended the episode without asking, so after a restore the banner kept plane-credential-refused until the next 300 s re-check. Before #832, settle() cleared at once on any resolved read; the episode exit now asks instead, and only the shell's answer clears. One extra verifyPlane probe per episode."
- 2026-10-28T16:46:27Z @neo-opus-ada referenced in commit `3142c29` - "test(visual): stamp the baseline inputs after the episode-exit ask, goldens unchanged (#828)"
- 2026-10-28T20:09:17Z @tobiu referenced in commit `a4420fd` - "feat(agentos): a PAT revoked or remapped mid-session reaches the spine banner (#828) (#832)

* feat(agentos): a PAT revoked or remapped mid-session reaches the spine: the shell is asked on a degraded presence read and a periodic re-check, and only its answer clears the cause (#828)

* test(visual): stamp the baseline inputs after the cockpit credential-check change, goldens unchanged (#828)

* fix(agentos): a wired read that ends a credential episode asks the shell once more, so a restored PAT clears on its answer, not the next re-check (#828)

Julia's review challenge on #832: observe() ended the episode without asking, so after a restore the banner kept plane-credential-refused until the next 300 s re-check. Before #832, settle() cleared at once on any resolved read; the episode exit now asks instead, and only the shell's answer clears. One extra verifyPlane probe per episode.

* test(visual): stamp the baseline inputs after the episode-exit ask, goldens unchanged (#828)"
- 2026-10-28T20:09:17Z @tobiu closed this issue

