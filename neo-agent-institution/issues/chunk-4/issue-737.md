---
id: 737
title: The installer refuses an update after an unclean FM exit; ADR 0034 §2.5 requires it to complete
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-19T02:07:47Z'
updatedAt: '2026-10-19T03:11:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/737'
author: neo-opus-vega
commentsCount: 0
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-19T03:11:03Z'
---
# The installer refuses an update after an unclean FM exit; ADR 0034 §2.5 requires it to complete

## Context

On 2026-10-19 the install of the FM candidate (Institution 2f9056a, Brain e4886d2a) failed four times at the custody step with `Launch admission journal refused: handoff-incomplete`. The installed FM (Brain 9cd1528b) refused its own continuity handoff at survivor-validation (`seat neo-gpt-emmy is not a live original Start`): Emmy's seat had been stopped from the FM, but its row keeps `cleanupUnresolved`, which no later boot clears. That old build cannot be changed, so no install could ever pass it. It was the third failed FM update within 24 hours.

ADR 0034 §2.5 lists this case among the required installed witnesses: *"Repeat the update with one seat's server failing its proof, and again after an unclean issuer exit: each completes, the first revoking only that server's grant, the second carrying the running generations stale."*

## The Problem

`prepareInstallContinuity().hold()` (`harness/install.mjs`) accepts only a sealed handoff (`holdLaunchAdmissionHandoff`) or a vacant journal. An unsealed journal whose issuer has exited is refused, and `install.spec.mjs` asserts that refusal ("Quit with an unclean exit refuses before staging"). So an installed FM whose handoff has a bug can never be replaced by the build that fixes it.

## The Fix

When the predecessor's handoff is not proved, hold its unsealed journal under the issuer's own exclusion lease. Taking that lease proves no issuer is alive, and the hold re-verifies the record before every slot mutation. Skip successor recovery: an unclean record grants no authority, and the successor crosses its generations stale on boot, as every boot after an unclean exit already does. A live issuer, a changed record or a successor issuer still refuses.

## Acceptance Criteria

- [ ] An update after an unclean issuer exit completes: the custody set is unchanged, the journal is unchanged, and successor recovery is not run.
- [ ] A live issuer still refuses the update, and so does a changed issuer.
- [ ] A sealed handoff still takes the existing path, recovery included.
- [ ] Installed witness: the 2026-10-19 install completes over Brain 9cd1528b and the new FM boots.

## Out of Scope

- Emmy's sticky `cleanupUnresolved` row (Brain follow-up).
- The 15 s Fleet-ready window and the missing roster reload (#734).
- Reopening the FM after a refused install (#733).

Parent: #732.


## Timeline

- 2026-10-19T02:07:48Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-19T02:07:48Z @neo-opus-vega added the `bug` label
- 2026-10-19T02:07:49Z @neo-opus-vega added the `ai` label
- 2026-10-19T02:07:51Z @neo-opus-vega added parent issue #732
- 2026-10-19T02:10:41Z @neo-opus-vega cross-referenced by PR #738
- 2026-10-19T02:56:23Z @neo-opus-vega referenced in commit `16506b6` - "fix(harness): the unclean-predecessor hold's summary carries no decision-record reference (#737)"
- 2026-10-19T03:11:03Z @tobiu referenced in commit `48c4c5c` - "fix(harness): an update after an unclean FM exit completes; the successor crosses its record stale (#737) (#738)

* fix(harness): an update after an unclean FM exit completes; the successor crosses its record stale (#737)

The installer refused an unsealed admission journal whose issuer had exited, so an
installed FM with a broken handoff could never be replaced. ADR 0034 §2.5 requires
that update to complete. On handoff-incomplete the installer now holds the unsealed
record under the journal's own issuer lease, which is free only once the issuer has
exited, re-verifies it before each slot mutation, and skips successor recovery.

* fix(harness): the unclean-predecessor hold's summary carries no decision-record reference (#737)"
- 2026-10-19T03:11:03Z @tobiu closed this issue
- 2026-10-19T03:13:56Z @neo-fable-mnemosyne cross-referenced by #191

