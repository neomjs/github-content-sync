---
id: 721
title: A setRepo or setRepos refusal crosses the Fleet wire without its reason
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T18:54:47Z'
updatedAt: '2026-10-01T20:03:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/721'
author: neo-opus-ada
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
closedAt: '2026-10-01T20:03:44Z'
---
# A setRepo or setRepos refusal crosses the Fleet wire without its reason

## Context

neomjs/neo-agent-institution#407 gives the Accounts card a repo-set picker. Its AC-2 shows the Brain's reason when `setRepos` refuses a list (an invalid slug, a duplicate, the working repository). That reason never reaches the card. The add form's working repository (`setRepo`) has the same gap.

*Revised 2026-10-01 ~19:10Z:* `setRepo` is folded in. Grace and Vega both pointed out that it is the same file and the same fix, and that #711 adds four more `setRepo` refusals (forge, GitLab clone URL, slug depth, checkout collision) whose reasons would die on the wire too.

## The Problem

`setRepo` and `setRepos` (#683) refuse by throwing, so every refusal reaches the wire as a dispatch failure. Measured at `dev@5041af0` with the real `FleetManager` behind the real `FleetControlBridge` and `dispatchFleetRequest`:
- `setRepos` given a duplicate, the working repository or a malformed slug;
- `setRepo` given a malformed slug or a credentialed clone URL.

All five answered:

```
{ok: false, state: 'operation-failed', error: "fleet: '<verb>' failed"}
```

The same calls with valid input answered `state: 'ok'`. The manager's own messages (for example "a repository is listed twice.") name the rule and never the value, but they stop at the dispatcher's log line.

The wire contract says where they belong: domain outcomes such as a rejection "remain inside result", and the response states "describe only the wire/dispatch layer" (`src/fleet/contract/wire.mjs:48-50`).

## The Architectural Reality

- `FleetManager.setRepo` (`ai/services/fleet/FleetManager.mjs:443`) and `setRepos` (`:468`) validate and throw with their caller prefix. Each returns the updated definition, or `null` for an unknown agent.
- `FleetControlBridge.setRepo` / `setRepos` delegate without a catch.
- `dispatchFleetRequest` (`ai/services/fleet/dispatchFleetRequest.mjs:50-58`) logs any throw and answers `operation-failed` with a generic error, by design.
- **Precedent:** `FleetControlBridge.configureAgent` (`:325`) already does this. It turns a `FleetRegistryService.configureAgent:` refusal into `{status: 'rejected', reason}` and success into `{status: 'accepted', agent}`, and rethrows anything else. Its JSDoc: "Validation failures become an explicit domain outcome the Accounts card may render".
- **Consumers of `setRepo`'s answer:**
  - the Brain's `ai/scripts/fleet/onboardPeer.mjs`, whose commit awaits `setRepo` over the wire and relies on the throw to stop;
  - the Institution's `AddAgentFlow`, which reads the bare definition.
- Structure map: `ai/services/fleet` owns both the bridge and the manager. No new file.

## The Fix

Both bridge verbs adopt `configureAgent`'s outcome shape through one helper:
- an updated definition → `{status: 'accepted', agent}`;
- `null` → `{status: 'rejected', reason: "Unknown agent '<id>'."}`;
- a throw prefixed with the manager verb → `{status: 'rejected', reason}` with the prefix cut;
- anything else rethrows, so the dispatcher still sanitizes it.

`FleetManager` keeps throwing; its Node-side contract and its spec stay as they are. `onboardPeer`'s repo commit stops on a rejection with the Fleet's reason, as it stopped on the throw before.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| wire verbs `setRepo`, `setRepos` (`FleetControlBridge`) | `wire.mjs:48-50`; sibling `configureAgent` (`FleetControlBridge.mjs:325`) | `{status: 'accepted', agent}` or `{status: 'rejected', reason}` | a failure that is not a manager refusal still throws → `operation-failed` | the verbs' JSDoc | `FleetControlBridge.spec.mjs` arms, each refusal also through `dispatchFleetRequest` |
| `onboardPeer` commit (CLI) | this ticket | a refused working repository stops the commit with the Fleet's reason | none: it never launches past a refusal | `commitRepoSegment` JSDoc | `onboardPeer.spec.mjs` repo-commit arm |

neomjs/neo-agent-institution#407 owns the Institution's adoption (its AC-5): the pin that carries this, and `AddAgentFlow.assignRepo` reading `setRepo`'s new answer. Until then the Institution pins the Brain by SHA, so nothing changes there.

## Decision Record impact

None. The change aligns two verbs with the wire contract's response-state rule.

## Acceptance Criteria

- [ ] AC-1: `FleetControlBridge.setRepo` and `setRepos` answer `{status: 'accepted', agent}` with the updated definition. They answer `{status: 'rejected', reason}` for an unknown agent and for every refusal their manager verb names: for `setRepo`, a malformed slug or a clone URL that is not a remote naming it; for `setRepos`, also not an array, a duplicate, the working repository and a seat without one. The reason is the manager's message without the caller prefix.
- [ ] AC-2: through `dispatchFleetRequest`, those refusals arrive as `state: 'ok'` with the rejection inside `result`. A throw that is not the verb's refusal still arrives as `operation-failed`.
- [ ] AC-3: a credentialed clone URL's secret appears in no reason (the manager's rule, asserted at the wire).
- [ ] AC-4: `onboardPeer`'s commit stops on a refused working repository, with the Fleet's reason, instead of launching the seat.

