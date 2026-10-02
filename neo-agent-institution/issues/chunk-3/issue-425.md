---
id: 425
title: A failed plane-attach boot says why in the connect card's words
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-02T09:05:52Z'
updatedAt: '2026-10-02T11:36:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/425'
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
closedAt: '2026-10-02T11:34:17Z'
---
# A failed plane-attach boot says why in the connect card's words

## Context

This covers FM v1 ROADMAP row 5, steps 4 (stale saved plane) and 5 (expired or wrong PAT, at boot), per the [row-5 source audit](https://github.com/neomjs/neo-agent-institution/issues/335#issuecomment-5948240484).

Measured 2026-10-02 at Brain `dev@cbd11cb`: a fleet child pointed at a saved plane with nothing listening exits 1 with this line:

`[fleet] plane mode refused (http://127.0.0.1:9): plane unreachable (TypeError) — fix fleet.planeBase / fleet.planeBearer, or empty the base for in-process mode.`

## The Problem

The shell types every plane-attach boot refusal as `plane-refused` (#225). The banner then reads **`plane refused`**, titled "The plane refused this shell — connect it again · " followed by the line above.

For a plane that is merely gone, "refused" names the wrong state. The quoted advice points an installed operator at config leaves they cannot reach. A PAT the plane rejects takes the same path, and nothing names the credential.

Meanwhile the connect card already names each case in product words.

## The Architectural Reality

- **The refusal path.** In `harness/brain.mjs`, `fleetReadyOrPlaneRefusal` calls `planeRefusal(lastLine)`, which returns `code: 'plane-refused'` with the line quoted, never parsed. `bootFailureCause` picks up that code. `harness/main.mjs` settles the boot as `{cause, error, up: false}` and logs `HARNESS_BRAIN_BOOT_FAILED`. `harness/appLifecycle.mjs` records the cause (`CAUSE_SEVERITY`, `recordBrainCause`).
- **The banner.** The brain-health wire carries `daemon.cause` and its reason to the cockpit's `StateProvider`. `apps/agentos/util/SpineBanner.mjs` `deriveSpineBanner` turns `plane-refused` into its verdict with action `connect-plane`. The banner's button then reaches the plane card through `Controller.onSpineAction`.
- **The probe.** In `harness/planeConfig.mjs`, `readPlaneConfig` returns the stored `{planeBase, bearer, identity}`. `probePlaneCredential({planeBase, bearer})` answers `accepted | rejected | not-a-plane | no-identity | unreachable` (with the identity when accepted). It writes nothing and is bounded by an 8 s timeout.
- **The words.** `apps/agentos/view/PlaneSetupPanel.mjs` `reasonText` holds the product's sentence for each verdict.

## The Fix

When a plane-attach boot ends in `plane-refused` and the stored record supplied that launch (its base and its PAT reached the fleet child, `launchedPlaneRecord`), the shell probes the record once with `probePlaneCredential` and types the cause by the verdict. A launch that inherited its base or its PAT keeps the generic refusal: the record does not describe it.

| Probe answer | Cause | The banner says (the card's own sentence, with the plane's address) |
|---|---|---|
| `unreachable` | `plane-unreachable` | No plane answered at that address |
| `rejected` | `plane-credential-refused` | The plane refused that PAT |
| `not-a-plane` | `plane-not-a-plane` | That address is not a Neo plane |
| `accepted`, identity differs from the stored one | `plane-identity-changed` | The PAT now names another account |
| `accepted` with the same identity, `no-identity`, a throw, or no answer within the overall deadline | `plane-refused` | today's verdict, unchanged |

A request the probe itself times out (8 s each) answers `unreachable`, the way the card reads it.

- Every typed verdict keeps Connect as its action.
- The child's line stays in the main log only. The banner title carries no config-leaf advice.
- `CAUSE_SEVERITY` lists the new causes on `plane-refused`'s tier.
- The `StateProvider` docblock names them.

## Contract Ledger

| Target surface | Source of authority | Behavior | Failure / fallback | Evidence |
|---|---|---|---|---|
| Boot cause codes on the brain-health wire | `harness/brain.mjs` typing, `harness/appLifecycle.mjs` `CAUSE_SEVERITY` | Adds `plane-unreachable`, `plane-credential-refused`, `plane-not-a-plane`, `plane-identity-changed` | Any other answer keeps `plane-refused` | harness unit arms with a fake probe per verdict |
| Banner verdicts | `apps/agentos/util/SpineBanner.mjs` | One pill word and title per cause: the card's sentence plus the plane's address, action `connect-plane` | An unknown cause renders today's `plane refused` | spineBanner matrix rows |
| The probe | `harness/planeConfig.mjs` `probePlaneCredential`, `launchedPlaneRecord` | Read-only, once per failed boot, only for a launch the stored record supplied, bounded by one overall deadline (`PLANE_PROBE_DEADLINE_MS`, 10 s) over its per-request 8 s timeouts | A throw or a missed deadline keeps `plane-refused`; a request the probe times out reads `unreachable` | `brain.spec` composition arms with the real probe and `planeEnvFragment`; `planeConfig.spec` |
| Main log | `harness/mainLog.mjs` | Keeps the child's refusal line | — | existing |

## Acceptance Criteria

- [x] AC-1: a plane-attach boot whose saved plane is gone shows a `plane unreachable` verdict. Its title carries the card's "No plane answered at that address." sentence and the plane's address, and its action is Connect. The banner says nothing about `fleet.planeBase`, `fleet.planeBearer` or in-process mode.
- [x] AC-2: a saved PAT the plane rejects (401/403) shows the PAT-refused verdict in the card's words, with Connect.
- [x] AC-3: `not-a-plane` and a changed identity each get their own verdict. Any other refusal keeps today's `plane refused`, and so does a probe that throws or misses its overall deadline. A request the probe itself times out reads `unreachable`, as on the card. *(Amended 2026-10-02 at @neo-gpt's #427 review: it first said any timeout keeps `plane refused`, which contradicted the card's own probe reading a timed-out request as unreachable.)*
- [x] AC-4: the probe runs once per failed boot and writes nothing. No credential byte reaches the banner, the brain-health payload or the main log; a sentinel-bearer arm witnesses this.
- [x] AC-5: only a launch the stored record supplied is typed. An inherited plane base, an inherited PAT, or no stored record keeps `plane refused`, and the probe is never asked about another launch. *(Added 2026-10-02 at @neo-gpt's #427 review.)*

## Post-Merge Validation

- [ ] On #12's next package: the row-5 sitting's steps 4 and 5 (#424).

## Out of Scope

- **A credential that fails while the shell runs.** Reconnect cannot fix it; that is #424's next leaf.
- **#15's other remote states:** connecting, connected-empty, and scoped-empty-with-reason. #15 keeps them. This leaf carries #15's auth-refused and plane-unreachable states, on the boot path only.
- Typing the refusal on the Brain side.

## Avoided Traps

- **Parsing the child's line.** It breaks `brain.mjs`'s own quote-never-parse rule.
- **A Brain exit-code channel.** That adds a cross-repo contract and a pin, for an answer the shell's existing probe already gives in the card's vocabulary.
- **Re-probing on a timer.** One probe per failed boot is enough. Recovery is the operator's Connect.

## Related

Parent: #424 · #225 (the typed refusal and Connect this refines) · #15 (the banner vocabulary) · #335 (the row-5 script and audit) · #12

Decision Record impact: `none`. Structure map: N/A (`harness/` and `apps/agentos/`, no `ai/` placement). The sibling precedent is #225's `planeRefusal` in `harness/brain.mjs`.

Sweeps: same live, A2A, Memory Core and own-assignment runs as #424 (2026-10-02T09:02–09:05Z). The overlap with #15 is carved out above and noted on #15.

Origin Session ID: 6f7d14a3-e126-4b47-888f-fc28c748ae83
Retrieval Hint: "plane-attach boot refusal unreachable PAT refused probePlaneCredential spine banner connect card words"

⚖️ Ada (Claude Opus 5.5, Claude Code)



## Timeline

- 2026-10-02T09:05:53Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-02T09:05:54Z @neo-opus-ada added the `bug` label
- 2026-10-02T09:05:54Z @neo-opus-ada added the `agent-os` label
- 2026-10-02T09:05:54Z @neo-opus-ada added the `ai` label
- 2026-10-02T09:05:59Z @neo-opus-ada added parent issue #424
- 2026-10-02T09:06:02Z @neo-opus-ada cross-referenced by #15
- 2026-10-02T09:10:14Z @neo-opus-vega cross-referenced by #426
- 2026-10-02T09:17:36Z @neo-opus-ada referenced in commit `f3db473` - "feat(harness): a failed plane-attach boot says why in the connect card's words (#425)"
- 2026-10-02T09:18:22Z @neo-opus-ada cross-referenced by PR #427
- 2026-10-02T11:01:07Z @neo-opus-ada referenced in commit `5e7c60d` - "feat(harness): a failed plane-attach boot says why in the connect card's words (#425)"
- 2026-10-02T11:01:07Z @neo-opus-ada referenced in commit `96a34f3` - "fix(harness): a refusal is typed only for the launch the stored record supplied, inside one overall probe deadline (#425)"
- 2026-10-02T11:34:17Z @tobiu referenced in commit `c91e90f` - "feat(harness): a failed plane-attach boot says why in the connect card's words (#425) (#427)

* feat(harness): a failed plane-attach boot says why in the connect card's words (#425)

* fix(harness): a refusal is typed only for the launch the stored record supplied, inside one overall probe deadline (#425)"
- 2026-10-02T11:34:18Z @tobiu closed this issue
- 2026-10-02T12:08:15Z @neo-opus-grace cross-referenced by #335
- 2026-10-02T12:20:29Z @neo-opus-grace cross-referenced by #436
- 2026-10-02T13:34:29Z @neo-opus-ada cross-referenced by #446
- 2026-10-02T13:56:00Z @neo-gpt-sophie cross-referenced by PR #437
- 2026-10-02T14:57:35Z @neo-gpt-sophie cross-referenced by PR #447
- 2026-10-02T15:29:38Z @neo-opus-ada cross-referenced by #424

