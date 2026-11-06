---
id: 289
title: session-sunset Step 1 pulls by hand what Fleet Start now does
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-06T03:25:10Z'
updatedAt: '2026-11-06T05:15:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/289'
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
# session-sunset Step 1 pulls by hand what Fleet Start now does

## Context

neomjs/neo-agent-brain#1480 makes a Fleet Start bring each seat checkout current before the harness launches. It fast-forwards a clean checkout that is on its default branch, reinstalls when the checkout's lockfile changed, and runs the repository's own `seat:start`. `session-sunset` Step 1 is the discipline twin of that refresh: a clean primary clone switches to `dev`, pulls, and runs the Brain's `--migrate-config`. #1480 names Step 1's retirement as its deletion.

## The Problem

Once #1480 ships, Step 1's pull repeats what the next Start does, and it is the weaker copy: it never installs, and that is how nine of ten seats drifted weeks behind npm (#276). Two parts of Step 1 have no Start equivalent yet, though:

- **`git switch dev`.** Start fast-forwards only a checkout that is already on its default branch, and it never switches one.
- **`--migrate-config`.** Start runs no config migration. Whether it should is an open question, because ADR-0019 §5.6 says an overlay inherits a newer template instead of being reconciled at source.

## The Architectural Reality

- `.agents/skills/session-sunset/references/session-sunset-workflow.md:52`: Step 1. On a clean tree it runs `git switch dev`, `git pull origin dev`, then `node ai/scripts/setup/initServerConfigs.mjs --migrate-config` where that script exists.
- neomjs/neo-agent-brain#1480: Start's fast-forward follows the orchestrator's dev-sync ladder (`PrimaryRepoSyncService.syncDevRoot`). It skips a checkout that is off its default branch, dirty or diverged, and records why on the seat's start row.
- neomjs/neo-agent-brain `ai/scripts/setup/initServerConfigs.mjs`: `--migrate-config` patches pure Tier-1 import drift in place. It refuses broader drift with a typed conversion requirement.
- Decision Record impact: none for the skill itself. The `--migrate-config` disposition is `aligned-with` neomjs/neo-agent-brain#adr:aiconfig-reactive-provider-ssot, and ADR-0019 is read before that half is decided.

## The Fix

After neomjs/neo-agent-brain#1480 merges:

1. Step 1's pull becomes one line that points at Start's refresh.
2. Each remaining half gets a stated disposition:
   - `git switch dev`: kept in Step 1 or dropped. Start deliberately never switches a branch.
   - `--migrate-config`: kept, moved into the Brain's own `seat:start`, or retired if overlays inherit.
3. The package version bumps in the same PR.

## Acceptance Criteria

- [ ] Before Step 1 changes, one Fleet restart of a seat whose neo checkout was stale shows the refresh working. Its start row shows the fast-forward, the install and `seatStart: {state: 'ran'}`, its themes are current, and its `node_modules/neo-agent-skills` matches the pin. This receipt moved here from neomjs/neo#20429, which closes with its PR.
- [ ] Step 1 no longer pulls a checkout that the next Start fast-forwards, and says so in one line that links the refresh.
- [ ] The `git switch dev` and `--migrate-config` halves each carry a disposition: kept with its reason, moved, or retired.
- [ ] The corpus lint passes, with the version bump.

## Out of Scope

- Start's own behavior (neomjs/neo-agent-brain#1480).
- neo's `seat:start` implementation (neomjs/neo#20429). Its installed receipt is the first AC here, because Step 1 retires on that same restart.

## Related

neomjs/neo-agent-brain#1480 · #276 (closed into #1480) · #253 · #87 · neomjs/neo#20429

Live latest-open + closed sweep: checked the latest 20 open neomjs/neo-agent-skills issues and a closed search (`session-sunset Step 1 pull`) at 2026-11-06T03:24Z. The closed hits #276, #253 and #87 changed Step 1's text, and none retires its pull. No equivalent found.
A2A claim sweep: the latest 30 messages; no claim on `session-sunset` Step 1.
MC sweep: the #1480 drift probe (2026-11-06) found this ticket's open halves.
Own-assignment sweep: neomjs/neo-agent-brain#1480 is the producer, and this ticket is its named deletion.

Self-assigned: it lands after my neomjs/neo-agent-brain#1480 and reuses that context.

Origin Session ID: 0cbc6245-a3c6-4896-be26-dc550c38afa8
Retrieval Hint: "session-sunset Step 1 pull retire Fleet Start fast-forward git switch dev migrate-config ADR-0019 overlay inherits"


## Timeline

- 2026-11-06T03:25:10Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-06T03:25:12Z @neo-opus-grace added the `enhancement` label
- 2026-11-06T03:25:12Z @neo-opus-grace added the `ai` label
- 2026-11-06T03:25:12Z @neo-opus-grace added the `model-experience` label
- 2026-11-06T03:25:20Z @github-actions added the `needs-design-authority` label
- 2026-11-06T03:25:46Z @neo-opus-grace cross-referenced by #1480
- 2026-11-06T03:27:09Z @neo-opus-grace cross-referenced by PR #1500
- 2026-11-06T05:14:07Z @neo-opus-grace cross-referenced by PR #20432
- 2026-11-06T05:15:39Z @neo-opus-grace cross-referenced by #20429

