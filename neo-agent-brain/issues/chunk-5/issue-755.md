---
id: 755
title: A GitLab seat's repositories default to its own instance
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-02T14:06:15Z'
updatedAt: '2026-10-02T16:15:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/755'
author: neo-opus-grace
commentsCount: 0
parentIssue: 684
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 448 The add-agent form defines a GitLab seat on its own instance'
closedAt: '2026-10-02T16:15:12Z'
---
# A GitLab seat's repositories default to its own instance

## Context

neomjs/neo-agent-institution#448 lets the cockpit define a GitLab seat and its repositories. Its intake checked whether the fix belongs where the ticket put it (Stage 2: the prescription). The answer was no: composing a GitLab seat's clone URL belongs in the Brain's seat-aware verbs, not in the Body.

## The Problem

- `repoCoordinates` (`FleetManager.mjs`) requires a GitLab entry to name its clone URL, *"because a self-hosted host cannot be derived"*. That was true when #710 landed (PR #711, 2026-10-01), because a seat recorded no instance then.
- Since #729 (PR #749, `a9dd22f`), `setRepo` checks the working repository against the seat's own `forgeHost` (`assertOnSeatForge`). Inside `setRepo` and `setRepos` the host is therefore known.
- Yet `setRepo({id, repoSlug})` on a GitLab seat still composes `https://github.com/<slug>.git`, and `assertOnSeatForge` then refuses it.
- So every client must compose `<forgeHost>/<slug>.git` and pass `forge: 'gitlab'` itself. The cockpit would have to duplicate that composition, as it already duplicates GitHub's in `AddAgentFlow.repoOf`.

## The Architectural Reality

- `repoCoordinates({repoSlug, cloneUrl, forge='github'}, caller)` takes no seat.
  - GitHub defaults to `https://github.com/<slug>.git`.
  - GitLab requires the clone URL, and the entry records `forge: 'gitlab'`.
- `setRepo({id, cloneUrl, forge, repoSlug})` reads the seat, runs `repoCoordinates`, then `assertOnSeatForge(repo, seat)`.
- `setRepos({id, repos})` maps each entry through `repoCoordinates`.
- A GitLab seat's public definition holds `forge: 'gitlab'` and `forgeHost`, its instance's `https` origin, from `FleetRegistryService` `forgeAccount`. A GitHub seat holds neither.

## The Fix

1. In `setRepo` and `setRepos`, an entry with no `cloneUrl` takes the seat's forge when it names none.
2. When that forge is the seat's GitLab forge, the clone URL is `<forgeHost>/<repoSlug>.git`.
3. This is a private helper beside `repoCoordinates`, applied before it. An entry that names its `cloneUrl` is unchanged.
4. `repoCoordinates` itself stays seat-free, and its GitLab rule keeps holding for a bare call.
5. Both verbs' JSDoc names the default.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetManager.setRepo` / `setRepos` entries | the seat's public `forge` / `forgeHost` (`forgeAccount`) | On a GitLab seat, an entry without `cloneUrl` becomes `{repoSlug, cloneUrl: <forgeHost>/<repoSlug>.git, forge: 'gitlab'}`. | A GitHub seat, and any entry naming its `cloneUrl`, behave as today. | verb JSDoc | unit |

## Acceptance Criteria

- [ ] AC-1: On a GitLab seat, `setRepo({id, repoSlug: 'group/sub/project'})` records `cloneUrl: '<forgeHost>/group/sub/project.git'` and `forge: 'gitlab'`, and passes `assertOnSeatForge` (unit).
- [ ] AC-2: On a GitLab seat, `setRepos` entries given as `{repoSlug}` get the same composition (unit).
- [ ] AC-3: A GitHub seat, and an entry naming its `cloneUrl`, behave as today (existing arms, plus one explicit-entry arm).
- [ ] AC-4: A bare `repoCoordinates` call still refuses a GitLab entry without a clone URL (existing arm).

## Out of Scope

- Whether `setRepos` should assert each extra repository is on the seat's forge. It does not today.
- The Institution consumer, neomjs/neo-agent-institution#448.

## Decision Record impact

None.

## Related

#684 (parent) · #710 (the rule's origin) · #729 (`assertOnSeatForge`) · neomjs/neo-agent-institution#448 (the consumer, which sends `{repoSlug}` once a pin carries this).

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-02T14:05Z. No equivalent.
A2A in-flight sweep: last 30 messages, all read-states, to 14:03Z. No claim.
MC sweep: `GitLab repository clone URL cannot be derived self-hosted host repoCoordinates setRepo seat forgeHost default nested groups`, 5 results, all noise. No prior decision. The rule is mine (`425667d`) and predates the seat's `forgeHost` check (`a9dd22f`).
Own-assignment sweep: #751, #684 (parent; none of its closed leaves covers this), #517 and older. None overlapping.
Exact: `gh search issues --owner neomjs "setRepo GitLab clone URL forgeHost"`, 0 results.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca
Retrieval Hint: `query_raw_memories("GitLab seat repository clone URL default forgeHost setRepo setRepos seat-aware")`


## Timeline

- 2026-10-02T14:06:16Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-02T14:06:16Z @neo-opus-grace added the `enhancement` label
- 2026-10-02T14:06:16Z @neo-opus-grace added the `ai` label
- 2026-10-02T14:06:17Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T14:06:21Z @neo-opus-grace added parent issue #684
- 2026-10-02T14:19:59Z @neo-opus-grace cross-referenced by #448
- 2026-10-02T14:20:02Z @neo-opus-grace marked this issue as blocking #448
- 2026-10-02T14:22:49Z @neo-opus-grace cross-referenced by PR #756
- 2026-10-02T14:36:49Z @neo-opus-grace cross-referenced by #760
- 2026-10-02T14:53:50Z @neo-opus-grace cross-referenced by PR #450
- 2026-10-02T16:15:12Z @tobiu referenced in commit `bb10149` - "feat(fleet): a GitLab seat's repositories default to its own instance (#755) (#756)

setRepo and setRepos read the seat before checking an entry: one without a clone URL takes the seat's forge, and on a GitLab seat its clone URL is the slug on the seat's forgeHost, so no client composes it."
- 2026-10-02T16:15:13Z @tobiu closed this issue
- 2026-10-02T17:30:46Z @neo-opus-grace cross-referenced by #684

