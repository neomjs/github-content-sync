---
id: 1762
title: Record sanitized proxy failures for local Memory Core POSTs
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-15T14:01:40Z'
updatedAt: '2026-11-15T14:52:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1762'
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
closedAt: '2026-11-15T14:52:26Z'
---
# Record sanitized proxy failures for local Memory Core POSTs

## Context

Child of #904. A native MC client saw an empty-body 502 at 13:34:06 UTC on November 15. The MC log recorded no matching POST error. The closest ingress warning was for a different GET. The four-second keep-alive setting was already active. [Installed observation](https://github.com/neomjs/neo-agent-brain/issues/904#issuecomment-6675574898)

## The Problem

In a disposable instance of the installed Caddy v2.11.3 image, refusing or resetting the MC upstream returned an empty 502 without a request-level error entry. Removing the existing error handler exposes request details and breaks the unavailable Fleet probe’s 404. The incident owner needs to distinguish a proxy-generated failure from an upstream application returning its own status.

## The Architectural Reality

Design authority: the [parent’s recurrence and proposed ingress diagnostic](https://github.com/neomjs/neo-agent-brain/issues/904#issuecomment-6895071295) and the existing routing contract in `deploy/cloud/Caddyfile.local-agent-os`. Caddy restores the original request before its error route. A scoped named access logger can select the original `POST /mc/mcp` there. The deployed Caddy binary supported this in a 12-case isolated baseline/candidate comparison.

## The Fix

Add a bounded local-ingress diagnostic for handled 502s on the original MC POST. Classify upstream error text into fixed labels. Keep ordinary access output discarded. Preserve the existing client response and Fleet fallback. Give the behavior a regression check and verify the actual source in an isolated Caddy instance.

| Surface | Authority | Behavior | Fallback | Evidence |
| --- | --- | --- | --- | --- |
| Local proxy diagnostics | Caddy handled error, original method/path and final status | One ERROR entry containing time, level, logger, message, duration, status, fixed service and category | Unrecognized transport error maps to a fixed generic category | Real proxy fault controls with an exact output-key check and synthetic private values |
| Upstream-supplied responses, other methods and routes | Existing local Caddyfile | Existing client status, headers and body | Fleet keeps its existing unavailable response | Positive and negative route matrix against the unchanged base |

## Acceptance Criteria

- [ ] A refused upstream and a reset on `POST /mc/mcp` each produce exactly one sanitized diagnostic with the appropriate fixed category; opaque or unfamiliar errors have a generic fallback.
- [ ] Normal requests, other methods and paths, and upstream-supplied 5xx responses produce no MC gateway diagnostic. Fleet’s unavailable response and upstream response bodies are preserved.
- [ ] Diagnostic fields exclude request and response headers, query, raw path, request/response body, session IDs, user identity and raw error text. An isolated executable check exercises private sentinels and the emitted key set.

## Out of Scope

CPU limits, general access logging, cloud deployment, retries, transport timeouts, deployment to the running plane and assigning a cause to the reported 502s. The parent stays open for a real incident after installation.

Ownership: Emmy is taking this diagnostic leaf. Related: #904, #1458.

Live latest-open + closed sweep: latest 20 Brain issues and closed ingress/502 search, 158 open (red). No open Caddy PR or matching leaf. The existing child #1458 repaired the idle pool. Existing assignment #904 owns incident attribution. Reviewed all-state recent A2A and prior memories on silent gateway failures. Placement is the existing local Caddyfile, with no new runtime module or dependency.

Origin Session ID: 01f23f7e-6e0a-4e04-993d-5106dc4bce25

## Timeline

- 2026-11-15T14:01:42Z @neo-gpt-emmy added the `bug` label
- 2026-11-15T14:01:42Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T14:01:42Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-15T14:01:58Z @neo-gpt-emmy added parent issue #904
- 2026-11-15T14:01:59Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-15T14:05:50Z @neo-gpt-emmy cross-referenced by #904
- 2026-11-15T14:09:23Z @neo-gpt-emmy cross-referenced by PR #1763
- 2026-11-15T14:52:26Z @tobiu referenced in commit `254fda1` - "feat(deploy): classify local Memory Core gateway failures (#1762) (#1763)"
- 2026-11-15T14:52:27Z @tobiu closed this issue

