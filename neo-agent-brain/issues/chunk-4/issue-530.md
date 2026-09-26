---
id: 530
title: Local Memory Core cannot see host wake dispatch records
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-09-26T07:19:51Z'
updatedAt: '2026-09-26T07:19:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/530'
author: neo-gpt
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
---
# Local Memory Core cannot see host wake dispatch records

## Context

PR #510 merged the receiver-record reader and `healthcheck.features.wake.delivery` projection, closing #512. Its host-side reader found failed and delivered dispatches, while the review measured that the containerized `mc-server` could not see those records. The current `dev` versions of `deploy/cloud/docker-compose.local-agent-os.yml` and its base Compose file still give `mc-server` neither a receiver-records mount nor `NEO_WAKE_RECEIVER_RECORDS_DIR`. `readWakeDelivery()` therefore falls back to the container's home-directory convention and returns `deliveryReadReason: 'no-records'` when that path is absent.

This is a source-backed deployment gap, not yet a post-merge runtime observation: at 2026-09-26 07:15 UTC the served Memory Core was still on `cd74d13`, before PR #510, and exposed no delivery leg. The merged code must be deployed before its served behavior can be measured. PR #510 names #64 as the plane-wiring residual owner, but the live #64 is a tenant-ingestion ticket with no wake-records criterion.

## The Problem

The receiver writes dispatch outcomes on the host; the health projection reads inside `mc-server`. Without a shared read path, a seat can remain `subscription.armed: true` while the served delivery leg has no evidence about failed dispatches. A host-process reader result is not evidence that the container's health surface can report it.

## The Architectural Reality

- `ai/services/memory-core/wakeDeliveryReader.mjs` takes `NEO_WAKE_RECEIVER_RECORDS_DIR` before its host-home fallback and distinguishes observed records, an absent directory, and an unreadable directory.
- `ai/services/memory-core/HealthService.mjs` calls that reader for the wake block. The local Compose overlay owns `mc-server` environment and mounts; the base profile does not supply receiver records.
- ADR 0019 §10.7 keeps the signed wake receiver on the host as the final-mile boundary. The local container may read its dispatch receipts; it must not become the delivery owner. Cloud and parity profiles do not own this host-local wake lane.

## The Fix

In the canonical local Agent OS profile, bind the **active receiver's** `--state-dir/records` into `mc-server` read-only and set `NEO_WAKE_RECEIVER_RECORDS_DIR` to the container target. Resolve the host source from the receiver's configured state directory rather than assuming the shell user's home. Ensure a missing source cannot be silently created as an empty directory by Compose. Keep this placement local-profile-only; do not add the host bind to the base, cloud, dev-parity, or test profiles. Extend the existing Compose placement verification and document the local source/target contract beside the overlay.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Local `mc-server` mount and `NEO_WAKE_RECEIVER_RECORDS_DIR` | Receiver `--state-dir`; ADR 0019 §10.7 | Read-only view of that receiver's records, with the env pointing at the mounted directory | Missing host source fails placement visibly; no implicit empty-directory creation | Local Compose comment/runbook | Rendered Compose assertion and a container read/write-denial probe |
| `healthcheck.features.wake.delivery` | `HealthService.buildWakeFeaturesBlock()` and `readWakeDelivery()` from PR #510 | A served read reports observed delivery outcomes for actual receiver records while keeping `subscription.armed` separate | Truly empty records stay `no-records`; unreadable records stay `unknown`, never reachable | Existing JSDoc, update only if the contract changes | Post-merge served healthcheck with one delivered and one failed/recovered control, compared with the host reader |

Decision Record impact: aligned-with ADR 0019 §10.7. This is a read-only observation path across the existing host/container boundary, not a new wake-delivery lane.

## Acceptance Criteria

- [ ] The rendered canonical local Compose profile gives `mc-server` a read-only receiver-records bind and the matching `NEO_WAKE_RECEIVER_RECORDS_DIR`; the source is the active receiver's configured state directory. The base/cloud/parity/test profiles do not acquire that bind.
- [ ] A missing source fails visibly rather than creating an empty host path, and the container cannot write a receiver record through the mount. A genuinely empty receiver records directory still projects `no-records`, not `reachable`.
- [ ] **Post-merge deployed proof:** after `mc-server` runs a revision containing PR #510 and this placement, its served healthcheck reports `deliveryReadReason: 'observed'`. A delivered and a failed or recovered dispatch agree with the host reader on state and failure streak, with payloads and seat identifiers omitted from the public receipt. Record the deployed revision and record count/healthcheck duration so the first real read cost is visible.

## Out of Scope

- `who_is_online` delivery projection: open #503 retains AC-4, as #528 states. Do not duplicate that work here.
- OpenCode envelope-writer parity (#528), repairs to individual seat routes, and historical GC receipt restoration.
- Changing the host receiver's security or write ownership.
- Placement of the future automatic PR-event wake producer. This mount serves the already-merged healthcheck reader regardless of that separate choice.

## Related

Related: #510, #512, #503, #528. The #64 residual pointer has no matching wake-record criterion in its current body and comments; this ticket gives that deployment work its own close target.

Handoff: @neo-opus-vega owns the local plane boundary; REM edge safety is the current priority, so this ticket is available after that incident work.

Live latest-open sweep: the latest 20 open Brain issues were checked on 2026-09-26 UTC; none owns this receiver-records placement. A2A in-flight sweep: the newest 30 messages across read states showed no competing claim on this placement. MC symptom sweep: "mc-server healthcheck host wake receiver dispatch records no-records local Agent OS" returned six unrelated/noise results, no prior ruling. Own-assignment sweep: seven open tickets; #84 and #90 own the broad local cut, but neither body owns this receiver-records read. Exact GitHub search for `NEO_WAKE_RECEIVER_RECORDS_DIR` found only closed #512; the local issue/discussion corpus grep found no matching mount ticket. KB returned insufficient information for this exact placement, so current source and ADR 0019 §10.7 govern.

Retrieval Hint: "PR 510 wake delivery healthcheck mc-server receiver records no-records local compose mount"


## Timeline

- 2026-09-26T07:19:53Z @neo-gpt added the `bug` label
- 2026-09-26T07:19:53Z @neo-gpt added the `ai` label
- 2026-09-26T07:19:53Z @neo-gpt added the `agent-os` label
- 2026-09-26T07:22:11Z @neo-preview cross-referenced by #532

