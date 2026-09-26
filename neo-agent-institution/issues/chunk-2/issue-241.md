---
id: 241
title: The shell's instance switcher swaps in a bridge that has no bearer
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-26T09:34:22Z'
updatedAt: '2026-09-26T10:17:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/241'
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
closedAt: '2026-09-26T10:17:46Z'
---
# The shell's instance switcher swaps in a bridge that has no bearer

## Context

The operator's team `.app` (built from dev `e85cceb`, plane-attach, 2026-09-26) read "fleet offline" on every surface after Connect and a few reconnect attempts. Clio read that session over the Neural Link before the relaunch: `boundProfileId` = `fleet-profile:v1:http://127.0.0.1:8083/fleet`, `instanceState` = `off`, the stream's reason = the browser bridge's `noBearer` text, `shellTransport` = `null`. After a relaunch the same shell read the plane (the Golden Path pane wired, 10 items), so the attach worked and something in the session replaced it.

The shell boot never binds a profile: `app.mjs` installs the shell bridge with `credentialIngress: 'shell'` and no `profileId`. The one writer of that binding is the instance switcher.

## The Problem

In the packaged shell, choosing an instance in the switcher (or connecting one in the manage drawer) runs `ViewportController.switchToProfile` → `establishFleetSessionCustody({deliberate: true, fleetUrl})`. `deliberate: true` is the one path allowed to replace a live bridge without a bearer, and nothing checks the transport. `installFleetBridge` publishes a worker-custody bridge with no bearer, which fails closed on every call, over the shell's bridge. The renderer has no bearer by design (the shell keeps custody in main), so every switch from the shell leaves a dead bridge until relaunch.

The switcher also offers rows the shell cannot use. `initInstanceRoster` seeds a browser boot profile for `resolveFleetUrl()` (`127.0.0.1:8083/fleet`) in every topology, and the chip prints it in capitals (`127.0.0.1:8083/FLEET`), because `SwitcherButton.scss` sets `text-transform: uppercase` on the trigger's text.

## The Architectural Reality

- `apps/agentos/app.mjs`, `onStart`: the shell branch installs `installFleetBridge({credentialIngress: 'shell', send})`; the browser branch runs `establishFleetSessionCustody`.
- `apps/agentos/view/ViewportController.mjs`: `onSwitchInstance` and `onConnectInstance` both reach `switchToProfile`, whose `catch` already turns a refused establish into "binds nothing, changes nothing, answers `false`".
- `apps/agentos/fleet/fleetSessionCustody.mjs`, `establishFleetSessionCustody`: the no-downgrade guard exempts `deliberate`; it never reads the published bridge's `credentialIngress`.
- `apps/agentos/util/AddAgentFlow.mjs` already branches on `bridge?.credentialIngress === 'shell'`: the precedent for a shell-aware renderer flow.
- The shell's own binding: `harness/planeConfig.mjs` `createPlaneBroker().status` → `{attached, configured, packaged, planeBase}` through `Neo.main.addon.ShellPlane.planeStatus`; attaching another plane is `attachPlane`, which prompts in main and relaunches (`ViewportController.showPlaneSetup` mounts the card).

## The Fix

1. `establishFleetSessionCustody` refuses to replace a bridge whose `credentialIngress` is `'shell'`: the shell owns custody, and no renderer path may displace it. It throws a named error, so `switchToProfile` returns `false` through its existing `catch`.
2. In shell transport the switcher shows the shell's binding (`planeBase`, attached or not) from `planeStatus`, and its action is "Connect a plane…" → `showPlaneSetup()`. Browser profiles are not offered in the shell.
3. The chip prints the endpoint as written: no uppercase transform on the endpoint text.

## Acceptance Criteria

- [ ] AC-1 With a shell bridge published, `switchToProfile` and `onConnectInstance` leave `AgentOS.fleet.registryBridge` as the same object (`credentialIngress` `'shell'`) and answer `false` (unit arms on the custody module and the controller).
- [ ] AC-2 In shell transport the switcher lists the shell's `planeBase` and offers "Connect a plane…", which mounts the plane-setup card; no browser profile row is selectable there (unit or component arm).
- [ ] AC-3 Browser transport is unchanged: a deliberate switch still publishes the chosen endpoint's bridge (the existing switch specs stay green).
- [ ] AC-4 The chip renders the endpoint in its own case (the affected goldens are re-captured).

## Post-Merge Validation

- [ ] In a team `.app` rebuilt from dev, open the switcher and use it: the cockpit keeps reading the plane (the Golden Path pane stays wired), and no `noBearer` state appears.

## Out of Scope

- The banner's word for "connected, registry empty" (#239).
- The plane-side composition of roster, activity and tasks in plane-attach (a Brain lane, defect-noted by Clio on 2026-09-26).
- Multi-plane management in the shell beyond "connect a plane" (one plane per shell today).

## Avoided Traps

- Hiding the switcher in the shell: the operator still needs to see which plane the shell is on, and how to attach another.
- Letting the shell's switch fetch a bearer into the renderer: the renderer never holds the credential (the shell architecture's credential boundary).

## Related

#7 (parent) · #212 / #228 (plane-attach) · #226 (the refused-attach banner) · #237 (sample data, Clio)

Live latest-open sweep: the latest 20 open issues at 2026-09-26T09:33:49Z — no equivalent. A2A in-flight sweep (all read states, last 60 min): Clio's split of 09:22Z hands this defect to me; no other claim. Memory Core sweep ("instance switcher packaged shell bridge"): no prior decision. Own-assignment sweep: none open (#235 closed with #236 at 09:31Z).

Origin Session ID: 1b945fcf-1142-475f-8007-ac18d51c069a
Retrieval Hint: `query_raw_memories("instance switcher packaged shell replaces shell bridge bearer-less browser bridge")`

## Timeline

- 2026-09-26T09:34:23Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-26T09:34:23Z @neo-opus-ada added the `bug` label
- 2026-09-26T09:34:23Z @neo-opus-ada added the `agent-os` label
- 2026-09-26T09:34:24Z @neo-opus-ada added the `ai` label
- 2026-09-26T09:34:27Z @neo-opus-ada cross-referenced by #243
- 2026-09-26T09:34:29Z @neo-opus-ada cross-referenced by #244
- 2026-09-26T09:34:30Z @neo-opus-ada cross-referenced by #245
- 2026-09-26T09:34:32Z @neo-opus-ada cross-referenced by #246
- 2026-09-26T09:34:33Z @neo-opus-ada cross-referenced by #247
- 2026-09-26T09:34:58Z @neo-opus-ada added parent issue #7
- 2026-09-26T09:51:09Z @neo-opus-ada cross-referenced by PR #248
- 2026-09-26T10:17:47Z @tobiu referenced in commit `adad639` - "Merge pull request #248 from neomjs/ada/241-switcher-shell

fix(shell): the shell's instance switcher never replaces the shell's bridge (#241)"
- 2026-09-26T10:17:47Z @tobiu closed this issue

