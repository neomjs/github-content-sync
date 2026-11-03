---
id: 1442
title: Retire the completed stdio-to-HTTP latency comparison
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
assignees:
  - neo-gpt-euclid
createdAt: '2026-11-02T23:28:49Z'
updatedAt: '2026-11-02T23:58:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1442'
author: neo-gpt-euclid
commentsCount: 0
parentIssue: 84
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-02T23:58:54Z'
---
# Retire the completed stdio-to-HTTP latency comparison

## Context
Part of #84's accepted post-cutover cleanup. The machine now consumes one Docker Agent OS; the old latency actor still constructs a second stdio plane solely to compare the retired migration alternatives. This leaf replaces the capture row's execution scope in #84; it does not duplicate the broader cleanup.

Design authority: #84's current consumer ledger and [scope census](https://github.com/neomjs/neo-agent-brain/issues/84#issuecomment-5427326350). Decision Record impact: aligned-with ADR 0019 §10.7–10.8; no profile, role, config-leaf or ADR change.

## Problem and architectural reality
At Brain `613df5d7c99283051d78ccbe485fed5f2e4e17c4`, `ai/scripts/diagnostics/captureParityLatencyPair.mjs` and `parityLatencyPair.mjs` own the comparison (1,085 + 233 code LOC measured by structure-map). The capture actor exclusively uses `deploy/cloud/docker-compose.parity-capture.yml` and `Caddyfile.parity-capture`.

`FleetLifecycleService.capturePreparedRemoteMcpLatencyPair` is the only production importer. It has no caller in the current source tree or MCP OpenAPI surface. `inspectPreparedRemoteMcpAdapter` still builds its capture-only `capturePlan`, although its real startup caller, `startAgentProvisioned.mjs:783`, only requires successful inspection. That startup guard must remain.

## Fix and contract ledger
Retire the complete comparison chain in one PR; use the existing owning files, with no replacement abstraction or compatibility shim.

| Target surface | Authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Comparison modules, dedicated specs, capture Compose/Caddy | #84 capture retirement row | Delete together; remove package-boundary/provider-lane fixture entries | None: comparison is retired | Existing module intent plus this ledger records retirement | Whole-tree reference census; retained package/provider tests |
| `FleetLifecycleService.capturePreparedRemoteMcpLatencyPair` | #84 callable-wrapper census | Remove the unused wrapper with its driver | None | Method removal is explicit here and in the PR | No source or MCP caller; source/import checks |
| `inspectPreparedRemoteMcpAdapter` | Real `startAgentProvisioned` caller | Preserve all input, installed-adapter and secret-isolation checks; return the inspection receipt without `capturePlan` | Existing non-Codex behavior remains | Update return JSDoc | Existing positive and rejection tests, including packaged Electron runtime and Start |
| Canonical/parity coverage and scheduler capabilities | #84 retained rows | Unchanged | Existing behavior | ADR unchanged | Protected-file diff and owning guard suites |

## Acceptance criteria
- [ ] Remove the two comparison modules, their dedicated tests and capture-only Compose/Caddy artifacts as one closed dependency chain.
- [ ] Remove the unused Fleet capture wrapper and capture-plan construction; preserve installed-adapter inspection and actual Start consumption, with its existing positive/rejection/secret-isolation coverage passing.
- [ ] Update fixture enumerations, leave no dangling executable references, and keep `integration-parity`, canonical volume guards, host scheduling and the three permanent equivalence guards unchanged and passing.
- [ ] Report deleted production code separately from test/config/docs changes.

Validation used the existing locked Engine 13.3.1 after restoring a stale local installation (13.1.0). The strict guide parity guard then passed unchanged; no dependency pin, guide, OpenAPI or tolerance change is needed.

## Out of scope and avoided traps
No runtime restart, deployment, credential change, data deletion, profile retirement, scheduler removal or broad Fleet refactor. Keep `inspectPreparedRemoteMcpAdapter`; removing its obsolete receipt payload is not removing its startup gate. Do not delete files merely because their names contain parity.

## Provenance and ownership
Owner: @neo-gpt-euclid. Live latest-open sweep: 20 newest open Brain issues checked on 2026-11-03; no equivalent leaf. Recent all-status A2A claims show no overlap. Own-assignment sweep found #84/#90 as the existing outcome, not a competing capture leaf; current #84 body was read. MC recovered the 2026-11-02 consumer audit (session `adb24960-e16c-4eaf-9604-b6e2ad519624`); no newer retirement decision surfaced. Structure-map measured the two diagnostic owners above. Independent source reader confirmed the complete closure and retained Start consumer. Cross-family PR review checks the final prescription.

Origin Session ID: adb24960-e16c-4eaf-9604-b6e2ad519624

Retrieval Hint: `Brain 84 stdio HTTP latency capture retirement FleetLifecycleService capturePlan`.


## Timeline

- 2026-11-02T23:28:49Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-11-02T23:28:51Z @neo-gpt-euclid added the `enhancement` label
- 2026-11-02T23:28:51Z @neo-gpt-euclid added the `ai` label
- 2026-11-02T23:28:51Z @neo-gpt-euclid added the `refactoring` label
- 2026-11-02T23:39:32Z @neo-gpt-euclid cross-referenced by PR #1444
- 2026-11-02T23:41:11Z @neo-gpt-euclid cross-referenced by #84
- 2026-11-02T23:58:54Z @tobiu referenced in commit `41d360a` - "fix(fleet): retire the completed latency comparison (#1442) (#1444)"
- 2026-11-02T23:58:55Z @tobiu closed this issue

