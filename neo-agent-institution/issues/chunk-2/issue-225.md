---
id: 225
title: A plane-attach boot the plane refuses says why and offers Connect
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-25T22:08:38Z'
updatedAt: '2026-09-26T08:47:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/225'
author: neo-opus-ada
commentsCount: 1
parentIssue: 15
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-26T08:47:07Z'
---
# A plane-attach boot the plane refuses says why and offers Connect

## Context

A packaged shell attaches to a plane from its stored record (#211, #223). When the plane refuses the fleet child, the child logs why and exits 1 before it is ready. The shell then says only "fleet: Brain is not ready" over the sample roster. The known refusals:
- `plane mode refused (<base>): plane identity mismatch`
- `… plane unreachable (…)`
- `[fleet] startup refused: no viewer identity resolved`

It is the same generic state #221 removed for the beside-plane case, arriving by a second road. From Finder the child's line goes nowhere until #220's `main.log` lands, and even then it is a log, not the surface the user is looking at.

## The Problem

The harness knows it planned `plane-attach`, and it knows the fleet child exited before ready. It throws `fleet transport exited before ready (code=1 …)`. `bootFailureCause` names a cause only for `organism-beside-plane`, so the lifecycle records `boot-not-ready`. The user gets no reason and no way back to the card, because a stored record reads as configured and the card only mounts when it doesn't.

## The Architectural Reality

- `harness/main.mjs` `bootProductBrain`: `resolveProductBrainPlan` yields `plan.mode`. `startBrainChild({…, onLog: brainLog})` spawns the fleet child, and `awaitFleetReady` rejects on its early exit (`harness/brain.mjs`). Its caller maps the error through `bootFailureCause(error)` into `appLifecycle.settleBrainBoot(false, cause)` (#222).
- `harness/brain.mjs` `besidePlaneRefusal` / `bootFailureCause` is the typed-refusal precedent (#222). `appLifecycle.mjs` `CAUSE_SEVERITY` ranks causes, `BrainHealthRead` projects `daemonCause`, and `SpineBanner.deriveSpineBanner` turns a cause into a verdict with an `action`. `connect-plane` already reopens the card (`ViewportController#showPlaneSetup`).
- The child's lines reach `onLog` line by line (`forwardLines`), so the harness can keep the last one without parsing any Brain wording.

## The Fix

1. **Keep the child's last line.** For a `plane-attach` boot, the fleet child's `onLog` keeps its last line beside `brainLog`. When `awaitFleetReady` rejects, `bootProductBrain` throws a typed `planeRefusal(detail)` (`code: 'plane-refused'`). The detail is the child's last line, truncated, and blanked when it carries a known secret (#220's `carriesSecret`).
2. **Name the cause.** `bootFailureCause` maps `plane-refused`, and `CAUSE_SEVERITY` ranks it with `organism-beside-plane`.
3. **Say it and offer the way back.** `SpineBanner` gets a `plane-refused` verdict: "plane refused · The plane refused this shell — reconnect it · <detail>", with `action: 'connect-plane'`, so the card comes back even though the record reads as configured.
4. **Name the env composition** in `startBrainChild`'s JSDoc: a caller's env can add or override an inherited variable, never remove one. This is @neo-preview's note from the #224 and #19234 reviews, and every env-fragment builder is bound by it.

## Acceptance Criteria

- [ ] AC-1: A `plane-attach` boot whose fleet child exits before ready settles with cause `plane-refused`, carrying the child's last line (truncated; blanked when it carries a known secret). Any other boot's early exit keeps `boot-not-ready`.
- [ ] AC-2: The banner names the refusal with that line and offers Connect, which reopens the plane card. A control: every other cause keeps its verdict.
- [ ] AC-3: `startBrainChild`'s JSDoc states the env composition.
- [ ] AC-4 (post-merge, operator's machine): a refused attach shows "plane refused" with the reason, and Connect reopens the card.

## Out of Scope

- The rest of #15: connecting, connected-empty, scoped-empty-with-reason.
- A structured refusal channel from the Brain. The last line is a quotation, not a parsed field.

## Related

Parent #15 (the auth-refused state of its AC-1). #221/#222 (the typed-refusal precedent), #223/#224 (the identity pair), #220 (`main.log` and `carriesSecret`, which this reuses, so it lands after #220).

Live latest-open sweep: the latest 20 open neo-agent-institution issues at 2026-09-25T22:07:50Z; no equivalent (#15 is the parent, #214 the smoke). A2A in-flight sweep: no claim on plane refusals. MC sweep: "plane-attach boot refused banner says Brain is not ready fleet transport exited before ready cause", 5 results, no prior decision (neighbours: tonight's NL read of `boot-not-ready`, #211's filing). Own-assignment sweep: 1 open (#219), not overlapping.

Origin Session ID: 0f80515e-7682-4313-8101-b926da48c55c


## Timeline

- 2026-09-25T22:08:39Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-25T22:08:40Z @neo-opus-ada added the `bug` label
- 2026-09-25T22:08:40Z @neo-opus-ada added the `ai` label
- 2026-09-25T22:08:47Z @neo-opus-ada added parent issue #15
- 2026-09-25T22:15:27Z @neo-opus-ada cross-referenced by PR #226
- 2026-09-25T22:51:13Z @neo-opus-ada referenced in commit `b68263a` - "Merge dev into ada/225-plane-refused: #220's main log lands first (#225)"
- 2026-09-25T22:51:13Z @neo-opus-ada referenced in commit `f2d6a06` - "refactor(shell): a plane refusal uses the main log's secret rule and list (#225)

#220 landed first, so the fold #226 declared is due. planeRefusal calls
carriesSecret instead of its inline copy, and main passes the log's own
secret list (mainSecrets) instead of a second, narrower one."
### @neo-opus-ada - 2026-09-25T22:55:26Z

**Handover (session sunset, 2026-09-25 23:0xZ).** Owner: @neo-opus-ada.

**State:** PR #226 is this ticket's PR. Head `f2d6a06105`, CI 13/13, CLEAN. @neo-preview is requested; no review has been posted, and an A2A was sent.
- `0a6b9faf3d`: a plane-attach boot the plane refuses throws `plane-refused`. The detail is the fleet child's last line, drained on close, at most 240 characters, and dropped if it carries a secret. The lifecycle ranks it, and the spine banner reads "plane refused" with the reason and a Connect action.
- `b68263a223` + `f2d6a06105`: dev is merged in, and the fold the PR declared is done. #220 landed first, so `planeRefusal` now calls `mainLog.mjs`'s `carriesSecret`, and main hands the log and the refusal one secret list (`mainSecrets`). Dev's merge touches no `apps/agentos` file, so the visual stamp holds.

**Evidence:** the brain, mainLog, appLifecycle and pack specs pass 87/87 against a prepared Brain root (`NEO_AGENTOS_RUNTIME_ROOT` must hold a generated `ai/config.mjs` and `node_modules`; the pack root and an unprepared worktree both fail the resolveBrainPaths arm for that reason alone). The full unit suite was 929 at `0a6b9faf3d`.

**Pickup:**
1. If there is a review round, address it on `ada/225-plane-refused` (worktree `wt-inst-225`). On approval, hand off to @tobiu for the merge.
2. The post-merge box needs a shell that contains this PR. The `.app` built tonight from dev `4ce50eb9f3` (#222 + #224 + #220) does not. Rebuild after the merge with `cd harness && npm install && NEO_AGENTOS_RUNTIME_ROOT=<assembled Brain root> npm run dist`.
3. Installing into `/Applications` quits the operator's running shell, so it waits for their go.
4. On the team machine, a refused attach should show "plane refused" with the child's reason, and Connect should reopen the plane card.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-09-26T07:15:18Z @neo-opus-ada cross-referenced by #227
- 2026-09-26T07:21:12Z @neo-fable-clio cross-referenced by #228
- 2026-09-26T08:47:07Z @tobiu referenced in commit `e8b88da` - "Merge pull request #226 from neomjs/ada/225-plane-refused

fix(shell): a plane-attach boot the plane refuses says why and offers Connect (#225)"
- 2026-09-26T08:47:08Z @tobiu closed this issue
- 2026-09-26T08:50:42Z @neo-preview cross-referenced by PR #231

