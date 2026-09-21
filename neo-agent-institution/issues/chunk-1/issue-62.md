---
id: 62
title: Fresh boot never goes live against an armed fleet server — only Reconnect establishes the bridge
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-08-30T00:15:57Z'
updatedAt: '2026-09-04T13:12:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/62'
author: neo-fable-clio
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-04T13:12:09Z'
---
# Fresh boot never goes live against an armed fleet server — only Reconnect establishes the bridge

## Problem Scope

A fresh browser boot of the cockpit against a RUNNING, handshake-armed fleet server lands on the fail-closed sample state ("fleet offline") — only a manual Reconnect click establishes the bridge and goes live. Reproduced 3× on 2026-08-30 (headless chromium, fresh pages, devFleetServer up on 8083 with `NEO_FLEET_BEARER_HANDSHAKE=true` and the page origin allow-listed; 14s settle without the click stays cold; one Reconnect click connects within ~5s).

**Measured 2026-09-04 — the wire transcript (headless Chromium, fresh boot, a logging reverse proxy in front of the armed dev fleet server):** the boot heal WORKS. `GET /fleet/handshake` 200 at t≈0.25s, `POST resolveViewerIdentity` 200 at t≈0.26s (verified → promoted → bridge published), then no wire read at all until t=15s — the first liveness cadence tick — where `fleetRoster` / `fleetActivity` / `fleetTasks` answer 200 and the cockpit leaves the cold state at t≈17s. The 2026-08-30 "14s settle" stopped one second short of that tick. The defect is precise: the construct-time reads run on the fail-closed bridge, custody is promoted 0.3s later, and nothing re-drives the liveness owner until the next tick — the first-run moment shows "fleet offline" and the static roster for a full cadence after the server has already answered.

This defeats the launch contract's whole point: the armed handshake exists so "a fresh boot lands on a live cockpit instead of the fail-closed sample seeds" (devCockpit's own words) — the first-run moment must not require knowing about a chrome button, and must not sit on a stale verdict for 15 seconds.

## Evidence Trail

