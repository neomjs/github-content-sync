---
id: 917
title: 'A stored plane down at launch reads "fleet timed out", not "Agent OS starting"'
state: OPEN
labels:
  - bug
  - ai
  - regression
assignees:
  - neo-opus-ada
createdAt: '2026-11-11T18:26:30Z'
updatedAt: '2026-11-11T18:26:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/917'
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

**Unknown:** which input made the cold verdict win. It could be the transport fact on the brain-health wire (missing, or `settled` rather than `starting`) or the verdict precedence. The fix's first spec finds out.

## The Architectural Reality

**Design authority**, from `#847`'s Contract Ledger row for the `plane-starting` branch of `coldFallbackFor`:
> "Under 180 s unchanged: `Agent OS starting · N s`, its lead, `Longer than usual.` from 60 s. From 180 s: text `Agent OS not answering · N min`, lead 'Your Agent OS has not answered for N min. Start it, or connect to another; the Fleet joins by itself when it answers.'"

`apps/agentos/util/SpineBanner.mjs`:
- `coldFallbackFor(transport)` owns the silence fallback. The comment at its read-timeout branch says the 10 s roster timeout during a starting transport "is a consequence of the starting transport, never a separate verdict that should shadow `Fleet starting` / `Agent OS starting`". On 2026-11-11 it shadowed them for 3 minutes.
- `PLANE_SILENT_SECONDS = 180` is the renderer-local threshold.

`harness/brain.mjs` `bootFleetWithPlaneRecovery` is the main-side wait that logs `HARNESS_BRAIN_PLANE_STARTING`. It worked: it re-probes every 5 s and attached 6 s after the plane answered.

## The Fix

1. **Reproduce in `spineBanner.spec.mjs`, red on `6f00d3da`.** Use the facts seen on 2026-11-11: a stored plane typed `plane-unreachable` at boot, main in the plane-starting wait, and the roster read timing out after 10 s. The spine must show `fleet timed out`, as it did.
2. **Make the `plane-starting` branch win** at the input the reproduction names, either the transport fact the renderer receives or the verdict precedence:
   - `Agent OS starting · N s` from first paint, with `Longer than usual.` from 60 s;
   - `Agent OS not answering · N min` from 180 s, with `#847`'s lead.

## Acceptance Criteria

- [ ] With the stored plane unreachable at boot, the spine never shows `fleet timed out`. It shows `#847` §3's pill and lead before and after 180 s. A spine banner spec is red on `6f00d3da` and green after.
- [ ] The roster read's 10 s timeout during the plane-starting wait does not produce its own verdict. Same spec.
- [ ] *(post-merge, installed)* Row 5's H1 re-walk on the next candidate passes: the reason and the next step are readable without hovering. One receipt on #516 and #424.

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

