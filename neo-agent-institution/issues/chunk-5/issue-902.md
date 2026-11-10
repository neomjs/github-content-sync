---
id: 902
title: Agent Detail replaces a stopped seat's GitHub token
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees: []
createdAt: '2026-11-09T20:03:58Z'
updatedAt: '2026-11-09T20:05:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/902'
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
---
# Agent Detail replaces a stopped seat's GitHub token

## Context

Operator, 2026-11-09: "can we add a new PAT for a peer inside the FM UI?" Today the answer is no.

`@neo-fable-clio`'s PAT lacks `read:org`, so every one of the open-work producer's 1,344 retained pulses fails for that seat. Any failed read refuses the wake gate, which keeps the CI-finished wakes off (neomjs/neo-agent-brain#1535; measured in neomjs/neo-agent-brain#1541's intake). The operator can mint a token but has no product path to install it.

## The Problem

- **Brain has the operation.** neomjs/neo-agent-brain#815 shipped `replaceSeatCredential({id, credential, scope})`:
  - a stopped-seat and profile-custody gate;
  - pre-write proof of the new token;
  - per-seat delta writes with readback;
  - typed refusals (`authentication-rejected`, `insufficient-capability`, …).

  #815's body assigned "the cockpit wiring" to #503, and #503 closed without it.
- **The shell has no ingress for it.** `harness/fleetCapability.mjs` → `projectPublicCredentialIntent` projects only `defineAgent`, `connectTenant` and `setPlaneCredential`; `replaceSeatCredential` falls through to `null`.
- **The existing button doesn't fix this.** The credential prompt's "Update token" runs `setPlaneCredential`, which changes the seat's plane binding. It never changes the GitHub token in `credentials.enc` that the seat and the open-work producer read with.

## The Architectural Reality

- Credential bytes enter only through the shell's credential prompt (ADR 0034 §2.3). Brain's `FLEET_CREDENTIAL_METHODS` names `replaceSeatCredential` as a credential-bearing verb that the trusted launcher handles.
- A refused Start reports `nextStep: 'replace-seat-credential'` (Brain `FleetManager.mjs`). #1541 names `INSUFFICIENT_SCOPES` as a refusal that needs a new token.

## The Fix

1. **Shell.** `projectPublicCredentialIntent` admits `replaceSeatCredential` with the public intent `{id, scope}`. The credential prompt gets its own title and copy, e.g. "Replace token: use a new GitHub token for this agent's account". The bytes reach Brain only through the shell-owned provider, as with the other three verbs.
2. **Agent Detail.** A stopped seat offers "Replace token". The scope is `forge-and-bound-plane` when the seat has a bound plane row and `forge` otherwise, per #815's receipt fields. A running seat shows why the action is unavailable (`seat-running`).
3. **Next step.** A credential refusal on a seat (`replace-seat-credential`, `insufficient-capability`) links to that action instead of telling the operator to wait.
4. **Expiring tokens.** Operators may use PATs with an expiration date (operator, 2026-11-09), so a token's end must not arrive as a surprise refusal. GitHub returns a `GitHub-Authentication-Token-Expiration` header on API responses made with an expiring token. The seat's existing reads record that date, Agent Detail shows it, and the cockpit flags a seat whose token expires within 7 days, with "Replace token" as the action. A token without the header never expires and shows nothing.

## Acceptance Criteria

- [ ] The shell projects a `replaceSeatCredential` intent to `{id, scope}` and never reads credential bytes from it (unit).
- [ ] Agent Detail offers "Replace token" only for a stopped seat and explains the refusal for a running one (unit).
- [ ] A seat whose Start or read is refused for its credential shows "Replace token" as its next step (unit).
- [ ] A seat token whose recorded expiration is within 7 days is flagged in the cockpit, with "Replace token" as the action. A token without the header shows no expiry (unit).
- [ ] Installed witness, #815's deferred AC, carried by neomjs/neo-agent-brain#571: the operator replaces a refused seat's token in the product, and the seat's next read and Start succeed.

## Out of Scope

- Rotating the token of a running seat.
- Minting tokens.
- The plane-credential flow, which already exists.

## Related

neomjs/neo-agent-brain#815 (the operation) · #503 (closed; held the wiring) · neomjs/neo-agent-brain#571 (carries the witness) · neomjs/neo-agent-brain#1535 / neomjs/neo-agent-brain#1541 (the refusal this unblocks) · #424 (row 5: ordinary failures recover by product guidance)

Live latest-open + closed sweep: latest 20 open plus a search for `seat token Agent Detail replace credential refused` at 2026-11-09T20:05Z. No equivalent found; #503 is the closed predecessor whose scope this restores. Backlog band: 46 open (orange). Memory: Brain #815 / PR #1228 history.

unowned-rationale: shell plus cockpit work, needing an installed witness. Any maintainer with Institution context can take it.

Origin Session ID: 1ef4ba01-d8ab-4b0e-bb7c-1ee8426dc5fe

Retrieval Hint: `query_raw_memories("replace seat token Agent Detail replaceSeatCredential shell ingress")`

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-11-09T20:03:59Z @neo-opus-ada added the `enhancement` label
- 2026-11-09T20:03:59Z @neo-opus-ada added the `agent-os` label
- 2026-11-09T20:03:59Z @neo-opus-ada added the `ai` label

