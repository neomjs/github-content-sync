---
id: 243
title: Seat credential routing lives in an untracked dotfile and fails silently — agents authored as the operator
state: CLOSED
labels: []
assignees: []
createdAt: '2026-08-29T22:29:34Z'
updatedAt: '2026-08-29T22:32:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/243'
author: tobiu
commentsCount: 1
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
closedAt: '2026-08-29T22:32:51Z'
---
# Seat credential routing lives in an untracked dotfile and fails silently — agents authored as the operator

Seat credentials are routed by a `case` block in an **untracked, untested shell dotfile** on the host.
It went stale, and the failure mode was silent misattribution: agents authored GitHub artifacts as the
operator for an unknown period, with nothing anywhere reporting an error.

## What happened

The mapping matched `<seat-root>/neomjs/neo/*` — the engine repo only. Every seat root now holds
several org repos:

| seat root | repos present |
|---|---|
| one Claude seat | `neo`, `neo-agent-brain`, `devindex` |
| another | `neo`, `neo-agent-brain`, `neo-agent-institution` |
| another | `neo`, `neo-agent-brain`, a rewrite checkout |

Only `neo` was mapped. Working in any sibling dropped `GH_TOKEN`, and `gh` **does not fail** without
it — it silently falls back to the keyring account, which is the operator's. Git commits stayed
correctly attributed (git author is configured separately), so nothing looked wrong locally. The only
visible symptom was the author on the finished artifact.

Observed by the operator as agents "creating PRs using the global account more and more often" across
at least three seats. The rising frequency was not drifting discipline — it tracked work migrating
**into** `neo-agent-brain` as the Agent OS moved there, i.e. out of the single mapped path.

Three artifacts I created in one session (an issue, a PR, an engine issue) carry the wrong author.
GitHub authorship is immutable, so those cannot be corrected.

## Already fixed on the host

The mapping arms were widened from `<seat-root>/neomjs/neo/*` to `<seat-root>/neomjs/*`, so every
current and future org repo under a seat root resolves. The env file stays pinned per seat, so no seat
can resolve to another's. A `gh` guard was added that refuses artifact-creating verbs when a seat tree
has no token, printing the recovery command — reads and `gh auth` stay available for diagnosis.

Verified: 25 sandbox assertions including negatives and cross-seat isolation, plus live checks that
the previously-broken paths now resolve to the correct seat identity and that an unmapped path still
resolves to nothing.

## Why this still needs a ticket

**The fix is correct and the arrangement is not.** Credential routing for the whole swarm lives in a
file that is untracked, unreviewable, unversioned, has no test, and fails silently. Every property
that made this bug expensive is still true of the next one:

- **It will go stale again.** The list is enumerated by hand; a new seat, a repo cloned outside
  `neomjs/`, or a renamed root breaks it, and the guard added above only narrows the blast radius —
  it does not keep the list current.
- **No review.** A change to how every agent authenticates gets no diff, no second reader.
- **No test in CI**, though the mapping is pure and trivially testable — the sandbox test written for
  this fix is 25 assertions of plain string matching against a `case` block.
- **Silent by default.** Absent credentials producing a *working* command with the wrong identity is
  the worst available failure mode, and it is `gh`'s default.

## Acceptance Criteria

- [ ] The seat mapping has a repo-tracked source of truth, alongside the existing host setup in
      `ai/scripts/lifecycle/local-agent-os/README.md`, rather than existing only in a host dotfile.
- [ ] A test covers the mapping — every seat resolves to its own env, no seat resolves to another's,
      and unmapped paths resolve to nothing. Negative and cross-seat arms are required; a
      positive-only test passes for a mapping that matches everything.
- [ ] The test runs in a job that actually executes it. (Note: `brain-unit.yml` currently runs
      `--list` plus a three-spec smoke, so adding a spec is not by itself sufficient.)
- [ ] Adding a seat or a repo root is a reviewed change to the tracked source, not a hand edit on the
      host.
- [ ] A seat tree without credentials fails loudly for anything that would create an artifact, rather
      than succeeding under the wrong identity. (Implemented on the host; belongs in the tracked
      source.)

## Out of Scope

- The `.env` files themselves and their contents. Only the routing that selects one is in scope.
- Retroactive attribution. GitHub authorship cannot be changed after the fact.


## Timeline

### @neo-opus-grace - 2026-08-29T22:32:50Z

Superseded by #244 — same content, filed correctly.

This issue is itself an instance of the bug it describes: I created it from a scratch directory minutes after fixing the path mapping, and the first version of the guard was keyed on the working directory, so it stayed silent outside a seat tree. Author here is the operator; author on #244 is the seat. Closing this rather than editing, because the misattribution is not editable and #244 carries the corrected guard rationale.

- 2026-08-29T22:32:51Z @neo-opus-grace closed this issue

