---
id: 917
title: 'A stored plane down at launch reads "fleet timed out", not "Agent OS starting"'
state: CLOSED
labels:
  - bug
  - ai
  - regression
assignees:
  - neo-opus-ada
createdAt: '2026-11-11T18:26:30Z'
updatedAt: '2026-11-13T01:08:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/917'
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
closedAt: '2026-11-12T20:50:47Z'
---
# A stored plane down at launch reads "fleet timed out", not "Agent OS starting"

## Context

Row 5's H1 (provocation 1) failed on the 2026-11-11 candidate: Institution `6f00d3da`, Brain `7219f4e1`, Engine `c7e19744`. The receipt is [#424 comment 6803365485](https://github.com/neomjs/neo-agent-institution/issues/424#issuecomment-6803365485), and the defect-note fingerprint is `20a7ac2a6b05ea80`.

**Sweeps, 2026-11-11 18:24:54Z:**
- **Live latest-open + closed:** checked the latest 20 open (52 open, red band; `#911`..`#651`). Closed and all-state searches covered `plane starting`, `plane unreachable`, `PLANE_SILENT_SECONDS` and `fleet timed out`. I read `#847`, `#849`, `#735`, `#841` and `#533`. This is a regression against `#847` §3 / AC-6, not a duplicate. The fix folds under #424, the red band's owner.
- **A2A in-flight:** the 20 newest messages, all read-states, 16:16Z–18:24Z. No claim on this surface. Clio's 18:09:56Z defect-note covers the same provocation's effect on seats' MCP, a separate surface.
- **MC:** `stored plane unreachable at launch spine shows fleet timed out…`, 5 results. The only prior decision is `#847` / PR `#848`.
- **Own-assignment:** 2 open (#424, #516). This leaf is theirs.

## The Problem

`neo-local-agent-os-ingress-1` was stopped at 17:48:23Z, so `127.0.0.1:3102` was unreachable. The operator quit the FM and reopened it.

**`main.log` knew the plane was down from the first second:**
- 17:53:10.999Z: `boot phase plane init: 3 ms (refused: plane unreachable (TypeError))`
- 17:53:11.008Z: `HARNESS_BRAIN_PLANE_STARTING plane at http://127.0.0.1:3102 not reachable at boot; re-probing while Agent OS starts`

**The spine never said so.** From launch through 17:56:18Z, past the 180 s mark at about 17:56:11Z, it read:

```
fleet timed out · Roster read timed out — no fleet data yet · fleet read exceeded 10000ms
```

It used the cold class, offered only Reconnect, and showed an empty roster. The operator clicked Reconnect with the plane still down, and it "timed out again". The source is window-only frames every 2 s plus Neural Link reads of `SpineBannerComponent`; the still is `plane-down-at-launch`.

**Recovery itself worked without any click.** The ingress came back at 17:56:14Z. Main re-probed and booted the fleet at 17:56:20Z (plane init 36 ms, verified), and the spine went `live` between 17:57:24Z and 17:57:30Z. Only the words on screen are wrong.

**Root cause, found 2026-11-12 18:03Z** (installed FM: Brain `dc3082cc`, Institution `821376a`, Engine 13.4.0; the plane's ingress was down 18:02:46Z → 18:04:17Z, and the FM was quit and reopened):
- **The pulls carry the fact.** 3 s after `HARNESS_BRAIN_PLANE_STARTING`, the cockpit controller (`neo-fm-fleet-cockpit-controller-1`) read `brainHealthReadGeneration: 11`, `_brainWasStarting: true` and its 1 s timer armed. The console showed one `brainHealth` call per second and no error. Only a `starting` answer arms that timer.
- **The provider drops it.** At the same moment the provider (`neo-state-provider-2`) read `data.shellTransport: null`, and the spine read `fleet timed out · Roster read timed out — no fleet data yet` (cold), which reproduces 2026-11-11.
- **Why.** `BrainHealthRead.apply` calls `provider.setData('shellTransport', transport)`. The engine's `state.Provider#internalSetData` drills a plain-object value into leaf paths (`shellTransport.phase`, …) and bubbles each leaf into its parent, but it stops when the parent's current value is not an object ("Stop if parent is not an object"). The cockpit provider declares `shellTransport: null`, so the parent stays `null`, and `deriveBannerVerdict` hands the spine `transport: null`.
- **Why the spec is green.** `makeProviderFake` (`cockpitFakes.mjs`) assigns `data[key] = value` whole, so it differs from the real Provider exactly here. Main's fact, the verdict precedence and `coldFallbackFor` are all correct.
- **A second key, same drop:** `FleetAdmission` writes `presenceCapability: capabilities?.presence` (an object) into another `null`-declared key. It read `null` on the healthy FM as well.
- **Probe used** (no custom build): Neural Link `get_instance_properties` with dotted paths. `inspect_state_provider` on that provider closed the bridge on 2026-11-12, so it is not used.

## The Architectural Reality

**Design authority**, from `#847`'s Contract Ledger row for the `plane-starting` branch of `coldFallbackFor`:
> "Under 180 s unchanged: `Agent OS starting · N s`, its lead, `Longer than usual.` from 60 s. From 180 s: text `Agent OS not answering · N min`, lead 'Your Agent OS has not answered for N min. Start it, or connect to another; the Fleet joins by itself when it answers.'"

`apps/agentos/util/SpineBanner.mjs`:
- `coldFallbackFor(transport)` owns the silence fallback. The comment at its read-timeout branch says the 10 s roster timeout during a starting transport "is a consequence of the starting transport, never a separate verdict that should shadow `Fleet starting` / `Agent OS starting`". On 2026-11-11 it shadowed them for 3 minutes.
- `PLANE_SILENT_SECONDS = 180` is the renderer-local threshold.

`harness/brain.mjs` `bootFleetWithPlaneRecovery` is the main-side wait that logs `HARNESS_BRAIN_PLANE_STARTING`. It worked: it re-probes every 5 s and attached 6 s after the plane answered.

## The Fix

1. **Reproduce at the seam, red on `6f00d3da`:** use the REAL cockpit `StateProvider`, not the fake. `BrainHealthRead.apply` with `{state: 'degraded', transport: {phase: 'starting', reason: 'plane-starting', since, elapsedMs}}` should leave the fact in `data.shellTransport` and the spine on `Agent OS starting · N s`. Today it leaves `null` and `fleet timed out`. Add the same arm for `presenceCapability`.
2. **Keep each fact whole in the real Provider:** a fact (a value whose keys vary between writes) must not be merged leaf by leaf, or stale keys from the previous fact survive. Choose in the PR between a declared full key set that is written in full every time (with the formula reading an all-`null` shape as `null`) and an atomic write. Then the `plane-starting` branch shows:
   - `Agent OS starting · N s` from first paint, with `Longer than usual.` from 60 s;
   - `Agent OS not answering · N min` from 180 s, with `#847`'s lead.

## Acceptance Criteria

- [x] With the stored plane unreachable at boot, the spine never shows `fleet timed out`. It shows `#847` §3's pill and lead before and after 180 s. A spec at the seam that loses the fact is red on `6f00d3da` and green after. *(Delivered by PR #943; the arms were red on `821376a`, the same path.)*
- [x] The roster read's 10 s timeout during the plane-starting wait does not produce its own verdict. Same spec. *(PR #943.)*
- [x] `presenceCapability`, as `FleetAdmission` writes it, reaches `data.presenceCapability` whole in the real Provider. Same spec. *(PR #943, including the producer's typed `reasonCode`.)*
- [x] *(post-merge, installed)* Row 5's H1 re-walk on the next candidate passes: the reason and the next step are readable without hovering. One receipt on #516 and #424. *(2026-11-13, Institution `e717bbf`: `Agent OS starting · 9 s` → `Longer than usual.` at 71 s → `Agent OS not answering · 3 min` at 183 s, never `fleet timed out` though the roster read had timed out; live again without a click. Receipt [#516 6840712914](https://github.com/neomjs/neo-agent-institution/issues/516#issuecomment-6840712914); row line on #424.)*

## Out of Scope

- The `main.log` wording. It suggests "fix fleet.planeBase / fleet.planeBearer" for a down plane and labels a refused PAT "plane unreachable" (defect-note `49788b8e814e231a`).
- The activity feed's lag after boot (defect-note `b4be35b967093242`).
- The refusal tone, yellow versus red (a design note on #424).

## Related

- #424 and #516: the walk.
- `#847`: the contract that regressed.
- `#735` and `#849`: the boot refusals and Retry.
- `#477`: a reason and a next step on every surface.

Origin Session ID: b261f993-5994-4a53-bb9c-9fac974a9029
Retrieval Hint: "plane down at launch fleet timed out shadows Agent OS starting"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-11T18:26:30Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-11T18:26:31Z @neo-opus-ada added the `bug` label
- 2026-11-11T18:26:32Z @neo-opus-ada added the `ai` label
- 2026-11-11T18:26:32Z @neo-opus-ada added the `regression` label
- 2026-11-11T18:26:37Z @neo-opus-ada added parent issue #424
- 2026-11-11T18:40:02Z @neo-opus-ada cross-referenced by #921
- 2026-11-11T18:40:13Z @neo-opus-ada cross-referenced by #922
- 2026-11-11T20:08:57Z @neo-opus-ada cross-referenced by #923
- 2026-11-12T00:39:20Z @neo-opus-grace cross-referenced by #925
- 2026-11-12T02:15:43Z @neo-opus-ada cross-referenced by #927
- 2026-11-12T02:15:56Z @neo-opus-ada cross-referenced by #928
- 2026-11-12T02:31:04Z @neo-opus-ada cross-referenced by #929
- 2026-11-12T17:30:37Z @neo-opus-ada cross-referenced by #424
### @neo-opus-ada - 2026-11-12T18:09:02Z

**Handover (session refresh; the owner stays me):** the seam is found and written into the body (*Root cause, found 2026-11-12 18:03Z*). No branch exists yet.

**Pickup:**
1. Write a spec on the real `AgentOS.view.fleet.cockpit.StateProvider` (not `makeProviderFake`). `BrainHealthRead.apply` with a `plane-starting` fact must leave it in `data.shellTransport`, and the spine must read `Agent OS starting · N s`. It is red today. Add the same arm for `presenceCapability` written by `FleetAdmission`.
2. The fix keeps each fact whole in the real Provider. Choose between a full declared key set written in full every time and an atomic write, so that no stale key survives from the previous fact. Ask the engine owners whether `setData('k', {…})` on a key declared `null` should land whole; this ticket fixes the cockpit side either way.
3. Then an installed candidate and the H1 + provocation 1 re-walk at the operator's slot. Receipt on #424 and #516.

**Probe for the re-walk:** Neural Link `get_instance_properties` with dotted paths on the provider and on `neo-fm-fleet-cockpit-controller-1`. Never `inspect_state_provider`; it closed the bridge.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-12T18:29:52Z @neo-opus-ada cross-referenced by PR #943
- 2026-11-12T18:38:46Z @neo-opus-ada referenced in commit `3d460bb` - "chore(agentos): restamp baselines; comments carry no refs (#917)

The visual stamp follows the four touched apps/agentos blobs; every screenshot golden verifies unchanged at this head. Test comments drop their ticket references for the archaeology check."
- 2026-11-12T20:32:28Z @neo-opus-ada referenced in commit `de0952c` - "fix(agentos): keep the presence capability's typed reasonCode (#917)

The pinned producer adds reasonCode (e.g. viewer-binding-unavailable) to a refused presence read; the leaf-complete shape dropped it. It is now a sixth leaf, cleared by an answer without it and by retirement, pinned on the real provider. The visual stamp follows the two touched apps/agentos blobs."
- 2026-11-12T20:50:46Z @tobiu referenced in commit `e717bbf` - "fix(agentos): the cockpit provider keeps its object facts whole (#917) (#943)

* fix(agentos): the cockpit provider keeps its object facts whole (#917)

state.Provider#setData drills an object into leaf paths and stops the
bubble at a parent whose value is not an object. The cockpit provider
declared shellTransport and presenceCapability as null, so every
transport fact and presence envelope was dropped: the spine read the
roster timeout instead of "Agent OS starting", and the presence chip
never named a degraded producer.

Both keys are now declared leaf-complete, and their writers write the
full key set every time (BrainHealthRead.transportOf,
FleetAdmission.presenceCapabilityOf), so no key of a previous fact
survives. A transport with no phase reads as no fact.

* chore(agentos): restamp baselines; comments carry no refs (#917)

The visual stamp follows the four touched apps/agentos blobs; every screenshot golden verifies unchanged at this head. Test comments drop their ticket references for the archaeology check.

* fix(agentos): keep the presence capability's typed reasonCode (#917)

The pinned producer adds reasonCode (e.g. viewer-binding-unavailable) to a refused presence read; the leaf-complete shape dropped it. It is now a sixth leaf, cleared by an answer without it and by retirement, pinned on the real provider. The visual stamp follows the two touched apps/agentos blobs."
- 2026-11-12T20:50:48Z @tobiu closed this issue
- 2026-11-13T01:07:34Z @neo-opus-ada cross-referenced by #516

