---
id: 567
title: 'The death channel never reads, and its failed read projects as no death'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-27T09:30:09Z'
updatedAt: '2026-09-27T10:29:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/567'
author: neo-opus-vega
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
closedAt: '2026-09-27T10:29:00Z'
---
# The death channel never reads, and its failed read projects as no death

## Context

On 2026-09-27 at 07:24:28Z the local plane's mc-server aborted with a V8 heap OOM (`FATAL ERROR: Ineffective mark-compacts near heap limit`, `--max-old-space-size=768`, 9.7 h up) and Docker restarted it at 07:24:29Z (`RestartCount 1`). The Memory Core `healthcheck` at 08:12Z and again at 08:22Z read `lastDeath: {status: 'available', record: null}` — "observed, no death" — for a service that had died 48 minutes earlier. Receipt on #64 (comment 5854182787). #466 (closed by PR #497) built this channel precisely so that a plane death reaches an observer; this is the first real death since it shipped, and the channel did not carry it.

## The Problem

Two defects, one channel, read at Brain dev `4bc885b` on the plane:

1. **The Docker event read fails every cycle.** The deployment snapshot's mc-server entry carries `deathRead: {status: 'unavailable', unavailableReason: 'runtime-access-error'}` and, in `errors`, the Engine's answer to `GET /events?since=2026-09-27T09:22:48.208Z&until=…`: `HTTP 500 {"message":"strconv.ParseInt: parsing \"2026-09-27T09:22:48\": invalid syntax"}`. Reproduced against the plane's socket (Docker 29.2.1, API 1.53): an RFC3339 `since` is refused with that message, a Unix-seconds `since` is accepted. `DeploymentRuntimeAccessService#readTargetEvents` builds the query from `normalizeDockerTime`, which returns the ISO string on purpose — its JSDoc says "Docker accepts RFC3339Nano", which is true of the `docker` CLI (it converts) and false of the Engine API's `/events`, whose `since` / `until` are Unix timestamps (seconds, optionally fractional). Observation, not inference: the snapshot's own error text and the socket reproduction.
2. **A failed read is projected as "observed, no death".** `selectLastServiceDeath` (`ai/services/memory-core/helpers/deploymentStateBridgeStore.mjs`, imported by both toolServices) answers `available` / `record: null` whenever the snapshot itself was readable and the service is present — it never consults the service entry's `deathRead.status`. So the healthcheck's `lastDeath` says exactly what #466 RA-1 forbade at the snapshot level ("a FAILED read is `unavailable`, never 'no death'"), one level down. The `McpServerToolLimits.spec` arms pass because their fixtures never carry a per-service `deathRead`.

## The Architectural Reality

- `ai/daemons/orchestrator/services/DeploymentRuntimeAccessService.mjs`: `normalizeDockerTime` (validate-and-return-ISO), `readTargetEvents` (`/events?since=…&until=…&filters=…`, :818–845), `readTargetLogs` (the `/containers/{id}/logs` read, whose `since` / `until` the Engine likewise documents as Unix timestamps — to be verified in the same pass, its snapshot object shows `appliedSince` / `appliedUntil`).
- `ai/daemons/orchestrator/services/DeploymentStateBridgeService.mjs`: `resolveEventWindow` (:1512, cursor or `eventLookbackMs` = 5 min), the events read and `deathRead` disposition (:760–812), `foldDockerDeathEvents` (:2936–, every `die`, `oomKilled` from a preceding `oom`), `recentDeathsByService` / `eventCursorByService`. With the read failing, the cursor never advances and the 5-minute lookback never reaches a death older than the last cycle — the window itself is fine once the read works.
- `selectLastServiceDeath` (the shared projection) and its consumers in `ai/mcp/server/memory-core/toolService.mjs:348` and the KB toolService; the `#466 RA-1` arms in `McpServerToolLimits.spec.mjs:485–540`.
- #463 / #466 history: a death that reads as clean is the named failure class; #497 shipped the channel with L2 evidence and this plane never produced a positive L4 receipt because the read has been failing since.

## The Fix

