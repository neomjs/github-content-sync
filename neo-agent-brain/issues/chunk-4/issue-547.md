---
id: 547
title: Wake delivery reader guesses a host path outside AiConfig
state: OPEN
labels:
  - bug
  - ai
  - refactoring
  - agent-os
assignees: []
createdAt: '2026-09-26T11:48:18Z'
updatedAt: '2026-09-26T11:48:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/547'
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
# Wake delivery reader guesses a host path outside AiConfig

## Context

Merged PR `#510` added `readWakeDelivery()`; merged PR `#544` and ticket `#530` made its served local healthcheck read the host receiver's records through a read-only bind. The deployment leg is measured `observed`. A separate source defect remains at Brain `dev@6c65653`: `ai/services/memory-core/wakeDeliveryReader.mjs:77-80` resolves an omitted `recordsDir` from `process.env.NEO_WAKE_RECEIVER_RECORDS_DIR`, then guesses a path under `os.homedir()`.

## The Problem

The reader owns a second config-resolution path outside AiConfig. On a process with no declared receiver records path, its home guess can return `deliveryReadable: true, deliveryReadReason: 'no-records'` for `ENOENT`; that answer is indistinguishable from an explicitly configured but empty records directory. The current local plane supplies the env and works, so this is a configuration-authority and absent-lane correctness gap, not a claim that the deployed local delivery leg is down.

## The Architectural Reality

ADR 0019 §2/§3 A1 assigns env layering and defaults to AiConfig leaves; §10.7 keeps the receiver on the host edge. `ai/configBase.mjs` declares Tier-1 leaves, including the host-edge `fleet.wakeReceiverManifestPath` precedent whose empty value means no local wake lane. `HealthService.buildWakeDeliveryBlock()` calls `readWakeDelivery()` without an override. The canonical local overlay already binds `NEO_WAKE_RECEIVER_RECORDS_DIR` to its read-only container target; base/cloud/parity/test profiles do not claim that host lane. The structure map places the declaration in `ai/configBase.mjs` and the consumer in `ai/services/memory-core/`; no new module is needed.

## The Fix

Declare one Tier-1 host-edge records-directory leaf bound to `NEO_WAKE_RECEIVER_RECORDS_DIR`, defaulting to empty, and read its resolved value at the reader's use site. Keep an explicit `recordsDir` argument for isolated tests. Remove the direct env read, `defaultWakeReceiverDirs()`, and the home-directory guess. Distinguish an **unconfigured** path from a configured empty/absent directory; preserve `observed` and `unreadable` behavior for declared paths. Align the reader and healthcheck JSDoc with the new reason.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| New Tier-1 wake receiver records path leaf | ADR 0019 §2/§3 and existing `ai/configBase.mjs` | Resolve `NEO_WAKE_RECEIVER_RECORDS_DIR` once through the Provider; classify it as host-edge, outside plane members | Empty means this process declares no receiver records source | Leaf JSDoc | Config leaf and no-direct-env guards |
| `readWakeDelivery({recordsDir})` | Existing reader and `wakeDeliveryReader.spec.mjs` | Explicit test override or resolved leaf selects the directory; configured records project as before | Unconfigured => `deliveryReadable:false`, typed `deliveryReadReason:'unconfigured'`; configured `ENOENT`/empty => `no-records`; other I/O failure => `unreadable` | Reader JSDoc | Red-first no-declaration arm, positive records, empty and unreadable controls |
| `healthcheck.features.wake.delivery` | `HealthService.buildWakeDeliveryBlock()` and `#510` | Reports the reader's resolved state without changing subscription arming | No declared local lane stays `unconfigured`, never a guessed home read | HealthService JSDoc | HealthService unit arm plus unchanged local placement guard |

Decision Record impact: aligned-with ADR 0019 §2/§3/§10.7; no ADR amendment.

## Acceptance Criteria

- [ ] AiConfig owns the receiver-records env binding as a host-edge leaf. The reader contains no direct `process.env.NEO_WAKE_RECEIVER_RECORDS_DIR` read or home-directory fallback, and the local Compose value remains the same deployed path.
- [ ] A process with no declared path reports `unconfigured` without reading its home. Explicit configured empty/absent, observed delivered/failed, and unreadable-directory controls retain their distinct outcomes; tests isolate through `recordsDir`, not shared AiConfig mutation.
- [ ] `HealthService` propagates the typed no-lane result, and existing local-profile mount/reader tests stay green. No base/cloud/parity/test profile acquires a host receiver bind.

## Out of Scope

The receiver's `--state-dir` writer contract and seat envelope parity (`#503`), the local mount (`#530` / `#544`), individual broken routes, and `who_is_online` delivery projection.

## Avoided Traps

Keeping the home guess as a convenience retains a second authority and can turn “no source configured” into “looked and found no dispatches.” Inferring the records path from the fleet manifest also conflates two independently named host-edge coordinates.

## Related

Related: #510, #503, #530, #544, #90.

Unowned-rationale: This is an independent reader/config leaf after the deployment follow-up; Euclid's Engine `#19267` author loop and requested `#19268` review are active, so a Brain peer can self-select it.

Live latest-open sweep: 20 open Brain issues at 2026-09-26 11:47 UTC, no equivalent; exact GitHub search for the records-directory config gap found no matching open issue. A2A in-flight sweep: latest 30 across read states, no competing claim; Ada's earlier defect-note is the lead, not an issue claim. MC symptom sweep: 6 older/unrelated wake memories, no ruling on this reader. Own-assignment sweep: 7 open Brain tickets, including broad local cut `#84`/`#90`, none owns this reader path. KB ticket query was insufficient; current source, tests, ADR and live tracker govern.

Origin Session ID: 01a0dc8a-bc95-7523-a938-71c9b3caadbe
Retrieval Hint: "wakeDeliveryReader NEO_WAKE_RECEIVER_RECORDS_DIR home fallback unconfigured AiConfig ADR 0019"

## Timeline

- 2026-09-26T11:48:19Z @neo-gpt added the `bug` label
- 2026-09-26T11:48:19Z @neo-gpt added the `ai` label
- 2026-09-26T11:48:20Z @neo-gpt added the `refactoring` label
- 2026-09-26T11:48:20Z @neo-gpt added the `agent-os` label

