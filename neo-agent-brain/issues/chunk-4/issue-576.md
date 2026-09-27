---
id: 576
title: The Fleet registry defines and starts an agent without a PAT
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-27T12:44:01Z'
updatedAt: '2026-09-27T14:41:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/576'
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
closedAt: '2026-09-27T14:41:25Z'
---
# The Fleet registry defines and starts an agent without a PAT

## Context

Operator ruling, 2026-09-27, on the Fleet Manager: *"FM ALWAYS ALWAYS ALWAYS needs a PAT. at least 1 per agent."* The seats' `.env` files are a temporary stopgap until FM is usable. neomjs/neo-agent-institution#289 reverts the FM Add agent path that registered seats without one (neomjs/neo-agent-institution#281).

Euclid found, reviewing that revert, that the Brain still makes PAT-less agents; I verified it on `dev` (`230593f`).

## The Problem

The Fleet registry is where an agent is born, and it does not hold the ruling:

- **`FleetRegistryService.defineAgent` treats the PAT as optional.** Its JSDoc reads "Create an agent and, optionally, store its credential" (`[opts.credential]`). A define without one writes the registry row; it is refused only when an orphan credential already exists.
- **`ai/scripts/fleet/onboardPeer.mjs` never passes one.** Its `define` step sends `{id, githubUsername, harnessType}`. Its `auth` step covers the harness's own sign-in, not the GitHub PAT.
- **Start proceeds without one.** `FleetLifecycleService` sets the credential env var only `if (pat != null)`.
- **No verb attaches a PAT later.** `storeCredential` has no caller.

A Fleet-started seat without a PAT runs `gh` with no token, and `gh` falls back to the machine's keyring account, the operator's. That is the silent misattribution #244 recorded.

The original registry design (neomjs/neo#13031, June) was "define agent = username + PAT". The optional credential came later, for the credential-free external rows the team converged on this morning, and the ruling retires them.

## The Fix

1. `defineAgent` requires `credential`: a creation without one is refused with a reason that names it. The orphan-credential guard and the two-store transaction stay.
2. `onboardPeer` reads the PAT from an environment variable named on the command line (never from argv, never printed), refuses to plan a `define` without it, and passes it through.
3. Start refuses an agent with no resolvable credential, before any provisioning, with a reason that names the missing PAT. That covers rows written before this change.
4. Specs that define agents without a credential pass one; the refusals get their own arms.

## Acceptance Criteria

- [ ] AC-1: `defineAgent` without `credential` throws, naming the missing PAT, and writes no row and no credential. With a credential, creation behaves as today.
- [ ] AC-2: `onboardPeer --commit` refuses without the named PAT env var and passes the PAT to `defineAgent` when it is set. The dry run says which variable it reads; the value is never printed.
- [ ] AC-3: Starting an agent whose credential does not resolve refuses before any checkout or harness spawn, with the reason.
- [ ] AC-4: The existing Fleet suites pass with credentials supplied; each new refusal arm is red on `dev` first.

## Out of Scope

- The FM side: neomjs/neo-agent-institution#289 restores its PAT-required ingress.
- Launch ownership (#565/#566): who launches a seat is a separate question from whether it holds a PAT.
- Containerized Fleet control and request-time seat identity (#83).

## Sweeps

- Live latest-open sweep: the latest 20 open Brain issues at 2026-09-27T12:43:01Z, plus issue and PR searches for "defineAgent credential"; no equivalent. #565 (closed) stamps `launchOwnerSince` and leaves the credential optional.
- A2A: the last 30 messages, all read-states; no claim on the registry's PAT rule. Euclid's review note and my defect-note `c3f36245` are the origin.
- MC sweep: "Fleet registry defineAgent without PAT credential optional agent authors as operator keyring gh fallback", 5 results. Grace's neomjs/neo#13031 design (username + PAT) and this morning's external-row convergence; no ruling for optional PATs.
- Own-assignment sweep: my open Brain tickets, none on the registry's credential rule.

Related: neomjs/neo-agent-institution#289 · #565 · #566 · #244 · #83

Origin Session ID: f3d50317-fe3b-4773-b4ac-db05e1fa6812

Authored by Ada (Claude Opus 5.5, Claude Code).


## Timeline

- 2026-09-27T12:44:02Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-27T12:44:02Z @neo-opus-ada added the `bug` label
- 2026-09-27T12:44:02Z @neo-opus-ada added the `ai` label
- 2026-09-27T12:44:02Z @neo-opus-ada added the `agent-os` label
- 2026-09-27T12:59:53Z @neo-opus-ada cross-referenced by PR #577
- 2026-09-27T14:12:56Z @neo-opus-ada referenced in commit `2a081dd` - "fix(fleet): both start gates refuse a blank stored PAT, and the contract copy says every agent holds one (#576)

A registry written before the requirement could store '' or whitespace, and
readCredentials returns it as stored, so the start gates apply the creation
test instead of a null check. The lifecycle env comment, the bridge's define
docs and onboardPeer's header no longer describe a PAT as optional."
- 2026-09-27T14:34:26Z @neo-opus-ada referenced in commit `c0104a4` - "fix(fleet): every agent holds its GitHub PAT, so the registry refuses to define or start one without it (#576)

Operator ruling: Fleet Manager needs at least one PAT per agent. defineAgent
now requires the credential after its shape checks, so the orphan guard and
the credential-less branch go. The provisioned start resolves the PAT once,
after its structural refusals and before any checkout, and hands that value
to the spawn; start() refuses a null PAT for every path, restarts included.
onboardPeer reads the PAT from the variable --credential-env names. The two
specs carrying the new arms join brain-unit's list."
- 2026-09-27T14:34:26Z @neo-opus-ada referenced in commit `1c96181` - "fix(fleet): both start gates refuse a blank stored PAT, and the contract copy says every agent holds one (#576)

A registry written before the requirement could store '' or whitespace, and
readCredentials returns it as stored, so the start gates apply the creation
test instead of a null check. The lifecycle env comment, the bridge's define
docs and onboardPeer's header no longer describe a PAT as optional."
- 2026-09-27T14:41:25Z @tobiu referenced in commit `742de62` - "Merge pull request #577 from neomjs/ada/576-registry-pat-required

fix(fleet): the registry refuses to define or start an agent without a PAT (#576)"
- 2026-09-27T14:41:25Z @tobiu closed this issue

