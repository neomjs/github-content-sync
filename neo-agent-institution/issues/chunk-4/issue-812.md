---
id: 812
title: A held walk run boots live and cleans up its sealed runtime store
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-25T16:28:23Z'
updatedAt: '2026-10-25T18:05:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/812'
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
closedAt: '2026-10-25T18:03:59Z'
milestone: FM v1
---
# A held walk run boots live and cleans up its sealed runtime store

### Context

On walk candidate 2 (Institution `84e86b5`, Brain `75b015f9`, engine `0c1a373918`), the peer-side walk of row 5 cannot reach its baseline. I measured it on the operator's displays on 2026-10-25, 17:10–17:22 CET; the receipts are in [#516: 6400114089](https://github.com/neomjs/neo-agent-institution/issues/516#issuecomment-6400114089).

Design authority: #523 (the held run exists so a walker can provoke failures on a plane of the run's own) and #424's done signal (an ordinary failure returns to `live` by the product's own guidance). A walk that cannot start from `live` cannot witness a return to it.

### The Problem

1. **The held run boots outside `live`.** It opens on first-run setup ("Set up your institution", Create / Connect). The cockpit reads "fleet degraded — Fleet feed degraded — showing last-known data", "plane admission unknown" and "wake off". Rows 2–6 of #516 provoke from `live`.
2. **Connect cannot get it there.** A successful Connect relaunches the shell. The held run's teardown stops its fixture plane with the relaunch, and the successor boots a new plane on a new port. The successor showed the saved plane `127.0.0.1:61035` while its own plane ran on `61970`.
3. **The walker has no token.** The walk manifest names only identities, and the seat-token registry stores only hashes. For the measurement I minted one through the Brain's `mintSeatToken`.
4. **Cleanup fails and cannot be retried.** `walkControl.mjs cleanup` fails with `ENOTEMPTY`. The seat-runtime store under `userData/brain/runtime/<hash>`, which arrived with #702 after candidate C, is read-only: 4,283 of 4,287 directories are `r-x`. The walk manifest goes before the failure, so a retry refuses with "no held smoke run".

### The Architectural Reality

