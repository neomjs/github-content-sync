---
id: 751
title: A Fleet start refusal crosses the wire as a generic failure
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-02T13:18:15Z'
updatedAt: '2026-10-02T16:14:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/751'
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
closedAt: '2026-10-02T16:14:49Z'
---
# A Fleet start refusal crosses the wire as a generic failure

## Context

FM v1 row 2's script (neomjs/neo-agent-institution#335, step 4) expects a failed start to name its reason and its next step. @neo-gpt-emmy's installed witness on 2026-09-30 is defect-note `99da0365`: *"FM Start failure feedback is wrong — the roster reports only fleet startAgent failed while the shell log identifies the missing runtime template"*. She asked for the actionable, non-secret detail at the product boundary, with no broad raw-error propagation. The pre-sitting trace ([#335 issuecomment-5952885068](https://github.com/neomjs/neo-agent-institution/issues/335#issuecomment-5952885068)) shows why on `dev@a9dd22f`. @neo-fable-clio routed the leaf at 13:02Z, which promotes that note.

## The Problem

- Every refusal on the start path is thrown, and `dispatchFleetRequest` turns any throw into `operation-failed` with `"fleet: 'startAgent' failed"`. That is deliberate: "Never expose the raw error across the wire".
- So the operator sees the same words for a missing PAT, a released seat, a missing plane credential and a missing harness binary.
- The Fleet already words each of these for the operator, for example `startAgentProvisioned.mjs:239`: *"agent 'a' has no GitHub PAT stored; store one before starting it."*

## The Architectural Reality

- `src/fleet/contract/wire.mjs`: "Domain outcomes such as an admission rejection remain inside result; these states describe only the wire/dispatch layer."
- `FleetControlBridge.mjs:19-33` already implements that rule for definition updates. `rejectionOf(error, caller)` turns a `<caller>: <rule>` throw into `{status: 'rejected', reason}`, and `setRepo`, `setRepos` and `defineAgent` answer their refusals as data that way (#724 renders them in Accounts).
- `startAgent` and `restartAgent` (`:448`, `:467`) pass straight to the manager, so their refusals stay throws.
- The start path's named refusals carry four prefixes:
  - `FleetManager.startAgent:` / `FleetManager.restartAgent:` (`assertStartPermitted`, and restart's cleanup refusal);
  - `startAgentProvisioned:`;
  - `FleetLifecycleService.start:`.
- They are authored text: agent ids, the operator's own plane endpoint, the configured harness binary, a placement reason, and `FleetTenantService`'s closed readiness vocabulary. None reads a secret: the code says *"Structural refusals first: none of them may read a secret"* (`startAgentProvisioned.mjs:178`).
- Emmy's own case was not a worded refusal. It was a `ManagedWorkspacePreparationError` (`prepareManagedAgentWorkspace.mjs:53`), whose `code` is a closed set (`FLEET_WORKSPACE_PREPARATION_FAILED`, `FLEET_WORKSPACE_DIVERGENT`, `FLEET_WORKSPACE_UNSUPPORTED`) and whose message can carry local paths and a wrapped cause. The code is non-secret; the message is not for the wire.
- A successful start answers the lifecycle record (`id`, `state`, `pid`, …, plus `wakeRoute`), which has no `status` key, so `{status: 'rejected'}` is unambiguous.

## The Fix

1. `startAgent` and `restartAgent` answer a refusal named by any of the start path's callers as `{status: 'rejected', reason}`, using the same `rejectionOf` that serves definition updates, widened to a caller list.
2. A `ManagedWorkspacePreparationError` answers `{status: 'rejected', reason: "the seat's workspace could not be prepared (<code>); the Fleet log names the artifact."}`, built from its code and never from its message.
3. Any other throw rethrows and stays `"fleet: '<method>' failed"`.
4. Arms run through the real `dispatchFleetRequest` → `FleetControlBridge` chain with a manager stub:
   - a refused start answers `ok` with `result: {status: 'rejected', reason: "agent 'a' has no GitHub PAT stored; store one before starting it."}`;
   - a preparation failure whose message carries a path answers its code sentence, with no path;
   - an unexpected throw still answers `operation-failed` with the generic text;
   - no stack and no unprefixed message ever crosses.

The Institution consumer (the lifecycle adapter reads any resolved result as settled) is its own leaf in neomjs/neo-agent-institution, linked below.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetControlBridge.startAgent(id)` / `restartAgent(id)` result | `FleetControlBridge` over `FleetManager` | The lifecycle record on success. `{status: 'rejected', reason}` for a refusal its start path names. | Any other throw rethrows (`operation-failed`, generic text). | method JSDoc | unit (dispatch chain) |
| `rejectionOf` | `FleetControlBridge` | Matches one caller or a list. | An unprefixed message gives `null` (rethrow). | function JSDoc | unit |
| A `ManagedWorkspacePreparationError` on the start path | `prepareManagedAgentWorkspace.mjs` (its closed `code` set) | `{status: 'rejected', reason: "the seat's workspace could not be prepared (<code>); the Fleet log names the artifact."}`, built from the code only. The error is matched by `name` plus one of the three codes its producer raises (`WORKSPACE_PREPARATION_CODES`, pinned to the producer by a spec). Importing the class into the bridge would break `prepareManagedAgentWorkspace.spec`'s one-production-caller guard. | A code outside that list rethrows (generic). | `startOutcome` JSDoc | unit: a message carrying a local path never crosses (AC-2); an unknown code stays generic (AC-3) |
| The wire envelope (`dispatchFleetRequest`) | `src/fleet/contract/wire.mjs`: "Domain outcomes … remain inside result" | A rejection rides the `ok` envelope as its `result`. The response-state vocabulary is unchanged, with no new state. | An unnamed throw answers `operation-failed` with `"fleet: '<method>' failed"`, carrying no message and no stack. | `wire.mjs` | unit: AC-1 and AC-3 through the real chain |

## Acceptance Criteria

- [ ] AC-1: A start or restart refused by any of the four named callers answers `{status: 'rejected', reason}` through `dispatchFleetRequest`, with the reason being the caller's text after its prefix (unit).
- [ ] AC-2: A `ManagedWorkspacePreparationError` answers its code sentence and never its message: a path in the message never crosses (unit).
- [ ] AC-3: An unexpected throw on the same path still answers `operation-failed` with `"fleet: '<method>' failed"`, and no stack or unprefixed message crosses (unit).
- [ ] AC-4: `setRepo`, `setRepos` and `defineAgent` keep their behavior (existing arms unchanged).

## Out of Scope

- `stopAgent` and `removeAgent` refusals. The start path is the row-2 sitting's step.
- The Institution adapter (its own leaf).
- Any new wire state.

## Avoided Traps

- **A typed error mapped to `operation-failed` text.** It would need no consumer change, but it puts a domain refusal on the wire layer the contract reserves for dispatch, and the bridge tags `operation-failed` as `failed-upstream`, a connection word, for what is a refusal.
- **Forwarding `error.message` for any throw.** That is exactly what the dispatch guard exists to prevent. Only a caller-named refusal crosses.

## Decision Record impact

None: `aligned-with` the wire contract's own domain-outcome rule.

## Related

neomjs/neo-agent-institution#335 (row 2) · neomjs/neo-agent-institution#423 (`repoOutcomes`, the redacted-reason neighbor) · #724 (definition refusals in Accounts) · the Institution consumer leaf (filed with this one).

Live latest-open sweep: checked the latest 20 open Brain and Institution issues at 2026-10-02T13:16Z. Nothing equivalent; Institution #442 is Emmy's pin with MCP controls.
A2A in-flight sweep: last 30 messages, all read-states, to 13:09Z. No claim on start refusals.
MC sweep: `fleet startAgent failed generic reason card start refusal no GitHub PAT stored operator never sees cause`, 6 results. It found Emmy's capture `99da0365` (09-30, not promoted; promoted by this ticket) and no prior decision against it.
Own-assignment sweep: open Brain #684, #517 and older; Institution #436, #414, #11. None overlapping.
Exact: `gh search issues --owner neomjs "startAgent rejected reason"`, 0 results.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca
Retrieval Hint: `query_raw_memories("start refusal fleet startAgent failed rejectionOf domain outcome dispatch")`


## Timeline

- 2026-10-02T13:18:16Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-02T13:18:16Z @neo-opus-grace added the `bug` label
- 2026-10-02T13:18:16Z @neo-opus-grace added the `ai` label
- 2026-10-02T13:18:37Z @neo-opus-grace cross-referenced by #443
- 2026-10-02T13:26:01Z @neo-opus-grace cross-referenced by PR #444
- 2026-10-02T13:30:31Z @neo-opus-grace cross-referenced by PR #753
- 2026-10-02T14:06:16Z @neo-opus-grace cross-referenced by #755
- 2026-10-02T14:14:27Z @neo-opus-grace referenced in commit `b24efe5` - "fix(fleet): a start refusal carries a lease failure's system code and only the declared workspace codes (#751)

The lease writer keeps a caught failure's Node system code and otherwise answers bounded words, logging the cause in the Fleet; the bridge answers only the preparation codes its producer raises, pinned to the producer by a spec."
- 2026-10-02T14:36:49Z @neo-opus-grace cross-referenced by #760
- 2026-10-02T16:14:49Z @tobiu referenced in commit `5a5ebad` - "feat(fleet): a start the Fleet refuses answers its reason as data, not a generic failure (#751) (#753)

* feat(fleet): a start the Fleet refuses answers its reason as data, not a generic failure (#751)

Every refusal on the start path was a throw, which the dispatcher turns into
"fleet: 'startAgent' failed", so a missing PAT, a released seat and a missing
harness binary all read the same on the card. startAgent and restartAgent
now answer the refusals their callers word (FleetManager's start gate,
startAgentProvisioned, FleetLifecycleService.start) as {status: 'rejected',
reason}, the domain outcome rejectionOf already gives definition updates. A
workspace that could not be prepared answers its code, never its message.
Anything else still reaches the dispatcher's generic failure.

* fix(fleet): a start refusal carries a lease failure's system code and only the declared workspace codes (#751)

The lease writer keeps a caught failure's Node system code and otherwise answers bounded words, logging the cause in the Fleet; the bridge answers only the preparation codes its producer raises, pinned to the producer by a spec."
- 2026-10-02T16:14:49Z @tobiu closed this issue
- 2026-10-02T16:17:37Z @tobiu referenced in commit `698c4d4` - "feat(agentos): a lifecycle refusal the Fleet answers as data reads as rejected (#443) (#444)

* feat(agentos): a lifecycle refusal the Fleet answers as data reads as rejected (#443)

The adapter treated any resolved bridge answer as settled, so a refusal the
Fleet answers as data ({status: 'rejected', reason}, the bridge's domain
outcome that neomjs/neo-agent-brain#751 extends to starts) would clear the
card's control line silently. Such an answer now ends rejected with its
reason. The labelled-value redaction now requires its separator, so the
bare words of a refusal ("no GitHub PAT stored") survive while `PAT: …` and
`token=…` values stay redacted.

* test(visual): refresh the baseline input stamp for the lifecycle adapter (#443)"
- 2026-10-02T17:12:43Z @neo-opus-ada cross-referenced by #772
- 2026-10-02T17:18:30Z @neo-opus-ada cross-referenced by PR #774

