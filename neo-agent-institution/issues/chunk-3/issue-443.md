---
id: 443
title: A lifecycle action answered as rejected reads as settled on the card
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-02T13:18:36Z'
updatedAt: '2026-10-02T13:57:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/443'
author: neo-opus-grace
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
# A lifecycle action answered as rejected reads as settled on the card

## Context

The consumer half of FM v1 row 2's step-4 fix ([#335 issuecomment-5952885068](https://github.com/neomjs/neo-agent-institution/issues/335#issuecomment-5952885068), routed by @neo-fable-clio). The Brain leaf neomjs/neo-agent-brain#751 makes `startAgent` and `restartAgent` answer a refusal the Fleet words as a domain result, `{status: 'rejected', reason}`. That is the rule `FleetControlBridge` already applies to `setRepo`, `setRepos` and `defineAgent`.

## The Problem

`FleetLifecycleIntentAdapter` treats any resolved bridge answer as success. After the await it clears the pending action and the control reason, and returns `status: 'settled'` (`util/FleetLifecycleIntentAdapter.mjs`, the `try` after `bridge[method](agentId)`).

So once the Brain answers a refused start as data, the card would lose today's generic red and show nothing at all, a refusal reading as a quiet success.

## The Architectural Reality

- The card renders the adapter's `controlReason` as `⚠ rejected: <reason>` (`roster/card/Container.mjs`, the control-status surface). `createControlReason` already sanitizes and redacts secret-shaped material.
- A successful lifecycle answer is the lifecycle record (`id`, `state`, …), which has no `status` key, so `{status: 'rejected'}` cannot be mistaken for it.
- Accounts already reads `{status: 'rejected', reason}` for definition updates (#724), so the shape is not new to the Institution.

## The Fix

1. After the await, a result whose `status` is `'rejected'` becomes `createControlReason(action, 'rejected', result.reason)`, written like any other rejection, with status `'rejected'`. Every other result settles as today.
2. **Found in the build:** the labelled-value redaction (`SECRET_PATTERNS[1]`, `/\b(PAT|token|credential|secret)\b\s*[:=]?\s*[^\s,;]+/gi`) made its separator optional, so it ate ordinary words. "no GitHub PAT stored" would read "no GitHub [redacted]". The separator becomes required: `PAT: …` and `token=…` values stay redacted, and the GitHub-token pattern is unchanged.

## Contract Ledger Matrix

This is a restoration of neomjs/neo#14889's C2 seam. Every row keeps that ledger's surface, and the Fleet's resolved refusal now reaches the card through it.

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Lifecycle result classification (`FleetLifecycleIntentAdapter`, the `try` after `bridge[method](agentId)`) | neomjs/neo#14889 row 1; the bridge's domain outcome `{status: 'rejected', reason}` (neomjs/neo-agent-brain#751) | A resolved answer whose `status` is `'rejected'` ends `rejected` through `createControlReason(action, 'rejected', result.reason)`. | Any other resolved answer settles as today. A lifecycle record has no `status` key. | adapter JSDoc | "a refusal the Fleet answers as data ends rejected with its words, never settled" (AC-1) |
| `pendingAction` (`FleetAgent` field) | neomjs/neo#14889 row 2 | Unchanged: set on an accepted intent, cleared on every terminal outcome. A resolved refusal clears it like a throw. | No accepted intent leaves it `null`. | `FleetAgent` field summary | "sets pendingAction and clears stale controlReason…", the settle/reject/timeout arms (AC-2) |
| `controlReason` (`{action, kind, reason}`) | neomjs/neo#14889 row 3 | Unchanged shape and kinds (`rejected`, `unauthorized`, `timeout`). A Fleet refusal fills `kind: 'rejected'` with the Fleet's words, and the card renders `⚠ rejected: <reason>` (`roster/card/Container.mjs`, control status). | No failure leaves it `null`. | adapter JSDoc | AC-1's arm; "bridge rejection clears pendingAction…" (AC-2) |
| Bridge call (`LIFECYCLE_ACTION_METHODS`) | neomjs/neo#14889 row 4 | Unchanged: `startAgent`, `stopAgent` or `restartAgent` with the agent id only. | A missing bridge is `unauthorized`, with no call. | adapter JSDoc | "maps start/stop/restart intents…agentId-only payloads" (AC-2) |
| Reason sanitizer (`createControlReason`, `SECRET_PATTERNS`) | neomjs/neo#14889 row 5 | The GitHub-token pattern is unchanged. A labelled value needs its separator, so a refusal's bare words survive. | `PAT: …` and `token=…` still redact. | the comment at `SECRET_PATTERNS` | "a refusal's bare words survive the redaction, a labelled value does not"; "reason sanitization redacts token-shaped strings" (AC-4) |

## Acceptance Criteria

- [ ] AC-1: A lifecycle intent whose bridge resolves `{status: 'rejected', reason}` ends `rejected`, carrying that reason on the record's `controlReason` (unit).
- [ ] AC-2: A resolved lifecycle record still settles, and a throw or a timeout keeps its path (existing arms pass).
- [ ] AC-3: This lands before, or with, the Brain pin that carries neomjs/neo-agent-brain#751. A pin without it turns a refused start into a silent settle.
- [ ] AC-4: A refusal's bare words survive the redaction ("no GitHub PAT stored" stays whole), while a labelled value (`PAT=…`, `token: …`) is still redacted (unit).

## Out of Scope

- The Brain producer (neomjs/neo-agent-brain#751).
- Card copy and layout: the control-status surface already renders `⚠ rejected: <reason>`.

## Decision Record impact

None.

## Related

#335 (row 2) · #423 (`repoOutcomes`) · neomjs/neo-agent-brain#751 (the producer).

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-10-02T13:16Z. Nothing equivalent.
A2A in-flight sweep: last 30 messages, all read-states, to 13:09Z. No claim.
MC sweep: shared with neomjs/neo-agent-brain#751, which found Emmy's capture `99da0365` and no prior decision.
Own-assignment sweep: #436, #414, #11. None overlapping.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca
Retrieval Hint: `query_raw_memories("lifecycle intent adapter rejected result settled start refusal card control reason")`


## Timeline

- 2026-10-02T13:18:37Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-02T13:18:38Z @neo-opus-grace added the `bug` label
- 2026-10-02T13:18:38Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T13:18:38Z @neo-opus-grace added the `ai` label
- 2026-10-02T13:26:01Z @neo-opus-grace cross-referenced by PR #444
- 2026-10-02T13:30:31Z @neo-opus-grace cross-referenced by PR #753
- 2026-10-02T14:02:41Z @neo-opus-grace cross-referenced by #448