1. The query bounds of `readTargetEvents` / `readTargetLogs` are spelled the way the Engine API takes them: Unix seconds with the RFC3339 fraction carried **verbatim** — Docker's inspect stamps carry nanoseconds and `Date.parse` keeps milliseconds, so a conversion through `Date.parse` sends one value for two bounds inside the same millisecond (a wire interval that disagrees with the `appliedSince` / `appliedUntil` receipt; found in review, PR #569 R1). The receipts keep the RFC3339 spelling the caller supplied. One arm per endpoint pins the query shape, one arm proves the RFC3339 form is no longer produced, and one arm holds two nanosecond bounds inside one millisecond distinct on the wire and equal to their receipt.
2. `selectLastServiceDeath` consults the service entry's `deathRead`: `status: 'unavailable'` (or `'disabled'`) is forwarded with its `unavailableReason`; `record` stays non-null only under a real `available`. A fixture arm with `deathRead.status: 'unavailable'` and `deaths: null` expects `unavailable` / `runtime-access-error`, never `available` / `null`.
3. Post-merge (L4, receipt on #64): after the recreate, the snapshot's `deathRead.status` reads `available` for every service, and the next real death (or a deliberate `docker kill -s KILL` of fleet-server, the smallest service) appears in `lastDeath.record` with its `exitCode`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `healthcheck.lastDeath` (MC + KB tools) | `selectLastServiceDeath` | `unavailable` + reason when the service's event read failed; `available` + record/null only after a successful read | snapshot unreadable → unchanged (`unavailable` / snapshot reason) | tool description unchanged (the vocabulary already exists) | projection arms + the L4 receipt |
| `DeploymentRuntimeAccessService#readTargetEvents` / `readTargetLogs` query bounds | Engine API `/events`, `/containers/{id}/logs` | Unix-seconds `since` / `until`, the RFC3339 fraction verbatim (up to nanoseconds); receipts unchanged | — | JSDoc | query-shape arms + the same-millisecond arm |
| snapshot `services[].deathRead` / `deaths` | `DeploymentStateBridgeService` | unchanged shape; `available` once the read works | unchanged | — | existing bridge arms |

## Acceptance Criteria

- [ ] AC-1: `readTargetEvents` and `readTargetLogs` send Unix-seconds bounds with every fractional digit of the RFC3339 input kept, so the wire interval equals the `appliedSince` / `appliedUntil` receipt; arms pin the query string on both endpoints, prove the RFC3339 form is no longer emitted, and hold two bounds inside one millisecond distinct (red first).
- [ ] AC-2: `selectLastServiceDeath` returns `{status: 'unavailable', reason: <deathRead.unavailableReason>}` for a service whose `deathRead.status` is `unavailable`, and `{status: 'available', record}` only otherwise; the existing #466 RA-1 arms stay green.
- [ ] AC-3 (post-merge, L4; owner #64, ledger row in comment 5854871745): on the recreated plane every service's `deathRead.status` reads `available`, and one deliberate SIGKILL of fleet-server appears in `lastDeath.record` with `exitCode: 137` on the next healthcheck.

## Out of Scope

- The V8 abort's `exitCode` semantics (134 vs Docker's report) — recorded as observed once the channel reads.
- The 5-minute lookback and cursor design (#466 as shipped) — unchanged.
- mc-server's heap sizing (done on the plane 2026-09-27, #64) and the growth question (#73 owns the heap-ceiling proof).

## Related

#466 (PR #497), #463, #464, #469, #64, #73

Live latest-open sweep: latest 20 open Brain issues at 2026-09-27T09:26Z hold no equivalent; exact sweeps for `lastDeath` / `death record` / `mcpHealthcheck` / `OOM` return #466 (closed, this amends its shipped read), #463 (closed, not planned), #73 (heap ceilings, open, Grace), #140 (the probe's token gate). A2A in-flight sweep (last 30, any read state): no claim on the death channel. MC sweep: `query_raw_memories` on the symptom → my 09-24 #463/#466 turns and Ada's kernel-log finding; no later decision. Own-assignment sweep: 8 open, none overlapping. Structure map: owning folder `ai/daemons/orchestrator/services` (runtime access + bridge) and the shared projection in `ai/services/memory-core/helpers/deploymentStateBridgeStore.mjs`; no new file.

Decision Record impact: none (ADR 0025/0026 self-diagnosis unchanged; this makes the shipped channel read).

Origin Session ID: 574ae0b8-b8d0-40d3-8cf6-1693ec48674a
Retrieval Hint: "Docker events since RFC3339 strconv.ParseInt invalid syntax deathRead unavailable runtime-access-error; selectLastServiceDeath available null after a failed read; mc-server V8 OOM 2026-09-27 07:24Z"


## Timeline

- 2026-09-27T09:30:10Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-27T09:30:11Z @neo-opus-vega added the `bug` label
- 2026-09-27T09:30:11Z @neo-opus-vega added the `ai` label
- 2026-09-27T09:30:11Z @neo-opus-vega added the `agent-os` label
- 2026-09-27T09:37:06Z @neo-opus-vega referenced in commit `2585173` - "chore(orchestrator): two ADR references carry the typed not-ticket-ref escape the archaeology gate reads (#567)"
- 2026-09-27T09:37:31Z @neo-opus-vega cross-referenced by PR #569
- 2026-09-27T10:01:17Z @neo-opus-vega cross-referenced by #64
- 2026-09-27T10:09:44Z @neo-opus-vega referenced in commit `db20bae` - "fix(orchestrator): the Docker query bound keeps every fractional digit its receipt echoes (#567)

`toDockerQueryTime` converted through `Date.parse`, which keeps milliseconds
while Docker's inspect stamps carry nanoseconds: two bounds inside one
millisecond reached the wire as ONE value while `appliedSince` / `appliedUntil`
echoed the distinct originals (PR #569 R1, found by @neo-gpt-emmy). The
fraction now travels verbatim; the logs and events arms pin the digits, and one
arm holds two same-millisecond bounds distinct on the wire and equal to their
receipt."
- 2026-09-27T10:21:16Z @neo-opus-vega cross-referenced by PR #570
- 2026-09-27T10:29:00Z @tobiu referenced in commit `25e7966` - "Merge pull request #569 from neomjs/vega/567-death-channel-reads

fix(orchestrator): Docker event and log bounds travel as Unix seconds, and a failed death read is projected as unavailable (#567)"
- 2026-09-27T10:29:00Z @tobiu closed this issue
- 2026-09-27T10:49:47Z @neo-opus-vega cross-referenced by PR #564

