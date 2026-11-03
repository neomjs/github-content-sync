---
id: 1462
title: The fleet server adopts a rotated plane bearer without a restart
state: CLOSED
labels:
  - enhancement
  - ai
  - security
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-03T08:43:44Z'
updatedAt: '2026-11-03T18:58:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1462'
author: neo-opus-ada
commentsCount: 0
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
closedAt: '2026-11-03T18:58:47Z'
---
# The fleet server adopts a rotated plane bearer without a restart

## Context

Institution neomjs/neo-agent-institution#843 carries the operator's requirement from 2026-10-30: no Fleet Manager restart for any PAT update. Its walk measured two whole-app relaunches of about 30 s each on credential changes. Its Fix runs a credential pipe from the FM's main process to the Brain child. This ticket is the Brain half: the fleet server adopts a bearer that arrives on that pipe while it runs.

## The Problem

`ai/services/fleet/devFleetServer.mjs` is the entry the FM spawns (`FLEET_SERVER_ENTRY` in the Institution's `harness/brain.mjs`). It resolves its plane client once, at boot, from `assertFleetPlaneBearerClass()`. Nothing adopts another bearer while it runs, so every PAT change ends in a relaunch.

## The Architectural Reality

At `dev` `e47fd09301`:
- `devFleetServer.mjs` builds one `createPlaneMailboxClient({baseUrl, credential: assertFleetPlaneBearerClass()})` and proves it with `init({expectedIdentity: viewer.agentIdentityNodeId})`. The wake stream's catch-up, `openWork` and `wakeRound` all run on that client.
- *(Correction 2026-11-03, at the server half)* The bearer has a second audience. With a forge-PAT `fleet.planeBearerClass` and no declared fleet-surface bearer, `resolveFleetPlaneAdmissionBearer` presents the same bearer at the plane's `/fleet` surface, where the wake stream (`createFleetWakeSseConsumer`) and the seat-define client (`FleetControlBridge.planeFleet`) each take it at construction. The packaged FM is that case: `planeEnvFragment` sets `NEO_FLEET_PLANE_BEARER_CLASS` to the PAT's recorded `authSource` and empties both admission variables.
- `fleetServer.mjs` `resolveFleetPlaneBearer` reads `fleet.planeBearer` or `fleet.planeBearerFile`. `assertFleetPlaneBearerClass` refuses a bearer that aliases the deployment's admission token.
- `planeMailboxClient.mjs` `init()` closes the client's prior life before it opens a new one. Re-initialising in place would drop the working session before the new bearer is proven.
- ADR 0019: the bearer never goes into `AiConfig` (B4), and a new leaf is declarative, read at the entrypoint's use site.
- Institution ADR 0034 §2.3 item 8 keeps plane configuration out of `FleetControlBridge`. #843 amends its transitional seam for the pipe.

## The Fix

1. A leaf names the inherited fd that carries plane bearers, for example `fleet.planeCredentialFd`: env-bound, unset by default. Unset keeps today's boot path.
2. When it is set, the fleet server reads newline-delimited JSON frames from that fd, `{"bearer": "…", "bearerClass": "github-pat"}`: the class travels with its bearer, as the FM's `plane.json` records it. The first carries the boot bearer, each later one a rotation. The pipe is then the bearer's only home: a configured `planeBearer` / `planeBearerFile`, or an empty `planeBase`, refuses the boot.
3. A rotation meets the teeth a configured bearer meets (admission token, fleet-surface alias, viewer mint), then `planeMailboxClient#rotate` proves it against the bound viewer identity (see the implementation notes). On success every consumer moves to the successor and the old session closes; for a forge-PAT class the wake stream and the seat-define client re-arm with it. On a refusal, or a subject other than the bound identity, the old binding keeps serving.
4. The server answers each frame on the same fd: adopted with the identity, or refused with the reason. For another account, the answer names the subject the plane resolved.
5. The bearer lives in the server's own state only.

## Implementation notes

*(Intake 2026-11-03, from `planeMailboxClient.mjs` at `e47fd09301`.)* The swap belongs inside the client, behind a `rotate({credential})` method. Every consumer then keeps the reference it holds. Today `buildSession` closes over a `const credential`; it reads the current bearer instead. The client's invariants bound the change:
- **Single-flight acquisition** (`establishing`, `acquireProven`): a lazy re-proof that started with the old bearer must not store its session over a rotated one. `connectProven` checks a generation before storing and tears its own candidate down when it lost.
- **The close barrier** (`closing`): a rotation that finishes during or after `close()` tears its candidate down and answers `client closed`.
- **Calls in flight on the old session**: `callTool` already ignores a session that was replaced under it. Tearing the old session down mid-call would still fail that call as ambiguous, so the old session retires once its in-flight calls settle, enrolled in `pendingTeardowns` so `close()` drains it.
- **A proof that hangs or times out (#964) is a refusal**: the old session keeps serving.
- Rotations run one at a time.

## Acceptance Criteria

- [ ] AC-1: With the fd set, a frame with a valid bearer for the bound identity moves the running consumers to a session on the new bearer (for a forge-PAT class, the wake stream and the seat-define client too), closes the old session, and answers adopted.
- [ ] AC-2: A bearer the plane refuses, or resolves to another identity, leaves the old session serving and answers refused with the reason, naming the other identity in that case.
- [ ] AC-3: A piped bearer that aliases the admission token is refused exactly like a configured one.
- [ ] AC-4: With the fd unset, boot is unchanged.
- [ ] AC-5: The bearer reaches neither `AiConfig` nor a log line: `check-aiconfig-antipatterns` and `check-aiconfig-test-mutation` stay clean, and a spec asserts the logs.

## Out of Scope

- The FM side: the pipe at spawn, `attach()`, the "verifying PAT" state, persisting `plane.json` (Institution #843).
- The ADR 0034 amendment, which ships with #843.
- Changing the identity the fleet runs as.

## Related

Consumer: neomjs/neo-agent-institution#843, under Row 5's outcome neomjs/neo-agent-institution#424. Adjacent: neomjs/neo-agent-institution#849, the in-process Retry of a refused boot. Delivery order: this leaf, then the Institution's Brain pin move, then #843.

Decision Record impact: aligned-with `adr:0019-aiconfig-reactive-provider-ssot`; depends-on the ADR 0034 amendment in neomjs/neo-agent-institution#843.

Structure map (`npm run ai:structure-map -- --files --loc`): `ai/services/fleet` (138 files) owns both the plane client (`planeMailboxClient.mjs`) and the server. A credential-channel module beside them matches its siblings.

Live latest-open + closed sweep: checked the latest 20 open issues, plus all-state searches for `plane bearer rotate`, `credential pipe fleet` and `PAT without restart`, at 2026-11-03T08:42Z; no equivalent or prior exclusion found. A2A claim sweep (last 12 messages): no claim on this scope. Memory Core sweep: no prior decision on runtime bearer rotation. Own-assignment sweep: #133, #201, #571, #143 and #121 cover other outcomes (#121 is the July runtime-freshness epic for MCP restarts).

Ownership: self-assigned (@neo-opus-ada); I steward Row 5's outcome.

Origin Session ID: 0d705b6b-6706-4013-b114-91e7490cf576
Retrieval Hint: "fleet server rotate plane bearer credential pipe no restart"



## Timeline

- 2026-11-03T08:43:44Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-03T08:43:46Z @neo-opus-ada added the `enhancement` label
- 2026-11-03T08:43:47Z @neo-opus-ada added the `ai` label
- 2026-11-03T08:43:47Z @neo-opus-ada added the `security` label
- 2026-11-03T08:43:47Z @neo-opus-ada added the `agent-os` label
- 2026-11-03T08:44:11Z @neo-opus-ada cross-referenced by #843
- 2026-11-03T17:50:05Z @neo-opus-ada referenced in commit `6b709a0` - "feat(fleet): the fleet server adopts a piped plane bearer (#1462)

fleet.planeCredentialFd names an inherited fd that carries the plane
bearer and its class as JSON lines: the first frame is the boot bearer,
each later one a rotation, and every frame is answered on the same fd.
The pipe is then the bearer's only home.

A piped bearer meets the teeth a configured one meets (admission token,
fleet-surface alias, viewer mint), then the plane's proof for the bound
viewer through planeMailboxClient#rotate. A forge-PAT class also serves
the plane's fleet surface, so the wake stream and the seat-define client
re-arm with the successor. A refusal leaves the current bearer serving
and names another account beside the bound one.

The end-to-end spec runs the entry as the Fleet Manager spawns it, fd 3
included, against a loopback plane."
- 2026-11-03T17:51:56Z @neo-opus-ada referenced in commit `21dc206` - "feat(fleet): the credential pipe reads through the shared JSONL reader (#1462)"
- 2026-11-03T17:51:58Z @neo-opus-ada cross-referenced by PR #1463
- 2026-11-03T18:22:57Z @neo-opus-ada referenced in commit `5118008` - "feat(fleet): a piped boot bearer meets every class tooth before its answer (#1462)"
- 2026-11-03T18:58:47Z @tobiu referenced in commit `be1bde5` - "feat(fleet): the fleet server adopts a piped plane bearer (#1462) (#1463)

* feat(fleet): the plane client adopts a proven successor bearer (#1462)

planeMailboxClient gains rotate({credential}): it proves a candidate
session for the held viewer and only then makes the successor the bearer
every new session presents, so every consumer keeps its client. A
refusal, including a subject for another account (named as
resolvedIdentity), leaves the current binding serving.

The client's invariants hold across a rotation: a re-proof that started
with the replaced bearer never stores over the successor (generation
check), the replaced session retires only after the calls riding it
settle, and close() awaits a rotation in flight, which tears its
candidate down. Rotations run one at a time.

Each guard has a spec arm that a mutation disabling it turns red.

* feat(fleet): the fleet server adopts a piped plane bearer (#1462)

fleet.planeCredentialFd names an inherited fd that carries the plane
bearer and its class as JSON lines: the first frame is the boot bearer,
each later one a rotation, and every frame is answered on the same fd.
The pipe is then the bearer's only home.

A piped bearer meets the teeth a configured one meets (admission token,
fleet-surface alias, viewer mint), then the plane's proof for the bound
viewer through planeMailboxClient#rotate. A forge-PAT class also serves
the plane's fleet surface, so the wake stream and the seat-define client
re-arm with the successor. A refusal leaves the current bearer serving
and names another account beside the bound one.

The end-to-end spec runs the entry as the Fleet Manager spawns it, fd 3
included, against a loopback plane.

* feat(fleet): the credential pipe reads through the shared JSONL reader (#1462)

* feat(fleet): a piped boot bearer meets every class tooth before its answer (#1462)"
- 2026-11-03T18:58:48Z @tobiu closed this issue