- `harness/main.mjs` `attachSmokePlane()` already writes the plane record: `startFixturePlane` writes it into `userData`, and the Fleet child attaches from it through `readPlaneConfig` and `planeEnvFragment`. *(Corrected 10-25, 17:4x CET: as filed, this line said the record stays empty.)* ~~What the smoke arm never sets is `launchedPlane`. Only `bootProductBrain()` derives it (`launchedPlaneRecord(storedPlane, planeFragment)`). So the plane broker's `getLaunchedPlane()` answers `null` in a held run, the shell reads as not launched from a plane ("this machine"), and the plane card offers first-run setup.~~ *(Struck 10-25, 18:0x CET: a packaged build that sets it, `bbe19eb`, opened word for word as before; reverted in `be0de65`.)*
- **Why the held cockpit is not `live`** (measured over CDP, 10-25 17:5x CET; [#516: 6400333196](https://github.com/neomjs/neo-agent-institution/issues/516#issuecomment-6400333196)). The roster is `live` off the fixture plane. The banner's "fleet degraded" names the PR-lane activity slot. That slot reads a synced GitHub content tree (`fleet.contentRoot`), and in a held run the directory it scans does not exist (`ENOENT`). `SpineBanner` ranks transport failures and plane refusals above a stale feed, so this baseline hides none of row 5's words.
- `planeConfig.attach()` probes the credential first (`probePlaneCredential`). It writes the plane record and relaunches only on `accepted`. A dead endpoint, a non-plane endpoint and a wrong PAT are refused without a relaunch, so #516's connect-card rows keep the plane alive; only a successful Connect loses it.
- The held run's `will-quit` teardown stops the fixture plane with the organism, as `holdForWalk`'s docblock says. So `app.relaunch()` cannot carry the plane over.
- `cleanupHeldRun` checks the pid and both plane ports, then calls `fs.rmSync(smokeRoot, {force: true, recursive: true})`. The manifest sits under `<root>/walk/`.

### The Fix

1. ~~**The smoke arm records the plane it launched with.** `attachSmokePlane()` derives `launchedPlane` exactly as `bootProductBrain()` does, so the shell knows it launched attached to its fixture plane. The walk then starts from the product's attached state and needs no Connect. Whether that alone clears "fleet degraded" is the first thing a held run measures; any further cause is named on #516 (AC-1).~~ *(Struck 10-25, 18:0x CET: measured no effect.)* Instead, the walk starts from the held run's own baseline (the steward's decision on #516). It needs no Connect and no token, and the `[human]` half witnesses `live` on the real plane.
2. **Cleanup unseals, then removes.** `cleanupHeldRun` makes the root owner-writable before removing it, and removes the manifest last, so a failed cleanup can run again.
3. **The step list follows.** #516's step list names candidate 2 and the CDP instrument: launch with `--remote-debugging-port` and `--use-mock-keychain`, read and capture through `connectOverCDP`, and take no OS screenshot.

### Acceptance Criteria

- [x] A held run of a candidate cut from `dev` reaches `live`, or the first remaining blocker is named on #516 with its receipt (CDP page text and capture). *Named on #516 (6400333196, corrected 18:0x CET), with the CDP page text: the PR-lane slot's missing content tree. The captures stay on the measuring machine, because the seat cannot attach an image.*
- [x] #516's connect-card refusals (rows 2–4) do not relaunch the shell, and the cockpit stays attached to its fixture plane (receipt). *#516: 6400333196. The held pid and the page target were unchanged after each refusal; the plane stops only with the `will-quit` teardown.*
- [x] `walkControl cleanup` removes a root that holds a read-only subtree. A unit arm builds such a root, is red on `dev` and green with the fix. *Merged in #813 (`568f665c9d`, 2026-10-25T18:03Z).*
- [x] A cleanup that fails after its checks can run again: the manifest is still there (unit arm). *Merged in #813 (`568f665c9d`, 2026-10-25T18:03Z).*
- [x] #516's step list names the candidate and the CDP instrument. *#516: 6400333196.*

### Out of Scope

- The trigger of the 10-09 macOS prompt. On candidate 2 the held app's TCC checks were preflight-only and raise no prompt (#516: 6400114089).
- Keeping a fixture plane alive across a successful Connect's relaunch.
- Giving a held run a content tree, so its banner can read `live`. Untested.
- #516's open question on the "at launch" boot paths of rows 1 and 4.

### Related

#424 (row 5) · #516 (the walk) · #523 (the held run) · #702 (the runtime store) · #727 (Connect's other door).

Decision Record impact: none.

Live latest-open sweep: the latest 20 open Institution issues at 2026-10-25T16:27Z, no equivalent; an exact search for "held run" / "walkControl" found only #516, #424, #534 and the closed #523 / #736. A2A in-flight claim sweep: the latest 30 `lane-claim` messages, none on this scope. Memory Core sweep: only this session's own measurement. Own-assignment sweep: #516 and #424, which this ticket serves. Structure map: N/A. Institution harness files, owning files `harness/main.mjs` and `harness/walkControl.mjs`; no new file.

Origin Session ID: 531ac06b-0bfa-40a3-8577-2cff0699ff08

Retrieval Hint: "held walk run live baseline fixture plane connect relaunch cleanup sealed runtime store ENOTEMPTY"



## Timeline

- 2026-10-25T16:28:25Z @neo-opus-ada added the `bug` label
- 2026-10-25T16:28:25Z @neo-opus-ada added the `agent-os` label
- 2026-10-25T16:28:25Z @neo-opus-ada added the `ai` label
- 2026-10-25T16:28:26Z @neo-opus-ada added the `testing` label
- 2026-10-25T16:28:31Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-25T16:28:34Z @neo-opus-ada added parent issue #424
- 2026-10-25T16:40:04Z @neo-opus-ada referenced in commit `be0de65` - "revert(harness): drop the held run's launchedPlane assignment, which changed nothing measured (#812)

Reverts bbe19eb. A packaged held run carrying it (Institution bbe19eb, Brain 418e62d3, engine a97cb2d5f8)
still opened on "this machine", the first-run setup card and "fleet degraded", word for word as before.
So the launched-plane record was not why the held cockpit is not live, and the change has no evidence to
ship on. The fixture plane serves none of the cockpit's feeds ("plane admission unknown", "merge queue
unavailable", "Activity unavailable"); #516 records the walk's baseline decision."
- 2026-10-25T16:43:25Z @neo-opus-ada cross-referenced by #516
- 2026-10-25T16:46:29Z @neo-opus-ada cross-referenced by #424
- 2026-10-25T16:46:30Z @neo-opus-ada added this to the **FM v1** milestone
- 2026-10-25T17:03:31Z @neo-opus-ada cross-referenced by PR #813
- 2026-10-25T17:21:02Z @neo-opus-ada cross-referenced by #814
- 2026-10-25T18:03:59Z @tobiu referenced in commit `568f665` - "fix(harness): walk cleanup unseals a read-only runtime store and removes the manifest last (#812) (#813)

* fix(harness): walk cleanup unseals a read-only runtime store and removes the manifest last (#812)

On walk candidate 2, `walkControl.mjs cleanup` failed with ENOTEMPTY: the seat-runtime store under
`userData/brain/runtime/<hash>` publishes its runtime read-only, and `fs.rmSync` cannot remove entries of a
directory without its write bit. The manifest was already gone by then, so a retry refused.

`cleanupHeldRun` now gives every directory its owner write bit first (following no link), removes
everything but the walk directory, and removes the manifest last. Two unit arms are red on dev and green
here: a sealed store is removed, and a cleanup that fails partway keeps the manifest for its retry (an
`fsModule` seam, as `planeConfig` uses).

* fix(harness): the held run's shell records the fixture plane it launched with (#812)

`attachSmokePlane()` attaches the Fleet child from the plane record the fixture plane writes, but only
`bootProductBrain()` derived `launchedPlane`. So in a held run `getLaunchedPlane()` answered null, and the
shell read as launched from no plane: "this machine" and the first-run setup card, which sends a walker to
Connect, whose successful relaunch stops the held run's plane. The smoke arm now derives it the same way.

Unverified until a held run boots on a build carrying this; #812 AC-1 measures it.

* revert(harness): drop the held run's launchedPlane assignment, which changed nothing measured (#812)

Reverts bbe19eb. A packaged held run carrying it (Institution bbe19eb, Brain 418e62d3, engine a97cb2d5f8)
still opened on "this machine", the first-run setup card and "fleet degraded", word for word as before.
So the launched-plane record was not why the held cockpit is not live, and the change has no evidence to
ship on. The fixture plane serves none of the cockpit's feeds ("plane admission unknown", "merge queue
unavailable", "Activity unavailable"); #516 records the walk's baseline decision."
- 2026-10-25T18:03:59Z @tobiu closed this issue