## Out of Scope

- A dispatcher-wide typed refusal. That would change every verb's result shape at once. The per-verb opt-in is the established pattern (`configureAgent`, `connectTenant`).
- The Institution side: neomjs/neo-agent-institution#407 owns it, covering the pin, `AddAgentFlow`'s adaptation (AC-5) and the repos picker.

## Related

#683 / #682 (`setRepo` / `setRepos`) · #711 (more `setRepo` refusals) · neomjs/neo-agent-institution#407 (consumer) · neomjs/neo-agent-institution#403 (pin 7, which predates this)

## Sweeps

- Live latest-open sweep: the latest 20 open Brain issues at 2026-10-01T18:54:16Z. No equivalent; the nearest are #710 / #712 (the forge model).
- Keyword: `setRepos` across neomjs (#682 closed, #684, #710, Institution #402 / #407) and "operation-failed reason". No equivalent.
- A2A: the last 30 messages, all read-states. No claim on this verb.
- MC sweep: one query on the problem's nouns (a `setRepos` refusal's reason lost on the Fleet wire), 6 results, no prior decision found.
- Own-assignment sweep: 20 open, none overlapping. #571 and #142 touch the Fleet, but not its wire verbs.
- Agent OS structure map: `ai/services/fleet`.

Retrieval Hint: "setRepos refusal reason operation-failed wire"

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a

Authored by Ada (Claude Opus 5.5, Claude Code).


## Timeline

- 2026-10-01T18:54:48Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T18:54:49Z @neo-opus-ada added the `bug` label
- 2026-10-01T18:54:49Z @neo-opus-ada added the `ai` label
- 2026-10-01T18:54:50Z @neo-opus-ada added the `agent-os` label
- 2026-10-01T18:56:06Z @neo-opus-ada cross-referenced by #407
- 2026-10-01T19:03:11Z @neo-opus-ada cross-referenced by PR #724
- 2026-10-01T19:14:38Z @neo-opus-ada changed title from **A setRepos refusal crosses the Fleet wire without its reason** to **A setRepo or setRepos refusal crosses the Fleet wire without its reason**
- 2026-10-01T19:16:58Z @neo-opus-ada referenced in commit `6ebb191` - "fix(fleet): setRepo answers a refusal as data too, and onboarding stops on one (#721)

setRepo joins setRepos on the shared definitionOutcome helper, so a malformed slug or a credentialed clone URL reaches its caller as {status: 'rejected', reason} instead of a bare operation-failed. onboardPeer's commit used to stop on the thrown refusal; commitRepoSegment now stops on the rejection, with the Fleet's reason, instead of launching past it. The Institution's add form reads setRepo's bare definition and adapts in the pin that carries this."
- 2026-10-01T20:03:44Z @tobiu referenced in commit `887f8f5` - "fix(fleet): setRepo and setRepos answer a refusal with its reason inside the wire result (#721) (#724)

* fix(fleet): a setRepos refusal answers its reason inside the wire result (#721)

FleetControlBridge.setRepos adopts configureAgent's outcome shape: {status: 'accepted', agent} or {status: 'rejected', reason}. A refusal the manager names no longer reaches the dispatcher, which sanitized it into a bare operation-failed, so the Accounts card could not show why a repository list was refused. Any other failure still throws. configureAgent's inline prefix check moves into the shared rejectionOf helper.

* fix(fleet): setRepo answers a refusal as data too, and onboarding stops on one (#721)

setRepo joins setRepos on the shared definitionOutcome helper, so a malformed slug or a credentialed clone URL reaches its caller as {status: 'rejected', reason} instead of a bare operation-failed. onboardPeer's commit used to stop on the thrown refusal; commitRepoSegment now stops on the rejection, with the Fleet's reason, instead of launching past it. The Institution's add form reads setRepo's bare definition and adapts in the pin that carries this."
- 2026-10-01T20:03:45Z @tobiu closed this issue

