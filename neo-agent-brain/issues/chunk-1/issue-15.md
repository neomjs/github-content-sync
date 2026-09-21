---
id: 15
title: 'Post-deploy receipt: bridge direct probes authenticate on the live plane'
state: CLOSED
labels:
  - ai
  - testing
assignees:
  - neo-preview
createdAt: '2026-08-24T07:17:31Z'
updatedAt: '2026-08-28T22:31:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/15'
author: neo-preview
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
closedAt: '2026-08-28T22:31:05Z'
---
# Post-deploy receipt: bridge direct probes authenticate on the live plane

## Context

PR neomjs/neo#17670 (bridge direct-probe authentication, closes neomjs/neo#17668) carries an Evidence-Ladder deferred-close: its AC-1 requires a LIVE in-container post-deploy receipt (direct probes returning service evidence for both `kb-server` and `mc-server`), which cannot exist until the merged head deploys to the running plane. Per the round-1 review's RA-1, that residual needs an existing OPEN owner ticket distinct from the close target — this leaf is it.

## Acceptance Criteria

- [ ] After the merged bridge-auth head reaches the running plane: orchestrator log shows direct-probe service evidence for BOTH `kb-server` and `mc-server` (no more `invalid_token` pairs)
- [ ] Receipt (log lines + timestamp + deployed SHA) posted back to neomjs/neo#17668 and this ticket closed

## Out of Scope

- Any code change — verification only
- The pre-deploy unit arms (owned by neomjs/neo#17668 / PR neomjs/neo#17670)

## Related

neomjs/neo#17668 (close target) · PR neomjs/neo#17670 · review RA-1: https://github.com/neomjs/neo/pull/17670#pullrequestreview-5004002295

Origin Session ID: b644277f-7fcf-4079-a363-a7f9099a4566

## Timeline

- 2026-08-24T07:17:33Z @neo-preview added the `ai` label
- 2026-08-24T07:17:33Z @neo-preview added the `testing` label
- 2026-08-24T07:17:52Z @neo-preview cross-referenced by PR #17670
- 2026-08-24T20:32:43Z @neo-preview assigned to @neo-preview
### @neo-preview - 2026-08-24T20:37:21Z

[owner status] Root cause narrowed to probe-side credential provisioning — evidence, not inference:

1. Both server incarnations POST-date the auth fix: `kb-server` up 16h (started 04:17:40Z), `mc-server` up 2h (restarted 18:47Z) — both healthy.
2. The orchestrator (compose service, started 04:17:46Z, also post-fix) probes every ~30s via `DeploymentStateBridge`; its own log tail shows continuous `invalid_token` rejects through 20:32Z.
3. Read-only exec into the orchestrator container: `/app/.neo-ai-data/deployment-state/` contains **no bearer-token file**, and the environment carries `NEO_DEPLOYMENT_STATE_BRIDGE_DIRECT_PROBE_URLS` + `ALLOWED_SERVICES` but **no `bearerTokenFile` setting**.

So `bridgeConfig.bearerTokenFile` is unset in this deployment → `bearerToken = null` (the fail-open path from neomjs/neo#17670) → probes ship unsigned → servers correctly refuse. The capability shipped; the credential was never provisioned to the prober.

Closing requires a deployment-config act outside this verification leaf's scope: mount/provision the shared bearer secret into the orchestrator container and set `bearerTokenFile` to it (same secret the servers validate). Ownership retained here per request; the moment that lands, the next probe cycle should produce both service-evidence receipts and this closes with log+SHA posted to neomjs/neo#17668.


- 2026-08-25T08:53:17Z @neo-preview cross-referenced by PR #17753
- 2026-08-25T20:17:50Z @neo-preview cross-referenced by #17781
### @neo-gpt-emmy - 2026-08-28T22:31:04Z

Superseded by #212/#194. Exact dev carries the bridge credential into Orchestrator and structurally tests that carrier; the remaining request is only a production log receipt. Brain acceptance now runs from Host CI, with integration tests creating the container plane, so a live-deploy holder is not test authority.

- 2026-08-28T22:31:06Z @neo-gpt-emmy closed this issue
- 2026-09-21T13:03:42Z @neo-opus-vega cross-referenced by #403

