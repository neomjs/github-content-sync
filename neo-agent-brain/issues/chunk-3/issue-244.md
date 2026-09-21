---
id: 244
title: Seat credential routing lives in an untracked dotfile and fails silently — agents authored as the operator
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-08-29T22:32:39Z'
updatedAt: '2026-09-06T12:00:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/244'
author: neo-opus-grace
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
closedAt: '2026-09-06T12:00:18Z'
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

**The guard needed a second pass, and the reason generalizes.** It was first keyed on the working
directory — refuse when a *seat tree* has no token. That missed the most common case immediately:
agents `cd` into a scratch directory to pass `--body-file`, which is outside every seat tree, so a
cwd-scoped guard stayed silent for precisely the command that creates the artifact. The first ticket
filed about this bug was itself misattributed that way, from the scratch directory, minutes after the
mapping was fixed. The guard is now keyed on the **agent shell** (`CLAUDECODE` / `AI_AGENT`), which is
the property that actually matters and which no directory can move you out of; the operator's own
terminal is unaffected because those variables are absent there.

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

- 2026-08-29T22:32:51Z @neo-opus-grace cross-referenced by #243
- 2026-08-29T22:32:53Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-08-29T22:51:06Z @neo-gpt cross-referenced by PR #242
- 2026-09-06T10:05:09Z @neo-opus-grace cross-referenced by #335
- 2026-09-06T10:40:04Z @neo-opus-grace added the `bug` label
- 2026-09-06T10:40:04Z @neo-opus-grace added the `ai` label
- 2026-09-06T10:40:05Z @neo-opus-grace added the `agent-os` label
- 2026-09-06T10:48:55Z @neo-opus-grace cross-referenced by PR #339
- 2026-09-06T10:53:12Z @neo-opus-grace referenced in commit `b0cfc2d` - "fix(agent-os): the zsh arms resolve zsh through PATH, and CI installs it (#244)

`/bin/zsh` is macOS's path. On the Linux runner it is /usr/bin/zsh, so all five
shell arms died with `spawnSync /bin/zsh ENOENT` — an error about my hardcoded
path, saying nothing about the routing under test. Resolved through PATH now.

ubuntu-latest ships no zsh at all, so the workflow installs it. The alternative
was skipping these arms where zsh is absent, and that was rejected: they are the
only arms that execute the artifact this module actually ships. Skipping them
would leave every other arm green while the emitted `case` globs went unverified
outside a maintainer's Mac, which is the precise gap they exist to close. The
prerequisite is asserted as a test instead, so a missing shell reds with a
sentence that names the cause.

Found by CI, not locally: the arms were written and validated on macOS only, so
the environment axis was never exercised until a Linux runner did it."
### @neo-opus-grace - 2026-09-06T12:00:17Z

Closing NOT_PLANNED. The reported defect was real and is **already fixed on the host**; what remains in this ticket is an AC set pointing the wrong way, and it should not sit in the backlog inviting someone to build it.

## What was real, and is done

Seat credential routing went stale, `gh` fell back to the keyring account, and agents authored GitHub artifacts as the operator. That happened, it cost three permanently misattributed artifacts, and the host repair described in this body — widened mapping arms plus a `gh` guard — landed at the time. **Nothing operational regresses by closing.**

## Why the remaining ACs are wrong, not just unfinished

AC-1 asked for "a repo-tracked source of truth" for the seat mapping. PR #339 implemented exactly that and it was rejected at a level above the implementation. Operator ruling:

> *"OUR fixed team setup. klarso using gitlab PAT access that auto-creates identities inside the graph. fleet manager is meant for OTHER operators and THEIR agent teams. ROI -1000. LOCKING our team in."*

and, widening it:

> *"not just fleet manager. agent os => dockerized => KB and MC for OTHER teams, ingesting OTHER repos."*

**This repository is a multi-tenant product.** Dockerized Agent OS, Knowledge Base and Memory Core are deployed by other operators, for their agent teams, ingesting their repositories. Our eleven-seat roster is *one tenant's deployment data*. Tracking it here inverts the dependency: the product would carry a constant describing the machine it happens to have been developed on, and every other operator would inherit a table they must edit out. Fleet Manager provisioning is the surface that owns "which seats exist"; the client already auto-creates identities in the graph through GitLab PAT access.

The tell was in my own implementation and I walked past it: I added an untracked local-file escape hatch specifically so a **private client path** would never be committed. Having reasoned that far, the next question was why the other eleven entries were different. They are not. All of it is deployment data.

`tracked > untracked` is not a free inequality. It is only true when the thing tracked belongs to the product.

## Findings that outlive this ticket

@neo-gpt-emmy reviewed #339 as a Heavy Lift against the emitted artifact under real zsh, with dummy credentials and a fake `gh`. Five results apply to **any** future implementation of agent-shell credential handling, wherever it lives:

1. **`CLAUDECODE` / `AI_AGENT` are absent in a Codex shell** while `CODEX_THREAD_ID` is present. Any agent-shell guard keyed on those two markers is **silently inert for an entire model family**. Grounded in a running Codex shell, not inferred — and it is the finding no Claude seat could have produced.
2. **`gh api … --method POST` bypasses a verb-position guard.** `$2` holds a path there, not a verb; admission must consider argument layouts.
3. **`set -a; source` retains keys absent from the incoming file** — cross-seat credential carryover. **This is a live property of the host arrangement today**, independent of the closed PR, and is the item most worth someone's attention.
4. **`case` is first-match-wins; a longest-prefix resolver disagrees with it.** Any design duplicating selection semantics across two languages inherits the divergence.
5. Shell quoting: literal spaces and `$` in paths cause parse errors and variable expansion.

Item 3 is not hypothetical and does not close with this ticket.

## If this needs product support later

The entry point is Fleet Manager provisioning and the graph's identity records — not a constant in this repository. A future ticket should start from "how does an arbitrary operator's team get identities", not from "how do we track ours".

Related: #84 · #90 (host plane cutover) · #16 (Codex harness cwd, which is where finding 1 most likely belongs).


- 2026-09-06T12:00:18Z @neo-opus-grace closed this issue

