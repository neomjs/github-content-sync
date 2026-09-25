---
id: 221
title: An unconfigured shell beside a running plane refuses its Brain and says only "not ready"
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-25T20:54:59Z'
updatedAt: '2026-09-25T21:30:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/221'
author: neo-opus-ada
commentsCount: 0
parentIssue: 7
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-25T21:30:28Z'
---
# An unconfigured shell beside a running plane refuses its Brain and says only "not ready"

## Context

The operator's packaged shell (launched 2026-09-25 19:53Z, Neural Link session `ac07d3c8`, no plane record) sat at `daemonState: degraded / boot-not-ready` for 52 minutes. Its cockpit said only "fleet: Brain is not ready" and showed the static sample roster. The team machine runs the canonical local Agent OS in docker, and its Chroma publishes `127.0.0.1:8000` (`neo-local-agent-os-chroma-1`).

## The Problem

On any machine that already runs a plane, an unconfigured packaged shell can never boot its Brain, and it never says why:
- `bootProductBrain` (`harness/main.mjs`) plans `own` mode: there is no plane record, and no orchestrator pid in its own data root.
- `buildPackagedBrainEnv` sets no `NEO_CHROMA_PORT`, so the Brain takes its production default, `8000` (Brain `ai/configBase.mjs`, `chroma.portProd`).
- The packaged guard refuses to own an organism beside a held Chroma port, by design, and throws "chroma port 8000 is already held … the packaged harness cannot own an organism beside it".
- `appLifecycle.settleBrainBoot(false)` records the generic cause `boot-not-ready`. The thrown message goes to stdout, which a Finder launch discards (#219).

The remedy exists: the plane card (#211) attaches this shell to the plane that holds the port. But the cockpit never connects the two. The banner blames a Brain "not ready", the card can be dismissed for the session, and nothing says "a plane runs here: connect to it".

## The Architectural Reality

- `harness/main.mjs` `bootProductBrain` holds the Chroma-port guard (`probePort({host: 'localhost', port: paths.chromaPort})`) and the fleet-port refusal, each as a bare `throw`.
- `harness/appLifecycle.mjs`: `recordBrainCause(code)` and `settleBrainBoot(up)`. The cause vocabulary today is `boot-not-ready` and similar; `brainHealth` carries it to the cockpit through `brain-health`.
- `apps/agentos/util/BrainHealthRead.mjs` maps `brainHealth` into `daemonState` / `daemonDegradedReason`, and the spine banner reads those.
- `ViewportController#mountPlaneSetup` creates the card only when `planeStatus()` says packaged and unconfigured. `PlaneSetupPanel#onDismissClick` hides it for the session.

## The Fix

1. **A typed cause.** The packaged guard's refusal records `organism-beside-plane`, with the held port, instead of the generic `boot-not-ready`. `brainHealth` answers it.
2. **The cockpit names it.** `BrainHealthRead` passes the cause through, and the spine banner reads "A plane already runs on this machine: connect to it" rather than "Brain is not ready".
3. **The card comes back.** When the cause is `organism-beside-plane`, the banner carries a Connect action that re-mounts the plane card, even after "Not now".

## Acceptance Criteria

- [ ] A packaged boot whose Chroma port is held settles with cause `organism-beside-plane` and the port, and `brainHealth` answers it (unit arm on the boot path's refusal).
- [ ] The cockpit's banner for that cause names the running plane and offers Connect. The action re-mounts the card after a dismissal (unit arms on the derivation and the controller).
- [ ] Every other boot failure keeps its current cause and banner (control arms).
- [ ] Post-merge, on the team machine: an unconfigured Finder launch shows the plane-running banner, and Connect from it reaches plane-attach.

## Out of Scope

- Owning an organism beside a plane: the guard stays, and two organisms on one machine is the failure it prevents.
- Auto-attaching without the viewer's PAT: the credential stays the viewer's (#211).
- The persisted log itself, which is #219 / #220.

## Avoided Traps

- **Moving the packaged Chroma to another port to "fix" the boot.** That creates the competing organism the guard exists to prevent.
- **A second banner.** The spine banner is the one cold-state surface; the cause refines its line and action.

## Related

#211 / #212 (the plane card) · #219 (the log that would have shown the thrown message) · #12 (the first-run spec) · #7 (the Electron shell epic)

Sweeps: the latest open issues were checked at 2026-09-25 20:56Z, with no equivalent (#219 is the log, #214 the smoke arm). A2A claims in the last 60 min: none on the packaged boot cause. Memory Core: recall degraded tonight (#506 incident), attested. Own open assignments: #219.

Origin Session ID: 0f80515e-7682-4313-8101-b926da48c55c
Retrieval Hint: "packaged shell own-mode chroma port held beside plane boot-not-ready organism-beside-plane banner connect card"


## Timeline

- 2026-09-25T20:55:00Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-25T20:55:00Z @neo-opus-ada added the `bug` label
- 2026-09-25T20:55:01Z @neo-opus-ada added the `ai` label
- 2026-09-25T20:55:11Z @neo-opus-ada added parent issue #7
- 2026-09-25T21:03:58Z @neo-opus-ada cross-referenced by PR #222
- 2026-09-25T21:16:29Z @neo-opus-ada referenced in commit `62ead98` - "test(shell): the banner spec states the label-content law without ticket refs (#221)

The source-comment archaeology check reads every file a PR touches, so the two
inherited refs in spineBanner.spec.mjs failed it. The law's wording stays; the
ticket number belongs in history, not in the comment."
- 2026-09-25T21:30:28Z @tobiu referenced in commit `63ed707` - "Merge pull request #222 from neomjs/ada/221-beside-plane

fix(shell): a shell beside a running plane names it and offers Connect (#221)"
- 2026-09-25T21:30:29Z @tobiu closed this issue
- 2026-09-25T21:32:45Z @neo-opus-ada cross-referenced by #223
- 2026-09-25T22:08:39Z @neo-opus-ada cross-referenced by #225

