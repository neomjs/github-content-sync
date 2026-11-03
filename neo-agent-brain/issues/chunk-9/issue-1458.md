---
id: 1458
title: Retire local MC upstream sockets before the server idle timeout
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-03T07:05:58Z'
updatedAt: '2026-11-03T14:26:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1458'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 904
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-03T14:26:29Z'
---
# Retire local MC upstream sockets before the server idle timeout

## Context

Source implementation split from #904. That existing issue retains the operational outcome and installed authenticated observation after human merge; this leaf owns the already-tested local ingress change. It is not a new residual placeholder.

## The Problem

The local MC route can reuse a connection as the upstream closes it, yielding an empty HTTP 502. [The experimental receipt](https://github.com/neomjs/neo-agent-brain/issues/904#issuecomment-6597108413) records unauthenticated, session-free POST controls and Caddy's upstream error:

```
read: connection reset by peer
```

A paired eight-round run produced three 502s with default pooling and eight expected 401 responses with a four-second idle lifetime. Earlier clean controls and a no-pool timeout are retained; this does not diagnose every transport failure.

## The Architectural Reality

`deploy/cloud/Caddyfile.local-agent-os` owns the canonical local `/mc/*` proxy. At `e47fd093` it uses Caddy's default upstream pooling. The measured Node backend advertises a five-second keep-alive lifetime. [Caddy documents the timeout mismatch and POST failure mode](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy#the-http-transport).

Design authority: n/a — incident restoration of successful MCP transport; no API, consumer obligation or accepted architecture is changed.

## The Fix

Set `transport http { keepalive 4s }` on only the local MC reverse proxy. Explain the relation to the upstream's advertised idle limit in one adjacent comment.

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Local `/mc/*` HTTP upstream pool | #904 accepted scope and linked reproducer | Retire idle pooled connections after four seconds | Preserve errors, active request/stream settings and no-write-retry behavior | Adjacent intent comment | Paired same-backend probe; exact-file offline Caddy validation; adapted JSON delta |

Decision Record impact: none — deployment transport tuning, no topology or AiConfig change.

## Acceptance Criteria

- [ ] The local MC upstream has a four-second idle pooling lifetime; offline Caddy adaptation and validation accept the exact edited file.
- [ ] Comparing adapted base and candidate JSON changes only that MC transport timeout; KB, Fleet, path routing, identity-header stripping and error handling remain identical.
- [ ] The source handoff links the paired reproducer and its negative controls without claiming authenticated installed acceptance. #904 remains open for that observation.

## Out of Scope

Production activation; authenticated installed acceptance (owned by #904); KB/Fleet/cloud proxy changes; session lifecycle; write retries.

## Avoided Traps

Do not mask failed writes with blind retries: write tools lack idempotency keys. Do not enlarge server or active-request timeouts to address an idle connection pool. Do not close #904 from a source-only receipt or borrow an unrelated deployment ticket as its owner.

## Related

#904 — existing symptom, diagnosis, and installed-verification owner.

Ownership: Emmy; this is the concrete source part of her active incident lane. Not marked good-first-issue because the measured fix is already prepared and actively owned.

Live latest-open + closed sweep: latest 20 open and closed keepalive search checked 2026-11-03 07:05 UTC. #904 is the matching existing outcome; this splits its implementation without replacing it. Earlier bridge fixes address consequences. No competing ingress PR in the live three-PR queue.
A2A sweep: latest 30 all-state messages; only Emmy's #904 source claim overlaps.
MC sweep: empty-body 502/upstream reset queries returned unrelated initialization rows; KB recovered Ada's prior #904 hypothesis, verified against live source and experiment.
Own-assignment sweep: five open, only #904 overlaps; it remains the outcome owner.
Structure map: `npm run ai:structure-map -- --root deploy/cloud --files --loc` identifies the declarative deployment folder; no new module.

Origin Session ID: 02af80f1-0e2f-446e-9b13-c283f27e8931
Retrieval Hint: "local MC empty-body 502 upstream reset idle pool"


## Timeline

- 2026-11-03T07:05:59Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-03T07:06:00Z @neo-gpt-emmy added the `bug` label
- 2026-11-03T07:06:00Z @neo-gpt-emmy added the `ai` label
- 2026-11-03T07:06:00Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-03T07:06:42Z @neo-gpt-emmy added parent issue #904
- 2026-11-03T07:07:36Z @neo-gpt-emmy cross-referenced by PR #1459
- 2026-11-03T07:13:51Z @neo-opus-grace cross-referenced by #904
- 2026-11-03T14:26:30Z @tobiu closed this issue

