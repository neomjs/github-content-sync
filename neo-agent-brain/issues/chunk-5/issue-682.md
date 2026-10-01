---
id: 682
title: 'A seat holds more than one repository, and Fleet clones each before launch'
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T13:12:03Z'
updatedAt: '2026-10-01T17:26:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/682'
author: neo-opus-ada
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-01T17:26:46Z'
---
# A seat holds more than one repository, and Fleet clones each before launch

## Context

The operator, 2026-09-30, on the first FM-launched peer (relayed on neomjs/neo-agent-institution#245, comment 5913348117): *"this could be an FM enhancement: picking the repos (multiple ones) that a peer should get clones for."* The follow-up decision (comment 5915627761): Fleet prepares the supported repositories by default before the first launch, with visible progress and a skip option. A peer has one stable primary folder with several working folders attached. On 2026-10-01 the operator set the order: resolve onboarding issues before moving more peers into the Fleet Manager.

Every existing peer works in several repositories (Ada: `neo`, `neo-agent-brain`, `neo-agent-institution`, `neo-agent-skills`, `create-app`). Sophie, the operator's acceptance example, needs three.

## The Problem

A Fleet seat holds exactly one repository. Its definition carries `metadata.repo = {repoSlug, cloneUrl}`, and `startAgentProvisioned` ensures that one checkout, prepares it and launches the harness in it. A peer's other repositories have to be cloned by hand into the seat folder, after the first Start, with a credential the operator supplies. That is the step the Fleet exists to remove. Until it is gone, no existing peer can move into the Fleet Manager without losing access to most of its work.

## The Architectural Reality

- `FleetManager.setRepo({id, repoSlug, cloneUrl})` validates the slug (`assertRepoSlug`) and the clone URL (a remote naming that repo, no credentials, no local source). It then replaces `metadata.repo` wholesale through the registry's partial update.
- The wire verb is registered in `src/fleet/contract/wire.mjs`, in both maps of `ai/services/fleet/fleetServerPolicy.mjs`, and on `FleetControlBridge.setRepo`. Its callers are the Institution's `AddAgentFlow` and `ai/scripts/fleet/onboardPeer.mjs`.
- `startAgentProvisioned` resolves the seat's PAT once, calls `ensureRepo` for `metadata.repo` (clone-or-reuse, never clobber, the PAT authenticating the clone), runs `prepareWorkspace` on that checkout and spawns the harness with `cwd` set to it.
- Checkouts derive to `<agentsRoot>/<agentId>/<owner>/<repo>` (`deriveAgentRepoPath`), so extra repositories already have a home beside the working one.

## The Fix

1. **A second facet, `metadata.repos`:** an ordered list of `{repoSlug, cloneUrl}`, the seat's repositories beside the working one. A new verb `setRepos({id, repos})` validates each entry by `setRepo`'s rules, through one shared helper, and refuses duplicates and the working repository's own slug. `{id, repos: []}` clears the facet. It is registered on the wire like `setRepo`. `metadata.repo` keeps its meaning: the working repository, where the harness launches.
2. **Provisioning:** before launch, `startAgentProvisioned` ensures every entry of `metadata.repos` with `ensureRepo` and the seat's PAT, after the working checkout. A failed entry is recorded, and the launch goes on. A failed working checkout still refuses the start, as today.
3. **The outcome travels:** the start status carries `repos: [{repoSlug, state: 'prepared' | 'failed', reason?}]`, the way it carries `seatInstructions`, so the cockpit can show it.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `metadata.repos` (new definition facet) | this ticket; the operator's 2026-09-30 request | ordered `{repoSlug, cloneUrl}` list beside `metadata.repo` | absent ⇒ a single-repo seat, unchanged | `FleetManager` JSDoc | AC-1 |
| `FleetManager.setRepos` + `FleetControlBridge.setRepos` (new verb) | `setRepo`'s validation rules | sets the facet as a unit; refuses a duplicate or the working slug | `{id, repos: []}` clears it | JSDoc | AC-1, AC-2 |
| `src/fleet/contract/wire.mjs`, `fleetServerPolicy.mjs` (existing) | the wire contract | registers `setRepos` with `setRepo`'s policy | — | — | AC-2 |
| `startAgentProvisioned` (existing) | #571 §3 | ensures each extra checkout with the seat's PAT before launch | a failed extra checkout is reported, never fatal | JSDoc | AC-3, AC-4 |
| start status `repos[]` (new field) | this ticket; `redactReadFailure` for a failed entry's `reason` | per-repository outcome; a failed entry's `reason` is credential-redacted and bounded to 240 chars, since it travels as response data past the dispatcher's sanitizer | absent for a single-repo seat | JSDoc | AC-4 |

## Decision Record impact

`none`: the layout follows #571 §1, and no AiConfig leaf is involved.

## Acceptance Criteria

- [ ] AC-1 (unit): `setRepos` stores the validated list as `metadata.repos`.
  - It refuses a malformed slug, a credential-bearing or local clone URL, a duplicate, or the working repository's slug, each with a message that names the rule, never the value.
  - `{id, repos: []}` clears the facet.
- [ ] AC-2 (unit): `setRepos` is reachable on the wire exactly as `setRepo` is: in `wire.mjs`, in both policy maps and on the bridge.
- [ ] AC-3 (unit): a start ensures every extra checkout at `<agentsRoot>/<agentId>/<owner>/<repo>` with the seat's PAT, after the working checkout and before the spawn.
- [ ] AC-4 (unit): a failed extra checkout appears as `{repoSlug, state: 'failed', reason}` in the status, and the seat still starts. A failed working checkout still refuses the start.
- [ ] AC-5 (unit): a seat without `metadata.repos` starts exactly as before, with no `repos` field on its status.

## Out of Scope

- The Institution's form for the repository set, and its default from the Knowledge Base's tenant repositories: neomjs/neo-agent-institution#245 consumes this verb.
- Progress events while the clones run, and the skip option: the UI half of the operator's decision, after this lands.
- Each harness family's write access to the extra checkouts. It is not yet measured per family, and Codex seats run sandboxed, so it gets measured before anything is filed.
- Dependency installation in the extra checkouts.
- Removing a repository from the set: its checkout stays on disk (#142 owns reconciliation).

## Avoided Traps

- **Growing `setRepo`'s payload:** today's callers (`AddAgentFlow`, `onboardPeer`) set the working repository as a unit, so a wider payload would let them clear a seat's other repositories silently.
- **Launching in the seat folder instead of a checkout:** the projected seat hooks resolve `git rev-parse --show-toplevel`, and Claude memory follows the checkout unless pinned (#675).

## Related

#571 (parent) · #675 (one memory per seat, which this needs) · #142 · neomjs/neo-agent-institution#245 (the form) · neomjs/neo-agent-institution#351 (the wizard)

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-01T13:12Z, no equivalent; an org-wide search for "repository set seat clones" finds only Institution #245 and #571. A2A in-flight sweep (all read states, last 60 min): no claim on multi-repository seats. Memory Core sweep ("peer seat clones several repositories"): the move runbooks of 09-29/09-30; no decision against it. Own-assignment sweep: #571 (parent), #142 (adjacent), Institution #245 (the consumer). Structure map: owning folder `ai/services/fleet`, no new file.

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a
Retrieval Hint: `query_raw_memories("Fleet seat several repositories setRepos metadata.repos clone each before launch")`


## Timeline

- 2026-10-01T13:12:03Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T13:12:05Z @neo-opus-ada added the `enhancement` label
- 2026-10-01T13:12:06Z @neo-opus-ada added the `ai` label
- 2026-10-01T13:12:06Z @neo-opus-ada added the `architecture` label
- 2026-10-01T13:12:06Z @neo-opus-ada added the `agent-os` label
- 2026-10-01T13:12:10Z @neo-opus-ada added parent issue #571
- 2026-10-01T13:20:45Z @neo-opus-ada cross-referenced by PR #683
- 2026-10-01T13:28:14Z @neo-opus-grace cross-referenced by #684
- 2026-10-01T13:31:25Z @neo-fable-clio cross-referenced by #685
- 2026-10-01T16:29:01Z @neo-opus-grace cross-referenced by #704
- 2026-10-01T17:15:24Z @neo-opus-ada referenced in commit `c538200` - "fix(fleet): a failed extra repository's reason is credential-redacted and bounded (#682)"
- 2026-10-01T17:26:46Z @tobiu referenced in commit `31c7073` - "feat(fleet): a seat holds more than one repository, and Fleet clones each before launch (#682) (#683)

* feat(fleet): a seat holds more than one repository, and Fleet clones each before launch (#682)

A seat's other repositories live in metadata.repos ([{repoSlug, cloneUrl}],
beside the working metadata.repo), set through a new setRepos wire verb.
setRepos validates each entry by setRepo's rule (now one shared helper) and
refuses a duplicate, the working repository and a seat without one; [] clears
it. startAgentProvisioned clones each after the working checkout, with the
seat's PAT, before the spawn. A failed extra clone is reported on the status
(repos: [{repoSlug, state, reason?}]) and the launch goes on. A failed working
checkout still refuses the start.

* fix(fleet): a failed extra repository's reason is credential-redacted and bounded (#682)"
- 2026-10-01T17:26:47Z @tobiu closed this issue
- 2026-10-01T17:52:15Z @neo-opus-grace cross-referenced by #710
- 2026-10-01T18:01:10Z @neo-opus-ada cross-referenced by #402
- 2026-10-01T18:41:35Z @neo-opus-ada cross-referenced by #407

