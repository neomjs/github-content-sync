---
id: 314
title: 'The deployment-state snapshot gets a fleet wire verb, bounded and observe-only'
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T20:27:11Z'
updatedAt: '2026-09-04T23:31:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/314'
author: neo-fable-clio
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
closedAt: '2026-09-04T23:31:20Z'
---
# The deployment-state snapshot gets a fleet wire verb, bounded and observe-only

## Context

The cockpit (neomjs/neo-agent-institution#21, the System keeper view) needs to render the health of the PLANES that produce its fleet truth — the connected instance's Compose services (chroma, kb-server, mc-server, orchestrator, fleet-server, ingress). That truth already exists: the orchestrator's `DeploymentStateBridgeService` writes a bounded deployment-state snapshot to `AiConfig.orchestrator.deploymentStateBridge.snapshotPath` every `writeIntervalMs` (`ai/daemons/orchestrator/services/DeploymentStateBridgeService.mjs:370` `writeSnapshotIfDue`, atomic write), and the KB/MC read tools (`get_deployment_state_snapshot`, `inspect_deployment`) consume that file without Docker or exec authority. The cockpit is a pure client of the fleet wire (D#16720; neomjs/neo-agent-institution#17 dissolves its last checkout-tree import), and the wire carries no verb for it: `FLEET_WIRE_METHODS` (`ai/services/fleet/fleetWireMethods.mjs:58`) serves agents, roster, activity, mailbox, wake routes, tasks, boot identity — never the deployment state. Verified 2026-09-04 against Brain `dev@25d374e`.

## The Problem

A mission control that cannot show its own engine room sends the operator to an MCP log-proxy or a terminal at exactly the wrong moment (the operator's ask, live session 2026-08-18: "diagnostics views (e.g. OC, MC, KB container logs)"). Tonight's live example: a peer's defect-note at 20:17Z read the Memory Core healthcheck as DEGRADED — backup never succeeded, corpus projection overdue, heavy-maintenance tasks starved — facts the snapshot carries and the cockpit cannot show. The cockpit cannot call the MCP tools — it holds a viewer-bound fleet bearer, not a plane MCP bearer (the credential-class ledger, `fleetServer.mjs:664–772`), and it must not import `ai/` (D#16720). The only honest path is the one every other cockpit truth takes: a read verb on the fleet wire, assembled server-side, projected for a client.

Two disciplines are already decided for this class of surface and this ticket inherits them rather than re-deciding: reads are **bounded and redacted** (neomjs/neo-agent-brain#27, the sibling plane-LOG verb), and a probe **declares observe-vs-mutate** (neomjs/neo-agent-brain#47).

## The Architectural Reality

- **Writer, existing:** `DeploymentStateBridgeService.collectSnapshot()` (`:434`) → `{generatedAt, services[], bridgeDiagnostics, tenantRepoSync, maintenance, …}`; each service row (`collectServiceSnapshot`, `:708–1015`) is `{schemaVersion: 1, recordType: 'deployment-service-state', serviceKey, targetIdentity, observedAt, status: {status: 'available'|'degraded', disposition}, memoryPressure, inspect, stats, logs, providerResidency, providerActivity, providerLaneShape, providerModelIdentity, heapObservation, resolvedConfig, restartChurn, classification, diagnosis, proofs}`. `resolvedConfig` and `inspect` carry host paths and container detail — a client projection must drop them. The MCP tool's wrapped output measures ~82 K characters (a peer's 2026-08-19 note): the raw file is not a wire payload.
- **Leaves, existing** (`ai/configBase.mjs:1488–1493`): `orchestrator.deploymentStateBridge.snapshotPath` (`NEO_DEPLOYMENT_STATE_BRIDGE_SNAPSHOT_PATH`, default `<plane.dataRoot>/deployment-state/snapshot.json`, `planeMember: true`), `staleAfterMs` (2 min), `maxSnapshotBytes` (256 KiB). No new leaf is needed.
- **The cross-process read-source pattern, existing:** `ai/services/fleet/createBootIdentityReadSource.mjs` builds `{produceBootIdentityFact()}` over the orchestrator's fact file (absent/unreadable → an honest advisory-`unknown`, never fabricated); `wireBootIdentityReadSource.mjs` injects it into `FleetControlBridge.bootIdentitySource` (`FleetControlBridge.mjs:87`), and `getBootIdentity()` (`:468`) serves it, advisory-`unknown` when unwired. The fleet-server boot reads the leaf at the use site: `devFleetServer.mjs:146` `wireBootIdentityReadSource({dir: AiConfig.orchestrator.dataDir})` — the ADR 0019 §5.5 entrypoint boundary. The general chain (source → wiring → bridge verb → allowlist entry → boot binding) is the same one `fleetMemories` and `fleetWakeRoutes` took.
- **The wire, existing:** `dispatchFleetRequest` (`ai/services/fleet/dispatchFleetRequest.mjs`) negotiates protocol, rejects any method not in `FLEET_WIRE_METHODS` as the `unsupportedMethod` closed state, then calls `bridge[method](params)` and returns one finite envelope; thrown operations become `operationFailed` without leaking the error. Capabilities stay `method-schema-v1` + `closed-response-states-v1` — an additive verb needs no new capability, and a client asking an older server gets the closed `unsupportedMethod` state it already renders honestly.
- **The consumer side, existing:** the cockpit's twin list `apps/agentos/config/fleetWireMethods.mjs:15` (neomjs/neo-agent-institution) and `installFleetBridge` (`apps/agentos/fleet/installFleetBridge.mjs:217`) expose every listed method as `bridge.<method>(params)`; the tasks pane (`view/fleet/tasks/Container.mjs:36`) is the youngest example of one envelope projected into a bound Store.

## The Fix

Three new modules beside their siblings in `ai/services/fleet/`, one bridge seam, one list entry:

1. `projectDeploymentStateForFleet(snapshot, {now, staleAfterMs})` — **pure**: `{state: 'ok'|'stale'|'unavailable', generatedAt, ageMs, reason, services: [{serviceKey, observedAt, status: {status, disposition}, memoryPressure: {disposition, reason}, restartChurn: {baseline, detecting}, classification: {serviceClassDeclared, appliedMemoryThreshold, sampleCount}, diagnosis: {recoveryClass, confidence, actionClass, classificationReason}}], maintenance: {backup: {phase, lastSuccessAt, lastSuccessAgeMs, lastBackup: {finishedAt, kind, status}}, starvation: {posture, breachCount}}}` — every nested block `null` when the record lacks it`. Never `resolvedConfig`, `inspect`, `logs`, `proofs`, `stats`, `heapObservation`, or any absolute path / env value / container id beyond `serviceKey`. Field names come from the snapshot's own vocabulary; nothing is renamed.
2. `createDeploymentStateReadSource({path, staleAfterMs, maxBytes, readImpl})` → `{produceDeploymentState()}`: reads the file, an oversized file is `unavailable` / `snapshot-too-large` unparsed, an absent file `snapshot-missing`, a failed or unparsable read `snapshot-read-failed`, an old one `stale` with `ageMs` from `generatedAt` — the projection above, never the raw file. It is a thin adapter over the Memory Core's `readDeploymentStateSnapshot` (`ai/services/memory-core/helpers/deploymentStateBridgeStore.mjs`), which owns the file's interpretation — the byte cap, the snapshot's own `generatedAt` aged against `staleAfterMs`, the schema diagnostics, and an absent file kept distinct from a failed read; every verdict short of a usable snapshot projects as `unavailable` under the reader's own reason (`snapshot-missing`, `snapshot-read-failed`, `snapshot-too-large`, `snapshot-path-unconfigured`, `snapshot-section-missing`, `snapshot-producer-metadata-missing`). The file's mtime is not a clock: a retained envelope copied into place stays stale.
3. `wireDeploymentStateReadSource({path, staleAfterMs, maxBytes, bridge, createSource})` — injects into `FleetControlBridge.deploymentStateSource` (new member, `null` default, mirroring `bootIdentitySource`); an empty path leaves it unwired. Called from BOTH fleet-server entrypoints — `devFleetServer.mjs` (the in-process dev transport) and `fleetServer.mjs#startFleetServer` (the composed service Compose starts) — each reading the three leaves at its own boot use site; the composed service classifies the verb in both S1 ledgers (`fleetServerPolicy.mjs`: `ready` / `read-observe`) and mounts the orchestrator's `shared-deployment-state-data` volume read-only at `/app/.neo-ai-data/deployment-state`, the same share the public KB/MC containers consume.
4. `FleetControlBridge#fleetDeploymentState()` — serves the source; unwired → `{state: 'unavailable', reason: 'deployment-state-source-unwired', services: []}`. Declared **observe-only** in its JSDoc (the #47 discipline) and it carries no actuator.
5. `'fleetDeploymentState'` appended to `FLEET_WIRE_METHODS` (Brain), with the module summary's verb inventory extended by one line.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `FLEET_WIRE_METHODS` + `fleetDeploymentState` (existing list `fleetWireMethods.mjs:58`; new verb) | this ticket; consumer neomjs/neo-agent-institution#21 | `dispatchFleetRequest({method: 'fleetDeploymentState'})` → `{state: 'ok', result: <projection>}` | older server: existing `unsupportedMethod` closed state; a thrown read: existing `operationFailed` | JSDoc inventory line | `dispatchFleetRequest` spec arm (allowlist) + bridge spec |
| `FleetControlBridge.deploymentStateSource` (new seam) + `#fleetDeploymentState()` (new) | this ticket; sibling `bootIdentitySource` `FleetControlBridge.mjs:87` | wired source → its `produceDeploymentState()` | `null` → `{state: 'unavailable', reason: 'deployment-state-source-unwired', services: []}` | JSDoc: observe-only, no actuator | bridge spec: wired / unwired arms |
| `createDeploymentStateReadSource({path, staleAfterMs, maxBytes, readImpl})` (new) | this ticket; sibling `createBootIdentityReadSource.mjs` | adapts `readDeploymentStateSnapshot`: `available` → `ok`, `stale` → `stale` + `ageMs` from `generatedAt` | every other reader verdict → `unavailable` under the reader's own reason (`snapshot-missing` ≠ `snapshot-read-failed`, `snapshot-too-large`, `snapshot-path-unconfigured`, the schema reasons) | JSDoc | spec on real files through the producer's own writer: fresh, copied-old-stays-stale, `{}`, a directory at the path, absent, oversized, corrupt, unconfigured |
| `wireDeploymentStateReadSource({path, staleAfterMs, maxBytes, bridge, createSource})` (new) | this ticket; sibling `wireBootIdentityReadSource.mjs`; ADR 0019 §5.5 | injects the source at BOTH fleet-server boots (`devFleetServer.mjs`, `fleetServer.mjs#startFleetServer`); returns it; the composed service classifies the verb in `FLEET_S1_METHOD_POLICY` / `FLEET_METHOD_SCOPE_CLASSES` and mounts the snapshot share read-only | empty `path` → returns `null`, seam stays unwired | JSDoc | spec: wired / empty-path arms; `fleetServer.spec`: both ledgers + the composed-path receipt (unwired → `unavailable`, wired from a producer-written file → `ok`) |
| `projectDeploymentStateForFleet(snapshot, {now, staleAfterMs})` (new, pure) | this ticket; #27's bounded-redacted discipline | the field set in Fix 1, names from the snapshot vocabulary | `null`/non-object snapshot → `unavailable`; a service row missing a field → that field `null`, never invented | JSDoc | spec: a fixture snapshot with `resolvedConfig`/`inspect`/paths → none survive (red-first on the leak) |
| `AiConfig.orchestrator.deploymentStateBridge.{snapshotPath, staleAfterMs, maxSnapshotBytes}` (existing, `configBase.mjs:1490–1493`) | ADR 0019 | read once at each fleet-server boot use site (`devFleetServer.mjs`, `fleetServer.mjs#startFleetServer`), passed into the wiring | unchanged leaves; no new leaf, no default outside the leaf | none | `lint-config-template-ssot` green |
| Institution twin `FLEET_WIRE_METHODS` (existing, `apps/agentos/config/fleetWireMethods.mjs:15`) | neomjs/neo-agent-institution#21 (consumer) | gains `'fleetDeploymentState'` so `bridge.fleetDeploymentState()` exists | an older client never offers the method | JSDoc | Institution's bridge spec |

## Decision Record impact

`aligned-with ADR 0018` (Brain truth reaches the Body cockpit over the fleet wire, never through local `ai/` imports) · `depends-on ADR 0019` (the boot-boundary leaf read, §5.5; no new leaf, no pass-along beyond the named bootstrap boundary the sibling already uses) · `aligned-with` the D#14501 lineage (observe-only exposure; the restart actuator stays out) · outside ADR 0020's scope (the cockpit remains a pure client).

## Acceptance Criteria

- [ ] `projectDeploymentStateForFleet` is pure and unit-proven: the fixture snapshot's `resolvedConfig`, `inspect`, `logs`, `proofs`, absolute paths and env values do not survive projection (red-first).
- [ ] The read-source adapts the Memory Core's `readDeploymentStateSnapshot`: `ok`, `stale` (from `generatedAt` — a copied old envelope stays stale) and `unavailable` under the reader's own reasons (`snapshot-missing` ≠ `snapshot-read-failed`, `snapshot-too-large`, `snapshot-path-unconfigured`, `snapshot-section-missing`) each have a spec arm on real files written by the producer's own writer.
- [ ] `FleetControlBridge#fleetDeploymentState()` serves a wired source and degrades honestly when unwired; `dispatchFleetRequest` accepts the method and still rejects an unknown one.
- [ ] Both fleet-server entrypoints (`devFleetServer.mjs`, `fleetServer.mjs#startFleetServer`) wire the source reading the three leaves at their boot use site; the composed profile mounts `shared-deployment-state-data` read-only into `fleet-server`; `fleetDeploymentState` is `ready` / `read-observe` in both S1 ledgers; a composed-path receipt boots `startFleetServer` against a producer-written snapshot and serves the projection without pre-populating the bridge; `lint-config-template-ssot` and `check-aiconfig-antipatterns` stay green.
- [ ] The focused fleet specs are green at the head (CI executes its smoke set and collects the rest; pre-existing reds are named in the PR); both boot logs name the wiring (`[fleet] deployment-state read-source wired (<path>)` in the dev transport, `[FleetServer] deployment-state read-source wired (<path>)` in the composed service, and the `… unwired — fleetDeploymentState answers unavailable` counterpart) — the boot-identity wiring is silent, this one is not, because an unwired seam reads exactly like an empty plane from the cockpit's side.
- [ ] Observe-only is declared in the verb's JSDoc and the wire inventory; no params are accepted (a `params` object is ignored, not validated into an attack surface).

## Out of Scope

Plane LOG reads (neomjs/neo-agent-brain#27 owns them) · restart / remediation (neomjs/neo-agent-brain#54, D#14501 lineage) · the System view itself and its consumer twin change (neomjs/neo-agent-institution#21) · a new wire capability flag · the in-process (Option A) live-service injection — the file read-source is mode-agnostic like its sibling, and a live source is a follow-up only if a measurement shows the file lagging.

## Avoided Traps

- **Forwarding the raw snapshot file.** It is written for KB/MC tools inside the plane's trust boundary and carries `resolvedConfig` paths and container detail; the wire projection is the redaction, decided here, not left to the client.
- **A fleet-server-side prober.** The fleet server must not gain Docker/exec authority to re-probe what the orchestrator already observed (the credential-class ledger); it reads the orchestrator's evidence.
- **A new leaf or a threaded config object.** Three leaves exist; the boot site reads them (ADR 0019 §5.5), the sibling wiring is the shape.

## Related

neomjs/neo-agent-institution#21 (consumer; blocked by this ticket) · neomjs/neo-agent-brain#27 (sibling plane-log verb) · neomjs/neo-agent-brain#47 (observe-vs-mutate declaration) · neomjs/neo-agent-brain#54 (recovery, out of scope) · neomjs/neo-agent-brain#217 (the client-safe contract subpath this verb rides once it exists) · D#14501 (observe-only exposure lineage) · D#16720 (FM as pure client) · neomjs/neo-agent-institution#17 (harness demotion) · `ai/services/fleet/createBootIdentityReadSource.mjs` + `wireBootIdentityReadSource.mjs` (the sibling pair).

Structure map (`npm run --silent ai:structure-map -- --files --loc`, 2026-09-04): owning folder `ai/services/fleet/`, sibling precedent the boot-identity read-source pair.

Live latest-open sweep: checked the latest 20 open issues (created-desc) at 2026-09-04T20:25:08Z — no equivalent (nearest: #217, the contract subpath; #27, the log sibling). A2A claim sweep: latest 30 messages, all read-states, at 20:25Z — no claim or intent on a deployment-state verb or the System view. Memory Core rationale sweep (`query_raw_memories`, problem nouns): no prior decision against a wire verb; my own 2026-08-03 `fleetWakeRoutes` chain is the general precedent, and a peer's 2026-08-19 note records the MCP snapshot at ~82 K characters — the projection's bound is the point. Own-assignment sweep: my open Brain tickets (#37, #50, #51, #53) touch grants, roster and wake ingress, not this surface.

Origin Session ID: 46962d8b-08f3-49a3-8049-d74e2052af37

Retrieval Hint: `query_raw_memories("fleetDeploymentState deployment-state snapshot fleet wire read-source cockpit system view")`

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 46962d8b-08f3-49a3-8049-d74e2052af37


## Timeline

- 2026-09-04T20:27:12Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T20:27:13Z @neo-fable-clio added the `enhancement` label
- 2026-09-04T20:27:13Z @neo-fable-clio added the `ai` label
- 2026-09-04T20:27:14Z @neo-fable-clio added the `architecture` label
- 2026-09-04T20:27:14Z @neo-fable-clio added the `agent-os` label
- 2026-09-04T20:28:16Z @neo-fable-clio cross-referenced by #21
- 2026-09-04T20:41:05Z @neo-fable-clio cross-referenced by PR #315
- 2026-09-04T23:14:32Z @neo-fable-clio referenced in commit `9dfa498` - "fix(fleet): the deployment-state reader adapts the Memory Core's snapshot reader (#314)

The reader no longer interprets the snapshot file on its own: it adapts readDeploymentStateSnapshot, so freshness comes from the envelope's generatedAt (a retained snapshot copied into place stays stale), a missing section is the reader's degraded verdict, and a failed read is distinct from an absent file. The spec runs on real files written by the producer's own writer."
- 2026-09-04T23:14:32Z @neo-fable-clio referenced in commit `0f84771` - "feat(fleet): the composed fleet server serves fleetDeploymentState from the shared snapshot mount (#314)

startFleetServer wires the deployment-state read-source from the three resolved leaves at its own boot use site, both S1 ledgers classify the verb (ready, read-observe), and the composed profile mounts the orchestrator's shared-deployment-state-data volume read-only into fleet-server. A composed-path receipt boots startFleetServer twice: unwired it answers unavailable, wired from a producer-written snapshot it answers the redacted projection."
- 2026-09-04T23:15:17Z @neo-fable-clio referenced in commit `3ae5f12` - "feat(fleet): the composed fleet server serves fleetDeploymentState from the shared snapshot mount (#314)

startFleetServer wires the deployment-state read-source from the three resolved leaves at its own boot use site, both S1 ledgers classify the verb (ready, read-observe), and the composed profile mounts the orchestrator's shared-deployment-state-data volume read-only into fleet-server. A composed-path receipt boots startFleetServer twice: unwired it answers unavailable, wired from a producer-written snapshot it answers the redacted projection."
- 2026-09-04T23:31:20Z @tobiu referenced in commit `97335d2` - "Merge pull request #315 from neomjs/agent/314-fleet-deployment-state-verb

feat(fleet): the deployment-state snapshot gets a wire verb, bounded and observe-only (#314)"
- 2026-09-04T23:31:21Z @tobiu closed this issue
- 2026-09-04T23:31:49Z @neo-fable-clio cross-referenced by #113
- 2026-09-04T23:32:06Z @neo-fable-clio cross-referenced by #322
- 2026-09-05T00:48:45Z @neo-fable-clio cross-referenced by #323
- 2026-09-05T00:54:44Z @neo-fable-clio cross-referenced by #324
- 2026-09-05T01:26:14Z @neo-fable-clio cross-referenced by PR #114
- 2026-09-05T01:56:18Z @neo-gpt-emmy cross-referenced by PR #325
- 2026-09-05T13:04:05Z @neo-fable-clio cross-referenced by PR #329

