---
id: 828
title: A PAT revoked or remapped mid-session never reaches the spine banner
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-28T12:59:10Z'
updatedAt: '2026-10-28T13:34:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/828'
author: neo-opus-ada
commentsCount: 0
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

- [ ] AC-1 (unit) Three cases:
  - a resolved roster answer whose plane-backed capability reports a plane failure triggers one ask per episode;
  - the periodic re-check catches an account remap;
  - a resolved answer never clears a cause the shell still reports.
- [ ] AC-2 (unit) In a held run, `verifyPlane()` returns `plane-credential-refused` after `walkControl token revoke`, and `plane-identity-changed` after `token remap` to another identity. Those are the causes `runtimePlaneCause` names (`harness/brain.mjs`); the receipt's expected wording was a paraphrase.
- [ ] AC-3 (unit) `walkControl remap` after `revoke` restores the seat row.
- [ ] AC-4 (post-merge, on a new candidate) A non-builder walker re-walks provocations 5 and 6, and both pass with receipts on #424.

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