- `apps/agentos/app.mjs` wires the heal: bearer-less boot → `browserHeal` → `healBrowserFleetSession` (redeem-until-available + establish) fires after `Neo.app()` — the path exists, is reached, and promotes (measured).
- Server side verified independently: `GET /fleet/handshake` with the allow-listed Origin returns the bearer (200), foreign origins 403; the bearer is the process bearer (a second redemption returns the same token — no rotation, so a second window redeeming is harmless).
- The reconnect handler (`reconnectFleet` → liveness re-drive) succeeds against the same server seconds later because by then the bridge IS promoted — the click only re-drives the reads the cadence would have re-driven at its next tick.
- Wire transcript 2026-09-04, fresh boot: `GET /fleet/events` 401 (the bearer-less wake stream at boot) · `GET /fleet/handshake` 200 · `OPTIONS /fleet` 204 · `POST resolveViewerIdentity` 200 · silence until t=15.0s · `POST fleetTasks` / `fleetActivity` / `fleetRoster` / `fleetHistory` 200 · banner `fm-spine-banner-cold` → `fm-spine-banner-degraded` at t≈17s, cards 11 (sample) → 9 (the real registry). Degraded only because the dev checkout lacks `resources/content` (the pr-lane scan's ENOENT) — an environment residual, not this ticket.
- The construct-time reads (`Container#onConstructed` → `loadActivity` / `loadRoster` / `loadTasks`) produce no wire request: the fail-closed bridge throws locally before any fetch, which is why the transcript is silent between 0.26s and 15s.

## The Fix

`app.mjs` publishes the in-flight browser heal as a promise on the fleet global (`AgentOS.fleet.custodyHeal`) BEFORE `Neo.app()` — a deferred, so the redeem fetch still starts only after shell creation (the pinned `app → fetch` order) — and settles it with the heal's verdict. The liveness owner (`LivenessController#startLiveness`) chains one immediate `reconnectFleet()` onto a `true` resolution: the same re-drive the Reconnect button performs, fired by the promotion instead of a click. A heal that ends without promotion changes nothing; a cockpit constructed after the heal settled sees a resolved promise and re-drives once (the reads are idempotent and capped).

## Acceptance Criteria

- [ ] AC-1 A fresh page load against a running, armed fleet server reaches the live spine (banner hidden — or degraded by the feed, never cold — and the real roster) with ZERO clicks, within ~1s of custody promotion: before the first liveness cadence tick, not at it. Witness: the headless boot transcript — first `fleetRoster` read ≤ 2s after `resolveViewerIdentity`.
- [ ] AC-2 A boot against a DOWN server keeps today's fail-closed sample state and the Reconnect affordance (no regression of the honest cold path); a heal that exhausts re-drives nothing.
- [ ] AC-3 Unit witnesses: (a) the liveness owner re-drives exactly once when the published heal resolves `true`, never on `false`, never after `stopLiveness()`; (b) `onStart` publishes the heal slot before `Neo.app()` runs while the redeem fetch still starts after it (the existing order arm extended, not duplicated).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| `AgentOS.fleet.custodyHeal` — App-worker fleet global, NEW slot | `apps/agentos/app.mjs` `onStart` (the boot module owns the heal) | A `Promise<Boolean>` published before `Neo.app()`; resolves `true` exactly when the browser heal retained promotion authority, `false` on exhaustion or lost authority; never rejects; a later joining window that starts a fresh heal window replaces it | Absent on the shell transport and on a bearer-carrying boot: consumers treat `undefined` as "no heal in flight" and do nothing | JSDoc on the slot in `app.mjs` and on `followCustodyHeal` | `app.spec.mjs` arm: slot published before `Neo.app`, fetch after |
| `LivenessController#startLiveness` — existing, protected | `apps/agentos/view/fleet/cockpit/LivenessController.mjs` | Additionally calls `followCustodyHeal()`: one `reconnectFleet()` when the slot resolves `true` while liveness is running | No slot, `false`, or stopped liveness → no re-drive | JSDoc | cockpit unit arm (AC-3a) |

Surface-Anchor V-B-A (2026-09-04): `resolveFleetUrl`, `healBrowserFleetSession`, `browserFleetHealPromise`, `Container#onConstructed`, `startLiveness`, `stopLiveness`, `reconnectFleet`, `livenessPollInterval` (15000) verified by grep in the current tree; `AgentOS.fleet.custodyHeal` has no existing reader or writer.

## Out of Scope

Operator-seat conflation (server-side, documented remediation) · the containerized fleet path (pre-split images; brain deployment tickets own it) · content-root config leaves (brain-side sibling ticket) · `GET /fleet/events` answering 404 WITH a bearer on the dev fleet server (401 without) — the wake stream never arms; a separate sighting · the in-app Browser pane blocking the App worker's cross-port POSTs (a harness artifact, not the product).

## Related

Surfaced during the 2026-08-30 "get online again" session (operator + @neo-fable-clio); measured on the wire 2026-09-04. Parent arc: #10. The launch contract: `neo-agent-brain/ai/services/fleet/devFleetServer.mjs` + `fleetLaunchContract.mjs`; browser half: `apps/agentos/fleet/redeemFleetBearerHandshake.mjs`, `apps/agentos/fleet/fleetSessionCustody.mjs`, `apps/agentos/app.mjs`, `apps/agentos/view/fleet/cockpit/LivenessController.mjs`.

Origin Session ID: 729e92a3-fa8a-4c4f-bc72-6c0ad1a5e604 · measurement session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

Retrieval Hint: `query_raw_memories("boot heal promotes cadence tick custodyHeal reconnectFleet fresh boot cold 15s")`

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3


## Timeline

- 2026-08-30T00:16:17Z @neo-fable-clio cross-referenced by #246
- 2026-08-30T00:16:30Z @neo-fable-clio cross-referenced by #184
- 2026-08-30T00:23:44Z @neo-fable-clio cross-referenced by #23
### @neo-fable-clio - 2026-09-04T10:46:41Z

Observation 2026-09-04 (pin 5 read, PR #91): a stronger form than the body states. Served cockpit on `localhost:8091` (`useSharedWorkers: true`), first opened WITHOUT a `fleetUrl` param (the default `127.0.0.1:8083/fleet`, whose origin allowlist did not include 8091), then re-navigated with `?fleetUrl=http://127.0.0.1:8093/fleet` against a fresh, handshake-armed server whose allowlist DID include the origin. The pill read `127.0.0.1:8093/FLEET`, `curl` with the page origin redeemed the bearer (200, logged server-side), yet neither the boot heal nor TWO Reconnect clicks ever reached the server (zero handshake log lines, zero console errors).

Hypothesis worth the first probe: two truths. `onStart` resolves `fleetUrl` ONCE per App Worker (`resolveFleetUrl()` at boot, `app.mjs:97`); a later window on the same SharedWorker never re-runs it, so the worker's custody keeps the first window's URL (8083, origin-refused) while the page-side pill renders the current window's param (8093). The heal loop and Reconnect then keep re-driving the wrong endpoint. If that holds, the fix is to make the bridge endpoint a per-window fact (re-resolve on window connect / on Reconnect from the live window's URL) — and AC-1 gains a second arm: a window whose URL names a different server goes live against THAT server.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3


- 2026-09-04T12:34:15Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T12:44:56Z @neo-fable-clio cross-referenced by PR #96
- 2026-09-04T12:46:46Z @neo-fable-clio cross-referenced by #97
- 2026-09-04T12:55:09Z @neo-gpt added the `bug` label
- 2026-09-04T12:55:09Z @neo-gpt added the `ai` label
- 2026-09-04T12:55:09Z @neo-gpt added the `agent-os` label
### @neo-gpt - 2026-09-04T12:55:18Z

Triaged per `ticket-triage` skill. Applied: `bug`, `ai`, `agent-os`. Stage retrospective passed: the wire transcript and exact cadence isolate a real first-boot defect; the prescription hands an existing custody verdict to the existing liveness owner without adding a transport or polling authority; the consumer and fallback paths are explicit; no ADR boundary changes. Adjacency search found no duplicate.

- 2026-09-04T13:00:58Z @neo-fable-clio referenced in commit `e7a10f7` - "fix(agentos): the custody heal hands its promotion to the liveness owner, so a fresh boot goes live now instead of at the next cadence tick (#62)"
- 2026-09-04T13:00:59Z @neo-fable-clio referenced in commit `d6c0891` - "fix(agentos): the custody-heal re-drive is fenced to the liveness generation that attached it — a stop/restart before the heal settles delivers once (#62)"
- 2026-09-04T13:12:09Z @tobiu referenced in commit `75df349` - "Merge pull request #96 from neomjs/agent/62-custody-heal-handoff

fix(agentos): the custody heal hands its promotion to the liveness owner, so a fresh boot goes live now instead of at the next cadence tick (#62)"
- 2026-09-04T13:12:09Z @tobiu closed this issue
- 2026-09-04T15:34:11Z @neo-fable-clio cross-referenced by PR #104

